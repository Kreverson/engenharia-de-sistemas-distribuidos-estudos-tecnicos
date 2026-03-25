# 19 — Carreira em Engenharia de Software: Crescimento, AI e Reflexões Práticas

> **Fonte:** Conversas com engenheiros staff/director de Meta, Google, Databricks.  
> **Foco:** Como engenheiros técnicos podem crescer, o impacto da AI na profissão, e conselhos práticos para progressão de carreira.

---

## 1. O Impacto Real da AI na Profissão

### O que a AI está eliminando
- Tarefas mecânicas de codificação (boilerplate, bugs simples, pesquisa básica)
- O "drudgery" — horas no Stack Overflow procurando sintaxe
- Implementação de soluções bem-definidas

### O que a AI NÃO está eliminando (ainda)
- Entender o **problema** antes de definir a solução
- Navegar **trade-offs** técnicos e de produto
- **Comunicação e colaboração** com times interdisciplinares
- **Decisões estratégicas**: o que construir, para quem, com quais recursos
- Lidar com **ambiguidade**: requisitos contraditórios, stakeholders divergentes

> *"AI é ótima em seguir instruções. Ela não está criando as instruções por conta própria em complexidade suficiente para substituir humanos."* — Rendra, Head of Applied AI, Databricks

### O novo valor do engenheiro de software
```
Antes: "Esse engenheiro escreve código rápido e limpo"
Agora: "Esse engenheiro entende o problema profundamente 
        e sabe qual solução construir"
```

A comparação com garbage collection é perfeita: quando C foi substituído por Java, os engenheiros que trabalhavam com memória manual não sumiram — eles pararam de gerenciar memória manualmente e passaram a resolver problemas mais complexos.

---

## 2. A Hierarquia Semântica de Trabalho

Rendra descreve o crescimento como "subir na hierarquia semântica":

```
Nível 0 (será automatizado):
  "Implemente este CRUD endpoint"
  "Corrija este bug"

Nível 1 (exige julgamento técnico):
  "Como devemos estruturar o sistema X?"
  "Qual banco de dados usar para este caso de uso?"

Nível 2 (exige julgamento técnico + de produto):
  "Devemos construir feature X ou Y primeiro?"
  "Qual a arquitetura que nos dá mais velocidade nos próximos 2 anos?"

Nível 3 (estratégico):
  "Qual é nossa estratégia de produto para competir com Z?"
  "Como estruturamos o time para entregar esta visão?"
```

**À medida que a AI assume o Nível 0, engenheiros precisam operar no Nível 1+.**

---

## 3. Comunicação como Veículo de Progressão

### Por que comunicação importa mais do que parece

Lideranças avaliam profissionais com tempo limitado — frequentemente baseadas em 10 frases de uma reunião ou um documento de design. Comunicação clara sinaliza capacidade de pensar claramente.

```
❌ "Precisamos de escalabilidade e alta disponibilidade"
   → Todo mundo diz isso. Vazio.

✅ "Nossa maior vulnerabilidade é o single point of failure 
   no serviço de pagamento. Uma falha de 5 minutos durante 
   a Black Friday pode custar R$2M. Proponho um plano de 
   failover em 3 semanas."
   → Específico, quantificado, com proposta de ação.
```

### O que "comunicar bem" significa na prática

1. **Estudar o problema em profundidade antes de falar**
   - Não fale de algo sem ter ido fundo
   - Encontre conexões não óbvias
   - Traga uma perspectiva que ninguém mais tem

2. **Mostrar o raciocínio, não só a conclusão**
   - "Escolhi Kafka porque..." > "Escolhi Kafka"

3. **Escrever documentos de design de alta qualidade**
   - Alternativa à comunicação verbal
   - Mostra nível de pensamento de forma permanente

4. **Deixar espaço para a liderança também falar**
   - Staff engineers conduzem conversas, não monólogos

---

## 4. Como Crescer em uma Grande Empresa

### Primeiros 3-6 meses em um novo time

```
Semana 1-2: Setup básico, leitura de documentos, code review silencioso
  ❌ Não: ler documentos o dia inteiro (retenção baixa)
  ✅ Sim: fazer perguntas, achar um task pequeno para mexer

Semana 3-4: Fazer deploy de algo pequeno
  → Mesmo que seja 1 linha de código
  → O objetivo é passar pelo processo completo
  → Builds confiança interna e externa

Mês 2-3: Ir mais fundo antes de ir mais amplo
  → Entender profundamente uma área específica
  → Evitar o erro de tentar aprender tudo de uma vez
  
Mês 3-6: Comunicar o que você aprendeu
  → Documentos de design
  → Apresentações em team reviews
  → Proposta de melhorias
```

