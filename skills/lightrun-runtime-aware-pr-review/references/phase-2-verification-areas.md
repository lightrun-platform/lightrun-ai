## Phase 2 — Verification Areas

After feasibility passes, derive verification areas from the PR diff and resolve each area's deployed equivalent. This phase always runs and does not depend on the runtime profile MCP. Phase 4 and Phase 5 use these mapped areas even when Phase 3 is unavailable.

### Step 1 — Derive verification areas

Derive **verification areas** from the PR diff (`pr_base_sha` → `pr_head_sha`) — behavioral regions that need production evidence (changed files, methods/functions, branches, and variables in scope at those changes).

### Step 2 — Deployed mapping

For each PR verification area, resolve its deployed equivalent:

1. **If `deployed_sha` is known**, map the area to an equivalent location in that revision by logical method/function/branch. Never copy PR line numbers to instrumentation coordinates.
2. **If an equivalent deployed location exists**, classify the area as `uncovered` (pending profile search or live sampling).
3. **If no equivalent deployed location exists:**
   - classify the area as `not runtime-verifiable`;
   - do not query it using PR coordinates;
   - do not create a snapshot for it;
   - carry it into the final verdict as unsampled.
4. **If `deployed_sha` cannot be resolved:**
   - do not create PR-derived live snapshots;
   - do not claim deployed production behavior;
   - classify the area as `not runtime-verifiable`.

Never send `not runtime-verifiable` areas to Phase 3, Phase 4, or Phase 5.

### Per-area outcome

Evaluate each PR verification area independently and record:

| Field | Value |
|---|---|
| Deployed mapping | deployed file:line or method at `deployed_sha`, or `no deployed equivalent` |
| Status | `uncovered` or `not runtime-verifiable` |

| Status | Meaning |
|---|---|
| `uncovered` | Deployed equivalent exists — eligible for Phase 3 profile search and/or Phase 4–5 live sampling |
| `not runtime-verifiable` | No deployed equivalent, or `deployed_sha` unavailable — unsampled; never query or instrument using PR coordinates |

### Next phase

Proceed to **Phase 3 — Runtime Profile Check** with the mapped areas from this phase. If Phase 3 is unavailable, keep every mapped area as `uncovered` and proceed to **Phase 4 — Sampling Plan** for those areas.
