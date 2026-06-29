---
name: lightrun-pr-review
description: >-
  Use when reviewing a pull request with runtime or production evidence — for
  example to review a PR with runtime verification, gather production evidence,
  or simulate a patch on live samples. Reviews a pull request by diffing against
  the deployed code, checking historical coverage, collecting live samples when
  needed, and simulating the patch on captured production inputs.
version: 0.1.0
---

# Goal

Provide a single PR review backbone that uses runtime production evidence when possible and falls back to scenario-driven review when runtime verification is incomplete.

# Scope

- In scope: deployed-commit diffing, runtime verifiability checks, historical hit discovery, live sampling, runtime patch simulation, and review findings.
- Out of scope: code changes, rollout execution, and repository-specific automation beyond what is documented in the phase references.

# How To Use This Skill

Read and follow these references in order. Each phase produces a specific artifact; confirm that artifact exists and meets the phase's exit criteria before advancing to the next phase.

1. [Intro](references/intro.md) — agent role, the seven-step process, and stage-vs-automation output separation
2. [Constraints](references/constraints.md) — the hard rules every phase must obey
3. [Phase 0 - Get deployed commit](references/phase-0-get-deployed-commit.md) — the baseline commit and production→PR-head diff
4. [Phase 1 - Feasibility check](references/phase-1-feasibility-check.md) — a verifiable / not-verifiable determination
5. [Phase 2 - Runtime profile](references/phase-2-runtime-profile.md) — per-area hit coverage from existing profile data
6. [Phase 3 - Sampling plan](references/phase-3-sampling-plan.md) — snapshot entries only for uncovered areas
7. [Phase 4 - Execute snapshots](references/phase-4-execute-snapshots.md) — captured production samples per verification location
8. [Phase 5 - Patch simulation](references/phase-5-patch-simulation.md) — per-hit pre/post behavior, the runtime verdict, and scenario-driven findings
9. [Output format](references/output-format.md) — the final automation-response transcript

# Checkpoints

Before moving from one phase to the next, confirm the current phase's exit criteria are met:

- After **Phase 0**: a baseline commit is chosen and the production→PR-head (or base→PR-head) diff is in hand.
- After **Phase 1**: feasibility is determined; if not verifiable, the run stops here with the Phase 1 message.
- After **Phase 2**: every diff-derived verification area is marked covered or uncovered with a hit source or reason.
- After **Phase 3**: snapshot entries exist only for areas still uncovered by the runtime profile (or the plan is skipped/partial as appropriate).
- After **Phase 4**: each verification location has captured samples (or is recorded as having no hits).
- After **Phase 5**: the runtime verdict is justified with specific hit values and code lines, and the full Output Format response is emitted.

Do not advance a phase until its exit criteria are satisfied; loop back to the prior phase when evidence is incomplete.

# Working Rules

- Treat the deployed production code as the baseline for all runtime instrumentation.
- Use runtime profile evidence before planning live snapshots.
- Use live snapshots only for verification areas still uncovered after historical search.
- Keep phase messages and the final automation output separate per the intro reference.
- Follow every constraint in [Constraints](references/constraints.md).

# Deliverable

At the end of a completed run, emit the full automation response described in [Output format](references/output-format.md). Keep intermediate phase communication in chat using only the content defined for that phase.
