# Konta (WAH_Aegis) — Interview Deep Dive (Implementation-Accurate)

This document is a **code-grounded** walkthrough of the Konta Chrome extension in this repo (folder name: `WAH_Aegis`). It’s written to help you explain the system end-to-end in an interview: architecture, core flows, algorithms, constants/thresholds, storage layout, and the message API.

> Scope note: This describes what exists in the codebase today. A few modules are explicitly marked **legacy/disabled** in-code (e.g. `projectSuggestions.ts`).

---

## 0) 30-second explanation

**Konta** is a **local-first** MV3 Chrome extension that passively captures browsing activity, sessionizes it, embeds page titles locally using a MiniLM model, and surfaces:

- **Sessions** (auto-created + editable names/labels)
- **Knowledge graph** visualization (page similarity + clustering)
- **Semantic search** (ML → TF‑IDF → keyword fallback)
- **Project detection** (resource-based candidates and project grouping)
- **Focus Mode** (dynamic DNR blocking rules + tab sweep)
- **COI** (Cognitive Overload Indicator) based on ephemeral behavior + derived metrics, with optional in-page alerts

All data stays on-device: sessions are stored in **IndexedDB**; most configuration/state is stored in **`chrome.storage.local`**.

---

## 1) System architecture (MV3 / Plasmo)

Plasmo provides the MV3 build + file-based entrypoints.

### UI entrypoints
- `src/popup.tsx`: small “launcher” UI (e.g. open sidepanel, onboarding actions)
- `src/sidepanel.tsx`: the primary UI (sessions/graph/projects/focus/settings)
- `src/tabs/graph.tsx`: full-page graph view (separate tab UI)

### Background service worker
- `src/background/index.ts`: **the orchestration hub**
  - starts up by loading sessions from IndexedDB
  - installs listeners (page visits, sidepanel open/close, alarms)
  - handles `chrome.runtime.onMessage` API for all UI/content script commands
  - rebuilds and serves the knowledge graph
  - periodically broadcasts COI updates

### Content scripts
- `src/contents/indicator.tsx`: “indicator hub” on web pages
  - sends `CONTENT_SCRIPT_READY` handshake
  - displays in-page notifications (similar pages, project candidate prompts, reminders, COI alerts)
- Others include scroll tracker, consent UI, project notification UI, etc.

### Storage
- **IndexedDB**: sessions + pages (including title embeddings)
  - implemented in `src/background/sessionStore.ts`
- **chrome.storage.local**: settings, candidates/projects, flags, cached notifications, etc.

### Why MV3 constraints matter (real design constraints)
MV3 service workers are **ephemeral** (they can be suspended/restarted). This repo actively handles that by:

- Reloading sessions from IndexedDB when in-memory cache is empty (see `GET_SESSIONS`, `GET_GRAPH`).
- Avoiding fragile runtime injection for notifications; instead, using manifest-declared content scripts + readiness tracking.
- Running ML inference in a MV3-safe configuration (WASM-only; blocks unsupported provider module loads).

---

## 2) Data model (mental model)

### 2.1 Session + PageEvent
A session is a time-ordered list of visited pages (`PageEvent`s) with metadata.

Key ideas (see `src/background/sessionManager.ts`):
- Sessions are stored and loaded from IndexedDB.
- Within a session, page visits are **deduped by URL** (recency kept by moving updated entries to the end).
- `visitCount` is incremented on repeated visits.
- `inferredTitle` is computed from the pages (`inferSessionTitle`).
- `isImported?: boolean` marks history-import sessions so they don’t merge with live browsing.

### 2.2 Projects
Two related but distinct concepts exist:

1) **Project candidates** (incremental, real-time): `src/background/candidateDetector.ts`
- Stored under `aegis-project-candidates`
- Trigger “candidate ready” notifications based on thresholds + score

2) **Projects** (persisted set): `src/background/projectManager.ts`
- Stored under `aegis-projects`
- Can be auto-detected from sessions (batch clustering) or created manually

### 2.3 Labels + context learning
- Labels stored under `aegis-labels` (`src/background/labelsStore.ts`)
- Context-learning associations stored under `aegis-context-learning` (`src/background/contextLearning.ts`)
- Label suggestion is generated if a session is unlabeled.

### 2.4 COI (Cognitive Overload Indicator)
- Ephemeral behavior (tab switches, idle transitions, scroll patterns) tracked in `src/background/ephemeralBehavior.ts`
- Scores computed in `src/lib/coi.ts` using:
  - ephemeral behavior features
  - derived metrics from `src/derived/*`
