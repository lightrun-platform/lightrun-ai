## Output Format (automation response only)

Emit this structure as your **final automation message in chat** at the end of the run. This is the run transcript for the automation.

Do **not** mix this section with intermediate phase messages. Phase-specific requests or summaries should be emitted separately in chat only when the relevant phase calls for them.

**PR**
[Link to pull request]

**Feasibility Determination**
[Verifiable / Not verifiable + reason]

**Runtime Profile**
`runtime_profile_mcp: available | unavailable`

For each diff-derived verification area: `covered` (hit count, source) or `uncovered` (reason no relevant hits found).

**Sampling Plan**
[Focused snapshot entries from Phase 3, or `skipped (full profile coverage)` when Phase 2 covered all areas, or `partial (uncovered areas only)` when Phase 3 planned only gaps]

**Tool Calls Executed**
- Runtime Profile: [query_hits_state / query_variable_values / get_snapshot calls, params]
- Per location: [live snapshot tool(s) called, params - or "skipped (covered by runtime profile)"]

**Captured Samples**
[Raw or summarized values per verification location. Mark locations with zero hits as "No hits".]

**Intended change**
[short summary from Phase 4 Step 1]

**Risk surfaces**
[bullets from Phase 4 Step 2]

**Runtime patch simulation**

For each verification location:
| Hit # | Captured values | Pre-change behavior | Post-change behavior | Unexpected? |
|-------|----------------|---------------------|----------------------|-------------|
| 1     | ...            | ...                 | ...                  | Yes / No    |

[Repeat table per verification location if more than one.]

**Runtime verdict**
`LIKELY_SAFE_ON_PROD_SAMPLES` / `RISK_ON_PROD_SAMPLES` / `INSUFFICIENT_PROD_SAMPLES`

[One paragraph justifying the verdict, referencing specific hit values, sanitized values, code paths from the diff, and any unsampled verification locations.]

**Simulated scenarios**
[numbered list with full scenario structure from Phase 4 Step 3]

**Design verdict**
- Work-item fit:
- Main strengths:
- Main weaknesses:
- Highest-risk missing validation:
- Highest-risk missing test:

**Review findings**
[0 to 5 findings, highest severity first; include only High confidence findings]

**Suggested tests**
[concrete tests to add]

**Merge recommendation**
Choose one:
- Safe to merge
- Merge after minor fixes
- Needs design changes before merge

Factor in both the runtime verdict and scenario-driven findings. A `LIKELY_SAFE_ON_PROD_SAMPLES` verdict does not override High-confidence P0/P1 findings from scenario review.

**Next Actions**
[If LIKELY_SAFE or RISK: state what the verdict means for the PR reviewer.]
[If INSUFFICIENT: list unsampled locations and suggest higher-traffic lines or looser conditions for a follow-up run.]
