# FutureEngineer — Multi‑Agent Desktop Agents (Cross‑Platform Desktop App)

> **Status:** MVP Spec v0.1 (Locked decisions included)
>
> **Platforms:** Windows 10/11, macOS 13+
>
> **Packaging:** exe (NSIS) & dmg (notarized)

A cross‑platform desktop application where anyone can create AI agents that run scheduled or on‑demand tasks, collaborate with other agents, control the local computer (apps, browser, terminal), ask humans for help via WhatsApp, and remember project history across the full lifecycle.

---

## MVP Decisions (Locked)

- **Product name:** **FutureEngineer**
- **Distribution:** Website download only (MVP)
- **AI providers:** Claude + OpenAI (pluggable). Default agent can route to either.
- **Default agent:** **General Assistant** with permissioned full OS access
- **Human‑loop:** **Meta WhatsApp Cloud API** with **cloud relay (default)**; local relay optional
- **Runtime mode:** **Foreground only** (runs when the user is logged in)
- **Approval policy:** Only **file deletion outside the project directory** requires WhatsApp approval; all other actions proceed automatically
- **Filesystem scope:** Ask for project root on first run; agents may create folders anywhere after explicit confirmation (e.g., D:, Desktop)
- **Scheduling UI:** **One‑shot**, **Interval**, **Cron**, plus **Natural language** (e.g., “every weekday 9am”)
- **Concurrency:** **Per‑agent 1**, **Global 10**
- **Logs retention:** Keep last **30 days** (older logs pruned); artifacts retained unless user cleans up
- **Vector DB:** **LanceDB** (free, embedded)
- **Embeddings default:** Project files, agent chats, relevant webpages, artifacts, and key logs
- **Secrets storage:** Encrypted file in app data
- **Auto‑updates:** GitHub Releases
- **Git provider:** GitHub by default
- **Timezone:** Set in App Settings; **Language:** English
- **Templates (MVP):** General Assistant, Bug‑Fixer, Presentation‑Maker, Email Assistant, Frontend Engineer, Backend Engineer, Database Engineer, Full‑Stack Coder

---

## 1) Product Goals

- **Human‑like work**: Agents operate apps, browsers, files, and terminals like a person would.
- **Multi‑agent orchestration**: Multiple agents run in parallel, hand off work, and query each other.
- **Scheduling & control**: Run now, schedule later, pause/resume/stop, retries, priorities, and concurrency limits.
- **Human‑in‑the‑loop**: Agents can message a human on WhatsApp when clarification or approval is needed and continue from the reply.
- **Project memory**: Short‑term, long‑term (vector/semantic), and episodic logs for full traceability and retrieval.
- **Local‑first & safe**: Default local processing, explicit capability permissions, audit logs, easy export.

**Success criteria (MVP):**

- Create an agent with tools and permissions, define a task, run it, and observe step‑by‑step logs and artifacts.
- Agent opens the browser, performs actions (e.g., search + scrape + save file), asks a question via WhatsApp, resumes on reply, and delivers a document.
- Tasks can be scheduled, paused/resumed, and retried; all runs are stored with memory and artifacts.

---

## 2) Architecture Overview

**UI Shell:** Electron (React + TypeScript)

**Privileged Runtime:** Node.js (TypeScript) service with native helpers

- **Windows:** Rust addon (Win32/UIAutomation), PowerShell; low‑level input
- **macOS:** Swift helper (Accessibility API), AppleScript/JXA bridge

**Automation:** Playwright for browser; native input + OCR (OpenCV/Tesseract or RapidOCR) for arbitrary apps; app‑specific adapters where possible.

**Orchestrator:** Graph‑based agent runner (LangGraph/Autogen style) with tool‑calling and state machine per task.

**Scheduler/Queue:** Local persisted queue (SQLite) with worker\_threads; backoff/retry/priority; cron parser.

**Memory:** SQLite (metadata) + LanceDB (vectors) + file artifacts; background indexers for embeddings.

**Human Loop:** WhatsApp Cloud API webhook + outbound sender via **cloud relay**; local relay optional.

**Security:** Capability permissions; approval gates; audit trail; signed binaries.

```
[Electron UI]
   │ IPC
   ▼
[Local Orchestrator Service (Node)] ───► [Scheduler/Queue]
   │            │                       
   │            ├──► [Tools: Browser/Terminal/File/Git/Email/Docs]
   │            ├──► [Native Helpers: Rust (Win) / Swift (mac)]
   │            ├──► [Memory: SQLite + LanceDB + Artifacts]
   │            └──► [WhatsApp Service: Cloud Relay + Webhook]
   ▼
[Audit Log + Timeline]
```

