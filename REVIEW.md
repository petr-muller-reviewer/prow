---
pr: kubernetes-sigs/prow#1017
title: "chore(deps): bump github.com/tektoncd/pipeline from 1.16.0 to 1.17.0"
head_sha: 3841f98a9c56e604aa60dc9aeba675b5515532ad
base: main
reviewed_at: 2026-10-22T00:53:14Z
verdict: request-changes
---

# Review

## Verdict

Request changes. This is a dep-only bump of `github.com/tektoncd/pipeline` from `v1.16.0` to `v1.17.0`, but it also raises the module’s Go requirement to `1.27.0` while the repo is still pinned to Go `1.26.0`. That toolchain move breaks unit, race, and lint CI and introduces reachable Go `1.27.0` standard-library vulnerabilities. The dependency itself is reasonably fresh and Prow’s use is mostly API types and generated client machinery, so the bump is not inherently unsafe once the toolchain and tests are updated.

Take: wait until Go is moved to at least `1.27.3` and the two test failures are fixed.

## What this PR does

- Bumps `github.com/tektoncd/pipeline` from `v1.16.0` to `v1.17.0`.
- Raises the Go directive from `1.26.4` to `1.27.0`.
- Updates the indirect dependency graph and `go.sum`; no project source changes.

## Findings

### [blocking] Go 1.27.0 breaks the repo’s current test suite
- where: `go.mod:3`, `.go-version:1`
- concern: The module now requires Go `1.27.0`, but the repo still pins Go `1.26.0`. Under Go `1.27.0`, `pkg/jira/jira_test.go:42` and `pkg/jira/jira_test.go:59` fail vet because `%w` now rejects a `*JiraError` operand, and `pkg/plugins/updateconfig/updateconfig_test.go:1616` fails because gzip output differs. Unit, race, and lint CI all fail on this head.
- excerpt: |
    go 1.27.0

### [blocking] Go 1.27.0 is a security regression at this pin
- where: `go.mod:3`
- concern: `govulncheck` reports 15 reachable findings at head versus 6 at base. The new findings are Go `1.27.0` standard-library issues fixed in Go `1.27.2` or `1.27.3`. Taking this bump as-is therefore moves the build onto an older, vulnerable toolchain. Update to at least Go `1.27.3` before merging, or keep Go `1.26.x` until the repo is ready.
- excerpt: |
    go 1.27.0

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
