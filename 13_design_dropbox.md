# 13 — Design Dropbox/Google Drive: Sync de Arquivos em Escala

> **Questão:** Desenhe um sistema de armazenamento e sincronização de arquivos como o Dropbox ou Google Drive.  
> **Por que é interessante:** combina uploads grandes (até 50GB), resumable uploads, sincronização automática entre dispositivos, e integridade de dados.

---

## 1. Requisitos

**Funcionais:**
- Upload de arquivo para o storage remoto
- Download de arquivo do storage remoto
- **Sincronização automática:** mudanças locais → remoto e remoto → local

**Fora de escopo:**
- Implementar o blob storage (usar S3)
- Compartilhamento com outros usuários

**Não Funcionais:**
- **Disponibilidade > Consistência:** está OK se o arquivo demorar alguns segundos para aparecer em outro dispositivo
- Suportar arquivos grandes: até 50GB
- **Uploads resumáveis:** se a conexão cair no meio, não recomeçar do zero
- **Alta integridade:** o que está local deve estar igual ao remoto (eventual consistency OK)

---

## 2. O Problema do Upload Grande

**Problema 1: Upload redundante**
```
Cliente → POST /files (com bytes no body) → File Service → S3
                                                  ↑
                              Upload desnecessário! 2x banda desperdiçada
```

**Problema 2: Limite de tamanho do body**
- API Gateway (AWS): limite de 10MB no body
- Um arquivo de 50GB nunca passaria pela API

**Solução: Pre-signed URLs do S3**
```
1. Cliente envia só os METADADOS (nome, tamanho, tipo)
   POST /files
   body: { name: "video.mp4", size: 50GB, mimeType: "video/mp4" }

2. File Service cria registro no DB e pede URL assinada ao S3
   S3 responde: "pode fazer upload direto nesta URL por 30 minutos"

3. Cliente faz upload DIRETO para S3 (sem passar pelo servidor)
   PUT https://s3.amazonaws.com/bucket/file123?X-Amz-Signature=...

4. Upload confirmado → cliente avisa o servidor
   PATCH /files/:id { status: "uploaded" }
```

**Vantagem:** sem redundância, sem limite de tamanho, servidor não processa bytes.

---

## 3. Chunking para Uploads Grandes e Resumáveis

**Problema:** upload de 50GB leva ~1h12min com 100Mbps. Se cair no meio, perder tudo.

**Solução: Dividir em chunks de ~5MB**

```
Arquivo de 50GB → 10.000 chunks de 5MB

[chunk_001][chunk_002][chunk_003]...[chunk_10000]
```

**Fluxo:**
```
1. Cliente divide o arquivo em chunks localmente
2. Calcula fingerprint (hash SHA-256) de cada chunk
3. Envia metadados + lista de fingerprints para o servidor
   POST /files
   body: { chunks: [{fingerprint: "abc123", index: 0}, ...] }
4. Servidor salva no DB com status "pending" para cada chunk
5. Cliente faz upload de cada chunk para S3 (via pre-signed URL por chunk)
6. Para cada chunk confirmado, servidor atualiza status → "uploaded"
```

**Resumo de upload (após interrupção):**
```
1. Cliente solicita estado atual do arquivo
   GET /files/:id → retorna lista de chunks com status
2. Cliente compara fingerprints locais com os do servidor
3. Faz upload APENAS dos chunks com status "pending"
4. Continua de onde parou!
```

**Por que fingerprint (hash) em vez de índice?**
- Fingerprint é determinístico: mesmo arquivo sempre gera mesmo hash
- Permite deduplicação: se dois usuários fazem upload do mesmo vídeo, o S3 só armazena uma vez
- Permite detectar corrupção: hash inválido = arquivo corrompido

---

## 4. Schema do Banco

```sql
-- Arquivo e seus chunks
CREATE TABLE file_metadata (
    id           UUID PRIMARY KEY,
    owner_id     UUID,
    folder_id    UUID,
    name         VARCHAR,
    mime_type    VARCHAR,
    size_bytes   BIGINT,
    status       ENUM('uploading', 'ready', 'deleted'),
    s3_link      VARCHAR,
    created_at   TIMESTAMP,
    updated_at   TIMESTAMP
);

CREATE TABLE file_chunks (
    id           UUID PRIMARY KEY,
    file_id      UUID REFERENCES file_metadata(id),
    chunk_index  INT,
    fingerprint  VARCHAR(64),  -- SHA-256
    s3_link      VARCHAR,
    status       ENUM('pending', 'uploaded'),
    updated_at   TIMESTAMP
);
```

---

## 5. Verificação da Integridade (Trust but Verify)

**Problema:** como o servidor sabe que o chunk foi realmente enviado ao S3?

**Abordagem ingênua (insegura):**
```
Cliente: "Enviei o chunk 3, pode marcar como uploaded"
Servidor: OK ✓
```
Um usuário malicioso poderia mentir e fingir que enviou um arquivo de 50GB sem realmente fazer o upload.

