# 14 — Design Ad Click Aggregator: Stream Processing e Arquiteturas Lambda/Kappa

> **Questão:** Desenhe um sistema que coleta e agrega cliques em anúncios, permitindo que anunciantes consultem métricas em tempo real.  
> **Por que é interessante:** combina ingestão de alta escala, stream processing em tempo real, reconciliação para garantia de integridade, e as arquiteturas Lambda e Kappa.

---

## 1. Requisitos

**Funcionais:**
- Usuário clica em um anúncio → é redirecionado para o site do anunciante
- Anunciante consulta métricas de cliques por período (granularidade mínima: 1 minuto)

**Não Funcionais:**
- Escala: 10 milhões de anúncios, **10.000 cliques/segundo no pico**
- Latência de queries < 1 segundo
- Dados em tempo real (granularidade de 1 minuto)
- Fault tolerant: não podemos perder cliques (impacto financeiro direto)
- Idempotência: um usuário não pode inflar artificialmente o contador de um anúncio

---

## 2. O Redirecionamento (Garantindo que o Clique é Registrado)

**Abordagem ingênua (insegura):**
```html
<a href="https://nike.com">Compre agora</a>
```
O usuário acessa nike.com diretamente sem passar pelo servidor de ads → clique não registrado.

**Abordagem correta:**
```
1. O ad placement service envia apenas o AD_ID (não a URL de destino)
2. Usuário clica → POST /clicks { adId: "nike_123" }
3. Servidor registra o clique, busca a redirect URL no banco
4. Servidor responde com HTTP 302 → https://nike.com
```

```
Client                          Ad Server
  |-- POST /clicks {adId} -----→|
  |                             |-- lookup redirect URL
  |                             |-- log click
  |←-- 302 → nike.com ----------|
  |-- GET https://nike.com --→ Nike
```

Assim, o clique SEMPRE passa pelo servidor antes do usuário chegar ao destino.

---

## 3. Por que a Arquitetura Simples Não Escala

**Abordagem ingênua (armazenar + agregar no banco):**
```sql
-- Tabela de cliques brutos
CREATE TABLE click_events (
    event_id  UUID,
    ad_id     VARCHAR,
    user_id   VARCHAR,
    timestamp TIMESTAMP
);

-- Query de anunciante (lenta!)
SELECT ad_id, COUNT(*), COUNT(DISTINCT user_id)
FROM click_events
WHERE ad_id = 'nike_123' 
  AND timestamp > NOW() - INTERVAL '1 week'
GROUP BY ad_id, DATE_TRUNC('minute', timestamp);
```

Com 10.000 cliques/seg, isso é **864 milhões de registros por dia**. Essa query de agregação seria muito lenta para retornar em < 1 segundo.

---

## 4. Solução: Stream Processing com Flink

**Ideia:** em vez de agregar em query-time, pré-agregar em write-time usando um stream processor.

```
Click Events
    ↓
[Kinesis/Kafka Stream]   ← particionado por ad_id
    ↓
[Apache Flink]           ← agrega em janelas de 1 minuto
    ↓
[OLAP Database]          ← já armazena dados agregados, pronto para query
```

### Como o Flink funciona neste contexto

O Flink mantém um **estado em memória** para cada janela de tempo:

```
Flink recebe eventos:
  T=10:00:00 → clique no ad nike_123
  T=10:00:15 → clique no ad nike_123
  T=10:00:45 → clique no ad adidas_456
  T=10:00:58 → clique no ad nike_123

Estado em memória após processar:
  { "nike_123": { minute: "10:00", clicks: 3, unique_users: 2 } }
  { "adidas_456": { minute: "10:00", clicks: 1, unique_users: 1 } }

T=10:01:00 → janela fechada!
  → Flush para OLAP database:
    INSERT INTO ad_metrics(ad_id, minute, clicks, unique_users)
    VALUES ('nike_123', '2024-01-01 10:00:00', 3, 2),
           ('adidas_456', '2024-01-01 10:00:00', 1, 1)

Próxima janela começa...
```

**Flush interval vs. Aggregation window:**
- **Aggregation window:** 1 minuto (quando fechar a janela definitivamente)
- **Flush interval:** 10 segundos (escrever dados parciais frequentemente)

Com flush interval de 10s, um anunciante pode ver dados quase em tempo real (com indicação de "parcial" para o minuto atual).

---

## 5. Idempotência de Cliques (Anti-Fraude Básico)

**Problema:** bots ou usuários maliciosos que clicam 100 vezes no mesmo anúncio.

**Solução: Ad Impression ID assinado**

```
1. Ad placement service gera um impression_id único por exibição
2. Impression ID é assinado com chave privada do servidor
3. Quando o usuário clica, envia: { adId, impressionId, signature }
4. Click processor verifica a assinatura
5. Verifica se impressionId já foi usado (Redis lookup)
6. Se não usado → registra clique + adiciona impressionId ao Redis com TTL
7. Se já usado → descarta (clique duplicado)
```

```python
# Click processor
def process_click(ad_id, impression_id, signature):
    # 1. Verificar assinatura
    if not verify_signature(impression_id, signature, PUBLIC_KEY):
        return error("Invalid impression")
    
    # 2. Verificar duplicata (Redis com TTL de 24h)
    if redis.get(f"impression:{impression_id}"):
        return error("Duplicate click")
    
    # 3. Registrar impressão como usada
    redis.set(f"impression:{impression_id}", "1", ex=86400)
    
    # 4. Publicar no stream
    kinesis.put_record(ad_id=ad_id, user_id=get_user_id())
    
    # 5. Retornar redirect
    return redirect(get_ad_url(ad_id))
```

