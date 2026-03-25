# 11 — Design Ticket Master: Ingresso, Lock Distribuído e Surge de Popularidade

> **Questão:** Desenhe um serviço de venda de ingressos como o Ticket Master.  
> **Por que é popular:** combina consistência forte (sem double-booking), disponibilidade alta para busca, geospatial search, e o problema clássico de surge de usuários em eventos populares.

---

## 1. Requisitos

**Funcionais:**
- Buscar eventos (por termo, localização, data)
- Ver detalhes de um evento (incluindo mapa de assentos)
- Reservar um ingresso (processo em duas fases)

**Não Funcionais:**
- **Consistência forte para bookings:** um ingresso só pode pertencer a um usuário
- **Alta disponibilidade para busca/visualização:** não importa se um evento recém-adicionado demora 2s para aparecer
- **Read >> Write:** ~100:1 (muita gente vê, pouca compra)
- **Escalabilidade para surges:** Taylor Swift, Super Bowl — pico de 100.000 usuários simultâneos

---

## 2. O Problema Central: Double Booking

**Sem proteção:**
```
T=0: Usuário A vê assento B3 disponível
T=0: Usuário B vê assento B3 disponível
T=1: Usuário A reserva B3 → status = "reserved"
T=1: Usuário B reserva B3 → status = "reserved"  ← BUG: dois donos!
```

Esse é o problema mais crítico do sistema. Há três abordagens:

---

## 3. Abordagem 1: Timestamp + Status no Banco (Cron Job)

Adicionar `status` e `reserved_at` na tabela de tickets:

```sql
-- Tabela ticket
id          UUID
event_id    UUID  
seat        VARCHAR
price       DECIMAL
status      ENUM('available', 'reserved', 'booked')
reserved_at TIMESTAMP
user_id     UUID  -- quem reservou

-- Query para buscar disponíveis:
SELECT * FROM tickets 
WHERE event_id = $1 
  AND (status = 'available' 
    OR (status = 'reserved' AND reserved_at < NOW() - INTERVAL '10 minutes'))
```

Um **Cron Job** roda a cada N minutos e libera reservas expiradas:
```sql
UPDATE tickets 
SET status = 'available', user_id = NULL, reserved_at = NULL
WHERE status = 'reserved' AND reserved_at < NOW() - INTERVAL '10 minutes'
```

**Problema:** Delta `n` — se o cron roda de 10 em 10 minutos, um ticket pode ficar "reserved" por até 19 minutos em vez de 10.

---

## 4. Abordagem 2 (Ótima): Lock Distribuído com Redis TTL

```
Reserve → Redis.SET(ticketId, true, EX=600)   ← TTL de 10 minutos
Confirm → Redis.DEL(ticketId) + UPDATE tickets SET status='booked'
Expirar → automaticamente pelo Redis após 600 segundos
```

**Fluxo:**
```
Usuário clica no assento B3:
  1. POST /bookings/reserve { ticketId: "B3" }
  2. Booking Service: Redis.SET("lock:B3", userId, EX=600)
  3. Se SET retornou OK → sucesso, redireciona para pagamento
  4. Se SET retornou null → outro usuário já tem o lock → erro 409

10 minutos sem confirmação:
  → Redis deleta automaticamente a chave "lock:B3"
  → B3 volta a aparecer como disponível no mapa de assentos
```

**Por que Redis em vez do banco?**
- Operação atômica `SET NX EX` (set if not exists + expiry) — sem race condition
- TTL nativo — sem cron job
- Muito rápido (~1ms) — não adiciona latência ao checkout

**Verificar disponibilidade:**
```python
# Event service ao montar o mapa de assentos:
available_tickets = db.query("SELECT * FROM tickets WHERE event_id = $1")

# Para cada ticket, checar o lock:
for ticket in available_tickets:
    if redis.exists(f"lock:{ticket.id}"):
        ticket.status = "reserved"  # ocupado temporariamente

return available_tickets
```

**Se o Redis cair:**
- Provisionar novo nó imediatamente (detecção automática)
- Janela de inconsistência: usuários podem tentar comprar o mesmo ticket
- O banco PostgreSQL com ACID resolve o conflito: quem fizer o `UPDATE ... WHERE status='available'` primeiro ganha
- O segundo recebe erro e tem experiência ruim — trade-off aceitável para um evento raro

---

## 5. Busca de Eventos com Elasticsearch

A query SQL com `LIKE '%taylor%'` faz full table scan — inaceitável.

**Elasticsearch** resolve com índice invertido:

