# 45 — Sharding: Particionamento Horizontal de Bancos de Dados

> Como dividir dados entre múltiplas instâncias de banco quando uma única não é mais suficiente. Um dos tópicos mais cobrados em entrevistas de System Design para posições senior+.

---

## 1. Por que Sharding Existe

### A evolução natural

```
Fase 1 — Startup:
  Banco único PostgreSQL
  Funciona perfeitamente para 99% das empresas

Fase 2 — Crescimento:
  Write throughput: 20K/s > limite de 10K/s do banco
  Opção: vertical scaling (máquina maior)
  AWS max: 50K writes/s, 140TB storage

Fase 3 — Hipercrescimento:
  Write throughput: 500K/s
  Storage: 500TB
  Sem hardware suficientemente grande
  → Sharding é inevitável
```

### Escalabilidade vertical vs horizontal

| Escalabilidade | Abordagem | Limite |
|---|---|---|
| **Vertical** (scale-up) | Máquina maior, mais RAM/CPU/disco | Físico (hardware máximo disponível) |
| **Horizontal** (scale-out) | Múltiplas máquinas menores | Teórico ilimitado |

**Sharding = escalabilidade horizontal para bancos de dados.**

---

## 2. O que é Sharding

**Sharding** é o processo de dividir dados entre múltiplas instâncias de banco, onde cada shard:
- É um banco de dados standalone independente
- Tem seu próprio CPU, RAM, storage e connection pool
- Armazena apenas um **subconjunto** dos dados

```
Sem sharding:
[users: 500M linhas]  ← 1 banco, sobrecarregado

Com sharding por user_id:
[users: id 0-166M]    ← Shard 1
[users: id 167M-333M] ← Shard 2  
[users: id 334M-500M] ← Shard 3
```

---

## 3. Shard Key — A Decisão Mais Importante

A **shard key** é o campo usado para determinar em qual shard um registro fica.

### Propriedades de uma boa shard key

| Propriedade | Descrição | Mau exemplo | Bom exemplo |
|---|---|---|---|
| **Alta cardinalidade** | Muitos valores únicos | `is_premium` (2 valores) | `user_id` (M valores) |
| **Distribuição uniforme** | Shards com carga parecida | `created_date` (novos usuarios em 1 shard) | `user_id` (hash uniforme) |
| **Alinhamento com queries** | Dados relacionados juntos | shard por `country` para app global | shard por `user_id` para queries de usuário |

### Exemplos de escolha de shard key

**Instagram (posts e comentários):**
```
Query principal: "Carregar todos os posts de um usuário"
→ Shard key: user_id
→ Todos os posts de alice ficam no mesmo shard
→ Query = 1 hop, 1 shard ✅

Trade-off: posts de usuários populares ficam em 1 shard (hotspot)
Solução: compound shard key = user_id + bucket_id
```

**E-commerce (pedidos):**
```
Query principal: "Ver detalhes de um pedido específico"
→ Shard key: order_id
→ Alta cardinalidade, distribuição uniforme ✅
→ Histórico completo do cliente = cross-shard ⚠️
```

**Sistema de pagamentos:**
```
Query principal: "Transferir de conta A para conta B"
→ Shard key: account_id
→ Problema: transferência entre contas = cross-shard transaction ⚠️
```

---

## 4. Estratégias de Distribuição

### 4.1 Range-Based Sharding (mais intuitivo)

Divide o espaço de valores em intervalos:

```
Shard 1: user_id 0 - 10M
Shard 2: user_id 10M - 20M
Shard 3: user_id 20M+
```

**Problema — hotspot temporal:**
```
user_ids são monotonicamente crescentes
→ Todos os novos usuários vão para o shard mais alto
→ Shard 3 está sempre sobrecarregado (mais usuários ativos = mais recentes)
```

**Quando funciona:** dados que naturalmente crescem em "buckets" balanceados (ex: por região geográfica com distribuição conhecida).

### 4.2 Hash-Based Sharding (padrão de produção)

```
shard = hash(user_id) % num_shards

hash("alice")  = 5432167 % 3 = 1 → Shard 1
hash("bob")    = 7654321 % 3 = 2 → Shard 2
hash("carol")  = 3218765 % 3 = 0 → Shard 0
```

