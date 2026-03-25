# 17 — Design Tinder: Matching, Geolocalização e Swipes em Escala

> **Questão:** Desenhe um sistema como Tinder — app de relacionamento com swipe, matching e notificações.  
> **Características únicas:** geospatial queries para stack de perfis, consistência em swipes (sem match perdido), evitar perfis repetidos, pico de writes em swipes.

---

## 1. Requisitos

**Funcionais:**
- Configurar preferências (faixa etária, gênero, distância)
- Ver stack de perfis próximos que correspondem às preferências
- Swipe left (não curtiu) ou right (curtiu)
- Se dois usuários se curtiram → match + notificação

**Não Funcionais:**
- **Consistência em swipes:** match não pode ser perdido
- **Baixa latência no stack:** < 300ms para carregar perfis
- **Escala de writes:** 10M usuários × 100 swipes/dia → ~10.000 swipes/seg (pico)
- Evitar mostrar perfis já vistos

---

## 2. O Problema do Stack (Feed de Perfis)

**Query ingênua — por que não funciona:**
```sql
SELECT p.*
FROM profiles p
WHERE p.age BETWEEN $min_age AND $max_age
  AND p.gender = $preferred_gender
  AND ST_Distance(p.last_location, ST_Point($lat, $lng)) < $max_distance * 1000
  AND p.id NOT IN (SELECT swiped_on FROM swipes WHERE user_id = $user_id)
ORDER BY RANDOM()
LIMIT 100;
```

**Problemas:**
1. `ST_Distance` faz full table scan — lento
2. `NOT IN (subquery de milhões de rows)` — lento
3. Com 10M usuários: impraticável sem otimizações

---

## 3. Solução: Elasticsearch + Pre-computation

### Elasticsearch para Geo Queries

```json
{
  "query": {
    "bool": {
      "must": [
        { "range": { "age": { "gte": 25, "lte": 35 }}},
        { "term": { "gender": "female" }}
      ],
      "filter": {
        "geo_distance": {
          "distance": "50km",
          "last_location": { "lat": -23.5, "lon": -46.6 }
        }
      }
    }
  }
}
```

### Pre-computation da Stack (Cron Job Noturno)

```
Para cada usuário ativo:
  1. Rodar geo query no Elasticsearch
  2. Filtrar por preferências
  3. Salvar resultado no Redis: "stack:{user_id}" → [id1, id2, ..., id100]

Quando usuário abre o app:
  → GET "stack:{user_id}" do Redis → O(1) → < 1ms
```

**Invalidação do cache:**
- Usuário muda preferências → deletar cache + recalcular
- Usuário muda localização significativamente → invalidar
- Perfil na stack é deletado → tratar na entrega (skip)

**Proactive loading:**
```python
# Quando stack tem < 20 perfis restantes:
def on_swipe(user_id):
    remaining = redis.llen(f"stack:{user_id}")
    if remaining < 20:
        trigger_async_regeneration(user_id)
```

---

## 4. Consistência em Swipes: O Race Condition

**Problema:**
```
T=0ms: Alice swipe right em Bob → servidor A
T=1ms: Bob swipe right em Alice → servidor B
T=2ms: Servidor A grava no banco
T=2ms: Servidor B grava no banco
T=3ms: Servidor A verifica "Bob curtiu Alice?" → réplica desatualizada → "Não"
T=3ms: Servidor B verifica "Alice curtiu Bob?" → réplica desatualizada → "Não"
→ Match perdido para sempre!
```

### Opção 1: Eventual Consistency + Reconciliação (Simples)

```python
# Cron job a cada hora:
def reconcile_matches():
    unmatched_pairs = db.query("""
        SELECT a.user_id, b.user_id
        FROM swipes a
        JOIN swipes b ON a.swiped_on = b.user_id 
                      AND b.swiped_on = a.user_id
        WHERE a.decision = 'yes' AND b.decision = 'yes'
        AND NOT EXISTS (
            SELECT 1 FROM matches
            WHERE (user1 = a.user_id AND user2 = b.user_id)
               OR (user1 = b.user_id AND user2 = a.user_id)
        )
    """)
    for user1, user2 in unmatched_pairs:
        create_match(user1, user2)
        send_notification(user1, "Você tem um novo match!")
        send_notification(user2, "Você tem um novo match!")
```

**Trade-off:** notificação de match pode atrasar até 1 hora. Aceitável?

### Opção 2: Redis para Check Atômico

