r
# Skillmap — Full-stack Deep Dive & Interview Prep

This document covers both parts of the codebase you'll be asked about in interviews:

- Frontend: `skillmap` (Next.js + React, UI components, pages, auth)
- Backend: `skillmap_engine` (Node.js services, OpenAI embeddings, Qdrant vector store)

Read this as a single reference you can use to walk through the full flow (upload -> parse -> embed -> suggest) and answer system-design and implementation questions.

## Summary — what each repo does

- `skillmap` (frontend)
  - Built on Next.js (app dir), Tailwind CSS, and a component library in `components/`.
  - Handles user-facing pages: onboarding, dashboard, uploading resumes, review/search, and peer matching.
  - Uses contexts (e.g., `contexts/AuthContext.tsx`) and hooks (e.g., `useAuthRedirect.ts`) for auth and app flow.

- `skillmap_engine` (backend)
  - Node.js server that parses uploads, generates embeddings (OpenAI), stores vectors (Qdrant), and runs domain services: resume analysis, LeetCode suggestions, skill search.
  - Contains `config/`, `services/`, `utils/`, `scripts/`, and `taxonomy/`.

## How the front-end and back-end interact

Typical request sequence:

1. User uploads a resume from `skillmap` UI (a Next.js form, page under `app/upload/page.tsx`).
2. The front-end POSTs the file to an API endpoint served by `skillmap_engine` (multipart/form-data).
3. Backend saves the file, extracts text, generates embeddings, and stores them in Qdrant.
4. Front-end polls or requests processed results (skills, suggestions, problems) from backend endpoints.
5. Front-end displays results using UI components from `components/ui/*`.

Important cross-cutting concerns:

- Authentication: the front-end will include auth tokens/cookies; backend must verify/authorize requests.
- API contract: payload shapes, status codes, and error semantics must be consistent. Consider adding an OpenAPI spec.
- Latency: embedding calls are slow; front-end should show progress or process uploads asynchronously.

## Frontend (skillmap) — key files and responsibilities
2. Environment variables (create a `.env` or export in shell)

These are the exact environment variable names and scripts used by the codebase (extracted from `config/*.js` and `package.json`). Use Bash (your shell) to export variables or put them in a `.env` file.

Required backend environment variables (exact names used in `skillmap_engine`):

- `OPENAI_API_KEY` — OpenAI API key (required by `config/openai.js`).
- `QDRANT_URL` — Qdrant endpoint (required by `config/qdrant.js`), e.g. `http://localhost:6333`.
- `QDRANT_API_KEY` — Qdrant API key (optional for local installs, used if set).
- `SUPABASE_URL` — Supabase URL (required by `config/supabase.js`).
- `SUPABASE_SERVICE_ROLE_KEY` — Supabase service role key (required by `config/supabase.js`).
- `PORT` — optional, defaults to `5005` (the server uses `process.env.PORT || 5005`).

Backend install & run (from the `skillmap_engine` folder):

```bash
cd skillmap_engine
npm install

# export required env vars (example)
export OPENAI_API_KEY="sk-..."
export QDRANT_URL="http://localhost:6333"
export QDRANT_API_KEY=""
export SUPABASE_URL="https://your-project.supabase.co"
export SUPABASE_SERVICE_ROLE_KEY="your-service-role-key"
export PORT=5005

# Start Qdrant locally (if you need a local vector DB)
docker run -p 6333:6333 -v qdrant_storage:/qdrant/storage qdrant/qdrant

# Development (auto reload using nodemon)
npm run dev

# Production / simple start
npm start
```

Important backend scripts (from `skillmap_engine/package.json`):

- `npm start` — runs `node index.js`.
- `npm run dev` — runs `nodemon index.js`.
- `npm run seed-taxonomy` — run `scripts/seedTaxonomy.js` to seed the taxonomy data.
- `npm run docker:build` / `npm run docker:run` — docker helpers included in package.json.

Quick smoke-check (after backend is running):

```bash
curl http://localhost:5005/health
```

If it returns JSON with status OK, the backend is reachable.
## Backend (skillmap_engine) — key files and responsibilities

