# 50 — Concorrência e Message Queues: Do Lock ao Kafka

> Dois tópicos fundamentais frequentemente cobrados em entrevistas de LLD e System Design: como gerenciar threads que compartilham estado, e como desacoplar serviços com filas de mensagem.

---

## PARTE 1 — Concorrência em Low-Level Design

### O Problema Fundamental

```
Multi-threaded environment:
  Thread A e Thread B acessam o mesmo dado ao mesmo tempo
  → Resultados imprevisíveis
  → Bugs impossíveis de reproduzir
  → Corrupção silenciosa de dados
```

### Os 3 Padrões de Problema

---

## 1.1 Correctness — Check-Then-Act e Read-Modify-Write

### Pattern 1: Check-Then-Act

```python
# Sistema de tickets de show
def book_seat(self, seat_id: str) -> bool:
    if self.seats[seat_id].is_available():   # ← check
        self.seats[seat_id].book()            # ← act
        return True
    return False
```

**Race Condition:**
```
Thread Alice: is_available("7A") → True
Thread Bob:   is_available("7A") → True   (Alice ainda não bookinou!)
Thread Alice: book("7A")
Thread Bob:   book("7A")  ← overwrite de Alice!
→ Dois clientes com o mesmo assento
```

**Lugares onde aparece:**
```
- Reserva de ingressos/quartos/assentos
- Verificação de limite de rate (check limit, then increment)
- Verificação de estoque antes de processar pedido
- Qualquer "if condition: take_action"
```

**Solução: Lock / Synchronized Block**
```java
// Java: synchronized garante que apenas 1 thread executa por vez
public synchronized boolean bookSeat(String seatId) {
    if (seats.get(seatId).isAvailable()) {
        seats.get(seatId).book();
        return true;
    }
    return false;
}
```

```python
# Python: with lock() como context manager
def book_seat(self, seat_id: str) -> bool:
    with self.lock:  # adquire lock, libera ao sair do bloco
        if self.seats[seat_id].is_available():
            self.seats[seat_id].book()
            return True
    return False
```

### Pattern 2: Read-Modify-Write

```python
# Parece simples, mas são 3 operações:
self.count += 1
# 1. READ:  temp = self.count
# 2. MODIFY: temp = temp + 1
# 3. WRITE:  self.count = temp
```

**Race Condition:**
```
count = 5
Thread A: READ → 5
Thread B: READ → 5 (A ainda não escreveu!)
Thread A: MODIFY → 6, WRITE → count = 6
Thread B: MODIFY → 6, WRITE → count = 6  ← perdemos um increment!
```

**Solução 1: Atomic Variable (para variável única)**
```java
// Java: AtomicInteger usa CPU compare-and-swap (hardware-level atomic)
AtomicInteger count = new AtomicInteger(0);

count.incrementAndGet();  // read-modify-write em 1 instrução de CPU
```

```python
# Python: sem atomic nativo, usar Lock
with self.lock:
    self.count += 1
```

**Quando usar Atomic vs Lock:**
```
Atomic:  atualizar 1 variável independente
         flags booleanos, contadores simples, referências

Lock:    quando 2+ variáveis precisam ser atualizadas juntas
         quando a lógica tem múltiplos passos
         transferências bancárias (debitar A + creditar B = 1 operação)
```

---

## 1.2 Coordination — Producer/Consumer

### O Problema

```
Contexto: web server recebe uploads, workers processam em background

Naive approach:
  while True:
      if queue.has_work():
          task = queue.pop()
          process(task)
      # else: busy loop → 100% CPU desperdiçado!
```

**Dois problemas:**
1. Worker não sabe quando há trabalho novo → polling desperdiçador
2. Queue cresce sem limite se workers são lentos → OOM crash

### Solução: Blocking Queue

```python
# Python: queue.Queue é blocking por padrão
from queue import Queue

email_queue = Queue(maxsize=1000)  # bounded: backpressure automático

# Producer (web server)
def handle_signup(user):
    db.save(user)
    email_queue.put(user.email)  # block se queue cheia → backpressure!
    return success_response()    # retorna imediatamente

# Consumer (email worker)  
def email_worker():
    while True:
        email = email_queue.get()  # BLOCK aqui se queue vazia (sem busy loop!)
        send_welcome_email(email)
        email_queue.task_done()    # sinaliza que processou
```

