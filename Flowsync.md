# FlowSync — Interview Deep Dive (Single-File)

> Purpose: a single, code-grounded document to help you explain **what FlowSync is**, **how it works end-to-end**, and **why it was built this way**.
>
> This doc is based on the actual repo implementation in this workspace (extension + MCP server + AWS CDK + Lambdas + Next.js dashboard).

---

## 0) Elevator pitch (30 seconds)

**FlowSync gives AI coding agents persistent project memory**.

- Agents can **log decisions / risks / tasks / reasoning** via `log_context`.
- Agents can **retrieve and search prior context** via `search_context` and other MCP tools.
- Even if a developer doesn’t use an agent, FlowSync still grows a “project brain” by **auto-capturing push diffs** and using Bedrock to extract structured context.

The output is not “chat history”; it’s **structured, searchable, branch-aware project context** with traceability back to commits and events.

---

## 1) What problems this solves

### 1.1 Core pain
AI assistants are powerful but tend to be **stateless** across sessions and across team members.

- Git stores *what changed*, not *why*.
- Docs go stale.
- Chat logs are fragmented and hard to search.

### 1.2 What FlowSync stores that Git doesn’t
FlowSync’s context records deliberately capture:

- **Decision**: what was chosen (and sometimes why)
- **Risks**: what could break / security/ops concerns
- **Tasks**: what’s next / TODOs
- **Entities**: functions/classes/files touched
- **Agent reasoning**: the “why” you can’t infer from diffs

---

## 2) Repository map

Top-level directories (as shipped in this repo):

- `extension/` — VS Code extension (TypeScript) + Webview UI
- `mcp-server/` — TypeScript MCP server (stdio) used by agents
- `infra/` — AWS CDK stack + Lambda code
- `frontend/` — Next.js dashboard (static export to S3/CloudFront)
- `docs/` — planning/spec PDFs/notes, prototype submission, benchmark prompts
- `DOCUMENTATION/` — long-form docs created during the hackathon (deep dive + perf report)

Useful starting points:

- Root README: explains the concept + high-level architecture
- `infra/lib/infra-stack.ts`: authoritative list of AWS resources and routes
- `extension/src/extension.ts`: extension activation + hook listener + push capture
- `mcp-server/src/index.ts`: MCP tool definitions and API wiring

---

## 3) System architecture (runtime)

### 3.1 High-level diagram

```
Developer + VS Code extension               AWS Backend                          Agent
───────────────────────────────            ─────────────────────────────────     ─────────────────────────
1) pre-push hook triggers
   (curl → localhost listener)  ───────▶   API Gateway POST /api/v1/events
                                          │
                                          ▼
                                      Ingestion Lambda (Node.js 20)
                                      - auth + validate
                                      - store flowsync-events
                                      - archive raw payload to S3
                                      - async invoke AI processing
                                          │
                                          ▼
                                      AI Processing Lambda (Python 3.12)
                                      - Bedrock Nova Pro extract
                                      - Titan embedding
                                      - write flowsync-context
                                      - merge propagation on merges

Dashboard (Next.js)  ◀──────────────▶  GET /api/v1/projects/{id}/events (Query Lambda)
                                      POST /api/v1/query (Query Lambda)
                                      POST /api/v1/chat (Chat Lambda)

MCP Server (stdio)   ───────────────▶  POST /mcp (MCP Lambda)
(used by Copilot/Cursor/Claude)
```

### 3.2 Key design choices

- **Serverless**: Lambda + DynamoDB scales to zero, handles bursty pushes.
- **Async AI processing**: ingestion returns quickly; AI runs out-of-band.
- **Branch-aware context**: all context is keyed/scoped by branch + extractedAt.
- **RAG with citations**: answers include the top records used (with relevance scores).

---

## 4) VS Code Extension (client)

### 4.1 Activation behavior

- The extension activates when the workspace contains `.flowsync.json` (`activationEvents` in `extension/package.json`).
- On activation, it shows a **persistent status bar item**: `$(zap) FlowSync`.
- It reads `.flowsync.json` and then tries to load the API token from VS Code **SecretStorage**.

