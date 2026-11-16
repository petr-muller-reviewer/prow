---
pr: kubernetes-sigs/prow#1074
title: "chore(deps): bump github.com/go-jose/go-jose/v4 from 4.1.4 to 4.1.6 in /hack/tools"
head_sha: 88d5d3ce2ea905d961bbae0305d1df170b4f8037
base: main
reviewed_at: 2026-11-13T23:26:11Z
verdict: needs-discussion
---

# Review

## Verdict

Safe and well-soaked bump, but it lands a version that already carries five known advisories — take `v4.1.9` instead of `v4.1.6`.

`v4.1.6` was cut 2026-10-14, so it has a month of soak and removes three real CVEs from the tools module (`GHSA-4w3g-jfj2-cw2q`, `GHSA-vvm8-gf78-28g4`, `GHSA-mgcc-cww5-fp2q`, the last rated high). Dependabot simply picked the latest version at the time. Since then `v4.1.7` fixed five further advisories that affect `<= 4.1.6`, `v4.1.8` added ML-DSA, and `v4.1.9` fixed an OpaqueSigner regression introduced in 4.1.5. Bumping to `v4.1.9` is the same one-line change in the same file and needs no code adaptation.

## What this PR does

- Bumps `github.com/go-jose/go-jose/v4` from `v4.1.4` to `v4.1.6` in the separate `/hack/tools` module; only `go.mod` and `go.sum` change.
- The module is a tool-only dependency: it reaches `hack/tools` through `google/ko` → `sigstore/cosign/v3` → `sigstore-go` → `sigstore/oauthflow`, i.e. ko's cosign/keyless sign path, and never into a shipped prow binary.
- It aligns the tools module with the root module, which already moved to `v4.1.6` in #1065 (the merge base of this PR).
- No project source, tests, or generated files are touched.

## Findings

### [should-fix] Target release 4.1.6 has five open advisories; 4.1.9 is current
- where: `hack/tools/go.mod:142`
- concern: `v4.1.6` is affected by `GHSA-c67m-wr4h-2ccg` (medium, DoS on JWS/JWE JSON serialization), `GHSA-v6h2-qj3f-2h6v` (medium, DoS verifying certificates), `GHSA-p9j4-2j95-j4rv` (medium, unprotected JWE headers overriding empty/null protected values), `GHSA-27pc-rm86-68pq` (low, panic on JWT claim matching an embedded unexported field) and `GHSA-r39w-98wm-9pp6` (low, panic in RSA JWK thumbprint). All five are published as `<= v4.1.6` in the repo advisory list and fixed in `v4.1.7`. `v4.1.9` (2026-11-05) contains those fixes plus a fix for the 4.1.5 OpaqueSigner regression, and has no open advisories in OSV.
- excerpt: |
    github.com/go-jose/go-jose/v4 v4.1.6 // indirect

### [question] Root module stays on 4.1.6 as well
- where: `go.mod:202`
- concern: The root module is already on `v4.1.6` from #1065 and picks go-jose up at runtime via `pkg/io` → `cloud.google.com/go/storage` → gRPC xDS → `spiffe`, so it shares the five open advisories above. If this PR moves to `v4.1.9`, the root module should follow in the same or an immediate follow-up change.
- excerpt: |
    github.com/go-jose/go-jose/v4 v4.1.6 // indirect

## Checked

- Freshness: `v4.1.6` tagged 2026-10-14 22:46 UTC (~30 days); real tag, not a pseudo-version. `v4.1.9` is 8 days old.
- OSV at `v4.1.4`: three advisories, all fixed by this bump; none at `v4.1.6`.
- `govulncheck` v1.9.0 module scans of `hack/tools` at merge-base (4.1.4) and head (4.1.6): no findings either side — low signal here, because vuln.go.dev's newest go-jose/v4 entry is `GO-2026-4945` (fixed 4.1.4) and the September/October advisories are absent from that DB. Symbol-level scan skipped: `tools.go:25` imports `cmd/golangci-lint`, a program, so source-mode loading fails, and a module scan with no findings cannot have reachable ones.
- Usage: no direct imports anywhere in the repo; `go mod why -m` gives `hack/tools` → `google/ko` → `cosign/v3/pkg/oci` → `sigstore-go/pkg/sign` → `sigstore/pkg/oauthflow` → go-jose. Build-time only, exercised by `hack/tools/prowimagebuilder/main.go:74` when building `_bin/ko`.
- v4.1.6's RSA modulus cap (4096 bits by default, `MaxRSAModulusBits`) and v4.1.5's un-exported `jose.RandReader` do not affect cosign keyless flows; no build breakage.
- `go build -o /tmp/ko-check github.com/google/ko` succeeds; `go mod tidy -diff` is empty; `go mod verify` passes.

## Open questions

- Since 4.1.9 is out and the tools module needs no adaptation for it, could we bump straight to `v4.1.9` (and do the same for the root module) rather than landing 4.1.6 and following up? `v4.1.6` is now a known-vulnerable version.
- Is anything in this module pinning 4.1.6 deliberately, e.g. a `go mod tidy` constraint from `ko` or `cosign`? `go mod tidy` is clean at 4.1.6, so at least nothing conflicts today.

## Dependency followups

No improvement opportunities identified.

- Modules examined: `github.com/go-jose/go-jose/v4` `v4.1.4 → v4.1.6` (indirect, `/hack/tools`), including the intermediate `v4.1.5` release notes.
- Reason: Prow imports go-jose nowhere in its own code (grep for `go-jose` and `jose.` over non-vendor Go files returns nothing); it reaches the tools module only via `ko → cosign/v3 → sigstore-go → sigstore/oauthflow`. The 4.1.5/4.1.6 deprecations and breaking items (`jose.RandReader` un-exported, `OpaqueSigner.Public()` checks, 4096-bit RSA modulus cap) have no call site of ours to migrate, and nothing in the dependency graph implements `jose.OpaqueSigner` (grep of the sigstore/ko module cache finds none), so the 4.1.5 OpaqueSigner regression fixed in 4.1.9 does not reach us.
- The remaining action — retargeting to `v4.1.9` for the 2026-10-30 advisories — is already recorded under Findings above; it is not a changelog-unlocked code change.
