# KaryoSpace

> **AI-native enterprise workspace. One platform, every tool your company needs, on your own infrastructure.**
>
> **[karyospace.com](https://karyospace.com)** — live in production

[![Live](https://img.shields.io/badge/Live-karyospace.com-6366f1?style=flat-square)](https://karyospace.com)
[![Dev](https://img.shields.io/badge/Dev-dev.karyospace.com-64748b?style=flat-square)](https://dev.karyospace.com)
[![License](https://img.shields.io/badge/License-Elastic_2.0-informational?style=flat-square)](LICENSE)
[![Built with](https://img.shields.io/badge/Built_with-Claude_Code-orange?style=flat-square)](https://claude.ai/code)
[![Go](https://img.shields.io/badge/Go-1.24-00ADD8?style=flat-square&logo=go)](https://go.dev)
[![E2E Tests](https://img.shields.io/badge/E2E_Tests-292_passing-brightgreen?style=flat-square)](tests/)
[![Coverage](https://img.shields.io/badge/Go_Coverage-80%25_all_packages-brightgreen?style=flat-square)](backend/go/)

---

## The Problem

Enterprise knowledge is fragmented across 10 to 15 disconnected SaaS products that do not talk to each other in any meaningful way. AI layers bolted on top of fragmented data produce fragmented intelligence. Every AI assistant your company buys sees only the slice of data that one vendor decided to expose.

KaryoSpace solves this at the data layer: email, messaging, tasks, incidents, calendar, and documents in a single platform — or connected from existing tools via REST and MCP adapters. The AI is built on top of that unified layer. The result is an AI that actually knows your company because it has access to everything, not just what one vendor decided to expose.

---

## Three Product Modes

| Mode | Who It's For | Entry Point |
|------|-------------|-------------|
| **Mode 1 — Full Standalone** | Orgs ready to own their stack. Privacy-first. Complete data sovereignty. | Native SMTP/IMAP + 15 built-in modules. No external SaaS dependencies. |
| **Mode 2 — AI Layer** | Orgs keeping Gmail, Jira, Confluence, ServiceNow. Want unified AI. | Connect existing tools via REST or MCP. Point Claude Desktop at the MCP server. Instant org intelligence without migrating a single tool. |
| **Mode 3 — Gradual Migration** | Orgs starting with integrations, replacing tools over time. | Begin in Mode 2. Retire tools one by one. Land in Mode 1 at your own pace. |

The Mode 2 wedge in one sentence: **"Connect your Gmail and Jira in 5 minutes. Ask your AI anything about your org. Replace tools when you're ready."**

---

## What It Replaces

| KaryoSpace Module | Replaces | Status |
|------------------|----------|--------|
| Email (native SMTP/IMAP + OAuth bridge) | Outlook, Gmail | ✅ Production-ready. DKIM, SPF, STARTTLS, MIME. Gmail + Outlook OAuth bridge live. |
| Messaging (WebSocket + WebRTC) | Slack, Teams | ✅ Production-ready. Channels, DMs, threads, reactions, voice/video calls, screen share. |
| Incidents | ServiceNow, PagerDuty | ✅ Production-ready. Status machine, SLA worker, dashboard charts, timeline. |
| Projects / PM | Jira, Linear | ✅ Production-ready. Kanban, sprints, burndown, AI task creation. |
| Knowledge Base | Confluence, Notion | ✅ Production-ready. RAG-indexed, team-restricted visibility, AI chat per doc. |
| Notes | Notion personal | ✅ Complete. WYSIWYG, tags, pin, mobile single-panel slide. |
| Calendar | Google Calendar, Outlook | ✅ Built. Events, recurrence, RSVP, conflict detection. |
| Notifications | - | ✅ Complete. WebSocket bell, type filters, all event types wired. |
| Team / Directory | BambooHR, Workday | ✅ Built. Org chart, department filter, quick Email + Chat actions. |
| Admin | - | ✅ Built. User management, billing, app logs, integration cards. |
| Global Search | - | ✅ Built. Cmd+K across all modules. |
| Home / AI Assistant | - | ✅ Built. Unified AI with RAG across all org data. |
| PWA | - | ✅ Built. Installable. Offline shell. |

---

## AI Infrastructure

### RAG Pipeline — 7 Parallel Searches

Every query is classified before routing. No unnecessary context is passed to the LLM. No data question is answered without grounding.

```
User query
    │
    ├─ IsStatement?    →  "Got it, noted."        (zero API calls)
    ├─ isEmailQuery?   →  MongoDB + Groq summary   (bypass full RAG)
    └─ IsRAGQuery?     →  7 parallel goroutines:
                             ├─ Cache lookup      (cosine sim ≥ 0.88)
                             ├─ Email messages    ($text + cosine re-rank)
                             ├─ Email chunks      (256-word / 50-word overlap)
                             ├─ Chat messages     ($text + adjacent context)
                             ├─ Knowledge Base    ($text + cosine re-rank)
                             ├─ Incidents         (open + resolved)
                             ├─ Notes             (user-scoped)
                             └─ Refined atoms     (KRE-distilled facts)
                                      │
                          CACHE HIT (fresh)  →  return in <200ms
                          CACHE HIT (stale)  →  delta search → merge
                          CACHE MISS         →  build context → LLM → store
                                      │
                          Org context found?
                              YES  →  Local Gemma4  (Ollama — free, private)
                              NO   →  Groq API      (llama-3.3-70b-versatile)
```

### LLM Provider Architecture

All providers implement the same `LLMProvider` interface. Swap via environment variable.

```
LLM_PROVIDER=groq    →  llama-3.3-70b-versatile   (default — cloud, fast)
LLM_PROVIDER=gemini  →  gemini-2.0-flash           (cloud)
LLM_PROVIDER=ollama  →  gemma4 / qwen2.5:7b        (local — air-gapped, zero cost)
LLM_PROVIDER=claude  →  claude-sonnet              (Anthropic)
```

### Knowledge Refinement Engine (KRE)

A background loop that makes the system smarter with every interaction — no human curation required.

**Phase 1 — Semantic Query Cache**
Embeddings stored per response. Cosine similarity threshold 0.88. Fresh cache hits return in under 200ms with no LLM call. Stale cache (>7 days) triggers a delta search — only new data is fetched, then merged with the cached response.

**Phase 2 — Knowledge Distillation Worker**
Every 30 minutes: reads recent queries that hit real sources (not cache hits), calls local Gemma4, extracts structured knowledge atoms — `{topic, facts[], status, confidence}` — stores to `rag_refined_knowledge` with a 30-day TTL. Atoms are prepended to every future query context for that org. The AI accumulates institutional knowledge automatically.

**Phase 3 — Freshness Scoring**
Each atom has a volatility classification. Stable policy atoms (procedures, onboarding docs, architecture decisions) get a 30-day TTL. Volatile atoms (incidents, announcements, sprint status) get 7 days. Stale atoms are excluded from retrieval automatically via MongoDB TTL index. Combined ranking: `similarity × confidence × freshness_score`.

### Embedding Pipeline

```
Text input
    │
    ▼ Truncate to 380 words (nomic-embed-text 512-token limit)
    │
    ▼ POST /api/embeddings → Ollama :11434 → nomic-embed-text
    │
    ▼ 768-dimension float64 vector
    │
    ├─ Stored in MongoDB (email_messages, knowledge_chunks,
    │   rag_email_chunks, rag_query_cache, rag_refined_knowledge)
    └─ Cosine similarity re-ranking during RAG retrieval
```

### AIO: AI Observability Engine

Every LLM interaction produces a trace record stored in `aio_traces`.

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

Quality is flag-based. `empty_retrieval` −30, `low_similarity` −20, `no_sources` −25, `cache_hit` +10. Aggregated per org: cache hit rate, average quality score, flagged interactions, token spend by day.

---

## MCP Architecture — Bidirectional

KaryoSpace implements MCP in two directions simultaneously.

**WorkSpace as MCP Server** — any AI client (Claude Desktop, Cursor, GPT) connects and queries all org data via standard tools. Companies point their existing AI at KaryoSpace and get org-wide intelligence without migrating a single tool.

```json
{
  "mcpServers": {
    "workspace": {
      "command": "workspace-mcp",
      "args": ["--org", "yourcompany.com", "--token", "mcp_YOUR_API_KEY"]
    }
  }
}
```

Available tools: `search_workspace`, `list_incidents`, `get_knowledge`, `list_tasks`, `get_email_thread`. Transport: stdio + HTTP+SSE.

**WorkSpace as MCP Client** — connects to other tools' MCP servers (Slack, GitHub, etc.) as DataSource adapters. As vendors publish MCP servers, KaryoSpace replaces bespoke REST adapters with MCP client calls — zero custom API parsing.

---

## Integration Layer

All integrations implement a common `DataSource` interface: `Connect()`, `Sync()`, `Vectorize()`, `HandleWebhook()`. Transport-agnostic — data can arrive via REST, MCP, webhooks, or file imports.

| Integration | Status | Notes |
|------------|--------|-------|
| Gmail OAuth sync | ✅ Live | IMAP OAuth. Virtual folder dedup. Token auto-refresh. |
| Outlook OAuth sync | ✅ Live | IMAP OAuth for personal accounts. JWT decode for email extraction. |
| ServiceNow | ✅ Live | 449 chunks: 64 INC, 84 CHG, 24 PRB, 40 CMDB, 65 KB, 172 Activity. Auto-polls every 15 min. 40/40 tests pass. |
| Confluence Cloud | ✅ Built | OAuth + incremental CQL sync, HTML stripped, comments appended. |
| Jira Cloud | ✅ Built | JQL incremental sync. ADF→text. Rate-limited 10 req/s. 31/31 tests. |
| Slack MCP client | ✅ Built | Env-var driven (`SLACK_BOT_TOKEN + SLACK_MCP_COMMAND`). Awaiting org credentials. |
| Email vectorization | ✅ Built | Backfill loop on every sync. Self-heals Gemini 429/503 failures. |

SyncWorker polls every 15 minutes. Webhook-capable sources get 60-minute fallback. All synced content is vectorized and indexed into `rag_integration_chunks` — available to the RAG pipeline on every query.

---

## Email Infrastructure

A complete SMTP and IMAP server built from scratch in Go. No third-party email library. RFC-compliant from the ground up.

**Inbound pipeline:**
1. Accept connection on port 2525 (iptables redirects 25→2525)
2. EHLO negotiation and STARTTLS upgrade
3. SPF record lookup and verification for sender domain
4. Spam scoring: keyword detection, SPF result weighting, bulk header heuristics, all-caps ratio
5. Full MIME parsing: text/plain, text/html, multipart, CC, BCC, Reply-To, raw headers
6. Store to `email_messages` with org isolation
7. Route to recipient mailbox folder

**Outbound pipeline:**
1. DKIM signing with RSA-256, per-domain key stored AES-256-GCM encrypted in MongoDB
2. Relay via Brevo (smtp.brevo.com:587, STARTTLS + AUTH)
3. On relay failure: write to `email_outbound_queue`
4. Retry worker with exponential backoff: 5m → 30m → 2h → 6h → 24h
5. Bounce NDR generated and delivered to sender after 6 failures

**IMAP server:**
- 5 standard folders: INBOX, Sent, Drafts, Trash, Spam
- Commands: SELECT, FETCH, STORE, SEARCH, COPY, APPEND, SUBSCRIBE, NAMESPACE, EXPUNGE, CLOSE
- Tested against real IMAP clients

**End-to-end verified:** karyospace.com → Gmail and Gmail → karyospace.com, both directions, DKIM=pass confirmed in Gmail headers.

---

## WebRTC Voice and Video

Full in-browser voice and video built directly into the Messaging module.

- 1:1 calls from any DM
- Group calls from any channel
- Screen share
- coturn TURN relay running on Oracle VM port 3478
- No external calling infrastructure

---

## Infrastructure

Three completely independent Oracle Cloud environments. Each has its own MongoDB instance, own org domain, no shared data.

| Environment | URL | Shape | Role |
|------------|-----|-------|------|
| karyospace-prod | karyospace.com | ARM64 · 4 OCPU · 24 GB | Production. Docker `workspace-prod` container. |
| karyospace-dev | dev.karyospace.com | ARM64 · 4 OCPU · 24 GB | Development. Docker `workspace-dev` container. GitHub Actions runner. |
| workapp-server | workspace.samscorp.xyz | AMD64 · E2.1.Micro · 1 GB | Backup dev. systemd. Safety net. |

**Traffic flow (karyospace-prod):**
```
Browser → Namecheap DNS → karyospace.com → Oracle VM (40.233.102.246)
        → nginx (:443 TLS) → Docker workspace-prod (:8080)
        → Go binary → MongoDB (172.17.0.1:27017 / ai_pipeline)
                    → Ollama (:11434 / nomic-embed-text + gemma4)
                    → Groq API (cloud / llama-3.3-70b)
```

### One-Liner Install

```bash
curl -fsSL https://get.workspace.samscorp.xyz | bash
```

Downloads docker-compose.yml, prompts for domain and admin email, auto-generates all secrets (SESSION_STORE_KEY, DKIM_ENCRYPTION_KEY, MONGO passwords), runs `docker compose up -d`, polls `/health` until 200 OK.

---

## Security

| Layer | Implementation |
|-------|---------------|
| Encryption at rest | AES-256-GCM. Unique 12-byte nonce per call. Key from env var only — never stored. |
| Encryption in transit | TLS everywhere: nginx (public), SMTP STARTTLS, IMAP STARTTLS. |
| Authentication | OIDC/SAML (generic — works with Okta, Auth0, Entra, any OIDC provider). Local auth + TOTP. |
| OAuth | Google Calendar, Microsoft Outlook, Confluence, Jira. Token refresh handled automatically. |
| Email authentication | DKIM signing outbound, SPF verification inbound, DMARC at DNS. |
| Data isolation | org_id enforced at query level on every MongoDB operation. Cross-org queries are architecturally impossible. |
| Observability | All 5xx errors persisted to `app_logs` (MongoDB capped collection). Prometheus `/metrics` endpoint live. |
| Security grade | A — all known findings resolved. |

---

## Build Stats

Built by one person, during off-hours, while employed full-time, using Claude Code as the primary development tool. No team. No funding. No laptop.

| Metric | Value |
|--------|-------|
| Total commits | 1,614 |
| Claude Code sessions | 525+ |
| AI pair programming hours | 300+ |
| Time to first production deploy | 7 months |
| Go test coverage | 80%+ across every package (CI gate) |
| Playwright E2E tests | 292 tests, chromium + mobile-chrome (Pixel 5, 393×851px) |
| E2E result | 291 pass, 1 skip, 0 fail |

**Monthly commit cadence:**

| Period | Commits | Setup |
|--------|---------|-------|
| Nov 2024 – Feb 2026 | 53 | Traditional: local machine, part-time |
| March 2026 | 438 | Cloud-only: Oracle VM + Claude Code |
| April 2026 | 428 | Phone browser only — no laptop |
| May 2026 | 601 | Same phone setup. Full production velocity. |
| June 2026 | ongoing | Production-live. Active feature development. |
| **Total** | **1,614+** | |

The entire compiler, Claude Code environment, Docker runtime, and CI/CD pipeline run on a free Oracle Cloud ARM64 VM. The only client is a phone browser. No desk. No fixed schedule. 1,614 commits from that setup.

---

## Billing Infrastructure

Stripe billing is built and wired. Subscription management, webhook handler, per-org plan tracking, and an admin billing page are all in production — ready for when beta ends.

**Currently in beta: free and open to all.** See [karyospace.com/info#pricing](https://karyospace.com/info#pricing) for current access details.

---

## Demo Seed CLI

A CLI tool that seeds a realistic organization with production-scale data for demos and testing. Five industry verticals supported. Generates approximately 5,200 documents per domain: emails, incidents, KB articles, tasks, messages, calendar events. Seeds in under 2 minutes.

---

## Visitor Analytics

Self-hosted. No third-party tracking. No CDN leakage.

- IP geolocation via MaxMind GeoLite2 (bundled on server)
- World heatmap rendered in-browser via Leaflet.js
- All event data stored in MongoDB — never leaves your server
- Admin analytics dashboard at `/admin` → Analytics tab

---

## Tech Stack

**Backend**
Go 1.24, `net/http`, `net/smtp`, `crypto/aes`, `crypto/cipher`, `crypto/rand`, `sync`, goroutines, channels, MongoDB Go driver 1.17

**AI**
Groq API (llama-3.3-70b-versatile), Ollama (gemma4 + nomic-embed-text 768-dim), Google Gemini API (gemini-2.0-flash), Anthropic Claude API — all behind a single `LLMProvider` interface

**Integrations**
Gmail OAuth (IMAP), Outlook OAuth (IMAP), Confluence Cloud REST, Jira Cloud REST, ServiceNow REST, Slack MCP client — all behind a single `DataSource` interface

**Infrastructure**
Docker, Oracle Cloud ARM64 (4 OCPU / 24 GB), nginx, Let's Encrypt (certbot), GitHub Actions CI/CD, self-hosted Actions runner, coturn TURN relay

**Real-time**
WebSocket hub (goroutine per connection), WebRTC (voice, video, screen share), Server-Sent Events (MCP HTTP transport)

**Testing**
Go testing package, 80%+ coverage enforced as CI gate. Playwright E2E: 292 tests, chromium + mobile-chrome. No mocking policy — tests hit real MongoDB, real dev server.

**Email**
SMTP RFC 5321, IMAP RFC 3501, DKIM RFC 6376, SPF RFC 7208, DMARC RFC 7489, MIME RFC 2045, STARTTLS RFC 3207

**Auth**
Generic OIDC/SAML (Okta, Auth0, Entra, any provider), local auth + TOTP, OAuth 2.0 (Google + Microsoft), AES-256-GCM session encryption, secure HTTP-only cookies, org-level data isolation

---

## Architecture

See [ARCHITECTURE.md](ARCHITECTURE.md) for full system design: package structure, request lifecycle, AI query pipeline, KRE architecture, MCP bidirectional design, email state machines, data model, multi-tenancy enforcement, and deployment pipeline.

---

## License

Source-available under the [Elastic License 2.0](LICENSE). You may read, run, and modify this code. You may not offer KaryoSpace as a hosted or managed service to third parties.

---

## Author

**Suman Akkisetty**
[karyospace.com](https://karyospace.com) · [linkedin.com/in/sumanakkisetty](https://linkedin.com/in/sumanakkisetty) · sumanakkisetty@gmail.com · Kitchener, Ontario, Canada
