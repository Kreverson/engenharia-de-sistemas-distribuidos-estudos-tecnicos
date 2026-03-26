# 47 — Bancos de Dados de Séries Temporais

> Por que bancos relacionais falham com métricas de infraestrutura, e como os time series databases (TSDB) resolvem isso com técnicas de compressão, indexação e armazenamento sequencial.

---

## 1. O Problema — Por que PostgreSQL Não Serve

### Cenário: Monitoramento de infraestrutura

```
100.000 servidores
5 métricas por servidor (CPU, memória, disco, I/O, rede)
Coleta a cada 10 segundos

Writes por segundo = 100.000 × 5 / 10 = 50.000 writes/s

PostgreSQL max: ~10.000 writes/s
→ Impossível com banco relacional simples
```

### O schema "óbvio" e seus problemas

```sql
CREATE TABLE metrics (
  timestamp   TIMESTAMP,
  host        VARCHAR(100),
  metric_name VARCHAR(50),
  value       DOUBLE
);
```

**Problema 1 — Write volume:**
```
50.000 writes/s → impossível sem sharding massivo
```

**Problema 2 — Eficiência de armazenamento:**
```
Por registro:
  timestamp: 8 bytes
  host:      "server-prod-us-east-42.company.com" = 40 bytes
  metric:    "cpu_utilization" = 15 bytes
  value:     8 bytes
Total: ~71 bytes por registro

Por hora: 50.000 × 3600 = 180M registros × 71 bytes = ~12.7 GB/hora
Por dia: ~305 GB
Por mês: ~9 TB → crescimento insustentável
```

**Problema 3 — Query lenta:**
```sql
SELECT AVG(value) FROM metrics
WHERE host = 'server-42' AND metric_name = 'cpu'
AND timestamp >= NOW() - INTERVAL '8 hours';

→ Varre bilhões de linhas para responder uma pergunta simples
→ Minutos de latência para query que deveria ser sub-segundo
```

---

## 2. A Solução: Time Series Databases

### Princípios fundamentais

1. **Append-only writes** → sequential I/O, sem random seeks
2. **Compressão agressiva** → dados de séries temporais são altamente compressíveis
3. **Indexação temporal** → particionamento automático por tempo
4. **Query engine especializado** → otimizado para agregações temporais

---

## 3. Técnica 1: Append-Only Storage e LSM-Trees

### Por que random I/O é o inimigo

```
HDD spinning disk:
  Write sequencial: 100-200 MB/s
  Write aleatório:  100-200 ops/s (arm movement overhead)
  → 1000x mais lento para random writes!

SSD:
  Ainda tem penalidade para random writes (wear leveling, block erasure)
  Random write = 10-100x mais lento que sequential
```

### LSM-Tree (Log-Structured Merge-Tree)

Estrutura de dados que garante sequential I/O:

```
1. Toda escrita vai primeiro para o WRITE-AHEAD LOG (WAL)
   → Append ao fim do log = sequential I/O ✅
   → Garante durabilidade (crash recovery)

2. Dado também vai para MEMTABLE (RAM, ordenado)
   → Staging area para organização em memória

3. Quando MEMTABLE enche → flush para SSTABLE (disco)
   → SSTable é imutável e ordenada
   → Sequential write ✅

4. Periodicamente: COMPACTION
   → Merge de múltiplas SSTables → 1 SSTable nova
   → Remove versões antigas, mantém apenas mais recente
   → Processo offline, não no caminho crítico
```

```
Fluxo visual:

Write → [WAL] → [Memtable (RAM)] → (flush periódico) → [SSTable 1]
                                                         [SSTable 2]
                                                         [SSTable 3]
                                              (compaction)↓
                                                    [SSTable Merged]
```

**Resultado:** writes são sempre appends sequenciais → 10-100x mais throughput que B-Tree.

---

## 4. Técnica 2: Delta Encoding e Compressão

### O insight sobre séries temporais

Dados de métricas raramente mudam drasticamente entre observações:
```
CPU usage: 45.2%, 45.3%, 45.1%, 45.4%, 45.2%, ...
                                                 → Diferenças pequenas!
```

### Delta Encoding

Em vez de armazenar valores absolutos, armazena as **diferenças**:

```
Valores originais:   56.7, 58.4, 57.9, 59.1, 58.8
Deltas (diferenças): 56.7, +1.7, -0.5, +1.2, -0.3
```

