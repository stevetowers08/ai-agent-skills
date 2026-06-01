# Run Trace Schema — Prisma

Add to `prisma/schema.prisma`. Check first — add only missing fields if `AgentRun` model already exists.

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

```typescript
import { PrismaClient } from '@prisma/client'
const prisma = new PrismaClient()

export async function withRunLogging<T>(agentName: string, fn: (runId: string) => Promise<T>): Promise<T> {
  const run = await prisma.agentRun.create({
    data: { agentName, status: 'running', trigger: 'on_demand', startedAt: BigInt(Date.now()) },
  })
  try {
    const result = await fn(run.id)
    await prisma.agentRun.update({ where: { id: run.id }, data: { status: 'success', finishedAt: BigInt(Date.now()) } })
    return result
  } catch (err) {
    await prisma.agentRun.update({ where: { id: run.id }, data: { status: 'error', errorMsg: String(err), finishedAt: BigInt(Date.now()) } })
    throw err
  }
}
```
