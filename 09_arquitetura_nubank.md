# 09 — Arquitetura do Nubank: Escala com Imutabilidade

> **Escala (2025):** 122 milhões de clientes, 20 shards só no Brasil, 4.000+ microsserviços, 72 bilhões de eventos Kafka/dia, 600 TB de logs ingeridos diariamente.  
> **Decisões controversas que tornaram isso possível:** Clojure em 2013 (quando ninguém usava em produção), Datomic como banco principal, shards como unidades completas de infraestrutura.

---

## 1. A Decisão Controversa: Clojure

Em 2013, quando todo o mercado usava Java, Python ou Ruby, o Nubank escolheu **Clojure** — uma linguagem funcional que roda na JVM, considerada "acadêmica" na época.

**Por que Clojure? A imutabilidade.**

Em linguagens tradicionais, dados são mutáveis por padrão:

```java
// Java — mutação de estado compartilhado
Account account = getAccount(123);
account.setBalance(account.getBalance() + 100);  // modifica o objeto
save(account);

// Race condition: dois processos lendo e modificando ao mesmo tempo
// → inconsistência, saldo errado
```

Em Clojure, dados são imutáveis por design:

```clojure
;; Clojure — imutabilidade
(let [account (get-account 123)
      updated-account (assoc account :balance (+ (:balance account) 100))]
  (save updated-account))
;; `account` nunca muda. `updated-account` é um novo valor.
;; Não existe sobrescrever — só existe criar nova versão.
```

**Por que imutabilidade importa num banco:**

```
Dois processos tentam atualizar o mesmo saldo simultaneamente:

Com mutação:              Com imutabilidade:
  Processo A lê: 1000      Processo A lê versão 1: { saldo: 1000 }
  Processo B lê: 1000      Processo B lê versão 1: { saldo: 1000 }
  A escreve: 1100          A cria versão 2: { saldo: 1100 }
  B escreve: 900           B cria versão 2: { saldo: 900 }
  → saldo final: 900 ✗     → conflito detectado → retenta ✓
  (perda de R$ 100)        (nenhum dado perdido)
```

A imutabilidade **elimina classes inteiras de bugs** em sistemas concorrentes. Não é uma preferência estética — é uma propriedade de correção.

---

## 2. Datomic: O Banco de Dados onde o Tempo é Cidadão de Primeira Classe

O Nubank não usa PostgreSQL para o core. Usa **Datomic** — um banco de dados imutável onde cada mudança é um fato novo adicionado a uma timeline, nunca uma sobrescrita.

**Analogia:** pense no Datomic como um Git para seus dados.

```
PostgreSQL (tradicional):
  UPDATE accounts SET balance = 900 WHERE id = 123;
  → O valor anterior (1000) desaparece para sempre

Datomic:
  T=1: { account: 123, balance: 1000, tx: #tx-001 }
  T=2: { account: 123, balance: 900,  tx: #tx-002 }
  → Ambos os valores existem. O histórico é completo e navegável.
```

**O que isso resolve:**

| Problema | Com banco tradicional | Com Datomic |
|---|---|---|
| Auditoria | Tabelas de log separadas, difíceis de manter | Histórico nativo, sempre completo |
| Debugging | "Como ficou o saldo antes?" → sem resposta fácil | `(d/as-of db t)` → estado exato naquele momento |
| Compliance | Reports históricos complexos | Query temporal simples |
| Rollback lógico | Difícil sem backups completos | Trivial — o estado anterior ainda existe |

```clojure
;; Datomic: viagem no tempo é nativa
(def db-agora (d/db conn))
(def db-semana-passada (d/as-of db-agora #inst "2025-03-18"))

;; Qual era o saldo do cliente 123 semana passada?
(d/q '[:find ?saldo
       :where [123 :account/balance ?saldo]]
     db-semana-passada)
```

Não é backup — é o estado real navegável no tempo.

**Arquitetura do Datomic:**

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Transactor │────→│   Peers      │←────│   Storage    │
│  (1 processo)│     │ (N processos)│     │ (DynamoDB,   │
│  todas as    │     │  leituras    │     │  S3, etc.)   │
│  escritas    │     │  escaláveis  │     │              │
└──────────────┘     └──────────────┘     └──────────────┘
```

- **Transactor:** processa todas as escritas sequencialmente (garante consistência)
- **Peers:** leitura escalável horizontalmente (cada peer tem cache local)
- **Storage:** plugável (DynamoDB, PostgreSQL, S3)

Isso resolve o problema clássico: escritas precisam de consistência forte, leituras precisam de performance. O Datomic entrega os dois separando os papéis.

> O Nubank acreditou tanto no Datomic que **comprou a empresa desenvolvedora em 2020** (Cognitect).

---

## 3. Event Sourcing com Kafka: 72 Bilhões de Eventos/Dia

```clojure
;; No Nubank, toda ação gera um evento
{:event/type :transaction/approved
 :event/timestamp #inst "2025-03-25T10:30:00"
 :transaction/amount 150.00
 :transaction/from "account-123"
 :transaction/to "account-456"}