**Por que isso importa para armazenamento:**
```
Valor absoluto: 56.7 → double = 8 bytes
Delta pequeno:  +1.7 → pode ser representado em 2 bytes (signed int16)

Redução: 8 bytes → 2 bytes = 75% de economia!
```

### Delta of Deltas (para timestamps)

Timestamps com intervalo fixo de 10s:

```
Timestamps:          1700000000, 1700000010, 1700000020, 1700000030
Deltas:              -,          10,          10,          10
Delta de deltas:     -,          10,           0,           0

Depois de aplicar delta de deltas:
→ Sequência de zeros é comprimida trivialmente!
→ Facebook Gorilla paper: timestamps representados em MÉDIA com 1 bit
```

### XOR Encoding (para valores float)

```
value_1: 45.23 (IEEE 754 float: 01000010001101...)
value_2: 45.31 (IEEE 754 float: 01000010001101...)
XOR:              00000000000000...   ← maioria zeros!

Zeros são trivialmente compressíveis → 70-90% de redução
```

**Resultado combinado:** Gorilla paper (Facebook) reportou compressão de 12 bytes/valor para 1.37 bytes/valor em média — redução de ~9x.

---

## 5. Técnica 3: Block-Level Metadata e Query Planning

### O desafio das queries temporais

```sql
SELECT mean(cpu) FROM metrics
WHERE host = 'server-42' AND time > NOW() - 8h
```

Sem otimização: varre todos os dados históricos.

### Time-Based Partitioning

```
Dados organizados em blocos por janela temporal:

[Block: 2025-01-01 00:00 - 08:00] → só contém dados desse período
[Block: 2025-01-01 08:00 - 16:00] → só contém dados desse período
[Block: 2025-01-01 16:00 - 24:00] → só contém dados desse período

Query para últimas 8 horas:
→ Apenas 1 bloco relevante
→ 2/3 dos blocos são ignorados sem nem abrir os arquivos!
```

### Metadata por bloco

Cada bloco armazena metadados além dos dados:

```
Block header: {
  time_range:  [2025-01-01 00:00, 2025-01-01 08:00],
  hosts:       ["server-01", "server-02", "server-42"],  ← ou bloom filter
  metrics:     ["cpu", "memory", "disk"],
  values: {
    cpu: { min: 12.3, max: 89.7 },      ← para filtros de range
    memory: { min: 40.1, max: 92.3 }
  }
}
```

**Query planning com metadata:**
```
Query: "CPU > 80% em qualquer servidor ontem"

Para cada bloco do dia de ontem:
  Se max(cpu) em metadata ≤ 80%:
    → Bloco não pode ter registros que atendam a condição
    → SKIP sem abrir o bloco!

Resultado: talvez 80% dos blocos sejam ignorados
→ Query 5x mais rápida
```

### Bloom Filters para membership queries

```
Query: "Mostrar métricas de server-42"

Para cada bloco:
  bloom_filter.contains("server-42")?
    → "Definitivamente NÃO" → skip o bloco (zero false negatives)
    → "Talvez" → abrir o bloco e confirmar

Bloom filter: 10-100x menos espaço que um set real
→ Permite descartar blocos sem I/O
```

---

## 6. Técnica 4: Downsampling

```
Dados coletados a cada 10 segundos (granularidade fina)
Mas usuário consultando gráfico de 1 ANO não precisa de granularidade de 10s!

Política de downsampling:
  Dado < 1 semana:  granularidade de 10 segundos (alta precisão)
  Dado < 1 mês:     granularidade de 1 minuto (60x menos dados)
  Dado < 1 ano:     granularidade de 1 hora (360x menos dados)
  Dado > 1 ano:     granularidade de 1 dia (8640x menos dados)

Implementação: background job que periodicamente:
  1. Lê dados de alta granularidade
  2. Agrega (média, max, min, percentil)
  3. Escreve dados de baixa granularidade
  4. Deleta dados originais de alta granularidade
```

**Resultado prático:** gráfico de 1 ano carrega em <100ms em vez de minutos.

---

## 7. Arquitetura Interna de um TSDB

```
Write Path:
  Dado chega → Write-Ahead Log (durabilidade) → Memtable (RAM)
                                                    ↓ (quando cheio)
                                              SSTable (disco)
                                              (imutável, comprimido)

Read Path:
  Query chega → Query Planner
              → Identifica blocos relevantes por time range
              → Verifica bloom filters e min/max metadata
              → Lê apenas blocos necessários
              → Decomprime dados (delta decoding, XOR decoding)
              → Lê de Memtable (dados recentes não flushed)
              → Agrega e retorna resultado

Background:
  Compaction: merge de SSTables antigas
  Downsampling: reduce granularidade de dados antigos
  Retention: deleta dados além do retention period
```

