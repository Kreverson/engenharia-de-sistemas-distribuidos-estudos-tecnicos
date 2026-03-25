# 08 — Concorrência, Protocolos e o WhatsApp

> **Caso real:** WhatsApp — 100 bilhões de mensagens/dia, 2 bilhões de usuários, **2 milhões de conexões simultâneas em um único servidor**.  
> Quando o Facebook comprou por US$ 19 bilhões, 32 engenheiros mantinham o back-end. A métrica interna era **40 milhões de usuários por engenheiro**.  
> A stack: **Erlang** — uma linguagem dos anos 80.

---

## 1. Quanto um Servidor Aguenta?

A pergunta parece simples. A resposta é: **depende do que você faz com cada requisição**.

```
Fluxo 1: requisição → retorna arquivo estático do cache
  Tempo: ~0,01ms
  Um core: 100.000 req/s

Fluxo 2: requisição → consulta banco → processa → serializa JSON
  Tempo: ~100ms
  Um core: ~10 req/s

Mesma máquina. Diferença de 10.000x.
```

Em APIs típicas, o servidor não passa o tempo calculando. Passa **esperando**:
- Banco de dados responder
- API externa retornar
- Disco ser lido

Em uma requisição típica: **5% do tempo é processamento real, 95% é espera (I/O).**

A forma como o servidor lida com essa espera determina tudo.

---

## 2. Modelos de Concorrência

### 2.1 Thread por Requisição (modelo tradicional)

```
Requisição chega → cria/pega uma thread do pool
Thread chama o banco → BLOQUEADA esperando resposta
Thread fica parada segurando memória
Banco responde → thread processa → libera
```

**Custo:**
- Cada thread Java: ~1MB de stack
- 10.000 threads = 10GB só de stack
- Context switching: o SO gasta tempo gerenciando threads, não processando
- Resultado: 1.000 a 10.000 req/s em servidores Java típicos

### 2.2 Event Loop / I/O Não-Bloqueante

```
Requisição chega → começa a processar
Precisa do banco → registra callback, NÃO bloqueia
                → vai processar próxima requisição
Banco responde → callback é chamado → continua processando A
```

Uma única thread gerencia milhares de conexões. O tempo de espera do I/O é usado para processar outras requisições.

```
Thread única:
  t=0ms:   Requisição A chega, chama banco, registra callback
  t=1ms:   Requisição B chega, chama banco, registra callback
  t=2ms:   Requisição C chega, processa sem I/O, responde
  t=5ms:   Banco responde para A → callback executa → responde A
  t=6ms:   Banco responde para B → callback executa → responde B
```

É o que o **Node.js** faz com Event Loop, o **Nginx** internamente, e o **Go** com goroutines.

**Resultado:** 10.000 a 100.000 req/s na mesma máquina.

**Tradeoff:** código assíncrono é mais complexo de escrever e debugar ("callback hell"). Erros de lógica bloqueante travam o loop inteiro.

### 2.3 Processos Leves (Erlang/BEAM)

```
Cada conexão = um processo Erlang dedicado
Processo ocupa: ~2 KB
Gerenciado pela VM (BEAM), não pelo SO
Context switch: troca de ponteiro interno na VM
```

```
Threads Java: 10.000 threads = 10 GB de RAM
Processos Erlang: 2.000.000 processos = 4 GB de RAM
```

No mesmo servidor com 100GB de RAM, Erlang pode rodar literalmente 2 milhões de processos — um por usuário conectado.

**Isolamento:** cada processo tem sua própria memória (heap). Um processo com erro não afeta outros. Sem memória compartilhada = sem race conditions = sem locks.

---

## 3. Erlang e o Modelo Actor

Erlang implementa o **modelo Actor**: cada processo é um ator que:
- Tem estado interno privado
- Comunica-se apenas por troca de mensagens (mailbox)
- Nunca compartilha memória diretamente

```erlang
% Processo simples em Erlang
loop(State) ->
    receive
        {mensagem, De, Conteudo} ->
            NovoState = processar(State, Conteudo),
            De ! {ok, entregue},
            loop(NovoState);
        stop ->
            ok
    end.
```

**Supervisores e árvores de supervisão:**

```
Supervisor raiz
  ├── Supervisor de conexões
  │     ├── Processo usuário_1 (morre → supervisor reinicia em ms)
  │     ├── Processo usuário_2
  │     └── Processo usuário_3
  └── Supervisor de banco
        ├── Processo pool_1
        └── Processo pool_2
```

"Let it crash" — a filosofia Erlang: não tente tratar todos os erros possíveis. Se um processo encontra estado inválido, deixe-o morrer. O supervisor reinicia um novo em milissegundos. A falha nunca cascateia.

---

## 4. Fun XMPP: Protocolo Binário Comprimido

O WhatsApp é baseado em XMPP, um protocolo de mensagens em XML. O problema: XML é verboso.

```
Mensagem "oi" em XMPP puro:
<message from="a@server.com" to="b@server.com" type="chat">
  <body>oi</body>
</message>

= 180 bytes

Conteúdo real: 2 bytes ("oi")
Overhead do protocolo: 178 bytes (99% de overhead!)
```

O WhatsApp criou o **Fun XMPP** — uma codificação binária onde cada keyword do protocolo vira um único byte de um dicionário fixo:

```
"message" → 0x59
"from"    → 0x12
"to"      → 0x11
"body"    → 0x42
```

```
Mesma mensagem em Fun XMPP:
[0x59][0x12][endereço_a][0x11][endereço_b][0x42][oi]
= ~20 bytes

Redução: ~89%
```

