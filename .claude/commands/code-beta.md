# /code-beta — Synthetic Beta Program Harness

Run a Claude Code-native beta program on a code diff. The harness now has two layers:

1. **Setup / recruitment** — decide what should be beta-tested and recruit the right synthetic testers.
2. **Run / evidence loop** — define the best-case path, execute concrete beta activities, collect edge-case evidence, surface developer insight, triage gaps, and rerun failures.

No external services. The product surface is the Claude Code slash command plus this skill file.

## Usage

```text
/code-beta setup [options]        # design beta plan + tester recruitment brief
/code-beta run [options]          # execute recruited testers from saved config
/code-beta [options]              # one-shot: setup + run
```

Options:

```text
--diff <range>                 Git range to test (default: staged changes, then working tree)
--testers <n>                  Number of synthetic beta testers to recruit (default: 6)
--planner-model opus           Model policy for beta plan/recruitment (default: opus)
--recruiter-model opus         Alias for planner-model during recruitment (default: opus)
--tester-model haiku|sonnet    Default tester model (default: haiku)
--escalation-model sonnet      Recheck high-risk/inconclusive findings (default: sonnet)
--aggregator-model sonnet      Triage/dedupe/fix proposal model (default: sonnet)
--mix cheap|balanced|deep      Tester depth/cost profile (default: balanced)
--mode realistic|scoped|deep   Context policy (default: realistic)
--max-files-per-tester <n>     Context budget per tester (default: 6)
--focus <label>                Narrow setup to a named surface (onboarding, API, CI, security...)
--fix                          Apply fix proposals and rerun failing testers
--dry-run                      Write setup artifacts only; do not run testers
```

## Model policy

Use this default split unless the user overrides it:

| Role | Model | Why |
|---|---|---|
| Beta planner | `opus` | chooses beta objective, beta type, surfaces, and exit criteria |
| Tester recruiter | `opus` | designs quota, segments, context policy, and activities |
| Breadth beta testers | `haiku` | cheap, fast, diverse first-pass findings |
| High-risk / deep testers | `sonnet` | confirms security, API, CI, and ambiguous findings |
| Aggregator / triage | `sonnet` | dedupes, classifies, and proposes minimal fixes |

If Claude Code cannot actually switch subagent models in the current environment, still record the intended model policy in the artifacts and proceed with the available model.

## Workflow

**Before executing any step:** Read `.claude/skills/code-beta-harness.md` in full. All references to "the methodology" below point to sections in that file. Reading it now avoids missing-skill errors if the file was added mid-session or the skill is not registered.

```text
Read: .claude/skills/code-beta-harness.md
```

Follow these steps in order.

---

### Step 1 — Resolve target diff

```bash
# If --diff was provided, use that range:
git diff <range> --stat
git diff <range>

# Otherwise, use staged changes:
git diff --cached --stat
git diff --cached

# If nothing staged, fall back to working tree:
git diff --stat
git diff
```

If the diff is empty, tell the user and stop.

Summarize:
- files changed
- areas affected
- change type: feature, bug fix, refactor, config, docs, harness change
- likely user-visible or maintainer-visible surfaces

---

### Step 2 — Setup layer: beta plan

If command is `run` and `.harness/code-beta.config.json` exists, load it and skip to Step 4. Otherwise create a beta plan first.

Read bounded context only:
- `README.md`
- `CLAUDE.md` / `AGENTS.md` if present
- package/config files (`package.json`, `pyproject.toml`, `Cargo.toml`, etc.)
- changed files and one-hop callers only if needed
- existing tests only to infer test commands, not to overfit old behavior

Do **not** read the whole repo. Large-project behavior must be scoped.

Using the methodology sections **Beta Program Setup** and **Beta Type Selection**, write:

```text
.harness/runs/<run_id>/beta-plan.md
.harness/runs/<run_id>/scope-map.json
.harness/runs/<run_id>/risk-map.json
```

