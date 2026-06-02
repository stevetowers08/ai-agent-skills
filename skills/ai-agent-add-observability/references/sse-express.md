# SSE Live Event Feed - Express

Use the same `event-bus.ts` singleton from `references/sse-nextjs.md`. Only the route differs.

## Express SSE route

```typescript
import { Router } from 'express'
import { eventBus } from '../lib/event-bus'
import type { AgentEvent } from '../lib/event-bus'

export const agentsRouter = Router()

agentsRouter.get('/stream', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream')
  res.setHeader('Cache-Control', 'no-cache')
  res.setHeader('Connection', 'keep-alive')
  res.flushHeaders()

  const send = (e: AgentEvent) => res.write(`data: ${JSON.stringify(e)}\n\n`)
  const heartbeat = setInterval(() => res.write(': heartbeat\n\n'), 30_000)

  eventBus.on('event', send)

  req.on('close', () => {
    eventBus.off('event', send)
    clearInterval(heartbeat)
  })
})
```

Mount in your app: `app.use('/api/agents', agentsRouter)`
