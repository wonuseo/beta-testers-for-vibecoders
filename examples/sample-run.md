# Sample Run: Auth Session Refresh Bug

This annotated example shows what a `/code-beta` run looks like end-to-end.
The target repo is a Next.js SaaS app. The diff fixes a reported bug where users get
silently logged out when their session token expires.

---

## Invocation

```
/code-beta --diff HEAD~1..HEAD
```

---

## Step 1 — Diff resolved

```
git diff HEAD~1..HEAD --stat

 auth/session.ts    | 34 ++++++++++++-------
 ui/LoginForm.tsx   |  8 +++--
 api/session.ts     | 12 +++---
 3 files changed, 31 insertions(+), 23 deletions(-)
```

**Diff summary:** 3 files changed. Areas: authentication, session management, login UI.
Change type: **fix** (commit message: "fix: prevent double-refresh race condition on token expiry").

---

## Step 2 — Codebase context

Read: `README.md`, `package.json`, `auth/session.ts`, `ui/LoginForm.tsx`, `api/session.ts`.

> **App type:** Next.js 14 SaaS — team collaboration tool  
> **Stack:** TypeScript, Next.js App Router, Prisma, PostgreSQL, cookie-based JWT auth  
> **User-facing surfaces changed:** Session refresh logic (invisible to users when working), login form validation, refresh API endpoint  
> **Likely user types:** SaaS end users (team members), admins, REST API consumers (integrations)

---

## Step 3 — Personas inferred

```
Inferred 4 personas:

1. new-user
   Who: First-time signup, no account yet, on desktop Chrome
   Entry point: POST /api/auth/signup → login form
   Goal: Complete signup and land on dashboard

2. returning-user
   Who: Active user coming back after access token expired (valid refresh token)
   Entry point: SWR client calls /api/auth/refresh automatically on 401
   Goal: Resume session without being redirected to login

3. mobile-safari-user
   Who: Existing user on iPhone Safari, completing login after session expired
   Entry point: /login page, mobile viewport
   Goal: Log in and get back to their workspace

4. api-consumer
   Who: Integration script using Bearer token auth via REST
   Entry point: Authorization: Bearer <token> on any API route
   Goal: Refresh token programmatically when 401 received; get machine-readable error body
```

---

## Step 4 — Rubric

```
Rubric: 12 criteria across 4 personas

new-user (3 criteria)
  [new-1] new-user submits valid signup form → account created, redirected to /dashboard
  [new-2] new-user submits blank email → inline validation error shown immediately, not on submit
  [new-3] new-user submits duplicate email → 409 error shown in form, not a blank error page

returning-user (3 criteria)
  [ret-1] returning-user's expired access token → /api/auth/refresh called once; new token set
  [ret-2] concurrent page loads with expired token → only one refresh executes; others wait and reuse
  [ret-3] returning-user's refresh token also expired → redirected to /login with "Session expired" message

mobile-safari-user (3 criteria)
  [mob-1] mobile-safari-user taps Login → submit button not hidden behind iOS keyboard
  [mob-2] mobile-safari-user's session expires → same transparent refresh as desktop (ret-1)
  [mob-3] mobile-safari-user sees validation errors → errors visible without scrolling

api-consumer (3 criteria)
  [api-1] api-consumer's Bearer token expired → 401 with JSON body { error: string, error_code: "TOKEN_EXPIRED" }
  [api-2] api-consumer calls refresh endpoint with valid refresh token → 200 with { access_token, expires_in }
  [api-3] api-consumer calls refresh with expired/invalid refresh token → 401 with JSON body { error_code: "REFRESH_EXPIRED" }
```

---

## Step 5 — Persona subagents running

```
Spawning 4 persona subagents in parallel...
```

**Subagent prompt sent to `returning-user` agent (abbreviated):**

```
You are testing a code change as the "returning-user" persona.

PERSONA: returning-user
An active SaaS user whose access token just expired. They have a valid refresh token
in their httpOnly cookie. They want to resume their session transparently without
hitting the login page. They're on desktop Chrome.

CODEBASE CONTEXT:
Next.js 14 SaaS app, cookie-based JWT auth. The diff changes session.ts (refresh
logic) and api/session.ts (refresh endpoint). SWR client calls /api/auth/refresh
automatically when it receives a 401.

CHANGED FILES:
- auth/session.ts — refresh token mutex and lock logic (main change)
- api/session.ts — refresh endpoint response format
- ui/LoginForm.tsx — validation display timing (minor)

RUBRIC:
[ret-1] returning-user's expired access token → /api/auth/refresh called once; new token set
[ret-2] concurrent page loads with expired token → only one refresh executes; others wait and reuse
[ret-3] returning-user's refresh token also expired → redirected to /login with "Session expired" message

Read auth/session.ts and api/session.ts. Evaluate each criterion...
```

---

## Step 6 — Evidence collected

**returning-user evidence (abbreviated):**

