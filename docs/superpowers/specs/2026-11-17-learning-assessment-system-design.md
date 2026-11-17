# Multi-Agent Learning Assessment System — Design Spec

**Date:** 2026-11-17
**Source brief:** `AI Agent Creation Workshop Challeng1.docx`
**Deliverable:** Working prototype

## 1. Goal

A Next.js prototype in which several AI agents work together to assess learner submissions. Learners submit five kinds of work: written answers, code, Excel workbooks, case studies and PowerPoint presentations. The system produces evidence-backed scores, feedback, gap analysis, course recommendations and competency reports, and sends uncertain grades to a human reviewer.

### Mapping to the brief's requirements

| # | Requirement | Covered by |
|---|---|---|
| 1 | Understand objectives and competencies | `assessment.yaml` (competency, level, objectives) + Assessment Designer agent |
| 2 | Generate assessments | Assessment Designer agent + "New assessment" screen, which writes a new folder |
| 3 | Accept multiple submission types | Folder scan + extractors (txt/md/docx, code, xlsx, pptx) |
| 4 | Identify the type and route it | Router (code) |
| 5 | Evaluate against rubrics and objective checks | Five specialist evaluators + Excel Checker (`checks.yaml`) |
| 6 | Evidence-backed scores and feedback | Shared evaluator output schema (evidence per criterion) |
| 7 | Gaps and interventions | Gap & Recommendation agent + learning catalog |
| 8 | Reports for L&D and learners | Report agent + Competency report + My development plan |
| 9 | Human review, overrides, escalation | Confidence gate, Reviewer queue, Override audit log |

## 2. Key decisions

| Topic | Decision |
|---|---|
| Platform | Next.js (App Router, server actions), TypeScript, one app for UI and backend |
| LLM | OpenAI API, structured outputs validated with Zod |
| Orchestration | **Fixed pipeline**: code sets the order of steps; LLM agents do the grading and reasoning; every step is recorded and shown as an agent trace |
| Storage | SQLite + Prisma |
| Users | Role switcher (Learner / L&D / Reviewer); no login; seeded users |
| Input | An assessments folder holding both assessment definitions and submissions |
| Rubrics | Required in every folder; Excel assessments also need `checks.yaml` |
| Code evaluation | Reviewed by the LLM only; learner code is never run |
| Escalation | Only on low AI confidence (plus failed evidence checks, length cut-offs and type mismatches, see §4) |
| Sample data | Built by us: about 5 assessments, about 4 learners each, plus a learning catalog |
| Testing | Unit tests on pure logic + live grading-quality check; no mocks or fakes anywhere |
| UI | Tailwind + shadcn/ui, Recharts |

Out of scope: authentication, Microsoft 365 / Copilot Studio integration, running learner code, rendering Office files faithfully, conversational L&D copilot. The design leaves room to add the copilot later.

## 3. Input: assessments folder

The location comes from `ASSESSMENTS_DIR` (default `./assessments`). Each subfolder holds one assessment:

```
assessments/
  excel-l2-budget/
    assessment.yaml
    task.xlsx | task.md | task.docx
    rubric.yaml
    checks.yaml          # Excel assessments only
    submissions/
      alice.xlsx         # filename without extension = learner id
      bob.xlsx
learning-catalog.yaml    # courses that recommendations are chosen from
```

**assessment.yaml**
```yaml
id: excel-l2-budget
title: Excel Level 2 – Budget Model
type: excel            # text | code | excel | case_study | presentation
competency: excel-financial-modelling
level: 2               # 1–4
objectives:
  - Use conditional aggregation (SUMIF/SUMIFS)
  - Build a variance analysis
passMark: 70           # percent
codeLanguage: python   # code assessments only
```

**rubric.yaml**
```yaml
criteria:
  - id: accuracy
    name: Calculation accuracy
    weight: 40
    levels:
      excellent: All totals correct and traceable
      partial: Minor errors in one area
      low: Major errors or missing totals
  - id: approach
    name: Modelling approach
    weight: 30
    levels:
      excellent: Clear, reusable structure using SUMIF/SUMIFS and named inputs
      partial: Works but uses hard-coded values or repeated formulas
      low: No clear structure; totals typed in by hand
```
The weights must add up to 100. The weight of a criterion is its maximum score.

