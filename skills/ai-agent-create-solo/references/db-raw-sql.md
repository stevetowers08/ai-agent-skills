# Run Trace Schema — Raw SQL

Works with pg, mysql2, better-sqlite3, or any SQL database. Adapt the syntax (SERIAL vs AUTOINCREMENT, TEXT vs VARCHAR) as needed.

```sql
CREATE TABLE IF NOT EXISTS agent_runs (
  id           TEXT PRIMARY KEY,
  agent_name   TEXT NOT NULL,
  status       TEXT NOT NULL,
  trigger      TEXT NOT NULL DEFAULT 'on_demand',
  started_at   BIGINT NOT NULL,
  finished_at  BIGINT,
  tokens_in    INTEGER,
  tokens_out   INTEGER,
  cost_usd     REAL,
  action_taken TEXT,
  error_msg    TEXT
);
```

Run this once at app startup or in a migration file.

## withRunLogging (pg example)

```typescript
import { Pool } from 'pg'
import { randomUUID } from 'crypto'

const pool = new Pool({ connectionString: process.env.DATABASE_URL })

export async function withRunLogging<T>(agentName: string, fn: (runId: string) => Promise<T>): Promise<T> {
  const id = randomUUID()
  await pool.query(
    `INSERT INTO agent_runs (id, agent_name, status, trigger, started_at) VALUES ($1,$2,'running',$3,$4)`,
    [id, agentName, 'on_demand', Date.now()]
  )
  try {
    const result = await fn(id)
    await pool.query(`UPDATE agent_runs SET status='success', finished_at=$1 WHERE id=$2`, [Date.now(), id])
    return result
  } catch (err) {
    await pool.query(`UPDATE agent_runs SET status='error', error_msg=$1, finished_at=$2 WHERE id=$3`, [String(err), Date.now(), id])
    throw err
  }
}
```

Adapt positional params (`$1`) to `?` for MySQL/SQLite.
