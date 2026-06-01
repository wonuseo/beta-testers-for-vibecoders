---
description: 코드 diff에 합성 베타테스트 실행 — 베타 계획·페르소나 테스터 모집·해피패스/엣지케이스 증거 수집·트리아지·픽스 제안 (wonuseo/beta-testers-for-vibecoders 기반)
argument-hint: "[edge|happy|diverse] [setup|run] [--diff <range>] [--testers <n>] [--fix] [--dry-run] ..."
---

# /code-beta — Synthetic Beta Program Harness

Run a Claude Code-native beta program on a code diff. The harness now has two layers:

1. **Setup / recruitment** — decide what should be beta-tested and recruit the right synthetic testers.
2. **Run / evidence loop** — define the best-case path, execute concrete beta activities, collect edge-case evidence, surface developer insight, triage gaps, and rerun failures.

No external services. The product surface is the Claude Code slash command plus this skill file.

## Usage

```text
/code-beta [track] [options]        # one-shot: setup + run
/code-beta setup [track] [options]  # design beta plan + tester recruitment brief
/code-beta run [track] [options]    # execute recruited testers from saved config
```

`[track]` is an optional positional shorthand for a beta track — just type the name:

```text
/code-beta edge              # edge-case run only (= --track edge)
/code-beta happy             # happy-path feedback run only
/code-beta diverse           # diverse-opinions run only
/code-beta edge,diverse      # two tracks (comma-separated, no spaces)
/code-beta                   # all three tracks (default)
/code-beta setup edge        # setup-only, edge track
/code-beta edge --testers 10 # positional track + flags mix freely
```

Both of these also work (separate tokens, any order) — but the comma form is canonical:

```text
/code-beta edge happy        # ✓ two separate track tokens, unioned
/code-beta edge, diverse     # ✗ space after comma — use edge,diverse
```

Argument parsing — classify each token **by what it is, not where it sits**:
- the token `setup` or `run` sets the **mode** (default: one-shot setup+run)
- a track id (`edge`/`happy`/`diverse`), a comma-joined list (`edge,diverse`, no spaces), or multiple separate track tokens (`edge happy`) set the **track list** (separate tokens are unioned)
- order doesn't matter — `setup edge` and `edge setup` are identical
- track ids are **case-insensitive** (`EDGE` == `edge`)
- if a positional track and `--track` **disagree**, `--track` wins; if they agree they're deduped
- any token that is neither a mode, a track, nor a known `--flag` → **stop and list the valid modes and track ids** (do not silently guess)

`--track <list>` is an equivalent alias for the positional track form.

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
--track <list>                 Beta tracks to run: edge, happy, diverse (default: all three; or pass as positional)
--fix                          Apply fix proposals and rerun failing testers
--dry-run                      Write setup artifacts only; do not run testers
```

## Beta tracks

Every run is organized into three purpose-driven **tracks**. Each track is a self-contained mini-beta with its own objective, tester subset, activities, and report section:

| Track | id | Purpose | Activity role | Tester lean |
|---|---|---|---|---|
| Edge cases | `edge` | bend/break the change — find misuse, env mismatch, integration friction, docs gaps | `edge_case_probe` | sonnet-leaning, deeper evidence |
| Happy path | `happy` | real target-user feedback on the best-case path — does it actually feel good? | `happy_path_validation` | the primary user segment, realistic context |
| Diverse opinions | `diverse` | breadth of perspectives — overlooked risks, good signs to preserve, product ideas | `developer_insight` | many haiku, varied segments |

- **Default (no track given):** run all three tracks. The `--testers` budget is split across them (roughly even, min 1 each), and the report has one section per track.
- **Track given** (positional `edge` or `--track edge,diverse`): run only the named tracks. The full `--testers` budget is redistributed into the selected tracks so each runs **deeper** (more/stronger testers, stricter evidence).

Selecting a single track is the way to say "this run is only about X."

## Reporting to the user — 3 stages

Do **not** dump everything in one message. Surface progress to the user (in their language) as three distinct stages, each as its own message:

**Stage 1 — Plan ("이렇게 베타테스트할게").** Before spawning any tester, announce the plan:
- target diff summary (files, change type)
- active tracks
- tester roster: per track, count · model · segment
- best-case path (1–2 lines)
- If `--dry-run`, stop here.
- **`run` mode still reports Stage 1.** When `run` loads a saved config and skips to Step 4, reconstruct Stage 1 from the saved plan/roster and post it before Stage 2 — the cadence must never silently drop to two stages.

**Stage 2 — Running ("베타테스트 실행 중").** When the testers are launched, post a short running status — which testers are in flight, grouped by track. Not silence, not the full reports yet. Update/condense if it spans a while. **Post it as its own message even if testers finish within seconds — never merge Stage 2 into Stage 3.**

**Stage 3 — Results ("베타테스트 결과").** After aggregation, report per track.

- **When there are failing findings:** every FAIL/UNCLEAR leads with these two, **in this order** — 1. **재현 절차 (Reproduction)** (exact steps), 2. **Expected vs Actual** (what should happen vs what does) — then severity, evidence (file:line / quote), and recommendation.
- **When the run is clean (no FAIL/UNCLEAR):** Stage 3 is still required and still per-track — report best-case-path status, a one-line per-track summary, good signs worth preserving, and any edge-case notes/ideas. A green run is not an empty message.

Always close with the triage table and the ship/no-ship + best-case-path status. **On a clean run** the triage table still appears — show its header with a single `no findings` row (don't drop the table silently).

This 3-stage cadence applies whether or not `--fix` is set. If fixes run, append a short **after-fix delta** to Stage 3 covering **only the rerun testers** (before/after per closed or still-failing criterion), not a full re-aggregate. **If Step 8 rebuilt the full pool**, label it a *full-pool rerun delta* and cover all testers.

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

**Before executing any step:** Read the `code-beta-harness` methodology skill in full. All references to "the methodology" below point to sections in that file. Reading it now avoids missing-skill errors. Prefer the user-global copy; fall back to a repo-local copy if present.

```text
Read: ~/.claude/skills/code-beta-harness/SKILL.md
# fallback if the global copy is missing:
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

