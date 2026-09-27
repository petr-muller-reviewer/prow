---
pr: kubernetes-sigs/prow#980
title: "chore(deps): bump cloud.google.com/go/pubsub/v2 from 2.6.2 to 2.7.0"
head_sha: 6583f05345ff278a4b960bfbbc08aa22eeccc18a
base: main
reviewed_at: 2026-09-27T11:06:48Z
verdict: approve
---

# Review

## Verdict

Approve.

This is a direct, dep-only update to a tagged Google Cloud Pub/Sub release with 30 days of soak time. OSV reports no advisory for either the old or new version. The upstream regeneration adds API surface for message-transform compression and compiled Protocol Buffer schemas; it removes no public Go symbols and does not intersect Prow's publisher or subscriber use.

## What this PR does

- Updates the direct Go module dependency `cloud.google.com/go/pubsub/v2` from `v2.6.2` to `v2.7.0`.
- Updates only the associated checksums in `go.sum`; it changes no Prow source, tests, configuration, or vendored code.
- Takes regenerated Pub/Sub API definitions for message-transform compression and compiled Protocol Buffer schemas.
- Leaves Prow's use of Pub/Sub client creation, message publishing, subscription receiving, and message decoding unchanged.

## Findings

None.

## Resolved

None.

## Checked

- Classified the PR as dep-only from the `main` merge base: only `go.mod` and `go.sum` differ.
- Confirmed `cloud.google.com/go/pubsub/v2` is a direct requirement and resolves from Prow's Pub/Sub reporter package.
- Confirmed the module proxy identifies `v2.7.0` as tagged upstream commit `78017361caf0da206af1b7ece31471e698b89008`, published 2026-08-27 (30 days old at review time).
- Queried OSV for `v2.6.2` and `v2.7.0`; neither version has an advisory. `govulncheck` is unavailable in this environment.
- Reviewed the upstream `v2.6.2...v2.7.0` generated API diff: it adds (and removes no) public declarations, including compression message-transform and compiled-schema types. Prow does not use either API area in `pkg/crier/reporters/pubsub` or `pkg/pubsub/subscriber`.
- Ran `go test ./pkg/crier/reporters/pubsub ./pkg/pubsub/subscriber` successfully.

## Open questions

None.
