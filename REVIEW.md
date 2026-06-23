---
pr: kubernetes-sigs/prow#765
title: "fix utf-8 parsing errors with metrics"
head_sha: 0b95b4a15ccc3b8fd4a20d5b19f24045be34a0a8
base: main
reviewed_at: 2026-06-23T11:47:18Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-06-23T12:12:02Z
  gated_head_sha: 0b95b4a15ccc3b8fd4a20d5b19f24045be34a0a8
  reviewed_head_sha: 0b95b4a15ccc3b8fd4a20d5b19f24045be34a0a8
---

## Gate

**Decision: merge**

This is a crash fix for production deck pods. The code is unchanged since review (same SHA). The two should-fix suggestions (missing test, unsanitized `r.Method`) are not addressed but are genuinely non-blocking for a panic fix — the core fix is correct and uses the right stdlib function. No GitHub reviews requested changes. No inline comments. No merge risk: the change is internal to an unexported function, touches no API surface, no configuration, no behavioral semantics beyond "stop panicking." The PR has `lgtm` but is missing `approved` — needs `/approve` from a `pkg/OWNERS` approver (`cjwagner`); `dims`'s `/approve` was not sufficient per OWNERS. That's a process gate, not a code gate.

### Findings disposition (Area 1)
- [should-fix] No test for invalid UTF-8 handling (`REVIEW.md`): **not-addressed** — no new commits. Acceptable: the fix relies on well-tested `strings.ToValidUTF8`; a test is nice-to-have, not a merge gate for a crash fix.
- [should-fix] `r.Method` not sanitized (`REVIEW.md`): **not-addressed** — `r.Method` at `http.go:148` is still unwrapped. Acceptable: Go's HTTP server rejects invalid methods before they reach handlers; theoretical risk only.
- [nit] Redundant `validMetricLabel` on path: **not-addressed** — harmless, not a gate.
- [nit] Missing comment on `validMetricLabel`: **not-addressed** — not a gate.
- [question] UA cardinality: **not-addressed** — pre-existing, not introduced by this PR, not a gate.

### Merge risk (Area 2)
No notable merge risk. `validMetricLabel` is unexported. No exported API surface changed. No configuration schema changes. No behavioral changes beyond crash prevention. No flag/env/default changes. Safe to roll back (re-exposes panic, no state corruption). All existing Prow deployments benefit with zero action from operators.

### Process gates
- Missing `approved` label. `pkg/OWNERS` requires approval from `cjwagner`. `dims` issued `/approve` but is not listed as an approver for `pkg/`.

## Summary

Adds `validMetricLabel()` wrapping `strings.ToValidUTF8(value, "")` around the `path` and `user_agent` metric labels in `traceHandlerWithCustomTimer`. Prevents a panic in `prometheus.(*HistogramVec).With()` caused by invalid UTF-8 bytes in the User-Agent header (triggered by a JNDI/Log4Shell probe containing `\xbf`).

## Findings

### [should-fix] No test for invalid UTF-8 handling
- where: `pkg/metrics/http_test.go`
- concern: No test case exercises the new `validMetricLabel` behavior. `TestHandleWithMetricsCustomTimer` has all the scaffolding; a second case with a UA like `"bot\xbf"` would prevent regression and document the failure mode. Flagged by 2/3 reviewers.

### [should-fix] r.Method not sanitized
- where: `pkg/metrics/http.go:149`
- concern: `r.Method` used as a Prometheus label without `validMetricLabel()`. Go's HTTP server rejects non-token methods (RFC 7230 3.1.1) so practical risk is very low, but wrapping it uniformly costs nothing and eliminates per-field reasoning. Flagged by 2/3 reviewers.
- excerpt: |
    "method": r.Method,

### [nit] Redundant validMetricLabel on path
- where: `pkg/metrics/http.go:147`
- concern: `simplifier.Simplify()` always returns valid UTF-8 from compile-time string literals. Wrapping is harmless but could mislead future maintainers into thinking Simplify returns arbitrary user input.
- excerpt: |
    "path": validMetricLabel(simplifier.Simplify(r.URL.Path)),

### [nit] Missing comment on validMetricLabel
- where: `pkg/metrics/http.go:158-160`
- concern: Function name is clear but a one-liner explaining Prometheus panics on invalid UTF-8 label values would convey the severity of what happens without it.

### [question] User-Agent label cardinality as a DoS vector
- where: `pkg/metrics/http.go:84`
- concern: `user_agent` label stores full raw UA string; every distinct UA creates a new time series. Pre-existing issue not introduced here, but the attack payload shows active probing. Worth asking about follow-up plans.

## Checked
- `strconv.Itoa` for status label always produces ASCII
- `strings.ToValidUTF8` with empty replacement is the right stdlib choice
- No changes to metric label keys or histogram bucket definitions
- Import placement correct
- Fix applied at the right layer: boundary where user input enters metrics
- Shared `pkg/metrics/http.go` via `TraceHandler` benefits all components
- Deployment risk LOW: no config/API changes, zero upgrade friction, safe rollback

## Open questions
- Would you add a test case with an invalid UTF-8 User-Agent to `TestHandleWithMetricsCustomTimer`?
- Would you wrap `r.Method` in `validMetricLabel()` for uniform sanitization?
- Any follow-up planned for `user_agent` label cardinality?
