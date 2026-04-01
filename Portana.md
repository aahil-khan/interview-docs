# Portana / Portfolio OS - Complete Interview Guide

## 1. What This Project Is

Portfolio OS (Portana backend in this branch) is an AI-powered portfolio backend that turns static resume and project data into a conversational API using RAG (retrieval-augmented generation).

In practical terms, it lets a portfolio owner:
- ingest resume, GitHub README content, Medium posts, and custom Q&A knowledge
- store semantic vectors in Qdrant
- answer user questions via chat endpoints (streaming + non-streaming)
- expose command endpoints for deterministic UI flows (`/projects`, `/stack`, `/experience`, etc.)
- run onboarding and admin workflows for ingestion, diagnostics, and maintenance

This branch is backend-first. There is no frontend app folder in this workspace branch.

## 2. Core Product Value

### Problem solved
Traditional portfolios are static and high-maintenance. Every update requires manual edits, and users cannot naturally ask questions.

### Solution
A configurable backend that ingests content once and then answers context-aware questions with continuously refreshable data.

### Key differentiators
- conversational memory per session
- multi-source ingestion (resume, GitHub, Medium, cheatsheet)
- command APIs for predictable UX + chat APIs for open-ended UX
- webhook queue + DLQ for resilient automation
- deployment-ready via Docker/Compose with health checks and CI workflow

## 3. High-Level Architecture

## Runtime components
- Fastify API server (TypeScript)
- OpenAI for embeddings + chat completions
- Qdrant for vector retrieval
- in-memory session stores for chat/onboarding/admin metadata
- JSON file sources for resume/blogs/cheatsheet

## Major layers
1. API layer (`backend/src/routes`): validates requests, exposes endpoints, orchestrates services.
2. Domain services (`backend/src/services`, `backend/src/chat`, `backend/src/onboarding`, `backend/src/webhooks/services`): parsing, embedding, retrieval, queueing, generation.
3. Data layer:
   - vector DB: Qdrant collection `portfolio_content`
   - structured/local: JSON files + partial SQLite scaffolding
4. Infra/deploy: Dockerfile, docker-compose, GitHub Actions.

## 4. Tech Stack and Why

- Fastify 4: fast Node HTTP server with plugin ecosystem and low overhead.
- TypeScript 5: stronger contracts for routes/services.
- OpenAI SDK:
  - embeddings: `text-embedding-3-small` (1536-d vectors)
  - chat models: mixed usage (`gpt-3.5-turbo`, `gpt-4o`) depending on flow
- Qdrant: vector search with payload metadata and filtering.
- Zod: runtime schema validation for env and onboarding/chat contracts.
- better-sqlite3: planned structured persistence (currently mostly in-memory service stores + schema scaffolding).
- Docker + Compose: portable local/prod-like setup.
- Jest: service-level tests.

## 5. Repository Map (Important Paths)

- `backend/src/app.ts`: app bootstrap, plugins, CORS, health, route registration
- `backend/src/index.ts`: process entry
- `backend/src/env.ts`: environment validation
- `backend/src/routes/*.ts`: HTTP surface
- `backend/src/chat/chat-service.ts`: chat orchestration + prompting + retrieval integration
- `backend/src/services/*.ts`: embedding, retrieval, ingestion, memory, parser
- `backend/src/vector/qdrant.ts`: Qdrant manager
- `backend/src/webhooks/services/*.ts`: verifier, queue, processor, blog sync
- `backend/data/...`: ingestion assets (resume Q&A, cheatsheet)
- `resume.json`, `blogs.json`: root-level content inputs
- `backend/public`: downloadable assets
- `backend/tests/*.test.ts`: Jest test suite
- `docker-compose.yml`, `backend/Dockerfile`: deployment artifacts
- `.github/workflows/deploy.yml`: CI/CD pipeline

## 6. End-to-End Request Lifecycles

