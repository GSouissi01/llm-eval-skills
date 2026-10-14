# Claude Skills for LLM Evaluation and Agent Reliability

Four Claude Code skills I wrote to evaluate LLM systems and make agents reliable. Together they cover the loop I use on every agent project: audit the code, run the eval, triage what failed, change one thing, and record whether it helped.

I built them while developing my agent for the ARC Agent Reliability Challenge (WRFM-Ops benchmark), where it scored 100 on a scored session: [arc-ai-agent](https://github.com/GSouissi01/arc-ai-agent).

## The skills

| Skill | What it does | Use it when |
|---|---|---|
| [`llm-eval-harness`](skills/llm-eval-harness/SKILL.md) | Builds a reproducible eval for any LLM system: golden set with a held-out split, cheapest-first scorers, cached runner, baseline vs candidate comparison with a per-case regression list | You need to prove a prompt, model or retrieval change actually helped |
| [`agent-failure-triage`](skills/agent-failure-triage/SKILL.md) | Classifies a failed agent run into one of 11 failure patterns, names the root-cause step, proposes a pattern-level fix and logs it | An agent task failed or scored low |
| [`prompt-change-log`](skills/prompt-change-log/SKILL.md) | Turns every prompt or agent-logic edit into a falsifiable hypothesis with target and control cases, then records keep, revise or revert | You are about to edit a system prompt, tool description or agent rule |
| [`arc-agent-code-audit`](skills/arc-agent-code-audit/SKILL.md) | Static pre-run audit of an agent against a 30-check reliability list: run config, date and ID handling, grounding, computation, write safety, error handling, observability | Before spending paid runs on a new agent version |

## How they fit together

```
arc-agent-code-audit      static checks before any paid run
        |
llm-eval-harness          dev runs on a fixed case set, holdout kept aside
        |
agent-failure-triage      classify each failure, propose a pattern-level fix
        |
prompt-change-log         one change, one hypothesis, target + control cases
        |
llm-eval-harness          re-run, compare per case, keep or revert
```

## Design principles

- **Per-case regressions over averages.** With fewer than 50 cases, a 1 to 2 case delta is noise, and the skills say so.
- **Cheapest scorer that measures the thing.** Exact match and rule checks first, LLM-as-a-judge last, and only after checking judge agreement on hand-labelled cases.
- **Fixes target patterns, not tasks.** Any fix that mentions a specific ID, date or task wording is rejected as overfitting.
- **One change at a time.** Every prompt edit carries a falsifiable hypothesis and at least two control cases that could regress.
- **Ask before touching anything.** The skills ask before reading files or calling paid APIs, never open `.env`, and never invent scores.

## Install

Claude Code loads skills from `~/.claude/skills/`:

```bash
git clone https://github.com/GSouissi01/llm-eval-skills.git
cp -r llm-eval-skills/skills/* ~/.claude/skills/
```

On claude.ai, zip each skill folder so `SKILL.md` sits at the top level of the zip, then upload it under Skills.

## Notes

- `agent-failure-triage` and `arc-agent-code-audit` use the ARC WRFM-Ops outcome codes and tool names. The method carries over to other agent benchmarks; swap the pattern table and outcome codes.
- `arc-agent-code-audit` cites section numbers from a companion knowledge skill on oil and gas agent reliability, which is not included in this repo.

## License

MIT
