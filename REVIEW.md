---
pr: kubernetes-sigs/prow#1073
title: "chore(deps): bump cloud.google.com/go/auth from 0.24.1 to 0.24.3"
head_sha: 665699dc177185ba24adf01bb98d5b498f85fea8
base: main
reviewed_at: 2026-11-16T22:57:10Z
verdict: approve
---

# Review

## Verdict

Approve — routine dependency bump; no advisories at either version, and the changed auth code paths are not ones prow exercises directly.

Routine dependabot bump of a direct dependency with a single import site. The release is 11 days old (tagged 2026-11-05), which is inside the normal soak window. OSV reports no advisories at either version and `govulncheck -scan module` reports zero findings at both base and head, so the bump fixes nothing and introduces nothing. The substantive upstream fixes in the range land in `grpctransport` DirectPath/Interconnect authority selection and the AWS SigV4 signer for external-account credentials; prow only calls `credentials.DetectDefault` for ADC detection from an operator-supplied file, so exposure is light. Build and the relevant unit test pass at PR head.

## What this PR does

- Bumps direct dependency `cloud.google.com/go/auth` from v0.24.1 to v0.24.3 in `go.mod`, with the corresponding `go.sum` churn.
- Pulls in the transitive releases that ride along: `genproto` and `genproto/googleapis/api` pseudo-versions (2026-09-04 → 2026-10-12), `gax-go/v2` 2.26.2 → 2.27.0, `cloud.google.com/go/iam` 1.13.0 → 1.14.0, `cloud.google.com/go/longrunning` 1.2.0 → 1.3.0, and `otel*` 1.46.0 → 1.47.0 (plus the new `go.opentelemetry.io/otel/log` module).
- Dep-only: no project source, config, docs, or tests change.
- Requires no adaptation in prow code — the `credentials.DetectDefault` / `CredentialsFile` API we use is unchanged.

## Findings

No findings.

## Checked

- Classification: `git diff --stat 86b2c4908..HEAD` touches only `go.mod` and `go.sum` — dep-only, no code review needed.
- Freshness: auth v0.24.3 tagged 2026-11-05T18:54:59Z, v0.24.2 tagged 2026-10-28; both real tagged releases (not pseudo-versions), ~11 days of soak at review time.
- OSV: no advisories for `cloud.google.com/go/auth` at v0.24.1 or v0.24.3.
- `govulncheck -scan module`: 0 findings on both sides (base `86b2c49`, head `665699d`), so no symbol-level scan was needed. Note: govulncheck v1.9.0 refuses the pristine repo ("no Go files in <root>", the module has no root package); scans were run on throwaway copies with a root package added, dependency graphs unchanged.
- Usage: `cloud.google.com/go/auth` imported in one file, `pkg/io/credentials.go:22`, calling `credentials.DetectDefault`; consumed by `pkg/googlecloudbuild/client/client.go:96` and `pkg/io/opener.go:144` (GCS). `go mod why -m` confirms the single chain `sigs.k8s.io/prow/pkg/io` → `cloud.google.com/go/auth/credentials`.
- Changelog v0.24.1 → v0.24.3: v0.24.2 — additional validation for REST RPC requests (#20753, which only bumps `auth/go.mod`/`go.sum`) and storage-direct authority for Interconnect in `auth/grpctransport` (#20738); v0.24.3 — strip `-fips` suffix from the AWS service name in the SigV4 signer (#20805, `auth/credentials/internal/externalaccount/aws_provider.go`). None touch the ADC detection path we call; the AWS fix only applies if an operator configures AWS `external_account` credentials, and then it is a fix.
- Transitive churn: gax-go v2.27.0 (2026-10-21) adds a path-expression helper; otel v1.47.0 dated 2026-10-02; iam/longrunning patch releases from 2026-09-24. All benign, none imported directly by prow.
- Build/tests at head: `go build ./pkg/io/... ./pkg/googlecloudbuild/...` and `go test ./pkg/io -run TestGoogleCredentialsFileOption` pass.

## Open questions

- None.

## Dependency followups

No opportunities identified — no candidate items, no handoff prompts.

- Examined `cloud.google.com/go/auth` v0.24.1 → v0.24.3 (direct; releases v0.24.2, v0.24.3). Source diff is limited to `credentials/internal/externalaccount/aws_provider.go` (AWS SigV4 `-fips` fix), `grpctransport/directpath.go` (Interconnect authority selection), `internal/version.go`, and the module files. `credentials/detect.go` — the only auth file prow uses (`pkg/io/credentials.go:22`, `credentials.DetectDefault`) — is byte-identical, so the bump exposes no new/deprecated API at our call site; the `CredentialsFile` deprecation already exists in v0.24.1, so it is not attributable to this bump.
- Examined `github.com/googleapis/gax-go/v2` 2.26.2 → 2.27.0 (indirect; no prow import). v2.27.0 adds only a path-expression helper — no call site in prow.
- Examined `google.golang.org/genproto` and `genproto/googleapis/api` pseudo-version 2026-09-04 → 2026-10-12 (direct require, but prow imports only `resultstore/v2`, `protobuf/field_mask`, and `api/annotations`). The 21-file commit range (go-genproto `d0f1323..4341edf`) touches none of those packages — only "update all" dependency refreshes.
- Examined `cloud.google.com/go/iam` 1.13.0 → 1.14.0 and `cloud.google.com/go/longrunning` 1.2.0 → 1.3.0 (indirect, not imported by prow). Release notes contain only "Update supported go versions".
- Examined `go.opentelemetry.io/otel*` 1.46.0 → 1.47.0 plus the new `go.opentelemetry.io/otel/log` module (indirect; prow imports no `go.opentelemetry.io/otel` package and configures no SDK/exporters). The release's new APIs (logs API stability, `WithMaxExportBatchSize`, `WithAttributeValueDepthLimit`, probe samplers) and fixes have no call site in prow.
