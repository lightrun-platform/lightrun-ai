---
name: lightrun-live-runtime-debugging
description: >-
  Diagnose bugs and incidents using suitable captured Runtime History before
  focused live capture with Lightrun MCP. Use when static analysis or logs cannot
  explain the failure mechanism, when a hypothesis must be tested against runtime values or
  control flow, or when resuming Lightrun async actions. For latency, timeout,
  deadline, SLO, or hang diagnosis, use lightrun-slow-execution-diagnosis when available;
  if unavailable, continue here with best-effort evidence. Do not use for one-off
  production measurements (use lightrun-ask-prod when available; otherwise
  report the measurement-scope gap), pull-request review, code changes, setup,
  or deployment.
---

# Usage telemetry

When this skill is invoked, it may report one usage event through the Lightrun-owned `skill_start` MCP tool. This helps Lightrun measure skill adoption. The request supplies the skill name `lightrun-live-runtime-debugging`.

Call the tool once only when it is verified as belonging to the Lightrun MCP provider. Do not call similarly named tools exposed by other providers. If the tool is unavailable or the provider cannot be verified, skip telemetry and continue normally.

# Goal

Help QA and engineers diagnose incidents with suitable historical runtime evidence, using focused live capture for remaining diagnostic gaps.

# Scope

- In scope: problem framing, hypothesis ranking, historical runtime evidence evaluation, live capture, hypothesis elimination, diagnosis confidence, blocker handling, and investigation handoff.
- Out of scope: code changes, rollout decisions, or postmortem ownership.

# History and Hypothesis Evaluation

Read [MCP tool discovery](references/mcp-tool-discovery.md) before runtime calls. Runtime History reads do not require active agents or live-source preflight.

1. Frame and rank the hypotheses, identifying the historical values or execution-path evidence that would strengthen or weaken each.
2. When stored snapshots can supply a hypothesis signal, query and evaluate suitable history using the shared history guidance.
3. Update the hypotheses from suitable captures. Diagnose without a new action when that evidence is sufficient.
4. Otherwise, state the remaining evidence gap, preserve useful historical findings, and continue through the existing live preflight and capture workflow.

# Runtime Preflight