**checks.yaml** (Excel only; objective checks)
```yaml
- { id: c1, sheet: Summary, cell: D15, expect: 48250, tolerance: 1 }
- { id: c2, sheet: Summary, cell: D15, formulaContains: SUMIF }
- { id: c3, sheet: Variance, cell: E4, expect: "Over budget" }
```

**learning-catalog.yaml**
```yaml
- { id: xl-201, title: Conditional Aggregation in Excel, competency: excel-financial-modelling, level: 2, format: video, durationMins: 45 }
```

### Scanning
- Triggered when the app starts and by the **Scan folder** button.
- Validates each folder: YAML schema, rubric weights add up to 100, `checks.yaml` present for Excel, type known. Any failure sets the assessment to `status=INVALID` with `invalidReason`, and its submissions are not imported.
- Inserts or updates assessments. Imports new submissions, skipping any already imported by comparing `(assessmentId, learnerId, fileHash)`. Creates unknown learners on the fly.
- Recognised submission extensions: `.txt .md .docx` (text, case_study), `.py .js .ts .java .sql .cs` (code), `.xlsx` (excel), `.pptx` (presentation).

### Generation (requirement 2)
The L&D form asks for competency, objectives, level, type and pass mark. The Assessment Designer drafts a task and rubric (and checks for Excel), and L&D edits them in the UI. **Save** writes `assessment.yaml`, `task.md`, `rubric.yaml` and `checks.yaml` (Excel) to a new folder and runs a scan. For Excel, the generated task is a written specification in `task.md`; no `.xlsx` template is generated.

## 4. Grading pipeline

**Grade pending** runs the pipeline on every `PENDING` submission, three at a time by default (`GRADING_CONCURRENCY`). Each submission is handled on its own, so one failure never stops the others. Every step writes an `AgentStep` row.

```
Extract → Route → [Excel Checker] → Evaluate → Confidence Gate → Gap & Recommend → Report
```

### Plain-code steps
1. **Extractor** turns each file into text the LLM can read:
   - txt/md: kept as is.
   - docx: mammoth.
   - pptx: JSZip + XML parsing, giving per-slide title, bullets and notes.
   - xlsx: exceljs, giving a per-sheet list of cells with value and formula.
   - code: kept as is, with line numbers.
   - Content longer than `MAX_SUBMISSION_CHARS` is cut off. The cut is logged in the trace and is passed to the confidence gate.
2. **Router** maps `assessment.type` to an evaluator. If the file extension doesn't fit the type, the submission is set to `NEEDS_REVIEW` with reason "Type mismatch" and is not graded.
3. **Excel Checker** (Excel only) runs `checks.yaml` and returns `{id, passed, actual, expected}` for each check.

### LLM agents
All agents use OpenAI structured outputs and are validated with Zod. If validation fails, the agent retries once with the validation error included. If an API call errors, the agent retries twice with increasing waits. After that, the submission becomes `FAILED`.

| Agent | Input | Output |
|---|---|---|
| Assessment Designer | competency, objectives, level, type | task text, rubric, checks (Excel) |
| Text Evaluator | task, rubric, submission text | EvaluatorOutput |
| Code Evaluator | task, rubric, language, numbered code | EvaluatorOutput (reasons through correctness, edge cases, quality, readability; nothing is run) |
| Excel Evaluator | task, rubric, extracted cells, **Checker results (to be treated as fixed facts)** | EvaluatorOutput |
| Case Study Evaluator | task, rubric, submission text | EvaluatorOutput (problem framing, analysis, evidence, recommendations) |
| Presentation Evaluator | task, rubric, per-slide content | EvaluatorOutput (storyline, clarity, accuracy, use of data) |
| Gap & Recommendation | rubric, evaluation, competency, catalog entries for that competency | gaps[] + 1–3 course ids **chosen only from the catalog** |
| Report | evaluation, gaps, recommendations | learner feedback summary (markdown) |

The cohort/competency summary for L&D is built by a separate Report-agent call that runs when the Competency report screen is opened. It summarises the scores and gaps already in the database.

