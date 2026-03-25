# 15 — Design Web Crawler: Indexação Massiva com Politeness e Fault Tolerance

> **Objetivo:** Construir um WebCrawler que extrai texto de 10 bilhões de páginas em 5 dias, para treinar um LLM.  
> **Por que é desafiador:** escala massiva, politeness (respeito ao robots.txt), deduplicação eficiente, e fault tolerance em um pipeline distribuído.

---

## 1. Requisitos

**Funcionais:**
- Crawlear a web a partir de um conjunto de seed URLs
- Extrair texto de cada página HTML
- Armazenar o texto extraído

**Não Funcionais:**
- **Fault tolerant:** não perder progresso em caso de falha
- **Polite:** respeitar robots.txt e rate limits dos domínios
- **Eficiente:** crawlear 10 bilhões de páginas em 5 dias
- **Sem duplicatas:** não processar a mesma página duas vezes

---

## 2. Estimativa de Escala

```
10 bilhões de páginas × 2MB/página = 20 petabytes de dados

Um servidor AWS otimizado para rede: 400 Gbps
= 400 Gbps / 8 bits/byte / 2MB por página = 25.000 páginas/segundo (teórico)

Realidade: DNS lookups, rate limiting, robots.txt, retries, latências...
→ ~30% de utilização efetiva = ~7.500 páginas/segundo por máquina

Para 10B páginas em 5 dias:
  5 dias × 86.400 segundos = 432.000 segundos
  10.000.000.000 / 432.000 = ~23.150 páginas/segundo necessários

23.150 / 7.500 ≈ 3-4 máquinas grandes
→ Usar 4 máquinas com margem de segurança
```

---

## 3. Data Flow (Pipeline em Estágios)

**Erro comum:** fazer tudo em uma única função — fetch + parse + extract + store.

**Por que separar?**
1. Scales independently (fetch é I/O bound, parse é CPU bound)
2. Fault isolation — se o parse falhar, o HTML já está salvo e pode ser reprocessado sem re-crawlear
3. Flexibilidade — ML team pede OCR nas imagens? Adiciona um novo estágio sem re-crawlear

```
Frontier Queue
    ↓
[Crawler Workers]      → fetch HTML → S3 (raw HTML)
    ↓                               → update URL metadata DB
Parsing Queue
    ↓
[Parser Workers]       → extrair texto → S3 (text data)
                       → extrair URLs → Frontier Queue
```

**Separação crucial:** o crawler faz APENAS uma coisa — buscar a página e salvar no S3. Tudo mais é downstream.

---

## 4. Frontier Queue: SQS vs. Kafka

**Kafka:**
- Não suporta retries nativamente
- Requer implementação manual de retry topics
- Melhor para ordered stream processing

**SQS:**
- Suporte nativo a retries com exponential backoff configurável
- Dead Letter Queue (DLQ) nativo
- Visibility timeout nativo (mensagem "invisível" enquanto sendo processada)

```
SQS Visibility Timeout:
  Worker puxa URL da fila
  → URL fica "invisível" por 30 segundos (ninguém mais a processa)
  → Worker falha antes de confirmar
  → Após 30s, URL reaparece para outro worker
  
  Worker confirma sucesso:
  → URL é deletada permanentemente da fila
```

**Retry com exponential backoff no SQS:**
```yaml
# Configuração da fila SQS
maxReceiveCount: 5          # máximo de tentativas
visibilityTimeout: 30       # padrão: 30s
# Após 5 falhas → Dead Letter Queue automaticamente
```

---

## 5. Politeness: Respeitando robots.txt

**O que é robots.txt:**
```
# https://example.com/robots.txt
User-agent: *
Disallow: /private/
Disallow: /admin/
Crawl-delay: 10        # esperar 10 segundos entre requests
```

**Schema da tabela de domínios:**
```sql
CREATE TABLE domain_metadata (
    domain      VARCHAR PRIMARY KEY,
    robots_text TEXT,       -- conteúdo do robots.txt
    last_fetched TIMESTAMP, -- quando buscamos o robots.txt
    crawl_delay  INT,       -- segundos entre requests (do robots.txt)
    disallowed_paths TEXT[] -- paths proibidos
);
```

**Algoritmo do crawler:**
```python
def should_crawl(url):
    domain = extract_domain(url)
    path = extract_path(url)
    
    # 1. Buscar/cachear robots.txt do domínio
    domain_meta = get_or_fetch_domain_metadata(domain)
    
    # 2. Checar se o path está bloqueado
    if is_path_disallowed(path, domain_meta.disallowed_paths):
        acknowledge_and_skip()
        return False
    
    # 3. Checar crawl delay
    time_since_last_crawl = now() - domain_meta.last_crawl_time
    if time_since_last_crawl < domain_meta.crawl_delay:
        # Recolocar na fila com visibility timeout adequado
        delay = domain_meta.crawl_delay - time_since_last_crawl
        requeue_with_delay(url, delay)
        return False
    
    return True
```

**Rate limiter por domínio (Redis):**
```python
# Padrão da indústria: máximo 1 request/segundo por domínio
def check_rate_limit(domain):
    key = f"rate:{domain}"
    count = redis.incr(key)
    if count == 1:
        redis.expire(key, 1)  # janela de 1 segundo
    
    if count > 1:
        return False  # rate limited
    return True
```

---

