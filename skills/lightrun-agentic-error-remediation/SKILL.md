---
name: lightrun-agentic-error-remediation
description: >-
  Guide deterministic runtime investigations in environments using Lightrun
  MCP tools, with preflight gating, recovery/resume rules, evidence-first
  diagnosis, PR-first fix proposal delivery, and local source-code fallback
  only when PR creation is not possible.
---

# Goal

Provide a repeatable runtime debugging workflow that helps QA and engineers investigate incidents to a fix proposal with focused, high-signal runtime evidence.

# Scope

- In scope: problem framing, hypothesis ranking, runtime evidence capture and storage, hypothesis elimination, diagnosis confidence, blocker handling, PR-first fix proposal delivery, source-code fallback when PR creation is blocked.
- Out of scope: rollout decisions or postmortem ownership.

# Preconditions

- User can access the target service source path and line location.
- Lightrun MCP server is installed and authenticated.
- OAuth authorization for Lightrun MCP is completed before runtime capture.
- An MCP-backed state persistence tool is installed, configured, authenticated, and writable.

### State persistence tool discovery
This skill requires saving small investigation state (e.g., problem IDs, hypothesis IDs, runtime action IDs, timestamps, status flags, or cleanup status) to persist between unattended sessions.

**Storage Discovery Protocol:**
1. **Identify MCP storage:** Inspect available MCP tools/resources for durable storage before scheduling asynchronous runtime actions. Prefer tools described as "memory", "memories", "knowledge", "notes", "state", "database", "records", "tasks", "issues", "cases", "notebooks", or "documents".
2. **Priority:**
    - Highest priority: use a Memories/Memory MCP if available. Store one durable record per unique problem ID and update it as hypotheses, runtime action IDs, outputs, and cleanup status change.
    - Next priority: use another connected MCP-backed persistent store, such as a database, task tracker, case tracker, notebook, wiki, document store, or repository-native issue/PR metadata.
    - Lowest priority: use an MCP-exposed filesystem only when it is clearly a configured remote/shared persistent store. Do not use local filesystem paths through shell tools or filesystem MCPs for investigation state.
3. **No local-file fallback:** Do not write persistent investigation state to repository files, workspace files, temp directories, agent memory/conversation folders, user-home directories, or any ad hoc local `state.json`.
4. **Requirement:** The selected MCP-backed storage must be able to read and update a simple key-value pair or small JSON-like object keyed by a stable problem ID.
5. **Fail closed:** If no MCP-backed persistent storage is available and writable, do not create ad hoc files and do not ask for interactive clarification. Stop before scheduling asynchronous runtime actions and return a machine-readable blocker: `state-storage-unavailable`.

### Automation and portability
This skill may run unattended and with different coding agents. Do not rely on agent-specific directories, local files, user-home directories, or interactive prompts for storage, source-selection, or reproduction decisions. Use MCP-backed persistent storage only. If required input is unavailable, return a machine-readable blocker with an explicit retry condition instead of asking the user.

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

# Runtime Tool Selection Strategy

- Keep preflight fixed on `lightrun__get_runtime_sources`.
- For evidence collection, explore available Lightrun MCP tools and their descriptions, then select the best-fit async tool for each hypothesis signal.
- Record the selected tool identifier exactly as exposed by MCP.
- Prefer the smallest useful set of tools for each investigation step.

# Source Selection Confidence

- First evaluate candidate agent/tag/custom-source options and choose the best-fit target when confidence is sufficient.
- If several targets can fit, select one or multiple strongest candidates using explicit reasoning (service ownership, environment match, and expected trigger path).
- Choose the most appropriate option automatically when confidence remains low but a defensible target exists; if no defensible target can be selected without guessing, return `runtime-source-ambiguous` with an explicit retry condition.

# Investigation Principles

