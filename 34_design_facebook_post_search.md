# 34 — Design Facebook Post Search: Índice Invertido do Zero

> Baseado no breakdown da Hello Interview com Stefan. Problema especial: **sem usar Elasticsearch** (restrição do entrevistador). Cobre inverted index, sorted sets no Redis, write throughput com Kafka, CDN para search, e Count-Min Sketch.

---

## 📌 Contexto do Problema

Design de busca de posts do Facebook:
- Busca por **keywords**
- Ordenação por **likes** ou **tempo de criação**
- Sem fuzzy search, sem personalização
- **Sem Elasticsearch** (restrição do entrevistador)

### Por que essa restrição é interessante?

Ela força o candidato a entender **como um search engine funciona internamente**, em vez de apenas nomear uma tecnologia. É uma pergunta frequente em Meta, Google e OpenAI.

---

## 📊 Estimativas de Escala

```
1 bilhão de usuários
1 post por usuário por dia → 10.000 posts/segundo
10 likes por usuário por dia → 100.000 likes/segundo

→ Sistema é WRITE-HEAVY (não read-heavy como parece intuitivamente)

Reads (buscas): 1 busca/usuário/dia → 10.000 buscas/segundo
Writes: 100.000 likes/segundo → 10x mais writes que reads!

Storage:
  3.6 trilhões de posts (10 anos × 1B/dia)
  × 1 KB por post = 3.6 Petabytes
```

**Insight importante:** Sistemas de busca parecem read-heavy, mas são dominados por writes de indexação.

---

## 🎯 Requisitos

### Funcionais
```
1. Criar posts
2. Curtir posts (likes)
3. Buscar posts por keyword
   → Ordenação: por likes OU por tempo de criação
```

### Não-Funcionais
```
- Busca: latência < 500ms
- Indexação: eventual consistency (1 minuto)
- Alta disponibilidade
- Alto recall (cobrir a maioria dos posts)
```

---

## ❌ Por que NOT usar LIKE em SQL?

```sql
-- Abordagem ingênua:
SELECT * FROM posts WHERE content LIKE '%taylor%';

Problemas:
1. Full table scan → percorre 3.6 trilhões de linhas
2. LIKE com % inicial não usa índice B-Tree
3. Impossível em < 500ms mesmo com hardware infinito
```

---

## 🔑 Conceito Central: Inverted Index (Índice Invertido)

### O que é?

Em vez de "dado um post, qual é seu conteúdo?", o inverted index responde: "dado uma palavra, quais posts a contêm?"

```
Posts:
  post_1: "Taylor Swift lança novo álbum"
  post_2: "Taylor é a melhor cantora"
  post_3: "Swift é também um pássaro"

Inverted Index:
  "taylor" → [post_1, post_2]
  "swift"  → [post_1, post_3]
  "lança"  → [post_1]
  "novo"   → [post_1]
  "álbum"  → [post_1]
  "melhor" → [post_2]
  "cantora"→ [post_2]
  "pássaro"→ [post_3]
```

**Busca por "taylor":**
```
1. Lookup "taylor" no índice → [post_1, post_2]
2. Retorna posts → latência O(1) no índice, não O(N posts)
```

---

## 🏛️ High-Level Design

### Componentes Principais

```
                ┌──────────────────────────────────────┐
                │              Clients                 │
                └────────────────┬─────────────────────┘
                                 │
                         ┌───────▼───────┐
                         │  API Gateway  │
                         └───┬───────┬───┘
                 ┌───────────┘       └───────────┐
                 ▼                               ▼
        ┌─────────────┐                 ┌─────────────┐
        │  Post Svc   │                 │  Like Svc   │
        └──────┬──────┘                 └──────┬──────┘
               │                               │
               ▼                               ▼
        ┌─────────────┐                 ┌─────────────┐
        │  Posts DB   │                 │  Likes DB   │
        └──────┬──────┘                 └──────┬──────┘
               │ CDC                           │ CDC
               └───────────┬───────────────────┘
                           ▼
                  ┌─────────────────┐
                  │Ingestion Service│ ← tokeniza, escreve no índice
                  └────────┬────────┘
                           ▼
                  ┌─────────────────┐
                  │  Search Index   │ ← Redis Sorted Sets
                  │  (Redis Cluster)│
                  └────────┬────────┘
                           │
                  ┌────────▼────────┐
                  │  Search Service │
                  └─────────────────┘
```

