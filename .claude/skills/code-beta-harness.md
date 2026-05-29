# code-beta-harness — Synthetic Beta Program Methodology

This skill defines the methodology used by `/code-beta`. The command orchestrates; this skill explains *how* to design and run a synthetic beta program correctly.

The core loop is:

```text
plan beta → define best-case path → recruit testers → assign happy-path and edge-case activities → collect evidence and developer insight → triage → fix/accept/defer → rerun → exit decision
```

---

## Beta Program Setup

**Goal:** Decide what should be beta-tested before creating testers.

Do not start from generic personas. Start from the release risk.

### Inputs to inspect

Use bounded context:
- diff and diff stat
- README / public docs / examples
- package/config/entrypoints
- changed files
- one-hop callers only when necessary
- test commands only to understand available verification

Do not read the entire repo. A beta tester is not omniscient, and large projects do not fit in context.

### Beta plan fields

A valid beta plan includes:

```json
{
  "objective": "What hypothesis or release risk this beta validates",
  "best_case_path": {
    "primary_user": "Who benefits most if this works",
    "happy_path": ["Step 1", "Step 2", "Step 3"],
    "value_delivered": "What becomes easier, safer, or faster",
    "success_signal": "What evidence would prove the happy path is real"
  },
  "beta_type": "focused-technical-closed",
  "target_diff": "HEAD~1..HEAD",
  "surfaces": ["CLI onboarding", "slash command registration", "manual fallback"],
  "out_of_scope": ["browser automation", "full CI integration"],
  "primary_risks": ["new users cannot discover command", "CI examples imply false gating"],
  "activities": ["first run from README", "mid-session fallback", "CI docs review"],
  "evidence_schema": ["steps", "expected", "actual", "severity", "repro", "recommendation"],
  "exit_criteria": ["no unresolved P0/P1", "all failures have evidence", "high-risk findings confirmed by Sonnet"]
}
```

### Good beta objectives

Good objectives are concrete:
- "Validate first-time users can install and invoke the command from README without hidden context."
- "Validate API consumers can adapt to the changed response shape without breaking existing integrations."
- "Validate CI integrators are not misled about exit-code behavior."

Bad objectives are vague:
- "Check quality."
- "Find bugs."
- "Review the code."

### Best-case path definition

Every beta plan must define the optimistic path before hunting for edge cases. This keeps the harness useful even when no bug is found.

A strong best-case path answers:
- **Primary user:** who benefits most if the change works?
- **Happy path:** what 3–7 steps should feel smooth?
- **Value delivered:** what becomes easier, safer, faster, clearer, or more reliable?
- **Success signal:** what would convince the developer this path is real, not just imagined?

Good examples:
- "A first-time vibe coder copies the harness, starts a new Claude Code session, runs `/code-beta`, and receives a concise ship/no-ship report without reading internals."
- "A maintainer reads a generated beta report and immediately sees one docs gap, one non-blocking edge case, and one good design choice to preserve."

The best-case path is not a sales pitch. It is the reference path that testers validate, bend, and challenge.

---

## Beta Type Selection

Choose one or more beta types:

| Beta type | Use when | Synthetic translation |
|---|---|---|
| Closed beta | Targeted feedback from a selected group | quota-based tester pool |
| Open-like beta | Need broad, cheap coverage | many Haiku testers with shallow activities |
| Technical beta | Security/API/performance/reliability risk | Sonnet technical testers and stricter evidence |
| Focused beta | Specific feature or changed surface | narrow activities tied to diff surfaces |
| Marketing/docs beta | Messaging, docs, onboarding, positioning risk | public-docs-only testers |
| Post-release/staged beta | Rollout/release gating risk | staged tester waves and exit criteria |

Most code harness runs should default to **focused technical closed beta**:
- focused because the diff defines the surface
- technical because code changes have correctness/security/runtime risk
- closed because the tester pool is intentionally selected

---

## Tester Recruitment

**Goal:** Recruit a quota-based synthetic tester pool from the beta plan.

### Recruitment rules

1. **Recruit segments, not stereotypes.** A tester is valid only if tied to a changed surface and release risk.
2. **Set quota intentionally.** Each segment needs a reason for how many testers it gets.
3. **Assign model intentionally.** Use Haiku for breadth and Sonnet for high-risk/deep validation.
4. **Define prior knowledge boundary.** Real beta testers do not know internal implementation unless their segment would.
5. **Define context policy.** State what they can read first and what requires progressive disclosure.

### Model policy

Default:

```json
{
  "planner_model": "opus",
  "recruiter_model": "opus",
  "default_tester_model": "haiku",
  "escalation_model": "sonnet",
  "aggregator_model": "sonnet"
}
```

Recommended use:
- Opus: planner/recruiter only — beta design quality matters most here.
- Haiku: cheap breadth testers.
- Sonnet: high-risk testers, escalation, aggregation, fix triage.

If the runtime cannot enforce model selection per subagent, record intended model in the artifacts and continue.

### Recruitment object