Key file: `extension/src/extension.ts`.

### 4.2 The project config file: `.flowsync.json`

This is committed to the repo root and contains **no secrets**.

Shape (see `extension/src/config.ts`):

```jsonc
{
  "projectId": "...",
  "backendUrl": "https://.../prod",
  "defaultBranch": "main",
  "port": 38475
}
```

Notes:

- `port` is per-workspace and used for the local HTTP hook listener.
- The API token is stored in SecretStorage under key `flowsync.token.<projectId>`.

### 4.3 Initialization vs joining

FlowSync onboarding has two paths, both driven by the **webview panel**.

Authoritative handler: `extension/src/panels/FlowSyncPanel.ts`.

**Initialize** (team lead)
- In the webview UI, the user submits the init form → the webview posts `{ type: "initProject" }`.
- `FlowSyncPanel` calls `POST /api/v1/projects` (Ingestion Lambda) to create a project + mint an API token.
- `FlowSyncPanel` then writes:
  - `.flowsync.json`
  - `.github/copilot-instructions.md`
  - `.vscode/mcp.json`
- Stores the token in VS Code SecretStorage (`flowsync.token.<projectId>`).
- Installs a Git `pre-push` hook and starts the local hook listener.

**Join** (other devs)
- In the webview UI, the user pastes a token → the webview posts `{ type: "joinProject" }`.
- `FlowSyncPanel` validates via `GET /api/v1/projects/{projectId}` with `Authorization: Bearer <token>`.
- Stores the token in SecretStorage and writes `.vscode/mcp.json`.

Webview UI components (handy for demos / interviews):
- `extension/webview-ui/src/components/InitProject.tsx`
- `extension/webview-ui/src/components/JoinProject.tsx`

### 4.4 Push capture mechanism (Git hook + local HTTP listener)

FlowSync uses a **real Git hook**: `pre-push`.

- During init (and if port changes), the extension writes `.git/hooks/pre-push`.
- The hook executes a background `curl` to `http://localhost:<port>/flowsync-hook` and returns immediately.
  - This is the “near-zero overhead” claim: it runs in the background (`&`).

Local listener:

- The extension runs an HTTP server on `127.0.0.1:<port>`.
- When it receives `POST /flowsync-hook`, it triggers capture logic.

Files:

- Hook listener server: `extension/src/hookListener.ts`
- Hook writer: `extension/src/panels/FlowSyncPanel.ts` (`injectHook`) and `extension/src/extension.ts` (`updateHookPort`)

Important implementation detail:

- The hook script written by `extension/src/extension.ts` includes `remoteRef` (the old remote SHA) so the extension *could* diff a full push-range.
- **Current code note:** `extension/src/hookListener.ts` parses the incoming JSON but calls `onPush(branch)` without forwarding `remoteRef`, so the push-range SHA is effectively ignored today.
  - As a result `getDiff(remoteRef)` usually falls back to merge-base or `HEAD~1..HEAD` strategies.

### 4.5 How the extension computes diffs

Main logic: `extension/src/gitUtils.ts`.

- If `remoteRef` is available and non-zero hash, it attempts: `git diff <remoteRef> HEAD`.
- If it detects a merge commit (2+ parents), it diffs from the merge base: `git merge-base HEAD~1 HEAD~2` then `git diff <mergeBase> HEAD`.
- Otherwise it uses: `git diff HEAD~1 HEAD`.
- Truncates diffs at **50,000 characters** to protect payload sizes.

### 4.6 Event payload sent to backend

The extension builds a `CapturedEvent` (see `extension/src/eventTransmitter.ts`):

- `eventId`: UUID
- `eventType`: `push`
- `timestamp`: ISO
- `branch`: current branch
- `payload`:
  - `commitHash`, `message`, `author`, `diff`
  - optional merge metadata: `isMerge`, `sourceBranch`

It sends `POST <backendUrl>/api/v1/events` with `Authorization: Bearer <token>`.

