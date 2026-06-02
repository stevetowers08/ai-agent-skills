---
name: ai-agent-compact-memory
description: Update, deduplicate, and compact an agent's memory files - MEMORY.md, daily snapshots, LEARNINGS.md - and promote durable learnings to SOUL.md, TOOLS.md, or global CLAUDE.md based on recurrence and scope. Use when an agent's memory is getting too long, after a series of runs, to clean up stale memory, to promote a learning to a permanent rule, or to prep memory for a new phase of work. Triggers on: "compact agent memory", "agent memory is too long", "update agent memory", "save what the agent learned", "promote this to a rule", "clean up agent memory", "memory is full".
---

# ai-agent-compact-memory

Update and compact an agent's memory files. The goal is to keep MEMORY.md under 800 tokens (the prompt cache preservation limit) while promoting durable learnings to the right permanent home.

## The memory hierarchy

```
agents/<name>/MEMORY.md          ← volatile, max 800 tokens, injected every run
agents/<name>/.learnings/
  LEARNINGS.md                   ← accumulating insights, no cap
  ERRORS.md                      ← failure log, no cap
  FEATURE_REQUESTS.md            ← capability gaps noted, no cap
agents/<name>/SOUL.md            ← permanent behavioral rules
agents/<name>/TOOLS.md           ← permanent tool gotchas
~/.claude/CLAUDE.md              ← global rules (cross-project)
```

## Step 1 - Identify the agent

Ask which agent to compact if not clear from context. Read:
- `agents/<name>/MEMORY.md`
- `agents/<name>/.learnings/LEARNINGS.md` (last 30 entries)
- `agents/<name>/.learnings/ERRORS.md` (last 20 entries)
- `agents/<name>/SOUL.md`
- `agents/<name>/TOOLS.md`

## Step 2 - Compact MEMORY.md

MEMORY.md must stay under 800 tokens (~2,200 characters). If it's over:

1. Remove any entry older than 14 days that hasn't been referenced in a recent run
2. Collapse repeated context into a single summary line
3. Keep structure: **Current context**, **Open decisions**, **Known constraints**
4. Move anything that looks like a durable rule (not context-specific) to Step 3

Write the compacted version back to `agents/<name>/MEMORY.md`.

## Step 3 - Promote learnings to permanent homes

Read through `.learnings/LEARNINGS.md`. For each entry, apply the promotion rule:

**Promote to `SOUL.md`** when all three are true:
- Recurrence-Count >= 3 (Pattern-Key appears in 3+ entries)
- Seen across >= 2 different tasks or runs
- Within a 30-day window

Promoted format in SOUL.md:
```markdown
## Behavioral rules (from learnings)
- <rule derived from learning> <!-- promoted YYYY-MM-DD from Pattern-Key: <key> -->
```

**Promote to `TOOLS.md`** when:
- Category is `tool-gotcha` or `api-behavior`
- Seen in 2+ runs

Promoted format in TOOLS.md:
```markdown
## <tool_name> - Known behavior
- <gotcha> <!-- promoted YYYY-MM-DD -->
```

**Promote to `~/.claude/CLAUDE.md`** when:
- Applies across multiple projects, not just this agent
- Category is `workflow` or `best-practice`
- User confirms before writing

Mark promoted entries in LEARNINGS.md:
```markdown
Status: promoted → SOUL.md <!-- or TOOLS.md or CLAUDE.md -->
Promoted: <date>
```

Do not delete promoted entries - they serve as the audit trail.

## Step 4 - Create the next daily snapshot

Write `agents/<name>/memory/YYYY-MM-DD.md` (today's date) seeded with the compacted MEMORY.md state:

```markdown
# <Name> Agent - Memory Snapshot <date>

## Carried from previous session
<!-- paste compacted MEMORY.md content -->

## New this session
<!-- leave blank for the agent to fill during its next run -->
```

## Step 5 - Summarise

Tell the user:
- How many tokens MEMORY.md was reduced to
- Which learnings were promoted and to which file
- Whether any FEATURE_REQUESTS entries should be acted on now (list them if > 2 exist)
- Whether ERRORS.md has any unresolved patterns with Recurrence-Count >= 3 (flag for `/ai-agent-debug-run`)
