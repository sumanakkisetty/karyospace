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

Open core — not fully open source.

| Tier | Hosting | Price | Features |
|------|---------|-------|----------|
| Community | Self-hosted | Free | All core modules. No SLA. |
| Business | Cloud SaaS | $8–12/user/month | All modules + integrations. SSO. Support SLA. |
| Enterprise | On-prem or dedicated | Custom | ServiceNow, custom integrations, compliance docs, dedicated support. |

Premium-only (closed source): SSO/SAML, ServiceNow integration, advanced audit export, SLA support.

**Currently in beta: free and open to all.** See [karyospace.com/info#pricing](https://karyospace.com/info#pricing).

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

## How This Was Built

One person. No team. No funding. Phone browser as the primary client.

The development environment: Oracle Cloud ARM64 VM running the compiler, Claude Code, Docker, and the CI/CD pipeline. Every commit was written and pushed from a phone browser over 7 months. The constraint shaped the product — if it was too complex to manage from a phone, it was too complex for a self-hosted enterprise tool.

Claude Code was the primary development tool across 525+ sessions. It wrote code, reviewed architecture decisions, ran tests, diagnosed failures, and executed deploys. The 438 commits in March 2026 and 601 commits in May 2026 were both made from that setup.

The build velocity is a direct consequence of building in the AI-native way: natural language problem statement → code → test → deploy. No IDE. No local dev environment. No interruptions. The constraint produced a workflow that may become the default for solo technical founders in the next few years.
