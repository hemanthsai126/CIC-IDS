# Pivot Landing Chat Lead Finalization

## Why This Work Started

This worktree started after reviewing a production pivot landing webchat from a new lead, Aditya. The chatbot asked for information that the visitor had already provided. That exposed two related problems:

- Ben did not always have enough prior chat context when generating the next reply.
- Lead creation/extraction happened too early, before the visitor had finished giving useful business details.

The original goal was not only to stop repeated questions, but to make the pivot landing chat produce a complete, useful lead record from everything the visitor said.

## Original Expectations

The expected behavior from the beginning of this work was:

- Ben should not re-ask for facts already provided in the same chat.
- Every useful detail from the chat should be extracted into lead fields or lead metadata.
- The lead timeline should expose the full chat transcript so operators can review the actual conversation.
- The fix should be tested first on preview before production.
- Ben should use the full chat context, not only the last 16 turns.
- Oracle/mission planning should account for all information already provided before asking Ben to follow up.

## What Was Found

The current production-style flow is contact-triggered:

1. The visitor sends a chat message.
2. Data API creates or gets an `agent_run` for the anonymous landing chat session.
3. Data API calls Backoffice Ben for the next reply.
4. Data API persists the user and assistant messages into `agent_transcript_events`.
5. After each persisted turn, Data API projects the transcript into inbox and lead state.
6. If any user message contains an email or phone number, the backend creates or links a lead immediately.
7. Generic `createLead()` may create the default new-lead Ben mission and schedule follow-up sequence steps.

This means if a visitor gives contact info halfway through the chat, the lead can be created before the visitor gives name, revenue, years in business, coverage needs, employee count, or other important details. Later details may update the lead, but the initial lead creation and default automation can already have fired.

## Target Behavior

Pivot landing chat should be transcript-first and finalize-after-chat, not contact-info-first.

During an active chat:

- Persist every user and Ben message to the landing chat transcript.
- Let Ben continue the live chat experience.
- Do not create a lead just because contact info appeared.
- Do not start the default new-lead Ben mission or default follow-up sequence.
- Do not send outbound email/SMS follow-up while the visitor is still chatting.

When the chat is considered finished:

- Read the full persisted transcript.
- Extract all provided details from the complete conversation.
- Create or update the lead once, using the best available complete context.
- Store structured fields on the lead where supported.
- Store additional collected facts, QA pairs, and transcript turns in metadata.
- Add a timeline event that includes the full chat transcript.
- Trigger Oracle after the final lead record exists.
- Let Oracle decide whether anything is still missing.
- Only if Oracle finds missing required details should Ben draft or send a follow-up.

## How To Decide A Chat Is Finished

There is no reliable browser-close signal, so the best default is an idle finalization window.

Recommended behavior:

- Every new user message schedules or reschedules a `landing_chat_finalize` job for the same session.
- The job runs only after an idle period, for example 5-10 minutes after the latest user message.
- If a newer user message arrived after the job was scheduled, the job exits as stale.
- Optional later improvement: add an explicit "I'm done" / "Submit request" action that finalizes immediately.

This mirrors the safety pattern already used in Ben SMS/email flows: if newer customer context arrives before Ben sends, the older response is suppressed or made stale.

## Ben And Oracle Expectations

Ben should behave differently in two phases.

In-chat phase:

- Ben can answer and ask the next useful question in the webchat.
- Ben should use the full stored chat context.
- Ben should not trigger external follow-up just because he has contact info.
- If the visitor sends another message before a prior response is applied, the newest context should win.

Post-chat phase:

- Oracle reviews the finalized lead and the full transcript.
- Oracle decides whether the request is complete enough or needs follow-up.
- If follow-up is needed, Oracle creates the mission/sequence with the missing facts clearly scoped.
- Ben drafts or sends only from that Oracle-approved post-chat mission.

## Default Mission Exemption

`pivot-landing-chat` leads should be exempt from the generic default new-lead Ben mission/sequence at creation time.

Reason:

- The landing chat itself is already the intake interaction.
- Generic follow-up may duplicate questions already answered in chat.
- Oracle should create the correct post-chat mission after seeing the final transcript.

Expected implementation direction:

- Mark finalized pivot landing chat leads with metadata that suppresses automatic lead engagement/default sequence.
- Or explicitly skip default mission/sequence creation when `externalSource === "pivot-landing-chat"`.
- Prefer Oracle-created missions for this source after chat finalization.

## Data Extraction Expectations

The finalized extraction should capture, at minimum:

- Name.
- Email and/or phone.
- Company name.
- State/location.
- Business type / industry.
- Coverage requested.
- Years in business.
- Annual revenue.
- Employee count, contractor count, or clear notes when the visitor describes staffing differently.
- Any other useful facts stated in the chat, even if there is no first-class lead column.

Canonical lead fields should be populated where available. Extra facts should be stored in metadata, alongside the transcript and QA pairs.

## Acceptance Criteria

A successful pivot landing chat flow should satisfy:

- Contact info alone does not immediately create a lead during active chat.
- A completed/idle chat creates or updates one lead using the full transcript.
- The dashboard lead details show the extracted facts.
- The lead timeline includes the full chat transcript.
- No default Ben new-lead sequence starts for `pivot-landing-chat` before Oracle reviews the final context.
- Oracle receives the finalized lead context and transcript-derived facts.
- Ben only sends follow-up after Oracle determines there is still something to ask.
- Repeated questions are reduced because Ben and Oracle both see the whole chat context.

## Open Decisions

- Exact idle timeout duration: 5 minutes is faster, 10 minutes is safer.
- Whether to add an explicit "Submit request" button in the chat UI.
- Whether final extraction should remain deterministic, use an LLM, or combine both.
- Whether post-chat follow-up should draft for operator review first or send automatically when confidence is high.

