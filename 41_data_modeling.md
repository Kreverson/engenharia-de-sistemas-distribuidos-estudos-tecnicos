# 41 — Data Modeling para System Design

> Como estruturar dados de forma que o sistema escale, seja consistente e responda rápido. Um dos pontos mais cobrados em entrevistas de System Design mas frequentemente abordado de forma superficial.

---

## 1. O que é Data Modeling neste contexto

Data modeling em System Design não é sobre diagrama ER acadêmico. É sobre tomar decisões que impactam diretamente:
- **Performance de leitura e escrita**
- **Consistência dos dados**
- **Capacidade de escala horizontal**
- **Facilidade de manutenção**

Aparece em duas fases da entrevista:
1. **Core Entities** → quais tabelas/coleções existem
2. **High-Level Design** → quais campos, índices, chaves estrangeiras, sharding

---

## 2. Escolha do banco de dados

> **Regra geral para entrevistas:** Se você não tem experiência profunda com outro tipo, use **PostgreSQL**. É extremamente difícil justificar o contrário em 45 minutos.

### Bancos Relacionais (PostgreSQL, MySQL)

**Ideal quando:**
- Dados têm relações bem definidas
- Necessidade de transações ACID
- Queries ad-hoc e flexíveis são necessárias
- Schema relativamente estável

```sql
-- Exemplo: Instagram simplificado
CREATE TABLE users (id UUID PRIMARY KEY, username TEXT UNIQUE, email TEXT);
CREATE TABLE posts (id UUID PRIMARY KEY, user_id UUID REFERENCES users(id), content TEXT, created_at TIMESTAMP);
CREATE TABLE likes (user_id UUID REFERENCES users(id), post_id UUID REFERENCES posts(id), PRIMARY KEY(user_id, post_id));
```

### Bancos de Documentos (MongoDB, Firestore)

**Ideal quando:**
- Schema evolui frequentemente
- Dados naturalmente hierárquicos/aninhados
- Sem necessidade de joins complexos

```json
// Documento de usuário com posts embutidos
{
  "_id": "usr_123",
  "username": "alice",
  "posts": [
    { "id": "post_1", "content": "Hello world", "likes": 42 }
  ]
}
```

**Hot take importante:** Em entrevistas, você **define um escopo fixo** — não há schema evoluindo. Isso elimina a principal vantagem do MongoDB. Use Postgres.

### Key-Value Stores (Redis, DynamoDB)

**Ideal quando:**
- Lookups por chave exata (O(1))
- Cache em memória
- Sessões de usuário
- Contadores

```
key: "user:123:session"     value: { token, expires_at, permissions }
key: "user:123:feed"        value: [post_ids em ordem reversa]
key: "rate_limit:alice:60"  value: 47 (requests neste minuto)
```

**Limitação crítica:** Não suporta `WHERE`, `JOIN`, `ORDER BY`. Você só pode buscar pela chave exata.

### Wide Column Stores (Cassandra, HBase)

**Ideal quando:**
- Volume massivo de writes (> 100K writes/s)
- Query patterns bem definidos e limitados
- Tolerância a eventual consistency

Exemplo: Logs de atividade, métricas de IoT, feeds de redes sociais.

```
// Cassandra: cada linha pode ter colunas diferentes
user_messages:
  user_101 → { name: "Alice", email: "...", address: "..." }
  user_102 → { name: "Bob", email: "..." }  // sem address
```

### Bancos de Grafos (Neo4j, Neptune)

**Evitar em entrevistas.** Mesmo o Facebook modela seu grafo social com MySQL. Use relacional com índices adequados.

---

## 3. Os três fatores que guiam o schema

### Fator 1: Volume de dados

Determina onde os dados vivem fisicamente:
```
500M usuários × 5KB cada = 2.5TB
→ Cabe em 1 instância PostgreSQL? Sim (140TB max)
→ Precisamos sharding? Não por storage

500M usuários × 100 posts × 1KB = 50TB
→ Precisamos sharding? Provavelmente sim
```

### Fator 2: Padrões de acesso (o mais importante)

Mapeie suas APIs para queries:

