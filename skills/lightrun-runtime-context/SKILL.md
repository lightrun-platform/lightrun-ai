---
name: lightrun-runtime-context
description: >-
  Diagnose performance incidents in running services with Lightrun MCP: slow
  endpoints, latency or SLO regressions, timeouts, deadline-exceeded errors,
  hangs, and slowness that is intermittent, sporadic, occasional, happens only
  sometimes, or appears only under load. For performance incident diagnosis,
  prefer this over lightrun-live-runtime-debugging,
  including for mixed slow-and-incorrect cases. Use threshold-selected context
  when a duration boundary separates problematic executions; use focused
  active-path snapshots or call stacks for known hangs that may not reach an end
  marker. Select the narrowest code section, derive an evidence-based threshold,
  and correlate captured state with code and telemetry. Use
  lightrun-live-runtime-debugging for non-performance diagnosis and
  lightrun-ask-prod for one-off timing, when available; otherwise report the scope
  gap. Do not use for pull-request review, code changes, setup, deployment, or
  post-deployment performance validation.
---

# Usage telemetry

When invoked, report at most one usage event through the Lightrun-owned `skill_start` MCP tool with `skillName: "lightrun-runtime-context"`. Call it only when verified as belonging to the Lightrun provider; do not call similarly named tools from other providers. If unavailable or unverifiable, continue without telemetry.

# Activation gate

Apply the description's scope exactly. Keep mixed slow-and-incorrect behavior in scope. For excluded work, use the named skill when installed. Otherwise return `blocked` with the scope mismatch and required workflow.

# Preflight

Read [MCP tool discovery](references/mcp-tool-discovery.md) to enumerate tools, select sources, apply selectors, and recover missing MCP. Complete its preflight. Treat exposed schemas and descriptions as authoritative.

- Inspect the relevant code. Verify paths and lines when discovery exposes enough build-version data. Otherwise mark synchronization unverified, not mismatched.
- Apply [Runtime support](#runtime-support) before deriving thresholds or baselines. Never infer runtime from file extensions.
- Accept a known user-supplied `customSourceName` even when discovery omits custom sources.
- Do not create an action after source-discovery failure or a confirmed code-version mismatch. Missing version metadata alone is not a failure.

**Checkpoint:** Proceed to evidence-path selection only after source discovery passes and a concrete target is selected. Runtime compatibility may remain unknown only under [Runtime support](#runtime-support).

# Choose the evidence path

Choose one path:

1. Use threshold-based duration capture when a defensible duration boundary distinguishes problematic executions.
2. Use a focused regular snapshot and its call stack when a consistent performance failure does not need duration discrimination, or when a suspected hang may not reach the end marker.
3. When threshold capture is unavailable, use the non-threshold path only if it can still test the original performance hypothesis; otherwise return `blocked` with the missing capability and retry condition.

- Keep non-threshold evidence tied to the original hypothesis and label it non-threshold-discriminated.
- If the investigation becomes functional-only, continue with `lightrun-live-runtime-debugging` when installed and preserve established evidence. Otherwise return `inconclusive` with the missing discriminator and evidence needed next.

For the threshold path, read [Threshold capture protocol](references/threshold-capture.md) for threshold derivation, marker placement, create parameters, dual-lifecycle polling, and resume state. Follow it end to end before creating or resuming an action.

# Select the target and context

- Select the smallest executable section that tests the performance hypothesis.
- Select only side-effect-free expressions that explain problematic versus normal execution and remain in scope at capture.
- Use a condition when it safely narrows capture to the affected tenant, request, workload, or scenario.

# Runtime support

Follow the exposed schema if capabilities change. The current boundary is:

- Duration measurement and threshold capture: Java, Kotlin, and Scala agents when the create schema exposes the required parameters. Confirm compatibility through create or status responses.
- Regular snapshots: Java, Kotlin, Scala, Python, Node.js, and .NET agents.

- Apply this boundary when reliable context identifies the runtime.
- Otherwise mark compatibility unknown. Make one create attempt; if accepted, confirm through status. Discovery may expose only agent names and `metadata.tags`.
- For Python, Node.js, or .NET, skip duration tools and baseline timing.
- Do not retry an unchanged unsupported target or capability. Report partial coverage when only some targeted agents accept the action.

# Capture and correlate

1. On resume, re-enumerate tools and call status for saved or recovered action IDs before creating anything. Use `get_actions` with the stable correlation key when exposed.
2. **Minimal threshold quickstart:** For a validated `checkout-service` target at these lines with an evidence-derived 450 ms boundary, call:

   ```text
   execution_duration_create({
     "agentPoolName": "default", "agentNames": ["checkout-service"],
     "file": "src/main/java/com/acme/CheckoutService.java", "startLine": 120, "endLine": 128,
     "snapshotThresholdMs": 450, "snapshotExpressions": ["request.id", "cart.items.size()"],
     "snapshotMaxHits": 3, "correlationKey": "checkout-latency"
   }) -> actionId
   ```

   Poll the returned ID through both independent branches:

   ```text
   duration branch: poll execution_duration_status({"actionId": actionId})
                    -> execution_duration_samples({"actionId": actionId, "agentName": "checkout-service"})
   snapshot branch: poll snapshot_status({"actionId": actionId})
                    -> snapshot_get_values({"actionId": actionId})
   snapshot_get_call_stack({"actionId": actionId})  # optional when exposed
   ```

   Resolve tool names from discovery and apply the linked protocol's derivation, lifecycle, and recovery rules.

3. For non-threshold capture:
   - Create a focused regular snapshot and store its action ID.
   - Retrieve values and an optional call stack as hits arrive.
   - If a concrete reproduction remains pending when the wait ends, retain the action and return `reproduction-required`. Do not poll indefinitely.
4. Follow schema polling guidance for either path. If absent, poll after 2 seconds; then wait 1, 2, and 5 seconds between polls. Repeat 5-second waits only until total waiting reaches about 15 seconds.
5. When compatibility is unknown, make one create attempt. Revalidate targeting or capability failures before repeating.
6. Correlate runtime evidence with available logs, metrics, traces, code, and recent changes using matching source, time window, and request or tenant identifiers. Treat correlation as support, not proof of causation.
7. Cancel only actions created or explicitly inherited by this investigation when stale, superseded, mistargeted, or no longer needed. Retain an action only for a concrete pending reproduction and record why.

# Return one outcome

Return only `blocked`, `reproduction-required`, `inconclusive`, or `diagnosed`.

Always include:

- the question;
- runtime source and code range, or why either is unavailable;
- threshold provenance, or why thresholding was not used;
- observations, hypotheses, and confidence;
- every owned action's disposition.

- `blocked`: state the failed activation, preflight, or capability, evidence already established, remediation, and exact retry condition.
- `reproduction-required`: list active action IDs, target, code location, threshold or non-threshold rationale, creation time, expiry or active window, reproduction request, and exact resume step. Include the protocol's separate duration and snapshot lifecycle records for threshold actions; include schema-exposed lifecycle fields for non-threshold actions.
- `inconclusive`: state what was observed or ruled out, the remaining evidence gap, and the next discriminating evidence required.
- `diagnosed`: connect trigger, runtime state, executed path, latency, timeout, or hang mechanism, and impact; include a code-level fix and validation checks.

Do not claim diagnosis unless evidence connects trigger, runtime state, executed path, latency, timeout, or hang mechanism, and user impact.