```

O saldo de uma conta não é um número armazenado — é o **resultado de aplicar todos os eventos em sequência**. Exatamente como um extrato bancário real:

```
Saldo = depósito_1 + depósito_2 - saque_1 - transferência_1 + ...
```

Essa abordagem (Event Sourcing) garante rastreabilidade total e permite reprocessar o histórico com nova lógica.

Mesmo batch jobs são distribuídos como streams de mensagens no Kafka — unificando processamento batch e real-time na mesma infraestrutura.

---

## 4. Scalability Units: Clonar a Infraestrutura Inteira

Em 2016, o Nubank enfrentou um problema incomum: a AWS ficou sem máquinas para acompanhar o ritmo de crescimento.

A solução convencional seria **sharding do banco de dados** — dividir os dados em múltiplos bancos. O Nubank percebeu que isso resolve apenas o banco. O gargalo pode estar no Kafka, nos microsserviços, na rede.

**A solução radical: shards como unidades completas de infraestrutura.**

```
Shard Nubank = cópia completa de TODA a infraestrutura:
  ├── Todos os 4.000 microsserviços
  ├── Clusters Kafka dedicados
  ├── Bancos Datomic separados
  ├── Redes isoladas
  └── Monitoramento próprio

Brasil: 20 shards
```

Cada shard é responsável por um grupo de clientes. O sistema de roteamento sabe em qual shard você está e direciona todas as requisições para lá.

**Benefícios:**

| Benefício | Detalhe |
|---|---|
| **Isolamento de falha** | Se um shard cai, apenas uma fração dos clientes é afetada |
| **Previsibilidade** | Sabem exatamente a capacidade de cada shard |
| **Deploys seguros** | Testam em 1 shard antes de propagar para todos |
| **Feature flags naturais** | Novas features ativadas shard a shard |

**O custo:** é caro. Cada shard é uma infraestrutura completa. Mas com 122M de clientes, o isolamento de falha vale cada centavo.

---

## 5. Autorização de Transação em < 100ms

Quando você passa o cartão Nubank:

```
Maquininha → Adquirente → Mastercard/Visa → Nubank → resposta em < 100ms
```

O Nubank tem menos de 100ms para:
1. Verificar se o cartão é válido
2. Checar o limite disponível
3. Rodar análise de fraude
4. Registrar a transação
5. Retornar aprovado/negado

O time de engenharia reduziu a latência crítica de **10.000ms para 288ms no P90** — redução de 76%.

**Como?** Removendo dependências síncronas do caminho crítico:

```
Antes:
  Request → busca dados no banco em tempo real → processa → responde
  Cada consulta adiciona latência

Depois:
  Escrita: pré-computa e materializa as informações necessárias
  Leitura: um único lookup em datastore de baixíssima latência
  Cálculos pesados acontecem no momento da escrita, não da leitura
```

É o mesmo princípio do Fanout on Write do Twitter: pague na escrita para que a leitura seja trivial.

---

## 6. Observabilidade: Alexandria

Com 4.000 microsserviços gerando 600 TB de logs por dia, os engenheiros do Nubank rodam 15.000 queries por dia nos logs, escaneando 150TB de dados.

A solução de terceiros ficou tão cara que a citação ficou famosa:

> *"Chegou a um ponto em que poderíamos contratar o Lionel Messi como engenheiro de software, pagando o mesmo valor que pagávamos pela solução externa."*

**Alexandria:** plataforma de observabilidade in-house.

```
Ingestão:    Microsserviços → Kafka (streaming de logs)
Processamento: Filtros e agregações customizados
Storage:     AWS S3 com 95% de compressão (formato Parquet)
Query:       Engine distribuída para queries sobre os logs
```

Resultado: **50% mais barato** que a solução anterior, com controle total sobre os dados.

**Os três pilares de observabilidade integrados:**

```
Métricas  → "O serviço de pagamentos está respondendo 300ms acima do normal"
Logs      → "Request ID abc123 falhou com NullPointerException na linha 42"
Traces    → "A latência vem da chamada ao serviço de risco (200ms de 288ms total)"
```

Sem os três juntos, engenheiros ficam cegos ao depurar incidentes.

---

## 7. Fraude: Processamento de 450M de Eventos/Dia

```
Transação chega
  ↓
Regras simples (filtros rápidos)
  → Se já é suspeita → bloqueada sem gastar recurso de ML
  ↓ (se passar pelas regras)
Modelo de Machine Learning
  → Score de risco
  ↓
Ação: bloquear / alertar / deixar passar
```

**Otimização inteligente:** se uma regra simples já classifica como suspeita, o modelo de ML **não roda**. Economiza tempo e recurso computacional caro.

O orquestrador foi refatorado para um modelo baseado em **DAG (Directed Acyclic Graph)** — cada componente espera apenas pelos dados que precisa, sem dependências desnecessárias. Resultado: latência caiu de 550ms para 350ms em fluxos complexos.

---

## 8. O Padrão Recorrente

Olhando para toda a arquitetura do Nubank, um padrão emerge:

| Princípio | Implementação |
|---|---|
| **Imutabilidade** | Clojure + Datomic eliminam bugs de concorrência |
| **Histórico completo** | Datomic como "Git do banco de dados" |
| **Desacoplamento** | Kafka como sistema nervoso entre serviços |
| **Isolamento de falha** | Shards como unidades completas de infraestrutura |
| **Pré-computação** | Calcula na escrita para leituras em milissegundos |
| **Observabilidade própria** | Alexandria quando a escala torna soluções externas inviáveis |

---

## Referências

- [Nubank Engineering Blog](https://building.nubank.com.br/)
- [Datomic Documentation](https://docs.datomic.com/)
- [Clojure for the Brave and True](https://www.braveclojure.com/)
- [Event Sourcing — Martin Fowler](https://martinfowler.com/eaaDev/EventSourcing.html)