```markdown
## Persona: returning-user
### Attempt summary
Traced what happens when SWR receives a 401: it calls refreshToken() in session.ts,
which now uses a module-level Promise lock.

### Results
| Criterion | Result | Evidence |
|-----------|--------|----------|
| ret-1: refresh called once, new token set | PASS | session.ts:52 sets new cookie on resolve |
| ret-2: concurrent refresh → one executes, others wait | FAIL | See below |
| ret-3: refresh token expired → /login with message | PASS | session.ts:71 redirects with query param |

### Failures
**Criterion:** ret-2 — concurrent refresh → only one executes
**What breaks:** The lock in session.ts:38 is initialized inside the function scope,
not at module level. Each call creates its own lock variable, so parallel calls do not
share state and each fires its own refresh.
**Relevant code:** session.ts:38 — `let refreshLock: Promise<void> | null = null;`
(inside refreshToken function, not module scope)
**Reproduction:**
1. Open two browser tabs, both with expired tokens
2. Both tabs call SWR's revalidate simultaneously
3. Both call refreshToken() → both see refreshLock = null → both initiate refresh
4. Second refresh may revoke first token (if rotation enabled)

### Overall score: 2/3 criteria passing
```

**api-consumer evidence (abbreviated):**

```markdown
## Persona: api-consumer
### Results
| Criterion | Result | Evidence |
|-----------|--------|----------|
| api-1: 401 with error_code: "TOKEN_EXPIRED" | FAIL | See below |
| api-2: refresh endpoint returns { access_token, expires_in } | PASS | api/session.ts:89 |
| api-3: refresh with expired refresh token → 401 with error_code | FAIL | See below |

### Failures
**Criterion:** api-1
**What breaks:** api/session.ts:44 returns `{ error: "Unauthorized" }` — no error_code field.
**Relevant code:** `return NextResponse.json({ error: "Unauthorized" }, { status: 401 })`
**Reproduction:** Send any request with expired Bearer token; inspect response body.

**Criterion:** api-3
**What breaks:** Same location — session.ts refresh failure path returns the same body
without error_code. api-consumer cannot distinguish TOKEN_EXPIRED from REFRESH_EXPIRED.
```

---

## Step 6 — Score table

```
Results:

  Persona               Score    Failing criteria
  ──────────────────────────────────────────────
  new-user              3/3      —
  returning-user        2/3      ret-2
  mobile-safari-user    3/3      —
  api-consumer          1/3      api-1, api-3

  Total: 9/12 passing   Gap: 3 criteria failing
```

---

## Step 7 — Fix proposals

```
Fix 1: auth/session.ts — move refreshLock to module scope

Addresses: ret-2 (returning-user concurrent refresh)

Why: Lock initialized inside function doesn't persist between calls.
Moving to module scope makes all callers share the same lock.

--- auth/session.ts
+++ auth/session.ts
@@ -35,8 +35,9 @@
+let refreshLock: Promise<void> | null = null;   // module scope
+
 export async function refreshToken(): Promise<void> {
-  let refreshLock: Promise<void> | null = null;  // BUG: local scope
   if (refreshLock) {
     return refreshLock;
   }

Fix 2: api/session.ts — add error_code to 401 responses

Addresses: api-1, api-3 (api-consumer error body)

Why: Machine clients need error_code to distinguish error types programmatically.

--- api/session.ts:44
+++ api/session.ts:44
-  return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
+  return NextResponse.json({ error: "Unauthorized", error_code: "TOKEN_EXPIRED" }, { status: 401 });

--- api/session.ts:67
+++ api/session.ts:67
-  return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
+  return NextResponse.json({ error: "Unauthorized", error_code: "REFRESH_EXPIRED" }, { status: 401 });

Side effect check: these changes do not affect new-user or mobile-safari-user criteria (different code paths).
```

---

## Step 8 — Apply and rerun

```
Apply fixes? y

Applying Fix 1 to auth/session.ts...
Applying Fix 2 to api/session.ts (2 locations)...
Fix proposals written to: docs/fix-proposals/2026-05-29-auth-refresh.md

Rerunning returning-user and api-consumer subagents...

  returning-user    3/3  ✓ (was 2/3)
  api-consumer      3/3  ✓ (was 1/3)
```

---

## Step 9 — Final report

```
Final: 12/12 passing. Gap closed.

Gap closure: 3 failing → 0 failing
Confidence: HIGH (all evaluations from static code reading with citations)

Session report → docs/beta-sessions/2026-05-29-14-32.md
Fix proposals  → docs/fix-proposals/2026-05-29-auth-refresh.md
```

---

## Notes on this example

The two bugs found (`refreshLock` scope and missing `error_code`) are exactly the kind of issue that:
- Would pass all unit tests (tests mock the lock behavior)
- Would pass code review by someone focused on the logic, not the scope
- Would be caught by a QA engineer testing the concurrent-session edge case
- Would be caught by an API consumer the first time they try to handle errors programmatically

The harness found both without running the code.
