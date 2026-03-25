# 28 — Networking Essentials para System Design

> Fundamentos de rede, protocolos de aplicação, load balancing e padrões de resiliência

---

## O Modelo OSI (Camadas)

```
[Camada 7] Aplicação    → HTTP, gRPC, GraphQL, WebSocket, SSE
[Camada 6] Apresentação → TLS/SSL, codificação
[Camada 5] Sessão       → (abstrato em TCP/IP)
[Camada 4] Transporte   → TCP, UDP, QUIC
[Camada 3] Rede         → IP (IPv4, IPv6)
[Camada 2] Enlace       → Ethernet, Wi-Fi
[Camada 1] Física       → Cabos, fibra óptica
```

**Para System Design, as camadas relevantes são: 3 (IP), 4 (TCP/UDP) e 7 (HTTP e afins).**

---

## Camada 3: IP (Internet Protocol)

### IPv4 vs. IPv6

| | IPv4 | IPv6 |
|---|---|---|
| Tamanho | 4 bytes (ex: 192.168.1.1) | 16 bytes (ex: 2001:0db8::1) |
| Espaço | ~4 bilhões de endereços (esgotado) | Praticamente infinito |
| Uso típico | Externo (compatibilidade) | Interno (moderno) |

### Endereços Públicos vs. Privados

**Públicos:** conhecidos na internet global, roteáveis globalmente
```
Apple possui: 17.0.0.0/8 → todos os endereços começando com 17
```

**Privados:** uso interno, sem conflito com público
```
192.168.x.x → range reservado para redes locais
10.x.x.x → range de data centers
172.16-31.x.x → range privado intermediário
```

**Em System Design:**
- **Públicos:** API Gateway, Load Balancers externos, CDN nodes
- **Privados:** todos os microserviços internos, bancos, caches

---

## Camada 4: TCP vs. UDP

### TCP — Transmission Control Protocol

```
Garantias:
✓ Entrega garantida (retransmite pacotes perdidos)
✓ Ordenação garantida (reordena pacotes fora de ordem)
✓ Controle de fluxo (evita sobrecarga do receptor)

Custo:
- Three-way handshake ao conectar (SYN, SYN-ACK, ACK)
- Overhead de retransmissão
- Head-of-line blocking (pacote perdido bloqueia os seguintes)
```

**Default para sistemas de backend — use TCP a menos que tenha razão para não usar.**

### UDP — User Datagram Protocol

```
"Fire and forget" — envia e torce para chegar

Sem garantias:
✗ Pacotes podem ser perdidos
✗ Pacotes podem chegar fora de ordem
✗ Sem confirmação de entrega

Vantagem:
✓ Latência mínima (sem handshake, sem retransmissão)
✓ Menor overhead
```

**Quando usar UDP:**
- Video conferência (frame perdido → descarta, não retrasmite)
- Jogos multiplayer online (posição perdida → aceita a próxima)
- DNS queries (simples, rápido)
- Live streaming

**Nota:** browsers não suportam UDP nativamente — para apps web, WebRTC é a alternativa.

### QUIC

Protocolo moderno (Google, RFC 9000) que combina:
- Velocidade próxima ao UDP
- Garantias similares ao TCP
- Redução do handshake (0-RTT em conexões conhecidas)
- Multiplexação sem head-of-line blocking

Cada vez mais usado para HTTP/3.

---

## Camada 7: Protocolos de Aplicação

### HTTP/REST (Padrão)

```http
GET /users/1 HTTP/1.1
Host: api.example.com
Accept: application/json

→ 200 OK
{  "id": 1, "name": "Stefan" }
```

**REST (Representational State Transfer):**

| Verbo HTTP | Operação | Exemplo |
|---|---|---|
| GET | Leitura | GET /users/1 |
| POST | Criação | POST /users |
| PUT | Substituição | PUT /users/1 |
| PATCH | Atualização parcial | PATCH /users/1 |
| DELETE | Deleção | DELETE /users/1 |

**Recursos aninhados:** `GET /users/1/posts` → posts do usuário 1

**Use REST como padrão em 90%+ dos casos.**

---

### GraphQL

**Problema que resolve:** under-fetching (múltiplas requests) e over-fetching (dados desnecessários).

```graphql
# Uma única query para múltiplas fontes
query {
  user(id: "1") {
    profile {
      avatar
      fullName
    }
    groups {
      name
      category
    }
  }
}
```

