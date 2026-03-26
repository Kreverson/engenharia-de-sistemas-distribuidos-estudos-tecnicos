# 42 — Cassandra: Banco de Dados Distribuído para Alta Escrita

> Criado no Facebook em 2007 para resolver o inbox de mensagens. Inspirado no Amazon Dynamo e Google BigTable. Hoje usado por Netflix, Apple, Instagram e centenas de empresas que precisam de escrita massiva com zero downtime.

---

## 1. Por que o Cassandra existe

O Facebook precisava de um banco que:
- Suportasse **bilhões de mensagens** por dia (escrita massiva)
- **Escalasse horizontalmente** sem sharding manual
- Garantisse **zero downtime** (alta disponibilidade)
- Entregasse **baixa latência** globalmente

PostgreSQL não era opção para esse volume. O Cassandra foi a solução — e foi open-sourced em 2008.

**Tradeoff fundamental:**
```
Cassandra sacrifica:              Para ganhar:
- Joins                       →  Escalabilidade horizontal
- Strong consistency          →  Alta disponibilidade
- Flexibilidade de queries    →  Write throughput massivo
- Transações cross-table      →  Fault tolerance automático
```

---

## 2. Modelo de Dados

### Hierarquia

```
Keyspace (= database no Postgres)
  └── Table (= tabela, mas com diferenças importantes)
        └── Row (identificada por Primary Key)
              └── Columns (podem variar entre rows!)
```

### Wide Column Store — o detalhe que muda tudo

Diferente do Postgres onde toda linha tem as mesmas colunas, no Cassandra **cada linha pode ter colunas diferentes**:

```
# Postgres: todos têm 'address' mesmo que seja NULL
users: id | name | email | address
       1  | Alice| a@... | NULL
       2  | Bob  | b@... | "Rua X"

# Cassandra: Bob simplesmente não tem a coluna 'address'
users: id=1 → { name: "Alice", email: "a@..." }
       id=2 → { name: "Bob", email: "b@...", address: "Rua X" }
```

Isso torna o Cassandra eficiente para dados **esparsos** (onde muitos campos seriam NULL).

### Primary Key = Partition Key + Clustering Key

Esta é a decisão mais crítica ao usar Cassandra:

```sql
CREATE TABLE user_messages (
  user_id    UUID,
  message_id TIMEUUID,
  content    TEXT,
  
  PRIMARY KEY (user_id, message_id)
  --          ^^^^^^^^  ^^^^^^^^^^
  --        Partition   Clustering
  --           Key         Key
);
```

**Partition Key:** determina **em qual nó** os dados vivem. Todos os dados com o mesmo `user_id` ficam no mesmo nó.

**Clustering Key:** determina **como os dados são ordenados** dentro do partition. `message_id` como `TIMEUUID` = ordenação por tempo automática.

```
Nó 1: user_id=alice → [msg1, msg2, msg3...] (ordenados por message_id)
Nó 2: user_id=bob   → [msg1, msg5, msg9...]
Nó 3: user_id=carol → [msg2, msg4, msg7...]
```

**Por que esta escolha importa:**
```
Query: "Buscar todas as mensagens do usuário Alice"
→ Vai direto ao Nó 1 (1 hop, 1 nó)  ✅

Query: "Buscar todas as mensagens do dia 01/01"
→ Precisa consultar TODOS os nós e agregar  ❌ (nunca faça isso)
```

---

## 3. Escalabilidade — Como Cassandra distribui dados

### Consistent Hashing no Cassandra

O Cassandra não usa módulo simples (problemático ao adicionar/remover nós). Usa um **anel de hash virtual**:

```
Hash Ring (0 a 2^64):

         0
       ┌─┴─┐
  Nó D │   │ Nó A
       └───┘
  Nó C │   │ Nó B
       └─┬─┘
         2^64
```

Cada nó é responsável por um segmento do anel. Para encontrar onde `user_id=alice` fica:
1. `hash("alice")` → digamos que resulte em valor 1500
2. Caminha no sentido horário até encontrar o primeiro nó
3. Aquele nó é o responsável pelo dado

**Vantagem:** ao adicionar Nó E, apenas dados do segmento vizinho precisam migrar. Sem redistribuição total.

### Virtual Nodes (vnodes)

Para distribuição ainda mais uniforme, cada nó físico é responsável por **múltiplos segmentos** do anel:

```
Nó A → segmentos [100-200], [500-600], [900-1000]
Nó B → segmentos [200-300], [600-700], [1000-1100]
Nó C → segmentos [300-400], [700-800], [1100-1200]
```

Isso garante distribuição uniforme mesmo com nós de hardware diferente.

