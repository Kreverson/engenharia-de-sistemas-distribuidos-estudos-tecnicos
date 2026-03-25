# 37 — ML System Design: Detecção de Conteúdo Nocivo

> Baseado no breakdown da Hello Interview com Stefan (ex-Senior Manager de Applied ML no Meta, org "Dangerous Content"). Cobre o framework completo de ML System Design para entrevistas, aplicado ao problema de moderação de conteúdo.

---

## 📌 Por que ML System Design é diferente?

```
SWE System Design:
  → Databases, APIs, caching, sharding, latência
  → "Como eu implemento isso?"

ML System Design:
  → Dados, features, modelos, avaliação, feedback loops
  → "Como eu aprendo isso automaticamente?"
```

O papel de ML Engineer varia muito entre empresas:
- **Research-focused:** papers, novos modelos, poucas entregas de produto
- **Full-stack ML (Meta):** pesquisa aplicada + deploy em produção

Entenda o papel antes da entrevista.

---

## 🗺️ Framework: 6 Etapas de ML System Design

```
1. Problem Framing    → O que realmente estou resolvendo?
2. High-Level Design  → Como as peças se encaixam?
3. Data & Features    → Que dados uso? Como representar?
4. Modeling           → Qual arquitetura de modelo?
5. Inference & Eval   → Como avalio e coloco em produção?
6. Deep Dives         → Questões específicas do entrevistador
```

---

## 🎯 Etapa 1: Problem Framing

### Contexto
Design de um sistema para detectar conteúdo nocivo no Facebook.

### Clarificações essenciais

```
Q: Vídeo ou texto/imagens?
A: Apenas texto e imagens (sem vídeo)

Q: Que tipo de conteúdo nocivo?
A: Tudo (nudez, violência, discurso de ódio, etc.)

Q: O que acontece quando encontramos conteúdo nocivo?
A: Deletar (>95% confiança) ou demover (abaixo de 95%)

Q: Scale?
A: 1 bilhão de posts/dia, apenas 1% são nocivos
```

### Business Objective — A diferença entre pleno e staff

**Objetivo ingênuo:**
```
"Remover o máximo de conteúdo nocivo possível"
❌ Ignora o custo de falsos positivos (posts legítimos removidos)
```

**Objetivo nível sênior:**
```
"Maximizar remoções com precisão ≥ 95%"
✅ Guarda rail de precisão protege usuários legítimos
```

**Objetivo nível staff:**
```
"Minimizar views de conteúdo nocivo confirmado"
✅ Reconhece que impacto = nocividade × alcance
✅ Muda a estratégia: não preciso remover antes de qualquer view,
   mas preciso remover ANTES DE MUITAS views
✅ Permite ação tardia se o post não viralizou
✅ Justifica priorizar posts com alto potencial de alcance
```

### ML Objective

```
Classificação binária:
  Input: post (texto + imagens) em um dado momento
  Output: {harmful=true, harmful=false}
  → Tomamos ação downstream baseada no score de confiança
```

---

## 🏛️ Etapa 2: High-Level Design

```
Post criado / visualizado
         ↓
  ┌──────────────────┐
  │ Classification   │
  │ Model            │
  └────────┬─────────┘
           ↓
  ┌──────────────────┐
  │  Calibration     │ → ajusta score para probabilidade real
  └────────┬─────────┘
           ↓
    score ≥ 95%? → Deletar
    score < 95%? → Demover (reduzir distribuição no feed)
```

**Nota:** Alta disponibilidade é assumida. O foco do design é o pipeline de ML, não a infra de banco.

---

## 📊 Etapa 3: Data & Features

### Dados de Treinamento

**Apenas dados rotulados não é suficiente:**

```
Dado              Volume         Observação
─────────────────────────────────────────────────────────
Labels de experts 50.000         Poucos, mas alta qualidade
NSFW públicos     Milhões        Fotos/vídeos, sem contexto de post
User reports      Bilhões de     Parcial: nem todo nocivo é reportado,
                  posts          nem todo reportado é nocivo
Dados não rotuladosBilhões       Podem revelar padrões via self-supervision
```

**Como usar dados com supervisão parcial:**
- Reports/reações negativas como proxy de "provável nocivo"
- Normalizar por views: 25 reações negativas em 50 views ≠ 25 em 1M views

### Categorias de Features

#### 1. Content Features (óbvias, mas insuficientes)
```
- Texto do post: embeddings do conteúdo
- Imagens do post: embeddings visuais
```

