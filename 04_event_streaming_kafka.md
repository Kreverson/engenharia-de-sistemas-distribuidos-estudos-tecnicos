# 04 — Event Streaming e Apache Kafka

> **Caso real:** LinkedIn, 2008. Dezenas de sistemas internos conectados por scripts ponto-a-ponto, sem escalabilidade, sem tempo real. O time cria o Kafka — batizado em homenagem a Franz Kafka, famoso por retratar situações absurdas e burocráticas.  
> **Hoje:** Netflix (500 bilhões de eventos/dia), Nubank (72 bilhões/dia), Uber, Airbnb e praticamente toda empresa de tecnologia em escala.

---

## 1. O Problema: Pipelines Ponto-a-Ponto

Com N sistemas que precisam trocar dados entre si, a abordagem ingênua é conectar cada um diretamente aos outros. O número de conexões cresce quadraticamente:

```
5 sistemas  →  20 conexões possíveis
10 sistemas →  90 conexões
50 sistemas → 2.450 conexões
```

Cada pipeline é um pedaço de código customizado com seu próprio formato, sua própria lógica de retry, seu próprio ponto de falha. Ninguém tem visão do todo. Quando algo quebra, o impacto cascateia de forma imprevisível.

**O segundo problema:** ferramentas de batch (como Hadoop) processavam dados em janelas de horas. O LinkedIn precisava de tempo real — se alguém muda de emprego, o índice de busca precisa atualizar agora, não amanhã.

---

## 2. A Abstração Central: O Log Distribuído

O Kafka é fundamentalmente um **log append-only distribuído**. Pense em um caderno onde:
- Você só pode escrever na próxima linha em branco
- Não pode apagar nem editar entradas anteriores
- Cada linha tem um número sequencial único (o **offset**)

```
Offset: 0     1     2     3     4     5     6    ...
        │─────│─────│─────│─────│─────│─────│
        │ ev1 │ ev2 │ ev3 │ ev4 │ ev5 │ ev6 │ → append only
```

Essa estrutura simples entrega quatro propriedades poderosas:

| Propriedade | O que garante |
|---|---|
| **Ordenação** | Cada evento tem posição exata → sem ambiguidade sobre o que aconteceu primeiro |
| **Imutabilidade** | Evento escrito não muda → sem conflitos de concorrência |
| **Replay** | O dado não some após ser lido → sistemas novos podem processar o histórico completo |
| **Desacoplamento** | Produtor não sabe quem lê; consumidor avança no próprio ritmo |

---

## 3. Tópicos e Partições

**Tópico:** canal nomeado para um tipo de evento (`user-logins`, `payments`, `page-views`).

**Partição:** cada tópico é dividido em N partições, cada uma sendo um log independente. Isso permite paralelismo.

```
Tópico: "user-events"
├── Partição 0: [login_user_1, click_user_3, logout_user_1, ...]
├── Partição 1: [login_user_2, purchase_user_4, ...]
└── Partição 2: [login_user_5, click_user_2, ...]

Cada partição roda em um servidor (broker) diferente
```

**Garantia de ordem:** dentro de uma partição, a ordem é preservada. Entre partições, não há garantia. Para garantir que todos os eventos de um usuário cheguem na ordem certa, use o `user_id` como chave de particionamento — todos os eventos daquele usuário irão sempre para a mesma partição.

```python
producer.send(
    topic="user-events",
    key=str(user_id).encode(),  # garante mesma partição para o usuário
    value=evento.serialize()
)
```

---

## 4. Replicação e Líderes

Cada partição tem um **líder** (aceita leituras e escritas) e N **seguidores** (replicam os dados). Se o líder falha, um seguidor assume automaticamente.

```
Partição 0:
  Líder:    Broker 1  ←── escritas chegam aqui
  Seguidor: Broker 2  ←── replica do líder
  Seguidor: Broker 3  ←── replica do líder

Broker 1 cai:
  Broker 2 vira líder automaticamente
  Sistema continua operando sem interrupção
```

**Fator de replicação:** configure conforme a criticidade dos dados.

| Réplicas | Tolerância a falha | Custo |
|---|---|---|
| 1 | Nenhuma — perda total se o broker cair | Mínimo |
| 2 | 1 falha | Moderado |
| 3 | 2 falhas simultâneas | Padrão para produção |

---

## 5. Consumer Groups

Múltiplos consumidores podem ler do mesmo tópico de forma independente — cada um mantém seu próprio **offset** (posição de leitura). O dado não é deletado ao ser lido.

**Consumer Group:** permite que múltiplas instâncias de um serviço dividam o trabalho, cada uma processando partições diferentes.

```
Tópico com 6 partições + Consumer Group com 3 instâncias:

Instância A → processa Partições 0, 1
Instância B → processa Partições 2, 3
Instância C → processa Partições 4, 5

Se Instância B cai:
  Partições 2, 3 são redistribuídas automaticamente para A e C
```

