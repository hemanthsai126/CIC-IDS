# Plan: Arrakis Coterie Portal Agent

Date: 2026-06-12
Status: Proposed
Builds on:

- `apps/plans/arrakis-quoting-source-subagents-plan.md`
- `apps/arrakis/backoffice/agents/coterie_portal_agent/`
- `apps/arrakis/backoffice/agents/quote_sources/`
- `apps/arrakis/data_api/src/quote_sources/coterie_portal/otp.ts`
- `apps/arrakis/backoffice/agents/quoting_agent/data/coterie-quote-flow-questions.md`

## Objective

Build the Coterie browser-portal source agent at
`apps/arrakis/backoffice/agents/coterie_portal_agent`.

Desired end state:

- The quote-source orchestrator can route Coterie work to a real
  `browser_portal` subagent instead of the current `needs_build` stub.
- The subagent uses Browserbase for portal automation and keeps the Browserbase
  integration modular so future carrier portal agents can reuse the same
  session, login, OTP, artifact, selector, and timeout patterns.
- The subagent opens Coterie at
  `https://dashboard-v2.coterieinsurance.com/quotes`.
- If the Browserbase session is not authenticated, the subagent logs in with
  Coterie credentials from secrets and completes OTP using the existing carrier
  OTP webhook/polling path.
- For new quote work, the subagent navigates through the Coterie quote flow and
  stops at quote review. It never clicks `Continue to pay`, never binds, and
  never enters payment information.
- For existing quote work, the subagent can select a quote from the quotes list
  or load `/quote/edit/{portalQuoteId}`, open the editable accordion sections,
  apply allowed updates, and re-extract quote state.
- Session state, checkpoints, replay URLs, quote artifacts, blocking facts, and
  operator-handoff reasons are visible in the existing quote-source dashboard
  card/modal.

Out of scope for V0:

- Binding, payment, issuing policies, saving payment methods, or crossing the
  Coterie `Continue to pay` boundary.
- Solving CAPTCHA or any browser security challenge.
- Long-term Browserbase context reuse across many sessions. V0 can log in fresh
  and rely on OTP. Add persistent Browserbase contexts later if OTP volume
  becomes noisy.
- Full coverage of every Coterie industry-specific questionnaire branch. V0
  should cover the common GL/BOP/PL flow captured in the existing docs and hand
  off when the portal asks an unknown question.
- Portal automation for other carriers. The shared Browserbase modules should be
  designed for reuse, but only Coterie is implemented in this plan.

## Current Starting Point

Already present:

- `coterie_portal_agent` exists as a quote-source subagent stub:
  - `manifest.ts` declares `sourceKey = "coterie_portal"`,
    `kind = "browser_portal"`, and a portal-shaped state schema.
  - `runtime.ts` records updates, then returns `status: "needs_build"`.
  - `bundle.local.json` exists.
- `login-smoke.ts` already proves the key primitives:
  - create a Browserbase session,
  - connect Playwright over CDP,
  - create a Coterie OTP request in the Arrakis Data API,
  - poll for OTP,
  - fill the OTP,
  - consume the OTP request.
- Data API already has Coterie OTP support:
  - `POST /v1/internal/quote-sources/coterie-portal/otp-requests`
  - `GET /v1/internal/quote-sources/coterie-portal/otp-requests/:id?includeOtp=true`
  - `POST /v1/internal/quote-sources/coterie-portal/otp-requests/:id/consume`
  - inbound email projection into `arrakis.carrier_login_otps`.
- Quote-source sessions and updates already exist, including pause/resume/retry
  and dashboard rendering.
- Prior quote-flow capture already documents most Coterie questions in
  `coterie-quote-flow-questions.md`.

Live portal observations from the logged-in Chrome session on 2026-06-12:

- The quotes URL lands at `/quotes` when authenticated.
- The quotes list includes search, user filter, quote type filter, archive
  selection, row checkboxes, `Edit Quote ...`, and `See Quote Details ...`.
- `New Quote` navigates to `/quote` and starts with:
  - heading `What is the business name?`
  - `Legal Business Name`
  - optional DBA checkbox
  - `Next Step: Business Address`
- Existing quotes open at `/quote/edit/{portalQuoteId}`.
- The edit page exposes:
  - total due,
  - yearly/monthly payment toggle,
  - `Download Quote`,
  - `Quote Snapshot`,
  - policy effective date,
  - `Bind as`,
  - policy selection/detail controls,
  - `Additional Insureds`, `Business Details`, and `Contact Details`
    accordions,
  - `Continue to pay` as the bind/payment boundary.
- `Business Details` includes legal name, DBA, disabled industry/address,
  estimated revenue, annual payroll, employees, business start date, and
  premises square footage.
- `Contact Details` includes first name, last name, phone, policyholder email,
  mailing address, disabled city/state/zip, and `Update Address Manually`.
- `Additional Insureds` starts with an `Individual Additional Insured` checkbox.

## Architecture

Keep the existing quote-source contract, but make the quoting agent the
orchestration owner. Replace the Coterie stub internals with an execution agent
whose tools own browser side effects.

### Quoting-Agent-Owned Invocation

The quoting agent is the quote orchestrator. It owns carrier quote readiness,
carrier selection, and source invocation.

Quoting agent responsibilities:

- Decide whether Coterie is the right carrier/source for the lead based on the
  lead, conversation, research/appetite state, workflow state, and operator
  instructions.
- Ask customer/operator follow-up questions when Coterie-relevant data is
  missing.
- Call `coterie_portal` only when the quoting agent believes Coterie has enough
  data to produce a meaningful quote.
- Send the Coterie agent the known quote facts and the intended action
  (`new_quote` or `edit_existing`).
- Decide what to do after the Coterie agent returns a quote, handoff, decline,
  or failure.

Coterie agent responsibilities:

- Validate the quoting-agent-provided payload defensively.
- Map the provided facts to Coterie portal fields.
- Log in, complete OTP, operate the browser, and extract quote artifacts.
- Return structured results or structured handoff reasons.
- Never decide long-running customer intake strategy. If the payload is
  incomplete or ambiguous, return a structured handoff/rejection and let the
  quoting agent decide the next question or operator action.

This keeps readiness, missing-data collection, and carrier-routing decisions in
one place while keeping the Coterie agent focused on portal execution.

Recommended shared module:

```text
apps/arrakis/backoffice/agents/quote_sources/browser_portal/
  browserbase.ts
  stagehand.ts
  artifacts.ts
  auth.ts
  navigation.ts
  otp.ts
  selectors.ts
  state.ts
  telemetry.ts
```

Responsibilities:

- `browserbase.ts`
  - create Browserbase sessions,
  - connect the remote browser session,
  - compute replay URLs,
  - close sessions reliably,
  - attach carrier/source metadata.
- `stagehand.ts`
  - create a Stagehand instance on top of the Browserbase browser/session,
  - use Stagehand for page observation, resilient form filling, and extraction
    when Coterie selectors or labels are fragile,
  - keep deterministic Playwright/selector helpers available for stable fields,
  - centralize screenshots, downloads, and browser artifacts that Stagehand
    produces during a run.
- `auth.ts`
  - generic portal auth result types,
  - login-state detection helpers,
  - no credential logging.
