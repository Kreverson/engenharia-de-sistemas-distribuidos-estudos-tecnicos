# 36 — Design Instagram Auction: SSE, Redis Pub/Sub e Job Scheduling

> Baseado em mock interview da Hello Interview (candidato avaliado como "strong hire" para Staff). Cobre design de leilões em tempo real: WebSockets vs SSE, Redis Pub/Sub, consistência de lances, e scheduled jobs com SQS/Temporal.

---

## 📌 Contexto do Problema

Novo tipo de post no Instagram: **Auction Post**.
- Vendedor cria um leilão com item e data/hora de encerramento
- Usuários fazem lances (bids)
- Ao encerrar, o maior lance vence
- Vencedor e vendedor são notificados

---

## 🎯 Requisitos

### Funcionais
```
1. Criar leilão (auction post)
2. Submeter lances (bids)
3. Determinar e notificar o vencedor
4. Visualizar lances em tempo real (live ticker)
```

### Não-Funcionais
```
- Consistência forte para lances (sem dois vencedores, sem empate ambíguo)
- Latência de atualização de bids < 500ms (real-time UX)
- Alta disponibilidade
- Escala: dezenas de milhões de leilões/dia
- Durabilidade dos bids (leilão de Michael Jordan = $1M em jogo)
```

---

## 🏗️ Entidades e Schema

```sql
-- Leilão
CREATE TABLE auctions (
  id UUID PRIMARY KEY,
  seller_id UUID,
  title VARCHAR,
  current_highest_bid DECIMAL,
  status VARCHAR CHECK (status IN ('active', 'completed')),
  winner_id UUID,
  end_datetime TIMESTAMP,
  created_at TIMESTAMP
);

-- Item do leilão
CREATE TABLE auction_items (
  id UUID PRIMARY KEY,
  auction_id UUID,
  name VARCHAR,
  image_s3_url VARCHAR
);

-- Lance
CREATE TABLE bids (
  id UUID PRIMARY KEY,
  auction_id UUID,
  bidder_id UUID,
  amount DECIMAL,
  created_at TIMESTAMP
);
```

---

## 🔌 API Design

```http
POST /auctions
  headers: Authorization: Bearer {token}
  body: {
    title: "Camiseta vintage",
    end_datetime: "2024-01-20T18:00:00Z",
    item: { name: "...", images: [...] }
  }
  → 201 Created, { auction_id }

POST /bids
  headers: Authorization: Bearer {token}
  body: {
    auction_id: "abc123",
    amount: 150.00
  }
  → 201 Created, { bid_id }
  → 409 Conflict, { error: "bid below current highest" }
  → 410 Gone, { error: "auction expired" }

GET /auctions/{auction_id}/live
  headers: Authorization: Bearer {token}
  → SSE Stream: text/event-stream
  (estabelece conexão SSE para atualizações em tempo real)
```

---

## 🏛️ High-Level Design

```
                    ┌────────────────────┐
                    │    API Gateway     │
                    │ (Auth, Rate Limit) │
                    └────────┬───────────┘
           ┌─────────────────┼─────────────────┐
           ▼                 ▼                 ▼
  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐
  │Auction Service│  │  Bid Service │  │Streaming Service  │
  └──────┬───────┘  └──────┬───────┘  │(SSE connections)  │
         │                 │          └────────┬──────────┘
         └─────────────────┤                   │
                           ▼                   │
                  ┌─────────────────┐          │
                  │   PostgreSQL    │          │
                  │  auctions       │          │
                  │  auction_items  │          │
                  │  bids           │          │
                  └─────────────────┘          │
                                               │
                  ┌────────────────────────────┘
                  │
                  ▼
         ┌─────────────────┐
         │  Redis Pub/Sub  │ ← recebe eventos de bid
         └─────────────────┘
```

---

## 🚀 Deep Dives

### Deep Dive 1: SSE vs WebSockets para Live Bids

| Aspecto | WebSockets | Server-Sent Events (SSE) |
|---|---|---|
| **Comunicação** | Bidirecional | Unidirecional (server → client) |
| **Protocolo** | ws:// (custom) | HTTP padrão |
| **Reconexão** | Manual | Automática (built-in) |
| **Casos de uso** | Chat, games | Feeds, notificações, live updates |
| **Overhead** | Maior | Menor |

**Decisão:** SSE é suficiente — o cliente não precisa enviar dados pelo canal de streaming. Bids são enviados via POST normal.

```
Usuário abre página do leilão:
  GET /auctions/abc123/live → estabelece SSE connection

Servidor envia eventos:
  data: {"bid_amount": 150.00, "bidder": "alice", "timestamp": "..."}
  data: {"bid_amount": 200.00, "bidder": "bob", "timestamp": "..."}

Reconexão automática em caso de queda da rede.
```

---

### Deep Dive 2: Consistência de Lances — Transação ACID

