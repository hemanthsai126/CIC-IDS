# Research Paper Plan: Can AI Replace The Human Insurance Broker?

Updated: 2026-06-09

## One-Sentence Thesis

AI should not be evaluated as a yes/no legal replacement for a licensed insurance broker. The best paper asks a sharper question: across the broker task ladder, which sub-skills can AI perform at human or superhuman quality, which require licensed supervision, and what evidence would justify moving from "AI assists brokers" to "AI runs most of the broker workflow under accountable human oversight"?

## Executive Summary

The paper should argue that insurance brokerage is not one job. It is a bundle of knowledge, conversation, risk collection, market access, quote comparison, advice, documentation, compliance, and transaction-control tasks. Some tasks are already natural software tasks. Some tasks are broker judgment tasks. Some tasks are legally reserved to licensed producers even if the underlying cognitive work can be automated.

The winning academic contribution is therefore a task-level benchmark and operating study for AI-mediated insurance distribution:

- First, prove the knowledge baseline: AI can pass state insurance licensing exams, beginning with Property and Casualty and expanding state by state.
- Second, prove the work baseline: AI can collect complete risk information, identify missing facts, explain coverage concepts, and produce quote-ready files.
- Third, prove the judgment baseline: in shadow mode, AI can compare options and make bind-ready recommendations that licensed humans approve.
- Fourth, prove the safety baseline: AI can avoid advice-boundary, disclosure, unfair discrimination, payment, bind, and channel-compliance failures at a rate suitable for supervised deployment.
- Fifth, prove the economic baseline: AI reduces producer or CSR handling time, increases throughput, improves documentation, or expands the accounts a licensed producer can supervise without increasing E&O risk.

Arrakis gives Kinro a credible internal operating substrate for this paper, but the paper should not expose Arrakis as a product blueprint. Publicly, Arrakis can be described as an implemented insurance-workflow research platform with modules for intake, follow-up, call documentation, risk profiling, appetite screening, quote workflow, human handoff, and evaluation. The confidential details are the prompts, client names, carrier access paths, credentials, production URLs, customer data, and partner-specific workflows.

## The Core Research Question

The naive question is:

- Can AI replace a human insurance broker?

The publishable question is:

- Given the same facts, market access, compliance rules, and supervision constraints as a human broker, which broker tasks can AI perform correctly, reliably, safely, and economically?

This framing avoids two weak arguments:

- It avoids claiming that passing an exam means AI is ready to sell insurance autonomously.
- It avoids ending the paper with the trivial answer that AI is not currently a licensed producer.

The paper should explicitly distinguish three axes:

- Capability: can the AI do the cognitive task at all?
- Reliability: can it do the task consistently across states, products, edge cases, and adversarial inputs?
- Deployability: can the task be delegated safely today under licensing, carrier, compliance, and E&O constraints?

## What A Human Insurance Broker Must Be Good At

A strong human broker is not merely someone who knows insurance vocabulary. The broker converts a messy buyer situation into a compliant insurance outcome. The relevant competencies are:

1. Licensing knowledge
   - Understand insurance concepts, policy forms, coverage limits, exclusions, endorsements, duties, ethics, unfair trade practices, and state-specific rules.
   - Pass state licensing exams and maintain continuing education.

2. Customer discovery
   - Get the buyer to explain what they need, what changed, what they fear, what they can afford, and what they do not understand.
   - Ask follow-up questions without overwhelming the buyer.
   - Distinguish underwriting facts from preferences, constraints, and concerns.

3. Risk-data collection
   - Turn messy language into structured fields needed by applications, raters, MGAs, wholesalers, and carrier portals.
   - Identify missing, inconsistent, or suspicious facts.
   - Know which facts are required, optional, irrelevant, or prohibited in a given jurisdiction.

4. Product and coverage explanation
   - Explain limits, deductibles, exclusions, state minimums, optional coverages, certificates, binders, subjectivities, and payment requirements.
   - Use plain language without creating false certainty.

5. Appetite and market selection
   - Know which carriers, MGAs, wholesalers, or programs are worth checking for a given risk.
   - Avoid wasting time on markets that cannot write the class, state, exposure, or risk profile.

6. Quote and submission workflow
   - Submit the risk through carrier portals, comparative raters, APIs, wholesalers, or underwriters.
   - Coordinate supplemental questions, documents, appetite responses, declines, referrals, and quote terms.