First resolve which **tracks** are active (see "Beta tracks" above): the positional track arg or `--track` list, or all three (`edge`, `happy`, `diverse`) by default. Every later step is organized per active track.

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

Then add a **Tracks** section to `beta-plan.md` — for each active track, a focused sub-objective tied to the same diff:

```markdown
## Tracks (active: <edge, happy, diverse>)
### edge — break/bend the change
- Sub-objective: <what failure modes this track hunts>
- Surfaces it probes: <...>
### happy — target-user feedback on the best-case path
- Sub-objective: <does the happy path actually feel good to the primary user?>
- Reference path: <points at the Best-case path above>
### diverse — breadth of perspectives
- Sub-objective: <overlooked risks, good signs to preserve, ideas>
- Segments to sample: <varied user/maintainer viewpoints>
```

Only write sections for active tracks.

---

### Step 3 — Setup layer: tester recruitment

Using Opus-level planning/recruitment logic, recruit a quota-based tester pool from the beta plan. Do not generate generic personas.

Respect options:
- `--testers <n>` controls total quota. **Split it across active tracks** (roughly even, min 1 per track; e.g. default `6` over three tracks → `2/2/2`). `--testers` must be ≥ the active-track count — if lower, raise to the floor (one per track) and note it. If a subset is selected via `--track`, the whole budget goes into those tracks so each runs deeper.
- Every tester belongs to exactly one track; record `track` on each tester. Lean the model per track: `happy`/`diverse` favor `haiku` breadth, `edge` favors `sonnet` depth.
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
- `track` (`edge` | `happy` | `diverse`)
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

**→ Report Stage 1 (Plan) to the user now** (see "Reporting to the user — 3 stages"): target diff, active tracks, tester roster, best-case path. If `--dry-run` was provided, stop after this.

---

### Step 4 — Generate beta activities and rubrics

**→ If you arrived here via `run` mode** (config loaded at Step 2, Steps 1–3 skipped), report **Stage 1 (Plan)** from the saved config now — before continuing — so the cadence stays 3 stages.

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

Each activity's role is fixed by the tester's **track**:
- `happy` track → `happy_path_validation`: validates the best-case path from a realistic user context
- `edge` track → `edge_case_probe`: searches for plausible persona-specific misuse, confusion, environment mismatch, docs gap, or integration friction
- `diverse` track → `developer_insight`: looks for ideas, overlooked risks, and surprisingly good design choices worth preserving across varied viewpoints

Write:

```text
.harness/runs/<run_id>/test-activities.json
.harness/runs/<run_id>/rubric.json
```

Print the tester/activity/rubric table before running testers.

---

### Step 5 — Run recruited beta testers

**→ Post Stage 2 (Running) to the user** (a short "beta testing now running" status, testers grouped by track) as you spawn them.

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

The EVIDENCE REPORT FORMAT above is the tester **intake** schema. The user-facing **Stage 3** order is different (Reproduction first, then Expected vs Actual) — the aggregator (Step 6) reorders intake fields into the Stage 3 order; testers don't need to.

Run all tester subagents. Collect their evidence reports under:

```text
.harness/runs/<run_id>/testers/<tester-id>.md
```

---

### Step 6 — Aggregate, triage, and escalate

Using the aggregator model policy (`sonnet` by default), aggregate results. **First roll up per active track, then overall:**
- `edge` track: blockers/edge cases found, severity spread
- `happy` track: best-case path status + the target user's felt experience
- `diverse` track: overlooked risks, good signs, ideas, breadth of viewpoints
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

**→ Report Stage 3 (Results) to the user**, organized per track, following the **Stage 3** rules in "Reporting to the user — 3 stages" (failing findings lead with Reproduction → Expected vs Actual; clean runs still report per-track summary + good signs; close with triage + ship/no-ship; `--fix` appends the rerun-only after-fix delta). That section is the single source of truth for the finding format — don't restate it here.

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
- **Closure-only rerun (targeted):** re-run each previously-failing tester against the **same activity and the exact criteria that failed** — nothing else. The only question this rerun answers is *"is each originally-failing criterion now PASS?"* Report `gap closed: yes/no` per criterion with before/after evidence.
- **Do not hunt for new findings during the closure rerun.** The verifier's job is to confirm the patch, not to re-review the whole surface. If something new is noticed in passing, write it to the **next-round backlog** (below) — it does **not** re-enter the fix loop this session.

**Convergence / exit (this is deliberate — the harness must stop, not nitpick forever):**
- Run the closure rerun **once**. Do not auto-loop.
- A finding is only chased this session if it was in the **original** triage. New, lower-severity observations surfaced by fixes go to the backlog for the user to decide on later — they are *not* auto-fixed.
- Declare done when every original P0/P1 (and any user-approved P2/P3) is `closed`, even if the backlog is non-empty. A non-empty backlog with no open original blockers is a **successful** exit, not a reason to keep going.

Write:

```text
.harness/runs/<run_id>/gap-closure.md      # per original criterion: closed yes/no + before/after
.harness/runs/<run_id>/next-round-backlog.md  # new observations, NOT fixed this session
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

## Results by track
### edge — edge cases found
### happy — target-user feedback
### diverse — perspectives, good signs, ideas
(omit sections for tracks not run this session)

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
