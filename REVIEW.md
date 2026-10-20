---
pr: kubernetes-sigs/prow#879
title: "chore(deps): bump cloud.google.com/go/pubsub/v2 from 2.6.0 to 2.6.2"
head_sha: ad03089d119cfe3220a5492817075e47010301eb
base: main
reviewed_at: 2026-09-01T18:02:56Z
verdict: approve
---

## Verdict

Approve. This dependency-only update moves the direct Pub/Sub client from v2.6.0 to v2.6.2. The two patch releases fix publisher flow-control credit leakage and acknowledgement batches that can exceed Pub/Sub request limits; both are compatible with Prow's existing usage. The tagged v2.6.2 release was 13 days old at review time.

## What this PR does

- Updates the direct production dependency `cloud.google.com/go/pubsub/v2` from `v2.6.0` to `v2.6.2` in `go.mod`.
- Refreshes its checksums in `go.sum`.
- Removes stale indirect checksums left by module tidying.
- Includes the v2.6.1 flow-control credit-leak fix and v2.6.2 acknowledgement-batch-size fix.

## Findings

None.

## Checked

- Classified the change as dependency-only: only `go.mod` and `go.sum` change.
- Confirmed the dependency is direct and imported by three production files: `pkg/crier/reporters/pubsub/reporter.go`, `pkg/pubsub/subscriber/server.go`, and `pkg/pubsub/subscriber/subscriber.go`; three further imports are tests/integration fakes.
- Reviewed the upstream `pubsub/v2.6.0...pubsub/v2.6.2` release notes and component commits. v2.6.1 releases publisher flow-control credit after a scheduler error; v2.6.2 reduces acknowledgement batches to 1,000 to prevent `INVALID_ARGUMENT` failures with long acknowledgement IDs.
- Confirmed Prow creates a default publisher at `pkg/crier/reporters/pubsub/reporter.go:132-134`, and receives/acknowledges messages at `pkg/pubsub/subscriber/server.go:81` and `pkg/pubsub/subscriber/subscriber.go:123`; no changed API is used incompatibly.
- Confirmed the new version is a normal tag published 2026-08-19, not a pseudo-version; no CVE or breaking API change was identified.

## Open questions

None.

## Dependency followups

Examined 2026-10-20 for merged PR #879 (merge commit `c55704ffb386ddc489728f5cd406f701525059b2`). No actionable opportunities identified; no items or handoff prompts recorded.

- Direct module examined: `cloud.google.com/go/pubsub/v2 v2.6.0 → v2.6.2`, including intermediate v2.6.1. The [v2.6.1 release notes](https://github.com/googleapis/google-cloud-go/releases/tag/pubsub/v2.6.1) and [v2.6.2 release notes](https://github.com/googleapis/google-cloud-go/releases/tag/pubsub/v2.6.2) describe internal fixes to publisher flow-control credit release, exactly-once tracing, fake-server acknowledgement IDs, and acknowledgement batch size. They introduce no deprecations, replacement APIs, or new features requiring changes to Prow's call sites.
- Usage checked: default publishing in `pkg/crier/reporters/pubsub/reporter.go:132-134`; receive configuration in `pkg/pubsub/subscriber/server.go:95-100`; acknowledgements in `pkg/pubsub/subscriber/subscriber.go:122-126`; fake-server usage in `pkg/crier/reporters/pubsub/reporter_test.go:483-485`. The fixes apply automatically, and the existing receive settings do not override the internal acknowledgement batch size.
- Transitive scope checked: the PR's actual `go.mod`/`go.sum` diff upgrades no other modules. Although Pub/Sub's own dependency minimums increase between these versions, Prow's pre-bump manifest already requires those versions or newer (including IAM, GAX, OpenTelemetry, Google API, gRPC, genproto, and the affected `golang.org/x` modules). Consequently, those minimum changes unlock no additional dependency features for Prow in this PR.
- Removals examined: indirect `go.einride.tech/aip v0.83.0` and `gotest.tools/v3 v3.5.2` checksums disappear rather than upgrading; neither module is imported by Prow's Go source. No migration follows from their removal.

Scope: compared PR head `ad03089d119cfe3220a5492817075e47010301eb` against the PR's recorded pre-merge base `4cb91aa21d4341cbf124e1de44e507063c566e8f`; the current upstream merge-base already contains the merged PR and would hide its dependency diff.
