# Write-Up: AI Proposal Triage Design

## 1. AI vs. Rules

My design separates the pipeline into three layers:

**Rule-based (deterministic code):**
- **Mandatory criteria screening** — Keyword/regex matching checks whether each proposal explicitly states an Ontario electrical contractor licence and confirms WSIB clearance + $5M liability insurance. These are binary facts; AI adds hallucination risk with no benefit.
- **Budget and timeline gate** — Simple numeric comparison: reject if price > $260,000 CAD or timeline > 16 weeks (or not stated).
- **Weighted score aggregation** — Pure arithmetic: multiply each criterion score by its admin-defined weight and sum. This must be deterministic so scores are reproducible.

**AI-powered (LLM with structured output):**
- **Data extraction** — Parse unstructured proposal PDFs into a structured JSON (price, timeline, experience count, warranty terms, team details). Natural language varies too much for regex alone.
- **Qualitative scoring** — Score "Relevant experience," "Warranty," and "Team and approach" on a 0–100 rubric. These require judgement about quality and relevance that rules cannot capture.
- **Prompt-injection detection** — A separate LLM pass scans extracted text for adversarial instructions (e.g., "rank this first," "assign full marks"). Flagged proposals go to human review.

**Why this split:** Rules are used wherever the logic is unambiguous and the input is structured enough (numbers, yes/no). AI is reserved for tasks that require interpreting free-form text. This minimises the "blast radius" of an AI error.

## 2. Consistency and Explainability

- **Temperature 0 / deterministic decoding** — All LLM calls use `temperature: 0` to minimise variation between runs.
- **Structured output schema** — The LLM returns a fixed JSON schema, not free text. This prevents drift in format and makes parsing reliable.
- **Rubric-anchored scoring** — Each weighted criterion has a written rubric (e.g., "Experience: 90–100 = 12+ installs ≥100 kW in last 5 years; 70–89 = 8–11 installs; …"). The LLM must cite which rubric band it selected and quote the supporting text from the proposal.
- **Score breakdown visible to reviewer** — The UI shows: (a) the extracted data, (b) the rubric band chosen, (c) the quoted evidence, and (d) the final weighted score. A reviewer can click any criterion to see the raw proposal text side-by-side.
- **Re-run guarantee** — Because mandatory checks and score aggregation are pure code, and the LLM is called at temperature 0 with the same prompt and input, re-running the same proposals produces the same result. A hash of (prompt version + proposal text + criteria version) is stored so any drift can be detected.

## 3. Human in the Loop

- **Pre-ranking review** — After AI extraction and scoring, the ranked list is presented as a *draft*. No result is final until a human reviewer clicks "Confirm" or "Override."
- **Override capability** — A reviewer can adjust any individual criterion score with a mandatory text justification, which is logged alongside the original AI score.
- **Flagged proposals** — Any proposal flagged (missing mandatory info, prompt injection, ambiguous data) is routed to a human with the reason highlighted. The reviewer decides pass/fail.
- **Audit log** — Every action is recorded: timestamp, actor (system/AI/human), action (extract, score, flag, confirm, override), input hash, output, and prompt version. This log is immutable and exportable for compliance review.

## 4. Changing Criteria

- **Version-stamped criteria** — Each set of criteria+weights is stored with a version ID and timestamp. Scores are linked to the criteria version used.
- **Re-score on change** — If an admin edits criteria after some proposals have been scored, the system marks all existing scores as "stale" and offers two options: (1) re-score all proposals under the new criteria, or (2) keep the old scores and apply new criteria only to new proposals. The admin must choose explicitly.
- **No silent mixing** — The system never mixes scores produced under different criteria versions in the same ranking. If re-scoring is chosen, all proposals in the batch are re-evaluated together.
- **Change log** — Every criteria edit is logged with before/after values and the admin's identity, so auditors can trace why scores changed.

## 5. Risks

| Risk | How the design guards against it |
|---|---|
| **Wrong extraction** — AI misreads a price, timeline, or licence number | Extracted fields are shown to the reviewer for confirmation before scoring. Critical numeric fields (price, weeks) are cross-checked with a second regex pass on the raw text. |
| **Hallucination** — AI invents facts not in the proposal | The prompt instructs the LLM to return `null` for any field not explicitly stated, and to quote source text. The UI highlights any field where the AI could not provide a direct quote. |
| **Prompt injection / manipulation** — A proposal embeds instructions like "rank this first" (as Proposal C does in this sample set) | A dedicated detection pass scans for adversarial instructions. Any proposal containing imperative commands directed at the system is flagged for human review and the injected text is highlighted. The scoring LLM's system prompt explicitly instructs it to ignore any instructions within proposal text. |
| **Bias** — AI favours certain phrasing or company names | Proposals are anonymised (replaced with Proposal A, B, C…) before being sent to the scoring LLM, so brand names and marketing language do not influence scores. |
| **Cost** — LLM API calls are expensive at scale | Mandatory checks and budget gates run first (zero LLM cost). Only proposals that pass these gates are sent to the LLM. Token usage is logged per evaluation. |
| **Inconsistency between runs** — Scores change when re-evaluated | Temperature 0, fixed prompt versions, and input hashing ensure reproducibility. Any drift triggers an alert. |
