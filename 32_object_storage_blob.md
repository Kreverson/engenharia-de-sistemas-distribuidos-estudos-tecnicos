# 32 — Object Storage (Blob Storage): Armazenamento de Arquivos em Escala

> Baseado no deep dive da Hello Interview com Evan. Explica por que bancos relacionais são ruins para arquivos grandes, como o object storage funciona internamente e os três padrões essenciais para entrevistas.

---

## 📌 O que é Object Storage?

Object Storage (ou Blob Storage) é um banco de dados especializado em **arquivos grandes e estáticos**:

- Vídeos, fotos, áudios
- Arquivos JSON/CSV/Parquet em escala
- Assets estáticos (JS, CSS)
- Logs de aplicação
- Dados de treinamento de ML
- Backups

**Não é** um banco relacional — não tem SQL, joins, transações complexas. É otimizado para **leitura/escrita de bytes em escala**.

---

## ❌ Por que NÃO usar banco relacional para arquivos?

### Experimento mental: foto de perfil no PostgreSQL

```sql
CREATE TABLE users (
  id UUID,
  name VARCHAR,
  role VARCHAR,
  profile_image BYTEA  -- ← problema aqui
);
```

**Como o PostgreSQL armazena dados:**

```
Postgres empacota linhas em páginas de 8 KB
Uma foto de 4 MB = 500 páginas

SELECT id, name FROM users LIMIT 50;
-- Postgres precisa gerenciar 50 × 500 páginas só para os dados de imagem
-- mesmo que você não queira ver as fotos!
```

### Problemas concretos

| Problema | Impacto |
|---|---|
| **Overhead em queries** | Queries simples ficam lentas porque o DB gerencia páginas de imagem desnecessariamente |
| **Pressão de memória** | Buffer pool saturado com bytes de imagem, expulsando dados realmente quentes |
| **Replicação lenta** | 4 MB de imagem → replicado em cada replica a cada escrita → lag de replicação |
| **Backups lentos** | Backup noturno inclui todos os blobs → restore de horas em vez de minutos |
| **Custo** | Storage de banco é ~10-20x mais caro que object storage |

---

## ✅ Como o Object Storage funciona internamente

### Arquitetura simplificada

```
Cliente ──────────────────────────────────────────────────────┐
                                                              │
           ┌─────────────────────────────────────────────────┐│
           │              Object Storage                     ││
           │                                                  ││
           │  ┌──────────────┐    ┌───────────────────────┐  ││
           │  │  Metadata    │    │    Storage Nodes       │  ││
           │  │  Service     │    │                        │  ││
           │  │              │    │  Server A: file1, file5│  ││
           │  │  file1 → A   │    │  Server B: file2, file3│  ││
           │  │  file2 → B   │    │  Server C: file1, file4│  ││
           │  │  (+ índice)  │    │  (redundância)         │  ││
           │  └──────────────┘    └───────────────────────┘  ││
           └─────────────────────────────────────────────────┘│
                                                              │
Fluxo GET: Cliente → Metadata Service → "file1 está no Server A"
           → Server A → stream de bytes → Cliente
```

### Três características que tornam o object storage barato e durável

#### 1. Flat Namespace (Namespace Plano)

```
Sistema de arquivos tradicional:
  /home/user/documents/projetos/2024/video.mp4
  → navegar árvore de diretórios

Object Storage:
  "bucket/user/documents/projetos/2024/video.mp4"
  → string única → lookup O(1) no índice
```

As "pastas" no S3 são apenas prefixos da string — não existe hierarquia real. Isso torna a busca extremamente rápida.

#### 2. Immutable Writes (Escritas Imutáveis)

```
Banco relacional:
  UPDATE users SET photo = nova_foto WHERE id = 123;
  → precisa de lock na linha
  → pode criar race conditions
  → WAL (Write-Ahead Log) complexo

Object Storage:
  PUT /bucket/user-123-photo.jpg (versão nova)
  → cria novo objeto
  → versão antiga ainda existe
  → sem locks, sem race conditions
```

Sem mutabilidade → sem necessidade de locks → mais simples, mais rápido, mais barato.

#### 3. Redundância Automática (11 noves de durabilidade)

```
99.999999999% de durabilidade = "11 noves"

Cada arquivo é replicado ou erasure-coded
em múltiplos servidores, racks e data centers:

file1.mp4 → Server A (rack 1, DC São Paulo)
           → Server B (rack 3, DC Rio de Janeiro)
           → Server C (rack 2, DC Campinas)

Falha de um servidor → transparente para o cliente
                     → auto-healing em background
```

---

## 🎯 Três padrões essenciais para entrevistas

### Padrão 1: Arquivos no Object Storage, Metadata no Banco Relacional

```
Banco (Postgres/DynamoDB):
┌─────────────────────────────────────────┐
│ posts                                   │
│  id: uuid                               │
│  creator_id: uuid                       │
│  text: "Hello world!"                   │
│  image_url: "s3://bucket/img-abc.jpg" ◄─┼── apenas a URL
│  created_at: timestamp                  │
└─────────────────────────────────────────┘

S3:
┌─────────────────────────────────────────┐
│ s3://meu-bucket/img-abc.jpg             │ ← arquivo real (4 MB)
└─────────────────────────────────────────┘
```

