# 49 — ML System Design: Detecção de Bots

> Um dos problemas mais ricos de ML System Design: natureza adversarial, label scarcity, class imbalance, e a necessidade de encontrar comportamentos que nunca foram vistos antes.

---

## 1. Framework de ML System Design

```
1. Problem Framing      → Business objective, ML objective, constraints
2. High-Level Design    → Componentes do sistema, onde os modelos se encaixam
3. Data & Features      → Fontes de dados, engenharia de features
4. Modeling             → Arquitetura, loss function, treinamento
5. Evaluation           → Métricas offline e online, AB test
6. Deep Dives           → Tópicos avançados, trade-offs específicos
```

---

## 2. Problem Framing — Ler nas Entrelinhas

### Clarificando o problema

Ao receber o enunciado "Projete um sistema de detecção de bots para rede social", faça perguntas que revelam a natureza real do problema:

| Pergunta | Resposta típica | Implicação |
|---|---|---|
| "Qual a prevalência de bots?" | "50% do tráfego, mas apenas 1% passa de heurísticas básicas" | Classe muito desbalanceada |
| "Como os bots se comportam?" | "São evasivos, mudam padrões quando detectados" | Problema adversarial |
| "Como obtemos labels?" | "Equipe de investigadores, mas é caro e demorado" | Label scarcity |
| "O que fazemos com bots detectados?" | "Banimos, mas bots navegam nos recursos de apelação" | Appeals não são signal confiável |
| "Qual a escala?" | "500M DAU" | Inference em bilhões de eventos/dia |

### Business Objective — Formulação Correta

```
❌ "Maximizar o número de bots detectados"
   → Incentiva caçar os mais óbvios repetidamente
   → Não considera qualidade do sistema

❌ "Maximizar acurácia do modelo"
   → Business não se importa com F1 score diretamente
   → Pode levar a soluções desconectadas do impacto real

✅ "Minimizar o impacto da atividade de bots em usuários legítimos,
    sujeito a restrição de falsos positivos ≤ 5%"
   → Conecta ao impacto real de negócio
   → Inclui o counter-metric (não banir usuários reais)
   → Define o threshold de calibração
```

### ML Objective

```
Bot Classifier:
  Input:  features do usuário + sequência de ações
  Output: probabilidade calibrada P(bot | features)
  
  Calibrado: P(bot) = 0.9 significa que de 90 usuários com esse score,
             ~90 são bots (não apenas "é mais provável que seja bot")
  
  Por que calibração importa:
    Com P(bot) calibrado e FPR constraint de 5%:
    → Encontrar threshold T tal que Precision = 95% (5% FPs)
    → Banir automaticamente apenas acima de T
```

---

## 3. High-Level Design

```
[Bot Classifier]
      │
      ├── Score alto (> threshold_ban)    → Banir imediatamente
      ├── Score médio (entre thresholds)  → Aplicar limitações (rate limits)
      └── Score baixo (< threshold_limit) → Usuário normal
      
      │ (quando bane)
      ▼
[Vector DB / ANN Index]
  Armazena embeddings do conteúdo do usuário banido
  → Feature: "distância ao conteúdo removido mais próximo"
  → Captura bots que postam conteúdo similar ao de bots anteriores

      │
      ▼
[Unsupervised Bot Detection]
  Detecta bots que ainda não foram vistos
  → Isolation Forest / Autoencoder → anomaly score
  → Amostras de alto score → enviadas para labelamento
  → Labels novas alimentam o classificador principal
  
      │
      ▼
[Labeling Pipeline]
  Amostras de baixo score também enviadas (random sampling)
  → Permite medir recall do classificador
  → Previne "blind spots" onde o model nunca aprende
```

---

## 4. Data e Features

### Fontes de Labels (do mais para menos confiável)

```
1. Ground Truth Labels (investigadores humanos):
   Mais confiável, mas escasso e caro
   Uso: fine-tuning final, calibração

2. Weak Supervision Labels:
   - Reports de usuários (conta ou conteúdo)
   - Resultados de apelações
   - Bloqueios por outros sistemas
   Uso: pré-treinamento, treinamento com mais dados

3. Network-based Labels (sem supervisão):
   - Clusters de IP address
   - Clusters de comportamento similar
   Uso: labeling adicional, features de grafo

4. Unlabeled User Activity (99% é legítimo!):
   - Sequências de ações de usuários
   Uso: pré-treinamento de modelos de sequência
```

