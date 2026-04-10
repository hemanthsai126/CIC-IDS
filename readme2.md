# Chapter 6 — Implementation (section text)

This chapter describes the **actual** development stack and how transcript handling and action-item extraction are implemented in the repository. It separates the **Meeting Intelligence API** (FastAPI) from the **ingest service** (Station Alpha, Node.js), because the live transcript and LLM pipeline run in the latter. Subsections **6.3.1** and **6.3.2** state what is **not** yet built and how a production service would fit the existing data model.

---

## 6.1 Development Environment

The project is a **polyglot monorepo**: a Python API, a TypeScript SPA, a Node ingest worker, and MongoDB. Developers typically run all three application processes plus a local or hosted MongoDB instance.

### 6.1.1 Programming Languages and Frameworks

| Layer | Language | Framework / runtime | Role |
|--------|----------|---------------------|------|
| Web UI | **TypeScript** | **React 19**, **React Router 7**, **Vite 5** | SPA: dashboard, meetings, review queue, logs. |
| Public API | **Python 3.11+** | **FastAPI**, **Uvicorn**, **Motor** (async MongoDB), **Pydantic v2** | REST under `/api`, schemas, CORS, index bootstrap. |
| Ingest worker | **JavaScript (Node.js)** | **Express 4**, **Axios**, **Mongoose 8** (connection + native DB handle), **OpenAI** official SDK | Zoom webhooks, Recall.ai, OpenAI chat completions, writes to MongoDB. |
| Data | — | **MongoDB** (local or Atlas) | Shared database `meeting_intelligence` by default. |

**Libraries of note:** `pydantic-settings` and `python-dotenv` for backend configuration; `dotenv` on the ingest service; `openai` npm package for JSON-mode chat completions; ESLint + TypeScript-ESLint on the frontend.

### 6.1.2 Development Tools and Infrastructure

- **Package and environment management** — `pip` + `requirements.txt` for Python; `npm` for `frontend/` and `station-alpha/`; Python virtual environments recommended for the API.
- **Local serving** — `uvicorn app.main:app --reload --port 8000` for FastAPI; `npm run dev` (Vite) for the UI on the default dev port; `npm run dev` (nodemon) for Station Alpha on port **3001** by default.
- **API proxy during UI development** — Vite proxies `/api` to `http://127.0.0.1:8000`, so the browser can use same-origin relative paths without CORS friction.
- **Zoom / HTTPS** — Zoom requires a public HTTPS URL for webhooks; local development uses a tunnel (e.g. Cloudflare Quick Tunnel / `cloudflared`) with the tunnel base URL configured in Zoom and in Station Alpha’s environment (`PUBLIC_URL` for OAuth callbacks when used).
- **Configuration** — `.env` files (from `.env.example` in `backend/` and `station-alpha/`, `frontend/.env.example` for `VITE_*`) hold secrets and connection strings; never commit real secrets.
- **Build / deploy artifacts** — Frontend: `npm run build` → static `dist/` (e.g. Vercel via `vercel.json`). Backend and Station Alpha deploy as separate long-running services with environment variables set in the host.

---

## 6.2 Backend Implementation

In this codebase, **“backend”** spans two deployable services:

1. **FastAPI** — CRUD-style HTTP API for the UI: meetings, dashboard aggregates, action-item review, processing logs, projects. It does **not** call Recall or OpenAI in the current tree.
2. **Station Alpha** — Implements the **transcript processing pipeline** and **action item extraction** after Zoom events.

### 6.2.1 Transcript Processing Pipeline

The pipeline is **event-triggered** in `station-alpha/src/routes/zoomWebhooks.js` and uses `station-alpha/src/services/recallService.js` and `station-alpha/src/services/meetingIntelligenceSync.js`.

**Meeting start (`meeting.started`):**

1. Verify the request (Zoom URL validation or event delivery using secret tokens in `zoomAuth.js`).
2. `upsertMeetingStarted` — insert or update a `meetings` document keyed by `zoom_meeting_id`, set `processing_status` to `in_progress`, append a `processing_logs` row (`stage: ingestion`).
3. Build a Zoom join URL (password from payload or Zoom REST `getMeetingDetails` when needed).
4. `sendBotToMeeting` — POST to Recall’s `/bot` API with `meeting_url`, `bot_name`, and `recording_config.transcript` using meeting captions; persist returned bot id on the meeting (`recall_bot_id`).

**Meeting end (`meeting.ended`):**

