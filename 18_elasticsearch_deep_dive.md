# 18 — Elasticsearch Deep Dive: Índice Invertido e Arquitetura Distribuída

> **Por que estudar Elasticsearch:** aparece em system design sempre que há busca full-text, geo queries, ou análise de logs em escala. Entender os internals permite saber quando usar, quando não usar, e como justificar a escolha.

---

## 1. O que é Elasticsearch

Elasticsearch é uma **camada de orquestração distribuída** sobre o Apache Lucene.

```
Elasticsearch = Lucene + Distribuição + API REST

Lucene:          busca eficiente em um único nó
Elasticsearch:   gerencia múltiplos nós, APIs, replicação, sharding
```

**Melhor para:**
- Full-text search
- Geo queries
- Analytics/aggregations em tempo real
- Log indexing (stack ELK)

**Não usar como:**
- Banco de dados primário (durabilidade fraca)
- Workloads write-heavy (reindexação cara)
- Transações ACID

---

## 2. Conceitos Fundamentais

### Index
Coleção lógica de documentos (como uma tabela, mas schema-flexible).

### Documento
JSON blob — a unidade básica de armazenamento.
```json
{
  "id": "event_123",
  "name": "Taylor Swift Eras Tour",
  "venue": { "city": "São Paulo", "location": { "lat": -23.5, "lon": -46.6 }},
  "date": "2025-03-15",
  "price_min": 150.00,
  "tags": ["music", "pop", "concert"]
}
```

### Mapping
Define os tipos dos campos (como o schema que o Elasticsearch indexará):
```json
{
  "mappings": {
    "properties": {
      "name":     { "type": "text" },    // full-text search
      "category": { "type": "keyword" }, // exact match
      "price":    { "type": "float" },   // range queries
      "location": { "type": "geo_point" } // geo queries
    }
  }
}
```

**text vs keyword:**
- `text`: tokenizado para full-text search ("Taylor Swift" → ["taylor", "swift"])
- `keyword`: opaco, lookup exato ("Taylor Swift" só bate com "Taylor Swift")

---

## 3. O Índice Invertido (Inverted Index)

É a estrutura de dados central que torna o full-text search rápido.

**Problema:** dado 1 milhão de documentos, como encontrar todos que contêm "taylor" em < 100ms?

**Sem índice invertido:** escanear todos os documentos → O(N × M) onde M = tamanho médio do documento.

**Com índice invertido:** pré-computar um mapa de termo → documentos que o contêm.

```
Documentos:
  doc_1: "Taylor Swift Eras Tour São Paulo"
  doc_2: "Consulado Taylor São Paulo Eventos"
  doc_3: "Show de Rock São Paulo"

Tokenização:
  doc_1 → ["taylor", "swift", "eras", "tour", "são", "paulo"]
  doc_2 → ["consulado", "taylor", "são", "paulo", "eventos"]
  doc_3 → ["show", "rock", "são", "paulo"]

Índice invertido:
  "taylor"    → [doc_1, doc_2]
  "swift"     → [doc_1]
  "são"       → [doc_1, doc_2, doc_3]
  "paulo"     → [doc_1, doc_2, doc_3]
  "eras"      → [doc_1]
  "consulado" → [doc_2]
  "rock"      → [doc_3]

Query "taylor paulo":
  "taylor" → [doc_1, doc_2]
  "paulo"  → [doc_1, doc_2, doc_3]
  Interseção → [doc_1, doc_2] ← O(log N) com B-tree no índice
```

**Tokenização e Stemming:**
- Tokenização: divide texto em termos
- Stemming: "running" → "run", "correndo" → "corr"
- Stop words: "o", "a", "de" são frequentemente ignorados
- Isso melhora o recall (menos miss) ao custo de algumas precision

---

## 4. Relevance Score (TF-IDF)

Por padrão, Elasticsearch ordena por relevância, não por inserção.

**TF-IDF (Term Frequency × Inverse Document Frequency):**

```
TF(term, doc)  = frequência do termo no documento
                 "taylor" aparece 3x em doc_1 → TF = 3

IDF(term)      = log(N / documentos_com_o_termo)
                 "são" aparece em 1M docs → IDF baixo (pouco informativo)
                 "elasticsearch" aparece em 100 docs → IDF alto (muito informativo)

Score = TF × IDF
```

**Intuição:** um documento que menciona "elasticsearch" 5 vezes é mais relevante para a query "elasticsearch" do que um que menciona uma vez, mas a palavra "de" tem pouco valor mesmo aparecendo 100 vezes.

---

## 5. Arquitetura do Cluster

### Tipos de Nós

```
Master Node:
  - Gerencia o cluster (1 ativo, múltiplos elegíveis)
  - Decide particionamento, monitoramento
  - Precisa ser confiável (evitar splits)

Coordinating Node:
  - Recebe requests externos (a "porta de entrada")
  - Parse a query, distribui para os data nodes certos
  - Merge e return os resultados

Data Node:
  - Armazena os shards (dados reais)
  - Executa as queries localmente
  - I/O intensivo — precisa de bom disco/memória

Ingest Node:
  - Pré-processa documentos antes de indexar
  - CPU intensivo
```

### Shards e Réplicas

