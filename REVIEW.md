---
pr: kubernetes-sigs/prow#1065
title: "chore(deps): bump github.com/go-jose/go-jose/v4 from 4.1.5 to 4.1.6"
head_sha: 87cc77c1ce00c190a536fb0a14221c2ac6599e63
base: main
reviewed_at: 2026-11-07T22:25:11Z
verdict: request-changes
---

# Review

## Verdict

The bump fixes one high-severity DoS but is stale: v4.1.6 already carries five advisories that v4.1.7 fixes, and v4.1.9 is the current release.

Dependabot moved `github.com/go-jose/go-jose/v4` from v4.1.5 to v4.1.6 (an indirect dependency, not imported by any prow package), which resolves `CVE-2026-116587` (high, DoS when parsing JWKs with RSA keys). However, v4.1.6 was published 2026-10-14, and the go-jose project shipped v4.1.7 on 2026-10-30 with fixes for five further advisories (all affecting `<= v4.1.6`), then v4.1.8 and v4.1.9. Landing v4.1.6 as the final state reintroduces known medium/low security debt immediately. Exposure is light in practice — go-jose is linked only through `cloud.google.com/go/storage` → gRPC xDS → `go-spiffe` JWT bundle parsing, a path prow never configures — so this is not urgent, but the PR should be rebased onto v4.1.7 (8 days soaked, all security fixes) rather than merged at v4.1.6.

## What this PR does

- Bumps `github.com/go-jose/go-jose/v4` from `v4.1.5` to `v4.1.6` in `go.mod:202` (still `// indirect`) and refreshes the two `go.sum` hash lines (`go.sum:223-224`).
- No vendored tree (repo has no `vendor/`), no project source, config, or test changes — dep-only PR.
- Origin of the dependency: `pkg/io` → `cloud.google.com/go/storage` → `google.golang.org/grpc/xds/googledirectpath` → `grpc/internal/xds/bootstrap` → `grpc/credentials/tls/certprovider` → `github.com/spiffe/go-spiffe/v2/bundle/spiffebundle` → go-jose.
- Upstream v4.1.6 content: bounds RSA modulus size at 4096 bits during JWK parsing (`MaxRSAModulusBits` escape hatch), rejects JWE CEKs whose length mismatches `enc`, fixes the error message for future-issued tokens.
- `go build ./pkg/io/...` passes at head.

## Findings

### [should-fix] Bump targets a version that is already superseded by security releases

- where: `go.mod:202`
- concern: v4.1.6 is affected by five advisories published 2026-10-30, all patched in v4.1.7: `GHSA-c67m-wr4h-2ccg` and `GHSA-v6h2-qj3f-2h6v` (medium, DoS), `GHSA-p9j4-2j95-j4rv` (medium, unprotected JWE headers overriding protected values), `GHSA-r39w-98wm-9pp6` and `GHSA-27pc-rm86-68pq` (low, panics). Since the PR is being reviewed 24 days after v4.1.6 and 8 days after v4.1.7, landing v4.1.6 only to immediately accept a follow-up bump adds avoidable vulnerable state.
- excerpt: |
    -	github.com/go-jose/go-jose/v4 v4.1.5 // indirect
    +	github.com/go-jose/go-jose/v4 v4.1.6 // indirect

### [nit] v4.1.6 raises a behavioral cap on RSA modulus size

- where: `go.mod:202`
- concern: the CVE fix rejects RSA JWKs with moduli above 4096 bits unless `MaxRSAModulusBits` is raised. Prow never parses JWKs itself, and the only path (`go-spiffe` JWT bundle handling behind gRPC xDS) is not configured here, so this cannot break prow today — worth knowing only if that path is ever enabled.
- excerpt: |
    The fix ... involves capping the size of the RSA modulus of keys accepted by JWK parsing.
    This limit is 4096 bits. If you need this to be higher, set `MaxRSAModulusBits` ...

### [question] Why not v4.1.9?

- where: `go.mod:202`
- concern: v4.1.9 (2026-11-05) carries all of the above fixes plus a regression fix for `OpaqueSigner` introduced in v4.1.5. It is only 2 days old, so a soak argument against it is reasonable; if that is the reason, the PR should still say which version it intends to converge on.

## Checked

- PR diff is dep-only: `go.mod` and `go.sum`, 3 changed lines, no `vendor/` directory in the repo.
- `go mod why -m github.com/go-jose/go-jose/v4` chain and `go list -deps ./pkg/io/...` confirm the dependency is linked into binaries but only via cloud storage's gRPC xDS → SPIFFE cert-provider path.
- No imports of `github.com/go-jose/...` and no `spiffe`/`xds` references in `pkg/` or `cmd/`; prow's own JWT use is `github.com/golang-jwt/jwt`, unrelated to this module.
- OSV: `v4.1.5` → `GHSA-mgcc-cww5-fp2q` / `CVE-2026-116587` (patched in 4.1.6); `v4.1.6` → no advisories indexed.
- `govulncheck -scan module` at merge-base and head (`8261d0d7a` vs `87cc77c1c`): no go-jose finding on either side, because the Go vulndb/OSV have not ingested the 2026-10-14 and 2026-10-30 go-jose advisories yet; findings came from `gh api repos/go-jose/go-jose/security-advisories` and the release notes instead. Symbol-level scan not run (no module-level finding to trace).
- `go build ./pkg/io/...` at head: success.
- Release provenance: real tag `refs/tags/v4.1.6`, not a pseudo-version; `go.sum` hashes match the proxy record.

## Open questions

- Can this be rebased onto v4.1.7 (or v4.1.9), so we do not land a version known to still carry five open advisories?
- If you deliberately want v4.1.6 first, is there an expected follow-up bump, and when?

## Dependency followups

No improvement opportunities identified.

- Examined: `github.com/go-jose/go-jose/v4` `v4.1.5` → `v4.1.6` (indirect) — the only module version change in the PR (`go.mod:202`, `go.sum:223-224`).
- Our usage: none. No file under `pkg/` or `cmd/` imports `github.com/go-jose/...`; the module is linked only through `cloud.google.com/go/storage` → gRPC xDS → `github.com/spiffe/go-spiffe/v2/bundle/spiffebundle`, a path prow never configures.
- Changelog between the two versions (RSA modulus cap during JWK parsing, rejection of mismatched JWE CEK lengths, error-message fix for future-issued tokens) only touches JWK/JWE/JWS parsing entry points we never call directly. There are no deprecations in this range (the `ParseEncrypted`/`ParseSigned` deprecations land in v4.1.7), and the new `MaxRSAModulusBits` knob is only meaningful to code that parses JWKs — none exists here.
- The version-selection concern (v4.1.6 is superseded by v4.1.7/v4.1.9 and retains five open advisories) is already recorded as the `[should-fix]` finding above and needs no separate followup.
