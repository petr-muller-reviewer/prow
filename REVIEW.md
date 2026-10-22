---
pr: kubernetes-sigs/prow#901
title: "chore(deps): bump google.golang.org/api from 0.287.1 to 0.293.0"
head_sha: f048a8797315044122171dd6551990e7e1d2d12a
base: main
reviewed_at: 2026-09-01T00:30:25Z
verdict: approve
---

## What this PR does

- Dependabot bump of `google.golang.org/api` from v0.287.1 to v0.293.0 (direct dependency).
- Transitively bumps `github.com/googleapis/enterprise-certificate-proxy` v0.3.17 → v0.3.20 (indirect).
- Transitively bumps `google.golang.org/genproto/googleapis/rpc` pseudo-version (2026-06-30 hash → 2026-08-07 hash, indirect).
- Only `go.mod`/`go.sum` change (9 lines total); no project source touched. No vendoring in this repo.
- PR was already merged 2026-08-26 (this review happened after the fact, on 2026-09-01).

## Dependency analysis

### google.golang.org/api (direct, v0.287.1 → v0.293.0)

- Freshness: new version released 2026-08-11 (`proxy.golang.org` info). PR merged 15 days later (2026-08-26); today (2026-09-01) it's ~3 weeks old. Fine, no freshness concern.
- Usage: direct import, but only via `option`, `googleapi`, and `iterator` subpackages. Import sites: `pkg/io/credentials.go`, `pkg/io/option.go`, `pkg/io/opener.go`, `pkg/googlecloudbuild/client/client.go`, `cmd/webhook-server/secretmanager/secretmanager.go`, plus `pkg/io/opener_test.go` and `pkg/crier/reporters/pubsub/reporter_test.go`. This is GCS / Cloud Build / Secret Manager client auth-option plumbing, not raw transport code.
- Changelog & exposure: compare range googleapis/google-api-go-client v0.287.1..v0.293.0 (~6 releases) is dominated by automated `feat(all): auto-regenerate discovery clients` commits (discovery-doc-derived client regeneration, no manual behavior changes). One substantive fix in range: `fix(transport): use ds.GetUniverseDomain() instead of raw ds.UniverseDomain field` (google-api-go-client#3660), touching `transport/dial.go`'s universe-domain resolution. We don't import `google.golang.org/api/transport` directly; only relevant if exercised transitively through `option`/client construction, and only for non-default (non-GDU) universe domains, which we don't use. Low exposure.
- Take: safe to take, no concerns.

### github.com/googleapis/enterprise-certificate-proxy (indirect, v0.3.17 → v0.3.20)

- Patch bump, mTLS cert helper pulled in transitively by the Google auth stack. Not imported directly anywhere in the codebase (`grep` for the import path returned nothing outside vendor). No exposure.

### google.golang.org/genproto/googleapis/rpc (indirect, pseudo-version bump)

- Untagged pseudo-version bump, standard churn accompanying the `google.golang.org/api` update. Not analyzed further — indirect, no direct import, no large jump.

## Findings

None.

## Checked

- Classified as dep-only: no files outside `go.mod`/`go.sum` changed.
- Confirmed direct/indirect status of all three bumped modules via `go.mod` diff.
- Grepped for import sites of `google.golang.org/api` and `enterprise-certificate-proxy` outside `vendor/` (repo has no vendor dir).
- Read release notes / commit range for `google.golang.org/api` v0.287.1..v0.293.0 via `gh api compare`.
- Confirmed release freshness via `proxy.golang.org` module info endpoint.

## Open questions

None.

## Dependency followups

Examined 2026-10-22 for PR #901, merged 2026-08-26 as `8f2ff6ae2c6e13a980e8c8ef354a0ff0399e7577`. No actionable opportunities identified; no candidates or handoff prompts.

- **`google.golang.org/api` v0.287.1 → v0.293.0 (direct):** Reviewed all six intermediate release entries in the [tagged changelog](https://github.com/googleapis/google-api-go-client/blob/v0.293.0/CHANGES.md). They announce “Auto-regenerate discovery clients” and the v0.291.0 transport fix, “Use ds.GetUniverseDomain() instead of raw ds.UniverseDomain field.” Prow imports only `option`, `googleapi`, and `iterator`; relevant uses include `option.WithAuthCredentials` (`pkg/io/credentials.go:39`), `googleapi.DefaultUploadChunkSize` (`pkg/io/option.go:63`), `iterator.NewPager` (`pkg/googlecloudbuild/client/client.go:138`), and `iterator.Done` (`pkg/io/iterator.go:58`, `cmd/webhook-server/secretmanager/secretmanager.go:140`). These releases offer no new replacement or deprecation affecting those calls. No explicit universe-domain configuration or transport workaround was found to adjust.
- **`github.com/googleapis/enterprise-certificate-proxy` v0.3.17 → v0.3.20 (indirect):** The [13-commit range](https://github.com/googleapis/enterprise-certificate-proxy/compare/v0.3.17...v0.3.20) includes fixes for OpenSSL keychain import, mTLS hostname routing, PKCS11 serialization, and internal logging changes. Prow has no imports or configuration references to this module, so there is no concrete adoption or migration target in our code.
- **`google.golang.org/genproto/googleapis/rpc` v0.0.0-20260630182238-925bb5da69e7 → v0.0.0-20260807164820-c8921c73eeea (indirect):** Examined the [upstream range](https://github.com/googleapis/go-genproto/compare/925bb5da69e7...c8921c73eeea) and compared both module archives from `proxy.golang.org`: all nine module files are byte-for-byte identical. Prow also has no direct imports of this module; this bump unlocks no API changes to adopt.
