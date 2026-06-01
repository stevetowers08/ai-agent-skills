# SSE Live Event Feed — Next.js App Router

## Event bus singleton

`src/lib/event-bus.ts`:
```typescript
import { EventEmitter } from 'events'

class AgentEventBus extends EventEmitter {}
export const eventBus = new AgentEventBus()
eventBus.setMaxListeners(100)

export type AgentEvent =
  | { type: 'run.started';   agentName: string; runId: string }
  | { type: 'run.completed'; agentName: string; runId: string; durationMs: number; costUsd: number }
  | { type: 'run.errored';   agentName: string; runId: string; error: string }
  | { type: 'tool.called';   agentName: string; runId: string; toolName: string }
  | { type: 'tool.returned'; agentName: string; runId: string; toolName: string; durationMs: number }

export function emit(event: AgentEvent): void {
  eventBus.emit('event', event)
}
```

## Route Handler

`src/app/api/agents/stream/route.ts`:
```typescript
import { eventBus } from '@/lib/event-bus'
import type { AgentEvent } from '@/lib/event-bus'

export const dynamic = 'force-dynamic'

export async function GET(): Promise<Response> {
  const stream = new ReadableStream({
    start(controller) {
      const enc = new TextEncoder()
      const send = (e: AgentEvent) =>
        controller.enqueue(enc.encode(`data: ${JSON.stringify(e)}\n\n`))
      const heartbeat = setInterval(
        () => controller.enqueue(enc.encode(': heartbeat\n\n')),
        30_000
      )
      eventBus.on('event', send)
      return () => { eventBus.off('event', send); clearInterval(heartbeat) }
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

## Frontend subscription

```typescript
useEffect(() => {
  const es = new EventSource('/api/agents/stream')
  es.onmessage = (e) => {
    const event = JSON.parse(e.data)
    setEvents(prev => [...prev.slice(-59), event])
  }
  return () => es.close()
}, [])
```

## Emit from agents

```typescript
import { emit } from '@/lib/event-bus'

// At run start:
emit({ type: 'run.started', agentName, runId })

// At run end:
emit({ type: 'run.completed', agentName, runId, durationMs, costUsd })

// On error:
emit({ type: 'run.errored', agentName, runId, error: err.message })
```
