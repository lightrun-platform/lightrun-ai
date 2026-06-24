# Output Contract

## Terminal states

| State | When |
|-------|------|
| `RUNTIME_PENDING_HANDOFF` | 60–120s window done, 0 hits after verification; user must trigger or say `static only` |
| `RUNTIME_COMPLETE` | Hits captured; final review delivered |
| `RESUMED_RUNTIME_ACTION` | Phase 2 after checking prior action IDs with new hits |
| `STATIC_ONLY_COMPLETE` | User opted out or closeout after verified `NO_RUNTIME_TRAFFIC` |
| `SKIPPED_RUNTIME_WITH_REASON` | Not runtime-verifiable or no valid target after discovery |

## Merge recommendation

| Static | Runtime | Recommend |
|--------|---------|-----------|
| Blocker | any | Request changes or Block |
| No blockers; tests cover hypothesis | Hits → UNCHANGED/IMPROVEMENT at default config | Approve or Approve with notes |
| Risks only | Hits align with static | Approve with notes |
| Risks only | `NO_RUNTIME_TRAFFIC`; strong SCENARIO + tests | Approve with notes; call out unverified risks |
| Blocker contradicted by runtime | Conflict | Request changes; note conflict |

## Runtime verdict

| Verdict | When |
|---------|------|
| `CONFIRMED_ON_RUNTIME_SAMPLES` | Observed inputs → UNCHANGED or IMPROVEMENT at default/current prod config |
| `RISK_ON_RUNTIME_SAMPLES` | Hits show REGRESSION for observed inputs, or risky opt-in path untested but material |
| `RUNTIME_NOT_USED` | `static only`, not verifiable, or closeout after verified `NO_RUNTIME_TRAFFIC` |
| `INSUFFICIENT_RUNTIME_EVIDENCE` | Pending handoff only |

When valid target existed but no hits after trigger + resume + `static only`: note `NO_RUNTIME_TRAFFIC` in runtime evidence section.

## Evidence labels

| Label | When |
|-------|------|
| `LIVE` | Live retrieval returned values whose inputs match the **regression hypothesis** (same changed-logic branch — not merely the same line with different inputs) |
| `HISTORICAL` | Patch simulation uses historical values on capture line list rows; no qualifying live retrieval |
| `LIVE_AND_HISTORICAL` | Both qualifying live and historical captured values on changed logic |
| `NO_RUNTIME_TRAFFIC` | Valid target; zero values on listed lines after window + retry + `static only` closeout |

Same-line hits with different inputs (e.g. `agent.zip` vs `plugin.vsix` at the same line) supply context only — label those rows **context**, not **RUNTIME**, unless inputs match the hypothesis.

## Handoff block (Phase 1)

```markdown
## Review paused — awaiting runtime trigger

**Terminal state:** RUNTIME_PENDING_HANDOFF
**Correlation key:** pr-<number>-review
**Action IDs:** <id1>, <id2>  (max 2)
**MCP:** historical=<server> live=<server> / <pool> / <selector>
**Persisted at:** chat handoff block

**Collection window:** 60–120s elapsed, 0 hits after verification — trigger required.

**Trigger this flow:**
1. <exact steps>

**Expected signal:**
- `<package/path/File.java:line>` → `<expression>` = `<pattern>`

**Resume:** `resume pr-<number> review` (after triggering)
**Static-only:** `static only` — final recommendation from static + SCENARIO (no trigger access)
```

## Complete review (Phase 2)

Emit once. Include merge recommendation line.

```markdown
## Runtime-aware PR review — PR #<number>

**Merge recommendation:** Approve | Approve with notes | Request changes | Block
**Terminal state:** RUNTIME_COMPLETE | RESUMED_RUNTIME_ACTION | STATIC_ONLY_COMPLETE
**Runtime verdict:** CONFIRMED_ON_RUNTIME_SAMPLES | RISK_ON_RUNTIME_SAMPLES | RUNTIME_NOT_USED
**Evidence:** LIVE | HISTORICAL | LIVE_AND_HISTORICAL | NO_RUNTIME_TRAFFIC — see [Evidence labels](#evidence-labels); live labels require inputs matching the regression hypothesis
**Review scope:** focused | full-branch
**MCP:** historical=<server> live=<server>
**Baseline:** <base> → <head>

### Static findings
### Runtime evidence (+ branch coverage matrix if multi-path)
### Patch simulation (hits) OR Scenario analysis (SCENARIO rows)
### Static vs runtime
### Supplemental runtime trigger (when Evidence is HISTORICAL with live actions, or SCENARIO-only paths remain)
- **Action IDs:** <id1>, <id2>
- **Capture lines:** <package/path/File.java:line>, ...
- **Trigger:** <exact steps>
- **Expected signal:** `<expression>` = `<pattern>` at listed line
- **Resume:** `resume pr-<number> review` — upgrades `HISTORICAL` → `LIVE_AND_HISTORICAL`
### Next action

*Runtime evidence captured via Lightrun MCP.*
```
