## Phase 6 — Patch Simulation

After capturing samples, use the Phase 2 area statuses (updated by Phase 3 when available) and simulate valid hits against the PR diff. Only PR `not runtime-verifiable` and zero-hit areas are carried into the runtime assessment as unsampled.

Use:
- the PR title, description, changed files, and full diff (`pr_base_sha` → `pr_head_sha`)
- the linked work item, issue, spec, or ticket when available
- captured production expression values from Phase 3 and Phase 5
- any referenced tests, configs, migrations, API contracts, and permissions visible in the PR diff

Your goal is NOT to restate the PR. Simulate realistic scenarios that could break because of this change, evaluate whether the diff handles them, and verify captured production samples against the changed logic with correct PR attribution.

---

## Part A — Runtime patch simulation

### Isolate changed logic
For each valid sampled PR area, extract the specific changed lines from the PR diff (`pr_base_sha` → `pr_head_sha`) that affect behavior:
- New or changed conditions
- New or changed assignments
- New or changed branches
- Changed return values or outputs

Ignore whitespace-only, formatting-only, and rename-only changes.

### Per-hit simulation and attribution
For each valid captured hit in a PR area:

1. Substitute only the captured expression values into the logic at **PR base** (`pr_base_sha`) and **PR head** (`pr_head_sha`).
2. State **PR-base behavior** — what the code at `pr_base_sha` does with these values.
3. State **PR-head behavior** — what the code at `pr_head_sha` would do with the same values.
4. Apply attribution rules:
   - If PR base and PR head behave identically, do **not** attribute the outcome to this PR.
   - Attribute runtime risk only when PR head is unexpected relative to PR intent **and** differs from PR base.
   - A correct intended behavior change (PR base ≠ PR head, matches intent) supports a `0 — No observed regression` assessment when the evidence threshold is met.
   - Do not attribute validation, DTO, binding, or API-contract failures when the relevant contract is unchanged by the PR diff.
5. Apply the Phase 3 hit-revision label when the hit came from the runtime profile; otherwise use the capture revision from Phase 5. Only a hit at `deployed_sha` may describe deployed behavior.

### Uncertainty rule
If the changed logic cannot be fully traced with the captured values, state the uncertainty explicitly and explain what additional expression capture would resolve it.

Do not guess, infer, or fabricate missing input values.

### Runtime assessment
After simulating all hits, emit one structured runtime assessment. Do not emit a standalone runtime verdict.

The assessment has five required fields:

- **Risk score:** `0`–`5`, or `not scored`
- **Risk label:** `No observed regression`, `Negligible`, `Low`, `Moderate`, `High`, `Critical`, or `not scored`
- **Confidence:** `0`–`100%`
- **Evidence status:** `sufficient`, `limited`, `inconclusive`, or `not runtime-verifiable`
- **Basis:** concise explanation of observed behavior, impact, coverage gaps, and the relevant hit numbers, revisions, and PR-diff lines

#### Confidence and evidence status

Assess confidence from coverage of runtime-verifiable PR areas; number and diversity of captured hits; completeness of values needed to trace changed logic; certainty of deployed-to-PR mapping and hit revision; and remaining unsampled or non-runtime-verifiable areas. State material gaps in the basis.

- `sufficient`: confidence is at least 50% and coverage, values, and mapping support the score without a material unresolved gap.
- `limited`: confidence is at least 50%, but coverage, hit diversity, captured values, or `not runtime-verifiable` areas leave bounded gaps. A score may still be issued.
- `inconclusive`: confidence is below 50%. Set `Risk score: not scored` and `Risk label: not scored`; list unsampled locations or missing values.
- `not runtime-verifiable`: no changed area can produce meaningful runtime evidence. This includes areas that cannot be mapped to deployed code or safely instrumented, and the Phase 1 not-verifiable categories such as pure refactors, startup-only logic, dead or test-only code, and event-dependent paths that cannot be passively observed. Set `Risk score: not scored`, `Risk label: not scored`, and `Confidence: 0%`; list affected areas.

