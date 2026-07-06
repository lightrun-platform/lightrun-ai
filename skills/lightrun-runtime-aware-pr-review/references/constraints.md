## Constraints

- Never reference lines or variables that only exist in the new version of the code when selecting snapshot locations or capture expressions
- Never suggest redeployment or feature flags
- Read-only instrumentation only (no side effects)
- Never produce per-change entries when the PR is not verifiable — one explanation only
- Simulation must use only values that were actually captured — never infer or fabricate input values
- Do not simulate behavior of code paths not touched by the PR diff
- Verdict must reference specific hit numbers and code lines from the diff
- A **verification location** is a diff-derived area with profile hits (Phase 2) or a Phase 3 plan entry when live snapshots were needed
- `LIKELY_SAFE_ON_PROD_SAMPLES` requires at least one hit at every verification location. Partial coverage blocks only that verdict — when any verification location has zero hits and no sampled hit is risky, emit `INSUFFICIENT_PROD_SAMPLES`; when any sampled hit shows risk, emit `RISK_ON_PROD_SAMPLES` even under partial coverage


- When live async snapshots were created in this run and any verification location still has zero hits after the bounded in-session poll, emit the Phase 4 Sampling Request and stop. Do not emit the final Output Format or Phase 5 verdict in that same run unless this is a resumed run and correlated actions were re-checked first.

