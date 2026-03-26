# 46 — Caching: Estratégias, Padrões e Armadilhas

> A diferença entre 1ms e 100ms em um sistema de alta escala. Caching é a ferramenta mais impactante de performance, mas introduz complexidade que entrevistadores adoram explorar.

---

## 1. Por que Caching Existe — A Física do Problema

```
Disco (SSD):   ~1ms de latência
RAM (memória): ~100ns de latência = 10.000x mais rápido

100M DAU × 10 queries/usuário/dia = 1B queries/dia = ~11K queries/s
PostgreSQL típico: suporta ~10K reads/s (com índices)
→ Estamos no limite com apenas 1 banco!

Com cache (80% hit rate):
→ 8.800 queries/s vão para o cache (nanosegundos)
→ 2.200 queries/s chegam ao banco
→ Banco fica com folga de 5x
```

---

## 2. Onde Cachear — As 4 Camadas

### Camada 1: Cache Externo (Redis/Memcached)

```
[Application Servers]
    │   │   │
    └───┴───┘
        │
    [Redis]  ← compartilhado por TODOS os servidores
        │
    [PostgreSQL]
```

**Vantagem:** estado global compartilhado — se servidor A cacheou algo, servidor B também se beneficia.  
**Uso típico:** dados de usuário, feeds, sessões, resultado de queries caras.

### Camada 2: Cache In-Process (in-memory)

```
[Server A]              [Server B]
  [Local Cache]           [Local Cache]
  user:123 = {...}        user:123 = {...} ← cópias independentes!
```

**Vantagem:** zero latência de rede (dados na memória do mesmo processo).  
**Desvantagem:** inconsistência entre servidores.  
**Uso típico:** configurações que raramente mudam, lookup tables, feature flags.

### Camada 3: CDN (Content Delivery Network)

```
São Paulo ─────► [CDN Edge SP] ─► Origem (Virginia)
                       ↑
                  Cache hit: 20ms
                       ↑
               Cache miss: 350ms (viagem até Virginia)
```

**Não é só para imagens:** CDNs modernos cacheiam respostas de API, páginas HTML, até lógica no edge.  
**Uso típico:** media (imagens, vídeos), assets estáticos, respostas de API públicas e imutáveis.

### Camada 4: Client-Side Cache

```
Browser:     Cache-Control headers, Service Workers
Mobile App:  SQLite local, in-memory cache
```

**Uso típico:** apps com funcionalidade offline, dados que o usuário acessa repetidamente.

---

## 3. Padrões de Arquitetura de Cache

### 3.1 Cache-Aside (Lazy Loading) — O Padrão Padrão ✅

```
Read:
App → Redis: GET user:123
  HIT: retorna dado ← O(1), ~1ms
  MISS:
    App → PostgreSQL: SELECT * FROM users WHERE id = 123
    App → Redis: SET user:123 {data} EX 300  (cache por 5min)
    App → Cliente: retorna dado
```

**Por que é o padrão:**
- Simples de implementar
- Cache só contém dados que foram realmente pedidos
- Falha do Redis → aplicação continua funcionando (degrada para banco)

**Desvantagem:** primeira requisição após expiração é lenta (cold miss).

### 3.2 Write-Through

```
Write:
App → Redis: SET user:123 {dados_novos}
Redis → PostgreSQL: INSERT/UPDATE (síncrono)
App ← resposta (após ambos confirmarem)
```

**Vantagem:** cache sempre atualizado com dado mais recente.  
**Desvantagens:**
- Escrita mais lenta (duas operações síncronas)
- Cache polui com dados que talvez nunca sejam lidos
- **Dual-write problem:** se Redis aceitar mas Postgres falhar → inconsistência

**Quando usar:** sistemas onde leitura inconsistente é inaceitável e escrita pode ser um pouco mais lenta.

### 3.3 Write-Behind (Write-Back)

```
Write:
App → Redis: SET user:123 {dados} (responde imediatamente)
          ↓ (assíncrono, em batch, depois)
Redis → PostgreSQL: flush periódico
```

**Vantagem:** escrita ultra-rápida.  
**Desvantagem:** risco de perda de dados se Redis cair antes do flush.  
**Quando usar:** analytics, métricas, dados onde pequena perda é aceitável.

### 3.4 Read-Through

Cache age como proxy — o cache busca do banco automaticamente no miss:
```
App → Cache: GET user:123
  MISS: Cache → Banco → Cache armazena → App recebe
```

