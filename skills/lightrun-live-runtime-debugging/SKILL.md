---
name: lightrun-live-runtime-debugging
description: >-
  Diagnose live bugs and incidents that require runtime evidence from a running
  service using Lightrun MCP. Use when static analysis or logs cannot explain the
  failure mechanism, when a hypothesis must be tested against live values,
  timing, or control flow, or when resuming Lightrun async actions. Do not use
  for one-off production measurements (use lightrun-ask-prod), pull-request
  review, code changes, setup, or deployment.
---

# Usage Tracking Preflight

Before any other action in this skill, find every available MCP tool instance named `skill_start`, including provider-prefixed or namespaced instances. Call each instance with `skillName: "lightrun-live-runtime-debugging"` to record skill usage; do not stop after the first occurrence. Treat empty responses, unavailable tools, or tracking failures as non-blocking and continue the workflow.

# Goal

Provide a repeatable live runtime debugging workflow that helps QA and engineers investigate incidents to a diagnosis with focused, high-signal runtime evidence.

# Scope

- In scope: problem framing, hypothesis ranking, runtime evidence capture, hypothesis elimination, diagnosis confidence, blocker handling, and investigation handoff.
- Out of scope: code changes, rollout decisions, or postmortem ownership.

# Runtime Preflight

Before the first runtime capture, read [MCP tool discovery](references/mcp-tool-discovery.md) and complete its source-discovery preflight. Treat exposed tool schemas and descriptions as authoritative.

- If preflight fails, do not invoke capture tools; return the blocker, required remediation, and retry condition.
- Select the strongest target from available metadata. Ask for clarification only when viable candidates remain indistinguishable.
- On a later run, re-enumerate exposed tools and check inherited async action IDs before creating replacements.

# Async Activation Gate

- Async mode is optional, but MUST be activated when either condition is met:
  - two consecutive no-hit/timeout outcomes occur on correctly targeted synchronous runtime captures for the active hypothesis, or
  - reproduction is not available in the current session window.
- After the first failed synchronous cycle, force an explicit mode decision:
  - continue with synchronous capture when user can reproduce now in-session,
  - switch to asynchronous capture and pause investigation for resume in a later run when reproduction timing is uncertain or delayed.
- Do not run more than 2 consecutive no-hit synchronous probes on the same codepath before switching to async mode.
- When async mode is activated, create async action(s), persist action IDs, provide reproduction-required handoff, and stop active diagnosis until next run.
- If reproducibility confidence is low or user-reported failure is intermittent, favor async in the first evidence cycle.

# Async Runtime Actions

When async mode is activated or existing action IDs are provided, read [Async runtime actions](references/async-actions.md) before creating, polling, retrieving, or cancelling actions.

# Runtime Action Cleanup

Run cleanup only when this investigation created or explicitly inherited runtime actions.

- Track each owned action ID and purpose.
- Cancel an owned action when it is stale, duplicate, mistargeted, superseded, or no longer needed.
- Retain an owned action only when a concrete next reproduction depends on it; record the reason and expected window.
- Never cancel any other action.
- Include each owned action's `cancelled`, `retained`, or terminal disposition in the handoff.

# Tool Call Timing

- Use tool default collection timing unless the investigation clearly benefits from a different window.
- Avoid adding extra timeout constraints to runtime tool calls during normal investigation flow.
- When timing is adjusted, include a short reason describing the expected diagnostic benefit.
# Investigation Efficiency

- Keep evidence collection focused on actions that can change diagnosis or next steps.
- Once the bug mechanism is sufficiently confirmed for the current user-impact question, prefer synthesis and handoff over additional broad sampling.
- Choose practical capture windows for the current goal; use longer waits only when the expected diagnostic value justifies it.

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
- For other action errors, consult the Lightrun troubleshooting guide and apply the most relevant remediation:
  - https://docs.lightrun.com/troubleshooting/overview/
  - summarize which troubleshooting path was used and why it fits the observed error.
- Log mitigation decisions in the handoff (what changed and why).

# Bug Explanation and Fix Proposal Standard

- Explain the bug as a concrete execution path:
  - trigger conditions
  - key state values observed at runtime
  - exact code path or branch that produces the failure
  - visible impact for users or system behavior
- Tie each diagnosis claim to runtime evidence and code location.
- Diagnose only when evidence connects the trigger, runtime state, executed path, failure point, and impact; occurrence alone is insufficient.
- Propose a concrete fix at code level:
  - file/module to change
  - behavioral change to implement
  - why this change addresses the observed mechanism
  - risk notes and validation checks
- Use clear, specific language and avoid generic filler.

# Quick Use Guide

1. Define the investigation question, impact, and expected behavior.
2. Rank the smallest plausible hypothesis set and the signal that would strengthen or weaken each hypothesis.
3. Inspect exposed runtime tools, complete preflight, and select the best-fit source and evidence path.
4. Collect only evidence that can change a diagnosis decision; update the hypothesis ranking after each result.
5. Apply the async activation gate when immediate capture is insufficient.
6. Iterate until evidence supports one diagnosis or the remaining hypotheses are inconclusive.
7. Run cleanup and return the applicable handoff from the output contract.

# Output Contract

- Preflight pass:
  - selected source target(s) (`agentPoolName` + selector mode(s))
  - source-selection reasoning and pagination/filter note (if used)
  - next runtime action (first evidence tool and why)
- Preflight fail:
  - blocker category
  - exact remediation required
  - explicit retry condition
- Runtime blocker:
  - failed Lightrun MCP tool (exact exposed identifier)
  - reason/error class
  - mitigation applied (timeout/window update and/or source revalidation)
  - troubleshooting reference used (when applicable)
  - immediate next action
- Reproduction required:
  - active async action IDs (only when async branch is active)
  - selected source target(s)
  - exact reproduction instruction
  - action window used
  - expected next signal to capture
  - explicit retry condition
- Mode decision summary:
  - async activation rule met: yes/no
  - selected mode: synchronous continuation or asynchronous capture with later resume
  - if async not activated, explicit reason
- Final handoff:
  - selected source target(s) and source-selection note (if user clarification was needed)
  - async action state summary (only when async branch is used; per action: id, hypothesis mapping, latest status, retrieved hit count, cleanup decision)
  - cleanup summary:
    - cancelled action IDs
    - retained action IDs with reason and expected expiry window
    - already-terminal action IDs
  - reproduction instruction + action time window used
  - investigation question
  - hypothesis matrix result (leading, ruled out, inconclusive)
  - evidence summary separating observed facts from inference
  - bug mechanism summary (trigger, path, failure point, impact)
  - diagnosis statement with confidence level
  - disconfirming evidence considered
  - remaining unknowns and why they matter
  - concrete code-fix proposal (target files/modules, behavior change, validation plan)
  - recommended next step
