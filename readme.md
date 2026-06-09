# Arrakis Settings Reference

## Purpose

The Arrakis `Settings` page centralizes operational controls for Ben autopilot, Slack notifications, Ben engagement windows, Ben frequency limits, and outbound email caps.

The page is visible to all Arrakis users in the first implementation. Admin-only access can be added later once Arrakis has an explicit admin role or allowlist.

## Page Location

- Dashboard route: `/settings`
- Navigation label: `Settings`
- Preview route: `https://arrakis-preview-vvvkve42ma-uc.a.run.app/settings`

The page reads settings on load and writes changes through dashboard API routes that proxy to the Arrakis data API.

## Save Model

The main `Save settings` button writes all DB-backed setting sections:

- `autopilot_default_rules`
- `slack_notifications`
- `engagement_hours`
- `engagement_frequency_thresholds`
- `email_sending_limits`

The global Ben autopilot toggle is separate. It uses the existing Ben autopilot endpoint and existing global autopilot table rather than the generic settings table.

Settings are validated by backend schemas before they are persisted. Invalid shapes, invalid time strings, invalid day numbers, and negative limits are rejected.

## Data Model

Most settings live in:

```text
arrakis.settings
  key text primary key
  value jsonb not null
  updated_at timestamptz not null
  updated_by text null
  created_at timestamptz not null
```

Supporting tables:

- `arrakis.email_send_daily_usage`: atomic daily email counters.
- `arrakis.slack_notification_deliveries`: Slack delivery logging and deduplication.
- `arrakis.ben_autopilot_controls`: existing global Ben autopilot control.

## Header Status Cards

The Settings page header shows:

- `Global autopilot`: current global Ben autopilot state from the existing DB-backed control.
- `Engagement time`: always displayed as `Lead local`, because configured hours are interpreted in each lead's timezone.
- `Daily email cap`: the current total daily Arrakis email cap and today's usage.
- `Slack destinations`: shown as `Backend`, because raw Slack webhook URLs are not displayed or edited in the UI.

The page also shows whether settings are currently DB-backed or only loaded from code defaults.

## Global Autopilot

### Control

`Ben autopilot enabled globally`

### What It Means

This is the global kill switch for Ben autopilot.

When it is off, Ben should not initiate autopilot actions for any lead, even if a lead-level autopilot toggle is on.

When it is on, Ben still has to pass all other guardrails:

- Lead-level autopilot enabled.
- Compliance and do-not-contact checks.
- Contactability checks.
- Engagement hours.
- Engagement frequency thresholds.
- Email sending limits when the action is email.

### Storage

Stored in the existing `arrakis.ben_autopilot_controls` table with `control_key = 'global'`.

### API

- `GET /v1/ben/autopilot`
- `PATCH /v1/ben/autopilot`

## New Lead Autopilot Defaults

### Setting Key

`autopilot_default_rules`

### Controls

The page exposes source-level toggles:

- `All sources`
- `InsuranceLeads`
- `SmartFinancial`
- `Vapi voice intake`
- `Manual lead`
- `Web form`
- `SMS lead`
- `WhatsApp lead`
- `Email lead`
- `Referral`
- `Policy review`
- `Imported / unknown`

### What It Means

These toggles decide whether new leads start with Ben autopilot enabled by default.

`All sources` is the fallback when no specific source setting matches. Source-specific settings override the fallback.

These defaults apply when new leads are created. They do not rewrite existing leads.

### Current Defaults

- `All sources`: off
- `InsuranceLeads`: on
- `SmartFinancial`: on
- `Vapi voice intake`: off
- `Manual lead`: off
- `Web form`: off
- `SMS lead`: off
- `WhatsApp lead`: off
- `Email lead`: off
- `Referral`: off
- `Policy review`: off
- `Imported / unknown`: off

### Backend Enforcement

The lead creation path resolves the normalized external source and applies the matching default to lead metadata.

Explicit lead-level operator changes still win over source defaults.

### Future Reserved Field

The setting includes a `rules` array for future status and AMS policy conditions. The first implementation stores this field but does not expose rich rule editing in the UI.

## Slack Notifications

### Setting Key

`slack_notifications`

### Controls

- `Slack notifications enabled`: master switch for all rules.
- `Default destination key`: safe destination label, not a raw webhook.
- `Rule cooldown`: the minimum number of minutes before the same Slack rule can send another alert for the same event or lead. This prevents repeated Slack posts when the same lead or inbox activity is processed more than once.
- Rule toggles:
  - `New lead created`
  - `Inbox activity`
  - `Inbound SMS or WhatsApp`
  - `Inbound email received`
  - `Missed call or voicemail`
  - `Ben handoff required`
  - `Ben send failed`
  - `AMS policy ending soon`

### What It Means

Slack settings decide which Arrakis events can send Slack notifications.

The master toggle must be on, and an individual matching rule must also be enabled.

Each rule has:

- `id`
- `name`
- `enabled`
- `destinationKey`
- `eventTypes`
- `sourceIncludes`
- `sourceExcludes`
- `statusIncludes`
- `ownerIncludes`
- `timezoneIncludes`
- `coverageIncludes`
- `cooldownMinutes`

The first UI mainly exposes the master switch, destination key, rule cooldown, and common rule toggles. The schema already supports additional filters.

Example: if `New lead created` has a `15` minute cooldown and the same lead-created event is seen twice within 15 minutes, Arrakis should send the first Slack alert and skip the duplicate. A different lead or a different matching rule can still send its own alert.

