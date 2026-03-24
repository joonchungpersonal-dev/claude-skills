# AutoForge Convergence Rules

## Per-Feature Convergence Gate

A feature converges when ALL checks pass simultaneously on the same code version AND scores are stable for 2 consecutive passes.

### Check Thresholds

| Check | Tool | Pass Criterion |
|-------|------|---------------|
| Code quality | `/grill` | 0 CRITICAL, 0 HIGH, ≤3 MEDIUM |
| Doc accuracy | `/quick-check` | Score ≥ 90 (skip if no docs) |
| UX quality | Playwright + heuristic eval | Score ≥ 75/100, 0 critical a11y violations (skip if no HTML) |

### Stability

Scores must be within delta < 2 for 2 consecutive passes on each applicable check.

### Circuit Breaker

- Max 5 review passes per feature
- Max 3 integration passes
- If not converged at limit, write `NEEDS_HUMAN.md` listing unresolved issues
- Never exceed limits — no exceptions

## Fix Policy (Unattended)

| Severity | Action |
|----------|--------|
| CRITICAL | Auto-fix immediately |
| HIGH | Auto-fix immediately |
| MEDIUM | Auto-fix if confidence > 80%, else defer |
| LOW | Defer |

## Regression Prevention

- After applying fixes, re-run ONLY the failed checks (not full gauntlet)
- If a fix causes a previously-passing check to fail: REVERT that fix, try an alternative approach
- If 3 alternative approaches all cause regressions: defer the finding and move on
- Track per-check scores across passes to detect oscillation

## Integration Convergence

All features must pass integration tests simultaneously:
- Cross-feature smoke test (files load, data flows between features)
- End-to-end Playwright test (full workflow)
- Combined `/grill` on entire codebase
- 2 consecutive passes with 0 CRITICAL/HIGH
