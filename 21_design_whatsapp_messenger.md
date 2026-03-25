# 21 — Design WhatsApp / Messenger

> System Design de uma aplicação de mensagens em tempo real com escala de bilhões de usuários

---

## Requisitos Funcionais

| Funcionalidade | Detalhe |
|---|---|
| Criar chats em grupo | Grupos com múltiplos participantes |
| Enviar e receber mensagens | Entrega a todos os membros do chat |
| Suporte a mídia | Fotos, vídeos, áudios |
| Acesso offline | Mensagens recebidas enquanto offline disponíveis ao reconectar |

**Fora do escopo (below the line):** chamadas de voz/vídeo, status online/offline (explicado como extensão)

---

## Requisitos Não-Funcionais

| Requisito | Meta |
|---|---|
| Latência | < 500ms de entrega |
| Durabilidade | Garantia de entrega (at-least-once) |
| Escala | Bilhões de usuários |
| Privacidade | Mensagens não armazenadas além do necessário |
| Tolerância a falhas | Componentes individuais não derrubam o sistema |

---

## Entidades do Sistema

```
User / Device (distinção crucial para múltiplos dispositivos)
Chat (metadados: id, nome)
ChatParticipant (chat_id, participant_id)
Message (id, chat_id, sender_id, content, timestamp)
Inbox (recipient_id, message_id) ← mensagens não entregues
```

> **Insight**: um usuário não fica offline — um *device* fica offline. A distinção entre User e Device é fundamental para garantir entrega em múltiplos dispositivos.

---

## Protocolo de Conexão: WebSockets

### Por que não HTTP puro?

```
HTTP polling: latência = intervalo de polling (2-5 segundos)
WebSocket: push imediato → latência < 500ms ✓
```

### Comparação de Protocolos para Real-Time

| Protocolo | Direção | Quando usar |
|---|---|---|
| HTTP Polling | Client → Server | Latência tolerável (segundos) |
| SSE (Server-Sent Events) | Server → Client | Feed unidirecional (live comments) |
| **WebSockets** | **Bidirecional** | **Chat, mensagens em tempo real** |
| WebRTC | Peer-to-peer | Chamadas de voz/vídeo |

**WhatsApp real**: usa TLS connection direta, sem overhead adicional de WebSocket. O princípio é o mesmo: conexão persistente.

---

## API (via WebSocket)

### Commands do Cliente → Servidor
```
createChat(participants[])
sendMessage(chat_id, content)
createAttachment()
modifyParticipants(chat_id, add/remove)
```

### Commands do Servidor → Cliente
```
newMessage(message)
newChat(chat)
participantChanged(chat_id, participant)
```

---

## High-Level Design

### Modelo de Dados (DynamoDB)

**Tabela: Chat**
```
PK: chat_id | Atributos: name, created_at
```

**Tabela: ChatParticipant**
```
PK: chat_id (partition key) + participant_id (sort key)
GSI: participant_id → para buscar "todos os chats de um usuário"
```

**Tabela: Message**
```
PK: chat_id | SK: message_id (monotonically increasing)
Atributos: sender_id, content, timestamp
```

**Tabela: Inbox**
```
PK: recipient_id | SK: message_id
Propósito: garantir entrega para dispositivos offline
```

### Fluxo de Envio de Mensagem

```
1. Cliente A → WebSocket → Chat Server
2. Chat Server → query ChatParticipant → lista de destinatários
3. Chat Server → DynamoDB transaction → grava em Messages + Inbox (para cada participante)
4. Chat Server → lookup hashmap → encontra conexão WebSocket do destinatário
5. Chat Server → WebSocket → entrega mensagem ao Cliente B
6. Cliente B → ACK → Chat Server → deleta entrada do Inbox
```

> **Limite de transação DynamoDB**: máximo 100 itens por transação → grupos limitados a ~99 participantes.

---

## Mídia: Pre-signed URLs

### Problema do Upload Direto

Enviar vídeo via Chat Server:
- Chat Server sobrecarregado com payloads de GBs
- Disparity: mensagens de texto (bytes) vs. vídeos (MBs/GBs)

### Solução: Pre-signed URL + S3

```
1. Cliente solicita URL de upload ao Chat Server
2. Chat Server pede pre-signed URL para S3
3. S3 retorna URL temporária (com TTL, ex: 1 hora)
4. Cliente faz upload DIRETAMENTE para S3 (sem passar pelo Chat Server)
5. Cliente envia mensagem com a URL do S3 como conteúdo
6. Destinatário recebe URL e baixa a mídia diretamente do S3
```

