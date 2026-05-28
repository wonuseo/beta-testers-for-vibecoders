# MVP Plan

## Goal

Ship a working `/code-beta` command that a vibe-coder can drop into any repo and run on a diff to get useful persona-driven QA feedback in under 5 minutes.

"Useful" means: at least one real bug or UX gap identified that the developer would not have caught by reading the code themselves.

---

## Phase 1 — Static evidence, manual personas (now)

**What ships:** The scaffold: command file, skill file, schemas, docs.

**What works:**
- `/code-beta --diff HEAD~1..HEAD` on a Next.js or Python Flask repo
- Persona inference from diff + README + package.json
- Rubric generation (2–4 criteria per persona)
- Persona subagents producing evidence via code reading
- Score table output in terminal
- Session report written to `docs/beta-sessions/`

**What doesn't work yet:**
- Fix proposals (Step 7 is described but not battle-tested)
- `--fix` auto-apply (described but untested)
- `.code-beta/personas.json` custom loading (schema exists, parsing not wired)

**Success metric:** Running on 3 sample diffs produces at least one FAIL finding per run that is accurate (not a hallucination).

**How to validate:**
1. Take 3 real PRs from open-source repos with known bugs.
2. Run `/code-beta` on each diff.
3. Check if the harness identifies the known bug as a FAIL.
4. Adjust persona inference and rubric generation prompts until hit rate ≥ 2/3.

---

## Phase 2 — Fix proposals and auto-apply

**Prerequisite:** Phase 1 validated. Evidence quality is reliable enough to ground patches.

**What ships:**
- Hardened Step 7 (fix proposals with before/after diffs)
- `--fix` flag that applies proposals and reruns
- Side-effect check (proposed fix vs. other personas' passing criteria)
- Rerun logic for failing personas only

**Success metric:** On the 3 sample diffs from Phase 1, `--fix` closes the gap (gap = 0 after one rerun) at least twice.

**Key risk:** Fix proposals based on static reading may miss the real root cause when the bug is behavioral (race condition, async timing, etc.). Mitigation: require UNCLEAR rating (not FAIL) when cause is ambiguous; proposals only for deterministic FAILs.

---

## Phase 3 — Custom persona configuration

**Prerequisite:** Phase 2 validated.

**What ships:**
- `.code-beta/personas.json` loading (bypass inference)
- `/code-beta --save-personas` to persist inferred personas for future runs
- Persona refinement loop: user can edit `.code-beta/personas.json` and rerun

**Why this matters:** Projects with stable user types (B2B SaaS with admin/member/viewer, APIs with internal/external/partner consumers) want consistent personas across every diff, not re-inferred ones.

---

## Phase 4 — Scope narrowing and CI integration

**What ships:**
- `--scope <label>` flag to narrow persona inference to a feature area
- Markdown summary suitable for posting as a PR comment (GitHub Actions hook)
- Exit code: 0 if gap closed, 1 if gap remains (enables CI gating)

**CI integration pattern (Phase 4 — not yet implemented):**

> **Status:** The `--scope` flag and exit code support are not yet implemented. The YAML below shows the intended pattern. Today, `claude -p` always exits 0 — CI gating on gap requires parsing the output until exit code support ships.

```yaml
# .github/workflows/beta-test.yml  (Phase 4 target — not yet functional)
- name: Run /code-beta
  run: |
    claude -p "/code-beta --diff ${{ github.base_ref }}...${{ github.head_ref }}" \
           --output-format markdown > beta-report.md
    # Phase 4: claude will exit 1 when gap > 0.
    # Until then, gate on grep: grep -q "Gap closed: no" beta-report.md && exit 1 || exit 0

- name: Post beta-report as PR comment
  if: always()
  uses: actions/github-script@v7
  with:
    script: |
      const fs = require('fs');
      const report = fs.readFileSync('beta-report.md', 'utf8');
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: report
      });
```

**Manual CI gating (works today):** Until Phase 4 ships exit codes, use the grep fallback shown in the YAML comment: check whether the session report contains `"Gap closed: no"` and fail the step if so.

---

## What we're explicitly not building (in MVP)

| Idea | Why not now |
|------|-------------|
| Browser/E2E execution | Zero-dependency is the key differentiator; adding Playwright breaks it |
| Sentry/Linear integration | External service coupling; adds auth complexity |
| Multi-turn persona conversations | Diminishing returns vs. complexity; static reading catches 80% of issues |
| Regression tracking across runs | Requires state; session reports provide the audit trail manually |
| Web dashboard | This lives inside Claude Code, not in a browser |
| Language-specific plugins | Language-agnostic is the value prop; plugins can come after adoption |

---

## Open questions

1. **Evidence reliability threshold.** At what score do we consider evidence "too vague to trust" and re-run the subagent with tighter instructions? Hypothesis: if fewer than 50% of criteria have code citations, re-prompt.

2. **Diff size limit.** Very large diffs (500+ lines) produce low-confidence results because the surface area is too broad. Should we warn users and suggest `--scope`? Or split the diff automatically by file group?

3. **Persona count sweet spot.** 3–6 personas is the current guidance. Too few misses edge cases; too many dilutes focus and inflates cost. Need empirical data from Phase 1 validation.

4. **UNCLEAR vs. FAIL.** Some criteria can't be evaluated statically (race conditions, env-dependent behavior). UNCLEAR counts as FAIL for gap purposes. Is this too conservative? Could lead to false positives in the gap count.

5. **Fix proposal quality.** How do we prevent the harness from proposing fixes that "fix the test" (make the criterion pass) without actually fixing the underlying issue? Requiring the fix to change the actual behavior, not just the evidence artifact, is the intent — but needs explicit guardrails in the skill file.