For mixed-status PRs, do not let a `not runtime-verifiable` area erase conclusive captured runtime evidence from another area. If a captured hit demonstrates PR-attributable unexpected behavior, retain the corresponding score, use `Evidence status: limited`, reduce confidence to reflect the `not runtime-verifiable` areas, and list those areas in the basis. Do not lower confidence below 50% solely because other areas are `not runtime-verifiable` when the scored risk itself is conclusively demonstrated.

Sample quantity affects confidence, not the risk score. Do not inflate risk solely because samples are few, areas are unsampled, or values are incomplete.

#### Risk score rubric

| Score | Label | Criteria |
|---|---|---|
| 0 | No observed regression | Sampled PR-head behavior matches PR intent. |
| 1 | Negligible | Captured evidence confirms a PR-attributable regression with negligible, localized impact. |
| 2 | Low | Captured evidence confirms a bounded PR-attributable regression with straightforward mitigation. |
| 3 | Moderate | PR-attributable incorrect behavior for a supported subset of users or requests. |
| 4 | High | Broad functional, compatibility, or reliability impact. |
| 5 | Critical | Security vulnerability, data loss/corruption, or severe outage potential. |

Scores `1`–`5` require concrete PR-attributable unexpected behavior observed in captured runtime evidence: PR head must differ from PR base and be unexpected relative to PR intent. Score `0` means no observed regression on the sampled behavior. Scores `1`–`2` may represent confirmed regressions when their demonstrated impact is negligible or low. Scores `3`–`5` represent moderate through critical impact. A static finding, its priority, and test recommendations do not select or change the runtime score.

When the PR diff alone exposes a potential defect but the relevant behavior has no captured runtime evidence, report it as a high-confidence P0–P3 design finding. Do not assign or change a runtime risk score from that static conclusion alone. If no other captured runtime evidence independently supports a score, use an unscored assessment and name the missing or unmappable runtime evidence. Otherwise, retain the independently supported score, use `Evidence status: limited`, and list the unverified finding in the basis.

---

## Part B — Scenario-driven design review

Scenario-driven review uses the PR diff (`pr_base_sha` → `pr_head_sha`) only.

### Step 1 — Understand the intended change
Briefly infer:
- the business goal
- the technical approach
- the explicit acceptance criteria from the linked work item or PR context (if available)
- the implicit non-functional expectations affected by this PR:
  - security / authorization
  - backward compatibility
  - data integrity
  - observability
  - performance
  - reliability / retries / idempotency
  - testability
  - operability / rollout safety

If no linked work item is available, say so and continue using PR context only.

### Step 2 — Build a change-impact map
From the PR diff, identify:
- entry points changed
- permissions / auth / validation changes
- config or environment changes
- persistence / schema / migration changes
- async / concurrency / ordering risks
- external integrations and contracts
- user-visible flows affected
- existing tests added, removed, or missing

Then list the top 3–7 risk surfaces.

### Step 3 — Generate simulation scenarios
Generate realistic scenarios using BOTH:
- the intended work-item flow (or PR description when no linked work item is available)
- failure / edge / regression cases implied by the PR diff
- captured production hits from Part A, where they ground a scenario in observed runtime state

For each scenario, use this structure:
- Scenario name
- Why this scenario matters
- Initial state
- Trigger / intervention
- Expected correct behavior
- What in the diff makes this safe or unsafe
- Risk level: Medium / High
- Confidence: High

The scenarios must cover, where relevant:
1. happy path from the linked work item or PR intent
2. permission / auth boundary violations
3. invalid or missing input
4. stale state / race conditions / repeated requests
5. partial failure of dependencies
6. backward compatibility with old clients / configs / data
7. observability regression (metrics, logs, alerts, tracking)
8. rollout / configuration mismatch
9. missing tests for the most critical regression path

Do not generate generic scenarios. Each scenario must directly reference the actual changed code and linked work item or PR context.

