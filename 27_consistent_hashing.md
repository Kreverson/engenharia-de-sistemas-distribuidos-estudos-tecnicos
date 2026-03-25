# 27 — Consistent Hashing

> A solução elegante para distribuição e rebalanceamento de dados em sistemas distribuídos

---

## O Problema: Hash com Módulo

### Cenário Inicial

```
3 bancos de dados: DB1, DB2, DB3
Distribuir eventos entre eles:

evento_id → hash(evento_id) % 3 → DB (0, 1 ou 2)

Exemplo: evento "1234" → hash("1234") = 67211 → 67211 % 3 = 2 → armazenar no DB2
```

### O Problema ao Escalar

Ao adicionar um 4º banco, o módulo muda de 3 para 4:

```
Antes: hash(id) % 3
Depois: hash(id) % 4

Evento "1234": 67211 % 3 = 2 (estava no DB2)
               67211 % 4 = 3 (precisa ir para DB3)
```

**Resultado:** quase **todos os dados** precisam ser redistribuídos — não apenas os do novo banco.

**Impacto:** surge em escala de crescimento (adicionar banco) e redução (desativar banco):
- Pico de I/O nos bancos durante a redistribuição
- Lentidão ou crash do sistema
- Downtime para migração

---

## A Solução: Consistent Hashing

### Como Funciona

**Passo 1: Criar o Hash Ring**

```
Ring com posições de 0 a 100 (na prática, 0 a 2^32)

       0
   90     10
 80          20
 70          30
   60     40
       50
```

**Passo 2: Posicionar os bancos no ring**

```
4 bancos → posições: DB1=0, DB2=25, DB3=50, DB4=75 (distribuídos uniformemente)

       DB1(0)
   DB4(75)  DB2(25)
      
         DB3(50)
```

**Passo 3: Mapear itens para bancos**

```
Para armazenar evento com hash=16:
1. Encontre a posição 16 no ring
2. Caminhe no sentido horário até encontrar o próximo banco
3. Evento fica em DB2(25)

Para hash=80:
1. Posição 80 no ring
2. Sentido horário → encontra DB1(0) [ring é circular, volta ao início]
3. Evento fica em DB1
```

---

## Por que Consistent Hashing é Melhor

### Adicionando um banco

```
Adicionando DB5 na posição 90:

Apenas eventos na faixa [75, 90] precisam ser redistribuídos:
- Antes: iam para DB1(0)  
- Agora: vão para DB5(90)

Todos os outros eventos: permanecem onde estão ✓
```

**Proporção redistribuída:** ~1/N dos dados (onde N = número de bancos)  
**Hash com módulo:** ~(N-1)/N dos dados (quase tudo)

### Removendo um banco

```
Removendo DB2(25):

Apenas eventos na faixa [0, 25] precisam ser redistribuídos:
- Antes: iam para DB2(25)
- Agora: vão para DB3(50)

Todos os outros eventos: permanecem ✓
```

---

## Virtual Nodes: Distribuição Equilibrada

### O Problema sem Virtual Nodes

```
Remover DB2(25): todos os dados de DB2 vão para DB3

DB3 antes: dados de [25, 50]
DB3 depois: dados de [0, 50] = 2× os dados de DB1 e DB4

Carga desequilibrada ❌
```

### Solução: Múltiplas Posições por Nó

```
Em vez de 1 posição por banco, usar N posições virtuais:

DB1: posições 0, 20, 40, 60, 80
DB2: posições 5, 25, 45, 65, 85
DB3: posições 10, 30, 50, 70, 90
DB4: posições 15, 35, 55, 75, 95
```

**Ao remover DB2:**

```
Faixa [0, 5]  → vai para DB3 (próximo após posição 0)
Faixa [5, 10] → vai para DB4 (próximo após posição 5)  
Faixa [15, 25] → vai para DB1 (próximo após posição 15)
Faixa [25, 30] → vai para DB3 (próximo após posição 25)
...

Redistribuição espalhada entre múltiplos bancos → distribuição uniforme ✓
```

**Mais virtual nodes = melhor equilíbrio, mas mais memória para o mapeamento.**

---

## Implementação Conceitual

```python
import hashlib
import bisect

class ConsistentHashRing:
    def __init__(self, nodes, virtual_nodes=100):
        self.ring = {}
        self.sorted_keys = []
        
        for node in nodes:
            for i in range(virtual_nodes):
                key = self._hash(f"{node}:{i}")
                self.ring[key] = node
                bisect.insort(self.sorted_keys, key)
    
    def _hash(self, key):
        return int(hashlib.md5(key.encode()).hexdigest(), 16)
    
    def get_node(self, key):
        if not self.ring:
            return None
        
        hash_key = self._hash(key)
        idx = bisect.bisect(self.sorted_keys, hash_key) % len(self.sorted_keys)
        return self.ring[self.sorted_keys[idx]]
```

---

## Quando Consistent Hashing Aparece em Interviews

### Bastidores (não é necessário descrever)

A maioria dos bancos e sistemas modernos usa consistent hashing internamente:
- **Redis Cluster**: sharding automático
- **Cassandra**: particionamento nativo
- **DynamoDB**: hashing nas partition keys
- **CDNs**: roteamento de requests para edge nodes

### Quando descrever explicitamente

Você precisa explicar o algoritmo quando estiver **projetando o componente distribuído** em si:

| Problema | Consistent Hashing relevante? |
|---|---|
| Design de cache distribuído | ✅ Sim |
| Design de banco de dados distribuído | ✅ Sim |
| Design de message queue distribuída | ✅ Sim |
| Design de sistema de arquivos distribuído | ✅ Sim |
| Design de feature como "sistema de recomendação" | ❌ Não (apenas mencionar que usa DynamoDB/Redis) |

---

## Consistent Hashing em Sistemas de Chat (Exemplo Real)

Ao escalar servidores de WebSocket para um sistema de chat:

```
Problema: usuário A (Server 1) quer enviar mensagem para usuário B (Server 2)
          cada server tem seu hashmap de conexões ativas

Solução com Consistent Hash Ring:
- Cada usuário é mapeado deterministicamente para um servidor
- Mensagens são roteadas para o servidor "dono" do usuário destinatário
- Server-to-server communication apenas quando necessário

Chat Registry (ex: Zookeeper/etcd) mantém o mapeamento atual
```

---

## Comparativo: Hash Simples vs. Consistent Hashing

| Aspecto | Hash % N | Consistent Hashing |
|---|---|---|
| Adicionar nó | Redistribui ~(N-1)/N dos dados | Redistribui ~1/N dos dados |
| Remover nó | Redistribui ~(N-1)/N dos dados | Redistribui ~1/N dos dados |
| Distribuição | Uniforme (ideal) | Aproximadamente uniforme (melhor com virtual nodes) |
| Complexidade | O(1) lookup | O(log N) lookup |
| Implementação | Trivial | Moderada |

---

## Resumo

```
Sem consistent hashing:
  Scale up → redistribuir 80% dos dados → pico de I/O → downtime ❌

Com consistent hashing:
  Scale up → redistribuir ~20% dos dados → operação gradual ✓

Virtual nodes:
  Distribuição uniforme ao adicionar/remover → carga balanceada ✓
```

---

*Fonte: Hello Interview — Consistent Hashing*
