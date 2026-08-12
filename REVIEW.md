---
pr: kubernetes-sigs/prow#837
title: "chore(deps): bump github.com/sigstore/sigstore-go from 1.1.4 to 1.2.1 in /hack/tools"
head_sha: 53ea75efdb102675f98ab9e85df0716577d3802c
base: main
reviewed_at: 2026-08-12T10:06:24Z
verdict: approve
---

## What this PR does

- Dependabot bump of `github.com/sigstore/sigstore-go` `v1.1.4 -> v1.2.1` in `hack/tools/go.mod`, an indirect dependency pulled in transitively (via `google/ko`).
- `go mod tidy` churn on transitive deps as a side effect: docker/cli, go-openapi/*, grpc-gateway/v2, klog, otel, in-toto/attestation, sigstore/{rekor,sigstore,timestamp-authority}, theupdateframework/go-tuf/v2, etc.
- Only `hack/tools/go.mod` and `hack/tools/go.sum` change. No project source touched.
- New version fixes CVE-2026-54787 / GHSA-wqqc-jjcq-vfxm (signature-timestamp-vs-key-validity-window check for self-managed long-lived keys).

## Dependency analysis

- Freshness: v1.2.1 tagged 2026-06-09, ~2 months old at review time. Fine, no soak-time concern.
- Usage: indirect only, not imported by any of this repo's own code (`hack/tools/tools.go` only blank-imports golangci-lint, k8s code-generators, protoc-gen-go, gotestsum, misspell, google/ko). Reaches sigstore-go transitively through `google/ko`'s signing support; Prow's own binaries never link against it.
- Changelog/exposure: v1.2.1 is a security fix for the long-lived-key verification path (not the standard Fulcio-cert path); v1.2.0 bundled unrelated Rekor v2 / TUF changes. None of it lands on code this repo exercises, since sigstore-go isn't called directly here. Exposure is minimal — build-tooling transitive dep only.

## Findings

(none — dep-only PR, no project code to review)

## Checked
- Diff scope: confirmed only `hack/tools/go.mod`/`go.sum` changed via `git diff --stat`.
- Confirmed `sigstore-go` is indirect and not imported by any file in this repo outside `hack/tools/go.sum`.
- Confirmed CVE-2026-54787 fixed in v1.2.1 doesn't affect standard cert-based verification, and this repo doesn't use the affected long-lived-key workflow at all.

## Open questions
(none)
