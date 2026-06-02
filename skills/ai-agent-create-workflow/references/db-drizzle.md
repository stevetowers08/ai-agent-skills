# Scheduling Tables - Drizzle ORM

```typescript
import { pgTable, text, integer } from 'drizzle-orm/pg-core'

export const routines = pgTable('routines', {
  id:             text('id').primaryKey(),
  workflowName:   text('workflow_name').notNull(),
  cronExpression: text('cron_expression').notNull(),
  enabled:        integer('enabled').notNull().default(1),
  lastRunAt:      integer('last_run_at'),
  nextRunAt:      integer('next_run_at'),
  config:         text('config'),  // JSON blob for workflow-specific config
})

export const routineRuns = pgTable('routine_runs', {
  id:          text('id').primaryKey(),
  routineId:   text('routine_id').notNull(),
  status:      text('status').notNull(),  // running | success | error | skipped
  startedAt:   integer('started_at').notNull(),
  finishedAt:  integer('finished_at'),
  result:      text('result'),
  errorMsg:    text('error_msg'),
})
```

Apply: `npx drizzle-kit push`

## Concurrency guard (coalesce_if_active)

```typescript
import { db } from '@/lib/db'
import { routineRuns } from '@/lib/db'
import { eq, and } from 'drizzle-orm'

export async function isRoutineRunning(routineId: string): Promise<boolean> {
  const running = await db
    .select({ id: routineRuns.id })
    .from(routineRuns)
    .where(and(eq(routineRuns.routineId, routineId), eq(routineRuns.status, 'running')))
    .limit(1)
  return running.length > 0
}
```

Use at the start of the trigger endpoint: if `isRoutineRunning(routineId)` returns true, return early with `{ skipped: 'already_running' }`.
