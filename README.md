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
| `10_framework_system_design.md` | Roteiro completo: requisitos, entidades, API, highle design, deep dives |
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
