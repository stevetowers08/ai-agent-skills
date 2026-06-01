---
name: ai-agent-add-observability
description: Use when the user wants to add observability, tracing, run logging, cost tracking, or a live event feed to an AI agent. Triggers on: "I can't see what my agent is doing", "add tracing", "track token costs", "monitor agent runs", "add observability", "instrument my agent", "why did my agent do that", "set up a live dashboard feed", "agent monitoring".
---

# ai-agent-add-observability

Wire observability into an existing agent: OTel spans, persistent run logging, live SSE event feed, and token cost tracking. One-time setup per project — all agents share the same infrastructure.

## Step 1 — Detect the stack

Read these files before writing a single line of code:

1. **`package.json`** — detect ORM, DB client, and framework
2. **Existing schema files** — `drizzle.config.ts`, `prisma/schema.prisma`, `db/schema/*.ts`, `src/db/*.ts`, `lib/db.ts`
3. **Framework config** — `next.config.*`, `app.ts`, `server.ts`, `main.ts`
4. **Existing `agent_runs` table** — if it already exists, extend it; don't recreate

Then load the matching reference files:

| Detected in `package.json` | Read this file |
|---|---|
| `drizzle-orm` | `references/db-drizzle.md` |
| `@prisma/client` | `references/db-prisma.md` |
| `pg` or `postgres` (no ORM) | `references/db-raw-postgres.md` |
| `mysql2` (no ORM) | `references/db-raw-mysql.md` |
| `better-sqlite3` | `references/db-raw-sqlite.md` |
| None of the above | Ask: "What database are you using?" |

| Detected in `package.json` | Read this file |
|---|---|
| `next` | `references/sse-nextjs.md` |
| `express` | `references/sse-express.md` |
| `fastify` | `references/sse-fastify.md` |
| None | `references/sse-generic.md` |

Confirm what you found before proceeding — one line: "Detected Drizzle + Postgres on Next.js — generating code to match."

## Step 2 — Three observability layers

Build all three. They are independent — each works without the others, but together they give you complete visibility.

### Layer 1 — Persistent run log (your own DB)

This is the primary store. Every agent run is logged here regardless of any external tool. Read the DB reference file loaded in Step 1 for the schema and `withRunLogging` helper.

The `agent_runs` table captures: run ID, agent name, status, trigger source, start/end time, token counts, cost in USD, final result summary, and error message.

If `agent_runs` already exists in the project schema, check what columns are present and add only what's missing. Do not recreate a table that already exists.

### Layer 2 — OTel spans (optional, any OTLP backend)

Spans give you structured trace data you can send to any backend that accepts OTLP — Jaeger, Grafana Tempo, Zipkin, or a self-hosted Langfuse instance. The backend choice is entirely separate from the instrumentation.

Install once:
```bash
npm install @opentelemetry/api @opentelemetry/sdk-node @opentelemetry/exporter-trace-otlp-http @opentelemetry/resources @opentelemetry/semantic-conventions
```

`src/lib/tracing.ts`:
```typescript
import { NodeSDK } from '@opentelemetry/sdk-node'
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http'
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base'
import { Resource } from '@opentelemetry/resources'
import { SEMRESATTRS_SERVICE_NAME } from '@opentelemetry/semantic-conventions'

let sdk: NodeSDK | null = null

export function initTracing(serviceName: string): void {
  if (sdk || !process.env.OTLP_ENDPOINT) return
  const exporter = new OTLPTraceExporter({
    url: process.env.OTLP_ENDPOINT,
    headers: process.env.OTLP_AUTH_HEADER
      ? { Authorization: process.env.OTLP_AUTH_HEADER }
      : undefined,
  })
  sdk = new NodeSDK({
    resource: new Resource({ [SEMRESATTRS_SERVICE_NAME]: serviceName }),
    spanProcessors: [new BatchSpanProcessor(exporter)],
  })
  sdk.start()
  process.on('SIGTERM', () => sdk?.shutdown())
}

export { trace, context, SpanStatusCode } from '@opentelemetry/api'
```

Wrap each agent run in a span. Inside the span, record token usage after each LLM call:
```typescript
span.setAttributes({
  'gen_ai.usage.input_tokens': result.usage.promptTokens,
  'gen_ai.usage.output_tokens': result.usage.completionTokens,
  'gen_ai.usage.cost_usd': calcCost(model, result.usage),
})
```

`.env`:
```
OTLP_ENDPOINT=    # any OTLP/HTTP endpoint — leave blank to disable OTel silently
OTLP_AUTH_HEADER= # optional, e.g. "Basic base64(key:secret)"
```

### Layer 3 — SSE live event feed (optional)

A lightweight pub/sub bus that lets a dashboard subscribe to agent events in real time. Read the SSE reference file loaded in Step 1 for the framework-specific implementation.

The event bus emits typed events: `run.started`, `run.completed`, `run.errored`, `tool.called`, `tool.returned`. The SSE route keeps the connection alive with 30-second heartbeats.

## Step 3 — Cost tracking

Add this helper. Rates are mid-2026 — update if models change:

```typescript
const COST_PER_TOKEN: Record<string, { input: number; output: number }> = {
  'claude-sonnet-4-6':         { input: 0.000003,  output: 0.000015 },
  'claude-opus-4-8':           { input: 0.000015,  output: 0.000075 },
  'claude-haiku-4-5-20251001': { input: 0.0000008, output: 0.000004 },
}

export function calcCost(model: string, usage: { promptTokens: number; completionTokens: number }): number {
  const rates = COST_PER_TOKEN[model]
  if (!rates) return 0
  return usage.promptTokens * rates.input + usage.completionTokens * rates.output
}
```

Write `costUsd` to the `agent_runs` table and as an OTel span attribute. Both are queryable independently.

## Step 4 — Summarise

Tell the user:
- Which files were written
- How to trigger the SSE feed from a frontend: `new EventSource('/api/agents/stream')`
- `OTLP_ENDPOINT` is optional — leave it unset to disable OTel without errors
- DB columns are the primary store — the run log page works without any external tool
- If they want a trace UI, suggest Jaeger (zero-config local) or Langfuse (richer UI, self-hosted Docker)
