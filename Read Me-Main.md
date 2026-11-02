PRD Genie (Model: GPT-4o mini, Prompt v4)

PRD Genie is an n8n workflow that turns meeting transcripts, product briefs, and stakeholder notes into a structured **Product Requirements Document (PRD)**, **Epics**, and **User Stories**. After a product manager approves the output, it publishes the PRD to **Google Drive**, adds epics and stories to **Google Sheets**, and creates them in **Jira**.

It runs as a multi-agent pipeline with two human-in-the-loop approval gates. A built-in evaluation harness scores every agent with an LLM-as-judge and sends the results to Langfuse and an n8n Data Table.

- **Workflow name:** `PRD Genie -Model CHATGPT4 mini -PROMPT`
- **Workflow ID:** `kSnZMOBG7n10Mdi2`
- **Primary model:** OpenAI `gpt-4o-mini` (all pipeline agents)
- **Judge model:** OpenAI `gpt-4o` (evaluation only)
- **Prompt version:** `v4`
- **Nodes:** 63

---

## Table of Contents

1. [Key Features](#key-features)
2. [Architecture Overview](#architecture-overview)
3. [Triggers](#triggers)
4. [Pipeline Stages](#pipeline-stages)
5. [Human-in-the-Loop Gates](#human-in-the-loop-gates)
6. [Outputs and Destinations](#outputs-and-destinations)
7. [Evaluation Harness](#evaluation-harness)
8.[Versions of Workflows used for Evaluaton](#Versions-of-Workflows-used-for-Evaluaton)
9.[Final version Selection](#Final-Version-Selection)
10. [Prerequisites and Credentials](#prerequisites-and-credentials)
11. [Configuration Reference](#configuration-reference)
12. [How to Use](#how-to-use)
13. [Anti-Hallucination Design](#anti-hallucination-design)
14. [Troubleshooting](#troubleshooting)
15. [Future improvements](#Future improvements)

---

## Key Features

- **Chat interface with file upload.** Paste text or upload one or more `.txt` / `.md` files.
- **Automatic input classification.** Detects whether the input is a `transcript`, `brief`, or `notes`. Routes to `compare` mode when 2 or more files are uploaded or the user asks to "compare".
- **Grounded requirement extraction.** Every requirement must include a verbatim `source_quote`. A code check then confirms each quote and value appears in the source.
- **Sufficiency gate.** Vague or empty input stops early and returns targeted clarification questions instead of an invented PRD.
- **10-section PRD template** with a compliance check.
- **Epics and user stories** in "As a [persona], I want..., so that..." format, with verbatim acceptance criteria and absolute due dates.
- **Gap analysis and T-shirt sizing** run in parallel.
- **Two approval gates** in the chat before anything is published.
- **Publishing to Drive, Sheets, and Jira** after approval.
- **Evaluation mode** with 9 quality metrics, latency, cost, and Langfuse tracing.

---

## Architecture Overview

```
PM Chat (file upload) ──┐
                        ├─► C1 Ingest & Normalise ─► C2 Input Router ─► A1 Requirement Extractor
Evaluation Trigger ─► Eval Input ┘                                              │
                                                                                ▼
                                                                    C4 Verify & Validate
                                                                                │
                                                                    C3 Sufficiency Gate
                                       ┌────────────── insufficient ────────────┴──── sufficient ───────┐
                                       ▼                                                                ▼
                         A4 Gap Analyzer (short-circuit)                               Gate 1: Requirements Review
                                       │                                               (auto-approved in eval runs)
                         Format Insufficient Reply                                                      │
                                       │                                                   Apply Gate 1 Decision
                         Reply: Insufficient Input                                                      │
                                                                                         A2 PRD Generator (+ Google Docs template tool)
                                                                                                        │
                                                                                         C6 Template Compliance Check
                                                                                                        │
                                                                                         A3 Story Breakdown ─► Parse Stories
                                                                                                        │
                                                                                     ┌──────────────────┴──────────────────┐
                                                                                     ▼                                     ▼
                                                                          A4 Gap Analyzer (full)                 A5 Scope Estimator
                                                                                     └──────────► Join Gap & Scope ◄───────┘
                                                                                                        │
                                                                                         Assemble Review Package
                                                                                                        │
                                                                                     Gate 2: PRD & Stories Approval
                                                                                                        │
                                                                         ┌──────── approved ────────────┴──── rejected ──► Reply: Rejected
                                                                         ▼
                                     ┌──────────────────────────┬────────┴──────────────────────────────┐
                                     ▼                          ▼                                       ▼
                       C8 Render PRD ─► Upload to Drive   C7 Map Epics ─► Sheets + Jira Epics   C7 Map Stories ─► Sheets + Jira Stories
                                     └──────────────────────────┴──► Join Saved Outputs ─► Reply: Saved
```

In evaluation runs, both gates are skipped. Each run then goes to the scoring branch (see [Evaluation Harness](#evaluation-harness)).

---

## Triggers

| Trigger | Type | Purpose |
|---|---|---|
| **PM Chat (file upload)** | Chat Trigger (public, file uploads enabled, response mode = response nodes) | Main production entry point for PMs and TPMs |
| **Evaluation Trigger** | n8n Evaluation Trigger | Runs each test case from the `prd_genie_eval_ground_truth` data table |

---

## Pipeline Stages

### Code nodes (deterministic)

| Node | Responsibility |
|---|---|
| **C1 Ingest and Normalise** | Combines chat text and uploaded files. Cleans whitespace and splits multi-transcript files on `=== TRANSCRIPT` markers. Extracts `Meeting:`, `Date:`, `Attendees:`, and speaker names. Produces `normalized_text` with `### SOURCE:` labels. |
| **C2 Input Router** | Sets `input_type` (`transcript` / `brief` / `notes`) and `route` (`main` / `compare`). |
| **C4 Verify and Validate** | Parses A1 JSON safely. Checks every `source_quote`, `verbatim_values` entry, and acceptance criterion against the source, and raises **red flags** for anything not found. Builds the Gate 1 message. |
| **C3 Sufficiency Gate** (IF) | Continues only if `count > 0` and `status !== 'INSUFFICIENT'`. |
| **C6 Template Compliance Check** | Confirms all 10 PRD sections are present and in order, and that every FR/NFR row has a `source`. Fills missing sections with a placeholder. |
| **Parse Stories** | Parses A3 output and assigns story IDs `S001`, `S002`, and so on. |
| **Assemble Review Package** | Builds the Gate 2 message: PRD summary, compliance status, epics, sized stories, and gap questions. |
| **C8 Render PRD for Drive** | Renders the PRD to HTML as `PRD_<Product>_<YYYY-MM-DD>_v1.html`. |
| **C7 Map Epics / C7 Map Stories** | Maps epics and stories to Sheet and Jira rows. Normalises priority and adds T-shirt size, dependencies, and notes. |

### AI agents (all `gpt-4o-mini`)

| Agent | Role | Output |
|---|---|---|
| **A1 Requirement Extractor** | Extracts requirements verbatim and does not author new ones. Runs a 9-pass procedure: requirements, acceptance criteria, contradictions, ambiguity, named-but-undescribed items, stakeholders, deadlines and dependencies, missing info, and a self-check. | `input_status`, `stated_requirements`, `acceptance_criteria`, `ambiguous_items`, `contradictions`, `stakeholders`, `deadlines`, `dependencies`, `missing_info` |
| **A2 PRD Generator** (temp 0) | Fills the company PRD template using only approved requirements and PM notes. Reads the Google Docs template for structure only. | `prd` with 10 sections |
| **A3 Story Breakdown** | Turns the PRD into epics and per-persona stories, with verbatim acceptance criteria and absolute due dates. | `epics[]`, `stories[]` |
| **A4 Gap Analyzer (full)** | Carries forward every contradiction and ambiguity, scans the source, and compares the PRD and stories to it. Asks one specific, evidence-quoted question per gap. | `gaps[]`, `contradictions_to_resolve[]`, `risks[]`, `what_to_provide` |
| **A4 Gap Analyzer (short-circuit)** (temp 0) | Same prompt, run on the insufficient-input path. | Same as above |
| **A5 Scope Estimator** | Gives T-shirt sizes (S/M/L/XL) and flags underspecified stories. | `estimates[]` |

### PRD template sections
1. Product Overview  2. Goals and Objectives  3. User Personas  4. Feature Requirements (FR/NFR tables with Priority and Source)  5. Acceptance Criteria  6. Out of Scope  7. Dependencies  8. Assumptions  9. Open Questions  10. Timeline

---

## Human-in-the-Loop Gates

Both gates use the Chat node's **Send and Wait** operation and wait up to **24 hours** for a reply.

- **Gate 1: Requirements Review.** Reply `approve` / `approved` / `yes` / `ok` / `lgtm` to continue. Any other reply counts as PM notes and edits, which are passed to A2.
- **Gate 2: PRD and Stories Approval.** Reply `approve` to publish to Drive, Sheets, and Jira. Any other reply rejects the output, nothing is saved, and your comments are echoed back.

---

## Outputs and Destinations

| Destination | Node | Details |
|---|---|---|
| Google Drive | Upload PRD to Drive | HTML PRD in the `PRDGen` folder |
| Google Sheets: Epics | Append Epics Upload | Issue Type, Epic Name, Summary, Description, Priority, Status |
| Google Sheets: Stories | Append Stories Upload | Story ID, Issue Type, Epic Name/Link, Summary/Feature, Description, Acceptance Criteria, Priority, Status, Story Points, T shirt Sizing, Parent / Depends On, Notes |
| Jira: Epics | Create an issue in Jira Software, then Set Epic Due Date | Project `PRD-GEN3`, type `Epics` |
| Jira: Stories | Create Jira Stories, then Set Story Parent, Size and Due Date | Project `PRD-GEN3`, type `Story` |
| Chat | Reply: Saved / Rejected / Insufficient Input | Final message, including the Drive link when saved |

---

## Evaluation Harness

The Evaluation Trigger feeds each row of `prd_genie_eval_ground_truth` into the pipeline (`run_id = run-<date>`, `prompt_version = v4`). The `Eval Run?` nodes skip both gates and the publishing steps. Each run then goes through:

`Collect Pipeline Outputs → Build Judge Prompts → LLM Judge (gpt-4o) → Get Execution Usage → Compute Scores → Log Run Results → Langfuse Trace + Scores → Record Eval Outputs → Set Metrics`

| Metric | Agent | Measures | Pass threshold |
|---|---|---|---|
| M1 Extraction completeness | A1 | All expected requirements, with exact values | 1.0 (T1/T4/T7), else 0.9 |
| M2 Ambiguity flag accuracy | A1 | Correct status, ambiguities and contradictions flagged | 1.0 |
| M3 Groundedness | A2 | PRD items can be traced to approved requirements | 1.0 |
| M4 Format compliance | A2 | From the C6 code check | 1.0 |
| M5 Story format and persona | A3 | "As a..." format, priority, one story per persona | 1.0 |
| M6 AC fidelity | A3 | No added acceptance-criteria conditions | 1.0 |
| M7 Gap relevance | A4 | Questions are specific, not generic | 0.8 |
| M8 Gap recall | A4 | All expected gaps and risks surfaced | 1.0 (T2/T3/T5/T10), else 0.9 |
| M9 Underspecified flag | A5 | Flags match the expected values | 0.8 |

Operational metrics: **E1** task success, **E2** latency (s), **E3** cost in USD (gpt-4o-mini $0.15/$0.60, gpt-4o $2.50/$10.00 per 1M tokens), **M12** routing accuracy, `hallucination_free`, `metric_avg`, and per-agent scores and costs.

Results go to the `prd_genie_eval_runs` data table, the ground-truth table, Langfuse (US cloud, failures don't stop the run), and the n8n Evaluations tab.

---

## Versions of Workflows used for Evaluation

PRD Genie -Model GPT4o mini -PROMPT ENGINEERED -( This workflow uses GPT 4 but evaluated after Prompt Engineering)

PRD Genie -Model GPT4 mini( initial working flow with all nodes including auto Evaluation)

PRD Genie copy-Upgraded Models-5 and 6 ( This version is used to evaluate higher models. Models used were 5 and 6)

---
## Final version Selection

Final Version selected is " PRD Genie -Model GPT4o mini -PROMPT ENGINEERED"  This workflow showed better scores for metrics and reduced Hallucinations. Using Higher chat GPT models were not improvement

Note:Did not use RAG since score for the suggested PRD format compliance metric is perfect for all test data runs in evaluation
---
## Prerequisites and Credentials

| Credential | Used by |
|---|---|
| OpenAI (currently n8n Gateway credits) | All agents and the judge |
| Google Docs OAuth2 | PRD template tool |
| Google Drive OAuth2 | Upload PRD |
| Google Sheets OAuth2 | Epics and Stories sheets |
| Jira Software Cloud API | Issue creation and updates |
| n8n API | Cost calculation (evaluation only) |
| Langfuse API | Tracing and scores (evaluation only) |

---

## Configuration Reference

| Item | Value |
|---|---|
| PRD template | `1HFbyt1ii_OhYlV2KOhmhdt1TFdu33dFN` |
| Drive folder | `PRDGen` (`1VQCYJWwTBv5XJpPaYbYU0BXJx6K_Bn61`) |
| Epics sheet | `1GzigIhgV1K2vsPiEq5c5B7BewxPfNsLnQ28MmaKL4Cw` |
| Stories sheet | `1TNW51wpd6dJHWLD1cmsMDl2LGQ_Y90hd5Y36s6GVwFU` |
| Jira project | `PRD-GEN3` (10004), Epic 10019, Story 10022 |
| Ground-truth table | `prd_genie_eval_ground_truth` (`KXuXYkWUTVTT64Hq`) |
| Eval runs table | `prd_genie_eval_runs` (`6rEQNKNVpQcCqgLu`) |
| Gate wait time | 24 hours |

To switch models, edit the A1–A5 Model sub-nodes and update `PRICES` in **Compute Scores**.

---

## How to Use

**Production:** open the public chat URL'https://prdgenie.app.n8n.cloud/webhook/107c93fe-99c3-4dcf-8855-8980a931e64b/chat

then paste or upload your source. Review Gate 1 and reply `approve` or send edits. Review Gate 2 and reply `approve` to publish. You get a Drive link, and the epics and stories appear in Sheets and Jira.

**Recommended transcript format:**
```
=== TRANSCRIPT 1
Meeting: Q3 Reporting Kickoff
Date: 2026-10-15
Attendees: Priya (PM), Alex (Eng Lead)

Priya: Search must return results in under 200ms p95.
Alex: We need to integrate with Salesforce API v52.
```

**Evaluation:** fill `prd_genie_eval_ground_truth` (`test_id`, `category`, `input_text`, `expected_output`, `must_not`, `metrics`, `negative_case`, `expected_route`). Run it from the Evaluations tab, then review the results there, in `prd_genie_eval_runs`, and in Langfuse.

---

## Anti-Hallucination Design

- Every requirement has a verbatim source quote, and the quote is checked in code.
- Numbers, units, and versions stay exactly as written.
- A sufficiency gate stops vague input and asks specific questions instead.
- Empty sections say "Not stated in source - see Open Questions".
- Contradictions are listed, never resolved.
- Stories never get an invented benefit: "so that [benefit not stated - confirm]".
- Due dates come only from stated deadlines.
- Humans approve twice before anything is published.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| "No PRD was generated..." | Input was too vague. Answer the listed questions and resubmit. |
| `[RED FLAG: unverified]` at Gate 1 | Correct or remove that requirement in your Gate 1 reply. |
| Compliance ISSUES at Gate 2 | A2 returned partial JSON. Reject and resubmit. |
| Gate never resumes | Reply within 24 hours in the same chat session. |
| Jira due date or parent missing | Check Jira permissions and that the date is in `YYYY-MM-DD` format. |
| Langfuse scores missing | Check the Langfuse credential. Failures there don't stop the run. |
| Eval cost is null | The n8n API credential is missing or token usage couldn't be read. |

##Future improvements

Include Version comparison agent
Fine tune models to improve scores
Note:Did not use RAG since score for the suggested PRD format compliance metric is perfect for all test data runs in evaluation