É como Cache-Aside mas a lógica de populamento fica dentro do cache (ex: CDNs fazem isso).

---

## 4. Políticas de Eviction

Quando o cache enche, qual dado remover?

| Política | Lógica | Quando usar |
|---|---|---|
| **LRU** (Least Recently Used) | Remove o que foi acessado há mais tempo | Default em 90% dos casos |
| **LFU** (Least Frequently Used) | Remove o que foi acessado menos vezes | Workloads com "superstars" (alguns itens acessados 1000x mais) |
| **FIFO** | Remove o mais antigo | Raramente ideal |
| **TTL** (Time To Live) | Remove automaticamente após X segundos | Dados que têm "validade" natural |

**Redis defaults:** LRU com configuração `maxmemory-policy allkeys-lru`.

---

## 5. Os Problemas de Cache que Entrevistadores Adoram

### 5.1 Cache Stampede (Thundering Herd)

**Cenário:**
```
Feed da homepage está cacheado com TTL de 60 segundos
100K usuários acessam simultâneos o site
TTL expira → 100K requests simultâneos vão ao banco
Banco não aguenta → cascata de falhas
```

**Solução 1 — Request Coalescing (Single Flight):**
```
Quando TTL expira e 100K requests chegam:
→ Apenas o PRIMEIRO request vai ao banco
→ Os outros 99.999 ficam em espera (blocking)
→ Quando o primeiro voltar → responde todos
```

Implementação com Redis:
```
SETNX lock:feed:homepage 1 EX 5  ← Distributed Lock
  Se adquiriu: busca do banco → armazena → libera lock
  Se não adquiriu: aguarda lock liberar → lê do cache
```

**Solução 2 — Cache Warming (Proactive Refresh):**
```
Background job roda ANTES do TTL expirar:
  Aos 55 segundos (TTL de 60s) → refresh do cache
  Nunca deixa expirar "organicamente"
  
Resultado: usuário nunca experiencia cold miss
Trade-off: consumo de recursos para refresh constante
```

**Solução 3 — Staggered TTL (TTL com jitter):**
```
Em vez de TTL fixo de 60s para todos:
  TTL = 60s + random(0, 10s)
  
Diferentes entradas expiram em momentos diferentes
→ Sem avalanche simultânea
```

### 5.2 Cache Inconsistency (Stale Data)

**Cenário:**
```
Alice atualiza sua foto de perfil
→ Banco atualizado ✅
→ Cache ainda tem foto antiga ❌
→ Outros usuários veem foto antiga por até 5 minutos
```

**Abordagem 1 — Invalidate on Write:**
```python
def update_profile_photo(user_id, new_photo_url):
    db.update("UPDATE users SET photo = ? WHERE id = ?", new_photo_url, user_id)
    redis.delete(f"user:{user_id}")  ← invalida imediatamente
```
Próxima leitura → miss → busca dado novo do banco → popula cache.

**Abordagem 2 — TTL Curto:**
```
TTL = 60s para dados de perfil
→ Máximo 60s de staleness
→ Aceitável para foto de perfil (usuário pode esperar)
→ Inaceitável para saldo bancário
```

**Abordagem 3 — Eventual Consistency Explícita:**
```
"Aceitamos que o feed pode estar atrasado em até 5 minutos.
Documentado no SLA. Usuários entendem."
```

**Regra para entrevistas:** a resposta certa depende de criticidade:
```
Saldo bancário → invalidar imediatamente (zero stale)
Feed social    → TTL curto (1-5min de stale é OK)
Ranking global → pré-computado, stale de horas é OK
Configuração   → TTL longo ou push-based refresh
```

### 5.3 Hot Keys (Cache Hotspot)

**Cenário:**
```
Taylor Swift acabou de lançar um álbum
→ 10M usuários/minuto acessam seu perfil
→ Redis key "user:taylor_swift" → único nó Redis
→ Esse nó fica saturado mesmo com cache hit
```

**Solução 1 — Replicação do hot key:**
```python
# Ao invés de 1 key:
redis.get("user:taylor_swift")

# Usar N cópias com load balancing:
replica_id = random.randint(0, 9)
redis.get(f"user:taylor_swift:{replica_id}")

# Ao escrever: atualiza todas as réplicas
for i in range(10):
    redis.set(f"user:taylor_swift:{i}", data)
```

**Solução 2 — Local cache como fallback:**
```python
# Primeiro tenta cache local (in-process)
if user_id in local_cache:
    return local_cache[user_id]

# Depois Redis
data = redis.get(f"user:{user_id}")
if data:
    local_cache[user_id] = data  # guarda localmente por 30s
    return data
```

