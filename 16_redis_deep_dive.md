# 16 — Redis Deep Dive: Estruturas de Dados Distribuídas

> **Por que estudar Redis a fundo:** Redis aparece em praticamente todo system design — como cache, rate limiter, lock distribuído, leaderboard, geospatial index, message broker leve. Entender seus internals permite justificar escolhas e identificar limitações.

---

## 1. O que é Redis

Redis é um **servidor de estruturas de dados em memória**, single-threaded.

Três características fundamentais:

### 1.1 Single-Threaded
Incomum para sistemas distribuídos, mas traz uma vantagem enorme:

```
Requisição A: lê saldo (100) → vai debitar 50
Requisição B: lê saldo (100) → vai debitar 80

Com multi-threading (problema):
  A lê 100, B lê 100
  A debita → saldo = 50
  B debita → saldo = 20 (ERRADO! deveria ser 20 ou recusar)

Com single-threading (correto):
  A lê 100, A debita → saldo = 50
  B lê 50 → recusa (insuficiente para 80)
```

Isso garante que operações são serializadas por design — sem locks explícitos.

### 1.2 Em Memória
- Operações em ~microsegundos
- Mas: dados podem ser perdidos se o processo morrer (configurável)
- Persistência: RDB (snapshots periódicos) ou AOF (append-only file, mais durável)

### 1.3 Servidor de Estruturas de Dados
O valor de um key-value no Redis não precisa ser uma string. Pode ser:
- String / Number
- Hash (mapa de campos → valores)
- List (lista ordenada)
- Set (conjunto sem duplicatas)
- Sorted Set (conjunto ordenado por score)
- Stream
- Geospatial Index

---

## 2. Sharding e Keyspace

**Como o Redis distribui dados em cluster:**

```
hash(key) % 16384 = slot_number
slot_number → broker (servidor) responsável

Exemplo:
  hash("user:123") = 7834 → broker_2
  hash("user:456") = 3210 → broker_1
  hash("user:789") = 12345 → broker_3
```

**Por que o keyspace importa:**

Se você tem uma chave muito "quente" (ex: `popular_ad_nike`), ela sempre vai para o mesmo broker, que pode ficar sobrecarregado.

**Solução: compound key**
```python
# Em vez de uma chave única para um ad muito popular:
key = "ad:nike_123"  # sempre vai para o mesmo broker

# Distribuir em N=10 partições:
shard = random.randint(0, 9)
key = f"ad:nike_123:{shard}"  # distribui entre 10 brokers

# Na leitura, agregar os 10 shards
total_clicks = sum(redis.get(f"ad:nike_123:{i}") for i in range(10))
```

---

## 3. Cache com Redis

**Fluxo padrão (cache-aside):**
```python
def get_event(event_id):
    # 1. Tentar o cache primeiro
    cached = redis.get(f"event:{event_id}")
    if cached:
        return json.loads(cached)
    
    # 2. Cache miss → buscar do banco
    event = db.query("SELECT * FROM events WHERE id = $1", event_id)
    
    # 3. Salvar no cache com TTL
    redis.set(f"event:{event_id}", json.dumps(event), ex=3600)  # 1h
    
    return event
```

**TTL (Time to Live):**
```python
redis.set("key", "value", ex=60)    # expira em 60 segundos
redis.set("key", "value", px=1000)  # expira em 1000 milissegundos
```

**Políticas de eviction (quando a memória acaba):**
- `noeviction`: retorna erro (não recomendado para cache)
- `allkeys-lru`: evicta o menos recentemente usado (padrão para cache)
- `volatile-lru`: evicta apenas keys com TTL definido

---

## 4. Rate Limiter com Redis

**Problema:** limitar a 5 requests por minuto por usuário.

```python
def check_rate_limit(user_id, limit=5, window=60):
    key = f"rate:{user_id}"
    
    count = redis.incr(key)   # incrementa e retorna novo valor
    if count == 1:
        redis.expire(key, window)  # define TTL apenas na primeira vez
    
    if count > limit:
        return False, count
    return True, count
```

**Problema desta abordagem (Fixed Window):**
```
Minuto 10:59 → usuário faz 5 requests (atingiu o limite)
Minuto 11:00 → janela reinicia → pode fazer mais 5 requests
Total: 10 requests em 2 segundos!
```

**Solução melhorada: Sliding Window com Redis Sorted Set:**
```python
def check_rate_limit_sliding(user_id, limit=5, window=60):
    key = f"rate:{user_id}"
    now = time.time()
    window_start = now - window
    
    pipe = redis.pipeline()
    pipe.zremrangebyscore(key, 0, window_start)  # remove requests antigos
    pipe.zadd(key, {str(uuid.uuid4()): now})      # adiciona request atual
    pipe.zcard(key)                                # conta requests na janela
    pipe.expire(key, window)                       # limpa a key após window
    results = pipe.execute()
    
    count = results[2]
    return count <= limit, count
```

