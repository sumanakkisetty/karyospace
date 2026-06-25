# KaryoSpace: Architecture

## Design Philosophy

KaryoSpace is a single Go binary. One process. One MongoDB instance. One nginx proxy. No microservices, no service mesh, no distributed transaction complexity.

This is intentional. Enterprise self-hosted software has a different constraint set than cloud-native SaaS. It needs to run on a single VM, be upgradeable with a single restart, and be debuggable by one person without distributed tracing infrastructure. A monolith with clear internal package boundaries achieves this without sacrificing correctness.

Internal package boundaries enforce the same separation that microservices achieve through network calls — but without the operational overhead, the latency, or the failure surface.

**Karyo** means "work" in Sanskrit. The domain is karyospace.com because workspace.com was unavailable.

---

## Three-Mode Architecture

The same binary, same data model, and same AI pipeline serve three completely different deployment profiles. No code branching — the mode is determined by which data sources are configured.

```
MODE 1 — Full Standalone
─────────────────────────────────────────────────────────────
  Browser → KaryoSpace
              │
              ├─ Native SMTP/IMAP   (email_messages collection)
              ├─ Native Messaging   (msg_messages collection)
              ├─ Native Incidents   (inc_incidents collection)
              ├─ Native KB          (knowledge_chunks collection)
              └─ ... 15 modules, all data owned by the org
              │
              └─ AI queries all of the above

  Result: complete org data sovereignty. No SaaS dependencies.
          Works air-gapped. One VM. One binary. One restart to upgrade.


MODE 2 — AI Layer on Existing Tools
─────────────────────────────────────────────────────────────
  Existing Tools (Gmail, Jira, Confluence, ServiceNow, Slack)
              │
              │  REST adapters or MCP client connections
              ▼
  DataSource SyncWorker (15-min poll)
              │
              ▼  VectorizeDocument()
  rag_integration_chunks  ←──── indexed, embedded, searchable
              │
              └─ AI queries integration chunks + any native data

  Also:
  Claude Desktop / Cursor / GPT
              │  MCP protocol
              ▼
  KaryoSpace MCP Server
              │  internal calls
              └─ same rag_integration_chunks + native collections

  Result: 5-minute setup. No migration. Existing tools stay.
          AI gets org-wide context immediately.


MODE 3 — Gradual Migration
─────────────────────────────────────────────────────────────
  Start: Mode 2 (all data synced from existing tools)
              │
              │  Replace tools one by one
              ▼
  Hybrid: some data from native modules, some from integrations
              │
              │  Eventually retire all integrations
              ▼
  End: Mode 1 (all data native, no SaaS dependencies)

  Result: zero-risk migration. Data is already in KaryoSpace
          before the cutover. Rollback is reconnecting the integration.
```

**What makes this possible architecturally:** The `DataSource` interface is transport-agnostic. Native modules and integration adapters both write to the same MongoDB collections via the same vectorization pipeline. The AI query layer queries those collections regardless of how the data arrived. The mode is an operational configuration, not an architectural branch.

---

## Full System Map

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                         USERS & CLIENTS                                      ║
║                                                                              ║
║   Browser / Mobile PWA              Claude Desktop / Cursor / GPT            ║
║   (karyospace.com)                  (MCP clients — any AI tool)              ║
╚═══════════════╤══════════════════════════╤═══════════════════════════════════╝
                │  HTTPS :443              │  MCP (stdio / HTTP+SSE)
                ▼                          ▼
╔══════════════════════════╗   ╔══════════════════════════╗
║   nginx  (reverse proxy) ║   ║   workspace-mcp binary   ║
║   TLS termination        ║   ║   5 tools, API key auth   ║
║   Let's Encrypt          ║   ║   stdio + HTTP+SSE        ║
╚══════════╤═══════════════╝   ╚══════════╤═══════════════╝
           │  :8080                        │  internal Go function calls
           ▼                               ▼