```java
// Java: BlockingQueue
BlockingQueue<String> emailQueue = new LinkedBlockingQueue<>(1000);

// Producer
emailQueue.put(userEmail);  // blocks if full → backpressure

// Consumer
String email = emailQueue.take();  // blocks if empty → efficient wait
sendWelcomeEmail(email);
```

**Como funciona internamente:**
```
Queue vazia + Consumer.take():
  → Consumer vai para sleeping state
  → CPU liberada para outros threads
  → Quando Producer chama put() → JVM/OS acorda o Consumer automaticamente
  → Zero busy-waiting, latência mínima
```

**Por que Bounded é essencial:**
```
Queue ilimitada:
  Producers enfileiram 10K emails/s
  Workers processam 1K emails/s
  → Queue cresce 9K/s → em minutos: OutOfMemoryError

Queue limitada (maxsize=1000):
  Queue cheia → put() bloqueia
  → Producer fica em espera
  → Naturalmente throttles o upstream
  → Sistema se auto-regula
```

---

## 1.3 Scarcity — Semaphores e Connection Pools

### Semaphore: Limitar Concorrência

```python
# Máximo de 5 downloads simultâneos
import threading
download_semaphore = threading.Semaphore(5)

def download_file(url: str):
    download_semaphore.acquire()  # pega 1 "permissão"
    try:
        _do_download(url)
    finally:
        download_semaphore.release()  # SEMPRE libera, mesmo com exceção!
```

**Por que `finally` é crítico:**
```
Sem finally:
  _do_download lança exceção
  → release() nunca é chamado
  → "permissão" nunca devolvida
  → Após 5 falhas: semaphore = 0, nenhum download pode iniciar
  → Sistema travado silenciosamente
```

### Connection Pool: Reutilizando Recursos Caros

```python
# Problema: criar nova conexão TCP para cada query = lento + caro
def query_db(sql: str):
    conn = db.connect()  # 50-200ms para TCP handshake!
    result = conn.execute(sql)
    conn.close()

# Solução: pool de conexões pré-estabelecidas
from queue import Queue

class ConnectionPool:
    def __init__(self, size: int):
        self.pool = Queue(maxsize=size)
        for _ in range(size):
            self.pool.put(create_connection())  # pré-aquece as conexões
    
    def query(self, sql: str):
        conn = self.pool.get()   # bloqueia se todas ocupadas
        try:
            return conn.execute(sql)
        finally:
            self.pool.put(conn)  # devolve ao pool, mesmo com exceção!
```

**Por que connection pool:**
```
Sem pool: cada query = TCP handshake (50-200ms) + SQL overhead
Com pool: cada query = apenas SQL overhead (< 1ms para conexão)

Para 10K queries/s:
  Sem pool: impossível (criação de conexão é gargalo)
  Com pool de 10 conexões: funciona se cada query < 1ms
```

---

## 1.4 Checklist de Concorrência para Entrevistas

```
□ Múltiplos threads acessam estado compartilhado?
  → Correctness issue → Lock ou Atomic

□ Trabalho passa de um thread para outro?
  → Coordination issue → Bounded Blocking Queue

□ Há limite fixo de recursos (conexões, downloads, API calls)?
  → Scarcity issue → Semaphore ou pool via Blocking Queue
  
□ Sempre use finally para liberar locks/semaphores/connections
□ Sempre limite o tamanho de queues (backpressure)
□ Prefira Atomic para variáveis simples, Lock para múltiplas
```

---

## PARTE 2 — Message Queues em System Design

### Por que Message Queues Existem

```
Arquitetura direta (sem queue):
  Upload Service → processa imagem (4s) → retorna ao cliente
  
  Problemas:
  - Cliente espera 4 segundos (latência ruim)
  - Se Image Processor cai → upload falha completamente
  - Spike de 50K uploads → Image Processor não aguenta

Arquitetura com queue:
  Upload Service → salva arquivo + envia msg para queue → retorna OK (200ms)
  Image Processor ← consome msgs do queue → processa em background
  
  Resolve:
  - Latência: upload retorna em 200ms
  - Fault isolation: se processor cai, msgs ficam na queue → reprocessa quando volta
  - Spikes: queue absorve o pico, processor processa no seu ritmo
```