**Problema:** Dois usuários submetem lances simultaneamente. Qual vence? E o `current_highest_bid` precisa estar sempre correto.

```
Sem transação:
  Thread A lê: current_highest_bid = $100
  Thread B lê: current_highest_bid = $100
  Thread A escreve: bid=$150, UPDATE auction SET highest=$150
  Thread B escreve: bid=$120, UPDATE auction SET highest=$120  ← errado!
  → current_highest_bid = $120, mas bid de $150 existe no banco!
```

#### Solução: Transação Serializable

```python
def submit_bid(auction_id, bidder_id, amount):
    with transaction(isolation_level=SERIALIZABLE):
        # 1. Verifica se leilão ainda está ativo
        auction = SELECT * FROM auctions WHERE id = auction_id FOR UPDATE
        if auction.end_datetime < now():
            raise AuctionExpiredException()

        # 2. Verifica se o lance é maior que o atual
        if amount <= auction.current_highest_bid:
            raise BidTooLowException()

        # 3. Cria o bid
        INSERT INTO bids (auction_id, bidder_id, amount, created_at)

        # 4. Atualiza o leilão atomicamente
        UPDATE auctions SET current_highest_bid = amount WHERE id = auction_id

        # 5. Publica evento no Redis Pub/Sub
        redis.publish(f"auction:{auction_id}", {
            "bid_amount": amount,
            "bidder_id": bidder_id
        })
```

**O `FOR UPDATE` garante lock na linha do leilão** → nenhuma outra transação lê/escreve até o commit.

---

### Deep Dive 3: Redis Pub/Sub para Live Updates

```
Fluxo completo de um lance:

1. Cliente → POST /bids (amount=$200)
2. Bid Service → PostgreSQL (transação)
3. Bid Service → Redis PUBLISH "auction:abc123" {amount: 200, ...}
4. Redis → distribui para todos os subscribers do canal "auction:abc123"
5. Streaming Service → recebe evento
6. Streaming Service → SSE → todos os clientes conectados ao leilão
```

```
Scaling do Streaming Service:
  Por que separado do Bid Service?
  → Natureza diferente: Bid Service é stateless, Streaming Service mantém
    conexões SSE abertas (stateful)
  → Podem escalar independentemente
  → Bid Service: escala vertical para throughput de transações
  → Streaming Service: escala horizontal para número de conexões abertas
```

#### Sobre canais por auction_id no Redis

```
Dezenas de milhões de leilões → dezenas de milhões de canais Redis

Redis Pub/Sub usa channels (não topics com partições como Kafka)
  → Channels são apenas strings → sem limite físico relevante

Porém: leilões ativos simultaneamente em qualquer momento
  são muito menores que o total diário → OK!
```

---

### Deep Dive 4: Determinar o Vencedor — Do Cron Job ao SQS

#### Abordagem 1: Cron Job (ruim)

```python
# Roda a cada minuto
def check_winners():
    expired_auctions = SELECT * FROM auctions
        WHERE status = 'active' AND end_datetime < now()

    for auction in expired_auctions:
        winner_bid = SELECT * FROM bids
            WHERE auction_id = auction.id
            ORDER BY amount DESC LIMIT 1

        UPDATE auctions SET status = 'completed', winner_id = winner_bid.bidder_id
        notify(auction.seller_id, winner_bid.bidder_id)
```

**Problema:** Delay de até 1 minuto entre encerramento e notificação.

#### Abordagem 2: SQS com Visibility Timeout (melhor)

```
Quando leilão é criado:
  SQS.send_message(
    body={"auction_id": "abc123"},
    delay_seconds=time_until_end  ← mensagem só fica visível no futuro!
  )

Consumer (Winner Worker):
  Quando mensagem fica visível → processa → determina vencedor → notifica
```

```
Resultado: encerramento com precisão de segundos, sem polling constante.

SQS Visibility Timeout:
  - Mensagem fica invisível por N segundos após ser recebida
  - Se worker falhar, mensagem reaparece → outro worker processa
  - Durabilidade garantida
```

#### Problema do SQS: E se o lance mais alto mudar?

```
Leilão com regra "going-once, going-twice": vence quem está na frente
por 1 hora (como eBay).

SQS não permite atualizar mensagem já enqueued.

Solução: Criar nova mensagem a cada novo lance mais alto

Bid Service:
  novo_bid = $200
  SQS.send_message(
    body={"auction_id": "abc123", "expected_winner": "alice", "amount": 200},
    delay_seconds=3600  ← 1 hora
  )

Winner Worker recebe:
  Se auction.current_highest_bid == expected_amount → alice venceu ✅
  Se auction.current_highest_bid != expected_amount → mensagem stale, ignorar ✅
```

---

### Deep Dive 5: Temporal IO — Orquestração Durável

**Temporal** é uma plataforma de "durable execution" — orquestração de workflows com checkpoint automático.