**Quando usar GraphQL:**
- Frontend com requirements que mudam frequentemente (experimentos A/B)
- Múltiplos times de frontend consumindo o mesmo backend
- Mobile apps com consumo de dados controlado (só busca o que precisa)
- APIs públicas onde clientes têm necessidades variadas

**Quando NÃO usar:** requirements estáveis, sistema simples, team único.

---

### gRPC (Google RPC)

**Base:** Protocol Buffers (binary serialization — ~40% menor que JSON)

```protobuf
// Definição (.proto file)
message GetUserRequest { string user_id = 1; }
message UserResponse {
  string user_id = 1;
  string name = 2;
}
service UserService {
  rpc GetUser (GetUserRequest) returns (UserResponse);
}
```

**Vantagens:**
- 10× mais throughput que REST/JSON
- Type safety via schema compilado
- Suporte nativo a streaming bidirecional
- Client-side load balancing nativo

**Desvantagens:**
- Browsers não suportam nativo
- Binário dificulta debugging

**Use gRPC para:**
- Comunicação entre microserviços internos (não expostos ao browser)
- Sistemas de alto throughput onde latência é crítica
- Streaming de dados em tempo real

**Padrão híbrido:**
```
[Browser/Mobile] ←── REST/JSON ──→ [API Gateway] ←── gRPC ──→ [Microserviços]
```

---

### Server-Sent Events (SSE)

```http
GET /stream HTTP/1.1
Accept: text/event-stream

→ 200 OK
Content-Type: text/event-stream
Transfer-Encoding: chunked

data: {"event": "new_comment", "id": 123}

data: {"event": "new_comment", "id": 124}
```

**Características:**
- Unidirecional: servidor → cliente apenas
- Sobre HTTP nativo (sem upgrade de protocolo)
- Reconexão automática com `EventSource` browser API
- Proxies e firewalls não bloqueiam

**Use SSE quando:** servidor precisa notificar cliente, mas cliente raramente envia dados.

---

### WebSockets

```
Cliente → HTTP Upgrade → WebSocket connection ↕ (full-duplex)
```

