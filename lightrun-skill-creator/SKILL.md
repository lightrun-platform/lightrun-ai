---
name: lightrun-skill-creator
description: >-
  Create and review runtime-oriented Lightrun MCP skills with deterministic
  section structure, exact canonical tool identifiers, explicit recovery
  branches, and auditable quality gates before handoff.
---

# Goal

Produce runtime-oriented Lightrun skills that are deterministic to execute, exact in MCP tool naming, and consistently verifiable through an auditable definition-of-done checklist.

# Scope

- In scope: creating or refining runtime-oriented skill specifications, enforcing canonical `lightrun__*` tool identifiers, defining preflight and recovery behavior, and adding measurable handoff criteria.
- In scope: requiring created runtime skills to declare Lightrun MCP dependency in metadata and OAuth readiness in `SKILL.md` documentation.
- Out of scope: executing runtime debugging sessions, modifying product runtime logic, or replacing user-specific environment setup instructions beyond MCP access prerequisites.

# Preconditions

- The author can edit skill documentation files in the local skills workspace.
- Runtime-oriented skill outputs require at least one MCP capability gate.
- Canonical runtime tool names are validated against this embedded canonical list.
- If execution-frequency notes are included, traffic expectations are stated explicitly.

# MCP Preflight

- Required tools:
  - `lightrun__get_runtime_sources`
- Pass criteria:
  - Tool identifier is present in the skill and matches canonical naming.
  - Capability gate is explicit and executable as the first runtime prerequisite.
- Fail criteria:
  - Tool identifier is missing, misspelled, or replaced with paraphrased naming.
  - Skill allows runtime investigation steps before capability gate success.
- On fail:
  - Go to `# Missing-MCP Recovery`.
  - Do not proceed with runtime-tool flow drafting until corrected.

# Flow

1. Lock canonical runtime tools from source of truth.
   - Tools: none
   - Success: every planned runtime tool name is copied exactly from the canonical list below.
   - Failure: any runtime tool identifier cannot be validated from the canonical list.
2. Build the fixed section structure in required order.
   - Tools: none
   - Success: headings appear in the exact order from Goal through Definition of Done Checklist.
   - Failure: heading order differs or required sections are missing.
3. Author deterministic preflight and fail branch.
   - Tools: `lightrun__get_runtime_sources`
   - Success: preflight pass/fail criteria and fail routing are explicit and testable.
   - Failure: preflight outcome is ambiguous or lacks fail routing.
4. Add runtime-flow tool usage contracts.
   - Tools: `lightrun__get_runtime_expression_values`, `lightrun__get_runtime_execution_duration`, `lightrun__get_runtime_execution_duration_samples`, `lightrun__get_runtime_numeric_metric`, `lightrun__get_runtime_numeric_metric_samples`, `lightrun__get_runtime_callstack`, `lightrun__get_runtime_execution_count`
   - Success: each flow step declares action, tools, success condition, and failure condition.
   - Failure: flow omits branch behavior or uses non-canonical tool identifiers.
5. Require dependency and icon declaration in each created runtime skill.
   - Tools: none
   - Success: created skill includes `agents/openai.yaml` with a `dependencies.tools` entry for MCP `lightrun` using the exact schema in this skill, plus interface icons (`icon_small`, `icon_large`) and `brand_color` that reuse this creator skill's icon assets; OAuth requirement is documented in created skill `SKILL.md`.
   - Failure: dependency/OAuth/icon requirements are absent, dependency shape is invalid, or `agents/openai.yaml` does not match the required schema.
6. Define deterministic recovery and strict resume gating.
   - Tools: `lightrun__get_runtime_sources`
   - Success: recovery loops through preflight and resume is blocked until gate success.
   - Failure: recovery is one-way or resume can happen before gate success.
7. Finalize output contract and measurable checklist.
   - Tools: none
   - Success: outputs cover pass, fail, evidence, and blocker states with one next action or blocker.
   - Failure: outputs are vague or checklist items are not auditable.

