---
name: lightrun-runtime-aware-pr-review
description: >-
  Review a pull request with Lightrun runtime evidence and diff-aware risk
  analysis. Use when a user wants to review a PR, check their changes, do a
  code review with runtime data, validate changes against production, or get a
  merge-risk assessment.
---

# Goal

Produce a reviewer-ready PR assessment that combines live or historical production samples with diff analysis and a clear merge recommendation.

# Scope

- In scope: PR intake, scoped diff review, static scan, runtime feasibility, historical probe, live capture, handoff/resume, patch simulation, scenario analysis, and merge recommendation.
- Out of scope: code changes, mandatory ticket integration, and repository-specific deployment automation.

# Preconditions

- PR diff and changed files available (`gh`, git fallback, or user-provided).
- Lightrun MCP installed and authenticated on at least one server with agents for the target service.
- Persist review state in the handoff block: `correlationKey` (`pr-<number>-review`), action IDs, targets (chat is valid in Cursor).

# MCP Tool Discovery

Follow [MCP tool discovery](references/mcp-tool-discovery.md). Read tool schemas at run time; use exposed names and selector modes exactly as documented.

# MCP Preflight

Complete source discovery per [MCP tool discovery](references/mcp-tool-discovery.md) before runtime capture.

- Enumerate every enabled Lightrun MCP server until a pool with `agentCount > 0` matches the changed service.
- Dual-server setups: **Historical/query** and **Async lifecycle** capabilities may live on different MCP servers. Record `mcpHistorical`, `mcpLive`, `agentPoolName`, selector.
- When one server shows 0 agents, try the next before failing preflight.
- Pass and fail criteria: see [Source discovery (preflight)](references/mcp-tool-discovery.md#source-discovery-preflight).

# Resume Criteria

- Follow **Max 2 actions** on every resume.
- Load `correlationKey` and persisted action IDs before creating new actions.
- Skip full static rescan unless PR head changed.
- Re-enumerate exposed Lightrun MCP tools when resuming a later run.

# Review Principles

- **Runtime first** — After a quick static scan, probe historical hits, then create live actions when backend change is verifiable.
- **Pre-change lines only** — Instrument baseline executable locations; skip PR-head-only lines and undeployed symbols.
- **Collect in-session** — Poll live actions **60–120 seconds**; on hot paths stop early at ≥3 stable hits (historical + live count together).
- **Hits on changed logic → complete review in same pass** — Live or historical hits on the **changed branch** → Phase 2 immediately. Hand off only after verified zero hits.
- **Static blockers still get runtime** — Record static findings; continue runtime when the path is verifiable.
- **Hits that count** — Only locations executing the **changed logic**. Nearby endpoints (e.g. list API vs update API) supply context but do not satisfy Phase 2 alone.
- **Diff-anchored capture** — From the scoped diff, list baseline package path + line before any live action; place live actions on those lines only.
- **Max 2 actions** — At most 2 live actions active at any time. Rank capture points and drop the lowest-signal. On resume, poll existing actions before creating new ones; recreate only if expired or `FAILED`. This rule governs all steps below — reference it instead of restating.

# Flow

1. Intake and scope the diff.
   - Fetch PR (in order): `gh pr view` / `gh pr diff` / `gh pr view --json commits,files,title,body` → git `fetch origin pull/<N>/head:pr-<N>` → user-provided diff.
   - If `gh` fails: extract intent from commit messages + changed paths; note *PR description unavailable*.
   - Record `prCommits[]`, `changedFiles[]`, `reviewScope` (`focused` | `full-branch`).
   - **Focused recipe** when branch is stale or noisy:
     1. Read ticket from PR title/body/commits (e.g. `PROJ-123`).
     2. `git log --oneline <merge-base>..pr-<N> | grep PROJ-123`
     3. Diff from parent of first matching commit → head.
   - Baseline: PR base → head. When historical hits include a deploy commit, bind line numbers to that commit's file view.

2. Static scan (parallel with planning).
   - Scan **scoped** diff; record Blocker / Risk / Nit per [static and feasibility](references/static-and-feasibility.md).
   - Continue to runtime planning regardless of static findings.

3. Feasibility and hypothesis — see [static and feasibility](references/static-and-feasibility.md).

4. Capture plan (before tool calls).
   - From the scoped diff, write a **capture line list** — baseline package path + executable line per planned action; historical and live steps both target this list (subject to **Max 2 actions**).
   - Per location: file, line, expressions, optional condition, traffic class (**hot** | **trigger**), why it matters.
   - **Path:** runtime package format (`com/acme/billing/...`), not repo path.
   - **Line:** executable on deployed baseline.
   - **Expressions:** parameters, locals, fields in scope; max 3 per action. Java: prefer `isFoo()` when that is the bean accessor; simplify if an expression errors at retrieve time.
   - **Undeployed symbols:** snapshot upstream inputs (`fileName`, `platform`) instead of PR-only flags.

5. Historical probe (first or parallel).
   - Use **Historical/query** capability on capture line list rows; if variables truncated → variable-recovery capability per key input (per [MCP tool discovery](references/mcp-tool-discovery.md)).
   - Recent hits (<7d) on **changed logic** → use for patch simulation.
   - When hit count is ≥3, retrieve values from multiple distinct hits when variable recovery is available.
   - Historical hit count contributes to hot-path early exit (≥3 stable hits).

6. Live actions and in-session collection.
   - Use **Async lifecycle** capability on `mcpLive` at capture line list path:line with `correlationKey: pr-<number>-review`.
   - On line/path binding error: re-read baseline at same file, pick nearest executable line in same method, update list row, retry once.
   - Persist action IDs, capture line list, servers, target in handoff block.

   | Path class | Window |
   |------------|--------|
   | **Hot** | Up to 60s; stop early at ≥3 stable hits (historical + live) |
   | **Trigger** | Full **60–120s**; organic traffic may still arrive |

   Retrieve values whenever hit count increases.

7. Zero-hit verification (before handoff).
   - Confirm live actions **RUNNING** on capture line list rows; historical checked on same list; target pool matches service.
   - **Branch coverage matrix** when diff has multiple paths (e.g. zip vs script):

   | Path | Historical | Live | Label |
   |------|------------|------|-------|
   | Primary changed branch (hypothesis inputs) | hits | hits | **RUNTIME** |
   | Same line, different inputs | hits | hits | **context** |
   | Secondary branch | 0 | 0 | SCENARIO (tests) |

   Set **Evidence** per [output contract](references/output-contract.md#evidence-labels).

   - Hits on changed logic (live and/or historical) → step 8 (Phase 2).
   - Verified zero hits → step 9 (handoff). State uncovered paths in Static vs runtime.

8. Phase 2 — complete review.

   Enter when: hits on changed logic; resume with new hits; user says `static only`; or not verifiable.

   **Patch simulation** (live/historical hits):
   1. Use captured expression values as inputs.
   2. Evaluate baseline logic at the changed region.
   3. Evaluate PR-head logic with the same inputs.
   4. Label: `UNCHANGED` | `REGRESSION` | `IMPROVEMENT` | `INDETERMINATE`.
   5. Config-gated: simulate (a) default and (b) opt-in per hit when relevant.

   **Scenario analysis** (`static only` / `NO_RUNTIME_TRAFFIC`):
   - Extract inputs from Cypress/unit test arrange/assert blocks; label rows `SCENARIO`; cite test name.
   - Separate section from patch simulation.

   **Supplemental runtime trigger** — include in every Phase 2 closeout when **either** applies:
   - Evidence is `HISTORICAL` and a live action was created during this review, or
   - Branch coverage matrix has SCENARIO-only paths the user may want to verify before rollout.
   - Content: action IDs, capture line list rows, exact trigger steps, expected expressions, `resume pr-<number> review`.
   - Keep live actions **RUNNING** through closeout when Evidence is `HISTORICAL`; cancel only after this section is emitted or user opts out.

   Emit one review block; merge recommendation per Output Contract rubric.
   On `STATIC_ONLY_COMPLETE`, cancel persisted actions via **Async lifecycle** cancel capability unless user asks to keep them.

9. Phase 1 — handoff.

   When verified zero hits after collection window:
   - Preliminary static findings (no merge recommendation line).
   - Handoff block from Output Contract.
   - Terminal state: `RUNTIME_PENDING_HANDOFF`; runtime verdict: `INSUFFICIENT_RUNTIME_EVIDENCE` (pending).
   - Offer: reply **`static only`** now if you lack access to trigger environment (on-prem, rare config org, etc.).

10. Resume.
    1. Load `correlationKey` / action IDs; skip static rescan unless PR head changed.
    2. Poll per **Max 2 actions**.
    3. `COMPLETED` / hits increased → retrieve values → Phase 2.
    4. `FAILED` / expired → recreate same file, line, expressions, target.
    5. Still 0 hits → ask whether user ran the trigger; re-handoff; hold merge recommendation.
    6. **`static only`** → Phase 2 scenario analysis; cancel actions; `STATIC_ONLY_COMPLETE`.

# Output Contract

Follow the [output contract](references/output-contract.md) for terminal states, merge recommendation rubric, runtime verdict values, and output templates (handoff block and complete review).

# Examples

See [examples](references/examples.md) for fictional illustrations covering hot-path Phase 2, admin-path handoff, resume with zero hits, mixed PRs, and config-gated changes.
