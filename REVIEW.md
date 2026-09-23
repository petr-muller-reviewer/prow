---
pr: kubernetes-sigs/prow#963
title: "chore(deps): bump google.golang.org/grpc from 1.83.1 to 1.83.2 in /hack/tools"
head_sha: 02dea49858aba338a3008af775b697b1a7dd34a9
base: main
reviewed_at: 2026-09-23T12:19:40Z
verdict: approve
---

## Verdict

approve — This is a scoped, 29-day-old indirect dependency update for the developer-tools module. grpc-go v1.83.2 hardens server handling of malformed HTTP/2 authority headers; Prow's `hack/tools` reaches grpc through controller-gen and does not directly import or run a gRPC server.

## What this PR does

- Updates `google.golang.org/grpc` in the standalone `hack/tools` Go module.
- Moves the selected version from `v1.83.1` to `v1.83.2`.
- Updates only the corresponding module checksums.
- Does not alter Prow production code, tests, configuration, or the root Go module.

## Findings

None.

## Checked

- `hack/tools/go.mod:351` marks the bump indirect; `go mod why` resolves it through `controller-gen`, Kubernetes API-extension validation, and apiserver egress-selector code.
- `hack/tools` has no direct Go import of `google.golang.org/grpc`; the PR changes only `hack/tools/go.mod` and `hack/tools/go.sum`.
- grpc-go v1.83.2 was released 2026-08-25 (29 days before review), is a tagged release, and its relevant change rejects server requests missing both `:authority` and `Host` headers.
- The remaining release change adjusts grpc-go's own OpenTelemetry tests for Go 1.27 gzip output; no applicable API or behavior change was found for the tools path.
- `cd hack/tools && go mod verify` passed: `all modules verified`.

## Open questions

None.
