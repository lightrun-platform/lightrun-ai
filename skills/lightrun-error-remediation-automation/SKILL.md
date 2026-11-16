---
name: lightrun-error-remediation-automation
description: >-
  Guide deterministic runtime investigations in environments using Lightrun
  MCP tools, using suitable Runtime History before new live actions, with
  preflight gating, recovery/resume rules, evidence-first diagnosis, PR-first
  fix proposal delivery, and local source-code fallback only when PR creation
  is not possible.
---

# Usage Tracking Preflight

Before any other action in this skill, find every available MCP tool instance named `skill_start`, including provider-prefixed or namespaced instances. Call each instance with `skillName: "lightrun-error-remediation-automation"` to record skill usage; do not stop after the first occurrence. Treat empty responses, unavailable tools, or tracking failures as non-blocking and continue the workflow.

# Goal

Provide a repeatable runtime debugging workflow that helps QA and engineers investigate incidents to a fix proposal with suitable historical evidence and focused live evidence for remaining gaps.

# Scope

- In scope: problem framing, hypothesis ranking, Runtime History evaluation, runtime evidence capture and storage, hypothesis elimination, diagnosis confidence, blocker handling, PR-first fix proposal delivery, source-code fallback when PR creation is blocked.
- Out of scope: rollout decisions or postmortem ownership.

# Preconditions

- User can access the target service source path and line location.
- Lightrun MCP server is installed and authenticated.
- OAuth authorization for Lightrun MCP is completed before runtime capture.
- A durable state persistence tool or facility is installed/configured and writable when asynchronous live actions or resumable investigation state require it. A standalone history read does not require persistent state.

### State persistence tool discovery
When asynchronous live actions or durable resumption are used, this skill requires saving small investigation state (e.g., problem IDs, hypothesis IDs, runtime action IDs, timestamps, status flags, or cleanup status) to persist between unattended sessions. A standalone history-only investigation does not require state persistence.

**Storage Discovery Protocol:**
1. **Identify durable storage:** Inspect available MCP tools/resources and agent-provided durable automation memory facilities before scheduling asynchronous runtime actions. Prefer tools or facilities described as "AutomationMemory", "memory", "memories", "knowledge", "notes", "state", "database", "records", "tasks", "issues", "cases", "notebooks", or "documents".
2. **Priority:**
    - Highest priority: use a Memories/Memory MCP, Cursor `AutomationMemory`, or an equivalent agent-native durable automation memory when available. Store one durable record per unique problem ID and update it as hypotheses, runtime action IDs, outputs, and cleanup status change.
    - Next priority: use another connected persistent store, such as a database, task tracker, case tracker, notebook, wiki, document store, or repository-native issue/PR metadata exposed through MCP or the host agent.
    - Lowest priority: use an MCP-exposed filesystem only when it is clearly a configured remote/shared persistent store. Do not use local filesystem paths through shell tools or filesystem MCPs for investigation state.
3. **No local-file fallback:** Do not write persistent investigation state to repository files, workspace files, temp directories, agent conversation logs, agent-specific local memory folders, user-home directories, or any ad hoc local `state.json`. This ban does not apply to an explicitly exposed durable automation memory facility such as Cursor `AutomationMemory` when it supports read/update by stable key.
4. **Requirement:** The selected storage must be able to read and update a simple key-value pair or small JSON-like object keyed by a stable problem ID.
5. **Fail closed:** If no approved persistent storage is available and writable, do not create ad hoc files and do not ask for interactive clarification. Stop before scheduling asynchronous runtime actions and return a machine-readable blocker: `state-storage-unavailable`.

### Automation and portability
This skill may run unattended and with different coding agents. Do not rely on agent-specific directories, local files, user-home directories, or interactive prompts for storage, source-selection, or reproduction decisions. Use approved durable persistent storage only: a Memories/Memory MCP, Cursor `AutomationMemory`, an equivalent agent-native durable automation memory, or another connected persistent store. If required input is unavailable, return a machine-readable blocker with an explicit retry condition instead of asking the user.

# MCP Tool Discovery

Follow [MCP tool discovery](references/mcp-tool-discovery.md). Do not assume fixed tool names or client prefixes.

# MCP Preflight

Complete source discovery per [MCP tool discovery](references/mcp-tool-discovery.md) before creating a new live action. History reads do not require live-source preflight.