7. Quote comparison and advisory judgment
   - Compare available quotes across premium, limits, deductibles, exclusions, carrier strength, speed, eligibility, payment terms, and buyer goals.
   - Explain tradeoffs and, where legally permitted, recommend a path.

8. Bind readiness
   - Know whether the customer, quote, application, payment, signatures, disclosures, and effective-date conditions are ready for bind.
   - Avoid implying coverage is active before bind authority and conditions are satisfied.

9. Compliance and E&O hygiene
   - Make required disclosures.
   - Avoid misrepresentation, unfair discrimination, unsupported recommendation, hidden steering, and advice outside authority.
   - Preserve a record of what was asked, answered, disclosed, recommended, approved, and bound.

10. Relationship and trust
    - Build confidence, handle ambiguity, calm anxious buyers, and know when human empathy matters.
    - Keep the customer moving without being manipulative.

11. Post-bind service
    - Support policy changes, billing questions, certificates, renewals, claims routing, cancellations, and coverage-gap monitoring.

12. Operating fluency
    - Use AMS, CRM, raters, carrier portals, phone, SMS, email, document tools, and handoff workflows.
    - Coordinate with CSRs, producers, underwriters, carriers, wholesalers, and customers.

## What It Takes For A Normal Human To Become A Broker

The paper should include this section because it makes the first proof leg legible: if humans are allowed to enter the profession after passing a defined licensing and supervision path, then AI should at least be measured against the same knowledge baseline.

A typical human path includes:

1. Pre-licensing education
   - State-defined hours and curriculum.
   - Coverage fundamentals, state law, ethics, unfair trade practices, producer duties, policy structure, and line-specific knowledge.

2. State licensing exam
   - Usually multiple-choice.
   - Tests general insurance knowledge plus state-specific rules.
   - Passing threshold varies by state and line.

3. Background checks and administrative requirements
   - Fingerprinting, application, fees, identity checks, and sometimes bonds or business-entity requirements.

4. Agency affiliation and carrier access
   - A license alone does not give access to all markets.
   - The producer usually needs an agency, appointments, aggregator/cluster access, wholesaler relationships, MGA access, or carrier portals.

5. E&O and supervision
   - Firms rely on procedures, review, documentation, compliance training, QA, and errors-and-omissions insurance.

6. Practical apprenticeship
   - Real competence comes from seeing risks, carrier appetites, quote exceptions, bind conditions, customer objections, and claim/service failures.

The paper should argue that AI should be measured against both parts of broker formation:

- Formal competence: exam knowledge and rule understanding.
- Practical competence: supervised task performance on realistic quote-to-bind workflows.

## Claim Ladder For The Paper

The paper should not make one giant claim. It should advance a ladder of increasingly strong claims:

1. AI can pass insurance licensing knowledge tests.
2. AI can answer insurance questions with source-aligned explanations.
3. AI can collect and structure risk information from messy buyer conversations.
4. AI can identify missing or inconsistent facts and ask good follow-up questions.
5. AI can avoid prohibited questions, advice-boundary violations, and unsafe bind language.
6. AI can generate quote-ready files that licensed humans find usable.
7. AI can identify likely carrier appetite and route a risk to appropriate markets.
8. AI can compare quotes or offers under explicit buyer goals.
9. AI can produce bind-ready recommendation packages that licensed humans approve in shadow mode.
10. AI can produce a better audit trail than many human-only conversations.
11. AI can reduce cost, latency, or handling time at equal or better quality.
12. AI can expand what licensed humans supervise, even before AI can legally assume producer responsibility.

## Everything We Have To Prove Experimentally

### 1. Licensing Exam Competence Across States

Purpose:

- Establish the minimum knowledge baseline expected of a human entering the profession.

Experiments:

- Run Property and Casualty licensing-style multiple-choice exams for California first.
- Expand to 5-10 comparison states, then all states where source material is available.
- Separate universal P&C concepts from state-specific rules.
- Test cross-state conflict cases where the same scenario has a different correct answer depending on jurisdiction.
- Add Life and Health, Personal Lines, Commercial Lines, and Surplus Lines only after the P&C benchmark is stable.

Metrics:

