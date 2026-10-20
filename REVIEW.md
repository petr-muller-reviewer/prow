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