# Missing-MCP Recovery

1. Classify the failure as missing tool, tool call error, or empty sources.
2. In the created runtime skill `SKILL.md`, require installing the Lightrun MCP server and completing OAuth authorization.
3. Re-run `# MCP Preflight` using `lightrun__get_runtime_sources`.
4. Resume only when preflight passes; otherwise report blocker and stop.

# Resume Criteria

- Resume main flow only after `lightrun__get_runtime_sources` succeeds.
- Do not allow `lightrun__get_runtime_expression_values`, `lightrun__get_runtime_execution_duration`, `lightrun__get_runtime_execution_duration_samples`, `lightrun__get_runtime_numeric_metric`, `lightrun__get_runtime_numeric_metric_samples`, `lightrun__get_runtime_callstack`, or `lightrun__get_runtime_execution_count` before that gate is satisfied.

# Output Contract

- Preflight pass:
  - Report canonical tool verification status and selected next authoring action.
- Preflight fail:
  - Report failure type, blocking condition, and blocked downstream steps.
- Runtime evidence collected:
  - Report tool-by-tool evidence requirements and accepted success criteria.
- Runtime capture-time blocker:
  - Report failed tool identifier, blocker reason, and one explicit follow-up action.
- Final skill handoff:
  - Report file path, deterministic guarantees included, and checklist readiness.

# Definition of Done Checklist

- [ ] Required headings exist in exact required order.
- [ ] Canonical runtime tool identifiers match the embedded canonical list exactly.
- [ ] Created runtime skills include MCP dependency metadata for `lightrun`.
- [ ] Created runtime skills use `type`/`value` dependency schema (not `name`/`required`).
- [ ] Created runtime skills include OAuth requirement in `SKILL.md`.
- [ ] Created runtime skills define `icon_small`, `icon_large`, and `brand_color`.
- [ ] Created runtime skills reuse creator icon assets (no new icon files).
- [ ] Preflight pass and fail criteria are explicit and testable.
- [ ] Missing-MCP recovery loops through preflight before resume.
- [ ] Resume criteria block runtime flow before gate success.
- [ ] Each flow step has action, tools, success, and failure outcomes.
- [ ] Output contract covers pass, fail, evidence, and blocker states.
- [ ] Final output includes one clear next action or blocker.

## Canonical Runtime Tool List

- `lightrun__get_runtime_sources`
- `lightrun__get_runtime_expression_values`
- `lightrun__get_runtime_execution_duration`
- `lightrun__get_runtime_execution_duration_samples`
- `lightrun__get_runtime_numeric_metric`
- `lightrun__get_runtime_numeric_metric_samples`
- `lightrun__get_runtime_callstack`
- `lightrun__get_runtime_execution_count`

## Created Skill Metadata Requirements

For each runtime skill generated by this creator, require `agents/openai.yaml` to include exactly these top-level blocks:

- `interface`
- `dependencies`

For each runtime skill generated by this creator, require `agents/openai.yaml` to include:

- `interface.icon_small`
- `interface.icon_large`
- `interface.brand_color`
- `dependencies.tools` entry for MCP `lightrun` using the exact schema below

Reuse this creator skill's icon definitions from `agents/openai.yaml` for generated skills.

### Required Interface Icon Values

For generated skills, inherit these fields directly from this creator skill's `agents/openai.yaml`:

```yaml
interface:
  icon_small: "<same value as lightrun-skill-creator interface.icon_small>"
  icon_large: "<same value as lightrun-skill-creator interface.icon_large>"
  brand_color: "<same value as lightrun-skill-creator interface.brand_color>"
```

Do not hardcode repository-relative icon paths in the generated skill spec.

### Required MCP Dependency Shape

```yaml
dependencies:
  tools:
    - type: "mcp"
      value: "lightrun"
      description: "Lightrun MCP server"
      transport: "streamable_http"
      url: "https://app.lightrun.com/mcp"
```
