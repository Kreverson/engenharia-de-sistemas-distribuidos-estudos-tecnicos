# 24 — Arquitetura de Sistemas de Recomendação

> Como plataformas como YouTube, TikTok e Netflix escalam a descoberta de conteúdo para bilhões de usuários

---

## O Problema Central

**Objetivo:** dado um usuário, encontrar o conteúdo que ele mais provavelmente vai consumir.

**Escala do desafio:**
- YouTube: 800 milhões de vídeos
- Netflix: dezenas de milhares de títulos com metadados ricos
- TikTok: bilhões de posts

**Abordagem ingênua (que não funciona):**

```
Para cada vídeo no catálogo:
  score = ml_model(user, video)
  
sort(scores) → retornar top-10

Tempo: bilhões de chamadas de ML × milissegundos = semanas ❌
```

---

## A Arquitetura em 3 Estágios

```
[Bilhões de itens]
        ↓
[1. Candidate Generation] → reduz para ~1.000 candidatos
        ↓
[2. Ranking] → aplica ML pesado nos ~1.000
        ↓
[3. Re-ranking] → ajustes de negócio/privacidade
        ↓
[Top-N resultados para o usuário]
```

---

## Estágio 1: Candidate Generation

**Insight chave:** não precisamos avaliar todos os itens com ML. Precisamos de heurísticas rápidas para reduzir o espaço de busca.

### Tipos de Candidate Generators

| Generator | Lógica | Exemplo |
|---|---|---|
| Subscriptions | Canais que o usuário segue | "Novo vídeo do MrBeast" |
| Top-N platform | Vídeos mais assistidos globalmente | "Trending" |
| Similar content | Itens semelhantes ao que o usuário assistiu recentemente | "Você gostou de X, pode gostar de Y" |
| Similar users | O que usuários parecidos com você assistiram | Collaborative filtering |

**Arquitetura:** rodar múltiplos generators em paralelo → fazer union dos resultados → ~1.000 candidatos.

### Candidate Generation com Embeddings

Para generators como "similar content" ou "similar users", precisamos de busca por similaridade — e embeddings são a solução.

**O que é um embedding?**
```
video → função_ml → [0.23, -0.85, 1.2, 0.04, ...] (vetor N-dimensional)

Vídeos semanticamente similares → vetores próximos no espaço N-D
```

**Exemplos de agrupamento:**
- Vídeos de gatos e cachorros → próximos (animais de estimação)
- Vídeos de xadrez e gatos → distantes
- Usuários com padrões de watch history similares → próximos

---

## Vector Databases e Approximate Nearest Neighbor

### O Problema

Dado um vetor de query (ex: embedding do último vídeo assistido), encontrar os K vetores mais próximos em um banco com bilhões de vetores.

**Busca exata:** calcular distância para cada vetor → O(N) × custo de distância = impraticável

**Solução:** Approximate Nearest Neighbor (ANN) — aceita pequena margem de erro para ganho massivo de performance.

### Algoritmo HNSW (Hierarchical Navigable Small Worlds)

```
Grafo hierárquico onde:
- Camada superior: poucos nós, links de longa distância (navegação rápida)
- Camadas inferiores: mais nós, links locais (refinamento)

Query: entra na camada superior, navega rapidamente para região aproximada,
       desce as camadas para refinar → resultado em O(log N)
```

### Vector Databases Populares

| Banco | Tipo | Casos de Uso |
|---|---|---|
| Pinecone | Cloud-managed | Produção, sem ops |
| Weaviate | Open-source | Self-hosted |
| Faiss (Meta) | Library | Integração customizada |
| pgvector | PostgreSQL extension | Já tem Postgres, quer ANN |
| Redis | Built-in vectors | Redis já no stack |

### Fluxo Completo com Vector Search

```
1. Novo vídeo criado → gerar embedding → armazenar no Vector DB
2. Usuário assiste vídeo X → buscar embedding de X no Vector DB
3. ANN query: "top-100 vetores mais próximos de X"
4. Retorna video_ids dos candidatos similares
5. Union com outros generators → input para o Ranking
```