- `index.js` — server entrypoint and route wiring.
- `config/openai.js`, `config/qdrant.js`, `config/supabase.js` — external integrations.
- `utils/pdfParser.js`, `utils/vectorStore.js`, `utils/multer.js` — core helpers.
- `services/*` — business logic: resume parsing, embeddings, suggestion engines, leetcode helpers.
- `scripts/*` — maintenance scripts for seeding/deleting data.

Backend technical details to review for interview:

- How embedding requests are batched and retried.
- Qdrant collection schema and metadata choices.
- Error handling strategy when external APIs fail.
- Where to add authentication checks and rate limiting.

## End-to-end run instructions (dev)

Below are commands to run both projects locally. Run them in separate terminals.

```bash
cd skillmap_engine
npm install
export OPENAI_API_KEY=your_key_here
export QDRANT_URL=http://localhost:6333
docker run -p 6333:6333 -v qdrant_storage:/qdrant/storage qdrant/qdrant
npm start
```

2) Frontend (`skillmap`):

```bash
cd ../skillmap
pnpm install   # or npm install if you use npm
export NEXT_PUBLIC_API_BASE=http://localhost:3001   # or the backend URL
pnpm dev       # or npm run dev
Notes:
- Adjust ports and env var names to match the codebase (`index.js` or `next.config.mjs`).
- If the backend exposes authentication via Supabase, configure `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY` accordingly.

## Full-stack demo flow to show in an interview

1. Start Qdrant and backend. Show `index.js` startup logs.
2. Start frontend and open the upload page.
3. Upload a sample resume from `skillmap_engine/uploads/`.
4. Show backend logs: parse -> embedding -> upsert.
5. Refresh dashboard or poll for results to show skills / problem suggestions.
6. Walk through the code paths in the frontend (upload handler) and backend (resumeService, vectorStore).

## Likely interview questions (frontend + backend) and suggested answers

- Q: How do you handle long-running tasks (embedding large PDFs) without blocking the UI?
  - A: Use async background workers (e.g., Bull with Redis). Frontend shows progress and uses optimistic polling or webhooks/notifications when job completes.

- Q: How does next.js server/client rendering affect API calls and auth?
  - A: Server components can fetch data on the server and return HTML; client components handle interactions. Auth tokens must be forwarded in client requests; server-side calls can use service credentials stored in env variables.

- Q: How do you prevent accidental leakage of API keys from frontend builds?
  - A: Only expose public keys as NEXT_PUBLIC_*. Keep secret keys on the backend and use server-side endpoints for sensitive operations.

- Q: How would you implement role-based access control for endpoints?
  - A: Add middleware on backend to validate JWTs and check roles/claims, also enforce RBAC in the data layer (filtering queries by owner id).

- Q: How would you measure the effectiveness of recommended problems or skills?
  - A: Track user actions (accepted suggestions, solved problems), use A/B testing, and compute precision/recall vs labeled data.

## Troubleshooting & debugging tips

- If embeddings fail: inspect OpenAI error messages, check rate limits and API key validity, and add retries.
- If Qdrant queries return empty: verify collection creation, correct vector dimensionality, and metadata filters.
- If uploads are rejected: check `multer` file size/type settings and Next.js request size limits.

## Good enhancement ideas to discuss

- Add background jobs for heavy tasks and return immediate 202 responses.
- Add an explicit OpenAPI spec and typed client for the frontend.
- Add end-to-end tests that spin up a test Qdrant instance (Docker) and mock OpenAI.
- Add PII redaction and retention policies.

## Interview prep checklist (actionable)

- Walk the code paths for upload in `skillmap/app/upload/page.tsx` and backend `skillmap_engine/services/resumeService.js`.
- Be able to explain embeddings, kNN, and metadata filtering clearly in 2–3 sentences.
- Prepare to discuss trade-offs: sync vs async processing, cost vs quality for embedding models, privacy.

---
If you want, I can:

- Generate a short one-page cheat-sheet with ~12 flashcard questions and short answers covering both frontend and backend.
- Add example curl requests and sample responses for the most important backend endpoints.
- Extract exact env var names and commands by reading `index.js`, `config/*.js`, and `package.json` in both folders and update the run instructions accordingly.

Tell me which of the follow-ups you want and I'll implement it next.
