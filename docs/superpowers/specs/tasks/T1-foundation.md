# T1 — Foundation

**Parent spec:** [../2026-11-17-learning-assessment-system-design.md](../2026-11-17-learning-assessment-system-design.md)
**Depends on:** nothing
**Used by:** every other task

## 1. Purpose

Set up the shared base the other tasks build on:
- the Next.js project and tooling;
- validated configuration;
- the **complete** database schema;
- the app shell with the role switcher;
- the **agent wrapper**, through which every LLM call goes, and the **step recorder**, which records every pipeline step for the agent trace.

T1 adds no assessment features of its own.

## 2. Scope

**In:**
- Project setup with **pnpm**, using exactly this command:
  ```
  pnpm dlx shadcn@latest init --preset b0 --template next --pointer
  ```
  - This creates a Next.js (App Router, TypeScript) app with Tailwind and shadcn/ui, applies the `b0` preset and gives buttons a pointer cursor.
  - Keep whatever the preset chooses for theme, component base and folder layout. If it doesn't create a `src/` directory, the paths in this spec and the parent spec are relative to the app root instead.
  - The app must end up at the repository root, alongside `docs/`. If the command creates a subfolder, move its contents up to the root.
  - Then add: TypeScript `strict`, Vitest, Prisma, Zod, the `openai` SDK.
  - pnpm is the only package manager: commit `pnpm-lock.yaml` and don't add any other lockfile.
- `src/lib/config.ts`: environment variables validated with Zod.
- `prisma/schema.prisma`: every model from parent spec §5. `src/lib/db.ts`: a single Prisma client and typed JSON helpers.
- App shell: top bar, role switcher, learner picker, navigation that depends on the role, banner when the LLM is not configured.
- A **Setup status** page as the home page for every role.
- `src/lib/agents/run-agent.ts`: the agent wrapper.
- `src/lib/pipeline/steps.ts`: the step recorder for steps that don't call the LLM.
- `pnpm check:llm`: a check against the real API.

**Out:** any scanning, extraction, grading or seeding (T2 onward). The dashboard (T8) replaces the Setup status page as the L&D home page.

## 3. Configuration (`src/lib/config.ts`)

Read once, validated with Zod, and exported as a frozen `config` object.

| Variable | Type | Default | Notes |
|---|---|---|---|
| `DATABASE_URL` | string | `file:./dev.db` | Read by Prisma |
| `OPENAI_API_KEY` | string? | – | If missing, the LLM is reported as not configured |
| `OPENAI_MODEL` | string? | – | Required for the LLM to count as configured |
| `ASSESSMENTS_DIR` | string | `./assessments` | Resolved to an absolute path |
| `CONFIDENCE_THRESHOLD` | number 0–1 | `0.7` | |
| `GRADING_CONCURRENCY` | int 1–10 | `3` | |
| `MAX_SUBMISSION_CHARS` | int > 0 | `40000` | |

- `config.llmConfigured: boolean` is true only when both the API key and the model are set.
- Invalid values (e.g. `CONFIDENCE_THRESHOLD=abc`) stop startup with a message naming the variable.
- `.env.example` lists every variable with a comment.

## 4. Database (`prisma/schema.prisma`)

SQLite. Every status and enum-like field is a `String`, with its allowed values defined as TypeScript unions in `src/lib/types.ts` and checked with Zod when written. JSON fields are `String` columns, accessed only through the typed helpers in `src/lib/db.ts` (`parseJson(schema, value)`, `toJson(value)`). This avoids relying on Prisma's SQLite enum and JSON support.

