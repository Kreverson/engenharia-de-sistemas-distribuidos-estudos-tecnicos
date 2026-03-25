# 12 — Design Uber: Matching em Tempo Real e Geolocalização

> **Questão:** Desenhe um sistema como Uber — matching de passageiro com motorista em tempo real.  
> **Por que é difícil:** combina baixíssima latência de matching (<1 min), geospatial queries eficientes em alta escala, consistência de matching (um motorista = uma corrida), e gerenciamento de estado distribuído.

---

## 1. Requisitos

**Funcionais:**
- Usuário informa origem e destino → recebe estimativa de preço/ETA
- Usuário solicita corrida → é matched com motorista próximo disponível
- Motorista aceita/recusa → navega até o passageiro e depois ao destino

**Não Funcionais:**
- **Baixa latência de matching:** < 1 minuto para encontrar motorista ou reportar falha
- **Consistência de matching:** uma corrida só pode ser atribuída a UM motorista
- **Alta disponibilidade fora do matching:** estimativas, navegação, etc.
- **Alta escala:** 6 milhões de motoristas ativos → ~600.000 updates de localização/segundo

---

## 2. O Problema de Localização em Alta Escala

**Estimativa:**
```
6 milhões de motoristas ativos
Cada motorista envia localização a cada 5 segundos
→ 6.000.000 / 5 = 1.200.000 updates/segundo

Com otimização (Dynamic Location Updates):
  - Motorista parado → não envia
  - Motorista em corrida → intervalo maior
  - Motorista disponível em área sem demanda → intervalo maior
→ reduz para ~300.000-600.000 updates/segundo
```

**Por que PostgreSQL não é suficiente aqui:**
- PostgreSQL: ~2.000-4.000 writes/segundo em hardware comum
- Precisamos de ~600.000/segundo
- Índices B-tree são otimizados para 1 dimensão — lat/lon é 2D

---

## 3. Geohashing vs. QuadTree

### Geohashing
Divide o mundo em uma grade de células recursivas, cada uma codificada como string:

```
Divisão em 4 quadrantes (base-4 simplificada):
  Mundo → [0, 1, 2, 3]
  Zoom in → [00, 01, 02, 03, 10, 11, ...]
  Mais zoom → [000, 001, ...]

"9q5cs" = latitude -23.5, longitude -46.6 (São Paulo, precisão ~5km)
"9q5csk" = mais preciso (~150m)
"9q5cskg" = muito preciso (~5m)
```

**Propriedade chave:** células geograficamente próximas tendem a ter prefixos similares. Para buscar "motoristas próximos", busco pela célula do usuário + células vizinhas.

**Vantagem:** fácil de calcular, fácil de armazenar (string), não requer estrutura de dados separada. Redis suporta nativamente geohashing!

### QuadTree
Divide o espaço recursivamente em 4 quadrantes, mas apenas quando a densidade de pontos excede um limiar K:

```
Região com 3 motoristas (K=5): não divide
Região com 7 motoristas (K=5): divide em 4 sub-regiões
  ├── Sub-região NW: 2 motoristas → não divide
  ├── Sub-região NE: 1 motorista  → não divide
  ├── Sub-região SW: 0 motoristas → não divide
  └── Sub-região SE: 4 motoristas → não divide
```

**Vantagem:** adapta-se à densidade — São Paulo tem muito mais granularidade que o interior do Pará.

**Desvantagem:** estrutura de dados em memória que precisa ser reindexada a cada update — caro para alta frequência de writes.

### Quando usar cada um?

| | Geohashing | QuadTree |
|---|---|---|
| **Writes frequentes** | ✅ Excelente | ❌ Caro (reindexação) |
| **Distribuição irregular** | Menos eficiente | ✅ Adapta à densidade |
| **Implementação** | Simples | Complexa |
| **Redis suporte nativo** | ✅ Sim | ❌ Não |

**Para Uber → Geohashing + Redis** (alta frequência de writes, distribuição em clusters mas não extremamente desigual)

```python
# Redis Geo commands
redis.geoadd("drivers", longitude, latitude, driver_id)

# Buscar motoristas num raio de 5km
nearby = redis.georadius("drivers", user_lng, user_lat, 5, "km")
```

**Redis aguenta:** 100.000-1.000.000 writes/segundo em cluster — suficiente.

---

## 4. Dynamic Location Updates (Otimização)

Em vez de enviar a cada 5 segundos fixos, o app do motorista é inteligente:

```python
# No app do motorista (pseudocódigo)
def decide_send_update(driver):
    if driver.status == OFFLINE:
        return False  # não envia
    
    if driver.status == IN_RIDE:
        return time_since_last_update() > 15  # a cada 15s
    
    if driver.speed < 5:  # parado
        return time_since_last_update() > 30  # a cada 30s
    
    if nearby_demand_score < 0.1:  # área sem demanda
        return time_since_last_update() > 10  # a cada 10s
    
    return time_since_last_update() > 5  # padrão: 5s
```