#### 2. Behavioral Features (diferenciadoras)
```
- Taxa de reações negativas (normalizada por views)
- Frequência de reports (normalizada por views)
- Shares com legendas específicas
- Comentários com sentimento negativo
```

**Atenção à normalização:**
```
❌ raw: 1000 reações negativas em 1M views = não nocivo
❌ raw: 25 reações negativas em 50 views = provável nocivo

✅ ratio: negative_reactions / total_reactions
          → independente do alcance absoluto
```

#### 3. Creator Features (preditivas)
```
- User embedding: representação densa do histórico do usuário
- % de posts anteriores sinalizados como nocivos
- Idade da conta
- Métricas anti-spam/anti-compromise

"Vovó que posta fotos de gato há 5 anos" vs
"Conta nova de 2 dias com histórico suspeito"
```

---

## 🤖 Etapa 4: Modeling

### Abordagem 1: Classifiers Independentes (simples, mas fraca)

```
Texto → BERT → score_texto    ┐
Imagem → CNN → score_imagem  ├── MAX(scores) → threshold → decisão
Comportamento → LR → score_b ┘

Problema:
- Não captura interações entre modalidades
- Texto inócuo + imagem inócua podem juntos ser nocivos
- Baixa performance
```

### Abordagem 2: MLP sobre embeddings (intermediária)

```
Texto → BERT → embedding_512
                               ┐
Imagem → ViT → embedding_512   ├── Concatenar → MLP → score
                               ┘
Comportamento → Linear → dense_features

Problema:
- MLP não aprende bem interações entre modalidades
- 50K exemplos insuficientes para treinar bem
```

### Abordagem 3: Transformer Multimodal (recomendada para Staff)

```
Arquitetura moderna:
  - Imagens representadas como "patches" (como tokens de texto)
  - Texto tokenizado normalmente
  - Transformer joint aprende atenção cruzada entre modalities

  Texto tokens: [taylor] [swift] [novo] [álbum]
  Image patches: [patch_1] [patch_2] ... [patch_196]

  → Transformer atende a qualquer token/patch
  → Captura: "imagem de nudez + texto inocente" = nocivo
  → Captura: "imagem de arma + texto sobre segurança" = legítimo (contexto)
```

**Otimização para Inference:** Features comportamentais (que mudam ao longo do tempo) são fundidas na última camada (late fusion), permitindo:
```
Cacheamento:
  - Embeddings de texto/imagem → calculados uma vez ao criar o post
  - Features comportamentais  → atualizadas conforme o post ganha views

Re-classificação barata:
  embedding_cached + novas_features → nova_decisão
  (sem recalcular embeddings caros)
```

---

## 🎓 Multi-task Learning para Resolver Escassez de Dados

```
Problema: 50K labels não são suficientes para treinar um transformer grande.

Solução: Aprender múltiplas tarefas simultaneamente:

  Task 1 (principal):  harmful vs benign           → 50K labels
  Task 2 (auxiliar):   vai ser reportado?           → bilhões de exemplos!
  Task 3 (auxiliar):   qual tipo? nudez? violência? → explicabilidade

Modelo:
  Transformer → representação compartilhada
             → head 1: P(harmful)
             → head 2: P(reported)
             → head 3: P(nudity) | P(violence) | ...

Hipótese: Posts que são reportados correlacionam com harmful
→ Task 2 amplifica dados de treinamento enormemente
→ Task 3 dá interpretabilidade: "score 0.9 nocivo, 0.89 nudez"
```

**Loss function:**
```python
total_loss = (
    α × loss_harmful +      # task principal
    β × loss_reported +     # task auxiliar
    γ × loss_category       # task interpretabilidade
)

# α > β > γ (priorizar task principal)
# Incluir log(views) como peso: erros em posts virais têm mais peso
```

---

## ⚡ Etapa 5: Inference & Evaluation

### Arquitetura Multi-Estágio (reduzir custo de GPU)

```
PROBLEMA: 1B posts/dia × custo de transformer = absurdo

SOLUÇÃO: Filtrar antes de usar o modelo pesado

Todos os posts (1B/dia)
         ↓
  ┌─────────────────────┐
  │ Lightweight Classifier│ ← LightGBM ou MLP pequeno
  │ (CPU, rápido)        │   Treinado com destilação do transformer
  └─────────┬───────────┘
            │
    score > threshold?
    ↓ sim (top 10%)         ↓ não (90% ignorados)
  ┌──────────────────┐
  │Heavy Transformer │ ← GPU, caro, mas só em 10% dos posts
  └──────────────────┘
```

