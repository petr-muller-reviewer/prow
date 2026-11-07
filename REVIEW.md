---
pr: kubernetes-sigs/prow#1063
title: "chore(deps): bump golang.org/x/net from 0.61.0 to 0.62.0 in the golang-x group across 1 directory"
head_sha: 4d85ff64ce73a44fd76ec6f7243df23f0cca0ee7
base: main
reviewed_at: 2026-11-07T21:17:31Z
verdict: approve
---

# Review

## Verdict

Safe to bump now; the only reservation is an 11-day soak on a very recent `x/net` tag.

This is a dep-only Dependabot bump of `golang.org/x/net` from v0.61.0 to v0.62.0, touching only `go.mod`/`go.sum` in the main module and `hack/tools`. The new tag is a real release (not a pseudo-version), released 2026-10-27T19:00Z, 11 days before this review — past the "too fresh" threshold but worth noting. It carries two `html` CVE fixes that OSV reports for v0.61.0 and not for v0.62.0; `govulncheck` classifies both as imported-only for us, because the only `golang.org/x/net/html` call site uses the tokenizer, not the vulnerable parser/renderer symbols. No project code changes, no reachable-behavior changes in the packages we import.

## What this PR does

- Bumps `golang.org/x/net` v0.61.0 → v0.62.0 in the main module, where it is a direct dependency.
- Bumps the same module in `hack/tools/go.mod`, where it is indirect (tooling-only dependency graph).
- Updates the corresponding `go.sum` hashes in both modules; no other module versions change.
- No source, config, test, or vendor changes.

## Findings

None.

## Checked

- Full diff `9f4d1868e..4d85ff64c`: only `go.mod`, `go.sum`, `hack/tools/go.mod`, `hack/tools/go.sum`.
- Release provenance: real tag `refs/tags/v0.62.0` on `go.googlesource.com/net`, commit `3de643db506517b3e652a13cc0b56034cea6b8e6`; not a pseudo-version.
- OSV for v0.61.0: `GO-2026-6860` (CVE-2026-78661, quadratic text-node coalescing in `html`) and `GO-2026-6866` (CVE-2026-97045, missing ancestor namespace check when rendering raw text). OSV for v0.62.0: none.
- `govulncheck` module scan at base and head; symbol-level scan at base over `./pkg/...` marks both `x/net` advisories imported-only (package `golang.org/x/net/html` only, no called symbol). Head has no `x/net` findings, so no head symbol scan was needed.
- Import surface: 3 Go files outside `vendor/` — pkg/spyglass/lenses/html/html.go:30 (`html`, only `NewTokenizer` at :119) and pkg/githuboauth/githuboauth.go:33 (`xsrftoken.Generate` at :153, `Valid` at :240), plus one test file.
- No imports of `x/net/quic`, `x/net/ipv4`, or `x/net/ipv6`, which is where the rest of the changelog changes land.
- Changelog `v0.61.0...v0.62.0` (9 commits): the two `html` CVE fixes, a `quic` per-path connection-ID change, and `ipv4`/`ipv6` bounds/panic fixes.
- `go build ./pkg/spyglass/lenses/html/ ./pkg/githuboauth/` passes at head.
- Unrelated pre-existing imported-only advisories (`GO-2023-1901` tektoncd/pipeline, `GO-2026-5932` x/crypto) are identical at base and head and out of scope for this bump.

## Open questions

None.

## Dependency followups

No improvement opportunities identified. Examined `golang.org/x/net` v0.61.0 → v0.62.0 (direct in the main module, indirect in `hack/tools`); the 9 commits between the tags only fix vulnerabilities and panics in `html` parse/render, `quic`, `ipv4`, and `ipv6`. We import only `golang.org/x/net/html` (tokenizer API, unchanged) and `golang.org/x/net/xsrftoken` (untouched by this bump), and there are no deprecations, replacements, or new APIs in the release that would let us change our call sites.
