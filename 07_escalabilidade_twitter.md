# 07 — Escalabilidade Evolutiva: A Jornada do Twitter

> **Caso real:** Twitter/X, 2006–2013. De um monolito Ruby on Rails com 3 engenheiros a 500 milhões de tweets/dia.  
> **A lição central:** arquitetura não é uma decisão — é uma série de respostas a problemas reais. Cada peça foi adicionada porque algo concreto quebrou.

---

## 1. O Ponto de Partida: Monolito

```
2006 — "Monorail" (Ruby on Rails)

┌─────────────────────────┐
│         Monolito        │
│  ┌─────┐ ┌────────┐     │
│  │ Web │ │Lógica  │     │
│  │     │ │Negócio │     │
│  └─────┘ └────────┘     │
│          ┌────────┐     │
│          │ MySQL  │     │
│          └────────┘     │
└─────────────────────────┘
```

Funciona bem no começo. Um servidor, um banco, tudo junto. O problema é o **acoplamento**: a mesma thread que renderiza HTML também conecta no banco e serve a API. Uma query pesada trava toda a aplicação.

---

## 2. Escala Vertical → Horizontal

**Vertical (scale up):** máquina maior, mais CPU, mais RAM.
- Rápido de implementar (sem mudar código)
- Tem teto físico
- Custo sobe de forma não-linear (dobrar capacidade custa 3-4x mais)
- Continua sendo ponto único de falha

**Horizontal (scale out):** múltiplas máquinas menores em paralelo.
- Requer um mecanismo para distribuir o tráfego: **Load Balancer**

```
Internet → Load Balancer → Servidor 1 (cópia do monolito)
                         → Servidor 2 (cópia do monolito)
                         → Servidor 3 (cópia do monolito)
```

O Load Balancer distribui requisições entre os servidores. O algoritmo mais simples é **Round Robin**: a cada requisição, passa para o próximo servidor da lista.

**O problema de sessão:** o usuário loga no Servidor 1, a sessão fica na memória do Servidor 1. A próxima requisição vai para o Servidor 2 — que não conhece a sessão.

**Solução:** servidores **stateless** — nenhum estado local. A sessão vai para um armazenamento compartilhado (Redis):

```
Servidor 1, 2, 3 → todos consultam o mesmo Redis para sessões
Qualquer servidor pode atender qualquer requisição ✓
```

> **Regra de ouro:** se um servidor pode ser destruído e recriado sem perder nada, ele é stateless. Stateless = escalabilidade horizontal trivial.

---

## 3. Cache: A Primeira Linha de Defesa do Banco

Antes de escalar o banco (caro e complexo), coloque um cache na frente.

```
App → Redis/Memcached → [HIT]  → retorna dado em microsegundos
                      → [MISS] → consulta MySQL → armazena no cache → retorna
```

O Twitter mantinha a timeline de cada usuário como uma lista de até 800 tweet IDs no Redis. Ler a timeline = GET numa lista = milissegundos.

**Taxa de hit de 80%** significa que 80% das requisições ao banco são eliminadas.

**Cache Stampede:** quando uma chave popular expira, centenas de requisições descobrem o MISS ao mesmo tempo e todas vão ao banco simultaneamente. Soluções:
- **Mutex**: apenas uma requisição vai ao banco; as outras esperam
- **TTL com jitter**: adiciona variação aleatória ao tempo de expiração (evita expiração em massa)
- **Probabilistic early refresh**: começa a renovar o cache antes de expirar, com probabilidade crescente conforme o TTL se aproxima

---

## 4. Réplicas de Leitura

O Twitter é uma plataforma massivamente de leitura: **50x mais leituras que escritas** (300.000 req/s de leitura vs 6.000/s de escrita).

**Réplicas de leitura:** cópias do banco primário que servem apenas leituras. O primário aceita escritas e replica para as réplicas.

```
Escritas → MySQL Primário
              ↓ replicação assíncrona
Leituras → Réplica 1
         → Réplica 2
         → Réplica 3
```

**Tradeoff — Replication Lag:** a réplica pode estar milissegundos atrás do primário. Você posta um tweet e não vê seu próprio tweet porque leu de uma réplica desatualizada.

**Solução:** após uma escrita, direcione as leituras do mesmo usuário para o primário por alguns segundos ("read-your-own-writes").

---

## 5. Sharding de Escritas com Gizzard

Réplicas resolvem leitura. Mas escritas continuam batendo em um único primário.

O Twitter criou o **Gizzard** — um middleware de sharding que fica entre a aplicação e o MySQL. A aplicação envia queries para o Gizzard, que consulta uma tabela de roteamento e encaminha para o shard correto.

```
App → Gizzard → Tabela de roteamento → Shard A (user_id 1-1M)
                                      → Shard B (user_id 1M-2M)
                                      → Shard C (user_id 2M-3M)
```

**Idempotência obrigatória:** o Gizzard exige que todas as escritas sejam idempotentes — executar a mesma operação múltiplas vezes produz o mesmo resultado. Isso permite reenvio seguro em caso de falha sem duplicar dados.

```
# Idempotente ✓
UPDATE tweets SET likes = 42 WHERE id = 123

# NÃO idempotente ✗
UPDATE tweets SET likes = likes + 1 WHERE id = 123
(reenviar incrementa duas vezes)
```

---

## 6. Filas Assíncronas e o Kestrel

Até aqui, tudo era **síncrono**: o usuário posta um tweet, o servidor processa notificações, indexação para busca, entrega por SMS, atualização de contadores... e só depois responde.

Se qualquer passo demorar, o usuário espera. Se o serviço de SMS estiver lento, o tweet "trava".

**Solução: desacoplar o recebimento do processamento.**

