---
pr: kubernetes-sigs/prow#1030
title: "chore(deps): bump sigs.k8s.io/controller-tools from 0.22.0 to 0.23.0 in /hack/tools in the kubernetes group across 1 directory"
head_sha: 654cd8d1c3d4917b095c69e3cd6aa66969d3d055
base: main
reviewed_at: 2026-10-27T23:32:15Z
verdict: approve
---
# Review

## Verdict

Approve. The tool release is nearly two weeks old, OSV reports no advisories in either version, and the regenerated CRD adds bounds already described by the schema. The bounds will now be enforced during Kubernetes admission.

## What this PR does

- Updates the direct tooling dependency `sigs.k8s.io/controller-tools` from `v0.22.0` to `v0.23.0` in `hack/tools/go.mod`.
- Regenerates the ProwJob CRD with controller-gen `v0.23.0`.
- Adds admission validation for `evictionResponders`: non-empty names, priorities from 0 through 100000, and at most 10 responders.

## Findings

No findings.

## Checked

- Classified as dep-only: the other changed file is generated CRD output; there are no project source-code changes.
- The only project import is the blank import of `sigs.k8s.io/controller-tools/cmd/controller-gen` in `hack/tools/tools.go`; `go mod why` confirms the tooling dependency path.
- The new release is tagged and dated 2026-10-14 by the Go module proxy; GitHub published it on 2026-10-15.
- OSV returned no advisories for `v0.22.0` or `v0.23.0`.
- `govulncheck` was attempted at base and head, but both scans stop while loading `hack/tools`: `tools.go` blank-imports the `golangci-lint` command, which is a program and not an importable package.
- The new CRD limits match the constraints already stated in the `evictionResponders` field descriptions.

## Open questions

None.

## Dependency followups

No opportunities identified for `sigs.k8s.io/controller-tools` `v0.22.0` → `v0.23.0`: the relevant schema-validation changes are already reflected in the regenerated CRD, while the alias/list-marker fix does not match Prow's API markers. The `hack/tools/go.sum` changes also move `github.com/onsi/ginkgo/v2` `v2.32.1` → `v2.33.0` and `github.com/onsi/gomega` `v1.43.0` → `v1.44.0`; they are controller-tools test dependencies, and Prow has no imports of either module.