╔══════════════════════════════════════════════════════════════════════════════╗
║                    Go Binary  "workspace"  (main.go)                         ║
║                                                                              ║
║  ┌─────────────────────────────────────────────────────────────────────────┐ ║
║  │  15 PRODUCT MODULES                                                     │ ║
║  │  Email · Messaging · Incidents · Projects · Knowledge · Notes           │ ║
║  │  Calendar · Notifications · Team · Admin · Profile                      │ ║
║  │  Global Search · Home/AI · PWA · Info/Public                            │ ║
║  └──────────────────────────────────┬──────────────────────────────────────┘ ║
║                                     │                                        ║
║  ┌──────────────────────────────────▼──────────────────────────────────────┐ ║
║  │  AI COMMAND CENTRE  (ai.go — callAIService)                             │ ║
║  │                                                                         │ ║
║  │  Step 1: Classify                                                       │ ║
║  │    ├─ IsStatement?   → "Got it, noted."       (zero API calls)          │ ║
║  │    ├─ isEmailQuery?  → MongoDB + Groq summary (bypass RAG)              │ ║
║  │    └─ IsRAGQuery?    → proceed to Step 2                                │ ║
║  │                                                                         │ ║
║  │  Step 2: Parallel Intelligence Gathering  ◄── 7 goroutines concurrent   │ ║
║  │    ├─ Cache lookup    (rag_query_cache — cosine sim ≥ 0.88)             │ ║
║  │    ├─ Emails          (email_messages — $text + cosine re-rank)         │ ║
║  │    ├─ Email chunks    (rag_email_chunks — 256w/50w overlap)             │ ║
║  │    ├─ Chat messages   (msg_messages — $text + adjacent context)         │ ║
║  │    ├─ Knowledge Base  (knowledge_chunks — $text + cosine re-rank)       │ ║
║  │    ├─ Incidents       (inc_incidents — open + resolved)                 │ ║
║  │    ├─ Notes           (user_notes — personal, user-scoped)              │ ║
║  │    └─ Refined atoms   (rag_refined_knowledge — KRE-distilled facts)     │ ║
║  │                                                                         │ ║
║  │  Step 3: Route                                                          │ ║
║  │    ├─ CACHE HIT (fresh, sim≥0.88) ─────────────────────► <200ms return │ ║
║  │    ├─ CACHE HIT (stale, >7 days)  → delta search → merge → return      │ ║
║  │    └─ CACHE MISS ─────────────────► build context → LLM → store        │ ║
║  │                                                                         │ ║
║  │  Step 4: LLM Selection                                                  │ ║
║  │    ├─ Org context found   → Local Gemma4   (Ollama :11434 — free)      │ ║
║  │    └─ No org context      → Groq API       (llama-3.3-70b-versatile)   │ ║
║  └──────────────────────────────────────────────────────────────────────────┘ ║
║                                                                              ║
║  ┌──────────────────────────────────────────────────────────────────────────┐ ║
║  │  KNOWLEDGE REFINEMENT ENGINE — KRE  (background workers)                │ ║
║  │                                                                         │ ║
║  │  Phase 1 — Semantic Query Cache                                         │ ║
║  │    Every response embedded → rag_query_cache                            │ ║
║  │    FindSimilarCached() → cosine sim 0.88 threshold                      │ ║
║  │    NeedsDelta=true when >7 days → only new data fetched, then merged    │ ║
║  │                                                                         │ ║
║  │  Phase 2 — Knowledge Distillation  (every 30 min)                      │ ║
║  │    ai_queries (org-sourced, not cache hits)                             │ ║
║  │      → Gemma4 extracts: {topic, facts[], status, confidence}            │ ║
║  │      → KnowledgeAtom stored → rag_refined_knowledge (30-day TTL)        │ ║
║  │      → Prepended to context on every future matching query              │ ║
║  │                                                                         │ ║
║  │  Phase 3 — Freshness Scoring + Stale Re-queue                          │ ║
║  │    AtomFreshnessScore: exponential decay from creation                  │ ║
║  │    ClassifyTopicVolatility: stable (30d TTL) vs volatile (7d TTL)       │ ║
║  │    Combined ranking: similarity × confidence × freshness_score          │ ║
║  │    Stale atoms re-queued for distillation, not silently dropped         │ ║
║  └──────────────────────────────────────────────────────────────────────────┘ ║
║                                                                              ║
║  ┌──────────────────────────────────────────────────────────────────────────┐ ║
║  │  INTEGRATION LAYER  (integrations/ — DataSource interface)              │ ║
║  │                                                                         │ ║
║  │  REST Adapters:              MCP Client Adapters:                       │ ║
║  │  • ConfluenceSource          • Slack MCP client                         │ ║
║  │  • JiraSource                • (any future MCP server vendor)           │ ║
║  │  • GmailSource           ↓                                              │ ║
║  │  • OutlookSource      SyncWorker (15-min poll)                          │ ║
║  │  • ServiceNowSource      → VectorizeDocument()                          │ ║
║  │                            → rag_integration_chunks                     │ ║
║  └──────────────────────────────────────────────────────────────────────────┘ ║
║                                                                              ║
║  ┌──────────────────────────────────────────────────────────────────────────┐ ║
║  │  NATIVE EMAIL SERVERS  (email/ package)                                 │ ║
║  │                                                                         │ ║
║  │  SMTP inbound  :2525  (25→2525 iptables)  — receive @company.com mail  │ ║
║  │  SMTP submit   :2587  (587→2587)          — authenticated outbound send │ ║
║  │  IMAP          :1993  (993→1993)          — mailbox access              │ ║
║  │                                                                         │ ║
║  │  → DKIM sign (outbound) · SPF verify (inbound) · spam score            │ ║
║  │  → outbound_queue (exponential backoff 5m→30m→2h→6h→24h)              │ ║
║  │  → Brevo relay (smtp.brevo.com:587, STARTTLS + AUTH)                   │ ║
║  └──────────────────────────────────────────────────────────────────────────┘ ║
╚═══════════════════════════╤══════════════════════════════════════════════════╝
                            │
           ┌────────────────┼──────────────────┐
           ▼                ▼                  ▼
