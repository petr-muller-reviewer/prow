---
pr: kubernetes-sigs/prow#786
title: "chore(deps): bump github.com/sigstore/rekor from 1.5.0 to 1.5.2 in /hack/tools"
head_sha: 8503ffff1e69ebf8e9641315e6b7d1b1af8571dd
base: main
reviewed_at: 2026-07-01T12:15:25Z
verdict: approve
---

## Summary

Dependabot bump of `github.com/sigstore/rekor` from v1.5.0 to v1.5.2 in the `hack/tools` build-tooling module, dragging in a broad sweep of transitive indirect deps across the sigstore/AWS/OpenAPI/gRPC/OTel cluster. Only `hack/tools/go.mod` and `hack/tools/go.sum` changed. Classification: dep-only.

## Findings

None.

## Checked

- Classification: dep-only — only `hack/tools/go.mod` and `hack/tools/go.sum` changed; no prow source touched.
- `github.com/sigstore/rekor` is `// indirect` in `hack/tools/go.mod`; absent from the main `go.mod`; zero imports in any prow Go source file. Pulled in transitively via `github.com/google/ko`.
- Freshness of rekor v1.5.2: released 2026-05-21 (~40 days ago). Fine.
- All other bumped modules are also `// indirect` in `hack/tools` only and absent from the main module. All releases are well-soaked (oldest: `k8s.io/klog/v2 v2.140.0` at ~120 days, `google.golang.org/grpc v1.80.0` at ~91 days).
- Rekor v1.5.1/v1.5.2 changelog: DSSE memory optimization, inclusion proof bug fixes, nil pointer fuzzing crash fix, Alpine decompression size limit, TLS ServerName fix in gRPC dial, input validation improvements, 500 error detail leakage fix. None of these code paths are exercised by prow — we never call rekor directly.
- No bumped dep ships in any prow binary; hack/tools is build tooling only.

## Open questions

None.