### Destinations

The UI stores destination keys such as:

- `ops`
- `inbox`
- `sales`
- `renewals`

Raw Slack webhook URLs are backend-only and are never shown or edited from Settings.

### Backend Enforcement

The Slack alert layer loads the DB-backed settings before posting. It matches event type and filters, checks recent delivery cooldowns, posts only for matching enabled rules, and records deliveries in `arrakis.slack_notification_deliveries`.

Slack delivery failures should not block lead ingestion or inbox processing.

### Preview Caveat

Preview settings read/write works, but real Slack posting depends on Slack webhook secrets being configured in preview.

## Engagement Hours

### Setting Key

`engagement_hours`

### Controls

- `Enforce engagement hours for proactive Ben sends`
- Allowed days:
  - Sunday through Saturday
- Start time
- End time

### What It Means

Engagement hours control when Ben autopilot can proactively contact a lead.

The configured time window is interpreted in each lead's local timezone. For example, an 8:00 AM to 6:00 PM window means 8:00 AM to 6:00 PM Pacific for a California lead and 8:00 AM to 6:00 PM Central for a Texas lead.

If a lead has no stored or inferred timezone, the backend falls back to `America/Chicago`.

### Current Defaults

- Enabled: yes
- Days: Monday through Friday
- Start: `08:00`
- End: `18:00`
- Fallback timezone: `America/Chicago`
- Reactive replies outside hours: allowed

### Backend Enforcement

Ben proactive outbound actions are blocked outside the allowed window.

Customer-initiated inbound replies can still trigger Ben responses outside hours when autopilot is otherwise allowed.

Manual operator sends are not blocked by engagement hours. The intended operator behavior is a warning, not a hard block.

## Engagement Frequency Thresholds

### Setting Key

`engagement_frequency_thresholds`

### Controls

- `Total Ben touches per lead / day`
- `Total Ben touches per lead / 7 days`
- `SMS or WhatsApp per lead / day`
- `Emails per lead / day`
- `Voice calls per lead / day`
- `No-response follow-ups before pause`

### What It Means

These thresholds limit Ben autopilot only. They do not block manual operator sends.

### Current Defaults

- Total Ben touches per lead per day: `3`
- Total Ben touches per lead per 7 days: `8`
- SMS or WhatsApp per lead per day: `2`
- Emails per lead per day: `2`
- Voice calls per lead per day: `1`
- No-response follow-ups before pause: `5`
- Minimum minutes between outbound messages: `60`
- Minimum minutes after customer inbound before proactive follow-up: `15`

The current UI exposes the per-lead threshold values. Cooldown values are part of the stored schema and backend enforcement.

### Backend Enforcement

Before Ben sends or schedules proactive outbound work, the backend checks recent outbound activity for that lead and channel.

Counts are derived from existing Arrakis message and call history rather than a separate frequency counter table.

When a threshold blocks Ben, the action is skipped or delayed according to the calling path. Operator sends remain allowed unless blocked by another guardrail such as email sending limits.

## Email Sending Limits

### Setting Key

`email_sending_limits`

### Controls

- `Total Arrakis outbound emails / day`
- `Ben outbound emails / day`
- `Operator-authored emails / day`
- `Per-operator emails / day`

The page also displays:

- Used today
- Remaining
- Usage date

### What It Means

Email limits cap outbound email volume across Arrakis. They apply close to the provider-send boundary so manual sends, Ben sends, scheduled sends, and retry paths share the same guardrail.

### Current Defaults

- Total Arrakis outbound emails per day: `100`
- Ben outbound emails per day: `100`
- Operator-authored emails per day: `100`
- Per-operator emails per day: `25`

### Backend Enforcement

Before sending email, the backend reserves capacity in `arrakis.email_send_daily_usage`.

Counters are tracked by date and scope:

- `total`
- `ben`
- `operator`
- `operator_user`

If a provider send fails before acceptance, the reserved capacity is released.

Lowering a cap below current usage is allowed, but it immediately blocks additional sends in that scope until the next usage day.

## API Summary

Dashboard proxy routes:

- `GET /api/arrakis/settings`
- `PATCH /api/arrakis/settings/:key`

Data API routes:

- `GET /v1/settings`
- `PATCH /v1/settings/:key`
- `GET /v1/ben/autopilot`
- `PATCH /v1/ben/autopilot`

`GET /v1/settings` returns:

- Effective settings.
- Whether rows are persisted or defaults are being used.
- Latest update metadata.
- Today's email usage.

## Validation Summary

Backend validation rejects:

- Unknown setting keys.
- Invalid JSON payload shape.
- Invalid time strings.
- Invalid day numbers outside `0` through `6`.
- Negative caps or thresholds.
- Extremely large caps or thresholds above the schema maximum.
- Slack rules without required ids, names, destinations, or event arrays.

## Preview Test Status

Preview testing completed:

- Data API tests passed locally.
- Arrakis package builds passed locally.
- Preview authenticated `GET /v1/settings` passed.
- Preview `PATCH /v1/settings/slack_notifications` persisted a temporary marker and restored it.
- Preview no-op `PATCH` passed for all settings keys.
- Preview global autopilot `PATCH` passed.

Not tested end-to-end in preview:

- Real Slack delivery, because preview Slack webhook secrets may not be configured.
- Real outbound Ben/email sends, to avoid triggering customer-facing actions during smoke testing.