**EvaluatorOutput** (shared by all five evaluators)
```ts
{
  criteria: { id: string; score: number; maxScore: number;
              evidence: string[];   // exact quotes, cell refs ("Summary!D15"), slide refs ("Slide 3"), line refs ("L12-18")
              rationale: string }[];
  strengths: string[];
  improvements: string[];
  confidence: number;        // 0–1
  confidenceReason: string;
}
```
Code then works out `totalScore` (sum of criteria scores, each capped at its `maxScore`), `percent` and `passed` (percent ≥ `passMark`). The LLM's own arithmetic is never used.

### Confidence gate (code)
`NEEDS_REVIEW` if any of these holds:
- `confidence < CONFIDENCE_THRESHOLD` (default 0.7)
- an evidence quote is not found in the extracted submission. Matching ignores case and extra whitespace. Cell, slide and line references are checked to exist.
- the submission was cut off for length
- the type didn't match (from the Router)

Otherwise the status is `GRADED`. Gap & Recommendation and Report still run for `NEEDS_REVIEW` submissions so the reviewer sees the full picture. Learners don't see the results until they are finalized.

## 5. Data model (Prisma, SQLite)

| Model | Fields |
|---|---|
| Learner | id, name, team |
| Competency | id, name, description |
| Assessment | id, folderPath, title, type, competencyId, level, objectives (JSON), passMark, codeLanguage?, rubric (JSON), checks (JSON?), status `ACTIVE`/`INVALID`, invalidReason? |
| Submission | id, assessmentId, learnerId, filePath, fileHash, status `PENDING`/`GRADING`/`GRADED`/`NEEDS_REVIEW`/`FAILED`/`FINALIZED`, reviewReasons (JSON), error?, createdAt |
| Evaluation | id, submissionId, source `AI`/`HUMAN_OVERRIDE`, criteria (JSON), totalScore, percent, passed, strengths, improvements, confidence?, feedbackSummary, isCurrent, createdAt |
| AgentStep | id, submissionId, agent, order, inputSummary, output (JSON), durationMs, tokensIn, tokensOut, error?, createdAt |
| Gap | id, evaluationId, competencyId, criterionId, severity `LOW`/`MED`/`HIGH`, description, courseIds (JSON) |
| Override | id, evaluationId (new), previousEvaluationId, reviewer, criterionId, oldScore, newScore, reason, createdAt |

Competencies are seeded from `learning-catalog.yaml` and the competency ids used in `assessment.yaml`.

## 6. Human review (requirement 9)

- **Review queue:** all `NEEDS_REVIEW` submissions, showing their review reasons and confidence.
- **Review screen:** submission preview next to editable per-criterion scores, the AI's evidence and its rationale.
  - **Accept:** status becomes `FINALIZED`; the AI evaluation stays current.
  - **Save override:** one or more criteria get a new score, and every changed criterion needs a reason. This creates a new `Evaluation` (`source=HUMAN_OVERRIDE`, `isCurrent=true`; the previous one gets `isCurrent=false`), writes one `Override` row per changed criterion, recalculates the totals and pass/fail, reruns Gap & Recommendation and Report, and sets the status to `FINALIZED`.
- Reviewers can reopen and override `GRADED` submissions in the same way.
- The AI evaluation is never deleted. The dashboard shows the override rate and the average score change.

## 7. Screens

A role switcher in the top bar selects the role. In Learner mode, a second dropdown picks the learner.

**L&D**
1. Dashboard: counts by status, pass rate, review-queue size, override rate. Buttons for Scan folder and Grade pending (with progress).
2. Assessments: list with type, competency, level and validity (plus the reason if invalid). Each opens a detail page with task, rubric, checks and submissions.
3. New assessment: form → Generate → edit → Save to folder.
4. Competency report: learners × competencies heatmap, top gaps, most-recommended courses, AI cohort summary, CSV export.

**Shared:** Submission detail
- Score card (total, pass/fail, status, "AI" or "Human-reviewed" label).
- Per-criterion scores with evidence and rationale.
- Strengths and improvements, gaps and recommended courses.
- **Agent trace** tab (L&D and Reviewer only): a timeline of each step, showing agent, duration, tokens and expandable output.