- Exact-answer accuracy.
- Estimated pass/fail rate against state thresholds.
- Topic-level accuracy.
- Explanation alignment with the expected rule.
- Confidence calibration.
- Cost and latency.

Controls:

- Use official exam outlines and permitted practice materials.
- Track provenance for every question.
- Deduplicate near-identical questions.
- Keep a clean held-out set.
- Include expert-written questions that are not public to reduce contamination risk.

### 2. Coverage Knowledge And Explanation Quality

Purpose:

- Show that AI can explain insurance concepts in a way customers and licensed reviewers consider correct, clear, and safe.

Experiments:

- Ask common customer questions about limits, deductibles, exclusions, state minimums, certificates, proof documents, payment, and bind.
- Compare AI answers against licensed-agent or compliance-approved answers.
- Evaluate with and without retrieval from approved source documents.

Metrics:

- Factual correctness.
- Source support.
- Explanation clarity.
- Concision.
- Correct uncertainty and escalation.
- Hallucinated authority or unsupported claims.

### 3. Regulatory And Sales-Practice Compliance

Purpose:

- Prove that AI can stay inside the boundary between information, solicitation, advice, payment handling, and bind authority.

Experiments:

- Scenario bank across state-specific advice boundaries, AI disclosures, prohibited rating factors, unfair discrimination, payment financing, call/text consent, recording consent, synthetic voice disclosure, and prompt injection.
- Evaluate base models, RAG systems, and guarded agent workflows.
- Compare against human-agent transcripts where possible.

Metrics:

- Blocking compliance failure rate.
- Borderline review rate.
- Severity-weighted E&O risk.
- Correct refusal/escalation rate.
- Disclosure correctness.
- State-specific factuality.

### 4. Intake And Risk-Data Collection

Purpose:

- Show that AI can convert conversation into the structured facts a broker needs.

Experiments:

- Give AI messy customer conversations and ask it to build a structured intake record.
- Compare AI-guided intake against static forms and human call-center intake.
- Test personal lines and narrow small-commercial classes separately.

Metrics:

- Required-field completeness.
- Extraction accuracy.
- Missing-fact detection.
- Inconsistency detection.
- Number of turns to completion.
- Prohibited or unnecessary questions.
- Customer experience score.

### 5. Buyer Preference And Utility Capture

Purpose:

- Show that AI can recover buyer context that rigid comparison forms miss.

Experiments:

- Compare three contexts: underwriting facts only, underwriting facts plus static-form choices, and conversational AI-discovered preferences.
- Ask licensed advisors to judge whether the extra context changes the best recommendation or handoff.
- Run buyer studies on clarity, confidence, and perceived neutrality.

Metrics:

- Preference completeness.
- Recommendation change rate when preferences are included.
- Buyer confidence.
- Buyer comprehension.
- Advisor-rated usefulness.
- Abandonment or continuation intent.

### 6. Appetite And Market Selection

Purpose:

- Test whether AI can decide which markets are worth checking for a given risk.

Experiments:

- Give AI risk profiles and market appetite rules.
- Compare AI market selection against licensed broker choices, carrier appetite responses, and actual quote outcomes.
- Use narrow classes first: simple contractors, pressure washing, lawn care, daycare, beauty, or other controlled SMB classes.

Metrics:

- Eligible-market recall.
- Decline avoidance.
- Correct reason for appetite fit or mismatch.
- Follow-up question quality.
- Missed-market severity.

### 7. Quote-Ready File Quality

Purpose:

- This should be the paper's central operational artifact.
- A quote-ready file is the bridge between "AI talks to customers" and "AI can replace broker labor."

Experiments:

- Produce quote-ready files from conversations.
- Have licensed producers, CSRs, underwriters, or carrier/MGA users score whether the file is usable.
- Measure whether humans need to recollect information.

Metrics:

- Human approval rate.
- Missing-information rate.
- Time to review.
- Number of follow-up questions.
- Completeness of customer acknowledgments.
- Documentation quality.
- Compliance-audit sufficiency.

### 8. Quote Comparison And Recommendation In Shadow Mode

Purpose:

- Test broker judgment without letting AI execute regulated actions.

Experiments:

- Give AI a buyer profile, quote set, carrier/product metadata, and buyer goals.
- Ask it to compare offers, identify tradeoffs, and recommend or decline to recommend depending on the permitted workflow.
- Compare against licensed advisor recommendations, senior-advisor adjudication, actual buyer selection, comparator ranking, and bind/abandon outcomes.

