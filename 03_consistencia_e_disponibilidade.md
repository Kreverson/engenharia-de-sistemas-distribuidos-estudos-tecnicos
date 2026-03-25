# 03 — Consistência vs. Disponibilidade: O Teorema CAP e o Amazon Dynamo

> **Caso real:** Amazon, 2004-2007. O time de engenharia decide que o carrinho de compras **nunca pode ficar indisponível** — mesmo que isso signifique mostrar dados temporariamente inconsistentes.  
> **Resultado:** O paper do Amazon Dynamo (2007), que fundou o movimento NoSQL e moldou toda uma geração de bancos de dados distribuídos.

---

## 1. O Teorema CAP

Formulado por Eric Brewer (2000) e provado formalmente por Gilbert e Lynch (2002), o Teorema CAP afirma que em um sistema distribuído é impossível garantir simultaneamente as três propriedades a seguir em presença de falha de rede:

```
C — Consistency (Consistência)
    Todo nó vê os mesmos dados ao mesmo tempo

A — Availability (Disponibilidade)
    Todo request recebe uma resposta (mesmo que desatualizada)

P — Partition Tolerance (Tolerância a Partição)
    O sistema continua operando mesmo se a comunicação entre nós falhar
```

**A escolha real:**

Tolerância a partição (P) **não é opcional** em sistemas distribuídos reais. Redes falham. Pacotes se perdem. Portanto, a escolha genuína que existe é:

```
Durante uma falha de rede, você prefere:

CP → Consistência + Tolerância a Partição
     O sistema para de responder até ter certeza que os dados estão corretos
     "Ninguém vê dado errado, mas talvez ninguém veja dado nenhum"

AP → Disponibilidade + Tolerância a Partição  
     O sistema sempre responde, mesmo com dados potencialmente desatualizados
     "Todo mundo é atendido, mas nem todo mundo vê a mesma coisa"
```