```
Index "events" com 3 primary shards e 1 réplica:

  Primary Shard 0 → Data Node 1
  Replica  Shard 0 → Data Node 2

  Primary Shard 1 → Data Node 2
  Replica  Shard 1 → Data Node 3

  Primary Shard 2 → Data Node 3
  Replica  Shard 2 → Data Node 1
```

**Primary shards:** recebem writes, são a fonte da verdade.
**Réplicas:** servem reads, aumentam throughput de leitura.

**Quando uma query chega:**
1. Coordinating node recebe: `GET /events/_search?q=taylor`
2. Distribui para todos os 3 primary shards (ou suas réplicas)
3. Cada shard executa a query localmente e retorna top K resultados
4. Coordinating node faz merge e retorna o top K global

---

## 6. Lucene: Os Internals do Shard

Cada shard é um **Lucene Index** — contém múltiplos **segments**.

```
Shard 0:
  ├── Segment 1 (imutável, 10.000 docs)
  ├── Segment 2 (imutável, 5.000 docs)
  └── Segment 3 (em criação, em memória)
```

**Por que imutável?**
- Permite lock-free reads (múltiplos threads lendo simultaneamente)
- Permite compressão agressiva em disco
- Cache pages de forma confiável

**Update/Delete:** não modifica o segment existente. Cria um "tombstone" — marcação de deletado.

```
Segment 1: [doc_A] [doc_B(DELETED)] [doc_C]
                         ↑
                   marcado como deletado

Próxima query: doc_B é excluído dos resultados automaticamente
Compaction: periodicamente, Lucene mescla segments e remove tombstones
```

### Doc Values (Columnar Store para Sorting)

Para sorting e aggregations eficientes, Lucene mantém uma cópia columnar dos dados:

```
Row-based (para busca por documento):
  doc_1: { name: "Taylor", price: 150, date: "2025-03-15" }
  doc_2: { name: "Coldplay", price: 200, date: "2025-04-20" }

Columnar (Doc Values, para sorting/aggregation):
  price column: [150, 200, 300, 80, ...]  ← acesso sequencial rápido
  date column:  ["2025-03-15", "2025-04-20", ...]
```

Se você quer os 10 eventos mais baratos, não precisa carregar todos os campos de todos os docs — apenas a coluna `price`.

---

## 7. Paginação: From/Size vs. Search After

### From/Size (Simples, mas Limitado)
```json
{ "from": 0,  "size": 10 }   // página 1
{ "from": 10, "size": 10 }   // página 2
{ "from": 100000, "size": 10 } // PROBLEMA: carrega 100.010 docs em memória!
```

**Limite:** Elasticsearch rejeita `from` > 10.000 por padrão.

### Search After (Eficiente, Cursor-based)

```json
// Primeira page — sem search_after
{
  "size": 10,
  "sort": [{"date": "asc"}, {"_id": "asc"}]
}

// Segunda page — usar os valores do último doc retornado
{
  "size": 10,
  "sort": [{"date": "asc"}, {"_id": "asc"}],
  "search_after": ["2025-03-15", "event_123"]
}
```

Elasticsearch não precisa carregar docs anteriores — apenas começa a partir dos valores fornecidos.

### Point in Time (Snapshot para Consistência)

```python
# Criar snapshot do índice
pit = es.open_point_in_time(index="events", keep_alive="1m")
pit_id = pit["id"]

# Buscar com snapshot consistente
results = es.search(
    pit={"id": pit_id, "keep_alive": "1m"},
    sort=[{"date": "asc"}],
    size=10
)

# Fechar quando terminar
es.close_point_in_time(id=pit_id)
```

Sem PIT, se um documento é adicionado/removido entre páginas, você pode ver duplicatas ou pular documentos.

---

## 8. Sincronização com o Banco Principal (CDC)

Elasticsearch nunca deve ser o banco principal.

```
Fluxo com Change Data Capture:

PostgreSQL:
  INSERT event → wal log entry
  UPDATE event → wal log entry
  DELETE event → wal log entry

Debezium (CDC connector):
  Lê o WAL do PostgreSQL
  Publica no Kafka: { operation: "INSERT", table: "events", data: {...} }

Elasticsearch Sink Connector:
  Consome do Kafka
  Indexa no Elasticsearch

Latência: alguns segundos de lag (eventual consistency)
```

**Alternativa simples (menos robusta):**
```python
# No código da aplicação, após salvar no banco:
def create_event(event_data):
    event = db.save(event_data)
    es.index(index="events", id=event.id, body=event_data)
    return event
```

Problema: se o ES estiver indisponível, os dados ficam dessincronizados.

---

## 9. Boas Práticas em Entrevistas

```
✅ Use Elasticsearch quando:
   - Full-text search (busca por termo/frase)
   - Geo queries (localização + distância)
   - Aggregations em escala (logs, analytics)

❌ Não use quando:
   - É o banco primário
   - Writes muito frequentes (re-indexação cara)
   - Precisa de ACID transactions
   - Volume é pequeno (PostgreSQL com LIKE e índice GIN funciona)

⚠️ Sempre mencionar:
   - CDC para sincronização com o banco primário
   - Eventual consistency (segundos de delay)
   - Denormalization (dados redundantes para queries mais rápidas)
```

---

## Referências

- [Elasticsearch Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/)
- [Lucene Internals — Apache](https://lucene.apache.org/core/)
- [Hello Interview — Elasticsearch Deep Dive](https://hellointerview.com)
