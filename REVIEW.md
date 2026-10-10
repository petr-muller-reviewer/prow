---
pr: kubernetes-sigs/prow#992
title: "chore(deps): bump cloud.google.com/go/cloudbuild from 1.33.0 to 1.34.0"
head_sha: 692f1816cba167656b740485118c57b7be52b992
base: main
reviewed_at: 2026-10-05T22:35:36Z
verdict: needs-discussion
---

# Review

## Verdict

Needs discussion: the dependency change appears low risk, but I recommend waiting a few more days before merging because v1.34.0 is only 11 days old and contains no security fix that calls for early adoption.

## What this PR does

- Updates the direct Go dependency `cloud.google.com/go/cloudbuild` from v1.33.0 to v1.34.0 in `go.mod` and refreshes its checksums in `go.sum`.
- The release raises the module's Go directive from 1.25 to 1.26; Prow's `go 1.26.4` directive is compatible.
- The release adds `All()` iterator helpers. Prow's adapter uses `NextPage`, so it does not exercise the new helper.
- The dependency's gRPC and `golang.org/x/*` requirements move, but Prow already selects equal or newer versions; this bump does not change those effective versions.
- The PR changes only dependency metadata and checksums; it makes no project source changes.

## Findings

### [question] Let the release soak a little longer
- where: `go.mod:7`
- concern: The Go module proxy dates v1.34.0 to 2026-09-24, 11 days before this review. The upstream release notes report Go-version support updates and no security fix, so I recommend waiting until it has at least two weeks of soak.
- excerpt: |
    cloud.google.com/go/cloudbuild v1.34.0

## Checked

- Confirmed the PR is dep-only and the module is a direct dependency; `go mod why` traces it to `pkg/googlecloudbuild/client`.
- The SDK is imported in three Go files: the adapter implementation, its fake, and its tests. The adapter creates an authenticated Cloud Build client and wraps get, list, create, and cancel operations; no other in-repo call site was found.
- OSV reports no advisories for either v1.33.0 or v1.34.0.
- Full `govulncheck ./...` scans at base and head completed with no Cloud Build findings and the same findings at both revisions. Unrelated findings were `GO-2023-1901` with called traces and `GO-2026-5932` as imported-only.
- Compared the dependency's v1.33.0 and v1.34.0 source trees and release notes. The new iterator helper is not used by this adapter.

## Open questions

- Would you be willing to wait a few more days for v1.34.0 to complete a two-week soak before merging?
