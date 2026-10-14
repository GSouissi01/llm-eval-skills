---
name: prompt-change-log
description: Record every system prompt or agent-logic change as a hypothesis-driven entry, before and after the change. Use whenever the user is about to edit an LLM system prompt, tool descriptions, agent loop rules or decision checklists, and after the change is evaluated. Produces a defensible design history for reports and reviews.
---

# Prompt change log

## Ground rules
- Ask before reading or writing any file. Name the file and wait for a yes.
- One change per entry. If the user wants three edits, make three entries and test them separately when the budget allows.
- Never change prompt text that the entry does not name. Only modify what is explicitly requested.
- Never write task-specific content into a prompt (IDs, dates, well names, exact task wording). If the proposed text contains any, stop and flag it as overfitting.

## Log file
`results/prompt-changes.md`. If missing, ask before creating it with the title `# Prompt change log`.

Each entry has an ID: P-001, P-002, and so on. The same ID goes in the `fix_id` column of `results/log.csv` when a triage row triggered the change.

## Before the change: open an entry

```
## P-00X: <short title>
Status: proposed
Date: YYYY-MM-DD
Agent version before: vN

Target pattern: <code from agent-failure-triage, e.g. AUTH>
Evidence: <spec IDs and counts, e.g. 3 of 8 failures are AUTH>
Hypothesis: <if we change X, the agent will do Y, because Z>

Change type: prompt rule | tool description | code guard | tool ordering | model setting
Exact change:
  Before: <verbatim old text or "none">
  After:  <verbatim new text>

Regression risk: <which task types could break, and why>
Test plan:
  - Target specs: <specs that failed with this pattern>
  - Control specs: <specs that passed before and could break>
```

Rules for a good entry:
- The hypothesis must be falsifiable by the test plan.
- At least two control specs, chosen from task types the change could affect (for example, a write-permission rule needs a task that legitimately writes).
- Prefer the smallest change that could work. A one-line rule beats a rewritten section.

## After the change: close the entry

```
Result:
  Target specs: <spec: before -> after>
  Control specs: <spec: before -> after>
  Net score delta: <+/- n>
Verdict: kept | reverted | revised (see P-00Y)
Agent version after: vN+1
Lesson: <one sentence, reusable beyond this task>
```

Verdict rules:
- Any control spec regresses: do not keep as is. Revise or revert.
- Target improves and controls hold: keep.
- No change on target: revert. The hypothesis was wrong; record why in Lesson.
- LLM output is noisy. If a result looks marginal, say so and suggest one rerun before deciding, but remind the user each rerun costs a run.

## Summary view
On request, produce a table of all entries:

| ID | Pattern | Change (one line) | Delta | Verdict |
|---|---|---|---|---|

This table is the "design notes" section of any final report.
