## Phase 3 — Runtime History

After Phase 2 mapping, check whether existing captured snapshots in **Runtime History** can supply evidence for mapped `uncovered` PR areas — before planning or creating live snapshots.

This phase is optional. If Runtime History is unavailable, leave Phase 2 mapped areas as `uncovered` and proceed to Phase 4.

### Step 1 — Runtime History availability

Discover Runtime History read capabilities using [MCP tool discovery](mcp-tool-discovery.md#runtime-history-reads). Follow its schema-driven tool selection, bounded includes, pagination, and value-outcome guidance. History reads require neither active agents nor live-source preflight. Reuse the PR URL as the correlation key across history and live calls where supported.

- **If not installed / unavailable:** record `runtime_history: unavailable`. Leave every Phase 2 mapped area as `uncovered`. Proceed to **Phase 4 — Sampling Plan** and **Phase 5 — Execute Snapshots** for mapped `uncovered` PR areas only. Do not attempt history queries.
- **If available:** record `runtime_history: available` and continue to Step 2.

### Step 2 — History search for mapped uncovered areas

Query Runtime History **only** for Phase 2 PR verification areas still classified `uncovered` with a deployed equivalent. Do not query `not runtime-verifiable` areas.

Use the exposed history tools **only at deployed-equivalent locations**, following their schemas and descriptions rather than assuming tool names or provider prefixes.

Classify a mapped area as `covered`, `base-covered`, `head-covered`, or `uncovered` according to the definitions below.

#### Query scope — from earlier phases

Use the narrowest supported scope from known context: PR repository, verified `deployed_sha` or a safely mapped historical revision, deployed-equivalent file/function, relevant source/environment and capture time, and needed value paths. Apply repository/commit filters only when known and appropriate; their metadata may be absent from hits. Never use PR diff line numbers or silently relax a required revision, source, or time constraint. Follow exposed selector semantics.

#### Search strategy

Search is driven by the PR diff; queries always run only at deployed-equivalent coordinates from Phase 2:

1. Query each mapped `uncovered` area using its narrow historical scope.
2. Request only the captured values and stack/frame data required by the changed logic, following the exposed schema.
3. Evaluate value outcomes and completeness using the shared reference.
4. Record the **actual repository and revision attached to every hit**, or `unknown` for absent fields; never substitute the PR repository or `deployed_sha`.

#### Historical hits

- A verified hit at `deployed_sha` may describe deployed behavior within its capture-time and source limits.
- A hit at another revision is historical evidence only — label it with that exact revision.
- Different-revision values may support PR-base and PR-head simulation only after inspecting the captured revision's code and verifying compatibility with the Phase 2 mapping. Unknown revision or unverified compatibility leaves the area `uncovered`.
- Never describe a non-`deployed_sha` hit as current production behavior.

### Relevance criteria

A hit is **relevant** when:

- It is at a deployed-equivalent location for the verification area (same method/function/branch logic, not PR line numbers)
- When commit metadata is present, it matches `deployed_sha`, or was captured at another revision with a valid area mapping (label that revision explicitly)
- Capture time and source/environment scope match the evidence need; available repository/commit metadata, file/function, and returned line are consistent with the verified code mapping
- Selected-value outcomes and stack/frame completeness supply the inputs needed to simulate the changed logic at PR base, PR head, or both

### No-hit handling — widen the search

If the initial query returns no relevant historical hits for a deployed-equivalent area, widen the search before leaving it `uncovered`:

- Do not guess or fabricate samples; use only values returned by Runtime History.
- Do not widen into PR-only or unmapped coordinates.
- Check for over-narrow value filters, a function-name mismatch, a revision filter excluding relevant captures, or an incorrect source-root-relative file path.
- First relax supported filters within the same deployed file and broaden the capture window when appropriate, recording every hit's actual revision.
- Then consider logically adjacent mapped callers, callees, or helpers for the same verification area.
- Make up to **3 query attempts per verification area** before concluding that no relevant captures exist.

Empty history means only that no matching capture was found, not that code did not execute or a value did not occur. Hit counts are captured samples, not execution frequency. History that supports only one side leaves the area `base-covered` or `head-covered`; history that supports neither leaves it `uncovered` for live fallback.

### Per-area outcome

For each Phase 2 area that was `uncovered`, record any status update from this phase:

| Field | Value |
|---|---|
| Status | `covered`, `base-covered`, `head-covered`, or `uncovered` (unchanged from Phase 2 when no hits) |
| Evidence provenance | hit IDs, capture times, actual revisions, source scope, completeness limitations |

| Status | Meaning |
|---|---|
| `covered` | Relevant, sufficiently complete captures support both simulations at a deployed-equivalent location — record them as **Captured Samples** for this area |
| `base-covered` | Evidence supports PR-base simulation but not PR head |
| `head-covered` | Evidence supports PR-head simulation but not PR base |
| `uncovered` | Deployed equivalent exists but evidence supports neither simulation — needs live sampling via Phase 4 and Phase 5 |

Do not guess or fabricate samples. Only use values returned by Runtime History. Do not reclassify Phase 2 `not runtime-verifiable` areas.

### Next phase

- **All mapped PR areas `covered`:** skip Phase 4 and Phase 5 entirely; proceed to **Phase 6 — Patch Simulation** with history hits as captured samples and `not runtime-verifiable` areas recorded as unsampled.
- **One or more PR areas `base-covered`, `head-covered`, or `uncovered`:** proceed to **Phase 4 — Sampling Plan** for **`base-covered`, `head-covered`, or `uncovered` deployed-equivalent PR areas only** (up to 3 total), then **Phase 5 — Execute Snapshots**. Areas already `covered` or `not runtime-verifiable` must not be re-instrumented or sampled.
