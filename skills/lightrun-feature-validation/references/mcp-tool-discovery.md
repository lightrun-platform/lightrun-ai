# Lightrun MCP Tool Discovery

Shared rules for all Lightrun skills. MCP tool schemas and descriptions are the source of truth for tool names, parameters, sequencing, and pagination.

## Discovery rules (all steps)

- Do not assume fixed tool names, client prefixes, tool counts, or call order.
- At run start - and again when resuming a later run - enumerate exposed Lightrun MCP tools and read schemas/descriptions before calling.
- Select tools by **capability fit** to the active workflow step.
- Record the exact MCP-exposed identifier used for each call.
- Prefer read-only discovery tools before capture or mutating runtime tools.
- When tool descriptions define a usage flow or sequencing hints, follow them.

## Capability taxonomy

| Capability | Purpose | How to identify |
|------------|---------|-----------------|
| Source discovery | List agent pools, agents, tags, and related targeting metadata | Read-only tools whose descriptions indicate discovering runtime targets |
| Same-run evidence | Capture values, durations, counts, call stacks, numeric metrics | Tools accepting `file`, `line`, and runtime targeting parameters |
| Async lifecycle | Create action, poll status, retrieve results, cancel | Tools that accept or return `actionId` |
| Historical/query | Probe past hits or stored snapshots | Tools whose descriptions indicate historical or query semantics |

## Source discovery (preflight)

Before any runtime evidence capture:

1. Complete **source discovery** using read-only discovery tools exposed by Lightrun MCP.
2. Follow usage-flow and sequencing guidance in each tool's description (for example, discover pools before agents when descriptions indicate that order).
3. Use filters and pagination when descriptions support them and the target set is large; follow `hasMore` until the candidate is found or the list is exhausted.
4. Select a concrete runtime target for subsequent capture tools.

### Pass criteria

- At least one accessible agent pool with registered agents.
- A concrete target is selected for runtime tools:
  - `agentPoolName` plus exactly one selector mode among the modes accepted by the chosen evidence tool schema (`agentNames`, `tagNames`, and/or `customSourceName` as applicable).
- For `customSourceName`, proceed when the user supplies a known name if discovery tools do not list custom sources.

### Fail criteria

- No source-discovery capability is exposed, discovery calls fail, or no viable target can be identified after reasonable filter/pagination.

### Source selection guidance

- Inspect agent metadata (for example `metadata.tags`) when descriptions expose it; prefer a **shared tag** when targeting a group of related agents.
- `tagNames` typically use logical **AND** (all tags must match); `agentNames` typically use logical **OR**.
- Prefer the most specific targeting that answers the question: `agentNames` > `customSourceName` > `tagNames`.
- When the user names an environment or service, use name/tag patterns with wildcards when the schema supports them before scanning full unpaginated lists.
- If several targets fit, select the strongest candidate(s) using service ownership, environment match, and expected execution path; ask the user only when confidence remains low.

## Missing-MCP recovery

1. Classify failure: no source-discovery capability, discovery call error, empty pool list, or no agents/tags after filtering.
2. Remediate:
   - Install/enable Lightrun MCP.
   - Complete MCP OAuth authorization.
   - Verify access to the expected environment/agent pool.
   - See https://docs.lightrun.com/mcp/lightrun-tools/ when tool exposure is unclear.
3. Re-run the full source-discovery path.
4. Continue only after preflight pass criteria are met.

## Evidence tool selection

- Match tools to feature-validation signals using MCP descriptions and parameter fit.
- Before each action, state what decision the action can change; skip actions that cannot change validation/design next steps.
- Re-check exposed tools when resuming a later run and adapt if capabilities changed.
