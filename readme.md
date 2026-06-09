# Arrakis Settings Page Requirements

## Goal

Add a new Arrakis dashboard page called `Settings` for operational controls that currently live in scattered places or are hardcoded in backend behavior.

The page should let operators safely configure:

- Global Ben autopilot.
- Slack notification rules.
- Default autopilot rules for new leads by source, plus lead state and AMS policy conditions.
- Engagement hours using a single configured lead timezone.
- Ben engagement frequency thresholds.
- Global and per-actor email sending caps.

The settings should be DB-backed wherever possible and enforced by the backend paths that actually send messages, schedule events, and post Slack notifications.

## Current Context

Arrakis already has dashboard pages for Inbox, Leads, AMS, Metrics, Markets, and Licensing. The new Settings page should join that navigation as a first-class admin page.

Global Ben autopilot already exists as a DB-backed control in `arrakis.ben_autopilot_controls` with API support through the Ben autopilot endpoints. Today the control appears on the Leads page. The Settings page should become the canonical place to manage the global control. Lead-specific autopilot controls can remain on lead detail pages because those are per-lead decisions.

Slack notifications currently exist for new leads and inbox activity, but they are driven by environment webhook settings and code-level behavior. The new Settings page should make notification behavior configurable without redeploying.

Lead timezone and quiet-hours metadata already exist for leads when state can be inferred. For this first Settings version, engagement hours should use one lead-engagement timezone configured on the Settings page, with Central Time as the fallback.

## Page Structure

The Settings page should be organized into sections:

1. Global Autopilot
2. Slack Notifications
3. Default Autopilot Rules
4. Engagement Hours
5. Engagement Frequency Thresholds
6. Email Sending Limits

Each section should show:

- Current effective configuration.
- Last updated time and user.
- Whether the setting is active, disabled, or blocked by a higher-priority setting.
- Save/cancel controls.
- Validation errors before saving.

## Global Autopilot

This section controls the global Ben autopilot kill switch and the default autopilot state for new leads by source.

Required behavior:

- Show a single master toggle: `Ben autopilot enabled globally`.
- Use the existing DB-backed global Ben autopilot setting.
- Display last updated time and actor.
- When off, Ben should not initiate autopilot actions for any lead, even if lead-level autopilot is enabled.
- When on, Ben can only act for leads that are individually enabled and not blocked by guardrails such as consent, do-not-contact, engagement hours, or frequency thresholds.
- Move the global toggle out of the Leads page and into Settings as the canonical control.
- Add a `New leads autopilot defaults` subsection with one button/toggle for `All sources` and one button/toggle for each known lead source.
- Source-specific new-lead defaults should be DB-backed and should apply whenever a new lead is created from that source.
- The `All sources` default should apply to any source that does not have a more specific setting.

Recommended details:

- Link to filtered leads affected by the setting.
- Record last updated time and actor.
- Do not require two-step approval for global autopilot changes in the first version.

## Slack Notifications

This section controls which Arrakis events produce Slack notifications.

Required behavior:

- Add a master Slack notifications toggle.
- Store notification settings in the DB if possible.
- Allow multiple notification rules.
- Each rule should decide which events should notify Slack and where they should be sent.
- Slack messages should include the matching rule name or reason so operators know why they were notified.
- Backend notification code should check the DB-backed settings before sending Slack.
- If Slack notifications are off, no configured rule should send Slack messages.

Rule fields:

- Rule name.
- Enabled toggle.
- Destination, such as default operations channel, inbox channel, sales channel, renewals channel, or a secret reference to a Slack webhook.
- Event types to include.
- Event types to exclude.
- Lead source filters.
- Lead status filters.
- AMS policy filters.
- Priority filters.
- Owner or assigned operator filters.
- State/timezone filters.
- Coverage type filters.
- Deduplication and cooldown settings.

Initial event filters to support:

- New lead created.
- New lead from specific source.
- New lead with consent to contact.
- New lead missing consent.
- Lead assigned to operator.
- Inbound SMS or WhatsApp received.
- Inbound email received.
- Missed call.
- Voicemail or call summary ready.
- Ben handoff required.
- Ben autopilot paused itself.
- Ben message send failed.
- Email bounced or rejected.
- Lead replied after no response.
- Lead became high priority.
- Lead moved to quoted, bound, lost, or AMS.
- AMS policy ending soon.
- COI missing.
- Renewal follow-up due.
- Do-not-contact or opt-out detected.

Slack delivery should be resilient:

- Do not block lead ingestion or inbox processing if Slack fails.
- Log delivery failures.
- Avoid duplicate Slack posts for the same event and rule.
- Rate-limit noisy rules.

Security note:

