---
pr: kubernetes-sigs/prow#877
title: "chore(deps): bump gocloud.dev from 0.40.0 to 0.46.0"
head_sha: d69423cf4bd44339da13a2554cfc50d9330812ab
base: main
reviewed_at: 2026-09-01T18:24:37Z
verdict: approve
---

## Verdict

Approve. This dep-only change updates the directly used Go Cloud Development Kit after a 90-day soak. Its material `blob` changes are compatible with Prow's use, including the S3 upload-manager migration: Prow does not use `blob.Writer.As` or AWS manager types.

## What this PR does

- Updates direct dependency `gocloud.dev` from `v0.40.0` to `v0.46.0` in `go.mod`.
- Regenerates `go.sum` and the indirect module graph.
- Replaces the now-unused AWS SDK v1 and S3 manager modules with AWS SDK v2 transfermanager through Go Cloud.

## Findings

No findings.

## Checked

- Classification: dep-only (`go.mod`, `go.sum` only); `git diff --check HEAD^ HEAD` is clean.
- `v0.46.0` is a tagged release from 2026-06-02T22:21:46Z, 90 days old at review; no pseudo-version or freshness concern.
- Prow imports `gocloud.dev/blob` in five production files: `pkg/io/{opener.go,option.go,iterator.go}` and `pkg/io/providers/{providers.go,aws.go}`. It uses blob storage to read, write, list, inspect, and sign artifact objects for GCS, S3, local, and memory backends.
- The relevant upstream changes are the blob OpenTelemetry migration, S3 403-to-PermissionDenied mapping, file-bucket root-escape fix, typed error support, and v0.46 S3 upload migration from `s3/manager` to `s3/transfermanager`. Prow constructs its own `*s3.Client` in `pkg/io/providers/aws.go:54-92` and never calls the affected `blob.Writer.As` API; the breaking API change does not land in Prow.
- `go test ./pkg/io/...` passes.

## Open questions

None.

## Dependency followups

Outcome: 2 accepted, 0 skipped. PR #877 merged on 2026-08-25 as `4cb91aa21d4341cbf124e1de44e507063c566e8f`; execute these prompts against the merged default branch, checking current call sites before editing.

Examined `gocloud.dev v0.40.0 → v0.46.0` (all intermediate release notes), indirect `github.com/google/wire v0.6.0 → v0.7.0` (build-dependency fix; no Prow imports), and the replacement of indirect `github.com/aws/aws-sdk-go-v2/feature/s3/manager v1.17.10` with `feature/s3/transfermanager v0.2.3` (handled inside Go Cloud; no Prow imports). Removed dependencies `github.com/aws/aws-sdk-go v1.55.5` and `github.com/jmespath/go-jmespath v0.4.0` have no Prow imports. Other removed go.sum entries are checksum cleanup, not selected-version upgrades. No further candidates: the new list iterator does not directly replace Prow's provider-neutral Next interface; other changes either apply internally or concern drivers Prow does not use.

### Native conditional writes for S3 artifacts

- status: accepted
- category: api-upgrade
- necessity: should — replace a racy existence check and eliminate an unclosed reader.
- dependency: `gocloud.dev v0.40.0 → v0.46.0`; `IfNotExist` introduced in v0.42.0.
- source: [upstream #3533](https://github.com/google/go-cloud/pull/3533), release note: "blob: introduce conditional write option for storage".
- where: `pkg/io/opener.go:312-325`, `pkg/io/opener.go:523-561`; create-only callers in `pkg/crier/reporters/gcs/reporter.go:156` and `pkg/crier/reporters/gcs/reporter.go:177`.
- handoff:

```text
Mode: deliver
In kubernetes-sigs/prow, now that PR #877 — "chore(deps): bump gocloud.dev from 0.40.0 to 0.46.0" — merged as 4cb91aa21d4341cbf124e1de44e507063c566e8f, adopt native conditional writes for the Go Cloud blob path. Work against the merged default branch and first verify the relevant code is still present.

The v0.42.0 release introduced "blob: introduce conditional write option for storage" (https://github.com/google/go-cloud/pull/3533). In pkg/io/opener.go:312-325, opener.Writer currently implements PreconditionDoesNotExist with a BeforeWrite callback that opens a reader, then writes separately. This races with competing writers and leaves a successful reader unclosed. Replace the callback with blob.WriterOptions.IfNotExist, mapping the existing option's nil/false/true semantics. Account for the fact that failed conditions may now surface from Write or Close. Update isErrUnexpected/WriteContent's handling to recognize gcerrors.ErrFailedPrecondition without suppressing unrelated failures. Retain compatibility with existing exported precondition errors and their callers where needed.

Acceptance criteria: create-only writes leave existing content intact, including competing writes to one key; one concurrent creator wins; unconditional writes still overwrite. Expected failed-precondition errors from Write and Close retain WriteContent's existing treatment. Exercise the Go Cloud path (memblob can provide deterministic coverage) and run go test ./pkg/io/... plus affected crier reporter tests. Review the S3 driver's IfNoneMatch mapping and report any S3-compatible-provider limitation; do not claim a live S3 integration check unless performed.

Scope: change the blob conditional-write path and necessary error handling/tests only. Preserve the GCS SDK and local-file paths, the public Opener interface, and crier's intended started.json/finished.json behavior. Do not migrate providers or redesign iteration.
```

### Use Go Cloud's not-found sentinel

- status: accepted
- category: simplification
- necessity: could — remove a stale workaround and recognize the new public sentinel directly.
- dependency: `gocloud.dev v0.40.0 → v0.46.0`; error sentinels introduced in v0.46.0.
- source: [upstream #3708](https://github.com/google/go-cloud/pull/3708), release note: "all: support typed errors and errors.Is for error codes".
- where: `pkg/io/opener.go:162-180`, `pkg/io/opener_test.go:169-219`.
- handoff:

```text
Mode: deliver
In kubernetes-sigs/prow, now that PR #877 — "chore(deps): bump gocloud.dev from 0.40.0 to 0.46.0" — merged as 4cb91aa21d4341cbf124e1de44e507063c566e8f, use Go Cloud's public not-found sentinel in pkg/io. Work against the merged default branch and verify current call sites before editing.

The v0.46.0 release added "all: support typed errors and errors.Is for error codes" (https://github.com/google/go-cloud/pull/3708). pkg/io/opener.go:162-180 defines ErrNotFoundTest with the stale explanation that Go Cloud does not expose errors, special-cases that sentinel, and otherwise recognizes Go Cloud errors through gcerrors.Code(err) == gcerrors.NotFound. The latter does not recognize gcerrors.ErrNotFound itself. Migrate Go Cloud not-found recognition to errors.Is(err, gcerrors.ErrNotFound). Replace the stale test workaround while preserving source compatibility for exported ErrNotFoundTest, for example with a documented deprecated alias after checking compatibility needs. Preserve recognition of filesystem and GCS errors and remove only code made redundant by this migration.

Acceptance criteria: IsNotExist recognizes direct and wrapped gcerrors.ErrNotFound, an actual missing-object blob error, existing filesystem/GCS not-found errors, and the compatibility symbol. It returns false for nil and unrelated or permission-denied errors. Extend pkg/io/opener_test.go's existing cases as needed and run go test ./pkg/io/....

Scope: the not-found helper, compatibility symbol/documentation, and focused tests only. Do not redesign other error handling or alter provider behavior. Coordinate with the native conditional-write followup if both touch pkg/io/opener.go.
```
