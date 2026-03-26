# 40 — Design de Rate Limiter Distribuído

> Um dos problemas clássicos de System Design: como proteger seus serviços de abuso sem degradar a experiência de usuários legítimos.

---

## 1. O que é um Rate Limiter e por que existe

Um **rate limiter** controla quantas requisições um cliente pode fazer em uma janela de tempo. Objetivos principais:

- **Proteger o backend** de ataques DDoS e abuso
- **Garantir fairness** entre usuários
- **Prevenir degradação em cascata** de serviços

Sem rate limiting, um único usuário mal-intencionado (ou com bug) pode derrubar toda a infraestrutura.

---

## 2. Requisitos funcionais e não-funcionais

### Funcionais
1. **Identificar clientes** por User ID, IP ou API Key
2. **Limitar requests** com base em regras configuráveis
3. **Retornar headers/status adequados** para clientes saberem o que aconteceu

### Não-funcionais críticos

| Requisito | Métrica | Por quê |
|---|---|---|
| **Baixa latência** | < 10ms overhead | Rate check está no caminho crítico de TODA requisição |
| **Alta disponibilidade** | > 99.99% | Prefere-se eventuais falhas no limite do que derrubar o sistema |
| **Escalabilidade** | 1M req/s | Sistemas reais exigem distribuição |

**CAP Theorem aplicado:** Optamos por **disponibilidade > consistência**. É preferível que uma regra nova demore alguns segundos para propagar a que o rate limiter fique offline durante a propagação.

---

## 3. Onde posicionar o Rate Limiter

### Opção 1: Dentro de cada microserviço
```
Client → [MS1 com RL] 
       → [MS2 com RL]
       → [MS3 com RL]
```
✅ Latência zero (in-process)  
❌ **Sem visão global:** dois requests para MSs diferentes não são correlacionados  
❌ Lógica duplicada em todos os serviços

### Opção 2: Serviço dedicado
```
Client → Gateway → [Rate Limiter Service] → MS1, MS2, MS3
```
✅ Visão global do estado  
❌ **Network hop extra** em toda requisição  
❌ Single point of failure se não replicado

### Opção 3: No API Gateway / Edge (recomendado ✅)
```
Client → [API Gateway com Rate Limiter integrado] → MS1, MS2, MS3
                     ↕
              [Redis compartilhado]
```
✅ Única barreira no edge ("bouncer na porta do clube")  
✅ Todos os gateways compartilham estado via Redis  
✅ Microserviços ficam desacoplados da lógica de rate limiting  
⚠️ Contexto limitado ao que está no HTTP header/JWT

---

## 4. Como identificar clientes

Estratégia em camadas (mais sofisticada):

```
└── API Key: 50.000 req/s  (developers com contrato)
    └── User ID autenticado: 1.000 req/s  (usuários premium)
        └── IP Address: 100 req/s  (anônimos)
```

Dados extraídos do JWT (header da requisição):
- `user_id`
- `tier` (free, premium, enterprise)
- `api_key`

---

## 5. Algoritmos de Rate Limiting

### 5.1 Fixed Window Counter (janela fixa)

```
[12:00:00 - 12:00:59] → contador = 0
Alice faz 100 requests → contador = 100 → bloqueia próximas
[12:01:00] → contador reseta para 0
```

**Implementação:**
```
key = "alice:12:01"  # usuário + janela atual
value = incrementar(key) com TTL de 60s
if value > limite: rejeitar
```

**Problema — Boundary Effect:**
```
12:00:59 → Alice faz 100 requests (válido)
12:01:00 → Alice faz mais 100 requests (válido para nova janela)
Resultado: 200 requests em 2 segundos ← burst perigoso
```

### 5.2 Sliding Window Log (log deslizante)

Mantém timestamp de cada request em uma estrutura ordenada (heap/deque):
```
Janela móvel de 60s:
[12:00:30, 12:00:35, 12:00:40, ..., 12:01:25] ← remove tudo < agora-60s
Count = tamanho do log ≤ limite? → aceitar
```