Metrics:

- Alignment with buyer goals.
- Coverage adequacy.
- Cost/protection tradeoff quality.
- Expert approval rate.
- Disagreement with senior licensed advisors.
- Severity of bad recommendations.
- False confidence when market access is incomplete.

### 9. Bind-Ready Package And Human Approval

Purpose:

- Test whether AI knows when a transaction is ready for the accountable human to bind.

Experiments:

- Present AI with complete and incomplete quote scenarios.
- Ask AI to produce a bind-ready package or identify what blocks bind.
- Licensed producers approve, modify, or reject the package.

Metrics:

- Bind-readiness accuracy.
- Missing-condition detection.
- Premature-bind failure rate.
- Human approval rate.
- Human review time saved.
- Severity of missed subjectivities or disclosures.

### 10. Auditability And E&O Protection

Purpose:

- Show that AI-mediated brokerage can be more inspectable than a purely human workflow.

Experiments:

- Compare AI-generated logs, summaries, citations, and decision packages against human call notes.
- Ask compliance reviewers whether the record is sufficient to reconstruct the recommendation and defend it.

Metrics:

- Record completeness.
- Citation coverage.
- Disclosure traceability.
- Ability to identify who approved what.
- Compliance-review time.
- E&O risk rating.

### 11. Productivity And Unit Economics

Purpose:

- Show that even if a human remains accountable, AI changes the economics of brokerage.

Experiments:

- Compare human-only, form-plus-human, and AI-plus-human workflows.
- Measure producer or CSR time per quote-ready file, per bind-ready package, and per bound policy.

Metrics:

- Cost per conversation.
- Cost per quote-ready file.
- Cost per bind-ready package.
- Average handle time.
- Handoff quality.
- Quote completion.
- Bind conversion.
- Revenue or premium per human hour.

### 12. Robustness, Adversarial Safety, And Monitoring

Purpose:

- Prove the system can survive real users, not just clean benchmark prompts.

Experiments:

- Prompt injection.
- Off-topic recovery.
- Harmful user content.
- Fake discount or rebate requests.
- Proof-document safety.
- Political neutrality where relevant.
- Unfair-discrimination traps.
- Recommendation guardrails.

Metrics:

- Blocking failure rate.
- Recovery rate.
- Escalation correctness.
- Regression rate across agent versions.
- Human review load.

## What The Repo Already Supports Or Evidences With Arrakis

This section should be used carefully. "Evidences" here means implemented and test-backed workflow capability, not yet peer-reviewed proof of broker replacement. The paper can say these capabilities exist in an internal research platform, but it should not reveal product-specific implementation details.

### 1. Structured Lead Intake

Arrakis has lead schemas and provider-specific intake paths that capture insurance-relevant facts such as source, contact details, business context, location, coverage need, and metadata.

Public-safe claim:

- We have implemented structured insurance lead intake across channels and sources.

Evidence paths:

- `apps/arrakis/data_api/src/leads/schemas.ts`
- `apps/arrakis/data_api/src/voice-providers/vapi-intake.ts`
- `apps/arrakis/data_api/src/email-leads/providers/`
- `apps/arrakis/data_api/test/voice-providers/`
- `apps/arrakis/data_api/test/email-leads/`

Research use:

- Supports experiments on conversation-to-structured-intake accuracy, missing-field detection, and quote-ready file generation.

### 2. AI-Guided Conversation And Gap Filling

Arrakis includes an AI conversation workflow that can draft customer-facing follow-up messages, use known lead context, avoid re-asking known facts, and operate across text channels with guardrails.

Public-safe claim:

- We have implemented AI-assisted insurance intake and follow-up that can reason over prior lead context.

Evidence paths:

- `apps/arrakis/backoffice/agents/ben_agent/core/text.ts`
- `apps/arrakis/backoffice/agents/ben_agent/core/prompts.ts`
- `apps/arrakis/data_api/src/ben/missions.ts`
- `apps/arrakis/data_api/test/ben-agent/text-draft.test.ts`
- `apps/arrakis/data_api/test/ben-autopilot.test.ts`

Research use:

- Supports experiments on follow-up-question quality, turn count, context retention, and avoidance of unsafe customer-facing claims.

