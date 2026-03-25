# 35 — Design Yelp: Busca Geoespacial com PostGIS

> Baseado no guided practice da Hello Interview com Evan (ex-Staff Meta). Cobre busca geoespacial, full-text search, índices compostos e o design de um sistema de reviews com consistência transacional.

---

## 📌 Contexto do Problema

Design de um site de reviews de negócios locais (estilo Yelp):
- **100 milhões de usuários ativos diários**
- **10 milhões de negócios**
- Um usuário só pode deixar **uma review por negócio**

---

## 🎯 Requisitos

### Funcionais
```
1. Buscar negócios por: nome, localização (lat/lng), categoria
2. Visualizar negócio e suas reviews
3. Deixar review (1-5 estrelas + texto opcional)
[bonus] Paginação de reviews
```

### Não-Funcionais
```
- Alta disponibilidade >> consistência forte
- Eventual consistency OK para reviews
- Latência de busca < 500ms (desafio: busca geoespacial)
- Escala: 100M DAU, 10M negócios
```

---

## 🏗️ Entidades e Schema

```sql
-- Negócio
CREATE TABLE businesses (
  id UUID PRIMARY KEY,
  name VARCHAR,
  description TEXT,
  address VARCHAR,
  latitude FLOAT,
  longitude FLOAT,
  category VARCHAR,
  average_rating FLOAT,
  num_ratings INT,
  s3_image_url VARCHAR
);

-- Review
CREATE TABLE reviews (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  business_id UUID REFERENCES businesses(id),
  rating INT CHECK (rating >= 1 AND rating <= 5),
  text TEXT,
  created_at TIMESTAMP,
  UNIQUE (user_id, business_id)  ← constraint de 1 review por negócio
);

-- Usuário
CREATE TABLE users (
  id UUID PRIMARY KEY,
  -- ... outros campos
);
```

---

## 🔌 API Design

```http
GET /businesses?category=restaurant&lat=48.8566&lng=2.3522&name=bistro&page=1&limit=25
  → 200 OK, { businesses: [{ id, name, avg_rating, category, thumbnail_url }] }

GET /businesses/{id}
  → 200 OK, { id, name, description, address, avg_rating, ... }

GET /businesses/{id}/reviews?cursor={timestamp}&limit=25
  → 200 OK, { reviews: [...], next_cursor: timestamp }

POST /businesses/{id}/reviews   [Autenticado]
  body: { rating: 4, text: "Ótimo lugar!" }
  → 201 Created, { review_id }
```

### Decisão de API: Separar business details e reviews

```
❌ GET /businesses/{id} → retorna tudo (business + reviews)
  → Reviews podem demorar (paginação, volume)
  → Bloqueia renderização do negócio

✅ GET /businesses/{id}          → renderiza imediatamente
   GET /businesses/{id}/reviews  → carrega após (lazy loading)
  → Melhor UX (skeleton loading para reviews)
```

---

## 🏛️ High-Level Design

```
                    ┌────────────────────┐
                    │    API Gateway     │
                    └────────┬───────────┘
               ┌─────────────┼─────────────┐
               ▼             ▼             ▼
        ┌────────────┐ ┌──────────┐ ┌──────────────┐
        │Business Svc│ │Review Svc│ │  Search Svc  │
        └─────┬──────┘ └────┬─────┘ └──────┬───────┘
              └─────────────┼──────────────┘
                            ▼
                    ┌─────────────────┐
                    │    PostgreSQL   │
                    │  (primary DB)   │
                    │  businesses     │
                    │  reviews        │
                    │  users          │
                    └─────────────────┘
```

### Por que um único PostgreSQL em vez de microsserviços com DBs separados?

```
Negócios e reviews têm JOIN natural:
  SELECT b.*, r.* FROM businesses b JOIN reviews r ON b.id = r.business_id

Separar em 2 bancos: cross-database join = complexidade enorme

Escala: 10 milhões de negócios × ~1 KB/negócio = ~10 GB → trivial!

Regra: Só separe bancos quando PRECISA, não por princípio de microsserviços.
```

---

## 🚀 Deep Dives