---

## 4. Alta Disponibilidade — Replicação

### Replication Factor

Definido no Keyspace:
```sql
CREATE KEYSPACE instagram
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'us-east': 3,   -- 3 cópias na região us-east
  'eu-west': 2    -- 2 cópias na região eu-west
};
```

Se `hash("alice") = Nó A` e replication_factor = 3:
- Cópia 1: Nó A (primário)
- Cópia 2: Nó B (próximo no anel)
- Cópia 3: Nó C (seguinte)

Se Nó A cair, dados ainda disponíveis em B e C.

### Gossip Protocol — sem coordenador central

Em vez de um "master" que sabe o estado do cluster (single point of failure), cada nó troca informações com 2-3 vizinhos aleatórios a cada segundo:

```
Nó A sussurra para Nó B: "Nó F está lento"
Nó B sussurra para Nó D: "A me disse que F está lento"
Nó D sussurra para Nó G: "B me disse que F está lento"
... eventualmente todos sabem sobre F
```

Isso é **gossip protocol** — informação se propaga como um boato. Em segundos, todo o cluster converge para o mesmo estado.

**Vantagem sobre Zookeeper/etcd:** sem single point of failure para coordenação do cluster.

---

## 5. CAP Theorem — Cassandra como sistema AP

### O tradeoff

Quando uma mensagem é escrita em Nó A com replication_factor = 3:
```
Nó A ← escrita (imediata)
Nó A → replica para Nó B (assíncrona)
Nó A → replica para Nó C (assíncrona)
```

Se alguém lê de Nó B antes da replicação concluir → lê dado desatualizado.

Cassandra prioriza **Disponibilidade**: o sistema continua funcionando mesmo que alguns nós estejam desincronizados.

### Tunable Consistency — o diferencial

Cassandra permite ajustar o nível de consistência **por operação**:

```
ConsistencyLevel.ONE    → Responde após 1 réplica confirmar
                          Mais rápido, menos consistente

ConsistencyLevel.QUORUM → Responde após maioria confirmar (N/2 + 1)
                          Balanço entre velocidade e consistência

ConsistencyLevel.ALL    → Espera TODAS as réplicas confirmarem
                          Mais lento, mais consistente (CP behavior)
```

**Exemplo com RF=3:**
```
QUORUM = 3/2 + 1 = 2 réplicas devem confirmar

Write com QUORUM: Nó A confirma + Nó B confirma → responde ao cliente
Read com QUORUM:  Lê de 2 nós, pega o dado com timestamp mais recente

Se você usar QUORUM em writes E reads:
→ A interseção garante que SEMPRE lê o dado mais recente
→ Cassandra se comporta como CP nessa configuração!
```

---

## 6. Write Path — Como o Cassandra escreve

```
1. Request chega em QUALQUER nó (coordinator)
2. Coordinator determina réplicas responsáveis via consistent hashing
3. Write é enviado para as réplicas em PARALELO
4. Em cada réplica:
   ┌─────────────────────────────────────────────┐
   │   a) Escreve no COMMIT LOG (disco, append)  │  ← durabilidade
   │   b) Escreve na MEMTABLE (RAM, ordenada)    │  ← velocidade
   │   c) Responde "OK" ao coordinator           │
   └─────────────────────────────────────────────┘
5. Quando MEMTABLE enche → flush para SSTABLE (disco, imutável)
6. SSTables são periodicamente compactadas
```

### Por que as writes do Cassandra são tão rápidas

**Postgres/MySQL (B-Tree):**
```
INSERT → encontrar página correta na árvore → seek no disco → atualizar no lugar
→ Random I/O → ~200 seeks/s em HDD
```

**Cassandra (LSM-Tree via SSTables):**
```
INSERT → append no commit log → insert na memtable (RAM)
→ Sequential I/O → flush periódico em batch
→ Sem random seeks, sem locks, sem coordenação
```

A chave é o **append-only sequential I/O** — sempre escrevendo no fim, nunca buscando e atualizando no meio.

### Compaction — a housekeeping do Cassandra

SSTables são imutáveis. Para "atualizar" alice@old.com para alice@new.com:
```
SSTable 1: { user_id=alice, email: "old@..." }  (versão antiga)
SSTable 2: { user_id=alice, email: "new@..." }  (versão nova)
```

A compaction periodicamente merge as SSTables, mantendo apenas a versão mais recente (maior timestamp).

---

## 7. Read Path — Como o Cassandra lê