```
API: GET /users/{id}/posts?limit=25
SQL: SELECT * FROM posts WHERE user_id = ? ORDER BY created_at DESC LIMIT 25
Índice necessário: (user_id, created_at DESC)
```

```
API: GET /feed?user_id={id}
SQL: Posts dos usuários que sigo, ordenados por data
→ JOIN complexo → considerar desnormalização ou cache
```

### Fator 3: Consistência necessária

```
Transferência bancária → ACID forte → mesma instância, transação
Feed de notícias → eventual consistency aceitável → pode estar em cache separado
Contagem de likes → aproximada OK → HyperLogLog ou contador eventual
```

---

## 4. Chaves primárias e estrangeiras

### Chave Primária (PK)

Identifica unicamente cada registro. Características:
- Única por tabela
- Imutável (não muda)
- Não nula

```sql
-- UUID: distribuído, sem coordenação central (bom para sharding)
id UUID DEFAULT gen_random_uuid() PRIMARY KEY

-- Serial: simples mas problemático em sharding
id SERIAL PRIMARY KEY

-- Snowflake/KSUID: sortável por tempo (útil para cursored pagination)
id TEXT DEFAULT generate_ksuid() PRIMARY KEY
```

### Chave Estrangeira (FK)

Garante **integridade referencial** — impossível criar um post para um user que não existe:

```sql
CREATE TABLE posts (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  -- ON DELETE CASCADE: se o user for deletado, seus posts também são
  created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

### Constraints comuns

```sql
-- Unicidade
UNIQUE (email)
UNIQUE (user_id, post_id)  -- combinação única (likes)

-- Not null
NOT NULL

-- Check constraint
CHECK (price > 0)
CHECK (status IN ('pending', 'active', 'cancelled'))
```

> **Dica em entrevistas:** Não é preciso sempre escrever `REFERENCES` explicitamente se a relação for óbvia. Mencione verbalmente "user_id é FK para users".

---

## 5. Normalização vs Desnormalização

### Normalização

Cada dado em **um único lugar**. Elimina anomalias de atualização:

```sql
-- Normalizado
users:  id, username, email
posts:  id, user_id, content
-- Se email muda, muda em 1 lugar
```

**Problema:** para carregar post + username do autor → JOIN obrigatório.

### Desnormalização

Duplicar dados para eliminar JOINs e acelerar leituras:

```sql
-- Desnormalizado
posts: id, user_id, user_username, content
-- Se username muda → precisa atualizar TODOS os posts desse user
```

**Regra em entrevistas:** Sempre comece normalizado. Desnormalize apenas quando:
1. Existe gargalo de leitura comprovado
2. Os dados raramente mudam (ou mudança eventual é aceitável)
3. A desnormalização vai para um **cache** (Redis/Memcached)

```
PostgreSQL (normalizado, fonte da verdade)
        ↓ escrita dupla
Redis  (desnormalizado: { post + user_info + like_count })
        ↓ leitura rápida O(1)
    Feed do usuário (TTL de 5 minutos)
```

---

## 6. Indexação

### O que é um índice

Um índice é uma estrutura de dados auxiliar que mantém apontadores para registros, ordenados por um ou mais campos. O tipo mais comum é a **B-Tree**:

```
Sem índice:
SELECT * FROM posts WHERE user_id = '123'
→ Full table scan: O(N) — lê TODOS os registros

Com índice em user_id:
→ B-Tree lookup: O(log N) — navega a árvore
```

### Quando criar índices

Sempre que a API fizer queries por aquele campo:

```sql
-- API: GET /posts?user_id=X&order=desc
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at DESC);

-- API: GET /comments?post_id=X
CREATE INDEX idx_comments_post ON comments(post_id);

