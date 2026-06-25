# KaryoSpace: Product Vision

*This document captures the product thinking behind KaryoSpace — why it was built this way, what it's replacing, how the strategy evolved, and where it's going. It's a living record of decisions, not a static spec.*

---

## Core Thesis

Enterprise knowledge is fragmented across 10–15 disconnected SaaS products that do not talk to each other in any meaningful way. AI layers bolted on top of fragmented data produce fragmented intelligence. Every AI assistant your company buys sees only the slice of data that one vendor decided to expose.

The alternative: own the data layer entirely. Email, messaging, tasks, incidents, knowledge, and calendar all in one platform — or connected from existing tools. Then build AI on top of that unified layer. The result is an AI that actually knows your company because it has access to everything.

**"Karyo" means "work" in Sanskrit.** The domain is karyospace.com because workspace.com was unavailable. The company behind it is SAMSCORP (samscorp.xyz).

---

## The Three-Mode Strategy

The biggest strategic decision in the product was recognizing that "replace all your SaaS tools" is too high a bar to clear for an initial sale. That led to a three-mode architecture where the same platform serves very different buyer profiles without any code branching.

### Mode 1 — Full Standalone
**For:** Orgs ready to own their stack. Privacy-first. Complete data sovereignty.

Native SMTP/IMAP + 15 built-in modules. No external SaaS dependencies. A small company can run their entire operation — email, chat, tasks, incidents, docs — on a single self-hosted VM.

**The unlock:** Air-gapped deployments, industries with data residency requirements, companies that have been burned by SaaS vendor price hikes.

### Mode 2 — AI Layer on Existing Tools
**For:** Orgs keeping Gmail, Jira, Confluence, ServiceNow. They want AI intelligence across everything — but they're not ready to migrate.

Connect existing tools via REST adapters or MCP. The platform vectorizes all incoming data and makes it available to a unified AI. The entry point is frictionless: **"Connect your Gmail and Jira in 5 minutes. Ask your AI anything about your org. Replace tools when you're ready."**

The MCP wedge sharpens this further: **"Point Claude Desktop at KaryoSpace. It instantly knows your email, Jira tickets, Confluence pages, and incidents — without migrating a single tool."**

**Why Mode 2 changes the total addressable market:** KaryoSpace stops being "another SaaS app" and becomes AI infrastructure. The company doesn't need to migrate tools — they point their existing AI at KaryoSpace and get org-wide intelligence immediately. This is a 5-minute sale, not a 6-month procurement cycle.

### Mode 3 — Gradual Migration
**For:** Orgs that start in Mode 2 and replace tools at their own pace.

Land in Mode 2. Remove one SaaS tool at a time. The platform already has all the data (it was syncing it in Mode 2), so migration is a cutover, not a data migration project. Eventually land in Mode 1.

This is not a separate product mode — it's the natural progression path for Mode 2 customers who find that the native tools are good enough to replace the originals.

---

## Ideal Customer Profile

| Segment | Size | Starting Tools | Entry Point |
|---------|------|---------------|-------------|
| SMB | 10–50 | Gmail + Slack + Jira | Connect Gmail + Jira → AI context → replace Slack with native chat |
| Mid-market | 50–500 | Jira + Confluence + Outlook | Connect all 3 → unified AI search → replace tools gradually |
| Privacy-first | Any | Any | Full self-hosted. Native email. Complete data sovereignty. |
| Enterprise IT | 500+ | ServiceNow + Jira + Confluence | ServiceNow CMDB + tickets + Confluence + email → org-wide AI |

---

## What It Replaces — and How Much

Full product parity audit conducted against each replaced tool:

| Tool Replaced | Feature Coverage | Primary Remaining Gaps |
|--------------|-----------------|----------------------|
| Outlook / Gmail | ~85% | Email forwarding rules, auto-responders, calendar-email sync |
| Slack / Teams | ~90% | Pinned messages, scheduled messages (minor) |
| Jira | ~80% | Time tracking, custom fields, webhooks |
| Confluence / Notion | ~82% | Collaborative real-time editing, PDF export |
| ServiceNow | ~75% | Approval/escalation workflows, custom SLA policies, CAB |

