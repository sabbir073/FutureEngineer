# Multi‑Agent Desktop Agents (Cross‑Platform Desktop App)

> **Status:** Draft spec for v0.1 (MVP)
>
> **Platforms:** Windows 10/11, macOS 13+
>
> **Packaging:** exe (NSIS) & dmg (notarized)

A cross‑platform desktop application where anyone can create AI agents that run scheduled or on‑demand tasks, collaborate with other agents, control the local computer (apps, browser, terminal), ask humans for help via WhatsApp, and remember project history across the full lifecycle.

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

## 2) Key Features

### 2.1 Agent Studio

- Create/edit agents: name, role, objectives, prompts.
- Tool selection & **capability‑based permissions** (filesystem scopes, network allowlists, command allowlists, email send approval, etc.).
- Memory profile: enable/disable long‑term memory, per‑project scratch space.

### 2.2 Task Orchestrator

- Run **one‑shot**, **interval**, or **cron** schedules.
- Controls: start, pause, resume, stop, retry, set priority, set concurrency per agent.
- Backoff policies & failure thresholds.
- Artifacts & logs per task run.

### 2.3 Human‑in‑the‑Loop (WhatsApp)

- Outbound agent questions delivered to WhatsApp.
- Inbound replies resume the exact graph state.
- Approval gates (e.g., “send these 25 emails?”) with YES/NO or custom responses.

### 2.4 OS & App Control

- Browser automation (Playwright/CDP).
- Desktop control (keyboard/mouse, window focus, screenshots).
- Terminal/PowerShell/zsh execution with captured output.
- File I/O, Git operations, HTTP fetch, email send, slide/doc generation.

### 2.5 Memory System

- **Short‑term**: rolling context per agent + task.
- **Long‑term (semantic)**: vector store of files, commits, chat, webpages, artifacts.
- **Episodic (timeline)**: append‑only event log with timestamps and links to artifacts.
- Memory search & retrieval for planning, debugging, and “what’s next”.

### 2.6 Safety & Audit

- Fine‑grained capability grants per agent.
- Dry‑run mode; approval‑required tools.
- Immutable audit log; artifact hashing.

### 2.7 Offline‑first & Updates

- Local SQLite/LanceDB by default; optional cloud sync later.
- Auto‑update with code signing (Win) & notarization (macOS).

---

## 3) Architecture Overview

**UI Shell**: Electron (React + TypeScript)

**Privileged Runtime**: Node.js (TypeScript) service with native helpers

- **Windows**: Rust addon (Win32/UIAutomation), PowerShell, low‑level input
- **macOS**: Swift helper (Accessibility API), AppleScript/JXA bridge

**Automation**: Playwright for browser; native input + OCR (OpenCV/Tesseract or RapidOCR) for arbitrary apps; app‑specific adapters where possible.

**Orchestrator**: Graph‑based agent runner (LangGraph/Autogen style) with tool‑calling and state machine per task.

**Scheduler/Queue**: Local persisted queue (SQLite) with worker\_threads; backoff/retry/priority; cron parser.

**Memory**: SQLite (metadata) + LanceDB (vectors) + file artifacts; background indexers for embeddings.

**Human Loop**: WhatsApp Cloud API webhook + outbound sender (local relay if needed).

**Security**: Capability permissions; approval gates; audit trail; signed binaries.

```
[Electron UI]
   │ IPC
   ▼
[Local Orchestrator Service (Node)] ───► [Scheduler/Queue]
   │            │                       
   │            ├──► [Tools: Browser/Terminal/File/Git/Email/Docs]
   │            ├──► [Native Helpers: Rust (Win) / Swift (mac)]
   │            ├──► [Memory: SQLite + LanceDB + Artifacts]
   │            └──► [WhatsApp Service: Webhook + Sender]
   ▼
[Audit Log + Timeline]
```

---

## 4) Components & Responsibilities

### 4.1 Electron UI (Renderer)

- Agent Studio (create/edit agents, permissions, tools)
- Task Board (runs, schedules, status, controls)
- Timeline & Artifacts viewer (screenshots, files)
- Memory browser & retrieval queries
- Settings (API keys, WhatsApp, updates, permissions)

### 4.2 Main Process & IPC Contracts

- Secure IPC calls to the orchestrator service
- File dialogs; OS permissions bootstrap (mac TCC prompts, Win UIAccess notes)

### 4.3 Orchestrator Service (Node)

- Agent graph execution & tool‑calling
- State persistence per step & resumption
- Inter‑agent messaging bus (in‑proc pub/sub)
- Error handling, retries, backoff, cancellation

### 4.4 Scheduler/Queue

- Cron/interval/one‑shot
- Priority, concurrency, rate limits
- Durable resume after crash or reboot

### 4.5 Tools (Initial Set)

- **Browser** (Playwright): open tab, navigate, extract, fill forms, download
- **Desktop Control**: focus window, type, click, hotkeys, screenshots
- **Terminal**: run commands, capture stdout/stderr, exit code
- **File**: read/write/edit files, templates, zips
- **Git**: clone/pull/branch/commit/push
- **HTTP**: GET/POST with headers/auth
- **Email**: draft/send via SMTP/Google Workspace with approval
- **Slides/Docs**: from templates (python‑pptx/LibreOffice), optional Google Slides API
- **Vision/OCR**: image‑to‑text for UI when accessibility is limited

### 4.6 Memory Service

- SQLite schema for agents, tasks, runs, events, artifacts, messages, credentials
- LanceDB for embeddings (code, docs, chats, web pages)
- Background indexers on file change

