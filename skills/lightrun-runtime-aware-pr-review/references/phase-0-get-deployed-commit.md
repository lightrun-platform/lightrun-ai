## Phase 0 — Get Deployed Commit


Before analyzing the PR, resolve the PR diff and optionally the deployed production revision.

### Step 1 — Resolve PR base and head

1. Parse `owner`, `repo`, and PR number from the PR URL (`https://github.com/{owner}/{repo}/pull/{number}`), or call GitHub MCP `get_pull_request` with `owner`, `repo`, `pull_number`.
2. Read `base.sha` and `head.sha` from the PR.
3. Call the compare API:

   ```bash
   gh api "repos/{owner}/{repo}/compare/{base_sha}...{head_sha}"
   ```

4. Set:
   - `pr_base_sha` = `merge_base_commit.sha` from the compare response
   - `pr_head_sha` = `head.sha` from the PR response

5. Fetch the PR diff:

   ```bash
   gh api "repos/{owner}/{repo}/compare/{pr_base_sha}...{pr_head_sha}" \
     -H "Accept: application/vnd.github.diff"
   ```

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