- `otp.ts`
  - generic OTP request/poll/consume client over the Arrakis Data API,
  - timeout and retry helpers,
  - never include OTP codes in events, logs, checkpoints, or state.
- `selectors.ts`
  - stable element helpers around accessible roles, labels, text, and fallback
    CSS,
  - explicit uniqueness checks before click/fill,
  - consistent timeout errors.
- `navigation.ts`
  - checkpointed step runner,
  - idempotent `gotoUnlessAlreadyThere`,
  - side-effect boundaries for submit/save/bind actions.
- `artifacts.ts`
  - screenshot/replay metadata capture,
  - download artifact discovery from Browserbase/Stagehand,
  - upload downloaded quote PDFs to the Kinro bucket through the existing Arrakis
    document/attachment storage path.
- `state.ts`
  - normalized portal session state transitions,
  - checkpoint append/merge helpers,
  - field-ledger helpers for "known", "entered", "blocked", and "unknown".
- `telemetry.ts`
  - quote-source events, timings, Browserbase session ids, and redaction.

Coterie-specific module layout:

```text
apps/arrakis/backoffice/agents/coterie_portal_agent/
  AGENTS.md
  SYSTEM_PROMPT.md
  config.ts
  login.ts
  runtime.ts
  manifest.ts
  quote-target.ts
  quotes-page.ts
  new-quote-flow.ts
  edit-quote-flow.ts
  quote-review.ts
  field-mapping.ts
  coterie-selectors.ts
  login-smoke.ts
  test/
```

Responsibilities:

- `config.ts`
  - read `COTERIE_PORTAL_QUOTES_URL`,
    `COTERIE_PORTAL_BROKER_EMAIL`, optional `COTERIE_PORTAL_PASSWORD`,
    `KINRO_CARRIER_OTP_RECIPIENT_EMAIL`, Browserbase env, and Data API env.
  - default quotes URL to `https://dashboard-v2.coterieinsurance.com/quotes`.
- `login.ts`
  - open the quotes URL,
  - detect authenticated vs login/OTP screens,
  - submit email/password only when required by the page,
  - create OTP requests before triggering the OTP email when possible,
  - poll for OTP with a bounded timeout,
  - fill and consume OTP,
  - return an auth checkpoint without exposing secrets or OTP values.
- `quote-target.ts`
  - validate the quoting agent's intended target: new quote or existing quote
    edit.
  - require an explicit target shape for existing quote edits.
  - require an explicit `allowQuoteCreation: true` for any action that will
    create a real Coterie quote.
- `quotes-page.ts`
  - parse list rows,
  - search by business/contact,
  - open quote detail drawer,
  - open quote edit,
  - navigate to `/quote` for new quote only after the creation guard passes.
- `new-quote-flow.ts`
  - business name,
  - address/manual address fallback,
  - industry search and appetite gate parsing,
  - policy selection,
  - business details,
  - quote review extraction.
- `edit-quote-flow.ts`
  - load `/quote/edit/{portalQuoteId}` when known,
  - otherwise find the row on `/quotes`,
  - open and update accordions,
  - support safe recalculation/update actions only when the page clearly
    exposes them.
- `quote-review.ts`
  - extract total due, payment cadence, product/policy selections, limits,
    deductibles, quote snapshot URL, quote PDF download availability, effective
    date, and warnings.
  - download the quote PDF through Browserbase/Stagehand when Coterie exposes
    `Download Quote`.
  - save the downloaded PDF to the Kinro bucket and return the stored document
    reference.
- `field-mapping.ts`
  - map quoting-agent-provided lead/quote facts into Coterie field values.
  - normalize money, dates, phone, employee counts, business start month/year,
    and coverage choices.
  - validate that required portal fields are present before opening Browserbase.
- `runtime.ts`
  - keep `QuoteSourceSubagent.handleUpdate` as the only public entrypoint.
  - run deterministic browser tools and a small LLM decision layer only for
    ambiguous mapping, unknown portal questions, and operator-handoff summaries.

## Update Contract Changes

Expand `coteriePortalUpdateSchema` from generic facts into a typed browser
portal request while keeping backward compatibility with existing
`facts_updated` payloads.

Avoid making the Coterie agent own a separate orchestration model. The quoting
agent should send a practical execution packet: target, facts, coverage intent,
effective date, operator context, and safety flags. The facts object can stay
flexible so the quoting agent can pass the best available source facts without
forcing every upstream system to conform to a second rigid schema. Coterie's
`field-mapping.ts` owns the carrier-specific translation from those facts into
portal fields.

Recommended shape:

```ts
const coteriePortalTargetSchema = z.discriminatedUnion("mode", [
  z.object({
    mode: z.literal("new_quote"),
    allowQuoteCreation: z.boolean().default(false),
  }),
  z.object({
    mode: z.literal("edit_existing"),
    portalQuoteId: z.string().trim().min(1).max(256).optional(),
    businessName: z.string().optional(),
    contactName: z.string().optional(),
  }),
]);
```

Treat `portalQuoteId` as an opaque Coterie-provided quote identifier. Do not
validate it as a UUID unless the portal is confirmed to always use UUID-shaped
quote IDs. Store and pass it as a bounded string, matching the existing
`quote_applications.provider_application_id` pattern, so valid Coterie IDs can be
reopened or updated even if Coterie uses IDs like `Q-12345` or another
carrier-specific format.

Add update fields:

- `target`
- `facts`
- `coverageIntent`
- `effectiveDate`
- `operatorAnswer`
- `quoteReadinessAssessment`
  - why the quoting agent selected Coterie,
  - whether this is ready to execute or needs operator review,
  - any appetite/risk caveats the quoting agent wants preserved.
- `portalEvent`
- `dryRun`

Behavior:

- `start` should normally be created by the quoting agent only after it believes
  the lead is Coterie-ready.
- `start` with no target defaults to `new_quote` only when the source session
  has no previous `portalQuoteId`. It must still require
  `allowQuoteCreation: true` before creating a real portal quote.
- `facts_updated` merges facts and either continues the current portal flow or
  parks in `waiting_for_facts`.
- `operator_answer` resolves a specific `operatorHandoff`.
- `retry` resumes from the latest durable checkpoint.
- `cancel` closes the Browserbase session if open and returns `cancelled`.

The Coterie agent should not ask customer intake questions itself. If validation
fails or the portal exposes a question the agent cannot answer safely, the agent
returns a structured handoff/rejection and the quoting agent decides the next
question, operator task, or retry.

## State Schema Changes

Keep the current high-level phases, but add concrete browser/session fields.

Recommended state additions:

```ts
{
  portalPhase:
    | "not_started"
    | "login"
    | "business_profile"
    | "coverage_selection"
    | "underwriting_questions"
    | "quote_review"
    | "complete",
  auth: {
    status: "unknown" | "authenticated" | "login_required" | "otp_wait" | "failed",
    brokerEmail: string | null,
    lastAuthenticatedAt: string | null,
    lastOtpRequestId: string | null
  },
  browserbase: {
    sessionId: string | null,
    replayUrl: string | null,
    startedAt: string | null,
    endedAt: string | null
  },
  target: {
    mode: "new_quote" | "edit_existing" | null,
    portalQuoteId: string | null,
    businessName: string | null,
    contactName: string | null
  },
  fieldLedger: {
    knownFacts: Record<string, unknown>,
    enteredFields: Record<string, unknown>,
    blockingFacts: string[],
    unknownPortalPrompts: Array<{ label: string, phase: string, at: string }>
  },
  quoteReview: {
    status: "not_reached" | "quoted" | "pending_completion" | "declined" | "unknown",
    totalDueToday: string | null,
    paymentCadence: "yearly" | "monthly" | null,
    products: string[],
    effectiveDate: string | null,
    quoteSnapshotUrl: string | null,
    downloadQuoteEnabled: boolean | null,
    warnings: string[]
  }
}
```