## 6. Crawler Traps (Armadilhas)

Alguns sites geram URLs infinitas artificialmente para prender crawlers:
```
/category/1 → /category/2 → /category/3 → ... → /category/999999
/product?page=1&sort=name → /product?page=1&sort=date → ...
```

**Solução: Max Depth**
```sql
-- Adicionar coluna depth na tabela URL metadata
ALTER TABLE url_metadata ADD COLUMN depth INT DEFAULT 0;

-- Ao adicionar nova URL extraída de uma página:
new_url_depth = parent_url_depth + 1
if new_url_depth > MAX_DEPTH:
    # Ignorar URL (não colocar na fila)
    pass
```

Valor típico de MAX_DEPTH: 20-30 níveis.

---

## 7. Deduplicação: Não Crawlear a Mesma Página Duas Vezes

**Dois níveis de deduplicação:**

### Nível 1: Deduplicação por URL
```sql
CREATE TABLE url_metadata (
    url         VARCHAR PRIMARY KEY,  -- constraint de unicidade
    status      ENUM('pending', 'crawled', 'failed'),
    depth       INT,
    s3_html_link VARCHAR,
    created_at  TIMESTAMP
);
```

Antes de adicionar URL à fila, checar se já existe.

### Nível 2: Deduplicação por Conteúdo (Fingerprinting)

Problema: duas URLs diferentes com conteúdo idêntico.
```
https://site.com/article         (original)
https://site.com/article?ref=123 (mesmo conteúdo, URL diferente)
```

**Solução: Hash do conteúdo HTML**
```python
import hashlib

def should_parse(html_content):
    fingerprint = hashlib.sha256(html_content.encode()).hexdigest()
    
    # Checar se já processamos este conteúdo
    if url_metadata.fingerprint_exists(fingerprint):
        return False  # conteúdo duplicado
    
    # Salvar fingerprint para futuras verificações
    url_metadata.save_fingerprint(fingerprint)
    return True
```

**Armazenamento do fingerprint:**

Opção 1 (simples): Global Secondary Index no DynamoDB no campo `fingerprint`.

Opção 2 (mais eficiente): **Bloom Filter**

---

## 8. Bloom Filter: Trade-off entre Espaço e Precisão

Um bloom filter é uma estrutura de dados probabilística que verifica membership em espaço muito compacto.

```
Sem bloom filter:
  10B páginas × 32 bytes (SHA-256) = 320 GB de dados de fingerprint
  Caro, mas 100% preciso

Com bloom filter:
  ~50-100 GB para 10B itens com 1% de false positive rate
  Muito mais eficiente!
```

**Como funciona:**
```
Array de bits (simplificado):
  [0][0][0][0][0][0][0][0]   (8 bits, inicialmente tudo 0)

Adicionar "página_abc":
  hash("página_abc") → posições 2, 5, 7
  [0][0][1][0][0][1][0][1]

Verificar se "página_xyz" existe:
  hash("página_xyz") → posições 1, 3, 7
  Posição 1 = 0 → DEFINITIVAMENTE NÃO EXISTE (false negative impossível!)

Verificar se "página_abc" existe:
  hash("página_abc") → posições 2, 5, 7
  Todas = 1 → POSSIVELMENTE EXISTE (pode ser false positive!)
```

**Para nosso caso:**
- False negative impossível: se o bloom filter diz "não vimos", com certeza não vimos
- False positive possível: bloom filter diz "já vimos", mas pode ser que não
- Trade-off: com 1% de false positive, ~1% das páginas únicas são ignoradas — aceitável

**Quando usar:**
- Sistema com restrição severa de memória E tolerância a pequenos erros
- Para nosso crawler com budget ilimitado: provavelmente o índice secundário é mais simples

---

## 9. DNS Caching e Múltiplos Providers

**Problema:** 10.000 páginas/segundo × múltiplos domínios = muitos lookups DNS.

```
Solução 1: Cache DNS no Redis
  redis.set(f"dns:{domain}", ip_address, ex=3600)  # cachear por 1h

Solução 2: Múltiplos providers com Round Robin
  dns_providers = [cloudflare, google_dns, aws_route53]
  provider = dns_providers[request_count % len(dns_providers)]
```

---

## 10. Arquitetura Final

```
Seed URLs
    ↓
SQS Frontier Queue
    ↓
[4x Crawler Servers]
  ├── Rate limiter por domínio (Redis)
  ├── robots.txt check (domain_metadata DB)
  ├── DNS cache (Redis)
  ├── Fetch HTML
  ├── Save HTML → S3
  ├── Update URL metadata (DynamoDB)
  └── Enfileirar na SQS Parsing Queue
           ↓
    SQS Parsing Queue
           ↓
    [Parser Workers] (auto-scale via SQS depth)
      ├── Read HTML from S3
      ├── Extract text → S3
      ├── Fingerprint check (DynamoDB GSI ou Bloom Filter)
      ├── Extract URLs → Frontier Queue
      └── Update metadata

Dead Letter Queue → análise manual de páginas que falharam 5x
```

---

## Referências

- [AWS SQS — Visibility Timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [Bloom Filters — Wikipedia](https://en.wikipedia.org/wiki/Bloom_filter)
- [robots.txt specification](https://www.robotstxt.org/robotstxt.html)
- [Hello Interview — WebCrawler Breakdown](https://hellointerview.com)
