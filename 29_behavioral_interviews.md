# 29 — Entrevistas Comportamentais (Behavioral Interviews)

> Como se preparar, apresentar experiências e demonstrar o nível certo de senioridade

---

## Por que Behavioral Interviews Importam

Engenheiros frequentemente subestimam a importância das entrevistas comportamentais, achando que "a experiência fala por si". Mas há razões concretas para levá-las a sério:

1. **Desenvolve você como engenheiro:** ao refletir sobre o que funcionou e por quê, você internaliza padrões que aplica no trabalho seguinte.

2. **Preditor de desempenho:** performance passada é o melhor preditor de comportamento futuro — é isso que o entrevistador está medindo.

3. **Determinante de nível:** especialmente para Senior → Staff, as entrevistas comportamentais diferenciam os candidatos mais do que o system design.

> *"No Meta, um Staff engineer e um Senior engineer podem resolver o mesmo system design de forma similar — a diferença clara está no comportamento."*

---

## O que o Entrevistador Está Avaliando

### Sinais (Signals) por Nível

Cada sinal avaliado em uma escala que varia com o nível:

| Sinal | Nível Júnior | Nível Staff |
|---|---|---|
| **Iniciativa** | "Acabei minhas tarefas, fui perguntar ao tech lead o que fazer" | "Vi problema crônico na empresa, estruturei e liderei a solução" |
| **Handling Ambiguity** | Solicitar clareza quando a tarefa não está definida | Criar estrutura onde não existe nenhuma |
| **Perseverança** | Não desistir de um bug difícil | Continuar um projeto de 6 meses com múltiplos pivots |
| **Resolução de Conflitos** | Discordância técnica com colega | Alinhar múltiplos times com visões opostas |
| **Growth Mindset** | Receber e aplicar feedback | Construir cultura de feedback no time |
| **Comunicação** | Escrever docs claras | Influenciar sem autoridade, comunicar para executivos |

### Sinais Comuns Avaliados

- Iniciativa e proatividade
- Lidar com ambiguidade
- Perseverança sob pressão
- Resolução de conflitos
- Crescimento e aprendizado (growth mindset)
- Comunicação escrita e verbal
- Liderança técnica e influência

---

## Framework CARL (preferido)

Substitui o STAR (Situation-Task-Action-Result) com uma estrutura mais eficiente:

```
C → Context (Contexto — substitui Situation + Task combinados)
A → Action (Ações específicas que VOCÊ tomou)
R → Result (Resultado com impacto mensurável)
L → Learnings (O que você aprendeu — diferencial sênior)
```

### Por que CARL é melhor que STAR?

- Elimina a separação artificial entre Situation e Task
- Adiciona Learnings — demonstra maturidade e reflexão
- Encoraja chegar mais rápido às ações (o que o entrevistador quer ouvir)

---

## Preparação: O Brag Document

### O que é

Um documento pessoal (não para o entrevistador) com todos os seus projetos e conquistas, organizado para facilitar o preparo.

### Como construir

**Perguntas para responder:**

```
1. Qual foi o maior impacto para o usuário que você gerou?
2. Que métrica mensurável você moveu? (tempo de loading, receita, custo, engagement)
3. Qual projeto mais demorou e por quê?
4. Onde você sentiu mais orgulho do resultado?
5. Onde você cometeu um erro importante e o que aprendeu?
```

### Eixos de Avaliação dos Projetos

Ranqueie seus projetos em 3 dimensões:

| Eixo | Pergunta |
|---|---|
| **Envolvimento pessoal** | Quanto você foi o protagonista? |
| **Impacto no negócio** | Qual o efeito mensurável? |
| **Escopo** | Quantas pessoas/sistemas/meses envolveu? |

Escolha os projetos com **alta pontuação em múltiplos eixos**.

---

## Estruturando Respostas

### Contexto: quanto é suficiente?

Engenheiros experientes reconhecem os **arquetipos** de projetos:
- "Feature demandada pelo cliente"
- "Problema de performance/reliability"
- "Conflito técnico entre times"

Não explique o básico. Vá direto ao que é único no seu caso.

> **Dica:** se o entrevistador precisar de mais contexto, ele vai perguntar. Não adiante o óbvio.

### As ações são o coração

```
❌ Errado: "Nós construímos a feature e subimos para produção."
✓ Correto: "Eu liderei as decisões de arquitetura, convenci o time de segurança 
            a aprovar a abordagem X argumentando Y, e implementei o módulo Z 
            que foi o gargalo crítico."
```