**Vantagens:**
- Chat Server descarregado de tráfego pesado
- S3 otimizado para blob storage (GB+)
- Escalável independentemente

---

## Escalonamento para Bilhões de Usuários

### O Problema com Servidor Único

Com múltiplos Chat Servers, Cliente A pode estar conectado ao Server 1 e Cliente B ao Server 2 → roteamento quebrado.

### Solução 1: Consistent Hash Ring

```
Clients → DNS → Chat Registry → [determina qual servidor] → Chat Server
                                     ↓
                              Zookeeper/etcd
                         (mapa de usuário→servidor)
```

**Funcionamento:**
- Cada usuário é atribuído deterministicamente a um servidor
- Server-to-server communication quando destino está em outro server
- Scaling: rebalancear apenas ~20% dos usuários ao adicionar servidor

**Desvantagem:** Orquestração complexa durante scaling events.

### Solução 2: Redis Pub/Sub (preferida)

```
Chat Server 1 ← subscreve topic "user_B" ← Redis Pub/Sub
Chat Server 2 → publica "new message for user_B" → Redis Pub/Sub
```

**Fluxo:**
1. Cliente conecta ao Chat Server (qualquer, load balancer de mínimas conexões)
2. Chat Server subscreve tópico do usuário no Redis Pub/Sub
3. Mensagem chega → publica no Redis → Redis entrega ao servidor que tem a conexão
4. Servidor local entrega via WebSocket

**Redis Pub/Sub**: at-most-once delivery. Isso é aceitável porque:
- Mensagens duráveis estão no banco (Messages + Inbox)
- Redis serve apenas para entrega em tempo real
- Offline: ao reconectar, servidor descarrega o Inbox pendente

**Load Balancer para WebSockets:**

```
HTTP (stateless) → Layer 7 Load Balancer (L7 LB) ✓
WebSockets (stateful, conexão persistente) → Layer 4 Load Balancer (L4 LB) ✓
```

L4 LB cria conexão TCP simétrica: transparente para o Chat Server.

---

## Limpeza de Mensagens

```
WhatsApp: mensagens deletadas após 30 dias ou após entrega a todos os dispositivos
```

**Cleanup Service:**
- Index em `timestamp` na tabela Messages
- Deleta mensagens com `timestamp < now - 30 dias`
- Deleta entrada do Inbox quando ACK recebido
- Se último Inbox entry → deleta a própria mensagem

**Estimativa de Storage:**
```
1 bilhão de usuários × 100 mensagens/dia × 1 KB = 100 TB/dia
Retenção de 30 dias → ~3 PB (mas a maioria é entregue imediatamente)
Estimativa real: algumas centenas de TBs
```

---

## Extensões do Design

### Múltiplos Dispositivos por Usuário

Mudança principal: adicionar entidade **Device/Client**

```
Antes: Inbox.recipient_id = user_id
Depois: Inbox.recipient_id = device_id
```

**Impacto:** Ao enviar, buscar todos os dispositivos de cada participante → multiplicar inserts no Inbox. Limitar número de dispositivos ativos por usuário (ex: 3).

### Indicadores de Presença (Online/Offline)

```
Tabela: UserPresence
  PK: user_id | status: online/offline | last_seen: timestamp

Ao conectar: INSERT/UPDATE status = online
Ao desconectar: UPDATE status = offline

Notificação para contatos: Redis Pub/Sub (evento de mudança de status)
```

**Desafio:** Fan-out de notificações para todos os contatos de um usuário popular — limitar número de "watchers" simultâneos por usuário.

---

## Resumo da Arquitetura

```
[Cliente Mobile/Web]
        ↕ WebSocket (L4 LB)
[Chat Server Pool] — subscreve/publica → [Redis Pub/Sub]
        ↕
[DynamoDB: Chat, Participant, Message, Inbox]
        ↕
[S3: Mídia (via pre-signed URL)]
        ↕
[Cleanup Service (background)]
```

---

## Decisões de Design e Trade-offs

| Decisão | Alternativa | Por que a escolha |
|---|---|---|
| Redis Pub/Sub | Kafka | Kafka é pesado para bilhões de tópicos efêmeros; Redis é leve |
| Pre-signed URL | Upload via Chat Server | Descarrega Chat Server de payloads pesados |
| L4 Load Balancer | L7 Load Balancer | WebSockets precisam de conexão persistente (stateful) |
| DynamoDB | PostgreSQL | Schema flexível, escala horizontal nativa |
| Inbox table | Apenas Pub/Sub | Garantia de entrega (at-least-once) mesmo offline |

---

*Fonte: Hello Interview — Design WhatsApp / Messenger*
