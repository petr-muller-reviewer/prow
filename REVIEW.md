---
pr: kubernetes-sigs/prow#928
title: "chore(deps): bump google.golang.org/grpc from 1.82.1 to 1.83.1 in /hack/tools"
head_sha: c0eaaba6e7a9b5eb509f31376fd9f78141857f82
base: main
reviewed_at: 2026-09-08T20:42:11Z
verdict: approve
---

## Verdict

Approve. This is a dependency-only, indirect gRPC update in the isolated `hack/tools` module. v1.83.1 had about 20 days of soak time and its relevant upstream changes do not reach this module's usage.

## What this PR does

- Updates indirect `google.golang.org/grpc` from `v1.82.1` to `v1.83.1` in `hack/tools/go.mod`.
- Refreshes the corresponding gRPC checksums in `hack/tools/go.sum`.
- Updates transitive `cel.dev/expr` checksums to `v0.25.2`, as selected by the updated gRPC module requirements.
- Leaves Prow source code, the root module, and the tools module's direct requirements unchanged.

## Findings

No findings.

## Checked

- Classified the two-file change (`hack/tools/go.mod`, `hack/tools/go.sum`) as dependency-only.
- Confirmed `google.golang.org/grpc v1.83.1` is indirect at `hack/tools/go.mod:351` and is reached through `controller-gen` / Kubernetes API-server validation dependencies; no Go source in `hack/tools` imports gRPC.
- Reviewed grpc-go `v1.83.0` and `v1.83.1` release notes: HTTP/2 control-frame flood limiting, xDS/RBAC fail-open fixes, small-frame buffering bounds, and opt-in xDS/auth additions. This tools module has no xDS use.
- Confirmed v1.83.1 is a tagged release published 2026-08-19, approximately 20 days before review.
- Ran `cd hack/tools && go mod verify` successfully.

## Open questions

None.

## Dependency followups

No opportunities identified — nothing in this bump deprecates, replaces, or newly enables anything our code calls.

- `google.golang.org/grpc` v1.82.1 → v1.83.1 (indirect in `hack/tools`; v1.83.0/v1.83.1 release notes and the full 28-commit range checked for added `// Deprecated:` markers, found none): the changes are xDS/RBAC fixes and xDS-only features (we use no xDS), plus an automatic server-side HTTP/2 control-frame flood limit and a small-frame buffering bound that need no call-site change. Nothing in `hack/tools` imports gRPC.
- `cel.dev/expr` v0.25.1 → v0.25.2 (checksum-only, selected transitively by gRPC): upstream `google/cel-spec` changes are conformance tests and language-definition docs with no Go API change, and no package in the repo imports it.
- Main-module call sites (`cmd/gangway`, `pkg/gangway`, `pkg/resultstore`, `pkg/crier/reporters/pubsub`, `test/integration/...`) build against the root module's own `google.golang.org/grpc v1.83.2`, so this `hack/tools`-only bump neither enables nor requires any change there.