- Pass and fail criteria: see [Source discovery (preflight)](references/mcp-tool-discovery.md#source-discovery-preflight).
- Missing-MCP recovery: see [Missing-MCP recovery](references/mcp-tool-discovery.md#missing-mcp-recovery).

# Resume Criteria

- Resume live evidence collection after preflight pass criteria are met. Runtime History reads may be used without live-source preflight.
- Live evidence tools are activated only after preflight success.
- Re-enumerate exposed Lightrun MCP tools when resuming a later run.

# Runtime Tool Selection Strategy

- At run start, inspect currently exposed Lightrun runtime tools and their descriptions before selecting an evidence path.
- For each hypothesis signal, use suitable Runtime History when stored captures can provide it. Select the best-fit async live tool only for a remaining evidence gap.
- Record the selected tool identifier exactly as exposed by MCP.
- Prefer the smallest useful set of tools for each investigation step.

# Historical Evidence Decision

Follow [Runtime History reads](references/mcp-tool-discovery.md#runtime-history-reads). Query suitable history after ranking hypotheses and before creating new live actions. Evaluate capture time, source and version scope, code location, selected-value outcomes, and completeness. Empty history does not prove non-execution, and historical hit counts are not execution frequency. When history is unavailable or insufficient, preserve useful findings and continue with live capture only for the remaining evidence gap.

# Source Selection Confidence

Apply this section only when selecting a target for a new live action.

- Apply [Source selection guidance](references/mcp-tool-discovery.md#source-selection-guidance).
- Choose the most appropriate option automatically when confidence remains low but a defensible target exists; if no defensible target can be selected without guessing, return `runtime-source-ambiguous` with an explicit retry condition.

# Investigation Principles

- Start with hypotheses first, then choose tools.
- Collect suitable historical or live evidence that can confirm or falsify a specific hypothesis, even when a bug cause appears obvious.
- Treat scheduled Lightrun runtime actions as evidence gates, not as optional supporting data.
- Do not make a final diagnosis, open a PR, implement a fallback fix, or cancel a still-relevant action until its required results have been retrieved or the action has produced a definitive falsifying outcome.
- Treat terminal no-hit, timeout, failed, cancelled, or error action states without usable results as blockers or reproduction-required handoffs, not as permission to proceed to fix delivery.
- Do not block the active session waiting for a reproduction that may take a long time; check action status once per investigation run, then either retrieve available results or hand off the active reproduction requirement.
- Prefer eliminating wrong hypotheses quickly over collecting broad low-signal data.
- Choose the final diagnosis statement only after all hypotheses are ruled out with runtime evidence.
- Explain the bug mechanism with concrete runtime-to-code traceability.
- End with a concrete fix proposal that includes validation checks.

# Tool Call Timing

- Use tool default live-collection timing unless the investigation clearly benefits from a different window.
- Avoid adding extra timeout constraints to live runtime actions during normal investigation flow.
- When timing is adjusted, include a short reason describing the expected diagnostic benefit.
- Before using asynchronous live tools, verify state storage is available. If it is unavailable, return `state-storage-unavailable` and do not schedule runtime actions. When state storage is available, save action IDs, cleanup status, and related investigation state before ending the run. Do not gate standalone history reads on storage availability.

# Mandatory Runtime Action Result Gate

When a Lightrun runtime action is created for a hypothesis, the investigation MUST verify whether the expected result is available before moving to diagnosis or fix delivery. Verification is a single status or result check per investigation run, not an open-ended polling loop.

When no live action was created or inherited, suitable Runtime History may support diagnosis and fix-proposal delivery without this gate. History never substitutes for retrieving the expected result of an already required live action.

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
2. When durable state is available or the investigation is being resumed, verify whether the problem is new or known using persistent storage. Otherwise, treat it as a new investigation and continue to hypothesis generation and Runtime History without creating ad hoc state.
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
5. Query suitable Runtime History for those signals.
   - If history is sufficient, update the hypotheses, return the applicable final handoff, and stop. Propose and deliver a fix only when the evidence establishes a bug.
   - Steps 6–11 apply only when history is insufficient and live evidence is still required.
6. If history is insufficient, run preflight and pick a live runtime source target for the remaining gap.
7. Request focused live runtime signals with the minimal useful tool set.
8. Save the unique problem ID, hypotheses, action IDs or result identifiers, and action cleanup status to the selected persistent state storage when live actions or durable resumption require it.
9. For every created runtime action, check status/results once using the mandatory runtime action result gate before diagnosis or fix delivery.
10. If an action needs external reproduction and has no usable result yet, return `reproduction-required` with the active action ID, reproduction steps, and retry condition, then stop.
11. Finish the current investigation run after returning the applicable handoff. When live actions or durable resumption require state persistence, finish only after the required state, results, blocker, or reproduction-required handoff have been persisted.

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
2. When durable state is available or the investigation is being resumed, verify whether the problem is new or known using persistent storage. Otherwise, treat it as a new investigation and proceed without creating ad hoc state.
    - Tools: choose from currently available persistent storage tools/facilities based on their descriptions and fit to the problem's description. No storage tool is required for a standalone history-only investigation.
    - Success: the problem is verified against known persisted state, or the investigation proceeds as new when durable state is unavailable and no live action or durable resumption requires it.
3. If the problem is known according to the persistent storage,
    3.1. Read hypothesis and runtime action identifiers from persistent storage.
      - Tools: choose from currently available persistent storage tools/facilities based on their descriptions and fit to the problem's description.
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
5. Query suitable Runtime History for the hypothesis signals.
   - Tools: exposed Runtime History read tools selected by capability.
   - Success: suitable historical evidence updates the hypotheses, or a specific remaining evidence gap is identified for live capture.
   - History-sufficient branch: update the hypotheses, produce the applicable final handoff, and stop. Propose and deliver a fix only when the evidence establishes a bug.
   - Steps 6–10 are the live-evidence branch and apply only when history is insufficient and live evidence is still required.
6. When live evidence is still required, run preflight and select a target source.
   - Tools: source-discovery capabilities (see MCP Tool Discovery)
   - Success: one or multiple source targets are selected and justified by agent reasoning without interactive clarification, or `runtime-source-ambiguous` is returned with an explicit retry condition.
7. Map full codepath and choose triggerable evidence points.
   - Tools: none
   - Success: full assumed bug codepath is explored, and action points are placed on executable lines likely to trigger in reproduction.
8. Execute focused live evidence steps per hypothesis, saving action IDs or result identifiers to the selected persistent state storage.
   - Tools: choose from currently available Lightrun MCP runtime tools based on their descriptions and fit to the active hypothesis.
   - Typical tool categories include expression capture, call path capture, execution frequency, duration, and numeric metric sampling.
   - Success: each evidence step is mapped to one hypothesis, and every created action has a persisted cleanup status.
9. Check every live runtime action required by the active hypotheses once.
   - Tools: choose from currently available Lightrun MCP runtime tools based on their descriptions and fit to checking each action type and retrieving captured output.
   - Success: expected action output is retrieved and persisted, or a blocker/reproduction-required handoff is returned without fix delivery.
10. Finish the current investigation run only after the mandatory runtime action result gate has been satisfied by retrieved results, or a blocker/reproduction-required handoff has been persisted.
     - Tools: none
     - Success: The conversation has finished.

# Output Contract

- Preflight pass (live actions only):
  - selected source target(s) (`agentPoolName` + selector mode(s))
  - source-selection reasoning and pagination/filter note (if used)
  - next runtime action (first evidence tool and why)
- Preflight fail (live actions only):
  - blocker category
  - exact remediation required
  - explicit retry condition
- Runtime blocker:
  - blocker reason, including `state-storage-unavailable` when safe state persistence cannot be selected
  - failed Lightrun MCP tool, when applicable
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
  - persisted state record identifier, storage location, or MCP location
- Final handoff:
  - investigation question
  - hypothesis matrix result (leading, ruled out, inconclusive)
  - evidence summary (facts first), distinguishing Runtime History from newly captured live evidence
  - when live target selection or live actions were used: selected source target(s) and source-selection note, reproduction instruction and action time window, plus runtime action cleanup summary (cancelled actions, retained actions, and reasons)
  - when live target selection was attempted but no defensible target was found: `runtime-source-ambiguous`
  - bug mechanism summary (trigger, path, failure point, impact)
  - diagnosis statement with confidence level
  - disconfirming evidence considered
  - remaining unknowns and why they matter
  - concrete code-fix proposal (target files/modules, behavior change, validation plan)
  - fix delivery artifact (PR URL or identifier when opened; otherwise PR blocker reason and local source-code paths changed)
  - recommended next step
  - artifact identifier, storage location, or MCP location when state was persisted, plus checklist status

# Runtime Quality Checklist

Checklist items concerning preflight, source selection, live runtime actions, cleanup, or persisted action state apply only when live actions are created or inherited. For a history-only investigation, mark those items not applicable; do not perform live work solely to satisfy the checklist.

- [ ] Suitable Runtime History considered before new live actions when it can provide the needed signal.
- [ ] Source discovery completed before any new live action.
- [ ] Discovery followed tool-description usage flow and pagination when needed.
- [ ] Preflight produced `agentPoolName` plus selector required by the first evidence tool.
- [ ] For each selected runtime tool, the identifier matches the MCP-exposed name exactly.
- [ ] Evidence tool selection references MCP tool descriptions and hypothesis fit.
- [ ] Missing-MCP recovery includes MCP install + OAuth authorization + retry.
- [ ] Resume criteria gates new live actions, but not history reads, on preflight success.
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
