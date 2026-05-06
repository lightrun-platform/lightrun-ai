---
name: lightrun-live-runtime-debugging
description: >-
  Guides live runtime debugging of running applications using Lightrun MCP tools.
  Use when the user needs to debug a production or staging service without
  redeploying, add dynamic logs to a running process, capture variable snapshots
  at specific code locations, diagnose production errors in real time, investigate
  live incidents by collecting runtime evidence, or trace execution paths and
  metric samples from active Lightrun agents. Covers preflight source validation,
  hypothesis-driven evidence capture, timeout and error recovery, and a
  structured diagnosis handoff with a concrete code-fix proposal.
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

- Required gate tool: `lightrun__get_runtime_sources`
- Pass criteria: at least one valid agent pool is returned and a concrete target is selected (`agentNames`, `customSourceName`, or `tagNames`).
- Fail criteria: tool is unavailable, call fails, or source list is empty.

**Example preflight call:**
```
lightrun__get_runtime_sources()
// Returns: list of agent pools, tags, and custom sources available for targeting
// Example return shape: { agentPools: [{ name: "payments-prod", agents: ["payments-service-prod-1"] }], tags: ["prod", "staging"], customSources: [] }
```

# Missing-MCP Recovery

1. Classify failure: missing tool, runtime call error, or empty source list.
2. Remediate: install/enable Lightrun MCP, complete OAuth authorization, verify access to the expected environment/agent pool.
3. Re-run `lightrun__get_runtime_sources` and continue after preflight success.

Resume: runtime evidence tools are activated only after `lightrun__get_runtime_sources` returns valid sources.

# Source Selection and Revalidation

- Evaluate all candidate agent/tag/custom-source options returned by preflight and choose the best-fit target using service ownership, environment match, and expected trigger path.
- Select one or multiple strongest candidates with explicit reasoning when multiple targets fit.
- Ask for user clarification only when confidence remains low after this evaluation; present a short comparison of candidates and continue after the user selects.
- **Revalidation trigger:** re-confirm source fit after any failed capture (timeout or no-hit). Reassess candidate options and ask the user to confirm if confidence drops below a clear choice. Log the revalidation reason in the final handoff.

# Runtime Tool Selection

- Keep preflight fixed on `lightrun__get_runtime_sources`.
- For evidence collection, inspect available Lightrun MCP tools and their descriptions, then select the best-fit tool for each hypothesis signal.
- Record the selected tool identifier exactly as exposed by MCP.
- Prefer the smallest useful set of tools per investigation step.
- Typical tool categories: expression capture, call-path capture, execution frequency, duration, and numeric metric sampling.

**Example evidence tool call (substitute actual MCP-exposed identifier and all required parameters):**
```
lightrun__add_log(
  agentName: "payments-service-prod-1",   // exact agent name from preflight
  filename: "src/payments/processor.py",  // relative source path
  lineNumber: 142,                         // must be an executable line
  format: "amount={amount} status={status} userId={user_id}"  // expression placeholders
)
// Expected return: { actionId: "<uuid>", status: "ACTIVE", expiresAt: "<ISO timestamp>" }
// Injects a dynamic log at line 142 without redeployment
```

# Investigation Principles

- Start with hypotheses, then choose tools.
- Capture evidence that can confirm or falsify a specific hypothesis.
- Collect runtime evidence whenever feasible, even when a bug cause appears obvious.
- Prefer eliminating wrong hypotheses quickly over collecting broad low-signal data.
- End with a diagnosis statement that includes confidence and remaining uncertainty.

# Tool Call Timing

- Use tool default collection timing unless the investigation clearly benefits from a different window.
- If a runtime action returns no hits or a timeout-related failure: verify whether a custom timeout was set, increase the collection window, and ask the user to reproduce again within the updated window. Then revalidate source targeting (see **Source Selection and Revalidation**).
- When timing is adjusted, include a short reason describing the expected diagnostic benefit.

# Action Error Mitigation

- **No hits / timeout**: increase active collection window, ask for reproduction within the new window. If hits still do not appear, revalidate source targeting (see **Source Selection and Revalidation**).
- **Other errors**: consult the Lightrun troubleshooting guide (https://docs.lightrun.com/troubleshooting/overview/), summarize which path was used and why it fits the observed error.
- Log mitigation decisions in the handoff (what changed and why).

# Bug Explanation and Fix Proposal Standard

- Explain the bug as a concrete execution path: trigger conditions, key state values observed at runtime, exact code path or branch that produces the failure, visible impact.
- Tie each diagnosis claim to runtime evidence and code location.
- Propose a concrete fix: file/module to change, behavioral change to implement, why it addresses the observed mechanism, risk notes and validation checks.

# Quick Use Guide

Investigation template:

```
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
```

# Flow

1. **Frame the problem.** — Symptom, impact, expected behavior, and investigation question are explicit.
2. **Create hypothesis matrix.** — At least 2 plausible hypotheses, each with a planned confirming and falsifying signal.
3. **Run preflight and select source target.** — `lightrun__get_runtime_sources`; target selected and justified per **Source Selection and Revalidation**; user clarification only when confidence is low after agent evaluation.
4. **Map codepath and choose evidence points.** — Full assumed bug codepath explored; action points placed on executable lines likely to trigger during reproduction.
5. **Execute focused evidence steps.** — Choose from available Lightrun MCP runtime tools based on descriptions and hypothesis fit; each step mapped to one hypothesis.
6. **Request reproduction.** — User receives clear reproduction instructions and timing window while runtime actions are active.
7. **Iterate investigation loop.** — Repeat capture and assessment until one leading hypothesis remains or all are inconclusive; apply timeout and source mitigation per **Tool Call Timing** and **Action Error Mitigation** when captures fail.
8. **Synthesize diagnosis.** — Separate runtime facts from inferred conclusions; rule out hypotheses explicitly; bound remaining uncertainty.
9. **Produce decision-ready handoff.** — Fully populate the output contract below.

# Output Contract

**Preflight pass:**
- Selected source target(s) (`agentPoolName` + selector mode)
- Next runtime action (first evidence tool and why)

**Preflight fail:**
- Blocker category
- Exact remediation required
- Explicit retry condition

**Runtime blocker:**
- Failed `lightrun__*` tool + reason/error class
- Mitigation applied (window update and/or source revalidation)
- Troubleshooting reference used (when applicable)
- Immediate next action

**Final handoff:**
- Selected source target(s) and source-selection note (if user clarification was needed)
- Reproduction instruction + action time window used
- Investigation question
- Hypothesis matrix result (leading, ruled out, inconclusive)
- Evidence summary (facts first)
- Bug mechanism summary (trigger, path, failure point, impact)
- Diagnosis statement with confidence level
- Disconfirming evidence considered
- Remaining unknowns and why they matter
- Concrete code-fix proposal (target files/modules, behavior change, validation plan)
- Recommended next step