Isso reduz o volume de updates em ~70% sem impacto perceptível na experiência.

---

## 5. Matching: Consistência sem Double-Assignment

**Problema:** servidor A e servidor B processam duas solicitações diferentes, ambos buscam motoristas próximos, ambos encontram o motorista M. Sem sincronização, ambos enviam notificação para M.

**Solução: Distributed Lock com Redis TTL**

```python
# Ride Matching Service ao selecionar motorista:
lock_key = f"driver_lock:{driver_id}"
lock_ttl = 10  # segundos para o motorista aceitar/recusar

# Tentativa de adquirir o lock (atômica)
acquired = redis.set(lock_key, ride_id, nx=True, ex=lock_ttl)

if acquired:
    # Lock obtido — enviar notificação para o motorista
    send_push_notification(driver_id, ride_request)
    # Aguardar resposta por até 10 segundos
else:
    # Driver já está sendo processado por outra requisição
    # Tentar próximo motorista disponível
    next_driver = get_next_available_driver()
```

**Pseudocódigo do loop de matching:**
```python
def find_driver(ride_request):
    nearby_drivers = redis.georadius(
        "drivers", ride_request.origin_lng, ride_request.origin_lat, 
        3, "km"
    )
    
    for driver_id in nearby_drivers:
        # Verificar se está disponível (não em corrida, não locked)
        if not is_available(driver_id):
            continue
        
        # Tentar adquirir lock
        if redis.set(f"lock:{driver_id}", ride_request.id, nx=True, ex=10):
            send_ride_request_to_driver(driver_id, ride_request)
            
            # Aguardar resposta (10 segundos)
            response = await_driver_response(driver_id, timeout=10)
            
            if response == ACCEPTED:
                update_ride_status(ride_request.id, driver_id)
                return driver_id
            else:
                redis.delete(f"lock:{driver_id}")
                # Tenta próximo motorista
    
    return None  # Nenhum motorista disponível
```

---

## 6. Ride Request Queue para Surges

**Problema:** após um grande evento (show, jogo), há um pico súbito de solicitações.

**Solução:** inserir uma fila entre o usuário e o matching service.

```
Usuários → API Gateway → Ride Request Queue (SQS/Kafka) → Matching Service
```

A fila:
- Absorve picos (não sobrecarrega o Matching Service)
- Permite retry em caso de falha do Matching Service
- Pode ser particionada por região geográfica

---

## 7. Estimativa de Preço/ETA

Para simplicidade: usar Google Maps API ou similar.
```
POST /rides/estimate
body: { origin: {lat, lng}, destination: {lat, lng} }
→ { price: R$32, eta: "8 min" }
```

Em produção: modelo de ML com variáveis como:
- Tráfego em tempo real
- Demanda x oferta na região (surge pricing)
- Tipo do veículo
- Distância + tempo estimado

---

## 8. Fluxo Completo de uma Corrida

```
1. Passageiro abre app → localização capturada
2. GET /rides/estimate → retorna preço + ETA
3. Passageiro confirma → POST /rides/request
   → cria ride com status "searching"
   → coloca na Ride Request Queue
4. Matching Service consome da fila
   → busca motoristas próximos no Redis
   → para cada candidato, tenta lock
   → envia push notification (APNs/FCM)
5. Motorista aceita → 
   → lock fica no Redis até fim da corrida
   → ride status = "matched"
   → PATCH /rides/:id { driver_id, status: "matched" }
6. Passageiro recebe notificação em tempo real (SSE/polling)
7. Motorista navega → atualiza localização a cada N segundos
8. Passageiro embarcado → PATCH /rides/:id/status { status: "in_ride" }
9. Destino atingido → PATCH /rides/:id/status { status: "completed" }
   → libera lock do motorista no Redis
```

---

## 9. Arquitetura Final

```
Client (Passageiro + Motorista)
  ↓
API Gateway
  ├── Ride Service     → PostgreSQL (estado das corridas)
  ├── Location Service → Redis (geohash dos motoristas)
  ├── Matching Service ← Ride Request Queue (SQS)
  │     └── Redis (distributed locks)
  └── Notification     → APNs / FCM

Região separada → réplica completa por cidade/país
```

---

## Referências

- [Uber Engineering Blog — H3 (Geospatial Indexing)](https://eng.uber.com/h3/)
- [Redis — GEOADD / GEORADIUS](https://redis.io/commands/georadius/)
- [Hello Interview — Uber Breakdown](https://hellointerview.com)
