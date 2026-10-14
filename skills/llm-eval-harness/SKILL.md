---
name: llm-eval-harness
description: Design and run a reproducible evaluation for any LLM system (RAG pipeline, agent, chatbot, classifier, extraction prompt). Builds a golden set, picks scorers, runs baseline vs candidate, and reports per-case regressions, not just averages. Use when the user wants to evaluate, benchmark, compare models or prompts, or prove an LLM change helped.
---

# LLM eval harness

## Ground rules
- Ask before reading any data, logs, outputs or existing eval files. Name the file and wait for a yes. Never open `.env` or print API keys.
- Ask before running anything that calls a paid API. State the number of calls and the model first.
- Never fabricate scores, cases or results. If something was not run, say so.
- Only modify what the user asks for. Do not refactor the system under test.

## Step 1: Frame the eval (ask, do not assume)
Get answers to these before writing code. One short message, max 4 questions:
1. What is the system and its input/output contract?
2. What does "good" mean here? (correct answer, grounded in sources, right action, right format, refused when it should)
3. Is there an existing labeled set, or do we build one?
4. What is the comparison? (baseline vs candidate prompt, model A vs model B, before vs after a fix)

## Step 2: Golden set
- Store as `eval/golden.jsonl`, one case per line:
  `{"id": "c001", "input": ..., "expected": ..., "tags": ["category", "difficulty"], "notes": ""}`
- Size: 20 to 50 cases to start. Cover every category in proportion, plus deliberate edge cases (ambiguous, out-of-scope, adversarial, should-refuse).
- Every expected value must be verifiable by the user. Mark any case Claude drafted as `"source": "draft"` until the user confirms it.
- Keep a held-out split (`eval/holdout.jsonl`, about 20%) that is never used while iterating on prompts. Check it once at the end to detect overfitting.

## Step 3: Scorers
Choose the cheapest scorer that measures the thing. In order of preference:

| Scorer | Use for | Notes |
|---|---|---|
| Exact / normalized match | Labels, codes, numbers, IDs | Normalize case, whitespace, units first |
| Rule-based | Format, required fields, JSON schema, citations present | Deterministic, free |
| Set overlap (precision/recall/F1) | Retrieved docs, cited refs, extracted entities | Report P and R separately, not just F1 |
| Numeric tolerance | Computed values | State the tolerance explicitly |
| LLM-as-judge | Open-ended quality, faithfulness, helpfulness | Last resort, see below |

LLM-as-judge rules:
- Fixed judge prompt in `eval/judge_prompt.md`, versioned. Never edit it mid-comparison.
- Judge model differs from the model under test when possible.
- Rubric with 3 to 5 explicit levels and an example per level. Ask for the reasoning before the score.
- Temperature 0. Validate the judge on 10 hand-labeled cases and report agreement before trusting it.

RAG-specific metrics, when relevant: retrieval recall@k, context precision, answer faithfulness to retrieved context, answer correctness.
Agent-specific metrics, when relevant: final outcome correctness, action correctness (right writes, no forbidden writes), grounding/citation recall, steps used, failure-mode distribution.

## Step 4: Runner
- `eval/run.py`: loads cases, calls the system, applies scorers, writes `eval/runs/<run_name>.jsonl` with input, output, per-scorer scores, latency, tokens, cost estimate.
- Config in one place: model, temperature, prompt version, seed if supported.
- Cache outputs by (case id, system version) so rescoring never re-calls the API.
- Dry-run mode that runs 2 cases first, so failures surface cheaply.

## Step 5: Compare
`eval/compare.py baseline candidate` produces:
1. Aggregate table: each metric, baseline, candidate, delta.
2. Per-tag breakdown: where did it improve or regress.
3. Regression list: every case that went from pass to fail, with both outputs side by side. This is the most important section.
4. Fix list: every case that went from fail to pass.
5. Cost and latency delta.

Noise rule: with fewer than 50 cases, a delta of 1 or 2 cases is not a signal. Say so. If the user can afford it, run each system 3 times and report mean and range.

## Step 6: Report
Short markdown in `eval/report.md`:
- What was compared and why
- Headline result in one sentence
- Aggregate and per-tag tables
- Top regressions with a one-line cause each
- Holdout result (final run only)
- Limitations: set size, judge agreement, anything not measured
- Recommendation: ship, revise, or revert

## Applying this to ARC-style benchmarks
When the benchmark itself is the scorer (for example ARC WRFM-Ops), skip Steps 2 and 3. Use the benchmark's per-task score as the metric, build a fixed list of dev spec IDs as the "golden set", and use `agent-failure-triage` for the per-case analysis. Keep a few spec IDs untouched as a holdout until the final check.
