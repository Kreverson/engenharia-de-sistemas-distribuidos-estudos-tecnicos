# 38 — Como Aprender System Design e Crescer como Engenheiro

> Consolidação de insights de múltiplas conversas da Hello Interview: com Jordan (Jordan Has No Life), Gilad (ex-EM Amazon/Meta), e o próprio Evan. Cobre método de aprendizado, transição de carreira e o papel do AI no futuro da engenharia.

---

## 📌 A Jornada de Quem Ensina System Design

Jordan (criador do canal "Jordan Has No Life") começou com uma entrevista de estágio na HubSpot onde levou a pergunta "Design Tiny URL" — e não sabia nem o que era um banco de dados direito.

**O caminho que funcionou:**
1. Leu **Designing Data-Intensive Applications** (DDIA) de Martin Kleppmann
2. Tomou notas detalhadas e publicou no Blind
3. Consumiu vídeos de system design e notou a falta de profundidade
4. Começou a criar conteúdo combinando rigor técnico (DDIA) com clareza didática

---

## 🧠 Como Aprender Tecnologias Novas

### A armadilha da documentação oficial

```
Documentações são escritas por pessoas que já entendem a tecnologia.
São cheias de buzzwords e marketing.
O "por que existe" se perde no "o que faz".
```

### Método eficaz (Jordan)

```
1. Vídeo de 10 minutos no YouTube
   → "Qual problema essa tecnologia resolve?"
   → Entenda o WHY antes do WHAT

2. Documentação oficial
   → Agora faz sentido porque você tem contexto

3. White papers / papers originais
   → "The Kafka Paper", "The Dynamo Paper"
   → Revela complexidades que vídeos ignoram

4. ChatGPT como assistente
   → "Me explique o Consistent Hashing em termos simples"
   → Bom para parsear documentação densa
   → Ruim para inventar detalhes (alucinações)
```

### Breadth-First vs Depth-First

```
Jordan (BFS):
  "Se eu fizesse DFS, nunca teria coberto nada.
   Distribuídos é décadas de conteúdo."

Estratégia:
  → Amplo primeiro → identificar o que existe
  → Profundo sob demanda → quando o trabalho exige

"Lazy learning": entender o suficiente agora para saber
que a solução existe quando precisar.
```

---

## 🏗️ Como Praticar System Design

### O ciclo mais eficaz (Evan, Hello Interview)

```
Fase 1 (primeiras 3 questões):
  → Assista o vídeo passivamente → absorva o raciocínio

Fase 2 (questões 4-10):
  → Abra whiteboard em branco
  → Timer: 35 minutos
  → Tente resolver SOZINHO antes de assistir qualquer coisa
  → Anote tudo que ficou travado (known unknowns)
  → Pesquise os known unknowns
  → Assista o vídeo → descubra os unknown unknowns
  → Repita o ciclo
```

### 10 questões fundamentais (nesta ordem)

```
1.  Bitly (URL shortener)            → hashing, redirects
2.  Dropbox                          → chunking, sync
3.  Ticket Master                    → distributed locks, queues
4.  Rate Limiter                     → token bucket, Redis
5.  Design Chat (WhatsApp)           → WebSockets, fanout
6.  YouTube / Netflix                → streaming, transcoding
7.  Uber                             → geospatial, matching
8.  News Feed (Twitter/Facebook)     → fanout on write/read
9.  Web Crawler                      → queues, dedup
10. Post Search                      → inverted index
```

"Depois dessas 10, você vai reconhecer padrões em qualquer outra questão."

---

## 📚 Recursos e Bibliográfia

| Recurso | Tipo | Para |
|---|---|---|
| **DDIA** (Kleppmann) | Livro | Fundamentos sólidos de sistemas distribuídos |
| **The Kafka Paper** (LinkedIn) | Paper | Log distribuído, stream processing |
| **The Dynamo Paper** (Amazon) | Paper | Consistent hashing, vector clocks |
| **The Flink Paper** | Paper | Stateful stream processing |
| **YouTube / Jordan HNL** | Vídeos | Deep dives em tecnologias específicas |
| **Hello Interview** | Plataforma | Guided practice, mock interviews |

---

## 💼 Carreira e Entrevistas de Manager

> Insights de Gilad (ex-EM Amazon e Meta), especialista em coaching de entrevistas para managers.

### O que distingue um great manager em entrevistas

**Framing central de Gilad:**
```
"A saída de um manager é a saída da organização sob sua responsabilidade."
(Andy Grove)

Mas mais útil para entrevistas:
"Um manager cria mudança positiva e sustentável na performance da organização."

→ Não é o que a organização FAZ.
→ É a MUDANÇA que você CAUSOU.
→ Delta, não estado atual.
```

### Exemplos aplicados

**"Fale sobre crescimento de um engenheiro":**
```
❌ Fraca: "Ajudei a promover um engenheiro"
   → O engenheiro provavelmente seria promovido de qualquer jeito
   → Qual foi o SUA contribuição específica?

✅ Forte: "Este engenheiro tinha gap em X. Fiz sessões 1:1 com Y abordagem.
   Criei projeto Z para expô-lo a A. Em 6 meses, ele fechou o gap e foi promovido
   na primeira janela em que estava pronto."
   → Claro, específico, causal
```

