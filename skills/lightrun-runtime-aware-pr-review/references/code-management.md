## Code management

This workflow needs a code host that can return pull-request metadata, a merge base, a unified diff, and (when a phase posts a review comment) a PR comment. **GitHub** and **Bitbucket** (Cloud and Data Center/Server) are supported.

Treat “PR” as the host’s pull request (Bitbucket uses the same term). Discover MCP tool schemas at run time; do not hardcode tool names such as `get_pull_request`.

### Detect the host

Parse the PR URL before calling any API:

| Host | URL pattern | Identifiers |
|------|-------------|-------------|
| GitHub Cloud | `https://github.com/{owner}/{repo}/pull/{number}` | `owner`, `repo`, `pull_number` |
| GitHub Enterprise | `https://{host}/{owner}/{repo}/pull/{number}` | `host`, `owner`, `repo`, `pull_number` |
| Bitbucket Cloud | `https://bitbucket.org/{workspace}/{repo_slug}/pull-requests/{id}` | `workspace` → `owner`, `repo_slug` → `repo`, `id` → `pull_number` |
| Bitbucket Data Center / Server | `https://{host}/projects/{projectKey}/repos/{repo_slug}/pull-requests/{id}` or `https://{host}/{context_path}/projects/{projectKey}/repos/{repo_slug}/pull-requests/{id}` | optional `context_path` → `bitbucket_base_path`, `projectKey` → `owner`, `repo_slug` → `repo`, `id` → `pull_number` |

If the URL does not match, ask which host to use.

Record:

- `code_host`: `github` | `bitbucket_cloud` | `bitbucket_dc`
- `owner`, `repo`, `pull_number` (normalized as above)
- `pr_url` (canonical PR URL)
- `bitbucket_base_path` for Bitbucket Data Center/Server: the optional path between the origin and `/projects`, including its leading slash (for example, `/bitbucket`); normalize it with one leading slash and no trailing slash, and use an empty string when absent

For Bitbucket Data Center/Server, preserve `bitbucket_base_path` in every REST URL below. Do not fold it into `host` or discard it when canonicalizing `pr_url`.

### Verify access

Accept any of:

- A code-management MCP for that host (GitHub MCP, Bitbucket/Atlassian MCP, GitLens/GitKraken, or equivalent)
- Authenticated CLI or HTTP API for that host (`gh` for GitHub; Bitbucket REST via authenticated `curl` or a Bitbucket CLI)

If neither path works, **stop**. Tell the user the review cannot run until a code-management tool for this host is available.

Never use `gh` against Bitbucket. Never use Bitbucket REST against GitHub.

### Required operations

Map whatever MCP or CLI is available onto these operations. Host field names differ; always reduce to the same audit fields used by later phases.

#### Fetch PR metadata

Need author, destination (base) commit SHA, source (head) commit SHA, and canonical PR URL.

| Host | Typical request | Head SHA | Base/destination SHA | Author |
|------|----------------|----------|---------------------|--------|
| GitHub | `GET /repos/{owner}/{repo}/pulls/{pull_number}` | `head.sha` | `base.sha` | `user.login` |
| Bitbucket Cloud | `GET /2.0/repositories/{workspace}/{repo_slug}/pullrequests/{id}` | `source.commit.hash` | `destination.commit.hash` | `author.nickname` or `author.display_name` |
| Bitbucket DC | `GET {bitbucket_base_path}/rest/api/1.0/projects/{projectKey}/repos/{repo_slug}/pull-requests/{id}` | `fromRef.latestCommit` | `toRef.latestCommit` | `author.user.name` |

Set `pr_head_sha` to the source/head SHA.

#### Resolve merge base (`pr_base_sha`)

`pr_base_sha` is the common ancestor of destination and head — not merely the destination branch tip if it has moved.

| Host | Typical request | Merge-base field |
|------|-----------------|------------------|
| GitHub | `GET /repos/{owner}/{repo}/compare/{destination_sha}...{head_sha}` | `merge_base_commit.sha` |
| Bitbucket Cloud | `GET /2.0/repositories/{workspace}/{repo_slug}/merge-base/{head_sha}..{destination_sha}` | `hash` |
| Bitbucket DC | Local `git merge-base {destination_sha} {head_sha}`, or the host merge-base API if one is exposed | common ancestor SHA |

If the host API is unavailable, use `git merge-base {destination_sha} {head_sha}` on a local clone. Do not guess SHAs.

#### Fetch the PR review diff

Fetch a unified diff for **`pr_base_sha` → `pr_head_sha`**. Never substitute `deployed_sha` → `pr_head_sha` as the review diff.

| Host | Typical request |
|------|-----------------|
| GitHub | `GET /repos/{owner}/{repo}/compare/{pr_base_sha}...{pr_head_sha}` with `Accept: application/vnd.github.diff` |
| Bitbucket Cloud | `GET /2.0/repositories/{workspace}/{repo_slug}/diff/{pr_head_sha}..{pr_base_sha}` (two dots) |
| Bitbucket DC | `GET {bitbucket_base_path}/rest/api/1.0/projects/{projectKey}/repos/{repo_slug}/pull-requests/{id}.diff` (or commit-range diff when SHA range is supported) |

#### Commit history, commit → PR, commit diff

Used when tracing fixes or listing merges. Prefer host APIs; do not guess PR numbers.

| Operation | GitHub | Bitbucket Cloud | Bitbucket DC |
|-----------|--------|-----------------|--------------|
| List commits on a branch | `GET /repos/{owner}/{repo}/commits?sha={branch}` | `GET /2.0/repositories/{workspace}/{repo_slug}/commits/{branch}` | `GET {bitbucket_base_path}/rest/api/1.0/projects/{projectKey}/repos/{repo_slug}/commits?until={branch}` |
| PRs that contain a commit | `GET /repos/{owner}/{repo}/commits/{sha}/pulls` | `GET /2.0/repositories/{workspace}/{repo_slug}/commit/{sha}/pullrequests` | `GET {bitbucket_base_path}/rest/api/1.0/projects/{projectKey}/repos/{repo_slug}/commits/{sha}/pull-requests` |
| Commit unified diff | `GET /repos/{owner}/{repo}/commits/{sha}` with `Accept: application/vnd.github.diff` | `GET /2.0/repositories/{workspace}/{repo_slug}/diff/{sha}` | `GET {bitbucket_base_path}/rest/api/1.0/projects/{projectKey}/repos/{repo_slug}/diff?since={parent_sha}&until={sha}` (`since` optional; defaults to the parent of `until`) |

#### Post a PR comment

When a phase says to post a PR comment, use the detected host’s comment API (MCP equivalent if available). Post **only** the content that phase specifies.

| Host | Typical request |
|------|-----------------|
| GitHub | `POST /repos/{owner}/{repo}/issues/{pull_number}/comments` with `{ "body": "..." }` |
| Bitbucket Cloud | `POST /2.0/repositories/{workspace}/{repo_slug}/pullrequests/{id}/comments` with `{ "content": { "raw": "..." } }` |
| Bitbucket DC | `POST {bitbucket_base_path}/rest/api/1.0/projects/{projectKey}/repos/{repo_slug}/pull-requests/{id}/comments` with `{ "text": "..." }` |

### Other hosts

If a connected MCP clearly supports another git host with the same operations (metadata, merge base, unified diff, optional comments), use it the same way. GitHub and Bitbucket remain the required baseline.