**Ações que demonstram senioridade:**
- Tomada de decisão com incerteza
- Influência sem autoridade direta
- Antecipação de riscos e mitigação
- Coordenação cross-team

### Learnings: mostre reflexão

| Nível | Tipo de Learning |
|---|---|
| Júnior | "Aprendi a sempre falar com o designer antes de codar" |
| Sênior | "Percebo agora que não definimos métricas de sucesso no início — isso causou scope creep" |
| Staff | "Este projeto revelou um problema cultural no time X que vou endereçar em Q3" |

### Seja autêntico sobre erros

```
Não admitir erros → entrevistador não acredita em você ❌
Admitir erros + mostrar aprendizado → sinal de maturidade ✓

Cuidado: a gravidade do erro importa
  "Não consultei o designer" → ok para junior, problema para senior
  "Subi para 100% sem canary deploy" → sinal de imaturidade técnica
```

---

## As 3 Perguntas que Você Vai Receber

### 1. "Tell me about yourself" (apresentação)

Framework em 3 partes (2 minutos):

```
Parte 1 — Identidade profissional + toque pessoal:
"Sou engenheiro backend com 7 anos de experiência, especializado em 
sistemas distribuídos de alta escala. Sou apaixonado por problemas 
de performance."

Parte 2 — Lista de conquistas (não histórico cronológico):
"No meu papel atual, reduzi o tempo de carregamento da homepage em 
60% através de otimização de queries e caching. Antes disso, liderei 
a migração de monolito para microserviços que suporta 10× o volume."

Parte 3 — Foward-looking (passa a bola):
"Estou buscando uma oportunidade onde possa liderar times técnicos 
e ter impacto no produto — é exatamente o que essa posição oferece."
```

**Não faça:** histórico cronológico ("comecei como estagiário, depois fui para...").

### 2. Projeto favorito / mais impactante

Use CARL completo. Escolha o projeto com maior pontuação nos 3 eixos. Prepare para 5-7 minutos.

### 3. História de conflito

Quase certa em qualquer behavioral interview.

**Arquétipos comuns:**
- Conflito com arquiteto sobre decisão técnica
- Conflito com PM sobre prioridade/escopo
- Conflito com colega sobre abordagem de implementação

**Estrutura CARL para conflitos:**
- C: contexto do conflito (rapidamente)
- A: como você navegou (escutou, construiu argumento, buscou consenso)
- R: como foi resolvido
- L: o que você levaria para o próximo conflito

---

## Armadilhas Comuns

| Erro | Correção |
|---|---|
| Muito contexto, pouca ação | Frontload o que você fez, contexto por demanda |
| "Nós fizemos" sem "eu fiz" | Especifique sua contribuição individual |
| Não ter aprendizados | Prepare reflexões para cada projeto |
| Conto de fadas (sem obstáculos) | Inclua dificuldades — dá credibilidade |
| Sem números | "Reduzi latência em 40%" > "melhorei performance" |
| Explicar para leigo | Entrevistador tem experiência — vá direto ao ponto técnico |

---

## Perguntas ao Final da Entrevista

Evite: "Como é um dia típico de trabalho?"  
Use perguntas que demonstrem que você pesquisou:

```
"Vi que vocês anunciaram [feature X] — quais foram os maiores 
desafios técnicos de implementá-la?"

"Como o time aqui lida com trade-offs entre velocity e qualidade técnica?"

"O que diferencia as pessoas que crescem rapidamente nesta equipe?"

"Como você [para o hiring manager] tem ajudado engenheiros a 
alcançar o próximo nível?"
```

---

## Team Match (Meta, Google e empresas similares)

Reuniões com hiring managers que **parecem** apenas informativas mas **são avaliações**.

**O que demonstrar:**
- Você pesquisou o time/produto
- Perguntas sobre estratégia e direção técnica
- Interesse genuíno nos problemas que eles resolvem

---

## Resumo do Processo de Preparação

```
Semana 1-2:
├── Construir Brag Document (15-20 projetos)
├── Ranquear por eixos (envolvimento, impacto, escopo)
└── Selecionar top 5-8 histórias

Semana 2-3:
├── Escrever rascunho CARL para cada história
├── Mapear para sinais/valores da empresa alvo
└── Preparar apresentação de 2 min ("tell me about yourself")

Semana 3-4:
├── Gravar a si mesmo respondendo perguntas
├── Mock interview com alguém experiente
└── Iterar baseado no feedback
```

---

*Fonte: Hello Interview — Behavioral Interviews com Austin McDonald (ex-Meta Senior EM)*
