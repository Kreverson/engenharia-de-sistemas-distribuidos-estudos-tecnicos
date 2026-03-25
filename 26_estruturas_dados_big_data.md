# 26 — Estruturas de Dados para Big Data

> Bloom Filter, Count-Min Sketch e HyperLogLog — quando exatidão é trocada por eficiência de espaço

---

## Motivação

Em sistemas distribuídos de grande escala, surgem problemas onde:
- Manter contadores ou conjuntos exatos seria **proibitivamente caro em memória**
- Um **erro aceitável** (probabilístico) permite soluções ordens de magnitude mais eficientes

> **Aviso para engenheiros júnior:** prefira resolver o problema com estruturas exatas primeiro. Use estas estruturas apenas quando você tiver estimativas que **comprovam** que a solução exata é impraticável.

---

## 1. Bloom Filter

### Problema que resolve

**"Este item já foi visto antes?"** → com uso mínimo de memória

Bloom Filter aproxima um **set** (conjunto) com uma fração do espaço.

### Como Funciona

**Versão simplificada (1 hash):**

```
Participantes da reunião: Albert(3), Brian(2), Christina(3), Zed(1)
                                ↓ hash(nome) % 4

Bit array:  [0] [1] [2] [3]
             0   0   0   0  ← inicial

Adicionar Albert (hash=3): [0][0][0][1]
Adicionar Brian (hash=2):  [0][0][1][1]

Query "Christina?" → hash=3 → bit 3 = 1 → "TALVEZ presente" ⚠️
Query "Zed?" → hash=1 → bit 1 = 0 → "DEFINITIVAMENTE ausente" ✓
```

**O problema:** colisões causam **falsos positivos** (Albert e Christina compartilham o mesmo hash).

**Solução: múltiplos hashes**

```
Cada item é hasheado K vezes
Para estar "presente", TODOS os K bits devem estar setados

Albert: hash1=3, hash2=1 → seta bits 3 e 1
Brian:  hash1=2, hash2=0 → seta bits 2 e 0
Christina: hash1=3, hash2=2 → bits 3 e 2 já setados → "TALVEZ presente"

(se os bits de Christina coincidissem com combinações de outros, falso positivo)
```

### Propriedades

| Resposta | Garantia |
|---|---|
| "Definitivamente NÃO está no set" | **100% correto** — nunca falso negativo |
| "Provavelmente está no set" | Pode ser falso positivo (probabilidade configurável) |

**Configuração:** escolha de M (bits totais) e K (número de hashes) define a taxa de falsos positivos.

### Casos de Uso

**Web Crawler:**
```
Problema: banco de 1 bilhão de URLs × 1KB = 1 TB apenas para URLs ❌
Bloom Filter: 1 bilhão de itens → ~1 GB (100x menor) ✓

"Esta URL já foi rastreada?" → Bloom Filter responde em O(1)
Falso positivo ocasional? Apenas recrawl de uma página — aceitável
```

**Cache (otimização de cache miss):**
```
Fluxo sem Bloom Filter:
  Request → Cache miss? → Banco de dados (caro)

Fluxo com Bloom Filter:
  Request → Bloom Filter: "definitivamente não está no cache" → vai direto ao BD
           → "talvez esteja no cache" → consulta o cache → cache miss ou hit

Benefício: evita latência de consulta ao cache quando sabemos que o item não está lá
```

### Quando NÃO usar

- Tabela pequena → use hashset exato
- Não pode tolerar falsos positivos → use hashset exato
- Precisa remover itens (Bloom Filter não suporta deleção eficientemente)

---

## 2. Count-Min Sketch

### Problema que resolve

**"Quantas vezes este item apareceu?"** → contagem aproximada com memória constante

Count-Min Sketch é um Bloom Filter com **inteiros** em vez de bits.

### Como Funciona

```
Sketch = array 2D de contadores (K hashes × M buckets)

Inserir Albert:
  hash1(Albert) = 3 → sketch[0][3]++
  hash2(Albert) = 1 → sketch[1][1]++

Inserir Brian:
  hash1(Brian) = 2 → sketch[0][2]++
  hash2(Brian) = 0 → sketch[1][0]++

Inserir Christina (2 vezes):
  hash1(Christina) = 3 → sketch[0][3]++ (colide com Albert!)
  hash2(Christina) = 2 → sketch[1][2]++
```

**Query "Quantas vezes Christina apareceu?":**
```
hash1(Christina) = 3 → sketch[0][3] = 3 (albert+christina)
hash2(Christina) = 2 → sketch[1][2] = 2

min(3, 2) = 2 → "Christina apareceu no máximo 2 vezes"

"Count-MIN" = retorna o MÍNIMO entre todos os hashes
→ Upper bound (pode superestimar por colisões, nunca subestima)
```

### Padrão Heavy Hitters

Identificar os "Top-K itens mais frequentes" em stream de dados:

