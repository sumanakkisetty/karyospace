# KaryoSpace

**Self-hosted, AI-native enterprise workspace.**

KaryoSpace replaces Slack, Gmail, Jira, Notion, and Google Calendar with a single vertically-integrated platform built entirely in Go. Every component is owned: the email server, the AI layer, the data pipeline, the infrastructure. No vendor lock-in. No SaaS dependency. Company data stays on company infrastructure.

**Live:** [karyospace.com](https://karyospace.com)

---

## Build Stats

| Metric | Value |
|--------|-------|
| Commits | 1,614 |
| Development sessions | 525+ |
| AI pair programming hours | 300+ |
| Go test coverage | 80%+ across all packages |
| E2E tests | 292 Playwright tests across all surfaces |
| Primary build tool | Claude Code (Anthropic) |
| Time to market | 7 months, solo, off-hours |

---

## What Was Built

### Six Enterprise Modules

| Module | Replaces |
|--------|----------|
| Email (SMTP/IMAP) | Gmail, Outlook |
| Messaging | Slack, Teams |
| Project Management | Jira, Linear |
| Incident Tracking | PagerDuty, OpsGenie |
| Calendar | Google Calendar, Outlook Calendar |
| Knowledge Base | Notion, Confluence |

### AI Infrastructure

**RAG Pipeline**
Query classifier, MongoDB text search retriever, multi-collection context assembly, multi-LLM provider abstraction (Groq, Gemini, Ollama, Anthropic). Swap providers via environment variable with zero code changes.

**AIO — AI Observability Engine**
Per-query trace instrumentation with flag-based quality scoring. Tracks retrieval emptiness, similarity scores, latency, token cost, and cache effectiveness across every LLM interaction. Org-isolated aggregation. Built because understanding why AI responses fail is the foundation of fixing them.

**KRE — Knowledge Refinement Engine**
Autonomous background system that distills conversation history into persistent knowledge atoms using a local Gemma4 LLM. Semantic query cache at cosine similarity >= 0.88. 30-day TTL atoms with topic volatility tracking. The system gets smarter the more it is used, no human curation required.

### Email Infrastructure

Full SMTP and IMAP server built from scratch in Go.

- DKIM signing on all outbound mail
- SPF verification on inbound mail
- STARTTLS on all connections
- Exponential backoff outbound queue (5m, 30m, 2h, 6h, 24h) with bounce NDR after 6 failures
- Full MIME parsing: HTML, CC, BCC, attachments, raw MIME
- 5-folder IMAP: INBOX, Sent, Drafts, Trash, Spam
- Spam scoring: keyword, SPF, bulk headers, all-caps detection
- Brevo relay for outbound delivery
- End-to-end verified: samscorp.xyz to Gmail, both directions

### Authentication and Security

- Okta SSO integration
- OAuth 2.0 flows: Google Calendar, Microsoft Outlook
- AES-256-GCM encryption for all settings storage (unique nonce per call)
- Multi-tenant org-level data isolation enforced at every layer
- DMARC, DKIM, SPF fully configured

### Infrastructure

- Go backend with MongoDB driver, net/http, goroutines, channels
- Docker deployments on Oracle Cloud (ARM64)
- nginx reverse proxy with Let's Encrypt TLS
- GitHub Actions CI/CD pipeline with self-hosted runner
- Entire dev environment runs on a free Oracle Cloud instance, browser-only access from a phone

---

## Development Setup

This project was built entirely during off-hours while employed full-time, using a setup that proved cloud-native development is viable without a local machine:

- Compiler, Claude Code, Docker, and deployment pipeline all run on a free Oracle Cloud VM
- Phone browser is the only client
- No desk, no laptop, no fixed schedule required

**Commit cadence after moving to this setup:**

| Period | Commits |
|--------|---------|
| First 4 months (traditional setup) | 53 |
| March 2026 | 438 |
| April 2026 | 428 |
| May 2026 | 601 |
| Total | 1,614 |

---

## Tech Stack

**Backend:** Go, MongoDB, net/smtp, net/http, crypto/aes, goroutines

**AI:** Groq (llama-3.3-70b), Gemini, Ollama (Gemma4), Anthropic Claude, all swappable via env var

**Infrastructure:** Docker, Oracle Cloud, nginx, Let's Encrypt, GitHub Actions

**Testing:** Go testing package (80%+ coverage), Playwright E2E (292 tests, chromium + mobile-chrome)

**Email:** SMTP, IMAP, DKIM, SPF, DMARC, STARTTLS, MIME, Brevo relay

**Auth:** Okta SSO, OAuth 2.0, AES-256-GCM, multi-tenant isolation

---

## Architecture

See [ARCHITECTURE.md](ARCHITECTURE.md) for a detailed breakdown of the system design, data flow, and key engineering decisions.

---

## License

KaryoSpace is licensed under the [Elastic License 2.0](LICENSE).

Source is available for review. You may not provide KaryoSpace as a hosted or managed service to third parties.

---

## Author

**Suman Akkisetty**
sumanakkisetty@gmail.com
[linkedin.com/in/sumanakkisetty](https://linkedin.com/in/sumanakkisetty)
Kitchener, Ontario, Canada