### Deep Dive 1: Average Rating — Atualização Transacional

**Problema:** Como manter `average_rating` atualizado sem processar todas as reviews a cada busca?

#### Abordagem Ingênua (errada)
```
Cron job diário: recalcula average de todas as reviews
→ Não é "up to the minute"

Query dinâmica: SELECT AVG(rating) FROM reviews WHERE business_id = X
→ Lento em escala, sem aproveitar índice
```

#### Solução: Atualização Incremental em Transação

```python
# Fórmula da média incremental:
# nova_media = (media_atual × num_ratings + novo_rating) / (num_ratings + 1)

def create_review(business_id, user_id, rating, text):
    with transaction(isolation_level=SERIALIZABLE):
        # 1. Cria a review
        INSERT INTO reviews (business_id, user_id, rating, text, ...)

        # 2. Atualiza o negócio atomicamente
        UPDATE businesses
        SET
            average_rating = (average_rating * num_ratings + rating) / (num_ratings + 1),
            num_ratings = num_ratings + 1
        WHERE id = business_id
```

#### Por que usar transação com SERIALIZABLE?

```
Cenário sem transação:
  Thread A lê: average_rating=4.0, num_ratings=10
  Thread B lê: average_rating=4.0, num_ratings=10
  Thread A calcula: (4.0 × 10 + 5) / 11 = 4.09 → escreve
  Thread B calcula: (4.0 × 10 + 3) / 11 = 3.90 → SOBRESCREVE!
  → Review do Thread A perdida!

Com SERIALIZABLE isolation:
  Thread A e B não podem ler/escrever simultaneamente no mesmo registro
  → Uma executa, depois a outra → resultado correto
```

#### Opção alternativa: Row Locking Pessimista

```sql
BEGIN;
  SELECT * FROM businesses WHERE id = X FOR UPDATE;  -- lock na linha
  -- agora nenhuma outra transação pode modificar este negócio
  UPDATE businesses SET average_rating = ..., num_ratings = ... WHERE id = X;
  INSERT INTO reviews (...);
COMMIT;
```

---

### Deep Dive 2: Constraint "1 Review por Negócio"

#### Hierarquia de onde colocar a validação

```
1. Frontend (melhor UX — botão "deixar review" some se já foi)
   ⚠️ Fácil de bypassar — nunca suficiente

2. Application Logic (Review Service)
   if review_exists(user_id, business_id): raise Error
   ⚠️ Fácil de esquecer em outros serviços

3. Database Constraint (melhor — garante invariante)
   UNIQUE (user_id, business_id)
   → Qualquer serviço que tente inserir segunda review → erro de DB
   → Garantia absoluta mesmo com múltiplos serviços, batch jobs, etc.
```

**Regra de ouro:** Quanto mais próximo do dado, mais confiável a validação.

---

### Deep Dive 3: Busca Geoespacial — PostGIS

**Problema:** A busca exige encontrar negócios dentro de um raio de uma localização (lat/lng). Um índice B-Tree não serve para coordenadas 2D.

#### Por que não funciona com índice normal?

```sql
-- Índice B-Tree em latitude funciona para:
SELECT * FROM businesses WHERE latitude > 48.8 AND latitude < 49.0;
-- → Mas ainda retorna negócios muito distantes em longitude!

-- LIKE para nome também não usa índice eficientemente:
SELECT * FROM businesses WHERE name LIKE '%bistro%';
-- → Full table scan
```

#### Solução 1: Elasticsearch (padrão de mercado)

```
Elasticsearch tem:
  - Inverted index para texto (nome, descrição)
  - Geo index para coordenadas
  - Relevance scoring nativo

Desvantagem:
  - Precisa de CDC (Change Data Capture) para sincronizar com o DB
  - Mais infraestrutura para manter
  - Consistência eventual entre DB e ES
```

#### Solução 2: PostgreSQL + PostGIS (sem nova infra!)