Only include scenarios where the risk is meaningful and the concern is supported by the diff and linked work-item context.
Do not include speculative scenarios that depend on assumptions not evidenced in the PR, linked work item, or captured hits.

When a scenario uses production evidence, use only sanitized values from captured hits — never raw secrets, tokens, identifiers, or other sensitive values.

Skip any low-confidence or medium-confidence scenarios. Do not include "Risk level: Low".

### Step 4 — Evaluate the design
Based on the scenarios, judge whether the solution is well-designed.

Specifically answer:
- Does the implementation satisfy the linked work item (or PR intent when no linked work item is available)?
- What assumptions does the implementation make?
- Which assumptions are unsafe or unverified?
- Are there hidden regressions outside the linked work item scope?
- Are critical guards being removed or bypassed?
- Are there missing validations, missing negative tests, or missing rollout protections?
- Is the design resilient to misuse, retries, stale state, and partial failure?

If the linked work item conflicts with the implementation, call that out explicitly.

### Step 5 — Produce actionable review findings
Only produce findings for meaningful issues. Prefer fewer, high-signal findings.

For each finding include:
- Severity: P0 / P1 / P2 / P3
- Title
- Explanation grounded in the diff and linked work item or PR intent
- The concrete scenario that exposes the problem
- Recommended fix
- Recommended test to add
- Confidence: High

Prioritize:
- security regressions
- data corruption / integrity issues
- silent behavior changes attributable to the PR diff
- production operability regressions
- broken assumptions between linked work item intent and actual implementation

Do not include findings unless they are supported by concrete evidence in the diff, tests, linked work-item context, or captured hits.
Do not output low-confidence or medium-confidence findings.
If evidence is insufficient, omit the finding rather than speculate.

Be skeptical of removed checks, removed permissions, removed validations, removed metrics/logging, and broadening conditions.
Treat tiny diffs as potentially high risk when they change security, config, or control flow.

---

## Final Review Summary

After completing Part A and Part B, emit a final review summary in chat containing:
- The complete runtime assessment
- A short assessment basis (see style rules below)
- The merge recommendation from scenario-driven review
- Up to 3 High-confidence review findings (highest severity first)
- Next actions for the PR reviewer

Do not skip this summary when evidence is inconclusive or when Review findings is empty.

### Writing style
- Use short sentences. One idea per sentence.
- Lead with the conclusion. Add detail only when it supports the conclusion.
- Use plain, accurate language. Be specific, not exhaustive.
- No preamble, no generic review boilerplate, no unnecessary praise.
- Never include raw secrets, tokens, identifiers, or other sensitive values — only sanitized evidence (redacted, masked, or generalized).

### Assessment basis (2–3 sentences max)
- Sentence 1: State the risk score, label, confidence, and evidence status, and why (hit count + PR-attributable outcome, or which locations lack hits / are not runtime-verifiable).
- Sentence 2 (optional): One code path and sanitized values that support the assessment.
- For `inconclusive` or `not runtime-verifiable`: name unsampled locations and areas; do not invent hit values.

### Findings format
One bullet per finding. Use this structure — do not combine into long sentences:
- **Problem & Fix Title** — Problem in one short sentence. Fix in one short sentence.

Omit scenario names unless needed for clarity. Do not restate the diff or repeat the assessment paragraph.

Example format:
> **Runtime assessment: Risk score: 0 — No observed regression · 86% confidence**
>
> **Evidence status:** sufficient
>
> All 3 captured hits show expected PR-head behavior relative to PR base and PR intent. Branch at `PricingService.java:142` changes only as intended for sampled values (`order.total=<redacted>`, `user.tier=<masked>`).
>
> **Merge recommendation:** Merge after minor fixes
>
> **Findings:**
> - **P2 — Missing null guard on tier lookup** — `user.tier` can be null at `PricingService.java:138`. Add a null check and a negative test.
>
> **Next actions:** Fix the P2 finding before merge. A mixed-case-header test is recommended to preserve the sampled behavior; no PR-attributable runtime regression was observed.
