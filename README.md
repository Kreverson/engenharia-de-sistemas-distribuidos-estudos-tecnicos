# 📚 Engenharia de Sistemas Distribuídos — Estudos Técnicos

> Compilação de conceitos, padrões e experiências reais extraídas de casos como Knight Capital, Pix, Nubank, Slack, Instagram, Amazon Dynamo, LinkedIn/Kafka, Twitter, WhatsApp, Uber, Ticket Master, Tinder, Dropbox, WebCrawler, YouTube, Facebook e mais.

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

## Módulo 5 — Conceitos Avançados de System Design

> Arquivos baseados em deep dives e vídeos técnicos da série Hello Interview

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

## Módulo 6 — System Design Aplicado: Vídeo, Busca e Tempo Real

> Breakdowns completos de problemas clássicos com análise profunda de componentes

| Arquivo | Conteúdo |
|---|---|
| `30_design_youtube_video_streaming.md` | YouTube: multipart upload, pre-signed URLs, chunking, transcoding, HLS/DASH, CDN, ABR |
| `31_api_gateway.md` | API Gateway: história, componentes internos, middleware, roteamento, transformação de protocolo |
| `32_object_storage_blob.md` | Object Storage (S3): flat namespace, immutable writes, pre-signed URLs, multipart upload, 11 noves |
| `33_design_newsfeed_fanout.md` | Newsfeed (Facebook): fan-out on write/read, pre-computação, celebrity problem, hot key com múltiplas réplicas |
| `34_design_facebook_post_search.md` | Facebook Post Search: inverted index do zero, Redis Sorted Sets, Kafka, escala logarítmica, Count-Min Sketch, bigramas |
| `35_design_yelp_geoespacial.md` | Yelp: PostGIS, busca geoespacial, full-text search, constraint de unicidade, média incremental em transação |
| `36_design_instagram_auction.md` | Instagram Auction: SSE vs WebSockets, Redis Pub/Sub, SERIALIZABLE isolation, SQS delay, Temporal IO |
| `37_ml_system_design_content_moderation.md` | ML System Design: framework 6 etapas, transformer multimodal, multi-task learning, two-stage inference, calibração |
| `38_aprendizado_carreira_engenharia.md` | Como aprender system design, BFS vs DFS, entrevistas de manager, AI no futuro da engenharia |

---

## Módulo 7 — Estudos Técnicos Aprofundados

> Resumos consultivos com enriquecimento técnico de transcrições de vídeos da série Hello Interview. Foco em conceitos aplicados a entrevistas e prática profissional.

| Arquivo | Conteúdo |
|---|---|
| `39_api_design.md` | REST (recursos, verbos, paginação, autenticação), GraphQL (N+1, autorização por campo), gRPC/protobuf, JWT vs Session, segurança em APIs |
| `40_rate_limiter.md` | Token Bucket, Fixed/Sliding Window, Redis Lua Script, race condition, Redis Cluster, fail-open vs fail-closed, configuração dinâmica ZooKeeper |
| `41_data_modeling.md` | Tipos de banco (SQL/NoSQL/Wide-column), PKs e FKs, normalização vs desnormalização, indexação B-Tree, sharding, roteiro completo para entrevistas |
| `42_cassandra.md` | Wide column store, Partition Key + Clustering Key, LSM-Tree, Consistent Hashing, Gossip Protocol, Tunable Consistency, query-driven design, write/read path |
| `43_top_k_system.md` | Tumbling vs sliding windows, pré-agregação com Flink, multi-granularidade, Count-Min Sketch, min-heap para Top-K, TimescaleDB vs InfluxDB |
| `44_behavioral_interviews.md` | Framework CARL, escalas de escopo por nível, conflito com colega/gerente, projeto com falha, risco calculado, brag document, sinais de nível |
| `45_sharding.md` | Range-based vs Hash-based vs Consistent Hashing vs Directory-based, hotspots/celebrities, cross-shard queries, Saga Pattern vs 2PC, quando NÃO shardar |
| `46_caching.md` | Cache-aside vs write-through vs write-behind, LRU/LFU/TTL, Cache Stampede + soluções, cache inconsistency, hot keys, Redis além do cache |
| `47_time_series_databases.md` | LSM-Tree, delta encoding, delta of deltas, XOR encoding, time-based partitioning, bloom filters, downsampling, cardinalidade alta como limitação |
| `48_low_level_design.md` | Framework 5 etapas, SRP, enum vs booleans, Connect 4 completo, Amazon Locker completo, anti-padrões, extensibilidade |
| `49_ml_bot_detection.md` | Problem framing adversarial, late fusion architecture, GraphSAGE, GRU, self-supervised pre-training, multi-task learning, calibração, two-stage inference, Isolation Forest, Autoencoder |
| `50_concurrency_queues.md` | Check-then-act, Read-modify-write, Lock vs Atomic, Blocking Queue, Semaphore, Connection Pool, RabbitMQ vs Kafka, offset, delivery guarantees, DLQ |

---

## Mapa de Conceitos Transversais

### Distribuição e Escala
```
Consistent Hashing (27) → DynamoDB internals (23) → Redis Cluster (16)
                       → Chat servers stateful routing (21)
                       → Facebook Post Search sharding (34)
```

### Consistência e Disponibilidade
```
CAP Theorem (20) → Amazon Dynamo (03) → DynamoDB strong reads (23)
               → Chat delivery guarantees (21) → Live comments eventual (22)
               → Auction bids SERIALIZABLE (36) → Newsfeed eventual (33)
```

### Busca e Indexação
```
B-Tree (25) → Elasticsearch (18) → Inverted Index (25, 34)
Geohash (25) → Uber geospatial (12) → Redis geo (16) → PostGIS (35)
Vector DBs (24) → Recommendation Systems (24)
Inverted Index custom (34) → vs Elasticsearch (18, 34)
```

### Real-Time e Streaming
```
WebSockets (21, 28) vs SSE (22, 28, 36)
Kafka (04) → Ad Click Aggregator (14) → Live Comments Pub/Sub (22)
          → Facebook Post Search write buffer (34)
          → Instagram Auction durability (36)
Redis Pub/Sub (16) → WhatsApp (21) → Auction live bids (36)
```

### Upload e Armazenamento de Mídia
```
Object Storage (32) → Pre-signed URLs (13, 30, 32)
Multipart Upload (30, 32) → Chunking de playback (30)
Transcoding (30) → HLS/DASH (30) → Adaptive Bitrate (30)
CDN (30) → Cache de search results (34)
```

### Eficiência de Memória (Big Data)
```
Bloom Filter (26) → WebCrawler deduplication (15)
Count-Min Sketch (26) → Rate Limiting (02) → Heavy Hitters → Post Search cold/hot (34)
HyperLogLog (26) → DAU/MAU analytics
```

### Fan-out e Propagação de Eventos
```
Newsfeed fan-out (33) → Celebrity problem (33)
Hot Key solution (33) → Hot Shard em DynamoDB (23)
Notification patterns → Push vs Pull
```

### ML e Sistemas Inteligentes
```
Recommendation Systems (24) → Feature engineering → Model serving
Content Moderation (37) → Multimodal ML → Multi-task learning
                        → Two-stage inference → Calibration
```

### Job Scheduling e Workflows
```
SQS delay messages (36) → Cron job evolution
Temporal IO (36) → Durable execution → Workflow orchestration
```

### Carreira e Entrevistas
```
Behavioral framework (29) → Manager interviews (38)
System Design framework (10) → Deep dives (30-37)
ML System Design framework (37) → 6 etapas
```