✅ Precisão exata  
❌ **Memória O(requests):** para 1M usuários × 100 req/min = 100M timestamps em memória

### 5.3 Sliding Window Counter (estimativa deslizante)

Combina dois contadores fixos com estimativa ponderada:

```
Janela anterior (12:00 - 12:01): 80 requests
Janela atual (12:01 - agora, 42s = 70%): 35 requests

Estimativa = requests_atual + (1 - percentual_atual) × requests_anterior
           = 35 + (1 - 0.70) × 80
           = 35 + 24 = 59 requests estimados
```

Limite = 100? → 59 < 100 → aceitar  
✅ Apenas 2 inteiros por usuário  
⚠️ Aproximação (assume distribuição uniforme na janela anterior)

### 5.4 Token Bucket (recomendado para produção ✅)

```
Cada cliente tem um "balde" com N tokens.
Tokens são adicionados a uma taxa constante (refill rate).
Cada request consume 1 token.
Sem tokens → request rejeitado.
```

**Exemplo:**
```
Bucket capacity = 100 tokens  ← capacidade de burst
Refill rate = 10 tokens/minuto ← taxa sustentada

Alice tem 20 tokens, última recarga há 30s:
→ Tokens adicionados = 30s × (10/60) = 5 tokens
→ Tokens atuais = 20 + 5 = 25
→ 25 > 0: aceitar e decrementar para 24
```

**Por que Token Bucket é superior:**
- **Burst controlado:** permite picos temporários sem rejeitar legitimamente
- **Simplicidade:** apenas 2 valores por usuário (`tokens`, `last_refill_timestamp`)
- **Analogy:** é exatamente como APIs de grandes empresas funcionam (AWS, Stripe, GitHub)

---

## 6. Implementação com Redis

### Estrutura de dados no Redis

```
HSET alice:bucket tokens 25 last_refill 1700000000
```

### Fluxo de uma requisição

```
1. Gateway recebe request de Alice
2. HMGET alice:bucket tokens last_refill    ← leitura atômica
3. Calcular tokens_atuais com base no tempo decorrido
4. tokens_atuais > 0? → aceitar
5. HSET alice:bucket tokens (tokens_atuais - 1) last_refill (agora)
6. Retornar resposta ao cliente
```

### O problema de race condition

```
Thread A lê: tokens = 1
Thread B lê: tokens = 1
Thread A decide: aceitar → escreve tokens = 0
Thread B decide: aceitar → escreve tokens = 0  ← BUG! dois accepts com 1 token
```

**Solução: Lua Script no Redis**

Redis é single-threaded. Um Lua script executa atomicamente:

```lua
local key = KEYS[1]
local refill_rate = tonumber(ARGV[1])
local capacity = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1]) or capacity
local last_refill = tonumber(bucket[2]) or now

-- Calcular tokens ganhos desde último check
local elapsed = now - last_refill
local new_tokens = math.min(capacity, tokens + elapsed * refill_rate)

if new_tokens >= 1 then
    redis.call('HMSET', key, 'tokens', new_tokens - 1, 'last_refill', now)
    return 1  -- aceito
else
    return 0  -- rejeitado
end
```

O script roda como **operação atômica** — nenhum outro comando Redis pode interromper no meio.

---

## 7. Escalabilidade — Redis Cluster

### Problema de capacidade
```
Redis single node: ~100K ops/s
Token bucket = 2 ops por request (HMGET + HMSET)
Máx. requests suportados: ~50K req/s

Necessário: 1M req/s
Shards necessários: 1M / 50K = 20 shards mínimo
```

### Redis Cluster com Consistent Hashing

```
alice → hash("alice") → Shard 3
bob   → hash("bob")   → Shard 1
carol → hash("carol") → Shard 7
```

**Vantagem:** adicionar shards redistribui minimamente os dados (consistent hashing).