---

## Estágio 2: Ranking

Com ~1.000 candidatos (redução de 1M×), agora é viável aplicar o modelo ML pesado:

```
Para cada candidato:
  score = ml_ranker(user_features, item_features, context_features)

Ordenar por score → top-N candidatos
```

**Features típicas:**
- User: histórico de watch, likes, compartilhamentos, demografia, hora do dia
- Item: título, tags, thumbnail, CTR histórico, duração, creator popularity
- Context: dispositivo, localização, horário, idioma

---

## Estágio 3: Re-ranking

Ajustes pós-ranking por regras de negócio:

| Ajuste | Motivo |
|---|---|
| Boost para conteúdo novo | Exploração — dar chance a itens sem histórico |
| Penalizar creators bloqueados | Privacidade do usuário |
| Filtrar conteúdo sensível | Configurações do usuário |
| Diversidade temática | Evitar filter bubble |
| Posição de conteúdo patrocinado | Monetização |

**Por que Re-ranking separado do Ranking?**

```
Alternativa: incluir essas regras no modelo de ranking
Problema: qualquer mudança de regra requer retreinamento do modelo

Re-ranking pós-hoc: mudanças de negócio sem tocar no ML
→ mais ágil, mais barato, mais auditável
```

### Padrão Geral: "Design for 99%, adjust for 1%"

> Às vezes é mais fácil projetar para o caso comum e ter uma camada final que ajusta os 1% de exceções do que construir um sistema que trata 100% dos casos diretamente.

---

## Gestão dos Candidate Generators

Em sistemas de produção reais, há **dezenas de generators** rodando simultaneamente:

```
[Generator: subscriptions] ──────┐
[Generator: trending]             │
[Generator: similar_content]      ├──→ [Union] → [Deduplicate] → ~1000 candidatos
[Generator: collab_filtering]     │
[Generator: new_content_boost]   ─┘
[Generator: geolocation]
```

**Desafios de engenharia:**
- Manter embeddings atualizados (pipeline de atualização incremental)
- Monitorar qualidade de cada generator (qual contribui mais para engajamento?)
- Adicionar/remover generators sem impactar o resto

---

## Exploration vs. Exploitation

**Dilema fundamental:**

```
Exploitation: mostrar o que sabemos que o usuário vai gostar
  → Experiência confortável, mas filter bubble

Exploration: mostrar algo novo e diferente
  → Pode não engajar, mas expande horizontes e melhora o modelo
```

**Técnicas de balanceamento:**
- **ε-greedy:** X% do tempo explorar, (1-X)% explotar
- **Thompson Sampling:** baseado em incerteza bayesiana
- **UCB (Upper Confidence Bound):** preferir itens com alta incerteza + alta estimativa

---

## Arquitetura Completa

```
[Conteúdo Novo]
     ↓ Embedding Pipeline (batch/streaming)
[Vector DB] ←──────────────────────────────┐
                                            │
[User Request]                              │
     ↓                                      │
[Candidate Generation]                      │
  ├── Query Vector DB (similar content)  ───┘
  ├── Query DB (subscriptions)
  ├── Query Cache (trending)
  └── ... outros generators
     ↓
[~1.000 candidatos]
     ↓
[Ranking Model] (ML inference)
     ↓
[Top-25 ranqueados]
     ↓
[Re-ranking] (regras de negócio)
     ↓
[Top-10 para o usuário]
```

---

## Pontos-Chave para Interviews

1. **Multi-stage é obrigatório** para bilhões de itens — não existe ranking direto
2. **Candidate generation = heurísticas baratas** que aproximam o espaço de busca
3. **Vector databases** resolvem "similar content" de forma escalável
4. **Re-ranking** desacopla regras de negócio do ML — mais manutenível
5. **Exploration/exploitation** é um trade-off real em qualquer sistema de recomendação

---

*Fonte: Hello Interview — Arquitetura de Sistemas de Recomendação*