Evita que cada request chegue ao Redis para itens ultra-populares.

---

## 6. Quando Introduzir Cache em Entrevistas

**Sinal 1 — Read-heavy workload:**
```
"100M DAU × 20 reads/dia = 2B reads/dia = 23K reads/s
PostgreSQL suporta ~10K reads/s com índices
→ Precisamos de cache para absorver a diferença"
```

**Sinal 2 — Queries caras:**
```
"Calcular o feed personalizado de cada usuário requer JOIN de
posts + follows + likes de múltiplas tabelas
→ Pré-computar e cachear o feed por 60 segundos reduz
   a carga do banco em ~100x"
```

**Sinal 3 — Latência crítica:**
```
"NFR: P99 < 100ms
PostgreSQL: P99 = 80ms (no limite)
Com Redis cache: P99 = 5ms ← margem confortável"
```

---

## 7. Como Falar sobre Cache em Entrevistas

### Template de introdução

```
"Para atender ao requisito de [LATÊNCIA/ESCALA], vou introduzir
um cache Redis aqui.

O que vou cachear: [feed do usuário / dados de perfil / resultado da query X]
Chave do cache: [user:{id}:feed / post:{id} / trending:global]
TTL: [60 segundos / 5 minutos / sem TTL com invalidação explícita]

Usarei cache-aside: a aplicação verifica o Redis primeiro, e no
miss busca do banco e popula o cache.

[Se relevante:] O risco de inconsistência é aceitável aqui porque
[feeds podem ter 60s de lag / dados de perfil raramente mudam].

[Se relevante:] Para prevenir cache stampede no TTL, vou usar
cache warming — um background job que refresca o cache antes
de expirar."
```

---

## 8. Redis Além do Cache

Redis não é apenas um cache — é uma plataforma de estruturas de dados distribuída:

```
String:       SET user:123 {json}         → Cache simples
Hash:         HSET user:123 name "Alice"  → Objeto com campos
List:         LPUSH feed:123 post_id      → Feed ordenado por inserção
Set:          SADD user:123:follows user:456  → Relações únicas
Sorted Set:   ZADD leaderboard 1500 alice → Ranking com score
Bitmap:       SETBIT activity:date userId 1  → Presença de usuários
HyperLogLog:  PFADD dau:2025-01 user_id   → Contagem aproximada única
Pub/Sub:      PUBLISH channel message     → Notificações em tempo real
Streams:      XADD events * key value     → Event sourcing leve
```

### Exemplos de uso em sistemas reais

| Estrutura | Uso |
|---|---|
| Sorted Set | Leaderboards, trending topics (score = view count) |
| Bitmap | "Usuários que fizeram login hoje" (1 bit por user) |
| HyperLogLog | DAU/MAU sem armazenar todos os IDs |
| List | Fila de trabalhos (LPUSH + BRPOP) |
| Set | Tags, categorias, membros de grupo |
| Pub/Sub | Chat em tempo real, notificações |

---

## 9. Diagrama de Arquitetura com Cache

```
                    [Clients]
                        │
                  [Load Balancer]
                   │    │    │
                  [S1] [S2] [S3]  ← Application Servers
                   │    │    │
               [In-Process Cache] ← Layer 1 (local, 30s TTL)
                        │ miss
                   [Redis Cluster] ← Layer 2 (shared, 5min TTL)
                     │   │   │
                   [R1] [R2] [R3]  ← Redis Shards
                        │ miss
               [Read Replicas (PG)]  ← Layer 3 (read)
                        │ write
               [PostgreSQL Primary]  ← Layer 4 (source of truth)
```

---

## Checklist para Entrevistas

```
□ Identifiquei o bottleneck que justifica o cache?
□ Especifiquei o que está sendo cacheado (chave e valor)?
□ Escolhi o TTL apropriado para o nível de tolerância à inconsistência?
□ Mencionei o padrão de acesso (cache-aside, write-through)?
□ Abordei possíveis problemas (stampede, hot keys, inconsistência)?
□ Considerei o caso de falha do cache (graceful degradation)?
□ O TTL respeita os requisitos de freshness do sistema?
```

---

*Tópicos relacionados: `16_redis_deep_dive.md`, `06_thundering_herd.md`, `02_rate_limiting_e_seguranca.md`, `33_design_newsfeed_fanout.md`*
