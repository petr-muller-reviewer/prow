---
pr: kubernetes-sigs/prow#1016
title: "chore(deps): bump the golang-x group across 1 directory with 2 updates"
head_sha: c56126d16698930e8270367dd13f41a3f0c9f3fe
base: main
reviewed_at: 2026-10-22T01:02:38Z
verdict: approve
---

# Review

## Verdict

Approve. This is a dep-only bump to the current latest tagged `golang.org/x/net v0.61.0` and `golang.org/x/text v0.43.0`, both released on 2026-10-09. The `x/net` update fixes five known HTTP/2 advisories that are reachable in the current build, while the `x/text` update is a small, unrelated performance fix with no advisory exposure.

## What this PR does

- Bumps `golang.org/x/net` from `v0.59.0` to `v0.61.0` in the root and `hack/tools` modules.
- Bumps `golang.org/x/text` from `v0.42.0` to `v0.43.0` in the root and `hack/tools` modules.
- Moves the required companion versions of `golang.org/x/crypto`, `golang.org/x/sys`, `golang.org/x/term`, and `golang.org/x/tools` forward as demanded by `x/net v0.61.0`.
- Changes only `go.mod`, `go.sum`, `hack/tools/go.mod`, and `hack/tools/go.sum`; no project source is touched.

## Findings

None.

## Checked

- **Classification:** dep-only. Every changed file is a Go module manifest or lockfile.
- **Freshness:** `x/net v0.61.0` and `x/text v0.43.0` are the latest tags on the Go module proxy, both tagged 2026-10-09; they are 13 days old as of this review.
- **`golang.org/x/net` usage:** Direct in the root module. Our own imports are `pkg/githuboauth/githuboauth.go:33` (`xsrftoken`) and `pkg/spyglass/lenses/html/html.go:27` (`html`). The upstream changes are concentrated in `http2` and `internal/http3`, not in those directly imported subpackages.
- **`golang.org/x/net` vulnerabilities:** `v0.59.0` is affected by five HTTP/2 advisories—CVE-2026-78659, CVE-2026-78660, CVE-2026-78669, CVE-2026-78663, and CVE-2026-97032—all fixed in `v0.60.0` and therefore included in `v0.61.0`. At the merge base, `govulncheck` found reachable traces from `pkg/spyglass/lenses/buildlog/lens.go:310`, `pkg/spyglass/lenses/common/common.go:320`, `pkg/spyglass/lenses/fake/fake.go:65`, and `pkg/spyglass/lenses/links/links.go:176`; at head, those findings are gone.
- **`golang.org/x/text` usage:** Direct in the root module, imported only in `pkg/plugins/golint/suggestion/golint_suggestion.go:26` and `pkg/spyglass/lenses/links/links.go:31` for `cases` and `language`. No OSV advisories affect either version. The changes fix quadratic behavior in `unicode/bidi` and make `secure/bidirule` lazy; neither affects the subpackages we import.
- **Companion dependencies:** `x/crypto`, `x/sys`, `x/term`, and `x/tools` move only as far as `x/net v0.61.0` requires. `x/crypto` still carries the pre-existing `GO-2026-5932` advisory for the unmaintained `openpgp` package, but it is imported-only and its reachability is unchanged.
- **Validation:** `go mod tidy` is clean in both the root and `hack/tools` modules. Targeted tests pass for `pkg/githuboauth`, `pkg/spyglass/lenses/html`, `pkg/spyglass/lenses/links`, and `pkg/plugins/golint/suggestion`.

## Open questions

None.

## Dependency followups

No opportunities were identified.

Modules and ranges examined:

- `golang.org/x/net v0.59.0 → v0.61.0`
- `golang.org/x/text v0.42.0 → v0.43.0`
- `golang.org/x/crypto v0.57.0 → v0.58.0`
- `golang.org/x/sys v0.48.0 → v0.49.0`
- `golang.org/x/term v0.46.0 → v0.47.0`
- `golang.org/x/tools v0.49.0 → v0.51.0`

Reason: the changelogs are confined to HTTP/2, HTTP/3, QUIC, bidi handling, or tooling packages that this repository does not call directly, and no new API or deprecation affects the subpackages we actually import.
