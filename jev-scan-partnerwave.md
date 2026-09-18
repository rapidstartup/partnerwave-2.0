# Jev scan — PartnerWave

## Scope and research baseline

- **Board:** PartnerWave
- **Repository:** `rapidstartup/partnerwave-2.0`
- **Reviewed commit:** `25376defa18f3b0d69dce6f40192d7063215bb12` (`25376de`, “Initial commit”, 2025-10-11)
- **Review date:** 2026-09-18
- **Mode:** research and notes only. No product behavior, deployment, merge, or Autopilot setting was changed.
- **Suggested VD/site title:** **Autopilot OFF — PartnerWave Jev System One Evaluation**

The checked-out repository is a very small Mastra E2B coding-agent template, despite
its PartnerWave repository name. It does not currently contain product code for partner
records, forms, outreach, scraping, or a user-facing application. Consequently, the
replacement candidates below are proposed architecture for future PartnerWave work, not
observed production flows.

The requested `typesafe-ai` skill was added under `.agents/skills/typesafe-ai`; its lock
file records the source and content hash. The installation did not add the TypeSafe SDK
or any TypeSafe API key, and this review did not make API calls.

## Existing AI and integration inventory

| Location | Observed behavior | Implication |
| --- | --- | --- |
| `src/mastra/agents/coding-agent.ts` | One `@mastra/core` `Agent` uses `@ai-sdk/openai` with `openai('gpt-4.1')`. | This is the sole explicit generative-LLM execution path. |
| `src/mastra/agents/coding-agent.ts` | `Memory` enables semantic recall and working memory, with `fastembed`, `LibSQLStore`, and `LibSQLVector`. | Embedding/vector recall is separate from the GPT-4.1 generation path. |
| `src/mastra/tools/e2b.ts` | Exposes sandbox file, code, and shell-command tools. | The GPT agent can select tools that have high-impact side effects inside E2B; it is not an outreach/scraping implementation. |
| `src/mastra/index.ts` | Uses a local LibSQL store and Pino observability. | No lake/data-pipeline abstraction is present. |
| `.env.example` and `README.md` | Document OpenAI and E2B environment variables. | No provider configuration should be copied into client code or notes. |

Repository-wide checks found **no Firecrawl integration, Cloudflare Worker configuration,
Cloudflare code, lake/data-lake implementation, form workflow, outreach flow, scraper,
or Autopilot feature/configuration** in tracked source. There is also no `Firecrawl`,
`cloudflare`, `worker`, or `lake` dependency in `package.json`.

## What Jev is suitable for

TypeSafe’s Jev is a System One model: it evaluates supplied text/JSON state and returns
typed `Choice`, `Score`, or yes/no (`Noul`) answers with probabilities. It does not
generate prose, explain reasoning, choose its own actions, browse/crawl, or accept
images/audio/video. Code must own retrieval, validation, control flow, side effects,
and confidence-gated escalation.

The current TypeSafe guide says most queries complete in roughly 100 ms and describes a
target, not a guaranteed measured outcome, of more than 100x intelligence-to-speed-and-
cost. PartnerWave should measure latency, unit cost, and error rates on its own traffic
before claiming a savings or replacement result.

## Replace or cascade candidates

### 1. Form finding and filling

**Candidate workflow**

1. Code reads an allowed page or a user-provided form, collects DOM labels, field types,
   select options, validation constraints, and existing CRM values.
2. Jev answers narrow questions in parallel: which known field maps to which supplied
   profile value; whether a required value is missing; whether a declaration/consent is
   present; and whether a form is safe to submit.
3. Code fills only a selected, validated candidate value. Show the mapping, evidence,
   and confidence to the user; require confirmation before any submission.

**Why this can replace an LLM step:** a closed-set `Choice` over discovered controls or
known CRM values removes prompt parsing and lets code reject unknown fields. For emails,
phones, URLs, and similar values, first find candidates deterministically, then let Jev
select the requested span. This preserves the source string and avoids retyping it.

**Do not replace:** Jev cannot invent missing free-text answers, parse arbitrary image
CAPTCHAs, or safely decide that legal attestations may be accepted. Escalate ambiguous
mappings, free-text composition, and submission authorization to the user or (where
appropriate) a generative model.

### 2. Outreach preparation and reply handling

**Candidate workflow**

- Score a partner/prospect against explicit ICP dimensions: segment fit, geography,
  integration fit, evidence freshness, contact role, and consent/compliance status.
- Choose from approved templates, campaign steps, approved value propositions, and CRM
  records; verify claims against source evidence before a template is eligible.
- Classify incoming replies into closed actions such as interested, referral, objection,
  unsubscribe, out-of-office, or uncertain; then update a review queue or a deterministic
  CRM state machine.

**Why this can replace an LLM step:** partner qualification, routing, template selection,
claim verification, and reply classification are structured decisions. Jev can make
those faster, typed, inspectable, and confidence-gated instead of relying on generated
JSON or agentic loops.

