---
pr: kubernetes-sigs/prow#905
title: "chore(deps): bump github.com/golangci/golangci-lint/v2 from 2.12.2 to 2.13.1 in /hack/tools"
head_sha: 72c049d2b66a42a7a998e07246cd38bce3e9c754
base: main
reviewed_at: 2026-08-30T17:39:07Z
verdict: approve
---

## What this PR does

- Bumps `github.com/golangci/golangci-lint/v2` from `v2.12.2` to `v2.13.1` in `hack/tools/go.mod`/`go.sum` (direct dependency of the separate build-tooling module, plus the usual transitive churn of golangci-lint's own dependencies).
- Narrows the `staticcheck` exclude regex in `.golangci.yml:41` from a message-specific match to a looser `SA1019: .*TrustedOrg is deprecated` pattern, to keep matching after upstream reworded the deprecation string.
- Migrates `cmd/deck/main.go:1364-1370` from the deprecated `httputil.ReverseProxy.Director` field to `.Rewrite`, newly flagged by the updated linter set.
- Adds a documented `//nolint:staticcheck` on `pkg/io/credentials.go:34` for `CredentialsFile` (SA1019), also newly surfaced by the bump.

## Findings

None.

## Checked

- Freshness: `v2.13.1` tagged 2026-08-20 (verified via `proxy.golang.org` and `gh release view`) — 10 days old, real tag (not a pseudo-version). Acceptable soak time for a dev-only tool.
- Usage/exposure: `golangci-lint/v2` is a direct dependency of `hack/tools/go.mod`, referenced only via the tool-pinning pattern in `hack/tools/tools.go` (`grep -rln "github.com/golangci/golangci-lint" --include='*.go' .` → only that file). Build/CI tooling only, not linked into any shipped binary — no production exposure.
- Changelog (`v2.12.2`→`v2.13.0`→`v2.13.1`): almost entirely bot bumps of golangci-lint's own internal linter deps, plus minor fixes (package-facts caching, `funcorder`/`gomoddirectives` field additions, go1.27 support, a new `goconst` option). No CVEs, no security-relevant changes.
- `cmd/deck/main.go` `Director`→`Rewrite` migration: `grep -rn "Director:" --include='*.go' .` returns no remaining hits; the new code sets `pr.Out.URL`/`.ContentLength`/`.Body`, which is the doc-recommended replacement and semantically equivalent to the old full-replacement `Director` body.
- `pkg/io/credentials.go` nolint: justification in the comment (operator-supplied flag, not untrusted input; no single replacement API covers all credential types accepted here) is sound, not a blind suppression.
- `.golangci.yml` regex narrowing still scopes only to `TrustedOrg`-related `SA1019` findings — no risk of over-broadly suppressing unrelated staticcheck hits.

## Open questions

None.
