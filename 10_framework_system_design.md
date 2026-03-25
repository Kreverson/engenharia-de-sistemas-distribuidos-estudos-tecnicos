# 10 — Framework para Entrevistas de System Design

> **Fonte:** Hello Interview — metodologia usada por ex-staff engineers da Meta, Google, Amazon para conduzir e avaliar centenas de entrevistas de system design.

---

## 1. O Roteiro (Roadmap)

Todo bom system design segue uma sequência lógica:

```
1. Requirements        → O que o sistema precisa fazer?
2. Core Entities       → Quais dados são persistidos/trocados?
3. API Design          → Qual é o contrato com o usuário?
4. [Data Flow]         → (opcional) Como os dados fluem? (só para infra design)
5. High Level Design   → Design simples que satisfaz os requisitos funcionais
6. Deep Dives          → Como satisfazer os requisitos NÃO funcionais?
```

**Regra de ouro:** cada seção alimenta a próxima. Os requisitos informam as entidades, as entidades informam a API, a API informa o highle design.

---

## 2. Requisitos Funcionais vs. Não Funcionais

### Funcionais
São as *features* do sistema. Fraseados como: "O usuário deve ser capaz de..."

Exemplos para Ticket Master:
- Usuário pode buscar eventos
- Usuário pode ver detalhes de um evento
- Usuário pode comprar um ingresso

### Não Funcionais
São as *qualidades* do sistema. O erro mais comum é escrever apenas buzzwords genéricas (`escalabilidade`, `disponibilidade`).

**O que fazer:**
1. Identificar quais qualidades são **únicas e interessantes** para este sistema
2. Colocar no **contexto** do sistema
3. **Quantificar** sempre que possível

```
❌ Ruim:  "O sistema deve ser escalável"
✅ Bom:   "O sistema deve suportar surtos de Taylor Swift — 100.000 
           usuários simultâneos tentando comprar o mesmo ingresso"

❌ Ruim:  "O sistema deve ter baixa latência"
✅ Bom:   "O feed/stack deve carregar em < 300ms (percepção humana 
           de 'tempo real' é ~200ms)"
```

---

## 3. Estimativas (Back of the Envelope)

**Quando fazer:** apenas se o resultado das contas influenciar uma decisão de design. Não faça cálculos para "verificar uma caixa" no processo.

```
❌ "Temos 100M usuários, 50GB de armazenamento... uau, é muito. 
   Precisamos de um sistema distribuído."
   → Nada foi aprendido

✅ "Temos 10M usuários fazendo 100 swipes/dia = 1 bilhão/dia = 
   ~10.000/segundo. Um nó Postgres aguenta ~10K writes/seg com 
   hardware bom, então precisamos de ~1-2 nós para o caso médio, 
   ~10-20 para o pico → faz sentido usar Cassandra."
   → Decisão concreta foi tomada
```

---

## 4. Core Entities

Não documente o schema completo aqui — é muito cedo. Liste apenas as *tabelas* ou *coleções* que o sistema vai ter.

```
Ticket Master:
├── Event       (quem, quando, onde)
├── Venue       (local físico)
├── Performer   (artista/time)
└── Ticket      (assento específico com preço e status)
```

O schema detalhado emerge naturalmente durante o high level design, à medida que você vai colocando setas no diagrama.

---

## 5. API Design

**Processo:** ir requisito por requisito e criar um endpoint para cada.

**Boas práticas:**
- Use REST (default seguro — GraphQL como alternativa em casos justificados)
- Não coloque `user_id` no body — é vulnerabilidade de segurança. Use JWT/session no header
- Indique verbos HTTP corretos: `GET` para leitura, `POST` para criação, `PATCH`/`PUT` para atualização
- Adicione paginação quando a resposta puder ser grande
- Use `partial<Entity>` quando retornar lista (não precisa de todos os campos)
- Seja explícito sobre quais endpoints requerem autenticação

```
Exemplo Ticket Master:

GET  /events?term=taylor&location=NYC&date=2025-04  
     → list<partial Event>   (busca, sem auth)

GET  /events/:eventId  
     → Event + Venue + list<Ticket>   (detalhes, sem auth)

POST /bookings/reserve
     body: { ticketId }
     header: Authorization: Bearer <jwt>
     → 200 | 400

POST /bookings/confirm
     body: { ticketId, paymentDetails }
     header: Authorization: Bearer <jwt>
     → Booking | error
```

---

## 6. High Level Design

**Objetivo:** satisfazer os requisitos *funcionais*, simples. Não é o momento de pensar em escala.

**Processo:** ir endpoint por endpoint, garantindo que o sistema consegue processar o input e retornar o output esperado.

**Componentes típicos (presentes em ~90% das entrevistas):**
```
Client → Load Balancer → API Gateway → Microservices → Database
                                     ↓
                                  S3/CDN (para mídia)
```

**Sobre Microservices:** é o default seguro. A justificativa para criar um serviço separado é:
1. Precisam escalar **independentemente** (padrões de tráfego diferentes)
2. São computacionalmente distintos e caros

---

## 7. Deep Dives

Aqui está o coração da entrevista para seniors/staff. O processo:

1. Voltar para os **requisitos não funcionais**
2. Para cada um, verificar se o design atual os satisfaz
3. Onde não satisfaz, ir fundo em soluções

**Expectativa por nível:**

| Nível | Comportamento esperado |
|---|---|
| Mid-level | Responde bem às perguntas do entrevistador; resolve problemas apontados |
| Senior | Lidera 1-2 deep dives proativamente; mostra profundidade técnica |
| Staff | Lidera 2-3 deep dives; ensina algo ao entrevistador; conecta trade-offs de negócio |

---

## 8. O Anti-Padrão da Complexidade

Existe uma curva interessante de complexidade por nível:

```
Complexidade da solução
        │
 Alta   │         ★ Senior (over-engineers)
        │       ╱   ╲
 Média  │      ╱     ╲ ★ Staff (simples, mas com profundidade)
        │     ╱
 Baixa  │ ★ Junior/Mid (não sabe suficiente para complicar)
        └─────────────────────────────────
                    Nível
```

**A armadilha do senior:** candidatos seniores colocam message queues em todo lugar, adicionam caches onde não precisam, criam microservices desnecessários.

**O sinal de staff:** "Reconheço que este problema parece complexo, mas na verdade a solução ótima é simples porque [justificativa técnica detalhada]."

---

## 9. Diferença: Product Design vs. Infra Design

**Product Design** (Uber, Tinder, Ticket Master, Yelp):
- Foco em APIs e entidades
- O entrevistador quer ver pensamento de produto
- *Você* toma as decisões de produto — não espere o entrevistador decidir

**Infra Design** (Rate Limiter, Message Queue, Top-K):
- Usa Data Flow em vez de Core Entities + API
- Foco em algoritmos distribuídos e trade-offs de sistemas
- Mais comum em empresas grandes para candidatos sênior+

---

## 10. Dicas Práticas

```
✓ Não gaste mais de 5 minutos em requisitos
✓ Não faça math sem propósito — só quando influencia uma decisão
✓ Comunique explicitamente o que você está deixando out of scope
✓ Deixe espaço para o entrevistador falar — é uma conversa, não monólogo
✓ Se souber a solução ótima cedo, mencione e siga em frente (não redesenhe tudo)
✓ Conclua verificando: "satisfaz todos os requisitos funcionais e não funcionais?"
```

---

## Referências

- [Hello Interview — Delivery Framework](https://hellointerview.com)
- [Martin Fowler — Microservices](https://martinfowler.com/articles/microservices.html)
