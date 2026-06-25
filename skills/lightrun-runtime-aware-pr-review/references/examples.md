# Examples

Fictional illustrations — replace repo, paths, and numbers with the PR under review.

### Phase 2 in first pass — hot path (PR #842)

PR tightens validation on `POST /api/v1/checkout`. Snapshot on `PaymentService.java:88`. Historical probe shows 12 hits in 7d; 3 live hits in 40s with `amount=4999`, `currency=USD`.

```markdown
## Runtime-aware PR review — PR #842
**Merge recommendation:** Approve with notes
**Terminal state:** RUNTIME_COMPLETE
**Runtime verdict:** CONFIRMED_ON_RUNTIME_SAMPLES
**Evidence:** LIVE_AND_HISTORICAL
**Runtime confidence:** Strong
**Confidence reason:** 12 historical + 3 qualifying live hits on changed logic; patch simulation UNCHANGED.

### Patch simulation
| Inputs | Baseline | PR head | Outcome |
| amount=4999, currency=USD | Passes | Same validation path | UNCHANGED |
```

Deliver merge recommendation in the same response — no handoff needed.

### Phase 1 — admin path (PR #519)

PR adds optional CSV export. Snapshot on `ReportController.java:142`. 90s collection, 0 hits after verification.

```markdown
## Runtime-aware PR review — PR #519 (PAUSED)
**Terminal state:** RUNTIME_PENDING_HANDOFF
**Runtime verdict:** INSUFFICIENT_RUNTIME_EVIDENCE (pending)

### Preliminary static findings
Risk: export omits rate-limit header on large responses.

**Action IDs:** `a1b2c3d4-...` @ ReportController.java:142
**Trigger:** Admin console → Reports → Export usage CSV
**Resume:** `resume pr-519 review` | **Static-only:** `static only`
```

### Resume — still 0 hits (PR #519)

```markdown
## Runtime-aware PR review — PR #519 (PAUSED)
**Terminal state:** RUNTIME_PENDING_HANDOFF

Checked action IDs — still 0 hits. Did you run Admin → Reports → Export?
Reply after triggering, or `static only` for a final static-based recommendation.
```

### Mixed PR, no runtime on changed logic (PR #719)

Frontend form fix + backend role list alignment. Historical 0 on authority paths; Cypress covers hypothesis. User says `static only`.

```markdown
## Runtime-aware PR review — PR #719
**Merge recommendation:** Approve with notes
**Terminal state:** STATIC_ONLY_COMPLETE
**Runtime verdict:** RUNTIME_NOT_USED
**Evidence:** NO_RUNTIME_TRAFFIC
**Runtime confidence:** Insufficient
**Confidence reason:** Zero hits on capture line list; closeout via `static only` and SCENARIO tests.

### Scenario analysis (SCENARIO)
| Inputs | Baseline | PR head | Outcome |
| (from Cypress test) | 400 on save | 200 on save | IMPROVEMENT |
```

### Config-gated export (PR #901)

New property `billing.export.include-metadata` defaults `true`. Historical hits on invoice PDF path; live action collected 0 hits in 60s.

```markdown
## Runtime-aware PR review — PR #901
**Merge recommendation:** Approve with notes
**Terminal state:** RUNTIME_COMPLETE
**Runtime verdict:** CONFIRMED_ON_RUNTIME_SAMPLES
**Evidence:** HISTORICAL
**Runtime confidence:** Moderate
**Confidence reason:** 8 historical RUNTIME-row hits on invoice PDF path; 0 qualifying live hits this session.

### Patch simulation
| Inputs | Baseline | PR head (default) | PR head (opt-in) | Outcome |
| invoiceId=8842, format=pdf | Embeds metadata footer | Same | Omits footer block | UNCHANGED (default) / INTENTIONAL (opt-in) |

### Supplemental runtime trigger
- **Action IDs:** `d4e5f6...` @ `com/acme/billing/InvoiceRenderService.java:204`
- **Trigger:** Billing console → Invoices → Download PDF for any open invoice
- **Expected signal:** `invoiceId`, `format=pdf` at listed line
- **Resume:** `resume pr-901 review`
```

Merge recommendation ships in the same response — supplemental trigger lets the user corroborate live without blocking merge.
