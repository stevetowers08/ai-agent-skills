# Scheduling Tables — Prisma

```prisma
model Routine {
  id             String       @id @default(cuid())
  workflowName   String
  cronExpression String
  enabled        Boolean      @default(true)
  lastRunAt      BigInt?
  nextRunAt      BigInt?
  config         String?
  runs           RoutineRun[]

  @@map("routines")
}

model RoutineRun {
  id         String   @id @default(cuid())
  routineId  String
  status     String   // running | success | error | skipped
  startedAt  BigInt
  finishedAt BigInt?
  result     String?
  errorMsg   String?
  routine    Routine  @relation(fields: [routineId], references: [id])

  @@map("routine_runs")
}
```

Run `npx prisma migrate dev --name add_routines`.

## Concurrency guard

```typescript
const running = await prisma.routineRun.findFirst({
  where: { routineId, status: 'running' },
})
if (running) return { skipped: 'already_running' }
```