Retries: 4 attempts with delays `[0ms, 1000ms, 2000ms, 4000ms]`.

### 4.7 “Catch Me Up” feature

Command: `FlowSync: Catch Me Up`.

- Fetches recent events from `GET /api/v1/projects/{projectId}/events?since=...&limit=50`.
- Aggregates authors, decisions, risks, and tasks.
- Opens the webview and sends a summary payload.

Key file: `extension/src/commands/catchMeUp.ts`.

---

## 5) MCP Server (TypeScript, stdio)

### 5.1 What it is

The MCP server is the bridge between an AI agent and FlowSync’s backend.

- Transport: **stdio** (MCP standard)
- SDK: `@modelcontextprotocol/sdk`
- It is bundled into the extension as `extension/dist/mcp-server.mjs` via `esbuild`.

Source:

- `mcp-server/src/index.ts` (tool definitions)
- Build/bundle script: `mcp-server/package.json` → `npm run bundle`

### 5.2 How it’s configured

By environment variables or `.vscode/mcp.json`:

- `FLOWSYNC_API_URL`
- `FLOWSYNC_PROJECT_ID`
- `FLOWSYNC_TOKEN`

The extension writes `.vscode/mcp.json` for you during init/join (see `writeMcpConfig` in `extension/src/commands/initProject.ts`).

(Implementation detail: `FlowSyncPanel` calls `writeMcpConfig(...)` from the webview-driven init/join flow.)

### 5.3 Tools

The MCP server exposes tools that call the backend MCP Lambda (`POST /mcp`) and dashboard endpoints.

Common tools:

- `get_project_context` — fetch branch context (with inheritance/merge behavior handled by backend)
- `get_recent_changes` — recent records across branches
- `search_context` — RAG search and grounded answer generation
- `log_context` — write reasoning/decisions/tasks/risks

Additional tool:

- `get_events` — calls the dashboard timeline endpoint (`GET /api/v1/projects/{projectId}/events`) and **requires** a valid API token (provided via `FLOWSYNC_TOKEN`).

The MCP server defaults `search_context` branch scope to the current git branch (`git rev-parse --abbrev-ref HEAD`) to avoid cross-branch noise.

Routing “truth table” (useful in interviews):

| Tool | Where it runs | Calls | HTTP auth |
|------|--------------|-------|----------|
| `get_project_context` | MCP Lambda (Python) | `POST /mcp` | No Bearer token |
| `get_recent_changes` | MCP Lambda (Python) | `POST /mcp` | No Bearer token |
| `search_context` | MCP Lambda (Python) | `POST /mcp` | No Bearer token |
| `log_context` | MCP Lambda (Python) | `POST /mcp` | No Bearer token |
| `get_events` | MCP server (TypeScript) | `GET /api/v1/projects/{id}/events` | Bearer token required |

---

## 6) Backend (AWS) — CDK stack

Authoritative stack: `infra/lib/infra-stack.ts`.

### 6.1 DynamoDB tables

Provisioned tables include:

- `flowsync-projects` — project metadata + `apiTokenHash`
- `flowsync-events` — raw ingested push events
  - PK: `projectId`
  - SK: `timestampEventId` = `<timestamp>#<eventId>`
  - GSI `EventIdIndex` (PK `eventId`)
  - GSI `BranchIndex` (PK `projectId`, SK `branchTimestamp` = `<branch>#<timestamp>`)
- `flowsync-context` — AI-extracted context records + embeddings
  - PK: `eventId`
  - GSI `ProjectContextIndex` (PK `projectId`, SK `extractedAt`)
  - GSI `BranchContextIndex` (PK `projectId`, SK `branchExtractedAt` = `<branch>#<extractedAt>`)
- `flowsync-audit` — audit trail
- `flowsync-chat-sessions` — session state for chat, TTL-based cleanup
- `flowsync-cache` — RAG response cache (TTL via `expiresAt`)

### 6.2 S3 buckets

- `flowsync-raw-events-<account>` — archives raw event payload JSON.
- `flowsync-frontend-<account>-<region>` — static site hosting bucket for Next.js export.

