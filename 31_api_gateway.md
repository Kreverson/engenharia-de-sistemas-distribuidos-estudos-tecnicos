# 31 — API Gateway: O Porteiro dos Microsserviços

> Baseado no deep dive da Hello Interview. Explica a evolução histórica, responsabilidades, componentes internos e o papel do API Gateway em arquiteturas de microsserviços modernas.

---

## 📌 Por que o API Gateway existe?

### Anos 2000: Monólito

```
Cliente → URL única → Monólito → DB
```

Simples, fácil de raciocinar. Mas não escala.

### 2010–2012: Microsserviços

```
Cliente → ??? → Serviço A
              → Serviço B
              → Serviço C
```

**Problema:** O cliente precisa conhecer a URL de cada serviço — ou um serviço vira "roteador" dos outros, acoplando lógica de negócio com roteamento.

### 2013–2014: Primeira Geração de API Gateways

```
Cliente → API Gateway → Serviço A
                      → Serviço B
                      → Serviço C
```

**Solução:** Uma única URL para o cliente. O Gateway decide o roteamento.

### Evolução até hoje: Gateway com Middleware

O Gateway também absorveu responsabilidades repetidas que existiam em todo microsserviço:

```
Antes (repetido em cada serviço):
┌─────────────────────────────┐
│ Autenticação                │
│ Rate Limiting               │
│ Logging                     │
│ TLS Termination             │
│ Lógica de Negócio           │
└─────────────────────────────┘

Depois (centralizado no Gateway):
┌────────────────────────────────┐    ┌──────────────────┐
│ API Gateway                    │───▶│ Lógica de Negócio│
│  ├── Autenticação              │    └──────────────────┘
│  ├── Rate Limiting             │    ┌──────────────────┐
│  ├── Logging                   │───▶│ Lógica de Negócio│
│  ├── TLS Termination           │    └──────────────────┘
│  └── Roteamento                │
└────────────────────────────────┘
```

---

## ⚙️ O que acontece dentro do API Gateway?

### 4 Etapas Principais

```
Request ─▶ [1. Validação] ─▶ [2. Middleware] ─▶ [3. Roteamento] ─▶ [4. Transformação] ─▶ Response
```

#### 1. Validação da Request

- Headers corretos?
- Body com formato esperado?
- Campos obrigatórios presentes?
- Se não → rejeita imediatamente (sem chegar nos serviços)

#### 2. Middleware

Operações executadas antes de rotear:

| Middleware | O que faz | Dependência Externa |
|---|---|---|
| **Autenticação (OAuth)** | Verifica token JWT | Serviço OAuth externo |
| **Rate Limiting** | Conta requests por IP/usuário | Redis |
| **Logging** | Registra métricas | Serviço de logs |
| **TLS Termination** | Descriptografa HTTPS | Certificados |
| **Caching** | Retorna resposta cacheada | Redis/Memcached |

> ⚠️ **Atenção:** Cada request passa por todo o middleware. Performance é crítica aqui.

#### 3. Roteamento

Lookup em um mapa de configuração:

```yaml
routes:
  /messages:    messaging-service:8080
  /users:       user-service:8081
  /payments:    payment-service:8082
  /feed:        feed-service:8083
```

O Gateway lê a rota da request e encaminha para o serviço correto.

#### 4. Transformação da Response

Se serviços internos usam **gRPC** mas o cliente espera **REST/JSON**:

```
Serviço (gRPC response) → Gateway → converte para JSON → Cliente (REST)
```

Permite heterogeneidade interna sem expor complexidade para o cliente.

---

## 🛠️ Exemplos de API Gateways em Produção

### Managed (Cloud)

| Produto | Provedor |
|---|---|
| Amazon API Gateway | AWS |
| Azure API Management | Microsoft Azure |
| Cloud Endpoints / Apigee | Google Cloud |

### Open Source / Self-Hosted

| Produto | Características |
|---|---|
| **Kong** | Alta performance, plugins extensíveis |
| **Tyk** | Open source com dashboard |
| **Express Gateway** | Node.js, leve |
| **Traefik** | Cloud-native, integração com Docker/K8s |

---

## 🎯 API Gateway em Entrevistas de System Design

### O que fazer ✅

1. **Coloque o API Gateway no diagrama logo no início**
2. **Explique que ele faz roteamento + middleware**
3. **Mova para frente** — não gaste mais de 1 minuto aqui

### O que NÃO fazer ❌

- Passar muito tempo explicando o Gateway
- Entrar em detalhes de configuração
- Omitir o Gateway (é expectativa implícita em microsserviços)

> **Regra de ouro:** O único erro possível é gastar tempo demais aqui. É um componente assumido em arquiteturas modernas.

---

## 🔄 Padrão Completo em Arquitetura

```
                    ┌──────────────────────────────────┐
                    │           API Gateway            │
Internet ──────────▶│                                  │
(HTTPS)             │  Valida → Auth → Rate Limit      │
                    │  Logging → Roteamento             │
                    └──────┬──────────┬────────┬───────┘
                           │          │        │
                    ┌──────▼──┐ ┌─────▼──┐ ┌──▼──────┐
                    │User Svc │ │Post Svc│ │Feed Svc │
                    └─────────┘ └────────┘ └─────────┘
                         │           │          │
                    ┌────▼───────────▼──────────▼────┐
                    │              Redis              │
                    │         (Rate Limiting)         │
                    └─────────────────────────────────┘
```

---

## 💡 Conceitos Relacionados

| Conceito | Relação com API Gateway |
|---|---|
| **Load Balancer** | O Gateway frequentemente integra LB para distribuir carga entre instâncias do mesmo serviço |
| **Service Mesh** (Istio, Linkerd) | Gerencia comunicação *entre* microsserviços (diferente do Gateway, que é para tráfego *externo*) |
| **BFF (Backend for Frontend)** | Variação onde cada tipo de cliente (mobile, web) tem seu próprio Gateway customizado |
| **Circuit Breaker** | Pode ser implementado no Gateway para parar de rotear para serviços com falha |

---

## 📊 Trade-offs

| Aspecto | Vantagem | Desvantagem |
|---|---|---|
| **Ponto único de entrada** | Simplicidade para o cliente | Single point of failure (mitigado com redundância) |
| **Middleware centralizado** | DRY — sem repetição | Latência adicional em cada request |
| **Roteamento dinâmico** | Fácil de atualizar sem redeploy | Complexidade de configuração |
| **Transformação de protocolo** | Flexibilidade interna | Overhead de serialização |

---

*Arquivo gerado a partir do deep dive da Hello Interview — API Gateway*
