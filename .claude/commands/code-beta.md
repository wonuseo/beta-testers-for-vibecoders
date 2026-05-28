# /code-beta — Persona Beta Testing Harness

Run persona-driven beta testing on a code diff from inside Claude Code. No external services. No test scripts. Evidence comes from subagents roleplaying real user archetypes against the changed code.

## Usage

```
/code-beta [options]

Options:
  --diff <range>       Git range to test (default: staged changes)
  --fix                Apply fix proposals and rerun failing personas
  --personas custom    Load personas from .code-beta/personas.json instead of inferring
  --scope <label>      Narrow persona inference to a named feature area
  --dry-run            Show personas + rubric only, do not run subagents
```

## Workflow

Follow these steps in order. Use the `code-beta-harness` skill for the methodology behind each step.

---

### Step 1 — Resolve the diff

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

Summarize: how many files changed, which areas of the codebase (auth, UI, API, data, config…), what kind of change (new feature, bug fix, refactor, config).

---

### Step 2 — Understand the codebase context

Read enough of the repo to answer:
- What kind of app is this? (web app, CLI, library, API, mobile…)
- What tech stack? (language, framework, DB, deployment target)
- Who are the likely end users? (developers, consumers, admins, bots…)
- What is the changed surface? (what does the diff expose to users?)

Read: `README.md`, `package.json`/`pyproject.toml`/`Cargo.toml` (whichever exists), any `CLAUDE.md`, and the changed files themselves.

Do not read the entire codebase. 5–10 files is enough.

---

### Step 3 — Infer personas

If `--personas custom` was given and `.code-beta/personas.json` exists, load personas from that file.

Otherwise, infer 3–6 personas using the methodology in `code-beta-harness.md` (section: Persona Inference). Each persona must be:
- Specific to this codebase (not generic archetypes)
- Relevant to the changed surface
- Distinct from each other (different entry points, goals, or constraints)

Output persona list as a numbered summary before proceeding.

---

### Step 4 — Generate rubrics

For each persona, generate 2–5 acceptance criteria using the methodology in `code-beta-harness.md` (section: Rubric Generation).

Each criterion must be:
- Specific to this diff (not general quality statements)
- Falsifiable from the persona's perspective
- Actionable to test by reading code + reasoning about behavior

Print the rubric table before running subagents.

If `--dry-run` was given, stop here.

---

### Step 5 — Run persona subagents

Spawn one Claude Code subagent per persona using the `Agent` tool. Each subagent receives:

```
You are testing a code change as a specific user persona.

PERSONA: <name>
<persona description>

CODEBASE CONTEXT:
<2-3 sentence summary from Step 2>

CHANGED FILES:
<list of changed files with brief description of change>

RUBRIC (your criteria to evaluate):
<numbered criteria list for this persona>

YOUR TASK:
1. Read the changed files relevant to your persona's experience.
2. Reason through each rubric criterion: would it pass or fail based on the code?
3. For each FAIL: describe exactly what breaks, quote the relevant code, write reproduction steps.
4. For each PASS: one sentence explaining why it passes.
5. Output a structured evidence report (see format below).

EVIDENCE REPORT FORMAT:
## Persona: <name>
### Attempt summary
<what you tried to do as this persona>

### Results
| Criterion | Result | Evidence |
|-----------|--------|----------|
| <criterion text> | PASS/FAIL | <one-line evidence> |

### Failures (detail)
For each FAIL:
**Criterion:** <text>
**What breaks:** <description>
**Relevant code:** <file:line — quote>
**Reproduction:** <step-by-step>

### Overall score: <X>/<Y> criteria passing
```

Run all persona subagents. Collect their evidence reports.

---

### Step 6 — Score and triage

Aggregate results:
- Per-persona: X/Y criteria passing
- Overall: total passing / total criteria
- List all failing criteria with their persona

Print the score summary table.

If all criteria pass, print a success message and write the session report (Step 8). Done.

---

### Step 7 — Fix proposals (if gap > 0)

For each failing criterion, propose a fix using the methodology in `code-beta-harness.md` (section: Fix Proposals).

A fix proposal must:
- Reference the specific failing criterion and persona
- Identify the exact file and line range to change
- Describe the change in plain English
- Include a code snippet showing the proposed change
- Be minimal (only fix what is failing, nothing else)

Group fixes by file. If two failing criteria require changes to the same file, combine them into one proposal.

Write proposals to `docs/fix-proposals/<YYYY-MM-DD>-<slug>.md` in the repo being tested.

Print a summary of proposed fixes.

If `--fix` was NOT given, ask the user: "Apply fixes? (y/n)". Stop if they say no; write partial session report noting fixes were proposed but not applied.

---

### Step 8 — Apply fixes and rerun (if --fix or user said yes)

Apply each fix proposal by editing the relevant files.

After applying all fixes, rerun the subagents only for the personas that had failures (Step 5 format, same rubric).

Collect new evidence. Recompute scores.

If gap is now 0: report success with gap closure.
If gap > 0 after one rerun: report remaining failures and stop. Do not loop more than once automatically — surface remaining failures for human review.

---

### Step 9 — Write session report

Write a session report to `docs/beta-sessions/<YYYY-MM-DD>-<HH-MM>.md` containing:

```markdown
# Beta Session: <date> <time>

## Diff summary
<files changed, areas affected, change type>

## Personas tested
<numbered list with descriptions>

## Rubric
<full criteria table>

## Results (initial run)
<score table>

## Fix proposals
<link to fix-proposals file, or "none">

## Results (after fixes)
<score table, or "fixes not applied">

## Gap closure
<before: N/M → after: N/M, or "not closed">

## Evidence
<per-persona evidence reports>
```

Print: "Session report written to docs/beta-sessions/<filename>"
