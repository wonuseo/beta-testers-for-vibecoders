# beta-testers-for-vibecoders

A Claude Code-native harness that beta-tests your code changes before you ship — no external services, no CI pipeline, no test scripts. Just a slash command.

## The problem

Vibecoders ship fast. Real users are diverse. The gap between "it works on my machine" and "it works for my users" is where bugs live.

Traditional testing catches what you predicted. This catches what you didn't.

## How it works

Run `/code-beta` on any diff. The harness:

1. **Reads your diff** — understands what changed and in what codebase context
2. **Infers personas** — who actually uses this feature (new users, power users, mobile users, API consumers…)
3. **Generates rubrics** — specific acceptance criteria per persona for the changed surface
4. **Spawns subagents** — each one roleplays a persona and attempts to use the changed feature
5. **Collects evidence** — what worked, what broke, friction, errors, reproduction steps
6. **Scores results** — pass/fail per rubric criterion per persona
7. **Proposes or applies fixes** — `--fix` auto-patches failures and reruns
8. **Reports gap closure** — before/after breakdown with full audit trail

## Quickstart

**Copy the harness into your project:**

```bash
# from this repo
cp .claude/commands/code-beta.md   /your-project/.claude/commands/code-beta.md
cp .claude/skills/code-beta-harness.md  /your-project/.claude/skills/code-beta-harness.md
```

**Run it inside Claude Code on your project:**

```
/code-beta                         # test current staged diff
/code-beta --diff HEAD~1..HEAD     # test last commit
/code-beta --fix                   # test + auto-patch failures
/code-beta --personas custom       # use .code-beta/personas.json instead of inferring
/code-beta --scope auth            # narrow to a named feature area
```

## Example session

```
/code-beta --diff HEAD~1..HEAD

Diff: 3 files changed (auth/login.ts, ui/LoginForm.tsx, api/session.ts)
Codebase: Next.js SaaS app, email+password auth, REST API, mobile-responsive

Inferred personas (4):
  • new-user          First login, no account yet, on desktop
  • returning-user    Has saved session, coming back after token expiry
  • mobile-user       Completing signup flow on iPhone Safari
  • api-consumer      Integrating auth via REST, no UI

Generated rubric: 12 criteria across 4 personas

Running persona subagents...
  ✓ new-user          8/8   all criteria passed
  ✗ returning-user    5/8   3 failing  (session refresh race condition)
  ✓ mobile-user       6/6   all criteria passed
  ✗ api-consumer      2/4   2 failing  (401 body missing error_code field)

Gap: 5 / 12 criteria failing

Fix proposals:
  [1] session.ts:47 — refresh lock prevents double-trigger (returning-user)
  [2] api/session.ts:112 — add error_code to 401 response body (api-consumer)

Apply fixes? (y/n): y
Fixes applied → docs/fix-proposals/2026-05-29-auth-refresh.md

Rerunning failed personas...
  ✓ returning-user    8/8   all criteria passed
  ✓ api-consumer      4/4   all criteria passed

Final: 12/12 passing. Gap closed.
Report → docs/beta-sessions/2026-05-29-14-32.md
```

## Philosophy

- **No external services** — runs entirely inside Claude Code via Agent subagents
- **No test scripts** — evidence comes from agent behavior and code reading, not assertions
- **Persona-driven** — tests what matters to real user archetypes, not just code coverage
- **Diff-scoped** — only exercises the changed surface, not the entire app
- **Fix-aware** — patches are grounded in specific failure evidence, not guesses

## Repository layout

```
.claude/
  commands/code-beta.md          # /code-beta slash command (copy this to your project)
  skills/code-beta-harness.md    # persona methodology (copy this to your project)
docs/
  architecture.md                # system design and component map
  mvp-plan.md                    # phased build roadmap
examples/
  persona-schema.json            # persona object definition
  rubric-schema.json             # rubric + criterion definition
  session-report-schema.json     # full run output schema
  sample-run.md                  # annotated example session
```

## Status

Early scaffold. See `docs/mvp-plan.md` for what's built and what's next.