### 3. Multi-Channel Follow-Up And Human Handoff

Arrakis supports SMS, WhatsApp, email, and voice workflows, plus scheduling, previews, blocking checks, and handoff paths.

Public-safe claim:

- We have implemented multi-channel broker-style follow-up and human handoff workflows.

Evidence paths:

- `apps/arrakis/data_api/src/operations/repository.ts`
- `apps/arrakis/data_api/src/scheduler/worker.ts`
- `apps/arrakis/data_api/src/twilio/twiml.ts`
- `apps/arrakis/data_api/src/oracle/schemas.ts`
- `apps/arrakis/data_api/test/operations/`
- `apps/arrakis/data_api/test/twilio/`

Research use:

- Supports experiments on response latency, handoff quality, human supervision, and customer continuation.

### 4. Call Recording, Transcription, And Insurance-Specific Summaries

Arrakis includes call transcription and post-call summarization logic oriented around lead context, coverage needs, quote details, urgency, and handoff.

Public-safe claim:

- We have implemented call documentation and summarization for insurance workflows.

Evidence paths:

- `apps/arrakis/data_api/src/twilio/post-call-summarization.ts`
- `apps/arrakis/data_api/src/prompts/human-call-summary.ts`
- `apps/arrakis/data_api/src/operations/repository.ts`
- `apps/arrakis/data_api/test/operations/human-call-summary.test.ts`

Research use:

- Supports experiments comparing AI-generated audit trails against human notes.

### 5. Risk Profile And Appetite Screening

Arrakis can normalize a lead into a risk profile, evaluate market appetite, persist results, collect operator answers, and preserve questions requiring follow-up.

Public-safe claim:

- We have implemented risk-profile normalization and carrier/MGA appetite-screening workflows for insurance leads.

Evidence paths:

- `apps/arrakis/backoffice/src/appetite_check/lead-to-risk-profile.ts`
- `apps/arrakis/backoffice/src/appetite_check/providers/coverforce/client.ts`
- `apps/arrakis/data_api/src/appetite/schemas.ts`
- `apps/arrakis/data_api/src/appetite/repository.ts`
- `apps/arrakis/data_api/test/appetite/`

Research use:

- Supports experiments on market-selection accuracy, missing-question quality, and decline avoidance.

### 6. Quote Workflow And Quote Gap Questions

Arrakis has quoting schemas and workflow logic for creating or resuming quote workflows, persisting quote results, and recording question answers.

Public-safe claim:

- We have implemented quote-workflow orchestration around structured insurance leads.

Evidence paths:

- `apps/arrakis/data_api/src/quoting/schemas.ts`
- `apps/arrakis/data_api/src/quoting/repository.ts`
- `apps/arrakis/data_api/src/coverforce/lead-quotes.ts`
- `apps/arrakis/backoffice/src/quoting/coverforce-quotes.ts`
- `apps/arrakis/integration_tests/src/coverforce/quote-scenarios.test.ts`

Research use:

- Supports experiments on quote-ready files, quote-gap closure, and human review of generated quote artifacts.

### 7. Bind Record Capture, Not Autonomous Binding

Arrakis contains schema support for recording bind-related facts such as carrier, premium, term, effective date, policy number, and metadata.

Public-safe claim:

- We can represent bind outcomes and bind-related facts, while keeping the accountable bind step under licensed human control.

Evidence paths:

- `apps/arrakis/data_api/src/leads/schemas.ts`
- `apps/arrakis/data_api/test/leads/bind-schema.test.ts`

Research use:

- Supports shadow-mode bind-readiness experiments and downstream outcome measurement.

### 8. Evaluation Infrastructure

The repo contains an evaluation framework and scenario banks for insurance chatbot behavior, especially compliance and safety boundaries.

Public-safe claim:

- We have a reusable evaluation framework for insurance conversation traces, including blocking compliance failures, advisory issues, and informational quality metrics.

Evidence paths:

- `kinro-company/product/evals/eval-framework.md`
- `kinro-company/product/evals/us-regulatory-benchmark-brief.md`
- `sim/scenarios/thezebra/evals/`
- `docs/runbooks/cursor-cloud-zebra-prod-evals-automation.md`

Research use:

- Supports reproducible evaluation of state-specific compliance, recommendation guardrails, unfair discrimination, proof-document safety, prompt injection, and off-topic recovery.

