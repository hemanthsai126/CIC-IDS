# Programming effort — client, middle-tier, and data-tier implementation

This chapter describes the programming work sufficient for a reader to understand what was built, with emphasis on the **innovative or non-obvious** parts of Meeting Intelligence: automated meeting ingest, transcript normalization, LLM-based extraction with **human-in-the-loop review**, and a **shared-database** pattern between two server runtimes. It does not list every file; see the repository tree for completeness.

---

## Client implementation

The **client** is a single-page application (SPA) under `frontend/`, implemented with **React** and **TypeScript**, bundled by **Vite**. The entry point wires **React Router** for client-side navigation and a top-level **error boundary** so rendering failures do not blank the entire shell (`main.tsx`, `App.tsx`).

### Scope and routes

The UI supports the product narrative end to end:

- **Dashboard** (`/`) — Rolling metrics, initiative filters, and a paginated, sortable meeting table aligned with backend query parameters (`DashboardHome.tsx`).
- **Meetings** (`/meetings`, `/meetings/:id`) — List and rich **meeting detail**: metadata, transcript segments, participants, action items, related links, and inline editing of initiative context where the API allows (`MeetingsList.tsx`, `MeetingDetailPage.tsx`).
- **Review queue** (`/review`, `/review/item/:itemId`) — **Human-in-the-loop** workflow: operators inspect LLM-proposed action items, edit fields, approve or reject, including bulk actions per meeting (`ReviewQueuePage.tsx`, `ReviewItemPage.tsx`). This is a central “innovation” in the sense that raw model output is not treated as ground truth.
- **Observability** (`/logs`) — Paginated **processing logs** with optional filters, so users can correlate UI state with ingest and extraction stages (`LogsPage.tsx`).

Navigation and layout use a persistent sidebar (`NavSidebar.tsx`) and shared dashboard styling (`styles/dashboard.css`, `App.css`).

### API integration pattern

All server communication is centralized in **`src/api.ts`**: typed `fetch` wrappers, query-string construction for pagination and filters, and **normalization** of JSON into stable TypeScript shapes (`normalizeDashboardSummary`, `normalizeMeetingListItem`, `normalizePages`, etc.). That pattern defends the UI against minor API evolution and keeps pages readable.

In development, **`vite.config.ts`** proxies `/api` to the FastAPI process so the browser can use relative URLs without configuring CORS for every experiment. For production builds, **`VITE_API_BASE_URL`** targets the deployed API host.

### Why this matters for the reader

The client deliberately **does not** talk to the Zoom ingest service (Station Alpha). Every feature the user sees is backed by one contract: the **FastAPI** `/api` surface. That separation simplifies deployment and security (the ingest URL stays server-to-server with Zoom).

---

## Middle-tier implementation

The “middle tier” in this project is **split across two services**, which is intentional: one serves the browser; the other reacts to external events and third-party APIs.

### 1. Application API (FastAPI)

Location: `backend/app/`.

**Role:** Authoritative HTTP API for the SPA—validation, orchestration of reads/writes to MongoDB, and business rules for **action-item review** (state transitions, bulk operations, editable fields).

**Structure:**

- **`main.py`** — FastAPI app, CORS, lifespan hook that runs **`ensure_indexes()`** on startup.
- **`routers/`** — Route modules grouped by domain: `dashboard`, `projects`, `meetings`, `action_items`, `logs`, mounted under `settings.api_prefix` (default `/api`).
- **`schemas/`** — Pydantic models for request bodies and responses (strong typing and OpenAPI generation).
- **`services/`** — Domain logic (e.g. dashboard aggregation, meeting detail assembly, team roster updates, related-link resolution, observability-oriented queries).

**Innovation-relevant behavior:** The API implements the **review queue** and **approval/rejection** semantics that turn “model suggestions” into governed records. It also exposes **processing logs** so operators can debug ingest without SSH or raw database access.

OpenAPI documentation is available at `/docs` when the server runs, which documents the contract the client relies on.

### 2. Ingest and integration service (Station Alpha)

Location: `station-alpha/src/`.

