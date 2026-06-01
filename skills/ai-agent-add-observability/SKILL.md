---
name: ai-agent-add-observability
description: Instrument an existing agent, agent team, or workflow with OpenTelemetry span tracing, token cost tracking, and a live SSE event feed. Writes src/lib/tracing.ts, wires BatchSpanProcessor to Langfuse or any OTLP endpoint, and adds a Next.js SSE route for a live dashboard feed. Use when the user wants to add observability to an agent, instrument an agent, see what their agent is doing, add tracing, monitor agent runs, track token costs, or set up a live activity feed. Triggers on: "add observability", "instrument my agent", "I can't see what the agent is doing", "add tracing", "track token costs", "monitor agent runs", "set up logging for my agent", "why did my agent do that".
---

# ai-agent-add-observability

Wire OpenTelemetry tracing, cost tracking, and a live SSE event feed into an existing agent. This is a one-time setup per project — run it once, then all agents in the project share the same tracing infrastructure.

## Stack

- `@opentelemetry/api` + `@opentelemetry/sdk-node` for instrumentation
- Langfuse (self-hosted Docker) as the default trace backend — easiest self-host, MIT license
- OTel `BatchSpanProcessor` as the exporter — swap backend by changing `OTLP_ENDPOINT`
- Next.js Route Handler for the SSE event feed
- No third-party SaaS required

## Step 1 — Identify what to instrument

Read the project structure. Find:
- Solo agent files (`src/agents/*/index.ts`)
- Team supervisor files (`src/agents/*/supervisor.ts`)
- Workflow files (`src/workflows/*.ts`)

Note which ones already have OTel instrumentation (check for `@opentelemetry/api` imports). Only instrument what's missing.

## Step 2 — Install dependencies

```bash
npm install @opentelemetry/api @opentelemetry/sdk-node @opentelemetry/exporter-trace-otlp-http @opentelemetry/resources @opentelemetry/semantic-conventions
```

## Step 3 — Write `src/lib/tracing.ts`

```typescript
import { NodeSDK } from '@opentelemetry/sdk-node'
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http'
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base'
import { Resource } from '@opentelemetry/resources'
import { SEMRESATTRS_SERVICE_NAME } from '@opentelemetry/semantic-conventions'

// Singleton — call initTracing() once at app startup
let sdk: NodeSDK | null = null

export function initTracing(serviceName: string): void {
  if (sdk) return

  const exporter = new OTLPTraceExporter({
    url: process.env.OTLP_ENDPOINT ?? 'http://localhost:3100/v1/traces',
    headers: {
      'Authorization': `Basic ${Buffer.from(
        `${process.env.LANGFUSE_PUBLIC_KEY}:${process.env.LANGFUSE_SECRET_KEY}`
      ).toString('base64')}`,
    },
  })

  sdk = new NodeSDK({
    resource: new Resource({ [SEMRESATTRS_SERVICE_NAME]: serviceName }),
    spanProcessors: [new BatchSpanProcessor(exporter, {
      maxQueueSize: 2048,
      maxExportBatchSize: 512,
      scheduledDelayMillis: 5000,
    })],
  })

  sdk.start()

  process.on('SIGTERM', () => sdk?.shutdown())
}

// Helper: create a root span for an agent run
export { trace, context, SpanStatusCode } from '@opentelemetry/api'
```

Add to `.env.local`:
```
OTLP_ENDPOINT=http://localhost:3100/v1/traces
LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
```

## Step 4 — Instrument each agent

For each uninstrumented agent, wrap the run function. The span type tells the trace viewer what kind of operation this is.

**For solo agents** — add to `src/agents/<name>/index.ts`:

```typescript
import { initTracing, trace, SpanStatusCode } from '@/lib/tracing'
initTracing('<name>-agent')
const tracer = trace.getTracer('<name>-agent')

// Wrap existing run() body:
export async function run(input: unknown): Promise<void> {
  return tracer.startActiveSpan('<name>.run', async (span) => {
    span.setAttributes({
      'entity.type': 'agent',
      'agent.name': '<name>',
      'gen_ai.system': 'anthropic',
    })
    try {
      // ... existing run logic ...
      // After generateText(), add:
      span.setAttributes({
        'gen_ai.usage.prompt_tokens': result.usage.promptTokens,
        'gen_ai.usage.completion_tokens': result.usage.completionTokens,
      })
      span.setStatus({ code: SpanStatusCode.OK })
    } catch (err) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: String(err) })
      throw err
    } finally {
      span.end()
    }
  })
}
```

