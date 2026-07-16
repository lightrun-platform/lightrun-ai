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

# Preconditions

- User can access the target service source path and line location.
- Lightrun MCP server is installed and authenticated.
- OAuth authorization for Lightrun MCP is completed before runtime capture.

# MCP Tool Discovery

Follow [MCP tool discovery](references/mcp-tool-discovery.md). Do not assume fixed tool names or client prefixes.

# MCP Preflight

Complete source discovery per [MCP tool discovery](references/mcp-tool-discovery.md) before any runtime evidence capture.

- Pass and fail criteria: see [Source discovery (preflight)](references/mcp-tool-discovery.md#source-discovery-preflight).
- Missing-MCP recovery: see [Missing-MCP recovery](references/mcp-tool-discovery.md#missing-mcp-recovery).

# Resume Criteria

- Resume the investigation after preflight pass criteria are met.
- Runtime evidence tools are activated only after preflight success.
- For asynchronous runtime actions, resume by re-checking previously created action IDs before creating duplicate actions.
- Re-enumerate exposed Lightrun MCP tools when resuming a later run.

# Runtime Tool Selection Strategy

- At run start, inspect currently exposed Lightrun runtime tools and their descriptions before selecting an evidence path.
- For evidence collection, select the best-fit tool set for each hypothesis signal based on both investigation needs and currently exposed capabilities.
- Record the selected tool identifier exactly as exposed by MCP.
- Before each action, state what decision this action can change.
- If an action cannot change any diagnosis decision, do not run it.
- After each action, reassess information gain and change strategy when gain is low for two consecutive actions.
- Avoid repeating similar probes across many locations without new rationale.
- Re-check currently exposed runtime tools when resuming a later run, and adapt the evidence path if available capabilities changed.

# Source Selection Confidence

- Apply [Source selection guidance](references/mcp-tool-discovery.md#source-selection-guidance).
- Ask the user for source clarification when confidence remains low after this evaluation.
- When clarification is needed, present a short comparison of candidates and continue after the user selects the source.

# Investigation Principles

- Start with hypotheses first, then choose tools.
- Capture evidence that can confirm or falsify a specific hypothesis.
- Collect runtime evidence whenever feasible, even when a bug cause appears obvious.
- For user-complaint investigations, evidence must explain whether the observed failure was expected or unexpected for a concrete request context.
- Prefer regular (non-async) runtime tools for same-run investigation when they can produce required evidence in the current session.
- Use asynchronous runtime actions only when the expected signal likely needs a longer or uncertain reproduction window.
- When async actions are used, treat them as investigation state that can span multiple skill runs.
- Do not issue final diagnosis while required async actions are still pending/running without checking status and available results first.
- Prefer eliminating wrong hypotheses quickly over collecting broad low-signal data.
- End with a diagnosis statement that includes confidence and remaining uncertainty.
- Do not close investigation based only on occurrence evidence; closure requires mechanism evidence linking runtime state to failure path.

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

# Async Runtime Action Protocol

- Use this protocol when async runtime tools are available in MCP.
- Discover currently available runtime tool names from MCP at run time and use the exact exposed identifiers.
- The protocol requires these capabilities:
  - create an async runtime action for a hypothesis signal,
  - check async action status by action ID,
  - retrieve captured values and/or call stack when new hits are available,
  - cancel async action when it is no longer needed.
- At action creation time, persist: `actionId`, hypothesis ID, source target, code location, purpose, creation time, max wait, last known status, last retrieved hit count.
- For uncertain reproduction timing, use a long async window by default (recommended baseline: 1800-3600 seconds), then adjust only with explicit reason.
- After creating an async action, perform bounded in-session status polling for a short but meaningful window.
  - use a small polling budget (for example 60-90 seconds total in-session),
  - if new hits arrive in this window, retrieve data immediately and continue investigation in the same run,
  - if no usable results arrive by budget end, keep action active and switch to reproduction-required handoff for later resume.
- On each new skill run for the same investigation:
  - load persisted action IDs first,
  - call status first for each still-relevant action,
  - retrieve data only when status hit count increased beyond previously retrieved count.
- During bounded in-session polling, stop when status is terminal: `COMPLETED`, `FAILED`, `ERROR`, `TIMEOUT`, `CANCELLED`, or when the in-session polling budget is reached.
- If status reaches terminal during bounded in-session polling and hit count increased since last getter call, perform one final getter call before closing the action outcome.
- If status is still pending/running with no usable results by end of current run, return a handoff that includes:
  - active action IDs,
  - exact reproduction steps,
  - retry condition for the next run.
- Cancel stale or no-longer-needed actions and record cleanup decision in handoff.

# Runtime Action Cleanup Gate

- Cleanup review is mandatory before any final response in both same-run and async branches.
- Maintain an investigation-owned action list for this run/session state, including each created action ID and its purpose.
- Before final handoff, review each investigation-owned action and assign one of:
  - `cancelled` (no longer needed),
  - `retained` (still required for next reproduction window),
  - `already terminal` (completed/failed/timeout/cancelled by system or prior run).
- Cancel an investigation-owned action when:
  - the mapped hypothesis is ruled out or already confirmed with sufficient evidence,
  - the action is duplicate, stale, mistargeted, or replaced by a newer action,
  - the investigation is complete and the action is no longer needed.
- Retain an action only when a concrete next reproduction step depends on it; include retention reason and expected expiry window.
- Do not cancel actions outside the investigation-owned action list.
- Do not emit final handoff until cleanup review is complete and reported.

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
