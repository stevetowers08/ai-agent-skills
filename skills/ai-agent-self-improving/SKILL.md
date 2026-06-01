---
name: ai-agent-self-improving
description: Add self-improvement to any AI agent — learning capture, promotion to memory, and a performance observer that watches external signals and injects insights back into the agent. Use when the user wants an agent that gets better over time, wants to wire up a feedback loop, wants to track what the agent learned, wants a performance observer, or says anything like "make the agent learn", "agent that improves", "feedback loop for my agent", "track agent performance", "agent that adapts based on results". Works for any domain — sales, content, support, outreach, analysis.
---

# ai-agent-self-improving

Two mechanisms, one skill. Use one or both depending on what the agent does:

| Pattern | When to use |
|---|---|
| **Learning Capture** | Any agent — logs corrections, errors, discoveries, and promotes recurring patterns to permanent memory |
| **Performance Observer** | Agents whose outputs produce measurable real-world results — watches an external signal, generates insights, injects them at generation time |

Ask the user: "Does this agent produce outputs you can measure externally (e.g. emails that get replies, posts that get engagement, calls that convert, recommendations that get accepted)?" If yes, add both. If no, just Learning Capture.

---

## Pattern 1 — Learning Capture

### Step 1 — Initialise `.learnings/`

Check if `.learnings/` exists in the agent's directory. If not, create it:

```bash
mkdir -p agents/<name>/.learnings
```

Create three files if missing (never overwrite existing content):

**`agents/<name>/.learnings/LEARNINGS.md`**
```markdown
# Learnings

Corrections, insights, knowledge gaps, and best practices captured during runs.

**Categories:** correction | insight | knowledge_gap | best_practice
**Statuses:** pending | in_progress | resolved | wont_fix | promoted | promoted_to_skill

---
```

**`agents/<name>/.learnings/ERRORS.md`**
```markdown
# Errors

Tool failures, unexpected outputs, integration issues.

---
```

**`agents/<name>/.learnings/FEATURE_REQUESTS.md`**
```markdown
# Feature Requests

Capabilities the agent tried to use but doesn't have yet.

---
```

### Step 2 — Entry format

**Learning entry** (`LEARNINGS.md`):
```markdown
## [LRN-YYYYMMDD-XXX] category

**Logged:** ISO timestamp
**Priority:** low | medium | high | critical
**Status:** pending
**Area:** tool | prompt | data | integration | output-quality

### Summary
One line: what was learned.

### Details
What happened, what was wrong, what's correct.

### Suggested Action
Specific change to make.

### Metadata
- Pattern-Key: unique-slug (for deduplication)
- Recurrence-Count: 1
- First-Seen: YYYY-MM-DD
- Last-Seen: YYYY-MM-DD
- Related Files: path/to/file

---
```

**Error entry** (`ERRORS.md`):
```markdown
## [ERR-YYYYMMDD-XXX] tool-or-operation-name

**Logged:** ISO timestamp
**Priority:** high
**Status:** pending
**Reproducible:** yes | no | unknown

### Summary
What failed.

### Error
```
Paste actual error message here
```

### Context
What was attempted, what inputs were used.

### Suggested Fix
What might resolve it.

---
```

**Feature request** (`FEATURE_REQUESTS.md`):
```markdown
## [FEAT-YYYYMMDD-XXX] capability-name

**Logged:** ISO timestamp
**Priority:** medium
**Complexity:** simple | medium | complex

### Requested Capability
What the agent tried to do but couldn't.

### Why
What problem it was trying to solve.

### Suggested Implementation
How it could be added.

---
```

### Step 3 — When to log

Log automatically when:

| Situation | File | Category |
|---|---|---|
| Tool returns error or non-zero exit | ERRORS.md | - |
| Agent produced wrong output, user corrects | LEARNINGS.md | `correction` |
| Agent's knowledge was outdated or wrong | LEARNINGS.md | `knowledge_gap` |
| Better approach was found | LEARNINGS.md | `best_practice` |
| Agent tried a capability it doesn't have | FEATURE_REQUESTS.md | - |
| External API behaved unexpectedly | ERRORS.md | - |
| Output quality was consistently low in one area | LEARNINGS.md | `insight` |

