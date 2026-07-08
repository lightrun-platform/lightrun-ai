## Phase 3 — Sampling Plan

Plan live instrumentation only when Phase 2 did not fully cover the diff with existing production hits.

### When to run

| Phase 2 outcome | Action |
|---|---|
| `runtime_profile_mcp: unavailable` | Plan up to 3 highest-value instrumentation points across all verifiable diff areas |
| MCP available, one or more areas `uncovered` | Plan only for Phase 2 `uncovered` areas (up to 3 total) |
| MCP available, all areas `covered` | **Skip this phase entirely** — proceed to Phase 5 |

Do not re-plan locations already `covered` by Phase 2 runtime profile hits.

Identify the top (up to 3) highest-value instrumentation points in the **pre-change** production code for the applicable scope above.

### Line selection rules
The selected line **must**:
- Contain executable code
- Ensure all variables and expressions used in the modified or newly added lines in the PR are in scope

The selected line **must not**:
- Be a comment, blank line, or declaration-only line
- Contain only brackets of any kind
- Be between code sections (e.g., between methods or classes)
- Reference variables or methods introduced by the PR

### Snapshot timing
Lightrun captures data **immediately before** the selected line executes. Pick a line **after** the relevant values have already been assigned.

### Expression/Condition rules
- Specify 1–10 expressions per snapshot
- Use source-root relative file paths (e.g., `com/example/MyClass.java`) — not full filesystem paths
- Expressions can include variable names, field accesses, method calls, or compound expressions in scope at the line.
- Avoid state-changing expressions (e.g., calls that mutate objects or global state), and do not use lambdas or anonymous functions as snapshot expressions.
- Objects are captured to 3 nesting levels only — if a deeper field is needed, specify it directly (e.g., `order.payment.card.last4` instead of `order`)
- Syntax must match the target language

### Output the plan

**Snapshot [N]:**
- **Location:** `com/example/MyClass.java:LINE`
- **Expressions:** `foo`, `bar.baz()`, `items.size()`
- **Condition:** [Optional boolean expression to narrow capture scope, or `null`]
- **Why:** [One sentence: what production values here confirm the PR's assumption or reveal whether the change is safe.]
- **How to trigger within 600s:** [What traffic or action will hit this line in production.]