**Role:** **Event-driven** pipeline: Zoom webhooks → Recall.ai bot → transcript download → OpenAI extraction → **direct writes** to the same MongoDB collections the API reads.

**Key modules:**

- **`index.js`** — Express app, Mongo connection, mounts webhook and OAuth callback routes.
- **`routes/zoomWebhooks.js`** — Validates Zoom requests, handles `meeting.started` / `meeting.ended`, orchestrates Recall polling and OpenAI calls, and calls into persistence helpers with **salvage paths** (save transcript even if extraction fails).
- **`services/recallService.js`** — Recall REST calls (create bot, poll status, fetch transcript via download URL).
- **`services/openaiService.js`** — Chat completion with **JSON output** for structured action items.
- **`services/meetingIntelligenceSync.js`** — Idempotent-style writes: replace transcript for a meeting, replace extracted items, append `processing_logs`, update meeting lifecycle fields.

**Innovation-relevant behavior:** The middle tier here is not a classic “CRUD only” layer—it is an **integration orchestrator** with **defensive persistence** (logging failures, optional warnings when the LLM is unavailable) so operational data is still captured.

Zoom OAuth-related code (`oauthCallback.js`, `zoomAuth.js`, `zoomApi.js`) supports optional REST usage (e.g. meeting passcode) alongside webhooks.

---

## Data-tier implementation

The **data tier** is **MongoDB**, database name configurable (default `meeting_intelligence`). The schema is **document-oriented**: references by ObjectId, some denormalized fields for display (e.g. initiative labels on meetings), and embedded arrays where appropriate (transcript segments, related links).

### Collections and relationships

The logical model is documented in **`docs/database-schema.md`** (Mermaid ER diagram). Core entities include:

- **`meetings`** — Lifecycle and processing state, Zoom correlation fields (`zoom_meeting_id`, `recall_bot_id`), optional `project_id`, context overrides.
- **`transcripts`** — One document per meeting (unique index on `meeting_id`).
- **`action_items`** — Extracted tasks with `pending_review` → approved/rejected (and `ticket_created` for reporting scenarios).
- **`processing_logs`** — Append-only style operational history per meeting.
- **`projects`**, **`participants`**, **`meeting_participants`** — Initiatives and attendance modeling.

### Indexing and consistency

**`backend/app/db.py`** defines **`ensure_indexes()`** invoked at API startup: time-based and filter columns for the dashboard, uniqueness for one transcript per meeting, sparse unique `zoom_meeting_id` for ingest idempotency, compound uniqueness on meeting–participant pairs, and sparse unique email on participants.

### Dual-writer pattern

Two runtimes write the same database:

1. **FastAPI + Motor** — Async reads/writes for user-driven updates and queries.
2. **Station Alpha** — Uses Mongoose’s connection to obtain the native driver and write **compatible documents** expected by the API serializers.

The programming effort on the data tier therefore includes **cross-service schema discipline**: field names and types must stay aligned so the UI and API do not break when ingest runs.

---

## Summary

| Tier | Primary implementation | Main responsibility |
|------|------------------------|---------------------|
| **Client** | React + TypeScript (`frontend/`) | Operator UX: dashboard, meetings, **human review**, logs; calls FastAPI only. |
| **Middle-tier** | FastAPI (`backend/`) + Express ingest (`station-alpha/`) | REST API + webhook-driven **AI pipeline** and external integrations (Zoom, Recall, OpenAI). |
| **Data-tier** | MongoDB + indexes (`db.py`, shared writes) | Durable transcripts, action items, logs, projects/participants. |

For deeper file-level detail, see **`docs/architecture-chapter-6-sections.md`**. For citations to external products and standards, see **`docs/references.md`**.

### 4.2.3 Data Layer (Persistence and Storage)

MongoDB is the **authoritative store** for operational data.

- **Collections** — Core entities include `meetings`, `transcripts`, `action_items`, `processing_logs`, `projects`, `participants`, and `meeting_participants`, with references by ObjectId as described in the project’s database schema documentation.

- **Indexing** — The FastAPI application ensures indexes at startup for query patterns used by the dashboard and detail views (e.g. meeting time, processing status, action-item status, unique transcript per meeting, sparse unique Zoom meeting id for ingest idempotency).