**Regra importante:** o número de instâncias em um consumer group não pode ser maior que o número de partições. Instâncias excedentes ficam ociosas.

---

## 6. Por que o Kafka é Rápido

### 6.1 Escrita Sequencial

Discos (mesmo SSDs) são ordens de magnitude mais rápidos em acesso sequencial do que aleatório. O Kafka só faz escrita sequencial (append-only). Nunca reescreve posições do meio.

```
Escrita aleatória HDD: ~100 IOPS
Escrita sequencial HDD: ~3.000 MB/s
                          ↑ 30x mais rápido
```

### 6.2 Page Cache do Sistema Operacional

O Kafka **não faz cache em memória heap da JVM**. Ele delega completamente para o **page cache** do Linux (memória gerenciada pelo kernel).

```
Produtor escreve → dado vai para page cache do kernel → flush para disco
Consumidor lê → se dado ainda está no page cache → sai direto da memória
                sem tocar no disco
```

Por que não usar cache na JVM? Porque objetos na heap da JVM são coletados pelo Garbage Collector. Com milhões de mensagens por segundo, o GC se torna um gargalo severo. O page cache do SO não tem esse problema.

**Benefício extra:** se o processo Kafka reinicia, o page cache continua quente. Sem cold start.

### 6.3 Zero-Copy com sendfile()

No envio tradicional de dados do disco para a rede, os dados passam por 4 cópias:

```
Disco → Kernel buffer → User space → Socket buffer → NIC
         (1ª cópia)    (2ª cópia)   (3ª cópia)    (4ª cópia)
```

O Kafka usa a chamada de sistema `sendfile()` do Linux, que elimina as cópias intermediárias desnecessárias:

```
Disco → Kernel buffer ──────────────────→ NIC
         (page cache)   zero-copy        (1 cópia só)
```

Isso funciona porque o Kafka usa o **mesmo formato binário** do produtor ao consumidor — sem necessidade de deserializar/reserializar no broker.

### 6.4 Compressão em Batch

O produtor acumula mensagens em um batch e comprime o batch inteiro antes de enviar. 1000 mensagens sobre o mesmo assunto têm muita redundância — a compressão do batch inteiro é muito mais eficiente que comprimir mensagem por mensagem.

O batch permanece comprimido do produtor ao broker até o consumidor, que só descomprime no final.

---

## 7. Event Sourcing

O Kafka habilita naturalmente o padrão de **Event Sourcing**: em vez de armazenar o estado atual de uma entidade, você armazena todos os eventos que levaram a esse estado.

```
Abordagem tradicional (state-based):
  Tabela contas: { user_id: 123, saldo: 1000 }
  → Sobrescreve o saldo a cada transação

Event Sourcing (com Kafka):
  Evento 1: { user: 123, tipo: "depósito",     valor: 500 }
  Evento 2: { user: 123, tipo: "saque",         valor: 200 }
  Evento 3: { user: 123, tipo: "transferência", valor: 300 }
  → Saldo = soma de todos os eventos = 0
```

**Vantagens:**
- Histórico completo e auditável
- Qualquer estado anterior é reconstituível
- Múltiplos consumidores derivam visões diferentes do mesmo stream

O Nubank usa exatamente esse modelo: o extrato bancário não é um dado armazenado, é o resultado de aplicar todos os eventos em sequência.

---

## 8. Batch vs. Real-Time: A Unificação

O Kafka dissolveu a distinção artificial entre processamento batch e tempo real:

> *"Todo dado nasce como evento. Batch é o que acontece quando você acumula eventos e processa depois. O evento em si é sempre em tempo real."*

Com o mesmo tópico Kafka:
- Consumidor de tempo real → lê cada mensagem conforme chega
- Consumidor batch → lê janelas de tempo (últimas 24h) em um job agendado
- Consumidor de replay → começa do offset 0 e reprocessa o histórico inteiro

---

## 9. Tradeoffs do Kafka

| Limitação | Detalhe |
|---|---|
| **Complexidade operacional** | Kafka em produção exige time dedicado. Historicamente dependia de Zookeeper (outro sistema distribuído para gerenciar). O modo KRaft (sem Zookeeper) é recente. |
| **Ordem só por partição** | Ordem global exige uma única partição → sem paralelismo |
| **Semântica de entrega** | "Exatamente uma vez" (exactly-once) tem custo de performance. A maioria usa "pelo menos uma vez" (at-least-once) com processamento idempotente. |
| **Não é banco de dados** | Dados são descartados após o período de retenção configurado |

---

## Referências

- [The Log: What every software engineer should know — Jay Kreps (LinkedIn)](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)
- [Kafka Documentation](https://kafka.apache.org/documentation/)
- [Designing Data-Intensive Applications — Martin Kleppmann (Capítulo 11)](https://dataintensive.net/)
