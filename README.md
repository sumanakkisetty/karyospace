# KaryoSpace

> **Your work, your data, your AI — on one screen.**
>
> **[karyospace.com](https://karyospace.com)** — live in production
>
> *AI-native enterprise workspace. Self-hosted or cloud. Three deployment modes from one binary.*

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

## Why Now

Three things converged between 2025 and 2026 that did not exist before:

1. **Groq, Together, Cerebras** made cloud LLM inference roughly 50× cheaper than the GPT-4 era. This made local-model alternatives (Ollama + Gemma + nomic-embed-text) a viable production substrate for the first time — privacy without paying a 10× quality tax.
2. **MCP shipped.** "Give the AI access to your org data" went from an 8-week data warehouse pipeline project to a 5-minute Claude Desktop config. The Mode 2 wedge ("connect existing tools in 5 minutes") simply was not buildable before MCP.
3. **Free ARM64 compute** (Oracle, Hetzner, AWS Graviton spot) made self-hosting accessible to a 50-person company without an infra team. The Mode 1 promise of "your data, your VM" required $300K of Snowflake/Datadog spend in 2022. In 2026 it's a docker compose file.

Pre-2025, "your work, your data, your AI on one screen" required enterprise budget. In 2026 it requires a weekend. That's the window.

## Founder Context

KaryoSpace is built by Suman Akkisetty — 16 years inside Fortune 500 IT (Cognizant 2016–present, DXC 2010–2016), running 30+ person delivery programs and sitting in production war rooms watching teams switch between ServiceNow, Jira, Confluence, Outlook, Teams, and Excel to find out what is actually happening.

The product is the answer to a lived complaint, not a market hypothesis. The Mode 2 wedge — "your AI knows ServiceNow without your team learning ServiceNow" — is the specific ask heard from VP Engineering buyers in actual customer calls over a decade. KaryoSpace exists because the person building it has been on the receiving end of the problem for sixteen years.

---

## Three Product Modes

The biggest strategic decision in the product: "replace all your SaaS tools" is too high a bar to clear for an initial sale. The same platform serves three completely different buyer profiles without any code branching — same binary, same data model, different entry point.

### Mode 1 — Full Standalone
**For orgs ready to own their stack. Privacy-first. Complete data sovereignty.**

Native SMTP/IMAP + 15 built-in modules. No external SaaS dependencies. A company can run its entire operation — email, chat, tasks, incidents, docs — on a single self-hosted VM.

The unlock: air-gapped deployments, data residency requirements, companies that have been burned by SaaS vendor price hikes or lock-in.

### Mode 2 — AI Layer on Existing Tools
**For orgs keeping Gmail, Jira, Confluence, ServiceNow — but wanting unified AI.**

Connect existing tools via REST adapters or MCP. KaryoSpace vectorizes all incoming data and makes it available to a unified AI. No tool migration required.

The wedge: **"Connect your Gmail and Jira in 5 minutes. Ask your AI anything about your org. Replace tools when you're ready."**

The MCP wedge: **"Point Claude Desktop at KaryoSpace. It instantly knows your email, Jira tickets, Confluence pages, and incidents — without migrating a single tool."**

This is a 5-minute sale, not a 6-month procurement cycle. KaryoSpace becomes AI infrastructure, not a UI competitor. It doesn't need to win the UI battle against Slack or Jira — it becomes the data layer that makes those tools useful to AI.

### Mode 3 — Gradual Migration
**For orgs that start in Mode 2 and replace tools at their own pace.**

Because KaryoSpace was already syncing all the data in Mode 2, migration is a cutover — not a data migration project. Retire one SaaS tool at a time. Eventually land in Mode 1.

---

| Mode | Entry | Data Source | AI Coverage |
|------|-------|-------------|-------------|
| 1 — Standalone | Native modules only | All data in KaryoSpace | Full org context |
| 2 — AI Layer | Connect existing tools | Synced from Gmail / Jira / Confluence / ServiceNow / Slack | Full org context, no migration |
| 3 — Migration | Start in Mode 2 | Hybrid (native + synced, converging to native) | Full org context throughout |

---

## The Product Surface

KaryoSpace is one input field — the Home AI command bar — backed by five core modules. Everything else is infrastructure underneath.

```
┌─────────────────────────────────────────────────────────────┐
│  Home — natural language input + slash commands              │
│  ────────────────────────────────────────────────────────    │
│  > "What are the open P1 incidents blocking the Q3 launch?"  │
│  > /jira create — bug in checkout flow                       │
│  > /servicenow raise — VPN outage in EMEA                    │
└─────────────────────────────────────────────────────────────┘
                            │
       ┌────────────────────┼────────────────────┐
       ▼                    ▼                    ▼
   ┌────────┐         ┌──────────┐        ┌──────────┐
   │ Email  │         │ Messaging│        │Incidents │
   └────────┘         └──────────┘        └──────────┘
              ┌──────────┐         ┌──────────┐
              │Knowledge │         │  Notes   │
              └──────────┘         └──────────┘
```

The product is the command bar. Everything routes through it.

### What KaryoSpace replaces

In Mode 1 (full standalone) for SMB orgs:

| Module | Replaces | Coverage |
|--------|----------|----------|
| Email (native SMTP/IMAP + OAuth bridge) | Outlook, Gmail | ~85% — DKIM, SPF, STARTTLS, MIME, OAuth bridge live |
| Messaging (WebSocket + WebRTC) | Slack, Teams | ~90% — Channels, DMs, threads, voice/video, screen share |
| Incidents | ServiceNow, PagerDuty | ~75% (SMB scope — see below for enterprise positioning) |
| Knowledge Base | Confluence, Notion | ~82% — RAG-indexed, team-restricted visibility, AI chat per doc |
| Notes | Notion personal | ~100% — WYSIWYG, tags, pin, mobile single-panel slide |

In Mode 2 (AI layer on existing tools), KaryoSpace **augments** rather than replaces. ServiceNow, Jira, Confluence stay live — KaryoSpace makes them queryable from a single AI surface. Mode 1 replacement is for SMB orgs (10-100 employees). Enterprise IT stays on ServiceNow + Jira and uses KaryoSpace as the AI infrastructure layer over them.

**Infrastructure modules** (always running, not in the surface story): Calendar, Notifications, Team Directory, Admin, Global Search, Home/AI, PWA, MCP server, Visitor Analytics, AIO. See [ARCHITECTURE.md](ARCHITECTURE.md) for the full system map.

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
| Data isolation | `org_id` enforced at query level on every MongoDB operation. CI linter (`mongoctl-lint`) fails the build if any `Find/FindOne/Aggregate` is missing the `org_id` filter — defense in depth against accidental cross-org reads. |
| Observability | All 5xx errors persisted to `app_logs` (MongoDB capped collection). Prometheus `/metrics` endpoint live. |
| Security review | Internal full-codebase review 2026-05-08 — Grade A, zero critical or high findings. **Not third-party.** SOC 2 Type I targeted Q4 2026 once first paying customers fund the audit (Vanta-managed). Beta customers receive a security questionnaire response + DPA, not a SOC 2 letter. |

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

### Beta-to-paid commitment (in writing)

- **All beta orgs:** 12 months free on the lowest paid tier after beta ends. No surprise migration, no forced churn.
- **First 100 beta orgs ("Founding Members"):** lifetime grandfather at the lowest published price. The people who took the risk get the price for life.

This is contractual, not aspirational. Sign-up confirmation captures Founding Member status.

---

## Production Realities (the honest section)

This section exists because senior buyers ask these questions and deserve direct answers.

**Bus factor: 1.** KaryoSpace is founder-led and solo-built today. The standard customer contract includes (a) source escrow with a 30-day customer-side migration window, and (b) full source access for Founding Members regardless of company status. Source is also published under EL2 on this repo — customers can self-host the codebase at any time.

**Operational maturity.** Single founder + 7 months ≠ enterprise-grade incident response. No 24/7 on-call rotation today. Incident SLA in beta is best-effort with target 4-hour acknowledgement during business hours (ET). Mode 1 self-hosted customers are unaffected by KaryoSpace operational status — their VM, their uptime.

**Scale ceiling, today.** Production karyospace.com runs comfortably at single-org scale (current peak: ~8K total chunks across all RAG collections). MongoDB cosine search is the right substrate up to ~50K chunks per org. Past that, the planned migration is pgvector for SMB / Qdrant for enterprise — the `DataSource.Vectorize()` interface returns a transport-agnostic `Chunk`, the swap is plumbing, not a rewrite.

**Latency, measured.** Full 7-source parallel RAG gather: p50 280ms, p95 540ms, p99 740ms at current load. Cache-hit path: p99 under 200ms. LLM call (Groq llama-3.3-70b) typically adds 700-1400ms; local Gemma synthesis 1.2-2.4s.

**Unit economics, napkin.** At 10K daily active users × 30 AI queries/user/day × 60% cache hit: ~120K un-cached Groq queries/day ≈ $190/day infra. MongoDB Atlas at that scale ≈ $400/mo. Oracle ARM64 paid tier at 4 OCPU × 5 instances ≈ $300/mo. Total ≈ $6,500/mo to serve 10K DAU. At $8/user Business tier with 10% paid conversion: ~$8K revenue. The KRE cache hit rate and local-Gemma routing for org-context queries are the margin levers — not nice-to-haves.

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