```sql
-- Habilitar extensão
CREATE EXTENSION postgis;
CREATE EXTENSION pg_trgm;  -- para full text search

-- Índice geoespacial (GiST)
CREATE INDEX businesses_location_idx
  ON businesses USING GIST (ST_MakePoint(longitude, latitude));

-- Índice de texto com GIN (trigram)
CREATE INDEX businesses_name_idx
  ON businesses USING GIN (name gin_trgm_ops);

-- Índice para categoria
CREATE INDEX businesses_category_idx ON businesses (category);
```

**Query de busca:**
```sql
SELECT
  id, name, category, average_rating,
  ST_Distance(
    ST_MakePoint(longitude, latitude)::geography,
    ST_MakePoint($user_lng, $user_lat)::geography
  ) AS distance_meters
FROM businesses
WHERE
  category = $category
  AND name ILIKE $name_pattern  -- usa índice trigram
  AND ST_DWithin(
    ST_MakePoint(longitude, latitude)::geography,
    ST_MakePoint($user_lng, $user_lat)::geography,
    $radius_meters  -- ex: 5000 para 5 km
  )
ORDER BY distance_meters ASC
LIMIT 25;
```

#### Comparação das abordagens

| Aspecto | Elasticsearch | PostgreSQL + PostGIS |
|---|---|---|
| **Setup** | Nova infraestrutura | Extensão existente |
| **Consistência** | Eventual (CDC) | Imediata (mesmo DB) |
| **Manutenção** | Alta | Baixa |
| **Performance** | Melhor em escala extrema | Suficiente até centenas de milhões |
| **Custo** | Alto | Baixo |
| **Recomendação** | Quando PostGIS não escala mais | Primeiro passo |

---

## 📊 Análise Completa de Escalabilidade

```
Negócios: 10M × 1 KB = 10 GB → um único nó de Postgres
Reviews: 10M negócios × 100 reviews avg × 500 bytes = 500 GB → escala com particionamento

DAU: 100M
  Buscas: 1/usuário/dia → 1.000 buscas/segundo
  Views: 5/usuário/dia  → 5.000 views/segundo
  Reviews: 0.01/usuário/dia → 10 reviews/segundo (muito baixo)

→ Sistema é READ-HEAVY → candidato natural para caching de buscas populares
```

---

## 🔄 Diagrama Final

```
                  ┌────────────────────────────────────┐
                  │           API Gateway              │
                  └──────────────┬─────────────────────┘
         ┌──────────────┬────────┴────────┬────────────────┐
         ▼              ▼                 ▼                ▼
  ┌────────────┐ ┌────────────┐  ┌──────────────┐ ┌──────────┐
  │Business Svc│ │Review Svc  │  │  Search Svc  │ │  Feed CDN│
  └─────┬──────┘ └─────┬──────┘  └──────┬───────┘ └──────────┘
        │              │                 │
        └──────────────┴─────────────────┘
                       ▼
              ┌──────────────────┐
              │   PostgreSQL     │
              │ + PostGIS        │
              │ + pg_trgm        │
              │                  │
              │ businesses       │
              │ reviews          │
              │ users            │
              └──────────────────┘
```

---

## 💡 Padrões e Lições

| Padrão | Aplicação |
|---|---|
| **Database constraint para unicidade** | 1 review/negócio, 1 like/post, 1 voto/opção |
| **Média incremental em transação** | Qualquer métrica agregada em tempo real |
| **SERIALIZABLE isolation** | Quando duas operações precisam ser atômicas e consistentes |
| **PostGIS para geo** | Busca de restaurantes, motoristas, entregadores |
| **Extensão vs novo serviço** | Prefira estender tecnologia existente quando suficiente |
| **Lazy loading de dados pesados** | Reviews separadas dos detalhes do negócio |

---

## 🎓 Score e Feedback Real

Na prática da Hello Interview, o design recebeu **95/100** com críticas:
- ✅ Alta disponibilidade vs consistência
- ✅ Entidades e API
- ✅ Busca com PostGIS
- ✅ Constraint de unicidade no DB
- ⚠️ Faltou paginação nos parâmetros de busca (adicionado após feedback)
- ⚠️ Poderia separar business details e reviews na API de leitura

---

*Arquivo gerado a partir do guided practice da Hello Interview — Design Yelp*
