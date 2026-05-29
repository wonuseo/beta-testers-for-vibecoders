# beta-testers-for-vibecoders

A Claude Code-native harness that turns a code diff into a useful synthetic beta run: best-case path, persona edge cases, missed risks, good signs, and concrete next fixes.

No external service. No CI pipeline. No test script required. Just copy two Claude files into a repo and run `/code-beta`.

## Why this exists

Vibecoders ship fast. The risky part is not only “does the code compile?” It is:

- Does the intended happy path actually make sense?
- What would a first-time user misunderstand?
- What edge case would a maintainer, CI integrator, API consumer, or docs-only user hit?
- What risk did the developer probably miss?
- What is surprisingly good and should be preserved?

Traditional tests catch what you predicted. A beta run should also reveal what you did not know to ask.

## What `/code-beta` does

`/code-beta` runs a lightweight synthetic beta program on your current change.

```text
plan beta
→ define best-case path
→ recruit synthetic testers
→ assign happy-path and edge-case activities
→ collect evidence and developer insight
→ triage blockers
→ propose fixes or ship decision
```

The output is not just a bug list. A useful run can end with:

1. **Best-case path** — the ideal user/developer journey this change should enable.
2. **Persona edge cases** — where specific user types get confused, misuse the feature, or hit environment/docs gaps.
3. **Developer insight** — missed risks, good signs, and ideas sparked during the run.
4. **Ship/no-ship verdict** — what to fix now, what to accept, and what to revisit later.

## Quickstart

Copy the harness into your project:

```bash
mkdir -p /your-project/.claude/commands /your-project/.claude/skills
cp .claude/commands/code-beta.md        /your-project/.claude/commands/code-beta.md
cp .claude/skills/code-beta-harness.md  /your-project/.claude/skills/code-beta-harness.md
```

Open a new Claude Code session in the target project, then run:

```text
/code-beta --diff HEAD~1..HEAD --testers 4
```

Or use the two-step flow:

```text
/code-beta setup --diff HEAD~1..HEAD --testers 4
/code-beta run --diff HEAD~1..HEAD
```

Useful variants:

```text
/code-beta setup --focus onboarding --testers 3 --mix cheap
/code-beta setup --focus security --testers 6 --mix deep --tester-model sonnet
/code-beta run --fix
/code-beta --dry-run
```

> Claude Code discovers slash commands only when a session starts. If `/code-beta` is unknown after copying the files, restart Claude Code or use the manual fallback below.

## Manual fallback

If the command was copied into an already-open Claude Code session, paste this into Claude Code:

```text
Read .claude/commands/code-beta.md and .claude/skills/code-beta-harness.md, then follow the workflow defined in code-beta.md on the diff HEAD~1..HEAD.
```

The command is plain markdown instructions, so Claude can follow it directly even before slash-command registration.

## Example output

```text
/code-beta --diff HEAD~1..HEAD --testers 4

Best-case path:
  Primary user: first-time vibe coder adding the harness to a repo
  Happy path:
    1. Copy command + skill files
    2. Restart Claude Code
    3. Run /code-beta on the current diff
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
  - mid-session-user: copies files during an active Claude session and expects /code-beta to appear immediately.
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
## Ship/no-ship verdict
## Top blockers
## Persona edge cases
## Missed risks
## Good signs
## Ideas sparked
## Next fixes
## Evidence appendix
```

The harness should produce value even when no blocker is found. Clean runs still report what worked, what edge cases were considered, and what the developer should preserve.

## Core concepts

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

## Repository layout

```text
.claude/
  commands/code-beta.md          # /code-beta slash command
  skills/code-beta-harness.md    # beta run methodology
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

Early scaffold. The primary artifact is the Claude Code command/skill pair. The current direction is **useful harness first**: copy good patterns from real user testing, then adapt them to code diffs, vibe-coding, and release preparation.
