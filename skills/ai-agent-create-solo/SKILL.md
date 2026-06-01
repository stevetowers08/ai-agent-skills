---
name: ai-agent-create-solo
description: Scaffold a complete individual AI agent from scratch — identity file (SOUL.md), persistent memory, learnings directory, tool registry, Vercel AI SDK agent loop, and Drizzle run-trace schema. Use this skill whenever the user wants to create a new agent, build an AI agent, add an agent to their app, or scaffold any kind of autonomous AI worker. Triggers on: "create an agent", "build an agent for X", "add an AI agent", "scaffold an agent", "new agent that does X", "I want an agent to handle Y".
---

# ai-agent-create-solo

Scaffold a production-ready individual agent. This skill writes real files — identity, memory, learnings, tools, the agent loop, and database schema. By the end, the agent is runnable.

## Stack assumptions

- Next.js + Vercel AI SDK (`ai` package) for the agent loop
- Drizzle ORM + Postgres for run tracing
- TypeScript strict mode
- Files live under `agents/<name>/` (identity/memory) and `src/agents/<name>/` (code)

If the project uses a different stack, adapt the code generation but keep the file structure.

## Step 1 — Gather inputs

Ask these questions. Ask them all at once in a single message, not one at a time.

1. **Agent name** — snake_case, becomes the directory name (e.g. `lead_enricher`)
2. **Role** — one sentence: what does this agent do and why does it exist?
3. **Tools** — list the tools it needs (e.g. web search, email send, database read). If unsure, ask for the agent's actions and infer the tools.
4. **Trigger** — on-demand (called from code) or scheduled (cron)?
5. **Model** — which Claude model? Default: `claude-sonnet-4-6`
6. **Token budget** — max tokens per run before abort. Default: 50000

## Step 2 — Write identity and memory files

### `agents/<name>/SOUL.md`

```markdown
# <Name> Agent — Identity

## Role
<one-sentence role from user>

## Behavioral principles
- Complete the task or stop and say why — never fabricate results
- Stay within your tool list; do not improvise new capabilities
- If uncertain about an action with side effects, ask before proceeding
- Write one LEARNINGS entry after every run where something was unexpected

## Cost policy
Abort if projected token usage exceeds <budget> tokens in a single run.
Log the abort reason to ERRORS.md before stopping.

## Escalation triggers
- Tool returns an error three times in a row → stop and log
- Output would affect more than 10 records → pause for human approval
- Any action involving financial data or PII → log to LEARNINGS.md
```

### `agents/<name>/MEMORY.md`

```markdown
# <Name> Agent — Memory

_Cap: 800 tokens. Compact when approaching limit._

## Current context
<!-- What the agent currently knows about its operating environment -->

## Open decisions
<!-- Things that need a human call -->

## Known constraints
<!-- Rate limits, API quirks, data shape assumptions -->
```

### `agents/<name>/.learnings/LEARNINGS.md`

```markdown
# Learnings

Format: one entry per learning. Tag with date and run context.

<!-- Pattern-Key: unique slug to prevent duplicate entries -->
<!-- Example:
## 2026-06-01 | hubspot-rate-limit
Pattern-Key: hubspot-rate-limit
Category: tool-gotcha
Learning: HubSpot search API returns 429 after ~10 requests/minute with a free key.
Action: Added 6s delay between paginated calls.
Status: active
-->
```

### `agents/<name>/.learnings/ERRORS.md`

```markdown
# Errors

Format: date | error type | what failed | what was tried | resolution

<!-- Example:
## 2026-06-01 | tool-failure
Tool: web_search
Error: "429 Too Many Requests"
Tried: immediate retry x2
Resolution: add exponential backoff with jitter
-->
```

### `agents/<name>/.learnings/FEATURE_REQUESTS.md`

```markdown
# Feature Requests

Capabilities the agent tried to use but doesn't have yet.

<!-- Example:
## 2026-06-01
Wanted: ability to send Slack messages
Context: user asked for a summary notification after each run
-->
```

### `agents/<name>/TOOLS.md`

For each tool the user listed, write a stub:

```markdown
# <Name> Agent — Tool Registry

## <tool_name>

**Purpose:** <what it does>
**Input schema:**
\`\`\`typescript
{ param: string } // Zod shape
\`\`\`
**Known gotchas:**
- <!-- fill in from experience -->
**Error pattern:**
- On 429: wait 5s, retry once, then throw
- On 404: return null, do not throw
```

