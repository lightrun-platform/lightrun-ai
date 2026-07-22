## Constraints

Phase 2 owns verification-area derivation and deployed mapping. Phase 3 may reclassify mapped `uncovered` areas to `covered` when profile hits are found. Later phases must use that status rather than reclassifying an area.

A **verification location** is a runtime-verifiable PR area represented by Phase 3 profile hits or a Phase 4 sampling-plan entry.

- Use the PR diff (`pr_base_sha` → `pr_head_sha`) for review. Never use `deployed_sha` → `pr_head_sha` as the PR review diff.
- Query and instrument only Phase 2 deployed-equivalent locations. Never use PR-only coordinates or PR line numbers for runtime tooling.
- Record every hit's revision. Never describe a non-`deployed_sha` hit as current deployed behavior.
- Never suggest redeployment or feature flags
- Read-only instrumentation only (no side effects)
- Never produce per-change entries when the PR is not verifiable — one explanation only
- Simulation must use only values that were actually captured — never infer or fabricate input values
- Do not simulate behavior of code paths not touched by the PR diff
- Attribute runtime risk only when PR head is unexpected relative to PR intent and differs from PR base; identical PR-base and PR-head behavior is not PR-attributable
- Do not attribute validation, DTO, binding, or API-contract failures when the relevant contract is unchanged by the PR diff
- Runtime assessment basis must reference specific hit numbers, hit revisions, and code lines from the PR diff when hits exist.
- Apply the Phase 2/3 status to the runtime assessment: use `Risk score: not scored`, `Confidence: 0%`, and `Evidence status: not runtime-verifiable` only when no changed area can produce meaningful runtime evidence, including the Phase 1 not-verifiable categories. If another area has captured runtime evidence demonstrating PR-attributable risk, retain its score, reduce confidence, use `Evidence status: limited`, and list every `not runtime-verifiable` area in the basis.
- Keep runtime risk score, finding priority, and merge recommendation separate. Use static findings to target runtime validation and explain whether runtime evidence confirms, narrows, or fails to assess them. Do not translate a P0–P3 priority directly into a runtime risk score.

- When live async snapshots were created in this run and any verification location still has zero hits after the bounded in-session poll, emit the Phase 5 Sampling Request and stop. Do not emit the final Output Format or Phase 6 verdict in that same run unless this is a resumed run and correlated actions were re-checked first.

