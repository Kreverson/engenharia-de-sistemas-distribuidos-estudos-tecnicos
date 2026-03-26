# 39 — Design de APIs: REST, GraphQL e RPC

> Baseado na série Hello Interview. Foco em entrevistas de System Design, mas extensível para prática profissional.

---

## 1. O que é uma API e por que importa

Uma **API (Application Programming Interface)** é o contrato que define como dois sistemas de software se comunicam. Em uma arquitetura típica:

```
Cliente (browser/app) ──► API Gateway ──► Microserviço ──► Banco de Dados
```

Em entrevistas de System Design, o foco está nas **APIs externas** — aquelas expostas ao cliente — e não nas internas. Mas saber distinguir quando usar cada protocolo é sinal de maturidade técnica.

---

## 2. REST — O padrão dominante

### Princípios fundamentais

REST (**Representational State Transfer**) é construído sobre HTTP e organiza tudo em torno de **recursos** (substantivos no plural):

| Errado ❌ | Correto ✅ |
|---|---|
| `POST /createEvent` | `POST /events` |
| `GET /getUser/123` | `GET /users/123` |
| `DELETE /removeTicket` | `DELETE /tickets/456` |

> **Regra de ouro:** O verbo HTTP expressa a ação. A URL expressa o recurso.

### Verbos HTTP e suas semânticas

| Verbo | Semântica | Idempotente? | Exemplo |
|---|---|---|---|
| `GET` | Leitura | ✅ Sim | `GET /events/123` |
| `POST` | Criação | ❌ Não | `POST /events` |
| `PUT` | Substituição total | ✅ Sim | `PUT /events/123` |
| `PATCH` | Atualização parcial | ✅ Sim | `PATCH /events/123` |
| `DELETE` | Remoção | ✅ Sim | `DELETE /events/123` |

**Idempotência** significa que executar a mesma operação N vezes produz o mesmo resultado. Crítico para sistemas distribuídos com retries.

### Tipos de parâmetros

```
GET /events/123/tickets?status=available&page=1&limit=25
        ↑                  ↑                 ↑
   Path param          Query params       Pagination
  (obrigatório)        (opcionais)
```

**Path parameters** → identificam o recurso específico (obrigatório para localizar o objeto)

**Query parameters** → filtros opcionais que refinam a busca

**Request body** → dados enviados ao criar/atualizar (POST, PUT, PATCH)

```json
POST /events
{
  "title": "Rock in Rio",
  "date": "2025-09-15",
  "location": "Rio de Janeiro",
  "capacity": 100000
}
```

### Códigos de Status HTTP

Em vez de memorizar todos os códigos, agrupe em categorias:

| Faixa | Significado | Exemplos comuns |
|---|---|---|
| `2xx` | Sucesso | `200 OK`, `201 Created` |
| `4xx` | Erro do cliente | `400 Bad Request`, `401 Unauthorized`, `404 Not Found` |
| `5xx` | Erro do servidor | `500 Internal Server Error` |

### Paginação — tipos e trade-offs

#### Offset-based (mais simples)
```
GET /events?page=2&limit=25
```
- SQL: `SELECT * FROM events LIMIT 25 OFFSET 25`
- **Problema:** com writes simultâneos, itens podem aparecer duplicados ou desaparecer entre páginas

#### Cursor-based (mais robusto)
```
GET /events?cursor=evt_abc123&limit=25
```
- O cursor é o ID do último item recebido
- SQL: `SELECT * FROM events WHERE id > 'evt_abc123' ORDER BY id LIMIT 25`
- **Vantagem:** estável mesmo com inserções concorrentes. Usado pelo Twitter, Instagram, etc.

---

## 3. GraphQL — Flexibilidade para múltiplos clientes

### A origem e o problema que resolve

O Facebook criou o GraphQL em 2012 para resolver o problema de **over-fetching** e **under-fetching** em APIs REST quando múltiplos clientes (web, mobile, TV) precisam de dados diferentes.

