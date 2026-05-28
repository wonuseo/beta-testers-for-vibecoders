# code-beta-harness — Persona Beta Testing Methodology

This skill defines the methodology used by `/code-beta`. The command orchestrates; this skill explains *how* to execute each phase correctly.

---

## Persona Inference

**Goal:** Identify 3–6 user archetypes who would interact with the changed code surface, specific to this codebase.

**Algorithm:**

1. **Identify the changed surface.** What does the diff touch from a user perspective?
   - UI component → visual/interaction users
   - API endpoint → API consumers, frontend callers
   - Auth flow → new users, returning users, session-edge cases
   - CLI command → developer users, CI/CD bots
   - Data model / migration → admins, data consumers
   - Config / env → operators, deployers

2. **Read the codebase signals.** Look for:
   - User roles in auth code (admin, member, guest, api_key…)
   - Route patterns that suggest distinct entry points
   - Client types (mobile UA checks, API versioning, iframe embeds)
   - Error handling that implies different caller expectations

3. **Generate personas bottom-up.** Start from the changed surface, not from generic user types. A persona is valid if:
   - It has a specific entry point into the changed feature
   - It has a goal that the changed code either helps or could break
   - It has at least one constraint that distinguishes it (device, permission level, data state, integration method)

4. **Prune duplicates.** Two personas are redundant if they have the same entry point, same goal, and same constraints. Merge or drop one.

5. **Name personas with a hyphenated slug** (`new-user`, `api-consumer`, `returning-mobile-user`). Include a 2–3 sentence description covering: who they are, what they want from this feature, and what state they're in when they hit the changed code.

**Example inference for an auth diff:**

| Signal | Persona |
|--------|---------|
| signup form changed | `new-user` — first time, no account, expects success confirmation |
| session refresh changed | `returning-user` — has active session, token nearing expiry |
| `/api/auth/token` endpoint changed | `api-consumer` — machine client, expects RFC-compliant error bodies |
| iOS Safari UA check in middleware | `mobile-safari-user` — completing flow on iPhone, keyboard/viewport constraints |

---

## Rubric Generation

**Goal:** Produce 2–5 falsifiable, diff-specific acceptance criteria per persona.

**Rules for good criteria:**

- **Specific to the diff, not to the feature generally.** Don't write "user can log in" — write "user sees validation error immediately when email field is blank (not on submit)".
- **Falsifiable from code.** A subagent must be able to read the changed code and determine pass/fail. Avoid UI aesthetics or performance without concrete thresholds.
- **Written from the persona's perspective.** "The API consumer receives a JSON body with `error_code` on 401" not "the server returns an error code".
- **Scoped to what the diff could break.** If the diff doesn't touch error handling, don't write a criterion about error messages (unless it changes error handling).

**Criterion format:**
```
[Persona] [action] [observable outcome]
```

Examples:
- `new-user submits valid credentials → redirected to /dashboard within one redirect`
- `returning-user's expired token → refresh happens transparently without forcing re-login`
- `api-consumer calls POST /auth/token with wrong password → 401 with JSON body { error: string, error_code: string }`
- `mobile-safari-user taps Login → submit button remains visible above keyboard`

**Rubric quality check:**
- At least one criterion should be likely to fail (test the actual change)
- No criterion should duplicate another across personas
- Each criterion should be completable in under 2 minutes of code reading by an agent

---

## Persona Subagent Prompting

**Goal:** Get reliable, structured evidence from each subagent.

**Key prompting principles:**

1. **Give the persona a specific goal**, not just a description. "You want to log in after your session expired" is better than "you are a returning user".

2. **Constrain the subagent's reading scope.** Tell it which files to read. Don't let it wander — wandering produces vague evidence.

3. **Require evidence quotes.** The subagent must cite file:line for each FAIL. Unsupported failures are not useful.

4. **Require reproduction steps for FAILs.** This makes fix proposals possible.

5. **Prohibit speculation about runtime behavior** unless the code makes it deterministic. If the subagent can't tell from static reading, it should say "UNCLEAR — requires runtime verification" and explain why.

**Subagent scope** (what to read):
- The changed files (from the diff)
- Any direct callers of changed functions (one hop)
- Relevant type definitions / interfaces
- Do NOT read test files (they test the old behavior)

---

## Evidence Evaluation

**Scoring:**
- Each criterion is either PASS, FAIL, or UNCLEAR.
- UNCLEAR counts as FAIL for gap calculation (it means we don't know it works).
- Score = PASS count / total criteria count.
- A persona "passes" if all its criteria are PASS.

**Quality filters:**
- Reject evidence that cites no code (vague agent output). Re-run the subagent with tighter instructions.
- Reject FAILs with no reproduction steps. Ask the subagent to elaborate.
- Accept UNCLEAR only if the subagent explains the specific runtime dependency that prevents static verification.

---

## Fix Proposals

**Goal:** Produce a minimal, targeted patch for each failing criterion, grounded in the evidence.

**Fix proposal rules:**

1. **One fix per criterion** (unless two criteria have the exact same root cause — then one fix covers both, say so).

2. **Minimal surface area.** Fix only the lines that cause the failure. Don't refactor, don't add features, don't fix adjacent issues.

3. **Reference the evidence.** The fix proposal must name the criterion, the persona, and quote the failing code.

4. **Show before/after.** Use a diff-style block:
   ```diff
   - old code line
   + new code line
   ```

5. **Explain why the fix works** in one sentence. Don't just show the code change.

6. **Verify the fix doesn't break other personas.** Check the other personas' passing criteria against the proposed change. If a fix to one criterion could break another persona's passing criterion, flag it explicitly — don't apply silently.

**Fix proposal file format** (`docs/fix-proposals/YYYY-MM-DD-<slug>.md`):

```markdown
# Fix Proposal: <slug>
Date: <date>
Session: <session report filename>

## Failing criteria addressed
- [persona] criterion text

## Fix 1: <file>

**Why this fixes it:** <one sentence>

**Change:**
\```diff
- old line
+ new line
\```

## Potential side effects
<list any concerns, or "none identified">
```

---

## Gap Closure Reporting

**Goal:** Give the user a clear before/after view and confidence level.

**Report structure:**

```
Initial gap:   N failing / M total criteria
After fixes:   X failing / M total criteria
Gap closed:    yes / no / partial

Persona breakdown:
  ✓ persona-name    N/N  (initial: N/N)
  ✓ persona-name    N/N  (initial: X/N, fixed)
  ✗ persona-name    X/N  (still failing — see evidence)
```

**If gap is not fully closed after one rerun:**
- List the still-failing criteria with their evidence excerpts
- Do NOT auto-iterate further — surface to human
- Write a clear "requires human review" section in the session report

**Confidence levels:**
- **High confidence**: static code reading was sufficient to determine all PASSes and FAILs
- **Medium confidence**: some criteria were UNCLEAR (runtime-dependent); PASSes are likely but not certain
- **Low confidence**: diff was too large or too abstracted to evaluate meaningfully; recommend narrowing scope with `--scope`
