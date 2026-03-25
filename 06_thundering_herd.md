# 06 — Thundering Herd e a Arte de se Recuperar de uma Falha

> **Caso real:** Slack, 26 de fevereiro de 2025.  
> **Problema:** manutenção de rotina → falha no cache → sobrecarga no banco → sistema cai → todos os clientes tentam reconectar simultaneamente → sistema cai de novo.  
> **Duração do incidente:** mais de 10 horas.  
> **Causa final:** não foi a queda — foi a recuperação.

---

## 1. A Arquitetura do Slack (Vitess + MySQL)

O Slack usa MySQL como banco de dados principal, mas com uma camada de sharding chamada **Vitess** — originalmente desenvolvida pelo YouTube quando bateram no mesmo limite de escala.

```
Aplicação Slack
      │
   VTGate ← proxy que conhece a localização de cada dado
      │
  ┌───┴────────────────────┐
Shard 1   Shard 2   Shard 3  ...dezenas de shards
(MySQL)   (MySQL)   (MySQL)
```

O VTGate abstrai a complexidade: para o código da aplicação, é como se existisse apenas um MySQL. Por baixo, são dezenas de shards espalhados. No pico, o Slack serve **2,3 milhões de queries por segundo** com latência mediana de 2ms e P99 de 11ms.

---

## 2. A Sequência do Incidente

```
06:45 PT — Manutenção de rotina em cluster de banco de dados
           ↓
           Manutenção acorda defeito latente no sistema de cache
           ↓
           Cache falha → queries que deveriam retornar em microssegundos
                         agora vão direto para o banco
           ↓
           ~50% das instâncias que dependiam daquele banco ficam indisponíveis
           ↓
           Réplicas ficam defasadas em relação ao primário (replication lag)
           ↓
           Sistema de automação detecta réplica doente → tenta substituir
           ↓
           Nova réplica tenta sincronizar dados do primário (que está sob carga)
           ↓
           Nova réplica não sincroniza a tempo → automação a mata
           ↓
           Provisiona outra → mesma coisa → loop infinito de substituição
           ↓
11:00 ET — +3.000 reports no Downdetector. Time começa a mitigar manualmente.
           ↓
           Banco começa a se recuperar. Backend responde.
           ↓
           TODOS os clientes desconectados recebem sinal de vida ao mesmo tempo.
           ↓
           TODOS tentam reconectar ao mesmo tempo.
           ↓
           Sistema cai de novo.

→ Thundering Herd
```

---

## 3. Thundering Herd (A Manada Trovejante)

O termo vem de sistemas operacionais: quando múltiplos processos esperam por um recurso e o recurso fica disponível, **todos acordam ao mesmo tempo**, mas apenas um consegue usá-lo. Os outros desperdiçam ciclos de CPU para nada.

Em sistemas distribuídos modernos, o efeito é amplificado por ordens de magnitude:

```
Sistema dimensionado para:
  2.300.000 queries/segundo distribuídas ao longo do tempo

Thundering Herd entrega:
  2.300.000 queries no mesmo milissegundo

Não é aumento de volume — é compressão temporal do mesmo volume.
O sistema não foi projetado para taxa instantânea, foi projetado para taxa.
```

**A espiral da morte:**
```
Sistema cai
  → Clientes tentam reconectar simultaneamente
  → Avalanche de reconexões derruba sistema de novo
  → Mais clientes desconectados
  → Tentam reconectar simultaneamente de novo
  → Loop
```

Cada tentativa de recuperação causa a próxima falha.

---

## 4. Defesa 1: Backoff Exponencial com Jitter

**Backoff exponencial** resolve parte do problema: em vez de tentar a cada milissegundo, o cliente dobra o intervalo a cada falha.

```
Tentativa 1: espera 100ms
Tentativa 2: espera 200ms
Tentativa 3: espera 400ms
Tentativa 4: espera 800ms
Tentativa 5: espera 1600ms
...
```

O servidor tem tempo para respirar entre as tentativas. Mas **backoff exponencial sozinho não resolve o Thundering Herd** — se 10.000 clientes falharam no mesmo instante, todos usam o mesmo backoff e continuam sincronizados:

```
T=0ms:   10.000 clientes falham
T=100ms: 10.000 clientes tentam juntos → falham
T=200ms: 10.000 clientes tentam juntos → falham
T=400ms: 10.000 clientes tentam juntos → falham
```

Você trocou uma avalanche grande por avalanches menores, mas sincronizadas.

**A solução é adicionar jitter (aleatoriedade):**

```python
import random
import time

def connect_with_exponential_backoff_and_jitter(
    connect_fn,
    max_retries=10,
    base_delay_ms=100,
    max_delay_ms=30_000
):
    for attempt in range(max_retries):
        try:
            return connect_fn()
        except ConnectionError:
            if attempt == max_retries - 1:
                raise
            
            # Backoff exponencial
            exponential_delay = base_delay_ms * (2 ** attempt)
            
            # Jitter: número aleatório entre 0 e o delay exponencial
            jitter = random.uniform(0, exponential_delay)
            
            # Cap no delay máximo
            actual_delay = min(exponential_delay + jitter, max_delay_ms)
            
            time.sleep(actual_delay / 1000)
```

