---
name: ai-agent-create-workflow
description: Build a scheduled multi-step AI agent pipeline with typed step inputs/outputs, conditional branching, retry logic, checkpoint resume, and cron scheduling. Use when the user wants to automate a recurring multi-step process, build a scheduled pipeline, create a workflow that runs on a cron schedule, or wire sequential/parallel steps together. Triggers on: "create a workflow", "build a pipeline", "scheduled agent", "automate a process", "steps that run in order", "recurring job", "nightly/weekly/hourly agent".
---

# ai-agent-create-workflow

Build a scheduled multi-step workflow. Each step has typed input/output via Zod, can branch conditionally, retry on failure, and suspend for human approval. The workflow is triggered by a cron entry in the `routines` database table.

## Stack detection

Read `package.json` and existing schema files before writing any database code.

| Found in `package.json` | Read |
|---|---|
| `drizzle-orm` | `references/db-drizzle.md` |
| `@prisma/client` | `references/db-prisma.md` |
| `pg`, `postgres`, `mysql2`, or `better-sqlite3` | `references/db-raw-sql.md` |
| `@supabase/supabase-js` | `references/db-raw-sql.md` (Supabase uses Postgres — adapt pool to supabase client) |
| None | Ask: "What database are you using?" |

Also detect framework (`next`, `express`, `fastify`) for the trigger endpoint pattern.

Confirm: "Detected Drizzle + Postgres on Next.js — generating code to match."

**Fixed across all stacks:**
- Vercel AI SDK (`ai` package) for any LLM steps
- Zod for step input/output schemas
- Workflow code lives in `src/workflows/<name>.ts`

## Step 1 — Gather inputs

Ask all at once:

1. **Workflow name** — snake_case (e.g. `weekly_lead_enrichment`)
2. **Steps** — list each step by name and what it does. Note which steps are AI (LLM call) and which are deterministic (API call, DB write, etc.)
3. **Branching** — are there steps that only run if a condition is met? Which ones?
4. **Parallel steps** — are there steps that can run at the same time?
5. **Schedule** — cron expression + timezone (e.g. `0 9 * * 1` = every Monday 9am UTC)
6. **Human approval** — does any step need a human to approve before continuing?
7. **Concurrency** — if the schedule fires while a run is already active, should it skip or queue? Default: skip (coalesce_if_active)

## Step 2 — Write the workflow file

`src/workflows/<name>.ts`:

```typescript
import { generateText } from 'ai'
import { anthropic } from '@ai-sdk/anthropic'
import { z } from 'zod'

// --- Input/output schemas ---
const WorkflowInput = z.object({
  // TODO: define based on user's description
  target: z.string(),
})

const WorkflowOutput = z.object({
  // TODO: define final output shape
  result: z.string(),
  processed: z.number(),
})

// Shared state threaded through all steps
type WorkflowState = {
  input: z.infer<typeof WorkflowInput>
  // Add intermediate results here as steps complete
}

// --- Step definitions ---
// Each step: receives state, returns partial state update
// AI steps call generateText; deterministic steps call APIs/DB directly

async function step_<step_1_name>(state: WorkflowState): Promise<Partial<WorkflowState>> {
  // TODO: implement
  return {}
}

// Conditional step — only runs if condition is met
async function step_<conditional_name>(state: WorkflowState): Promise<Partial<WorkflowState> | null> {
  if (!/* condition from state */) return null // skip
  // TODO: implement
  return {}
}

// Parallel steps — these run concurrently
async function step_<parallel_a>(state: WorkflowState): Promise<Partial<WorkflowState>> {
  // TODO: implement
  return {}
}

async function step_<parallel_b>(state: WorkflowState): Promise<Partial<WorkflowState>> {
  // TODO: implement
  return {}
}

// --- Workflow runner ---
export async function run(rawInput: unknown): Promise<z.infer<typeof WorkflowOutput>> {
  const input = WorkflowInput.parse(rawInput)
  let state: WorkflowState = { input }

  // Step 1
  state = { ...state, ...await step_<step_1_name>(state) }

  // Conditional step
  const conditionalResult = await step_<conditional_name>(state)
  if (conditionalResult) state = { ...state, ...conditionalResult }

  // Parallel steps
  const [resultA, resultB] = await Promise.all([
    step_<parallel_a>(state),
    step_<parallel_b>(state),
  ])
  state = { ...state, ...resultA, ...resultB }

  return WorkflowOutput.parse({ result: '...', processed: 0 }) // TODO
}
```

For any AI step, the pattern is:

```typescript
async function step_ai_example(state: WorkflowState): Promise<Partial<WorkflowState>> {
  const { text } = await generateText({
    model: anthropic('claude-sonnet-4-6'),
    system: 'You are a ...',
    prompt: JSON.stringify(state.input),
    maxTokens: 2000,
  })
  return { aiResult: text }
}
```

