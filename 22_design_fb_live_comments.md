# 22 — Design Facebook Live Comments

> Sistema de comentários em tempo real para live streaming com escala de milhões de streams simultâneos

---

## O Problema

Live comments são o feed de comentários exibido abaixo de uma transmissão ao vivo. Diferente de um chat 1:1, aqui:
- **1 produtor de vídeo** → **milhares/milhões de espectadores**
- Comentários de **muitos usuários** → entregues a **todos os espectadores**
- Escala assimétrica: muito mais leituras do que escritas por usuário

---

## Requisitos Funcionais

| Funcionalidade | Detalhe |
|---|---|
| Postar comentário | Associado a um live video |
| Ver comentários em tempo real | < 200ms de latência |
| Ver comentários anteriores | Scroll para cima (paginação) |

**Below the line:** respostas a comentários, reações, moderação de conteúdo

---

## Requisitos Não-Funcionais

| Requisito | Decisão |
|---|---|
| **Escala** | 1 milhão de lives simultâneas, até 1.000 comentários/seg/vídeo |
| **CAP Theorem** | Disponibilidade > Consistência (lag de segundos é aceitável) |
| **Latência** | < 200ms (limiar de percepção humana de "tempo real") |

> **Fun fact:** o vídeo mais assistido do Facebook Live foi "Chewbacca Mom" com 180 milhões de views — dezenas de milhares de comentários/segundo.

---

## Entidades e API

### Entidades
```
Comment: comment_id, video_id, author_id, content, created_at
LiveVideo: video_id (gerenciado por outro serviço)
User/Viewer: user_id
```

### API REST

```http
# Postar comentário
POST /live_videos/{video_id}/comments
Body: { content: "..." }
Returns: comment_id, created_at

# Buscar comentários (cursor-based pagination)
GET /live_videos/{video_id}/comments?last_comment_id={id}&direction={after|before}&page_size=50
```

**Cursor-based pagination:**
- `direction=after`: novos comentários desde o cursor
- `direction=before`: comentários históricos antes do cursor
- Eficiente para dados append-only (cresce apenas para frente)

---

## High-Level Design

### Modelo de Dados

**Tabela: comments**
```
comment_id  | video_id (indexed) | author_id | content | created_at (sort key)
```

**Query pattern principal:** filtrar por `video_id`, ordenar por `created_at`

### Fluxo Básico (polling)

```
1. Usuário carrega live → GET /comments?direction=before → últimos 50 comentários
2. A cada 2-5s: GET /comments?direction=after&last_comment_id=X → novos comentários
3. Para postar: POST /comments → grava no banco
```

**Problema:** latência de 2-5s vs. meta de 200ms.

---

## Deep Dive: Latência — SSE vs WebSockets

### WebSockets

**Prós:**
- Conexão bidirecional persistente
- Push imediato do servidor para o cliente

**Contras:**
- Protocolo próprio (não HTTP)
- Precisa de infraestrutura compatível (proxies, firewalls)
- Overhead de setup maior

**Ideal quando:** reads ≈ writes (ex: chat 1:1, jogos multiplayer)

### Server-Sent Events (SSE)

**Prós:**
- Sobre HTTP/HTTPS nativo (sem upgrade de protocolo)
- Reconexão automática com suporte nativo nos browsers
- Mais leve, sem overhead de negociação
- Sem problemas com firewalls corporativos

**Contras:**
- Unidirecional (servidor → cliente apenas)

**Ideal quando:** reads >> writes (ex: live comments, stock tickers, notificações)

### Decisão para Live Comments

```
Live Comments:
- 1 comentário enviado (write HTTP POST)
- 1.000 comentários recebidos por segundo (reads via push)

Ratio: 1:1.000 → SSE é a escolha correta ✓
```

**Fluxo com SSE:**

```
1. Cliente abre SSE connection → GET /live_videos/{id}/sse
2. Server mantém connection aberta
3. Novo comentário chega ao Comment Service
4. Comment Service faz push via SSE para todos os clientes conectados ao vídeo
```

**In-memory mapping no Comment Service:**
```json
{
  "video_id_1": [sse_conn_1, sse_conn_4, sse_conn_7],
  "video_id_2": [sse_conn_2, sse_conn_5],
}
```

---

## Deep Dive: Escala

### Estimativa de Conexões SSE

```
1.000.000 lives × 1.000 viewers/live = 1.000.000.000 conexões SSE simultâneas
1 servidor aguenta ~1.000.000 conexões SSE
→ Precisamos de ~1.000 servidores real-time
```