### Search Index: Redis Sorted Sets

```
Duas chaves por keyword (uma para cada tipo de ordenação):

likes:taylor  → Sorted Set:
  score=1000000, member=post_abc  (1M likes)
  score=250000,  member=post_def  (250K likes)
  score=10,      member=post_ghi  (10 likes)

time:taylor   → Sorted Set:
  score=1703001600, member=post_abc  (timestamp mais recente)
  score=1703001000, member=post_def
  score=1702999200, member=post_ghi
```

**Busca por "taylor" ordenado por likes:**
```
ZREVRANGE likes:taylor 0 24  → top 25 posts por likes
```

**Busca por "taylor" ordenado por tempo:**
```
ZREVRANGE time:taylor 0 24   → 25 posts mais recentes
```

Latência: microsegundos → muito dentro dos 500ms.

---

## 🚀 Deep Dives

### Deep Dive 1: Alta Leitura — Cache em Múltiplas Camadas

#### Problema
100K usuários buscando "taylor swift" ao mesmo tempo → bombardeia o Redis.

#### Solução: Cache em Camadas

```
Camada 1: Cache local em cada Search Service
  → Uma busca por "taylor" por instância por intervalo
  → N instâncias = no máximo N queries ao Redis

Camada 2: CDN com cache de search results
  → "taylor" não é personalizado → todo mundo recebe o mesmo resultado
  → CDN cacheia por até 1 minuto (igual ao SLA de indexação)
  → Usuário em Berlim recebe resultado de Frankfurt, não de US-West

  HTTP Response Headers:
    Cache-Control: max-age=60, public
```

```
Usuário (Brasil) → CDN edge (São Paulo)
  → cache HIT → resposta imediata
  → cache MISS → vai ao Search Service → Redis → popula cache CDN
```

---

### Deep Dive 2: Alto Write — Kafka como Buffer

#### Problema
Pico de 100K likes/segundo e 10K posts/segundo → Ingestion Service pode cair.

#### Solução: Kafka como Amortecedor

```
Posts DB  ──CDC──▶ Kafka topic: "posts"    ┐
Likes DB  ──CDC──▶ Kafka topic: "likes"    ┤──▶ Ingestion Service Pool
                                            ┘    (auto-scaling)
```

**Benefícios:**
- **Desacoplamento:** Post Service não depende do Ingestion Service estar ativo
- **Buffer de pico:** Kafka absorve picos, Ingestion processa na velocidade que consegue
- **Fault tolerance:** Se Ingestion cair, Kafka guarda os eventos → reprocessamento ao reiniciar
- **Reprocessamento:** Offset do Kafka permite replay desde qualquer ponto

---

### Deep Dive 3: Likes não têm valor igual — Logaritmo para Reduzir Writes

#### Problema
100K likes/segundo × N keywords por post = volume de writes absurdo no Redis.

**Insight crucial:** A maioria dos likes não muda a ordenação dos resultados.

```
posts para "taylor":
  post_A: 1.000.000 likes
  post_B: 100 likes
  post_C: 1 like
  post_D: 0 likes

Post_A recebe mais um like → vai de 1.000.000 para 1.000.001
→ Ordering não mudou!
→ Escrever isso no Redis foi desperdício.
```

#### Solução: Escala Logarítmica

```
Armazenar log2(likes) em vez de likes:

post_A: 1.000.000 likes → score = log2(1M) ≈ 20
post_B: 100 likes       → score = log2(100) ≈ 7
post_C: 1 like          → score = log2(1) = 0

Gravar no Redis APENAS quando o score muda:
  1.000.000 → 1.000.001: log2 não muda → não escrever!
  1.000.000 → 2.000.000: log2 muda 20 → 21 → escrever!
```

**Resultado:** Reduz writes drásticamente, mantendo ordenação razoável.

#### Re-ranking no Search Service

```
Para compensar a perda de precisão:

1. Redis retorna top 100 posts por "likes" (score logarítmico)
2. Search Service: busca like counts REAIS de cada um dos 100 posts
3. Re-ordena com precisão antes de retornar para o cliente
4. Retorna top 25

→ Precisão nas respostas finais
→ Baixo write volume no Redis
```