-- API: GET /users?email=X (login)
CREATE UNIQUE INDEX idx_users_email ON users(email);
```

### Custo dos índices

**Escrita:** cada INSERT/UPDATE/DELETE precisa atualizar todos os índices da tabela. Muitos índices = writes lentos.

**Trade-off:** reads rápidos vs writes mais lentos. Para leitura-intensiva, vale a pena.

### Índices compostos — ordem importa

```sql
-- Este índice funciona para:
-- WHERE user_id = ?  ✅
-- WHERE user_id = ? AND created_at > ?  ✅
-- WHERE created_at > ?  ❌ (não usa o índice eficientemente)
CREATE INDEX ON posts(user_id, created_at);
```

A regra: queries que usam prefixo do índice o utilizam. Queries que pulam o primeiro campo não.

---

## 7. Sharding

### Quando é necessário

```
Calcule primeiro:
- Storage: 50TB > 140TB (max Postgres)? Não → sem sharding
- Write throughput: 100K w/s > 10K w/s (max Postgres)? Sim → sharding
```

### Escolha da shard key

A shard key determina em qual shard cada registro fica. Critérios:
1. **Alta cardinalidade** (muitos valores únicos)
2. **Distribuição uniforme** (sem hot shards)
3. **Alinhamento com queries** (dados relacionados no mesmo shard)

```
Ruim: shard por `is_premium` → apenas 2 shards possíveis
Ruim: shard por `created_date` → shard mais novo sempre sobrecarregado
Bom:  shard por `user_id` → alta cardinalidade + queries de usuário ficam no mesmo shard
```

### Evitar cross-shard JOINs

```sql
-- Posts e comentários shardados por user_id:
-- GET /users/123/posts → vai ao shard de user 123 ✅
-- GET /posts/456/comments → comentário shardado por post ou user?

-- Se por user_id: comentários de um post ficam em shards diferentes ❌
-- Solução: shard comentários por post_id (mesma lógica dos posts)
-- Ou: desnormalize comentários dentro do post
```

### Shard key no schema

```sql
-- Notação em entrevista
posts:
  id UUID PK
  user_id UUID FK     [SHARD KEY]
  content TEXT
  created_at TIMESTAMP
  INDEX: (user_id, created_at)

comments:
  id UUID PK
  post_id UUID FK     [SHARD KEY]
  user_id UUID
  content TEXT
```

---

## 8. Exemplo completo — Instagram

### Schema

```sql
-- Usuários
CREATE TABLE users (
  id        UUID PRIMARY KEY,
  username  TEXT UNIQUE NOT NULL,
  email     TEXT UNIQUE NOT NULL,
  bio       TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Posts
CREATE TABLE posts (
  id         UUID PRIMARY KEY,
  user_id    UUID NOT NULL REFERENCES users(id),
  image_url  TEXT NOT NULL,
  caption    TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_posts_user ON posts(user_id, created_at DESC);  -- SHARD KEY: user_id

-- Follows
CREATE TABLE follows (
  follower_id UUID REFERENCES users(id),
  followee_id UUID REFERENCES users(id),
  PRIMARY KEY (follower_id, followee_id),
  created_at  TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_follows_followee ON follows(followee_id);  -- para contar seguidores

-- Likes
CREATE TABLE likes (
  user_id UUID REFERENCES users(id),
  post_id UUID REFERENCES posts(id),
  PRIMARY KEY (user_id, post_id),
  created_at TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_likes_post ON likes(post_id);  -- para contar likes de um post
```

### Cache (Redis) — desnormalização para performance

```
feed:{user_id}  → [post_id1, post_id2, ...] (últimos 100 posts, TTL 5min)
post:{post_id}  → { id, image_url, caption, user_username, like_count }  (TTL 1min)
user:{user_id}:stats → { followers, following, post_count }  (TTL 1min)
```

---

## 9. Roteiro para entrevistas

```
1. Ao desenhar a caixa do banco, anuncie o tipo: "PostgreSQL aqui"

2. Liste as tabelas com campos principais:
   posts: id, user_id (FK), content, created_at

3. Marque PKs e FKs:
   id = PK
   user_id = FK → users

4. Adicione constraints relevantes:
   email UNIQUE

5. Derive índices das APIs:
   "API carrega posts por user → índice em (user_id, created_at)"

6. Considere desnormalização se necessário:
   "Feed frequente → cache Redis com dados pré-computados"

7. Avalie sharding se os números indicarem:
   "50TB de posts → shard por user_id para distribuir storage e writes"
```

---

*Tópicos relacionados: `03_consistencia_e_disponibilidade.md`, `23_dynamodb_deep_dive.md`, `27_consistent_hashing.md`, `25_indexacao_banco_dados.md`*
