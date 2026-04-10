# Chapter 4 — System architecture (section text)

Use the headings below as **4.1**, **4.2**, etc., in your report. Wording distinguishes **implemented** behavior from **planned / extension** items where the repository has hooks but no live integration.

---

## 4.1 Overall Architecture Overview

Meeting Intelligence is a distributed system whose responsibilities are split across a browser-hosted UI, a primary HTTP API, an asynchronous ingest worker, a document database, and third-party meeting and AI services. End users manage meetings, transcripts, and action-item review through the web application; automated capture and enrichment are triggered by external events rather than by user clicks alone.

### 4.1.1 Layered Service-Oriented Architecture

The system follows a **layered, service-oriented** layout:

1. **Presentation layer** — A React single-page application (TypeScript, Vite) implements the dashboard, meeting browser, per-meeting detail, human-in-the-loop action-item review, and processing-log views. It depends only on REST JSON over HTTP to the application layer.

2. **Application / API layer** — A FastAPI service exposes versioned resources under `/api` (dashboard aggregates, projects and rosters, meeting metadata and detail, action-item review operations, paginated processing logs). It enforces request/response schemas, applies business rules for review workflows, and ensures database indexes at startup. This layer is the **single integration point** for the UI.

3. **Ingest / integration service** — Station Alpha, a Node.js Express application, receives Zoom account webhooks, drives Recall.ai meeting bots, retrieves transcripts, and invokes an LLM for structured action-item extraction. It persists results by writing to the same MongoDB database the API uses, using compatible collection and field names.

4. **Data layer** — MongoDB stores meetings, transcripts, action items, processing logs, projects, participants, and join tables. Both FastAPI (Motor) and Station Alpha (native driver via Mongoose connection) are **consumers and producers** of this store.

5. **External service layer** — Zoom (events and optional OAuth-backed REST), Recall.ai (bot and transcript API), and OpenAI (extraction) sit outside the deployment boundary and are invoked only from Station Alpha or via configured credentials.

This decomposition keeps concerns separable: the UI team can evolve against a stable OpenAPI surface; ingest can scale or fail independently while leaving the API and data model as the contract; and the database remains the shared source of truth.

### 4.1.2 Event-Driven Processing Model

Downstream processing is **event-driven** at the boundary with Zoom. When Zoom emits lifecycle events (notably `meeting.started` and `meeting.ended`), Station Alpha handles them asynchronously relative to the user’s session: it may create or update a meeting document, dispatch a bot, wait for transcript completion after the meeting ends, run extraction, and then update meeting `processing_status`, transcript and action-item documents, and append **processing log** rows.

The model is **durable and observable**: each significant step can emit a `processing_logs` entry with a `stage` (e.g. ingestion, transcript processing, extraction) and `status` (success, failure, pending, skipped). The web application’s Logs view reads these records through the API, giving operators a timeline aligned with the pipeline rather than with individual HTTP requests from the browser.

Human review (approve / reject / bulk operations) remains **request-driven** over REST; it complements the event-driven ingest path by updating action-item state after automation has produced `pending_review` items.

---

## 4.2 Core System Layers

### 4.2.1 Input Layer (Transcript Ingestion)

The **input layer** is responsible for bringing meeting content into the system without manual upload in the primary flow.

- **Zoom webhooks** — Station Alpha exposes HTTPS endpoints for Zoom’s event subscription. Webhook payloads are verified using Zoom-issued secret tokens before handling. URL validation challenges from Zoom are answered so the subscription can activate.

- **Meeting metadata** — On `meeting.started`, the service upserts a `meetings` document keyed by Zoom’s meeting identifier, sets lifecycle fields appropriate to an in-flight capture, and records an ingestion-stage processing log.

- **Recall.ai bot** — The service constructs a join URL (including passcode when available, optionally from Zoom’s REST API) and instructs Recall to send a bot into the live meeting. The Recall bot identifier is stored on the meeting document for correlation when the meeting ends.

- **Transcript retrieval** — On `meeting.ended`, the service waits for Recall to finish, fetches the transcript artifact, normalizes it into plain text and speaker segments, and prepares it for persistence and downstream NLP.

Together, these components form the **ingress path** from live collaboration software into the application’s data model.

### 4.2.2 Processing Layer (AI / NLP Engine)

The **processing layer** turns raw transcript text into reviewable work items.

- **Structural transcript handling** — Recall output is mapped to a consistent segment structure (speaker labels where available, contiguous text) and stored as `raw_text` plus `segments` in the `transcripts` collection, with one transcript document per meeting.

- **Action-item extraction** — An LLM (OpenAI API) consumes the transcript and returns candidate action items (description, assignee, due date hints). The ingest service writes each candidate as an `action_items` document with metadata such as confidence and `pending_review` status so humans can correct or reject before any downstream ticketing.

- **Server-side business logic (API)** — FastAPI services compute dashboard metrics, enforce valid state transitions on action items (including statuses used for reporting, such as approved, rejected, and ticket-created), and merge initiative and meeting context for display. This layer does not replace the LLM for initial extraction; it governs **post-extraction** workflow and aggregation.

Pipeline stages beyond extraction—such as automated assignment or outbound notifications—are represented in the domain model as **future stages**; the extraction and review path is fully exercised in the current codebase.

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