Do not persist:

- passwords,
- OTP codes,
- raw full-page HTML,
- cookies,
- local storage,
- large screenshots with unrestricted access.

## Login And OTP Flow

V0 flow:

1. Create a Browserbase session with metadata:
   - `carrier: "coterie"`
   - `sourceKey: "coterie_portal"`
   - `quoteWorkflowId`
   - `quoteSourceSessionId`
2. Connect Stagehand to the Browserbase browser/session. Keep Playwright access
   available for deterministic low-level actions, but prefer Stagehand for
   observation, resilient form interactions, and extraction on fragile portal
   pages.
3. Navigate to `COTERIE_PORTAL_QUOTES_URL`.
4. Detect whether the portal is already authenticated.
5. If login is required:
   - create an OTP request in the Data API,
   - submit configured broker email and password if the page asks for them,
   - trigger OTP,
   - poll every 5 seconds until the OTP request is `otp_received`, `failed`,
     `expired`, or the timeout is reached,
   - fill OTP,
   - consume the OTP request.
6. Confirm the page reaches `/quotes` or another authenticated portal page.
7. Record a login checkpoint with Browserbase session id, replay URL, and OTP
   request id, but no OTP code.

Timeout handling:

- OTP timeout should return `waiting_for_operator` when the browser session is
  still usable and a human can intervene.
- Hard auth failures should return `failed` with a redacted reason.
- Expired OTP requests should create a new request on `retry`, not reuse the
  expired row.

Secrets/env:

- `BROWSERBASE_API_KEY`
- `BROWSERBASE_PROJECT_ID`
- `BROWSERBASE_API_BASE_URL` optional
- `BROWSERBASE_APP_BASE_URL` optional
- `COTERIE_PORTAL_QUOTES_URL`
- `COTERIE_PORTAL_BROKER_EMAIL`
- `COTERIE_PORTAL_PASSWORD` optional, depending on actual login page behavior
- `KINRO_CARRIER_OTP_RECIPIENT_EMAIL`
- `KINRO_ARRAKIS_DATA_API_BASE_URL`
- `KINRO_ARRAKIS_DATA_API_TOKEN`

## Portal Flow

### New Quote

Guardrails:

- Do not start a real new quote unless `allowQuoteCreation: true` is present.
- Local/dev smoke tests default to `dryRun: true` and stop before any submit that
  would create a quote.
- Never click `Continue to pay`.

Flow:

1. Preflight facts from lead/workflow:
   - business name,
   - industry class or NAICS/search phrase,
   - address,
   - revenue,
   - payroll,
   - employees,
   - years in business or start date,
   - coverage intent.
2. Navigate to `/quote`.
3. Business name:
   - fill legal business name,
   - set DBA when provided,
   - continue to address.
4. Address:
   - prefer verified autocomplete when exact match is available,
   - use `Update Address Manually` when autocomplete cannot be verified,
   - continue to industry.
5. Industry:
   - search by preferred Coterie industry label, then NAICS/description fallback,
   - parse appetite result and ineligible activity warnings,
   - stop in `waiting_for_operator` if the portal shows out-of-appetite,
     ambiguous stale appetite state, or ineligible activities that need
     confirmation.
6. Policies:
   - map coverage intent to BOP, GL, and/or PL.
   - respect BOP/GL mutual exclusion.
   - for BOP, select required property coverage when facts support it.
7. Business details:
   - fill revenue, payroll, employees, start date, premises fields.
   - block on missing/unknown facts rather than guessing.
8. Underwriting/additional questions:
   - answer only known captured prompts.
   - create an operator handoff for unknown prompts.
9. Quote review:
   - extract quote result,
   - persist application/quote summary,
   - stop before payment/bind.

### Existing Quote

Flow:

1. If `portalQuoteId` is known, navigate directly to
   `/quote/edit/{portalQuoteId}`.
2. Otherwise navigate to `/quotes`, search/filter by business/contact, and select
   the unique matching row.
3. Use the row detail drawer to inspect quick status when useful.
4. Open `Edit Quote ...`.
5. Read current quote review state and accordion values.
6. Apply requested updates only for fields explicitly included in facts or
   operator answers.
7. Re-extract quote review.
8. Persist `portalQuoteId` and quote summary back to source session state and
   application rows.

Editable areas observed:

- policy effective date,
- payment cadence,
- BOP/GL/PL product toggles where enabled,
- PL limits, deductible, years of professional experience, prior acts,
- additional insured checkbox/options,
- business details accordion,
- contact details accordion,
- mailing address manual update.

## Result Mapping

Return `QuoteSourceHandleResult` values:

- `waiting_for_facts`
  - defensive validation found that the quoting agent's payload is incomplete before
    opening or continuing portal flow.
  - the quoting agent owns the next question or operator task.
- `waiting_for_operator`
  - OTP/login handoff, appetite ambiguity, unknown portal question, CAPTCHA,
    PDF/download issue, or any action that would approach bind/payment.
- `quoted`
  - quote review reached, premium/summary extracted, quote snapshot/PDF
    artifacts handled when available, and the flow stopped before bind/payment.
- `declined`
  - portal gives a clear out-of-appetite/decline result.
- `failed`
  - Browserbase/session/automation failure that retry may or may not fix.
- `cancelled`
  - cancel update.

Structured handoff reasons:

- `incomplete_quoting_payload`
- `otp_timeout`
- `login_failed`
- `captcha_or_security_challenge`
- `portal_appetite_ambiguous`
- `portal_decline_confirmation_needed`
- `unknown_portal_question`
- `pdf_download_failed`
- `quote_snapshot_unavailable`
- `bind_or_payment_boundary`
- `selector_or_portal_changed`

Every handoff should include:

- `reasonCode`,
- short human-readable `summary`,
- current `portalPhase`,
- Browserbase replay URL when available,
- latest checkpoint,
- visible prompt/labels/options when safe to capture,
- recommended quoting-agent/operator next action.

Application upserts:

- `carrier: "coterie"`
- `product`: `BOP`, `CGL`, or `PL`
- `providerApplicationId`: Coterie portal quote id when available
- `status`: `quoted`, `declined`, `pending_completion`, or `failed`
- `quoteResult`:
  - total due,
  - premium/total amount fields used by Arrakis summaries,
  - payment cadence,
  - effective date,
  - selected products,
  - limits/deductibles,
  - quote snapshot URL,
  - quote PDF attachment/document reference when download succeeds,
  - quote PDF download status when it does not,
  - Browserbase replay URL,
  - warnings,
  - Coterie provider quote id,
  - bindable flag must be false.

PDF handling:

- Download the Coterie quote PDF when the portal exposes `Download Quote`.
- Fetch the downloaded PDF artifact from the Browserbase/Stagehand browser
  session rather than relying on a local filesystem path.
- Store the PDF in the Kinro bucket through the existing Arrakis
  document/attachment storage path so it can be sent to the customer by email.
- Persist the bucket object path/document id on the quote result and add a
  timeline/event reference to the stored PDF.
- Persist safe file metadata when available:
  - filename,
  - content type,
  - byte size,
  - downloadedAt,
  - quote workflow id,
  - quote application id,
  - Coterie provider quote id.
- If download fails, keep the quote snapshot URL and return
  `waiting_for_operator` with `reasonCode = "pdf_download_failed"` rather than
  silently marking the quote fully delivered.

Compliance boundary:

- The Coterie agent must never bind, pay, issue, save payment details, click
  `Continue to pay`, or cross any equivalent bind/payment boundary.
- The Coterie agent must never represent a quote as final or bound.
- Licensed/operator review owns final quote verification and any customer-facing
  language before purchase.

## Dashboard Changes

Minimal V0:

- Update `CoteriePortalSourceModal` to show:
  - auth status,
  - Browserbase replay link,
  - target mode and portal quote id,
  - quote review summary,
  - blocking facts,
  - operator handoff reason,
  - latest checkpoints.
- Remove the stub-only blue notice once `stub: false`.
- Add copy that clearly marks `Continue to pay` as not automated.

Optional V0.5:

- Add an operator action to send an `operator_answer` update from the modal.
- Add a "resume in Browserbase replay" link if Browserbase supports it for the
  session.

## Testing Plan

Unit tests:

- Browserbase session creation request shape and replay URL construction.
- Stagehand session construction and fallback behavior around stable selectors.
- OTP request/poll/consume behavior, including timeout, expired, failed, and
  redaction.
- Coterie fact-to-field mapping:
  - money formatting,
  - date/start-date formatting,
  - coverage intent mapping,
  - address/manual fallback inputs.
- State/checkpoint helpers.
- Quote result extraction from saved DOM fixtures.
- Browserbase/Stagehand download artifact handling and bucket upload metadata.
- Guard that `Continue to pay` and equivalent bind/payment selectors are never
  clicked by allowed automation steps.

Integration tests with mocked portal:

- Authenticated `/quotes` path.
- Login + OTP path.
- New quote happy path to quote review.
- Quote PDF download from Browserbase/Stagehand and storage in the Kinro bucket.
- Appetite decline path to `declined`.
- Unknown question path to `waiting_for_operator`.
- Existing quote edit path via `/quote/edit/{id}`.
- Existing quote selection path via `/quotes` search.

Smoke tests:

- Keep `login-smoke.ts`, but refactor it onto the shared Browserbase/auth/OTP
  modules.
- Add a read-only `quotes-page-smoke` that logs in, opens `/quotes`, parses rows,
  and exits.
- Add a gated `new-quote-dry-run-smoke` that opens `/quote`, verifies the first
  step, and exits without entering data or creating a quote.
- Add live quote-creation smoke only behind an explicit environment variable,
  for example `COTERIE_LIVE_QUOTE_SMOKE_ALLOW_CREATE=true`, and use clearly
  marked test data.

Verification commands:

```bash
pnpm --filter @kinro/arrakis-backoffice test
pnpm --filter @kinro/arrakis-backoffice lint
pnpm lint
```

Use repo auth/env flow before GCP-backed or Data API integration work:

```bash
pnpm gcp:auth:status
pnpm env:doctor arrakis-backoffice --env dev
pnpm env:pull arrakis-backoffice --env dev
pnpm env:exec arrakis-backoffice --env dev -- pnpm --filter @kinro/arrakis-backoffice test
```

## Implementation Milestones

### Milestone 0: Quoting Agent Invocation Contract

- Update quoting orchestration guidance so the quoting agent owns Coterie quote
  readiness, source selection, and invocation.
- Add a Coterie execution-packet shape that includes `target`, flexible
  `facts`, `coverageIntent`, `effectiveDate`, `quoteReadinessAssessment`, and
  safety flags without forcing a second rigid canonical lead schema.
- Teach the quoting agent to call `coterie_portal` only when it believes the
  lead is ready for Coterie execution.
- Keep a defensive Coterie-side validation step that rejects incomplete payloads
  before opening Browserbase.

Exit criteria:

- The quoting agent can explain why it is invoking Coterie.
- Missing data stays in quoting-agent/customer/operator question flow, not in
  the Coterie browser agent.
- Tests prove incomplete quoting-agent payloads do not open Browserbase or attempt
  Coterie login.
- Tests prove complete quoting-agent payloads enqueue the Coterie source with the
  intended `target` and safety flags.

### Milestone 1: Shared Browser Portal Runtime

- Add `quote_sources/browser_portal/*`.
- Add Browserbase + Stagehand setup helpers for browser sessions, observation,
  actions, extraction, and downloads.
- Refactor `login-smoke.ts` to use the shared Browserbase and OTP clients.
- Add unit tests for Browserbase request construction, Stagehand setup, OTP
  polling, download metadata, and redaction.

Exit criteria:

- Existing login smoke still works.
- No duplicated Browserbase session code remains inside Coterie-specific files
  except thin config.
- Stagehand is the default portal-observation/action layer, with deterministic
  selector helpers available for stable fields.

### Milestone 2: Coterie Login And Quotes Page Read

- Add `config.ts`, `login.ts`, `quotes-page.ts`, and selector helpers.
- Implement `ensureCoterieAuthenticated`.
- Implement list-row parsing, search, drawer read, and edit navigation.
- Add read-only smoke for `/quotes`.

Exit criteria:

- A live run can authenticate in Browserbase and parse quote rows without
  creating or modifying quotes.
- Session state records auth and Browserbase replay metadata.

### Milestone 3: Runtime Contract And State

- Expand update/state schemas.
- Set registry `stub: false` only when the runtime can at least authenticate and
  park safely.
- Implement `handleUpdate` state machine for cancel, retry, missing facts,
  login, and read-only quote selection.
- Return useful `waiting_for_facts` and `waiting_for_operator` results.

Exit criteria:

- Quote-source updates no longer return `needs_build`.
- Dashboard modal shows meaningful Coterie state.

### Milestone 4: New Quote Flow To Review

- Implement the guarded `/quote` flow through business name, address, industry,
  policies, business details, and quote review.
- Add appetite gate parsing and unknown-question handoff.
- Extract quote summary and upsert Coterie application results.
- Download the quote PDF from the Browserbase/Stagehand session and save it to
  the Kinro bucket through the existing document/attachment path.

Exit criteria:

- With explicit live-create permission, the agent can create a Coterie quote for
  approved test data and stop at quote review.
- Without explicit permission, the agent refuses to create and returns a clear
  operator handoff.
- Successful quote results include quote snapshot URL, stored PDF document
  reference, bucket object path, premium/total, limits, effective date, warnings,
  Browserbase replay URL, and Coterie provider quote id.

### Milestone 5: Existing Quote Edit Flow

- Implement direct `/quote/edit/{portalQuoteId}` load.
- Implement quote-list search fallback.
- Implement accordion read/update for business, contact, policy, and additional
  insured fields.