✅ Distribuição uniforme (hash function garante)  
❌ Problema ao adicionar shards: quase tudo precisa ser redistribuído

**Problema do rebalanceamento:**
```
Antes (3 shards): alice → hash % 3 = 1 → Shard 1
Depois (4 shards): alice → hash % 4 = ? → Shard diferente!
→ ~75% dos dados precisam migrar quando adicionamos 1 shard
```

### 4.3 Consistent Hashing (solução para rebalanceamento)

Em vez de modulo, usa um **anel virtual**:

```
Hash Ring (0 a 2^32):

           hash=0
              │
   Shard C ───┼─── Shard A
              │
   Shard B ───┘

Para encontrar o shard de "alice":
1. hash("alice") = 1234567
2. Localizar 1234567 no anel
3. Caminhar no sentido horário até o próximo shard
4. Esse shard é o responsável
```

**Adicionando Shard D:**
```
Antes: Shard A responde por segmento [8000, 3000]
Shard D inserido em posição 1500:
Depois: Shard D responde por [8000, 1500], Shard A responde por [1500, 3000]

Apenas dados do segmento que Shard D assumiu precisam migrar!
→ 1/N dos dados (onde N = número de shards)
```

**Virtual Nodes:** Cada shard físico é responsável por múltiplos pontos no anel, garantindo distribuição mais uniforme:
```
Shard A → posições [100, 1500, 3000, 7500]
Shard B → posições [500, 2000, 5000, 9000]
```

### 4.4 Directory-Based Sharding (máxima flexibilidade)

Mantém um lookup table explícito:
```
[Directory Service]
user_id → shard
alice   → Shard 3
bob     → Shard 1
carol   → Shard 7 (movida por ser celebrity)
```

✅ Flexibilidade total (pode mover qualquer usuário)  
❌ **Latência dupla:** toda query = 1 hop no directory + 1 hop no shard  
❌ **Single point of failure:** se o directory cai, nada funciona  
⚠️ Raramente correto em entrevistas — evitar a menos que justificável

---

## 5. Desafios do Sharding e Como Resolver

### 5.1 Hotspot / Hot Shard

**Problema:** Messi no Instagram → 500M seguidores → shard dele está sob pressão constante

**Solução 1 — Compound Shard Key:**
```
Antes: shard_key = user_id
       hash("messi") → Shard 2 (sobrecarregado)

Depois: shard_key = user_id + random_suffix (0-9)
        hash("messi_0") → Shard 2
        hash("messi_1") → Shard 5
        hash("messi_3") → Shard 8
        
Writes distribuídos entre 10 shards
Reads: precisa consultar os 10 e agregar (trade-off aceitável para celebrities)
```

**Solução 2 — Shard de celebrities (Directory-Based para casos especiais):**
```
IF user.is_celebrity:
    → Shard dedicado com hardware especializado
ELSE:
    → Hash-based sharding normal
```

### 5.2 Cross-Shard Queries

**Problema:** Query "Top 10 posts mais populares da plataforma" precisa dados de todos os shards.

```
Sem otimização:
GET /trending → consulta todos os 20 shards → agrega 20 listas → ordena → retorna top 10
Latência: proporcional ao número de shards
```

**Solução 1 — Cache de queries globais:**
```
Cache Redis: "trending_posts" → [top 10 calculados]
TTL: 5 minutos
Miss: scatter-gather + popular no cache
Hit: O(1) → single-digit ms
```

**Solução 2 — Desnormalização:**
```
Manter tabela separada "global_trending" em um shard especial
Atualizada por background job que agrega todos os shards periodicamente
```

**Regra de ouro:** Cross-shard queries devem ser a exceção, nunca a norma. Se você as tem frequentemente, sua shard key está errada.

### 5.3 Transações Cross-Shard

**Problema:** Transferência bancária de Alice (Shard 1) para Bob (Shard 3)

```
Sem proteção:
1. Debita Alice no Shard 1 ✅
2. Credita Bob no Shard 3 ← crash aqui! → Alice perdeu $50, Bob não recebeu
```

**Solução 1 — Two-Phase Commit (2PC):**
```
Coordinator:
  Fase 1 (Prepare): "Shard 1, você consegue debitar $50 de Alice?"
                    "Shard 3, você consegue creditar $50 para Bob?"
  
  Fase 2 (Commit): Se ambos disseram SIM → "Executem!"
                   Se qualquer um disse NÃO → "Abortem!"
```

