## Phase 1 — Feasibility Check

First, determine: can this PR be verified with runtime production data?

Before evaluating observability, check whether the changed files are in a Lightrun-supported runtime stack:
- `Java`
- `Python`
- `.NET`
- `Node.js`

If the changed files are outside these stacks, treat the PR as **NOT verifiable** with Lightrun in this workflow.

The PR is **NOT verifiable** if ALL meaningful changes fall into one or more of these categories:
- Build configuration, CI pipelines, static types, annotations, prompts, or comments only
- Pure structural refactoring with no behavioral difference (rename, reformat, extract with identical logic)
- Logic that only runs at startup or compile time with no observable state afterward
- Dead code or test-only changes
- Net-new code paths that do not yet exist in production (nothing to instrument before the change lands)
- Changes where correctness requires triggering a specific production event that cannot be passively observed

If the PR is **NOT verifiable**:

1. Emit the message below in chat.
2. Respond with only that same message and stop.

### User Message
> **This PR cannot be verified with runtime production data.**
> **Reason:** [Concise explanation of why none of the changes are observable in current production code.]