```
1. Request chega no coordinator
2. Coordinator identifica réplicas responsáveis
3. Query enviada para réplicas em paralelo (baseado no CL escolhido)
4. Em cada réplica:
   a) Verifica BLOOM FILTER → "esse dado possivelmente existe nesta SSTable?"
   b) Se sim, busca no INDEX da SSTable → offset exato na SSTable
   c) Lê da MEMTABLE (se ainda não flushou) + SSTables relevantes
5. Coordinator pega a resposta com maior timestamp
6. Se réplicas discordam → read repair em background
```

### Bloom Filter — otimização de leitura

Um bloom filter é uma estrutura probabilística que responde: **"esse item definitivamente NÃO está aqui" ou "pode estar aqui"**.

Sem bloom filter: abrir e varrer todas as SSTables para encontrar alice.  
Com bloom filter: 90% das SSTables são descartadas sem sequer abri-las.

---

## 8. Data Modeling no Cassandra — Query Driven Design

Esta é a diferença mais importante vs bancos relacionais:

**Postgres → Entity-Driven Design:**
```
1. Modele as entidades (users, posts, likes)
2. Normalize os dados
3. Use JOINs para combinar na query time
```

**Cassandra → Query-Driven Design:**
```
1. Comece pelas queries que o sistema precisa responder
2. Crie uma tabela para cada query
3. Duplique dados se necessário (sem JOINs!)
```

### Exemplo prático

**Queries necessárias:**
1. Buscar todas as mensagens de um usuário, ordenadas por data
2. Buscar uma mensagem específica por ID

```sql
-- Query 1: mensagens por usuário
CREATE TABLE messages_by_user (
  user_id    UUID,
  created_at TIMESTAMP,
  message_id UUID,
  content    TEXT,
  PRIMARY KEY (user_id, created_at, message_id)
) WITH CLUSTERING ORDER BY (created_at DESC);

-- Query 2: mensagem específica (tabela separada!)
CREATE TABLE messages_by_id (
  message_id UUID PRIMARY KEY,
  user_id    UUID,
  content    TEXT,
  created_at TIMESTAMP
);
```

Sim, os dados de `content` estão duplicados em duas tabelas. **Isso é intencional e correto no Cassandra.** Storage é barato; queries lentas são caras.

### Red flag em entrevistas

Se você mencionar Cassandra mas modelar tabelas normalizadas com FKs, o entrevistador saberá que você não entende realmente como ele funciona.

---

## 9. Quando usar Cassandra em entrevistas

### Use Cassandra quando:

| Cenário | Exemplo |
|---|---|
| Write throughput > 100K ops/s | Feed de eventos, logs de IoT |
| Patterns de query simples e conhecidos | "Mensagens por usuário ordenadas por data" |
| Dados com TTL natural | Sessões, logs temporários |
| Distribuição geográfica | Multi-region com réplicas por datacenter |

**Exemplos clássicos:**
- **Activity feeds** (Twitter, Instagram)
- **Messaging systems** (origin: Facebook Inbox)
- **Time-series data** (métricas, logs)
- **Recommendation data** (dados de comportamento de usuário)

### Não use Cassandra quando:

- Precisar de JOINs complexos ou queries ad-hoc
- Precisar de transações cross-table (ACID)
- Queries de analytics/aggregation complexas
- Times pequenos sem expertise em Cassandra (operacional complexo)

---

## 10. Comparativo Cassandra vs PostgreSQL

| Aspecto | PostgreSQL | Cassandra |
|---|---|---|
| Modelo | Relacional, normalizado | Wide-column, desnormalizado |
| Write throughput | ~10K ops/s | >100K ops/s |
| Escalabilidade | Vertical + read replicas | Horizontal automático |
| Queries | Flexíveis (SQL completo) | Limitadas (query-driven) |
| Consistência | ACID forte | Eventual (tunável) |
| Joins | ✅ | ❌ |
| Transações | ✅ Multi-table | ⚠️ Single-partition apenas |
| Operacional | Simples | Complexo |

---

## Resumo dos conceitos-chave

```
CASSANDRA = LSM-Tree + Consistent Hashing + Gossip Protocol + Tunable Consistency

LSM-Tree    → writes rápidos via append-only + compaction
Cons. Hash  → distribuição uniforme, rebalanceamento mínimo
Gossip      → cluster auto-gerenciado sem single point of failure
Tunable CL  → você escolhe consistência vs disponibilidade por operação

Partition Key  → ONDE fica o dado (qual nó)
Clustering Key → COMO está ordenado dentro do nó
Desnormalização → REGRA, não exceção — query-driven design
```

---

*Tópicos relacionados: `03_consistencia_e_disponibilidade.md`, `27_consistent_hashing.md`, `23_dynamodb_deep_dive.md`, `08_concorrencia_e_protocolo.md`*
