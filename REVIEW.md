---
pr: kubernetes-sigs/prow#919
title: "chore(deps): bump cloud.google.com/go/storage from 1.65.1 to 1.66.0"
head_sha: e675cf7a0a330384203b5d728201eee039bcf2ef
base: main
reviewed_at: 2026-09-04T17:10:37Z
verdict: needs-discussion
---

## Verdict

needs-discussion — this is a narrowly scoped, API-compatible dependency-only update, but storage v1.66.0 was published eight days ago. The project's stated preference is roughly two weeks of release soak; wait about another week unless there is a reason to take this release immediately.

## What this PR does

- Updates the direct Go module `cloud.google.com/go/storage` from `v1.65.1` to `v1.66.0` in `go.mod`.
- Regenerates the corresponding two checksums in `go.sum`.
- Does not alter Prow source, configuration, generated project code, or tests.
- Takes a tagged Google Cloud Storage client release published 2026-08-27.

## Findings

None.

## Checked

- Diff classification: only `go.mod` and `go.sum` changed.
- Release provenance: `storage/v1.66.0` is a tagged release, not a pseudo-version; the Go proxy resolves it to `googleapis/google-cloud-go` commit `e281a8fd41f0226b81c2c6e3659b8a239d417a38`.
- Upstream release notes list one substantive change: regenerated APIs from updated Google API sources. The actual storage changes are confined to `storage/control/apiv2/*`; Prow does not import that package.
- Usage surface: direct imports appear in production only under `pkg/io/option.go:20`, `pkg/io/opener.go:35`, and `pkg/io/iterator.go:25`; four additional imports are tests. Production use covers authenticated and anonymous GCS clients, object reads/writes, listings, metadata updates, and signed URLs.
- No newly published upstream GitHub security advisory was returned for the release window.
- `go test ./pkg/io` passed.

## Open questions

- Is there a reason to take a release with only eight days of soak before the usual two-week window, rather than letting Dependabot retry after approximately 2026-09-10?
