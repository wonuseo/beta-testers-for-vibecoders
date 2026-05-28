# Fix Proposal: scaffold-gaps
Date: 2026-05-29
Session: docs/beta-sessions/2026-05-29-15-00.md

## Failing criteria addressed

- [vibe-coder-first-install] vi-3 — command proceeds without referencing an unregistered skill
- [mid-session-user] ms-1 — session restart requirement documented
- [mid-session-user] ms-2 — manual fallback provided
- [mid-session-user] ms-3 — troubleshooting section exists
- [harness-developer] hd-2 — specific reproducible test scenario named
- [ci-integrator] ci-2 — CI YAML exit code contract accurate

---

## Fix 1: `.claude/commands/code-beta.md`

**Criterion:** vi-3

**Why this fixes it:** The command previously said "Use the `code-beta-harness` skill" without ever instructing Claude to read the file. The preamble makes the dependency explicit and file-path-based.

**Change:**
```diff
 ## Workflow
-
-Follow these steps in order. Use the `code-beta-harness` skill for the methodology behind each step.
+
+**Before executing any step:** Read `.claude/skills/code-beta-harness.md` in full. All references
+to "the methodology" below point to sections in that file. Reading it now avoids missing-skill
+errors if the file was added mid-session or the skill is not registered.
+
+```
+Read: .claude/skills/code-beta-harness.md
+```
+
+Follow these steps in order.
```

**Potential side effects:** None. Adds one read operation at the start of every run; no logic changes.

---

## Fix 2: `README.md`

**Criteria:** ms-1, ms-2, ms-3

**Why this fixes it:** The Quickstart had no session caveat and the repo had no troubleshooting section. Users who hit "unknown command" had no documented recovery path. The `> Important:` callout and Troubleshooting section address all three gaps in one place.

**Change:** Added after the `/code-beta` invocation examples in Quickstart:
```diff
+> **Important:** Claude Code only discovers command files at session start. If you copied
+> the files into an already-open session, `/code-beta` won't be available until you start
+> a new session.
+
+## Troubleshooting
+
+**`/code-beta` shows as unknown command**
+...
+**Option 2 — Invoke manually without restarting**: Paste this into the Claude Code chat:
+"Read .claude/commands/code-beta.md and .claude/skills/code-beta-harness.md, then follow
+the workflow defined in code-beta.md on the diff HEAD~1..HEAD."
```

**Potential side effects:** None. Additive documentation only.

---

## Fix 3: `CLAUDE.md`

**Criterion:** hd-2

**Why this fixes it:** "Pick any repo with a recent diff" is not reproducible between contributors. Naming this repo's own scaffold diff (`HEAD~2..HEAD`) gives every contributor the same starting point.

**Change:** Replaced step 1 of "Testing this harness" with a named reference scenario pointing to `HEAD~2..HEAD`, `examples/sample-run.md` (quality template), and `docs/beta-sessions/` (runtime baseline).

**Potential side effects:** None. Documentation only; the referenced files now exist.

---

## Fix 4: `docs/mvp-plan.md`

**Criterion:** ci-2

**Why this fixes it:** The YAML had a comment saying "fails CI if gap > 0" but no exit code handling — a CI integrator copying this verbatim would get a workflow that always exits 0. The fix adds a not-yet-implemented label and a working grep-based workaround for today.

**Change:**
```diff
-**CI integration pattern:**
-```yaml
-# .github/workflows/beta-test.yml
-- name: Run /code-beta
-  run: |
-    claude -p "/code-beta --diff ..."  --output-format markdown > beta-report.md
-  # Posts report as PR comment; fails CI if gap > 0
-```
+**CI integration pattern (Phase 4 — not yet implemented):**
+> **Status:** Exit code support not yet implemented. Today, `claude -p` always exits 0.
+```yaml
+# .github/workflows/beta-test.yml  (Phase 4 target — not yet functional)
+    # Phase 4: claude will exit 1 when gap > 0.
+    # Until then: grep -q "Gap closed: no" beta-report.md && exit 1 || exit 0
+```
+**Manual CI gating (works today):** grep for `"Gap closed: no"` in the report.
```

**Potential side effects:** None. The original YAML was already non-functional for CI gating; this makes that explicit.