**Overall: ~78% feature parity across all 5 replaced tools — production-ready for most enterprise workflows.**

Cost displacement per user per year:

| Tool | Savings |
|------|---------|
| Outlook / Gmail | $150–276 |
| Slack / Teams | $96–150 |
| Confluence / Notion | $60–120 |
| Jira / ServiceNow | $84–1,200 |
| **Total** | **$390–1,746/user** |

---

## The "One UI — No App Switching" Vision

> WorkSpace is a command centre, not another inbox. Users should never need to open Gmail, Outlook, Jira, Confluence, or ServiceNow. Every action should be possible from Home via natural language or slash commands.

| Instead of opening… | User types on Home… |
|--------------------|---------------------|
| Jira | `/jira create` → issue filed without tab switch |
| Confluence | `/confluence search docs` → results inline |
| ServiceNow | `/servicenow raise incident` → ticket opened |
| Gmail | `/gmail` → compose + send without opening Gmail |
| Outlook | `/outlook` → compose + send without opening Outlook |

Email integration scope (decided May 2026): **Read + Send + Briefing** — not a full inbox replacement in Mode 2. KaryoSpace is not trying to be an email client when Gmail/Outlook are connected. It's the AI layer that reads, summarizes, and acts on them.

---

## MCP: From Tool to Infrastructure

The MCP (Model Context Protocol) decision was the biggest strategic pivot in the product. It was added in April 2026 after recognizing that being "another SaaS app" is the wrong positioning.

**The shift:** KaryoSpace implements MCP in both directions.

**As MCP Server:** Any AI client (Claude Desktop, Cursor, GPT) connects and queries all org data via standard tools. This is Mode 2 in one line: companies don't need to use KaryoSpace's UI at all. They point their preferred AI at KaryoSpace's MCP server and get org intelligence.

**As MCP Client:** Instead of writing bespoke REST adapters for every integration, KaryoSpace connects to other tools' MCP servers directly. As vendors (Slack, GitHub, ServiceNow) publish MCP servers, the integration cost approaches zero — no custom API parsing, no schema maintenance.

**Why this changes the company's position:** KaryoSpace becomes AI infrastructure, not a UI competitor. It doesn't need to win the UI battle against Slack or Jira. It becomes the data layer that makes those tools useful to AI.

---

## The AI Architecture Decision Chain

Several non-obvious AI architecture decisions shaped the final system:

### Decision 1: RAG over fine-tuning
Org data changes constantly — incidents are opened and closed, emails arrive, docs are updated. Fine-tuning a model on org data would require continuous retraining. MongoDB `$text` search with cosine re-ranking gives retrieval that reflects today's data, not last month's training snapshot.

### Decision 2: Dual LLM routing, not a single model
Two models handle different jobs for different cost and quality profiles:
- **Groq (llama-3.3-70b):** Classification, email summaries, general answers — cloud, fast, low cost per query
- **Local Gemma4 (Ollama):** RAG synthesis, KRE distillation — runs locally, zero API cost, keeps org data off cloud pipes

When org context is found, local Gemma4 handles the synthesis. When no org context exists, Groq handles general queries. This preserves Groq rate limits for where they matter and keeps sensitive org data local.

### Decision 3: Knowledge Refinement Engine (KRE) — build self-improvement in from day one

The insight: every answered question is an opportunity to distill structured knowledge. A system that accumulates org knowledge over time has a compounding advantage that a fresh competitor cannot replicate.

KRE runs in three phases, all fully automated:

**Phase 1 — Semantic Query Cache**
Every response is embedded and stored. Future similar queries (cosine similarity ≥ 0.88) are served from cache in under 200ms with no LLM call. Cache entries older than 7 days trigger a delta search — only documents newer than the last refresh are fetched, merged with the cached response, and served.