```json
{
  "id": "first-time-cli-user",
  "segment": "First-time user installing and running the changed CLI command",
  "quota_group": "onboarding",
  "model": "haiku",
  "screening_criteria": [
    "has not seen internal design docs",
    "starts from README",
    "does not know session-start command discovery caveat"
  ],
  "context_policy": "public-docs-only-first",
  "allowed_context": ["README.md", "examples/"],
  "forbidden_initial_context": ["internal implementation files", "prior beta session reports"],
  "success_condition": "Can complete the target flow without hidden internal knowledge"
}
```

### Quota guidance

| Mix | Tester count | Suggested split |
|---|---:|---|
| cheap | 4–8 | mostly Haiku, one Sonnet escalation slot |
| balanced | 5–8 | Haiku breadth + 1–2 Sonnet high-risk testers |
| deep | 6–10 | more Sonnet technical/security/API testers |

---

## Beta Activity Design

**Goal:** Give each tester a concrete activity, not a vague review prompt.

A beta activity is a task that mirrors real beta testing:
- starts from a plausible entry point
- constrains available context
- asks for real evidence
- produces feedback that can be triaged

### Activity schema

```json
{
  "activity_id": "install-and-run-command",
  "tester_id": "first-time-cli-user",
  "role": "happy_path_validation",
  "task": "Start from README, copy the harness into a project, then attempt to run /code-beta on HEAD~1..HEAD.",
  "starting_context": ["README.md"],
  "allowed_context": ["README.md", "examples/", ".claude/commands/code-beta.md only if discovered through docs"],
  "forbidden_context": ["internal design notes", "previous session reports"],
  "commands": ["git diff --stat", "git status"],
  "evidence_required": ["steps attempted", "expected", "actual", "blocking error", "severity", "repro", "recommendation"]
}
```

### Activity quality checks

A good activity:
- can fail for reasons a real user would experience
- has an observable completion condition
- fits the tester's prior knowledge and context policy
- is tied to a changed surface
- has a clear role: `happy_path_validation`, `edge_case_probe`, or `developer_insight`

A bad activity:
- asks the tester to inspect everything
- depends on hidden implementation knowledge
- asks for a generic code review
- has no reproducible evidence requirement

### Activity roles

Use a mix of activity roles so the harness is not only a bug-finder:

| Role | Purpose | Example |
|---|---|---|
| `happy_path_validation` | Confirm the best-case path is coherent and valuable | first-time user follows README and gets a useful report |
| `edge_case_probe` | Find persona-specific confusion, misuse, environment mismatch, or docs gaps | CI integrator checks whether examples imply unsupported exit-code behavior |
| `developer_insight` | Surface good signs, product ideas, and risks the developer may have missed | maintainer evaluates whether the report format creates actionable next steps |

At least one tester should validate the best-case path. At least one tester should probe edge cases. For small `--testers 3` runs, the third tester can be either deep technical risk or developer insight depending on the diff.

---

## Rubric Generation

**Goal:** Produce 2–5 falsifiable, diff-specific acceptance criteria per tester/activity.

**Rules for good criteria:**

- **Specific to the diff, not to the feature generally.** Don't write "user can log in" — write "user sees validation error immediately when email field is blank (not on submit)".
- **Falsifiable from the tester's allowed context.** If the tester would need hidden internals, the criterion should surface a docs/UX gap or request progressive disclosure.
- **Written from the tester's perspective.** "The CI integrator can tell whether the YAML actually fails on beta gaps" not "the docs mention CI".
- **Scoped to what the diff could break.** If the diff doesn't touch error handling, don't write a criterion about error messages unless docs claim it.

**Criterion format:**

```text
[Tester] [activity/action] → [observable outcome]
```

Examples:
- `first-time-cli-user follows README quickstart → can tell a new Claude Code session is required before /code-beta appears`
- `mid-session-user sees Unknown command → can recover using documented manual fallback without restarting`
- `ci-integrator reads YAML example → understands it is report-only until exit-code support ships`
- `api-consumer calls POST /auth/token with wrong password → receives 401 with JSON body { error, error_code }`

**Rubric quality check:**
- At least one criterion should test a plausible failure mode.
- No criterion should duplicate another across testers.
- Each criterion should be completable within the tester's context budget.

---

## Tester Subagent Prompting

**Goal:** Get reliable, structured evidence from each tester subagent.

Key prompting principles:

1. **Make the tester bounded.** They are not a general reviewer. They follow the assigned activity and context policy.
2. **Start with realistic context.** Public-docs users should not inspect internals unless blocked and justified.
3. **Use progressive disclosure.** If blocked, the tester asks for the smallest additional context needed and explains why.
4. **Treat hidden knowledge as a gap.** If success requires undocumented internal behavior, record docs/UX failure.
5. **Require evidence.** FAIL/UNCLEAR requires file:line, docs quote, command output, or explicit missing-context note.
6. **Require reproduction.** Failures without reproduction are not actionable.
7. **No speculation.** If runtime behavior is not determinable, mark UNCLEAR and explain the missing condition.
8. **Always return insight.** Even if everything passes, report one edge case, missed risk, good sign, or idea. If none exists, explicitly say `none found` and why.

