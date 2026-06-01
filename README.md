# AI Agent Skills for Claude Code

Global slash commands for building, operating, and improving AI agents in production apps. Works with any Next.js + Vercel AI SDK + Drizzle/Postgres project.

## Install

**All 7 skills:**
```bash
npx skills add stevetowers08/ai-agent-skills
```

**Single skill:**
```bash
npx skills add stevetowers08/ai-agent-skills --skill ai-agent-create-solo
```

## Skills

### Build

| Command | What it does |
|---|---|
| `/ai-agent-create-solo` | Scaffold a complete individual agent — SOUL.md, MEMORY.md, .learnings/, tool registry, Vercel AI SDK loop, Drizzle run-trace schema |
| `/ai-agent-create-team` | Wire a supervisor + worker team with delegation protocol, handoff, and OTel parent-child span nesting |
| `/ai-agent-create-workflow` | Build a scheduled multi-step pipeline with Zod step schemas, branching, retries, checkpoints, and cron trigger via routines table |

### Operate

| Command | What it does |
|---|---|
| `/ai-agent-add-observability` | Instrument agents with OTel BatchSpanProcessor → Langfuse, token cost tracking, and SSE live event feed |
| `/ai-agent-debug-run` | Diagnose failing agent by category (bad tool call, context overflow, prompt drift, retry storm), apply fix, log to ERRORS.md |

### Improve

| Command | What it does |
|---|---|
| `/ai-agent-compact-memory` | Compact MEMORY.md to under 800 tokens, promote recurring learnings to SOUL.md / TOOLS.md / global CLAUDE.md |
| `/ai-agent-create-evals` | Build a golden-trace eval harness — extract fixtures from run history, write typed runner, score tool sequence + output shape |

## Recommended flow

```
/ai-agent-create-solo       ← scaffold
/ai-agent-add-observability ← instrument (run once per project)
                            ← run it, collect traces
/ai-agent-debug-run         ← fix issues
/ai-agent-compact-memory    ← weekly memory hygiene
/ai-agent-create-evals      ← before any breaking change
```

## Stack

- **Runtime:** Vercel AI SDK (`ai` package) + `@ai-sdk/anthropic`
- **Database:** Drizzle ORM + Postgres
- **Observability:** OpenTelemetry + Langfuse (self-hosted)
- **Language:** TypeScript strict mode

## Agent file structure

```
agents/
  <name>/
    SOUL.md          # Identity, behavioral rules, cost policy
    MEMORY.md        # Cross-session context (max 800 tokens)
    TOOLS.md         # Tool registry and known gotchas
    .learnings/
      LEARNINGS.md   # Accumulated insights
      ERRORS.md      # Failure log
      FEATURE_REQUESTS.md

src/agents/<name>/
  index.ts           # Agent loop (3-tier prompt, tool registry, run logging)

db/schema/
  agent_runs.ts      # Run trace table (Drizzle)
  routines.ts        # Cron scheduling tables (for workflows)
```

## License

MIT