- Start with hypotheses first, then choose tools.
- Capture evidence that can confirm or falsify a specific hypothesis.
- Collect runtime evidence whenever feasible, even when a bug cause appears obvious.
- Treat scheduled Lightrun runtime actions as evidence gates, not as optional supporting data.
- Do not make a final diagnosis, open a PR, implement a fallback fix, or cancel a still-relevant action until its required results have been retrieved or the action has produced a definitive falsifying outcome.
- Treat terminal no-hit, timeout, failed, cancelled, or error action states without usable results as blockers or reproduction-required handoffs, not as permission to proceed to fix delivery.
- Do not block the active session waiting for a reproduction that may take a long time; check action status once per investigation run, then either retrieve available results or hand off the active reproduction requirement.
- Prefer eliminating wrong hypotheses quickly over collecting broad low-signal data.
- Choose the final diagnosis statement only after all hypotheses are ruled out with runtime evidence.
- Explain the bug mechanism with concrete runtime-to-code traceability.
- End with a concrete fix proposal that includes validation checks.

# Tool Call Timing

- Use tool default collection timing unless the investigation clearly benefits from a different window.
- Avoid adding extra timeout constraints to runtime tool calls during normal investigation flow.
- When timing is adjusted, include a short reason describing the expected diagnostic benefit.
- Before using asynchronous tools, verify state storage is available. If it is unavailable, return `state-storage-unavailable` and do not schedule runtime actions. When state storage is available, save action IDs, cleanup status, and related investigation state before ending the run.

# Mandatory Runtime Action Result Gate

When a Lightrun runtime action is created for a hypothesis, the investigation MUST verify whether the expected result is available before moving to diagnosis or fix delivery. Verification is a single status or result check per investigation run, not an open-ended polling loop.

- Immediately check the action status or result once after creation, and once when resuming a known problem with a persisted action.
- Use the status/result tool that matches the created action type exactly as exposed by MCP. For example, use snapshot status and value/call-stack getters for snapshots, metric result/status tools for metrics, and duration/timing result/status tools for duration actions when those tools are available.
- Do not repeatedly poll, sleep, or wait in the active session for an action that has no usable result yet.
- If the single check reports captured values, call stacks, metric samples, duration measurements, counters, or other expected action output, retrieve and persist that output before taking any other diagnostic conclusion step.
- If status remains pending or running with no usable result, return `reproduction-required` with exact trigger instructions, persist the active action, and leave the action active when it is still needed; do not open a PR, implement a local fix, cancel the action, or issue a final diagnosis.
- If the action reaches a terminal no-hit, timeout, failed, canceled, or error state without usable results, record that result, apply the action error mitigation rules, and return a blocker or `reproduction-required`; do not open a PR, implement a local fix, or issue a final diagnosis unless the terminal action result itself definitively falsifies the hypothesis.
- Do not treat unrelated traces, logs, stack traces, static code inspection, or outputs from different actions as a substitute for retrieving the created action's expected results.
- Do not cancel an action created for the active leading hypothesis merely because a likely fix is clear.

# Runtime Action Cleanup

- Treat every asynchronous Lightrun action created by this skill as owned by the current investigation and persist its action ID, hypothesis ID, purpose, creation time, last known status, and cleanup status.
- Before ending any run, review all actions created by this skill for the active problem and cancel actions that are no longer required.
- Cancel an action when:
  - its hypothesis has been ruled out,
  - a final diagnosis has been reached and the action is not needed for handoff evidence,
  - the action is duplicate, stale, mistargeted, or replaced by a newer action,
  - the investigation is blocked before reproduction and the action would otherwise remain active without a clear retry path.
- Do not cancel actions created by another investigation, another user, or an unknown owner.
- Do not cancel active actions that are still required to collect an expected reproduction signal; leave them active only when the handoff explicitly states why they remain required and how to continue.
- Before cancelling, retrieve any newly available output when the action status reports unread or uncollected results.
- Use the most specific available Lightrun MCP cancellation tool for the action type, record the cancellation result in persistent state, and include a cleanup summary in the handoff.

# Action Error Mitigation