**Deduplication:** Before creating a new entry, search for an existing `Pattern-Key` match. If found, increment `Recurrence-Count` and update `Last-Seen`. Do not create a duplicate.

### Step 4 — Promotion rules

Promote a learning to permanent agent memory when **all three** are true:
- `Recurrence-Count >= 3`
- Seen across at least 2 distinct tasks or runs
- Occurred within a 30-day window

**Where to promote:**

| Learning type | Promote to |
|---|---|
| Behavioral rule or principle | `agents/<name>/SOUL.md` |
| Tool gotcha, rate limit, API quirk | `agents/<name>/TOOLS.md` |
| Project-wide fact or convention | `CLAUDE.md` |
| Workflow or automation pattern | `AGENTS.md` |

**How to promote:**
1. Distill the learning into one short prevention-focused rule (not an incident write-up)
2. Add to the appropriate section in the target file
3. Update the original entry: `Status: promoted`, add `Promoted: SOUL.md` (or target file)

**Example — before promotion (LEARNINGS.md):**
```markdown
## [LRN-20260601-003] best_practice
Pattern-Key: crm-reply-delay
Recurrence-Count: 4
Status: pending
### Summary
Waiting 24h before following up a cold email doubles reply rate vs same-day follow-up.
```

**After promotion (SOUL.md addition):**
```markdown
## Timing rules
- Never follow up a cold email within 24 hours — reply rate doubles with a 24h wait minimum.
```

---

## Pattern 2 — Performance Observer

A small scheduled agent that watches an external performance signal, generates a structured insight summary, and injects it into the main agent at generation time. Not tied to any specific domain.

### Architecture

```
[External Signal]         [Observer]              [Main Agent]
CRM win rates       →     scheduled job     →     reads insights
Email open rates          pulls metrics           at run time
Sales conversions         Claude analysis         injects as
Support resolutions       → insight summary       context block
Engagement metrics        → stored in DB
```

The observer is a **separate scheduled process** - not inline with the main agent. It runs on a cadence matched to how fast your signal changes (weekly for content, daily for sales, hourly for support queues).

### Step 1 — Define the signal

Ask the user:
1. **What is the external signal?** (e.g. email reply rate, CRM stage conversion, post engagement, support CSAT, call-to-meeting rate)
2. **Where does the data live?** (database table, API, CSV, webhook)
3. **What's the right cadence?** (how long before patterns become visible — days for content, hours for sales/support)
4. **What volume?** (how many data points are needed for a pattern to be meaningful — suggest minimum 5-10 for a first run)

### Step 2 — Build the observer

Create `src/agents/<name>/observer.ts` (or `observer.py`):

```typescript
import { createAnthropicClient } from '@/lib/anthropic'

// 1. Pull performance data from your signal source
async function fetchSignalData(): Promise<PerformanceRecord[]> {
  // Query your DB, call your API, read your CSV
  // Return records with: id, output (what the agent produced), result (the measured outcome), createdAt
  // Example: { id, emailSubject, body, replyReceived, replyRate, sentAt }
}

// 2. Score records — define what "good" looks like for your domain
function score(record: PerformanceRecord): number {
  // Return a single numeric score per record
  // Sales: conversion score (0-1), Content: engagement score, Support: resolution time (inverted)
  // Example for email outreach: record.replied ? 1 : 0
}

// 3. Generate insights using Claude
export async function generateInsights(): Promise<string> {
  const records = await fetchSignalData()
  if (records.length < 5) return '' // Not enough data yet

  const scored = records
    .map(r => ({ ...r, score: score(r) }))
    .sort((a, b) => b.score - a.score)

  const top = scored.slice(0, Math.ceil(scored.length / 2))
  const bottom = scored.slice(Math.ceil(scored.length / 2))

  const client = createAnthropicClient()
  const response = await client.messages.create({
    model: 'claude-haiku-4-5-20251001', // cheap — runs on a schedule
    max_tokens: 600,
    messages: [{
      role: 'user',
      content: `Analyze performance data for an AI agent. Identify what's working vs not.

TOP PERFORMING outputs (score: ${top[0]?.score.toFixed(2)} avg):
${top.map(r => formatRecord(r)).join('\n\n')}