**Casos de uso:**
- Fotos/vídeos em apps de social media
- Arquivos em ferramentas colaborativas (Dropbox)
- Assets estáticos de frontend
- Logs de aplicação
- Dados de treinamento de ML

### Padrão 2: Pre-Signed URLs para Upload/Download Direto

**Problema:** Fazer o cliente subir arquivo passando pelo servidor desperdiça banda e computação.

```
❌ Abordagem ingênua:
Cliente → POST /upload → Servidor → PUT → S3
         (4 MB de tráfego   (4 MB de tráfego
          até o servidor)    do servidor para S3)

✅ Com Pre-Signed URL:
1. Cliente → GET /upload-url → Servidor
2. Servidor → S3: "quero URL para upload de 4 MB"
3. S3 → Servidor: "use esta URL por 30 minutos"
4. Servidor → Cliente: URL temporária
5. Cliente → PUT → S3 (diretamente!)
```

**Pre-Signed URL:** URL gerada pelo S3 que inclui credenciais temporárias assinadas. Quem tem a URL pode fazer upload/download sem autenticação adicional.

```
https://bucket.s3.amazonaws.com/arquivo.jpg
  ?X-Amz-Algorithm=AWS4-HMAC-SHA256
  &X-Amz-Credential=AKID/20241215/us-east-1/s3/aws4_request
  &X-Amz-Date=20241215T120000Z
  &X-Amz-Expires=1800          ← expira em 30 min
  &X-Amz-Signature=abc123...   ← assinatura criptográfica
```

### Padrão 3: Multipart Upload para Arquivos Grandes

**Problema:** HTTP tem limites de tamanho de payload (5 MB no S3, 10 MB no API Gateway AWS).

```
Arquivo de 1 GB → dividir em 200 chunks de 5 MB

Cliente:
  chunk_1 = arquivo[0:5MB]
  chunk_2 = arquivo[5MB:10MB]
  ...
  chunk_200 = arquivo[995MB:1GB]

Upload em paralelo:
  PUT pre-signed-url-1 ← chunk_1  ┐
  PUT pre-signed-url-2 ← chunk_2  ├── paralelo
  PUT pre-signed-url-N ← chunk_N  ┘

S3: reconstrói o arquivo completo automaticamente
```

**Benefícios:**
- Suporta arquivos de qualquer tamanho
- Upload paralelo → mais rápido
- Retomada em caso de falha (basta re-enviar o chunk que falhou)
- Não passa pelo servidor de aplicação

---

## 🌐 Object Stores populares

| Produto | Provedor | Nota |
|---|---|---|
| **Amazon S3** | AWS | Mais popular, referência de mercado |
| **Google Cloud Storage (GCS)** | GCP | Equivalente ao S3 |
| **Azure Blob Storage** | Microsoft Azure | Equivalente ao S3 |
| **MinIO** | Open Source | Self-hosted, compatível com API do S3 |

Todos suportam: pre-signed URLs, multipart upload, versionamento, lifecycle policies.

---

## 📊 Comparação: Banco Relacional vs Object Storage

| Aspecto | Banco Relacional | Object Storage |
|---|---|---|
| **Estrutura** | Tabelas, linhas, colunas | Chave → blob de bytes |
| **Queries** | SQL complexo, joins | GET/PUT por chave |
| **Mutabilidade** | UPDATE in-place | Immutable (versioning) |
| **Custo/GB** | Alto (~$0.10-1.00/GB) | Baixo (~$0.02-0.03/GB) |
| **Latência** | Baixa (ms) para registros pequenos | Média (dezenas de ms) |
| **Throughput** | Limitado por IOPS | Muito alto (paralelo) |
| **Durabilidade** | Dependente de configuração | 11 noves (built-in) |
| **Caso de uso** | Dados estruturados e mutáveis | Arquivos grandes e estáticos |

---

## 🔄 Diagrama de Referência Completo

```
┌──────────────────────────────────────────────────────────┐
│                       Aplicação                          │
│                                                          │
│  ┌──────────┐    URL do arquivo    ┌─────────────────┐  │
│  │ Banco    │◄─────────────────────│  Post/User API   │  │
│  │ Postgres │                      └────────┬────────┘  │
│  │          │                               │            │
│  │ posts    │                      ┌────────▼────────┐  │
│  │ users    │                      │  Object Storage │  │
│  │ metadata │                      │  API / Gateway  │  │
│  └──────────┘                      └────────┬────────┘  │
│                                             │            │
│                                    ┌────────▼────────┐  │
│                                    │      S3         │  │
│                                    │  (videos, imgs) │  │
│                                    └─────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

---

## 💡 Quando mencionar em entrevistas

| Situação | Use Object Storage |
|---|---|
| Design de rede social | Fotos e vídeos dos posts |
| Design de YouTube/Netflix | Vídeos processados |
| Design de Dropbox/Google Drive | Arquivos dos usuários |
| Design de sistema de e-commerce | Imagens de produtos |
| Design de sistema de logs | Arquivos de log em escala |
| Qualquer sistema com "arquivos grandes" | Object storage é a resposta padrão |

---

*Arquivo gerado a partir do deep dive da Hello Interview — Object Storage*
