---
name: lightrun-live-runtime-debugging
description: >-
  Guide deterministic runtime investigations in live environments using Lightrun
  MCP tools, with preflight gating, recovery/resume rules, evidence-first
  diagnosis, and explicit blocker/handoff outputs.
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

# Runtime Tool Selection Strategy

- Keep preflight fixed on `lightrun__get_runtime_sources`.
- For evidence collection, explore available Lightrun MCP tools and their descriptions, then select the best-fit tool for each hypothesis signal.
- Record the selected tool identifier exactly as exposed by MCP.
- Prefer the smallest useful set of tools for each investigation step.

# Source Selection Confidence

- First evaluate candidate agent/tag/custom-source options and choose the best-fit target when confidence is sufficient.
- If several targets can fit, select one or multiple strongest candidates using explicit reasoning (service ownership, environment match, and expected trigger path).
- Ask the user for source clarification when confidence remains low after this evaluation.
- When clarification is needed, present a short comparison of candidates and continue after the user selects the source.

# Investigation Principles

- Start with hypotheses first, then choose tools.
- Capture evidence that can confirm or falsify a specific hypothesis.
- Collect runtime evidence whenever feasible, even when a bug cause appears obvious.
- Prefer eliminating wrong hypotheses quickly over collecting broad low-signal data.
- End with a diagnosis statement that includes confidence and remaining uncertainty.

# Tool Call Timing

- Use tool default collection timing unless the investigation clearly benefits from a different window.
- Avoid adding extra timeout constraints to runtime tool calls during normal investigation flow.
- When timing is adjusted, include a short reason describing the expected diagnostic benefit.

# Action Error Mitigation

- If a runtime action returns no hits or a timeout-related failure:
  - verify whether a custom timeout/window was set,
  - increase the active collection window when the scenario needs more trigger time,
  - ask the user to reproduce again within the updated window.
- Re-check source targeting after timeout/no-hit outcomes:
  - confirm selected source target(s) still match the suspected execution path,
  - ask the user to confirm source choice when confidence drops after failed captures.
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
3. Run preflight and pick runtime source target.
4. Collect focused runtime signals with the minimal useful tool set.
5. Update hypothesis status after each signal.
6. Publish diagnosis, confidence, and next best action.

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
3. Run preflight and select a target source.
   - Tools: `lightrun__get_runtime_sources`
   - Success: one or multiple source targets are selected and justified by agent reasoning, with user clarification only when confidence remains low.
4. Map full codepath and choose triggerable evidence points.
   - Tools: none
   - Success: full assumed bug codepath is explored, and action points are placed on executable lines likely to trigger in reproduction.
5. Execute focused evidence steps per hypothesis.
   - Tools: choose from currently available Lightrun MCP runtime tools based on their descriptions and fit to the active hypothesis.
   - Typical tool categories include expression capture, call path capture, execution frequency, duration, and numeric metric sampling.
   - Success: each evidence step is mapped to one hypothesis and either strengthens or weakens it.
6. Ask for issue reproduction within action window.
   - Tools: none
   - Success: user receives clear reproduction instructions and timing window while runtime actions are active.
7. Iterate investigation loop.
   - Tools: same minimal subset as step 5
   - Success: repeat capture and assessment until one leading hypothesis remains, or all hypotheses are inconclusive, including timeout/source mitigation when captures fail.
8. Synthesize diagnosis and confidence.
   - Tools: none
   - Success: findings differentiate facts from inference, ruled-out hypotheses are explicit, and uncertainty is bounded.
9. Produce decision-ready handoff with next actions.
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
  - immediate next action
- Final handoff:
  - selected source target(s) and source-selection note (if user clarification was needed)
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
- [ ] Source uncertainty is escalated to the user when confidence remains low after agent evaluation.
- [ ] Investigation starts with explicit hypotheses before tool selection.
- [ ] Investigation explores full assumed codepath before placing runtime actions.
- [ ] Runtime actions are placed on triggerable executable locations linked to the hypothesis.
- [ ] Reproduction guidance is provided to the user while actions are active.
- [ ] Timeout/no-hit outcomes trigger window review and optional timeout increase before concluding.
- [ ] Source fit is revalidated after failed captures, with user confirmation when confidence drops.
- [ ] Evidence collection stays aligned with hypothesis-required tools.
- [ ] Runtime evidence is collected whenever feasible, including when the likely cause appears clear.
- [ ] Each evidence step explicitly confirms or falsifies at least one hypothesis.
- [ ] Output contract includes explicit pass/fail/blocker/handoff states.
- [ ] Findings separate direct runtime facts from inferred conclusions.
- [ ] Final handoff includes a diagnosis statement, confidence, and ruled-out hypotheses.
- [ ] Final handoff explains bug mechanism with concrete runtime-to-code traceability.
- [ ] Final handoff includes a concrete code-fix proposal with validation checks.
