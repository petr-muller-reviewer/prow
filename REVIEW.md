---
pr: kubernetes-sigs/prow#1032
title: "chore(deps): bump google.golang.org/api from 0.300.0 to 0.305.0"
head_sha: 81f2bf15be8282462c1569ecbee07d685d9544c3
base: main
reviewed_at: 2026-10-27T20:43:30Z
verdict: approve
---

# Review

## Verdict

Approve: dependency-only bump with no known CVEs, no reachable vulnerabilities at either revision, and negligible behavioral surface in the parts of `google.golang.org/api` that prow actually imports.

v0.305.0 was released 2026-10-19, 8 days before this review — enough soak to be comfortable. Nearly all of the 0.301.0–0.305.0 changelog is discovery-client regeneration (generated per-API surface we do not import); the single feature, `option/internaloption.EnableDirectPathXdsOverInterconnect`, is internal-only and unused here. The indirect modules that move (s2a-go, enterprise-certificate-proxy, x/mod, genproto/googleapis/rpc) only activate through credential-file transport options prow never sets. The one sensitivity-adjacent piece is `cloud.google.com/go/auth`, which sits on the operator-supplied credentials path; its change since v0.24.0 is the same DirectPath-over-Interconnect plumbing, defaulted off. The only thing worth remarking on is that the `auth` requirement lands on a pre-release commit rather than the tagged v0.24.1.

## What this PR does

- Bumps `google.golang.org/api` 0.300.0 → 0.305.0 in `go.mod`.
- Carries `cloud.google.com/go/auth` along, because api v0.305.0's own `go.mod` requires `v0.24.1-0.20261001053825-dbc26066f70a`.
- Drags four indirect modules forward: `google/s2a-go` v0.1.11, `enterprise-certificate-proxy` v0.3.23, `golang.org/x/mod` v0.42.0, `genproto/googleapis/rpc` (2026-10-05 snapshot).
- Changes no project source; `go.sum` churn only.

## Findings

### [nit] Direct dependency pinned to a pre-release commit while the tagged release exists
- where: `go.mod:6`
- excerpt: |
    cloud.google.com/go/auth v0.24.1-0.20261001053825-dbc26066f70a