**Problema REST com múltiplos clientes:**
- Web precisa: evento + venue + ingressos
- Mobile precisa: apenas nome e data do evento
- Solução REST: criar endpoints específicos (proliferação) OU retornar tudo sempre (payload enorme)

**Solução GraphQL:** o cliente especifica exatamente o que quer.

```graphql
query {
  event(id: "123") {
    name
    date
    venue {
      name
      address
    }
    tickets {
      section
      price
      availability
    }
  }
}
```

Tudo isso em **uma única requisição HTTP** (POST para um único endpoint).

### O problema N+1

O principal pitfall do GraphQL em produção:

```
1 query para buscar 100 eventos → 1 query ao banco
100 queries para buscar o venue de cada evento → N queries ao banco
Total: N+1 queries
```

**Solução:** **DataLoader** — agrupa todas as buscas de venues em uma única query batched:
```
SELECT * FROM venues WHERE id IN (1, 2, 3, ..., 100)
```

### Autorização em nível de campo

GraphQL permite controlar acesso campo a campo via **resolvers**, algo impossível no REST tradicional:
```
event.name → público
event.revenue → apenas admins
event.costBreakdown → apenas financeiro
```

### Quando usar GraphQL vs REST

| Critério | REST | GraphQL |
|---|---|---|
| Múltiplos clientes com necessidades diferentes | ⚠️ Complexo | ✅ Ideal |
| API pública consumida por terceiros | ✅ Padrão | ⚠️ Curva de aprendizado |
| Schema estável e simples | ✅ Simples | ⚠️ Overhead desnecessário |
| Mobile com banda limitada | ⚠️ Over-fetching | ✅ Payload exato |

---

## 4. RPC — Comunicação entre microserviços

### Quando REST não é suficiente

REST carrega overhead por natureza:
- Headers HTTP verbosos
- URLs para parsing
- Serialização JSON (texto legível mas pesado)
- Status codes para interpretar

Para comunicação **interna entre microserviços**, isso é custo sem benefício.

### gRPC e Protocol Buffers

O **gRPC** (Google Remote Procedure Call) usa **Protocol Buffers** (protobuf) — formato binário compacto:

```protobuf
service TicketService {
  rpc GetEvent (GetEventRequest) returns (Event);
  rpc CreateBooking (CreateBookingRequest) returns (Booking);
  rpc GetAvailableTickets (TicketsRequest) returns (TicketList);
}

message GetEventRequest {
  string event_id = 1;
}

message Event {
  string id = 1;
  string title = 2;
  string date = 3;
  int32 capacity = 4;
}
```

**Vantagens do protobuf sobre JSON:**
- 5-10x mais compacto (bytes vs texto)
- Tipagem forte (schema enforçado)
- Geração automática de clientes em múltiplas linguagens
- Backward/forward compatibility por design

### Por que não usar gRPC para APIs externas?

| Critério | REST/HTTP | gRPC |
|---|---|---|
| Compatibilidade universal | ✅ Todo browser, cliente HTTP | ❌ Requer library específica |
| Debug | ✅ Legível por humanos | ❌ Binário |
| Firewall/proxy | ✅ Funciona everywhere | ⚠️ HTTP/2 requerido |
| Performance | ⚠️ Overhead de texto | ✅ Binário compacto |
| Contrato entre serviços | ⚠️ Informal | ✅ Schema enforçado |

**Regra prática:**
- **REST** → fronteira externa (cliente ↔ backend)
- **gRPC** → comunicação interna (serviço ↔ serviço)

---

## 5. Segurança em APIs

### JWT (JSON Web Token)

Estrutura: `header.payload.signature` (base64 encoded)

```json
// Payload (decodificado)
{
  "user_id": "usr_abc123",
  "role": "admin",
  "exp": 1735689600
}
```

A **assinatura** garante que o cliente não pode alterar o payload sem que o servidor detecte. Isso permite que o servidor valide o token sem consultar o banco de dados em cada request.

**Armadilha de segurança comum em entrevistas:**

