## Phase 0 — Get Deployed Commit


Before analyzing the PR, determine the best baseline for review and fetch the correct diff.

1. Parse `owner`, `repo`, and PR head SHA from the PR URL (`https://github.com/{owner}/{repo}/pull/{number}`), or call GitHub MCP `get_pull_request` with `owner`, `repo`, `pull_number` and read `head.sha`.

2. Choose the baseline commit for comparison:

### A. When a deployed production revision can be determined

If the repository or environment exposes a reliable deployed revision, resolve `deployed_sha`, then diff **production -> PR head** (not base branch -> PR head).

Possible sources for `deployed_sha` include:
- deployment metadata exposed by the application or platform
- release metadata that maps to the currently deployed build
- environment or CI/CD metadata available to the user
- a production SHA explicitly provided by the user

When using a deployed baseline:

```bash
gh api "repos/{owner}/{repo}/compare/{deployed_sha}...{pr_head_sha}" \
  -H "Accept: application/vnd.github.diff"
```

Use this diff for all subsequent phases.

Record in output how `deployed_sha` was determined.

### B. When no deployed production revision can be determined

Use the PR's default diff (base branch vs PR head).

Record in output that the review used the PR base rather than a deployed production baseline.