**For workflow steps** — wrap each step function with a child span:

```typescript
async function step_<name>(state: WorkflowState): Promise<...> {
  return tracer.startActiveSpan('workflow.step.<name>', async (span) => {
    span.setAttributes({
      'entity.type': 'workflow',
      'workflow.step.name': '<name>',
      'workflow.step.index': <index>,
    })
    try {
      const result = await /* ... step logic ... */
      span.setStatus({ code: SpanStatusCode.OK })
      return result
    } catch (err) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: String(err) })
      throw err
    } finally {
      span.end()
    }
  })
}
```

## Step 5 — Write the SSE event bus

`src/lib/event-bus.ts`:

```typescript
import { EventEmitter } from 'events'

// Singleton broadcast bus — agents emit here, SSE clients listen
class EventBus extends EventEmitter {}
export const eventBus = new EventBus()
eventBus.setMaxListeners(100)

export type AgentEvent =
  | { type: 'run.created';   agentName: string; runId: string; startedAt: string }
  | { type: 'run.completed'; agentName: string; runId: string; status: string; durationMs: number }
  | { type: 'run.error';     agentName: string; runId: string; error: string }
  | { type: 'step.completed'; workflowName: string; stepName: string; stepIndex: number }

export function emitAgentEvent(event: AgentEvent): void {
  eventBus.emit('agent-event', event)
}
```

`src/app/api/agents/stream/route.ts`:

```typescript
import { eventBus } from '@/lib/event-bus'
import type { AgentEvent } from '@/lib/event-bus'

export async function GET(): Promise<Response> {
  const stream = new ReadableStream({
    start(controller) {
      const encoder = new TextEncoder()

      const send = (data: AgentEvent) => {
        controller.enqueue(encoder.encode(`data: ${JSON.stringify(data)}\n\n`))
      }

      // Heartbeat every 30s to keep connection alive
      const heartbeat = setInterval(() => {
        controller.enqueue(encoder.encode(': heartbeat\n\n'))
      }, 30_000)

      eventBus.on('agent-event', send)

      // Cleanup on client disconnect
      return () => {
        eventBus.off('agent-event', send)
        clearInterval(heartbeat)
      }
    },
  })

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  })
}
```

Then add `emitAgentEvent(...)` calls inside each agent's run function at start and completion.

## Step 6 — Add cost tracking

Add a `costUsd` calculation after each `generateText` call. Rates as of mid-2026:

```typescript
const COST_PER_TOKEN = {
  'claude-sonnet-4-6': { input: 0.000003, output: 0.000015 },
  'claude-opus-4-8':   { input: 0.000015, output: 0.000075 },
  'claude-haiku-4-5':  { input: 0.0000008, output: 0.000004 },
} as const

function calcCost(model: string, promptTokens: number, completionTokens: number): number {
  const rates = COST_PER_TOKEN[model as keyof typeof COST_PER_TOKEN]
  if (!rates) return 0
  return promptTokens * rates.input + completionTokens * rates.output
}
```

Write `costUsd` to `agent_runs` and set it as a span attribute: `span.setAttribute('gen_ai.usage.cost_usd', costUsd)`.

## Step 7 — Langfuse self-host setup (optional but recommended)

If Langfuse isn't running yet, add `docker-compose.langfuse.yml`:

```yaml
version: '3'
services:
  langfuse:
    image: langfuse/langfuse:latest
    ports:
      - "3100:3000"
    environment:
      DATABASE_URL: postgresql://postgres:postgres@db:5432/langfuse
      NEXTAUTH_SECRET: changeme
      SALT: changeme
    depends_on: [db]
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: langfuse
    volumes:
      - langfuse_db:/var/lib/postgresql/data
volumes:
  langfuse_db:
```

Start with `docker compose -f docker-compose.langfuse.yml up -d`. Dashboard at `http://localhost:3100`.

## Step 8 — Summarise

Tell the user:
- Which files were written and which agents were instrumented
- How to view traces: open Langfuse at `http://localhost:3100`
- How to consume the SSE feed: `const es = new EventSource('/api/agents/stream')`
- What env vars to set: `OTLP_ENDPOINT`, `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`
- Cost tracking is live — check the `cost_usd` column in `agent_runs`
