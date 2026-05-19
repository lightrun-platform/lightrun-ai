---
name: lightrun-live-runtime-debugging
description: >-
  Guide deterministic runtime investigations in live environments using Lightrun
  MCP tools, with preflight gating, recovery/resume rules, evidence-first
  diagnosis, and explicit blocker/handoff outputs.
version: 0.1.0
---

# Goal

Provide a repeatable live runtime debugging workflow that helps QA and engineers investigate incidents to a diagnosis with focused, high-signal runtime evidence.

# Scope

- In scope: problem framing, hypothesis ranking, runtime evidence capture, hypothesis elimination, diagnosis confidence, blocker handling, and investigation handoff.
- Out of scope: code changes, rollout decisions, or postmortem ownership.

# Preconditions

- User can access the target service source path and line location.
- Lightrun MCP server is installed and authenticated.
- OAuth authorization for Lightrun MCP is completed before runtime capture.

# MCP Preflight

- Required gate tool:
  - `lightrun__get_runtime_sources`
- Pass criteria:
  - At least one valid agent pool is returned.
  - A concrete target is selected: `agentNames` or `customSourceName` or `tagNames`.
- Fail criteria:
  - Tool is unavailable, call fails, or source list is empty.

# Missing-MCP Recovery

1. Classify failure: missing tool, runtime call error, or empty source list.
2. Instruct remediation:
   - Install/enable Lightrun MCP.
   - Complete MCP OAuth authorization.
   - Verify access to the expected environment/agent pool.
3. Re-run `lightrun__get_runtime_sources`.
4. Continue investigation after preflight success.

# Resume Criteria

- Resume the investigation after `lightrun__get_runtime_sources` returns valid sources.
- Runtime evidence tools are activated after preflight success.
- For asynchronous runtime actions, resume by re-checking previously created action IDs before creating duplicate actions.

# Runtime Tool Selection Strategy

- Keep preflight fixed on `lightrun__get_runtime_sources`.
- At run start, inspect currently exposed Lightrun runtime tools and their descriptions before selecting an evidence path.
- For evidence collection, select the best-fit tool set for each hypothesis signal based on both investigation needs and currently exposed capabilities.
- Record the selected tool identifier exactly as exposed by MCP.
- Before each action, state what decision this action can change.
- If an action cannot change any diagnosis decision, do not run it.
- After each action, reassess information gain and change strategy when gain is low for two consecutive actions.
- Avoid repeating similar probes across many locations without new rationale.
- Re-check currently exposed runtime tools when resuming a later run, and adapt the evidence path if available capabilities changed.

# Source Selection Confidence

- First evaluate candidate agent/tag/custom-source options and choose the best-fit target when confidence is sufficient.
- If several targets can fit, select one or multiple strongest candidates using explicit reasoning (service ownership, environment match, and expected trigger path).
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

Use this skill in the following sequence:

1. Define the investigation question in one sentence.
2. List top hypotheses and the signal expected for each.
3. Inspect currently exposed runtime tools and choose the evidence path that best fits the hypotheses and available capabilities.
4. Run preflight and pick runtime source target.
5. Apply async activation gate after failed sync evidence cycles.
6. Update hypothesis status after each retrieved signal.
7. Publish diagnosis, confidence, and next best action, or reproduction-required handoff (async mode only).

Investigation template:

- Question:
- Impact:
- Hypotheses:
  - H1:
    - Confirms when:
    - Weakens when:
  - H2:
    - Confirms when:
    - Weakens when:
- Selected runtime target:
- Signals collected:
- Leading diagnosis:
- Confidence:
- Next action:

# Flow

1. Frame the investigation problem.
   - Tools: none
   - Success: symptom, impact, expected behavior, and investigation question are explicit.
2. Create a hypothesis matrix.
   - Tools: none
   - Success: at least 2 plausible hypotheses are listed, each with a planned confirming and falsifying signal.
   - Each hypothesis must include: what request/state would make this failure unexpected.
3. Inspect currently exposed runtime tools and choose the initial evidence path.
   - Tools: none
   - Success: selected path matches both investigation needs and currently exposed tool capabilities.
4. Run preflight and select a target source.
   - Tools: `lightrun__get_runtime_sources`
   - Success: one or multiple source targets are selected and justified by agent reasoning, with user clarification only when confidence remains low.
5. Map full codepath and choose triggerable evidence points.
   - Tools: none
   - Success: full assumed bug codepath is explored, and action points are placed on executable lines likely to trigger in reproduction.
6. Execute focused same-run evidence steps per hypothesis when immediate capture is feasible.
   - Tools: choose from currently available Lightrun MCP runtime tools based on their descriptions and fit to the active hypothesis.
   - Success: each evidence step is mapped to one hypothesis, changes or preserves a specific decision, and either strengthens or weakens it.
7. Apply async activation gate.
   - Tools: none
   - Success: choose synchronous continuation or asynchronous capture with later resume explicitly; switch to async no later than the second consecutive no-hit/timeout on same codepath.