**Do not replace:** Jev cannot write individualized email copy. Keep a generative LLM
only for draft copy when needed, bound it to approved evidence/template variables, and
run its output through claim/compliance checks. It must not send messages autonomously.

### 3. Scrape, research, and evidence selection

**Candidate workflow if Firecrawl and/or a Cloudflare Worker are introduced**

1. Firecrawl (or another compliant fetcher) retrieves permitted pages; a Cloudflare
   Worker can provide authenticated ingestion, rate limiting, cache/freshness metadata,
   and a queue boundary. Neither is present today.
2. Deterministic code extracts links, headings, JSON-LD, contacts, dates, and candidate
   spans; stores source URL, retrieval time, and source text.
3. Jev ranks which passages answer a defined partner-research question, evaluates
   whether an answer exists, selects among extracted candidates, and scores source
   relevance/freshness/ICP fit.
4. Code stores an evidence card with quoted lines and routes low-confidence,
   contradictory, stale, or missing-evidence records to review or an expensive LLM.

**Why this can replace an LLM step:** it removes broad “summarize this website into
JSON” calls when the required output is a finite decision or selection. The documented
line-search pattern pairs a ranking `Choice` with a yes/no answer-exists test; that
prevents treating the nearest irrelevant passage as evidence.

**Do not replace:** Jev is not a crawler, browser, OCR system, schema generator, or
free-form summarizer. Keep those concerns in fetch/parsing code and reserve generative
models for genuinely open-ended synthesis after the evidence is selected.

## Product opportunities enabled by typed judgments

1. **Evidence-backed partner-fit board:** reusable per-partner scores for ICP fit,
   readiness, strategic value, and data freshness. Users can change weights and see
   rankings without rerunning judgments.
2. **“Why this partner?” evidence cards:** source passages, retrieval time, confidence,
   and explicit “not found/partially supported” states instead of opaque summaries.
3. **Form mapping review:** a side-by-side preview showing which existing CRM fields map
   to target form fields, with a blank/ask-user option for anything uncertain.
4. **Outreach safety queue:** selects approved campaign/template/status actions and
   detects consent, unsubscribe, sensitive-data, or unsupported-claim risk; it never
   sends automatically.
5. **Reply and partner-intake router:** turns new replies/submissions into typed,
   confidence-aware CRM queues, with ambiguous cases assigned to a human.
6. **Deduplication and relationship review:** scores whether two company/contact records
   are the same or whether two sources make compatible claims, preserving separate
   records when confidence is inadequate.

## Risks, controls, and recommended rollout

| Risk | Control |
| --- | --- |
| The repository lacks PartnerWave product flows, so there is no measured LLM cost/latency baseline. | Instrument a real target workflow first; do not claim a replacement benefit from template code alone. |
| A typed answer is schema-safe, not necessarily factually correct. Confidence is calibrated across groups, not a guarantee for one record. | Build a labeled evaluation set; log state/question/version/outcome; set conservative, action-specific thresholds. |
| Candidate selection only works when all valid choices are supplied. | Use recall-oriented deterministic extraction, include a `none` option, validate after selection, and use staged search for more than 255 choices. |
| Jev accepts text only and does not generate prose. | Use separate OCR/browser/parser services for non-text sources and retain an LLM only for bounded drafting or hard synthesis. |
| Scraping and outreach create legal, platform-policy, privacy, and deliverability obligations. | Respect robots/terms and provider contracts; minimize and protect PII; enforce consent/unsubscribe rules; rate-limit; retain source provenance; require human approval before external submission or sending. |
| The existing GPT-4.1 agent can invoke file and command tools in E2B. | Do not use Jev as an authorization layer for destructive tools. Keep allowlists, user confirmation, sandbox limits, and audit logs in deterministic code. |
| Model or source drift can change outcomes. | Version questions and sources, capture freshness metadata, replay evaluation sets, and review regressions before threshold changes. |

### Recommended first experiment (Autopilot OFF)

Implement no autonomous action. Select one read-only workflow—e.g., ranking scraped
partner-evidence passages or classifying incoming replies—and run Jev in shadow mode.
Compare it with the current human/LLM result on a labeled sample. Display the raw
evidence, probabilities, and confidence; auto-suggest only high-confidence, reversible
actions. Keep every external effect (form submission, CRM mutation, contact export, and
outreach send) behind a user confirmation and existing policy checks.

## Primary sources consulted

- TypeSafe System One: <https://docs.typesafe.ai/concepts/system-one.md>
- Building with System One: <https://docs.typesafe.ai/concepts/how-to-build-with-system-one.md>
- Confidence guidance: <https://docs.typesafe.ai/confidence.md>
- JavaScript SDK: <https://docs.typesafe.ai/sdk/javascript.md>
- Function calling: <https://docs.typesafe.ai/cookbooks/function_calling.md>
- Pre-parsed value extraction: <https://docs.typesafe.ai/cookbooks/pre_parsed_value_extraction_cookbook.md>
- Line-by-line semantic search: <https://docs.typesafe.ai/cookbooks/semantic_find.md>
- Citation checking: <https://docs.typesafe.ai/cookbooks/citation_check.md>