### Separação de Responsabilidades

```
Comment Service (menos instâncias):
  - POST /comments (escrita)
  - GET /comments (leitura histórica)

Real-Time Comment Service (mais instâncias — ~1.000):
  - Mantém conexões SSE dos clientes
  - Recebe notificações de novos comentários
  - Faz push para clientes conectados
```

### O Problema de Roteamento

Novo comentário chega ao Comment Service → como saber **quais** Real-Time Servers têm clientes assistindo aquele vídeo?

---

### Solução 1: Dispatcher Central (Zookeeper)

```
Zookeeper mantém: { video_id → [server_1, server_3, server_7] }

Fluxo:
1. Cliente conecta → Real-Time Server registra no Zookeeper
2. Novo comentário → Comment Service consulta Zookeeper → lista de servidores
3. Comment Service envia comentário direto para esses servidores
4. Servidores fazem push via SSE para seus clientes
```

**Cache no Comment Service:** evita consulta ao Zookeeper a cada comentário.

**Prós:** direcionamento preciso  
**Contras:** estado distribuído entre Zookeeper e servidores, orquestração na adição/remoção de servidores

### Solução 2: Pub/Sub (preferida)

```
Comment Service → publica no Pub/Sub → Real-Time Servers escutam e filtram

Fluxo:
1. Novo comentário → Comment Service publica no canal do Pub/Sub
2. Todos os Real-Time Servers recebem
3. Cada servidor verifica seu in-memory mapping
4. Se tem clientes para aquele video_id → push via SSE
5. Se não tem → ignora
```

**Tecnologia:** Redis Pub/Sub (leve, in-memory, sem overhead de Kafka para tópicos efêmeros)

**Prós:** simples, sem estado externo, failover natural  
**Contras:** todos os servidores processam todos os eventos (fireHose)

### Otimização: Particionamento do Pub/Sub

```
Problema: 1.000 servidores processando 1.000 comentários/seg × 1M vídeos = muito processamento desperdiçado

Solução: particionar canais por hash(video_id) % N_channels

Cada Real-Time Server subscreve apenas os canais que têm clientes ativos:
- Server A subscreve canal #7 (tem clientes do video 123 cujo hash é 7)
- Server B subscreve canais #3 e #12
- Não precisa escutar tudo
```

### Colocação (Co-location)

```
Ideal: todos os viewers de um mesmo vídeo no mesmo Real-Time Server
→ Apenas 1 subscrição do Pub/Sub para aquele vídeo
→ Fan-out local via in-memory mapping (O(N) simples)

Para vídeo viral (160M viewers): distribuir entre N servidores dedicados
→ Ainda melhor que todos os 1.000 servidores escutando aquele canal
```

---

## Banco de Dados: Cassandra

### Por que Cassandra?

| Característica | Benefício |
|---|---|
| Partition key = video_id | Todos os comentários de um vídeo colocados juntos |
| Sort key = created_at / comment_id | Ordenação nativa eficiente |
| Write-optimized | Alto throughput de escritas |
| Horizontal scaling | Sharding por video_id nativo |
| Eventual consistency | Aceitável para live comments |

### Estimativa de Storage

```
1M vídeos × 100 comentários × 30 min = estimativa por dia
Assumindo cada comentário ~ 500 bytes
→ ordem de TBs/dia (escala gerenciável com Cassandra)
```

---

## Arquitetura Final

```
[Client Browser]
    │
    ├── POST /comments (HTTP) → [API Gateway] → [Comment Service] → [Cassandra]
    │                                                   ↓
    │                                           [Redis Pub/Sub]
    │                                                   ↓
    └── SSE connection → [API Gateway] → [Real-Time Comment Service Pool]
                                              (1.000 instâncias)
                                              ↕ in-memory: { video_id → SSE conns }
```

---

## Resumo de Decisões

| Problema | Solução | Alternativa Descartada |
|---|---|---|
| Push em tempo real | SSE | WebSocket (overkill para unidirecional) |
| Roteamento entre servidores | Redis Pub/Sub | Zookeeper dispatcher (mais complexo) |
| Escala de writes | Cassandra | PostgreSQL (funciona, mas menos otimizado) |
| Escala de conexões SSE | Separar Real-Time Service | Mesmo serviço de CRUD (gargalo) |
| Pub/Sub eficiente | Particionamento por hash | Broadcast total (fireHose) |

---

*Fonte: Hello Interview — Design Facebook Live Comments*