---

## 2.1 Mecanismos Fundamentais

### Acknowledgements (ACKs)

```
Sem ACK:
  Consumer recebe msg → CRASH → msg deletada → dado perdido para sempre!

Com ACK:
  Consumer recebe msg → processa → envia ACK
  Se consumer crasha antes do ACK:
    → Queue re-entrega a outro consumer
    → Nenhum dado perdido
```

**Visibility Timeout (SQS approach):**
```
Consumer A pega msg → msg fica "invisible" por 30s para outros consumers
  Se A faz ACK dentro de 30s → msg deletada ✅
  Se A não faz ACK em 30s → msg fica visível novamente → outro consumer pega
```

### Delivery Guarantees

| Guarantee | Comportamento | Quando usar |
|---|---|---|
| **At-most-once** | Pode perder, nunca duplica | Analytics, logs (perda aceitável) |
| **At-least-once** | Nunca perde, pode duplicar | Maioria dos sistemas |
| **Exactly-once** | Nunca perde, nunca duplica | Teoricamente ideal, praticamente limitado |

**At-least-once na prática:**
```python
# Consumer deve ser IDEMPOTENT
def process_message(msg):
    # Ruim: não idempotente
    db.execute("UPDATE account SET balance = balance + 100 WHERE id = ?", msg.user_id)
    # Se executado 2x → crédito dobrado!
    
    # Bom: idempotente
    db.execute("UPDATE account SET balance = ? WHERE id = ?", msg.new_balance, msg.user_id)
    # Se executado 2x → mesmo resultado
```

### Dead Letter Queue (DLQ)

```
Msg problemática ("poisoned message"):
  Consumer 1 pega → crash
  Consumer 2 pega → crash
  Consumer 3 pega → crash
  ... repetir N vezes → outros consumidores bloqueados

Solução: max_receive_count = 5
  Após 5 tentativas → msg vai para DLQ (Dead Letter Queue)
  Main queue continua processando normalmente
  DLQ: inspecionada por humanos/alertas para debug

Em entrevistas: mencionar proativamente DLQ = sinal de seniority
```

---

## 2.2 Particionamento e Escalabilidade

### Por que particionar

```
1 queue → 1 consumer → throughput limitado ao consumer mais lento

Particionamento (Kafka model):
  Topic → [Partition 0] → Consumer Group A: Consumer 1
          [Partition 1] → Consumer Group A: Consumer 2
          [Partition 2] → Consumer Group A: Consumer 3
  
  3 partições → 3 consumers em paralelo → 3x throughput
```

### Partition Key — Ordering vs Distribution

```
"Você não pode ter ambos: ordering E distribuição perfeita"

Partition key = user_id:
  ✅ Todas as msgs do mesmo user → mesma partição → ordering preservado
  ⚠️ Users muito ativos → 1 partição sobrecarregada (hotspot)

Partition key = message_id (random):
  ✅ Distribuição perfeitamente uniforme
  ❌ Msgs do mesmo user podem ir para partições diferentes → sem ordering

Caso banking:
  Partition key = account_id → deposit(+100) e withdraw(-50) na ordem certa ✅
  Partition key = random → podem processar na ordem errada → bug!
```

---

## 2.3 Kafka vs RabbitMQ — A Decisão Arquitetural

### Modelo Fundamental

```
RabbitMQ — Message Broker (Broker é inteligente):
  Producer → Exchange → Queue → Consumer
  Exchange roteia mensagens baseado em regras
  Mensagem deletada após ACK (fire and forget)
  "Broker faz o trabalho pesado"

Kafka — Distributed Log (Consumer é inteligente):
  Producer → Topic → Consumer
  Mensagens ficam no disco por período configurável (retention)
  Consumer controla seu próprio offset (posição de leitura)
  "Log simples, consumer faz o trabalho"
```

### Comparativo técnico

