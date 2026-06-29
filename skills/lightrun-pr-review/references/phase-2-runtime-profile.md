## Phase 2 - Runtime Profile Check

After feasibility passes, check whether existing production hits in the **runtime profile** MCP can supply evidence for the PR diff - before planning or creating live snapshots.

### Step 1 - MCP availability gate

Discover whether the **runtime profile** MCP server is available (for example `query_hits_state`, `query_variable_values`, `get_snapshot`).

- **If not installed / unavailable:** record `runtime_profile_mcp: unavailable` and proceed directly to **Phase 3 - Sampling Plan** (full plan) and **Phase 4 - Execute Snapshots**. Do not attempt profile queries.
- **If available:** record `runtime_profile_mcp: available` and continue to Step 2.

### Step 2 - Diff-driven search

Before any sampling plan exists, derive **verification areas** from the PR diff - behavioral regions that need production evidence (changed files, methods/functions, branches, and variables in scope at those changes).

Use the Runtime Profile MCP tools to search for relevant production hits. Discover available tools from the MCP server and adapt as needed.

#### How search tools support simulation

| Tool | Use for simulation |
|------|-------------------|
| `query_hits_state` | Find hits by `repo`, `commit`, `file`, `function_name`; narrow with `variable` and `value_match` when the diff suggests specific state |
| `query_variable_values` | Pull values for a named variable across matching hits when you know what state matters from the diff |
| `get_snapshot` | Start with `projection="metadata"`, then `projection="value"` with `path` to retrieve captured state for simulation |

#### Query parameters - from earlier phases

| Parameter | Source |
|---|---|
| `repo` | `{owner}/{repo}` from the PR |
| `commit` | Deployed commit from Phase 0 (when available) |
| `file` | Source-root relative path from a changed file in the diff |
| `function_name` | Method or function containing a behavioral change, when known |

#### Search strategy

Search is driven by the diff, not by a pre-planned instrumentation list:

1. For each verification area, call `query_hits_state` filtered by `repo`, `commit`, and `file` (and `function_name` when available).
2. When the diff references specific variables or conditions, use `variable` and `value_match` filters to narrow hits when the MCP supports it.
3. For hits returned, retrieve captured state with `get_snapshot` (`projection="metadata"` first, then `projection="value"` with `path` as needed) or `query_variable_values`.
4. Retrieve values per verification area.

### Relevance criteria

A hit is **relevant** when:

- It is in a diff-touched file at or near a behavioral change (same line, or an adjacent line in the same method/function), **or** during widened search it is in a logically adjacent file (caller, callee, shared helper, or other diff-touched file for the same verification area) with captured state sufficient to simulate the changed logic
- It matches the deployed commit, or was captured in the last 2 weeks
- Captured state is sufficient to simulate the changed logic at that area (retrievable via `get_snapshot` or `query_variable_values`)

### No-hit handling - widen the search

If the initial query returns no relevant historical hits for a verification area, **widen the search before marking it `uncovered`**:
- Do not guess or fabricate samples - only use values returned by the MCP
- Check: over-narrow `variable` / `value_match` filters, `function_name` mismatch, `commit` filter excluding recent captures, wrong source-root relative `file` path
- **First**: relax the query within the same `file` - drop `variable` / `value_match` filters, search adjacent lines or sibling methods, and broaden the time window (for example drop the `commit` filter to include hits from the last 2 weeks)
- **Then**: query logically adjacent files for the same verification area - callers, callees, shared helpers, or other diff-touched files that exercise the same behavior - using the same broadened parameters
- Re-run `query_hits_state` after each adjustment (up to **3 query attempts per verification area**) before concluding no relevant hits exist

Only mark an area `uncovered` after the widened search still returns no relevant hits.

### Per-area outcome

Evaluate each verification area independently. For every area derived from the diff, record one of:

| Status | Meaning |
|---|---|
| `covered` | One or more relevant hits found - record them as **Captured Samples** for this area |
| `uncovered` | No relevant hits found after the widened search - this area still needs live sampling via Phase 3 and Phase 4 |

Do not guess or fabricate samples. Only use values returned by the runtime profile MCP.

### Next phase

- **MCP unavailable:** proceed to **Phase 3 - Sampling Plan** (full plan, using a small focused set of locations) and **Phase 4 - Execute Snapshots**.
- **All areas `covered`:** skip Phase 3 and Phase 4 entirely; proceed to **Phase 5 - Patch Simulation** with profile hits as captured samples.
- **One or more areas `uncovered`:** proceed to **Phase 3 - Sampling Plan** for **uncovered areas only** (using a small focused set of locations), then **Phase 4 - Execute Snapshots** for those locations. Areas already `covered` must not be re-instrumented.
