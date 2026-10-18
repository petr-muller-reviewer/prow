---
pr: kubernetes-sigs/prow#917
title: "chore(deps): bump the prometheus group across 1 directory with 2 updates"
head_sha: 8bb5b9e0a9895cf3536576336b13627c4b5097df
base: main
reviewed_at: 2026-09-07T16:54:47Z
verdict: approve
---

## Verdict

approve — Dep-only update with no project-code change. Both releases are seven days old; their substantive upstream changes do not affect Prow's exercised Prometheus paths.

## What this PR does

- Updates `github.com/prometheus/client_model` from `v0.6.2` to `v0.6.3` in the root and tools modules.
- Updates `github.com/prometheus/common` from `v0.70.1` to `v0.71.0` in the root and tools modules.
- Refreshes the corresponding Go module checksums.

## Findings

### Blocking

None.

### Should fix

None.

### Nits

None.

### Questions

None.

## Checked

- Classification: only `go.mod` and `go.sum` files changed; no project code requires a separate code review.
- `client_model` is directly imported only by three test files; production reaches it through `client_golang`.
- `common` is used in `pkg/metrics/push.go` for protobuf metric encoding and label-name validation; its OpenMetrics 2.0 changes are outside this path.
- Module-proxy provenance: both new tagged releases resolve to 2026-08-31; neither is a pseudo-version.
- GitHub's published repository-advisory endpoints returned no entries for either upstream repository.
- `go mod verify` passed in the root and `hack/tools`; `git diff --check` passed.

## Open questions

None.
