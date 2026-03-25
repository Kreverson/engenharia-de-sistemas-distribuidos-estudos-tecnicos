# 01 — Falhas, Dívida Técnica e o Caso Knight Capital

> **Caso real:** Knight Capital Group, agosto de 2012.  
> **Prejuízo:** US$ 440 milhões em 45 minutos (~US$ 10 milhões/minuto).  
> **Causa raiz:** Deploy manual incompleto + reutilização de flag em dead code.

---

## 1. Dead Code (Código Morto)

Dead code são trechos de código que existem na base mas nunca são executados no fluxo normal da aplicação. Podem ser funções legadas, flags de feature toggle antigas ou lógica comentada.

**Por que é perigoso?**

- Cria falsa segurança: o código "não roda", logo ninguém o testa
- Pode ser reativado acidentalmente por uma flag, configuração ou mudança de contexto
- Acumula junto com o sistema e se torna arqueologia digital — ninguém sabe mais o que faz

**No caso Knight Capital:**  
A função `Power Peg`, desativada desde 2003, foi reativada acidentalmente quando uma nova feature flag — cujo nome foi reutilizado — foi deployada nos servidores. O código antigo interpretou a flag nova como "compre tudo, agora, sem limite".

```
[2003] Power Peg desativado → flag SMARS marcada como "legado"
[2012] Nova feature usa a mesma flag SMARS para novo comportamento
[Deploy] 7 de 8 servidores recebem código novo
         1 servidor ainda tem código antigo + flag nova = caos
```

**Lição:**
> Nunca reutilize nomes de flags antigas para comportamentos novos. Delete o dead code — código comentado é dívida com juros compostos.

---

## 2. Reutilização de Flags (Feature Toggles)

Feature flags (ou feature toggles) são mecanismos para ativar/desativar comportamentos em produção sem novo deploy. São poderosas — e por isso mesmo perigosas se mal gerenciadas.

**Boas práticas:**

| Prática | Motivo |
|---|---|
| Nomear flags com contexto único e versionado | Evita colisão semântica com flags antigas |
| Ter inventário/registro de flags ativas | Saber o que está ligado em produção |
| Expirar e deletar flags após uso | Reduz superfície de risco |
| Nunca reaproveitar o nome de uma flag deletada | O comportamento antigo pode estar em algum servidor esquecido |

---

## 3. Deploy Manual vs. Automatizado

O deploy da Knight Capital foi feito **arrastando arquivos manualmente** entre servidores — o que ficou famoso como "drag and drop em servidor de alta frequência".

**O problema estrutural:**

```
8 servidores para atualizar
7 foram atualizados corretamente
1 foi esquecido

→ Sistema rodando com versões diferentes em paralelo
→ Comportamento imprevisível e catastrófico
```

**Por que automação importa:**

Um deploy automatizado com ferramentas como Ansible, Terraform ou pipelines de CI/CD garante que **ou todos os servidores são atualizados, ou nenhum é**. Não existe "parcialmente deployado".

```yaml
# Exemplo conceitual com Ansible
- hosts: trading_servers
  tasks:
    - name: Deploy novo binário
      copy:
        src: /build/trading_app
        dest: /opt/trading/trading_app
      notify: restart trading service
```

> Se não é automatizado, não existe garantia de execução.

---

## 4. Circuit Breaker

O Circuit Breaker é um padrão de resiliência que protege o sistema de continuar executando operações que estão claramente falhando — análogo a um disjuntor elétrico.

**Estados:**

```
FECHADO (normal)
    ↓ (muitas falhas consecutivas)
ABERTO (bloqueando chamadas)
    ↓ (após timeout de recuperação)
SEMI-ABERTO (testando uma chamada)
    ↓ sucesso → FECHADO
    ↓ falha   → ABERTO novamente
```

**No caso Knight Capital:** Não havia circuit breaker. O algoritmo continuou comprando e vendendo de forma errada por **45 minutos inteiros** sem nenhum mecanismo automático para interrompê-lo.

Um circuit breaker simples baseado em volume financeiro teria parado o sangramento em segundos:

```python
class CircuitBreaker:
    def __init__(self, threshold_loss_usd):
        self.threshold = threshold_loss_usd
        self.accumulated_loss = 0
        self.state = "CLOSED"

    def record_trade(self, pnl):
        self.accumulated_loss += pnl
        if self.accumulated_loss < -self.threshold:
            self.state = "OPEN"
            self.emergency_halt()

    def emergency_halt(self):
        # Para todas as ordens imediatamente
        cancel_all_orders()
        alert_on_call_engineer()
```

---

## 5. Observabilidade e Alertas

Durante o incidente, os engenheiros da Knight Capital não conseguiam identificar qual dos 8 servidores estava causando o problema. Os logs passavam rápido demais, não havia agregação e o monitoramento era passivo.

**Os três pilares de observabilidade:**

| Pilar | O que é | Ferramenta comum |
|---|---|---|
| **Métricas** | Dados numéricos ao longo do tempo (latência, volume, erro) | Prometheus, Datadog |
| **Logs** | Registros de eventos textuais | Elasticsearch, Loki |
| **Traces** | Rastreamento de uma requisição por todo o sistema | Jaeger, Zipkin |

**Monitoramento passivo ≠ monitoramento:**  
Ter logs que ninguém lê é o mesmo que não ter logs. Alertas precisam ser ativos, configurados para disparo automático em anomalias.

---

## 6. Resumo das Lições

```
✗ Deploy manual         → ✓ CI/CD automatizado e idempotente
✗ Dead code esquecido   → ✓ Refatoração e deleção ativa de código legado  
✗ Flags reutilizadas    → ✓ Inventário de flags com ciclo de vida definido
✗ Sem circuit breaker   → ✓ Proteção automática contra comportamento aberrante
✗ Monitoramento passivo → ✓ Alertas ativos + dashboards em tempo real
✗ Checklist manual      → ✓ Validação automatizada pós-deploy
```

> **A dívida técnica sempre cobra — com juros.**  
> No caso da Knight Capital, os juros foram US$ 440 milhões em 45 minutos.

---

## Referências e Leitura Adicional

- [Post-mortem: Knight Capital](https://www.henricodolfing.com/2019/06/project-failure-case-study-knight-capital.html)
- [Martin Fowler — Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [Martin Fowler — Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html)