```prisma
model Learner {
  id          String       @id            // = submission filename stem
  name        String
  team        String?
  submissions Submission[]
}

model Competency {
  id          String       @id
  name        String
  description String?
  assessments Assessment[]
  gaps        Gap[]
}

model Assessment {
  id            String       @id          // from assessment.yaml
  folderPath    String
  title         String
  type          String                    // text|code|excel|case_study|presentation
  competencyId  String
  competency    Competency   @relation(fields: [competencyId], references: [id])
  level         Int
  objectives    String                    // JSON string[]
  passMark      Int
  codeLanguage  String?
  task          String                    // extracted task text
  rubric        String                    // JSON Rubric
  checks        String?                   // JSON ExcelCheck[]
  status        String                    // ACTIVE|INVALID
  invalidReason String?
  updatedAt     DateTime     @updatedAt
  submissions   Submission[]
}

model Submission {
  id            String       @id @default(cuid())
  assessmentId  String
  assessment    Assessment   @relation(fields: [assessmentId], references: [id])
  learnerId     String
  learner       Learner      @relation(fields: [learnerId], references: [id])
  filePath      String
  fileHash      String
  status        String                    // PENDING|GRADING|GRADED|NEEDS_REVIEW|FAILED|FINALIZED
  reviewReasons String       @default("[]") // JSON string[]
  error         String?
  createdAt     DateTime     @default(now())
  updatedAt     DateTime     @updatedAt
  evaluations   Evaluation[]
  steps         AgentStep[]
  @@unique([assessmentId, learnerId, fileHash])
}

model Evaluation {
  id              String     @id @default(cuid())
  submissionId    String
  submission      Submission @relation(fields: [submissionId], references: [id])
  source          String                  // AI|HUMAN_OVERRIDE
  criteria        String                  // JSON CriterionResult[]
  totalScore      Float
  percent         Float
  passed          Boolean
  strengths       String                  // JSON string[]
  improvements    String                  // JSON string[]
  confidence      Float?
  feedbackSummary String?                 // markdown, from Report agent (T6)
  isCurrent       Boolean    @default(true)
  createdAt       DateTime   @default(now())
  gaps            Gap[]
  overrides       Override[] @relation("NewEvaluation")
}

model AgentStep {
  id           String     @id @default(cuid())
  submissionId String
  submission   Submission @relation(fields: [submissionId], references: [id])
  runId        String                     // one grading run of this submission
  agent        String                     // e.g. "Extractor", "ExcelEvaluator"
  kind         String                     // LLM|CODE
  order        Int
  inputSummary String
  output       String?                    // JSON
  durationMs   Int
  tokensIn     Int?
  tokensOut    Int?
  error        String?
  createdAt    DateTime   @default(now())
}

model Gap {
  id           String     @id @default(cuid())
  evaluationId String
  evaluation   Evaluation @relation(fields: [evaluationId], references: [id])
  competencyId String
  competency   Competency @relation(fields: [competencyId], references: [id])
  criterionId  String
  severity     String                     // LOW|MED|HIGH
  description  String
  courseIds    String                     // JSON string[]
}

model Override {
  id                   String     @id @default(cuid())
  evaluationId         String
  evaluation           Evaluation @relation("NewEvaluation", fields: [evaluationId], references: [id])
  previousEvaluationId String
  reviewer             String
  criterionId          String
  oldScore             Float
  newScore             Float
  reason               String
  createdAt            DateTime   @default(now())
}
```

Differences from parent spec §5:
- `AgentStep` gains `runId`, so a retried grading run doesn't mix its steps with earlier runs, and `kind`, to tell LLM steps from plain-code steps.
- `Assessment` gains `task`, the extracted task text kept for prompts and display.

Shared types in `src/lib/types.ts`: `AssessmentType`, `SubmissionStatus`, `Role`, `Rubric`, `RubricCriterion`, `ExcelCheck`, `CriterionResult`, each with a Zod schema. Later tasks import these and never define their own copies.

## 5. App shell and roles

- **Role state** is kept in cookies: `role` (`lnd` | `reviewer` | `learner`, default `lnd`) and `learnerId`. Server components read them with `getSession()` from `src/lib/session.ts`; a server action `setRole(role, learnerId?)` writes them.
- **Top bar:**
  - app name;
  - role `<Select>`;
  - when the role is Learner, a second `<Select>` listing learners from the database, or the disabled text "No learners yet";
  - a warning banner when `!config.llmConfigured` saying "LLM not configured: set OPENAI_API_KEY and OPENAI_MODEL. Grading and generation are disabled."
- **Navigation:** defined in `src/lib/nav.ts` as `{ role, label, href }[]`. T1 adds only "Setup status". Later tasks add their own entries to this file.
- **Access control:** pages call `requireRole(...roles)`. A wrong role redirects to `/` with a toast. This is for the demo only, not security.
- **Setup status page (`/`)** shows real checks:
  - database reachable, with row counts per table;
  - `ASSESSMENTS_DIR` exists, with its subfolder count;
  - LLM configured, with the model name;
  - result of the last `check:llm` run, if one exists.