- Re-extract quote review after changes.

Exit criteria:

- The agent can refresh an existing quote and persist the updated summary.
- The agent can apply known safe field updates without touching bind/payment.

### Milestone 6: Operator UX And Observability

- Update the Coterie dashboard modal.
- Add structured events for checkpoints, Browserbase sessions, quote review, and
  handoffs.
- Add redaction checks in logs/events.
- Add retry guidance for auth, OTP timeout, and unknown portal questions.

Exit criteria:

- Operators can see exactly where the portal run stopped and why.
- No credentials, OTPs, cookies, or raw page dumps are persisted.

## Risks And Mitigations

- Portal UI changes:
  - Prefer accessible names and labels when stable.
  - Keep Coterie-specific selectors isolated in `coterie-selectors.ts`.
  - Add DOM fixture tests for key pages.
- OTP timing:
  - Create the OTP request before triggering email when possible.
  - Poll with a clear timeout and consume immediately after use.
  - Surface timeouts as retryable handoffs.
- Accidental bind/payment:
  - Centralize forbidden-action selectors.
  - Test that forbidden actions are not callable.
  - Stop every flow at quote review.
- Accidental quote creation in tests:
  - Require `allowQuoteCreation: true`.
  - Keep dry-run smoke as the default.
  - Gate live quote creation behind an explicit env variable.
- Sensitive data leakage:
  - Redact credentials and OTPs.
  - Store Browserbase replay URLs only in internal session state/events.
  - Avoid persisting raw HTML or unrestricted screenshots.
- Ambiguous appetite or unknown questions:
  - Return `waiting_for_operator` with captured prompt labels and visible
    choices.
  - Extend `coterie-quote-flow-questions.md` after verified manual exploration.

## Open Questions

- Does production Coterie login require a password in addition to broker email
  and OTP, or is email + OTP sufficient for all sessions?
- Should Browserbase use a persistent context per carrier account after V0 to
  reduce OTP frequency?
- Which exact Arrakis fact key should represent Coterie industry labels when the
  lead has only NAICS or a free-text operations description?
- Should Coterie quote PDFs be downloaded and attached to Arrakis notes in V0,
  or is quote snapshot URL plus extracted quote summary enough?
- For existing quotes, should the source session own the Coterie `portalQuoteId`
  as soon as it is discovered, or should it also be mirrored onto
  `quote_applications.provider_application_id` immediately?
  off when the portal asks an unknown question.
- Portal automation for other carriers. The shared Browserbase modules should be
  designed for reuse, but only Coterie is implemented in this plan.

## Current Starting Point

Already present:

- `coterie_portal_agent` exists as a quote-source subagent stub:
  - `manifest.ts` declares `sourceKey = "coterie_portal"`,
    `kind = "browser_portal"`, and a portal-shaped state schema.
  - `runtime.ts` records updates, then returns `status: "needs_build"`.
  - `bundle.local.json` exists.
- `login-smoke.ts` already proves the key primitives:
  - create a Browserbase session,
  - connect Playwright over CDP,
  - create a Coterie OTP request in the Arrakis Data API,
  - poll for OTP,
  - fill the OTP,
  - consume the OTP request.
- Data API already has Coterie OTP support:
  - `POST /v1/internal/quote-sources/coterie-portal/otp-requests`
  - `GET /v1/internal/quote-sources/coterie-portal/otp-requests/:id?includeOtp=true`
  - `POST /v1/internal/quote-sources/coterie-portal/otp-requests/:id/consume`
  - inbound email projection into `arrakis.carrier_login_otps`.
- Quote-source sessions and updates already exist, including pause/resume/retry
  and dashboard rendering.
- Prior quote-flow capture already documents most Coterie questions in
  `coterie-quote-flow-questions.md`.

Live portal observations from the logged-in Chrome session on 2026-06-12:

- The quotes URL lands at `/quotes` when authenticated.
- The quotes list includes search, user filter, quote type filter, archive
  selection, row checkboxes, `Edit Quote ...`, and `See Quote Details ...`.
- `New Quote` navigates to `/quote` and starts with:
  - heading `What is the business name?`
  - `Legal Business Name`
  - optional DBA checkbox
  - `Next Step: Business Address`
- Existing quotes open at `/quote/edit/{portalQuoteId}`.
- The edit page exposes:
  - total due,
  - yearly/monthly payment toggle,
  - `Download Quote`,
  - `Quote Snapshot`,
  - policy effective date,
  - `Bind as`,
  - policy selection/detail controls,
  - `Additional Insureds`, `Business Details`, and `Contact Details`
    accordions,
  - `Continue to pay` as the bind/payment boundary.
- `Business Details` includes legal name, DBA, disabled industry/address,
  estimated revenue, annual payroll, employees, business start date, and
  premises square footage.
- `Contact Details` includes first name, last name, phone, policyholder email,
  mailing address, disabled city/state/zip, and `Update Address Manually`.
- `Additional Insureds` starts with an `Individual Additional Insured` checkbox.

## Architecture

Keep the existing quote-source contract, but make Oracle the orchestration owner.
Replace the Coterie stub internals with an execution agent whose tools own
browser side effects.

### Oracle-Owned Invocation

Oracle is the main quote orchestrator. It owns carrier quote readiness, carrier
selection, and source invocation.

Oracle responsibilities:

- Decide whether Coterie is the right carrier/source for the lead based on the
  lead, conversation, research/appetite state, workflow state, and operator
  instructions.
- Ask customer/operator follow-up questions when Coterie-relevant data is
  missing.
- Call `coterie_portal` only when Oracle believes Coterie has enough data to
  produce a meaningful quote.
- Send the Coterie agent the known quote facts and the intended action
  (`new_quote` or `edit_existing`).
- Decide what to do after the Coterie agent returns a quote, handoff, decline,
  or failure.

Coterie agent responsibilities:

- Validate the Oracle-provided payload defensively.
- Map the provided facts to Coterie portal fields.
- Log in, complete OTP, operate the browser, and extract quote artifacts.
- Return structured results or structured handoff reasons.
- Never decide long-running customer intake strategy. If the payload is
  incomplete or ambiguous, return a structured handoff/rejection and let Oracle
  decide the next question or operator action.

This keeps readiness, missing-data collection, and carrier-routing decisions in
one place while keeping the Coterie agent focused on portal execution.

Recommended shared module:

```text
apps/arrakis/backoffice/agents/quote_sources/browser_portal/
  browserbase.ts
  artifacts.ts
  auth.ts
  navigation.ts
  otp.ts
  selectors.ts
  state.ts
  telemetry.ts
```

Responsibilities:

- `browserbase.ts`
  - create Browserbase sessions,
  - connect Playwright via `chromium.connectOverCDP`,
  - compute replay URLs,
  - close sessions reliably,
  - attach carrier/source metadata.
- `auth.ts`
  - generic portal auth result types,
  - login-state detection helpers,
  - no credential logging.
- `otp.ts`
  - generic OTP request/poll/consume client over the Arrakis Data API,
  - timeout and retry helpers,
  - never include OTP codes in events, logs, checkpoints, or state.
- `selectors.ts`
  - stable element helpers around accessible roles, labels, text, and fallback
    CSS,
  - explicit uniqueness checks before click/fill,
  - consistent timeout errors.
