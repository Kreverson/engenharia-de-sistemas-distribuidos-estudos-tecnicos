# 30 — Design YouTube: Streaming de Vídeo em Escala

> Baseado no breakdown da Hello Interview com Evan (ex-Staff Engineer Meta). Cobre upload de vídeos em larga escala, chunking, transcoding adaptativo e CDN para bilhões de usuários.

---

## 📌 Contexto do Problema

YouTube é a segunda plataforma mais visitada do mundo. O desafio de system design aqui é:

- **1 milhão de uploads por dia**
- **100 milhões de usuários ativos diários**
- **Vídeos de até 256 GB**

---

## 🎯 Requisitos Funcionais

| Requisito | Descrição |
|---|---|
| Upload de vídeo | Usuário envia vídeo com metadados |
| Streaming de vídeo | Usuário assiste com baixa latência |

## ⚙️ Requisitos Não-Funcionais

| Atributo | Meta |
|---|---|
| Consistência vs Disponibilidade | **Disponibilidade** (eventual consistency para uploads) |
| Suporte a arquivos grandes | Até 256 GB |
| Latência de streaming | < 500ms para o primeiro pixel |
| Baixa banda | Funcionar em conexões ruins (3G) |
| Escalabilidade | 1M uploads/dia, 100M views |

---

## 🏗️ Entidades Centrais

```
Video          → bytes brutos do vídeo
VideoMetadata  → título, descrição, status, URL do S3, chunks por resolução
User           → criador e espectador
```

---

## 🔌 API Design

```http
POST /videos
  body: { video_bytes, metadata }
  → retorna videoId

GET /videos/{videoId}
  → retorna metadata + S3 URL
```

---

## 🏛️ High-Level Design

```
Cliente → API Gateway → Video Service → S3 (blob)
                                      → VideoMetadata DB
```

**Problema imediato:** AWS API Gateway tem limite de **10 MB** por request. Não é possível enviar 256 GB numa única chamada.

---

## 🚀 Deep Dives

### 1. Upload de Arquivos Grandes: Multipart Upload

**Solução:** Upload direto para o S3, sem passar pelo servidor.

#### Fluxo Completo

```
1. Cliente → POST /videos (apenas metadata + tamanho)
2. Video Service → S3: "quero subir arquivo de Xgb"
3. S3 → Video Service: lista de N pre-signed URLs
4. Video Service → Cliente: pre-signed URLs
5. Cliente: divide o vídeo em chunks de 5-10 MB
6. Cliente → S3: PUT para cada URL (em paralelo)
7. S3: monta o vídeo completo (stitch)
8. S3 → Lambda/Worker: S3 Notification "upload completo"
9. Worker → VideoMetadata DB: atualiza status = "uploaded" + URL do S3
```

#### Por que Pre-Signed URLs?

- Evita tráfego desnecessário pelo servidor
- O cliente sobe diretamente para o S3
- A URL tem expiração (30 min, 1h) — segurança sem expor credenciais

#### Schema VideoMetadata

```json
{
  "id": "abc123",
  "title": "Meu Vídeo",
  "status": "pending | processing | uploaded",
  "s3_full_url": "s3://bucket/video.mp4",
  "chunks": {
    "4K": ["s3://...chunk1", "s3://...chunk2"],
    "1080p": ["s3://...chunk1", ...],
    "720p": [...],
    "240p": [...]
  }
}
```

---

### 2. Streaming com Chunks (Playback)

**Problema:** Baixar um vídeo completo de 10 GB antes de assistir é inviável.

**Solução:** Rechunking para reprodução — segmentos de **2 a 10 segundos**.

#### Chunking de Upload vs Chunking de Playback

| Aspecto | Chunking de Upload | Chunking de Playback |
|---|---|---|
| Tamanho | 5–10 MB | 2–10 segundos de vídeo |
| Propósito | Minimizar overhead HTTP | Início rápido de reprodução |
| Alinhamento | Offsets arbitrários de bytes | Key frames do vídeo |
| Ordem | Paralelo | Sequencial |

#### Fluxo do Chunker

```
S3 Notification → Chunker Workers
               → divide vídeo em segmentos de 2s
               → armazena chunks de volta no S3
               → atualiza VideoMetadata com lista ordenada de URLs
```

---

### 3. Transcoding: Múltiplas Resoluções

**Problema:** Conexão lenta (2 Mbps) não consegue carregar chunk 4K sem travamento.

**Solução:** Transcodar o vídeo em múltiplas resoluções em paralelo.

```
Chunker → Transcoder 4K   → S3 (chunks 4K)
        → Transcoder 1080p → S3 (chunks 1080p)
        → Transcoder 720p  → S3 (chunks 720p)
        → Transcoder 240p  → S3 (chunks 240p)
```

#### O que é um arquivo de vídeo?

```
Container (.mp4, .mov)
  ├── Video Codec (compressão de frames)
  │     ├── Key frames (imagens completas)
  │     ├── Delta frames (diferenças)
  │     └── Instruções de reconstrução
  ├── Audio Codec
  └── Parâmetros: bitrate, resolução, FPS, duração
```