### Alta disponibilidade — Read Replicas

```
Shard 1 (Primary) ←→ Shard 1 (Replica)
Shard 2 (Primary) ←→ Shard 2 (Replica)
```

**Trade-off:** replicação assíncrona → se o primary morrer antes de replicar, Alice pode fazer +1 request indevido. Aceitável (optamos por disponibilidade > consistência).

---

## 8. Respostas e headers HTTP

### Status code obrigatório

```
HTTP 429 Too Many Requests
```

### Headers de resposta (best practices)

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100          # limite total
X-RateLimit-Remaining: 0        # requests restantes
X-RateLimit-Reset: 1700000060   # quando reseta (unix timestamp)
Retry-After: 60                  # segundos até poder tentar novamente
```

### Fail-open vs Fail-closed

Se o Redis cair, qual deve ser o comportamento?

| Estratégia | Comportamento | Quando usar |
|---|---|---|
| **Fail-open** | Deixa todos passarem | Apps onde negação de serviço é mais crítico que abuso |
| **Fail-closed** | Bloqueia tudo | Apps onde abuso derruba backends frágeis |
| **Fail-graceful** | Usa contador local no gateway | Melhor das duas → degradação elegante |

**Fail-graceful (recomendado):**
```
Se Redis indisponível:
→ Usar Fixed Window Counter in-memory no gateway
→ Pior caso: coordenação imperfeita entre gateways
→ Melhor que: ou nenhum rate limiting ou total bloqueio
```

---

## 9. Configuração dinâmica de regras

### Problema
Regras hardcoded → requer redeploy para mudar limites.

### Solução: Push-based com ZooKeeper/etcd

```
[Admin Console] → atualiza regra em ZooKeeper
                         ↓
              [ZooKeeper notifica via TCP persistente]
                         ↓
              [Gateways atualizam regras em memória]
                         ↓
              [Sem latência extra em runtime: regras em memória local]
```

**Por que não poll periódico?**
- Pull a cada 1s: desperdício de CPU no gateway
- Pull a cada 20s: lag de 20s na aplicação de novas regras
- Push: instantâneo + zero overhead em runtime

---

## 10. Diagrama de arquitetura completo

```
                    Usuários
                       │
            ┌──────────▼──────────┐
            │    API Gateway      │◄── ZooKeeper (regras)
            │  [Rate Limiter]     │
            └──────────┬──────────┘
                       │ Lua Script
            ┌──────────▼──────────┐
            │   Redis Cluster     │
            │  ┌─────┐ ┌─────┐   │
            │  │Sh.1 │ │Sh.2 │...│
            │  └──┬──┘ └──┬──┘   │
            │    replica  replica │
            └─────────────────────┘
                       │
            ┌──────────▼──────────┐
            │    Microserviços    │
            └─────────────────────┘
```

---

## 11. O que avaliar por nível em entrevistas

| Nível | Expectativa |
|---|---|
| **Junior/Mid** | Conhecer os algoritmos, propor Redis, entender o race condition |
| **Senior** | Identificar proativamente o problema de escala, calcular shards necessários, discutir fail modes |
| **Staff** | Liderar para Lua scripting, discutir replicação assíncrona e suas implicações, abordar configuração dinâmica, calcular pool size de conexões |

---

## Comparativo dos algoritmos

| Algoritmo | Memória | Precisão | Burst | Complexidade |
|---|---|---|---|---|
| Fixed Window | O(1) | Baixa (boundary) | Permite double burst | Muito baixa |
| Sliding Window Log | O(requests) | Alta | Não controla | Alta |
| Sliding Window Counter | O(1) | Média (estimativa) | Controla parcialmente | Baixa |
| **Token Bucket** | **O(1)** | **Alta** | **Controla elegantemente** | **Baixa** |

---

*Tópicos relacionados: `02_rate_limiting_e_seguranca.md`, `16_redis_deep_dive.md`, `06_thundering_herd.md`*