- `navigation.ts`
  - checkpointed step runner,
  - idempotent `gotoUnlessAlreadyThere`,
  - side-effect boundaries for submit/save/bind actions.
- `artifacts.ts`
  - screenshot/replay metadata capture,
  - optional future GCS upload if screenshots need to live outside Browserbase.
- `state.ts`
  - normalized portal session state transitions,
  - checkpoint append/merge helpers,
  - field-ledger helpers for "known", "entered", "blocked", and "unknown".
- `telemetry.ts`
  - quote-source events, timings, Browserbase session ids, and redaction.

Coterie-specific module layout:

```text
apps/arrakis/backoffice/agents/coterie_portal_agent/
  AGENTS.md
  SYSTEM_PROMPT.md
  config.ts
  login.ts
  runtime.ts
  manifest.ts
  quote-target.ts
  quotes-page.ts
  new-quote-flow.ts
  edit-quote-flow.ts
  quote-review.ts
  field-mapping.ts
  coterie-selectors.ts
  login-smoke.ts
  test/
```

Responsibilities:

- `config.ts`
  - read `COTERIE_PORTAL_QUOTES_URL`,
    `COTERIE_PORTAL_BROKER_EMAIL`, optional `COTERIE_PORTAL_PASSWORD`,
    `KINRO_CARRIER_OTP_RECIPIENT_EMAIL`, Browserbase env, and Data API env.
  - default quotes URL to `https://dashboard-v2.coterieinsurance.com/quotes`.
- `login.ts`
  - open the quotes URL,
  - detect authenticated vs login/OTP screens,
  - submit email/password only when required by the page,
  - create OTP requests before triggering the OTP email when possible,
  - poll for OTP with a bounded timeout,
  - fill and consume OTP,
  - return an auth checkpoint without exposing secrets or OTP values.
- `quote-target.ts`
  - validate Oracle's intended target: new quote or existing quote edit.
  - require an explicit target shape for existing quote edits.
  - require an explicit `allowQuoteCreation: true` for any action that will
    create a real Coterie quote.
- `quotes-page.ts`
  - parse list rows,
  - search by business/contact,
  - open quote detail drawer,
  - open quote edit,
  - navigate to `/quote` for new quote only after the creation guard passes.
- `new-quote-flow.ts`
  - business name,
  - address/manual address fallback,
  - industry search and appetite gate parsing,
  - policy selection,
  - business details,
  - quote review extraction.
- `edit-quote-flow.ts`
  - load `/quote/edit/{portalQuoteId}` when known,
  - otherwise find the row on `/quotes`,
  - open and update accordions,
  - support safe recalculation/update actions only when the page clearly
    exposes them.
- `quote-review.ts`
  - extract total due, payment cadence, product/policy selections, limits,
    deductibles, quote snapshot URL, download availability, effective date, and
    warnings.
- `field-mapping.ts`
  - map Oracle-provided lead/quote facts into Coterie field values.
  - normalize money, dates, phone, employee counts, business start month/year,
    and coverage choices.
  - validate that required portal fields are present before opening Browserbase.
- `runtime.ts`
  - keep `QuoteSourceSubagent.handleUpdate` as the only public entrypoint.
  - run deterministic browser tools and a small LLM decision layer only for
    ambiguous mapping, unknown portal questions, and operator-handoff summaries.

## Update Contract Changes

Expand `coteriePortalUpdateSchema` from generic facts into a typed browser
portal request while keeping backward compatibility with existing
`facts_updated` payloads.

Avoid making the Coterie agent own a separate orchestration model. Oracle should
send a practical execution packet: target, facts, coverage intent, effective
date, operator context, and safety flags. The facts object can stay flexible so
Oracle can pass the best available source facts without forcing every upstream
system to conform to a second rigid schema. Coterie's `field-mapping.ts` owns the
carrier-specific translation from those facts into portal fields.

Recommended shape:

```ts
const coteriePortalTargetSchema = z.discriminatedUnion("mode", [
  z.object({
    mode: z.literal("new_quote"),
    allowQuoteCreation: z.boolean().default(false),
  }),
  z.object({
    mode: z.literal("edit_existing"),
    portalQuoteId: z.string().uuid().optional(),
    businessName: z.string().optional(),
    contactName: z.string().optional(),
  }),
]);
```

Add update fields:

- `target`
- `facts`
- `coverageIntent`
- `effectiveDate`
- `operatorAnswer`
- `oracleAssessment`
  - why Oracle selected Coterie,
  - whether this is ready to execute or needs operator review,
  - any appetite/risk caveats Oracle wants preserved.
- `portalEvent`
- `dryRun`

Behavior:

- `start` should normally be created by Oracle only after Oracle believes the
  lead is Coterie-ready.
- `start` with no target defaults to `new_quote` only when the source session
  has no previous `portalQuoteId`. It must still require
  `allowQuoteCreation: true` before creating a real portal quote.
- `facts_updated` merges facts and either continues the current portal flow or
  parks in `waiting_for_facts`.
- `operator_answer` resolves a specific `operatorHandoff`.
- `retry` resumes from the latest durable checkpoint.
- `cancel` closes the Browserbase session if open and returns `cancelled`.

The Coterie agent should not ask customer intake questions itself. If validation
fails or the portal exposes a question the agent cannot answer safely, the agent
returns a structured handoff/rejection and Oracle decides the next question,
operator task, or retry.

## State Schema Changes

Keep the current high-level phases, but add concrete browser/session fields.

Recommended state additions:

```ts
{
  portalPhase:
    | "not_started"
    | "login"
    | "business_profile"
    | "coverage_selection"
    | "underwriting_questions"
    | "quote_review"
    | "complete",
  auth: {
    status: "unknown" | "authenticated" | "login_required" | "otp_wait" | "failed",
    brokerEmail: string | null,
    lastAuthenticatedAt: string | null,
    lastOtpRequestId: string | null
  },
  browserbase: {
    sessionId: string | null,
    replayUrl: string | null,
    startedAt: string | null,
    endedAt: string | null
  },
  target: {
    mode: "new_quote" | "edit_existing" | null,
    portalQuoteId: string | null,
    businessName: string | null,
    contactName: string | null
  },
  fieldLedger: {
    knownFacts: Record<string, unknown>,
    enteredFields: Record<string, unknown>,
    blockingFacts: string[],
    unknownPortalPrompts: Array<{ label: string, phase: string, at: string }>
  },
  quoteReview: {
    status: "not_reached" | "quoted" | "pending_completion" | "declined" | "unknown",
    totalDueToday: string | null,
    paymentCadence: "yearly" | "monthly" | null,
    products: string[],
    effectiveDate: string | null,
    quoteSnapshotUrl: string | null,
    downloadQuoteEnabled: boolean | null,
    warnings: string[]
  }
}
```

Do not persist:

- passwords,
- OTP codes,
- raw full-page HTML,
- cookies,
- local storage,
- large screenshots with unrestricted access.

## Login And OTP Flow

V0 flow:

1. Create a Browserbase session with metadata:
   - `carrier: "coterie"`
   - `sourceKey: "coterie_portal"`
   - `quoteWorkflowId`
   - `quoteSourceSessionId`