#### Adaptive Bitrate Streaming (ABR)

O cliente monitora a qualidade da conexão e ajusta dinamicamente:

```
Usuário em casa (500 Mbps) → baixa chunks 4K
Usuário sai para a rua (3G) → cliente detecta queda
                            → próximos chunks: 720p
                            → sem interrupção percebida
```

---

### 4. Manifest File + CDN

#### Manifest File

É um arquivo JSON/YAML/XML que lista todos os chunks por resolução, permitindo ao cliente montar o vídeo:

```json
{
  "4K": ["url1", "url2", "url3"],
  "1080p": ["url1", "url2", ...],
  "720p": [...],
  "240p": [...]
}
```

#### CDN (Content Delivery Network)

**Problema:** Usuário na Alemanha buscando chunks do S3 em US-West adiciona ~150ms de latência.

**Solução:** CDN cacheia os chunks populares próximos dos usuários.

```
Cliente (Berlim) → CDN edge node (Frankfurt) → cache hit
                 → CDN edge node (Frankfurt) → cache miss → S3 (US-West)
```

```
Streaming otimizado:
1. Busca manifest file do CDN (em paralelo com metadata)
2. Determina resolução baseada na conexão atual
3. Baixa chunks do CDN mais próximo
4. Reproduz imediatamente o primeiro chunk
5. Monitora banda e ajusta resolução nos próximos chunks
```

---

### 5. Protocolos de Streaming: HLS e DASH

| Protocolo | Criador | Padrão |
|---|---|---|
| HLS (HTTP Live Streaming) | Apple | Proprietário |
| DASH (Dynamic Adaptive Streaming over HTTP) | ISO/IEC | Open source |

Ambos implementam automaticamente:
- Chunking em segmentos
- Manifest files
- Adaptive bitrate
- Compatibilidade com CDNs
- Comunicação via HTTP padrão

> **Nota para entrevistas:** Não é obrigatório conhecer HLS/DASH. O importante é entender os **conceitos subjacentes**: chunking, transcoding e ABR.

---

## 📊 Análise de Escalabilidade

| Componente | Estratégia |
|---|---|
| Video Service | Stateless → escala horizontal |
| API Gateway | Load balancer integrado |
| S3 | "Infinitamente" escalável |
| VideoMetadata DB | ~1 KB por vídeo → ~35 TB/ano, cabe em Postgres |
| Chunker/Transcoder | Stateless → escala horizontal via auto-scaling |
| CDN | Custo vs latência → trade-off de negócio |

### Estimativa de Volume

```
1M uploads/dia × 365 dias = 365M vídeos/ano
Metadata: 1 KB × 365M = 365 GB/ano → trivial
Vídeo: depende da duração e resolução → gerenciado pelo S3
```

---

## 🔄 Fluxo Completo de Upload

```
┌─────────┐     POST /videos (metadata)      ┌──────────────┐
│ Cliente │ ─────────────────────────────────▶│ Video Service│
└─────────┘                                   └──────┬───────┘
     │                                               │
     │         Pre-signed URLs                       │ Salva metadata
     │◀──────────────────────────────────────────── │ Solicita pre-signed URLs
     │                                               │
     │  PUT chunks diretamente no S3                 │
     │──────────────────────────────────▶ ┌─────────┐
     │                                    │   S3    │
     │                                    └────┬────┘
     │                                         │ S3 Notification
     │                                    ┌────▼────────────┐
     │                                    │ Chunker Workers  │
     │                                    └────┬────────────┘
     │                                         │ chunks 2s
     │                                    ┌────▼────────────┐
     │                                    │Transcoder (4K,  │
     │                                    │1080p, 720p...)  │
     │                                    └────┬────────────┘
     │                                         │
     │                                    Atualiza VideoMetadata
     │                                    status = "uploaded"
```

---

## 💡 Lições e Padrões Reutilizáveis

| Padrão | Aplicação |
|---|---|
| **Pre-signed URLs** | Dropbox, Instagram, qualquer upload direto para object storage |
| **Multipart Upload** | Qualquer arquivo > 10 MB |
| **Event-driven com S3 Notifications** | Processamento assíncrono pós-upload |
| **Chunking para streaming** | Netflix, Spotify, Twitch |
| **Transcoding paralelo** | Qualquer pipeline de mídia |
| **CDN para conteúdo popular** | Assets estáticos, vídeos, imagens |
| **Adaptive Bitrate** | Mobile apps, players de vídeo |

---

## 🎓 Expectativas por Nível

| Nível | Expectativa |
|---|---|
| **Júnior/Pleno** | Entender que vídeo vai no S3 e metadata no banco |
| **Sênior** | Chegar ao chunking e CDN proativamente, entender limitações |
| **Staff** | Tudo acima + transcoding, ABR, falhas no DAG, conhecimento de HLS/DASH |

---

*Arquivo gerado a partir do breakdown da Hello Interview — Design YouTube*
