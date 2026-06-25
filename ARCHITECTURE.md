# KaryoSpace: Architecture

## Design Philosophy

KaryoSpace is a single Go binary. One process, one MongoDB instance, one nginx proxy. No microservices, no service mesh, no distributed transaction complexity.

This is intentional. Enterprise self-hosted software has a different constraint set than cloud-native SaaS. It needs to run on a single VM, be upgradeable with a single restart, and be debuggable by one person without distributed tracing infrastructure. A monolith with clear internal package boundaries achieves this without sacrificing correctness.

Internal package boundaries enforce the same separation that microservices achieve through network calls, but without the operational overhead.

---

## Package Structure

```
backend/go/
    main.go                   HTTP server, route registration, startup
    auth.go                   Okta SSO, session management
    handlers.go               Core page and API handlers
    search.go                 Global search across all modules
    contacts.go               Contact form, feedback routing

    email/
        smtp_handler.go       Inbound SMTP: accept, SPF, spam, store
        smtp_server.go        SMTP listener, connection lifecycle
        smtp_session.go       Per-connection SMTP state machine
        imap_handler.go       IMAP command dispatch
        imap_session.go       Per-connection IMAP state machine
        email_store.go        MongoDB read/write for all email data
        email_models.go       Message, Mailbox, Domain structs
        message_parser.go     MIME parsing: multipart, headers, attachments
        spam_scorer.go        Keyword + SPF + heuristic scoring
        dkim_signer.go        RSA-256 DKIM signing, key management
        spf_verifier.go       SPF DNS lookup and evaluation
        outbound_queue.go     Retry worker, exponential backoff, bounce NDR
        domain_store.go       Email domain config, DKIM key storage
        oauth_handlers.go     Google/Microsoft OAuth token management
        sync_service.go       OAuth mailbox sync background worker

    llm/
        provider.go           LLMProvider interface, Message, Config types
        groq.go               Groq API client (llama-3.3-70b-versatile)
        gemini.go             Gemini API client (gemini-2.0-flash)
        ollama.go             Ollama local client (gemma4, qwen2.5:7b)
        registry.go           NewProvider() factory, env-based selection

    rag/
        classifier.go         IsRAGQuery(), IsStatement() intent detection
        retriever.go          MongoDB $text search, multi-collection
        assembler.go          BuildContextPrompt(), 3000-char context cap

    aio/
        engine.go             Trace recording, flag evaluation, scoring
        aggregator.go         Per-org summary: cache rate, avg quality
        models.go             AITrace, AIOSummary structs

    kre/
        distiller.go          Conversation -> knowledge atom extraction
        cache.go              Semantic query cache, cosine similarity
        worker.go             90-second background distillation loop
        models.go             KnowledgeAtom, SemanticCache structs

    pm/                       Project management: projects, tasks, sprints
    incidents/                Incident lifecycle, escalation, post-mortem
    calendar/                 Event management, OAuth sync
    messaging/                Real-time channels, message history
    knowledge/                KB articles, search, tagging
    notes/                    Personal notes, pinning, tag filtering
```

---

## Request Lifecycle

```
HTTPS request
    |
  nginx (:443)
  TLS termination, Let's Encrypt cert
    |
  Go HTTP Server (:8080)
    |
  Auth middleware
  |-- Valid session cookie?  -->  extract userID, orgID, isSuperUser
  |-- No session?            -->  redirect to /login (Okta SAML)
    |
  Route handler
  |-- Page route  -->  template.ExecuteTemplate() with PageData struct
  |-- API route   -->  JSON response
    |
  MongoDB operation
  |-- Every query carries orgID filter (enforced, not optional)
  |-- No cross-org query is possible by construction
    |
  Response
```

**Template loading:** All `frontend/*.html` and `frontend/partials/*.html` are parsed at server startup via `template.ParseFiles()`. They are immutable at runtime. A deploy replaces the binary and all templates atomically before restart.

---

## AI Request Lifecycle

```
POST /api/home/assistant
    {query: "What are the open P1 incidents?"}
    |
  storeUserQuery()           -- persist to query_history
    |
  callAIService()
    |
  IsStatement(query)?        -- "ok", "got it", "noted", "thanks"
    YES --> return "Got it, noted."  (no LLM call, no cost)
    |
  IsRAGQuery(query)?         -- 30+ trigger phrases:
    |                           "what", "show me", "find", "who",
    |                           "when did", "how many", "status of"...
    YES --> SearchEmails()
    |           |
    |       MongoDB $text search
    |       Collections searched in parallel:
    |         email_messages    (subject + body index)
    |         kb_articles       (title + content index)
    |         pm_tasks          (title + description index)
    |         incidents         (title + description index)
    |         rag_refined_knowledge  (topic + facts index)
    |           |
    |       Score by $meta textScore, limit 5 per collection
    |       Filter: exclude Trash, Spam
    |           |
    |       BuildContextPrompt()
    |       Format: numbered list, source label, 3000 char hard cap
    |           |
    |       LLM call with system prompt + context + conversation history
    |
    NO  --> LLM call with conversation history only
    |
  Append to conversationHistory[userID]  (rolling 20-message window)
    |
  StoreAIOTrace()            -- quality score, flags, latency, token cost
    |
  Return response
```

---

## Concurrency Model

The HTTP server uses Go's standard `net/http` which spawns a goroutine per connection. Background workers run as long-lived goroutines launched at startup with cancellable contexts.

```go
// Startup (main.go)
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

go outboundQueue.StartWorker(ctx)       // email retry loop
go kreWorker.StartDistillation(ctx)     // KRE 90-second loop
go oauthSyncService.StartSync(ctx)      // OAuth mailbox sync

// Graceful shutdown on SIGTERM
signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
<-quit
cancel()  // propagates to all background goroutines
```