```
Usuário posta tweet
  → Servidor aceita e responde "OK" imediatamente ao usuário
  → Joga tarefas em uma fila
     ↓
     Worker 1: envia notificações push
     Worker 2: entrega por SMS
     Worker 3: indexa para busca
     Worker 4: atualiza contadores
```

O Twitter criou o **Kestrel** — uma fila de mensagens escrita em Scala. O tweet só é removido da fila quando o worker confirma que processou (acknowledgment). Se o worker cai antes de confirmar, a mensagem retorna à fila para ser processada por outro worker.

> **Filas transformam sistemas síncronos em assíncronos. Sem elas, Fanout era impossível.**

---

## 7. O Problema Central: Fanout da Timeline

Como montar a timeline de cada usuário? Esta é a operação mais desafiadora do Twitter.

### 7.1 Fanout on Read (pull)

```
Usuário abre o app
  → Busca lista de quem segue (500 pessoas)
  → Busca tweets recentes de cada uma (500 queries)
  → Join, ordena por data
  → Retorna os mais recentes
```

- **Postar é barato:** armazena o tweet uma vez
- **Ler é caro:** join gigante a cada abertura do app
- Com 300.000 req/s de timeline, o banco explode

### 7.2 Fanout on Write (push)

Inverte a lógica: faz o trabalho pesado na escrita, para que a leitura seja instantânea.

```
Usuário posta tweet
  → Fan-out Service consulta quem segue esse usuário (FlocDB)
  → Insere o tweet ID na timeline Redis de CADA seguidor
```

```
Timeline de cada usuário no Redis = lista de IDs já prontos
Ler timeline = GET de uma lista = milissegundos ✓
```

- **Postar é caro:** 10.000 seguidores = 10.000 escritas no Redis
- **Ler é barato:** independente de quantas contas você segue

A conta fecha porque a leitura é 50x mais frequente que a escrita. Gastar mais na escrita para baratear 300.000 leituras/segundo faz sentido.

### 7.3 O Problema das Contas com Milhões de Seguidores

Fanout on Write falha para contas com 50M de seguidores:
- 50 milhões de escritas no Redis a cada tweet
- O Twitter levava até **5 minutos** para distribuir um tweet de conta grande
- Seguidores viam respostas antes do tweet original

**Solução híbrida:**

```
Conta comum (< ~threshold de seguidores):
  → Fanout on Write (push para Redis de cada seguidor)

Conta grande (celebridade, presidente, etc.):
  → Fanout on Read (tweet armazenado uma vez; buscado dinamicamente)
  → Na leitura: mescla timeline pré-computada do Redis
                com tweets de contas grandes buscados na hora
```

Não é uma solução limpa — tem dois caminhos de código, lógica de merge em milissegundos. Mas resolve os dois extremos.

---

## 8. De Monolito a Microsserviços

A Copa do Mundo de 2010 derrubou o Twitter repetidamente. O diagnóstico: o monolito Rails não escalava. Não bastava adicionar mais máquinas — o código era o problema.

**Migração para JVM (Java + Scala):** a equipe esperava melhoria de 10x. O resultado superou: redução de 3x na latência de busca.

**Decomposição em serviços:**
```
Monolito: um código que faz tudo

Microsserviços:
  ├── Serviço de Tweets (armazenar, buscar)
  ├── Serviço de Usuários (perfis, autenticação)
  ├── Serviço de Timeline (montar, servir)
  ├── Fan-out Service (distribuir tweets)
  ├── Serviço de Busca
  └── Serviço de Notificações
```

Cada serviço tem deploy independente. Falha no serviço de usuários não derruba o fan-out.

### Três novos problemas que aparecem com microsserviços:

**1. Service Discovery:** como um serviço encontra o outro?
→ Twitter usou **ZooKeeper**: ao subir, cada serviço se registra. Para chamar outro, pergunta ao ZooKeeper o endereço atual.

**2. Falhas em cascata:** se o Serviço A depende do B, e B fica lento, A acumula conexões esperando → A trava → cascata.
→ **Circuit Breaker** (ver arquivo 06): isola o serviço problemático, falha rápido, protege o chamador.

**3. Observabilidade:** com dezenas de serviços, onde está o gargalo?
→ Twitter criou o **Zipkin** (open-sourced 2012): sistema de distributed tracing, inspirado no paper Dapper do Google. Cada requisição recebe um trace ID; cada serviço registra quanto tempo gastou. Resultado: timeline visual de toda a cadeia.

---

## 9. A Evolução Completa

```
2006: Monolito Rails + 1 MySQL
  ↓ CPU e RAM no limite
2007: Escala vertical (máquinas maiores)
  ↓ teto físico, sem eliminar acoplamento
2008: Load balancer + múltiplos servidores + Redis para sessão
  ↓ banco vira gargalo
2009: Cache (Memcache/Redis) + réplicas de leitura
  ↓ escritas saturam o primário
2010: Sharding com Gizzard + filas com Kestrel
  ↓ timeline query impossível de escalar
2011: Fanout on Write para contas normais + modelo híbrido para contas grandes
  ↓ monolito Rails acoplado, sem escala de código
2012: Migração para JVM, decomposição em microsserviços
  ↓ service discovery, falhas em cascata, observabilidade
2013: ZooKeeper + Circuit Breaker + Zipkin. Zero downtime em 7 de 11 meses.
```

> **Cada passo foi forçado por um gargalo real. Nada foi feito de uma vez.**

---

## Referências

- [Twitter Engineering Blog](https://blog.twitter.com/engineering)
- [Zipkin — Distributed Tracing](https://zipkin.io/)
- [Google Dapper Paper](https://research.google/pubs/pub36356/)
- [High Scalability — Twitter Architecture](http://highscalability.com/blog/2013/7/8/the-architecture-twitter-uses-to-deal-with-150m-active-users.html)
