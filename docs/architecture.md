# Architecture

## Overview

The harness is a stateless Claude Code slash command that orchestrates a multi-agent QA workflow. No server, no database, no external API. Everything runs inside Claude Code via subagents.

```
User invokes /code-beta
        │
        ▼
┌───────────────────┐
│   Orchestrator    │  (Claude Code main agent, runs the command file)
│  code-beta.md     │
└─────────┬─────────┘
          │
    ┌─────┼────────────────┐
    ▼     ▼                ▼
┌──────┐ ┌──────────┐ ┌────────────────┐
│ Diff │ │Codebase  │ │  Persona       │
│Reader│ │Analyzer  │ │  Inferrer      │
└──────┘ └──────────┘ └───────┬────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Rubric Generator  │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │ Persona  │    │ Persona  │    │ Persona  │
        │Subagent 1│    │Subagent 2│    │Subagent N│
        └────┬─────┘    └────┬─────┘    └────┬─────┘
             │               │               │
             └───────────────┼───────────────┘
                             ▼
                   ┌──────────────────┐
                   │ Evidence Scorer  │
                   └────────┬─────────┘
                            │
              gap > 0?      │      gap = 0?
              ┌─────────────┴──────────────┐
              ▼                            ▼
    ┌──────────────────┐        ┌──────────────────┐
    │  Fix Proposer    │        │ Session Reporter  │
    └────────┬─────────┘        └──────────────────┘
             │ --fix or user says yes
             ▼
    ┌──────────────────┐
    │  Apply Patches   │
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │  Rerun Failing   │  (same persona subagents, narrowed to failures)
    │  Personas Only   │
    └────────┬─────────┘
             ▼
    ┌──────────────────┐
    │ Session Reporter │
    └──────────────────┘
```

## Components

### Orchestrator
The main Claude Code agent executing `.claude/commands/code-beta.md`. Coordinates all steps, makes tool calls, spawns subagents, reads evidence, writes reports. Stateless per invocation.

### Diff Reader
Runs `git diff` to extract the changeset. Produces: list of changed files, line ranges, change type classification (feature/fix/refactor/config).

### Codebase Analyzer
Reads README, package manifest, and changed files to build a codebase context summary. Produces: app type, stack, user-facing surfaces, likely user types.

### Persona Inferrer
Applies the methodology in `code-beta-harness.md` to generate 3–6 persona objects from the diff + codebase context. Can be replaced by loading `.code-beta/personas.json` if the user has pre-configured personas.

### Rubric Generator
For each persona, generates 2–5 acceptance criteria scoped to the changed surface. Criteria are falsifiable from static code reading.

### Persona Subagent
A Claude Code `Agent` subagent spawned with a persona-scoped prompt. Reads the relevant changed code (and one hop of callers), reasons through each rubric criterion, and produces a structured evidence report. Runs in parallel with other persona subagents.

### Evidence Scorer
Aggregates subagent output. Parses PASS/FAIL/UNCLEAR per criterion. Computes per-persona and overall scores. Identifies gap (failing criteria count).

### Fix Proposer
For each failing criterion, proposes a minimal code patch grounded in the evidence. Groups by file. Checks proposed changes against passing criteria of other personas.

### Session Reporter
Writes a full audit trail to `docs/beta-sessions/`. Includes diff summary, personas, rubric, evidence reports, scores, fix proposals, and gap closure result.

## Data flow

```
git diff → DiffSummary
DiffSummary + README + package.json → CodebaseContext
CodebaseContext + DiffSummary → [Persona]
[Persona] + DiffSummary → [Rubric]
[Persona] + [Rubric] + ChangedFiles → [EvidenceReport]  (parallel)
[EvidenceReport] → ScoreTable
ScoreTable → (gap > 0) → [FixProposal]
[FixProposal] + AppliedPatches → [EvidenceReport']  (rerun)
[EvidenceReport'] → FinalScoreTable → SessionReport
```

## Agent topology

```
Main agent (orchestrator)
  └── Persona subagent 1  (parallel)
  └── Persona subagent 2  (parallel)
  └── Persona subagent N  (parallel)
       — after first run —
  └── Failing persona subagent A  (parallel, rerun only)
  └── Failing persona subagent B  (parallel, rerun only)
```

Max depth: 2 (orchestrator → persona agents). No nested subagents.

Max subagents per run: 6 personas × 2 runs = 12. Typical: 4 × 2 = 8.

## File outputs (written to target repo)

| Path | Written when |
|------|-------------|
| `docs/fix-proposals/YYYY-MM-DD-<slug>.md` | Gap > 0, fix proposals generated |
| `docs/beta-sessions/YYYY-MM-DD-HH-MM.md` | Every run (success or failure) |
| `.code-beta/personas.json` | Only if user runs `/code-beta --save-personas` (future) |

## Configuration (target repo)

Optional file: `.code-beta/personas.json`

If present, the Persona Inferrer skips inference and loads from this file. Useful for projects with stable, well-known user archetypes. See `examples/persona-schema.json`.

## Constraints and tradeoffs

**Static analysis only.** Persona subagents read code and reason about behavior — they don't execute the code. This means:
- No runtime errors that only appear under load
- No environment-specific failures (missing env vars, misconfigured infra)
- No browser rendering issues
- Tradeoff: zero setup cost, zero flakiness, language-agnostic

**One rerun maximum.** The harness reruns failing personas once after applying fixes. It does not loop. Reasoning: more than one auto-iteration risks cascading incorrect patches. Surface remaining failures for human review.

**Diff-scoped.** The harness only evaluates what changed. Features not touched by the diff are not tested. This is intentional: the goal is to gate the diff, not audit the whole app.

**No historical state.** Each run is independent. The harness does not track regressions across sessions. Session reports provide the audit trail; integration with git history is a future capability.