---

## 3) Components & Responsibilities

### 3.1 Electron UI (Renderer)

- Agent Studio (create/edit agents, permissions, tools)
- Task Board (runs, schedules, status, controls)
- Timeline & Artifacts viewer (screenshots, files)
- Memory browser & retrieval queries
- Settings (API keys, WhatsApp, updates, permissions, timezone, language)

### 3.2 Main Process & IPC Contracts

- Secure IPC calls to the orchestrator service
- File dialogs; OS permissions bootstrap (mac TCC prompts, Win UIAccess notes)

### 3.3 Orchestrator Service (Node)

- Agent graph execution & tool‑calling
- State persistence per step & resumption
- Inter‑agent messaging bus (in‑proc pub/sub)
- Error handling, retries, backoff, cancellation

### 3.4 Scheduler/Queue

- One‑shot / Interval / Cron / Natural Language
- Priority, concurrency, rate limits
- Durable resume after crash or reboot

### 3.5 Tools (Initial Set)

- **Browser** (Playwright): open tab, navigate, extract, fill forms, download
- **Desktop Control**: focus window, type, click, hotkeys, screenshots
- **Terminal**: run commands, capture stdout/stderr, exit code
- **File**: read/write/edit files, templates, zips
- **Git**: clone/pull/branch/commit/push (GitHub default; others configurable)
- **HTTP**: GET/POST with headers/auth
- **Email**: ask via WhatsApp which method to use per run (webmail automation like a human, or SMTP/Google Workspace). No approval by default.
- **Slides/Docs**: from templates (python‑pptx/LibreOffice), optional Google Slides API
- **Vision/OCR**: image‑to‑text for UI when accessibility is limited

### 3.6 Memory Service

- SQLite schema for agents, tasks, runs, events, artifacts, messages, credentials
- LanceDB for embeddings (code, docs, chats, web pages)
- Background indexers on file change; embeddings stored for project files, chats, webpages, artifacts, and key logs

### 3.7 WhatsApp Service

- Outbound message API via cloud relay (recommended: Cloudflare Workers)
- Inbound webhook handler → maps to task state & resumes execution
- Local relay option using Cloudflare Tunnel/Ngrok for direct desktop exposure (optional)

---

## 4) Data Model (Initial)

### 4.1 SQLite Tables (draft)

- `agents(id, name, role, system_prompt, permissions_json, enabled_tools_json, created_at)`
- `projects(id, name, config_json, created_at)`
- `tasks(id, agent_id, project_id, title, schedule_cron, params_json, status, priority, created_at)`
- `task_runs(id, task_id, started_at, finished_at, status, error, result_json)`
- `events(id, task_run_id, agent_id, type, payload_json, created_at)`
- `artifacts(id, task_run_id, kind, path, hash, meta_json, created_at)`
- `messages(id, project_id, from_type, from_id, to_type, to_id, text, meta_json, created_at)`
- `memories(id, project_id, agent_id, kind, text, embedding_vector_ref, meta_json, created_at)`
- `credentials(id, provider, label, encrypted_json, created_at)`
- `settings(id, key, value_json)`

### 4.2 Vector Store (LanceDB — chosen for MVP)

- `embeddings(project_id, agent_id, content, mime, source, path, artifact_id, event_id, created_at, vector)`

---

## 5) Scheduling Semantics

- **One‑shot**: run at specific time
- **Interval**: every N minutes/hours/days
- **Cron**: full CRON syntax
- **Natural language**: e.g., “every weekday at 9am” (compiled to cron under the hood)
- **Controls**: pause/resume/stop, retries (exponential backoff), max attempts, timeout per step and per run
- **Priorities**: numeric priority
- **Concurrency**: **Per‑agent 1** (default), **Global 10** (configurable)
- **Logs retention**: keep last **30 days**; older logs auto‑pruned

---

## 6) Human‑in‑the‑Loop (WhatsApp)

### 6.1 Provider

- **Meta WhatsApp Cloud API** (first‑party). Cloud relay recommended (Cloudflare Workers + Durable Object/KV)

### 6.2 Flow

1. Agent reaches a **Question** node.
2. System sends a WhatsApp message with context and (optionally) a short link.
3. Incoming WhatsApp webhook hits the cloud relay; reply is persisted and queued for the desktop app.
4. The desktop app polls/streams from the relay and resumes the exact step with the reply content.