```python
def process_right_swipe(swiper_id, target_id):
    # Chave ordenada para evitar "alice_bob" e "bob_alice" serem chaves distintas
    key = f"likes:{min(swiper_id, target_id)}:{max(swiper_id, target_id)}"
    
    # Operação atômica: adicionar meu like E verificar se o outro já curtiu
    pipe = redis.pipeline()
    pipe.sadd(key, swiper_id)    # adiciono eu ao set
    pipe.smembers(key)            # verifico membros do set
    pipe.expire(key, 86400 * 30) # TTL de 30 dias
    results = pipe.execute()
    
    members = results[1]
    
    # Salvar no banco (pode ser async)
    db.save_swipe(swiper_id, target_id, 'yes')
    
    if target_id in members:  # o outro já curtiu!
        create_match(swiper_id, target_id)
        return {"matched": True}
    
    return {"matched": False}
```

**Por que funciona:** Redis é single-threaded — `SADD` e `SMEMBERS` são serializados. Sem race condition.

### Opção 3: PostgreSQL com Linha Única por Par

```sql
-- Uma linha por par, em vez de duas
CREATE TABLE swipe_pairs (
    user_a UUID,  -- menor ID
    user_b UUID,  -- maior ID
    a_decision ENUM('pending', 'yes', 'no') DEFAULT 'pending',
    b_decision ENUM('pending', 'yes', 'no') DEFAULT 'pending',
    PRIMARY KEY (user_a, user_b)
);

-- Alice (ID menor) curte Bob:
INSERT INTO swipe_pairs (user_a, user_b, a_decision)
VALUES ('alice_id', 'bob_id', 'yes')
ON CONFLICT DO UPDATE SET a_decision = 'yes'
RETURNING b_decision;
-- Se retornou b_decision = 'yes' → match!
```

PostgreSQL bloqueia a row em writes concorrentes → race condition eliminado.

---

## 5. Escala de Writes: Por que Cassandra

```
10M usuários × 100 swipes/dia ÷ 86.400s = 11.574 swipes/segundo
Pico (noite, fim de semana): 100.000 swipes/segundo
```

**PostgreSQL:** ~5.000-20.000 writes/s por nó (precisa de 5-20 nós)  
**Cassandra:** ~100.000+ writes/s por nó, escala linearmente adicionando nós

**Por que Cassandra escreve mais rápido:**

```
PostgreSQL write path:
  1. WAL (Write-Ahead Log) → disco sequential  ← lento
  2. Buscar página no disco (random seek)       ← lento
  3. Modificar página                           ← lento

Cassandra write path:
  1. Commit log → disco sequential              ← lento (mas sequencial)
  2. MemTable → memória RAM                     ← rápido
  → Responde ao cliente imediatamente!
  
  Background: periodicamente flush MemTable → SSTable (sequential)
```

**Trade-off:** Cassandra é eventual consistent por padrão. Para swipes isso pode ser problemático (ver seção 4).

---

## 6. Evitar Perfis Repetidos: Bloom Filter

**Problema de espaço:**
```
10M usuários × 1.000 swipes × UUID (16 bytes) = 160 GB
Por 30 dias: ~5 TB
```

**Solução: Bloom Filter por usuário**

```python
# Bloom filter para 10K swipes com 0.1% false positive rate:
# Usa ~15KB por usuário em vez de ~160KB

# Ao swipe:
bloom_filter.add(target_id)

# Ao gerar stack:
candidates = elasticsearch.geo_query(user_preferences)
filtered = [c for c in candidates if c.id not in bloom_filter]
```

**Trade-off:** 0.1% de false positive → 1 em 1.000 perfis novos pode ser incorretamente filtrado como "já visto".

**Dica de produto:** limpar o bloom filter após 30 dias → usuário pode ver perfis antigos novamente, o que melhora a UX (a pessoa pode ter mudado).

---

## 7. Arquitetura Final

```
Client (iOS/Android)
  ↓
API Gateway (JWT auth, rate limiting)
  ├── Profile Service → PostgreSQL (perfis)
  │                   → Elasticsearch (geo queries, CDC sync)
  ├── Stack Cache     → Redis (stacks pré-computadas)
  │       ↑
  │  Cron Job (noturno) → gera stacks via Elasticsearch
  ├── Swipe Service   → Cassandra (alta escala de writes)
  │   └── Redis       → check atômico de mutual likes
  │   └── Match Table → PostgreSQL (matches confirmados)
  └── Notification    → APNs (iOS) / FCM (Android)

Bloom Filters: Redis ou DynamoDB (por usuário, limpeza mensal)
```

---

## Referências

- [Tinder Engineering — Geosharded Recommendations](https://medium.com/tinder/geosharded-recommendations-part-1-sharding-approach-d5d54e0ec77a)
- [Hello Interview — Tinder Breakdown](https://hellointerview.com)
- [Cassandra — Write Path Architecture](https://cassandra.apache.org/doc/latest/)
