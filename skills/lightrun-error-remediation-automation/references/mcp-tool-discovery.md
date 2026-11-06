# Lightrun MCP Tool Discovery

Shared rules for all Lightrun skills. MCP tool schemas and descriptions are the source of truth for tool names, parameters, sequencing, and pagination.

## Discovery rules (all steps)

- Do not assume fixed tool names, client prefixes, tool counts, or call order.
- At run start — and again when resuming a later run — enumerate exposed Lightrun MCP tools and read schemas/descriptions before calling.
- Select tools by **capability fit** to the active workflow step.
- Record the exact MCP-exposed identifier used for each call.
- Prefer read-only discovery tools before capture or mutating runtime tools.
- When tool descriptions define a usage flow or sequencing hints, follow them.

## Capability taxonomy

| Capability | Purpose | How to identify |
|------------|---------|-----------------|
| Runtime History reads | Discover captured coverage, query stored hits, retrieve a known hit's details | Read-only descriptions for historical captures; filters and bounded includes, or a hit identifier |
| Runtime-source discovery | List agent pools, agents, tags, and related targeting metadata | Read-only descriptions for discovering live runtime targets |
| Live evidence/action creation | Capture new values, durations, counts, call stacks, numeric metrics | Descriptions for instrumentation at executable locations with runtime targeting |
| Live action lifecycle | Poll status, retrieve captured results, cancel | Descriptions referencing an action identifier returned by creation |

## Runtime History reads

Read suitable history before creating a live action when stored captures can support the current workflow's evidence need. Runtime History may answer a question, strengthen or weaken a hypothesis, support a review, or provide remediation evidence. No active-agent or source-discovery preflight is required for history reads.

- Select history tools and parameters from their exposed schemas and descriptions. Use the narrowest supported code, time, version, and runtime-source scope, bounded results, and only the values, frames, or stack data needed for the decision.
- Validate capture time, source scope, code identity/version, selected-value outcomes, and frame/stack completeness before drawing conclusions. Preserve required Lightrun attribution from response metadata.
- Empty history means only that no matching capture was found. Historical hit counts are captured samples, not execution frequency.
- Skip irrelevant history calls when stored captures cannot provide the required signal, such as a fresh duration, execution-count, or numeric-metric measurement. When history is missing, stale, incomplete, disabled, unavailable, or insufficient, use live evidence collection and its source preflight for the remaining evidence need.

## Source discovery (preflight)

Before creating a live action for an evidence need that suitable history cannot satisfy:

1. Complete **source discovery** using read-only discovery tools exposed by Lightrun MCP.
2. Follow usage-flow and sequencing guidance in each tool's description (for example, discover pools before agents when descriptions indicate that order).
3. Use filters and pagination when descriptions support them and the target set is large; follow `hasMore` until the candidate is found or the list is exhausted.
4. Select a concrete runtime target for subsequent capture tools.

### Pass criteria

- At least one accessible agent pool with registered agents.
- A concrete target satisfies the chosen evidence tool's required targeting fields.
- For `customSourceName`, proceed when the user supplies a known name if discovery tools do not list custom sources.

### Fail criteria

- No source-discovery capability is exposed, discovery calls fail, or no viable target can be identified after reasonable filter/pagination.

### Source selection guidance

- Inspect agent metadata (for example `metadata.tags`) when descriptions expose it; prefer a **shared tag** when targeting a group of related agents.
- Follow selector semantics from the chosen tool schema; do not assume how names or tags are combined.
- Choose the most specific source scope that answers the current evidence need without assuming a universal selector preference order.
- When the user names an environment or service, use name/tag patterns with wildcards when the schema supports them before scanning full unpaginated lists.
- If several targets fit, select the strongest candidate(s) using service ownership, environment match, and expected execution path; ask the user only when confidence remains low.

## Missing-MCP recovery

Apply source-preflight recovery only when live evidence collection is needed; it must not block use of suitable historical evidence. Missing history tools alone do not imply missing live capabilities.

1. Classify failure: no source-discovery capability, discovery call error, empty pool list, or no agents/tags after filtering.
2. Remediate:
   - Install/enable Lightrun MCP.
   - Complete MCP OAuth authorization.
   - Verify access to the expected environment/agent pool.
   - See https://docs.lightrun.com/mcp/lightrun-tools/ when tool exposure is unclear.
3. Re-run the full source-discovery path.
4. Create live actions only after preflight pass criteria are met.

## Evidence tool selection

- Match tools to the required evidence signal using MCP descriptions and parameter fit.
- Before each action, state which decision or next step it can change; skip actions that cannot change either.
- Re-check exposed tools when resuming a later run and adapt if capabilities changed.
