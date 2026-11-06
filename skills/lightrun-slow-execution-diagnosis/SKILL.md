---
name: lightrun-slow-execution-diagnosis
description: >-
  Diagnose performance incidents in running services with Lightrun MCP: slow
  endpoints, latency or SLO regressions, timeouts, deadline-exceeded errors,
  hangs, and slowness that is intermittent, sporadic, occasional, happens only
  sometimes, or appears only under load. Use relevant Runtime History for
  state/path context before creating live actions. Prefer this over lightrun-live-runtime-debugging,
  including for mixed slow-and-incorrect cases. Use slow-execution snapshots
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

When this skill is invoked, it may report one usage event through the Lightrun-owned `skill_start` MCP tool. This helps Lightrun measure skill adoption. The request supplies the skill name `lightrun-slow-execution-diagnosis`.

Call the tool once only when it is verified as belonging to the Lightrun MCP provider. Do not call similarly named tools exposed by other providers. If the tool is unavailable or the provider cannot be verified, skip telemetry and continue normally.

# Activation gate

Apply the description's scope exactly. Keep mixed slow-and-incorrect behavior in scope. For excluded work, use the named skill when installed. Otherwise return `blocked` with the scope mismatch and required workflow.

# Historical evidence

Frame the performance hypotheses and identify the evidence that would distinguish them. Follow [Runtime History reads](references/mcp-tool-discovery.md#runtime-history-reads). Treat exposed schemas and descriptions as authoritative. History reads do not require active agents or new-action preflight.

- Inspect the relevant code before interpreting historical or live evidence and declaring a mechanism. Verify paths and lines against captured code identity/version or available build metadata. When synchronization cannot be verified, mark it unverified, not mismatched.
- Consider history before new live actions only when captured state or paths can test a performance hypothesis. Use suitable evidence to identify the executed path or likely slow section, update hypotheses, and choose focused expressions, stack data, executable locations, or probes.
- Evaluate incident/capture time alignment, source/environment scope, repository, commit, file, function, and returned line. Check selected-value outcomes, completeness, and stack/frame truncation before deciding whether a capture distinguishes the hypotheses. Empty history means only no matching capture was found; it does not falsify a hypothesis.
- Skip irrelevant history queries when the missing signal requires a fresh duration, execution count, numeric metric, or slow-execution threshold capture. Historical hit counts are captured samples, not execution frequency. Never infer a latency baseline or threshold from historical snapshots.
- Diagnose without a new action when suitable history, together with trustworthy latency evidence from logs, metrics, traces, or another source, already establishes the required performance mechanism and meets the diagnosis gate below. Apply owned-action cleanup when relevant, then return the outcome.
- If history is missing, stale, incomplete, mismatched, unavailable, or cannot supply the required signal, state the gap, preserve useful findings, and continue with the live workflow for the remaining discriminator.

# Live-action preflight

Complete [source-discovery preflight](references/mcp-tool-discovery.md#source-discovery-preflight) before creating any live action, including baseline timing. Historical conclusions remain subject to the evidence checks above, not live-source availability.

- Apply [Runtime support](#runtime-support) before creating a live action. Never infer runtime from file extensions.
- Accept a known user-supplied `customSourceName` even when discovery omits custom sources.
- Do not create an action after source-discovery failure or a confirmed code-version mismatch. Missing version metadata alone is not a failure.

**Checkpoint:** Run source discovery and select a concrete target before creating a live action. Runtime compatibility may remain unknown only under [Runtime support](#runtime-support). Use existing latency evidence to select the code path and derive a threshold before live-source preflight. Run preflight only when creating a live action.

# Choose the evidence path

For a remaining live evidence need, choose one path:

1. Select problematic executions by duration when a defensible boundary distinguishes them.
2. Use a focused regular snapshot and its call stack when a consistent performance failure does not need duration discrimination, or when a suspected hang may not reach the end marker.
3. If path 1 is unavailable, use the non-threshold path only if it can still test the original performance hypothesis; otherwise return `blocked` with the missing capability and retry condition.

- Keep non-threshold evidence tied to the original hypothesis and label it non-threshold-discriminated.
- If the investigation becomes functional-only, continue with `lightrun-live-runtime-debugging` when installed and preserve established evidence. Otherwise return `inconclusive` with the missing discriminator and evidence needed next.

For path 1, read [Slow-execution snapshot workflow](references/slow-execution-snapshot-workflow.md) for threshold derivation, marker placement, create parameters, dual-lifecycle polling, and resume state. Follow it end to end before creating or resuming an action.

# Select the target and context

- Select the smallest executable section that tests the performance hypothesis.
- Select only side-effect-free expressions that explain problematic versus normal execution and remain in scope at capture.
- Use a condition when it safely narrows capture to the affected tenant, request, workload, or scenario.

# Runtime support

These gates apply to live action creation, including baseline timing, rather than history reads.

| Evidence | Runtime and version gate |
| --- | --- |
| Async duration | Java, Kotlin, or Scala; Lightrun 1.87+; compatible JVM agent |
| Slow-execution snapshot | Same runtimes; Lightrun and JVM agent 1.89+; create schema exposes `snapshotThresholdMs`, `snapshotExpressions`, and `snapshotMaxHits` |
| Regular snapshot | Java, Kotlin, Scala, Python, Node.js, or .NET |

- For non-JVM targets, use regular snapshots; skip duration and baseline timing.
- Never infer versions from agent names or `metadata.tags`.
- Treat missing tools or parameters as a Lightrun/MCP gap. Treat unsupported-version, compatibility, or `returnedError` as an agent gap.
- Name the gap, report partial fleet support, and recommend the applicable version from the table.

# Capture and correlate

1. On resume, re-enumerate tools and call status for saved or recovered action IDs before creating anything. Use `get_actions` with the stable correlation key when exposed.
2. **Minimal slow-execution snapshot quickstart:** For a validated `checkout-service` target at these lines with an evidence-derived 450 ms boundary, call:

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
- historical versus newly captured live evidence, with Runtime History hit IDs and capture times when available, source/version scope, and completeness limitations;
- every owned action's disposition.

- `blocked`: include the failed gate, established evidence, remediation, and exact retry condition.
- `reproduction-required`: include active action IDs, target and location, capture rationale, creation and expiry, reproduction request, and exact resume step. Record both lifecycles for slow-execution snapshot actions; use schema lifecycle fields otherwise.
- `inconclusive`: include observations and rule-outs, the remaining gap, and the next discriminating evidence.
- `diagnosed`: claim only when evidence connects the trigger, runtime state, executed path, latency, timeout, or hang mechanism, and impact. Include a code fix and validation checks.