## Existing Experiment Results And Artifacts In The Repo

### Broker-License Benchmark

The repo already has a dedicated benchmark project:

- `kinro-company/gtm/research/projects/broker-license-benchmark/experimental-protocol.md`
- `kinro-company/gtm/research/projects/broker-license-benchmark/research-paper-outline.md`
- `kinro-company/gtm/research/projects/broker-license-benchmark/paper/`

What exists:

- Goal: benchmark model performance on California P&C broker-agent licensing knowledge and applied broker reasoning.
- Dataset design: question, choices, answer, explanation, topic, difficulty, license line, split, rubric, provenance.
- Model matrix: OpenAI, Anthropic, Gemini, xAI, Qwen, Llama, DeepSeek, Mistral, Kimi/Moonshot, plus harness variants such as RAG, agent tools, self-consistency, critic/verifier, SFT, and RL.
- Metrics: accuracy, topic-level accuracy, calibration, explanation quality, compliance safety, cost, latency, and pass/fail estimate.
- Experiments: zero-shot, few-shot, RAG, prep-corpus RAG, SFT, RL, expert reasoning rubrics, scenario robustness, cross-state transfer, cross-state conflict tests, and sales-transcript recommendation quality.

What the paper draft currently claims:

- Frontier and strong compact models clear the licensing-knowledge bar on multiple-choice P&C exam-style questions.
- Accuracy alone is not enough; explanation alignment must also be measured.
- The figures expected are accuracy versus explanation alignment, score distribution by model, and topic-level heatmaps.

Important caveat:

- In this committed worktree, I did not find scored per-model result exports or the referenced chart assets. The doc should therefore keep the claim but mark the numeric result table as pending until the result files are attached, committed, or retrieved from the external run location.

### Zebra Production Eval Artifacts

The repo contains scenario banks and automation for The Zebra production evals:

- `sim/scenarios/thezebra/evals/`
- `docs/runbooks/cursor-cloud-zebra-prod-evals-automation.md`
- `docs/runbooks/cursor-cloud-zebra-prod-conversation-evals-automation.md`

What exists:

- Scenario banks for anti-advice, fake discounts/rebates, final quote triggering, harmful user content, off-topic recovery, political neutrality, prompt injection, proof-document safety, recommendation guardrails, and unfair discrimination.
- Most scenario suites contain 60 cases; proof-document safety has 12 cases; final-quote triggering has 30 cases.
- Automation writes report artifacts under `.tmp/sim/zebra-prod-eval-reports/<timestamp>/`.

Known number in documentation:

- The runbook's Slack comment example reports 9 evals, 99% pass rate, 486/492 passed, and 6 failing/error scenarios.

Important caveat:

- The 486/492 figure appears in a runbook example, not a committed report artifact. It should not be treated as publication evidence until the underlying report JSON/MD/PDF is retrieved and sanitized.

## How To Use Arrakis Without Revealing The Product

The paper should not say:

- Exact prompts or agent personas.
- Customer names, client names, lead data, or production volumes.
- Carrier, MGA, or partner access details.
- Internal URLs, credentials, environment variables, or operational runbooks that expose deployment details.
- Exact automation or prompt text that could be copied into a competing system.
- Any claim that AI currently binds coverage autonomously.

The paper can safely say:

- We implemented an internal insurance-workflow research platform.
- The platform supports structured intake, multi-channel follow-up, call summarization, risk profiling, appetite screening, quote-workflow orchestration, human handoff, bind-outcome capture, and eval-based regression testing.
- We use the platform to generate anonymized traces, quote-ready files, and human-review artifacts for controlled experiments.
- All production-facing binding or regulated advice remains under licensed human or authorized-entity control.

## What We Have To Do Next

### Immediate Work

1. Find or regenerate the broker-license result artifacts.
   - Required outputs: per-model CSV/JSON, accuracy chart, explanation-alignment chart, topic heatmap, cost/latency table, prompt version, dataset hash, and run date.

2. Create a clean benchmark dataset package.
   - Include provenance, permissions, deduplication, train/dev/test split, topic labels, state labels, and contamination controls.

3. Add expert-created held-out questions.
   - Hire or recruit licensed P&C brokers/compliance experts.
   - Use non-public questions to reduce benchmark leakage.

