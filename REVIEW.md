---
pr: kubernetes-sigs/prow#806
title: "chore(deps): bump golang.org/x/crypto from 0.50.0 to 0.52.0 in /hack/tools"
head_sha: 7fcb08791ceec267409b9c33369ae13a599da559
base: main
reviewed_at: 2026-07-27T08:48:24Z
verdict: approve
---

## Summary

Dependabot bump of `golang.org/x/crypto` v0.50.0 -> v0.52.0 in the `hack/tools` submodule (`hack/tools/go.mod`, `hack/tools/go.sum`). Sibling `golang.org/x/*` modules moved along with it as transitive deps: `x/net` v0.53.0->v0.54.0, `x/sys` v0.43.0->v0.45.0, `x/term` v0.42.0->v0.43.0, `x/text` v0.36.0->v0.37.0. Dep-only change, no project code touched.

## Findings

(none)

## Checked
- All changed modules are `// indirect` in `hack/tools/go.mod`; no direct import of `golang.org/x/crypto` (or siblings) anywhere in the repo outside vendor.
- `hack/tools` is a separate tooling-only Go module (golangci-lint, ko, controller-tools, code-generator, gotestsum) — not linked into any shipped prow binary. Main module's `go.mod` is untouched by this PR.
- New `x/crypto` v0.52.0 release is tagged (not a pseudo-version), dated 2026-05-22 — about 2 months old at review time, well past soak-time concerns.
- Commit range v0.50.0..v0.52.0 is routine upstream maintenance: NSS root-bundle (x509roots) refresh, `hkdf`/`pbkdf2` becoming wrappers around new stdlib `crypto/hkdf`/`crypto/pbkdf2`, ACME `OrderError` diagnostics improvement. No CVE/security advisory in this range.

## Open questions
(none)