Com jitter, os 10.000 clientes se distribuem no tempo:

```
T=0ms:     10.000 clientes falham
T=47ms:    cliente 1 tenta
T=83ms:    cliente 2 tenta
T=112ms:   cliente 3 tenta
...
T=400ms:   distribuição completa → carga suave e uniforme ✓
```

**Transformou um pico em uma rampa.**

---

## 5. Defesa 2: Circuit Breaker

O Circuit Breaker monitora o estado de um serviço e interrompe chamadas quando o serviço claramente está falhando — em vez de deixar requisições acumularem esperando timeout.

```
ESTADO: FECHADO (normal)
  ↓ N falhas consecutivas em curto período
ESTADO: ABERTO (bloqueando)
  → Toda chamada retorna erro imediatamente (fail fast)
  → Serviço ganha tempo para se recuperar
  ↓ após timeout (ex: 30s)
ESTADO: SEMI-ABERTO (testando)
  → Deixa passar UMA requisição de teste
  ↓ sucesso → FECHADO
  ↓ falha   → ABERTO novamente
```

**Por que fail fast importa:**

```
Requisição em sistema saudável:
  Processada em 10ms → libera recursos

Requisição em sistema sobrecarregado (sem circuit breaker):
  Aguarda 30s timeout → ocupa conexão + memória + threads durante 30s

Requisição em sistema sobrecarregado (com circuit breaker):
  Retorna erro em <1ms → libera recursos imediatamente
```

Uma requisição rejeitada imediatamente consome quase nada. Uma que espera 30s para dar timeout consome conexão, memória e threads durante todo esse tempo.

```python
import time
from enum import Enum

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30):
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.last_failure_time = None

    def call(self, fn, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
            else:
                raise Exception("Circuit OPEN — fail fast")

        try:
            result = fn(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise e

    def _on_success(self):
        self.failure_count = 0
        self.state = CircuitState.CLOSED

    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN
```

---

## 6. Defesa 3: Load Shedding (Descarte de Carga)

Quando o sistema está no limite, **rejeitar intencionalmente uma fração das requisições** permite que as restantes sejam atendidas com qualidade.

```
Sistema no limite, sem load shedding:
  Aceita 100% das requisições
  → Todas esperam → todas demoram → todas dão timeout
  → 100% dos usuários insatisfeitos

Sistema no limite, com load shedding (descartando 20%):
  Rejeita 20% imediatamente com HTTP 503
  → 80% são processadas adequadamente
  → 20% recebem erro, 80% recebem resposta de qualidade
  → 80% dos usuários satisfeitos
```

O Slack fez exatamente isso durante a recuperação: pausaram filas de eventos, limitaram endpoints que geravam carga excessiva, e — de forma mais radical — **decidiram não reprocessar o backlog de eventos acumulados durante a queda**. Perderam dados de baixa prioridade para garantir estabilidade do sistema.

> *"Isso não é gambiarra — é engenharia."*

---

## 7. Outros Exemplos do Mesmo Padrão

O Thundering Herd aparece em vários contextos:

| Padrão | Gatilho | Consequência |
|---|---|---|
| **Cache Stampede** | TTL de chave popular expira → todos vão ao banco | Banco sobrecarregado |
| **Tempestade de Reinício** | Deploy → todos os pods reiniciam e aquecem cache local ao mesmo tempo | Spike de CPU e banco |
| **Cron Storm** | N servidores com job agendado para meia-noite exata | Spike de carga toda madrugada |
| **Reconnect Storm** | Sistema volta após queda → todos clientes reconectam simultaneamente | Sistema cai de novo |

A causa é sempre a mesma: **eventos correlacionados que transformam carga distribuída em carga instantânea**.

A defesa é sempre a mesma: **quebrar a correlação com aleatoriedade**.

---

## 8. A Lição Fundamental

> *"Sistemas distribuídos não falham quando tudo dá errado ao mesmo tempo. Eles falham quando tudo tenta dar certo ao mesmo tempo."*
>
> *"A falha mais perigosa do Slack não foi a manutenção, não foi o bug do cache, não foi o banco sobrecarregado. Foi a recuperação — milhões de clientes bem-intencionados tentando voltar para casa ao mesmo tempo."*

---

## Referências

- [Slack Engineering — February 2025 Incident Post-Mortem](https://slack.engineering/)
- [AWS Architecture Blog — Avoiding The Thundering Herd](https://aws.amazon.com/builders-library/avoiding-thundering-herds/)
- [Google SRE Book — Handling Overload](https://sre.google/sre-book/handling-overload/)
- [AWS — Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