### 6.3 Notes

- Message templates stored in SQLite; variables injected from task state.
- Local relay mode available for offline or advanced users.

---

## 7) Security & Permissions

- **Capability model per agent**:
  - Filesystem scopes (allow/deny paths)
  - Network domain allowlist
  - Command allowlist (terminal)
  - **Deletion outside project dir requires WhatsApp approval**
  - Max runtime & memory
- **Audit log**: immutable events with hashes of artifacts
- **Dry‑run** mode & preview diffs for file changes
- **Signed builds**: Windows Authenticode; macOS notarization

---

## 8) Dev Setup

**Prerequisites**

- Node.js 20+
- Python 3.11+ (for pptx/docs tooling)
- Git
- **Windows:** Visual Studio Build Tools + Rust (stable toolchain)
- **macOS:** Xcode Command Line Tools; CocoaPods if needed

**Clone & Install**

```bash
git clone <repo>
cd futureengineer
pnpm i   # or npm/yarn
```

**Environment** (`.env.example`)

```
# Models
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
MODEL_DEFAULT=openai:gpt-4.1

# WhatsApp (Meta Cloud API)
WHATSAPP_PHONE_NUMBER_ID=
WHATSAPP_BUSINESS_ACCOUNT_ID=
WHATSAPP_ACCESS_TOKEN=
WHATSAPP_VERIFY_TOKEN=

# Cloud Relay (recommended)
RELAY_BASE_URL=
RELAY_AUTH_TOKEN=

# Storage
DATA_DIR=./data
SQLITE_PATH=./data/app.db
LANCEDB_DIR=./data/lancedb
LOG_RETENTION_DAYS=30

# Updates / Telemetry
AUTO_UPDATE=true
```

**Run in Dev**

```bash
pnpm dev
```

**Build Packages**

```bash
# Windows (NSIS)
pnpm build:win
# macOS (dmg, notarized)
pnpm build:mac
```

---

## 9) Packaging & Signing

- **electron‑builder** for NSIS (exe) and dmg.
- Windows code signing certificate (EV recommended), timestamping.
- macOS Developer ID Application cert + **notarization** (altool/notarytool) and stapling.
- Auto‑update via electron‑updater (GitHub Releases).
- **Distribution:** Website download only for MVP.

---

## 10) Example: Hello‑World Agent

**Goal:** Open browser → search query → save results → ask human whether to email them → if YES, draft email and proceed without approval.

High‑level steps:

1. Browser tool: navigate to search, collect top N results → artifact.json
2. Question node: “Send summary to your email?” via WhatsApp
3. If YES: generate markdown summary → convert to PDF → email tool (no approval required)
4. Store artifacts, events, and memory embeddings; mark run complete

---

## 11) Initial Agent Templates (MVP)

- **General Assistant** (default): full OS access within permissions; orchestrates others
- **Bug‑Fixer**: clone repo, run tests, reproduce issue, propose patch, open PR
- **Presentation‑Maker**: turn content brief into slides with template, export PDF/PPTX
- **Email Assistant**: draft personalized emails from CSV + template; no approval required
- **Frontend Engineer**
- **Backend Engineer**
- **Database Engineer**
- **Full‑Stack Coder**

---

## 12) Roadmap

- Vision‑based UI understanding (layout diffs, responsiveness checks)
- Template marketplace for agent recipes
- Team sharing + cloud relay enhancements
- Local LLM support via Ollama
- App‑specific adapters (Office, Adobe, Figma via plugins)
- Policy packs & enterprise audit exports (SOC2‑friendly)

---

## 13) Risks & Mitigations

- **OS permission prompts:** first‑run setup wizard; signed helpers to reduce friction.
- **Brittle UI automation:** prefer accessibility APIs; OCR fallback; retries and app‑specific adapters.
- **Security:** strict capability model, immutable audit log, deletion‑approval policy.
- **WhatsApp onboarding:** set up early; cloud relay default; local relay wizard later.

---

## 14) Contribution Guide (Draft)

- Conventional Commits; PR templates; codeowners.
- Linting/formatting: ESLint + Prettier.
- Tests: vitest for TS; integration tests with Playwright.

---

## 15) License

TBD (proprietary or dual‑license). Add third‑party notices for dependencies.

---

## 16) Appendix: IPC Contracts (Draft)

