---
pr: kubernetes-sigs/prow#749
title: "fix(pipeline): reject duplicate cluster endpoints at startup"
head_sha: 93096b1e06a8d9ddaf7ddc3b6c95f482a4d3c4fb
base: main
reviewed_at: 2026-07-26T23:33:47Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-06-16T16:36:39Z
  gated_head_sha: 93096b1e06a8d9ddaf7ddc3b6c95f482a4d3c4fb
  reviewed_head_sha: 93096b1e06a8d9ddaf7ddc3b6c95f482a4d3c4fb
merged_at: 2026-06-16T17:45:39Z
refresh_log:
  - from: 93096b1e06a8d9ddaf7ddc3b6c95f482a4d3c4fb
    to: 93096b1e06a8d9ddaf7ddc3b6c95f482a4d3c4fb
    at: 2026-07-26T23:33:47Z
    summary: No code changes; no new comments/reviews beyond the gate. PR merged 2026-06-16T17:45:39Z by k8s-ci-robot.
---

## Gate

**Decision: merge**

**Since previous review:** PR merged 2026-06-16T17:45:39Z (merged by `k8s-ci-robot`, the standard tide/CI merge bot). No code changes since the reviewed head (`93096b1e0`) and no new comments or reviews beyond the approval/gate already recorded.

PR has `lgtm` and `approved` labels, `reviewDecision: APPROVED`, no code changes since review (OLD_SHA == NEW_SHA). The only open findings are two `should-fix` suggestions (report all collisions at once; deterministic iteration order) and two `nits` — none are correctness bugs, and the original review verdict was `approve` not `request-changes`. No API, config, or behavioral merge risk: the change is purely additive, scoped to `cmd/pipeline/`, and only gates startup on a misconfiguration that was already causing data loss silently.

**Gating list:**
- `[should-fix] Only the first duplicate collision is reported` (`cmd/pipeline/main.go:114-119`) — not addressed; acceptable for merge, the code is correct for the common case. Fine to file as a follow-up.
- `[should-fix] Non-deterministic error output from map iteration` (`cmd/pipeline/main.go:114`) — not addressed; same disposition as above, follow-up material.

**Process blocker (not a code concern):** `needs-ok-to-test` label is present — CI tests are blocked until an org member issues `/ok-to-test`. This is a Kubernetes bot process gate, not a code quality gate. Issuing `/ok-to-test` unblocks CI and the PR can merge once tests pass.

**Area 2 — merge risk:** None. No exported API surface touched. No config schema, flags, or defaults changed. No CRD or Kubernetes API changes. Behavioral change is intentional and correctly scoped: operators with duplicate endpoints (who were already experiencing silent data loss) will see a startup failure with an actionable message.

## Findings

### [should-fix] Only the first duplicate collision is reported
- where: `cmd/pipeline/main.go:114-119`
- concern: Returns on first collision. Operators with multiple duplicate groups must restart repeatedly. Collect all violations into a single error. Flagged independently by all three review perspectives.
- excerpt: |
    for host, ctxs := range hostToCtxs {
        if len(ctxs) > 1 {
            sort.Strings(ctxs)
            return fmt.Errorf("contexts %v resolve to the same cluster endpoint %s; deduplicate them in the kubeconfig", ctxs, host)
        }
    }

### [should-fix] Non-deterministic error output from map iteration
- where: `cmd/pipeline/main.go:114`
- concern: `range hostToCtxs` iterates non-deterministically. Which collision is reported first varies between runs. If changed to collect all violations, groups also need sorting for stable output.
- excerpt: |
    for host, ctxs := range hostToCtxs {

### [nit] Test doesn't verify error message content
- where: `cmd/pipeline/main_test.go:120-128`
- concern: Duplicate-hosts test only checks `wantErr: true`. A `strings.Contains` check on colliding context names and endpoint would guard against message regressions.
- excerpt: |
    err := validateUniqueClusters(tc.configs)
    if (err != nil) != tc.wantErr {
        t.Errorf("validateUniqueClusters() error = %v, wantErr %v", err, tc.wantErr)
    }

### [nit] Missing test cases for 3-way collision and multiple duplicate groups
- where: `cmd/pipeline/main_test.go:86-128`
- concern: A 3-context-same-host case would exercise `sort.Strings` more meaningfully. A two-distinct-hosts-each-with-duplicates case would document first-duplicate-wins behavior.

### [question] String-based host comparison and URL normalization
- where: `cmd/pipeline/main.go:112`
- concern: `rest.Config.Host` values like `https://k8s.example.com` vs `https://k8s.example.com:443` won't match. In practice client-go kubeconfig loader produces consistent values. Known limitation, not worth complicating the code.
- excerpt: |
    hostToCtxs[cfg.Host] = append(hostToCtxs[cfg.Host], ctx)

## Checked
- Placement: runs after `delete(configs, kube.InClusterContext)` and `allContexts` truncation; validates exactly the config set that creates informers
- No impact on single-context mode: configs truncated to one entry before validation
- Underlying bug confirmed at `controller.go:462`: context mismatch falls through without setting `wantPipelineRun = true`
- Pure function shape: `map[string]rest.Config` -> `error`, no side effects
- `sort.Strings(ctxs)` stabilizes within-group ordering in error messages
- Test structure: table-driven, consistent with existing `TestOptions`, covers boundary cases
- Import additions: `sort` (stdlib) and `k8s.io/client-go/rest` (existing transitive dep)
- Clean failure: check runs before informers/clients; no partial init or resource leaks
- Rollback safe: reverting removes check, controller starts (data-loss bug returns)

## Open questions
- Have you considered collecting all collisions into a single error so operators don't need multiple restart cycles?
- Is URL normalization (default ports, trailing slashes) a practical concern, or are kubeconfig generators consistent enough that string comparison is reliable?

## Followups

### Report all duplicate collision groups in a single error
- category: deferred-review
- necessity: should
- where: `cmd/pipeline/main.go:114-119`

```
In kubernetes-sigs/prow, following PR #749 ("fix(pipeline): reject duplicate cluster endpoints at startup",
head commit 93096b1e06a8d9ddaf7ddc3b6c95f482a4d3c4fb), the function validateUniqueClusters in
cmd/pipeline/main.go currently returns on the first collision group it finds. An operator with multiple
independent sets of duplicate endpoints (e.g. contexts A/B both pointing at cluster X, and contexts C/D
both pointing at cluster Y) must fix one, restart, discover the second, fix that, and restart again.

Task: Modify validateUniqueClusters (cmd/pipeline/main.go:109-121) to collect all duplicate groups before
returning, then emit a single combined error listing every collision. Requirements:

1. Collect all host entries where len(ctxs) > 1 into a slice of formatted strings.
2. Sort the context names within each group (sort.Strings(ctxs) — already done for the single-group case).
3. Sort the resulting slice of group error strings so the combined error is deterministic across runs.
4. Join them into one error message that preserves the existing guidance ("deduplicate them in the kubeconfig").
5. Add a test case with two distinct hosts each having duplicate contexts (e.g. ctx-a and ctx-b both pointing
   at https://a.example.com, and ctx-c and ctx-d both pointing at https://b.example.com) that asserts:
   - err is non-nil
   - the error message contains both host endpoints
   - the error message contains all four colliding context names

Acceptance criteria:
- All existing tests in cmd/pipeline/main_test.go still pass.
- The new test case passes.
- The function still returns nil when all hosts are unique.
- go build ./cmd/pipeline/... succeeds.

Out of scope: Do not change the host-comparison logic (raw string comparison of cfg.Host is intentional).
Do not add URL normalization. Do not touch controller.go or any other file.
```
