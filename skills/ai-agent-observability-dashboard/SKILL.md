---
name: ai-agent-observability-dashboard
description: Add an observability dashboard to any app that runs AI agents, LLM calls, or workflows. Generates three pages - live agent runs, workflow history, and LLM call inspector - all reading from the existing database with no external service required. Use whenever the user wants to see what their agents are doing, add a dashboard for agents, build an agent monitoring UI, view LLM call history, inspect token costs, or see workflow run history. Triggers on: "I want to see what my agents are doing", "add a dashboard", "agent monitoring page", "build an observability UI", "show me my LLM calls", "trace viewer", "token usage dashboard", "workflow history page", "visualize agent runs".
---

# ai-agent-observability-dashboard

Add a three-page observability dashboard to an existing app. All three pages read from your existing database - no external tracing service required.

**Prerequisite:** the app needs `agent_runs` data being written. If nothing is writing to `agent_runs` yet, run `/ai-agent-add-observability` first, then come back.

---

## Step 1 - Detect the stack

Read these files before writing any code:

1. **`package.json`** - detect framework, UI library, database client
2. **`app/layout.tsx`** or **`pages/_app.tsx`** - confirm routing style (App Router vs Pages Router)
3. **`docs/design/design.md`** - if it exists, read color and spacing tokens before choosing any values
4. **Existing schema** (`db/schema/*.ts`, `prisma/schema.prisma`, `src/db/*.ts`) - confirm `agent_runs` table shape

Then load the matching reference files:

| Detected in `package.json` | Read |
|---|---|
| `next` + `app/` directory present | `references/pages-nextjs-app.md` |
| `next` + `pages/` directory, no `app/` | `references/pages-nextjs-pages.md` |
| `express` + React in `src/` or `client/` | `references/pages-express-react.md` |
| `vite` + `react` | `references/pages-vite-react.md` |

| Detected in `package.json` | Read |
|---|---|
| `drizzle-orm` | `references/queries-drizzle.md` |
| `@prisma/client` | `references/queries-prisma.md` |
| `pg`, `postgres`, or `@supabase/supabase-js` | `references/queries-raw-sql.md` |

| Detected in `package.json` | Note |
|---|---|
| `shadcn-ui` or `@radix-ui/*` | Use shadcn `<Table>`, `<Badge>`, `<Sheet>`, `<Tabs>` |
| Tailwind only (no Radix) | Use plain Tailwind utility classes |

Confirm before proceeding - one line: "Detected Next.js App Router + Drizzle + shadcn - generating dashboard to match."

---

## Step 2 - Three pages

Build all three. They share a `<ObservabilityNav>` component and a `<TraceDrawer>` sheet.

### Page 1 - `/agents` (live run feed)

Three sections stacked vertically.

**Stats bar** - four cards in a row:
- Total runs (all time)
- Runs today
- Total cost USD (all time)
- Average run duration

**Live feed** - SSE-powered. Subscribes to `/api/agents/stream` (from `add-observability`). Each event renders as a row: agent name, event type badge, timestamp, live duration counter for in-progress runs. New events slide in from the top. Pause button halts scroll without disconnecting. Cap at 200 events in memory.

**Recent runs table** - last 50 runs from `agent_runs`, sorted by `startedAt` desc. Columns: Run ID (first 8 chars), Agent, Status badge, Started, Duration, Tokens In, Tokens Out, Cost USD. Clicking a row opens `<TraceDrawer>`.

Read `references/component-runs-table.md` for the `<RunsTable>` component.

### Page 2 - `/workflows` (workflow history)

Two sections.

**Run history table** - queries `routine_runs` joined to `routines`. Columns: Workflow, Status, Started, Duration, Steps (e.g. `4/6`), Failure reason (if any). Filter bar: workflow name dropdown + status + date range.

**Step accordion** - clicking a row expands an inline accordion: step name, type badge (`llm` / `tool` / `condition`), status, duration.

If no `routine_runs` table exists: render a placeholder - "No workflow runs found. Run `/ai-agent-create-workflow` to add a scheduled pipeline."

### Page 3 - `/llm-explorer` (LLM call browser)

Two-panel layout (desktop: sidebar list left, detail right; mobile: stacked).

**Call list** - paginated, 20 per page. Each row: model name, truncated prompt (first 80 chars), token count, cost, timestamp. Search bar filters by prompt text. Clicking selects it.

**Call detail** - three tabs:
- **Prompt / Response** - full prompt and response in scrollable `<pre>` blocks, syntax-highlighted for JSON
- **Attributes** - key-value table: model, latency_ms, stop_reason, cost_usd, tokens_in, tokens_out
- **Raw** - pretty-printed JSON of the full `agent_runs` record, copy-to-clipboard button

---

## Step 3 - Shared components

### `<TraceDrawer>`

Right-side sheet (not a modal - the table stays visible). Three tabs: Span tree, In/Out, Raw.

**Span tree tab** - hierarchical list of spans for the selected run. Each row: indent level, span name, type badge, duration bar (proportional to total run time). Expand/collapse per node. If the app has no child spans yet, show the single top-level run row.

**In/Out tab** - full input and output from the `agent_runs` record.

**Raw tab** - full JSON, copy button.

Read `references/component-trace-drawer.md`.

### `<ObservabilityNav>`

Three links: Agents, Workflows, LLM Explorer. Highlight the active route. Import into each page - do not add to root app layout unless asked.

---

## Step 4 - DB queries

Do not add new tables. Query `agent_runs` (and `routine_runs` + `routines` if the schema has them). If a table is missing, render a placeholder rather than erroring.

Read the query reference file matched in Step 1 for typed helper functions: `getRecentRuns`, `getRunById`, `getWorkflowRuns`, `getLlmCalls`.

---

## Step 5 - Summarise

Tell the user:
- Files written (pages, components, query helpers)
- Routes: `/agents`, `/workflows`, `/llm-explorer`
- The `<TraceDrawer>` is shared - one component used across all three pages
- No external service needed - reads from their existing DB
- If `agent_runs` was missing: "Run `/ai-agent-add-observability` first to start logging runs"
- If `docs/design/design.md` was missing: "No design.md found - used Tailwind defaults. Run `/design-system` to codify your visual language."

---

## References files to create

| File | Contents |
|---|---|
| `references/pages-nextjs-app.md` | File paths for App Router pages (`app/agents/page.tsx`, etc.), server vs client component split, Suspense placement |
| `references/pages-nextjs-pages.md` | File paths for Pages Router, `getServerSideProps` stubs, `useRouter` for nav highlighting |
| `references/pages-express-react.md` | Frontend files under `client/src/pages/`, Express API stubs, `useEffect`+`fetch` data loading |
| `references/pages-vite-react.md` | SPA layout under `src/pages/`, React Router v6 `<Route>` declarations |
| `references/queries-drizzle.md` | `getRecentRuns`, `getRunById`, `getWorkflowRuns`, `getLlmCalls` - Drizzle variants with TypeScript return types (`RunRecord`, `WorkflowRunRecord`) |
| `references/queries-prisma.md` | Same helpers, Prisma variants using shared singleton |
| `references/queries-raw-sql.md` | Same helpers, raw pg/Supabase variants |
| `references/component-runs-table.md` | Full `<RunsTable>` component: columns, status badge color map, click handler prop. shadcn `<Table>` and plain Tailwind variants side by side. |
| `references/component-trace-drawer.md` | Full `<TraceDrawer>` sheet: three-tab layout, span tree recursive renderer with proportional duration bar, copy-to-clipboard for Raw tab. shadcn `<Sheet>` + `<Tabs>` and plain Tailwind slide-in variants. |
