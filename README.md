# 📚 Engenharia de Sistemas Distribuídos — Estudos Técnicos

> Compilação de conceitos, padrões e experiências reais extraídas de casos como Knight Capital, Pix, Nubank, Slack, Instagram, Amazon Dynamo, LinkedIn/Kafka, Twitter, WhatsApp, Uber, Ticket Master, Tinder, Dropbox, WebCrawler e mais.

---

## Módulo 1 — Fundamentos de Sistemas Distribuídos

| Arquivo | Conteúdo |
|---|---|
| `01_falhas_e_divida_tecnica.md` | Knight Capital: deploy manual, dead code, flags reutilizadas, circuit breaker |
| `02_rate_limiting_e_seguranca.md` | Pix/DICT: Token Bucket, Weighted Rate Limiting, Race Condition, Redis atômico |
| `03_consistencia_e_disponibilidade.md` | Amazon Dynamo: Teorema CAP, Consistent Hashing, Vector Clocks, eventual consistency |
| `04_event_streaming_kafka.md` | Kafka/LinkedIn: log distribuído, tópicos, partições, page cache, sendfile |
| `05_ids_distribuidos.md` | Instagram: Snowflake IDs, bit shifting, geração sem coordenação central |
| `06_thundering_herd.md` | Slack: Thunder Herd, backoff exponencial com jitter, circuit breaker, load shedding |
| `07_escalabilidade_twitter.md` | Twitter: monolito → microsserviços, fanout, sharding, filas assíncronas |
| `08_concorrencia_e_protocolo.md` | WhatsApp/Erlang: modelos de concorrência, Fun XMPP, processos leves, Signal Protocol |
| `09_arquitetura_nubank.md` | Nubank: Clojure, Datomic, Kafka, shards como unidades de escala |

## Módulo 2 — System Design em Entrevistas

| Arquivo | Conteúdo |
|---|---|
| `10_framework_system_design.md` | Roteiro completo: requisitos, entidades, API, high-level design, deep dives |
| `11_design_ticket_master.md` | Ticket Master: lock distribuído (Redis TTL), Elasticsearch, virtual waiting queue |
| `12_design_uber.md` | Uber: geohash vs QuadTree, Redis geo, distributed lock, dynamic location updates |
| `13_design_dropbox.md` | Dropbox: pre-signed URLs, chunking resumável, fingerprinting, sync, delta sync |
| `14_design_ad_click_aggregator.md` | Ad Click Aggregator: Flink, stream processing, Lambda/Kappa, reconciliação |
| `15_design_web_crawler.md` | WebCrawler: SQS retries, robots.txt, bloom filter, deduplicação, crawler traps |
| `17_design_tinder.md` | Tinder: geospatial stack, pre-computation, swipe consistency, Cassandra writes |

## Módulo 3 — Deep Dives em Tecnologias

| Arquivo | Conteúdo |
|---|---|
| `16_redis_deep_dive.md` | Redis: cache, rate limiting, sorted sets, pub/sub, geo index, streams, sharding |
| `18_elasticsearch_deep_dive.md` | Elasticsearch: índice invertido, Lucene, shards, relevance, paginação, CDC sync |

## Módulo 4 — Carreira e Desenvolvimento Profissional

| Arquivo | Conteúdo |
|---|---|
| `19_carreira_engenharia.md` | AI e o futuro da profissão, hierarquia semântica, comunicação, como crescer |

---

## Módulo 5 — Conceitos Avançados de System Design (Hello Interview Series)

> Novos arquivos baseados em deep dives e vídeos técnicos da série Hello Interview

| Arquivo | Conteúdo |
|---|---|
| `20_cap_theorem_consistencia.md` | CAP Theorem: C vs A, espectro de consistência, decisão por domínio (Ticket Master, Tinder, Netflix) |
| `21_design_whatsapp_messenger.md` | WhatsApp: WebSockets, pre-signed URLs para mídia, Redis Pub/Sub, escala para bilhões |
| `22_design_fb_live_comments.md` | Facebook Live Comments: SSE vs WebSockets, Pub/Sub, particionamento, Cassandra |
| `23_dynamodb_deep_dive.md` | DynamoDB: Partition/Sort Key, GSI, LSI, Transactions, DAX, DynamoDB Streams |
| `24_sistemas_recomendacao.md` | Recommendation Systems: Candidate Generation, Ranking, Re-ranking, Vector DBs, HNSW |
| `25_indexacao_banco_dados.md` | Database Indexing: B-Tree, Hash Index, Geospatial (Geohash, R-Tree), Inverted Index |
| `26_estruturas_dados_big_data.md` | Big Data Structures: Bloom Filter, Count-Min Sketch, HyperLogLog — quando e por que |
| `27_consistent_hashing.md` | Consistent Hashing: problema do módulo, hash ring, virtual nodes, redistribuição mínima |
| `28_networking_essentials.md` | Networking: OSI, IP, TCP/UDP, HTTP/REST, gRPC, GraphQL, SSE, WebSockets, Load Balancing, Circuit Breaker |
| `29_behavioral_interviews.md` | Behavioral Interviews: CARL framework, Brag Document, sinais por nível, armadilhas comuns |

---

## Mapa de Conceitos Transversais

### Distribuição e Escala
```
Consistent Hashing (27) → DynamoDB internals (23) → Redis Cluster (16)
                       → Chat servers stateful routing (21)
```

### Consistência e Disponibilidade
```
CAP Theorem (20) → Amazon Dynamo (03) → DynamoDB strong reads (23)
               → Chat delivery guarantees (21) → Live comments eventual (22)
```

### Busca e Indexação
```
B-Tree (25) → Elasticsearch (18) → Inverted Index (25)
Geohash (25) → Uber geospatial (12) → Redis geo (16)
Vector DBs (24) → Recommendation Systems (24)
```

### Real-Time e Streaming
```
WebSockets (21, 28) vs SSE (22, 28)
Kafka (04) → Ad Click Aggregator (14) → Live Comments Pub/Sub (22)
```

### Eficiência de Memória (Big Data)
```
Bloom Filter (26) → WebCrawler deduplication (15)
Count-Min Sketch (26) → Rate Limiting (02) → Heavy Hitters
HyperLogLog (26) → DAU/MAU analytics
```

---

*Última atualização: Módulo 5 adicionado com 10 novos arquivos de conceitos avançados*
