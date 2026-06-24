# Static scan and feasibility

## Static severity

| Level | Meaning |
|-------|---------|
| **Blocker** | Likely production break, security, or data integrity issue |
| **Risk** | Plausible regression or ops gap; merge with fix or follow-up |
| **Nit** | Style, docs, test gap; non-blocking |

## Runtime feasibility

| Change | Runtime approach |
|--------|------------------|
| Backend on live path | Verifiable — proceed |
| Frontend / browser only | Static review only for that file |
| Mixed PR | Backend → runtime; frontend → static |
| Docs / tests only | Static-only review |
| Config-gated change (`@Value`, env, YAML; default preserves prod) | Patch simulate **(a) default config** and **(b) opt-in** when relevant; regressions on (a) block merge |

## Regression hypothesis

One sentence before capture planning:

> If merged, **[change]** could cause **[failure]** under **[condition]**.