### 4.7 WhatsApp Service

- Outbound message API
- Inbound webhook handler -> maps to task state & resumes execution
- Local relay option using Cloudflare Tunnel/Ngrok for webhook exposure

---

## 5) Data Model (Initial)

### 5.1 SQLite Tables (draft)

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

### 5.2 Vector Store (LanceDB)

- `embeddings(project_id, agent_id, content, mime, source, path, artifact_id, event_id, created_at, vector)`

---

## 6) Scheduling Semantics

- **One‑shot**: run at specific time
- **Interval**: every N minutes/hours/days
- **Cron**: full CRON syntax
- **Controls**: pause/resume/stop, retries (exponential backoff), max attempts, timeout per step and per run
- **Priorities**: numeric priority; **Concurrency** per agent and global

---

## 7) Human‑in‑the‑Loop (WhatsApp)

### 7.1 Options

- **Meta WhatsApp Cloud API** (recommended): stable, first‑party
- **Twilio WhatsApp**: alternative provider

### 7.2 Flow

1. Agent reaches a **Question** node.
2. System sends a WhatsApp message with context and a short link to approve/reply (optional).
3. Incoming WhatsApp webhook hits local relay; reply is parsed and stored.
4. Orchestrator resumes the exact step with the reply content.

### 7.3 Notes

- Local relay can run inside the app and be exposed via Cloudflare Tunnel/Ngrok.
- Message templates stored in SQLite; variables injected from task state.

---

## 8) Security & Permissions

- **Capability model per agent**:
  - Filesystem scopes (allow/deny paths)
  - Network domain allowlist
  - Command allowlist (terminal)
  - Email send approval required
  - Max runtime & memory
- **Audit log**: immutable events with hashes of artifacts
- **Dry‑run** mode & preview diffs for file changes
- **Signed builds**: Windows Authenticode; macOS notarization

---

## 9) Dev Setup

**Prerequisites**

- Node.js 20+
- Python 3.11+ (for pptx/docs tooling)
- Git;
- **Windows**: Visual Studio Build Tools + Rust (stable toolchain)
- **macOS**: Xcode Command Line Tools; CocoaPods if needed

**Clone & Install**

```bash
git clone <repo>
cd desktop-agents
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
WHATSAPP_WEBHOOK_URL=http://localhost:3333/webhook/whatsapp

# Storage
DATA_DIR=./data
SQLITE_PATH=./data/app.db
LANCEDB_DIR=./data/lancedb

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

## 10) Packaging & Signing

- **electron‑builder** for NSIS (exe) and dmg.
- Windows code signing certificate (EV recommended), timestamping.
- macOS Developer ID Application cert + **notarization** (altool/notarytool) and stapling.
- Auto‑update via electron‑updater (S3/GitHub Releases).

---

## 11) Example: Hello‑World Agent

**Goal:** Open browser → search query → save results → ask human whether to email them → if YES, draft email and wait for approval.

High‑level steps:

1. Browser tool: navigate to search, collect top N results → artifact.json
2. Question node: “Send summary to your email?” via WhatsApp
3. If YES: generate markdown summary → convert to PDF → email tool (approval gate)
4. Store artifacts, events, and memory embeddings; mark run complete

---

## 12) Initial Agent Templates (MVP)

- **Bug‑Fixer**: clone repo, run tests, reproduce issue, propose patch, open PR
- **Presentation‑Maker**: turn content brief into slides with template, export PDF/PPTX
- **Email Assistant**: draft personalized emails from CSV + template; approval per batch

---

## 13) Roadmap

- Vision‑based UI understanding (layout diffs, responsiveness checks)
- Template marketplace for agent recipes
- Cloud relay & team sharing
- Local LLM support via Ollama
- App‑specific adapters (Office, Adobe, Figma via plugins)
- Policy packs & enterprise audit exports (SOC2‑friendly)

---

## 14) Risks & Mitigations

- **OS permission prompts**: first‑run setup wizard; signed helpers to reduce friction.
- **Brittle UI automation**: prefer accessibility APIs; fall back to OCR; implement retries and app‑specific adapters.
- **Security**: strict capability model, approval gates, immutable audit log.
- **WhatsApp onboarding**: set up early; use sandbox; provide local relay wizard.

---

## 15) Contribution Guide (Draft)

- Conventional Commits; PR templates; codeowners.
- Linting/formatting: ESLint + Prettier.
- Tests: vitest for TS; integration tests with Playwright.

---

## 16) License

TBD (proprietary or dual‑license). Add third‑party notices for dependencies.

---

## 17) Appendix: IPC Contracts (Draft)

```ts
// Renderer → Orchestrator
interface CreateAgent { name: string; role: string; tools: string[]; permissions: Record<string, any>; }
interface CreateTask { agentId: string; title: string; schedule?: string; params?: any; priority?: number; }
interface ControlTask { taskId: string; action: 'pause'|'resume'|'stop'|'retry'; }

// Orchestrator → Renderer events
interface TaskEvent { taskRunId: string; level: 'info'|'warn'|'error'; message: string; payload?: any; }
```

```http
# Webhook (WhatsApp)
POST /webhook/whatsapp
X-Verify-Token: <token>
{
  "from": "+8801xxxx",
  "text": "YES, send the emails"
}
```

---

**Notes**

- Project codename: **FutureEngineer**
- This README is the living spec for v0.1; we’ll evolve it as we implement.

