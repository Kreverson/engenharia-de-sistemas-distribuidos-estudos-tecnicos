# 43 — Design de Sistema Top-K (YouTube/Spotify)

> Problema clássico de data engineering: calcular os K itens mais populares em tempo real sobre grandes volumes de dados. Aborda stream processing, agregação, caching e estruturas de dados probabilísticas.

---

## 1. O Problema e suas Variantes

**Enunciado típico:** "Projete um sistema que mostre os K vídeos mais assistidos no YouTube."

**Por que é difícil:**
- 70 bilhões de views por dia (≈ 700K views/segundo)
- Janelas de tempo: all-time, último mês, última semana, última hora
- Latência de leitura: dezenas de milissegundos
- O enunciado simples esconde decisões arquiteturais profundas

### Clarificando requisitos antes de projetar

| Questão | Impacto no design |
|---|---|
| Janela deslizante ou tumbling? | Sliding = muito mais complexo |
| Precisão exata ou aproximada? | Aproximado = Count-Min Sketch, muito mais eficiente |
| Qual o K máximo? | K=1000: sem paginação necessária (~KB de dados) |
| Latência de escrita aceitável? | 1 minuto de lag é aceitável? Define cache TTL |

### Tumbling Window vs Sliding Window

```
Tumbling (semana = Seg-Dom):
├── [Seg 00:00 - Dom 23:59] = janela fixa, reseta todo início de semana

Sliding (últimos 7 dias):
├── Agora: Sab 15:00 → janela = [Sab-anterior 15:00 - Sab-atual 15:00]
├── 1 hora depois → [Sab-anterior 16:00 - Sab-atual 16:00]
└── A janela se move continuamente
```

**Regra em entrevistas:** Se não especificado, proponha tumbling. É muito mais simples e na maioria dos casos "mês passado" para usuários finais significa janela fixa.

---

## 2. High-Level Design

```
[YouTube Services]
      │
      ▼ eventos de view
[Kafka Topic: video-views]
  particionado por video_id
      │
      ▼ consumers
[View Consumer / Flink Job]
      │ agrega por janela de tempo
      ▼
[Views Database (PostgreSQL)]
  video_views_hourly
  video_views_daily
  video_views_monthly
      │
[Top-K Cron]          [Top-K Service]
(roda periodicamente) ←── REST API
      │                         │
      ▼                         ▼
[Redis Cache]          [Response ao cliente]
```

---

## 3. Escalando as Writes

### Cálculo de capacidade

```
70B views/dia ÷ 86.400 segundos = ~810K views/segundo (média)
Pico (Black Friday, evento ao vivo): 5x = ~4M views/segundo

PostgreSQL max: ~10K writes/s (com índices)
Shards necessários: 810K / 10K = 81 shards no mínimo
```

### Por que 81 shards é problemático

Cada view geraria 1 write ao banco:
```
Opção ruim:
810K views/s → 810K INSERTs/s → 81 shards → muito hardware
```

### Solução: Pré-agregação com Flink

Em vez de 1 write por view, agregar na janela temporal antes de persistir:

```
Flink lê do Kafka → agrupa por (video_id, janela_1h) → soma views
→ 1 write por (video_id, hora) quando a janela fecha

Redução:
Antes: 810K writes/s
Depois: ~1M vídeos únicos × 1 write/hora = ~277 writes/s ← 3000x menos!
```

### Flink — checkpointing automático

**Problema de escrita in-memory sem Flink:**
```
View Consumer guarda agregados em RAM
Crash após 45 minutos → perde 45min de dados
Reprocessar do Kafka: precisa consumir 45min × 810K = 2.2B eventos
→ Com capacity normal, leva 45min para "recuperar" → sempre atrasado
```

**Solução Flink:**
```
Flink faz checkpoint a cada 5 minutos → estado persistido no S3
Se crash: retoma do último checkpoint → máx. 5min de reprocessamento
```

---

## 4. Otimização das Reads

### Problema: queries SQL lentas

```sql
-- Query para top-K do último mês
SELECT video_id, SUM(views) as total
FROM video_views_hourly
WHERE created_at >= '2025-01-01' AND created_at < '2025-02-01'
GROUP BY video_id
ORDER BY total DESC
LIMIT 1000;
```

