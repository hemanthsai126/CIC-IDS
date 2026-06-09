# Arrakis Settings DB Change Plan

## Goal

Add the database backing needed for the new Arrakis Settings page without changing the existing UI behavior first.

The DB work should support:

- Global Ben autopilot, using the existing DB-backed global control.
- New-lead autopilot defaults by source.
- Slack notification filters and backend-only destinations.
- Engagement hours interpreted in each lead's timezone.
- Ben-only engagement frequency thresholds.
- Daily email caps for total system email, Ben email, operator email, and per-operator email.

## Existing DB Surface To Reuse

Keep using `arrakis.ben_autopilot_controls` for the global Ben autopilot toggle. The Settings page should read and write the existing `control_key = 'global'` row through the existing API path.

Keep using lead metadata for lead-level Ben autopilot state. Explicit operator overrides on a lead should continue to live under `leads.metadata.benAutopilot`.

Keep using existing outbound records for counting engagement:

- `arrakis.messages` for SMS, WhatsApp, and email.
- `arrakis.calls` for voice calls.
- `arrakis.scheduled_events` for pending Ben steps.

Keep using lead timezone metadata where available. Engagement hours are not stored per timezone; they are stored once as local clock times and evaluated against each lead's timezone.

## New Table: `arrakis.settings`

Use one generic settings table with strict API schemas. This keeps the DB migration small while letting each setting evolve in JSON.

```sql
CREATE TABLE arrakis.settings (
  key text PRIMARY KEY,
  value jsonb NOT NULL,
  updated_at timestamptz NOT NULL DEFAULT now(),
  updated_by text NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);
```

Expected keys:

- `autopilot_default_rules`
- `slack_notifications`
- `engagement_hours`
- `engagement_frequency_thresholds`
- `email_sending_limits`

The API should validate each key with a dedicated Zod schema before writing. The DB table should stay generic, but the application should not accept arbitrary JSON shapes.

## Setting Payloads

### `autopilot_default_rules`

Stores new-lead autopilot defaults and future rule dimensions.

```json
{
  "version": 1,
  "newLeadDefaults": {
    "allSources": false,
    "sources": {
      "insuranceleads": true,
      "smartfinancial": true,
      "vapi": false,
      "manual": false,
      "web_form": false,
      "email": false,
      "referral": false,
      "policy_review": false,
      "unknown": false
    }
  },
  "rules": []
}
```

Initial implementation should apply only the `newLeadDefaults` section. The `rules` array can exist but remain empty until status/AMS conditions are implemented.

### `slack_notifications`

Stores notification filtering and backend-defined destinations. Do not store raw Slack webhook URLs here.

```json
{
  "version": 1,
  "enabled": true,
  "destinations": {
    "default": "ops",
    "inbox": "inbox",
    "renewals": "renewals"
  },
  "rules": [
    {
      "id": "new-leads",
      "name": "New leads",
      "enabled": true,
      "destinationKey": "ops",
      "eventTypes": ["lead.created"],
      "sourceIncludes": [],
      "sourceExcludes": [],
      "statusIncludes": [],
      "cooldownMinutes": 15
    }
  ]
}
```

`destinationKey` should map to backend configuration or secret references. The settings UI should only display safe labels.

### `engagement_hours`

Stores allowed local clock windows. The same clock window is interpreted in each lead's timezone.

```json
{
  "version": 1,
  "enabled": true,
  "days": [1, 2, 3, 4, 5],
  "startTime": "08:00",
  "endTime": "18:00",
  "fallbackTimezone": "America/Chicago",
  "allowReactiveRepliesOutsideHours": true
}
```

`fallbackTimezone` is only used when a lead has no stored or inferred timezone. It is not user-facing in the first Settings UI.

### `engagement_frequency_thresholds`

Stores Ben-only thresholds. These should not block manual operator sends.

```json
{
  "version": 1,
  "enabled": true,
  "perLead": {
    "maxTouchesPerDay": 3,
    "maxTouchesPerSevenDays": 8,
    "maxSmsOrWhatsappPerDay": 2,
    "maxEmailsPerDay": 2,
    "maxVoiceCallsPerDay": 1,
    "maxNoResponseFollowUpsBeforePause": 5
  },
  "cooldowns": {
    "minMinutesBetweenOutbound": 60,
    "minMinutesAfterCustomerInboundBeforeProactiveFollowup": 15
  }
}
```

Counts should be computed from existing `messages` and `calls` rows where `direction = 'outbound'` and the actor/source is Ben.

### `email_sending_limits`

Stores daily email caps.

```json
{
  "version": 1,
  "enabled": true,
  "daily": {
    "total": 100,
    "ben": 100,
    "operator": 100,
    "perOperator": 25
  }
}
```

These caps should be checked immediately before provider send, not just when scheduling.

## New Table: `arrakis.email_send_daily_usage`

Use a counter table for race-safe daily email caps. Existing `arrakis.messages` can still be the source of truth for message history, but a counter row makes cap checks atomic.

```sql
CREATE TABLE arrakis.email_send_daily_usage (
  usage_date date NOT NULL,
  scope text NOT NULL,
  actor_id text NOT NULL DEFAULT '',
  sent_count integer NOT NULL DEFAULT 0,
  updated_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (usage_date, scope, actor_id),
  CHECK (scope IN ('total', 'ben', 'operator', 'operator_user')),
  CHECK (sent_count >= 0)
);
```

Scope behavior:

- `total` with `actor_id = ''`: all Arrakis outbound email sends.
- `ben` with `actor_id = ''`: Ben-authored email sends.
- `operator` with `actor_id = ''`: all operator-authored email sends.
- `operator_user` with `actor_id = <WorkOS user id>`: a specific operator.

