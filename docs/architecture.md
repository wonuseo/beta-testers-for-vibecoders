# Architecture

## Overview

The harness is a stateless Claude Code slash command that orchestrates a synthetic beta program. No server, no database, no external API. Everything runs inside Claude Code via the main agent and subagents.

The core product shape is two-layer:

```text
/betatest setup  → beta plan + tester recruitment
/betatest run    → tester execution + evidence + triage + rerun
```

One-shot `/betatest` executes both layers.

```text
User invokes /betatest
        │
        ▼
┌───────────────────┐
│   Orchestrator    │  (Claude Code main agent, runs command file)
│  betatest.md      │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│    Diff Reader    │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│ Beta Planner      │  Opus policy
│ objective/type    │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│ Tester Recruiter  │  Opus policy
│ quota/segments    │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│ Activity/Rubric   │
│ Generator         │
└─────────┬─────────┘
          │
          ▼
┌─────────┴────────────────────────────────────┐
│ Recruited beta tester subagents              │
│ Haiku breadth / Sonnet high-risk             │
└─────────┬────────────────────────────────────┘
          │
          ▼
┌───────────────────┐
│ Aggregator/Triage │  Sonnet policy
└─────────┬─────────┘
          │
   unresolved blocker?
     ┌────┴────┐
     ▼         ▼
┌──────────┐ ┌──────────────────┐
│ Fix /    │ │ Session Reporter │
│ Decision │ └──────────────────┘
└────┬─────┘
     │ --fix or user approval
     ▼
┌───────────────────┐
│ Rerun failed or   │
│ unclear testers   │
└─────────┬─────────┘
          ▼
┌───────────────────┐
│ Gap closure +     │
│ Session Reporter  │
└───────────────────┘
```

## Components

### Orchestrator
The main Claude Code agent executing `.claude/commands/betatest.md`. Coordinates steps, makes tool calls, spawns tester subagents, reads evidence, writes reports. Stateless per invocation except for `.harness/betatest.config.json`.

### Diff Reader
Runs `git diff` to extract the changeset. Produces: list of changed files, line ranges, change type classification, and likely user-visible or maintainer-visible surfaces.

### Beta Planner
Uses Opus-level policy. Reads bounded context and writes the beta plan: objective, beta type, scope, out-of-scope, risks, evidence schema, and exit criteria. It should not read the whole repo.

### Tester Recruiter
Uses Opus-level policy. Converts the beta plan into a quota-based synthetic tester pool. Each tester has segment, screening criteria, model policy, context policy, allowed context, assigned activity, and success condition.

### Activity / Rubric Generator
For each recruited tester, creates one concrete beta activity and 2–5 acceptance criteria. Activities must be actionable and evidence-producing, not vague review prompts.

### Beta Tester Subagent
A Claude Code `Agent` subagent spawned with a tester-scoped prompt. It follows its assigned activity under bounded context, records steps, evaluates criteria, and produces structured evidence. Breadth testers default to Haiku policy; high-risk/deep testers default to Sonnet policy.

### Aggregator / Triage
Uses Sonnet-level policy. Aggregates tester output, dedupes findings, classifies severity and type, escalates high-risk Haiku findings to Sonnet, and decides current-release action: fix now, accept known issue, defer, rerun with more context, or reject as false positive.

### Fix / Decision Log
For unresolved blockers, writes fix proposals grounded in evidence. If a finding is accepted or deferred, records the rationale in a decision log.

### Session Reporter
Writes a full audit trail to `docs/beta-sessions/`. Includes target diff, beta plan, tester recruitment, activities, evidence reports, triage, fix proposals, decisions, and gap closure result.

## Data flow

```text
git diff → DiffSummary
DiffSummary + bounded repo signals → BetaPlan
BetaPlan + options → TesterRecruitment
TesterRecruitment → [Activity] + [Rubric]
[Tester] + [Activity] + [Rubric] + bounded context → [EvidenceReport]  (parallel)
[EvidenceReport] → Aggregate + Triage
Triage → (unresolved blocker) → [FixProposal] / [DecisionLog]
[FixProposal] + AppliedPatches → [EvidenceReport']  (rerun failed/unclear only)
[EvidenceReport'] → GapClosure → SessionReport
```

## Agent topology

```text
Main agent (orchestrator)
  └── Beta tester subagent 1  (parallel)
  └── Beta tester subagent 2  (parallel)
  └── Beta tester subagent N  (parallel)
       — after first run —
  └── Failed/unclear tester A  (parallel, rerun only)
  └── Failed/unclear tester B  (parallel, rerun only)
```

Max depth: 2 (orchestrator → tester agents). No nested subagents.

Default tester count: 6. Typical run: 6 initial testers + rerun failed/unclear testers only.

## File outputs (written to target repo)

| Path | Written when |
|------|-------------|
| `.harness/betatest.config.json` | Setup writes repeatable beta config |
| `.harness/runs/<run_id>/beta-plan.md` | Setup |
| `.harness/runs/<run_id>/scope-map.json` | Setup |
| `.harness/runs/<run_id>/risk-map.json` | Setup |
| `.harness/runs/<run_id>/tester-recruitment.md` | Setup |
| `.harness/runs/<run_id>/personas.json` | Setup/recruitment, compatibility name |
| `.harness/runs/<run_id>/test-activities.json` | Before execution |
| `.harness/runs/<run_id>/rubric.json` | Before execution |
| `.harness/runs/<run_id>/testers/<tester-id>.md` | Each tester run |
| `.harness/runs/<run_id>/aggregate.md` | After aggregation |
| `.harness/runs/<run_id>/aggregate.json` | After aggregation |
| `.harness/runs/<run_id>/decision-log.md` | If findings are accepted/deferred/rejected |
| `.harness/runs/<run_id>/gap-closure.md` | After rerun |
| `docs/fix-proposals/YYYY-MM-DD-<slug>.md` | Gap > 0, fix proposals generated |
| `docs/beta-sessions/YYYY-MM-DD-HH-MM.md` | Every run |

## Configuration (target repo)

Primary file: `.harness/betatest.config.json`

This stores:
- target diff
- beta type
- model policy
- tester quota
- recruited tester pool
- context policies
- activity assignments
- exit criteria

## Constraints and tradeoffs

**Bounded context only.** The harness intentionally does not read the whole repo. Real users do not have full internal knowledge; requiring hidden knowledge is a docs/UX gap.

**Evidence-first, not assertion-first.** Tester subagents primarily produce evidence from docs/code/commands. Runnable test scripts are optional future work.

**One rerun maximum.** The harness reruns failed/unclear testers once after applying fixes. It does not loop indefinitely.

**Diff-scoped.** The harness evaluates changed surfaces and explicit beta risks, not the entire app.

**Model policy may be aspirational.** Claude Code environments may not enforce per-subagent Opus/Haiku/Sonnet. In that case, artifacts record intended model policy and execution uses the available model.