---

## 5. Lock Distribuído com Redis

**Padrão SET NX EX (Atomic Lock):**
```python
def acquire_lock(resource, owner, ttl_seconds):
    # SET NX = "set if not exists" — operação atômica
    acquired = redis.set(
        f"lock:{resource}",
        owner,
        nx=True,   # só seta se não existir
        ex=ttl_seconds  # TTL automático = auto-release
    )
    return acquired is not None

def release_lock(resource, owner):
    # Verificar que somos o dono antes de liberar
    if redis.get(f"lock:{resource}") == owner:
        redis.delete(f"lock:{resource}")
```

**Casos de uso:**
- Reserva de ingressos (Ticket Master)
- Assignment de motoristas (Uber)
- Prevenir processamento duplicado

---

## 6. Leaderboard com Sorted Set

```python
# Adicionar/atualizar score
redis.zadd("leaderboard:game_123", {"player_alice": 1500})
redis.zincrby("leaderboard:game_123", 50, "player_alice")  # adiciona 50 pontos

# Top 10 (maior score primeiro)
top10 = redis.zrevrange("leaderboard:game_123", 0, 9, withscores=True)

# Posição de um jogador específico
rank = redis.zrevrank("leaderboard:game_123", "player_alice")  # 0-indexed

# Score de um jogador
score = redis.zscore("leaderboard:game_123", "player_alice")
```

**Complexidade:**
- `ZADD`: O(log N)
- `ZRANGE`: O(log N + K) onde K = elementos retornados
- `ZRANK`: O(log N)

**Para Nubank (top K tweets por hashtag):**
```python
# Manter top 5 tweets mais curtidos por hashtag
redis.zadd("top:tiger", {"tweet_id_123": likes_count})

# Remover os que saíram do top 5
redis.zremrangebyrank("top:tiger", 0, -6)  # mantém apenas os 5 maiores
```

---

## 7. Geospatial Index

```python
# Adicionar localização
redis.geoadd("drivers", longitude, latitude, driver_id)

# Buscar no raio de 5km
nearby = redis.georadius("drivers", user_lng, user_lat, 5, "km")

# Com distâncias
nearby_with_dist = redis.georadius(
    "drivers", user_lng, user_lat, 5, "km",
    withcoord=True, withdist=True, sort="ASC"
)
```

**Como funciona internamente:** geohashing — latitude e longitude são convertidos em um score numérico para o Sorted Set.

**Limitação:** toda a chave fica em um único node. Para bilhões de pontos, é necessário fazer sharding manual por região.

---

## 8. Pub/Sub para Comunicação entre Serviços

**Problema:** servidor A precisa notificar servidor B sobre nova mensagem, mas B pode ser qualquer máquina do cluster.

```python
# Servidor A — publicar
redis.publish("channel:user_456", json.dumps({
    "type": "new_message",
    "from": "user_123",
    "content": "Olá!"
}))

# Servidor B — assinar
pubsub = redis.pubsub()
pubsub.subscribe("channel:user_456")

for message in pubsub.listen():
    if message["type"] == "message":
        send_to_websocket_client(message["data"])
```

**Características:**
- **At-most-once delivery** — mensagem pode se perder se o subscriber estiver offline
- Muito rápido (in-memory)
- Não persistente

**Quando usar:** notificações em tempo real onde eventual loss é aceitável (status online/offline, cursors em tempo real, etc.)

---

## 9. Redis Streams (Kafka-Lite)

Para quando você precisa de persistência e consumer groups:

```python
# Producer
redis.xadd("events", {"user_id": "123", "action": "purchase"})

# Consumer com consumer group
redis.xgroup_create("events", "payment_group", "$")

# Consumer lendo eventos
messages = redis.xreadgroup("payment_group", "worker_1", {"events": ">"})
for msg_id, fields in messages:
    process_payment(fields)
    redis.xack("events", "payment_group", msg_id)  # confirmar processamento
```

**Diferença de Kafka:**
- Redis Streams: simples, baixa latência, retenção limitada pela memória
- Kafka: mais complexo, alta retenção, muito maior escala

---

## 10. Quando NÃO usar Redis

```
❌ Como banco de dados primário (risco de perda de dados)
❌ Para dados que devem ser 100% duráveis sem configuração especial
❌ Para queries complexas (sem JOINs, sem full-text search)
❌ Para dados maiores que a memória disponível (fica caro)
```

---

## Referências

- [Redis Documentation](https://redis.io/docs/)
- [Redis Sorted Sets](https://redis.io/docs/data-types/sorted-sets/)
- [Stefan Tilkov — Hello Interview Redis Deep Dive](https://hellointerview.com)