---

## 8. Limitações: Alta Cardinalidade

### O problema

```
Tags de uma métrica:
  { host: "server-42", region: "us-east", env: "prod" }

Para likes do Instagram:
  { user_id: "alice", post_id: "post_123" }
```

Likes têm cardinalidade **bilhões de combinações** únicas de (user_id, post_id).

**Por que isso quebra TSDBs:**

```
TSDB organiza dados por "série" = combinação única de tags
Série 1: host=server-42, env=prod, metric=cpu
Série 2: host=server-43, env=prod, metric=cpu
...

Se user_id e post_id são tags:
Série para cada combinação única = bilhões de séries
→ Memtable explodiria
→ Index de séries ficaria gigantesco
→ Queries cruzando muitas séries seriam impossíveis
```

**Regra:** TSDBs funcionam bem quando o número de séries ativas é **milhares a dezenas de milhares**, não bilhões.

---

## 9. Principais TSDBs e Quando Usar

### InfluxDB

**Ideal para:** infraestrutura, IoT, métricas de aplicação.

```
Flux query language:
from(bucket: "metrics")
  |> range(start: -1h)
  |> filter(fn: (r) => r.host == "server-42")
  |> mean()
```

**Limitação:** cross-series queries são problemáticas (Top-K de 1M séries não é o use case).

### Prometheus

**Ideal para:** observabilidade de sistemas distribuídos (o padrão Kubernetes/cloud-native).

```promql
# P99 de latência de uma API nos últimos 5 minutos
histogram_quantile(0.99, 
  rate(http_request_duration_bucket[5m]))
```

**Integração:** exporta para Grafana, alertas via AlertManager.

### TimescaleDB (extensão PostgreSQL)

**Ideal para:** quando você quer SQL completo + performance de TSDB.

```sql
-- "Hypertable": tabela automáticamente particionada por tempo
SELECT time_bucket('1 hour', time) as hour,
       device_id, avg(temperature) as avg_temp
FROM sensor_data
WHERE time > NOW() - INTERVAL '24 hours'
GROUP BY 1, 2;
```

**Unique advantage:** é PostgreSQL. JOINs com outras tabelas funcionam normalmente. Perfect migration path.

### ClickHouse / Apache Druid

**Ideal para:** analytics em tempo real sobre grandes volumes.

```sql
-- ClickHouse: query sub-segundo sobre bilhões de linhas
SELECT toHour(timestamp) as hour, avg(response_time)
FROM http_logs
WHERE date = today()
GROUP BY hour;
```

---

## 10. Quando Usar TSDB em Entrevistas

### Use TSDB quando:

```
✅ Métricas de infraestrutura: CPU, memória, rede por servidor
✅ IoT: leituras de sensores com alta frequência
✅ Analytics de logs: requests/s, error rates, latência
✅ Dados financeiros: preços de ações em séries
✅ Monitoramento de aplicação (APM): traces, spans
```

### NÃO use TSDB quando:

```
❌ Alta cardinalidade (bilhões de séries únicas): likes, views por post
❌ Queries analíticas complexas com JOINs
❌ Dados sem dimensão temporal como protagonista
❌ Você precisa de ACID transacional
```

---

## Resumo dos Conceitos

| Técnica | Problema que Resolve | Ganho |
|---|---|---|
| LSM-Tree + Append-Only | Write throughput limitado | 10-100x mais writes/s |
| Delta Encoding | Armazenamento ineficiente | 75-90% menos espaço |
| Delta of Deltas | Timestamps grandes | 1 bit por timestamp em média |
| Time Partitioning | Queries varrendo todos os dados | Ignora 90%+ dos blocos |
| Block Metadata | Filtros sem abrir dados | Queries 5-10x mais rápidas |
| Bloom Filters | Membership queries caras | 100x menos I/O |
| Downsampling | Storage crescendo infinitamente | 360-8640x menos dados históricos |

---

*Tópicos relacionados: `26_estruturas_dados_big_data.md`, `04_event_streaming_kafka.md`, `14_design_ad_click_aggregator.md`, `43_top_k_system.md`*