2. Connect with Playwright over CDP.
3. Navigate to `COTERIE_PORTAL_QUOTES_URL`.
4. Detect whether the portal is already authenticated.
5. If login is required:
   - create an OTP request in the Data API,
   - submit configured broker email and password if the page asks for them,
   - trigger OTP,
   - poll every 5 seconds until the OTP request is `otp_received`, `failed`,
     `expired`, or the timeout is reached,
   - fill OTP,
   - consume the OTP request.
6. Confirm the page reaches `/quotes` or another authenticated portal page.
7. Record a login checkpoint with Browserbase session id, replay URL, and OTP
   request id, but no OTP code.

Timeout handling:

- OTP timeout should return `waiting_for_operator` when the browser session is
  still usable and a human can intervene.
- Hard auth failures should return `failed` with a redacted reason.
- Expired OTP requests should create a new request on `retry`, not reuse the
  expired row.

Secrets/env:

- `BROWSERBASE_API_KEY`
- `BROWSERBASE_PROJECT_ID`
- `BROWSERBASE_API_BASE_URL` optional
- `BROWSERBASE_APP_BASE_URL` optional
- `COTERIE_PORTAL_QUOTES_URL`
- `COTERIE_PORTAL_BROKER_EMAIL`
- `COTERIE_PORTAL_PASSWORD` optional, depending on actual login page behavior
- `KINRO_CARRIER_OTP_RECIPIENT_EMAIL`
- `KINRO_ARRAKIS_DATA_API_BASE_URL`
- `KINRO_ARRAKIS_DATA_API_TOKEN`

## Portal Flow

### New Quote

Guardrails:

- Do not start a real new quote unless `allowQuoteCreation: true` is present.
- Local/dev smoke tests default to `dryRun: true` and stop before any submit that
  would create a quote.
- Never click `Continue to pay`.

Flow:

1. Preflight facts from lead/workflow:
   - business name,
   - industry class or NAICS/search phrase,
   - address,
   - revenue,
   - payroll,
   - employees,
   - years in business or start date,
   - coverage intent.
2. Navigate to `/quote`.
3. Business name:
   - fill legal business name,
   - set DBA when provided,
   - continue to address.
4. Address:
   - prefer verified autocomplete when exact match is available,
   - use `Update Address Manually` when autocomplete cannot be verified,
   - continue to industry.
5. Industry:
   - search by preferred Coterie industry label, then NAICS/description fallback,
   - parse appetite result and ineligible activity warnings,
   - stop in `waiting_for_operator` if the portal shows out-of-appetite,
     ambiguous stale appetite state, or ineligible activities that need
     confirmation.
6. Policies:
   - map coverage intent to BOP, GL, and/or PL.
   - respect BOP/GL mutual exclusion.
   - for BOP, select required property coverage when facts support it.
7. Business details:
   - fill revenue, payroll, employees, start date, premises fields.
   - block on missing/unknown facts rather than guessing.
8. Underwriting/additional questions:
   - answer only known captured prompts.
   - create an operator handoff for unknown prompts.
9. Quote review:
   - extract quote result,
   - persist application/quote summary,
   - stop before payment/bind.

### Existing Quote

Flow:

1. If `portalQuoteId` is known, navigate directly to
   `/quote/edit/{portalQuoteId}`.
2. Otherwise navigate to `/quotes`, search/filter by business/contact, and select
   the unique matching row.
3. Use the row detail drawer to inspect quick status when useful.
4. Open `Edit Quote ...`.
5. Read current quote review state and accordion values.
6. Apply requested updates only for fields explicitly included in facts or
   operator answers.
7. Re-extract quote review.
8. Persist `portalQuoteId` and quote summary back to source session state and
   application rows.

Editable areas observed:

- policy effective date,
- payment cadence,
- BOP/GL/PL product toggles where enabled,
- PL limits, deductible, years of professional experience, prior acts,
- additional insured checkbox/options,
- business details accordion,
- contact details accordion,
- mailing address manual update.

## Result Mapping

Return `QuoteSourceHandleResult` values:

- `waiting_for_facts`
  - defensive validation found that Oracle's payload is incomplete before
    opening or continuing portal flow.
  - Oracle owns the next question or operator task.
- `waiting_for_operator`
  - OTP/login handoff, appetite ambiguity, unknown portal question, CAPTCHA,
    PDF/download issue, or any action that would approach bind/payment.
- `quoted`
  - quote review reached, premium/summary extracted, quote snapshot/PDF
    artifacts handled when available, and the flow stopped before bind/payment.
- `declined`
  - portal gives a clear out-of-appetite/decline result.
- `failed`
  - Browserbase/session/automation failure that retry may or may not fix.
- `cancelled`
  - cancel update.

Structured handoff reasons:

- `incomplete_oracle_payload`
- `otp_timeout`
- `login_failed`
- `captcha_or_security_challenge`
- `portal_appetite_ambiguous`
- `portal_decline_confirmation_needed`
- `unknown_portal_question`
- `pdf_download_failed`
- `quote_snapshot_unavailable`
- `bind_or_payment_boundary`
- `selector_or_portal_changed`

Every handoff should include:

- `reasonCode`,
- short human-readable `summary`,
- current `portalPhase`,
- Browserbase replay URL when available,
- latest checkpoint,
- visible prompt/labels/options when safe to capture,
- recommended Oracle/operator next action.

Application upserts:

- `carrier: "coterie"`
- `product`: `BOP`, `CGL`, or `PL`
- `providerApplicationId`: Coterie portal quote id when available
- `status`: `quoted`, `declined`, `pending_completion`, or `failed`
- `quoteResult`:
  - total due,
  - premium/total amount fields used by Arrakis summaries,
  - payment cadence,
  - effective date,
  - selected products,
  - limits/deductibles,
  - quote snapshot URL,
  - quote PDF attachment/document reference when download succeeds,
  - quote PDF download status when it does not,
  - Browserbase replay URL,
  - warnings,
  - Coterie provider quote id,
  - bindable flag must be false.

PDF handling:

- Download the Coterie quote PDF when the portal exposes `Download Quote`.
- Store the PDF through the existing Arrakis document/attachment path so it can
  be sent to the customer by email.
- Persist a timeline/event reference to the stored PDF.
- If download fails, keep the quote snapshot URL and return
  `waiting_for_operator` with `reasonCode = "pdf_download_failed"` rather than
  silently marking the quote fully delivered.

Compliance boundary:

- The Coterie agent must never bind, pay, issue, save payment details, click
  `Continue to pay`, or cross any equivalent bind/payment boundary.
- The Coterie agent must never represent a quote as final or bound.
- Licensed/operator review owns final quote verification and any customer-facing
  language before purchase.

## Dashboard Changes

Minimal V0:

- Update `CoteriePortalSourceModal` to show:
  - auth status,
  - Browserbase replay link,
  - target mode and portal quote id,
  - quote review summary,
  - blocking facts,
  - operator handoff reason,
  - latest checkpoints.
- Remove the stub-only blue notice once `stub: false`.
- Add copy that clearly marks `Continue to pay` as not automated.

Optional V0.5:

- Add an operator action to send an `operator_answer` update from the modal.
- Add a "resume in Browserbase replay" link if Browserbase supports it for the
  session.