```ts
// Renderer → Orchestrator
interface CreateAgent { name: string; role: string; tools: string[]; permissions: Record<string, any>; }
interface CreateTask { agentId: string; title: string; schedule?: string; params?: any; priority?: number; }
interface ControlTask { taskId: string; action: 'pause'|'resume'|'stop'|'retry'; }

// Orchestrator → Renderer events
interface TaskEvent { taskRunId: string; level: 'info'|'warn'|'error'; message: string; payload?: any; }
```

```http
# Webhook (WhatsApp via Cloud Relay)
POST /webhook/whatsapp
Authorization: Bearer <RELAY_AUTH_TOKEN>
{
  "from": "+8801xxxx",
  "text": "Proceed and send emails"
}
```

---

**Notes**

- Product name fixed: **FutureEngineer**.
- This README is the living spec for v0.1; we’ll evolve it as we implement.

---

## Project & File Structure

This repo uses a **pnpm workspace** layout so we can ship a clean Electron app while keeping the orchestrator, tools, and native helpers modular.

```
futureengineer/
├─ apps/
│  └─ desktop/                      # Electron app (main, preload, renderer)
│     ├─ package.json
│     ├─ src/
│     │  ├─ main/                  # Electron main (IPC, updater)
│     │  ├─ preload/               # secure context bridge
│     │  └─ renderer/              # React UI (Agent Studio, Task Board, Settings)
│     ├─ build/                    # builder configs (icons, nsis, dmg)
│     └─ vite.config.ts            # or webpack.config.ts
│
├─ packages/
│  ├─ orchestrator/                 # Agent graph runner + scheduler
│  │  ├─ src/graph/                # nodes/edges, supervisor, handoffs
│  │  ├─ src/scheduler/            # queue, cron, retry/backoff
│  │  ├─ src/agents/               # General, Bug‑Fixer, etc.
│  │  ├─ src/tools/                # browser, terminal, file, git, email, slides
│  │  ├─ src/whatsapp/             # cloud relay client, handlers
│  │  ├─ src/memory/               # API over SQLite + LanceDB
│  │  └─ src/logging/
│  ├─ memory/                       # LanceDB wrapper + indexers
│  ├─ tools-browser/                # Playwright helpers (profiles, downloads)
│  ├─ tools-system/                 # terminal, file I/O, git, HTTP
│  ├─ relay-sdk/                    # SDK to talk to Cloud Relay (fetch/stream)
│  ├─ native-win/                   # Rust addon (UIAutomation, input)
│  ├─ native-mac/                   # Swift helper (AX API, AppleScript/JXA)
│  └─ shared/                       # shared types & utilities
│
├─ resources/                       # icons, pptx templates, email templates
├─ scripts/                         # dev/build/release scripts
├─ data/                            # local DB, LanceDB, artifacts (gitignored)
├─ tests/                           # unit/integration tests
├─ .github/workflows/               # CI: build, sign, notarize, release
├─ package.json                     # root (workspaces, scripts)
├─ pnpm-workspace.yaml
├─ .env.example
└─ README.md
```

**Notes**

- The **desktop** app and **orchestrator** communicate via secure IPC.
- **Artifacts & logs** live under `./data` with a 30‑day retention policy for logs.
- **Native helpers** are optional for the first MVP; we can stub input control via `nut.js` and iterate.

---

## Environment Setup (Detailed)

### Common

1. Install **Node.js 20+** and **pnpm**: `npm i -g pnpm`
2. Install **Python 3.11+**; ensure `python` & `pip` are on PATH
3. Install **Git** (enable Git Credential Manager if desired)
4. Clone the repo and create local data dirs: `mkdir -p data`
5. Copy ``** → **`` and fill in:
   - `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`
   - WhatsApp: `WHATSAPP_PHONE_NUMBER_ID`, `WHATSAPP_BUSINESS_ACCOUNT_ID`, `WHATSAPP_ACCESS_TOKEN`, `WHATSAPP_VERIFY_TOKEN`
   - Cloud relay: `RELAY_BASE_URL`, `RELAY_AUTH_TOKEN`
6. Install Playwright browsers: `pnpm exec playwright install`

### Windows

1. Install **Visual Studio Build Tools** (C++ workload + Windows 10/11 SDK)
2. Install **Rust (MSVC)**: `winget install Rust.Rustup` → `rustup default stable-x86_64-pc-windows-msvc`
3. (Optional OCR) `winget install tesseract-ocr`
4. Reboot to finalize PATH/toolchains

### macOS

1. Install **Xcode Command Line Tools**: `xcode-select --install`
2. (Optional) **Homebrew** then `brew install rust python@3.11 tesseract`
3. On first app launch, grant **Accessibility** and **Screen Recording** permissions