- concern: `auth` is imported directly (`pkg/io/credentials.go:38`, `credentials.DetectDefault`), yet lands on an untagged 2026-10-01 commit. Tagged `v0.24.1` was cut 2026-10-08 and is a strict superset: it adds `fix(auth): return empty tls.Certificate on secureconnect helper failure` (#20632) on top of the same DirectPath commit. MVS picks v0.24.1 over the pseudo-version, so raising it in this PR is free and gets an auth-path dependency off a pseudo-version.

## Checked

- OSV: no advisories for `google.golang.org/api` 0.300.0/0.305.0, `cloud.google.com/go/auth` 0.24.0/pinned, `google/s2a-go` 0.1.10/0.1.11, `enterprise-certificate-proxy` 0.3.22/0.3.23.
- `govulncheck ./pkg/io/...` (the packages importing both bumped direct deps) at merge-base `6c4d88e3b` and at head: zero findings at both. `./...` OOM-killed on this 32 GB box at both revisions, so the comparison is scoped rather than repo-wide, but symmetric.
- The local toolchain needs `GOEXPERIMENT=jsonv2` for the repo's Go 1.27-regenerated deepcopy code (`encoding/json/jsontext`); with it, the affected packages type-check at head.
- Import surface is small: 9 files use `google.golang.org/api`, and only through `option`, `iterator`, `googleapi` (`pkg/io/credentials.go`, `pkg/io/opener.go:550`, `pkg/io/option.go:63`, `pkg/io/iterator.go:58`, `pkg/googlecloudbuild/client/client.go:138`, `cmd/webhook-server/secretmanager/secretmanager.go:140`, plus tests). `cloud.google.com/go/auth` is a single call site in `pkg/io/credentials.go`.
- Changelog of v0.301.0–v0.305.0 (5 releases, 26 commits): discovery-client regeneration, one `internaloption` addition, and generator-pipeline changes that do not ship in the library. None of the packages prow imports changed behavior.
- auth diff v0.24.0 → pinned commit: `feat(auth/grpctransport): support DirectPath over Interconnect` plus gax/otel bumps; unreachable without `internaloption.EnableDirectPathXdsOverInterconnect`, which prow does not set.
- s2a-go v0.1.11 only changes ALPN `NextProtos` selection; ECP v0.3.23 is a version/toolchain bump. Both require an `external_account` credential configuration to activate.

## Open questions

- Would you rather refresh the `cloud.google.com/go/auth` requirement to the tagged `v0.24.1` in this PR (`go get cloud.google.com/go/auth@v0.24.1 && go mod tidy`)? It is strictly newer than what api v0.305.0 requires and carries the secureconnect `tls.Certificate` fix we would otherwise skip.

## Dependency followups

One accepted item. Nothing else in the same version ranges offered an adoptable change (see "Examined" below).

### Followup 1: migrate off deprecated `credentials.DetectOptions.CredentialsFile`
- category: deprecation
- where: `pkg/io/credentials.go:33-45` (callers: `pkg/io/opener.go:144`, `pkg/googlecloudbuild/client/client.go:96`)
- module: `cloud.google.com/go/auth` `v0.24.1-0.20261001053825-dbc26066f70a` — the deprecation predates this bump (it is present in v0.24.0 too); recorded here because the bump re-confirmed it is the only deprecated Google-auth call left in the tree.
- necessity: should — upstream deprecates the field for a security reason (unvalidated credential type) and names the replacement; no removal date is published, so it is not urgent, but it is a suppressed `staticcheck` finding rather than a solved problem.
- what the bump offers: `// Deprecated: This field is deprecated because of a potential security risk. It does not validate the credential configuration. ... If you know that you will be loading credential configurations of a specific type, it is recommended to use a credential-type-specific NewCredentialsFromFile method.`
- handoff prompt:

```
In kubernetes-sigs/prow, migrate pkg/io/credentials.go off the deprecated
cloud.google.com/go/auth credentials.DetectOptions.CredentialsFile field, which
upstream deprecated in favour of the credential-type-specific loaders because an
unvalidated configuration can load an unexpected credential type.

Current code (pkg/io/credentials.go:31-46) does:

    creds, err := credentials.DetectDefault(&credentials.DetectOptions{
        CredentialsFile: file,
        Scopes:          scopes,
    })
    if err != nil {
        return nil, fmt.Errorf("load Google credentials from %q: %w", file, err)
    }
    return option.WithAuthCredentials(creds), nil

wrapped in a //nolint:staticcheck whose rationale is "callers legitimately pass
any credential type, and the file is fed by an operator-supplied flag".

Task: keep the "any credential type" contract but go through the typed loader.
Read the file, unmarshal just the type discriminator
(struct{ Type string `json:"type"` }), map it to the matching
credentials.CredType constant (credentials.ServiceAccount,
credentials.AuthorizedUser, credentials.ExternalAccount,
credentials.ImpersonatedServiceAccount, credentials.GDCHServiceAccount,
credentials.ExternalAccountAuthorizedUser), then call
credentials.NewCredentialsFromFile(credType, file, &credentials.DetectOptions{Scopes: scopes}).
An empty, unknown or unsupported type must return an error wrapped the same way
("load Google credentials from %q: ..."), naming the type found.

Acceptance criteria:
- GoogleCredentialsFileOption's signature and semantics are unchanged; its two
  callers (pkg/io/opener.go:144, pkg/googlecloudbuild/client/client.go:96) need
  no edits.
- The //nolint:staticcheck directive and its explanatory comment are gone;
  `staticcheck`/`golangci-lint` reports no deprecation warning for this file.
- `go build ./...` and `go test ./pkg/io/... ./pkg/googlecloudbuild/...` pass
  (on this repo the Go toolchain needs GOEXPERIMENT=jsonv2 for the regenerated
  deepcopy code).
- `grep -rn CredentialsFile --include='*.go' . | grep -v vendor` finds no
  remaining use in non-test code.
- pkg/io/credentials_test.go is extended with a case for an unrecognised
  credential type expecting an error.

Scope guard: do not change the --google-credentials flag, the GCS/Cloud Build
call sites, or any other Google-auth behaviour; do not start validating
credential file contents beyond the type discriminator; do not touch other
deprecated APIs elsewhere in the tree; no new dependencies.
```

### Examined (no opportunity)

- `google.golang.org/api` v0.300.0 → v0.305.0 (direct): the only new code in the packages we import is `option/internaloption.EnableDirectPathXdsOverInterconnect` (EXPERIMENTAL, documented as "should only be used internally by generated clients"); `option`, `iterator` and `googleapi` are otherwise identical between the two versions. The rest of the range is discovery-client regeneration for APIs we do not import.
- `cloud.google.com/go/auth` v0.24.0 → `v0.24.1-0.20261001053825-dbc26066f70a` (direct): the `credentials` package is identical; all changes sit in `grpctransport/` and `internal/transport/{cba,s2a}.go` as DirectPath-over-Interconnect plumbing behind an option and the `GOOGLE_CLOUD_ENABLE_DIRECT_PATH_XDS_OVER_INTERCONNECT` env var, neither of which prow sets.
- `github.com/google/s2a-go` v0.1.10 → v0.1.11, `github.com/googleapis/enterprise-certificate-proxy` v0.3.22 → v0.3.23, `golang.org/x/mod` v0.41.0 → v0.42.0, `genproto/googleapis/rpc` 2026-09-21 → 2026-10-05: transitive only — prow imports none of them, so there is no call site of ours that could adopt anything they add.