Slack webhook URLs should be managed backend-only. The UI should let operators select from backend-defined destinations, but it should not expose or edit raw Slack webhook URLs. Store a destination key or encrypted secret reference in the DB, with actual webhook secrets managed through environment or secret storage.

## Default Autopilot Rules

This section controls whether Ben autopilot defaults on or off when a new lead is created, with additional rule dimensions for operational states and AMS conditions.

Required behavior:

- DB-backed rules.
- Operators can configure new-lead defaults for all lead sources or specific lead sources.
- The UI should present one clear toggle/button for `All sources` and one for each configured source.
- Operators can also configure rules for lead statuses and AMS conditions.
- Rules should be applicable in all matching scenarios, not only in the lead list view.
- Rules apply to new future leads by default.
- If supporting retroactive updates later, require preview and confirmation before changing existing leads.

Rule precedence:

1. Global autopilot off blocks all autopilot.
2. Do-not-contact, missing consent, invalid contact points, or compliance blocks must override every default.
3. Explicit lead-level operator override wins over source defaults.
4. The most specific enabled rule wins.
5. If no rule matches, use the fallback default.

Initial rule dimensions:

- All lead sources.
- InsuranceLeads.
- SmartFinancial.
- Vapi voice intake.
- Manual lead.
- Web form lead.
- Email lead.
- Referral lead.
- Policy review lead.
- Imported lead.
- Unknown source.
- New lead.
- Contacted lead.
- Stale contacted lead with no response.
- Quoted lead.
- Quote follow-up lead.
- Lost lead.
- Bound lead.
- Lead with consent.
- Lead without consent.
- Lead with verified phone.
- Lead with email only.
- Lead assigned to an operator.
- Unassigned lead.
- High-priority lead.
- AMS policy ending in 7 days.
- AMS policy ending in 14 days.
- AMS policy ending in 30 days.
- AMS policy missing COI.
- AMS renewal follow-up due.

Rule actions:

- Enable autopilot by default.
- Disable autopilot by default.
- Require operator review before enabling.
- Use a specific initial mission template.
- Use a specific channel preference such as SMS first, email first, or voice only after manual approval.

The page should explain whether a rule changes only future leads or also existing leads. For the first version, default rules should focus on new leads. If we later add existing-lead application, the UI should show an estimated affected count before saving.

## Engagement Hours

This section controls when Ben may proactively engage a lead.

Required behavior:

- DB-backed global engagement hours.
- Operators can select allowed days of week.
- Operators can set the allowed start and end time.
- Use a single lead-engagement timezone from Settings for all leads in the first version.
- Fall back to Central Time when no timezone has been configured.
- Block Ben autopilot from proactive outbound engagement outside allowed hours.
- If the customer texts, emails, or calls first during non-engagement hours, Ben may reply to that inbound conversation when autopilot is otherwise allowed.
- Keep the current manual operator send flow. If an operator sends a message to a specific lead outside configured engagement hours, show a small warning but allow the operator to continue.

Configurable fields:

- Allowed days of week.
- Start time.
- End time.
- Lead-engagement timezone.
- Holiday exceptions.

The backend should distinguish proactive sends from reactive replies. Scheduled follow-ups should be delayed to the next allowed window, while reactive replies can proceed if the latest customer inbound event is the trigger.

## Engagement Frequency Thresholds

This section controls how often Ben can engage leads, especially when a person is not responding.

Required behavior:

- DB-backed thresholds.
- Enforced before Ben schedules or sends messages.
- Thresholds should be evaluated per lead and per channel.
- Ben should never exceed configured thresholds.
- When a threshold blocks an action, the system should record why and optionally create an inbox item for operator review.
- These thresholds should not block manual operator sends. Operator sends can still count against email sending limits where those limits apply.

Initial thresholds:

- Max total outbound touches per lead per day.
- Max total outbound touches per lead per 7 days.
- Max SMS or WhatsApp messages per lead per day.
- Max emails per lead per day.
- Max voice calls per lead per day.
- Minimum cooldown between outbound messages.
- Minimum cooldown after customer inbound before the next proactive follow-up.
- Maximum no-response follow-ups before pausing autopilot.
- Maximum failed-send retries.
- Maximum cross-channel touches in a sequence.

Suggested channel rules:

- SMS/WhatsApp should have stricter caps than email.
- Voice should require a longer cooldown after missed calls or voicemail.
- Customer replies should reset no-response counters but not erase compliance or daily caps.
- Manual operator sends should not be blocked by Ben engagement frequency thresholds.

The page should show what happens when Ben hits a threshold:

- Skip the action.
- Reschedule for later.
- Pause autopilot for that lead.
- Create an inbox item.
- Notify Slack if a matching Slack rule exists.

## Email Sending Limits

This section controls global outbound email volume.

Required behavior:

- DB-backed email sending caps.
- Enforced before email is sent through SendGrid or any other provider.
- If the total daily cap is reached, block further outbound email sends.
- Support separate caps for total system email, Ben-authored email, and per-operator email.
- Show current usage and remaining quota for the current day.
- Log blocked sends with a clear reason.

Initial caps:

- Total Arrakis outbound emails per day.
- Ben autopilot outbound emails per day.
- Ben scheduled follow-up emails per day.
- Operator-authored emails per day.
- Per-operator outbound emails per day.
- Optional per-lead emails per day.
- Optional per-domain emails per day.

Validation:

- Ben and operator sub-caps should not exceed the total cap unless the product intentionally allows oversubscription.
- Caps should be non-negative integers.
- Saving a lower cap below current usage should be allowed, but it should immediately block additional sends until the next window.
- The UI should warn when current usage is already over the new cap.

Enforcement should happen close to the send operation, not only in the scheduler. This avoids bypasses from manual sends, retry paths, or future email features.

## Data Model Direction

Use DB-backed settings with typed payloads. A generic settings table would make sense for most new settings:

```text
arrakis.settings
  key text primary key
  value jsonb not null
  updated_at timestamptz not null
  updated_by text null
```

Potential keys:

- `slack_notifications`
- `autopilot_default_rules`
- `engagement_hours`
- `engagement_frequency_thresholds`
- `email_sending_limits`

The existing `arrakis.ben_autopilot_controls` table can continue to own the global Ben autopilot toggle unless we decide to migrate it later. The Settings page can read from both the existing global control and the new settings store.

## API Direction

Add dashboard-facing API coverage for settings:

- `GET /v1/settings`
- `GET /v1/settings/:key`
- `PATCH /v1/settings/:key`
- Existing `GET /v1/ben/autopilot`
- Existing `PATCH /v1/ben/autopilot`

Settings responses should include:

- Current value.
- Default value if no DB value exists.
- Effective value after applying defaults.
- Last updated metadata.
- Validation warnings.

Write APIs should validate payloads with strict schemas. Invalid settings should never be persisted.

## Enforcement Points

Settings are only useful if the backend paths enforce them.

Slack notification settings should be checked in the Slack alert layer before a payload is posted.

Default autopilot rules should be checked when:

- A lead is created.
- A lead source is normalized.
- A lead status changes.
- A lead enters AMS or a policy renewal condition.
- A bulk retroactive apply is explicitly confirmed.

Engagement hours and Ben frequency thresholds should be checked when:

- Ben creates scheduled mission steps.
- The scheduler claims due events.
- Ben sends SMS, WhatsApp, email, or voice calls.
- Ben reacts to customer inbound messages.

Manual operator sends should check engagement hours only to show a warning. They should not be blocked by engagement hours.

Email limits should be checked immediately before provider send, so all sending paths share the same cap.

## Product Decisions

These decisions are set for the first implementation:

- Settings should be visible to all Arrakis users in the first implementation.
- The current repo has WorkOS organization-level access through `KINRO_ARRAKIS_OPERATOR_ORG_ID`, but it does not yet define an Arrakis admin role. Admin gating can be added later if needed, preferably from WorkOS organization roles or an environment-backed allowlist of WorkOS user ids/emails.
- Global autopilot changes should not require two-step approval.
- Slack destinations should be backend-only. The UI should select preconfigured destinations, not edit raw webhooks.
- New-lead autopilot defaults should have a section with controls for all sources and each specific source.
- Default autopilot rules should be applicable in all matching scenarios.
- If a lead has no timezone, use Central Time.
- Manual operator sends should count against email caps where applicable, but should not be blocked by Ben engagement frequency thresholds.
- Customer-initiated replies outside engagement hours should be allowed for all channels when autopilot is otherwise allowed.
- The first default cap for total daily email sending should be 100. The first global Ben daily engagement cap should also start at 100 unless a narrower per-lead cap is selected.

## Suggested Implementation Phases

Phase 1:

- Add Settings route and navigation item.
- Move global Ben autopilot toggle to Settings.
- Add new-lead autopilot default toggles for all sources and each known source.
- Display read-only current defaults for Slack, engagement hours, and caps.

Phase 2:

- Add DB-backed settings table or equivalent repository.
- Add editable Slack notification master toggle and filters.
- Enforce Slack filters in notification sending.

Phase 3:

- Add default autopilot rules by source/status/AMS condition.
- Enforce rules on lead creation and state transitions.
- Add preview for retroactive changes.

Phase 4:

- Add engagement hours and frequency thresholds.
- Enforce engagement hours and Ben frequency thresholds in Ben scheduler and send paths.
- Show non-blocking warnings for manual operator sends outside engagement hours.
- Add blocked-action logging.

Phase 5:

- Add email sending caps and usage display.
- Enforce caps at provider-send boundary.
- Add operator usage breakdown.