---

## Steps to Run (Dev)

```bash
# 0) One-time browser install for Playwright
pnpm exec playwright install

# 1) Install workspace deps
pnpm i

# 2) (Optional) Build packages once
pnpm -r build

# 3) Start orchestrator + Electron together
pnpm dev
```

What `pnpm dev` does:

- starts the **orchestrator** (ts-node/nodemon) with hot reload
- boots the **Electron** app (renderer dev server + main process)
- writes logs into `./data/logs` and streams to console

Run individually (advanced):

```bash
pnpm --filter @futureengineer/orchestrator dev
pnpm --filter @futureengineer/desktop dev
```

First‑run wizard will:

- Ask for **project root** location
- Set **timezone** (user‑selectable) and confirm **language** (English)
- Validate **API keys** and **WhatsApp Cloud** credentials
- Test **Cloud Relay** connectivity

---

## Detailed TODO Checklist (MVP v0.1)

**Legend:**  
- [ ] = not started · [~] = in progress · [x] = done · (P1/P2/P3) = priority · (A/C) = acceptance criteria

### Phase 0 — Repo & CI Bootstrap (P1)
- [ ] Monorepo init with **pnpm workspaces** (`apps/desktop`, `packages/*`, `shared`)  
  (A) `pnpm i` installs; `pnpm dev` starts Electron + orchestrator
- [ ] Lint/format (**ESLint + Prettier**)  
  (A) `pnpm lint`/`pnpm format:check` green locally and in CI
- [ ] Unit tests (**vitest**) + a sample test  
  (A) `pnpm test` green on Win/mac
- [ ] GitHub Actions: build on PR; release on tag (unsigned artifacts)  
  (A) CI produces NSIS/dmg artifacts

### Phase 1 — Storage & Schema (P1)
- [ ] SQLite schema (Prisma recommended) for agents, tasks, runs, events, artifacts, messages, settings, credentials  
  (A) `prisma migrate dev` creates DB; CRUD validated
- [ ] Log retention job (**30 days**)  
  (A) Old logs are pruned on a daily schedule
- [ ] Artifacts directory + manifest  
  (A) Files hashed; manifest rows stored and linked to runs

### Phase 2 — Orchestrator & Scheduler (P1)
- [ ] Task state machine: resume, cancel, failure, backoff  
  (A) Simulated tasks drive deterministic transitions
- [ ] Queue + workers via **worker_threads**  
  (A) Per‑agent=1 and global=10 concurrency enforced
- [ ] Schedules: **one‑shot / interval / cron / natural language**  
  (A) All schedules trigger at correct times (timezone aware)

### Phase 3 — Model Provider Layer (P1)
- [ ] OpenAI + Anthropic clients with retry/backoff; streaming  
  (A) Provider switch per agent works; token/latency metrics captured
- [ ] Tool‑calling bridge (functions/tools)  
  (A) Model can request tools; orchestrator executes and returns results

### Phase 4 — Core Tools (P1)
- [ ] **Browser** (Playwright): open, navigate, download, scrape helpers  
  (A) Headed Chromium runs; downloads saved to artifacts
- [ ] **Terminal**: PowerShell/zsh exec with timeout and env sandbox  
  (A) Exit code + stdout/stderr captured per event
- [ ] **File I/O**: read/write/edit, templates, safe path joins  
  (A) Writes restricted to allowed scopes
- [ ] **HTTP**: GET/POST with headers/auth; JSON helper  
  (A) Propagate HTTP errors cleanly
- [ ] **Git**: clone/pull/branch/commit/push (GitHub)  
  (A) PAT via encrypted secrets; PR opened on sample repo
- [ ] **Email**: runtime choice (webmail automation or SMTP)  
  (A) Method selected via WhatsApp prompt; message‑ID logged
- [ ] **Slides/Docs**: md → PPTX/PDF via python‑pptx/LibreOffice  
  (A) Template applied; artifact written

### Phase 5 — Native Helpers & Input Control (P2)
- [ ] Stub via **nut.js** for keyboard/mouse/screenshots  
  (A) Cross‑platform clicks/typing/screenshots reliable
- [ ] Windows **Rust** addon (UIAutomation, low‑level input)  
  (A) Builds on CI; smoke tests pass
- [ ] macOS **Swift** helper (AX API, AppleScript/JXA)  
  (A) Permissions wizard triggers; focused app actions succeed

