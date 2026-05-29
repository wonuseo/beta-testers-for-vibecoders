# beta-testers-for-vibecoders

A Claude Code-native harness that runs a synthetic beta program on your code changes before you ship — no external services, no CI pipeline, no test scripts required. Just a slash command.

## The problem

Vibecoders ship fast. Real users are diverse. The gap between "it works on my machine" and "it works for my users" is where bugs live.

Traditional tests catch what you predicted. Real beta programs catch what you did not: onboarding confusion, docs gaps, broken assumptions, integration friction, and user-specific edge cases.

## How it works

`/code-beta` is a two-layer beta program manager:

1. **Setup / recruitment**
   - reads the diff with bounded context
   - chooses what should be beta-tested
   - selects beta type: focused, technical, closed, open-like breadth, docs/marketing, staged
   - recruits quota-based synthetic beta testers
   - assigns model policy: Opus planner/recruiter, Haiku/Sonnet testers, Sonnet triage
   - writes `.harness/code-beta.config.json`

2. **Run / evidence loop**
   - gives each tester a concrete beta activity, not a vague review prompt
   - constrains context to what that user would realistically know
   - collects steps, expected vs actual, severity, reproducibility, and recommendations
   - dedupes and triages findings
   - escalates high-risk Haiku findings to Sonnet
   - proposes or applies fixes and reruns failed testers

The loop mirrors real beta operations:

```text
plan beta → recruit testers → assign activities → collect evidence → triage → fix/accept/defer → rerun → exit decision
```

## Quickstart

**Copy the harness into your project:**

```bash
# from this repo
mkdir -p /your-project/.claude/commands /your-project/.claude/skills
cp .claude/commands/code-beta.md        /your-project/.claude/commands/code-beta.md
cp .claude/skills/code-beta-harness.md  /your-project/.claude/skills/code-beta-harness.md
```

**Open a new Claude Code session, then run:**

```text
/code-beta setup --diff HEAD~1..HEAD --testers 6
/code-beta run --diff HEAD~1..HEAD
```

Or one-shot:

```text
/code-beta --diff HEAD~1..HEAD --testers 6 --planner-model opus --tester-model haiku --escalation-model sonnet
```

Useful options:

```text
/code-beta setup --focus onboarding --testers 4 --mix cheap
/code-beta setup --focus security --testers 6 --mix deep --tester-model sonnet
/code-beta run --fix
/code-beta --dry-run                 # setup artifacts only
```

> **Important:** Claude Code only discovers command files at session start. If you copied the files into an already-open session, `/code-beta` won't be available until you start a new session.

## Troubleshooting

**`/code-beta` shows as unknown command**

Claude Code registers slash commands from `.claude/commands/` only when a session starts. If you added the file mid-session, you have two options:

**Option 1 — Restart the session** (recommended): Close and reopen Claude Code in your project directory. `/code-beta` will be available immediately.

**Option 2 — Invoke manually without restarting**: Paste this into the Claude Code chat:

```text
Read .claude/commands/code-beta.md and .claude/skills/code-beta-harness.md, then follow the workflow defined in code-beta.md on the diff HEAD~1..HEAD.
```

The manual fallback works because the command file is plain instructions that Claude can read and follow directly, even without slash command registration. This repo's own first beta test was run this way.

## Example session

```text
/code-beta setup --diff HEAD~1..HEAD --testers 6 --mix balanced

Beta plan:
  Objective: Validate first-time and integration users can adopt the changed auth flow safely.
  Beta type: focused technical closed beta
  Surfaces: README onboarding, LoginForm, session refresh API, CI docs
  Exit criteria: no unresolved P0/P1, all high-risk findings confirmed by Sonnet

Recruited testers:
  • first-time-user-1       haiku   public-docs-only-first
  • returning-user          haiku   app-flow context
  • mobile-safari-user      haiku   UI docs + changed component
  • api-consumer            haiku   API docs + changed endpoint
  • ci-integrator           haiku   docs/config only
  • security-reviewer       sonnet  auth/session internals allowed

/code-beta run --diff HEAD~1..HEAD

Running beta activities...
  ✓ first-time-user-1       3/3
  ✗ returning-user          2/3   P1 session refresh race unclear
  ✓ mobile-safari-user      3/3
  ✗ api-consumer            1/3   P1 401 body missing error_code
  ✓ ci-integrator           2/2
  ✓ security-reviewer       4/4

Escalation:
  returning-user P1 finding escalated to Sonnet → confirmed UNCLEAR, needs runtime verification

Triage:
  P1 fix now: api-consumer error_code response
  P1 accepted known issue: session refresh requires runtime test

Report → docs/beta-sessions/2026-05-29-14-32.md
```

## Philosophy

- **Beta-program first** — plan, recruit, assign activities, collect evidence, triage, rerun
- **No external services** — runs entirely inside Claude Code via subagents
- **Realistic context** — testers start with what real users would know, not full repo omniscience
- **Diff-scoped** — focuses on changed surfaces and explicit release risks
- **Evidence-based** — every blocker needs steps, expected vs actual, severity, reproducibility, and recommendation
- **Model-aware** — Opus for planning/recruiting, Haiku/Sonnet for testers, Sonnet for triage

## Repository layout

```text
.claude/
  commands/code-beta.md          # /code-beta slash command (copy this to your project)
  skills/code-beta-harness.md    # beta program methodology (copy this to your project)
docs/
  architecture.md                # system design and component map
  mvp-plan.md                    # phased build roadmap
examples/
  persona-schema.json            # tester/persona object definition
  rubric-schema.json             # rubric + criterion definition
  session-report-schema.json     # full run output schema
  sample-run.md                  # annotated example session
```

## Status

Early scaffold. See `docs/mvp-plan.md` for what's built and what's next.