### Feature Engineering por Categoria

#### Padrões de Atividade

```python
features = {
  # Frequência
  "posts_per_hour": stats.mean(hourly_post_counts),
  "posts_per_hour_variance": stats.var(hourly_post_counts),
  
  # Ritmo circadiano
  "hours_covered_per_day": len(set(hour for ts in timestamps)),
  "sleep_absence_score": 1 - (nighttime_hours / total_hours),
  
  # Burstiness
  "max_posts_in_5min": max(5_minute_buckets),
  "inter_post_time_cv": std(inter_post_times) / mean(inter_post_times),
}
```

#### Sinais de Conteúdo

```python
features = {
  # Padrões de digitação
  "backspace_ratio": backspaces / total_keystrokes,  # bots raramente erram
  "typing_speed": chars_typed / time_seconds,  # bots tendem a ser constantes
  "copy_paste_ratio": paste_events / total_input_events,
  
  # Qualidade linguística
  "typo_rate": typos / total_words,
  "slang_usage": slang_words / total_words,  # bots usam menos gírias
  "repeated_content_ratio": exact_duplicates / total_posts,
}
```

#### Sinais de Rede Social

```python
features = {
  # Crescimento suspeito
  "follower_growth_rate": followers_gained / days_old,
  "follower_to_following_ratio": followers / max(1, following),
  
  # Padrões de rede
  "mutual_follow_ratio": mutual_follows / total_follows,
  "bot_cluster_proximity": distance_to_nearest_bot_cluster,  # grafo
}
```

#### Features em Tempo Real (stream)

```python
features = {
  # Acumuladores em tempo real
  "content_reports_7d": sum(reports in last 7 days),
  "rate_limits_applied_24h": count(rate_limit_events today),
  "failed_captchas_30d": count(failed_captcha_events),
}
```

---

## 5. Modelagem — Late Fusion Architecture

### Por que não um modelo único gigante?

```
❌ "Mega model" (transformer grande):
   - Content pode ser facilmente alterado por bots adversariais
   - Overfitting em padrões de conteúdo específicos
   - Cara inferência para escala de bilhões de eventos

✅ Late fusion de múltiplas modalidades:
   - Grafo (IP, rede social) → difícil de falsificar
   - Sequência de ações → revela padrões comportamentais
   - Features densas → rápido, robusto a mudanças de conteúdo
```

### Arquitetura

```
INPUTS:
┌─────────────┐  ┌──────────────────┐  ┌────────────────────┐
│ Graph       │  │ Action Sequence  │  │ Dense/Counter      │
│ Features    │  │ Features         │  │ Features           │
│             │  │                  │  │                    │
│ [IP cluster]│  │ [visited_profile,│  │ [reports_7d: 5,    │
│ [follow     │  │  followed_user,  │  │  rate_limits: 2,   │
│  graph]     │  │  posted_content] │  │  account_age: 14d] │
└──────┬──────┘  └────────┬─────────┘  └─────────┬──────────┘
       │                  │                       │
       ▼                  ▼                       ▼
┌──────────────┐  ┌───────────────┐  ┌───────────────────┐
│ GraphSAGE    │  │ GRU           │  │ MLP               │
│ (Inductive)  │  │ (Sequence)    │  │ (Dense features)  │
└──────┬───────┘  └───────┬───────┘  └─────────┬─────────┘
       │                  │                     │
       └──────────────────┴─────────────────────┘
                          │
                   [Attention Layer]
                          │
                       [MLP]
                          │
                      [Sigmoid]
                          │
                    P(bot) ∈ [0,1]
```

### Branch de Grafo: GraphSAGE

**Por que não outros Graph Neural Networks?**
```
Maioria das GNNs requer o grafo completo no treino (transductive learning):
  → Não funciona para novos usuários (cold start)
  
GraphSAGE é inductive:
  → Aprende a agregar features dos vizinhos
  → Funciona para nós não vistos no treinamento
  → Permite inferência em tempo real para novas contas
```

**Como funciona:**
```
Para user_id = alice:
  1. Colete K-hop neighborhood (vizinhos no grafo de IP/follow)
  2. Para cada vizinho, obtenha suas features
  3. GraphSAGE aprende a agregar essas features
  4. Output: embedding rico que codifica o "contexto social" da alice
```

### Branch de Sequência: GRU (Gated Recurrent Unit)