- Weights stored under `coi-weights`

---

## 3) Core end-to-end flow (what happens while you browse)

This is the “story” to tell in an interview.

### 3.1 Startup
`src/background/index.ts` does, in order:
1) `initializeSessions()` to load sessions from IndexedDB into memory
2) Builds the knowledge graph if needed
3) Initializes context learning, focus mode state, reminder alarms
4) Installs listeners:
   - page visit listener
   - consent listener
   - sidepanel open/close listeners
   - alarms listener
   - runtime message router

### 3.2 Page visit ingestion
The ingestion logic is coordinated from `src/background/page-event-listeners.ts`.
At a high level:
1) A page visit arrives (from browser activity / content scripts)
2) It checks excluded domains from settings
3) It attempts to generate or reuse an embedding (title/query embedding)
4) Calls session manager to add/update the page inside a session
5) Marks the knowledge graph as needing rebuild
6) Runs optional secondary signals:
   - project candidate detection
   - similarity notifications
   - project main-site notifications and blocklist checks

### 3.3 Sessionization algorithm (exact thresholds)
Implemented in `src/background/sessionManager.ts`.

A new session begins if **any** of the following are true:
- **Inactivity gap**: `SESSION_GAP_MS = 30 min`
- **Max duration**: `MAX_SESSION_DURATION_MS = 2 hours`
- **Imported boundary**: if the last session `isImported` → always start new live session
- **Context switch**: if gap > `CONTEXT_SWITCH_THRESHOLD_MS = 15 sec` and `!isSameContext(lastPage, newPageEvent)`

Within the current session:
- If URL already exists, update that page entry, increment `visitCount`, preserve earliest `openedAt`, and move to end.
- Otherwise append a new page entry.

### 3.4 History import (onboarding)
Implemented in `src/background/historyImporter.ts`.

- Imports a fixed window (in code: last 7 days)
- Creates sessions marked `isImported: true`
- Generates embeddings in a batch and persists them
- Stores onboarding progress flags:
  - `onboarding-embeddings-in-progress`
  - `onboarding-embeddings-complete`
- Broadcasts `ONBOARDING_PROGRESS` updates
- Broadcasts `EMBEDDINGS_COMPLETE` when finished, which triggers:
  - session reload from IndexedDB
  - graph rebuild
  - `GRAPH_UPDATED` broadcast

---

## 4) Embeddings (local ML) — MV3-hardened

Embedding generation is in `src/background/embedding-engine.ts`.

### 4.1 Model
- Model ID: `Xenova/all-MiniLM-L6-v2`
- Task: `feature-extraction`
- Output: pooled mean embedding, normalized

### 4.2 MV3 constraints and safeguards
The code enforces a conservative configuration to reduce MV3 CSP/runtime issues:
- Forces WASM-only execution: `device: "wasm"` + `executionProviders: ["wasm"]`
- Disables advanced ONNX WASM features:
  - `numThreads = 1`
  - `simd = false`
  - `proxy = false`
- Explicitly disables WebGPU/WebNN backends if present
- Sets WASM asset paths to `assets/` inside the extension bundle
- Blocks loading `.jsep.mjs` modules (not supported in MV3) by overriding `globalThis.import` when present

### 4.3 Performance accounting
Embedding generation time is recorded via `recordEmbeddingTime(...)` (analytics).

---

## 5) Search pipeline (3 layers)

Search is coordinated by `src/background/search-coordinator.ts`.

### 5.1 Corpus flattening
Before searching, pages are “flattened” so you get the **most recent unique page per URL**:
- Iterate sessions from newest → oldest
- Insert first-seen URL into a `Map` and skip duplicates

This avoids duplicate results and biases toward recency.

### 5.2 Layer 3: ML ranking (cosine on title embeddings)
`src/background/layer3-ml-ranker.ts`
- Query embedding generated at search-time
- Score = cosine similarity (dot product, embeddings are normalized)
- Default `minScore = 0.3`
- Returns sorted results

If ML fails (e.g. model not available), search falls back.

### 5.3 Layer 2: TF‑IDF semantic search
`src/lib/layer2-semantic-search.ts`
- Tokenize: lowercase, split on non-alphanumerics
- Builds vocabulary over page titles
- Computes TF‑IDF vectors for docs and query
- Normalizes vectors and uses cosine similarity
- Default `minScore = 0.1`

