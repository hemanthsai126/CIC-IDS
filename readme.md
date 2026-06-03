# Plan: Arrakis As The Brain For Ben Voice Calls

Date: 2026-06-02
Status: Ready to reapply on latest main after PR #991 merge

## Objective

Make Vapi voice calls behave like the existing Arrakis-controlled Ben flows for
SMS, WhatsApp, and email.

Vapi should not be the source of truth for Ben's reasoning. Vapi should be the
phone interface: it listens, speaks, and calls silent tools. Arrakis should be
the brain: it decides whether the call is inbound or outbound, looks at lead and
mission data, decides what Ben already knows, decides what is still missing or
unclear, and returns the next thing Ben should say or do.

The expected customer experience is simple:

- For inbound calls, Ben answers as Kinro and starts from the current lead or
  intake context.
- For outbound calls, Ben says he is calling about the customer's insurance
  quote request.
- Ben confirms existing details once, naturally.
- Ben asks only for missing or unclear details after that.
- Ben does not ask for details Arrakis already knows unless the customer says
  they are wrong.
- If Arrakis has no useful details for the caller, Ben runs the full intake.
- If autopilot is enabled and details are missing, Ben can initiate an outbound
  call to collect those details.

## Guardrails

- Do not let this change break SMS, WhatsApp, or email Ben flows.
- Do not let Vapi become an independent Ben brain with duplicated business
  logic.
- Do not put all intelligence into a long static Vapi assistant prompt.
- Do not deploy production changes until preview is validated.
- Do not change the database schema for the first version unless a real runtime
  need appears.
- Build on top of the merged PR #991 changes instead of reviving the temporary
  preview-only branch.
- Keep preview Vapi assistant IDs separate from production assistant IDs.
- Keep the existing Arrakis dashboard workflow intact.

## Current Base

PR #991, `Refine Arrakis Oracle Ben engagement flow`, has merged and is now part
of the base this work should build on. It updated the Oracle and Ben engagement
flow in areas this voice work also needs:

- Ben text/backoffice prompting
- Ben runtime payloads
- Oracle handoff behavior
- default missions and SMS sequences
- Data API operation logic
- research completion handoff triggers

Because of that overlap, the voice brain work should be reapplied from latest
`main`, preserving PR #991 behavior and avoiding the temporary preview revision
used for the first Vapi smoke test.

## Ownership Model

### Arrakis Owns

Arrakis decides:

- whether the call is `inbound` or `outbound`;
- which lead, customer, call, and active mission the call belongs to;
- which facts are already known;
- which required details are missing;
- which optional or collected details are unclear;
- which fields Ben should not ask again;
- the call agenda for this turn;
- whether Ben should ask a question, confirm a detail, complete intake, hold the
  mission, or end the call;
- whether a research/oracle agent should be consulted before answering.

### Vapi Owns

Vapi should only handle:

- telephony;
- speech-to-text;
- text-to-speech;
- turn-level tool calls;
- voicemail detection;
- call ending.

Vapi should receive a thin assistant prompt that says, in effect:

- use Arrakis as the source of truth;
- call the Arrakis voice oracle after meaningful caller turns;
- say only what Arrakis tells you to say;
- call tools silently;
- never mention JSON, function names, tool names, or internal commands.

## Voice Context Contract

Arrakis should build a voice context for every meaningful call turn.

Required context:

```ts
type BenVoiceContext = {
  callId: string | null;
  vapiCallId: string | null;
  leadId: string | null;
  phoneE164: string | null;
  direction: "inbound" | "outbound";
  latestCustomerText: string;
  transcriptSoFar: Array<{
    role: "assistant" | "customer" | "tool";
    text: string;
    at?: string | null;
  }>;
  knownDetails: Record<string, unknown>;
  missingDetails: Record<string, unknown>;
  unclearDetails: Record<string, unknown>;
  doNotAskFields: string[];
  callAgenda: string[];
  priorContextSummary: string | null;
  activeMission: Record<string, unknown> | null;
  assistantState: Record<string, unknown> | null;
};
```

This does not require a new table for V0. Most of this can be computed from
existing lead data, active mission data, call metadata, and the current
transcript.

## Detail Classification

Arrakis should classify details before Ben speaks.

### Known Details

Fields Arrakis already trusts enough to use conversationally.

Examples:

- first name;
- last name;
- phone number;
- email;
- company name;
- business type;
- state;
- zip code;
- requested coverage;
- current policy or renewal timing if available.

Ben should confirm these once in one concise sentence, not one by one.

Example:

> I have that you're looking for workers' comp for a restaurant in Texas. Is
> that still right?

### Missing Details

Required fields Arrakis does not have yet.

For full intake, the required fields are:

- coverage type;
- business type;
- state;
- email;
- company name;
- phone number;
- first name;
- last name;
- zip code.

For an existing lead, Arrakis should only ask for the required fields that are
missing for that lead or mission.

