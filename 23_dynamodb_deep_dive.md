# 23 — DynamoDB Deep Dive

> Tudo que você precisa saber sobre DynamoDB para usar com confiança em System Design Interviews

---

## O Modelo de Dados

DynamoDB organiza dados em três conceitos principais:

```
Table → Item (row) → Attribute (column)
```

| Conceito DynamoDB | Equivalente SQL |
|---|---|
| Table | Tabela |
| Item | Row / Registro |
| Attribute | Column |

### Schemaless (sem esquema fixo)

Diferente do SQL, itens na mesma tabela **podem ter atributos diferentes**:

```json
// Item 1
{ "user_id": "1", "name": "Stefan", "email": "stefan@..." }

// Item 2 — sem "email", mas com "employer"
{ "user_id": "2", "name": "Sarah", "employer": "Meta" }
```

**Prós:** flexibilidade, sem migrations, paga só pelo que armazena  
**Contras:** aplicação precisa tratar atributos ausentes, análise cross-item mais difícil

---

## Indexação: Partition Key e Sort Key

### Partition Key (PK)

- Identificador que determina **em qual nó físico** o item é armazenado
- Passa por função de hash consistente → define a partição
- Queries por PK = O(1) lookup

### Sort Key (SK)

- Opcional, mas poderoso
- Habilita **ordenação e range queries** dentro de uma partição
- Implementado como B-Tree em memória

### Primary Key = PK + SK

O par (PK, SK) deve ser **único** em toda a tabela.

**Por que usar SK como ID e não timestamp?**

```
Problema: dois itens criados no mesmo milissegundo teriam PK+SK duplicados
Solução: usar IDs monotonicamente crescentes (Snowflake IDs, ULIDs)
  → ordenação por ID = ordenação por tempo
  → unicidade garantida
```

### Exemplos Práticos

| Sistema | Partition Key | Sort Key | Rationale |
|---|---|---|---|
| Chat App | `chat_id` | `message_id` (crescente) | Coloca msgs do mesmo chat juntas, ordena por tempo |
| E-commerce | `user_id` | `order_id` (crescente) | Todos pedidos de um usuário juntos |
| Social Media | `user_id` | `post_id` (crescente) | Posts de um usuário ordenados por tempo |

---

## Índices Secundários

### Global Secondary Index (GSI)

**Problema que resolve:** preciso de outro padrão de query além do PK original.

```
Tabela de mensagens:
  PK: chat_id → "todos os msgs de um chat" ✓
  Preciso também: "todas as msgs enviadas por um user" ← GSI!

GSI com PK: user_id → cria "réplica" da tabela particionada por user_id
```

**Analogia:** criar uma tabela espelho com reorganização diferente dos dados.

**Custo:** storage adicional (apenas os atributos projetados), write overhead

### Local Secondary Index (LSI)

**Problema que resolve:** quero ordenar pelo mesmo PK, mas por uma coluna diferente do SK original.

```
Tabela de mensagens:
  PK: chat_id | SK: message_id (ordenação por tempo) ← original
  Quero também: ordenar por número de attachments ← LSI!

LSI: mesmo PK (chat_id), SK alternativo (num_attachments)
```

**Regra:** LSI deve ser criado no momento de criação da tabela (não pode ser adicionado depois).

---

## Arquitetura Interna

### Consistent Hashing

DynamoDB distribui dados entre nós usando **consistent hashing** na partition key:

```
hash(partition_key) → posição no anel → nó responsável
Adição/remoção de nós → rebalanceia apenas vizinhos imediatos
```

### Replicação e Consistência

```
Write → Leader node → replica assíncrona para múltiplas AZs

Leitura padrão (eventual consistency):
  → pode ir para qualquer réplica
  → possível lag de milissegundos

Leitura com strong consistency (opt-in):
  → sempre vai para o Leader
  → sem lag, mas maior custo e menor throughput
```

**Sloppy Quorum:** DynamoDB aceita writes mesmo durante partições de rede, corrigindo inconsistências depois — favorece disponibilidade.

---

## Transações

Introduzidas em 2018, DynamoDB suporta operações atômicas tipo SQL:

```javascript
// Equivalente a BEGIN TRANSACTION no SQL
dynamoDB.transactWrite({
  TransactItems: [
    {
      Update: {
        TableName: "accounts",
        Key: { user_id: "A" },
        UpdateExpression: "SET balance = balance - :amount",
        ExpressionAttributeValues: { ":amount": 100 }
      }
    },
    {
      Update: {
        TableName: "accounts",
        Key: { user_id: "B" },
        UpdateExpression: "SET balance = balance + :amount",
        ExpressionAttributeValues: { ":amount": 100 }
      }
    }
  ]
});
```

**Limite importante:** máximo de **100 itens por transação** (relevante para grupos em chat apps).

---

## DAX — DynamoDB Accelerator

Cache in-memory nativo do DynamoDB:

```
Client → DAX (microsegundos) → DynamoDB (milissegundos)
```

**Quando usar DAX vs. Redis:**

| Situação | Recomendação |
|---|---|
| Já usando DynamoDB | Use DAX (plug-and-play) |
| Precisa de estruturas avançadas (sorted sets, pub/sub) | Redis |
| Cache multi-banco | Redis |
| Latência < 1ms em queries específicas | DAX |

> **Dica de interview:** se introduziu DynamoDB e precisa de cache, diga "vou habilitar DAX" — não introduza Redis desnecessariamente.

---

## DynamoDB Streams (CDC)

Change Data Capture nativo: qualquer insert/update/delete pode triggar um evento downstream.

### Caso de Uso: Sincronização com Elasticsearch

```
[Events DB (DynamoDB)]
     ↓ DynamoDB Streams
[Lambda Function]
     ↓
[Elasticsearch]
  (full-text search, geospatial queries)
```

Quando usar DDB Streams em vez de outras abordagens:
- Sincronização eventual entre DynamoDB e outro banco
- Triggering de processamentos assíncronos (envio de email, notificações)
- Auditoria de mudanças

---

## Quando Usar DynamoDB (e quando não usar)

### Use DynamoDB quando:

- Padrões de query são simples e bem definidos
- Alta escala de writes/reads com baixa latência
- Schema flexível é desejável
- Dados podem ser modelados com PK/SK bem definidos

### Evite DynamoDB quando:

| Situação | Motivo |
|---|---|
| JOINs complexos entre múltiplas tabelas | SQL é mais adequado |
| Transações com >100 itens | Limite da API de transações |
| Muitos GSIs/LSIs (>5) | Sinal de modelagem errada — use SQL |
| Analytics complexas | SQL/OLAP é mais adequado |

### Recomendação Final

> Aprenda **PostgreSQL** e **DynamoDB** — você consegue usar um deles na maioria das entrevistas. Ambos são capazes para a maior parte dos casos de uso; a diferença está nos trade-offs específicos de cada problema.

---

## Queries no Código

```javascript
// Scan completo (Full Table Scan — evitar em produção)
dynamoDB.scan({ TableName: "users" });

// Query por PK
dynamoDB.query({
  TableName: "users",
  KeyConditionExpression: "user_id = :id",
  ExpressionAttributeValues: { ":id": "101" }
});

// Query com sort (descendente)
dynamoDB.query({
  TableName: "orders",
  KeyConditionExpression: "user_id = :id",
  ExpressionAttributeValues: { ":id": "user_1" },
  ScanIndexForward: false  // false = descendente
});
```

---

## Resumo dos Recursos

| Recurso | Descrição | Quando mencionar no Interview |
|---|---|---|
| Partition Key | Determina o nó físico | Sempre — define padrão de acesso |
| Sort Key | Ordenação dentro da partição | Sempre que precisar de range queries |
| GSI | Query por outro atributo | Quando precisar de 2 padrões de acesso |
| LSI | Sort por outra coluna | Quando precisar de 2 ordenações |
| Transactions | Atomicidade multi-item | Quando precisar de consistência entre tabelas |
| DAX | Cache in-memory nativo | Quando latência é crítica |
| Streams | CDC nativo | Quando precisar sincronizar com outro sistema |
| Strong Consistency | Reads sempre do leader | Quando CAP → Consistency |

---

*Fonte: Hello Interview — DynamoDB Deep Dive*