### O que diferencia Senior de Staff

```
Senior:
  - Executa bem projetos complexos definidos
  - Soluciona problemas técnicos difíceis
  - Mentora engenheiros mais novos

Staff:
  - Define quais projetos devem existir
  - Identifica problemas que ninguém estava vendo
  - Alinha times com visões divergentes
  - Ensina ALGO ao entrevistador numa entrevista técnica
```

### Como se posicionar para o próximo nível

1. **Mostrar que você pode fazer mais do que faz agora**
   - Documentos de design ambiciosos
   - Voluntariar para projetos mais complexos (ex: on-call)
   - Propor soluções, não apenas executar as dadas

2. **Entender o que sua liderança valoriza**
   - Perguntar explicitamente: "O que esperas de alguém no próximo nível?"
   - Adapter-se antes de assumir

3. **Calibrar suas expectativas com o manager**
   - "Qual é o tempo esperado para eu estar 100% produtivo?"
   - Evita frustração desnecessária por expectativas não alinhadas

---

## 5. Quando Mudar de Empresa

```
Sinais de que é hora de mudar:

✓ Você não tem mais problemas difíceis para resolver
✓ A empresa não reconhece profundidade técnica como valor
✓ Você depende muito do manager para ter acesso a projetos interessantes
✓ O mercado paga significativamente mais pelo seu perfil

Sinais de que NÃO é hora:
✗ Você só quer mudar por salário (vai se repetir)
✗ Você ainda não entregou impacto real onde está
✗ Você quer mudar antes de entender por que não cresceu
```

**Switching cost é alto:** ~6 meses para se tornar totalmente produtivo, impressões iniciais que são difíceis de desfazer, curva de aprendizado cultural.

**O primeiro switch é o mais difícil** — mas fica mais fácil com prática.

---

## 6. Hobbies e Saúde Mental como Estratégia de Carreira

Contra-intuitivo, mas apoiado por evidências práticas:

**Por que funciona:**
1. **Antidote ao burnout:** se o trabalho vai mal, ter outras fontes de realização previne que uma frustração se torne uma crise
2. **Fonte de criatividade:** problemas de outras áreas frequentemente inspiram soluções técnicas
3. **Perspectiva:** força você a ser iniciante periodicamente — algo valioso para empatia com usuários

**O que conta como hobbie "estratégico":**
Não é apenas "toco guitarra às vezes." É algo onde você:
- Tem accountability (uma apresentação, uma exposição, um prazo)
- Recebe feedback externo sobre seu trabalho
- Progride mensuravelmente ao longo do tempo

```
Exemplo: Rendra produz peças de teatro como hobby
  → Tem deadline: a estreia
  → Recebe feedback: audiência reage bem ou mal
  → Progressão mensurável: cada produção é melhor
  → Se algo vai mal no trabalho, ainda tem a peça para se sentir valioso
```

---

## 7. Carreira em AI: O Paradoxo da Demanda

**O paradoxo:** muitos querem trabalhar com AI (treinar modelos, escrever PyTorch), o que reduz o salário desse trabalho específico (oferta alta).

```
Escassez real está em:
  - Pessoas que sabem QUAL problema de AI resolver
  - Pessoas que melhoram iterativamente a QUALIDADE de sistemas AI
  - Pessoas que conectam AI com impacto de negócio real
  - Pessoas com expertise técnica profunda + capacidade de comunicação
```

**O loop de valor em AI:**
```
Lançar produto AI → Avaliar qualidade → Identificar falhas → 
Experimentar soluções → Medir melhoria → Lançar de novo
```

Pessoas que percorrem este loop rapidamente e com rigor são raras e valiosas.

---

## 8. Resumo: Perguntas para Reflexão

```
Sobre progressão:
  □ Estou operando em qual nível semântico? O que me mantém no nível atual?
  □ Minha comunicação reflete a profundidade do meu pensamento?
  □ Estou em uma empresa que reconhece o tipo de impacto que gero?

Sobre AI:
  □ Quais partes do meu trabalho podem ser automatizadas nos próximos 3 anos?
  □ Estou desenvolvendo as habilidades que machines não conseguem replicar?

Sobre bem-estar:
  □ Tenho outras fontes de satisfação além do trabalho?
  □ Estou me forçando a ser iniciante periodicamente?
```

---

## Referências

- Entrevista com Rendra (Head of Applied AI, Databricks) — Hello Interview
- Entrevista com Christian (Engineering Manager, Meta) — Hello Interview
- [Andrew Bosworth — Cold Start Problem](https://boz.com/articles/cold-start)
