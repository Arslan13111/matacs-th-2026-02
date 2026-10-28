# AI Proposal Triage Design

**Candidate:** Anas M  
**Reference:** MATACS-TH-2026-02  
**Time spent:** ~60 minutes  

## Repository Contents

| File / Folder | Description |
|---|---|
| [diagram/](diagram/) | Process-flow diagram (PNG) showing AI steps, rule-based steps, and human checkpoints |
| [WRITEUP.md](WRITEUP.md) | Answers to the five design questions |
| [PROMPTS.md](PROMPTS.md) | AI prompts for extraction and scoring, exactly as they would be sent |
| [ranking.json](ranking.json) | Expected ranking of proposals A–E |
| [prototype/](prototype/) | Optional prototype script that runs the prompt on the samples |

## AI Tool Usage

I used Claude (Anthropic) as an AI assistant during this exercise to help structure my thinking and draft sections of the write-up. I reviewed and edited all output for accuracy and completeness.

**Where I disagreed with the AI:** The AI initially suggested using AI for *all* evaluation steps including mandatory pass/fail checks. I corrected this because mandatory criteria (licence, WSIB/insurance) are binary factual checks that are more reliably handled by deterministic rules with keyword matching — AI adds hallucination risk for no benefit here. I also insisted on explicitly flagging the prompt-injection attack embedded in Proposal C, which the AI did not initially identify as a distinct risk category.
