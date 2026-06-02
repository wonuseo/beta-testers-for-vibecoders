# beta-testers-for-vibecoders

A Claude Code-native harness that turns a code diff into a useful synthetic beta run: best-case path, persona edge cases, missed risks, good signs, and concrete next fixes.

No external service. No CI pipeline. No test script required. Just copy two Claude files into a repo and run `/betatest`.

## Why this exists

Vibecoders ship fast. The risky part is not only “does the code compile?” It is:

- Does the intended happy path actually make sense?
- What would a first-time user misunderstand?
- What edge case would a maintainer, CI integrator, API consumer, or docs-only user hit?
- What risk did the developer probably miss?
- What is surprisingly good and should be preserved?

Traditional tests catch what you predicted. A beta run should also reveal what you did not know to ask.

## What `/betatest` does

`/betatest` runs a lightweight synthetic beta program on your current change.

```text
plan beta
→ split into tracks (edge / happy / diverse)
→ define best-case path
→ recruit per-track synthetic testers
→ assign track activities
→ collect evidence and developer insight
→ triage per track
→ propose fixes → closure-only rerun → ship decision
```

The output is not just a bug list. A useful run can end with:

1. **Best-case path** — the ideal user/developer journey this change should enable.
2. **Persona edge cases** — where specific user types get confused, misuse the feature, or hit environment/docs gaps.
3. **Developer insight** — missed risks, good signs, and ideas sparked during the run.
4. **Ship/no-ship verdict** — what to fix now, what to accept, and what to revisit later.

Three things make it more than a one-shot review:

- **Tracks** — every run is split into three purpose-driven tracks (`edge`, `happy`, `diverse`) you can run together or one at a time.
- **3-stage reporting** — you get a *Plan* before anything runs, a *Running* status, then *Results* — not one wall of text at the end.
- **Closure-only rerun + convergence** — after a fix, it re-tests *only* what failed and stops; it does not nitpick forever.

## Quickstart

Copy the harness into your project:

```bash
mkdir -p /your-project/.claude/commands /your-project/.claude/skills
cp .claude/commands/betatest.md        /your-project/.claude/commands/betatest.md
cp .claude/skills/betatest-harness.md  /your-project/.claude/skills/betatest-harness.md
```

Open a new Claude Code session in the target project, then run:

```text
/betatest --diff HEAD~1..HEAD --testers 4
```

Scope a run to a single track — just type its name (positional shorthand):

```text
/betatest edge              # only break/bend the change
/betatest happy             # only happy-path user feedback
/betatest diverse           # only diverse perspectives / insight
/betatest edge,diverse      # two tracks (comma, no spaces)
/betatest                   # all three tracks (default)
```

Or use the two-step flow:

```text
/betatest setup --diff HEAD~1..HEAD --testers 4
/betatest run --diff HEAD~1..HEAD
```

Useful variants:

```text
/betatest setup --focus onboarding --testers 3 --mix cheap
/betatest setup --focus security --testers 6 --mix deep --tester-model sonnet
/betatest edge --testers 10        # positional track + flags mix freely
/betatest run --fix
/betatest --dry-run
```

> Claude Code discovers slash commands only when a session starts. If `/betatest` is unknown after copying the files, restart Claude Code or use the manual fallback below.

## Manual fallback

If the command was copied into an already-open Claude Code session, paste this into Claude Code:

```text
Read .claude/commands/betatest.md and .claude/skills/betatest-harness.md, then follow the workflow defined in betatest.md on the diff HEAD~1..HEAD.
```

The command is plain markdown instructions, so Claude can follow it directly even before slash-command registration.

## Example output

```text
/betatest --diff HEAD~1..HEAD --testers 4

Best-case path:
  Primary user: first-time vibe coder adding the harness to a repo
  Happy path:
    1. Copy command + skill files
    2. Restart Claude Code
    3. Run /betatest on the current diff
    4. Read a concise ship/no-ship report
  Value delivered: catches docs/onboarding gaps before the change ships
  Status: partially validated

Recruited testers:
  • happy-path-first-timer     haiku   happy_path_validation   README only
  • mid-session-user           haiku   edge_case_probe         README + command docs
  • ci-integrator              haiku   edge_case_probe         docs/config only
  • maintainer-reviewer        sonnet  developer_insight       diff + public docs

Results:
  ✓ happy-path-first-timer   3/3
  ✗ mid-session-user         2/3   P1: unknown command recovery depends on restart note
  ✓ ci-integrator            2/2
  ✓ maintainer-reviewer      3/3

Persona edge cases:
  - mid-session-user: copies files during an active Claude session and expects /betatest to appear immediately.
  - ci-integrator: may assume the report creates a hard CI gate even when the docs only promise a report.

Missed risks:
  - The quickstart needs to distinguish “install files” from “start a new Claude Code session.”

Good signs:
  - Manual fallback keeps the harness usable even when command discovery fails.
  - Bounded-context tester instructions prevent omniscient code review from masquerading as user testing.

Ideas sparked:
  - Add a tiny “first run checklist” to the final report.
  - Consider a future `--ci-gate` mode only when exit-code support exists.

Verdict:
  Ship with docs fix.

Next fixes:
  1. Add restart warning near Quickstart.
  2. Clarify CI examples are report-only unless a future gate mode is enabled.

Report:
  docs/beta-sessions/2026-05-29-14-32.md
```

