# 25 — Indexação de Banco de Dados

> Do problema fundamental aos tipos de índice e quando usar cada um em System Design

---

## O Problema que Índices Resolvem

### Full Table Scan (sem índice)

```
Banco de dados armazena dados em Pages (páginas)
Cada page = ~8KB = ~100 rows

Para encontrar um item SEM índice:
1. Carrega page 1 na RAM → escaneia 100 rows → não achou
2. Carrega page 2 na RAM → escaneia 100 rows → não achou
3. ... repete até encontrar

100 milhões de usuários = 1 milhão de pages
Cada round-trip SSD→RAM = ~100 microsegundos
Tempo máximo: 100 segundos ❌

(Na prática, com prefetching: 3-5s — ainda inaceitável)
```

### Com Índice

```
Índice = estrutura de dados em disco que mapeia valor → página

1. Carrega índice na RAM
2. Consulta o índice: "user_id = 101 está na page 42"
3. Carrega apenas a page 42
→ 2-3 I/Os no total → milissegundos ✓
```

---

## B-Tree Index (O mais comum)

### Estrutura

```
                    [25 | 50 | 75]  ← Nó raiz
                   /    |    |    \
             [10|20] [30|40] [60|70] [80|90]  ← Nós intermediários
                ↓       ↓      ↓       ↓
            [pages] [pages] [pages] [pages]   ← Folhas → ponteiros para data pages
```

### Funcionamento

**Query exata:** `SELECT * FROM users WHERE age = 51`
```
1. Carrega raiz: 51 > 50, < 90 → vai para ramo direito
2. Carrega página intermediária: 51 < 55 → vai para page 3
3. Carrega page 3: encontra todos os usuários de 51 anos
Total: 3 I/Os ✓
```

**Range query:** `SELECT * FROM users WHERE age > 50`
```
1. Navega até o ponto de início (age=51)
2. Percorre as folhas sequencialmente (B-trees são duplamente ligadas nas folhas)
3. Retorna todos os registros dentro do range
→ Excelente para ranges! ✓
```

### Casos de Uso do B-Tree
- Equality queries: `WHERE col = val`
- Range queries: `WHERE col BETWEEN a AND b`
- Prefix queries: `WHERE col LIKE 'pizza%'`
- Ordenação: `ORDER BY col`

### Limitação
- **Não funciona** para busca por substring: `WHERE name LIKE '%pizza%'`
- **Não eficiente** para dados 2D (latitude, longitude simultâneos)

---

## Hash Index

### Estrutura

```
hash(email) → pointer para data page

hash("john@...") = 0x4A2F → page 4
hash("sarah@...") = 0x8B1C → page 7
```

### Quando usar

- Equality queries: `WHERE email = 'john@example.com'` → O(1)
- **Não suporta** range queries
- **Principalmente em stores in-memory** (Redis, MemCache)

> **Na prática:** B-Trees são quase tão rápidos em equalities e suportam ranges. Hash indexes são raramente usados em bancos de dados em disco.

---

## Geospatial Indexes

### Por que B-Tree não funciona para 2D?

```
Query: WHERE latitude BETWEEN 100 AND 400 AND longitude BETWEEN 20 AND 200

B-Tree em latitude → retorna "faixa horizontal" enorme
B-Tree em longitude → retorna "faixa vertical" enorme
Intersect dos dois → operação custosa em memória

[===latitude range===]
        ↓
    ████████  ← precisamos apenas desta área
████[████████]████  ← mas estamos carregando esta faixa toda
        ↑
[===longitude range===]
```

### Geohashing

**Conceito:** converter coordenadas 2D em string 1D preservando proximidade geográfica.

```
Mapa → dividir em 4 quadrantes (0, 1, 2, 3)
Cada quadrante → subdividir recursivamente
Resultado: string de precisão crescente

Albuquerque ≈ "310"
Vizinhança imediata: "311", "312", "313", "320"...

Propriedade chave: locais próximos compartilham prefixo comum!
```

**Na prática:**
- Geohash de Los Angeles ≈ `9q5cs`
- Encode em base32 para eficiência
- Construir B-Tree sobre os hashes → range queries em 1D ✓

**Disponível nativamente:** Redis (comando `GEOADD`, `GEODIST`, `GEORADIUS`)

### Quad Trees