### 6.3 Lambda functions

- `flowsync-ingestion` (Node.js 20) — routes:
  - `POST /api/v1/projects` (create project + mint token)
  - `POST /api/v1/events` (ingest push payload)
  - `GET  /api/v1/projects/{projectId}` (join flow token check)
- `flowsync-ai-processing` (Python 3.12) — async invoked; does Bedrock extraction + embeddings + context write
- `flowsync-mcp` (Python 3.12) — MCP tool dispatcher (`POST /mcp`)
- `flowsync-query` (Python 3.12) — dashboard timeline and `/api/v1/query`
- `flowsync-chat` (Python 3.12) — conversational interface using Nova Lite; optionally uses RAG pipeline for factual queries

### 6.4 API Gateway routes

From CDK:

- `POST /api/v1/events` → ingestion
- `POST /api/v1/projects` → ingestion
- `GET  /api/v1/projects/{projectId}` → ingestion
- `GET  /api/v1/projects/{projectId}/events` → query
- `POST /api/v1/query` → query
- `POST /api/v1/chat` → chat
- `POST /mcp` → mcp

---

## 7) Backend logic — Ingestion Lambda (Node.js)

File: `infra/lambda/ingestion/index.js`.

### 7.1 Auth model (project token)

- Tokens are generated at project creation time.
- Backend stores **only a scrypt hash** of the token (`salt:hash`), never plaintext.

Implementation:

- Mint: `crypto.randomBytes(32).toString('hex')` (64 hex chars)
- Hash: `crypto.scryptSync(plaintext, salt, 64)`
- Verify: recompute and use `crypto.timingSafeEqual`

### 7.2 Event ingestion

Steps on `POST /api/v1/events`:

1. Parse JSON
2. Authenticate Bearer token against `flowsync-projects.apiTokenHash`
3. Validate schema (UUID v4, ISO timestamps, commit hash shape, diff max 50k, etc.)
4. Put raw event record into `flowsync-events`
5. Archive full payload JSON into S3
6. Invoke AI processing Lambda asynchronously (`InvocationType: 'Event'`)
7. If merge commit metadata exists, also invoke AI processing with `{propagate: true}`

Key performance property: the ingestion path returns after the DynamoDB write and async invoke — it does not block on Bedrock.

---

## 8) Backend logic — AI Processing Lambda (Python)

File: `infra/lambda/ai_processing/handler.py`.

### 8.1 Extraction using Bedrock Nova

- Primary model: `us.amazon.nova-pro-v1:0`
- Converse API with strict JSON output rules
- Temperature = 0 (deterministic intent extraction)
- Fallback to Nova Lite on throttle/timeouts/unavailability

Output schema:

- `feature`, `decision`, `tasks`, `stage`, `risk`, `entities`

### 8.2 Confidence scoring

Confidence is computed deterministically from field completeness:

- Base 0.55
- +0.15 if `decision`
- +0.15 if `risk`
- +0.10 if tasks non-empty
- +0.05 if 2+ entities

### 8.3 Embeddings

- Embedding model: Titan embed v1 (`amazon.titan-embed-text-v1`)
- Vector length expected: 1536

### 8.4 Context record persisted

Writes to `flowsync-context` with:

- indexing fields: `projectId`, `branch`, `branchExtractedAt`, `extractedAt`
- traceability fields: `eventId`, `commitHash`, `author`
- content fields: feature/decision/tasks/stage/risk/entities/agentReasoning
- embedding: `embedding`

### 8.5 Orphaned record merge (log-first workflow)

FlowSync supports a “log-first” path:

- If an agent logs reasoning before a commit is captured, the MCP Lambda can create an **uncommitted** record (`commitHash = null`).
- When AI processing later sees a commit from the same author+branch within a 30-minute window, it binds the commit hash to the orphan record rather than creating a duplicate.

(Implementation detail: `find_orphaned_record` + `update_orphaned_record`.)

### 8.6 Merge propagation

If ingestion flags a merge (`isMerge` + `sourceBranch`), AI processing copies context records from source branch to target branch by writing new records with:

- new UUID `eventId`
- `branch` = target
- `mergedFrom` = source
- `branchExtractedAt` updated

---

## 9) Backend logic — MCP Lambda (Python)

File: `infra/lambda/mcp/handler.py`.

This Lambda implements the server-side tool logic used by the MCP server.

Tools:

- `get_project_context`
  - Fetches branch records (BranchContextIndex)
  - If branch != main, it also fetches main records and merges “inheritance”
- `get_recent_changes`
  - Uses ProjectContextIndex (or BranchContextIndex) with optional `since`
- `search_context`
  - Calls shared RAG pipeline `search_context_rag()`
- `log_context`
  - Updates most recent record within 30 minutes (same author+branch, status complete)
  - Otherwise creates an orphan `uncommitted` record
  - Re-embeds after enrichment (non-fatal if embedding fails)

Note: the **MCP Lambda** endpoint (`POST /mcp`) does not require a Bearer token on the HTTP layer. However, the standalone **TypeScript MCP server** also exposes `get_events`, which calls the dashboard API (`/api/v1/projects/{id}/events`) and **does** require Bearer auth.

---

## 10) Backend logic — Query Lambda (Python)

File: `infra/lambda/query/handler.py`.

Routes:

- `GET /api/v1/projects/{projectId}/events`
  - Auth required (Bearer)
  - Reads from `flowsync-context` (BranchContextIndex or ProjectContextIndex)
  - Strips embeddings to keep response small
- `POST /api/v1/query`
  - Auth required (Bearer)
  - Calls the shared RAG pipeline (Titan embed + cosine similarity + Nova answer)

---

## 11) Shared RAG pipeline + caching

File: `infra/lambda/shared/python/flowsync_common/helpers.py`.

### 11.1 RAG steps

1. Optional cache check (DynamoDB `flowsync-cache`)
2. Embed query via Titan
3. Query all relevant context records (paginated)
4. Cosine similarity vs stored embeddings
5. Select top-5 records
6. Generate a grounded answer with Nova Pro (fallback on throttle)
7. Cache the response for 1 hour (TTL)
8. Return answer + sources with relevance

### 11.2 Cache key

- `sha256(f"{projectId}:{query}:{branch or 'all'}")`

### 11.3 Branch affinity

- If no branch filter is specified, non-main records are penalized by `0.85×` to avoid cross-branch pollution.

---

## 12) Dashboard (frontend)

Directory: `frontend/`.

### 12.1 What it does

- Shows a timeline of context records
- Search via `/api/v1/query`
- Chat via `/api/v1/chat`
- Branch filtering and analytics

### 12.2 API client

File: `frontend/src/lib/api.ts`.

- Uses Axios with base URL from `NEXT_PUBLIC_API_BASE_URL` or default prod.
- Supports mock mode via `NEXT_PUBLIC_USE_MOCK=true`.

### 12.3 Auth model

Project-based auth (not user accounts):

- User enters `projectId` + token (from `.flowsync.json` / init token modal).
- Stored in localStorage (see `frontend/src/lib/constants.ts` for key name).

---

## 13) Deployment (CDK)

### 13.1 Backend deploy

From `infra/`:

```bash
npm install
npm run build
npx cdk bootstrap
npx cdk deploy
```

### 13.2 Frontend deploy

- Next.js is built/exported to `frontend/out/`.
- CDK deploy includes an S3 bucket deployment that uploads `frontend/out/` and invalidates CloudFront (`distributionPaths: ['/*']`).

In practice: run `npm run build` inside `frontend/`, then deploy via CDK.

---

## 14) Observability + debugging

### 14.1 Logs

- Extension logs to a dedicated output channel (see `extension/src/logger.ts`).
- Lambdas print structured logs; AI processing prints a `BENCHMARK_LOG` JSON object that can be parsed.

### 14.2 Common failure modes

Client-side:

- Token missing in SecretStorage → extension prompts to join.
- Hook port conflicts → extension picks a new port and rewrites `.flowsync.json` + hook.
- Git commands fail (no commits yet, detached head) → extension warns and skips.