**Distilação (Teacher-Student):**
```
Teacher (transformer grande) → prediz scores para todos os posts
Student (modelo leve) → aprende a aproximar os scores do teacher

Resultado: Student filtra o que o Teacher removeria,
sem o custo do Teacher em todos os posts
```

### Calibração de Score

```
Transformer output: score bruto 0.87
≠ P(harmful) = 0.87

Calibração (Isotonic Regression):
  Experts rotulam posts em cada faixa de score
  Aprende função: score_bruto → probabilidade_calibrada

Após calibração:
  score_calibrado = 0.95 significa "95% dos posts com esse score são nocivos"
  → ação automática justificável pelo negócio
```

### Avaliação Online e Offline

**Offline metrics:**
```
❌ ROC-AUC: satura em dados muito desbalanceados (1% nocivo)
✅ PR-AUC: área sob curva Precision-Recall (melhor para desbalanceamento)
✅ Recall @ Precision=95%: "dado que quero 95% de precisão, quanto recall consigo?"
```

**Online metrics (A/B test):**
```
Objetivo: minimizar views de conteúdo nocivo

Otimização de labels:
  Modelos A e B: quando CONCORDAM → nenhuma diferença
  Modelos A e B: quando DISCORDAM → labele apenas esses casos!
  → 10x menos labels necessários para mesmo poder estatístico
```

**Amostragem Importância (Importance Sampling):**
```
Problema: 99% dos posts são benignos → pagar para rotular coisas óbvias

Solução: Amostrar com probabilidade proporcional ao score do modelo
  → Dedique orçamento de labeling ao conteúdo ambíguo (scores médios)
  → Conteúdo claramente benignos (score 0.01): raramente rotular
```

---

## 🔍 Deep Dives

### User Embeddings: Como treinar?

**Abordagem ingênua (overfitting):**
```
Incluir user_id como feature categórica no modelo
→ Memoriza usuários do treino
→ Não generaliza para novos usuários
→ Frágil a mudança de comportamento
```

**Graph Neural Networks (melhor):**

```
Social Graph:
  alice ── amigo ── bob ── segue ── justin
  alice ── grupo ── "gaming"
  alice ── interage ── post_xyz

GNN Inductive (GraphSAGE):
  Para cada usuário: agrega features dos vizinhos
  → Embedding não é lookup table → generaliza para novos usuários
  → Reagrega com novos vizinhos periodicamente

  embedding_alice = f(features_alice, mean(features_vizinhos_alice))
```

### Adversarial Creators

```
Problema: Criador testa o que é removido → adapta o conteúdo

Exemplo:
  "gun" → removido
  "g-u-n" → não removido (tokenização word-level falha)

Soluções:
  1. Byte-level tokenization: "g-u-n" → bytes [g,-,u,-,n]
     → resistente a ofuscação de caracteres

  2. Data augmentation: gerar variações ofuscadas dos exemplos positivos
     → modelo aprende a generalizar

  3. Adversarial training: incluir exemplos de adversários no treino
```

---

## 📊 Resumo de Trade-offs

| Decisão | Opção A | Opção B | Escolha |
|---|---|---|---|
| **Fusão de modalities** | Early (transformer conjunto) | Late (MLP sobre embeddings) | Early para interações; Late para features dinâmicas |
| **Dados comportamentais** | No transformer | Última camada (late fusion) | Late → permite cache de embeddings |
| **Multi-task** | Single task | Multi-task | Multi-task → mais dados efetivos |
| **Threshold** | Fixo 95% | Calibrado por score | Calibrado → mais preciso |
| **Inference** | Um modelo | Two-stage (light + heavy) | Two-stage → reduz custo 10x |

---

## 🎓 Expectativas por Nível

| Nível | Expectativa |
|---|---|
| **Pleno** | Trabalhar o pipeline end-to-end, boa seleção de features, métricas básicas |
| **Sênior** | Profundidade em modeling, avaliação offline/online, considerações de escala |
| **Staff** | Business objective sofisticado (views-aware), multi-task learning, adversarial robustness, insights originais que o entrevistador não esperava |

---

*Arquivo gerado a partir do breakdown da Hello Interview — ML System Design: Content Moderation*