## Step 3 — Write the agent loop

Create `src/agents/<name>/index.ts`:

```typescript
import { generateText, tool } from 'ai'
import { anthropic } from '@ai-sdk/anthropic'
import { z } from 'zod'
import { db } from '@/lib/db'
import { agentRuns } from '@/db/schema/agent_runs'
import { readFileSync } from 'fs'
import { join } from 'path'
import { randomUUID } from 'crypto'

const AGENT_NAME = '<name>'
const MODEL = '<model>'
const TOKEN_BUDGET = <budget>

// --- 3-tier prompt injection ---
// Stable tier: read once, cache-warm across all turns
const soul = readFileSync(join(process.cwd(), `agents/${AGENT_NAME}/SOUL.md`), 'utf-8')
const tools_doc = readFileSync(join(process.cwd(), `agents/${AGENT_NAME}/TOOLS.md`), 'utf-8')

// Volatile tier: read fresh each run (memory snapshot)
function loadMemory(): string {
  try {
    return readFileSync(join(process.cwd(), `agents/${AGENT_NAME}/MEMORY.md`), 'utf-8')
  } catch {
    return ''
  }
}

const STABLE_SYSTEM = `${soul}\n\n${tools_doc}`

export async function run<TInput>(input: TInput): Promise<void> {
  const runId = randomUUID()
  const startedAt = new Date()

  // Log run start
  await db.insert(agentRuns).values({
    id: runId,
    agentName: AGENT_NAME,
    status: 'running',
    startedAt,
    invocationSource: 'on_demand',
  })

  try {
    const memory = loadMemory()
    const systemPrompt = `${STABLE_SYSTEM}\n\n## Memory (snapshot at run start)\n${memory}\n\nRun ID: ${runId} | Started: ${startedAt.toISOString()}`

    const result = await generateText({
      model: anthropic(MODEL),
      system: systemPrompt,
      prompt: JSON.stringify(input),
      maxTokens: TOKEN_BUDGET,
      tools: {
        // TODO: add tools here — each matches an entry in TOOLS.md
        example_tool: tool({
          description: 'Example tool — replace with real tools',
          parameters: z.object({ input: z.string() }),
          execute: async ({ input }) => {
            return { result: input }
          },
        }),
      },
      maxSteps: 10,
    })

    // Log success
    await db
      .update(agentRuns)
      .set({
        status: 'success',
        finishedAt: new Date(),
        tokensIn: result.usage.promptTokens,
        tokensOut: result.usage.completionTokens,
        actionTaken: result.text.slice(0, 500),
      })
      .where(eq(agentRuns.id, runId))

  } catch (error) {
    await db
      .update(agentRuns)
      .set({
        status: 'error',
        finishedAt: new Date(),
        errorMsg: error instanceof Error ? error.message : String(error),
      })
      .where(eq(agentRuns.id, runId))

    throw error
  }
}
```

## Step 4 — Write the run-trace schema

Check if `src/db/schema/agent_runs.ts` exists. If it does, add any missing columns. If it doesn't, create it:

```typescript
import { pgTable, uuid, text, timestamp, integer, numeric } from 'drizzle-orm/pg-core'

export const agentRuns = pgTable('agent_runs', {
  id:               uuid('id').primaryKey().defaultRandom(),
  agentName:        text('agent_name').notNull(),
  status:           text('status').notNull(), // queued | running | success | error | skipped
  invocationSource: text('invocation_source').notNull().default('on_demand'), // on_demand | scheduled
  startedAt:        timestamp('started_at', { withTimezone: true }).notNull(),
  finishedAt:       timestamp('finished_at', { withTimezone: true }),
  tokensIn:         integer('tokens_in'),
  tokensOut:        integer('tokens_out'),
  costUsd:          numeric('cost_usd', { precision: 10, scale: 6 }),
  actionTaken:      text('action_taken'),
  errorMsg:         text('error_msg'),
})
```

## Step 5 — Write the migration

```bash
npx drizzle-kit generate
```

Run this and confirm with the user before running `npx drizzle-kit migrate`.

## Step 6 — Summarise what was created

Print a file tree of everything written. Then tell the user:

- What tool stubs need real implementations (point to `src/agents/<name>/index.ts`)
- Whether to run the migration
- How to invoke the agent: `import { run } from '@/agents/<name>'; await run({ ... })`
- Next skill to run: `/ai-agent-add-observability` to wire tracing, or `/ai-agent-create-evals` to build a test harness