Cap check flow:

1. Start a DB transaction before calling SendGrid.
2. Lock or upsert the relevant usage rows.
3. Check configured caps.
4. If a cap is exceeded, abort before provider send and return a clear blocked reason.
5. If allowed, increment rows and proceed with provider send.
6. If provider send fails before acceptance, decrement the counters in the same failure path or record the send as not accepted.

For the first implementation, count only accepted provider sends. If we later need strict attempt caps, add a separate attempt counter.

## New Table: `arrakis.slack_notification_deliveries`

Use this for dedupe and delivery failure logging.

```sql
CREATE TABLE arrakis.slack_notification_deliveries (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  rule_id text NOT NULL,
  destination_key text NOT NULL,
  event_type text NOT NULL,
  subject_type text NULL,
  subject_id uuid NULL,
  lead_id uuid NULL REFERENCES arrakis.leads(id) ON DELETE SET NULL,
  idempotency_key text NOT NULL,
  status text NOT NULL,
  status_code integer NULL,
  error text NULL,
  sent_at timestamptz NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  metadata jsonb NOT NULL DEFAULT '{}'::jsonb,
  UNIQUE (idempotency_key),
  CHECK (status IN ('sent', 'skipped', 'failed'))
);

CREATE INDEX slack_notification_deliveries_lead_created_idx
  ON arrakis.slack_notification_deliveries (lead_id, created_at DESC);
```

The notification sender should insert or upsert with a stable idempotency key before posting. This prevents duplicate Slack posts for the same event/rule/destination.

## No New Table For Ben Frequency Counts

Do not add a separate Ben engagement counter table initially. Use existing history:

- `arrakis.messages` for outbound SMS, WhatsApp, and email.
- `arrakis.calls` for outbound voice.
- `arrakis.scheduled_events` for pending events that should be skipped or rescheduled.

Suggested indexes if current performance is not enough:

```sql
CREATE INDEX messages_lead_outbound_channel_at_idx
  ON arrakis.messages (
    lead_id,
    channel,
    COALESCE(sent_at, created_at) DESC
  )
  WHERE direction = 'outbound';

CREATE INDEX calls_lead_outbound_at_idx
  ON arrakis.calls (
    lead_id,
    COALESCE(started_at, created_at) DESC
  )
  WHERE direction = 'outbound';
```

Only add these if query plans show the current indexes are insufficient.

## No Schema Change For Global Autopilot

Do not migrate global autopilot into `arrakis.settings` now.

Reasons:

- It already works.
- It already has update metadata.
- Existing data API and metrics code read `arrakis.ben_autopilot_controls`.
- Moving it now would increase risk without improving the Settings UI.

The Settings API can aggregate from both:

- `arrakis.ben_autopilot_controls` for the global Ben switch.
- `arrakis.settings` for all new settings.

## Seed Defaults

Insert defaults during migration or first-read fallback. Prefer first-read fallback in code plus an optional seed migration so missing rows do not break local/dev DBs.

Default values:

- `autopilot_default_rules`: all sources off except `insuranceleads` and `smartfinancial` on, matching current source default behavior.
- `slack_notifications`: enabled, with current new-lead and inbox behavior represented as rules.
- `engagement_hours`: weekdays, `08:00` to `18:00`, lead-local, Central Time fallback.
- `engagement_frequency_thresholds`: Ben per-lead daily defaults from the Settings UI.
- `email_sending_limits`: total `100`, Ben `100`, operator `100`, per-operator `25`.

## Rollout Order

1. Add `arrakis.settings`.
2. Add `arrakis.email_send_daily_usage`.
3. Add `arrakis.slack_notification_deliveries`.
4. Add default settings readers in the data API.
5. Add settings write APIs with strict validation.
6. Wire Settings UI to read/write settings.
7. Enforce Slack filters and delivery dedupe.
8. Enforce engagement hours and Ben thresholds.
9. Enforce email caps at the email provider send boundary.

## Backfill

No required destructive backfill.

Recommended one-time seed:

- Create `autopilot_default_rules` from current hardcoded Ben source defaults.
- Create `slack_notifications` from current environment-backed notification behavior.
- Create `engagement_hours`, `engagement_frequency_thresholds`, and `email_sending_limits` from the default payloads above.

Do not rewrite existing leads when adding default autopilot rules. Apply new-lead defaults only to future leads unless a separate explicit retroactive workflow is built.

## Validation And Safety

Settings writes should reject:

- Unknown setting keys.
- Invalid JSON shape.
- Negative caps or thresholds.
- Invalid day numbers.
- Invalid time strings.
- Slack rules with unknown destination keys.
- Email sub-caps that exceed total cap unless the product intentionally allows oversubscription.

Runtime enforcement should be fail-closed for risky outbound actions:

- If email caps cannot be read, block automated Ben email and allow manual operator email only if product decides that is acceptable.
- If engagement hours cannot be read, use safe defaults.
- If Slack settings cannot be read, skip Slack rather than blocking lead/inbox processing.

## Testing Plan

Add unit tests for:

- Settings payload validation.
- Default settings fallback when rows are missing.
- New-lead autopilot default resolution by source.
- Lead-local engagement hour evaluation across Pacific, Central, Eastern, and unknown timezone fallback.
- Ben threshold blocking using existing message/call rows.
- Email cap increments and blocks inside a transaction.
- Slack notification dedupe by idempotency key.

Add integration-style repository tests for:

- Reading and writing `arrakis.settings`.
- Concurrent email cap checks against `arrakis.email_send_daily_usage`.
- Slack delivery insert/upsert behavior.