**"Fale sobre resolução de conflito":**
```
❌ Fraca: "Havia conflito entre times A e B. Escalei para minha liderança.
   Resolvemos."
   → Você só foi um carteiro

✅ Forte: "Havia tensão crônica entre times A e B. Aproveitei o conflito específico
   para entender os incentivos de cada lado. Redesenhei o processo de handoff para
   alinhar objetivos. Após 2 meses, os times colaboram sem minha intervenção."
   → Mudança sistêmica, não apenas resolução pontual
```

### Armadilhas comuns

```
1. "Novo contratado de baixa performance"
   → O entrevistador pensa: "Quem falhou na contratação/onboarding?"
   → Escolha histórias onde o contexto favorece VOCÊ

2. "Na Microsoft, fazemos assim..."
   → Você está defendendo um processo, não demonstrando raciocínio próprio
   → "Nós fazíamos X, mas eu pessoalmente acreditava em Y e adaptei Z"

3. Preparar para levar a código e esquecer o comportamental
   → Staff EM: esperam excelência técnica E comportamental
   → Time box: 50% comportamental, 50% técnico

4. "Sou 10 anos de experiência, não preciso me preparar"
   → O maior diferencial de candidatos experientes É a preparação específica
```

### Entrevistas para New Managers

```
Desafio: Pouca experiência formal como manager.

Solução:
  1. Autenticidade sobre o nível real — não finja ter 5 anos se tem 1
  2. Histórias de IC com perspectiva de liderança contam
     → "Quando atuei como tech lead do projeto X..."
  3. Identifique gaps antes da entrevista e acumule experiência real
     → "Nunca gerenciei baixa performance? Peça ao seu manager atual essa oportunidade agora"
  4. Não tente se vender para Senior EM se você é New Manager
     → Perderá o job de Staff AND o de Pleno
```

### Managers com Longo Tenure em Uma Empresa

```
Preocupação do entrevistador: "Cresceu ou ficou estagnado?"

Prepare narrativa de crescimento mesmo sem promoção:
  "Passei de suportar times de infra para times de AI → domain switch"
  "Passei de tech lead a manager a manager de managers → scope growth"
  "Passei de empresa de 50 para 5000 pessoas → complexity growth"

A mensagem principal:
  "Não fui promovido nesses 9 anos, mas nunca deixei de crescer.
   Aqui estão as evidências..."
```

---

## 🤖 O Papel do AI no System Design

### Limites do AI atual (perspectiva de Jordan, 2024)

```
✅ AI é útil para:
   - Busca semântica melhorada ("explique consistent hashing")
   - Parsear documentação densa
   - Gerar código boilerplate
   - Revisar decisões técnicas isoladas

❌ AI ainda não substitui:
   - Decisões com contexto organizacional
     "Qual banco usar?" depende de: knowhow do time, contratos de cloud,
      budget, relacionamentos com vendors, cultura técnica
   - Benchmarks reais: "Cassandra é mais rápido para escritas que MongoDB"
      só importa SE você implementou corretamente ambos
   - Trade-offs nuançados de negócio:
      "Devemos migrar do monólito?" envolve contexto que AI não tem
```

### O valor do contexto humano

```
Jordan: "Chat GPT pode sugerir um sistema, mas se seus stakeholders
nunca ouviram falar e não estão interessados, isso não importa.
Você precisa navegar desafios técnicos E interpessoais."

Gilad: "A síntese de múltiplos contextos — técnico, organizacional,
de produto, de pessoas — é onde engenheiros adicionam valor
crescentemente à medida que sobem na carreira."
```

---

## 🔄 Tendências em System Design (2024+)

**Do mais antigo ao mais recente:**

| Tendência | Descrição |
|---|---|
| **Stateful stream processing** | Flink, Spark Structured Streaming — mais fácil que construir do zero |
| **Columnar storage** | Parquet, ORC — queries analíticas 10-100x mais rápidas |
| **Compute-storage separation** | S3 + Athena, Snowflake — CPU e storage escalam independentemente |
| **Open table formats** | Apache Iceberg, Delta Lake — transações em data lakes |
| **Apache Arrow** | Formato de memória columnar padronizado → zero-copy entre processos |
| **Vertical scaling de volta** | CPUs mais poderosos — às vezes uma máquina grande basta |
| **NoSQL perdendo steam?** | SQL moderno (CockroachDB, Spanner) cada vez mais relevante |

---

## 💡 Lições Síntese

| Princípio | Aplicação |
|---|---|
| **Teach to Learn** | Criar conteúdo, fazer pair programming, mentorar — você aprende 3x |
| **BFS antes de DFS** | Mapeie o território antes de ir fundo |
| **Boring Tech funciona** | Postgress + Redis resolve 90% dos problemas de escala |
| **Contexto é tudo** | A "melhor" solução técnica depende do ambiente organizacional |
| **Growth > Promoção** | Demonstre que você continua crescendo, independente de título |
| **Known vs Unknown Unknowns** | Praticar revela o que você não sabe que não sabe |

---

*Arquivo gerado a partir de múltiplos episódios da Hello Interview — Aprendizado e Carreira*