### 5.4 Layer 1: keyword overlap
`src/lib/layer1-keyword-search.ts`
- Token overlap between query terms and title terms
- Score = `matchCount + densityBoost`, where `densityBoost = 1 / titleTokenCount`

### 5.5 Latency tracking
`executeSearch(...)` records latency (`recordSearchLatency(...)`).

---

## 6) Knowledge graph (page similarity graph + clustering)

Graph building lives in `src/lib/knowledge-graph.ts` and is orchestrated by the background (`src/background/index.ts`).

### 6.1 When graphs rebuild
- New page visits set a `graphNeedsRebuild` flag.
- `GET_GRAPH` rebuilds if needed.
- `EMBEDDINGS_COMPLETE` forces a session reload + rebuild.

### 6.2 Build inputs
Background flattens **all pages across all sessions**, then calls:

```ts
buildKnowledgeGraph(allPages, {
  similarityThreshold: 0.35,
  maxEdgesPerNode: 8,
  maxNodes: 500
})
```

Note: the library function has its own defaults, but the background currently overrides the similarity threshold to **0.35**.

### 6.3 Node filtering + dedupe
Inside `buildKnowledgeGraph(...)`:
- Utility pages are filtered out (heuristics in the module)
- Dedupes primarily by URL
- Also merges duplicates by title

### 6.4 Edge weighting (composite)
`calculateEdgeWeight(...)` combines several signals:
- embedding similarity weight: `0.7`
- keyword Jaccard overlap weight: `0.2`
- temporal proximity weight: `0.1`
- same-domain multiplier: `1.3`
- output weight is clamped to `[0, 1]`

Edges are created by selecting top‑K neighbors per node above the similarity threshold.

### 6.5 Clustering
`louvainClustering(...)` runs a simplified iterative assignment:
- max iterations: 10
- uses modularity gain to adjust communities
- renumbers communities at the end

### 6.6 Project graph
`buildProjectGraph(projects, allPages, 500)` builds a project-oriented graph used by the UI.

---

## 7) Project detection

There are two different approaches in this repo:

### 7.1 Real-time candidate detection (incremental)
`src/background/candidateDetector.ts`

This runs “per-page visit” and tracks **specific/deep resources** rather than homepages.

**Resource extraction** (`src/lib/resource-extractor.ts`):
- `homepage`: `/` or empty path
- `category`: 1 path segment
- `specific`: 2–3 path segments OR meaningful ID query params (`v`, `id`, `q`, `post`, etc.)
- `deep`: ≥4 path segments

**Production thresholds** (when `DEV_MODE = false`):
- Enforced today: `MIN_VISITS = 3`, `MIN_SCORE = 50`, `MAX_AGE_DAYS = 7`.
- Also defined (used indirectly via scoring / future tuning, but not hard-gated): `MIN_SESSIONS = 2`, `MIN_DURATION_HOURS = 1`.

**Score (0–100)** combines:
- visits: up to 40 points (sqrt scaling, saturates at 10 visits)
- sessions: up to 30 points (sqrt scaling, saturates at 5 sessions)
- resource count: up to 20 points (sqrt scaling, saturates at 3 resources)
- time span: up to 10 points (linear up to 24 hours)

**Snooze behavior**:
- each snooze increases `snoozeCount`
- effective visit threshold increases by `2 * snoozeCount`

Candidates are stored under `aegis-project-candidates`.

### 7.2 Batch project detection (session clustering)
`src/background/projectManager.ts`

This attempts to cluster sessions into projects based on **meaningful resource overlap** and temporal proximity.

Key thresholds:
- MIN_SESSIONS: 2
- MIN_RESOURCES: 2
- MIN_RESOURCE_VISITS: 2
- MIN_SCORE: 50
- MIN_DURATION_HOURS: 2
- MAX_DURATION_DAYS: 30

It filters routine resources (more than 10 visits/day) and scores candidates based on:
- specific resource count
- temporal consistency
- session count
- label consistency (optional boost)

Projects are persisted under `aegis-projects`.

### 7.3 Legacy project suggestion module
`src/background/projectSuggestions.ts` is explicitly marked **DISABLED** (kept for reference).

---

## 8) Similarity notifier ("you’ve seen something like this")

`src/background/similarity-notifier.ts`

When triggered, it:
- Looks for older pages (age ≥ 1 hour)
- Requires embeddings
- Excludes common utility domains (Google, YouTube, GitHub, etc.)
- Only considers pages with low visit count (`visitCount < 5`)
- Computes cosine similarity and keeps matches with score ≥ **0.4** (top 5)

