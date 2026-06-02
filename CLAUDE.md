# beta-testers-for-vibecoders

You are working on the harness itself, not on a target project. The harness is a set of Claude Code slash commands and skills designed to be **copied into other repos** and run there.

## What this project produces

Two files that target repos install:

| File | Role |
|------|------|
| `.claude/commands/betatest.md` | The `/betatest` slash command — orchestration logic |
| `.claude/skills/betatest-harness.md` | Synthetic beta program methodology — referenced by the command |

Everything else in this repo (docs, examples, schemas) supports development of those two files.

## Core concepts

**Beta plan** — the setup-layer artifact that states objective, beta type, changed surfaces, out-of-scope areas, evidence schema, and exit criteria.

**Tester recruitment** — quota-based selection of synthetic tester segments. Each tester has a model policy, screening criteria, prior-knowledge boundary, context policy, and assigned activity.

**Tester** — a recruited beta tester segment, not an omniscient code reviewer. A tester may be a first-time user, maintainer, CI integrator, API consumer, security reviewer, etc. See `examples/persona-schema.json` until it is renamed.

**Beta activity** — the concrete task assigned to a tester. Example: “Start from README and try to run `/betatest` on HEAD~1..HEAD.” Activities replace vague “review this diff” prompts.

**Rubric** — acceptance criteria generated for a (tester × activity × diff) tuple. Each criterion is specific, falsifiable, and evaluated from that tester’s context policy. See `examples/rubric-schema.json`.

**Evidence** — structured prose from a tester run. Includes: context used, steps attempted, expected vs actual, severity, reproducibility, and recommendation.

**Gap** — unresolved FAIL/UNCLEAR criteria after triage. Gap closure = reducing unresolved blocker findings to zero or accepted known issues.

**Fix proposal** — a concrete, file-level patch addressing one or more failing criteria. Grounded in evidence. Written to `docs/fix-proposals/` in the target repo.

**Session report** — the full audit trail of a `/betatest` run: target diff, beta plan, tester recruitment, activities, evidence, triage, fixes, and gap closure. Written to `docs/beta-sessions/` in the target repo.

## Development conventions

- The command file (`.claude/commands/betatest.md`) is a workflow orchestrator. It should read like a recipe: steps, decisions, tool calls. Keep it action-oriented.
- The skill file (`.claude/skills/betatest-harness.md`) is a methodology reference. It defines *how* to design a beta plan, recruit testers, create activities, evaluate evidence, and triage gaps. The command file delegates to it.
- Do not put detailed methodology in the command file or orchestration in the skill file.
- Schemas in `examples/` use JSON with `$comment` fields. Keep them honest — if the command produces output, the schema should match.
- All changes to the two distributable files should be tested by manually copying them into a sample project and running `/betatest` there.

## Testing this harness

**Reference scenario (recommended baseline):** Use this repo's own scaffold diff. It is a known test case with a written expected output in `examples/sample-run.md` and session reports under `docs/beta-sessions/`. Any change to the command or skill files should be validated against this scenario first.

```bash
# From this repo, run the harness on the scaffold commit range
# (manual invocation — paste into Claude Code chat):
# "Read .claude/commands/betatest.md and .claude/skills/betatest-harness.md,
#  then follow the workflow on the diff HEAD~2..HEAD."
```

**Steps:**

1. Make your change to `.claude/commands/betatest.md` or `.claude/skills/betatest-harness.md`.
2. Use the **manual invocation fallback** (see README Troubleshooting) — paste the read+follow instruction into Claude Code chat with diff `HEAD~2..HEAD`. This avoids needing to restart your session.
3. Compare: did setup produce a concrete beta objective/type? Did recruitment produce a quota-based tester pool with realistic context policies? Were activities concrete and evidence-based?
4. Check the session report written to `docs/beta-sessions/` — compare gap closure to the baseline session.
5. If results diverge significantly from baseline, your change may have regressed beta-plan/recruitment quality. Investigate before committing.

**Quality reference:** Use `examples/sample-run.md` as the template for what good output looks like. Once a real session exists at `docs/beta-sessions/2026-05-29-*.md`, that becomes the reproducible baseline — compare tester count, activity specificity, evidence quality, and gap result against it.

## What's out of scope (for now)

- Browser automation (no Puppeteer/Playwright dependency)
- External service integrations (Sentry, Linear, Slack)
- Runtime monitoring (this is a pre-push gate, not a live monitor)
- Generating runnable test scripts (evidence-from-behavior is the model here)
- Full repo omniscience (bounded, realistic context is intentional)
- Language-specific plugins (harness is language-agnostic via code reading)
