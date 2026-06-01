# Run Logging — Raw MySQL (mysql2)

```sql
CREATE TABLE IF NOT EXISTS agent_runs (
  id          VARCHAR(36) PRIMARY KEY,
  agent_name  VARCHAR(255) NOT NULL,
  status      VARCHAR(50) NOT NULL,
  trigger     VARCHAR(50) NOT NULL DEFAULT 'on_demand',
  started_at  BIGINT NOT NULL,
  finished_at BIGINT,
  tokens_in   INT,
  tokens_out  INT,
  cost_usd    FLOAT,
  result      TEXT,
  error_msg   TEXT
);
```

## withRunLogging helper (mysql2/promise)

```typescript
import mysql from 'mysql2/promise'
import { randomUUID } from 'crypto'

const pool = mysql.createPool(process.env.DATABASE_URL!)

export async function withRunLogging<T>(
  agentName: string,
  trigger: 'on_demand' | 'cron' | 'webhook' = 'on_demand',
  fn: (runId: string) => Promise<T>
): Promise<T> {
  const id = randomUUID()
  await pool.execute(
    `INSERT INTO agent_runs (id, agent_name, status, \`trigger\`, started_at) VALUES (?, ?, 'running', ?, ?)`,
    [id, agentName, trigger, Date.now()]
  )
  try {
    const result = await fn(id)
    await pool.execute(`UPDATE agent_runs SET status = 'success', finished_at = ? WHERE id = ?`, [Date.now(), id])
    return result
  } catch (err) {
    await pool.execute(`UPDATE agent_runs SET status = 'error', error_msg = ?, finished_at = ? WHERE id = ?`, [String(err), Date.now(), id])
    throw err
  }
}
```
