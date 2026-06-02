---
name: ai-agent-create-solo
description: Scaffold a complete individual AI agent from scratch - identity file (SOUL.md), persistent memory, learnings directory, tool registry, agent loop, and run-trace schema. Use this skill whenever the user wants to create a new agent, build an AI agent, add an agent to their app, or scaffold any kind of autonomous AI worker. Triggers on: "create an agent", "build an agent for X", "add an AI agent", "scaffold an agent", "new agent that does X", "I want an agent to handle Y".
---

# ai-agent-create-solo

Scaffold a production-ready individual agent. This skill writes every file the agent needs to be runnable: identity, memory, learnings, tools, the agent loop, and database schema for run tracing.

## Step 1 - Gather inputs

Ask these questions all at once in a single message:

1. **Agent name** - snake_case, becomes the directory name (e.g. `lead_enricher`)
2. **Role** - one sentence: what does this agent do and why does it exist?
3. **Tools** - what does it need to do? (e.g. search the web, send email, read a database). List actions; infer the tools.
4. **Trigger** - on-demand (called from code) or scheduled (cron)?
5. **Model** - which Claude model? Default: `claude-sonnet-4-6`
6. **Token budget** - max tokens per run before abort. Default: 50,000

## Step 2 - Detect the stack

Read `package.json` and any existing schema files before writing the run-trace table.

| Found in `package.json` | Read |
|---|---|
| `drizzle-orm` | `references/db-drizzle.md` |
| `@prisma/client` | `references/db-prisma.md` |
| `pg` or `postgres` (no ORM) | `references/db-raw-sql.md` |
| `@supabase/supabase-js` | `references/db-raw-sql.md` (Supabase uses Postgres - adapt pool to supabase client) |
| `mysql2` (no ORM) | `references/db-raw-sql.md` (adapt syntax) |
| `better-sqlite3` | `references/db-raw-sql.md` (adapt syntax) |
| None | Ask: "What database are you using, or should I skip the run-trace table?" |

Also detect framework (`next`, `express`, `fastify`) to use the right module/import style.

## Step 3 - Write identity and memory files

### `agents/<name>/SOUL.md`

```markdown
# <Name> Agent - Identity

## Role
<one-sentence role from user>

## Behavioral principles
- Complete the task or stop and say why - never fabricate results
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
# <Name> Agent - Memory

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

<!-- Pattern-Key: unique-slug to prevent duplicate entries -->
<!-- Example:
## 2026-06-01 | api-rate-limit
Pattern-Key: api-rate-limit
Category: tool-gotcha
Learning: API returns 429 after ~10 requests/minute.
Action: Added 6s delay between paginated calls.
Status: active
-->
```

### `agents/<name>/.learnings/ERRORS.md`
### `agents/<name>/.learnings/FEATURE_REQUESTS.md`

Empty files with header comment.

### `agents/<name>/TOOLS.md`

For each tool the user listed, write a stub:

```markdown
## <tool_name>

**Purpose:** <what it does>
**Known gotchas:** <!-- fill from experience -->
**Error pattern:**
- On 429: wait 5s, retry once, then throw
- On 404: return null, do not throw
```

## Step 4 - Write the agent loop

Use the Vercel AI SDK (`ai` package) if present. If not present, use the Anthropic SDK directly. Detect from `package.json`.

**3-tier prompt structure** - this is why it matters:
- **Stable tier** (SOUL.md + TOOLS.md): loaded once, stays at the top of the prompt. Anthropic's prompt cache preserves this prefix across thousands of runs - you pay for it once.
- **Volatile tier** (MEMORY.md + current task): injected fresh each run. Only this tier changes.

```typescript
import { generateText, tool } from 'ai'
import { anthropic } from '@ai-sdk/anthropic'
import { z } from 'zod'
import { readFileSync } from 'fs'
import { join } from 'path'
import { randomUUID } from 'crypto'

const AGENT_NAME = '<name>'
const MODEL = '<model>'
const TOKEN_BUDGET = <budget>

// Stable tier - read once at module load, cache-warm across runs
const soul = readFileSync(join(process.cwd(), `agents/${AGENT_NAME}/SOUL.md`), 'utf-8')
const toolsDocs = readFileSync(join(process.cwd(), `agents/${AGENT_NAME}/TOOLS.md`), 'utf-8')
const STABLE_SYSTEM = `${soul}\n\n${toolsDocs}`

function loadMemory(): string {
  try { return readFileSync(join(process.cwd(), `agents/${AGENT_NAME}/MEMORY.md`), 'utf-8') }
  catch {
    console.warn(`[${AGENT_NAME}] MEMORY.md not found - running without memory context`)
    return ''
  }
}

export async function run<TInput>(input: TInput): Promise<string> {
  const runId = randomUUID()
  const memory = loadMemory()
  const system = `${STABLE_SYSTEM}\n\n## Memory\n${memory}\n\nRun ID: ${runId}`

  const result = await generateText({
    model: anthropic(MODEL),
    system,
    prompt: JSON.stringify(input),
    maxTokens: TOKEN_BUDGET,
    tools: {
      // Replace with real tools matching TOOLS.md
      example_tool: tool({
        description: 'Example - replace with real tools',
        parameters: z.object({ input: z.string() }),
        execute: async ({ input }) => ({ result: input }),
      }),
    },
    maxSteps: 10,
  })

  return result.text
}
```

## Step 5 - Write the run-trace schema and wire it

Read the reference file detected in Step 2 for the exact schema and `withRunLogging` helper.

The reference file shows both the schema and a call example. Generate them as a single integrated file - do not leave the wiring as two separate disconnected blocks. The final `index.ts` should compile and run without manual stitching.

The `run()` export should be wrapped at the call site so token counts and cost are written on every successful run:

```typescript
// In src/agents/<name>/index.ts - the wired version
export async function run<TInput>(input: TInput): Promise<string> {
  return withRunLogging(AGENT_NAME, async (_runId) => {
    const memory = loadMemory()
    const system = `${STABLE_SYSTEM}\n\n## Memory\n${memory}`

    const result = await generateText({
      model: anthropic(MODEL),
      system,
      prompt: JSON.stringify(input),
      maxTokens: TOKEN_BUDGET,
      tools: { /* real tools from TOOLS.md - no no-op placeholders */ },
      maxSteps: 10,
    })

    return {
      result: result.text,
      usage: {
        tokensIn: result.usage.promptTokens,
        tokensOut: result.usage.completionTokens,
        costUsd: calcCost(MODEL, result.usage),
      },
    }
  })
}

## Step 6 - Summarise

Print a file tree of everything written. Tell the user:

- Which tool stubs need real implementations (`agents/<name>/TOOLS.md` + `src/agents/<name>/index.ts`)
- How to invoke: `import { run } from './agents/<name>'; await run({ ... })`
- Whether to run a migration (if a new table was created)
- What to run next: `/ai-agent-add-observability` to wire tracing, or `/ai-agent-create-evals` to build a test harness