Para 1 mês de dados:
```
Linhas a ler: 30 dias × 24h × 1M vídeos únicos/hora = 720M linhas
Resultado: query levando minutos → viola nosso SLA de dezenas de ms
```

### Solução 1: Tabelas pré-agregadas por granularidade

Em vez de só gravar `video_views_hourly`, Flink grava em múltiplas tabelas:

```
video_views_hourly  → timestamp por hora
video_views_daily   → total do dia (soma das horas)
video_views_monthly → total do mês (soma dos dias)
```

Agora a query mensal lê de `video_views_monthly`:
```sql
SELECT video_id, views
FROM video_views_monthly
WHERE month = '2025-01'
ORDER BY views DESC
LIMIT 1000;
```

Com índice em `(month, views DESC)` → query em O(log N + K) → **milissegundos**.

### Solução 2: Cron de pré-computação + Redis Cache

```
┌─────────────────────────────────────────────────────┐
│  Top-K Cron                                         │
│  - Roda a cada hora (para janela horária)           │
│  - Roda diariamente (para janela diária)            │
│  - Query no PostgreSQL → resultado → armazena Redis │
└─────────────────────────────────────────────────────┘

Redis key: "top_k:monthly:2025-01"
Value: [{"video_id": "abc", "views": 5000000}, ...]
TTL: 1 hora (ou até o próximo cron)
```

**Latência de leitura:** O(1) no Redis → single-digit milliseconds ✅

### Solução 3: Índice na tabela de views

```sql
-- Permite query eficiente direto na tabela mensal
CREATE INDEX idx_views_monthly ON video_views_monthly(month, views DESC);
```

Com esse índice, o `ORDER BY views DESC LIMIT 1000` usa um index scan, não um sort — muito mais rápido.

---

## 5. Sliding Windows — A Complexidade Real

### Por que é mais difícil

```
Tumbling window (hora):
→ 1 janela ativa por hora
→ 24 janelas por dia

Sliding window (última hora, atualizada a cada minuto):
→ 60 janelas ativas simultâneas
→ 1440 janelas por dia (24 × 60)
→ Para uma janela mensal: 30 × 24 × 60 = 43.200 janelas!
```

### Implementação eficiente

Em vez de armazenar dados por janela, use matemática incremental:

```
Valor da janela [t-60min, t] = Valor[t-59min, t] + views_do_minuto_t - views_do_minuto_(t-60)

                 janela anterior    + novo dado         - dado que saiu
```

**Schema para sliding windows:**

```sql
-- Granularidade minutal (necessária para sliding window de 1h)
CREATE TABLE video_views_minute (
  video_id   UUID,
  minute_ts  TIMESTAMP,
  views      INT,
  PRIMARY KEY (video_id, minute_ts)
);

-- Janelas deslizantes pré-computadas (atualizadas a cada minuto)
CREATE TABLE video_views_sliding_hour (
  video_id UUID PRIMARY KEY,
  views    BIGINT  -- total da última hora (deslizante)
);
CREATE INDEX ON video_views_sliding_hour(views DESC);
```

**Update incremental (a cada minuto):**
```sql
UPDATE video_views_sliding_hour
SET views = views + new_minute_views - expired_minute_views
WHERE video_id = ?;
```

---

## 6. Count-Min Sketch — Solução Aproximada em Memória

Quando precisão exata não é necessária (e raramente é para trending), estruturas probabilísticas permitem escala radical.

### O que é

Um Count-Min Sketch é como uma tabela hash 2D onde:
- Linhas = funções hash diferentes (4-8 linhas)
- Colunas = índices de hash (largura w)
- Células = contadores

```
     Col: 0   1   2   3   4   5   6   7
Hash1:  [  0,  0,  5,  0,  2,  0,  0,  3 ]
Hash2:  [  0,  5,  0,  2,  0,  0,  3,  0 ]
Hash3:  [  2,  0,  0,  5,  0,  3,  0,  0 ]
Hash4:  [  0,  3,  0,  0,  5,  0,  2,  0 ]
```

**Inserir "video_abc" com 5 views:**
```
hash1("video_abc") = 2 → incrementa CMS[0][2] em 5
hash2("video_abc") = 1 → incrementa CMS[1][1] em 5
hash3("video_abc") = 3 → incrementa CMS[2][3] em 5
hash4("video_abc") = 4 → incrementa CMS[3][4] em 5
```