## A) Chat lifecycle
1. Client calls `/api/chat/ask` (SSE) or `/api/chat/message` (JSON).
2. `ChatService` validates input with Zod.
3. Session is created/loaded in memory service.
4. User message is embedded using OpenAI embeddings.
5. Retriever searches Qdrant top-K and filters by score.
6. Context is assembled from retrieval results + blog links.
7. System prompt enforces strict JSON output format.
8. OpenAI completion runs:
   - streaming chunks for SSE endpoint
   - full response for non-streaming endpoint
9. Assistant output is stored in session memory and returned.

## B) Resume ingestion lifecycle
1. Admin calls `/api/admin/ingest/resume`.
2. `ResumeIngestor` loads `resume.json` and project/experience Q&A JSONs.
3. Old vectors with source `resume`, `resume_qa`, `resume_structured` are deleted.
4. Resume + Q&A are chunked and batch-embedded.
5. Vectors upsert to Qdrant with metadata tags (`source`, `section`, `projectId`, etc.).

## C) Webhook lifecycle
1. Webhook arrives at `/api/webhooks/*`.
2. Request is authenticated by HMAC or bearer token.
3. Payload goes into in-memory queue.
4. Queue processor executes with retries and backoff.
5. Repeated failures go to DLQ.
6. Medium webhook optionally syncs `public/blogs.json`.

## 7. API Surface (Interview-Useful Inventory)

## Chat APIs
- `POST /api/chat/ask`: SSE streaming chat.
- `POST /api/chat/message`: standard JSON chat.
- `GET /api/chat/:sessionId/history`: session message history.
- `POST /api/chat/:sessionId/clear`: clear chat history.

## Command APIs (deterministic content)
- `GET /api/commands/start`
- `GET /api/commands/projects`
- `GET /api/commands/stack`
- `GET /api/commands/experience`
- `GET /api/commands/education`
- `GET /api/commands/summary`
- `GET /api/commands/achievements`
- `GET /api/commands/timeline`
- `GET /api/commands/blog`
- `GET /api/commands/resume`
- `GET /api/commands/contact`
- `GET /api/commands/misc`
- `GET /api/commands/all`

## Misc APIs
- `GET /api/misc/resume`: download resume PDF.
- `GET /api/misc/extensions`: download VS Code extension list.
- `POST /api/misc/contact`: validates + forwards contact form to n8n webhook.

## Onboarding APIs
- `POST /api/onboarding/start`
- `GET /api/onboarding/:sessionId`
- `POST /api/onboarding/:sessionId/step/1..5`
- `POST /api/onboarding/:sessionId/complete`
- `POST /api/onboarding/parse-resume`
- `POST /api/onboarding/upload-resume`
- `POST /api/onboarding/:sessionId/checkpoint`
- `GET /api/onboarding/:sessionId/checkpoint-resume`
- `POST /api/onboarding/:sessionId/finalize`
- `DELETE /api/onboarding/:sessionId`
- debug/test endpoints for step isolation and diagnostics

## Admin APIs
Profiles:
- `POST /api/admin/profiles`
- `GET /api/admin/profiles`
- `GET /api/admin/profiles/:id`
- `PATCH /api/admin/profiles/:id`
- `DELETE /api/admin/profiles/:id`

Projects:
- `POST /api/admin/projects`
- `GET /api/admin/projects`
- `GET /api/admin/projects/:id`
- `PATCH /api/admin/projects/:id`
- `DELETE /api/admin/projects/:id`

Analytics:
- `GET /api/admin/analytics/metrics`
- `GET /api/admin/analytics/events`
- `POST /api/admin/analytics/events`

Ops + ingest:
- `GET /api/admin/health`
- `GET /api/admin/vectors`
- `DELETE /api/admin/vectors/:projectId`
- `GET /api/admin/logs`
- `DELETE /api/admin/logs`
- `GET /api/admin/ingest/status`
- `POST /api/admin/ingest/github`
- `POST /api/admin/ingest/resume`
- `POST /api/admin/ingest/cheatsheet`
- `POST /api/admin/ingest/medium`
- `GET /api/admin/ingest/medium/articles`
- `GET /api/admin/blogs/sync`
- `POST /api/admin/debug/retrieve`