![Diagrama CAP](https://upload.wikimedia.org/wikipedia/commons/thumb/b/be/CAP_Theorem_Venn_Diagram.png/600px-CAP_Theorem_Venn_Diagram.png)

---

## 2. A Decisão de Negócio da Amazon

Werner Vogels (CTO da Amazon) formulou o requisito de negócio de forma direta:

> *"Clientes devem conseguir adicionar itens ao carrinho mesmo se discos estiverem falhando, rotas de rede estiverem instáveis ou data centers estiverem sendo destruídos por tornados."*

Traduzindo isso para o Teorema CAP:

| Opção | Comportamento | Custo de negócio |
|---|---|---|
| **CP (consistência)** | Carrinho indisponível durante falha | Venda perdida para sempre |
| **AP (disponibilidade)** | Carrinho pode mostrar item já deletado | Um clique a mais para deletar de novo |

**A Amazon escolheu AP.** Dado errado temporariamente é melhor que sistema fora do ar.

> Esta não é uma decisão técnica — é uma decisão de produto e negócio que se torna arquitetura.

---

## 3. Consistent Hashing

Com dezenas de servidores, como decidir qual servidor armazena qual dado?

**Abordagem ingênua (range-based):**
```
Servidor 1: usuários A-C
Servidor 2: usuários D-F
...
```
Problema: adicionar ou remover um servidor exige redistribuição de todos os dados.

**Consistent Hashing:**

Imagine um relógio circular com posições de 0 a 2³² (o "anel de hash"). Cada servidor ocupa uma posição no anel. Cada chave de dado é mapeada para uma posição e "caminha no sentido horário" até encontrar o primeiro servidor.

```
         Servidor A (hash: 100)
              |
   Anel: 0 ──┤──── 50 ────── 100 ──── 150 ──── 200 ──── 0
              |         |          |
           Dado X     Dado Y    Servidor B
           (hash:60)  (hash:140) (hash:180)
```

- `Dado X` (hash 60) → caminha até → `Servidor A` (hash 100)
- `Dado Y` (hash 140) → caminha até → `Servidor B` (hash 180)

**Vantagem crítica:** ao adicionar um novo servidor, **apenas os dados do vizinho imediato precisam ser redistribuídos**. Não há migração global.

```python
import hashlib
import bisect

class ConsistentHash:
    def __init__(self, replicas=150):
        self.replicas = replicas  # nós virtuais para distribuição uniforme
        self.ring = {}
        self.sorted_keys = []

    def add_node(self, node):
        for i in range(self.replicas):
            key = self._hash(f"{node}:{i}")
            self.ring[key] = node
            bisect.insort(self.sorted_keys, key)

    def get_node(self, key):
        if not self.ring:
            return None
        hash_key = self._hash(key)
        idx = bisect.bisect(self.sorted_keys, hash_key) % len(self.sorted_keys)
        return self.ring[self.sorted_keys[idx]]

    def _hash(self, key):
        return int(hashlib.md5(key.encode()).hexdigest(), 16)
```

---

## 4. Replicação e Quórum (N, R, W)

O Dynamo replica cada dado em **N nós** (tipicamente 3). E introduz dois parâmetros configuráveis:

- **W (Write Quorum):** quantos nós precisam confirmar uma escrita
- **R (Read Quorum):** quantos nós precisam responder a uma leitura

**A regra de consistência forte:** `R + W > N`  
Se R + W > N, leitura e escrita sempre compartilham pelo menos um nó, garantindo que o dado mais recente seja sempre visto.

```
N=3, W=2, R=2 → R+W=4 > 3 → consistência forte
  Escrita: precisa de 2 confirmações
  Leitura: consulta 2 nós, sempre pega o mais recente

N=3, W=1, R=1 → R+W=2 ≤ 3 → consistência eventual
  Escrita: qualquer 1 nó basta → rápido
  Leitura: qualquer 1 nó → pode retornar dado desatualizado
```

Isso é chamado de **consistência sintonizável** — um dial que você ajusta conforme o caso de uso.

---

## 5. Vector Clocks (Relógios Vetoriais)

Quando dois nós aceitam escritas concorrentes para o mesmo dado (por causa de uma partição de rede), o Dynamo precisa rastrear qual versão veio depois — e se são conflitantes.

**O problema com timestamps simples:** clocks de máquinas diferentes não são sincronizados perfeitamente. Não dá para comparar tempos absolutos de forma confiável.

**Vector Clock:** cada versão de um dado carrega um vetor que registra quantas atualizações cada nó fez.

```
Estado inicial:
  Dado X: { valor: "azul", vc: {} }

Nó A atualiza:
  Dado X: { valor: "verde", vc: {A: 1} }

Nó B atualiza (sem ver a versão de A — partição):
  Dado X: { valor: "vermelho", vc: {B: 1} }

Rede volta. Dynamo tem duas versões:
  v1: { valor: "verde",    vc: {A:1} }
  v2: { valor: "vermelho", vc: {B:1} }

Nenhuma descende da outra → CONFLITO REAL
→ Dynamo retorna as duas para a aplicação resolver
```

**Para o carrinho de compras da Amazon:**  
A estratégia de resolução é **união** — todos os itens de ambas as versões são combinados. Uma adição nunca é perdida (adição = venda potencial). Se um item "fantasma" aparecer, o usuário deleta com um clique extra.

---

## 6. Consistência Eventual na Prática

Consistência eventual significa que, **na ausência de novas escritas**, todos os nós convergirão para o mesmo valor eventualmente. Não é uma promessa de dados eternamente errados — é uma promessa de convergência assíncrona.

```
T=0: Usuário adiciona item ao carrinho → Nó A confirma, Nó B ainda não sincronizou
T=1: Usuário lê do Nó B → vê carrinho sem o item (inconsistência temporária)
T=2: Replicação completa → Nó B agora tem o item
T=3: Usuário lê do Nó B → vê carrinho atualizado ✓
```

A janela de inconsistência é tipicamente de milissegundos a segundos — imperceptível para a maioria dos casos de uso.

---

## 7. Legado do Dynamo

O paper de 2007 inspirou diretamente:

| Sistema | Herança |
|---|---|
| **Apache Cassandra** | Arquitetura distribuída do Dynamo + modelo de dados do BigTable (Google) |
| **Riak** | Implementação open-source quase fiel ao paper original |
| **Voldemort (LinkedIn)** | Explicitamente inspirado no Dynamo |
| **DynamoDB (AWS)** | Os princípios do Dynamo como serviço gerenciado |

**A mudança de mentalidade:**

> Antes do Dynamo: consistência era o padrão. Abrir mão dela precisava de justificativa.  
> Depois do Dynamo: a pergunta passou a ser — "qual o custo real de uma inconsistência temporária para o seu negócio?"

---

## Referências

- [Amazon Dynamo Paper (2007) — SOSP](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- [Werner Vogels — Eventually Consistent](https://www.allthingsdistributed.com/2008/12/eventually_consistent.html)
- [Martin Fowler — CAP Theorem](https://martinfowler.com/bliki/CapTheorem.html)