**Shared state:**
- `conversationHistory map[string][]llm.Message` protected by `sync.RWMutex`
- MongoDB driver handles its own connection pool (default 100 connections)
- No other shared mutable state between goroutines

---

## Email State Machine

Each SMTP connection runs through a state machine in `SMTPSession`:

```
INIT
  |
  +-- EHLO / HELO  -->  GREETED
                              |
                         STARTTLS (if offered)  -->  TLS_ACTIVE
                              |
                         AUTH (submission port only)  -->  AUTHENTICATED
                              |
                         MAIL FROM  -->  MAIL
                              |
                         RCPT TO (one or more)  -->  RCPT
                              |
                         DATA  -->  DATA_COLLECTION
                              |
                         End of data (CRLF.CRLF)
                              |
                         +----+----+
                         |         |
                    SPF check   Spam score
                         |         |
                    storeMessage() + deliver to mailbox
                         |
                    250 OK  -->  GREETED (ready for next message)
                         |
                    QUIT  -->  CLOSED
```

The IMAP server runs a parallel state machine per connection covering NOT_AUTHENTICATED, AUTHENTICATED, SELECTED, and LOGOUT states per RFC 3501.

---

## Data Model

All collections are org-scoped. `org_id` is always a required field and always included in queries.

```
email_messages
    _id, org_id, message_id, from, to, cc, bcc
    subject, body_text, body_html, raw_mime
    folder, mailbox_id, account_id
    spam_score, flags[], received_at

email_mailboxes
    _id, org_id, email, display_name, user_id
    folders[], quota_mb

aio_traces
    _id, org_id, user_id, query_type
    quality_score, flags[], similarity_score
    source_count, latency_ms, token_cost
    model, provider, retrieved_at

rag_refined_knowledge
    _id, org_id, topic, facts[]
    status (stable|volatile), confidence
    source_query_ids[], created_at, expires_at

rag_semantic_cache
    _id, org_id, query_hash, embedding_vector
    cached_response, hit_count, last_hit_at

pm_projects
    _id, org_id, name, key, columns[], created_by

pm_tasks
    _id, org_id, project_id, title, description
    status, priority, assignee_id, sprint_id
    created_at, updated_at

incidents
    _id, org_id, title, description, severity
    status, assignee_id, timeline[]
    created_at, resolved_at

channels
    _id, org_id, name, description, members[]
    created_by, created_at

channel_messages
    _id, org_id, channel_id, sender_id
    content, thread_id, created_at

calendar_events
    _id, org_id, user_id, title, description
    start_at, end_at, attendees[], recurrence
    external_id, source (google|outlook|internal)

kb_articles
    _id, org_id, title, content, tags[]
    author_id, status, created_at, updated_at

notes
    _id, org_id, user_id, title, content
    tags[], pinned, created_at, updated_at
```

**MongoDB indexes:**
- Text indexes on email subject+body, kb title+content, task title+description (for RAG $text search)
- Compound indexes: `(org_id, folder, received_at)` for mailbox queries
- TTL index on `rag_refined_knowledge.expires_at` for automatic atom expiry
- Compound index on `(org_id, user_id, pinned, updated_at)` for notes

---

## Multi-Tenancy

Org isolation is enforced at the query level, not the application level. Every handler extracts `orgID` from the authenticated session and passes it to every store function. Store functions include `orgID` in every query filter.

There is no admin override that bypasses org isolation. There is no query path that does not carry an `orgID` filter. Cross-org data leakage is not possible by construction.

```go
// Example from email_store.go
func (s *Store) ListMessages(ctx context.Context, orgID, folder string) ([]Message, error) {
    filter := bson.D{
        {Key: "org_id", Value: orgID},   // always required
        {Key: "folder", Value: folder},
    }
    // ...
}
```

---

## Deployment Pipeline

```
git push origin main
    |
GitHub Actions (self-hosted runner on Oracle Cloud)
    |
  +-- go vet ./...
  +-- go test ./... (coverage report)
  +-- Playwright E2E (292 tests)
    |
  GOOS=linux GOARCH=arm64 go build -o workspace .
    |
  docker cp workspace workspace-prod:/app/workspace
  docker cp frontend/  workspace-prod:/app/frontend/
  docker cp frontend/partials/ workspace-prod:/app/frontend/partials/
    |
  docker restart workspace-prod
    |
  Health check: GET /health until 200 OK
    |
  Deploy complete
```

Binary and all frontend templates are deployed atomically. A binary-only deploy is explicitly prevented: templates are parsed from disk at startup, so a stale template directory causes silent drift between what the code expects and what gets rendered. The deploy script enforces this.

---

## Testing Strategy

**Unit and integration tests** (`go test ./...`)
- Every package has a `_test.go` file
- MongoDB-dependent tests use a shared `sync.Once` availability check and skip cleanly when MongoDB is not present
- 80%+ coverage enforced as a CI gate

**E2E tests** (Playwright)
- 292 tests across chromium and mobile-chrome (Pixel 5, 393x851px)
- Tests run against `https://dev.karyospace.com`, not a local mock
- Mobile viewport tests use `element.evaluate(el => el.click())` for elements inside CSS `display:none` sidebars, bypassing Playwright's visibility checks while still firing real DOM events
- Result: 291 pass, 1 skip, 0 fail

**No mocking policy:** Tests that require MongoDB hit a real MongoDB instance. Tests that require a live server hit the real dev server. Mocked tests were removed after discovering they masked a real production bug during email integration.