## Webhook APIs
- `POST /api/webhooks/github`
- `POST /api/webhooks/medium`
- `POST /api/webhooks/ingest`
- `GET /api/webhooks/status`
- `GET /api/webhooks/dlq`
- `POST /api/webhooks/dlq/retry/:id`

## Health
- `GET /health`

## 8. Data and Storage Design

## Qdrant
- collection: `portfolio_content`
- vector size: 1536
- distance: cosine
- common payload fields:
  - `text`
  - `source`
  - `projectId`
  - `chunkIndex`
  - metadata block (`section`, links, tags, etc.)

## In-memory stores (current behavior)
- chat sessions and messages (TTL-based in `MemoryService`)
- onboarding sessions and progress
- admin profile/project/analytics service stores
- webhook queue + DLQ

Note: SQLite schema/migration files exist but many current admin flows are in-memory. This is a good interview talking point about MVP pragmatism vs. production persistence hardening.

## File-based sources
- `resume.json`: main structured resume source for command APIs + resume ingestion
- `blogs.json` / `public/blogs.json`: blog metadata feed for command/chat context
- `backend/data/resume ingestion/jsons/*.json`: synthetic Q&A knowledge files
- `backend/data/cheatsheet/cheatsheet.json`: additional Q&A knowledge

## 9. RAG and AI Strategy

## Retrieval
- embed user query via OpenAI embeddings
- retrieve top-K candidates from Qdrant
- threshold filtering (`score >= 0.3`)
- optional filter support by `projectId`/`source`

## Prompting behavior
- assistant identity is bound to portfolio owner persona
- strict instruction to return JSON payloads (text/hybrid command suggestion)
- incorporates retrieved evidence and blog links when available

## Resume parsing design
- separate extraction calls for skills, experience, education, summary
- explicit anti-hallucination prompts
- skill taxonomy constraints for category normalization
- retry/backoff handling for transient API failures

## 10. Security and Reliability

## Security controls
- environment schema validation on startup (required secrets/API keys)
- webhook HMAC signature verification with timing-safe comparison
- bearer token fallback for webhook ingest
- CORS allowlist support via `ALLOWED_ORIGINS`
- rate limiting at Fastify layer (`100 requests/min` default)
- multipart size limits (10 MB)
- honeypot + email validation for contact form

## Reliability controls
- webhook queue retries with backoff and dead-letter queue
- graceful error handling in routes/services
- health check endpoint + Docker healthcheck
- log buffering endpoint for operational introspection

## 11. Deployment and DevOps

## Local/container deployment
`docker-compose.yml` launches:
- backend service on host `3200 -> container 3000`
- qdrant service on `6333/6334`

Backend container behavior:
- multi-stage TypeScript build
- production dependency install
- copies `resume.json`, `blogs.json`, `backend/data`, `backend/public`, `extensions.txt`
- starts with `node dist/index.js`

## CI/CD workflow
- triggered on push to `dev` and `main`
- test + build job on GitHub-hosted runner
- `dev` deploy job to self-hosted environment using Docker Compose
- `main` deploy job pushes to Heroku remote

## 12. Testing Strategy

Current test suite includes:
- chat service behavior and request validation
- onboarding flow step validations/progress
- admin service CRUD operations
- webhook verifier, processor, queue behavior
- config manager tests

Pragmatic observation:
- tests are mostly unit/integration-lite around service logic
- external API key dependent paths are conditionally skipped or expected to fail without key

## 13. Most Important Implementation Trade-Offs

1. In-memory admin/onboarding/chat stores vs persistent DB
- Pros: speed of delivery, lower complexity.
- Cons: data loss on restart, not horizontally scalable as-is.

2. Simple lexical chunking and threshold retrieval
- Pros: straightforward and fast.
- Cons: no reranker, no hybrid retrieval, no semantic compression.

