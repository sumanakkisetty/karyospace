# KaryoSpace — Architecture

## Overview

KaryoSpace is a single Go binary that serves all product surfaces. There is no microservices overhead, no inter-service networking, no distributed transaction complexity. One binary, one MongoDB instance, one nginx proxy.

This is an intentional architectural choice: enterprise self-hosted software should be operationally simple. A single binary with a single database is deployable on a single VM, upgradeable with a single restart, and debuggable without distributed tracing infrastructure.

---

## Request Flow

```
Browser / API Client
        |
      nginx (TLS termination, reverse proxy)
        |
   Go HTTP Server (:8080)
        |
   +-----------+----------+----------+
   |           |          |          |
  Auth      Router    Handlers   Templates
   |                     |         (parsed at startup)
   |              +------+------+
   |              |             |
   |           MongoDB      AI Layer
   |                        (LLM Providers)
```

---

## AI Layer

### LLM Provider Abstraction

```
LLMProvider interface
    |
    +-- GroqProvider      (llama-3.3-70b-versatile)
    +-- GeminiProvider    (gemini-2.0-flash)
    +-- OllamaProvider    (gemma4, local)
    +-- AnthropicProvider (claude-sonnet)
```

Swap providers via `LLM_PROVIDER` environment variable. No code changes required.

### RAG Pipeline

```
User Query
    |
IsRAGQuery() classifier
    |           |
   YES          NO
    |            |
    v            v
Retriever    Plain LLM call
    |
MongoDB $text search
(email_messages + knowledge_atoms)
    |
Assembler (BuildContextPrompt, 3000 char cap)
    |
LLM call with context
    |
Response + AIO trace stored
```

### AIO — AI Observability Engine

Every LLM interaction produces a trace stored in `aio_traces`:

```json
{
  "org_id": "example.com",
  "query_type": "rag",
  "quality_score": 84,
  "flags": ["cache_hit"],
  "similarity_score": 0.91,
  "latency_ms": 1240,
  "token_cost": 1850,
  "source_count": 3
}
```

Flag types: `empty_retrieval`, `low_similarity`, `cache_hit`, `statement_only`, `anomaly`

### KRE — Knowledge Refinement Engine

Background worker runs every 90 seconds per org:

```
Conversation history (rolling 20-message window)
    |
Gemma4 LLM (local Ollama)
    |
Knowledge atoms:
{
  topic: string
  facts: string[]
  status: "stable" | "volatile"
  confidence: 0.0-1.0
  expires_at: 30d (7d for volatile)
}
    |
Semantic query cache
(cosine similarity >= 0.88, 5000-entry cap per org)
```

---

## Email Architecture

```
Inbound (port 25 -> 2525)
    |
  SMTPSession
    |
  +-- SPF verification
  +-- Spam scoring
  +-- MIME parsing
  +-- MongoDB storage
  +-- Mailbox delivery

Outbound
    |
  DKIM signing
    |
  Brevo relay (STARTTLS)
    |
  OutboundQueue (on failure)
    |
  Backoff: 5m, 30m, 2h, 6h, 24h
  Bounce NDR after 6 failures

IMAP (port 993 -> 1993)
    |
  5 folders: INBOX, Sent, Drafts, Trash, Spam
  Commands: SELECT, FETCH, STORE, SEARCH,
            COPY, APPEND, SUBSCRIBE, EXPUNGE
```

---

## Multi-Tenancy

Org isolation is enforced at the data layer. Every MongoDB query includes an `org_id` filter. There is no shared collection where a missing filter would leak cross-org data.

| Collection | Isolation field |
|------------|----------------|
| email_messages | org_id |
| aio_traces | org_id |
| rag_refined_knowledge | org_id |
| pm_tasks | org_id |
| incidents | org_id |
| kb_articles | org_id |
| channels | org_id |
| calendar_events | org_id |

---

## Security

- **Encryption at rest:** AES-256-GCM, unique 12-byte nonce per call, key from environment variable, never stored in database
- **Encryption in transit:** TLS everywhere — nginx handles public TLS, SMTP uses STARTTLS, IMAP uses STARTTLS
- **Authentication:** Okta SSO for all user login, OAuth 2.0 for third-party integrations (Google, Microsoft), session tokens in secure HTTP-only cookies
- **Email authentication:** DKIM signing on all outbound, SPF verification on inbound, DMARC configured at DNS level

---

## Testing

```
Unit + Integration
    go test ./...
    80%+ coverage across all packages

E2E (Playwright)
    292 tests — chromium + mobile-chrome
    Covers all 6 modules + AI, auth, admin
    291 pass, 1 skip, 0 fail
```

---

## Deployment

```
Dev   (dev.karyospace.com)  — bash scripts/deploy-dev.sh
Prod  (karyospace.com)      — bash scripts/deploy-prod.sh

Both: Docker on Oracle Cloud ARM64
      Binary + frontend built and copied atomically
      Health check before marking deploy complete
```
