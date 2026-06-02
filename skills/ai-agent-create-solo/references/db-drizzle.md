# Run Trace Schema — Drizzle ORM

Check if `agent_runs` already exists in the project schema. Add only missing columns if so.

```typescript
import { pgTable, text, integer, real } from 'drizzle-orm/pg-core'

export const agentRuns = pgTable('agent_runs', {
  id:          text('id').primaryKey(),
  agentName:   text('agent_name').notNull(),
  status:      text('status').notNull(),  // queued | running | success | error | skipped
  trigger:     text('trigger').notNull().default('on_demand'),
  startedAt:   integer('started_at').notNull(),
  finishedAt:  integer('finished_at'),
  tokensIn:    integer('tokens_in'),
  tokensOut:   integer('tokens_out'),
  costUsd:     real('cost_usd'),
  actionTaken: text('action_taken'),
  errorMsg:    text('error_msg'),
})
```

Apply: `npx drizzle-kit push` (dev) or `npx drizzle-kit generate && npx drizzle-kit migrate` (prod).

## withRunLogging

```typescript
import { db } from '@/lib/db'
import { agentRuns } from '@/lib/db'
import { eq } from 'drizzle-orm'

type RunUsage = { tokensIn?: number; tokensOut?: number; costUsd?: number }

export async function withRunLogging<T>(
  agentName: string,
  fn: (runId: string) => Promise<{ result: T; usage?: RunUsage }>,
): Promise<T> {
  const id = crypto.randomUUID()
  await db.insert(agentRuns).values({ id, agentName, status: 'running', startedAt: Date.now(), trigger: 'on_demand' })
  try {
    const { result, usage } = await fn(id)
    await db.update(agentRuns)
      .set({
        status: 'success',
        finishedAt: Date.now(),
        tokensIn: usage?.tokensIn ?? null,
        tokensOut: usage?.tokensOut ?? null,
        costUsd: usage?.costUsd ?? null,
      })
      .where(eq(agentRuns.id, id))
    return result
  } catch (err) {
    await db.update(agentRuns)
      .set({ status: 'error', errorMsg: String(err), finishedAt: Date.now() })
      .where(eq(agentRuns.id, id))
    throw err
  }
}
```

Call from the agent loop:
```typescript
return withRunLogging(AGENT_NAME, async (runId) => {
  const result = await generateText({ ... })
  return {
    result: result.text,
    usage: {
      tokensIn: result.usage.promptTokens,
      tokensOut: result.usage.completionTokens,
      costUsd: calcCost(MODEL, result.usage),
    },
  }
})
```