```
[Evento stream: visualizações, clicks, IPs, posts]
        ↓
[Count-Min Sketch] ← atualiza contador
        ↓ retorna estimate do count
[Priority Queue / Min-Heap]
  ├── Atualiza se item já está no top-K
  └── Se novo count > menor no heap → substitui
        ↓
[Top-K itens estimados] ← sempre disponível em O(log K)
```

### Casos de Uso

| Caso | Detalhe |
|---|---|
| Posts virais | "Quais posts estão recebendo mais views neste segundo?" |
| Detecção de DDoS | "Quais IPs geraram mais requisições na última hora?" |
| Query optimization | "Qual o valor mais frequente desta coluna?" (evitar full scan) |
| Anti-fraud | "Este usuário fez mais de X ações em Y minutos?" |

### Restrições para Uso

1. Precisa **consultar itens conhecidos** (o sketch não enumera itens, apenas conta os que você pergunta)
2. Precisa de **restrição de memória real** — se tem RAM sobrando, contador exato é melhor
3. Precisa **tolerar superestimativa** — o count pode ser maior que o real (nunca menor)

---

## 3. HyperLogLog

### Problema que resolve

**"Quantos itens únicos existem?"** → estimativa de cardinalidade com memória O(1)

**DAU (Daily Active Users):** não me importa o total de page views, quero saber quantos usuários distintos acessaram hoje.

### Intuição: Raridade de Sequências

```
Moeda lançada:
  1 vez  → P(cara) = 50%
  2 vezes → P(cara,cara) = 25%
  3 vezes → P(cara,cara,cara) = 12.5%

Se alguém me conta que tirou "cara,cara,cara,cara" (4 consecutive):
  → Estimativa: lançou ~16 vezes

Quanto mais rara a sequência observada, mais lançamentos houve
```

### Como Funciona

**Usando zeros à direita no hash:**

```
Para cada usuário:
  1. hash(user_id) → número binário
  2. Contar zeros no final (trailing zeros)

Albert → ...10100 → 2 trailing zeros
Brian  → ...11000 → 3 trailing zeros
Fred   → ...10000 → 4 trailing zeros (mais raro!)

Max trailing zeros observado = M
Estimativa de itens únicos ≈ 2^M
```

**Problema:** um usuário com hash incomum (muitos zeros) inflaria a estimativa.

**Solução: múltiplos registros (registers)**

```
hash(user_id) → split em dois partes:
  - Primeiros bits: determina o BUCKET (qual register)
  - Demais bits: conta trailing zeros neste bucket

Resultado: K registers independentes, cada um com seu max trailing zeros

Estimativa final = harmonic_mean(registers) × correction_factor
```

**Vantagem:** outliers afetam apenas 1 register, média harmônica atenua o impacto.

### Precisão

HyperLogLog padrão: ~1.6% de erro com ~1.5KB de memória para qualquer cardinalidade.

```
Cardinalidade real: 1.000.000 usuários únicos
HLL estimativa: 984.000 a 1.016.000 (±1.6%)
Memória: 1.5 KB (independente de quantos usuários!)
```

### Casos de Uso

| Caso | Detalhe |
|---|---|
| DAU/MAU | "Quantos usuários únicos hoje?" |
| Cardinality estimation em SQL | Otimizador de query usa HLL para decidir plano de execução |
| Cache sizing | "Quantas keys únicas estão sendo requisitadas?" |
| A/B testing | "Quantos usuários únicos viram a variante B?" |

> **Redis:** implementa HyperLogLog nativamente com comandos `PFADD` e `PFCOUNT`.

### Quando NÃO usar

- Dataset pequeno (< 1M itens) → hashset exato é trivial
- Precisa de exatidão absoluta → hashset exato
- Precisa enumerar os itens únicos → impossível com HLL

---

## Tabela Comparativa

| Estrutura | Problema | Resposta | Erro | Memória |
|---|---|---|---|---|
| Bloom Filter | "Item está no set?" | "Sim" (talvez) ou "Não" (certo) | Falsos positivos | O(N bits) — muito menor que hashset |
| Count-Min Sketch | "Quantas vezes apareceu?" | Upper bound do count | Pode superestimar | O(K×M) constante |
| HyperLogLog | "Quantos itens únicos?" | Estimativa de cardinalidade | ~1-2% | O(1) — constante! |

---

## Identificação em Interviews

| Sinal no problema | Estrutura provável |
|---|---|
| "Evitar reprocessar URLs/itens já vistos" | Bloom Filter |
| "Detectar os top-K mais frequentes em stream" | Count-Min Sketch + Heap |
| "Contar usuários únicos sem armazenar todos" | HyperLogLog |
| "Sistema embarcado/raspberry pi" | Qualquer uma das três |
| "Memória constante" | Count-Min Sketch ou HyperLogLog |

---

*Fonte: Hello Interview — Data Structures for Big Data*
