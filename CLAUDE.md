# beta-testers-for-vibecoders

You are working on the harness itself, not on a target project. The harness is a set of Claude Code slash commands and skills designed to be **copied into other repos** and run there.

## What this project produces

Two files that target repos install:

| File | Role |
|------|------|
| `.claude/commands/code-beta.md` | The `/code-beta` slash command — orchestration logic |
| `.claude/skills/code-beta-harness.md` | Persona + rubric methodology — referenced by the command |

Everything else in this repo (docs, examples, schemas) supports development of those two files.

## Core concepts

**Persona** — a user archetype inferred from the codebase and diff (e.g. `new-user`, `api-consumer`, `mobile-user`). A persona has a name, description, entry point, and behavioral constraints. See `examples/persona-schema.json`.

**Rubric** — acceptance criteria generated for a (persona × diff) pair. Each criterion is a specific, falsifiable statement about what must be true from that persona's perspective. See `examples/rubric-schema.json`.

**Persona subagent** — a Claude Code `Agent` subagent spawned with a persona-scoped prompt. It reads the relevant changed code, roleplays the persona, and produces structured evidence: what it attempted, what succeeded, what failed, and why.

**Evidence** — freeform structured text from a persona run. Includes: attempt log, success list, failure list, reproduction steps, and error excerpts. Not JSON — prose with clear section headers is more reliable from LLM agents.

**Gap** — the count of failing rubric criteria. Gap = 0 means all personas passed. Gap closure = reducing gap from N to 0.

**Fix proposal** — a concrete, file-level patch addressing one or more failing criteria. Grounded in evidence. Written to `docs/fix-proposals/` in the target repo.

**Session report** — the full audit trail of a `/code-beta` run: diff summary, personas, rubric, per-persona evidence, scores, fixes, and gap closure. Written to `docs/beta-sessions/` in the target repo.

## Development conventions

- The command file (`.claude/commands/code-beta.md`) is a workflow orchestrator. It should read like a recipe: steps, decisions, tool calls. Keep it action-oriented.
- The skill file (`.claude/skills/code-beta-harness.md`) is a methodology reference. It defines *how* to do each step (persona inference algorithm, rubric generation rules, scoring criteria). The command file delegates to it.
- Do not put methodology in the command file or orchestration in the skill file.
- Schemas in `examples/` use JSON with `$comment` fields. Keep them honest — if the command produces output, the schema should match.
- All changes to the two distributable files should be tested by manually copying them into a sample project and running `/code-beta` there.

## Testing this harness

**Reference scenario (recommended baseline):** Use this repo's own scaffold diff. It is a known test case with a written expected output in `examples/sample-run.md` and session reports under `docs/beta-sessions/`. Any change to the command or skill files should be validated against this scenario first.

```bash
# From this repo, run the harness on the scaffold commit range
# (manual invocation — paste into Claude Code chat):
# "Read .claude/commands/code-beta.md and .claude/skills/code-beta-harness.md,
#  then follow the workflow on the diff HEAD~2..HEAD."
```

**Steps:**

1. Make your change to `.claude/commands/code-beta.md` or `.claude/skills/code-beta-harness.md`.
2. Use the **manual invocation fallback** (see README Troubleshooting) — paste the read+follow instruction into Claude Code chat with diff `HEAD~2..HEAD`. This avoids needing to restart your session.
3. Compare: did persona inference produce 4 personas similar to the baseline? Were rubrics specific and falsifiable? Did scoring match the expected gaps?
4. Check the session report written to `docs/beta-sessions/` — compare gap closure to the baseline session.
5. If results diverge significantly from baseline, your change may have regressed persona quality. Investigate before committing.

**Quality reference:** Use `examples/sample-run.md` as the template for what good output looks like (correct persona count, specific rubric criteria, code-cited evidence). Once a real session exists at `docs/beta-sessions/2026-05-29-*.md`, that becomes the reproducible baseline — compare persona names, rubric count, and gap result against it.

## What's out of scope (for now)

- Browser automation (no Puppeteer/Playwright dependency)
- External service integrations (Sentry, Linear, Slack)
- Runtime monitoring (this is a pre-push gate, not a live monitor)
- Generating runnable test scripts (evidence-from-behavior is the model here)
- Language-specific plugins (harness is language-agnostic via code reading)
