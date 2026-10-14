---
name: "arc-agent-code-audit"
description: "Audit ARC agent code (arc-sample-agent, sample-agent-wrfm: agent.py, main.py) before runs against the ARC reliability checklist. Use when asked to audit, review or pre-flight the ARC agent code."
---

# ARC agent code audit

Static pre-run audit of an ARC Agent Reliability Challenge agent (WRFM-Ops by default; also works for Maintenance-Ops). It finds reliability gaps in code before paid runs expose them.

Workflow position: **code audit (this skill) → `--spec` dev runs → `agent-failure-triage` on failures → final scored session.**

Criteria source: the `oil-gas-agent-reliability` skill. Load it first and cite its section numbers (2 guardrails, 3 fix table, 4.x patterns, 5 ARC checklist) in every fix. Do not duplicate its content here. Do not re-open its original papers, links or repo; if more detail is needed, ask the user to provide the source.

## Ground rules
- Ask before reading any file. Name the exact file and wait for a yes. One request may list several files.
- Never open, print or quote `.env` or any API key. If a key appears in code, report its location only, masked.
- Never run `main.py` or any command that calls the ARC platform or a paid LLM API. Static reading only.
- Do not edit any file. Propose fixes; apply only after explicit approval, and only to what was approved. Never overwrite the user's manual edits.
- Audit the code as it is now, not the original sample. If the user changed it, audit their version.
- Never put concrete IDs or dates in a proposed fix. Every run has shifted IDs and a random operational date.
- Mark every finding with evidence (file:line). If something cannot be verified statically, mark it `UNVERIFIED`, never `PASS`.

## Step 1: Scope
Ask in one short message:
1. Path to the agent folder (default: `arc-sample-agent/sample-agent-wrfm`).
2. Benchmark (WRFM-Ops or Maintenance-Ops).
3. Permission to read: `agent.py`, `main.py`, `requirements.txt`, `.gitignore`, plus any extra modules they added (helpers, prompts). Never `.env`.

## Step 2: Map the code
After permission, read the files and write a short map:
- **Bootstrap:** what it calls (system, wiki_tree, data_schema) and where the operational date is stored.
- **Loop:** step model (e.g. NextStep: current_state, plan, task_completed, function), MAX_STEPS, stop condition.
- **Tools:** where each action is dispatched (wiki_*, wrfm_data_*, wrfm_respond).
- **LLM:** call site, provider, model ID source, temperature, structured-output method.
- **Prompt:** where the system prompt lives and its rules.
- **Run modes:** how `main.py` handles `--spec`, `--workspace` and no-flag runs.

## Step 3: Run the checks
Score each check PASS / PARTIAL / FAIL / UNVERIFIED / N/A.

### A. Run configuration
| ID | Check | What to look for | Fix ref |
|---|---|---|---|
| A1 | Model pinned to a dated ID | MODEL_ID default / env; undated alias (e.g. `gpt-4.1-mini`) = PARTIAL | §2 determinism, §5.1 |
| A2 | Temperature explicitly 0 | Absent = FAIL (provider default is not 0) | §2 |
| A3 | Secrets only from env; `.env` git-ignored | Literal keys in code = FAIL (report masked location) | §2 operations |
| A4 | Accidental scored submission guarded | No-flag `python main.py` submits silently = FAIL; fix = require explicit flag/confirmation | ARC rules |
| A5 | Step-limit exit is graceful | Hitting MAX_STEPS still sends a valid `wrfm_respond` outcome | §4.5 |
| A6 | Dependencies pinned | `arc-ogchallenge>=` unbounded = LOW; pin tested version | §2 determinism |

### B. Environment robustness
| ID | Check | What to look for | Fix ref |
|---|---|---|---|
| B1 | Date from environment | `datetime.now`, `date.today`, `time.time` used for task logic = FAIL | §4.6 |
| B2 | No hardcoded IDs/dates | Literal well IDs, event keys, dates, years in code or prompt (grep digits, `20[0-9]{2}`, ID-like tokens) | §2, §6 |
| B3 | Tool outputs parsed by key | Positional indexing of rows/columns, reliance on field order or date format | §2, §4.9 |

