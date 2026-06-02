# MVP Plan

## Goal

Ship a working `/betatest` command that a vibe-coder can drop into any repo and run on a diff to get useful synthetic beta-program feedback in under 5 minutes.

"Useful" means: at least one real bug, docs/UX gap, integration risk, or release blocker identified that the developer would not have caught by reading the code themselves.

---

## Product shape

`/betatest` should mirror real beta program management:

```text
plan beta → recruit testers → assign activities → collect evidence → triage → fix/accept/defer → rerun → exit decision
```

Two-layer UX:

```text
/betatest setup --diff HEAD~1..HEAD --testers 6
/betatest run --diff HEAD~1..HEAD
```

One-shot remains supported:

```text
/betatest --diff HEAD~1..HEAD --testers 6
```

Default model policy:

| Role | Default |
|---|---|
| Planner | Opus |
| Recruiter | Opus |
| Breadth tester | Haiku |
| High-risk/deep tester | Sonnet |
| Escalation | Sonnet |
| Aggregator/triage | Sonnet |

---

## Phase 1 — Beta setup + static evidence (now)

**What ships:** Command file, skill file, schemas, docs, and two-layer setup/run instructions.

**What works:**
- `/betatest setup --diff HEAD~1..HEAD`
- beta objective and beta type selection
- quota-based tester recruitment
- model policy recording: planner/recruiter Opus, testers Haiku/Sonnet, triage Sonnet
- beta activity generation instead of vague review prompts
- tester evidence via bounded code/docs reading
- score table and triage output
- session report written to `docs/beta-sessions/`

**What doesn't work yet:**
- Actual per-subagent model enforcement may depend on Claude Code runtime capability; artifacts record intended model policy even when runtime cannot enforce it.
- Fix proposals are described but not battle-tested.
- `--fix` auto-apply is described but untested.
- `.harness/code-beta.config.json` persistence is specified but not yet backed by a standalone runner.

**Success metric:** Running on 3 sample diffs produces at least one accurate finding per run with reproducible evidence and correct severity.

**How to validate:**
1. Take 3 real PRs from open-source repos with known bugs or docs gaps.
2. Run `/betatest setup` and inspect beta plan/recruitment quality.
3. Run `/betatest run` on each diff.
4. Check if the harness identifies the known issue as a FAIL/UNCLEAR with evidence.
5. Adjust beta setup, recruitment, and activity prompts until hit rate ≥ 2/3.

---

## Phase 2 — Fix proposals and rerun loop

**Prerequisite:** Phase 1 validated. Evidence quality is reliable enough to ground patches.

**What ships:**
- Hardened fix proposals with before/after diffs
- `--fix` flag that applies proposals and reruns failed testers
- Side-effect check against other testers' passing criteria
- Rerun logic for failed/unclear testers only
- `gap-closure.md` before/after comparison

**Success metric:** On the 3 sample diffs from Phase 1, `--fix` closes critical gaps at least twice.

**Key risk:** Fix proposals based on static reading may miss behavioral root causes. Mitigation: require UNCLEAR rating for ambiguous runtime issues; proposals only for deterministic FAILs unless user explicitly approves exploratory fixes.

---

## Phase 3 — Persistent configuration and custom tester pools

**Prerequisite:** Phase 2 validated.

**What ships:**
- `.harness/code-beta.config.json` as first-class saved setup
- `/betatest setup --save` and `/betatest run` using saved config
- custom tester pool loading
- tester refinement loop: user edits config and reruns
- per-project stable beta segments for repeated diffs

**Why this matters:** Projects with stable user types (B2B SaaS admin/member/viewer, APIs with internal/external/partner consumers, CLIs with local/CI users) want consistent tester pools across every diff.

---

## Phase 4 — Scope narrowing and CI integration

**What ships:**
- `--focus <label>` flag to narrow beta setup to a feature area
- Markdown summary suitable for PR comments
- Exit code: 0 if exit criteria pass, 1 if unresolved blockers remain

**CI integration pattern (Phase 4 — not yet implemented):**

> **Status:** The `--focus` flag and exit code support are not yet implemented in a standalone runner. The YAML below shows the intended pattern. Today, `claude -p` may exit 0 even when beta gaps exist — CI gating requires parsing the report until exit code support ships.

```yaml
# .github/workflows/beta-test.yml  (Phase 4 target — not yet functional)
- name: Run /betatest
  env:
    BASE_REF: ${{ github.base_ref }}
    HEAD_REF: ${{ github.head_ref }}
  run: |
    claude -p "/betatest --diff ${BASE_REF}...${HEAD_REF} --testers 6" \
           --output-format markdown > beta-report.md
    # Phase 4: claude will exit 1 when unresolved P0/P1 remains.
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

**Manual CI gating (works today):** Until Phase 4 ships exit codes, parse the session report for unresolved P0/P1 or `Gap closed: no`.

---

## What we're explicitly not building in MVP

| Idea | Why not now |
|------|-------------|
| Browser/E2E execution | Zero-dependency is the key differentiator; adding Playwright breaks it |
| External beta-tester SaaS | This should live inside Claude Code |
| Sentry/Linear integration | External service coupling; adds auth complexity |
| Full repo understanding | Large projects cannot fit; realistic beta context is bounded |
| Web dashboard | This lives inside Claude Code, not in a browser |
| Language-specific plugins | Language-agnostic is the value prop; plugins can come after adoption |

---

## Open questions

1. **Model enforcement.** Can Claude Code reliably select Opus/Haiku/Sonnet per subagent, or should the command only record intended policy and rely on available runtime model?
2. **Evidence reliability threshold.** At what score do we consider evidence too vague to trust and rerun the tester with tighter instructions?
3. **Diff size limit.** Should large diffs automatically split into beta surfaces before recruitment?
4. **Tester count sweet spot.** Is 4–8 enough for MVP, or should setup propose count dynamically by risk?
5. **UNCLEAR vs. FAIL.** UNCLEAR counts as a gap for exit purposes. Is this too conservative for docs-only testers?
6. **Fix proposal quality.** How do we prevent the harness from fixing the prompt/rubric rather than the product issue?
