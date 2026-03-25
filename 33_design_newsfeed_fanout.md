# 33 — Design Newsfeed: Fan-out, Pre-computação e Hot Keys

> Baseado no breakdown da Hello Interview com Stefan (ex-interviewer Meta/Amazon). Cobre o problema clássico de newsfeed com foco em fan-out de escritas, pre-computação de feeds e o problema de hot shards.

---

## 📌 Contexto do Problema

Design de um newsfeed estilo Facebook (versão simplificada):
- Usuários podem se seguir unidirecionalmente
- Feed cronológico de posts de quem você segue
- **2 bilhões de usuários**
- Eventualmente consistente (1 minuto de delay aceitável)
- Latência < 500ms

---

## 🎯 Requisitos

### Funcionais
```
1. Criar posts
2. Seguir usuários
3. Ver feed cronológico
4. Paginação infinita (scroll)
```

### Não-Funcionais
```
- Eventual consistency (1 minuto de delay OK)
- Latência de leitura < 500ms
- Alta disponibilidade >> consistência forte
- 2 bilhões de usuários
```

---

## 🏗️ Entidades e Schema

```
User:
  id, name, ...

Post:
  id, creator_id, content, created_at

Follow:
  user_following (PK), user_followed (SK)    ← GSI reverso disponível
```

---

## 🔌 API Design

```http
POST /posts
  body: { content: "..." }
  → 201 Created, { post_id }

PUT /users/{id}/followers
  (sem body — apenas autenticação)

GET /feed?page_size=25&cursor={timestamp}
  → { posts: [...], next_cursor: timestamp }
```

### Cursor-based Pagination

```
Primeira chamada:    GET /feed?page_size=25
  → retorna 25 posts + cursor = "2024-01-15T10:30:00Z"

Segunda chamada:     GET /feed?page_size=25&cursor=2024-01-15T10:30:00Z
  → retorna 25 posts ANTERIORES ao cursor

Vantagem vs offset pagination:
  - Não tem problema de "registro novo aparecendo na página errada"
  - Funciona mesmo com inserções durante scroll
  - Consistente independente do tamanho total
```

---

## 🏛️ High-Level Design (Simples, sem otimização)

```
                    ┌─────────────────┐
                    │   API Gateway   │
                    │ + Load Balancer │
                    └───────┬─────────┘
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
       ┌──────────┐  ┌──────────┐  ┌──────────┐
       │Post Svc  │  │Follow Svc│  │Feed Svc  │
       └────┬─────┘  └────┬─────┘  └────┬─────┘
            │              │              │
            ▼              ▼              │
       ┌──────────────────────────┐       │
       │      DynamoDB            │◄──────┘
       │  posts table             │
       │  follows table + GSI     │
       └──────────────────────────┘
```

### Feed Service — Abordagem Ingênua

```python
def get_feed(user_id, cursor, page_size):
    # 1. Busca todos os usuários que user_id segue
    following = follow_table.query(pk=user_id)  # pode ser 10k+

    # 2. Para cada seguido, busca posts recentes
    all_posts = []
    for followed_id in following:
        posts = posts_gsi.query(
            pk=followed_id,
            sk__lt=cursor,
            limit=page_size
        )
        all_posts.extend(posts)

    # 3. Ordena e retorna
    return sorted(all_posts, by=created_at, desc=True)[:page_size]
```

**Problema:** Se você segue 10.000 pessoas → 10.000 queries de banco → impossível em < 500ms.

---

## 🚀 Deep Dives

### Deep Dive 1: Fan-out on Write (Pre-computação do Feed)

**Insight:** Pre-computar o feed quando o post é *criado*, não quando o usuário *lê*.

#### Tabela: `precomputed_feed`

```
PK: user_id (quem receberá no feed)
SK: created_at (ordenação)
Atributos: post_id

Exemplo:
  user_id=alice, created_at=2024-01-15T10:00, post_id=xyz
  user_id=alice, created_at=2024-01-15T09:55, post_id=abc
  user_id=bob,   created_at=2024-01-15T10:00, post_id=xyz
```

#### Fluxo com Fan-out

```
Bob cria um post
  ↓
Post Service salva no posts table
  ↓
Evento assíncrono → Queue (Kafka/SQS)
  ↓
Feed Workers (pool)
  ↓
Para cada seguidor de Bob:
  INSERT precomputed_feed(user_id=seguidor, post_id=post_bob, ...)
  ↓
Quando Alice lê o feed:
  SELECT FROM precomputed_feed WHERE user_id='alice' LIMIT 25
  → Uma única query rápida!
```

#### Trade-offs

| Aspecto | Fan-out on Write | Fan-out on Read (anterior) |
|---|---|---|
| **Leitura** | ✅ O(1) — uma query | ❌ O(N seguindo) queries |
| **Escrita** | ❌ O(N seguidores) escritas | ✅ Uma única escrita |
| **Latência de leitura** | ✅ Muito baixa | ❌ Alta para muitos seguidos |
| **Consistência** | Eventual (worker assíncrono) | Imediata |
| **Problema** | Celebrity problem (Justin Bieber) | Performance de leitura |

#### Estimativa de Custo de Storage

```
200 posts por usuário no feed (cap)
10 bytes por post_id
= 2 KB por usuário

2 bilhões de usuários × 2 KB = 4 TB de storage
→ Completamente viável! Facebook ganha ~$100/usuário/ano nos EUA
```

---

