---
pr: kubernetes-sigs/prow#1027
title: "chore: update go to 1.27.3"
head_sha: 4b994a5c28187995853642362629d9ca23f85963
base: main
reviewed_at: 2026-10-22T13:03:21Z
verdict: request-changes
---
# Review

## Verdict

Request changes for one compatibility issue: the generated code now imports a standard-library package that is unavailable by default with Go 1.26, while the module still declares Go 1.26.4 as its minimum. The toolchain pin and PR CI use Go 1.27.3, but downstream builds that honor the module's declared minimum can fail.

## What this PR does

- Pins the repository's build toolchain to Go 1.27.3.
- Updates golangci-lint to v2.15.0 and refreshes its tool dependencies.
- Regenerates the config deepcopy code and applies Go 1.27 formatting.
- Updates expected gzip bytes in the updateconfig tests.

## Findings

### [should-fix] Declare the Go 1.27 minimum required by the generated import
- where: `pkg/config/zz_generated.deepcopy.go:25`
- concern: This generated import requires the Go 1.27 standard library behavior, but the root `go.mod` still declares `go 1.26.4`. Under Go 1.26.9, `encoding/json/jsontext` is excluded by default build constraints, so consumers using Go 1.26 cannot build the module despite its declared minimum; raise the module directive to reflect the new requirement.
- excerpt: |
    import (
    	jsontext "encoding/json/jsontext"

    	pipelinev1 "github.com/tektoncd/pipeline/pkg/apis/pipeline/v1"

## Checked

- `.go-version` is consumed by `hack/build/setup-go.sh`.
- The root `go.mod` directive remains `go 1.26.4`; only the tools module's dependency graph changes.
- The changed deepcopy import is not available by default in the local Go 1.26.9 toolchain (`go list encoding/json/jsontext` reports that build constraints exclude all files).
- PR CI reported unit, integration, image-build, race-detector, and lint jobs passing. No local tests were run.

## Open questions

- Could the root module's minimum Go version be raised to match the new generated standard-library import?

## Followups

Accepted: 2; skipped: 0.

### [deferred-review] Declare the Go 1.27 minimum required by the generated import

**Necessity:** should  
**Where:** `go.mod:3`, `pkg/config/zz_generated.deepcopy.go:25`

```text
Mode: deliver

In `kubernetes-sigs/prow`, following PR #1027 — “chore: update go to 1.27.3” (merge commit `3f5cc7178d5a85a64fcf32c9cab401c937ee7218`), update the root `go.mod` Go directive to the earliest Go release that supports the generated `encoding/json/jsontext` import in `pkg/config/zz_generated.deepcopy.go` by default.

Acceptance criteria:
- The root module's Go directive no longer claims Go 1.26 compatibility while this import is required.
- The config package builds with the repository-pinned Go toolchain.
- The change does not alter runtime behavior.

Scope guard: keep the change to Go-version metadata and required validation; do not modify runtime code or regenerate unrelated files.
```

### [test-maintenance] Assert gzip content instead of Go-version-specific bytes

**Necessity:** could  
**Where:** `pkg/plugins/updateconfig/updateconfig_test.go:892` and the other gzip expected ConfigMap cases

```text
Mode: deliver

In `kubernetes-sigs/prow`, following PR #1027 — “chore: update go to 1.27.3” (merge commit `3f5cc7178d5a85a64fcf32c9cab401c937ee7218`), revise the gzip expectations in `pkg/plugins/updateconfig/updateconfig_test.go` so they validate gzip semantics rather than exact compressed bytes.

Acceptance criteria:
- The existing gzip cases still verify ConfigMap keys, binary-versus-text storage, and decoded payload content.
- Assertions reject invalid gzip data and a valid gzip stream with the wrong payload.
- Expected results no longer depend on a particular Go compressor byte stream.

Scope guard: keep changes to the updateconfig test assertions; do not change production compression behavior or unrelated tests.
```
