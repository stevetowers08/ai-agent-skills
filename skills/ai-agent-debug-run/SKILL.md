---
name: ai-agent-debug-run
description: Diagnose and fix a failing or misbehaving AI agent by reading run traces, identifying the failure category, applying a targeted fix, and logging the root cause to the agent's ERRORS.md. Use when an agent is failing, stuck, behaving unexpectedly, producing wrong output, hitting errors repeatedly, or when the user wants to understand why an agent did something. Triggers on: "my agent is failing", "agent got stuck", "agent keeps erroring", "why did the agent do X", "agent is producing wrong results", "debug my agent", "agent isn't working", "agent ran out of tokens", "agent called the wrong tool".
---

# ai-agent-debug-run

Diagnose why an agent failed or misbehaved. This skill reads traces and logs before touching any code - the Iron Law of agent debugging is: identify root cause first, fix second. No guessing.

## Failure categories

The fix depends entirely on which failure category applies. Read the evidence before deciding.

| Category | Symptoms | Fix |
|---|---|---|
| Bad tool call | Tool called with wrong args, or wrong tool chosen | Fix tool description or Zod schema |
| Context overflow | `tokensIn` > 80% of model context limit | Prune memory, reduce system prompt |
| Prompt drift | Agent doing something different than intended | Edit SOUL.md, tighten behavioral rules |
| Retry storm | Same tool called 3+ times with same args | Add result-check logic, cap maxSteps |
| Tool error (external) | API/network error from a tool | Add retry + backoff, update TOOLS.md |
| Suspension not resumed | Agent suspended for approval, never resumed | Check approval queue, wire resume logic |
| Schema mismatch | Tool input doesn't match Zod schema | Fix schema or calling code |
| Cost abort | Run hit token budget and stopped | Increase budget or split into smaller runs |

## Step 1 - Read the evidence

Do all of these before forming a hypothesis:

1. Read `agents/<name>/.learnings/ERRORS.md` - has this happened before?
2. Read the last 5 entries in `activity-log.json` (or `agent_runs` table) for this agent - look at `status`, `errorMsg`, `tokensIn`, `tokensOut`, `actionTaken`
3. If OTel tracing is set up, look at the span tree for the failing run - which span failed and what was its `status`?
4. Read `agents/<name>/SOUL.md` - what is the agent supposed to do?
5. Read `agents/<name>/TOOLS.md` - what are the declared tools and their known issues?

## Step 2 - Identify the failure category

Match the evidence to one of the categories in the table above. State the category explicitly before proposing any fix. If the evidence is ambiguous, say so - do not guess.

**Context overflow check:**
```typescript
// Model context limits (tokens)
const CONTEXT_LIMITS = {
  'claude-sonnet-4-6': 200_000,
  'claude-opus-4-8':   200_000,
  'claude-haiku-4-5':  200_000,
}
// Flag if tokensIn > 80% of limit
```

**Retry storm check:** Look for repeated `tool_name` entries in the span tree or `actionTaken` field with near-identical arguments.

**Prompt drift check:** Compare the agent's `SOUL.md` behavioral rules to `actionTaken` in recent run logs. If the action doesn't match the stated role, the prompt is drifting.

## Step 3 - Apply the targeted fix

Apply exactly one fix per identified root cause. Do not refactor surrounding code.

**Bad tool call - fix tool description:**
Edit the tool's `description` in the agent's tool registry. Make it unambiguous: say what the tool does AND what it should NOT be used for.

**Bad tool call - fix Zod schema:**
Edit the schema to match what the LLM should provide. Add `.describe()` to every field.

**Context overflow - prune memory:**
Read `agents/<name>/MEMORY.md`. Remove stale entries. If > 800 tokens, compact: keep only current context, open decisions, and known constraints.

**Prompt drift - tighten SOUL.md:**
Add a specific behavioral rule that addresses the observed drift. Example: "Do not send any message to external services unless the task explicitly requests it."

**Retry storm - cap steps or add result check:**
```typescript
// In the agent loop, after tool call:
const seenArgs = new Set<string>()
// In execute():
const argKey = JSON.stringify(args)
if (seenArgs.has(argKey)) throw new Error(`Retry storm detected: same args called twice for ${toolName}`)
seenArgs.add(argKey)
```
Or reduce `maxSteps`.

**Tool error (external) - add retry:**
Wrap the tool's `execute()` with exponential backoff (see `ai-agent-create-workflow` for the `withRetry` helper).

## Step 4 - Log the finding

Append to `agents/<name>/.learnings/ERRORS.md`:

```markdown
## <date> | <failure-category>
Pattern-Key: <short-slug-unique-to-this-error>
Tool: <tool name if applicable>
Error: <exact error message or behaviour>
Root cause: <one sentence>
Fix applied: <what was changed>
Status: resolved
```

If the same Pattern-Key already exists in ERRORS.md, update it instead of adding a duplicate.

## Step 5 - Verify

After applying the fix, tell the user exactly how to re-run the agent and what to look for:
- Which field in `agent_runs` confirms the fix worked (e.g. `status = 'success'`, `tokensIn` within budget)
- What span in the trace should now show `status: OK`
- Whether to run `/ai-agent-create-evals` to capture this scenario as a regression test
