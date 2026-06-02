---
name: ai-agent-create-evals
description: Build a golden-trace eval harness for an agent or workflow - extract representative run fixtures, write a typed test runner, score pass/fail per fixture, and output a results report. Use when the user wants to test their agent, create evals, make sure the agent still works after a change, write regression tests, or validate agent behavior before shipping. Triggers on: "create evals for my agent", "test my agent", "write tests for the agent", "regression test the agent", "make sure the agent still works", "validate agent behavior", "agent tests", "eval harness".
---

# ai-agent-create-evals

Build a golden-trace eval harness. Extracts real run history as fixtures, writes a test runner that replays them, and scores actual vs expected behavior. This gives you a regression test you can run before every deploy.

## What makes a good agent eval

Unlike unit tests, agent evals don't test exact output - LLMs are non-deterministic. Instead they test:
- **Tool call sequence** - did the agent use the right tools in roughly the right order?
- **Output shape** - does the result match the expected Zod schema?
- **No-hallucination** - did the agent fabricate a result when it should have stopped?
- **Cost bounds** - did the run stay within expected token budget?

## Step 1 - Find run history

Read `activity-log.json` or query the `agent_runs` table for this agent. Look for:
- 3-5 runs with `status = 'success'` that cover different input types
- 1-2 runs with `status = 'error'` that represent known failure cases (these become negative evals)

If there's no run history yet, ask the user to describe 2-3 representative scenarios and create synthetic fixtures from those descriptions.

## Step 2 - Extract golden fixtures

For each selected run, create a fixture in `evals/<agent-name>/fixtures/`:

`evals/<agent-name>/fixtures/<scenario-name>.json`:
```json
{
  "id": "<scenario-name>",
  "description": "What this scenario tests",
  "input": { /* the exact input that was passed to run() */ },
  "expected": {
    "toolSequence": ["tool_a", "tool_b"],      // tools that should be called, in order
    "outputShape": { /* Zod-compatible shape description */ },
    "maxTokensIn": 15000,                       // cost bound
    "shouldSucceed": true                        // false for error-case fixtures
  }
}
```

For error-case fixtures, set `"shouldSucceed": false` and add:
```json
"expectedError": "partial match of expected error message"
```

## Step 3 - Write the test runner

`evals/<agent-name>/runner.ts`:

```typescript
import { run } from '@/agents/<name>'
import { readFileSync, readdirSync, writeFileSync } from 'fs'
import { join } from 'path'

const FIXTURES_DIR = join(process.cwd(), 'evals/<name>/fixtures')
const RESULTS_PATH = join(process.cwd(), 'evals/<name>/results.json')

type Fixture = {
  id: string
  description: string
  input: unknown
  expected: {
    toolSequence?: string[]
    maxTokensIn?: number
    shouldSucceed: boolean
    expectedError?: string
  }
}

type EvalResult = {
  id: string
  description: string
  passed: boolean
  failures: string[]
  tokensIn: number
  durationMs: number
}

async function runEval(fixture: Fixture): Promise<EvalResult> {
  const start = Date.now()
  const failures: string[] = []
  let tokensIn = 0

  const toolCallLog: string[] = []

  // Patch: intercept tool calls to capture sequence
  // (inject a spy wrapper around execute() for each tool - or read from agent_runs after the run)

  try {
    const result = await run(fixture.input as any)

    if (!fixture.expected.shouldSucceed) {
      failures.push('Expected run to fail but it succeeded')
    }

    // Tool sequence check
    if (fixture.expected.toolSequence && toolCallLog.length > 0) {
      const expected = fixture.expected.toolSequence
      const actual = toolCallLog
      for (let i = 0; i < expected.length; i++) {
        if (actual[i] !== expected[i]) {
          failures.push(`Tool sequence mismatch at step ${i}: expected "${expected[i]}", got "${actual[i] ?? 'nothing'}"`)
        }
      }
    }

  } catch (err) {
    if (fixture.expected.shouldSucceed) {
      failures.push(`Unexpected error: ${String(err)}`)
    } else if (fixture.expected.expectedError) {
      if (!String(err).includes(fixture.expected.expectedError)) {
        failures.push(`Wrong error: expected to contain "${fixture.expected.expectedError}", got "${String(err)}"`)
      }
    }
  }

  // Cost check (read from agent_runs after the run)
  // TODO: query agent_runs for the run just completed, check tokensIn

  return {
    id: fixture.id,
    description: fixture.description,
    passed: failures.length === 0,
    failures,
    tokensIn,
    durationMs: Date.now() - start,
  }
}

async function main() {
  const fixtureFiles = readdirSync(FIXTURES_DIR).filter(f => f.endsWith('.json'))
  const fixtures: Fixture[] = fixtureFiles.map(f =>
    JSON.parse(readFileSync(join(FIXTURES_DIR, f), 'utf-8'))
  )

  console.log(`Running ${fixtures.length} evals for <name> agent...\n`)

  const results: EvalResult[] = []
  for (const fixture of fixtures) {
    process.stdout.write(`  ${fixture.id}... `)
    const result = await runEval(fixture)
    results.push(result)
    console.log(result.passed ? 'PASS' : `FAIL (${result.failures.join(', ')})`)
  }

  const passed = results.filter(r => r.passed).length
  console.log(`\n${passed}/${results.length} passed`)

  writeFileSync(RESULTS_PATH, JSON.stringify({ results, runAt: new Date().toISOString() }, null, 2))
  console.log(`Results written to evals/<name>/results.json`)

  if (passed < results.length) process.exit(1)
}

main().catch(console.error)
```

Add to `package.json`:
```json
"scripts": {
  "eval:<name>": "npx tsx evals/<name>/runner.ts"
}
```

## Step 4 - Add a README

`evals/<agent-name>/README.md`:
```markdown
# <Name> Agent Evals

Run: `npm run eval:<name>`

## Fixtures
| ID | Description | Success? |
|---|---|---|
<!-- one row per fixture -->

## How to add a new fixture
1. Copy a successful run from activity-log.json or agent_runs
2. Save as evals/<name>/fixtures/<scenario>.json
3. Fill in expected.toolSequence from the run's span tree
4. Run the suite to confirm it passes
```

## Step 5 - Summarise

Tell the user:
- How many fixtures were created and what scenarios they cover
- How to run: `npm run eval:<name>`
- What the tool sequence check requires (OTel tracing must be on to capture tool call order automatically - if not, show them the manual log pattern)
- Suggest running evals in CI: "Add `npm run eval:<name>` to your deploy workflow to catch regressions before they reach production"
