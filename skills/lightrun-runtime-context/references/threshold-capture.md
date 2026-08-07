# Threshold capture protocol

Read this reference only after selecting threshold-based duration capture. Treat exposed Lightrun MCP schemas and descriptions as authoritative.

## Derive and validate the threshold

1. Derive the boundary from a timeout, deadline, SLO, reported slow limit, or existing logs, metrics, or traces. Never invent one.
2. If no boundary exists, collect baseline timing with a synchronous or non-threshold duration tool only when the target runtime supports it. Use the baseline only to derive the threshold.
3. Account for strict semantics: capture occurs only when elapsed duration is greater than `snapshotThresholdMs`.
   - To capture near-timeout or near-SLO executions, choose an evidence-based threshold slightly below the limit and record the margin.
   - Use the limit itself only when executions beyond it are the target.
4. Normalize the threshold to the schema unit. `snapshotThresholdMs` uses milliseconds. Record the value, source, and rationale.
5. Require an async execution-duration create schema with threshold and snapshot-expression parameters, plus duration status/samples and snapshot status/values. Synchronous-only duration or missing threshold parameters cannot provide this evidence.
6. Do not combine unrelated actions or substitute historical/query tools to imitate threshold capture. User-supplied historical results may provide supporting context.

## Place and configure the action

1. Place start and end markers around the smallest section that tests the performance hypothesis.
2. Remember that duration includes the end line and `snapshotExpressions` are evaluated there. Place the end marker after the relevant work while every expression remains in scope.
3. Use one relevant exit path when the schema accepts one end line. Create separate focused actions for additional exits.

Use a bounded lifetime and set `snapshotMaxHits` to the smallest useful count. Raise it above one only when comparing slow executions can change the diagnosis.

## Track and poll

Define a **lifecycle record** as `state`, `outcome`, `terminationCause` when present, and summed `stats[].count`.

1. Store the action ID, source, location, purpose, creation time, threshold provenance, and separate duration and snapshot lifecycle records.
2. Poll at the bounded cadence defined in the main workflow:
   - Poll duration status and update its lifecycle record.
   - When the summed duration count increases, identify every agent whose count increased. Retrieve samples once per changed agent, passing all schema-required filters, including `agentName` when required.
   - Poll snapshot status independently with the same action ID and update its lifecycle record.
   - When the summed snapshot count increases, retrieve snapshot values. Use a separate call-stack retrieval tool with the same action ID only when its schema permits it.
   - Use each hit's `executionDurationMs`, when exposed, to associate captured state with its slow execution.
3. Keep the lifecycles independent:
   - Snapshot `FINISHED` after `snapshotMaxHits` does not require duration completion before diagnosis.
   - Duration `FINISHED` or duration `totalHits` does not stop snapshot polling.
   - Continue each lifecycle until its own terminal state or the bounded wait ends. Never infer snapshot readiness from duration state or counts.
4. When the bounded wait ends with a concrete reproduction pending, retain the action and return `reproduction-required` with both lifecycle records and the exact resume step. Do not poll indefinitely.
5. For terminal no-hit, targeting, or compatibility failures, revalidate before repeating. If the end marker cannot execute, return to the evidence-path decision and retarget.