**Por que GRU e não Transformer?**
```
Transformer: 100M+ parâmetros → GPU obrigatório para inferência
GRU: 1-10M parâmetros → pode rodar em CPU

Com 500M DAU × N ações/dia = bilhões de eventos:
→ Custo de inference com Transformer seria proibitivo
→ GRU retém informação relevante com muito menos compute
```

**Sequência de input:**
```
[
  ("view_profile", user_id=X, t=1000),
  ("like_post", post_id=Y, t=1005),
  ("follow_user", user_id=Z, t=1010),
  ("post_content", "Buy crypto now!", t=1015),
  ...
]
→ Tokenizado e passado pelo GRU
→ Output: hidden state que captura padrão comportamental
```

### Pré-treinamento com Self-Supervised Learning

**Problema:** apenas milhares de labels, mas precisamos de modelo rico.

**Solução: pré-treinamento auto-supervisionado antes do fine-tuning:**

```
Branch de Grafo — Link Prediction:
  Task: "esse edge (conexão) existe no grafo?"
  Durante treinamento: remove arestas aleatoriamente
  Modelo aprende a prever se deveriam existir
  → Aprende representações ricas do grafo sem labels de bot

Branch de Sequência — Next Action Prediction:
  Task: "qual a próxima ação do usuário?"
  Input: [A1, A2, A3, ?, A5]
  Modelo aprende a prever A4
  → Aprende padrões comportamentais sem labels de bot

Fine-tuning final:
  Congela camadas inferiores
  Treina apenas as camadas superiores com os raros labels de bot
  → Transfer learning: aproveita representações do pré-treinamento
```

### Multi-task Learning

```
Uma mesma rede treinada para múltiplos tasks:

         [Bot or Not?]  [Account Reported?]  [Content Violation?]
                ↑              ↑                     ↑
              [Head 1]       [Head 2]              [Head 3]
                   ↖            ↑            ↗
                         [Shared Backbone]
                                ↑
                         [User Features]

Benefício: "Account Reported" tem 100x mais labels que "Bot or Not"
→ Modelo aprende features que se correlacionam com bots
  usando os labels de reports como proxy
→ Fine-tuning final com labels de bot é mais eficiente
```

---

## 6. Calibração e Thresholds

```
Modelo raw:        score = 0.73 → "73% provável que seja bot"
Modelo calibrado:  de 100 usuários com score 0.73, exatamente 73 são bots

Por que calibração importa:
  Business constraint: "5% de falsos positivos aceitáveis"
  = Precision ≥ 95%
  
  Com modelo calibrado:
    Find threshold T such that Precision(T) = 0.95
    → Tudo acima de T → ação automática
    → Entre threshold_limit e T → soft action (rate limiting)
    → Abaixo de threshold_limit → usuário normal
```

---

## 7. Inference em Escala

### Two-Stage Inference

```
Stage 1: Filter Model (CPU, muito rápido)
  Logistic regression simples
  Input: apenas features densas/contadores
  Output: P(bot classifier tomaria ação)
  
  Se score baixo → descartar, não chamar o bot classifier
  Se score alto → passar para Stage 2

Stage 2: Bot Classifier (GPU, mais preciso)
  Modelo completo com grafo + sequência + features densas
  
Ganho: 90%+ dos eventos descartados no Stage 1
→ Custo de GPU reduzido em ~10x

Risco: bots que aprendem a enganar o Stage 1 nunca chegam ao Stage 2
→ Mitigação: random sampling de 1% para Stage 2 independente do Stage 1
```

### Caching de Embeddings

```
Graph embeddings do usuário:
  → Muda apenas quando vizinhança no grafo muda
  → Cache por 1 hora: reusa embedding sem recalcular
  → Atualiza quando evento de follow/unfollow ocorre

Sequência de ações:
  → GRU é stateful: pode atualizar incrementalmente
  → Não recalcula desde o início a cada nova ação
```

---

## 8. Evaluation

### Métricas Offline

```
❌ ROC-AUC: insensível ao desbalanceamento de classes
   (1% bots → modelo que sempre diz "não bot" tem ROC-AUC ~0.5)

✅ PR-AUC (Precision-Recall Area Under Curve):
   Foca na classe positiva (bots)
   Não é inflado pela classe majoritária (não-bots)

✅ Precision @ 95% recall: corresponde diretamente ao business constraint

Temporal validation (crítico para problemas adversariais):
  Treino: dados de t=0 até t=6 semanas
  Teste:  dados de t=6 semanas até t=8 semanas
  
  Por que não hold-out aleatório?
  → Bots mudam comportamento com o tempo
  → Modelo precisa generalizar para NOVOS padrões
  → Validação temporal simula o ambiente real de produção
```