**Por que importa:** a diferença entre funcionar e não funcionar em redes 2G/3G precárias. Cada byte economizado é um byte que pode não se perder. Em escala de 100 bilhões de mensagens/dia, a redução de banda é enorme.

**Conexão TCP persistente:** o WhatsApp mantém uma única conexão TCP aberta enquanto o app está rodando. Não há handshake novo para cada mensagem. Rick Reid encontrava clientes Nokia com conexões abertas há 45 dias contínuos.

---

## 5. Mnesia: Banco de Dados Nativo do Erlang

Para rastrear qual usuário está conectado em qual servidor, o WhatsApp usa o **Mnesia** — um banco de dados distribuído que faz parte da linguagem Erlang, rodando em memória.

```
Mnesia armazena:
  { user_id → server_id }
  
Consulta: "onde está o usuário B?"
→ user_id_B → server_47

Latência: microssegundos (em memória, no mesmo processo runtime)
```

Isso permite que o processo do usuário A mande uma mensagem **diretamente** para o processo do usuário B no servidor 47, via comunicação nativa entre nós Erlang — sem passar por um message broker intermediário como Kafka ou RabbitMQ.

**Problema em escala:** a replicação do Mnesia usava uma fila única. Se um nó tinha problema, a fila travava a replicação para todos os nós (incluindo os saudáveis).

**Solução:** filas de replicação separadas por nó destino. O problema de um nó não bloqueia os outros.

---

## 6. Arquitetura de Ilhas (Wisps)

O back-end é organizado em **ilhas** — pares de nós (primário + secundário):

```
Ilha A: [Nó primário] ←→ [Nó secundário]
Ilha B: [Nó primário] ←→ [Nó secundário]
Ilha C: [Nó primário] ←→ [Nó secundário]
```

Cada ilha é responsável por um subconjunto de dados. Se o primário cai, o secundário assume (apenas 2 nós necessários, sem quorum complexo).

**Isolamento de falha:** se a Ilha A tem problemas, as Ilhas B e C continuam funcionando normalmente. A falha não escala para o cluster inteiro.

Dentro de cada ilha, dados são divididos em **fragmentos (DB Frags)**. Cada fragmento é gerenciado por um único processo Erlang — sem race condition, sem locks, sem transações necessárias. O acesso é serializado por design.

**98% das mensagens em RAM:** a persistência em disco é assíncrona. O disco é backup, não caminho principal.

---

## 7. Entrega de Mensagens: Três Cenários

### Cenário 1: Destinatário online
```
A envia "oi" para B
  → Processo de A consulta Mnesia → B está no servidor 47
  → Processo de A manda mensagem direta para processo de B
  → B recebe → empurra para celular via TCP
  → Celular de B confirma (ACK) → ✓ entregue (1 tick)
  → B abre conversa → ✓ lido (azul)
```
O servidor **não armazena a mensagem**. Entregou → deletou. Sem armazenamento = sem vazamento = menos IO.

### Cenário 2: Destinatário offline
```
A envia "oi" para B (offline)
  → Mnesia não acha processo de B
  → Mensagem vai para "offline cache" da ilha de B
  → ACK para A (✓ chegou no servidor — 1 tick)
  → B abre o app → servidor entrega
  → B abre conversa → ✓ lido (azul)
```

### Cenário 3: Grupo de 100 pessoas
```
A envia mensagem para grupo
  → Celular de A faz 1 upload
  → Servidor consulta lista de membros do grupo
  → Cria processos temporários (10-20 destinatários cada)
  → Entrega em paralelo para os 100 membros
```
O celular de A pagou por uma mensagem. O cluster fez 100 entregas em paralelo.

---

## 8. Criptografia End-to-End (Signal Protocol)

Desde 2016, todo conteúdo do WhatsApp é criptografado com o **Signal Protocol**:

```
Instalação do app:
  → Gera par de chaves: pública (vai para o servidor) + privada (nunca sai do celular)

Troca de mensagem (A → B):
  → Celulares de A e B calculam independentemente uma "chave de sessão" compartilhada
  → Ninguém transmite essa chave — ela é derivada matematicamente dos dois lados
  → Mensagem cifrada com AES-256
  → A cada mensagem, a chave muda (forward secrecy)
```

**Impacto arquitetural:** o servidor se torna um **carteiro cego**. Não lê o conteúdo, não armazena (já que não conseguiria ler). Entregou → deletou.

**Forward Secrecy:** se a chave da mensagem 5 for comprometida, apenas a mensagem 5 é legível. Mensagens anteriores e posteriores usam chaves diferentes — estão protegidas.

---

## 9. Resumo: Por que Erlang e não Java?

| Aspecto | Java (threads) | Erlang (processos) |
|---|---|---|
| Memória por unidade | ~1 MB por thread | ~2 KB por processo |
| Conexões simultâneas (100GB RAM) | ~100.000 | ~2.000.000 |
| Isolamento de falha | Sem (thread compartilha heap) | Total (heaps separados) |
| Race condition | Possível (precisa de locks) | Impossível (sem memória compartilhada) |
| Recuperação de falha | Manual ou try/catch | Automática via supervisor |
| Complexidade operacional | Ecossistema familiar | Curva de aprendizado alta |

> *"Se tivessem escolhido Java, precisariam de 20x mais servidores e um message broker externo."*

---

## Referências

- [Rick Reed — WhatsApp at Scale (USENIX)](https://www.usenix.org/conference/lisa14/conference-program/presentation/reed)
- [Signal Protocol Documentation](https://signal.org/docs/)
- [Erlang — Learn You Some Erlang](https://learnyousomeerlang.com/)
- [The BEAM Book — Erlang Runtime System](https://github.com/happi/theBeamBook)