Delivery strategy:
1) stores a persistent fallback under `pendingNotification`
2) tries `chrome.runtime.sendMessage({ type: "similar-pages-found", pages })` (sidepanel)
3) sends `SHOW_SIMILAR_PAGES` to the indicator hub (content script) if a `tabId` is available

---

## 9) Focus Mode (blocking + tab sweep)

`src/background/focusModeManager.ts`

- Focus mode state stored under `focus-mode-state`
- Blocking is implemented using `chrome.declarativeNetRequest.updateDynamicRules(...)`
- When focus mode activates, it also **sweeps and closes already-open tabs** matching blocked domains.

Rules are generated from the blocklist (`src/background/blocklistStore.ts`) and filtered by enabled categories.

---

## 10) Reminders (projects)

Reminders are managed by `src/background/reminderManager.ts`.

- Uses `chrome.alarms`
- Alarm names are prefixed with `project-reminder-`
- Background listens for alarms and dispatches `handleReminderAlarm(...)`
- UI triggers operations via message types:
  - `SET_PROJECT_REMINDER`, `CANCEL_PROJECT_REMINDER`, `SNOOZE_REMINDER`, `DISMISS_REMINDER`, `DISMISS_SNOOZE`

---

## 11) COI (Cognitive Overload Indicator)

### 11.1 Behavior capture
`src/background/ephemeralBehavior.ts` tracks:
- `tabSwitchCount`
- `idleTransitions` (idle→active transitions after 30s inactivity)
- scroll:
  - `maxDepthBucket`: `top | middle | bottom`
  - `burstCount`

### 11.2 COI scoring
`src/lib/coi.ts`

- COI weights are loaded from `coi-weights` or defaults
- Weights are normalized to sum to 1 within page/session scope

Session COI features include:
- normalized tab switching rate
- idle transitions rate
- scroll burst per page
- shallow depth score (`top=1.0`, `middle=0.3`, `bottom=0`)
- dwell variance (with IQR outlier filtering)
- domain churn, revisit, duration, foreground drop

A dampening factor reduces false positives in the first 5 minutes.

### 11.3 Broadcasting updates
In `src/background/index.ts`:
- `broadcastCoiUpdate()` runs every 5 seconds and on tab/window focus changes.
- It sends `COI_UPDATE` to registered session listener tabs.

### 11.4 Optional COI alerts (in-page)
Also in `src/background/index.ts`:
- Alerts are gated behind `settings.developer.showCoiNotifications`
- Default threshold: `0.7` (if not set)
- Default cooldown: 5 minutes (if not set)
- When fired, it targets the **most recently accessed regular tab** (non `chrome://`, non extension pages)
- It attempts to message the existing content script with `COI_ALERT`
- It **does not** attempt runtime injection into restricted pages

---

## 12) Background message API (what the UI calls)

All of these are handled in `src/background/index.ts` via `chrome.runtime.onMessage.addListener`.

### 12.1 Session + graph
- `CONTENT_SCRIPT_READY`
- `GET_SESSIONS`
- `LISTEN_FOR_SESSIONS`
- `GET_GRAPH`
- `GET_PROJECT_GRAPH`
- `REFRESH_GRAPH`
- `EMBEDDINGS_COMPLETE`
- `IMPORT_HISTORY`

### 12.2 Search
- `SEARCH_QUERY`

### 12.3 Behavior / COI
- `GET_BEHAVIOR_STATE`
- `SCROLL_DEPTH_UPDATE`
- `SCROLL_BURST_DETECTED`

### 12.4 Labels / learning
- `GET_LABELS`
- `GET_LABEL_SUGGESTION`
- `UPDATE_SESSION_LABEL`
- `UPDATE_SESSION_NAME`
- `ADD_LABEL`
- `DELETE_LABEL`

### 12.5 Session deletion
- `DELETE_PAGE_FROM_SESSION`
- `DELETE_SESSION`

### 12.6 Projects
- `GET_PROJECTS`
- `DETECT_PROJECTS`
- `CREATE_PROJECT`
- `ADD_PROJECT`
- `UPDATE_PROJECT`
- `DELETE_PROJECT`
- `ADD_SITE_TO_PROJECT`
- `DISMISS_PROJECT_SUGGESTION`
- `OPEN_PROJECT_IN_TAB_GROUP`
- `NEVER_SHOW_PROJECT_NOTIFICATION`