- UI kit: shadcn/ui components `button card badge table select tabs dialog input textarea label sonner skeleton`.

## 6. Agent wrapper (`src/lib/agents/run-agent.ts`)

Every LLM call in the system goes through this function.

```ts
type RunAgentArgs<T> = {
  agent: string;              // display name, e.g. "TextEvaluator"
  system: string;
  user: string;
  schema: z.ZodType<T>;       // structured output schema
  schemaName: string;
  trace?: StepContext;        // when grading a submission
  inputSummary?: string;      // short, human-readable, stored on the AgentStep
  temperature?: number;       // default 0.2
};

type RunAgentResult<T> = { output: T; tokensIn: number; tokensOut: number; durationMs: number };

function runAgent<T>(args: RunAgentArgs<T>): Promise<RunAgentResult<T>>;
```

How it behaves:
1. If `!config.llmConfigured`, throws `LlmNotConfiguredError`.
2. Calls OpenAI using the SDK's Zod structured-output helper, with `config.openaiModel`.
3. **If the API call fails** (network error, 429, 5xx, timeout of 60s): retries up to 2 more times, waiting 1s then 4s. A 4xx other than 429 is not retried.
4. **If the output fails Zod validation** or can't be parsed: retries once, adding the validation error to the user message ("Your previous output was invalid: …").
5. If all attempts fail, it throws `AgentError { agent, cause, attempts }`.
6. When `trace` is given, writes exactly one `AgentStep` (`kind=LLM`) at the end: its output, total duration, tokens summed across attempts and the number of attempts in `inputSummary`; or, on failure, the error.

## 7. Step recorder (`src/lib/pipeline/steps.ts`)

```ts
type StepContext = { submissionId: string; runId: string };

function newRun(submissionId: string): StepContext;   // runId = cuid
function recordStep<T>(ctx: StepContext, agent: string, inputSummary: string,
                       fn: () => Promise<T>): Promise<T>;  // kind=CODE
```

- `recordStep` times `fn` and writes an `AgentStep` with the result serialised as JSON. If the result is longer than 20,000 characters, it is cut short and the cut is marked. If `fn` throws, it records the error and throws it again.
- `order` is the number of steps already in this `runId`. Steps within one submission run one after another, so the count is enough.
- `runAgent` uses the same function to write its LLM steps.

## 8. Scripts (`package.json`)

| Script | Does |
|---|---|
| `dev`, `build`, `start`, `lint` | standard (run as `pnpm <script>`) |
| `test` | `vitest run` |
| `db:push` | `prisma db push` (prototype: no migration history) |
| `db:reset` | delete `dev.db` and push again |
| `check:llm` | real call through `runAgent` with a tiny schema `{ ok: boolean, model: string }`. Prints the result, tokens and time, and writes the result to `.llm-check.json` for the Setup status page. Exits with a non-zero code on failure. |

## 9. Tests (Vitest; no mocks or fakes)

- `config`: valid environment variables parse; each invalid value is rejected with its variable named; `llmConfigured` is true only when both the key and the model are set. These tests pass plain objects to the exported `parseConfig(env)` function, which is ordinary input, not a fake.
- `db` JSON helpers: values survive a round trip; invalid JSON or a schema mismatch throws a clear error.
- `types`: each Zod schema accepts valid examples and rejects invalid ones (e.g. rubric weights that don't add up to 100).
- `steps`: against a real SQLite test database (`file:./test.db`, recreated per run), `recordStep` writes rows with increasing `order`, records the error and throws it again, and cuts off large outputs.
- `runAgent` has no unit tests; it is verified against the real API with `pnpm check:llm`.

## 10. Done when

- [ ] `pnpm install && pnpm db:push && pnpm dev` starts without errors on a clean clone.
- [ ] The Setup status page shows real database, folder and LLM status.
- [ ] The role switcher changes the navigation, and learner picking works (shows "No learners yet" while the database is empty).
- [ ] The banner appears when the API key or model is missing, and disappears once both are set.
- [ ] `pnpm check:llm` succeeds with a real key and writes `.llm-check.json`.
- [ ] `pnpm test` and `pnpm lint` pass.
- [ ] `.env.example` documents every variable.
