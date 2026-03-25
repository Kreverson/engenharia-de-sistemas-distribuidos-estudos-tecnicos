# 20 — CAP Theorem & Modelos de Consistência

> Conceitos fundamentais para decisões arquiteturais em sistemas distribuídos

---

## O que é o CAP Theorem?

O **CAP Theorem** (Brewer, 2000) afirma que sistemas distribuídos só podem garantir **2 das 3 propriedades** simultaneamente:

| Propriedade | Definição |
|---|---|
| **C — Consistency** | Todos os nós enxergam os mesmos dados ao mesmo tempo |
| **A — Availability** | Toda requisição recebe uma resposta (sucesso ou erro) |
| **P — Partition Tolerance** | O sistema continua funcionando mesmo com falhas de rede entre nós |

### Por que P é obrigatório?

Em qualquer sistema distribuído real, **partições de rede são inevitáveis**. Portanto, a escolha real sempre é entre **C** e **A** na presença de uma partição.

```
Sistema Distribuído → P é obrigatório → Escolha: C ou A
```

---

## A Escolha C vs. A na Prática

### Cenário Ilustrativo

```
Servidor USA ←——— rede cortada ———→ Servidor Europa
    ↑                                       ↑
Usuário A grava dado            Usuário B tenta ler
```

Quando a rede é interrompida **antes da replicação**:
- **Consistência**: recusar a leitura para não servir dado obsoleto
- **Disponibilidade**: responder com dado possivelmente desatualizado

---

## Quando Priorizar Consistência

Cenários onde dado desatualizado seria **catastrófico**:

### 1. Reserva de Assentos (Ticket Master, Aéreas)
- Dois usuários reservando o mesmo assento 6A = problema real
- Solução: **strong consistency + distributed lock**

### 2. Controle de Estoque (Amazon)
- Último item disponível comprado por dois usuários
- Solução: **atomic transactions no banco**

### 3. Sistemas Financeiros (Order Book de Ações)
- Preço muda conforme volume negociado
- Solução: **single node ou ACID completo**

### Tecnologias para Consistência Forte
- PostgreSQL / MySQL (ACID nativo)
- Google Spanner (SQL + consistência global)
- DynamoDB com **strongly consistent reads** (habilitado por flag)

---

## Quando Priorizar Disponibilidade

Cenários onde dado levemente desatualizado é **aceitável**:

### Exemplos Comuns
- **Perfil de usuário**: nome desatualizado por segundos — sem impacto
- **Posts em redes sociais**: post não aparece imediatamente para alguns — aceitável
- **Yelp (informações de negócio)**: horário ligeiramente desatualizado — preferível a não mostrar o negócio
- **Catálogo Netflix**: novo filme aparece com delay — totalmente aceitável

### Tecnologias para Alta Disponibilidade
- DynamoDB (modo padrão — eventual consistency)
- Cassandra
- Redis (pub/sub, caching)
- CouchDB

---

## Nuances para Engenheiros Sênior/Staff

### Sistemas com Requisitos Mistos

O mesmo sistema pode ter partes com requisitos diferentes:

```
Ticket Master:
├── Busca/visualização de eventos → Disponibilidade
└── Compra de ingressos           → Consistência

Tinder:
├── Atualização de perfil/foto    → Disponibilidade
└── Matching (swipe bilateral)    → Consistência
```

### Espectro de Consistência

Do mais forte ao mais fraco:

```
Strong Consistency
      │
      ↓
Causal Consistency      → eventos relacionados preservam ordem
      │                   (ex: resposta de comentário nunca aparece antes do original)
      ↓
Read Your Own Writes    → o autor vê suas próprias mudanças imediatamente
      │                   outros usuários podem ver versão antiga
      ↓
Eventual Consistency    → sistema converge para estado consistente
                          com algum delay (padrão ao escolher disponibilidade)
```

---

## Aplicando no System Design Interview

### Framework de Decisão

```
1. Identificar operações críticas do sistema
2. Para cada operação: "se dois usuários virem dados inconsistentes, é catastrófico?"
   → Sim: priorizar CONSISTÊNCIA
   → Não: priorizar DISPONIBILIDADE
3. Mencionar explicitamente nos non-functional requirements
4. Deixar a escolha guiar decisões de tecnologia e arquitetura (Deep Dives)
```

### Como Citar no Interview

> *"Para o módulo de reservas, vou priorizar consistência — dado stale seria catastrófico. Para busca e listagem de eventos, vou priorizar disponibilidade com eventual consistency, dado que um evento levemente desatualizado não causa problema real."*

---

## Impacto no Design Técnico

### Se escolheu Consistência:
- Distributed transactions (ex: 2PC — Two-Phase Commit)
- Single node database ou replicação síncrona
- Aceitar maior latência
- Usar PostgreSQL, Spanner, ou DynamoDB com strong reads

### Se escolheu Disponibilidade:
- Múltiplas réplicas de leitura
- CDC (Change Data Capture) — eventual consistency por design
- DynamoDB multi-AZ, Cassandra
- Pode usar caching agressivo

---

## Resumo Visual

```
         Consistency
              /\
             /  \
            /    \
           /  CA  \   ← Impossível em sistemas distribuídos reais
          /        \
         /____  ____\
        /      \/    \
       /   CP  /\  AP \
      /       /  \     \
     /________/    \____\
   Partition      Availability
   Tolerance
   (obrigatório)

CP: consistência + tolerância a partições (sacrifica disponibilidade)
AP: disponibilidade + tolerância a partições (sacrifica consistência forte)
```

---

## Referências e Tecnologias Mencionadas

- **Google Spanner**: banco SQL com consistência global via TrueTime
- **Amazon DynamoDB**: NoSQL com modo de strong consistency opcional
- **Cassandra**: AP por design, write-optimized
- **PostgreSQL/MySQL**: CP por design (ACID)
- **Redis**: CP na maioria dos casos, usado frequentemente como cache

---

*Fonte: Hello Interview — CAP Theorem para System Design Interviews*
