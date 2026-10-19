---
pr: kubernetes-sigs/prow#876
title: "chore(deps): bump cloud.google.com/go/storage from 1.56.0 to 1.65.0"
head_sha: 71779405cdfd3bb63a4890570991edf7d4272992
base: main
reviewed_at: 2026-09-01T18:48:20Z
verdict: approve
---

## Verdict

Approve. This is dependency-only churn. Storage v1.65.0 has adequate soak time, its Go 1.25 requirement is compatible with Prow's Go 1.26.4 declaration, and the intervening behavior changes do not overlap Prow's ordinary GCS object I/O surface in a concerning way.

## What this PR does

- Updates the direct GCS Storage client from `cloud.google.com/go/storage v1.56.0` to `v1.65.0`.
- Resolves Storage's newer Google Cloud, OpenTelemetry, monitoring, and generated-proto dependencies.
- Updates direct `cloud.google.com/go/secretmanager` from `v1.16.0` to `v1.20.0` and `google.golang.org/genproto` to a newer pseudo-version as part of module resolution.
- Changes only `go.mod` and `go.sum`; no Prow source, configuration, or tests change.

## Findings

None.

## Checked

- PR diff is dependency-only: 29 additions and 29 deletions across `go.mod` and `go.sum`.
- Storage v1.65.0 is a tagged 2026-08-19 release; it was six days old at merge and is 13 days old at review.
- Storage requires Go 1.25.0; Prow declares Go 1.26.4.
- Prow imports Storage in seven files, with production use concentrated in `pkg/io/opener.go` for GCS client construction, object reads/writes/listing, metadata updates, preconditions, and signed URLs.
- Intervening Storage releases add checksum/retry/writer and reader/download fixes plus APIs Prow does not call; no CVE/security advisory was identified in their release notes.
- Secret Manager remains used only by `cmd/webhook-server/secretmanager`; its intervening releases are API-source regeneration.
- `git diff --check` passes.

## Open questions

None.

## Followups

Assessed 2026-10-19. PR #876 merged on 2026-08-25 as `568cdd7970e81242b3ac58fbd94836a73fd37cce`. No unresolved review findings, introduced TODOs, or deferred human review requests were found. Accepted: 1; skipped: 0.

### Probe GCS resumable uploads and failure handling

- Category: tests
- Necessity: could
- Mode: probe
- Where: `pkg/io/opener.go:274` (Writer), `pkg/io/opener.go:523` (WriteContent), `pkg/io/option.go:60` (writer options); existing small-object integration examples in `pkg/pod-utils/gcs/upload_test.go:342`.
- Why followup: the Storage bump crosses nine minor versions with upload checksum and retry changes. Existing fake-GCS tests cover small uploads; focused coverage of uploads spanning chunks and upload-finalization failures would strengthen confidence in artifact I/O. No regression has been demonstrated, so this did not block the merge.

```text
In kubernetes-sigs/prow, carry out a throwaway bug-finding probe following PR #876, "chore(deps): bump cloud.google.com/go/storage from 1.56.0 to 1.65.0", merged as 568cdd7970e81242b3ac58fbd94836a73fd37cce.

Work in a fresh worktree based on the current default branch, main. Recheck the current Storage version and relevant tests first; the review checkout contains the historical PR head. Focus on pkg/io/opener.go (Writer and WriteContent) and pkg/io/option.go (WriterOptions.apply). Existing emulator-based small-upload examples are in pkg/pod-utils/gcs/upload_test.go, Test_openerObjectWriter_Write.

Write temporary tests in the normal test location or scratch space, and run them to exercise these behaviors through Prow's I/O wrapper:
1. Upload a deterministic object larger than multiple effective upload chunks, then verify its exact length and bytes. Respect the SDK's actual chunk rounding and keep the payload bounded.
2. Inject a transient failure during a resumable upload. For an upload eligible for retry, verify eventual success without duplicated or missing bytes; otherwise verify that failure is reported. Establish retry eligibility from the current SDK and Prow options rather than assuming every write is retried.
3. Inject a terminal failure during finalization, including a case where Write succeeds but Close fails. Verify that Writer callers can observe the close error and that WriteContent returns it.

Use the existing fake-GCS setup where suitable. If its protocol support cannot model a case, use a controlled local HTTP handler or transport, configured only in the probe. Do not access real buckets or credentials. Triage each failure with evidence: a Prow implementation or SDK compatibility bug, an emulator limitation, or a mistaken test expectation. Do not treat an emulator's missing feature as a production regression. If a dependency regression is suspected, compare versions in isolation before attributing it to PR #876.

Acceptance criteria: exercise all three behaviors, or document concrete limitations for any behavior that cannot be exercised faithfully. Classify every failure. A real bug means artifact corruption, incorrect success reporting, or a failure of Prow's supported upload contract against valid protocol behavior. Apply only the minimal fix for a demonstrated real bug; report the failing case and commands so the maintainer can choose whether to retain a regression test. If no real bug is found, leave no files changed.

Scope guard: delete all probe tests and temporary instrumentation before finishing. The deliverable is a minimal bug fix if warranted, never the exploratory tests. Do not fix emulator-only problems, change general retry policy, introduce a transport migration, bump unrelated dependencies, or perform unrelated cleanup. Do not publish, push, open a PR, or comment on GitHub.
```