**Características:**
- Bidirecional (cliente e servidor podem enviar a qualquer momento)
- Protocolo próprio (ws:// ou wss://)
- Stateful — conexão persistente requer infraestrutura dedicada
- Compatível com browsers

**Use WebSockets quando:**
- Comunicação bidirecional frequente (chat, games)
- Latência ultra-baixa necessária
- Dados fluem igualmente nos dois sentidos

**Implicação de infra:** WebSocket connections são stateful → load balancer Layer 4 necessário.

### WebRTC

- Peer-to-peer (clientes se comunicam diretamente)
- Sobre UDP (baixa latência)
- Processo de setup: Signaling Server + STUN/TURN servers

**Use apenas para:** video/audio calling, collaborative editors.

---

## Load Balancing

### Escalonamento Horizontal vs. Vertical

| | Vertical (scale up) | Horizontal (scale out) |
|---|---|---|
| Abordagem | Servidor maior/mais potente | Mais servidores |
| Complexidade | Simples | Requer load balancing |
| Limite | Hardware tem teto | Praticamente ilimitado |
| Custo | Caro em grandes escalas | Mais barato com commodity hardware |
| Downtime | Pode precisar | Rolling updates possível |

### Client-Side Load Balancing

```
Clients conhecem todos os servidores diretamente

[Client A] ──→ [Server 1]
[Client B] ──→ [Server 3]  (conhece [S1, S2, S3, S4])
[Client C] ──→ [Server 2]
```

**Quando usar:**
- Poucos clientes conhecidos (microserviços internos)
- gRPC (suporte nativo)
- DNS (funciona, mas atualização lenta — TTL de minutos a horas)

### Dedicated Load Balancer

```
[Clients] ──→ [Load Balancer] ──→ [Server 1]
                                ──→ [Server 2]
                                ──→ [Server 3]
```

**Algoritmos de distribuição:**

| Algoritmo | Quando usar |
|---|---|
| Round Robin | Requests stateless (HTTP), carga homogênea |
| Random | Similar ao Round Robin, sem estado |
| Least Connections | WebSockets (longa duração), requests heterogêneos |
| IP Hash | Sessão deve ir para o mesmo servidor |

### Layer 4 vs. Layer 7 Load Balancer

**Layer 4 (TCP):**
```
Client TCP → LB → [cria TCP paralelo] → Server
Totalmente transparente para o server
Alto desempenho (não precisa parsear HTTP)
Necessário para: WebSockets, qualquer protocolo stateful
```

**Layer 7 (HTTP):**
```
Client HTTP → LB → [parseia HTTP request] → Server
Permite routing por URL, headers, cookies
Termina HTTPS (TLS offloading)
Pode fazer fan-out de múltiplas requests HTTP pela mesma TCP connection
AWS ALB, nginx proxy
```

---

## Resiliência: Timeouts, Retries e Circuit Breaker

### Timeouts

```
Sempre defina timeouts! Sem timeout, request pendente = thread presa para sempre.

Regra: timeout > tempo máximo esperado de resposta do servidor
       mas curto o suficiente para fail-fast
```

### Retries com Exponential Backoff + Jitter

**Problema com retry simples:**
```
3 clients falham simultaneamente → todos retentam em 5s → falham juntos → 5s → falham...
"Retry storm" ou "Thundering Herd"
```

**Solução:**
```
Exponential Backoff:
  Tentativa 1 falhou → espera 2s
  Tentativa 2 falhou → espera 4s
  Tentativa 3 falhou → espera 8s → desiste

+ Jitter (aleatoriedade):
  Client A: espera 2.3s
  Client B: espera 1.8s
  Client C: espera 2.7s
  → Distribuição temporal → servidor não recebe avalanche simultânea ✓
```

### Circuit Breaker

**Problema:** cascading failures — falha em DB degrada Server B que sobrecarrega Server A.

```
Estado FECHADO (normal):
  Requests fluem normalmente
  Monitora taxa de erros

Estado ABERTO (circuit tripped):
  Falhas ultrapassam threshold → circuit abre
  Requests falham imediatamente (sem tentar)
  Dá tempo para o sistema downstream se recuperar

Estado HALF-OPEN (testando):
  Após timeout → deixa alguns requests passarem
  Se sucederem → volta ao estado FECHADO
  Se falharem → volta ao estado ABERTO
```

**Exemplo:**
```
DB com snapshot rodando → 50% mais lento
Server B → requests com timeout → retries → sobrecarga
Server A → requests para Server B → timeouts

SEM circuit breaker: Server B oscila, Server A continua sobrecarregando
COM circuit breaker: Server A detecta falha, para de tentar, Server B se recupera
```

---

## Regionalização

### O Problema de Latência Geográfica

```
London ←──── fibra ────→ New York: ~80ms latência mínima (velocidade da luz)
```

**Princípios de design regional:**

1. **Co-localização:** web server e banco de dados na mesma região
   ```
   ❌ Web server em London, DB em New York → cada request = 160ms de latência
   ✓ Ambos em London → request local = 1-5ms
   ```

2. **Particionamento geográfico:** dados dos usuários próximos aos usuários
   ```
   Uber: riders pedem motoristas na mesma cidade
   → Dados de NYC em datacenter de Virginia
   → Dados de Londres em datacenter de Dublin
   ```

3. **Replicação cross-region:** dados escritos localmente, replicados para outras regiões
   ```
   Eventual consistency aceitável para leitura cross-region
   Writes sempre na região primária do usuário
   ```

### CDN (Content Delivery Network)

```
Usuário em São Paulo → CDN edge em São Paulo → conteúdo cacheado
                                                → se miss → origem (ex: US) → cache → resposta

Benefícios:
- Latência de leitura reduzida (conteúdo próximo ao usuário)
- Redução de carga na origem
- Proteção DDoS (distribuição de tráfego)
```

**Use CDN para:** assets estáticos, conteúdo cacheável (vídeos, imagens, JS, CSS), APIs com alta leitura.

---

## Resumo: Protocolos por Caso de Uso

| Caso | Protocolo | Por quê |
|---|---|---|
| API REST padrão | HTTP/REST | Universal, simples |
| Frontend dinâmico (experimentos) | GraphQL | Flexibilidade de queries |
| Microserviços internos | gRPC | 10× mais eficiente |
| Notificações unidirecionais | SSE | Mais simples que WS |
| Chat, jogos, real-time bidirecional | WebSockets | Full-duplex necessário |
| Video/audio conferência | WebRTC | Peer-to-peer, UDP |
| Padrão para resiliência | TCP + Timeout + Exp. Backoff + Jitter | Sempre |
| Alta escala de reads | CDN + Caching | Reduz latência e custo |

---

*Fonte: Hello Interview — Networking Essentials para System Design*
