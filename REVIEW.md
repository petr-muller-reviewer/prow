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

## Dependency followups

Examined on 2026-10-18: no concrete opportunities identified; no items or handoff prompts. PR #917 merged on 2026-09-07 as `bf6cebaa027a194bc2b1bb811bf59367075a1bfb`.

- `github.com/prometheus/common` `v0.70.1 → v0.71.0` (direct in root, indirect in tools): reviewed the complete upstream commit range and relevant implementation changes. The new `NegotiateAccept` API replaces deprecated `Negotiate` and `NegotiateIncludingOpenMetrics`, but Prow calls neither. OpenMetrics 2.0 support, duration constants/`Milliseconds`, label-signature optimizations, and the no-op logger improvement have no matching Prow call sites. Prow's sole importing production file uses label validation at `pkg/metrics/push.go:52` and explicitly selects protobuf encoding at `pkg/metrics/push.go:67`; the bump supplies no useful replacement for those calls. Source: [upstream comparison](https://github.com/prometheus/common/compare/v0.70.1...v0.71.0).
- `github.com/prometheus/client_model` `v0.6.2 → v0.6.3` (direct in root, indirect in tools): reviewed the complete upstream range. Native-histogram stabilization changes a proto comment; there is no generated Go API change to adopt. Prow's three direct imports are DTO assertions in `cmd/exporter/collector_test.go:29`, `pkg/metrics/prowjobs/collector_test.go:27`, and `pkg/plugins/updateconfig/updateconfig_test.go:28`. Source: [upstream comparison](https://github.com/prometheus/client_model/compare/v0.6.2...v0.6.3).
- Transitive dependencies: the PR changes no other selected module versions. Upstream increases protobuf, `x/net`, `x/text`, and testify requirements and switches its YAML v3 dependency, but Prow's root and tools modules already selected protobuf `v1.36.12`, `x/net v0.58.0`, `x/text v0.41.0`, testify `v1.12.1`, and `go.yaml.in/yaml/v3 v3.0.5` before this PR. These upstream requirement changes unlock no additional version range in Prow.
- Range selection: the current tracking branch already contains the merged PR, so its merge-base with the review branch is the PR head. Used the original dependency commit's parent (`8bb5b9e0a^`) through PR head `8bb5b9e0a9895cf3536576336b13627c4b5097df` to examine the actual bump.
