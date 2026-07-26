---
name: lightrun-runtime-aware-pr-review
description: >-
  Use when reviewing a pull request with runtime or production evidence — for
  example to review a PR with runtime verification, gather production evidence,
  or simulate a patch on live samples. Reviews a pull request by diffing against
  the PR merge base, collecting live samples, and simulating the
  patch on captured production inputs.
---

# Usage telemetry

When this skill is invoked, it may report one usage event through the Lightrun-owned `skill_start` MCP tool. This helps Lightrun measure skill adoption. The request supplies the skill name `lightrun-runtime-aware-pr-review`.

Call the tool once only when it is verified as belonging to the Lightrun MCP provider. Do not call similarly named tools exposed by other providers. If the tool is unavailable or the provider cannot be verified, skip telemetry and continue normally.

# Goal

Provide a single PR review backbone that uses runtime production evidence when possible and falls back to scenario-driven review when runtime verification is incomplete.

# Scope

- In scope: PR merge-base diffing, deployed-revision resolution for instrumentation, runtime verifiability checks, live sampling, runtime patch simulation with PR-base vs PR-head attribution, and review findings.
- Out of scope: code changes, rollout execution, historical runtime-profile hit discovery (coming soon), and repository-specific automation beyond what is documented in the phase references.

# How To Use This Skill

Read and follow these references in order. Each phase produces a specific artifact; confirm that artifact exists and meets the phase's exit criteria before advancing to the next phase.

1. [Intro](references/intro.md) — agent role, the process steps, and stage-vs-automation output separation
2. [Constraints](references/constraints.md) — the hard rules every phase must obey
3. [Phase 0 - Get deployed commit](references/phase-0-get-deployed-commit.md) — PR base/head SHAs, PR diff, and optional deployed revision
4. [Phase 1 - Feasibility check](references/phase-1-feasibility-check.md) — a verifiable / not-verifiable determination
5. [Phase 2 - Verification areas](references/phase-2-verification-areas.md) — PR diff areas mapped to deployed equivalents
6. [Phase 3 - Runtime profile](references/phase-3-runtime-profile.md) — coming soon; record unavailable and continue
7. [Phase 4 - Sampling plan](references/phase-4-sampling-plan.md) — snapshot entries for uncovered verification areas
8. [Phase 5 - Execute snapshots](references/phase-5-execute-snapshots.md) — captured production samples at deployed locations only
9. [Phase 6 - Patch simulation](references/phase-6-patch-simulation.md) — PR-base vs PR-head behavior, runtime assessment, and scenario-driven findings
10. [Output format](references/output-format.md) — the final automation-response transcript

# Checkpoints

Before moving from one phase to the next, confirm the current phase's exit criteria are met:

- After **Phase 0**: `pr_base_sha`, `pr_head_sha`, and the PR diff (`pr_base_sha` → `pr_head_sha`) are in hand; `deployed_sha` is recorded when resolved.
- After **Phase 1**: feasibility is determined; if not verifiable, the run stops here with the Phase 1 message.
- After **Phase 2**: every PR verification area has a deployed mapping and status (`uncovered` or `not runtime-verifiable`).
- After **Phase 3**: record `runtime_profile_mcp: unavailable` (coming soon) and proceed with Phase 2 mapped areas still `uncovered`.
- After **Phase 4**: snapshot entries exist for the highest-value uncovered verification areas (up to 3).
- After **Phase 5**: every verification location has at least one hit, **or** the run stopped with a Sampling Request because one or more locations still had zero hits (resume required before Phase 6).
- After **Phase 6**: the runtime assessment is justified with PR-base vs PR-head behavior on valid samples, and the full Output Format response is emitted.

Do not advance a phase until its exit criteria are satisfied; loop back to the prior phase when evidence is incomplete.

# Working Rules

- Use the PR diff (`pr_base_sha` → `pr_head_sha`) for all verification areas, attribution, findings, runtime assessment, and merge recommendation.
- Use `deployed_sha` only to locate safe runtime instrumentation and describe true deployed production behavior.
- Never use `deployed_sha` → `pr_head_sha` as the PR review diff.
- Never query, instrument, or snapshot PR-only or unmapped coordinates.
- Skip runtime profile search in this skill (coming soon); plan and run live snapshots for Phase 2 mapped `uncovered` areas directly.
- Keep phase messages and the final automation output separate per the intro reference.
- For async live snapshots: partial hits do not complete Phase 5 — if any verification location still has zero hits after the bounded in-session poll, emit the Sampling Request and stop instead of proceeding to Phase 6.
- Follow every constraint in [Constraints](references/constraints.md).

# Deliverable

At the end of a completed run, emit the full automation response described in [Output format](references/output-format.md). Keep intermediate phase communication in chat using only the content defined for that phase.
