# Run Trace Schema - Prisma

Add to `prisma/schema.prisma`. Check first - add only missing fields if `AgentRun` model already exists.

```prisma
model AgentRun {
  id          String  @id @default(cuid())
  agentName   String
  status      String  // queued | running | success | error | skipped
  trigger     String  @default("on_demand")
  startedAt   BigInt
  finishedAt  BigInt?
  tokensIn    Int?
  tokensOut   Int?
  costUsd     Float?
  actionTaken String?
  errorMsg    String?

  @@map("agent_runs")
}
```

Run `npx prisma migrate dev --name add_agent_runs` or `npx prisma db push`.

## withRunLogging

Import a shared singleton - do NOT instantiate `new PrismaClient()` inline; it creates a
new connection pool on every module import, exhausting connections in serverless environments.

```typescript
// Use your project's existing singleton (typically lib/prisma.ts or lib/db.ts)
import { prisma } from '@/lib/prisma'

export async function withRunLogging<T>(
  agentName: string,
  fn: (runId: string) => Promise<T>,
  usage?: { tokensIn?: number; tokensOut?: number; costUsd?: number }
): Promise<T> {
  const run = await prisma.agentRun.create({
    data: { agentName, status: 'running', trigger: 'on_demand', startedAt: BigInt(Date.now()) },
  })
  try {
    const result = await fn(run.id)
    await prisma.agentRun.update({
      where: { id: run.id },
      data: {
        status: 'success',
        finishedAt: BigInt(Date.now()),
        tokensIn: usage?.tokensIn ?? null,
        tokensOut: usage?.tokensOut ?? null,
        costUsd: usage?.costUsd ?? null,
      },
    })
    return result
  } catch (err) {
    await prisma.agentRun.update({
      where: { id: run.id },
      data: { status: 'error', errorMsg: String(err), finishedAt: BigInt(Date.now()) },
    })
    throw err
  }
}
```

Call from the agent loop after `generateText` resolves:
```typescript
return withRunLogging(AGENT_NAME, async (runId) => {
  const result = await generateText({ ... })
  return result.text
}, {
  tokensIn: result.usage.promptTokens,
  tokensOut: result.usage.completionTokens,
  costUsd: calcCost(MODEL, result.usage),
})
```