### Deep Dive 2: Celebrity Problem (Justin Bieber Problem)

**Problema:** Justin Bieber tem 100 milhões de seguidores. Quando ele posta, precisamos escrever em 100M entradas de `precomputed_feed`.

```
Justin posts → 100 milhões de escritas simultâneas
→ Workers sobrecarregados
→ DynamoDB rate limiting
→ Feeds demoram horas para atualizar
```

#### Solução Híbrida: Flag de Pre-computação

```
follows table:
  user_following: alice
  user_followed: justin_bieber
  is_precomputed: FALSE  ← !!
```

**Regra:**
- Contas com `< 100K seguidores` → fan-out normal, `is_precomputed = TRUE`
- Contas com `>= 100K seguidores` (celebrities) → `is_precomputed = FALSE`

#### Feed Service com Abordagem Híbrida

```python
def get_feed(user_id, cursor, page_size):
    # 1. Busca do precomputed_feed (maioria dos seguidos)
    precomputed_posts = precomputed_feed.query(user_id, cursor)

    # 2. Busca ao vivo apenas dos celebrities
    non_precomputed_follows = follows.query(
        user_id,
        filter=is_precomputed=False
    )
    celebrity_posts = []
    for celebrity in non_precomputed_follows:
        celebrity_posts.extend(
            posts_gsi.query(celebrity.id, cursor, limit=page_size)
        )

    # 3. Merge e ordenação dos dois conjuntos
    all_posts = precomputed_posts + celebrity_posts
    return sorted(all_posts, by=created_at, desc=True)[:page_size]
```

**Resultado:**
- Usuários normais → fan-out rápido (poucos seguidores)
- Celebrities → apenas os usuários que os seguem buscam ao vivo (N pequeno, pois são poucos celebrities)

---

### Deep Dive 3: Hot Key / Hot Shard Problem

**Cenário:** Justin Bieber posta. Seu post aparece no topo do feed de milhões de usuários. Todos buscam os detalhes do mesmo `post_id` no DynamoDB.

```
DynamoDB particiona dados por chave
Post de Justin → partition X → 1M requests/segundo nessa partição
→ DynamoDB throttle → erros → degradação do serviço
```

#### Solução: Cache com Múltiplas Instâncias

**Solução ingênua (ainda tem hot key):**
```
Feed Service → Cache Redis (sharded por post_id) → DynamoDB
Problema: Cache também vai ter hot key na mesma instância!
```

**Solução correta:**
```
Múltiplas instâncias de cache Redis (sem sharding entre elas):
  Cache Instance 1: cópia do post do Justin
  Cache Instance 2: cópia do post do Justin
  Cache Instance 3: cópia do post do Justin
  ...
  Cache Instance N: cópia do post do Justin

Feed Service → escolhe instância ALEATORIAMENTE
```

```
Resultado:
  - 1M requests → distribuídos entre N instâncias
  - Cada instância recebe 1M/N requests
  - DynamoDB recebe no máximo N requests (cache miss de cada instância)
  - N << 1M → problema resolvido!
```

#### Eviction Policy

```
Cache LFU (Least Frequently Used) + TTL curto:
- Posts virais ficam no cache (alta frequência)
- Posts antigos são evicted automaticamente
- TTL evita dados stale (ex: post editado ou deletado)
```

---

## 🔄 Arquitetura Final

```
                    ┌────────────────────┐
                    │    API Gateway     │
                    └─────────┬──────────┘
                              │
               ┌──────────────┼──────────────┐
               ▼              ▼              ▼
        ┌────────────┐  ┌──────────┐  ┌──────────────┐
        │ Post Svc   │  │Follow Svc│  │  Feed Svc    │
        └─────┬──────┘  └────┬─────┘  └──────┬───────┘
              │               │               │
              ▼               │         ┌─────▼──────────┐
        ┌──────────┐          │         │ Cache (N inst) │
        │  Kafka   │          │         └─────┬──────────┘
        └────┬─────┘          │               │ cache miss
             │                │         ┌─────▼──────────┐
        ┌────▼──────────────────────────────────────────┐
        │              DynamoDB                         │
        │  posts table                                  │
        │  follows table (+ GSI)                        │
        │  precomputed_feed table                       │
        └───────────────────────────────────────────────┘
             │
        ┌────▼──────┐
        │Feed Worker│ (pool escalável)
        │  Pool     │ → escreve no precomputed_feed
        └───────────┘
```

---

## 💡 Padrões e Lições

| Padrão | Aplicação |
|---|---|
| **Fan-out on Write** | Newsfeed, notificações push, propagação de eventos |
| **Fan-out on Read** | Dados em tempo real, celebrities, dados que mudam muito |
| **Hybrid Fan-out** | Sistemas com distribuição bimodal de seguidores |
| **Hot Key com múltiplas réplicas** | Qualquer cache com acesso concentrado |
| **Cursor pagination** | Feeds infinitos, listas que crescem durante navegação |
| **Cap de feed** | Limitar storage sem degradar experiência (200 posts) |

---

## 🎓 Expectativas por Nível

| Nível | Expectativa |
|---|---|
| **Pleno** | Identificar o problema de N queries para N seguidos |
| **Sênior** | Propor fan-out on write com workers assíncronos |
| **Staff** | Celebrity problem, solução híbrida, hot key com múltiplas instâncias |

---

*Arquivo gerado a partir do breakdown da Hello Interview — Design Newsfeed*