**Trust but Verify:**
```
1. Chunk enviado ao S3
2. Cliente avisa servidor: "chunk 3 enviado"
3. Servidor verifica com S3: "você recebeu o chunk 3?"
4. S3 confirma → servidor marca como "uploaded"
```

**Alternativa: S3 Notifications**
```
S3 → SNS/SQS → File Service (automático a cada upload)
```
Não depende do cliente reportar — S3 notifica diretamente o servidor.

---

## 6. Sincronização entre Dispositivos

### Detecção de mudanças locais

Cada SO oferece APIs nativas para monitorar mudanças em diretórios:
- **Windows:** `FileSystemWatcher`
- **macOS:** `FSEvents`
- **Linux:** `inotify`

O client app usa essas APIs para detectar arquivos novos, modificados ou deletados e triggerar o upload.

### Polling para mudanças remotas

O client app faz polling periódico para verificar mudanças no servidor:

```
GET /changes?since=<timestamp>
→ list<{ fileId, event: "created"|"modified"|"deleted", updatedAt }>
```

O cliente usa esse timestamp como "cursor" — a cada poll, passa o timestamp do último evento que já processou.

**Frequência adaptativa:**
- App aberto + mudanças recentes → poll a cada 5s
- App em background → poll a cada 30s
- App inativo por horas → poll manual ao reabrir

**Por que não WebSocket?**
- Sincronização de arquivos não precisa de sub-segundo de latência
- WebSocket permanente para cada dispositivo de cada usuário seria caríssimo
- Polling a cada 5s é mais simples e suficiente

### Delta Sync

Após mudança detectada, não baixar o arquivo inteiro — apenas os chunks que mudaram.

```
Arquivo de 50GB, apenas o capítulo 3 foi editado:
  → comparar fingerprints de todos os chunks
  → baixar apenas os 10 chunks do capítulo 3
  → restituir o arquivo localmente
```

---

## 7. Event Bus + Cursor (Para Auditoria e Versioning)

**Abordagem alternativa ao polling simples:**

Cada mudança gera um evento em um stream (Kafka):
```
{ fileId: "abc", event: "modified", version: 4892, timestamp: ... }
{ fileId: "xyz", event: "created",  version: 4893, timestamp: ... }
```

O cliente mantém um **cursor** — o ID do último evento processado.
```
Da próxima vez que sincronizar:
GET /events?after=cursor_4891
→ [ event_4892, event_4893 ]

Processar → atualizar cursor para 4893
```

**Vantagem:** permite versioning (ver estado do arquivo em qualquer ponto do passado), rollback, e auditoria. Funciona como "Git para arquivos".

**Quando usar:** se o sistema precisar de versioning ou auditoria. Para sincronização simples, o polling do banco é suficiente e mais simples.

---

## 8. Reconciliação Periódica

Mesmo com todos os mecanismos acima, inconsistências podem surgir (bugs, falhas de rede, race conditions).

**Solução: Job de reconciliação periódico (diário ou semanal):**
```
1. Buscar estado atual de todos os arquivos do usuário no servidor
2. Comparar fingerprints com os arquivos locais
3. Identificar e corrigir divergências
   - Arquivo no servidor mas não local → baixar
   - Arquivo local mas não no servidor → fazer upload
   - Fingerprints diferentes → resolver conflito (última modificação vence)
```

---

## 9. Compressão Inteligente

Reduzir a quantidade de bytes transferidos:

```python
def should_compress(file_type, file_size):
    # Tipos que comprimem bem:
    COMPRESSIBLE = ["text/plain", "application/json", ".docx", ".xlsx"]
    
    # Tipos já comprimidos:  
    ALREADY_COMPRESSED = ["image/jpeg", "image/png", "video/mp4", "audio/mp3"]
    
    if file_type in ALREADY_COMPRESSED:
        return False  # pouca ou nenhuma redução, custo de CPU não vale
    
    if file_type in COMPRESSIBLE and file_size > 100_000:  # > 100KB
        return True  # potencial redução de 60-80%
    
    return False

# Se comprimir, salvar o algoritmo usado nos metadados
```

---

## 10. Arquitetura Final

```
Client App (desktop/mobile)
  ├── FSEvents/FileSystemWatcher (detecta mudanças locais)
  ├── Polling Thread (detecta mudanças remotas)
  └── Local DB (cache de metadados + cursor de sync)
      ↓
API Gateway
  ├── File Metadata Service → PostgreSQL / DynamoDB
  │     └── CDC → File Metadata (S3 links, chunks, status)
  └── Sync Service → consulta metadata DB
                   → retorna lista de mudanças

Uploads diretos:
  Client → Pre-signed URL → S3 (bypass do servidor)
  S3 → SNS → File Service (confirmar uploads)

Downloads diretos:
  Client → GET /files/:id → File Service retorna metadados
  Client → download direto do S3 via link
```

---

## Referências

- [AWS S3 Multipart Upload](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [Dropbox Tech Blog — Sync Engine](https://dropbox.tech/infrastructure/sync-engine-foundation)
- [Hello Interview — Dropbox Breakdown](https://hellointerview.com)