### Phase 6 — WhatsApp Cloud Relay (P1)
- [ ] Cloudflare Workers endpoint (verify token, signature, rate‑limit)  
  (A) Receives webhooks; 200 OK within SLA
- [ ] Durable storage (KV/Durable Object) for pending replies  
  (A) At‑least‑once delivery to desktop
- [ ] Desktop **relay‑sdk** (long‑poll or SSE/WebSocket)  
  (A) E2E: agent question → WhatsApp → reply → resume

### Phase 7 — Memory Pipeline (P1)
- [ ] **LanceDB** setup + embeddings index  
  (A) Upsert/search with metadata filters (project/agent)
- [ ] Indexers: watch project root; embed new/changed files, artifacts, key logs  
  (A) Background processing doesn’t block UI
- [ ] Recall APIs: “what did I do yesterday?”, “open last artifact for task X”  
  (A) Deterministic retrieval in tests

### Phase 8 — Electron UI (P1)
- [ ] **Agent Studio**: create/edit agent, pick tools, set permissions  
  (A) Validates and persists to DB
- [ ] **Task Board**: list, filter, controls (start/pause/resume/stop/retry)  
  (A) Real‑time status via IPC
- [ ] **Run Detail**: event stream, artifacts viewer, screenshot gallery  
  (A) Streams append smoothly; pagination on large logs
- [ ] **Settings**: API keys, WhatsApp, timezone, language, updates  
  (A) Secrets encrypted; test buttons validate credentials

### Phase 9 — Security & Permissions (P1)
- [ ] Capability enforcement (filesystem scopes, network allowlist, command allowlist)  
  (A) Violations blocked; clear error + audit entry
- [ ] **Deletion guard**: deleting outside project dir ⇒ WhatsApp approval  
  (A) Requires affirmative reply; denial aborts step
- [ ] Audit trail: immutable events + artifact hashes; export  
  (A) JSON/CSV export works
- [ ] **Dry‑run** toggle; diff previews for file writes  
  (A) No changes applied in dry‑run

### Phase 10 — Agent Templates (P2)
- [ ] **General Assistant** (default orchestrator)  
  (A) Can call other agents; respects permissions
- [ ] **Bug‑Fixer**  
  (A) Repro test → patch → PR on sample repo
- [ ] **Presentation‑Maker**  
  (A) Given brief → PPTX/PDF with template
- [ ] **Email Assistant**  
  (A) CSV inputs → drafts/sends via chosen method
- [ ] **Frontend/Backend/Database/Full‑Stack Engineer**  
  (A) Scaffolds tasks; runs commands; produces artifacts

### Phase 11 — Example Flows & E2E (P1)
- [ ] Hello‑World flow (search → WhatsApp decision → email)  
  (A) Green on Win & mac
- [ ] Failure/resume scenarios (network drop, model error)  
  (A) Run resumes from last safe step

### Phase 12 — Packaging & Updates (P1)
- [ ] **electron‑builder** configs (NSIS, dmg)  
  (A) Local builds produce installers
- [ ] Signing (Win) & notarization (mac) in CI  
  (A) Signed artifacts uploaded on tag
- [ ] Auto‑updates via GitHub Releases  
  (A) Delta updates apply; rollback path documented

### Phase 13 — QA, Telemetry, Docs (P2)
- [ ] Crash reporting (local file; optional consent to share)  
  (A) Errors captured with stack traces
- [ ] Performance baselines (cold start, task latency, memory)  
  (A) Metrics documented per platform
- [ ] User Guide + Troubleshooting (`/docs`)  
  (A) First‑run wizard & common fixes covered

### Phase 14 — Launch Checklist (P1)
- [ ] Security pass (permissions, approvals, logs)  
  (A) Checklist completed
- [ ] Licensing & third‑party notices  
  (A) `NOTICE` file present
- [ ] Release notes for v0.1  
  (A) CHANGELOG entry; download links

---

## Definition of Done (MVP)

A user can install FutureEngineer, create an agent, schedule a task, watch it run with OS control, receive a WhatsApp question, reply, and see the task finish with artifacts saved. All major features respect permissions, logs retain 30 days, and updates are delivered via GitHub Releases.

---

## Release Checklist

- [ ] CI green on main; tests passing
- [ ] Version bumped; CHANGELOG updated
- [ ] Windows EXE signed; mac DMG notarized & stapled
- [ ] Installers uploaded to GitHub Releases; auto-update feed valid
- [ ] Smoke test on clean Windows & mac VMs
- [ ] README updated; .env.example current