**Phase 2 — Knowledge Distillation**
Every 30 minutes, a background worker reads recent queries that hit real org sources (not cache hits). Local Gemma4 extracts structured knowledge atoms: `{topic, facts[], status, confidence}`. Atoms are stored in `rag_refined_knowledge` with a 30-day TTL. Every future query gets pre-distilled context prepended automatically — no human curation required.

**Phase 3 — Freshness Scoring**
Not all knowledge ages at the same rate. Stable atoms (SLA policy, onboarding docs, architecture decisions) live 30 days. Volatile atoms (active incidents, sprint status, announcements) live 7 days. Combined retrieval ranking: `similarity × confidence × freshness_score`. Stale atoms trigger background re-distillation — they're not just deleted, they're refreshed.

**The compounding moat:** A customer who asks about incident status on Day 1 gets a synthesized response from raw search. On Day 5, they get instant cache plus any updates since Day 1. On Day 30, they get pre-structured facts extracted from 30 days of related Q&A — all without any user action. Competitors starting fresh cannot replicate 6 months of accumulated `KnowledgeAtom` context.

**Privacy model:** Org-scoped atoms are shared across the org. User-scoped cache entries are only retrievable by the owning user. Email content never enters distillation — only Incidents, KB, Chat, Confluence, Jira, and ServiceNow data qualifies.

---

## Integration Architecture: The DataSource Pattern

All integrations implement a single transport-agnostic interface:

```go
type DataSource interface {
    Connect(ctx context.Context) error
    Sync(ctx context.Context) ([]Document, error)
    Vectorize(doc Document) ([]Chunk, error)
    HandleWebhook(payload []byte) error  // optional — falls back to 15-min polling
}
```

This decision was made early to avoid integration-specific code paths proliferating through the codebase. Every integration — whether Gmail IMAP OAuth, Jira REST, ServiceNow REST, or Slack MCP — is just another DataSource. The vectorization, chunking, and embedding pipeline is the same for all of them.

Live integrations as of June 2026:
- Gmail (IMAP OAuth, virtual folder dedup, token auto-refresh)
- Outlook (IMAP OAuth, JWT email extraction, provider-prefixed storage)
- ServiceNow (REST — 449 chunks live: 64 INC, 84 CHG, 24 PRB, 40 CMDB, 65 KB, 172 Activity)
- Confluence Cloud (OAuth + incremental CQL sync)
- Jira Cloud (JQL incremental sync, ADF→text, rate-limited 10 req/s)
- Slack (MCP client adapter — wired, awaiting org credentials)

### Sandbox Strategy for Demos

Demoing integrations without commercial agreements is a real operational problem. The strategy: every major vendor has a free sandbox tier.

| Service | Free Tier | Setup Time |
|---------|-----------|-----------|
| Gmail | Google Cloud OAuth Testing mode (no app review) | ~30 min |
| Outlook | M365 Developer Program — free 90-day renewable sandbox | ~1 hour |
| Confluence + Jira | Atlassian free Cloud (≤10 users, forever free) | ~30 min |
| ServiceNow | PDI — Personal Developer Instance (free, always-on) | ~1 hour |
| Slack | Free workspace + bot token | ~20 min |

Total setup for a fully live demo environment: ~4 hours.

---

## Product Navigation Philosophy

Showing too many features at once fragments the product story. The navigation decision (made April 2026) was to expose a clean 7-module story and hide everything else. Routes remain fully intact — nothing is deleted. Re-surfacing is a one-line change.

| Module | Nav Status | Reasoning |
|--------|-----------|-----------|
| Home, Email, Messaging, Knowledge, Incidents, Notes, Team | ✅ Visible | Core product story |
| Projects / PM | ✅ Visible (restored April 2026) | Added back for leadership use case: "What's happening in Project X?" |
| Calendar | Hidden | A siloed calendar without two-way Google/Outlook sync creates a second calendar problem. Will re-surface when bidirectional sync is built. |
| Notifications page | Bell-only | The bell icon is the canonical entry point. A separate nav link was redundant. |
| DataView | Hidden | Internal AI query inspector — admin and developer use only. |