1. `onMeetingEnded` — ensure meeting row exists, update duration, log `transcript_processing` as pending.
2. Resolve the Recall bot id (in-memory map or persisted `recall_bot_id`).
3. **Asynchronous continuation** (`setImmediate`): `waitForBotDone` polls Recall until status `done` (or error / timeout).
4. `getBotTranscript` — reads bot status, follows `media_shortcuts.transcript` download URL, retrieves payload.
5. `transcriptToTextAndSegments` — normalizes Recall JSON (word arrays or plain string) into `raw_text` and `segments` `{ speaker, text }`.
6. **Persistence** — `finalizeSuccess` (in `meetingIntelligenceSync.js`) replaces prior `transcripts` for that `meeting_id`, inserts segments, updates meeting status to `completed` / `processing_status: processed`, and appends extraction-stage logs. If Recall returns no transcript, `finalizeNoTranscript` runs; on hard failures before a salvage path, `finalizeError` sets `processing_status: failed`.

**Resilience:** If OpenAI fails after a transcript exists, the handler still saves the transcript and records a warning in logs (`extractionWarning`). If a later error occurs after fetch, a **salvage** path attempts `finalizeSuccess` with empty action items so text is not lost.

The **FastAPI** backend serves stored transcripts to the UI via `GET /api/meetings/{id}` and does not re-run this pipeline.

### 6.2.2 Action Item Extraction Module

Extraction is implemented in **`station-alpha/src/services/openaiService.js`** (not in Python).

- **Client:** `OpenAI` from the `openai` package, constructed with `OPENAI_API_KEY`.
- **API:** `chat.completions.create` with model **`gpt-4o`**, `temperature: 0.3`, and `response_format: { type: 'json_object' }` for deterministic JSON.
- **Prompting:** A system message instructs the model to extract action items with `text`, `assignee`, and `dueDate`; the user message includes meeting topic and full transcript.
- **Parsing:** Response content is `JSON.parse`d; the code accepts `actionItems`, `items`, `action_items`, or a top-level array. Parse errors yield an empty list.
- **Downstream mapping:** `zoomWebhooks.js` passes the array to `finalizeSuccess`, which maps each item to MongoDB `action_items` fields (`description`, `owner_name`, `due_date`, `priority`, `confidence`, `status: pending_review`, `source_snippet`, timestamps).

**FastAPI’s role after extraction:** Endpoints under `/api/action-items` support **human review**—PATCH fields, approve, reject, bulk approve/reject—implemented in `backend/app/services/action_items.py` and routers. That is the post-extraction workflow, not the LLM call itself.

---

## 6.3 Integration Implementation

### 6.3.1 Jira Ticket Creation Service

**Current state.** There is **no** standalone Jira ticket-creation microservice or Jira REST client in the repository. Initiative and meeting **related links** are modeled as `{ title, url }` lists; `backend/app/services/related_links.py` supplies **placeholder** Atlassian-style URLs for demos. The UI states that real Jira/Confluence OAuth can be wired later. The domain enum includes `ticket_created` on action items for **reporting and workflow semantics**; dashboard metrics count such items (`dashboard_metrics.py`), but nothing in the codebase calls Jira to create issues automatically.

**Implementation sketch (future service).** A small service or FastAPI router module could: (1) store Atlassian OAuth tokens or API tokens securely; (2) expose `POST /integrations/jira/issues` (or a queue consumer) that accepts an approved `action_item` id; (3) map description, optional `source_snippet`, and project configuration to Jira REST `POST /rest/api/3/issue`; (4) persist the returned key/URL on the action item or as a related link; (5) transition status to `ticket_created`. Idempotency keys would prevent duplicate issues on retries.

### 6.3.2 Notification and Digest Generation

**Current state.** There is **no** email SMTP client, SendGrid SDK, or Slack Web API integration in the shipped code. Seed data and `LogStage.NOTIFICATION` illustrate where notification events would appear in `processing_logs`; they do not send real messages.

**Implementation sketch (future).** **Digests** could be generated on a schedule (cron or worker) or after batch thresholds: query MongoDB for `action_items` in `pending_review` grouped by project or owner, render a template (HTML + text), and send via a transactional email provider or Slack incoming webhook / chat.postMessage. **Real-time notifications** might trigger on `finalizeSuccess` (new review items) or on FastAPI after approve/reject, using the same template layer. Configuration would include channel addresses, quiet hours, and secrets in environment variables; delivery failures should write to `processing_logs` with `stage: notification` and `status: failed` for parity with the rest of the observability model.

---

## Cross-references

- Environment variables and run commands: root [`README.md`](../README.md), [`station-alpha/README.md`](../station-alpha/README.md).
- Data shapes: [`database-schema.md`](./database-schema.md).
- High-level architecture narrative: [`architecture-chapter-4-sections.md`](./architecture-chapter-4-sections.md), [`system-architecture.md`](./system-architecture.md).
