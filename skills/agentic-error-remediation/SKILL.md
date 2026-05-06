---
name: lightrun-agentic-error-remediation
description: >-
  Guide deterministic runtime investigations in environments using Lightrun
  MCP tools, with preflight gating, recovery/resume rules, evidence-first
  diagnosis, and opening a fix proposal in a repository.
---

# Goal

Provide a repeatable runtime debugging workflow that helps QA and engineers investigate incidents to a fix proposal with focused, high-signal runtime evidence.

# Scope

- In scope: problem framing, hypothesis ranking, runtime evidence capture and storage, hypothesis elimination, diagnosis confidence, blocker handling, fix proposal.
- Out of scope: rollout decisions or postmortem ownership.

# Preconditions

- User can access the target service source path and line location.
- Lightrun MCP server is installed and authenticated.
- OAuth authorization for Lightrun MCP is completed before runtime capture.
- A state persistence tool is installed, configured, and authenticated, or a deterministic local state fallback under the Lightrun home directory is writable.

### State persistence tool discovery
This skill requires saving small investigation state (e.g., problem IDs, hypothesis IDs, runtime action IDs, timestamps, or status flags) to persist between unattended sessions.

**Storage Discovery Protocol:**
1. **Identify Tool:** Look for any available MCP tool that supports "storing", "logging", "creating a task", or "writing data".
2. **Priority:**
    - If a Database MCP is available (e.g., SQLite, Postgres), create/use a table named `skill_metadata`.
    - If a Task Tracker MCP is available (e.g., Jira, Linear), create/update a dedicated "State Tracking" issue.
    - If a File System MCP is available, use only a configured external state root outside the repository/workspace.
3. **Local fallback:** If no connected persistent storage is available, use the user-level Lightrun home directory:
    - If `LIGHTRUN_HOME_DIRECTORY` is set, write to `${LIGHTRUN_HOME_DIRECTORY}/lightrun-agentic-error-remediation/state.json`.
    - Otherwise, on Linux/macOS, write to `$HOME/.lightrun/lightrun-agentic-error-remediation/state.json`.
    - Otherwise, on Windows, write to `%USERPROFILE%\.lightrun\lightrun-agentic-error-remediation\state.json`.
4. **Path safety:** Resolve the selected local path before writing. The resolved path must not be inside the current repository/workspace, source subdirectories, generated build/output folders, generic agent memory/conversation folders, or temporary workspace directories.
5. **Requirement:** The storage must be able to hold a simple key-value pair or a small JSON object.
6. **Fail closed:** If no connected storage exists and no valid local state path is writable, do not create ad hoc files and do not ask for interactive clarification. Stop before scheduling asynchronous runtime actions and return a machine-readable blocker: `state-storage-unavailable`.

### Automation and portability
This skill may run unattended and with different coding agents. Do not rely on agent-specific directories or interactive prompts for storage, source-selection, or reproduction decisions. Use the Lightrun home directory convention for local state. If required input is unavailable, return a machine-readable blocker with an explicit retry condition instead of asking the user.

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
- Prefer eliminating wrong hypotheses quickly over collecting broad low-signal data.
- Choose the final diagnosis statement only after all hypotheses are ruled out with runtime evidence.
- Explain the bug mechanism with concrete runtime-to-code traceability.
- End with a concrete fix proposal that includes validation checks.

# Tool Call Timing

- Use tool default collection timing unless the investigation clearly benefits from a different window.
- Avoid adding extra timeout constraints to runtime tool calls during normal investigation flow.
- When timing is adjusted, include a short reason describing the expected diagnostic benefit.
- Before using asynchronous tools, verify state storage is available. If it is unavailable, return `state-storage-unavailable` and do not schedule runtime actions. When state storage is available, save action IDs and related investigation state before ending the run.

# Action Error Mitigation

- If a runtime action returns no hits or a timeout-related failure:
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

# Quick Use Guide

Use this skill in the following sequence:

1. Define the investigation question in one sentence.
2. Verify whether the problem is new or known using the persistent storage.
3. If the problem is known according to the persistent storage,
   3.1. Read hypothesis and requests identifiers from a persistent storage.
   3.2. Verify related Lightrun action statuses using request IDs from persistent storage.
   3.3. If any related Lightrun actions are in progress, notify the user that the known problem already has active runtime actions, finish the current investigation run, and stop the chat immediately.
   3.4. If all related Lightrun actions failed and no further hypothesis investigation is possible from them, cancel these snapshots, quit the known-problem logical branch, and continue as if there were no snapshots at all; add new snapshots only after generating a new hypothesis.
   3.5. Update hypothesis statuses after verification of signals using requests IDs from persistent storage.
   3.6. Summarize final diagnosis based on hypothesis statuses.
   3.7. Publish a fix proposal in a repository.
   3.8. Finish the current investigation run and stop the chat immediately
