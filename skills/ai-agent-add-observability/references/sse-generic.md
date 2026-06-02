# SSE Live Event Feed - Generic (Node.js HTTP)

Use the same `event-bus.ts` singleton from `references/sse-nextjs.md`. This works with any Node.js HTTP server.

```typescript
import http from 'http'
import { eventBus } from './lib/event-bus'
import type { AgentEvent } from './lib/event-bus'

const server = http.createServer((req, res) => {
  if (req.url === '/api/agents/stream') {
    res.writeHead(200, {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    })

    const send = (e: AgentEvent) => res.write(`data: ${JSON.stringify(e)}\n\n`)
    const heartbeat = setInterval(() => res.write(': heartbeat\n\n'), 30_000)

    eventBus.on('event', send)

    req.on('close', () => {
      eventBus.off('event', send)
      clearInterval(heartbeat)
    })
    return
  }
  res.writeHead(404).end()
})
```

Adapt the routing logic to whatever router your project uses.
