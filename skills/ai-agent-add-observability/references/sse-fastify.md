# SSE Live Event Feed — Fastify

Use the same `event-bus.ts` singleton from `references/sse-nextjs.md`. Only the route differs.

```typescript
import type { FastifyPluginAsync } from 'fastify'
import { eventBus } from '../lib/event-bus'
import type { AgentEvent } from '../lib/event-bus'

export const agentRoutes: FastifyPluginAsync = async (fastify) => {
  fastify.get('/api/agents/stream', (request, reply) => {
    reply.raw.setHeader('Content-Type', 'text/event-stream')
    reply.raw.setHeader('Cache-Control', 'no-cache')
    reply.raw.setHeader('Connection', 'keep-alive')
    reply.raw.flushHeaders()

    const send = (e: AgentEvent) => reply.raw.write(`data: ${JSON.stringify(e)}\n\n`)
    const heartbeat = setInterval(() => reply.raw.write(': heartbeat\n\n'), 30_000)

    eventBus.on('event', send)

    request.raw.on('close', () => {
      eventBus.off('event', send)
      clearInterval(heartbeat)
    })

    // Keep the reply open
    return new Promise(() => {})
  })
}
```