## Testing Plan

Unit tests:

- Browserbase session creation request shape and replay URL construction.
- OTP request/poll/consume behavior, including timeout, expired, failed, and
  redaction.
- Coterie fact-to-field mapping:
  - money formatting,
  - date/start-date formatting,
  - coverage intent mapping,
  - address/manual fallback inputs.
- State/checkpoint helpers.
- Quote result extraction from saved DOM fixtures.
- Guard that `Continue to pay` and equivalent bind/payment selectors are never
  clicked by allowed automation steps.

Integration tests with mocked portal:

- Authenticated `/quotes` path.
- Login + OTP path.
- New quote happy path to quote review.
- Appetite decline path to `declined`.
- Unknown question path to `waiting_for_operator`.
- Existing quote edit path via `/quote/edit/{id}`.
- Existing quote selection path via `/quotes` search.

Smoke tests:

- Keep `login-smoke.ts`, but refactor it onto the shared Browserbase/auth/OTP
  modules.
- Add a read-only `quotes-page-smoke` that logs in, opens `/quotes`, parses rows,
  and exits.
- Add a gated `new-quote-dry-run-smoke` that opens `/quote`, verifies the first
  step, and exits without entering data or creating a quote.
- Add live quote-creation smoke only behind an explicit environment variable,
  for example `COTERIE_LIVE_QUOTE_SMOKE_ALLOW_CREATE=true`, and use clearly
  marked test data.

Verification commands:

```bash
pnpm --filter @kinro/arrakis-backoffice test
pnpm --filter @kinro/arrakis-backoffice lint
pnpm lint
```

Use repo auth/env flow before GCP-backed or Data API integration work:

```bash
pnpm gcp:auth:status
pnpm env:doctor arrakis-backoffice --env dev
pnpm env:pull arrakis-backoffice --env dev
pnpm env:exec arrakis-backoffice --env dev -- pnpm --filter @kinro/arrakis-backoffice test
```

## Implementation Milestones

### Milestone 0: Oracle Invocation Contract

- Update Oracle/quoting orchestration guidance so Oracle owns Coterie quote
  readiness, source selection, and invocation.
- Add a Coterie execution-packet shape that includes `target`, flexible
  `facts`, `coverageIntent`, `effectiveDate`, `oracleAssessment`, and safety
  flags without forcing a second rigid canonical lead schema.
- Teach Oracle to call `coterie_portal` only when it believes the lead is ready
  for Coterie execution.
- Keep a defensive Coterie-side validation step that rejects incomplete payloads
  before opening Browserbase.

Exit criteria:

- Oracle can explain why it is invoking Coterie.
- Missing data stays in Oracle/customer/operator question flow, not in the
  Coterie browser agent.
- Tests prove incomplete Oracle payloads do not open Browserbase or attempt
  Coterie login.
- Tests prove complete Oracle payloads enqueue the Coterie source with the
  intended `target` and safety flags.

### Milestone 1: Shared Browser Portal Runtime

- Add `quote_sources/browser_portal/*`.
- Refactor `login-smoke.ts` to use the shared Browserbase and OTP clients.
- Add unit tests for Browserbase request construction, OTP polling, and
  redaction.

Exit criteria:

- Existing login smoke still works.
- No duplicated Browserbase session code remains inside Coterie-specific files
  except thin config.

### Milestone 2: Coterie Login And Quotes Page Read

- Add `config.ts`, `login.ts`, `quotes-page.ts`, and selector helpers.
- Implement `ensureCoterieAuthenticated`.
- Implement list-row parsing, search, drawer read, and edit navigation.
- Add read-only smoke for `/quotes`.

Exit criteria:

- A live run can authenticate in Browserbase and parse quote rows without
  creating or modifying quotes.
- Session state records auth and Browserbase replay metadata.

### Milestone 3: Runtime Contract And State

- Expand update/state schemas.
- Set registry `stub: false` only when the runtime can at least authenticate and
  park safely.
- Implement `handleUpdate` state machine for cancel, retry, missing facts,
  login, and read-only quote selection.
- Return useful `waiting_for_facts` and `waiting_for_operator` results.

Exit criteria:

- Quote-source updates no longer return `needs_build`.
- Dashboard modal shows meaningful Coterie state.

### Milestone 4: New Quote Flow To Review

- Implement the guarded `/quote` flow through business name, address, industry,
  policies, business details, and quote review.
- Add appetite gate parsing and unknown-question handoff.
- Extract quote summary and upsert Coterie application results.

Exit criteria:

- With explicit live-create permission, the agent can create a Coterie quote for
  approved test data and stop at quote review.
- Without explicit permission, the agent refuses to create and returns a clear
  operator handoff.

### Milestone 5: Existing Quote Edit Flow

- Implement direct `/quote/edit/{portalQuoteId}` load.
- Implement quote-list search fallback.
- Implement accordion read/update for business, contact, policy, and additional
  insured fields.
- Re-extract quote review after changes.

Exit criteria:

- The agent can refresh an existing quote and persist the updated summary.
- The agent can apply known safe field updates without touching bind/payment.

### Milestone 6: Operator UX And Observability

- Update the Coterie dashboard modal.
- Add structured events for checkpoints, Browserbase sessions, quote review, and
  handoffs.
- Add redaction checks in logs/events.
- Add retry guidance for auth, OTP timeout, and unknown portal questions.

Exit criteria:

- Operators can see exactly where the portal run stopped and why.
- No credentials, OTPs, cookies, or raw page dumps are persisted.

## Risks And Mitigations

- Portal UI changes:
  - Prefer accessible names and labels when stable.
  - Keep Coterie-specific selectors isolated in `coterie-selectors.ts`.
  - Add DOM fixture tests for key pages.
- OTP timing:
  - Create the OTP request before triggering email when possible.
  - Poll with a clear timeout and consume immediately after use.
  - Surface timeouts as retryable handoffs.
- Accidental bind/payment:
  - Centralize forbidden-action selectors.
  - Test that forbidden actions are not callable.
  - Stop every flow at quote review.
- Accidental quote creation in tests:
  - Require `allowQuoteCreation: true`.
  - Keep dry-run smoke as the default.
  - Gate live quote creation behind an explicit env variable.
- Sensitive data leakage:
  - Redact credentials and OTPs.
  - Store Browserbase replay URLs only in internal session state/events.
  - Avoid persisting raw HTML or unrestricted screenshots.
- Ambiguous appetite or unknown questions:
  - Return `waiting_for_operator` with captured prompt labels and visible
    choices.
  - Extend `coterie-quote-flow-questions.md` after verified manual exploration.

## Open Questions

- Does production Coterie login require a password in addition to broker email
  and OTP, or is email + OTP sufficient for all sessions?
- Should Browserbase use a persistent context per carrier account after V0 to
  reduce OTP frequency?
- Which exact Arrakis fact key should represent Coterie industry labels when the
  lead has only NAICS or a free-text operations description?
- Should Coterie quote PDFs be downloaded and attached to Arrakis notes in V0,
  or is quote snapshot URL plus extracted quote summary enough?
- For existing quotes, should the source session own the Coterie `portalQuoteId`
  as soon as it is discovered, or should it also be mirrored onto
  `quote_applications.provider_application_id` immediately?