### 12.7 Project candidates
- `GET_READY_CANDIDATES`
- `DISMISS_CANDIDATE`
- `SNOOZE_CANDIDATE`
- `PROMOTE_CANDIDATE`

Dev-only helpers:
- `CREATE_TEST_CANDIDATE`
- `CLEAR_ALL_CANDIDATES`
- `GET_CANDIDATES_COUNT`

### 12.8 Sidepanel open routing
- `OPEN_SIDEPANEL_TO_PROJECTS`

### 12.9 Focus mode + blocklist
- `TOGGLE_FOCUS_MODE`
- `GET_FOCUS_STATE`
- `TOGGLE_CATEGORY`
- `SET_ENABLED_CATEGORIES`
- `GET_BLOCKLIST`
- `ADD_BLOCKLIST_ENTRY`
- `UPDATE_BLOCKLIST_ENTRY`
- `DELETE_BLOCKLIST_ENTRY`
- `UPDATE_CATEGORY_STATES`
- `IMPORT_BLOCKLIST`
- `EXPORT_BLOCKLIST`

### 12.10 Reminders
- `SET_PROJECT_REMINDER`
- `CANCEL_PROJECT_REMINDER`
- `SNOOZE_REMINDER`
- `DISMISS_REMINDER`
- `DISMISS_SNOOZE`

### 12.11 Settings + analytics + data ops
- `GET_SETTINGS`
- `UPDATE_SETTINGS`
- `RESET_ALL_SETTINGS`
- `GET_DETAILED_ANALYTICS`
- `CLEAR_ALL_DATA`
- `CLEAR_ALL_PROJECTS`
- `CLEAR_ALL_SESSIONS`
- `EXPORT_ALL_DATA`
- `IMPORT_DATA` (note: sessions import is not implemented; labels/projects/settings are written)
- `GET_ONBOARDING_PROGRESS`

---

## 13) Storage keys (quick reference)

### 13.1 IndexedDB
- DB: `aegis-sessions`
- store: `sessions`
- key: `all-sessions`

### 13.2 chrome.storage.local
Common keys used across the system:

**Onboarding / history import**
- `history-imported`
- `onboarding-embeddings-in-progress`
- `onboarding-embeddings-complete`

**Settings & consent**
- `aegis-settings`
- `aegis-consent`

**Sessions (legacy/compat / additional)**
- `aegis-sessions` (used by `CLEAR_ALL_SESSIONS` handler; sessions are primarily in IndexedDB)

**Graph / UI state**
- `manualLinks` (graph UI manual edge persistence)
- `sidepanel-active-tab`
- `sidepanel-onboarding`

**Notifications**
- `pendingNotification`
- `project-notification-blocklist`

**Projects and candidates**
- `aegis-projects`
- `aegis-project-candidates`

**Focus mode**
- `focus-mode-state`
- `focus-mode-blocklist`

**Learning + labels**
- `aegis-context-learning`
- `aegis-labels`

**COI**
- `coi-weights`

---

## 14) How to run, debug, and evaluate

### 14.1 Dev / build
From `README.md`:
- `pnpm dev` → outputs `build/chrome-mv3-dev/`
- `pnpm build` → outputs `build/chrome-mv3-prod/`

Load unpacked at `chrome://extensions/`.

### 14.2 Debugging tips
- Background logs appear in the service worker DevTools.
- Content script logs appear in the target tab’s DevTools console.
- Graph + sessions can be empty after SW restart; handlers reload from IndexedDB on demand.

### 14.3 Evaluation scripts
The repo has TS eval scripts (see `README.md` and `docs/evaluation.md`). Examples:
- `pnpm exec ts-node evaluation/knowledge-graph-eval.ts`
- `pnpm exec ts-node evaluation/layer1_2.ts`
- `pnpm exec ts-node evaluation/project_detection.ts`

---

## 15) Interview-ready “tradeoffs” you can call out

- **Local-first**: no cloud sync by default; privacy-friendly; avoids accounts.
- **MV3 robustness**: session reload + no runtime injection keeps things stable.
- **Search fallback**: ML is best when embeddings exist; TF‑IDF/keyword provide resilience.
- **Graph quality**: composite edge weighting reduces false edges vs pure embedding similarity.
- **Project detection**: resource-based approach avoids "everything on a domain becomes one project"; explicit skip of home/category pages.

---

## 16) What’s explicitly not “active”

- `src/background/projectSuggestions.ts` is labeled as legacy and disabled because it was noisy.
- `IMPORT_DATA` currently does not import sessions (commented as TODO); it imports labels/projects/settings only.