| Aspecto | RabbitMQ | Kafka |
|---|---|---|
| **Throughput** | ~10K msgs/s | >1M msgs/s |
| **Latência** | 1-5ms | 5-50ms (batch mode) |
| **Persistência** | Opcional, passa pelo broker | Sempre, configurable retention |
| **Ordenação** | Global (1 consumer) | Por partição |
| **Replay** | ❌ Msg deletada após ACK | ✅ Pode re-ler qualquer ponto histórico |
| **Múltiplos consumers** | Compite pelo mesmo msg | Cada consumer group lê independentemente |
| **Routing complexo** | ✅ Exchanges com regras | ❌ Básico |
| **Operacional** | Simples | Complexo (Zookeeper/Raft) |

### Quando usar cada um

**Use RabbitMQ quando:**
```
✅ Task queues clássicas: envio de email, processamento de imagem
✅ Precisar de routing complexo (mensagem vai para X ou Y baseado em conteúdo)
✅ Baixa/média escala (< 100K msgs/s)
✅ Time pequeno sem expertise em Kafka
✅ Low-latency sub-5ms é crítico

Exemplos reais:
  Instagram: processa uploads de foto via RabbitMQ workers
  Reddit: processa votes e atualiza karma scores
```

**Use Kafka quando:**
```
✅ Múltiplos sistemas consomem os mesmos eventos
   (analytics + fraud detection + billing lendo o mesmo stream)
✅ Precisar replay: reprocessar dados históricos, debugar, migrar
✅ Escala massiva (> 1M events/s)
✅ Event sourcing: log permanente de tudo que aconteceu
✅ Streaming analytics em tempo real

Exemplos reais:
  Netflix: petabytes diários para recommendations + billing
  Uber: precificação dinâmica + detecção de fraude em tempo real
  LinkedIn: inventou o Kafka, usa para feeds e mensagens
```

### Kafka: Conceito de Offset

```
Topic: video-views
Partition 0: [view1, view2, view3, view4, view5, view6...]
              offset=0  1     2     3     4     5

Consumer Group A (analytics):
  Lendo no offset 6 → viu todos até view6

Consumer Group B (recommendations):
  Lendo no offset 3 → ainda processando

Se Consumer Group B cair e voltar:
  → Verifica onde parou no Zookeeper/Kafka internals: offset=3
  → Continua de view4
  → Sem perda de dados

Replay (após deploy com bug):
  Consumer Group A diz: "meus dados de ontem estão errados, 
  quero reprocessar desde offset 0"
  → Kafka: "tudo lá, pegue de volta"
  → Impossível com RabbitMQ
```

---

## 2.4 Problemas de Produção (Para Entrevistas Sênior)

### Consumers não conseguem acompanhar o throughput

```
Producers: 300 msgs/s
Consumers: 200 msgs/s
Queue cresce: 100 msgs/s → infinitamente

Soluções:
  1. Scale up consumers (auto-scaling baseado em queue depth)
  2. Backpressure explícita: retornar 429 para producers quando queue cheia
  3. Alertas: notificar quando queue depth > threshold
```

### Garantindo Ordering em Sistemas Distribuídos

```
Cenário: mensagens de banking em Kafka

Partition key = account_id:
  Deposit(+100) → offset 5
  Withdraw(-50) → offset 6
  → Garantido no mesmo partition → ordem preservada ✅

Se usarmos partition key = random:
  Deposit(+100) → Partition 0, offset 5
  Withdraw(-50) → Partition 2, offset 3
  Consumer 2 processa antes de Consumer 0:
  → Withdraw before Deposit → NSF error! ❌
```

---

## 2.5 Resumo de Primitivas por Problema

| Problema | Nível | Solução |
|---|---|---|
| Check-then-act | LLD | `synchronized` / `Lock` |
| Read-modify-write | LLD | `AtomicInteger` / `Lock` |
| Producer/consumer | LLD | `BlockingQueue` (bounded) |
| Limitar concorrência | LLD | `Semaphore` |
| Pool de recursos | LLD | `BlockingQueue` com pre-warmed objects |
| Async work decoupling | System Design | RabbitMQ ou Kafka |
| Bursty traffic absorption | System Design | Message Queue qualquer |
| Multi-consumer de mesmo evento | System Design | Kafka |
| Task queue simples | System Design | RabbitMQ ou SQS |
| Event sourcing / replay | System Design | Kafka |

---

*Tópicos relacionados: `04_event_streaming_kafka.md`, `07_escalabilidade_twitter.md`, `14_design_ad_click_aggregator.md`, `06_thundering_herd.md`*
