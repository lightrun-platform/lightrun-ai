## Output Format (automation response only)

Emit this structure as your **final automation message in chat** at the end of the run. This is the run transcript for the automation.

Do **not** mix this section with intermediate phase messages. Phase-specific requests or summaries should be emitted separately in chat only when the relevant phase calls for them.

**PR**
[Link to pull request]

**Revision audit**
- `pr_base_sha`: [merge base SHA]
- `pr_head_sha`: [PR head SHA]
- `deployed_sha`: [deployed SHA when resolved, or `unavailable`]

**Feasibility Determination**
[Verifiable / Not verifiable + reason]

**Verification Areas**
For each PR diff-derived verification area from Phase 2:
- **Area:** [description from PR diff]
- **Deployed mapping:** [deployed file:line or method at `deployed_sha`, or `no deployed equivalent`]
- **Status:** `covered` | `uncovered` | `not runtime-verifiable`
- **Hit revision:** [revision for each hit, or `n/a`]
- **Hit count / reason:** [count and source when covered; reason when uncovered or not runtime-verifiable]

**Runtime Profile**
`runtime_profile_mcp: unavailable` (coming soon — not available in this skill)

Skip per-area profile coverage; treat all Phase 2 mapped `uncovered` areas as needing live sampling.

**Sampling Plan**
[Up to 3 snapshot entries from Phase 4 for Phase 2 mapped uncovered areas]

**Tool Calls Executed**
- Runtime Profile: skipped (coming soon)
- Per location: [live snapshot tool(s) called, params]

**Captured Samples**
[Raw or summarized values per runtime-verifiable verification location, with hit revision. Mark locations with zero hits as "No hits". List `not runtime-verifiable` areas as unsampled.]

**Intended change**
[short summary from Phase 6 Step 1]

**Risk surfaces**
[bullets from Phase 6 Step 2]

**Runtime patch simulation**

For each runtime-verifiable PR verification location with hits:
| Hit # | Hit revision | Captured values | PR-base behavior | PR-head behavior | PR-attributable unexpected? |
|-------|--------------|-----------------|------------------|------------------|----------------------------|
| 1     | ...          | ...             | ...              | ...              | Yes / No                   |

[Repeat table per verification location if more than one.]

**Runtime assessment**
- **Risk score:** `0` / `1` / `2` / `3` / `4` / `5` / `not scored`
- **Risk label:** `No observed regression` / `Negligible` / `Low` / `Moderate` / `High` / `Critical` / `not scored`
- **Confidence:** `[0–100]%`
- **Evidence status:** `sufficient` / `limited` / `inconclusive` / `not runtime-verifiable`

**Basis**
[One concise paragraph justifying the assessment. Reference specific hit values, hit revisions, sanitized values, code paths from the PR diff, unsampled runtime-verifiable locations, and `not runtime-verifiable` areas as applicable.]

When confidence is below 50%, use `Risk score: not scored`, `Risk label: not scored`, and `Evidence status: inconclusive`. When no changed area can produce meaningful runtime evidence, including the Phase 1 not-verifiable categories, use `Risk score: not scored`, `Risk label: not scored`, `Confidence: 0%`, and `Evidence status: not runtime-verifiable`. For a mixed-status PR with captured runtime evidence demonstrating PR-attributable risk, retain the corresponding score, use `Evidence status: limited`, reduce confidence without lowering it below 50% solely because other areas cannot produce meaningful runtime evidence, and list those areas in the basis.

**Simulated scenarios**
[numbered list with full scenario structure from Phase 6 Step 3]

**Design verdict**
- Work-item fit:
- Main strengths:
- Main weaknesses:
- Highest-risk missing validation:
- Highest-risk missing test:

**Review findings**
[0 to 5 findings, highest severity first; include only High confidence findings grounded in the PR diff]

**Suggested tests**
[concrete tests to add]

**Merge recommendation**
Choose one:
- Safe to merge
- Merge after minor fixes
- Needs design changes before merge

Factor in both the runtime assessment and scenario-driven findings. A low runtime risk score does not override High-confidence P0/P1 findings from scenario review, and a finding priority does not automatically change the runtime risk score.

**Next Actions**
[State what the runtime assessment means for the PR reviewer, independently from findings and merge recommendation.]
[If evidence is `inconclusive` or `not runtime-verifiable`: list unsampled runtime-verifiable locations and affected areas; suggest higher-traffic deployed lines or looser conditions for a follow-up run when safe and applicable.]