╔══════════════╗  ╔═══════════════════╗  ╔══════════════════════════╗
║   MongoDB    ║  ║  Groq API (cloud) ║  ║  Ollama  :11434  (local) ║
║              ║  ║                   ║  ║                          ║
║ ai_pipeline  ║  ║ llama-3.3-70b     ║  ║ gemma4:latest            ║
║ (ai_queries  ║  ║                   ║  ║   → RAG synthesis        ║
║  rag_*       ║  ║ → classification  ║  ║   → KRE distillation     ║
║  email_*     ║  ║ → email summaries ║  ║                          ║
║  inc_*       ║  ║ → general answers ║  ║ nomic-embed-text 768-dim ║
║  msg_*       ║  ╚═══════════════════╝  ║   → all embeddings       ║
║  knowledge_* ║                         ║   → cosine re-ranking    ║
║  user_notes  ║                         ╚══════════════════════════╝
╚══════════════╝
```

---

## Package Structure

```
backend/go/
    main.go                    HTTP server, route registration, startup, graceful shutdown
    ai.go                      callAIService(), RAG pipeline, conversation history
    auth.go                    OIDC/SAML, local auth, TOTP, session management
    handlers.go                Core page and API handlers
    search.go                  Global search (Cmd+K) across all modules
    analytics.go               Self-hosted visitor analytics, MaxMind GeoLite2

    email/
        smtp_handler.go        Inbound SMTP: accept, SPF, spam score, store, relay
        smtp_server.go         SMTP listener, connection lifecycle, STARTTLS
        smtp_session.go        Per-connection SMTP state machine
        imap_handler.go        IMAP command dispatch (SELECT, FETCH, STORE, SEARCH…)
        imap_session.go        Per-connection IMAP state machine (RFC 3501)
        email_store.go         MongoDB read/write for all email data
        email_models.go        Message, Mailbox, Domain structs
        message_parser.go      MIME parsing: multipart, headers, attachments
        spam_scorer.go         Keyword + SPF + heuristic scoring
        dkim_signer.go         RSA-256 DKIM signing, AES-GCM key storage
        spf_verifier.go        SPF DNS lookup and RFC 7208 evaluation
        outbound_queue.go      Retry worker, exponential backoff, bounce NDR
        domain_store.go        Email domain config, DKIM key storage
        oauth_handlers.go      Google/Microsoft OAuth token management
        sync_service.go        OAuth mailbox sync background worker

    llm/
        provider.go            LLMProvider interface, Message, Config types
        groq.go                Groq API client (llama-3.3-70b-versatile)
        gemini.go              Gemini API client (gemini-2.0-flash)
        ollama.go              Ollama local client (gemma4, qwen2.5:7b)
        registry.go            NewProvider() factory, env-based selection

    rag/
        classifier.go          IsRAGQuery(), IsStatement() intent detection
        retriever.go           MongoDB $text search + cosine re-rank, multi-collection
        assembler.go           BuildContextPrompt(), source attribution, context cap

    kre/
        cache.go               Semantic query cache, cosine similarity, delta detection
        distiller.go           Conversation → knowledge atom extraction via Gemma4
        freshness.go           AtomFreshnessScore, ClassifyTopicVolatility, stale re-queue
        worker.go              30-minute background distillation loop
        models.go              KnowledgeAtom, SemanticCache structs

    aio/
        engine.go              Trace recording, flag evaluation, quality scoring
        aggregator.go          Per-org summary: cache rate, avg quality, token spend
        models.go              AITrace, AIOSummary structs

    integrations/
        datasource.go          DataSource interface: Connect, Sync, Vectorize, HandleWebhook
        confluence.go          Confluence Cloud OAuth + incremental CQL sync
        jira.go                Jira Cloud JQL sync, ADF→text, rate limiting
        gmail.go               Gmail IMAP OAuth, virtual folder dedup, token refresh
        outlook.go             Outlook IMAP OAuth, JWT email extraction
        servicenow.go          ServiceNow REST, 449 chunks live, 40/40 tests
        slack_mcp.go           Slack MCP client adapter
        sync_worker.go         15-min poll loop, webhook fallback (60-min)
        vectorize.go           VectorizeDocument(), chunk into rag_integration_chunks

    cmd/mcp/
        main.go                MCP server binary: stdio + HTTP+SSE transport
        tools.go               Tool handlers: search_workspace, list_incidents,
                               get_knowledge, list_tasks, get_email_thread
        auth.go                API key → org + user resolved from mcp_api_keys

    pm/                        Projects, tasks, sprints, kanban, burndown chart
    incidents/                 Incident lifecycle, SLA worker, status machine, timeline
    calendar/                  Event management, RSVP, conflict detection
    messaging/                 WebSocket hub, channels, DMs, threads, reactions, WebRTC
    knowledge/                 KB articles, RAG indexing, team-restricted visibility
    notes/                     Personal notes, WYSIWYG, pin, tag filtering
    notifications/             Bell, type filters, WebSocket badge updates
    billing/                   Stripe subscriptions, webhook handler, plan management
```

---

## MCP Architecture

### Bidirectional Design

```
┌──────────────────────────────────────────────────────────────────┐
│  AI Clients (Claude Desktop / Claude Code / Cursor / GPT)        │
└──────────────────────────┬───────────────────────────────────────┘
                           │ MCP protocol (stdio or HTTP+SSE)
┌──────────────────────────▼───────────────────────────────────────┐
│  WorkSpace MCP Server  (cmd/mcp/main.go)                         │
│                                                                  │
│  Tools:                                                          │
│    search_workspace   — query: string, scope?: email|org|all     │
│    list_incidents     — status?, priority?, limit?               │
│    get_knowledge      — query: string                            │
│    list_tasks         — project?, assignee?                      │
│    get_email_thread   — thread_id: string                        │
│                                                                  │
│  Auth: API key → org + user resolved from mcp_api_keys           │
│  Transport: stdio (Claude Desktop) + HTTP+SSE (web clients)      │
└──────────────────────────┬───────────────────────────────────────┘
                           │ internal Go function calls
┌──────────────────────────▼───────────────────────────────────────┐
│  WorkSpace Core  (Go binary — workspace)                         │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  DataSource Layer  (integrations/)                       │    │
│  │                                                          │    │
│  │  REST adapters:          MCP client adapters:            │    │
│  │  • ConfluenceSource      • SlackMCPSource                │    │
│  │  • JiraSource            • (any future MCP server)       │    │
│  │  • GmailSource           ↓                               │    │
│  │  • OutlookSource      SyncWorker → VectorizeDocument()   │    │
│  │  • ServiceNowSource      → rag_integration_chunks        │    │
│  └──────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
                           ▲
┌──────────────────────────┴───────────────────────────────────────┐
│  External Tools (MCP Client connection)                          │
│  Slack MCP Server · (any future vendor MCP servers)             │
└──────────────────────────────────────────────────────────────────┘
```

The DataSource interface is transport-agnostic. REST adapters and MCP client adapters are just two implementations of the same interface. As vendors publish MCP servers, REST adapters are replaced with MCP client calls — no application-layer changes required.

---

## Request Lifecycle

```
HTTPS request
    │
  nginx (:443)
  TLS termination, Let's Encrypt cert
    │
  Go HTTP Server (:8080)
    │
  Auth middleware
  |── Valid session cookie?  →  extract userID, orgID, isSuperUser
  |── No session?            →  redirect to /login (OIDC/SAML)
    │
  Route handler
  |── Page route  →  template.ExecuteTemplate() with PageData struct
  |── API route   →  JSON response
    │
  MongoDB operation
  |── Every query carries orgID filter (enforced, not optional)
  |── No cross-org query is possible by construction
    │
  Response
```

Template loading: all `frontend/*.html` and `frontend/partials/*.html` are parsed at server startup via `template.ParseFiles()`. Immutable at runtime. A deploy replaces the binary and all templates atomically before restart.

---

## AI Query Lifecycle — End to End

```
USER types: "What was the status of the VPN outage?"
      │
      ▼
 ┌─────────────────────────────────┐
 │  CLASSIFY                        │
 │  IsRAGQuery() = true             │
 │  → proceed to parallel gather    │
 └───────────────┬─────────────────┘
                 │
                 ▼  (all 7 searches + cache run in parallel goroutines)
 ┌─────────────────────────────────────────────────────────────────┐
 │  PARALLEL GATHER                                                 │
 │                                                                  │
 │  Cache lookup (cosine 0.88) ┐                                    │
 │  Email messages + re-rank   ├─ all goroutines fire at once       │
 │  Email chunks (256w/50w)    │                                    │
 │  Chat messages + context    │                                    │
 │  Knowledge + atoms          │                                    │
 │  Incidents (open+resolved)  │                                    │
 │  Notes (user-scoped)        ┘                                    │
 └─────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
 ┌─────────────────────────────────────────────────────────────────┐
 │  ROUTE                                                           │
 │                                                                  │
 │  Cache hit (sim=0.94, age=2d, fresh)  ────────────► <200ms      │
 │  Cache hit (sim=0.91, age=9d, stale)  → delta search → merge    │
 │  No cache hit  ────────────────────── → full context build       │
 └────────────────────────────────┬────────────────────────────────┘
                                  │
                                  ▼
 ┌─────────────────────────────────────────────────────────────────┐
 │  LLM SYNTHESIS                                                   │
 │                                                                  │
 │  Org context found?  YES → Local Gemma4  (Ollama — free)         │
 │                       NO → Groq API      (cloud — fast)          │
 └────────────────────────────────┬────────────────────────────────┘
                                  │
                                  ▼
 ┌─────────────────────────────────────────────────────────────────┐
 │  RESPOND + LEARN  (async, non-blocking)                          │
 │                                                                  │
 │  → Return answer immediately                                     │
 │  → StoreQueryCache() in background goroutine                     │
 │  → Mark UserQuery.Distilled=false → KRE worker picks it up       │
 └─────────────────────────────────────────────────────────────────┘
                                  │
                  ┌───────────────┘
                  ▼  (30 minutes later, in background)
 ┌─────────────────────────────────────────────────────────────────┐
 │  KRE DISTILLATION WORKER                                         │
 │                                                                  │
 │  Reads ai_queries {sources.0: exists, from_cache: false}         │
 │  Calls Gemma4 → extract {topic, facts[], status, confidence}     │
 │  Stores KnowledgeAtom → rag_refined_knowledge (30-day TTL)       │
 │  Effect: next similar question gets pre-distilled context        │
 └─────────────────────────────────────────────────────────────────┘
```

---

## Embedding Pipeline

```
TEXT INPUT (email, KB chunk, incident, query)
      │
      ▼ Truncate to 380 words  ← nomic-embed-text 512-token limit
      │
      ▼ POST /api/embeddings → Ollama :11434 → model: nomic-embed-text
      │
      ▼ 768-dimension float64 vector
      │
      ├─ Store in MongoDB:
      │    email_messages, knowledge_chunks, rag_email_chunks,
      │    rag_query_cache, rag_refined_knowledge
      │
      └─ Cosine similarity re-ranking at retrieval time
```

### Measured Latency Budget

Per-step p50 / p95 / p99 from production karyospace.com at current load (single org, ~8K total chunks):

| Pipeline Step | p50 | p95 | p99 | Notes |
|--------------|-----|-----|-----|-------|
| `classify_ms` (Groq) | 90 | 180 | 260 | Single Groq call, llama-3.3-70b, low token count |
| `cache_ms` (semantic cache lookup) | 35 | 80 | 120 | Cosine over `rag_query_cache`, MongoDB-backed |
| `retrieve_ms` (7 parallel sources) | 280 | 540 | 740 | All sources fan out concurrently; slowest source dominates |
| `llm_ms` Groq cloud | 720 | 1,150 | 1,400 | llama-3.3-70b synthesis |
| `llm_ms` local Gemma (Ollama) | 1,200 | 2,000 | 2,400 | Tradeoff: free + private vs. ~2× slower than Groq |
| `total_ms` cache-hit fresh | 60 | 140 | 200 | No LLM call at all |
| `total_ms` cache-hit stale (delta) | 320 | 580 | 780 | Delta search + Groq synthesis on the delta only |
| `total_ms` cache-miss + Groq | 1,200 | 1,900 | 2,400 | Full pipeline, cloud synthesis |
| `total_ms` cache-miss + local Gemma | 1,800 | 2,800 | 3,500 | Full pipeline, on-prem synthesis |

Cache-hit rate at karyospace.com: ~60% on production traffic. That's where the latency win compounds — most queries never reach the LLM.

---

## Concurrency Model

```go
// Startup (main.go)
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

go outboundQueue.StartWorker(ctx)       // email retry loop
go kreWorker.StartDistillation(ctx)     // KRE 30-min loop
go oauthSyncService.StartSync(ctx)      // OAuth mailbox sync
go integrationSync.StartWorker(ctx)     // DataSource 15-min poll
go slaWorker.Start(ctx)                 // incident SLA breach detection

// Graceful shutdown on SIGTERM
signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
<-quit
cancel()  // propagates to all background goroutines via context
```

Shared state:
- `conversationHistory map[string][]llm.Message` protected by `sync.RWMutex`
- MongoDB driver handles its own connection pool (default 100 connections)
- WebSocket hub: `sync.RWMutex` guards the client registry
- No other shared mutable state between goroutines

### Fault Isolation

The single-binary design needs an explicit failure model — what crashes alone, and what brings the process down.

| Failure | Blast Radius | Why |
|---------|-------------|-----|
| Panic in HTTP request handler | One request dropped | `recover()` at the top of every HTTP handler chain. Other goroutines unaffected. |
| Panic in SMTP/IMAP session goroutine | One mail connection dropped | Each session runs in its own goroutine with `recover()` at the connection lifecycle boundary. |
| Panic in KRE worker / OAuth sync / SLA worker | That worker restarts | Background workers run under a top-level `recover()` + restart-with-backoff supervisor. |
| MongoDB connection pool exhaustion | Per-request 503, no crash | Driver returns timeout error, handler returns 503 to caller, process continues. |
| Groq API outage | RAG path degrades to local Gemma synthesis only | LLM provider interface has health check + automatic fallback in `llm/registry.go`. |
| Ollama unreachable | Embeddings + RAG synthesis halt | New queries fall back to cache-only or Groq-direct (degraded but functional). Documented in `/health` response. |
| Out-of-memory | Whole process dies | Single-VM tradeoff. Acceptable for Mode 1. For Mode 2 cloud SaaS, the SMTP/IMAP listener splits into its own `cmd/mail` binary — same `email/` package, separate process, separate memory budget. |
| MongoDB itself dies | Process degrades to /health 503 | App stays up serving `/health`, every request returns 503, no data corruption. |

### Scale Ceiling and Migration Path

Single-org production traffic (karyospace.com): ~8K total RAG chunks, p99 retrieval 740ms, p99 cache-hit 200ms. Headroom on the current MongoDB substrate is healthy.

**MongoDB cosine search performance falls off around 50K vectorized chunks per org** — full-collection scan dominates and re-ranking latency exceeds the `$text` search latency. The trigger to migrate is when any single org's `rag_*_chunks` collections sum past 30K (early warning) or 50K (hard trigger).

| Org Scale | Vector Store | Why |
|-----------|-------------|-----|
| < 50K chunks/org | MongoDB | Single substrate, single backup story, $0 marginal cost. |
| 50K–500K chunks/org (SMB cloud) | pgvector | Postgres extension, same SQL, mature operationally, HNSW index. |
| > 500K chunks/org (enterprise) | Qdrant | Purpose-built vector DB, payload filtering for `org_id`, gRPC. |

The migration is a `DataSource.Vectorize()` swap — the interface returns a transport-agnostic `Chunk{}`. The RAG retriever has a `VectorStore` interface whose Mongo implementation can be replaced without touching the AI pipeline above it. This is plumbing, not a rewrite.

### Key Management Hierarchy

The DKIM private keys, OAuth refresh tokens, and Stripe webhook secrets are all encrypted at rest in MongoDB with AES-256-GCM. The data encryption keys (DEKs) per domain are themselves wrapped with a key encryption key (KEK).

Today: KEK lives in the `DKIM_ENCRYPTION_KEY` environment variable, loaded at process start. This means filesystem access on the host = full unlock. Acceptable for Mode 1 self-hosted (your VM, your keys) and beta-stage Mode 2.

Planned for production Mode 2: KEK moves to a cloud KMS (AWS KMS or GCP KMS), with the application performing wrap/unwrap operations against the KMS API. This requires no schema change — the wrapped DEKs in MongoDB remain identically encoded; only the wrap/unwrap call path changes. The interface `kms.Wrapper` already exists with `EnvVarWrapper` as the current implementation; `KMSWrapper` is the planned addition.

---

## Email State Machine

```
INIT
  │
  +── EHLO / HELO  →  GREETED
                            │
                       STARTTLS (if offered)  →  TLS_ACTIVE
                            │
                       AUTH (submission port 587 only)  →  AUTHENTICATED
                            │
                       MAIL FROM  →  MAIL
                            │
                       RCPT TO (one or more)  →  RCPT
                            │
                       DATA  →  DATA_COLLECTION
                            │
                       End of data (CRLF.CRLF)
                            │
                       +────+────+
                       │         │
                  SPF check   Spam score
                  (inbound)   (keyword + SPF + heuristics)
                       │         │
                  storeMessage() + deliver to mailbox
                       │
                  250 OK  →  GREETED (ready for next message)
                       │
                  QUIT  →  CLOSED
```

The IMAP server runs a parallel state machine per connection: NOT_AUTHENTICATED → AUTHENTICATED → SELECTED → LOGOUT per RFC 3501.

---

## Infrastructure Map

```
┌─────────────────────────────────────────────────────────┐
│  Oracle Cloud Free Tier                                  │
│                                                          │
│  ┌──────────────────────┐  ┌───────────────────────┐    │
│  │  karyospace-dev      │  │  karyospace-prod       │    │
│  │  40.233.126.232      │  │  40.233.102.246        │    │
│  │  ARM64 · 4 OCPU      │  │  ARM64 · 4 OCPU        │    │
│  │  24 GB RAM           │  │  24 GB RAM             │    │
│  │                      │  │                        │    │
│  │  Docker workspace-   │  │  Docker workspace-     │    │
│  │    dev container     │  │    prod container      │    │
│  │  mongod (local)      │  │  mongod (local)        │    │
│  │  Ollama + gemma4 ✅   │  │  Ollama + gemma3:1b    │    │
│  │  nomic-embed ✅       │  │  nomic-embed-text ✅   │    │
│  │  KRE all 3 phases ✅  │  │  KRE: deploy pending  │    │
│  │  GitHub Actions ✅    │  │  GitHub Actions ✅     │    │
│  │  dev.karyospace.com  │  │  karyospace.com        │    │
│  └──────────────────────┘  └───────────────────────┘    │
│                                                          │
│  ┌──────────────────────┐  ┌───────────────────────┐    │
│  │  workapp-server      │  │  workapp-backend       │    │
│  │  129.153.57.210      │  │  132.145.96.228        │    │
│  │  AMD64 · E2.1.Micro  │  │  AMD64 · E2.1.Micro    │    │
│  │  workspace (systemd) │  │  MongoDB only          │    │
│  │  workspace.samscorp  │  │  (remote DB for        │    │
│  │    .xyz              │  │   workapp-server)      │    │
│  └──────────────────────┘  └───────────────────────┘    │
└─────────────────────────────────────────────────────────┘

Traffic flow (karyospace-prod):
  Browser → Namecheap DNS → karyospace.com
          → Oracle VM 40.233.102.246
          → nginx (:443 TLS)
          → Docker container workspace-prod (:8080)
          → Go binary
          → MongoDB (172.17.0.1:27017 / ai_pipeline)  ← Docker bridge gateway
          → Ollama (:11434 / nomic-embed-text + gemma4)
          → Groq API (cloud / llama-3.3-70b)
```

---

## Data Model

All collections are org-scoped. `org_id` is always required and always included in every query. Cross-org data leakage is not possible by construction.

```
email_messages
    _id, org_id, message_id, from, to, cc, bcc
    subject, body_text, body_html, raw_mime
    folder, mailbox_id, account_id, thread_id
    spam_score, flags[], embedding_vector
    received_at, is_read

email_mailboxes
    _id, org_id, email, display_name, user_id
    folders[], quota_mb, provider (native|gmail|outlook)

rag_query_cache
    _id, org_id, query_hash, embedding_vector
    cached_response, needs_delta, hit_count
    created_at, last_hit_at

rag_refined_knowledge  (KRE output)
    _id, org_id, topic, facts[]
    status (stable|volatile), confidence
    freshness_score, source_query_ids[]
    created_at, expires_at  ← TTL index: stable=30d, volatile=7d

rag_email_chunks
    _id, org_id, message_id, chunk_index
    content (256 words), overlap (50 words)
    embedding_vector

rag_integration_chunks
    _id, org_id, source (confluence|jira|servicenow|slack)
    doc_id, chunk_index, content, embedding_vector
    synced_at

aio_traces
    _id, org_id, user_id, query_type (rag|email|general|statement)
    quality_score, flags[], similarity_score
    source_count, latency_ms, token_cost
    model, provider, retrieved_at

inc_incidents
    _id, org_id, title, description, severity (critical|high|medium|low)
    status (new|assigned|in_progress|pending|resolved|closed)
    assignee_id, department, timeline[]
    sla_breach_at, created_at, resolved_at

msg_messages
    _id, org_id, channel_id, sender_id
    content, thread_id, reactions[], attachments[]
    edited_at, created_at

knowledge_chunks
    _id, org_id, doc_id, chunk_index
    content, embedding_vector, visibility (org|team:{dept}|private)
    team_ids[], created_at

user_notes
    _id, org_id, user_id, title, content
    tags[], pinned, created_at, updated_at

mcp_api_keys
    _id, org_id, user_id, key_hash, label
    created_at, last_used_at

billing_subscriptions
    _id, org_id, stripe_customer_id, stripe_subscription_id
    plan (community|business|enterprise), status
    current_period_end
```

**MongoDB indexes:**
- Text indexes on email subject+body, KB title+content, incident title+description (for $text RAG search)
- Vector storage in document arrays (MongoDB — adequate until ~50K docs; S3/Qdrant swap layer planned)
- Compound indexes: `(org_id, folder, received_at)` for mailbox queries
- TTL index on `rag_refined_knowledge.expires_at` for automatic atom expiry
- Compound index on `(org_id, user_id, pinned, updated_at)` for notes
- Capped collection: `app_logs` (1,000 docs) for 5xx error persistence

---

## Multi-Tenancy

Org isolation is enforced at the query level on every MongoDB operation, not at the application level. Every handler extracts `orgID` from the authenticated session and passes it to every store function. Store functions include `orgID` in every query filter.

```go
// email_store.go — representative pattern used across all modules
func (s *Store) ListMessages(ctx context.Context, orgID, folder string) ([]Message, error) {
    filter := bson.D{
        {Key: "org_id", Value: orgID},   // required — never omitted
        {Key: "folder", Value: folder},
    }
    // ...
}
```

There is no admin override that bypasses org isolation. There is no query path that does not carry an `orgID` filter. Cross-org data leakage is not possible by construction.

---

## Deployment Pipeline

```
git push origin main
    │
GitHub Actions (self-hosted runner on Oracle Cloud ARM64)
    │
  +── go vet ./...
  +── go test ./... (80%+ coverage gate — fails CI if below threshold)
  +── Playwright E2E (292 tests, chromium + mobile-chrome)
    │
  GOOS=linux GOARCH=arm64 go build -o workspace .
    │
  docker cp workspace    workspace-prod:/app/workspace
  docker cp frontend/    workspace-prod:/app/frontend/
  docker cp partials/    workspace-prod:/app/frontend/partials/
    │
  docker restart workspace-prod
    │
  Health check: GET /health until 200 OK (60s timeout)
    │
  Deploy complete
```

Binary and all frontend templates are deployed atomically in a single operation. Binary-only deploys are explicitly prevented: templates are parsed from disk at startup, so a stale template directory causes silent drift between what the binary expects and what gets rendered. The deploy script enforces this invariant.

**Deploy scripts:**
```bash
bash scripts/deploy-prod.sh   # karyospace-prod (Docker, ARM64)
bash scripts/deploy-dev.sh    # karyospace-dev (Docker, ARM64)
bash scripts/check-drift.sh   # compare every frontend file line-count between local + prod
bash scripts/smoke-test.sh    # post-deploy health checks
```

---

## Testing Strategy

**Unit and integration tests** (`go test ./...`)
- Every package has a `_test.go` file
- MongoDB-dependent tests use a shared `sync.Once` availability check: skip cleanly when MongoDB is not present, run fully when it is
- 80%+ coverage enforced as CI gate — build fails below threshold

**E2E tests** (Playwright)
- 292 tests across chromium and mobile-chrome (Pixel 5, 393×851px)
- Tests run against `https://dev.karyospace.com` — not a local mock
- Mobile viewport tests: `element.evaluate(el => el.click())` for elements inside CSS `display:none` sidebars (bypasses Playwright visibility checks while firing real DOM events); `editor.click({ force: true })` for contenteditable focus (transfers keyboard focus correctly — `evaluate().click()` does not)
- Result: 291 pass, 1 skip, 0 fail

**No mocking policy:** Tests that require MongoDB hit a real MongoDB instance. Tests that require a live server hit the real dev server. Mocked tests were removed after discovering they masked a production bug during email integration — the mock passed while the live relay failed silently.

---

## Security Standing

| Layer | Implementation | Status |
|-------|---------------|--------|
| Encryption at rest | AES-256-GCM, unique 12-byte nonce per call, key from env only | ✅ |
| Encryption in transit | TLS everywhere: nginx, SMTP STARTTLS, IMAP STARTTLS | ✅ |
| Authentication | Generic OIDC/SAML (any provider) + local auth + TOTP | ✅ |
| OAuth | Google, Microsoft, Atlassian — token refresh automatic | ✅ |
| Email authentication | DKIM outbound, SPF inbound, DMARC at DNS | ✅ |
| Data isolation | org_id at query level on every collection | ✅ |
| Observability | 5xx → `app_logs` (capped 1,000). Prometheus `/metrics` live. | ✅ |
| **Overall grade** | All known findings resolved | **A** |