- If a runtime action returns no output, no hits, no samples, no measurements, or a timeout-related failure:
  - verify whether a custom timeout/window was set,
  - increase the active collection window when the scenario needs more trigger time,
  - return `reproduction-required` with the updated action window and explicit retry condition.
- Re-check source targeting after timeout/no-hit outcomes:
  - confirm selected source target(s) still match the suspected execution path,
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

# Fix Proposal Delivery Policy

- Prefer opening a PR with the fix proposal whenever the final diagnosis is conclusive and repository/source-management access makes PR creation possible.
- A final diagnosis is not conclusive while any required runtime action for the active leading hypothesis is still pending, running, or has unread results.
- Treat PR creation as possible when the target repository is identified, source-management tools or MCPs are available and authorized, a branch/commit or equivalent proposal can be created, and the fix can be represented as a patch.
- Before making local source-code edits, check available source-management tools or MCPs for a way to create a PR, pull request, merge request, or repository-native change proposal.
- Do not implement local source-code changes as the primary output when a PR can be opened.
- Implement local source-code changes only as a fallback when PR creation is blocked by missing or unauthorized source-management tools, missing repository/remote information, branch or PR permission failures, or an offline environment.
- When using the local fallback, keep edits limited to source-code files directly required for the fix; do not modify documentation, generated files, build metadata, workflows, or unrelated repository files unless the user explicitly asks.
- Record the delivery mode in the handoff:
  - PR mode: PR URL or identifier, branch name, fix summary, and validation checks.
  - Local fallback mode: PR blocker reason, modified source-code paths, fix summary, and validation checks.

# Quick Use Guide

Use this skill in the following sequence:

1. Define the investigation question in one sentence.
2. Verify whether the problem is new or known using the persistent storage.
3. If the problem is known according to the persistent storage,
   3.1. Read hypothesis and runtime action identifiers from persistent storage.
   3.2. Verify related Lightrun action statuses/results using action IDs or result identifiers from persistent storage.
   3.3. If any related runtime actions are in progress and still required, check status/results once; retrieve results when available, otherwise return `reproduction-required` with the action retained.
   3.4. If all related Lightrun actions failed and no further hypothesis investigation is possible from them, cancel these actions, quit the known-problem logical branch, and continue as if there were no actions at all; add new actions only after generating a new hypothesis.
   3.5. Cancel any related Lightrun actions created by this skill that are no longer required.
   3.6. Update hypothesis statuses after verification of signals using action IDs or result identifiers from persistent storage.
   3.7. Summarize final diagnosis based on hypothesis statuses.
   3.8. Open a PR with the fix proposal when possible; implement a minimal local source-code fallback only if PR creation is blocked.
   3.9. Finish the current investigation run and stop the chat immediately
4. List top hypotheses and the signal expected for each.
5. Run preflight and pick runtime source target.
6. Request focused runtime signals with the minimal useful tool set.
7. Save the unique problem ID, hypotheses, action IDs or result identifiers, and action cleanup status to the selected persistent state storage.
8. For every created runtime action, check status/results once using the mandatory runtime action result gate before diagnosis or fix delivery.
9. If an action needs external reproduction and has no usable result yet, return `reproduction-required` with the active action ID, reproduction steps, and retry condition, then stop.
10. Finish the current investigation run only after retrieved runtime action results, blocker, or reproduction-required handoff has been persisted.

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
2. Verify whether the problem is new or known using the persistent storage.
    - Tools: choose from currently available MCPs persistent storage tools based on their descriptions and fit to the problem's description.
    - Success: the problem is verified against the list of known problems in the persistent storage.
