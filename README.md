# KaryoSpace

> Self-hosted, AI-native enterprise workspace. One platform. Every tool your company needs. Your infrastructure. Your data.

[![Live](https://img.shields.io/badge/Live-karyospace.com-6366f1?style=flat-square)](https://karyospace.com)
[![License](https://img.shields.io/badge/License-Elastic_2.0-informational?style=flat-square)](LICENSE)
[![Built with](https://img.shields.io/badge/Built_with-Claude_Code-orange?style=flat-square)](https://claude.ai/code)
[![Go](https://img.shields.io/badge/Go-1.22-00ADD8?style=flat-square&logo=go)](https://go.dev)
[![Tests](https://img.shields.io/badge/E2E_Tests-292_passing-brightgreen?style=flat-square)](tests/)

---

## The Problem

Enterprise knowledge is fragmented across 10 to 15 disconnected SaaS products that do not talk to each other in any meaningful way. AI layers built on top of fragmented data produce fragmented intelligence. Every AI assistant your company buys sees only the slice of data the vendor gives it access to.

KaryoSpace solves this by owning the entire data layer: email, messaging, tasks, incidents, calendar, and documents in a single platform. Then it builds the AI on top of that unified layer. The result is an AI that actually knows your company because it has access to everything, not just what one vendor decided to expose.

---

## What It Replaces

| KaryoSpace Module | Replaces | Key Capability |
|-------------------|----------|----------------|
| Email (SMTP/IMAP) | Gmail, Outlook | Full email server built from scratch in Go. DKIM, SPF, STARTTLS, MIME. |
| Messaging | Slack, Teams | Real-time channels with persistent history and AI context. |
| Project Management | Jira, Linear | Kanban board, sprints, task tracking with AI-powered triage. |
| Incident Management | PagerDuty, OpsGenie | Incident lifecycle, escalation paths, post-mortem knowledge capture. |
| Calendar | Google Calendar, Outlook | Event management with Outlook and Google sync via OAuth. |
| Knowledge Base | Notion, Confluence | Structured org knowledge with AI distillation and semantic search. |

---

## Build Stats

This was built by one person, during off-hours, while employed full-time, using Claude Code as the primary development tool.

| Metric | Value |
|--------|-------|
| Total commits | 1,614 |
| Claude Code sessions | 525+ |
| AI pair programming hours | 300+ |
| Time to market | 7 months |
| Go test coverage | 80%+ across every package |
| Playwright E2E tests | 292 tests, chromium + mobile-chrome |
| E2E result | 291 pass, 1 skip, 0 fail |

**Commit cadence tells the story of how this was built:**

| Phase | Commits | Setup |
|-------|---------|-------|
| Nov 2025 to Feb 2026 | 53 | Traditional local machine |
| March 2026 | 438 | Full cloud dev environment on Oracle Cloud |
| April 2026 | 428 | Phone browser only, no laptop |
| May 2026 | 601 | Same setup, full production feature velocity |

The compiler, Claude Code, Docker, and the deployment pipeline all run on a free Oracle Cloud VM. The only client is a phone browser. No desk, no laptop, no fixed schedule. 1,614 commits shipped from that setup.

---

## AI Infrastructure

### RAG Pipeline

Every user query is classified before routing. Queries that reference company data (emails, tickets, incidents, knowledge articles) trigger retrieval across all relevant MongoDB collections before hitting the LLM. Queries that are conversational bypass retrieval entirely. The LLM never sees unnecessary context and never answers data questions without grounding.

```
User query
    |
    +-- IsStatement()  -->  "Got it, noted."  (no LLM call)
    |
    +-- IsRAGQuery()   -->  Retriever
    |                           |
    |                       MongoDB $text search
    |                       (email_messages, kb_articles,
    |                        pm_tasks, incidents, channels)
    |                           |
    |                       BuildContextPrompt()  (3000 char cap)
    |                           |
    |                       LLM call + AIO trace stored
    |
    +-- General query  -->  LLM call with conversation history
                            (rolling 20-message window per user)
```

### LLM Provider Abstraction

All four providers implement the same `LLMProvider` interface. Swap via environment variable at runtime, no code changes required.

```
LLM_PROVIDER=groq     -->  llama-3.3-70b-versatile  (default, cloud)
LLM_PROVIDER=gemini   -->  gemini-2.0-flash          (cloud)
LLM_PROVIDER=ollama   -->  gemma4, qwen2.5:7b        (local, on-prem)
LLM_PROVIDER=claude   -->  claude-sonnet             (Anthropic)
```

Each provider handles its own message format, system role constraints, token counting, and error handling. The application layer is provider-agnostic.

### AIO: AI Observability Engine

Every LLM interaction is instrumented. Every query produces a trace record with a quality score, flag set, and performance metrics stored in `aio_traces`. Nothing goes unmonitored.

```json
{
  "org_id":           "acme.com",
  "user_id":          "alice@acme.com",
  "query_type":       "rag",
  "quality_score":    84,
  "flags":            ["cache_hit"],
  "similarity_score": 0.91,
  "source_count":     4,
  "latency_ms":       1240,
  "token_cost":       1850,
  "retrieved_at":     "2026-06-25T10:00:00Z"
}
```

Quality scoring is flag-based. Each flag reduces the score by a weighted amount:

| Flag | Meaning | Score impact |
|------|---------|-------------|
| `empty_retrieval` | RAG found nothing, LLM answered blind | -30 |
| `low_similarity` | Retrieved content weakly matched the query | -20 |
| `no_sources` | Response generated with no grounding | -25 |
| `cache_hit` | Answered from semantic cache, no LLM call | +10 |
| `anomaly` | Score below threshold, flagged for review | marker |

Aggregated per org: cache hit rate, average quality score, flagged interaction count, token spend by day.

### KRE: Knowledge Refinement Engine

A background worker that runs every 90 seconds per active org. It reads recent conversation history and uses a local Gemma4 LLM to extract structured knowledge atoms, writes them to MongoDB, and makes them available to the RAG retriever on future queries. The system gets smarter without human curation.

```
Conversation history  -->  Gemma4 (local Ollama)
                               |
                    Knowledge atom extracted:
                    {
                      topic:      "SLA Breach Protocol",
                      facts:      ["Escalate to L2 within 15 min", ...],
                      status:     "stable",
                      confidence: 0.91,
                      expires_at: now + 30 days
                    }
                               |
                    Semantic query cache
                    (cosine similarity >= 0.88)
                    (5,000-entry cap per org)
```

Atom TTLs are topic-aware. Stable policy and process atoms live 30 days. Volatile atoms (incidents, announcements, forecasts) live 7 days. Expired atoms are excluded from retrieval automatically.

---

## Email Infrastructure

A complete SMTP and IMAP server built from scratch in Go. No third-party email library. RFC-compliant from the ground up.

**Inbound pipeline:**
1. Accept connection on port 2525 (iptables redirects 25 to 2525)
2. EHLO negotiation and STARTTLS upgrade
3. SPF record lookup and verification for sender domain
4. Spam scoring: keyword detection, SPF result weighting, bulk header heuristics, all-caps ratio
5. Full MIME parsing: text/plain, text/html, multipart, CC, BCC, Reply-To, raw headers
6. Store to `email_messages` with org isolation
7. Route to recipient mailbox folder

**Outbound pipeline:**
1. DKIM signing with RSA-256, per-domain key stored encrypted in MongoDB
2. Relay via Brevo (smtp.brevo.com:587, STARTTLS + AUTH)
3. On relay failure: write to `email_outbound_queue`
4. Retry worker with exponential backoff: 5m, 30m, 2h, 6h, 24h
5. Bounce NDR generated and delivered to sender after 6 failures

**IMAP server:**
- 5 standard folders: INBOX, Sent, Drafts, Trash, Spam
- Commands: SELECT, FETCH, STORE, SEARCH, COPY, APPEND, SUBSCRIBE, NAMESPACE, EXPUNGE, CLOSE
- Tested against real IMAP clients

**End-to-end verified:** samscorp.xyz to Gmail and Gmail to samscorp.xyz, both directions, DKIM=pass confirmed in Gmail headers.

---

## Security

| Layer | Implementation |
|-------|---------------|
| Encryption at rest | AES-256-GCM, unique 12-byte nonce per call, key from env var only |
| Encryption in transit | TLS everywhere: nginx (public), SMTP STARTTLS, IMAP STARTTLS |
| User authentication | Okta SSO (SAML), session tokens in secure HTTP-only cookies |
| Third-party auth | OAuth 2.0: Google Calendar, Microsoft Outlook, token refresh handled |
| Email authentication | DKIM signing outbound, SPF verification inbound, DMARC at DNS |
| Data isolation | org_id enforced at query level on every MongoDB collection |

---

## Tech Stack

**Backend**
Go 1.22, net/http, net/smtp, crypto/aes, crypto/cipher, crypto/rand, sync, goroutines, channels, MongoDB Go driver

**AI**
Groq API (llama-3.3-70b), Google Gemini API (gemini-2.0-flash), Ollama (gemma4, qwen2.5:7b), Anthropic Claude API. All behind a single `LLMProvider` interface.

**Infrastructure**
Docker, Oracle Cloud ARM64 (4 OCPU / 24GB), nginx, Let's Encrypt (certbot), GitHub Actions CI/CD, self-hosted Actions runner

**Testing**
Go testing package, 80%+ coverage across all packages. Playwright E2E: 292 tests, chromium + mobile-chrome (Pixel 5), all surfaces covered.

**Email**
SMTP RFC 5321, IMAP RFC 3501, DKIM RFC 6376, SPF RFC 7208, DMARC RFC 7489, MIME RFC 2045, STARTTLS RFC 3207, Brevo relay

**Auth**
Okta SSO, OAuth 2.0, AES-256-GCM, PBKDF2 key derivation, secure HTTP-only cookies, multi-tenant org isolation

---

## Architecture

See [ARCHITECTURE.md](ARCHITECTURE.md) for system design, package structure, data flow, and key engineering decisions.

---

## License

Source-available under the [Elastic License 2.0](LICENSE). You may read, run, and modify this code. You may not offer KaryoSpace as a hosted or managed service to third parties.

---

## Author

**Suman Akkisetty**
[karyospace.com](https://karyospace.com) | [linkedin.com/in/sumanakkisetty](https://linkedin.com/in/sumanakkisetty) | sumanakkisetty@gmail.com | Kitchener, Ontario, Canada