**Reviewer**
5. Review queue.
6. Review screen.

**Learner**
7. My results: only `GRADED` and `FINALIZED` submissions; others show "Under review" or "Pending".
8. My development plan: gaps grouped across all assessments, plus recommended courses.

## 8. Error handling

| Failure | Behaviour |
|---|---|
| Invalid assessment folder | `INVALID` with reason; scan continues |
| Corrupt or unreadable file | Submission `FAILED` with extractor error; Retry button |
| Type mismatch | `NEEDS_REVIEW`, not graded |
| OpenAI error or timeout | 2 retries with increasing waits → `FAILED` |
| Schema validation failure | 1 retry with the error → `FAILED` |
| Score inconsistency | Code caps scores at max and recalculates totals |
| Submission cut off for length | Logged in the trace; forces `NEEDS_REVIEW` |
| Missing `OPENAI_API_KEY` | UI banner; scan works, grading and generation disabled |

`FAILED` and `NEEDS_REVIEW` submissions can be sent back to `PENDING` with **Retry**.

## 9. Sample data

The seed script (`pnpm seed`) creates `./assessments` and `learning-catalog.yaml`:

| Assessment | Type | Submissions |
|---|---|---|
| Written: Data privacy principles | text (.md/.docx) | strong, weak, borderline, ambiguous |
| Code: Python data cleaning function | code (.py) | strong, buggy, borderline, wrong-type (.pptx) |
| Excel L2: Budget model | excel (.xlsx) | strong, wrong formulas, borderline, hard-coded values |
| Case study: Customer churn | case_study (.docx) | strong, weak, borderline, off-topic |
| Presentation: Q3 business review | presentation (.pptx) | strong, weak, borderline, sparse |

The .xlsx files are generated with exceljs, .docx with the `docx` package and .pptx with pptxgenjs. Each submission has an expected result in `src/seed/expectations.yaml`:
```yaml
excel-l2-budget/alice: { min: 85, max: 100, passed: true }
excel-l2-budget/dave:  { min: 55, max: 80, expectReview: true }
```

## 10. Testing

- **Unit tests (Vitest, no LLM):**
  - YAML and rubric validation.
  - Each extractor, run on the sample files.
  - Excel Checker: values, tolerance, `formulaContains`, missing sheet or cell.
  - Router mapping and mismatch detection.
  - Score recalculation and capping.
  - Confidence gate, including evidence-quote and reference checks.
  - Override recalculation and audit rows.
  - Scanner skipping files already imported.
- **No fakes or mocks.** Unit tests cover only pure logic that runs on real inputs (the real sample files and real data structures). Anything that calls the LLM, including the retry and schema-retry paths, runs only against the real OpenAI API through `pnpm eval`. Those failure paths have no automated tests; they show up only when they happen in real runs, where they are recorded in the agent trace.
- **Live grading-quality check (`pnpm eval`, real OpenAI):**
  - Grades every sample submission.
  - Reports, for each submission: score inside the expected range, pass/fail correct, review flag correct.
  - Overall agreement rate.
  - Prints a table and writes `eval-report.md`.

## 11. Configuration

`.env`: `OPENAI_API_KEY`, `OPENAI_MODEL` (required; any OpenAI model that supports structured outputs), `ASSESSMENTS_DIR=./assessments`, `CONFIDENCE_THRESHOLD=0.7`, `GRADING_CONCURRENCY=3`, `MAX_SUBMISSION_CHARS=40000`, `DATABASE_URL=file:./dev.db`.

## 12. Code layout

```
src/
  app/                    # routes: (lnd)/, (reviewer)/, (learner)/, submissions/[id]
  components/
  lib/
    config.ts
    db.ts
    scanner/              # folder scan + YAML schemas
    extractors/           # text, docx, code, xlsx, pptx
    excel-checker.ts
    router.ts
    agents/               # one file per agent + shared openai client, prompts, schemas
    pipeline/             # orchestrator, confidence gate, scoring, retry
    review/               # accept/override logic
  seed/                   # sample data generators + expectations.yaml
  eval/                   # live grading-quality runner
prisma/schema.prisma
tests/
```
