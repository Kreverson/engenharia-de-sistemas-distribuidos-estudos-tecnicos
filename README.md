# 📚 Engenharia de Sistemas Distribuídos — Estudos Técnicos

> Compilação de conceitos, padrões e experiências reais extraídas de casos como Knight Capital, Pix, Nubank, Slack, Instagram, Amazon Dynamo, LinkedIn/Kafka, Twitter e WhatsApp.

---

## Arquivos deste estudo

| Arquivo | Conteúdo |
|---|---|
| `01_falhas_e_divida_tecnica.md` | Knight Capital: deploy manual, dead code, flags reutilizadas, circuit breaker |
| `02_rate_limiting_e_seguranca.md` | Pix/DICT: Token Bucket, Weighted Rate Limiting, Race Condition, Redis atômico |
| `03_consistencia_e_disponibilidade.md` | Amazon Dynamo: Teorema CAP, Consistent Hashing, Vector Clocks, eventual consistency |
| `04_event_streaming.md` | Kafka/LinkedIn: log distribuído, tópicos, partições, page cache, sendfile |
| `05_ids_distribuidos.md` | Instagram: Snowflake IDs, bit shifting, geração sem coordenação central |
| `06_thundering_herd.md` | Slack: Thunder Herd, backoff exponencial com jitter, circuit breaker, load shedding |
| `07_escalabilidade_twitter.md` | Twitter: monolito → microsserviços, fanout, sharding, filas assíncronas |
| `08_concorrencia_e_protocolo.md` | WhatsApp/servidores: modelos de concorrência, Erlang, Fun XMPP, processos leves |
| `09_arquitetura_nubank.md` | Nubank: Clojure, Datomic, Kafka, shards como unidades de escala |

---

## Mapa Mental dos Grandes Temas

```
Sistemas Distribuídos
├── Confiabilidade
│   ├── Circuit Breaker
│   ├── Idempotência
│   ├── Thunder Herd / Backoff com Jitter
│   └── Load Shedding
├── Escala
│   ├── Sharding (lógico vs físico)
│   ├── Consistent Hashing
│   ├── Fanout on Write vs Read
│   └── Scalability Units (Nubank)
├── Dados
│   ├── Imutabilidade (Clojure + Datomic)
│   ├── Event Sourcing (Kafka)
│   ├── Teorema CAP
│   └── Vector Clocks
├── Performance
│   ├── Page Cache (Kafka)
│   ├── Sendfile (zero-copy)
│   ├── IDs sem coordenação (Instagram)
│   └── Processos leves (Erlang)
└── Segurança
    ├── Weighted Rate Limiting (DICT)
    ├── Atomicidade Distribuída (Redis + Lua)
    └── End-to-end Encryption (WhatsApp Signal Protocol)
```