The beta plan must include:
- beta objective / hypothesis
- best-case scenario: the ideal user journey or developer outcome this change should enable
- beta type: closed, focused, technical, open-like breadth, marketing/docs, post-release/staged, or hybrid
- surfaces to test
- explicit out-of-scope areas
- core beta activities
- evidence schema
- exit criteria

Before recruiting testers, write a short **Best-case path** section in `beta-plan.md`:

```markdown
## Best-case path
- Primary user: <who benefits most if this works>
- Happy path: <3-7 steps from entry point to successful outcome>
- Value delivered: <what becomes easier/safer/faster>
- Success signal: <what evidence would show the happy path is real>
```

This is not marketing copy. It is the optimistic reference path that edge-case testers try to bend, break, or clarify.

---

### Step 3 — Setup layer: tester recruitment

Using Opus-level planning/recruitment logic, recruit a quota-based tester pool from the beta plan. Do not generate generic personas.

Respect options:
- `--testers <n>` controls total quota
- `--mix` controls distribution:
  - `cheap`: mostly Haiku breadth testers, shallow activities
  - `balanced`: Haiku breadth + Sonnet escalation/deep tester
  - `deep`: more Sonnet high-risk testers, stricter evidence
- `--mode` controls context access:
  - `realistic`: each tester sees only what their real user segment would plausibly see first
  - `scoped`: changed surface + selected docs/config
  - `deep`: maintainers/security may inspect internals, but normal users still start with public context

Each recruited tester must have:
- id / segment / beta type fit
- quota rationale
- intended model (`haiku` or `sonnet`)
- context policy
- allowed context list
- screening criteria / prior knowledge boundary
- activity assignment
- success condition

Write:

```text
.harness/runs/<run_id>/tester-recruitment.md
.harness/runs/<run_id>/personas.json
.harness/code-beta.config.json
```

If `--dry-run` was provided, stop after printing the beta plan and recruitment table.

---

### Step 4 — Generate beta activities and rubrics

For each tester, generate one concrete beta activity and 2–5 falsifiable criteria using methodology sections **Beta Activity Design** and **Rubric Generation**.

A beta activity is not a vague review. It is a task like:

```text
Start from README only. Try to discover and run /code-beta on HEAD~1..HEAD. Record the exact step where you get blocked.
```

Each activity must specify:
- task / starting point
- allowed context
- forbidden context unless progressively requested
- commands the tester may run, if any
- required evidence: steps attempted, environment/context, expected vs actual, severity, reproducibility, recommendation

Each activity must also specify one of these roles:
- `happy_path_validation`: validates the best-case path from a realistic user context
- `edge_case_probe`: searches for plausible persona-specific misuse, confusion, environment mismatch, docs gap, or integration friction
- `developer_insight`: looks for ideas, overlooked risks, and surprisingly good design choices worth preserving

Write:

```text
.harness/runs/<run_id>/test-activities.json
.harness/runs/<run_id>/rubric.json
```

Print the tester/activity/rubric table before running testers.

---

### Step 5 — Run recruited beta testers

Spawn one Claude Code subagent per tester/activity row using the `Agent` tool when available.

Each subagent receives:

```text
You are a recruited beta tester, not an omniscient code reviewer.

TESTER:
<id, segment, model policy, prior knowledge boundary>

BETA ACTIVITY:
<task, starting point, allowed context, forbidden context, commands>

CONTEXT POLICY:
Start only with the allowed context. If blocked, request the smallest additional file/context needed and explain why. Hidden internal knowledge required for success is a docs/UX gap, not a pass.

RUBRIC:
<criteria>

YOUR TASK:
1. Attempt the beta activity under the context policy.
2. Record steps attempted.
3. For each criterion, return PASS, FAIL, or UNCLEAR.
4. For each FAIL/UNCLEAR, provide expected vs actual, evidence, reproducibility, severity, and recommendation.
5. Add an "Edge cases / ideas" section even if all criteria pass.
6. Call out one "Good sign" if the change has a robust or valuable aspect worth preserving.
7. Do not speculate. If runtime verification is needed, mark UNCLEAR and state the exact missing runtime condition.

EVIDENCE REPORT FORMAT:
## Tester: <id>
### Segment / beta type
### Activity attempted
### Context used
### Steps attempted
### Results
| Criterion | Result | Evidence | Severity |
### Failures / unclear findings
For each FAIL/UNCLEAR:
- Expected
- Actual
- Evidence: file:line, command output, docs quote, or explicit missing-context note
- Reproduction
- Recommendation
### Overall score: <X>/<Y> criteria passing
### Edge cases / ideas
- Persona-specific edge case: <plausible risk, or "none found">
- Missed risk: <risk the developer may not have considered, or "none found">
- Good sign: <what worked well or should be preserved>
- Idea sparked: <optional improvement idea, or "none">
```

Run all tester subagents. Collect their evidence reports under:

```text
.harness/runs/<run_id>/testers/<tester-id>.md
```

---

### Step 6 — Aggregate, triage, and escalate

Using the aggregator model policy (`sonnet` by default), aggregate results:
- best-case path status: validated / partially validated / not validated
- per-tester score
- total pass/fail/unclear
- persona-specific edge cases
- missed risks
- good signs worth preserving
- ideas sparked during the run
- deduped findings
- severity: P0/P1/P2/P3
- classification: code defect, docs/UX gap, missing test, environment issue, known issue, false positive, deferred roadmap input
- current-release action: fix now, accept known issue, defer, re-run with more context, escalate to Sonnet

High-risk Haiku findings must be escalated to Sonnet before final blocker status:
- security/privacy
- data loss/corruption
- auth/permission
- CI/release gating
- public API breakage
- findings with weak or ambiguous evidence

Write:

```text
.harness/runs/<run_id>/aggregate.md
.harness/runs/<run_id>/aggregate.json
```

If all exit criteria pass, write the session report and finish.

The aggregate must produce value even when no blocker is found. A clean run still needs best-case validation, edge-case notes, good signs, and optional ideas.

---

### Step 7 — Fix proposals or decision log

For each unresolved blocker, propose a minimal fix using methodology section **Fix Proposals**.

A fix proposal must:
- reference tester, activity, criterion, and severity
- quote the evidence
- identify exact file/line range to change
- describe minimal change
- include before/after diff snippet where possible
- state whether it fixes code, docs, test, or harness ambiguity

Write:

```text
docs/fix-proposals/<YYYY-MM-DD>-<slug>.md
.harness/runs/<run_id>/decision-log.md
```

If `--fix` was NOT given, ask the user whether to apply fixes. Stop if they say no; write partial session report noting fixes proposed but not applied.

---

### Step 8 — Apply fixes and rerun failed testers

If `--fix` was given or user approved:
- Apply minimal fixes.
- Rerun only failed/unclear testers first.
- Compare before/after evidence for the same activity and rubric.
- If critical gaps close, optionally rerun the full tester pool once.

Do not loop more than once automatically. Surface remaining failures for human review.

Write:

```text
.harness/runs/<run_id>/gap-closure.md
```

---

### Step 9 — Write session report

Write a session report to `docs/beta-sessions/<YYYY-MM-DD>-<HH-MM>.md` containing:

```markdown
# Beta Session: <date> <time>

## Target diff

## Best-case path
- primary user
- happy path
- value delivered
- status: validated / partially validated / not validated

## Beta plan
- objective
- beta type
- surfaces
- out of scope
- exit criteria

## Tester recruitment
- tester count
- model policy
- segment quota table

## Activities and rubric

## Results: initial run

## Persona edge cases

## Missed risks

## Good signs

## Ideas sparked

## Escalations

## Triage / decisions

## Fix proposals

## Results: after fixes

## Gap closure

## Evidence appendix
```

Print: `Session report written to docs/beta-sessions/<filename>`.