## Output sections

A final session report should include:

```markdown
## Best-case path
## Results by track        # edge / happy / diverse
## Ship/no-ship verdict
## Top blockers
## Persona edge cases
## Missed risks
## Good signs
## Ideas sparked
## Next fixes
## Gap closure             # after --fix: closed yes/no per original finding
## Next-round backlog      # new observations, deliberately not fixed this session
## Evidence appendix
```

The harness should produce value even when no blocker is found. Clean runs still report what worked, what edge cases were considered, and what the developer should preserve.

## Core concepts

### Beta tracks

Every run is organized into up to three purpose-driven **tracks**. A track is a self-contained mini-beta on the *same diff*, with its own sub-objective, tester subset, and report section:

| Track | id | Purpose | Activity role |
|---|---|---|---|
| Edge cases | `edge` | bend/break the change — misuse, environment mismatch, integration friction, docs gaps | `edge_case_probe` |
| Happy path | `happy` | real target-user feedback on the best-case path — does it actually feel good? | `happy_path_validation` |
| Diverse opinions | `diverse` | breadth of perspectives — overlooked risks, good signs to preserve, ideas | `developer_insight` |

- **Default = all three.** The `--testers` budget is split across them (roughly even, min 1 per track).
- **Pick a subset** (positional `edge`, or `--track edge,diverse`) to put the whole budget into those tracks so each runs deeper. Selecting one track is how you say "this run is only about X."

### Best-case path

The optimistic reference path for the change.

It answers:

- Who benefits most?
- What should the smooth journey look like?
- What value is delivered?
- What evidence would prove this path is real?

### Recruited testers

Synthetic beta testers are selected from release risk, not generated as generic personas.

Examples:

- first-time user
- returning maintainer
- docs-only evaluator
- CI integrator
- API consumer
- security reviewer
- impatient power user

### Activity roles

Each tester gets one concrete activity with one role:

| Role | Purpose |
|---|---|
| `happy_path_validation` | Validate the best-case path from a realistic user context. |
| `edge_case_probe` | Find plausible confusion, misuse, docs gaps, integration friction, or environment mismatch. |
| `developer_insight` | Surface good signs, missed risks, and ideas that help the developer think better about the change. |

Each role maps **1:1 to a track** (`happy`→`happy_path_validation`, `edge`→`edge_case_probe`, `diverse`→`developer_insight`). A tester never mixes roles.

### Bounded context

Testers do not start omniscient. They only see what their real-world segment would plausibly see.

Examples:

- README only
- docs + examples only
- changed files + one-hop callers
- CI config only
- public API docs only
- internals allowed only for security/deep technical testers

Hidden knowledge required for success is treated as a docs/UX gap, not a pass.

### Evidence

Every FAIL or UNCLEAR finding needs:

- steps attempted
- expected vs actual
- evidence quote, file line, command output, or missing-context note
- severity
- reproducibility
- recommendation

### 3-stage reporting

Progress is surfaced as three separate messages, not one dump at the end:

1. **Plan** — before any tester runs: target diff, active tracks, tester roster, best-case path. (`--dry-run` stops here.)
2. **Running** — a short status while testers are in flight, grouped by track.
3. **Results** — per track. Each failing finding leads with **Reproduction → Expected vs Actual**, then severity/evidence/recommendation. A clean run still gets a real Stage 3 (per-track summary + good signs), not an empty message.

### Closure-only rerun & convergence

When a fix is applied (`--fix` or approval), the rerun is **targeted**: it re-tests only the criteria that originally failed, answering one question — *"did the fix close this?"* It does **not** re-review the whole surface.

Beta testing a fixed surface will always surface new, smaller observations — that does not mean the work is unfinished. New findings go to a **next-round backlog**, not back into the fix loop. The run exits when every original in-scope finding is closed, even if the backlog is non-empty. This keeps each session bounded instead of nitpicking forever.

## Repository layout

```text
.claude/
  commands/betatest.md          # /betatest slash command
  skills/betatest-harness.md    # beta run methodology
examples/
  persona-schema.json            # recruited tester schema
  rubric-schema.json             # activity/rubric schema
  session-report-schema.json     # full run data model
docs/
  architecture.md
  mvp-plan.md
  beta-sessions/
  fix-proposals/
```

## Status

Working harness. The primary artifact is the Claude Code command/skill pair (`/betatest` + `betatest-harness`). It supports three beta tracks, 3-stage reporting, and a closure-only rerun with a convergence rule. The design principle is **useful harness first**: borrow good patterns from real user testing, then adapt them to code diffs, vibe-coding, and release preparation.

The harness is developed by dogfooding it on its own changes — most features above were found and refined by running `/betatest` against its own diffs.