4. List top hypotheses and the signal expected for each.
5. Run preflight and pick runtime source target.
6. Request focused runtime signals with the minimal useful tool set.
7. Save the unique problem ID, hypotheses, and request identifiers to the selected persistent state storage.
8. Finish the current investigation run and stop the chat immediately

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
    3.1. Read hypothesis and requests identifiers from a persistent storage.
      - Tools: choose from currently available MCPs persistent storage tools based on their descriptions and fit to the problem's description. 
      - Success: hypothesis and requests identifiers correspond to the problem's description are found in the persistent storage.
    3.2. Verify related Lightrun action statuses using request IDs from persistent storage.
      - Tools: choose from currently available Lightrun MCP runtime tools based on their descriptions and fit to the task of getting information about completed actions by request ID. 
      - Success: related Lightrun action statuses are known.
    3.3. If any related Lightrun actions are in progress, notify the user that the known problem already has active runtime actions, finish the current investigation run, and stop the chat immediately.
      - Tools: none
      - Success: The conversation has finished.
    3.4. If all related Lightrun actions failed and no further hypothesis investigation is possible from them, cancel these snapshots, quit the known-problem logical branch, and continue as if there were no snapshots at all; add new snapshots only after generating a new hypothesis.
      - Tools: choose from currently available Lightrun MCP runtime tools based on their descriptions and fit to the task of cancelling snapshots by request ID.
      - Success: failed snapshots are cancelled when cancellation is available, and the investigation returns to new hypothesis generation without treating prior snapshots as usable evidence.
    3.5. Update hypothesis statuses after verification of signals using requests IDs from persistent storage.
      - Tools: choose from currently available Lightrun MCP runtime tools based on their descriptions and fit to the task of getting information about completed actions by request ID. 
      - Success: hypothesis statuses are updated according to the runtime tool results.  
    3.6. Summarize final diagnosis based on hypothesis statuses.
      - Tools: none
      - Success: final diagnosis is summarized based on hypothesis statuses.
    3.7. If the final diagnosis is conclusive.
        3.8. Publish a fix proposal in a repository.
          - Tools: choose from currently available MCPs for source code management based on their descriptions. 
          - Success: PR is created with the fix proposal. 
        3.9. Produce decision-ready handoff.
          - Tools: none
          - Success: output contract is fully populated with diagnosis quality fields and concrete fix proposal details.
        3.10. Finish the current investigation run and stop the chat immediately.
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
7. Execute focused evidence steps per hypothesis, saving action IDs to the selected persistent state storage as request IDs.
   - Tools: choose from currently available Lightrun MCP runtime tools based on their descriptions and fit to the active hypothesis.
   - Typical tool categories include expression capture, call path capture, execution frequency, duration, and numeric metric sampling.
   - Success: each evidence step is mapped to one hypothesis.
8. Finish the current investigation run and stop the chat immediately.
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
- Final handoff:
  - selected source target(s) and source-selection note, or `runtime-source-ambiguous` when no defensible target was selected
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
- [ ] Evidence tool selection references MCP tool descriptions and hypothesis fit.
- [ ] Missing-MCP recovery includes MCP install + OAuth authorization + retry.
- [ ] Resume criteria explicitly gates evidence tools on preflight success.
- [ ] Source selection is attempted by the agent first using explicit fit reasoning.
- [ ] Source selection supports one or multiple targets when multi-source coverage improves confidence.
- [ ] Investigation starts with explicit hypotheses before tool selection.
- [ ] Investigation explores full assumed codepath before placing runtime actions.
- [ ] Runtime actions are placed on triggerable executable locations linked to the hypothesis.
- [ ] Non-timeout action errors trigger consultation of the Lightrun troubleshooting guide before concluding.
- [ ] Evidence collection stays aligned with hypothesis-required tools.
- [ ] Runtime evidence is collected whenever feasible, including when the likely cause appears clear.
- [ ] Each evidence step explicitly confirms or falsifies at least one hypothesis.
- [ ] Output contract includes explicit pass/fail/blocker/handoff states.
- [ ] Findings separate direct runtime facts from inferred conclusions.
- [ ] Final handoff includes a diagnosis statement, confidence, and ruled-out hypotheses.
- [ ] Final handoff explains bug mechanism with concrete runtime-to-code traceability.
- [ ] Final handoff includes a concrete code-fix proposal with validation checks.