### AB Test Online

```
Grupos:
  Control: modelo atual
  Treatment: novo modelo

Métricas primárias (alinhadas ao business objective):
  - Reports de spam enviados por usuários (proxy de impacto de bots)
  - Prevalência estimada de bots (via important sampling)

Counter-metrics (garantir que não estamos banindo usuários reais):
  - Taxa de falsos positivos (estimada via amostragem + labeling)
  - Taxa de apelações bem-sucedidas

Desafio de contamination:
  Bots no grupo Treatment interagem com usuários no grupo Control
  → Grafo de interação contamina os grupos
  → Aceitar alguma contaminação ou usar cluster-based assignment
```

---

## 9. Anomaly Detection — Encontrando o Desconhecido

### O problema que a anomaly detection resolve

```
Bot classifier supervisionado:
  → Muito bom em detectar padrões que já viu
  → Cego para novos tipos de bots que nunca foram vistos

Necessário: sistema que detecta comportamentos incomuns
  mesmo sem labels de como bots "deveriam" parecer
```

### Isolation Forest

**Intuição:** itens anômalos são mais fáceis de isolar.

```
Imagine uma floresta de árvores com splits aleatórios:
  Usuário normal: requer muitos splits para ficar sozinho
    → profundidade alta → score baixo de anomalia

  Usuário anômalo: fica sozinho rapidamente
    → profundidade baixa → score alto de anomalia

Score de anomalia = média da profundidade inversa em N árvores
```

**Por que é bom para este caso:**
```
✅ Funciona em CPU (sem GPU)
✅ Muito rápido (apenas if-statements)
✅ Não requer labels
✅ Funciona bem com features tabulares

Limitação: aprende hiperplanos ortogonais
→ Não captura relações não-lineares complexas
```

### Autoencoder

**Intuição:** hard to compress = anômalo.

```
[User Features (1000-dim)]
         ↓ (encoder)
[Embedding (64-dim)]      ← compressão
         ↓ (decoder)
[Reconstruction (1000-dim)]
         
Loss = ||original - reconstruction||²

Treinamento: minimizar erro de reconstrução em usuários normais
  → Modelo aprende a comprimir e descomprimir usuários normais eficientemente

Inference: calcular erro de reconstrução para novo usuário
  Alto erro → não é similar aos usuários normais → potencial bot
```

**Complementar ao Isolation Forest:**
```
Isolation Forest: detecta outliers em espaço de features tabular
Autoencoder:     detecta outliers em espaço de features não-linear

Ensemble: média dos dois scores → ranking de anomalias mais robusto
```

### Pipeline de Anomaly Detection → Novos Labels

```
1. Score de anomalia calculado para todos os usuários
2. Amostrar usuários com alto score → enviar para investigadores
3. Investigadores criam labels (bot vs não-bot)
4. Novos labels adicionados ao training set do bot classifier
5. Modelo retreinado com distribuição atualizada
```

---

## 10. Resumo dos Conceitos-Chave

| Conceito | Aplicação em Bot Detection |
|---|---|
| **Adversarial ML** | Bots mudam comportamento → modelo deve adaptar |
| **Class Imbalance** | 1% bots → PR-AUC em vez de ROC-AUC |
| **Label Scarcity** | Pré-treinamento auto-supervisionado + multi-task learning |
| **GraphSAGE** | Representa contexto social do usuário (inductive) |
| **GRU** | Captura padrões comportamentais em sequências de ações |
| **Calibração** | P(bot) real permite definir threshold para FPR desejado |
| **Two-stage inference** | Stage 1 leve filtra 90%+; Stage 2 pesado processa o restante |
| **Isolation Forest** | Detecta anomalias em features tabulares sem labels |
| **Autoencoder** | Detecta anomalias por dificuldade de compressão |
| **Important Sampling** | Amostragem eficiente para estimar recall com labels caros |

---

*Tópicos relacionados: `37_ml_system_design_content_moderation.md`, `24_sistemas_recomendacao.md`, `26_estruturas_dados_big_data.md`*