### C. Grounding and refs
| ID | Check | What to look for | Fix ref |
|---|---|---|---|
| C1 | IDs seen this run are recorded | Collection of well IDs, EVENT_KEYs, table names, wiki paths from tool outputs | §4.4 |
| C2 | Refs validated before respond | Code-level check that every cited ref was seen; prompt-only = PARTIAL | §4.4 |
| C3 | Ref types constrained | Only wiki / table / well / deferment_event; deferment_event id = EVENT_KEY | §5 |

### D. Computation
| ID | Check | What to look for | Fix ref |
|---|---|---|---|
| D1 | Aggregations in Python | Sums/counts/averages left to the LLM (V1 has no group-by) = FAIL | §4.3 |
| D2 | Numbers normalized | Units/format handled in code before answer | §3 |

### E. Write safety
| ID | Check | What to look for | Fix ref |
|---|---|---|---|
| E1 | Code-level write gate | create/update reachable without a gate function; prompt-only rule = PARTIAL | §4.5 |
| E2 | Read-only tasks blocked | Review / anomaly / clarification tasks cannot reach writes | §2 |
| E3 | Governance read before write | Gate requires a wiki/governance load in this run | §4.5 |
| E4 | Outcome codes validated | Literal/enum of the 5 codes in the response model; free string = FAIL | §4.5 |

### F. Error handling
| ID | Check | What to look for | Fix ref |
|---|---|---|---|
| F1 | Retry with backoff | Around LLM calls and ARC client calls; none or fixed sleep = FAIL | §4.1 |
| F2 | Errors never become "no data" | `except: return []` / empty result on failure = FAIL | §3 |
| F3 | One strict-JSON retry | Parse/validation failure handling on the step model | §4.2 |
| F4 | Fatal errors still respond | Unhandled exception path ends without `wrfm_respond(error_internal)` = FAIL | §4.5 |

### G. Observability
| ID | Check | What to look for | Fix ref |
|---|---|---|---|
| G1 | Per-step trace log | current_state, plan, function, args, result saved per task | §2 operations |
| G2 | Replay cache | Prompt/response cache for offline re-scoring | §4.7 |
| G3 | Cost/token log | Tokens and steps per task recorded | §4.8 resource consistency |

### H. System prompt
| ID | Check | What to look for | Fix ref |
|---|---|---|---|
| H1 | Core rules present | Compare against §4.12 list; report missing rules only | §4.12 |
| H2 | No leaking examples | Example IDs/dates in prompt that could be copied into answers | §2 |
| H3 | Abstain path explicit | Clarify/refuse preferred over guessing | §2 |

### I. Testability
| ID | Check | What to look for | Fix ref |
|---|---|---|---|
| I1 | Repeat runs supported | Easy to run one spec K times without submitting | §4.8 |
| I2 | Results log | Per-run outcome/score recorded for consistency tracking | §4.8 |

## Step 4: Severity
- **Critical:** can trigger an accidental scored submission, an unauthorized write, or a leaked key.
- **High:** likely to lose points on many tasks (LLM arithmetic, unvalidated refs, wrong date source, free-string outcome codes, no error→respond path).
- **Medium:** hurts consistency or debuggability (no retry, no trace log, unpinned model, temperature unset).
- **Low:** hygiene (dependency pins, cache, cost log).

## Step 5: Report
Output in this order:

**1. Code map** (from Step 2, max 8 lines).

**2. Findings table** (FAIL, PARTIAL and UNVERIFIED rows only; list PASS IDs on one line below the table)

| ID | Status | Severity | Evidence (file:line) | Risk to ARC score | Fix (skill ref) |
|---|---|---|---|---|---|

**3. Top 3 fixes** by expected score gain. Each one gets: what to change, where, which pattern (§ ref), and how to verify it with a `--spec` run.

**4. Platform feedback candidates** for the benchmark maintainers (doc gaps, unsafe defaults found during the audit), max 3.

**5. Next step offer**, one line: apply the approved fixes, then dev-run with `--spec`, then `agent-failure-triage` on any failure.

Keep the report concise: no praise, no restating the code, no generic advice not tied to a finding.

## Step 6: Applying fixes (only on approval)
- Apply only the fixes the user approved, one at a time. Show a minimal diff for each.
- Edit the user's current file, not the original sample.
- After each fix, list the `--spec` command to verify it. Never run it yourself unless the user asks, and never without `--spec`.
- Re-audit only the changed checks and update their status.