3. If the problem is known according to the persistent storage,
    3.1. Read hypothesis and runtime action identifiers from persistent storage.
      - Tools: choose from currently available MCPs persistent storage tools based on their descriptions and fit to the problem's description. 
      - Success: hypothesis and runtime action identifiers that correspond to the problem description are found in persistent storage.
    3.2. Verify related Lightrun action statuses/results using action IDs or result identifiers from persistent storage.
      - Tools: choose from currently available Lightrun MCP runtime tools based on their descriptions and fit to the task of getting information about completed actions by action ID or result identifier. 
      - Success: related Lightrun action statuses are known.
    3.3. If any related runtime actions are in progress and still required, check their status/results once according to the mandatory runtime action result gate.
      - Tools: choose from currently available Lightrun MCP runtime tools based on their descriptions and fit to checking the action type and retrieving its output.
      - Success: runtime action results are retrieved, or a blocker/reproduction-required handoff is produced without fix delivery.
    3.4. If all related Lightrun actions failed and no further hypothesis investigation is possible from them, cancel these actions, quit the known-problem logical branch, and continue as if there were no actions at all; add new actions only after generating a new hypothesis.
      - Tools: choose from currently available Lightrun MCP runtime tools based on their descriptions and fit to the task of cancelling actions by action ID.
      - Success: failed actions are cancelled when cancellation is available, and the investigation returns to new hypothesis generation without treating prior actions as usable evidence.
    3.5. Cancel any related Lightrun actions created by this skill that are no longer required.
      - Tools: choose from currently available Lightrun MCP runtime tools based on their descriptions and fit to the task of cancelling active actions by action ID.
      - Success: unneeded skill-owned actions are canceled or explicitly retained with a current purpose, and cleanup status is saved.
    3.6. Update hypothesis statuses after verification of signals using action IDs or result identifiers from persistent storage.
      - Tools: choose from currently available Lightrun MCP runtime tools based on their descriptions and fit to the task of getting information about completed actions by action ID or result identifier. 
      - Success: hypothesis statuses are updated according to the runtime tool results.  
    3.7. Summarize final diagnosis based on hypothesis statuses.
      - Tools: none
      - Success: final diagnosis is summarized based on hypothesis statuses.
    3.8. If the final diagnosis is conclusive.
        3.9. Open a PR with the fix proposal when possible.
          - Tools: choose from currently available MCPs or CLIs for source code management based on their descriptions.
          - Success: PR is created with the fix proposal, and the PR URL or identifier is recorded.
        3.10. If PR creation is not possible, implement a minimal local source-code fallback.
          - Tools: local source-code editing tools only.
          - Success: local edits are limited to source-code files directly required for the fix, and the PR blocker reason is recorded.
        3.11. Produce decision-ready handoff.
          - Tools: none
          - Success: output contract is fully populated with diagnosis quality fields and concrete fix proposal details.
        3.12. Finish the current investigation run and stop the chat immediately.
          - Tools: none
          - Success: The conversation has finished.
4. Create a hypothesis matrix.
   - Tools: none
   - Success: at least 2 plausible hypotheses are listed, each with a planned confirming and falsifying signal.
5. Run preflight and select a target source.
   - Tools: `lightrun__get_runtime_sources`
   - Success: one or multiple source targets are selected and justified by agent reasoning without interactive clarification, or `runtime-source-ambiguous` is returned with an explicit retry condition.
6. Map full codepath and choose triggerable evidence points.
   - Tools: none
   - Success: full assumed bug codepath is explored, and action points are placed on executable lines likely to trigger in reproduction.
7. Execute focused evidence steps per hypothesis, saving action IDs or result identifiers to the selected persistent state storage.
   - Tools: choose from currently available Lightrun MCP runtime tools based on their descriptions and fit to the active hypothesis.
   - Typical tool categories include expression capture, call path capture, execution frequency, duration, and numeric metric sampling.
   - Success: each evidence step is mapped to one hypothesis, and every created action has a persisted cleanup status.
8. Check every runtime action required by the active hypotheses once.
   - Tools: choose from currently available Lightrun MCP runtime tools based on their descriptions and fit to checking each action type and retrieving captured output.
   - Success: expected action output is retrieved and persisted, or a blocker/reproduction-required handoff is returned without fix delivery.
