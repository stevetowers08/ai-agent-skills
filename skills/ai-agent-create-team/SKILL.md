---
name: ai-agent-create-team
description: Scaffold a supervisor + worker agent team with delegation protocol, handoff wiring, shared trace context, and OTel parent-child spans. Use when the user wants to build a multi-agent system, create a team of agents, add a supervisor agent, set up agent orchestration, or wire one agent to delegate to another. Triggers on: "create an agent team", "build a supervisor agent", "set up multi-agent", "agent that delegates to other agents", "orchestrate multiple agents", "supervisor and worker agents".
---

# ai-agent-create-team

Scaffold a supervisor-worker agent team. The supervisor receives a task and delegates subtasks to specialist workers. Each agent gets its own identity and memory. Spans nest so the full team trace is visible in one tree.

## Stack assumptions

Same as `ai-agent-create-solo`. Workers are individual agents; the supervisor is an agent with a `delegate_task` tool that invokes them.

## Step 1 — Gather inputs

Ask all at once:

1. **Team name** — snake_case (e.g. `outreach_team`)
2. **Supervisor role** — what does the supervisor decide and coordinate?
3. **Workers** — list each worker by name and single-sentence role (e.g. `researcher: finds company info`, `writer: drafts the email`)
4. **Delegation mode** — sequential (one worker at a time) or parallel (workers run concurrently where possible)?
5. **Output mode** — does the supervisor get each worker's full message history (`full_history`) or just the final result (`last_message`)? Default: `last_message`.
6. **Model** — one model for all, or different models per agent?

## Step 2 — Write identity files per agent

### `agents/<team>/AGENTS.md` (team topology)

```markdown
# <Team Name> — Agent Topology

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

## Step 3 — Write the supervisor

`src/agents/<team>/supervisor.ts`:

```typescript
import { generateText, tool } from 'ai'
import { anthropic } from '@ai-sdk/anthropic'
import { z } from 'zod'
import { trace, context, SpanStatusCode } from '@opentelemetry/api'
import { run as runWorker1 } from './<worker_1>'
import { run as runWorker2 } from './<worker_2>'
import { logRun } from '@/lib/agent-runs'

const AGENT_NAME = '<team>_supervisor'
const tracer = trace.getTracer(AGENT_NAME)

// Worker registry — add entries for each worker
const WORKERS: Record<string, (task: string, parentSpan?: Span) => Promise<string>> = {
  '<worker_1>': (task, span) => runWorker1({ task }, span),
  '<worker_2>': (task, span) => runWorker2({ task }, span),
}

export async function run(input: { task: string }): Promise<string> {
  return tracer.startActiveSpan(`${AGENT_NAME}.run`, async (rootSpan) => {
    rootSpan.setAttributes({
      'agent.name': AGENT_NAME,
      'entity.type': 'agent',
      'conversation.id': input.task.slice(0, 40),
    })

    const runId = await logRun.start(AGENT_NAME)

    try {
      const result = await generateText({
        model: anthropic('<model>'),
        system: loadSoul(AGENT_NAME),
        prompt: input.task,
        tools: {
          delegate_task: tool({
            description: 'Delegate a subtask to a specialist worker agent. Choose the worker whose role matches the subtask.',
            parameters: z.object({
              worker: z.enum(['<worker_1>', '<worker_2>']).describe('Which worker to delegate to'),
              task: z.string().describe('The specific subtask for this worker, with full context'),
            }),
            execute: async ({ worker, task }) => {
              // Child span nests under supervisor root span
              return tracer.startActiveSpan(`delegate.${worker}`, async (childSpan) => {
                childSpan.setAttributes({ 'agent.name': worker, 'entity.type': 'agent' })
                try {
                  const workerFn = WORKERS[worker]
                  if (!workerFn) throw new Error(`Unknown worker: ${worker}`)
                  const result = await workerFn(task, childSpan)
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

## Step 4 — Write each worker

`src/agents/<team>/<worker_name>.ts` — use the same structure as `ai-agent-create-solo` but:
- The worker's `run()` function accepts `(input: { task: string }, parentSpan?: Span)` 
- It creates a child span under `parentSpan` if provided, so traces nest correctly
- Workers do not write to `agent_runs` directly — the supervisor owns the run record

## Step 5 — Write `AGENTS.md` escalation rules

After writing code, add to `agents/<team>/AGENTS.md`:

```markdown
## Known handoff issues
<!-- Fill in after first real run -->

## Conversation ID convention
All agents in this team share the same conversationId derived from the supervisor's runId.
This groups all worker spans under one trace in the observability dashboard.
```

## Step 6 — Summarise

Print the file tree. Tell the user:

- How to invoke: `import { run } from '@/agents/<team>/supervisor'; await run({ task: '...' })`
- That worker spans will nest under the supervisor in any OTel-compatible trace viewer
- Next steps: `/ai-agent-add-observability` to configure the trace exporter, `/ai-agent-create-evals` to test delegation routing
