---
pr: kubernetes-sigs/prow#723
title: "fix(clonerefs): fall back to full clone when sparse checkout fails"
head_sha: b39aefab51e996cb2d94d5224685bd0e9727e3c7
base: main
reviewed_at: 2026-05-26T11:43:10Z
verdict: approve
refresh_log:
  - old_sha: 644f75be0c9cd7f6b685f16523e2f1a3cc32f093
    new_sha: b39aefab51e996cb2d94d5224685bd0e9727e3c7
    refreshed_at: 2026-05-26T11:36:38Z
    summary: "Tests refactored to table-driven style per droslean; fallback log level Info→Warn; cmdStart reverted to startTime; droslean COMMENTED with open design concern about permanent sparse failures"
---

## Findings

### [should-fix] record.Refs reflects fallback refs, not original request
- where: `pkg/pod-utils/clone/clone.go:75-88`
- concern: After fallback, `record = runClone(fullRefs, ...)` overwrites `record.Refs` with `fullRefs` which has `SparseCheckoutFiles: nil`. Downstream consumers inspecting `Record.Refs.SparseCheckoutFiles` will not know sparse checkout was originally requested. Add `record.Refs = refs` after the fallback `runClone` call.
- excerpt: |
    record = runClone(fullRefs, dir, gitUserName, gitUserEmail, cookiePath, env, user, token)
    record.Commands = append(failedCommands, record.Commands...)

### [should-fix] os.RemoveAll failure proceeds into doomed retry
- where: `pkg/pod-utils/clone/clone.go:81-83`
- concern: If `os.RemoveAll(cloneDir)` fails (e.g. permission denied), code logs a warning but proceeds to `runClone` into a dirty directory. The second attempt will almost certainly fail with a confusing unrelated error. Consider returning the original failed record when cleanup fails.
- excerpt: |
    if err := os.RemoveAll(cloneDir); err != nil {
        logrus.WithError(err).Warn("Failed to clean up clone directory for fallback")
    }

### [nit] Missing comment explaining why fallback exists
- where: `pkg/pod-utils/clone/clone.go:77`
- concern: Code is clear about what it does but not why. A one-line comment noting the submodule/gitlink edge case would save future maintainers from needing to find this PR.

### [nit] Fallback test could assert record.Failed more directly
- where: `pkg/pod-utils/clone/clone_test.go` (table-driven fallback case)
- concern: The fallback case checks `record.Failed` indirectly via an `if` branch rather than a direct assertion. Minor readability improvement; still present after table-driven refactor.

### [question] Silent permanent sparse checkout degradation
- where: `pkg/pod-utils/clone/clone.go:77-88`
- concern: droslean (2026-05-25): if sparse checkout is permanently broken for a repo (e.g. always-present submodule conflict), every clone silently degrades to full clone with no operator signal beyond a Warn in `clone-log.txt`. Log level was raised to Warn in current revision (was Info). Prucek's position: logs are in `clone-log.txt` in PJ artifacts; failures are expected rare edge cases. Open question for assigned reviewers: is Warn-in-clone-log adequate, or does this need a metric or ProwJob annotation?

## Checked
- `runClone` extraction is clean; no shadowing risk between `startTime` in `runClone` and `Run()` (separate functions); duration accounting correctly moved to `Run()` caller
- `fullRefs := refs` is a safe struct copy; nil-ing `SparseCheckoutFiles` correctly makes `isSparseCheckoutSet` return false
- `append(failedCommands, record.Commands...)` is safe — `failedCommands` is a separate slice reference
- Fallback only triggers when `isSparseCheckoutSet(refs)` is true AND `record.Failed` is true — no impact on non-sparse clones
- Tests use real git repos and assert on behavioral outcomes; table-driven refactor (per droslean) is clean
- No configuration, API, CLI flag, or ProwJob spec changes — purely internal resilience improvement
- No breaking changes, safe to roll back, zero upgrade friction for operators
- Deployment risk is LOW; only observable change is previously-failing sparse jobs now succeed via full clone

## Open questions
- Should `record.Refs` after a successful fallback reflect the original request (with `SparseCheckoutFiles`) or what was actually executed (without)? Debugging visibility argues for the original.
- Is warn-and-continue the right behavior when `os.RemoveAll` fails, or should the fallback be skipped entirely?
- Is Warn-level logging in `clone-log.txt` sufficient operator visibility for silent permanent sparse checkout degradation, or is a metric/ProwJob annotation needed? (droslean's open concern, 2026-05-25)