## Step 3 — Write the Drizzle scheduling schema

Check if `src/db/schema/routines.ts` exists. If not, create it:

```typescript
import { pgTable, uuid, text, timestamp, boolean, integer } from 'drizzle-orm/pg-core'

export const routines = pgTable('routines', {
  id:                uuid('id').primaryKey().defaultRandom(),
  name:              text('name').notNull().unique(),
  cronExpression:    text('cron_expression').notNull(),
  timezone:          text('timezone').notNull().default('UTC'),
  enabled:           boolean('enabled').notNull().default(true),
  concurrencyPolicy: text('concurrency_policy').notNull().default('coalesce_if_active'),
  // coalesce_if_active: skip new run if one is already active
  // queue: run after current finishes
  nextRunAt:         timestamp('next_run_at', { withTimezone: true }),
  lastRunAt:         timestamp('last_run_at', { withTimezone: true }),
  createdAt:         timestamp('created_at', { withTimezone: true }).defaultNow(),
})

export const routineRuns = pgTable('routine_runs', {
  id:                  uuid('id').primaryKey().defaultRandom(),
  routineId:           uuid('routine_id').notNull().references(() => routines.id),
  status:              text('status').notNull().default('received'),
  // received | running | success | error | skipped
  startedAt:           timestamp('started_at', { withTimezone: true }),
  completedAt:         timestamp('completed_at', { withTimezone: true }),
  failureReason:       text('failure_reason'),
  dispatchFingerprint: text('dispatch_fingerprint').unique(), // idempotency key
})
```

## Step 4 — Seed the routine

Write a seed script or migration that inserts the routine:

```typescript
await db.insert(routines).values({
  name: '<workflow_name>',
  cronExpression: '<cron_expression>',
  timezone: '<timezone>',
  concurrencyPolicy: 'coalesce_if_active',
})
```

## Step 5 — Write the trigger route

`src/app/api/workflows/<name>/trigger/route.ts`:

```typescript
import { NextResponse } from 'next/server'
import { db } from '@/lib/db'
import { routines, routineRuns } from '@/db/schema/routines'
import { eq, and } from 'drizzle-orm'
import { run } from '@/workflows/<name>'
import { randomUUID } from 'crypto'

export async function POST(req: Request) {
  const routine = await db.query.routines.findFirst({
    where: eq(routines.name, '<name>'),
  })
  if (!routine || !routine.enabled) {
    return NextResponse.json({ skipped: true }, { status: 200 })
  }

  // Coalesce check
  if (routine.concurrencyPolicy === 'coalesce_if_active') {
    const activeRun = await db.query.routineRuns.findFirst({
      where: and(eq(routineRuns.routineId, routine.id), eq(routineRuns.status, 'running')),
    })

    if (activeRun) return NextResponse.json({ skipped: 'active_run_exists' }, { status: 200 })
  }

  const runId = randomUUID()
  await db.insert(routineRuns).values({
    id: runId,
    routineId: routine.id,
    status: 'running',
    startedAt: new Date(),
    dispatchFingerprint: req.headers.get('x-dispatch-fingerprint') ?? runId,
  })

  // Run async — don't await so the 202 returns immediately
  // On Vercel: replace with `waitUntil(run(...))` from `@vercel/functions` — plain floating
  // promises are killed when the serverless function freezes after the Response is returned.
  const body = await req.json().catch(() => ({}))
  run(body).then(async () => {
    await db.update(routineRuns)
      .set({ status: 'success', completedAt: new Date() })
      .where(eq(routineRuns.id, runId))
  }).catch(async (err) => {
    await db.update(routineRuns)
      .set({ status: 'error', failureReason: String(err), completedAt: new Date() })
      .where(eq(routineRuns.id, runId))
  })

  return NextResponse.json({ runId }, { status: 202 })
}
```

## Step 6 — Add retry logic to high-risk steps

For any step that calls an external API, wrap it:

```typescript
async function withRetry<T>(
  fn: () => Promise<T>,
  attempts = 3,
  delayMs = 1000,
): Promise<T> {
  for (let i = 0; i < attempts; i++) {
    try {
      return await fn()
    } catch (err) {
      if (i === attempts - 1) throw err
      await new Promise(r => setTimeout(r, delayMs * Math.pow(2, i)))
    }
  }
  throw new Error('unreachable')
}
```

## Step 7 — Summarise

Print the file tree. Tell the user:

- How to trigger manually: `curl -X POST /api/workflows/<name>/trigger`
- How to wire an external cron (Vercel Cron Jobs, GitHub Actions, etc.) to hit the trigger route on the cron schedule
- Which step stubs need real implementations
- Next steps: `/ai-agent-add-observability` to trace step spans, `/ai-agent-create-evals` to test the pipeline