3. JSON file based portfolio source of truth
- Pros: easy to edit and version.
- Cons: requires careful sync when ingest pipelines and runtime reads diverge.

4. Mixed model usage (`gpt-3.5-turbo`, `gpt-4o`)
- Pros: cost/perf tuning by path.
- Cons: response behavior variance unless standardized.

## 14. Known Gaps and Production Hardening Opportunities

- Persist admin/profile/project/analytics to SQLite/Postgres instead of in-memory maps.
- Persist chat memory sessions externally for scale-out.
- Add authentication/authorization guardrails on admin endpoints.
- Add idempotency keys and durable storage for webhook queue.
- Add reranking and better retrieval quality controls (MMR, hybrid BM25+vector, re-ranker).
- Add observability stack (metrics/traces) beyond log buffer.
- Enforce schema versioning for `resume.json` and ingestion payloads.
- Consolidate chat model selection and structured output parser robustness.

## 15. Interview Talking Points (Strong Answers)

## "What was the hardest engineering problem?"
Keeping data quality high while ingesting heterogeneous sources (resume + GitHub + Medium + custom Q&A), then making retrieval relevant enough for conversational answers.

## "How did you avoid hallucination?"
- constrained extraction prompts
- taxonomy-driven skill extraction
- retrieval grounding from Qdrant payloads
- explicit fallback when knowledge is missing

## "How did you design for operations?"
- health checks, log buffer endpoint, admin diagnostics
- retry + DLQ for webhook reliability
- explicit ingest endpoints for reindexing and maintenance

## "How would you scale this?"
- move all in-memory state to persistent stores
- add Redis for session/cache, Postgres for metadata
- run multiple stateless API replicas behind load balancer
- externalize queue and worker processing

## 16. Practical Runbooks

## Start locally
```bash
cp .env.production.template .env
# fill required secrets
cd backend && npm install
cd .. && docker-compose up -d
curl http://localhost:3200/health
```

## Re-ingest core knowledge
```bash
curl -X POST http://localhost:3200/api/admin/ingest/resume
curl -X POST http://localhost:3200/api/admin/ingest/cheatsheet
curl -X POST http://localhost:3200/api/admin/ingest/github
curl -X POST http://localhost:3200/api/admin/ingest/medium
```

## Validate vector state
```bash
curl http://localhost:3200/api/admin/vectors
```

## Debug retrieval quality
```bash
curl -X POST http://localhost:3200/api/admin/debug/retrieve \
  -H "Content-Type: application/json" \
  -d '{"query":"Tell me about your project experience"}'
```

## Check webhook queue health
```bash
curl http://localhost:3200/api/webhooks/status
curl http://localhost:3200/api/webhooks/dlq
```

## 17. Quick "System Design" Summary You Can Say in 30 Seconds

"I built a Fastify + TypeScript backend that ingests resume, GitHub, Medium, and curated Q&A content into Qdrant embeddings. Chat requests are embedded, retrieved against that vector store, and grounded into a strict JSON prompt pipeline for both streaming and non-streaming answers. I also built deterministic command endpoints for portfolio sections, onboarding workflows with resume parsing, and resilient webhook automation with retries and dead-letter queue support. The main next step is replacing in-memory stores with persistent backing services for horizontal scalability."

## 18. Quick "Deep Dive" Summary You Can Say in 2 Minutes

"The architecture has API routes, domain services, and a vector data layer. On ingestion, each content source is normalized and chunked; embeddings are generated with OpenAI and upserted into a single Qdrant collection with rich payload metadata. For chat, I validate input with Zod, keep session memory, retrieve top semantic matches, assemble contextual prompts, and stream generated output over SSE when needed. Admin endpoints support observability and operations, including vector stats and re-ingestion. Webhooks are authenticated with HMAC or bearer tokens, then processed through a retry queue with a dead-letter fallback. CI runs tests/build on every push to dev/main and deploys via Docker-based flows."

---

If you want, I can also generate a second, shorter one-page "interview cheat sheet" derived from this document with only likely Q&A prompts and bullet-point answers.
