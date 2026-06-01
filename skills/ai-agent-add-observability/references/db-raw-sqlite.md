# Run Logging — SQLite (better-sqlite3)

```typescript
import Database from 'better-sqlite3'
import { randomUUID } from 'crypto'

const db = new Database(process.env.DB_PATH ?? 'agent.db')

db.exec(`
  CREATE TABLE IF NOT EXISTS agent_runs (
    id          TEXT PRIMARY KEY,
    agent_name  TEXT NOT NULL,
    status      TEXT NOT NULL,
    trigger     TEXT NOT NULL DEFAULT 'on_demand',
    started_at  INTEGER NOT NULL,
    finished_at INTEGER,
    tokens_in   INTEGER,
    tokens_out  INTEGER,
    cost_usd    REAL,
    result      TEXT,
    error_msg   TEXT
  )
`)

export function withRunLogging<T>(
  agentName: string,
  trigger: 'on_demand' | 'cron' | 'webhook' = 'on_demand',
  fn: (runId: string) => T
): T {
  const id = randomUUID()
  db.prepare(`INSERT INTO agent_runs (id, agent_name, status, trigger, started_at) VALUES (?, ?, 'running', ?, ?)`)
    .run(id, agentName, trigger, Date.now())
  try {
    const result = fn(id)
    db.prepare(`UPDATE agent_runs SET status = 'success', finished_at = ? WHERE id = ?`).run(Date.now(), id)
    return result
  } catch (err) {
    db.prepare(`UPDATE agent_runs SET status = 'error', error_msg = ?, finished_at = ? WHERE id = ?`)
      .run(String(err), Date.now(), id)
    throw err
  }
}
```

Note: better-sqlite3 is synchronous. If the agent uses async tools, use the async wrapper pattern with `Promise.resolve(fn(id)).then(...).catch(...)` instead.