8. If same-run evidence is insufficient due to longer/uncertain reproduction window, switch to async branch when matching capabilities are currently exposed.
   - Tools: async action creation/status/data retrieval/cancel capabilities, discovered from current MCP tool list.
   - Success: async action IDs are created or resumed only for hypotheses that require delayed evidence.
9. In async branch, run bounded in-session status polling after async action creation.
   - Tools: async status/getter tools.
   - Success: if hits arrive within the in-session polling budget, retrieve them and continue investigation in this run; otherwise proceed to later-resume handoff.
10. In async branch, resume existing async actions before creating new ones.
   - Tools: async status/getter tools for existing action IDs.
   - Success: existing action outcomes are incorporated, duplicate action creation is avoided.
11. Ask for issue reproduction within action window (when async branch is active and bounded in-session polling had no usable results).
   - Tools: none
   - Success: user receives clear reproduction instructions and timing window while runtime actions are active.
12. Iterate investigation loop.
   - Tools: same minimal subset as steps 6-10, depending on active branch
   - Success: repeat capture and assessment until one leading hypothesis remains, or all hypotheses are inconclusive, including timeout/source mitigation when captures fail; reproduced-no-hit loops must include retargeting before next repro request.
13. Synthesize diagnosis and confidence.
   - Tools: none
   - Success: findings differentiate facts from inference, ruled-out hypotheses are explicit, and uncertainty is bounded.
14. Run mandatory runtime action cleanup gate.
   - Tools: async status/cancel capabilities when relevant.
   - Success: each investigation-owned action is marked cancelled/retained/already-terminal, and cancellations are applied when required.
15. Produce decision-ready handoff with next actions.
   - Tools: none
   - Success: output contract is fully populated with diagnosis quality fields and concrete fix proposal details.

# Output Contract

- Preflight pass:
  - selected source target(s) (`agentPoolName` + selector mode(s))
  - next runtime action (first evidence tool and why)
- Preflight fail:
  - blocker category
  - exact remediation required
  - explicit retry condition
- Runtime blocker:
  - failed `lightrun__*` tool
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
  - evidence summary (facts first)
  - bug mechanism summary (trigger, path, failure point, impact)
  - diagnosis statement with confidence level
  - disconfirming evidence considered
  - remaining unknowns and why they matter
  - concrete code-fix proposal (target files/modules, behavior change, validation plan)
  - recommended next step
  - artifact path + checklist status

# Runtime Quality Checklist

- [ ] `lightrun__get_runtime_sources` appears before any runtime evidence tool.
- [ ] For each selected runtime tool, the identifier matches the MCP-exposed name exactly.
- [ ] Each runtime action has an explicit decision it can change.
- [ ] Two consecutive low-information actions trigger strategy change.
- [ ] Missing-MCP recovery includes MCP install + OAuth authorization + retry.
- [ ] Resume criteria explicitly gates evidence tools on preflight success.
- [ ] Source selection is attempted by the agent first using explicit fit reasoning.
- [ ] Source selection supports one or multiple targets when multi-source coverage improves confidence.
- [ ] Source uncertainty is escalated to the user when confidence remains low after agent evaluation.
- [ ] Investigation starts with explicit hypotheses before tool selection.
- [ ] Investigation explores full assumed codepath before placing runtime actions.
- [ ] Runtime actions are placed on triggerable executable locations linked to the hypothesis.
- [ ] Hypotheses include expected-vs-unexpected conditions for concrete request/state context.
- [ ] Async activation gate is evaluated after failed sync cycle(s).
- [ ] Intermittent or low-reproducibility complaints trigger async-first evidence cycle unless strong reason is documented.
- [ ] Existing async action IDs are checked before creating duplicate actions for the same hypothesis signal.
- [ ] Pending/running async actions without results produce reproduction-required handoff instead of premature diagnosis.
- [ ] Mandatory cleanup gate is executed before final handoff.
- [ ] Async actions no longer needed are canceled or explicitly retained with reason and expected expiry.
- [ ] Final handoff includes explicit cleanup summary.
- [ ] Reproduction guidance is provided to the user while actions are active.
- [ ] Timeout/no-hit outcomes trigger window review and optional timeout increase before concluding.
- [ ] Source fit is revalidated after failed captures, with user confirmation when confidence drops.
- [ ] Non-timeout action errors trigger consultation of the Lightrun troubleshooting guide before concluding.
- [ ] Evidence collection stays aligned with hypothesis-required tools.
- [ ] Runtime evidence is collected whenever feasible, including when the likely cause appears clear.
- [ ] Each evidence step explicitly confirms or falsifies at least one hypothesis.
- [ ] Occurrence-only evidence is not used as closure for correctness complaints.
- [ ] Output contract includes explicit pass/fail/blocker/handoff states.
- [ ] Findings separate direct runtime facts from inferred conclusions.
- [ ] Final handoff includes a diagnosis statement, confidence, and ruled-out hypotheses.
- [ ] Final handoff explains bug mechanism with concrete runtime-to-code traceability.
- [ ] Final handoff includes a concrete code-fix proposal with validation checks.
