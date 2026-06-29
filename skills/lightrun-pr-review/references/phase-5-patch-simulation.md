## Phase 5 - Patch Simulation

After capturing samples (from Phase 2 runtime profile hits and/or Phase 4 live snapshots), simulate how each captured hit would behave under the new PR code, then run a scenario-driven review of the diff.

Simulate every **verification location** - the union of:
- Phase 2 profile hit locations (when the sampling plan was skipped or only partially needed)
- Phase 3 planned locations (when live snapshots were needed)

A verification location with zero hits across Phase 2 and Phase 4 cannot be simulated - treat it as unsampled, not as safe.

### Review inputs

Use:
- the PR title, description, changed files, and full diff (deployed-commit-to-PR)
- the linked work item, issue, spec, or ticket when available
- captured production expression values from Phase 2
- any referenced tests, configs, migrations, API contracts, and permissions visible in the diff

Your goal is not to restate the PR. Simulate realistic scenarios that could break because of this change, evaluate whether the diff handles them, and verify captured production samples against the changed logic.

## Part A - Runtime patch simulation

### Isolate changed logic

For each verification location, extract the specific changed lines from the deployed-commit-to-PR diff that affect behavior at or after that location:
- New or changed conditions
- New or changed assignments
- New or changed branches
- Changed return values or outputs

Ignore whitespace-only, formatting-only, and rename-only changes.

### Per-hit simulation

For each captured hit:
1. Substitute only the captured expression values into the changed PR logic.
2. State what the pre-change production code does with these values.
3. State what the post-change PR code would do with the same values.
4. Flag any unexpected value:
   - A value that should have changed but did not
   - A value that changed in a way that is inconsistent with the PR's intent

### Uncertainty rule

If the changed logic cannot be fully traced with the captured values, state the uncertainty explicitly and explain what additional expression capture would resolve it.

Do not guess, infer, or fabricate missing input values.

### Runtime verdict

After simulating all hits, emit exactly one of these verdicts:

- `LIKELY_SAFE_ON_PROD_SAMPLES`: **Every** verification location has at least one captured hit, and the new code produces correct or expected behavior on all captured hits with no divergence from PR intent.
- `RISK_ON_PROD_SAMPLES`: At least one captured hit produces an unexpected value under the new code. State which hit and why. (Partial location coverage does not block this verdict when sampled hits show risk.)
- `INSUFFICIENT_PROD_SAMPLES`: Any verification location has zero hits but no unexpected values were found. List which locations were unsampled and which locations are safe.

## Part B - Scenario-driven design review

### Step 1 - Understand the intended change

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

### Step 2 - Build a change-impact map

From the diff, identify:
- entry points changed
- permissions / auth / validation changes
- config or environment changes
- persistence / schema / migration changes
- async / concurrency / ordering risks
- external integrations and contracts
- user-visible flows affected
- existing tests added, removed, or missing

Then list the top 3-7 risk surfaces.

### Step 3 - Generate simulation scenarios

Generate realistic scenarios using both:
- the intended work-item flow (or PR description when no linked work item is available)
- failure / edge / regression cases implied by the diff
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

When a scenario uses production evidence, use only sanitized values from captured hits - never raw secrets, tokens, identifiers, or other sensitive values.

Skip any low-confidence or medium-confidence scenarios. Do not include `Risk level: Low`.

### Step 4 - Evaluate the design

Based on the scenarios, judge whether the solution is well-designed.

Specifically answer:
- Does the implementation satisfy the linked work item (or PR intent when no linked work item is available)?
- What assumptions does the implementation make?
- Which assumptions are unsafe or unverified?
- Are there hidden regressions outside the intended change scope?
- Are critical guards being removed or bypassed?
- Are there missing validations, missing negative tests, or missing rollout protections?
- Is the design resilient to misuse, retries, stale state, and partial failure?

If the linked work item conflicts with the implementation, call that out explicitly.

### Step 5 - Produce actionable review findings

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
- silent behavior changes
- production operability regressions
- broken assumptions between linked work item intent and actual implementation

Do not include findings unless they are supported by concrete evidence in the diff, tests, linked work-item context, or captured hits.
Do not output low-confidence or medium-confidence findings.
If evidence is insufficient, omit the finding rather than speculate.

Be skeptical of removed checks, removed permissions, removed validations, removed metrics/logging, and broadening conditions.
Treat tiny diffs as potentially high risk when they change security, config, or control flow.

## Final Review Summary

After completing Part A and Part B, emit a final review summary in chat containing:
- The runtime verdict (`LIKELY_SAFE_ON_PROD_SAMPLES`, `RISK_ON_PROD_SAMPLES`, or `INSUFFICIENT_PROD_SAMPLES`)
- A short verdict justification (see style rules below)
- The merge recommendation from scenario-driven review
- Up to 3 High-confidence review findings (highest severity first)
- Next actions for the PR reviewer

Do not skip this summary even when the verdict is `INSUFFICIENT_PROD_SAMPLES` or when Review findings is empty.

### Writing style

- Use short sentences. One idea per sentence.
- Lead with the conclusion. Add detail only when it supports the conclusion.
- Use plain, accurate language. Be specific, not exhaustive.
- No preamble, no generic review boilerplate, no unnecessary praise.
- Never include raw secrets, tokens, identifiers, or other sensitive values - only sanitized evidence (redacted, masked, or generalized).

### Verdict justification (2-3 sentences max)

- Sentence 1: State the verdict and why (hit count + outcome, or which locations lack hits).
- Sentence 2 (optional): One code path and sanitized values that support the verdict.
- For `INSUFFICIENT_PROD_SAMPLES`: name unsampled locations/conditions; do not invent hit values.

### Findings format

One bullet per finding. Use this structure - do not combine into long sentences:
- **Problem & Fix Title** - Problem in one short sentence. Fix in one short sentence.

Omit scenario names unless needed for clarity. Do not restate the diff or repeat the verdict paragraph.

Example format:

> **Runtime verification verdict: `LIKELY_SAFE_ON_PROD_SAMPLES`**
>
> All 3 captured hits behave the same under the old and new code. Branch at `PricingService.java:142` is unchanged for sampled values (`order.total=<redacted>`, `user.tier=<masked>`).
>
> **Merge recommendation:** Merge after minor fixes
>
> **Findings:**
> - **P2 - Missing null guard on tier lookup** - `user.tier` can be null at `PricingService.java:138`. Add a null check and a negative test.
>
> **Next actions:** Fix the P2 finding before merge. No runtime risk on captured samples.
