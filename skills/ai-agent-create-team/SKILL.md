---
name: ai-agent-create-team
description: Scaffold a supervisor + worker agent team with delegation protocol, handoff wiring, shared trace context, and OTel parent-child spans. Use when the user wants to build a multi-agent system, create a team of agents, add a supervisor agent, set up agent orchestration, or wire one agent to delegate to another. Triggers on: "create an agent team", "build a supervisor agent", "set up multi-agent", "agent that delegates to other agents", "orchestrate multiple agents", "supervisor and worker agents".
---

# ai-agent-create-team

Scaffold a supervisor-worker agent team. The supervisor receives a task and delegates subtasks to specialist workers. Each agent gets its own identity and memory. Spans nest so the full team trace is visible in one tree.

## Step 0 - Detect the stack

Read `package.json` before writing any code - same detection as `ai-agent-create-solo`.

| Found in `package.json` | Use |
|---|---|
| `drizzle-orm` | Drizzle for `logRun` queries |
| `@prisma/client` | Prisma singleton (`import { prisma } from '@/lib/prisma'`) |
| `pg`, `postgres`, or `@supabase/supabase-js` | Raw SQL / pg pool |
| None | Ask before generating any DB code |

Confirm in one line: "Detected Drizzle + Next.js - generating supervisor to match."

## Step 1 - Gather inputs

Ask all at once:

1. **Team name** - snake_case (e.g. `outreach_team`)
2. **Supervisor role** - what does the supervisor decide and coordinate?
3. **Workers** - list each worker by name and single-sentence role (e.g. `researcher: finds company info`, `writer: drafts the email`)
4. **Delegation mode** - sequential (one worker at a time) or parallel (workers run concurrently where possible)?
5. **Output mode** - does the supervisor get each worker's full message history (`full_history`) or just the final result (`last_message`)? Default: `last_message`.
6. **Model** - one model for all, or different models per agent?

## Step 2 - Write identity files per agent

### `agents/<team>/AGENTS.md` (team topology)

```markdown
# <Team Name> - Agent Topology

## Supervisor
**Name:** <supervisor_name>
**Role:** <role>
**Delegation mode:** <sequential|parallel>
**Output mode:** <last_message|full_history>

## Workers
| Name | Role | Tools |
|------|------|-------|
| <worker_1> | <role> | <tools> |
| <worker_2> | <role> | <tools> |

## Handoff rules
- Supervisor always waits for worker results before producing final output
- Workers never communicate with each other directly
- If a worker fails, supervisor logs the failure and either retries or skips, per task type
```

For each agent (supervisor + each worker), write `agents/<team>/<agent_name>/SOUL.md` with:
- Role and scope boundaries (what this agent does AND what it explicitly does not do)
- Which agents it may communicate with (workers: none; supervisor: all workers)
- Escalation triggers

## Step 3 - Write the supervisor

`src/agents/<team>/supervisor.ts`:

```typescript
import { generateText, tool } from 'ai'
import { anthropic } from '@ai-sdk/anthropic'
import { z } from 'zod'
import { randomUUID } from 'crypto'
import { readFileSync } from 'fs'
import { join } from 'path'
import { trace, context, SpanStatusCode } from '@opentelemetry/api'
import type { Span } from '@opentelemetry/api'
import { run as runWorker1 } from './<worker_1>'
import { run as runWorker2 } from './<worker_2>'
import { logRun } from '@/lib/agent-runs'

const AGENT_NAME = '<team>_supervisor'
const tracer = trace.getTracer(AGENT_NAME)

// Stable tier - read once at module load, cache-warm across runs
const soul = readFileSync(join(process.cwd(), `agents/<team>/supervisor/SOUL.md`), 'utf-8')
const toolsDocs = readFileSync(join(process.cwd(), `agents/<team>/supervisor/TOOLS.md`), 'utf-8')
const STABLE_SYSTEM = `${soul}\n\n${toolsDocs}`

// Worker registry - workers accept only the task string; OTel context propagates via context.with()
const WORKERS: Record<string, (task: string) => Promise<string>> = {
  '<worker_1>': (task) => runWorker1({ task }),
  '<worker_2>': (task) => runWorker2({ task }),
}

export async function run(input: { task: string }): Promise<string> {
  const teamRunId = randomUUID() // stable ID - correlates all spans in this team run

  return tracer.startActiveSpan(`${AGENT_NAME}.run`, async (rootSpan) => {
    rootSpan.setAttributes({
      'agent.name': AGENT_NAME,
      'entity.type': 'agent',
      'team.run_id': teamRunId,
    })

    const runId = await logRun.start(AGENT_NAME)

    try {
      const result = await generateText({
        model: anthropic('<model>'),
        system: STABLE_SYSTEM,
        prompt: input.task,
        tools: {
          delegate_task: tool({
            description: 'Delegate a subtask to a specialist worker agent. Choose the worker whose role matches the subtask.',
            parameters: z.object({
              worker: z.enum(['<worker_1>', '<worker_2>']).describe('Which worker to delegate to'),
              task: z.string().describe('The specific subtask for this worker, with full context'),
            }),
            execute: async ({ worker, task }) => {
              return tracer.startActiveSpan(`delegate.${worker}`, async (childSpan) => {
                childSpan.setAttributes({ 'agent.name': worker, 'team.run_id': teamRunId })
                try {
                  const workerFn = WORKERS[worker]
                  if (!workerFn) throw new Error(`Unknown worker: ${worker}`)
                  // context.with() propagates the active span into the worker call -
                  // worker's internal spans nest automatically without passing a Span object
                  const result = await context.with(
                    trace.setSpan(context.active(), childSpan),
                    () => workerFn(task)
                  )
                  childSpan.setStatus({ code: SpanStatusCode.OK })
                  return result
                } catch (err) {
                  childSpan.setStatus({ code: SpanStatusCode.ERROR, message: String(err) })
                  throw err
                } finally {
                  childSpan.end()
                }
              })
            },
          }),
        },
        maxSteps: 15,
      })

      await logRun.end(runId, 'success', result.usage)
      rootSpan.setStatus({ code: SpanStatusCode.OK })
      return result.text

    } catch (error) {
      await logRun.end(runId, 'error', undefined, String(error))
      rootSpan.setStatus({ code: SpanStatusCode.ERROR, message: String(error) })
      throw error
    } finally {
      rootSpan.end()
    }
  })
}
```