---

## 6. Fault Tolerance e Reconciliação

### Retention Policy no Kafka/Kinesis

```yaml
# Kinesis: manter eventos por 7 dias
retention_period: 7 days
```

Se o Flink cair, ao reiniciar ele pode reprocessar os últimos 7 dias do stream usando o cursor (offset) de onde parou.

### Por que NÃO usar Checkpointing do Flink aqui

Checkpointing salva o estado do Flink em S3 periodicamente para recovery:
```
T=10:00 → processa eventos
T=10:15 → checkpoint (salva estado em S3)
T=10:17 → Flink cai
T=10:18 → Flink reinicia → restaura estado do checkpoint das 10:15
         → apenas 2 minutos de re-processamento necessário
```

**Por que não é necessário aqui:** nossa janela de agregação é apenas 1 minuto. Se o Flink cair, no máximo precisamos reprocessar 1 minuto de eventos do stream — trivial. Checkpointing seria overhead desnecessário.

> **Regra prática:** checkpointing vale a pena quando a aggregation window > 5 minutos.

### Reconciliação Periódica (Batch)

Para garantir integridade máxima, rodar um job batch periódico (diário):

```
1. Kinesis despeja eventos brutos no S3 automaticamente
2. Spark lê todos os eventos do S3 do dia anterior
3. Calcula agregações "ground truth"
4. Reconciliation Worker compara com dados no OLAP DB
5. Se divergência → corrigir + alertar a equipe
```

```
Stream Processing (Flink):  quase em tempo real, pode ter small errors
Batch Processing (Spark):   exato, mas atrasado (D+1)

Combinação → resultado final sempre correto
```

---

## 7. Arquiteturas Lambda e Kappa

### Arquitetura Lambda

Tem **duas camadas** paralelas:
```
Events → [Speed Layer (stream)]  → resultados em tempo real (aproximados)
       → [Batch Layer (Hadoop)]  → resultados exatos (mas atrasados)

Query Layer → merge das duas camadas
```

**Problema:** código duplicado, duas stacks para manter.

### Arquitetura Kappa

Tem **apenas a camada de stream**:
```
Events → [Stream Processing] → resultados em tempo real

Para recalcular: replay o stream histórico com lógica nova
```

**Problema:** stream deve ter retenção longa; recálculo pode ser lento.

### Nossa Abordagem: Híbrido Pragmático

```
Normal:     [Flink]     → OLAP DB (quase tempo real)
Reconcília: [Spark/D+1] → corrige divergências no OLAP DB
```

Não é nem Lambda nem Kappa puro — é a solução mais prática para este problema específico.

> **Lição:** padrões arquiteturais são guias, não leis. Adapte conforme o problema real.

---

## 8. Hot Shard Problem

Se Nike lança um anúncio com o LeBron James, 99% dos cliques vão para o mesmo ad_id, sobrecarregando uma única partição do Kinesis.

**Solução: Compound Key para Hot Shards**
```python
# Click processor detecta hot ads
if is_hot_ad(ad_id):
    # Distribui em N=10 partições
    random_suffix = random.randint(0, 9)
    partition_key = f"{ad_id}_{random_suffix}"
else:
    partition_key = ad_id

kinesis.put_record(partition_key=partition_key, data=click_event)
```

**Flink:** um único job/task por ad_id ainda é responsável pela agregação, mesmo com múltiplas partições de origem. O Flink lê das N partições e agrega internamente.

---

## 9. OLAP Database

**Por que OLAP?** Queries de anunciantes são tipicamente aggregations sobre grandes volumes:
```sql
SELECT 
    DATE_TRUNC('hour', minute) as hour,
    SUM(clicks) as total_clicks,
    SUM(unique_users) as reach
FROM ad_metrics
WHERE ad_id = 'nike_123' 
    AND minute BETWEEN '2024-01-01' AND '2024-01-08'
GROUP BY 1
ORDER BY 1;
```

Essas queries são perfeitamente adequadas para OLAP (columnar storage, otimizado para aggregations).

Exemplos: ClickHouse, Apache Druid, BigQuery, Redshift.

---

## 10. Arquitetura Final

```
Browser
  ↓ (clique)
Load Balancer → Click Processor
  ├── Verifica assinatura do impression_id
  ├── Redis: dedup de cliques
  ├── Retorna HTTP 302 (redirect)
  └── Publica no Kinesis (particionado por ad_id)
       ↓
  Apache Flink
  ├── Janela de 1 minuto por ad_id
  ├── Flush parcial a cada 10 segundos
  └── Grava no OLAP DB
       ↓
  OLAP Database
       ↑ query
  Advertiser Dashboard (< 1s latência)

Reconciliação (diária):
  Kinesis → S3 → Spark → Reconciliation Worker → OLAP DB (correção)
```

---

## Referências

- [Apache Flink — Windowing](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/operators/windows/)
- [Lambda Architecture — Nathan Marz](https://nathanmarz.com/blog/how-to-beat-the-cap-theorem.html)
- [Jay Kreps — Questioning the Lambda Architecture](https://www.oreilly.com/radar/questioning-the-lambda-architecture/)