```
Dividir o mapa em 4 quadrantes → árvore quaternária
Profundidade recursiva apenas onde há alta densidade

         [mundo]
        /  |  |  \
      [NW][NE][SW][SE]
              |
         [muitos pontos]
        /  |  |  \
    [SW1][SE1][NW1][NE1]
```

**Parâmetro K:** se um quadrante tem > K pontos, subdivide novamente.

**Usado em:** Mapas históricos, alguns sistemas legados. Menos comum hoje.

### R-Trees

- Evolução dos Quad Trees
- Agrupamento dinâmico (não divide em 4 fixos)
- Bounding boxes com sobreposição permitida
- **Usado em produção:** PostGIS (extensão geospatial do PostgreSQL)

```sql
-- Com PostGIS:
CREATE INDEX ON businesses USING GIST(location);
SELECT * FROM businesses WHERE ST_DWithin(location, ST_MakePoint(-73.9, 40.7), 1000);
```

### Comparativo

| Algoritmo | Usado em | Status |
|---|---|---|
| Geohashing | Redis, sistemas modernos | ✅ Recomendado |
| Quad Trees | Fundação teórica | ⚠️ Pouco usado hoje |
| R-Trees | PostGIS, Elasticsearch | ✅ Produção |

---

## Inverted Index (Full-Text Search)

### Por que B-Tree não funciona para substring?

```
B-Tree é ordenado lexicograficamente
"pizza" aparece em "Best Pizza NYC", "NY Pizza Palace", "The Pizza Place"
Busca por '%pizza%' não tem ponto de início no B-Tree → Full Scan
```

### Estrutura do Inverted Index

```
Documentos:
  Doc 1: "B-trees are fast and reliable"
  Doc 2: "Hash tables are fast but limited"
  Doc 3: "B-trees handle range queries well"

Inverted Index:
  "fast"    → [Doc 1, Doc 2]
  "btrees"  → [Doc 1, Doc 3]
  "range"   → [Doc 3]
  "hash"    → [Doc 2]
  "limited" → [Doc 2]
```

**Query:** `WHERE content CONTAINS 'fast'`
```
1. Lookup "fast" no hashmap → [Doc 1, Doc 2]
2. Busca esses documentos → retorna direto ✓
```

### Onde Usar Inverted Index

- **Elasticsearch**: motor de busca full-text (built on Lucene)
- **PostgreSQL full-text search**: `to_tsvector()`, `to_tsquery()`
- **Qualquer busca por palavra em texto livre**

---

## Fluxograma de Decisão

```
Precisa de acesso eficiente a dados?
├── NÃO → Full Table Scan (OK para tabelas pequenas)
└── SIM → Tabela grande (>100k rows)?
          ├── NÃO → Full Table Scan ainda OK
          └── SIM → Que tipo de dado?
                    ├── TEXTO livre (busca por palavra)
                    │   → Inverted Index (Elasticsearch, PG full-text)
                    ├── LOCALIZAÇÃO (lat/lng)
                    │   → Geospatial Index (Redis geo, PostGIS)
                    ├── EQUALITY em memória (muito rápido)
                    │   → Hash Index (Redis, MemCache)
                    └── TUDO MAIS (ranges, sorts, equality)
                        → B-Tree Index ✓ (default)
```

---

## Boas Práticas em Interviews

### Quando mencionar índices

1. Ao definir o modelo de dados, mencione quais colunas terão índice
2. Justifique: "query por `video_id` será frequente, então vou indexar essa coluna"
3. Para dados 2D: mencione geospatial index explicitamente
4. Para busca full-text: mencione Elasticsearch ou PG full-text

### O que interviewers esperam

- Que você identifique **queries ineficientes** no seu design
- Que você saiba **qual índice** aplicar em cada situação
- Que você entenda o **trade-off**: índice acelera reads, mas ocupa espaço e adiciona overhead em writes

---

## Resumo Técnico

| Tipo | Estrutura | Complexity | Uso |
|---|---|---|---|
| B-Tree | Árvore balanceada | O(log N) | Ranges, sorts, equality |
| Hash Index | Hash map | O(1) | Equality in-memory |
| Geohash + B-Tree | Hash + B-Tree | O(log N) | Latitude/longitude |
| R-Tree / Quad Tree | Árvore espacial | O(log N) | Dados 2D complexos |
| Inverted Index | Hash map invertido | O(1) lookup | Full-text search |

---

*Fonte: Hello Interview — Database Indexing Deep Dive*
