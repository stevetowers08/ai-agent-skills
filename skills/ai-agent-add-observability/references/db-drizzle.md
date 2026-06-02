# Run Logging - Drizzle ORM

Check if `agent_runs` already exists in the schema. If it does, add only missing columns. If not, add to your existing schema file (usually `lib/db.ts` or `db/schema/agent_runs.ts`):

```typescript
import { pgTable, text, integer, real } from 'drizzle-orm/pg-core'

export const agentRuns = pgTable('agent_runs', {
  id:          text('id').primaryKey(),
  agentName:   text('agent_name').notNull(),
  status:      text('status').notNull(),       // queued | running | success | error
  trigger:     text('trigger').notNull().default('on_demand'),
  startedAt:   integer('started_at').notNull(), // unix ms
  finishedAt:  integer('finished_at'),
  tokensIn:    integer('tokens_in'),
  tokensOut:   integer('tokens_out'),
  costUsd:     real('cost_usd'),
  result:      text('result'),                  // short summary of what was done
  errorMsg:    text('error_msg'),
})
```

Apply with `drizzle-kit push` or `drizzle-kit generate && drizzle-kit migrate`.

## withRunLogging helper

```typescript
import { db } from '@/lib/db'
import { agentRuns } from '@/lib/db'
import { eq } from 'drizzle-orm'
import { randomUUID } from 'crypto'

export async function withRunLogging<T>(
  agentName: string,
  trigger: 'on_demand' | 'cron' | 'webhook' = 'on_demand',
  fn: (runId: string) => Promise<T>
): Promise<T> {
  const id = randomUUID()
  const startedAt = Date.now()

  await db.insert(agentRuns).values({ id, agentName, status: 'running', trigger, startedAt })

  try {
    const result = await fn(id)
    await db.update(agentRuns)
      .set({ status: 'success', finishedAt: Date.now() })
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

Usage:
```typescript
await withRunLogging('my-agent', 'cron', async (runId) => {
  // ... agent logic
  // optionally update tokens/cost mid-run:
  await db.update(agentRuns).set({ tokensIn: 1200, tokensOut: 400, costUsd: 0.0042 }).where(eq(agentRuns.id, runId))
})
```