---

### Deep Dive 4: Redis Cluster e Particionamento

#### Problema
Um único Redis: ~100K TPS. Nosso sistema precisa de mais.

#### Opção 1: Particionar por keyword

```
keyword "trump" → Redis Node 1
keyword "taylor" → Redis Node 2
keyword "swift" → Redis Node 3

Problema: Hot key!
  Se "taylor swift" vira tendência → Node 2 explode
```

#### Opção 2: Particionar por shard do post_id

```
likes:taylor:shard_0 → Redis Node 1 (posts com id % 4 == 0)
likes:taylor:shard_1 → Redis Node 2 (posts com id % 4 == 1)
likes:taylor:shard_2 → Redis Node 3 (posts com id % 4 == 2)
likes:taylor:shard_3 → Redis Node 4 (posts com id % 4 == 3)

Write: distribui uniformemente
Read: busca em 4 nós → merge → N vezes mais trabalho para Search Service

Parâmetro N ajustável conforme necessidade
```

---

### Deep Dive 5: Hot vs Cold Storage

#### Problema
3.6 Petabytes no Redis → inviável financeiramente.

#### Solução: Tier de Storage

```
Hot tier (Redis):  keywords com acesso frequente → latência < 1ms
Cold tier (S3):    keywords raramente buscadas  → latência > 100ms

Fluxo:
1. Search Service busca no Redis
2. Cache miss → busca no S3
3. Carrega no Redis para próximas queries
```

#### Como identificar o que é "frio"? Count-Min Sketch

```
Count-Min Sketch: estrutura probabilística para contar frequências
  - Memória: muito menor que um HashMap
  - Garantia: upper bound da frequência (pode superestimar, nunca subestima)

Uso:
  Para cada keyword buscada → incrementa no CMS
  Periodicamente: escaneia Redis → remove keywords com CMS count < threshold → move para S3
```

```
Exemplo:
  "taylor": 50.000 buscas/dia → hot → permanece no Redis
  "stefan": 12 buscas/dia    → cold → move para S3
```

---

### Deep Dive 6: Busca Multi-keyword (Bigramas)

#### Problema
Buscar "taylor swift" (duas palavras juntas, não separadas)

#### Abordagem 1: Interseção (lenta)

```
posts com "taylor" → set A
posts com "swift"  → set B
interseção → candidatos

Para cada candidato: verificar se "taylor" e "swift" são adjacentes
→ N × custo de verificação
```

#### Abordagem 2: Bigramas (rápida, mais storage)

```
Indexar pares de palavras adjacentes:
  "taylor swift lança" → bigramas: "taylor swift", "swift lança"

Índice:
  "taylor swift" → [post_1, post_5, ...]
  "swift lança"  → [post_1, ...]

Busca direta por "taylor swift" → resultado imediato
```

**Trade-off:** 100 palavras → 99 bigramas → ~2x mais storage no índice.

#### Combinação inteligente com Count-Min Sketch

```
Só indexar bigramas que são frequentemente buscados:
  CMS["taylor swift"] > 1000 searches/dia → indexar como bigrama
  CMS["stefan swift"] < 10 searches/dia  → não indexar, usar interseção
```

---

## 📊 Resumo de Decisões Técnicas

| Problema | Solução |
|---|---|
| Full table scan em SQL | Inverted Index em Redis Sorted Sets |
| Alta leitura | Cache local + CDN |
| Picos de escrita | Kafka como buffer |
| Volume de likes no índice | Score logarítmico + re-ranking no serviço |
| Hot key no Redis | Sharding por shard_id do post |
| Memory de Redis | Hot/Cold tiers com S3 |
| Multi-keyword | Bigramas seletivos com Count-Min Sketch |

---

## 🎓 Expectativas por Nível

| Nível | Expectativa |
|---|---|
| **Pleno** | Conceito de inverted index, identificar problema do LIKE em SQL |
| **Sênior** | Redis Sorted Sets para ordenação, Kafka para write throughput |
| **Staff** | Logaritmo para reduzir writes, sharding strategy, bigramas, Count-Min Sketch |

---

*Arquivo gerado a partir do breakdown da Hello Interview — Facebook Post Search*