4. Build the state-expansion set.
   - California first.
   - Then comparison states that differ meaningfully on producer terminology, exam structure, prohibited rating factors, minimum coverage requirements, and reciprocity.

5. Convert Arrakis traces into research artifacts.
   - Anonymized conversation.
   - Structured intake record.
   - Quote-ready file.
   - Risk profile.
   - Appetite result.
   - Human handoff summary.
   - Human review decision.

6. Design the licensed-human review protocol.
   - Reviewers score usefulness, correctness, missing information, compliance risk, and approval/rejection.
   - Report inter-rater agreement.

7. Decide the first narrow commercial class.
   - Pick a class where underwriting and quote paths are simple enough to evaluate rigorously.
   - Good candidates from strategy docs include pressure washing, lawn care, simple contractors, and other clean small-commercial classes.

### 30-Day Research Sprint

Deliverables:

- California P&C licensing benchmark with reproducible model results.
- 100-200 expert-reviewed applied broker reasoning scenarios.
- Sanitized Arrakis capability map.
- First quote-ready file rubric.
- First paper draft with result placeholders replaced by real numbers.

### 60-Day Research Sprint

Deliverables:

- Cross-state licensing benchmark.
- Compliance benchmark covering advice, disclosures, prohibited factors, channel consent, and bind/payment boundaries.
- Arrakis-generated quote-ready files scored by licensed reviewers.
- Human baseline from broker/CSR notes or calls where available.
- First productivity estimate: time saved per quote-ready file or bind-ready package.

### 90-Day Research Sprint

Deliverables:

- Full task-ladder benchmark.
- Shadow-advisor recommendation experiment.
- Human approval rate for bind-ready packages.
- Auditability and E&O review study.
- Conference submission draft with sanitized data appendix and reproducibility package.

## Proposed Paper Contribution

The best paper should make four contributions:

1. A broker task ladder.
   - A decomposition of insurance brokerage into measurable sub-skills rather than a binary "replace or not replace" debate.

2. A licensing and regulatory benchmark.
   - State-aware evaluation of insurance knowledge, state-specific rules, explanation alignment, and compliance boundaries.

3. A shadow-broker operating study.
   - Measurement of AI-generated quote-ready and bind-ready packages under licensed-human review.

4. An economic and auditability analysis.
   - Evidence that AI can lower broker labor cost or increase throughput while improving documentation and maintaining safety.

## Suggested Paper Title Options

- Can AI Do The Work Of An Insurance Broker? A Task-Level Benchmark For Regulated Distribution
- From Licensed Knowledge To Bind-Ready Files: Evaluating AI On The Insurance Broker Task Ladder
- Shadow Brokers: Measuring AI Capability, Reliability, And Deployability In Insurance Distribution
- AI-Native Brokerage: A Benchmark And Operating Study For Supervised Insurance Distribution

## Draft Abstract

Insurance brokerage is a regulated, high-stakes workflow that combines licensing knowledge, customer discovery, risk-data collection, market selection, quote comparison, advice, documentation, and transaction control. Prior discussions of AI in insurance often collapse the question into whether software can legally replace a licensed producer. We instead introduce a task-level evaluation framework for measuring which parts of broker work AI systems can perform correctly, reliably, and safely under supervision. The framework begins with state insurance licensing exams, then evaluates coverage explanation, conversational intake, compliance boundaries, quote-ready file generation, market appetite selection, quote comparison, bind-readiness, auditability, and productivity. Using an internal insurance-workflow research platform, we show how AI outputs can be converted into structured artifacts for licensed-human review while preserving accountability for regulated actions. The resulting benchmark separates capability from deployability and provides an empirical path for deciding when AI can replace broker labor, when it should augment licensed producers, and which controls are required before broader delegation.

## Recommended Paper Structure

1. Introduction
   - The wrong question is "is AI licensed?"
   - The right question is task-level replacement under supervision.

2. What Brokers Do
   - Broker competency map.
   - Human path to licensing and practical competence.
   - Quote-to-bind workflow.

3. Task Ladder And Claim Ladder
   - Capability, reliability, deployability.
   - From licensing knowledge to bind-ready packages.

4. Benchmark Design
   - Licensing exams.
   - Applied broker reasoning.
   - Compliance and sales-practice scenarios.
   - Intake, quote-ready file, quote comparison, and bind-readiness tasks.