```
Problema que resolve:
  Worker processando pagamento → crash → perde estado → usuário cobrado mas sem produto?

Temporal:
  Worker → chama atividade 1 → checkpoint
           → chama atividade 2 → checkpoint
           → crash!
  Temporal reinicia → retoma do último checkpoint → zero perda de estado
```

#### Uso no contexto de leilões

```python
# Workflow temporal para leilão
@workflow.defn
class AuctionWorkflow:
    @workflow.run
    async def run(self, auction_id: str, end_time: datetime):
        # Agenda execução para o momento do encerramento
        await workflow.sleep(end_time - datetime.now())

        # Executa com durabilidade garantida
        winner = await workflow.execute_activity(
            determine_winner,
            auction_id,
            start_to_close_timeout=timedelta(minutes=5)
        )

        await workflow.execute_activity(
            notify_participants,
            winner,
            start_to_close_timeout=timedelta(minutes=5)
        )

# Sinais (como atualizar um workflow em execução)
@workflow.signal
def update_leading_bid(self, new_bidder: str, amount: float):
    self._current_leader = new_bidder
    self._current_amount = amount
```

#### Temporal vs SQS

| Aspecto | SQS | Temporal |
|---|---|---|
| **Durabilidade** | Alta (mensagem persiste) | Muito alta (checkpoint de estado) |
| **Scheduling** | Visibility timeout | `workflow.sleep()` nativo |
| **Update mid-flight** | Não (cria nova mensagem) | Sim (signals) |
| **Observabilidade** | Básica | Rico (UI, traces) |
| **Complexidade** | Baixa | Alta |
| **Caso de uso** | Jobs simples | Workflows longos e complexos |

---

### Deep Dive 6: Durabilidade com Kafka

**Cenário:** Bid Service cai durante pico de lances. PostgreSQL e Redis são recuperáveis, mas e os eventos intermediários?

```
Solução: Kafka como stream de events

Bid Service:
  1. Persiste bid no PostgreSQL (transação)
  2. Publica no Kafka topic "bids" (durável, com retenção)
  3. Kafka entrega para Redis Pub/Sub (streaming)
             ↓
  Se Bid Service cai:
  → Kafka guarda eventos → ao reiniciar, processa do offset correto
  → Nenhum evento perdido

Para debugging/audit:
  → Kafka topic "bids" tem histórico completo
  → Leilão de $1M disputado? Replay do Kafka para reconstruir timeline
```

---

## 📊 Arquitetura Final

```
                    ┌─────────────────────────────────┐
                    │          API Gateway            │
                    └──────────────┬──────────────────┘
           ┌──────────────┬────────┴──────────────────┐
           ▼              ▼                           ▼
  ┌──────────────┐ ┌──────────────┐         ┌──────────────────┐
  │Auction Service│ │  Bid Service │         │ Streaming Service │
  └──────┬───────┘ └──────┬───────┘         └────────┬─────────┘
         │                │                          │
         ▼                ▼                          │
  ┌──────────────────────────────┐                   │
  │         PostgreSQL           │                   │
  │  auctions, bids, items       │                   │
  └──────────────────────────────┘                   │
                 │                                   │
                 │ publica evento                    │
                 ▼                                   │
  ┌─────────────────────────────┐                   │
  │       Kafka (bids)          │──────────────────▶│
  └─────────────────────────────┘    stream         │
                 │                                   │
         ┌───────▼────────┐                         │
         │  Redis Pub/Sub  │◄───────────────────────┘
         └───────┬─────────┘    subscribe por auction_id
                 │
         SSE para clientes
```

---

## 💡 Lições e Padrões

| Padrão | Aplicação |
|---|---|
| **SSE para one-way streaming** | Live prices, notificações, feeds de eventos |
| **Transação SERIALIZABLE com FOR UPDATE** | Qualquer recurso disputado (ingressos, lances, estoque) |
| **Redis Pub/Sub para fan-out de eventos** | Chat em grupos, live updates |
| **SQS delay messages** | Tarefas agendadas para o futuro |
| **Temporal IO** | Workflows longos com garantia de execução |
| **Kafka para durabilidade** | Eventos de alto valor que não podem ser perdidos |

---

## 🎓 Avaliação do Mock Interview

O candidato recebeu **"strong hire"** com:

**✅ Acertou:**
- Entidades e API design
- Consistência forte com transação SERIALIZABLE
- SSE vs WebSockets (escolha justificada)
- Redis Pub/Sub para live updates
- Error handling de bids expirados e abaixo do mínimo

**⚠️ Oportunidade de melhoria:**
- Durabilidade (Kafka) — identificado pelo entrevistador como importante para leilões de alto valor
- Solução do cron job para scheduled jobs → SQS com visibility timeout

---

*Arquivo gerado a partir do mock interview da Hello Interview — Instagram Auction*