### Unclear Details

Fields that exist but are ambiguous, low-confidence, contradictory, or not
specific enough for quoting.

Ben should ask one short clarification question for one unclear detail at a
time.

### Do-Not-Ask Fields

Fields that Arrakis already considers confirmed or not useful to ask again.

Ben must not ask these again unless the customer says the detail is wrong or
wants to correct it.

## Inbound Call Behavior

Inbound opening:

> Thank you for calling Kinro, this is Ben.

After the opening, Arrakis decides what Ben says next.

If the caller is matched to an existing lead:

- Ben should use the known lead and mission context.
- Ben should confirm known details once if useful.
- Ben should ask only for missing or unclear details.

If the caller is not matched to a lead or Arrakis has no useful details:

- Ben should run full intake one question at a time.
- Ben should create or attach the correct lead according to existing Arrakis
  lead rules.

## Outbound Call Behavior

Outbound opening:

> Hi {firstName}, this is Ben calling from Kinro about your insurance quote
> request.

If first name is unavailable:

> Hi, this is Ben calling from Kinro about your insurance quote request.

After the opening:

- Ben should not say "How can I help you today?"
- Ben should not ask generic discovery questions if Arrakis already has the
  answer.
- Ben should explain the purpose of the call when needed: Kinro is following up
  on the customer's insurance quote request.
- Ben should confirm known details once.
- Ben should ask only the next missing or unclear detail.
- Ben should ask one question at a time.

If the customer says "you called me" or asks why Ben is calling:

> I'm following up on your Kinro insurance quote request and just need to confirm
> a couple of details.

Then Ben should ask the next Arrakis-approved question.

## Autopilot Behavior

When autopilot is enabled:

- Arrakis/Oracle can decide that a lead needs an outbound call.
- The Data API can initiate a Vapi outbound call for that existing lead.
- The call must include lead, mission, known detail, and missing detail context.
- Ben should call with a specific purpose, not a generic greeting.
- Ben should collect only the details needed to advance the active mission.
- Ben should persist captured fields silently.
- Ben should complete or hold the mission when appropriate.

Autopilot should not call customers when:

- there is no usable phone number;
- the customer is marked do-not-contact;
- consent or eligibility rules fail;
- there is no active mission or missing detail worth calling about.

## AI Call Screening And Gatekeepers

Ben must handle AI screening systems, receptionists, and phone gatekeepers.

If the other side asks who is calling:

> This is Ben from Kinro. I'm calling about the customer's business insurance
> quote request.

If asked why:

> I'm following up to confirm a few details needed for their quote.

If asked to identify the intended recipient:

- use the customer's name if Arrakis has it;
- otherwise ask to be connected to the person who requested the insurance quote.

Ben should not collect quote details from a screening system unless it is
clearly authorized to answer on behalf of the customer.

If the gatekeeper refuses, cannot connect, or asks for a callback:

- Ben should state the outcome briefly;
- Arrakis should record the outcome;
- the mission can be held, retried, or routed according to existing workflow
  rules.

## Silent Tool Expectations

Tool calls are internal actions. Ben must never say tool names or JSON aloud.

Required tools:

- `ask_arrakis_voice_oracle`: ask Arrakis what Ben should say or do next.
- `record_intake_field`: persist a provided, confirmed, or corrected field.
- `complete_intake`: mark the voice intake or mission turn complete.
- `hold_mission`: pause the mission and route to a human/operator path.
- `endCall`: end the Vapi call after the final goodbye.
- `leave_voicemail`: only for clear voicemail or answering-machine systems.

Rules:

- When the caller provides multiple details in one answer, capture all of them
  silently.
- When all missing and unclear details are handled, call `complete_intake`.
- Before ending a live call, ask once: "Is there anything else you'd like to
  add?"
- End with: "The Kinro team will review everything and follow up with quote
  options or any quick clarification questions. Goodbye."
- Immediately call `endCall` after the final goodbye.
- If the caller says bye or indicates the call is over, use one short closing
  sentence and call `endCall`.
- Only call `leave_voicemail` when the other side is clearly automated
  voicemail or an answering machine.
- Do not say the voicemail message directly if `leave_voicemail` will play it.

## Arrakis Voice Oracle

Add or preserve a Data API path that lets Vapi ask Arrakis for the next voice
turn.

Expected route:

- `POST /v1/ben/voice/oracle`

Expected responsibilities:

- validate the voice oracle request;
- resolve lead and mission context;
- compute known, missing, unclear, and do-not-ask fields;
- include call direction;
- include recent transcript;
- call the backoffice Ben execution agent with channel `voice`;
- return the next speakable Ben reply and any assistant state.

The backoffice Ben agent should support channel `voice` separately from `sms`,
`whatsapp`, and `email` so voice can stay short, natural, and turn-based.

## Vapi Assistant Prompt Shape