- **Shared access pattern** — Station Alpha writes ingest and NLP outputs directly; the API reads and updates the same documents for UI-driven operations. This **shared-database** pattern minimizes sync latency but requires discipline so both codepaths preserve schema compatibility.

---

## 4.3 Integration Layer

### 4.3.1 Jira REST API Integration

**Current implementation.** The product surface includes **related links** on projects and meetings (title + URL pairs) so initiatives can point to external tools. For demonstration, some links use placeholder Atlassian-style URLs; the UI copy notes that real Jira or Confluence OAuth and deep linking can be wired later. The action-item domain includes a `ticket_created` status, and dashboard metrics count items in that state—supporting a workflow where an approved item is later linked to an external ticket, whether that link is created manually or by automation.

**Intended REST integration (extension).** A full Jira integration would typically: (1) authenticate using Atlassian OAuth 2.0 or an API token scoped to the target site; (2) on user or policy trigger after approval, call Jira’s REST API to create an issue (summary from action-item description, optional description from transcript snippet, project and issue type from configuration); (3) persist the returned issue key or URL on the action item or in related links; (4) transition the action item to `ticket_created`. Webhooks from Jira could optionally update status when the remote issue closes. This design keeps Jira outside the critical path of ingest while still closing the loop from meeting commitments to trackable work items.

### 4.3.2 Email and Slack Notification Systems

**Current implementation.** There is no dedicated outbound email or Slack client in the repository’s production paths. Demo seed data and log-stage enumerations include **notification** as a conceptual pipeline stage, illustrating where “review queue notified” or similar messages would appear in `processing_logs` once implemented.

**Recommended extension.** **Email** could be implemented with a transactional provider (SMTP or HTTPS API), templated messages for “new items pending review” or “meeting processing failed,” and idempotent enqueueing keyed by meeting or batch. **Slack** could use incoming webhooks for a fixed channel, or a Slack app with OAuth for workspace-scoped posting. In both cases, the FastAPI layer (after DB commit) or a small background worker would read from the same MongoDB state the UI uses, respecting user or channel preferences stored in configuration or future `users` / `subscriptions` collections. Secrets would remain server-side (environment or vault), never in the SPA.

---

## 4.4 Monitoring and Security Components

### 4.4.1 Observability and Monitoring Infrastructure

**Application-level observability** is implemented around **processing logs**: each ingest and extraction step can append structured documents (`meeting_id`, `stage`, `status`, `message`, timestamps, optional duration). The API exposes paginated, filterable access to these logs for the operator UI.

**Health endpoints** — FastAPI serves `GET /health` for liveness; Station Alpha exposes a similar health route for process and dependency sanity checks.

**Gaps and extensions.** The codebase does not ship a full APM stack (e.g. distributed tracing, metrics exporters). For production, one would add structured logging aggregation, alerts on repeated `failed` log statuses or stuck `in_progress` meetings, and optional integration with cloud monitoring or open telemetry collectors in front of FastAPI and Station Alpha.

### 4.4.2 Security and Authentication Framework

**Webhook and API key security** — Zoom webhook requests are validated with secret tokens before business logic runs. Recall and OpenAI calls use server-side API keys (never exposed to the browser). Zoom OAuth, where used for REST calls (e.g. meeting details), exchanges authorization codes for tokens in Station Alpha; tokens must be stored and refreshed according to Zoom’s guidelines.

**Browser security** — The SPA talks to FastAPI over HTTPS in deployment; CORS is configured via allowlisted origins (and optional regex for preview deployments) so only trusted front-end origins can call the API with credentials if enabled.

**End-user authentication** — The current course project UI does not implement a full end-user login (no JWT/session layer in the reviewed API). Production hardening would add identity (OIDC, SAML, or similar), role-based access to meetings and approve actions, and rate limiting on public endpoints. The ingest service must remain reachable only from Zoom’s IP ranges or via verified signatures, not as an open generic HTTP API.

Together, these measures align with a **defense-in-depth** approach: verify external callers, protect secrets on the server, constrain browser access with CORS, and plan explicit identity for multi-tenant or sensitive deployments.
