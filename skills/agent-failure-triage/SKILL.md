---
name: agent-failure-triage
description: Triage a failed or low-scoring agent benchmark task (ARC WRFM-Ops or similar) from its evaluator output. Classifies the failure by pattern, proposes a pattern-level fix, and appends a row to the results log. Use after any --spec run, or when the user shares evaluator output or an agent trace.
---

# Agent failure triage

## Ground rules
- Ask before reading any log, output or result file. Name the exact file and wait for a yes. Never open `.env`.
- Never run `python main.py` without `--spec <id>`. A run without it submits a scored session.
- Never put IDs or dates in a fix. Every run gets a new snapshot, shifted IDs and a random operational date.
- Propose fixes only. Do not edit `agent.py` or the system prompt until the user approves.

## Inputs
- Spec ID and evaluator score/log.
- Agent trace: each step's `current_state`, `plan`, function call and result, plus the final `wrfm_respond` payload.

## Step 1: Extract facts
Produce this table:

| Field | Value |
|---|---|
| Spec ID | |
| Task (one line) | |
| Expected outcome | |
| Actual outcome | |
| Score | |
| Steps used / 30 | |
| Writes performed | create/update calls, or none |
| Ground refs cited | |
| Ground refs expected (if the evaluator shows them) | |

Outcome codes: `ok_answer`, `denied_security`, `none_clarification_needed`, `none_unsupported`, `error_internal`.
Ground ref types: `wiki`, `table`, `well`, `deferment_event` (id = the `EVENT_KEY` value from the deferments table).

## Step 2: Classify
Pick one primary pattern, plus one secondary if clearly present.

| Code | Pattern | Typical signal |
|---|---|---|
| OUT | Wrong outcome code | `ok_answer` where `denied_security` was expected, or the reverse |
| CLAR | Clarification or unsupported misjudged | Answered an ambiguous task, or asked for clarification on a clear one |
| REF | Missing or wrong ground refs | Answer correct, refs incomplete; `deferment_event` id not the `EVENT_KEY` |
| AUTH | Write without authority | Create/update on a review, anomaly or clarification task, or role lacks authority per governance docs |
| NOWRITE | Required write missing | Task asked to record or update; agent only answered |
| WIKI | Docs skipped | Answered or wrote without loading the relevant wiki or governance page |
| DATE | Date or ID error | Used a date other than system "today", wrong period window, stale or invented IDs |
| DATA | Wrong query or computation | Wrong filter, unit, sum vs average, missed rows (V1 has no joins or group-by, so the agent aggregates itself) |
| LOOP | Step exhaustion or repetition | Hit MAX_STEPS, repeated the same call |
| PARSE | Structured output failure | Invalid NextStep, malformed function arguments |
| INFRA | Platform or SDK error | `error_internal`, timeouts, HTTP errors |

## Step 3: Root cause
One or two sentences naming the trace step where it went wrong and why: prompt gap, missing tool use, model limitation, or code bug.

## Step 4: Pattern-level fix
- The fix must work across tasks of the same pattern. Reject any fix that mentions this task's wells, events, dates or wording.
- Preferred order:
  1. A general rule or decision checklist in the system prompt
  2. A code guard (for example, block write actions when the task is a review or anomaly check)
  3. Tool-call ordering (for example, force a governance wiki read before any write)
- State the regression risk: which other task types the fix could break.
- If the fix changes the prompt, hand off to the `prompt-change-log` skill.

## Step 5: Log row
Append to `results/log.csv`. If the file is missing, ask before creating it with this header:

```
run_date,spec_id,expected_outcome,actual_outcome,score,pattern,secondary,root_cause,fix_id,agent_version
```

Keep `root_cause` under 15 words. `fix_id` matches the prompt-change-log entry (for example P-003) or `none`.

## Output to the user
1. Facts table
2. Pattern and root cause
3. Proposed fix (exact prompt text or code diff) and regression risk
4. The log row
5. Pattern tally across `results/log.csv` (count per pattern), so the next fix targets the biggest bucket
