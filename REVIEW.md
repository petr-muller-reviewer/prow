---
pr: kubernetes-sigs/prow#810
title: "chore(deps): bump the kubernetes group across 2 directories with 6 updates"
head_sha: 26bd2e37d3316d2fe9133aaaa98f4a4c913abb32
base: main
reviewed_at: 2026-08-05T21:47:16Z
verdict: request-changes
---

## Summary

Dependabot group bump, dep-only (only `go.mod`/`go.sum` in root and `hack/tools/` changed, no project source). Root direct bumps: `k8s.io/api`, `k8s.io/apimachinery`, `k8s.io/client-go` v0.33.11→v0.36.3, `k8s.io/utils` pseudo-version bump, `sigs.k8s.io/controller-runtime` v0.21.0→v0.24.1. `hack/tools/go.mod`: `k8s.io/code-generator` v0.32.8→v0.36.3, `sigs.k8s.io/controller-tools` v0.17.3→v0.21.0. Large indirect churn consistent with these major jumps (structured-merge-diff v4→v6, kube-openapi, klog, gnostic-models, etc.).

CI on the PR (`pull-prow-unit-test`, `pull-prow-unit-test-race-detector-nonblocking`, `pull-prow-verify-lint`) is red — confirmed compile failures, not flakes.

## Findings

### [blocking] apimachinery removed legacy `util/diff` helpers, breaking ~20 packages
- where: multiple, e.g. `pkg/simplifypath/simplify_test.go:146`, `pkg/config/branch_protection_test.go:318,474,477,480,484,487,490`, `pkg/git/v2/executor_test.go:195,198,201,215`, `pkg/plugins/config_test.go:150,209,554,738,975`, `cmd/branchprotector/protect_test.go:1660`, `cmd/deck/main_test.go:372`, `cmd/pipeline/controller_test.go:944,946,1098,1135`, `pkg/bugzilla/client_test.go:807`, `pkg/clonerefs/parse_test.go:166`, `pkg/config/org/org_test.go:129`, `pkg/crier/reporters/gerrit/reporter_test.go:2475`, `pkg/gerrit/adapter/adapter_test.go:453,591`, `pkg/gerrit/adapter/trigger_test.go:74,77,171`, `pkg/ghcache/coalesce_test.go:114,219`, `pkg/git/v2/interactor_test.go:90,202,281,343,407,469`, `pkg/kube/config_test.go:112`, `pkg/kube/metrics_test.go:138`, `pkg/pjutil/filter_test.go:414`, `pkg/pjutil/pjutil_test.go:830,833,836,906,909,1503`, `pkg/config/config_test.go:9536`, `pkg/config/tide_test.go:1288,1579`
- concern: `k8s.io/apimachinery/pkg/util/diff` dropped `StringDiff`/`ObjectDiff`/`ObjectReflectDiff`/`ObjectGoPrintDiff`/`ObjectGoPrintSideBySide` between v0.33.11 and v0.36.3, replaced by a single `Diff(a, b any) string`. Every test file above calls the old removed names and fails to compile with `undefined: diff.ObjectReflectDiff` / `diff.StringDiff` / `diff.ObjectDiff`. This is not exhaustive — CI truncated some file lists with "too many errors".
- excerpt: |
    pkg/simplifypath/simplify_test.go:146:
        t.Errorf("%s: got incorrect simplification: %v", diff.StringDiff(actual, expected))

### [blocking] controller-runtime fake informer field became unexported
- where: `pkg/plank/reconciler_test.go:153-154`
- concern: build fails with `unknown field Synced in struct literal of type controllertest.FakeInformer, but does have unexported synced`. `sigs.k8s.io/controller-runtime/pkg/internal/controller/controllertest.FakeInformer` changed shape between v0.21.0 and v0.24.1; the exported `Synced` field is gone.
- excerpt: |
    pkg/plank/reconciler_test.go:153-154 (struct literal setting `Synced: ...`)

### [question] freshness of the k8s.io/* trio
- where: `go.mod` (`k8s.io/api`, `k8s.io/apimachinery`, `k8s.io/client-go`, `k8s.io/code-generator` all → tags cut 2026-07-23)
- concern: these releases are ~6 days old as of this review, under the usual soak-time guideline. Not the primary blocker (the build break is), but worth factoring into the timing of a re-bump once the code fixes land.

## Checked
- No project code changed outside dependency manifests — dep-only PR, no separate code review needed.
- No usage of controller-runtime webhook `Validator`/`Defaulter` (0.23 breaking API change) or `GetEventRecorderFor` (events API migration) in the tree — those breaking changes from the controller-runtime release notes don't add further blast radius beyond the `FakeInformer` issue found above.
- `sigs.k8s.io/controller-runtime` v0.24.1 (2026-05-11) and `sigs.k8s.io/controller-tools` v0.21.0 (2026-05-06) are ~11 weeks old — fine on freshness.
- `k8s.io/utils` bump is a pseudo-version (untagged commit, 2026-02-10) — normal for that module, not a provenance concern.
- Import surface confirmed heavy/core for all root direct deps (`client-go` 76 files, `controller-runtime` 54 files, `apimachinery` 276 files) — consistent with how broadly the breakage above landed.

## Open questions
- Is there a companion PR (or does one need to be opened) to migrate the `diff.ObjectReflectDiff`/`diff.StringDiff`/`diff.ObjectDiff` call sites to the new `diff.Diff` API, or to `cmp.Diff`/testify assertions?
- Same question for `pkg/plank/reconciler_test.go`'s use of the now-unexported `FakeInformer.Synced` field.
- Given the k8s.io/* releases are only 6 days old, is there a preference to wait a bit longer before re-attempting this bump once the code fixes are ready?