The Vapi assistant prompt should be short. It should not duplicate all Arrakis
business logic.

It should say:

- this is a Kinro voice agent;
- Arrakis is the source of truth;
- call `ask_arrakis_voice_oracle` after meaningful customer turns;
- speak only the concise text Arrakis returns;
- use tools silently;
- never mention tools, functions, JSON, or internal state;
- handle voicemail only through `leave_voicemail`;
- end calls through `endCall` after the final goodbye.

The long outbound behavior prompt can be used as product guidance, but the final
implementation should move those decisions into Arrakis/backoffice prompts and
runtime context.

## Data API Outbound Call Creation

When creating a Vapi outbound call, Data API should:

- use the correct preview or production outbound assistant ID;
- use the correct Vapi phone number for the environment;
- send variable values for lead and mission context;
- set a good outbound first message from Arrakis context;
- avoid invalid `assistantOverrides.model` payloads unless provider/model are
  included correctly;
- prefer assistant-level tool configuration when possible.

Important Vapi lesson from preview:

- sending `assistantOverrides.model.tools` without a model `provider` caused a
  Vapi `400` error;
- the safer V0 is to configure tools on the Vapi assistant and avoid per-call
  model overrides unless absolutely needed.

## Database Schema Expectation

No database schema change is expected for V0.

The first implementation should compute voice behavior from existing data:

- lead fields;
- lead metadata;
- active mission context;
- operation/call metadata;
- timeline events;
- transcript turns;
- assistant state returned during the call.

A schema change should only be considered later if we need to persist structured
voice-only state such as:

- per-call confirmation checklist;
- per-call already-asked field set;
- voice mission attempt state;
- callback scheduling state;
- durable call-screening outcomes separate from existing timeline/call metadata.

For preview V0, keep this runtime-computed unless testing proves it is unstable.

## Implementation Sequence After PR #991

1. Update local `main`.
2. Create a new branch for the voice brain reapply.
3. Reapply the Arrakis voice oracle changes on top of latest `main`.
4. Resolve conflicts carefully in Ben, Oracle, and Data API operation files.
5. Make Vapi prompt/tool config thin and assistant-level where possible.
6. Build and test Data API and backoffice locally.
7. Deploy backoffice preview.
8. Deploy Data API preview.
9. Verify preview Cloud Run revisions.
10. Run inbound preview call smoke test.
11. Run outbound preview call smoke test.
12. Validate field persistence and mission completion/hold behavior.
13. Only then consider production rollout planning.

## Preview Validation Checklist

### Static Checks

- Data API build passes.
- Backoffice build passes.
- Focused Data API tests pass.
- Lints for changed files pass.

### Vapi Assistant Checks

- Preview outbound assistant ID exists in the active Vapi account.
- Assistant server URL points to preview Data API:
  `https://arrakis-data-api-preview-965370637467.us-central1.run.app/v1/webhooks/vapi/voice`
- Assistant model has a valid provider and model.
- Assistant tools include:
  - `ask_arrakis_voice_oracle`
  - `record_intake_field`
  - `complete_intake`
  - `hold_mission`

### Outbound Smoke Test

Use an existing preview lead with at least one known detail and at least one
missing detail.

Expected:

- Ben uses the customer's first name if available.
- Ben says he is calling from Kinro about the insurance quote request.
- Ben does not say "How can I help you today?"
- Ben confirms known details once.
- Ben asks only the next missing or unclear detail.
- Captured answers appear on the lead/timeline/mission as expected.
- Call recording and transcript attach to the call/lead surfaces.

### Inbound Smoke Test

Call the preview inbound number.

Expected:

- Ben opens with "Thank you for calling Kinro, this is Ben."
- If matched to a lead, Ben uses that lead context.
- If not matched, Ben runs full intake.
- Ben asks one question at a time.
- Ben captures fields silently.
- Ben ends cleanly when intake is complete or the caller ends the call.

### Gatekeeper Smoke Test

Simulate an AI screening assistant or receptionist.

Expected:

- Ben identifies himself as Ben from Kinro.
- Ben says he is calling about the customer's insurance quote request.
- Ben asks to be connected to the customer.
- Ben does not collect quote details from an unauthorized screening system.
- Refusal or callback outcomes are recorded for Arrakis.

## Success Criteria

This work is successful when:

- Arrakis, not Vapi, controls Ben's voice reasoning.
- Inbound and outbound calls have different openings.
- Outbound calls are purpose-driven and never generic.
- Known details are confirmed once and used conversationally.
- Missing and unclear details are asked one at a time.
- Existing confirmed details are not re-asked.
- Autopilot can call to collect missing details when eligible.
- AI screening and gatekeeper calls are handled naturally.
- Tool calls are silent.
- The existing Arrakis SMS, WhatsApp, email, dashboard, and mission workflows
  continue to work.
- Preview smoke tests pass before production planning starts.
