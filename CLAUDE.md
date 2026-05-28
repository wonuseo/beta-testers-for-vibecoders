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

1. Pick any repo with a recent diff.
2. Copy the current `.claude/commands/code-beta.md` and `.claude/skills/code-beta-harness.md` into that repo's `.claude/` directories.
3. Run `/code-beta --diff HEAD~1..HEAD` in Claude Code from that repo.
4. Evaluate: did persona inference make sense? Were rubrics specific? Was evidence useful? Did scoring feel right?
5. Iterate on the harness files.

## What's out of scope (for now)

- Browser automation (no Puppeteer/Playwright dependency)
- External service integrations (Sentry, Linear, Slack)
- Runtime monitoring (this is a pre-push gate, not a live monitor)
- Generating runnable test scripts (evidence-from-behavior is the model here)
- Language-specific plugins (harness is language-agnostic via code reading)