```
❌ ERRADO:
POST /tweets
{
  "text": "Hello world",
  "user_id": "malicious_id"  // qualquer um pode forjar isso!
}

✅ CORRETO:
POST /tweets
{
  "text": "Hello world"
  // user_id extraído do JWT no header Authorization
}
```

O `user_id` NUNCA deve vir do request body para operações de escrita — deve ser extraído do token de autenticação.

### Session Token vs JWT

| | JWT | Session Token |
|---|---|---|
| Armazenamento | No cliente (stateless) | No servidor/Redis |
| Revogação | Difícil (aguarda expirar) | Imediata (deleta da store) |
| Escalabilidade | ✅ Sem state no servidor | ⚠️ Precisa de store compartilhada |
| Tamanho | Maior (payload codificado) | Pequeno (apenas um ID) |

---

## 6. Padrões de resposta e contratos de API

### Estrutura de resposta consistente

```json
// Sucesso
{
  "data": { "id": "evt_123", "title": "Rock in Rio" },
  "meta": { "timestamp": "2025-01-01T00:00:00Z" }
}

// Erro
{
  "error": {
    "code": "EVENT_NOT_FOUND",
    "message": "Event with id evt_999 not found",
    "details": { "requested_id": "evt_999" }
  }
}
```

### Versionamento de APIs

Estratégias comuns:
- **URL versioning:** `/api/v1/events`, `/api/v2/events` (mais explícito)
- **Header versioning:** `Accept: application/vnd.api+json;version=2` (mais RESTful purista)
- **Query param:** `/events?version=2` (menos comum)

---

## 7. Como abordar API Design em entrevistas

### Framework recomendado (máx. 5 minutos)

```
1. Identifique os recursos (vêm das entidades)
2. Defina os endpoints principais (mapeiam os requisitos funcionais)
3. Especifique inputs (path, query, body)
4. Esboce outputs (usando os tipos das entidades)
5. Mencione autenticação onde relevante
6. Mencione paginação onde necessário
```

### Exemplo completo — Ticket Master

```
// Listar eventos com filtros
GET /events?city=rio&date=2025-09-15&page=1&limit=25
→ 200: { events: [Event], meta: { total, cursor } }

// Detalhar evento específico
GET /events/{event_id}
→ 200: Event | 404: Not Found

// Listar ingressos disponíveis
GET /events/{event_id}/tickets?section=pista
→ 200: { tickets: [Ticket] }

// Reservar ingresso (autenticado)
POST /bookings   🔒
body: { event_id, ticket_id, quantity }
→ 201: Booking | 409: Conflict (já reservado)

// Cancelar reserva (autenticado, apenas o dono)
DELETE /bookings/{booking_id}   🔒
→ 204: No Content | 403: Forbidden
```

---

## 8. Pontos de diferenciação para entrevistas sênior

- **Discutir idempotência:** como garantir que retries não causem efeitos duplicados? (idempotency keys)
- **Rate limiting por endpoint:** estratégias diferentes para leitura vs escrita
- **Backward compatibility:** como evoluir a API sem quebrar clientes existentes?
- **Contract testing:** como garantir que produtor e consumidor estejam alinhados?
- **API Gateway vs BFF:** quando criar um Backend for Frontend específico por tipo de cliente?

---

## Resumo Visual

```
                    CLIENTES
                  /    |    \
               Web   Mobile  3rd-party
                 \    |    /
              [API GATEWAY]
                  REST/HTTP
              /      |      \
        [Auth]  [Rate Limit]  [Routing]
            \       |       /
         Microserviço A  Microserviço B
              gRPC/Protobuf
                    |
              [Banco de Dados]
```

| Camada | Protocolo | Por quê |
|---|---|---|
| Cliente → Gateway | REST ou GraphQL | Universalidade, legibilidade |
| Gateway → Serviços | gRPC | Performance, contrato forte |
| Serviços → Serviços | gRPC | Idem |

---

*Próximos tópicos relacionados: `10_framework_system_design.md`, `31_api_gateway.md`, `28_networking_essentials.md`*