Before creating a live action for a remaining diagnostic gap, complete [source-discovery preflight](references/mcp-tool-discovery.md#source-discovery-preflight). Treat exposed tool schemas and descriptions as authoritative. Preflight failure must not block a diagnosis already supported by sufficient historical evidence.

- Inspect relevant code before capture. Place each probe on an executable location tied to a hypothesis discriminator.
- If preflight fails, do not invoke capture tools; return the blocker, required remediation, and retry condition.
- Select the strongest target from available metadata. Ask for clarification only when viable candidates remain indistinguishable.
- On a later run, re-enumerate exposed tools and check inherited async action IDs before creating replacements.

# Async Activation

When live evidence is still needed, use async when reproduction is unavailable, uncertain, or intermittent, or after two consecutive no-hit/timeouts from correctly targeted synchronous captures on the active codepath. Prefer same-run capture when the user can reproduce now; after its first failure, decide whether to retarget synchronously or activate async.

# Async Runtime Actions

When async mode is activated or existing action IDs are provided, read [Async runtime actions](references/async-actions.md) before creating, polling, retrieving, or cancelling actions.

# Runtime Action Cleanup

Run cleanup only when this investigation created or explicitly inherited runtime actions.

- Track each owned action ID and purpose.
- Cancel an owned action when it is stale, duplicate, mistargeted, superseded, or no longer needed.
- Retain an owned action only when a concrete next reproduction depends on it; record the reason and expected window.
- Never cancel any other action.
- Include each owned action's `cancelled`, `retained`, or terminal disposition in the handoff.

# Investigation Efficiency

- Use tool-default collection timing unless another window has a concrete diagnostic benefit.
- Keep evidence collection focused on actions that can change diagnosis or next steps.
- After each result, reassess information gain. If repeated actions do not change the hypothesis ranking or next step, change the source, location, signal, or hypothesis before continuing.
- Once the bug mechanism is sufficiently confirmed for the current user-impact question, prefer synthesis and handoff over additional broad sampling.

# Action Error Mitigation

- If a runtime action returns no hits or a timeout-related failure:
  - verify whether a custom timeout/window was set,
  - increase the active collection window when the scenario needs more trigger time,
  - ask the user to reproduce again within the updated window.
- For async actions, distinguish:
  - pending/running with no hits yet: `reproduction-required` handoff, keep action active if still needed,
  - terminal with no usable hits: blocker/mitigation path, then either retry with refined targeting or close hypothesis as inconclusive.
- If reproduction is confirmed but action has no hits, treat as targeting mismatch:
  - do not repeat the same reproduction request with unchanged source/location/signal/hypothesis,
  - retarget at least one of source, location, signal definition, or leading hypothesis before next reproduction request.
- Re-check source targeting after timeout/no-hit outcomes:
  - confirm selected source target(s) still match the suspected execution path,
  - ask the user to confirm source choice when confidence drops after failed captures.
- For action errors not covered above or by exposed tool guidance, consult the Lightrun troubleshooting guide:
  - https://docs.lightrun.com/troubleshooting/overview/
  - summarize which troubleshooting path was used and why it fits the observed error.
- Log mitigation decisions in the handoff (what changed and why).

# Bug Explanation and Fix Proposal Standard

- Explain the bug as a concrete execution path:
  - trigger conditions
  - key state values observed at runtime
  - exact code path or branch that produces the failure
  - visible impact for users or system behavior
- For user-reported failures, state the concrete request or runtime state, the expected behavior, and why the observed execution path violates that expectation.
- Tie each diagnosis claim to runtime evidence and code location.
- Diagnose only when evidence connects the trigger, runtime state, executed path, failure point, and impact; occurrence alone is insufficient.
- Propose a concrete fix at code level:
  - file/module to change
  - behavioral change to implement
  - why this change addresses the observed mechanism
  - risk notes and validation checks

# Quick Use Guide

1. Define the investigation question, impact, and expected behavior.
2. Rank the smallest plausible hypothesis set and the historical or live signal that would strengthen or weaken each hypothesis.
3. Inspect exposed tools and check inherited async action IDs before creating replacements.
4. Query suitable history for hypothesis signals and update dispositions after evaluating relevance and completeness. If the diagnosis evidence gate is met, proceed to cleanup and synthesis.
5. For remaining diagnostic gaps, complete live preflight and select the best-fit source and executable location.
6. Collect only evidence that can change a diagnosis decision; retain historical findings and update the hypothesis ranking after each result.
7. Apply the async activation gate when immediate capture is insufficient.
8. Iterate until evidence supports one diagnosis or the remaining hypotheses are inconclusive.
9. When actions are owned, run cleanup; return the applicable outcome below.

# Return One Outcome

Emit only the applicable branch and omit empty fields.

Include established investigation state needed to audit or resume: question, selected source, hypothesis dispositions, evidence-to-code mapping, and owned action dispositions. Omit unavailable fields and workflow narration.

Distinguish historical captures from newly collected evidence. Cite supporting hit IDs when available, capture times, source/version scope, and completeness limits.

## Blocked

- Failed capability and blocker
- Attempted mitigation
- Exact retry condition

## Reproduction Required

Use only when relevant owned actions remain active.

- Owned action state
- Exact reproduction instruction and active window
- Resume condition

## Inconclusive

- Evidence obtained and hypotheses weakened
- Missing discriminator and why it matters
- Safest next probe

## Diagnosed

- Evidence-backed mechanism: trigger → state → path → failure → impact
- Observed facts separated from inference
- Confidence, disconfirming evidence, and remaining uncertainty
- Recommended fix and validation plan

Append cleanup disposition only when owned actions exist.