### Subagent scope

Allowed by default:
- assigned starting context
- changed files relevant to activity
- one-hop callers only when the activity justifies it
- config/docs/examples relevant to the tester

Forbidden by default:
- entire repo scan
- unrelated tests/docs
- prior beta session reports unless the activity is maintainer baseline comparison

---

## Developer Insight Capture

**Goal:** Make every run useful to the developer, even when no blocker appears.

Each tester report should include:

```json
{
  "persona_specific_edge_case": "A plausible edge case this tester would hit, or none found",
  "missed_risk": "A risk the developer may not have considered, or none found",
  "good_sign": "A robust or valuable aspect worth preserving",
  "idea_sparked": "Optional improvement idea discovered during the activity"
}
```

Guidelines:
- Keep insights grounded in the activity and allowed context.
- Do not turn `idea_sparked` into scope creep; mark it as optional unless it blocks the best-case path.
- `good_sign` matters. Developers need to know what not to break in the next iteration.
- Persona-specific edge cases can be side conclusions, not necessarily release blockers.

---

## Evidence Evaluation

**Scoring:**
- Each criterion is PASS, FAIL, or UNCLEAR.
- UNCLEAR counts as a gap for exit purposes.
- Score = PASS count / total criteria count.
- A tester passes only if all assigned criteria are PASS.

**Evidence fields:**

```json
{
  "tester_id": "first-time-cli-user",
  "activity_id": "install-and-run-command",
  "criterion": "README quickstart explains session restart requirement",
  "result": "FAIL",
  "severity": "P1",
  "expected": "User can recover from Unknown command",
  "actual": "README does not mention command discovery timing",
  "evidence": "README.md:44 only says copy files, no restart warning",
  "reproduction": ["Copy command file mid-session", "Type /code-beta", "Observe Unknown command"],
  "recommendation": "Add restart warning and manual fallback"
}
```

**Quality filters:**
- Reject vague findings with no evidence.
- Reject FAILs with no reproduction steps.
- Accept UNCLEAR only if the missing runtime/context condition is explicit.
- Escalate high-risk Haiku findings to Sonnet before marking them blockers.

---

## Triage and Exit Decision

**Goal:** Convert feedback flood into release decisions while preserving useful product/developer insight.

Before blocker triage, summarize:
- best-case path status: validated / partially validated / not validated
- persona-specific edge cases worth knowing
- missed risks that are not yet proven blockers
- good signs worth preserving
- ideas sparked that may improve the next iteration

Then classify each finding:
- code defect
- docs/UX gap
- missing test
- environment-specific issue
- known issue
- false positive
- deferred roadmap input
- harness/rubric ambiguity

Assign action:
- fix before merge
- accept as known issue
- defer
- rerun with more context
- escalate to Sonnet
- reject as false positive

Severity:
- P0: data loss, security breach, total outage, irreversible damage
- P1: release blocker for target user flow or public API/CI correctness
- P2: important but workaround exists
- P3: minor polish/docs improvement

Exit criteria should usually require:
- no unresolved P0/P1
- all failures have reproducible evidence or are marked inconclusive
- high-risk Haiku findings confirmed or rejected by Sonnet
- accepted known issues are documented
- rerun shows critical gap closure

---

## Fix Proposals

**Goal:** Produce a minimal, targeted patch for each failing criterion, grounded in the evidence.

**Fix proposal rules:**

1. **One fix per root cause** unless criteria share an exact root cause.
2. **Minimal surface area.** Fix only the lines that cause the failure.
3. **Reference the evidence.** Name tester, activity, criterion, severity, and quote failing evidence.
4. **Show before/after.** Use a diff-style block where possible.
5. **Explain why the fix works** in one sentence.
6. **State fix type:** code, docs, test, config, or harness/rubric.
7. **Check side effects.** If a fix could break another tester's passing criterion, flag it.

**Fix proposal file format** (`docs/fix-proposals/YYYY-MM-DD-<slug>.md`):

```markdown
# Fix Proposal: <slug>
Date: <date>
Session: <session report filename>

## Failing findings addressed
- [tester/activity] criterion — severity

## Fix 1: <file>

**Fix type:** docs/code/test/config/harness
**Why this fixes it:** <one sentence>
**Evidence:** <quote>

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

```text
Initial gap:   N failing / M total criteria
After fixes:   X failing / M total criteria
Gap closed:    yes / no / partial

Tester breakdown:
  ✓ tester-id    N/N  (initial: N/N)
  ✓ tester-id    N/N  (initial: X/N, fixed)
  ✗ tester-id    X/N  (still failing — see evidence)
```

**If gap is not fully closed after one rerun:**
- List still-failing criteria with evidence excerpts.
- Do not auto-iterate further.
- Write a clear `requires human review` section.

**Confidence levels:**
- **High confidence:** evidence is concrete and all high-risk findings confirmed.
- **Medium confidence:** some criteria are UNCLEAR but not release-blocking.
- **Low confidence:** diff/context too broad; recommend narrowing with `--focus` or lower tester quota.
