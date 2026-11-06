## Phase 0 — Get Deployed Commit


Before analyzing the PR, resolve the PR diff and optionally the deployed production revision.

### Step 1 — Resolve PR base and head

Follow [Code management](code-management.md).

1. Detect `code_host` from the PR URL and verify code-management access for that host (GitHub or Bitbucket).
2. Fetch PR metadata. Set `pr_head_sha` to the source/head commit SHA.
3. Resolve `pr_base_sha` as the merge base of destination and head.
4. Fetch the unified diff for `pr_base_sha` → `pr_head_sha`.

Use this diff (`pr_base_sha` → `pr_head_sha`) for **all** subsequent phases: verification areas, attribution, findings, verdict, and merge recommendation.

**Never** use `deployed_sha` → `pr_head_sha` as the PR review diff.

### Step 2 — Resolve deployed revision (instrumentation only)

Resolve `deployed_sha` only when a trustworthy deployed revision is available. `deployed_sha` is used **only** to locate safe runtime instrumentation and interpret true deployed production behavior — not to define the PR diff.

Possible sources for `deployed_sha` include:
- deployment metadata exposed by the application or platform
- release metadata that maps to the currently deployed build
- environment or CI/CD metadata available to the user
- a production SHA explicitly provided by the user

When `deployed_sha` cannot be resolved, do not create PR-derived live snapshots and do not claim deployed production behavior.

Record in output how `deployed_sha` was determined when resolved.

### Step 3 — Record audit fields

Record in final output:
- `pr_base_sha`
- `pr_head_sha`
- `deployed_sha` (when resolved, otherwise note that it is unavailable)