```
Tokenização: "Taylor Swift Eras Tour São Paulo"
→ tokens: ["taylor", "swift", "eras", "tour", "são", "paulo"]

Índice invertido:
  "taylor" → [event_1, event_5, event_12]
  "swift"  → [event_1, event_5]
  "paulo"  → [event_1, event_8, event_23]

Query "taylor paulo" → intersecção → event_1
```

**Sincronização Banco → Elasticsearch:**
```
Opção 1 (simples): 
  No código do serviço, após salvar no banco, também salva no ES

Opção 2 (Change Data Capture):
  Banco emite stream de mudanças → worker consome → atualiza ES
  Eventual consistency de segundos → OK para eventos (não mudam com frequência)
```

**Geo queries no Elasticsearch:**
```json
{
  "query": {
    "bool": {
      "must": { "match": { "name": "taylor" }},
      "filter": {
        "geo_distance": {
          "distance": "50km",
          "venue.location": { "lat": -23.5, "lon": -46.6 }
        }
      }
    }
  }
}
```

**Cache para buscas populares:**
- CDN pode cachear resultados de endpoints de busca por 30-60 segundos
- Para buscas únicas (lat/lon muito específico), menos útil
- Para buscas populares ("shows em SP"), enorme economia

---

## 6. Surge de Popularidade: Virtual Waiting Queue

**Problema:** 1.000.000 pessoas abrindo o Ticket Master no mesmo segundo para Taylor Swift.

**Solução: Fila Virtual (Virtual Waiting Queue)**

```
Evento popular → admin habilita fila virtual

1 milhão de usuários tentam acessar
    ↓
Cada um entra na fila (Redis Sorted Set, score = timestamp)
    ↓
Sistema exibe: "Você é o #47.832 na fila. Aguarde..."
    ↓
A cada N ingressos confirmados, libera próximos N usuários
    ↓
Usuário liberado → pode navegar no mapa de assentos
    ↓
SSE connection mantém o usuário atualizado sobre sua posição
```

**Implementação com Redis Sorted Set:**
```python
# Usuário entra na fila
redis.zadd("queue:event_123", {user_id: timestamp})

# Backend libera lotes
users_to_release = redis.zrange("queue:event_123", 0, 99)  # primeiros 100
for user in users_to_release:
    notify_user(user, "Você pode comprar agora!")
    redis.zrem("queue:event_123", user)
```

**Server-Sent Events para atualização em tempo real:**
```javascript
// Cliente
const es = new EventSource('/queue/position');
es.onmessage = (e) => {
    const { position, estimatedWait } = JSON.parse(e.data);
    updateUI(position, estimatedWait);
};

// Servidor empurra a cada 30 segundos
```

**Por que não WebSocket?** SSE é unidirecional (servidor → cliente), mais simples de implementar e suficiente para este caso de uso.

---

## 7. Real-Time Seat Map

**Problema:** usuário abre o mapa, vê B3 disponível, mas outro usuário reservou B3 5 segundos atrás.

**Solução: Long Polling ou SSE para atualizações**

```
Opção 1 (simples): Long Polling
  Cliente faz GET /events/:id/seats a cada 5 segundos
  Simples de implementar, latência de até 5s

Opção 2 (sofisticada): SSE
  Cliente abre conexão SSE para /events/:id/seat-updates
  Servidor empurra cada mudança de status em ~100ms
  Mais responsivo, mais complexo de escalar
```

**Trade-off:** para a maioria dos eventos, polling de 5s é suficiente. Para eventos ultra-populares (Super Bowl), SSE faz mais sentido.

---

## 8. Arquitetura Final

```
Client
  ↓
API Gateway (auth, rate limiting, routing)
  ├── Event CRUD Service → PostgreSQL + Elasticsearch (CDC)
  ├── Booking Service   → PostgreSQL (ACID) + Redis (locks TTL)
  └── Notification      → APNs / FCM

CDN → assets estáticos + cache de buscas populares

Para surge:
  Virtual Waiting Queue (Redis Sorted Set + SSE)
```

---

## 9. Resumo dos Trade-offs

| Decisão | Escolha | Motivo |
|---|---|---|
| Lock de ingresso | Redis TTL | Atômico, expira sozinho, rápido |
| Busca | Elasticsearch | Full-text + geo queries |
| Consistência booking | Forte (ACID) | Sem double booking |
| Consistência busca | Eventual | Não importa ver evento imediatamente |
| Surge | Virtual Queue | Protege o backend, melhora UX |

---

## Referências

- [Hello Interview — Ticket Master Breakdown](https://hellointerview.com)
- [Redis — SET NX EX (Atomic Lock)](https://redis.io/commands/set/)
- [Elasticsearch — Geo Distance Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-geo-distance-query.html)
