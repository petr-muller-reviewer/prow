---
pr: kubernetes-sigs/prow#1017
title: "chore(deps): bump github.com/tektoncd/pipeline from 1.16.0 to 1.17.0"
head_sha: bab4347ace9e20f1e65da4bacb5ee51ff6d4a168
base: main
reviewed_at: 2026-10-22T12:42:52Z
verdict: request-changes
refresh_log:
  - old_sha: 3841f98a9c56e604aa60dc9aeba675b5515532ad
    new_sha: bab4347ace9e20f1e65da4bacb5ee51ff6d4a168
    summary: "Refreshed against the current main base: Go is pinned to 1.27.3 and gzip expectations are updated, while Jira vet failures remain."
---

# Review

## Verdict

Request changes. This is a dep-only bump of `github.com/tektoncd/pipeline` from `v1.16.0` to `v1.17.0` and raises the module’s Go requirement to `1.27.0`. The current base pins Go `1.27.3`, resolving the toolchain security concern, but unit and race CI still fail because vet rejects the `*JiraError` operands to `%w`, and verify-lint fails its `go` check. Prow’s use of Tekton is mostly API types and generated client machinery, so the bump is not inherently unsafe once the test issue and failing checks are resolved.

Take: fix the Jira vet errors and get the required checks green before merging.

## What this PR does

- Bumps `github.com/tektoncd/pipeline` from `v1.16.0` to `v1.17.0`.
- Raises the Go directive from `1.26.4` to `1.27.0`.
- Updates the indirect dependency graph and `go.sum`; no project source changes.

Since previous review:
- The PR head changed from `3841f98a9c56e604aa60dc9aeba675b5515532ad` to `bab4347ace9e20f1e65da4bacb5ee51ff6d4a168` and now shares current `main` at `3f5cc7178d5a85a64fcf32c9cab401c937ee7218`; the PR diff remains limited to `go.mod` and `go.sum`.
- The base now pins Go `1.27.3` and updates gzip expectations. Unit and race CI still fail on the Jira `%w` vet errors; verify-lint also fails its `go` check.

## Findings

### [blocking] Jira vet errors still fail unit and race CI
- where: `go.mod:3`, `pkg/jira/jira_test.go:42`, `pkg/jira/jira_test.go:59`
- concern: The base now pins Go `1.27.3` and updates the gzip expectations, but the current unit and race jobs still fail because vet rejects the `*JiraError` operands to `%w` at lines 42 and 59. The verify-lint job also fails its `go` check. Resolve the Jira vet errors and get the required checks green before merging.
- excerpt: |
    go 1.27.0


## Resolved

### Go 1.27.0 toolchain security regression
- The updated base pins `.go-version` to Go `1.27.3`, and the current CI logs report Go `1.27.3`. This addresses the prior finding about the repo building with vulnerable Go `1.27.0`; the `go 1.27.0` directive remains the module minimum version.

## Checked

- Classification: dep-only. Only `go.mod` and `go.sum` change.
- Dependency: `github.com/tektoncd/pipeline` `v1.16.0 → v1.17.0`, direct.
- Freshness: `v1.17.0` was tagged on 2026-09-30 and released on 2026-10-01, about 22 days old.
- Usage: direct imports in 32 files, concentrated in `pkg/pipeline/**` generated clients, `pkg/config`, `pkg/apis/prowjobs/v1`, `cmd/pipeline`, and integration tests. The module is used for Tekton API types and generated client machinery, not the Tekton controller.
- Vulnerabilities: OSV reports the same Tekton advisory at both versions (`GO-2023-1901`, `CVE-2023-37264`); this bump fixes no Tekton vulnerability. `govulncheck` reaches it in both base and head.
- Changelog: `v1.17.0` adds `tt.params` as a valid variable-reference prefix, compiles the step-result regex once, adds `StripDescriptions` for TaskSpec and PipelineSpec status snapshots, and migrates the CEL import to `cel.dev/cel-go`. It also bumps Kubernetes and other dependencies and removes `gopkg.in/yaml.v3`.
- Exposure: heavy on API types but not controller logic; the substantive changes are mostly validation, status snapshots, and dependency churn, so the direct blast radius is low.
- CI: image build and integration pass; unit, race detector, and lint fail on the latest head.

## Open questions

- None.
