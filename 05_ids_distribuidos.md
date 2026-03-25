# 05 — IDs Distribuídos: O Problema que Derruba Sistemas em Escala

> **Caso real:** Instagram, 2011. Três engenheiros, 14 milhões de usuários, dezenas de bancos de dados shardados.  
> **Problema:** como gerar IDs únicos para fotos sem um coordenador central?  
> **Solução:** uma função PL/pgSQL de menos de 20 linhas com 3 operações de bit shift — hoje referência em entrevistas de System Design.

---

## 1. Por que IDs Únicos são Difíceis em Escala

Com um único banco de dados, IDs únicos são triviais: use `AUTO_INCREMENT`. O banco gera ID 1, ID 2, ID 3... sequencialmente, sem colisão.

O problema aparece com **sharding** — quando os dados são distribuídos entre múltiplos bancos independentes:

```
Shard A: INSERT foto → gera ID 1
Shard B: INSERT foto → gera ID 1  ← colisão!
Shard C: INSERT foto → gera ID 1  ← colisão!
```

Três fotos diferentes com o mesmo ID. Caos.

---

## 2. Abordagens Descartadas pelo Instagram

### 2.1 UUID (128 bits)
```
IDs do tipo: 550e8400-e29b-41d4-a716-446655440000
```
- 128 bits = dobro de 64 bits → índices maiores, mais lentos
- Não é ordenável por tempo (precisa de consulta extra para saber a ordem cronológica)
- Não carrega contexto do shard de origem

### 2.2 Ticket Server (Flickr)
Dois bancos dedicados gerando IDs alternados (par/ímpar):
```
Banco A: 1, 3, 5, 7, 9...
Banco B: 2, 4, 6, 8, 10...
```
- Funcional, mas é ponto único de falha — se os dois bancos caem, tudo para

### 2.3 Twitter Snowflake
Serviço dedicado para geração de IDs de 64 bits. O mais promissor, mas:
- Exige Zookeeper para coordenar os nós geradores
- Servidores dedicados = mais infraestrutura para operar
- Com 3 engenheiros, custo operacional proibitivo

---

## 3. A Solução do Instagram: Snowflake Embutido no PostgreSQL

Em vez de um serviço externo, o Instagram embutiu a lógica de geração de IDs **dentro do próprio PostgreSQL**, como uma função PL/pgSQL em cada schema.

### 3.1 Estrutura do ID de 64 bits

```
64 bits totais
├── 41 bits: timestamp (milissegundos desde a época do Instagram)
├── 13 bits: shard ID (identifica o shard de origem)
└── 10 bits: sequência local (auto-incremento dentro do milissegundo)
```

Visualizando:
```
Bit: 63        22 21       10 9        0
     [────────────][──────────][──────────]
     [ timestamp  ][ shard_id ][ sequence ]
     [  41 bits   ][  13 bits ][  10 bits ]
```

### 3.2 Por que 41 bits para timestamp?

O timestamp Unix começa em 1970, desperdiçando bits com décadas já passadas. O Instagram definiu uma **época própria**: 1º de janeiro de 2011.

Quantos anos 41 bits cobrem?
```
2^41 milissegundos = 2.199.023.255.552 ms
                   = ~69 anos a partir de 2011
                   = até ~2080
```

Como o timestamp fica nos bits mais significativos (à esquerda), IDs gerados depois são sempre numericamente maiores — **ordenação cronológica de graça**, sem consulta ao banco.

### 3.3 Por que 13 bits para shard?

```
2^13 = 8.192 shards possíveis
```
Com 2.000 shards lógicos do Instagram, sobra margem para crescimento enorme. Ao olhar para um ID qualquer, você extrai o shard de origem e já sabe em qual banco buscar — sem tabela de lookup, sem serviço de descoberta.

### 3.4 Por que 10 bits para sequência?

```
2^10 = 1.024 IDs por milissegundo por shard
```
Com 2.000 shards:
```
1.024 × 2.000 = 2.048.000 IDs por milissegundo
              = ~2 bilhões de IDs por segundo
```
Capacidade muito além do necessário para o Instagram de 2011 — ou de 2024.

---

## 4. A Montagem com Bit Shift

As três partes são combinadas com operações de deslocamento de bits:

```sql
-- PL/pgSQL simplificado (conceitual)
CREATE OR REPLACE FUNCTION next_id(OUT result BIGINT) AS $$
DECLARE
    epoch          BIGINT := 1314220021721;  -- 25/08/2011 em ms (época Instagram)
    seq_id         BIGINT;
    now_ms         BIGINT;
    shard_id       INT := 5;  -- ID do shard atual, configurado por schema
BEGIN
    -- Pega o auto-incremento local (módulo 1024)
    SELECT nextval('local_id_sequence') % 1024 INTO seq_id;
    
    -- Timestamp atual em ms desde a época
    now_ms := (EXTRACT(EPOCH FROM clock_timestamp()) * 1000)::BIGINT - epoch;

    -- Monta o ID de 64 bits:
    result := (now_ms   << 23)  -- timestamp nos 41 bits mais significativos
           OR (shard_id << 10)  -- shard_id nos 13 bits do meio
           OR seq_id;           -- sequência nos 10 bits finais
END;
$$ LANGUAGE plpgsql;
```

**O bit shift explicado:**

```
now_ms    = 0b...00000001101  (timestamp)
<< 23     = desloca 23 posições à esquerda
            abre espaço para shard_id (13 bits) + seq (10 bits)

shard_id  = 0b...00000101     (shard 5)
<< 10     = desloca 10 posições à esquerda
            abre espaço para seq (10 bits)

seq_id    = 0b...01001010     (sequência 74)
            não desloca — ocupa os 10 bits finais

OR de tudo = ID final de 64 bits ✓
```

---

## 5. Exemplo com Números Reais

```
Data: 09/09/2011 17:00:00 UTC

Milissegundos desde a época Instagram:
  now_ms = 1315594800000 - 1314220021721 = 1.374.778.279 ms

Shard: 
  user_id = 31.341 → 31.341 % 2000 = 1341

Sequência local:
  seq_id = 5001 % 1024 = 905

Montagem:
  timestamp (41 bits): 1374778279 << 23
  shard_id  (13 bits): 1341       << 10
  sequence  (10 bits): 905

  ID = 7514412775494959113
```

Qualquer sistema que receba esse ID pode extrair:
```python
EPOCH = 1314220021721

def decode_instagram_id(id: int):
    timestamp_ms = (id >> 23) + EPOCH
    shard_id     = (id >> 10) & 0b1111111111111  # 13 bits
    sequence     = id & 0b1111111111              # 10 bits
    return timestamp_ms, shard_id, sequence
```

---

## 6. Propriedades da Solução

| Propriedade | Como é garantida |
|---|---|
| **Unicidade global** | Combinação de timestamp + shard_id + sequência é única por design |
| **Ordenável por tempo** | Timestamp nos bits mais significativos → IDs mais novos são maiores |
| **Sem coordenação central** | Cada shard gera IDs de forma totalmente independente |
| **Sem ponto único de falha** | Lógica embutida no PostgreSQL, junto com os dados |
| **Roteamento pelo ID** | shard_id extraível do ID → sem lookup externo |
| **Alta capacidade** | ~2 bilhões de IDs/segundo com 2000 shards |

---

## 7. Quem Usa Variações Dessa Abordagem Hoje

| Sistema | Variação |
|---|---|
| **Twitter** | Snowflake: 41 bits timestamp + 10 bits machine ID + 12 bits sequence |
| **Discord** | Snowflake IDs adaptados para mensagens e canais |
| **Mastodon** | Variação do Snowflake para posts federados |
| **MongoDB** | ObjectID: 4 bytes timestamp + 5 bytes random + 3 bytes counter |
| **ULID** | Padrão moderno: 48 bits timestamp + 80 bits random, base32 |

---

## 8. A Lição de Engenharia

> **"Às vezes a engenharia mais impressionante não é a mais complexa — é a que resolve o problema inteiro com a menor quantidade possível de peças."**

O Instagram resolveu um problema que levaria muitos a construir um serviço distribuído complexo (com Zookeeper, heartbeats, failover, etc.) com uma função de banco de dados de 20 linhas.

Três princípios extraídos:
1. **Minimize coordenação** — cada dependência eliminada é resiliência ganha
2. **Faça o dado carregar seu próprio contexto** — eliminar um lookup é eliminar um servidor inteiro em escala
3. **Comece simples e dimensione para crescer** — você desenha pro futuro, mas paga só pelo presente

---

## Referências

- [Instagram Engineering Blog — Sharding IDs](https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c)
- [Twitter Snowflake — GitHub](https://github.com/twitter-archive/snowflake)
- [ULID Specification](https://github.com/ulid/spec)
