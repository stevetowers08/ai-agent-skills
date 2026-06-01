# AI Agent Skills for Claude Code

> Slash commands for building and operating production AI agents. Each skill is a step in a lifecycle: scaffold, instrument, improve, debug, test.

```bash
npx skills add stevetowers08/ai-agent-skills
```

---

## The idea

Building an AI agent involves the same steps every time: define its identity, wire its tools, instrument it so you can see what it's doing, give it a memory it can accumulate learnings into, and build a test harness so changes don't break it silently.

These skills encode that lifecycle as Claude Code slash commands. Each one does one job well. They're designed to compose - you run them in order, and by the end you have a production-grade agent with identity, memory, observability, a self-improvement loop, and a regression suite.

The skills detect your stack from `package.json` before generating any code - Drizzle, Prisma, raw SQL, Next.js, Express, Fastify. No hardcoded assumptions.

---

## Skills

### Build

**`/ai-agent-create-solo`** - Scaffold a complete individual agent from scratch.

Writes every file the agent needs to be runnable: a `SOUL.md` identity file (role, behavioral principles, cost policy), a `MEMORY.md` with a strict 800-token cap, a `.learnings/` directory for accumulating errors and corrections, a `TOOLS.md` registry documenting each tool and its known quirks, and the agent loop itself in TypeScript using the Vercel AI SDK's `generateText` with a 3-tier prompt (stable identity + tool docs stay cache-warm across runs; fresh memory snapshot injected per run). Also writes the run-trace schema so every execution is logged to your database with token counts and cost.

The 3-tier prompt matters: Anthropic's prompt cache preserves the prefix. Putting stable content (SOUL.md + TOOLS.md) first means the expensive context is cached across thousands of runs. Only the volatile memory tier (MEMORY.md, current task) changes each call.

```
agents/<name>/
  SOUL.md          # Who the agent is, what it will and won't do
  MEMORY.md        # Cross-run context, capped at 800 tokens
  TOOLS.md         # Tool registry with gotchas and error patterns
  .learnings/
    LEARNINGS.md   # Accumulated corrections and insights
    ERRORS.md      # Failure log with reproducibility status
    FEATURE_REQUESTS.md

src/agents/<name>/
  index.ts         # generateText loop, tool definitions, run logging

db/schema/
  agent_runs.ts    # id, agentName, status, tokensIn, tokensOut, costUsd, actionTaken, errorMsg
```

---

**`/ai-agent-create-team`** - Wire a supervisor + worker agent team.

The supervisor receives a task and breaks it into subtasks, each delegated to a specialist worker via a `delegate_task` tool. Workers are individual agents (each with their own SOUL.md and memory). OTel spans nest so the full team execution is visible as a single trace tree - you can see the supervisor call, each worker span, which tools each worker used, and how long each step took.

Asks upfront whether workers run sequentially or in parallel, and whether the supervisor receives the full worker message history or just the final result (default: last message only - avoids context bloat on the supervisor side).

```
agents/<team>/
  AGENTS.md            # Team topology: who does what, delegation rules
  supervisor/
    SOUL.md            # Supervisor identity and coordination rules
    MEMORY.md
  <worker-name>/
    SOUL.md            # Worker specialist identity
    MEMORY.md
    TOOLS.md

src/agents/<team>/
  supervisor.ts        # delegate_task tool + OTel parent span
  <worker-name>.ts     # Worker loop accepting parentSpan for trace nesting
```

---

**`/ai-agent-create-workflow`** - Build a scheduled multi-step pipeline.

Workflows are different from agents: no conversation loop, no tool autonomy. A workflow is a linear (or branching) pipeline of typed steps that runs on a cron schedule. Each step's input and output is validated with Zod - if a step produces the wrong shape, the workflow fails fast rather than passing bad data downstream.

Writes the workflow file, the `routines` + `routine_runs` scheduling tables, and a trigger endpoint. The routines table uses a `coalesce_if_active` concurrency policy by default - if the cron fires while a run is still active, it skips rather than stacking.

```
src/workflows/<name>.ts     # createWorkflowChain with typed steps, branching, retries
db/schema/routines.ts       # routines + routineRuns tables
app/api/workflows/<name>/
  trigger/route.ts          # POST endpoint for manual + cron triggering
```

---

### Operate

**`/ai-agent-add-observability`** - Instrument an existing agent with OTel tracing, cost tracking, and a live event feed.

Detects your stack from `package.json` first (Drizzle/Prisma/raw SQL, Next.js/Express/Fastify), then generates matching code. Three layers:

1. **Persistent run log** - every run written to your own database (primary store, no external dependency)
2. **OTel spans** - structured traces sent to any OTLP backend you choose (Jaeger, Grafana Tempo, Zipkin, or a self-hosted Langfuse instance via `OTLP_ENDPOINT` env var; leave it unset to disable silently)
3. **SSE live event feed** - a pub/sub bus so a dashboard can subscribe to agent events in real time without polling

Cost rates are baked in per model so the tracing layer computes `costUsd` on every span.

---

**`/ai-agent-debug-run`** - Diagnose a failing agent by category, fix it, log the root cause.

The Iron Law: identify root cause before touching any code. This skill reads run traces and error logs first, then maps the failure to one of eight categories - bad tool call, context overflow, prompt drift, retry storm, tool error (external), suspension not resumed, schema mismatch, cost abort. Each category has a specific fix path.

The most common failure modes and what they actually mean:

- **Context overflow** (`tokensIn > 80%`): the agent is reading too much on each run - compact MEMORY.md, strip stale LEARNINGS entries from the system prompt
- **Retry storm**: `maxSteps` set too high with no result-check logic - the agent calls the same tool repeatedly because it doesn't know the previous call worked
- **Prompt drift**: SOUL.md is too vague and the agent is improvising outside its role - tighten behavioral rules with specific prohibitions

