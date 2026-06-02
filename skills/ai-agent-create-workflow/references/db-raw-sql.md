# Scheduling Tables - Raw SQL

```sql
CREATE TABLE IF NOT EXISTS routines (
  id              TEXT PRIMARY KEY,
  workflow_name   TEXT NOT NULL,
  cron_expression TEXT NOT NULL,
  enabled         INTEGER NOT NULL DEFAULT 1,
  last_run_at     BIGINT,
  next_run_at     BIGINT,
  config          TEXT
);

CREATE TABLE IF NOT EXISTS routine_runs (
  id          TEXT PRIMARY KEY,
  routine_id  TEXT NOT NULL,
  status      TEXT NOT NULL,
  started_at  BIGINT NOT NULL,
  finished_at BIGINT,
  result      TEXT,
  error_msg   TEXT
);
```

## Concurrency guard (pg)

```typescript
const { rows } = await pool.query(
  `SELECT id FROM routine_runs WHERE routine_id = $1 AND status = 'running' LIMIT 1`,
  [routineId]
)
if (rows.length > 0) return { skipped: 'already_running' }
```

Adapt `$1` to `?` for MySQL/SQLite.