The principle: every visible nav item should be instantly comprehensible to a new user and immediately useful. Anything that needs explanation or setup before it's useful gets hidden until it's complete.

---

## Licensing Model

Open core — not fully open source. Currently in beta and free to use. See [karyospace.com/info#pricing](https://karyospace.com/info#pricing) for current access details.

The planned model is tiered: self-hosted community access, cloud SaaS for teams that don't want to manage infrastructure, and an enterprise tier with ServiceNow integration, compliance docs, and dedicated support. Pricing will be announced when beta ends.

Public showcase source is available under the [Elastic License 2.0](LICENSE) — read, run, and modify freely; may not be offered as a hosted service to third parties.

---

## Demo Seed CLI

Demoing a workspace tool with an empty database is a failed demo. The seed CLI generates realistic multi-module data for any of 5 industry verticals in under 2 minutes.

Per domain, it generates:
- 80 employees with a realistic social graph (dept-first connection algorithm)
- 30 projects + 90 sprints + 540 tasks
- 120 incidents + 360 update comments
- 90 email threads (60% inbox / 40% sent) attached to the demo user
- 80 calendar events (past/current/future)
- 50 knowledge docs with 250 RAG-ready chunks
- 8 public channels + ~120 DM pairs with messages that cross-reference real seeded data
- 50 notifications (30 unread, 20 read)

The DM messages reference real incidents, projects, and docs from the same domain — so when you ask the AI "what was the status of the payment gateway incident?" the answer exists in the seeded chat history and incident record.

Safety: every seeded document carries `_seeded: true` and `_seed_domain` tags. A single command wipes one domain or all domains without touching real user data.

Industries: Insurance, Banking, Healthcare, Manufacturing, Logistics.

---

## AIO: AI Observability Engine

Every LLM interaction is instrumented. Every query produces a complete trace record stored in `aio_traces`. Nothing goes unmonitored.

The design principle: observability is not a dashboard bolted on after the fact — it's a first-class pipeline step. The `Tracer` object is created at the start of every query and passed through the full pipeline, accumulating measurements at each step before being finalized and persisted asynchronously.

### What gets traced per query

Each `AITrace` record captures the full story of one query:

**Pipeline step timing (milliseconds)**
- `classify_ms` — time to classify intent (RAG, email, general, statement)
- `cache_ms` — time to query the semantic cache
- `retrieve_ms` — time for all 7 parallel RAG searches combined
- `llm_ms` — time for the LLM to respond
- `total_ms` — wall clock for the full pipeline

**Retrieval quality**
- Per-source stats: `{source, count, max_sim, avg_sim}` for each of the 7 sources searched
- `total_chunks` — total retrieved chunks across all sources
- `top_sim` — highest cosine similarity score found
- `cache_hit` + `cache_sim` — whether and how confidently the cache answered

**Token usage and cost**
- `prompt_tokens`, `completion_tokens` (estimated from character count when exact counts unavailable)
- `est_cost_usd` — estimated at Groq pricing ($0.59/M input, $0.79/M output). Local Ollama queries cost $0 — this shows directly in the economics dashboard

**User feedback**
- `user_rating` — thumbs up (+1) or thumbs down (−1) stored per trace. Closes the loop from user signal back to trace record

### Anomaly flags and quality scoring

Eight flags, computed automatically at `Finalize()`:

| Flag | Condition | Quality Impact |
|------|-----------|---------------|
| `empty_retrieval_answered` | RAG query, zero chunks found, LLM responded anyway | −35 |
| `no_sources` | RAG query returned zero source categories | −20 |
| `low_similarity` | Best cosine similarity < 0.35 | −15 |
| `slow_llm` | LLM step > 10 seconds | −10 |
| `slow_total` | Full pipeline > 20 seconds | −10 |
| `high_cost` | Estimated cost > $0.05 per query | −5 |
| `cache_hit` | Served from semantic cache (informational) | — |
| `statement_only` | Input was a statement, no LLM needed (informational) | — |

Quality score starts at 100 and deductions accumulate. Score floor is 0. A query that retrieved nothing and the LLM answered blind lands at 45 or lower — immediately visible in the admin dashboard.

### Admin observability dashboard

`/admin` → Observability tab:

- **24-hour health summary:** total traces, average latency, average quality score, flagged count, cache hit rate, flag breakdown by type
- **Trace table:** recent queries, filterable to flagged-only, sortable, with expandable source stats and chunk detail
- **Flag breakdown:** count per flag type across the time window — shows at a glance whether the system is retrieving well or answering blind

All traces carry a 30-day TTL index — auto-deleted after 30 days. Storage scales with usage, not time.

### Why AIO matters for enterprise sales

Regulated industries (healthcare, finance, legal) require auditability before AI gets past compliance. AIO answers: "What did the AI use to answer that question, and why did it score the way it did?" That's not a nice-to-have — it's a procurement gate.

---

## Visitor Analytics — Self-Hosted, Zero Third Parties

The public-facing pages at karyospace.com track visitors with zero reliance on Google Analytics, Cloudflare, or any external service. All data stays in MongoDB. All processing runs in Go on the same server.

**Why self-hosted analytics:** Product dogfooding. If KaryoSpace is making the case for data sovereignty, it should practice it. The analytics stack is also a demonstration of what the platform can do — build complex data pipelines without external dependencies.

### What gets collected

Every page load on `/info`, `/story`, `/docs`, and `/help` calls `GET /api/visit?page=<key>`. The handler:

1. Validates the page key against a whitelist (4 allowed values)
2. Rejects bots via 50-signal UA pattern matching (curl, Googlebot, Slackbot, headless browsers, and more)
3. Enriches with GeoIP via **MaxMind GeoLite2** (bundled on-server, ~55MB — no external API call)
4. Stores the event and returns `{"count": N}` — unique visitor count for that page

Each event records:
- Page, timestamp (UTC), `ip_hash` (SHA-256 truncated to 8 chars — GDPR-safe, never raw IP)
- Session ID (30-minute rolling cookie, HttpOnly + Secure)
- Country, region, city, lat/lon (for heatmap clustering)
- Device type (desktop/mobile/tablet), browser, OS, referrer hostname

### Admin analytics dashboard

`/admin/analytics` — admin-only, no external CDN dependencies:

- **KPIs:** unique visitors, total page views, sessions, top country, top device, top browser — per selected date range
- **Daily trend:** unique visitors per day (double-`$group` aggregation: first by `{date + ip_hash}`, then by date — accurate deduplication)
- **Breakdowns:** by country, device, browser, OS — percentages computed against full result set, not just top-10
- **World heatmap:** Leaflet.js rendered entirely from local GeoJSON (`/assets/world.geojson`, 252KB Natural Earth 110m). Pulsing dot markers, size and opacity scaled by visit count, dual staggered rings animate outward. Zero external tile server. Zero CDN.
- **Raw event table:** paginated, sortable, 100 rows per page

Country filter scoped to current date range — changes as the range changes.

### Unique visitor counter on public pages

Each of the 4 public pages displays a live visitor count, pulled from the same `/api/visit` endpoint on every page load. The count is deduplicated by `ip_hash` — it's unique visitors, not page views. This is visible social proof that requires zero third-party JavaScript.

---

## How This Was Built

One person. No team. No funding. Phone browser as the primary client.

The development environment: Oracle Cloud ARM64 VM running the compiler, Claude Code, Docker, and the CI/CD pipeline. Every commit was written and pushed from a phone browser over 7 months. The constraint shaped the product — if it was too complex to manage from a phone, it was too complex for a self-hosted enterprise tool.

Claude Code was the primary development tool across 525+ sessions. It wrote code, reviewed architecture decisions, ran tests, diagnosed failures, and executed deploys. The 438 commits in March 2026 and 601 commits in May 2026 were both made from that setup.

The build velocity is a direct consequence of building in the AI-native way: natural language problem statement → code → test → deploy. No IDE. No local dev environment. No interruptions. The constraint produced a workflow that may become the default for solo technical founders in the next few years.