## Step 4 - Write each worker

`src/agents/<team>/<worker_name>.ts` - use the same structure as `ai-agent-create-solo` but:
- The worker's `run()` function accepts `(input: { task: string })` - no Span parameter needed
- OTel context propagates automatically via `context.with()` in the supervisor - worker spans nest correctly without passing a Span object across call boundaries
- Workers do not write to `agent_runs` directly - the supervisor owns the run record

## Step 5 - Write `AGENTS.md` escalation rules

After writing code, add to `agents/<team>/AGENTS.md`:

```markdown
## Known handoff issues
<!-- Fill in after first real run -->

## Conversation ID convention
All agents in this team share the same conversationId derived from the supervisor's runId.
This groups all worker spans under one trace in the observability dashboard.
```

## Step 6 - Summarise

Print the file tree. Tell the user:

- How to invoke: `import { run } from '@/agents/<team>/supervisor'; await run({ task: '...' })`
- That worker spans will nest under the supervisor in any OTel-compatible trace viewer
- Next steps: `/ai-agent-add-observability` to configure the trace exporter, `/ai-agent-create-evals` to test delegation routing

## Step 7 - Optional: team management UI

Ask once: "Do you want a developer dashboard for this team? It adds a topology view, MCP server setup, and agent config panel."

If yes:

### Pages to generate

**`/team`** - topology canvas
- Install `reactflow` if not present
- ReactFlow graph: supervisor node + worker nodes, directed delegation edges
- Each node: agent name, role badge, status dot (idle/busy/error/offline), last-run timestamp
- Click a node: sidebar with agent detail (model, tools enabled, recent runs)
- Auto-refresh every 10s via `useInterval` polling `/api/team/agents`; pause when tab is hidden

**`/team/mcps`** - MCP server setup
- Table of registered MCP servers: name, command, env var keys (values masked), status, last-tested
- "Add server" form: name, command (e.g. `npx -y @modelcontextprotocol/server-memory`), key-value env var repeater
- "Test" button per row - POSTs to `/api/team/mcps/[id]/test`, spawns the command as a child process with 5s timeout, shows success/error inline
- Env var values are write-only in the UI; warn the developer to move secrets to a secrets manager before production

### API routes

| Method | Route | What it does |
|---|---|---|
| GET | `/api/team/agents` | Returns `Agent[]` from AGENTS.md + heartbeat overlay |
| GET | `/api/team/mcps` | Reads `.agent-team.json` store |
| POST | `/api/team/mcps` | Validates with Zod, persists to store |
| DELETE | `/api/team/mcps/[id]` | Removes from store, returns 204 |
| POST | `/api/team/mcps/[id]/test` | Child process spawn with 5s timeout |

**File-backed store:** `.agent-team.json` at project root (add to `.gitignore`). Export `readStore()` and `writeStore(data)` from `lib/team/store.ts`.

**Agent status detection:** check for heartbeat files in `.agent-heartbeats/<agentId>` - content is `{ status, lastRunAt }`. Return `status: 'offline'` for any agent with no heartbeat file; developer wires their agent loop to write these.