Problemas do 2PC:
- Lento (2 round-trips de rede)
- Frágil (se coordinator cai durante commit → shards ficam bloqueados)
- Na prática, raramente usado em sistemas de alta escala

**Solução 2 — Saga Pattern (preferível):**

```
Cada operação tem uma "compensating action" correspondente:

Passo 1: Debitar Alice no Shard 1
  ↓ (falha no passo 2)
Passo 2 falhou: Creditar Bob no Shard 3
  ↓
Compensação: REVERTER — Creditar Alice de volta no Shard 1

Não há rollback automático, mas cada passo tem seu "undo"
```

**Solução 3 — Evitar (melhor opção):**
```
Se possível, redesenhe o sistema para manter dados relacionados no mesmo shard.

Exemplo: Para pagamentos, shard por payment_id (não por user_id).
A transferência completa fica em 1 shard.
```

---

## 6. Como Falar sobre Sharding em Entrevistas

### Quando sharding é necessário?

**Calcule primeiro:**

```
Storage: 
  500M usuários × 5KB = 2.5TB → PostgreSQL aguenta (140TB max) → NÃO shard
  
Write throughput:
  50K writes/s > 10K writes/s (Postgres max) → SHARD
  
Read throughput:
  100M DAU × 10 queries/dia / 86400s ≈ 11.5K reads/s
  Com read replicas consegue cobrir → read replicas antes de shard
```

> **Sinal de senioridade:** mostrar que você calculou e concluiu que sharding NÃO é necessário é tão impressionante quanto saber quando é.

### Os 4 passos quando sharding é necessário

```
1. Proponha a shard key:
   "Para este sistema, a query dominante é buscar posts por usuário,
   então vou shardar por user_id."

2. Escolha a estratégia de distribuição:
   "Usarei hash-based sharding com consistent hashing para facilitar
   adição de novos shards sem redistribuição massiva."
   (Isso é o default esperado — pode ser implícito em entrevistas senior)

3. Exponha os trade-offs:
   "O trade-off é que queries globais como trending posts precisarão
   consultar todos os shards. Vou resolver isso com cache Redis e
   um background job de pré-computação."

4. Explique o crescimento:
   "Começamos com 10 shards. Com consistent hashing, adicionar
   o 11º shard migra apenas 1/11 dos dados automaticamente."
```

---

## 7. Diagrama Completo

```
                    Clientes
                       │
               [API Gateway]
                       │
               [Application]
                       │
            ┌──────────┴──────────┐
            │  Routing Layer      │
            │  hash(user_id) % N  │
            └────┬────┬────┬──────┘
                 │    │    │
              ┌──▼─┐ ┌▼──┐ ┌▼──┐
              │ S1 │ │ S2│ │ S3│   ← Shards
              │(PG)│ │(PG│ │(PG│
              └──┬─┘ └─┬─┘ └─┬─┘
                 │     │     │
              ┌──▼─┐ ┌─▼─┐ ┌─▼─┐
              │Rep1│ │Rep│ │Rep│   ← Read Replicas
              └────┘ └───┘ └───┘
                              
              [Cache Redis]         ← Cross-shard queries pré-computadas
              
              [Background Jobs]     ← Aggregate global stats
```

---

## 8. Resumo das Decisões-Chave

| Decisão | Opção A | Opção B | Quando B |
|---|---|---|---|
| Escalar primeiro | Vertical (máquina maior) | Sharding | Quando vertical atinge limite |
| Shard key | `user_id` | `order_id`, `post_id` | Baseado na query dominante |
| Distribuição | Range-based | Hash + Consistent Hashing | Quando distribuição uniforme importa |
| Cross-shard queries | Scatter-gather | Cache + pré-computação | Quando latência é crítica |
| Transações cross-shard | 2PC | Saga Pattern | Sistemas de alta escala |
| Hotspots | Compound shard key | Shard dedicado | Celebrities/itens viral |

---

*Tópicos relacionados: `03_consistencia_e_disponibilidade.md`, `27_consistent_hashing.md`, `23_dynamodb_deep_dive.md`, `16_redis_deep_dive.md`*