9. Finish the current investigation run only after the mandatory runtime action result gate has been satisfied by retrieved results, or a blocker/reproduction-required handoff has been persisted.
     - Tools: none
     - Success: The conversation has finished.

# Output Contract

- Preflight pass:
  - selected source target(s) (`agentPoolName` + selector mode(s))
  - next runtime action (first evidence tool and why)
- Preflight fail:
  - blocker category
  - exact remediation required
  - explicit retry condition
- Runtime blocker:
  - blocker reason, including `state-storage-unavailable` when safe state persistence cannot be selected
  - failed `lightrun__*` tool, when applicable
  - reason/error class
  - mitigation applied (timeout/window update and/or source revalidation)
  - troubleshooting reference used (when applicable)
  - immediate next action
- Reproduction required:
  - active runtime action ID(s), grouped by action type
  - selected source target(s)
  - exact reproduction instruction
  - action time window used
  - expected captured output, such as values, call stack, metric sample, counter, or duration
  - retry condition
  - persisted state record identifier or MCP location
- Final handoff:
  - selected source target(s) and source-selection note, or `runtime-source-ambiguous` when no defensible target was selected
  - reproduction instruction + action time window used
  - investigation question
  - hypothesis matrix result (leading, ruled out, inconclusive)
  - evidence summary (facts first)
  - runtime action cleanup summary (cancelled actions, retained actions, and reasons)
  - bug mechanism summary (trigger, path, failure point, impact)
  - diagnosis statement with confidence level
  - disconfirming evidence considered
  - remaining unknowns and why they matter
  - concrete code-fix proposal (target files/modules, behavior change, validation plan)
  - fix delivery artifact (PR URL or identifier when opened; otherwise PR blocker reason and local source-code paths changed)
  - recommended next step
  - artifact identifier or MCP location + checklist status

# Runtime Quality Checklist

- [ ] `lightrun__get_runtime_sources` appears before any runtime evidence tool.
- [ ] For each selected runtime tool, the identifier matches the MCP-exposed name exactly.
- [ ] Evidence tool selection references MCP tool descriptions and hypothesis fit.
- [ ] Missing-MCP recovery includes MCP install + OAuth authorization + retry.
- [ ] Resume criteria explicitly gates evidence tools on preflight success.
- [ ] Source selection is attempted by the agent first using explicit fit reasoning.
- [ ] Source selection supports one or multiple targets when multi-source coverage improves confidence.
- [ ] Investigation starts with explicit hypotheses before tool selection.
- [ ] Investigation explores full assumed codepath before placing runtime actions.
- [ ] Runtime actions are placed on triggerable executable locations linked to the hypothesis.
- [ ] Every created runtime action is checked exactly once per investigation run; results are retrieved when available, otherwise blocker/reproduction-required handoff is produced without fix delivery.
- [ ] No PR, local fix, final diagnosis, or cancellation occurs while a required runtime action is still pending/running or has unread results.
- [ ] Runtime actions created by this skill are cancelled when no longer required, or explicitly retained with a current purpose.
- [ ] Action cleanup status is persisted before the run ends.
- [ ] Non-timeout action errors trigger consultation of the Lightrun troubleshooting guide before concluding.
- [ ] Evidence collection stays aligned with hypothesis-required tools.
- [ ] Runtime evidence is collected whenever feasible, including when the likely cause appears clear.
- [ ] Each evidence step explicitly confirms or falsifies at least one hypothesis.
- [ ] Output contract includes explicit pass/fail/blocker/handoff states.
- [ ] Findings separate direct runtime facts from inferred conclusions.
- [ ] Final handoff includes a diagnosis statement, confidence, and ruled-out hypotheses.
- [ ] Final handoff explains bug mechanism with concrete runtime-to-code traceability.
- [ ] Final handoff includes a concrete code-fix proposal with validation checks.
- [ ] A PR with the fix proposal is opened whenever possible.
- [ ] Local source-code changes are implemented only as a fallback after recording why PR creation was not possible.