Logs a structured entry to `ERRORS.md` after every debug session so the pattern is captured for promotion.

---

### Improve

**`/ai-agent-compact-memory`** - Trim MEMORY.md to under 800 tokens and promote durable learnings to permanent files.

The 800-token cap on MEMORY.md isn't arbitrary - it's the point where the Anthropic prompt cache prefix can be preserved across runs. Beyond that, the cache breaks and you pay for the full context on every call.

Compaction works in two passes: first, deduplicate and summarise volatile memory (things that are no longer current); second, scan LEARNINGS.md for entries with `Recurrence-Count >= 3` seen across two or more distinct tasks in the last 30 days - those get promoted to SOUL.md (behavioral rules), TOOLS.md (tool gotchas), or global CLAUDE.md (project-wide facts). The promoted entries are distilled into one-line prevention rules, not verbose incident descriptions.

Creates a dated snapshot at `agents/<name>/memory/YYYY-MM-DD.md` before compacting so history isn't lost.

---

**`/ai-agent-self-improving`** - Add a feedback loop so the agent gets better over time.

Two patterns in one skill - use one or both depending on whether the agent's outputs produce measurable real-world results.

**Learning Capture** (any agent): Initialises the `.learnings/` directory with structured log files. Defines when to log (tool errors, user corrections, outdated knowledge, capability gaps), the entry format (`LRN-YYYYMMDD-XXX` with Pattern-Key for deduplication), and the promotion rules (Recurrence-Count >= 3, 2+ tasks, 30-day window - promote to SOUL.md / TOOLS.md / CLAUDE.md as a distilled prevention rule).

**Performance Observer** (agents with measurable outputs): A small scheduled agent that watches an external performance signal - email reply rates, CRM conversion, engagement metrics, support resolution times, sales win rates, whatever your agent produces that you can measure. Fetches the signal data, splits into top and bottom performers, sends to Claude Haiku for pattern analysis, and saves a structured insight summary to the database. The main agent reads this summary at generation time and injects it as a context block before running.

The observer runs on a cadence matched to how fast your signal changes (weekly for content, daily for outreach, hourly for support). The insight summary is grounded in specific examples from the data - "emails under 80 words with a concrete question get 3x more replies" is grounded; "be more concise" is not. Based on the Reflexion architecture (Shinn et al., NeurIPS 2023), adapted for external performance signals rather than internal tool feedback.

---

**`/ai-agent-create-evals`** - Build a golden-trace eval harness from real run history.

Extracts 3-5 successful runs and 1-2 known failure runs from the `agent_runs` table, writes them as typed JSON fixtures, and generates a runner that replays each fixture and scores four dimensions: tool call sequence (right tools, roughly right order), output shape (matches expected Zod schema), no-hallucination (didn't fabricate when it should have stopped), and cost bounds (stayed within expected token budget).

Agent evals don't test exact output - LLMs are non-deterministic. They test behavioral patterns. A regression is when the agent starts calling the wrong tool, producing a different shape, or hallucinating results it used to stop on.

```
evals/<agent-name>/
  fixtures/
    <scenario>.json    # input + expected.toolSequence + expected.outputShape + expected.maxTokensIn
  runner.ts            # executes fixtures, scores, reports pass/fail
```

Adds `npm run eval:<name>` to package.json so evals run in CI.

---

## Recommended sequence

```
/ai-agent-create-solo           # scaffold
/ai-agent-add-observability     # instrument (once per project)
                                # run it, collect real traces
/ai-agent-debug-run             # fix issues using trace evidence
/ai-agent-self-improving        # add feedback loop
/ai-agent-compact-memory        # weekly - keep memory from bloating
/ai-agent-create-evals          # before any breaking change
```

For teams:

```
/ai-agent-create-team           # supervisor + workers
/ai-agent-add-observability     # nested trace tree across all agents
/ai-agent-create-evals          # eval at the supervisor level (input to final output)
```

For scheduled pipelines:

```
/ai-agent-create-workflow       # typed steps, cron trigger, concurrency policy
/ai-agent-add-observability     # instrument each step as a span
/ai-agent-self-improving        # performance observer if steps produce measurable outputs
```

---

## Install a single skill

```bash
npx skills add stevetowers08/ai-agent-skills --skill ai-agent-create-solo
npx skills add stevetowers08/ai-agent-skills --skill ai-agent-create-team
npx skills add stevetowers08/ai-agent-skills --skill ai-agent-create-workflow
npx skills add stevetowers08/ai-agent-skills --skill ai-agent-add-observability
npx skills add stevetowers08/ai-agent-skills --skill ai-agent-debug-run
npx skills add stevetowers08/ai-agent-skills --skill ai-agent-compact-memory
npx skills add stevetowers08/ai-agent-skills --skill ai-agent-self-improving
npx skills add stevetowers08/ai-agent-skills --skill ai-agent-create-evals
```

---

## Stack

- **Runtime:** Vercel AI SDK (`ai` package) + `@ai-sdk/anthropic`
- **Database:** detected from your project - Drizzle, Prisma, or raw SQL (Postgres, MySQL, SQLite)
- **Observability:** OpenTelemetry + any OTLP backend (Jaeger, Grafana Tempo, Langfuse, Zipkin)
- **Language:** TypeScript strict mode
- **Framework:** detected from your project - Next.js, Express, or Fastify

The identity/memory patterns (SOUL.md, MEMORY.md, .learnings/) work on any stack.

---

## License

MIT