**Consultar "video_abc":**
```
Lê CMS[0][2], CMS[1][1], CMS[2][3], CMS[3][4]
Retorna o MÍNIMO (colisões só inflam, nunca deflam)
→ Resultado: upper bound do count real
```

### Propriedades

- **Espaço:** O(w × d) ao invés de O(elementos únicos)
- **Erro:** controlado pela largura e profundidade do sketch
- **Falsos negativos:** impossíveis (subestimação não ocorre)
- **Falsos positivos:** possíveis (superestimação por colisão)

### Top-K com Count-Min Sketch + Min-Heap

```
Para cada view:
  1. Incrementar CMS para o video_id
  2. Consultar CMS para o count aproximado desse video
  3. Se count > mínimo do heap de Top-K:
     → Substituir o elemento mínimo do heap por este video
  
Heap mantém apenas K elementos
→ Estrutura total: O(w×d) + O(K) ← muito pequena!
```

**Redis suporta nativamante:**
```
# Incrementar
CMS.INCRBY topk_sketch video_abc 1

# Consultar
CMS.QUERY topk_sketch video_abc

# Sorted Set para Top-K
ZADD topk_videos [score] video_abc
```

---

## 7. Bancos de Dados Especializados — Quando e se usar

### TimescaleDB (extensão do Postgres)

**Quando usar:** métricas de séries temporais com SQL familiar.
```sql
-- Hypertable: particiona automaticamente por tempo
SELECT time_bucket('1 hour', created_at) AS hour, 
       video_id, SUM(views)
FROM video_views
GROUP BY 1, 2
ORDER BY total DESC;
```

**Continuous aggregates:** materializa automaticamente agregações para leituras rápidas.

### InfluxDB / Prometheus

Otimizados para uma série por vez (CPU de um servidor, requests de uma API). Problemático para Top-K porque:
- "Top K de 1M séries" requer leitura de 1M séries separadas
- Não há suporte nativo para cross-series aggregation eficiente

### Apache Druid / ClickHouse / Pinot

Real-time OLAP databases. Combinam:
- Ingestão de stream (como Kafka)
- Views materializadas automáticas
- Queries SQL rápidas sobre dados agregados

**Prós:** eliminam Flink + PostgreSQL em uma solução só  
**Cons:** operacionalmente complexos, entrevistadores podem não conhecer

---

## 8. Arquitetura Final Completa

```
[YouTube App]
     │ view events
     ▼
[Kafka: video-views]
  particionado por video_id
     │
     ▼ consumers em paralelo
[Flink Cluster]
  ├── tumbling windows (1h, 1d, 1mo)
  ├── checkpointing automático → S3
  └── escreve agregados → PostgreSQL
           │
     ┌─────┴──────────────────────┐
     │ video_views_hourly          │
     │ video_views_daily           │
     │ video_views_monthly         │
     │ INDEX(month, views DESC)    │
     └─────────────────────────────┘
           │
     [Top-K Cron]    ← roda a cada hora
           │ escreve resultados
           ▼
     [Redis Cache]
       top_k:hourly:2025-01-01-15
       top_k:daily:2025-01-01
       top_k:monthly:2025-01
       TTL: próxima execução do cron
           │
     [Top-K Service] ← lê do Redis O(1)
           │
     [API Gateway]
           │
     [Clientes]      ← dezenas de ms
```

---

## 9. Resumo dos Conceitos-Chave

| Conceito | Aplicação |
|---|---|
| **Tumbling vs Sliding Windows** | Sliding = 60x mais janelas, use tumbling quando possível |
| **Pré-agregação (Flink)** | Reduz writes de 810K/s para ~300/s |
| **Multi-granularidade** | Escreve hourly + daily + monthly para queries eficientes |
| **Índice em views DESC** | O(log N + K) ao invés de sort completo |
| **Cron + Cache Redis** | Pré-computa resultado, read O(1) |
| **Count-Min Sketch** | Aproximação com memória O(constante) |
| **Checkpointing Flink** | Recovery de até 5min de lag após falha |

---

*Tópicos relacionados: `04_event_streaming_kafka.md`, `14_design_ad_click_aggregator.md`, `26_estruturas_dados_big_data.md`, `16_redis_deep_dive.md`*