Server-side:

- Token verification fails → 401.
- Bedrock throttling → adaptive retries; fallback model.
- Cache failures → non-fatal; pipeline continues.

---

## 15) Security model (what to emphasize in interviews)

- **No secrets committed**: `.flowsync.json` contains no token.
- Tokens are stored locally in VS Code **SecretStorage**.
- Tokens are stored server-side only as **scrypt hashes** (salted).
- Comparisons use timing-safe methods (`timingSafeEqual` / `compare_digest`).
- TLS is handled by API Gateway.

---

## 16) Performance and cost (talking points)

The repo includes a performance report with benchmark claims; the implementation supports:

- async ingestion (fast response to pushes)
- embedding + RAG search
- cache hits returning quickly (DynamoDB GetItem vs Bedrock)

For interview prep, the best way to explain performance is:

- Ingestion is designed to be <500ms and not wait for Bedrock.
- AI processing dominates time; everything else is IO.
- Caching is a direct cost/latency lever for repeated questions.

---

## 17) “Explain it like I’m an interviewer” — end-to-end walkthrough

### 17.1 Day 1 setup

1. Install extension
2. Initialize project (creates project + token)
3. Token stored in SecretStorage
4. `.flowsync.json` committed (share projectId/port/backend URL)
5. `.vscode/mcp.json` written for tool discovery
6. `.git/hooks/pre-push` written to trigger local capture

### 17.2 Normal dev loop

1. Developer pushes code
2. Git `pre-push` runs → background curl → local HTTP listener
3. Extension computes diff + commit metadata
4. Extension sends event to `POST /api/v1/events`
5. Ingestion writes raw event + archives to S3 + invokes AI processing
6. AI processing writes structured context + embedding to `flowsync-context`
7. Dashboard and agents can now query/search.

### 17.3 Agent loop

- Before working: agent calls `search_context` / `get_project_context`
- After working: agent calls `log_context` to store reasoning/risks/tasks

---

## 18) Known documentation mismatches (be aware)

This repo contains multiple docs written at different points in the build. A few statements differ from the current code:

- Ingestion runtime: code deploys **Node.js 20** ingestion (`infra/lib/infra-stack.ts` + `infra/lambda/ingestion/index.js`), but some docs mention Python.
- Git hook: code installs a **pre-push** hook (Git has no post-push hook), and runs curl in the background.

For interviews, rely on the code files referenced throughout this document.

---

## 19) Where to look for specific answers quickly

- “How is the push captured?” → `extension/src/extension.ts`, `extension/src/hookListener.ts`, `extension/src/gitUtils.ts`
- “What’s in the DynamoDB schema?” → `infra/lib/infra-stack.ts`
- “How is auth done?” → `infra/lambda/ingestion/index.js` (Node) and `infra/lambda/shared/python/flowsync_common/auth.py` (Python)
- “How does RAG work?” → `infra/lambda/shared/python/flowsync_common/helpers.py`
- “What does `log_context` do?” → `infra/lambda/mcp/handler.py`
- “How does the MCP server expose tools?” → `mcp-server/src/index.ts`
- “What does the dashboard call?” → `frontend/src/lib/api.ts`

---

## 20) Suggested interview Q&A prompts (practice)

- Why MCP instead of a custom protocol?
- How do you prevent hallucinations / ensure traceability?
- How do you keep cost bounded at scale?
- How do you handle branch context and merges?
- How do you handle retries and throttling from the model provider?
- What would you change for a production multi-tenant version?

---

## Appendix A — Build commands (per component)

### Extension

From `extension/`:

```bash
npm install
npm run compile
npm run vsix
```

(Also note `vscode:prepublish` bundles the MCP server into the extension.)

### MCP server

From `mcp-server/`:

```bash
npm install
npm run build
npm run bundle
```

### Infra

From `infra/`:

```bash
npm install
npm run build
npx cdk deploy
```

### Frontend

From `frontend/`:

```bash
npm install
npm run dev
npm run build
```
