# 02 — Rate Limiting, Segurança Distribuída e o Pix

> **Caso real:** Arquitetura de segurança do DICT (Diretório de Identificadores de Contas Transacionais) — sistema central de chaves do Pix, operado pelo Banco Central do Brasil.  
> **Problema:** Ataques de varredura de chaves (enumeração de CPFs/telefones cadastrados no Pix).

---

## 1. O Problema: Varredura de Chaves (Key Enumeration Attack)

Um atacante escreve um script que gera milhões de números de telefone aleatórios e consulta o DICT para descobrir quais têm chave Pix cadastrada — e em qual banco.

```
for numero in gerar_telefones_aleatorios(1_000_000):
    resultado = consultar_dict(numero)
    if resultado.existe:
        base_alvos.append({ numero, banco: resultado.banco })
```

Isso é chamado de **key enumeration** ou **varredura de chaves**. É o primeiro passo para ataques de engenharia social — o criminoso sabe que você tem Pix, em qual banco e pode começar a aplicar golpes direcionados.

---

## 2. Primeira Abordagem: Token Bucket (e por que não basta)

O Token Bucket é o algoritmo clássico de rate limiting:

```
Balde com capacidade de N fichas
A cada segundo, X fichas são adicionadas
Cada requisição consome 1 ficha
Se não há fichas → requisição bloqueada
```