LOWER PERFORMING outputs (score: ${bottom[0]?.score.toFixed(2)} avg):
${bottom.map(r => formatRecord(r)).join('\n\n')}

Write a concise insight summary in this format:

## Performance Insights

### What's working
- [2-3 specific patterns from top performers — be concrete, not generic]

### What's not working  
- [2-3 specific patterns from lower performers]

### Recommended approach for next outputs
- [2-3 concrete adjustments based on the data]

### Key metric
- Best score: [X] | Average: [X] | Worst: [X] | Sample size: [N]

Base observations on actual patterns in the data, not general advice.`,
    }],
  })

  return response.content[0].type === 'text' ? response.content[0].text : ''
}

function formatRecord(r: PerformanceRecord & { score: number }): string {
  // Format one record for Claude — include the key fields that distinguish outputs
  // Show enough context for Claude to spot the pattern
  return `Score ${r.score.toFixed(2)}: ${r.output.slice(0, 150)}`
}
```

### Step 3 — Store and retrieve insights

Store insights in your persistent layer (DB or file). Two options:

**Option A — DB (recommended for production):**
```typescript
// Upsert to agentMemories or equivalent key-value store
await db.insert(agentMemories).values({
  scope: 'agent',
  scopeId: '<agent-name>-insights',
  kind: 'performance-insights',
  content: insights,
  updatedAt: now,
})
// On conflict: update content + updatedAt
```

**Option B — File (simpler, works locally):**
```
agents/<name>/PERFORMANCE_INSIGHTS.md
```
Write on each observer run, overwrite previous.

**Retrieval at generation time:**
```typescript
async function getInsights(): Promise<string | null> {
  // Read from DB or file — return null if not generated yet
  // Fail silently: if insights aren't available, proceed without them
}
```

### Step 4 — Inject at generation time

In the main agent's generation function, prepend insights before the user prompt:

```typescript
const insights = await getInsights().catch(() => null)

const systemPrompt = [
  agentSystemPrompt,
  insights ? `## Current Performance Insights\n${insights}` : '',
].filter(Boolean).join('\n\n')
```

Keep injection lightweight - the insights summary should be under 400 words. Claude handles the rest.

**Grounding rule (from Reflexion pattern):** The insights should cite specific examples from the data, not just generic advice. "Emails under 100 words with a specific question at the end get 3x more replies than longer emails" is grounded. "Keep emails concise" is not.

### Step 5 — Wire the schedule

Add the observer as a cron job:

```typescript
// Route: /api/cron/<agent-name>-observer
export async function GET(request: NextRequest) {
  // Auth check (CRON_SECRET)
  const insights = await generateInsights()
  if (insights) await saveInsights(insights)
  return NextResponse.json({ success: true })
}
```

Suggested cadence:
- **Content agents** (posts, articles): weekly - patterns need time to accumulate
- **Sales/outreach agents** (emails, calls): daily - feedback arrives within 24-48h
- **Support agents** (tickets, chat): every 4-6 hours - resolution data is immediate
- **Recommendation agents**: weekly - enough volume needed

### Step 6 — Verify the loop is working

After the first few observer runs, check:

1. Are the insights grounded? ("emails with X pattern" not "be more concise")
2. Is the main agent's output visibly changing? Compare outputs before/after injection
3. Is the score distribution shifting? Top performers should score higher over time

If insights are too generic, reduce the data window (fewer records, more recent) so Claude sees clearer patterns. If there's not enough data, increase the cadence minimum (e.g. don't run until N >= 10).

---

## Combining both patterns

For agents with measurable outputs, run both:

1. **Learning Capture** handles corrections, tool errors, prompt drift — things that happen during runs
2. **Performance Observer** handles outcome quality — whether the agent's outputs are actually working in the real world

They feed different parts of the agent:
- Learnings → `SOUL.md` / `TOOLS.md` (permanent identity and tool knowledge)
- Observer insights → runtime context block (injected fresh each run from latest data)

---

## Next skills

After setting this up:
- `/ai-agent-compact-memory` — keep MEMORY.md and LEARNINGS.md from bloating
- `/ai-agent-create-evals` — build a golden-trace harness to catch regressions
- `/ai-agent-debug-run` — diagnose when a run goes wrong