5. Internal Research Platform
   - Public-safe description of Arrakis as a workflow instrumentation platform.
   - No product details, no client details, no prompt disclosure.

6. Experiments
   - Model sweep.
   - RAG and agent harness variants.
   - Human expert review.
   - Shadow-advisor protocol.
   - Productivity and auditability.

7. Results
   - Licensing pass rates.
   - Explanation alignment.
   - Topic/state ablations.
   - Compliance failure rates.
   - Quote-ready file approval rates.
   - Human time saved.

8. Discussion
   - What AI can replace now.
   - What AI can only support under supervision.
   - What remains human, legal, or market-access constrained.

9. Limitations
   - Public-question contamination.
   - State-law variation.
   - Lack of universal carrier access.
   - Human trust and relationship quality.
   - Licensing and accountable bind authority.

10. Conclusion
    - AI can replace meaningful broker labor before it replaces the licensed broker-of-record.

## Figures The Paper Should Include

1. Broker task ladder.
2. Human broker formation path.
3. Insurance quote-to-bind workflow.
4. Licensing accuracy versus explanation alignment by model.
5. Topic-level licensing heatmap.
6. State-specific conflict accuracy.
7. Compliance scenario failure rates by model or harness.
8. Quote-ready file approval funnel.
9. Human time saved per task.
10. Capability versus reliability versus deployability matrix.

## Conference Direction

The strongest venue depends on which part becomes the main empirical contribution:

1. NeurIPS Datasets and Benchmarks track
   - Best if the core contribution is a new insurance broker benchmark with clean datasets, model sweeps, contamination controls, and reproducible scoring.

2. FAccT or AIES
   - Best if the paper emphasizes regulated decision support, compliance, unfair discrimination, auditability, disclosures, and accountable human oversight.

3. CHI or CSCW
   - Best if the paper emphasizes buyer interaction, trust, conversational disclosure, human-agent collaboration, and licensed producer review workflows.

4. KDD, WWW, or applied ML venues
   - Best if the paper has large-scale production traces, funnel outcomes, conversion, and economic lift.

5. AAAI or IJCAI applied AI tracks
   - Best if the contribution is a complete agent evaluation framework across reasoning, tool use, compliance, and human review.

6. ICAIF or insurance/fintech research venues
   - Best if the paper needs finance/insurance domain reviewers and practical industry credibility, though it may be less likely to become a top general AI paper.

Best-paper strategy:

- Do not submit a product case study.
- Submit a benchmark plus human-supervised operating study.
- Make the task decomposition general enough that it applies beyond Kinro.
- Use insurance as a high-stakes regulated domain where the "AI replaces worker" question can be measured carefully.
- Include real-world workflow artifacts, but sanitize all proprietary implementation details.

## Result Table We Need To Fill

The final paper should report results by task, not as one aggregate score.

Required result fields:

- Task name.
- Dataset size.
- Human baseline.
- Best base model.
- Best RAG or agent harness.
- Best post-trained system, if any.
- Accuracy or approval rate.
- Compliance failure rate.
- Human-review time.
- Cost.
- Latency.
- Deployability status.
- Main residual risk.

Initial task rows:

- P&C licensing knowledge.
- State-specific licensing knowledge.
- Explanation alignment.
- Coverage Q&A.
- Advice-boundary compliance.
- Prohibited rating factor handling.
- Conversational intake.
- Structured extraction.
- Quote-ready file generation.
- Appetite/market selection.
- Quote comparison.
- Shadow recommendation.
- Bind-readiness.
- Handoff summary.
- Audit trail completeness.
- Productivity.

## Bottom Line

The paper should not try to prove that AI can legally be a broker today. It should prove something stronger and more defensible: most of the broker workflow can be decomposed, measured, instrumented, and moved into AI-supervised systems, while the accountable licensed human remains at the points where law, carrier authority, or E&O risk require it.

If the experiments succeed, the final claim is:

- AI can replace large parts of broker labor.
- AI can make licensed brokers more productive and more auditable.
- AI can produce quote-ready and bind-ready artifacts that humans can approve faster than they can create from scratch.
- The remaining barrier is not only model capability; it is licensing, market access, human trust, carrier permission, and accountable governance.

That is the path to a top paper at the intersection of insurance and AI.