![Token Bucket](https://upload.wikimedia.org/wikipedia/commons/thumb/3/3a/Token_bucket.svg/600px-Token_bucket.svg.png)

**O problema: burst attack**

Se o balde acumula fichas enquanto está ocioso (ex.: 1000 fichas acumuladas), um atacante pode disparar 1000 requisições no mesmo milissegundo. O sistema aceita todas porque o balde estava cheio. O dano (1000 dados vazados) ocorre antes que o limite seja aplicado.

```
Balde capacidade: 1000 fichas
Atacante idle por 100 segundos → balde cheio
Atacante dispara 1000 requisições simultâneas → todas aceitas
Balde: 0 fichas → agora bloqueia
Mas os 1000 dados já vazaram
```

---

## 3. A Solução do Pix: Weighted Rate Limiting (Limitador Ponderado)

O DICT não cobra um custo fixo por requisição. **O custo depende do resultado.**

| Cenário | Resposta da API | Custo em fichas |
|---|---|---|
| Chave encontrada | `200 OK` | 1 ficha |
| Chave não encontrada | `404 Not Found` | **20 fichas** |

**Por que isso é genial?**

- Usuário legítimo que erra um dígito: paga 20 fichas, corrige, acerta. O balde se recompõe naturalmente.
- Atacante fazendo varredura: cada chave inexistente custa 20x mais. Com 100 fichas no balde, ele consegue fazer apenas **5 tentativas erradas** antes de ser bloqueado (HTTP 429).

```
Balde: 100 fichas

Usuário normal:
  Erro (chave errada)  → -20 fichas → balde: 80
  Acerto               →  -1 ficha  → balde: 79
  ✓ Fez o Pix. Balde recarrega.

Atacante:
  Erro 1 → -20 → balde: 80
  Erro 2 → -20 → balde: 60
  Erro 3 → -20 → balde: 40
  Erro 4 → -20 → balde: 20
  Erro 5 → -20 → balde: 0 → HTTP 429 → BLOQUEADO
```

> **O sistema julga a intenção pelo comportamento.** Errar custa 20x mais caro do que acertar.

---

## 4. Race Condition em Sistemas Distribuídos

O DICT não é um servidor único — é um cluster com múltiplas instâncias rodando em paralelo para suportar bilhões de consultas por mês. Isso cria o problema da **race condition** no rate limiting.

**O cenário:**

```
Atacante dispara 10 requisições simultâneas
↓
Cada requisição cai em um servidor diferente do cluster
↓
Cada servidor lê o saldo do balde: vê 100 fichas disponíveis
↓
Todos aprovam (balde parecia cheio para todos)
↓
Todos tentam descontar depois: balde vai a -100
```

O limite foi burlado porque a leitura e o desconto não foram **atômicos**.

---

## 5. Solução: Atomicidade com Redis + Scripts Lua

A solução é centralizar o controle do balde em um **cache distribuído de alta performance** (tipicamente Redis) e garantir que a operação "verificar saldo + descontar fichas" seja **atômica** — ou acontece tudo de uma vez, ou não acontece nada.

No Redis, isso é feito com **scripts Lua** que rodam diretamente no servidor Redis, eliminando qualquer janela de race condition:

```lua
-- Script Lua rodando no Redis (operação atômica)
local key = KEYS[1]           -- chave do bucket do participante
local custo = tonumber(ARGV[1])  -- 1 para acerto, 20 para erro
local capacidade = tonumber(ARGV[2])
local recarga_por_segundo = tonumber(ARGV[3])

local agora = tonumber(redis.call('TIME')[1])
local bucket = redis.call('HMGET', key, 'fichas', 'ultimo_update')

local fichas = tonumber(bucket[1]) or capacidade
local ultimo_update = tonumber(bucket[2]) or agora

-- Recarrega fichas baseado no tempo passado
local delta = agora - ultimo_update
fichas = math.min(capacidade, fichas + delta * recarga_por_segundo)

if fichas >= custo then
    -- Saldo suficiente: desconta e aprova
    redis.call('HMSET', key, 'fichas', fichas - custo, 'ultimo_update', agora)
    return 1  -- aprovado
else
    -- Saldo insuficiente: rejeita
    redis.call('HSET', key, 'ultimo_update', agora)
    return 0  -- bloqueado (HTTP 429)
end
```

**Por que Lua no Redis?**

O Redis é single-threaded para execução de comandos, e scripts Lua rodam como uma única operação indivisível. Não existe estado intermediário que outra conexão possa observar.

```
Conexão A lê saldo: 20 fichas
Conexão B lê saldo: 20 fichas  ← sem Lua, ambas veem 20 e aprovam
...com Lua:
Conexão A executa script: lê + desconta = atômico
Conexão B executa script: encontra balde zerado = bloqueada ✓
```

---

## 6. Defesa em Camadas: PSP + Banco Central

Se o Banco Central detecta abuso vindo de um banco (PSP), ele bloqueia **a conexão daquele banco inteiro** — não apenas o IP do atacante. Isso significa que todos os clientes daquele banco ficam sem Pix.

Isso cria um incentivo fortíssimo para os bancos implementarem **suas próprias defesas internas** (circuit breaker, rate limiting, detecção de padrão de abuso) antes que as requisições maliciosas cheguem ao Banco Central.

```
Camada 1: Usuário/App → validações no frontend
Camada 2: Banco (PSP) → rate limit + circuit breaker próprio
Camada 3: Banco Central (DICT) → Weighted Rate Limiting + bloqueio por PSP
```

**Arquitetura de segurança em camadas:** cada camada protege a seguinte. Se todo participante faz sua parte, o sistema como um todo é resiliente.

---

## 7. Infraestrutura do Pix: Visão Geral

| Componente | Função |
|---|---|
| **DICT** | DNS financeiro — traduz chave Pix em conta bancária |
| **RSFN** | Rede física de fibra dedicada, isolada da internet pública |
| **ISO 20022** | Padrão de mensagem XML para transações financeiras |
| **HSM (Hardware Security Module)** | Assina digitalmente cada transação; qualquer alteração invalida a assinatura |
| **SPI** | Sistema de Pagamentos Instantâneos — núcleo de processamento |
| **Active-Active** | Múltiplos data centers espelhando transações em tempo real |

**Idempotência no SPI:**  
Cada transação tem um ID único e imutável. Se o Banco Central recebe o mesmo ID duas vezes (ex.: por falha de rede e retentativa), ele reconhece e ignora a duplicata — impedindo cobranças duplas.

```
POST /pagamento
  { "id": "uuid-abc-123", "valor": 50.00, ... }

Primeira vez  → processado ✓
Segunda vez (retry) → ID já existe → ignorado ✓
```

---

## Referências

- [Manual Operacional do DICT — Banco Central do Brasil](https://www.bcb.gov.br/content/estabilidadefinanceira/pix/Regulamento_Pix/II_ManualdePadroesTecniclosdoPix.pdf)
- [Martin Fowler — Rate Limiting Patterns](https://martinfowler.com/articles/patterns-of-distributed-systems/rate-limit.html)
- [Redis — Scripting with Lua](https://redis.io/docs/manual/programmability/eval-intro/)
