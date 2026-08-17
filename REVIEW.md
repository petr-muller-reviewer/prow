---
pr: kubernetes-sigs/prow#858
title: "chore(deps): bump cloud.google.com/go/pubsub from 1.47.0 to 1.51.0"
head_sha: f44144f805fc1d8f59b92be0780161458a9226ae
base: main
reviewed_at: 2026-08-17T10:08:04Z
verdict: approve
---

## What this PR does

- Bumps direct dependency `cloud.google.com/go/pubsub` v1.47.0 -> v1.51.0 (dependabot).
- Cascades ~20 transitive `cloud.google.com/go/*`, `go.opentelemetry.io/*`, `google.golang.org/*` version bumps in go.mod/go.sum as a result.
- Adds one new indirect dep, `cloud.google.com/go/pubsub/v2 v2.6.0`, pulled in transitively; not imported anywhere in this repo.
- Dep-only change: only `go.mod` (49 lines) and `go.sum` (124 lines) touched, no project source.

## Dependency analysis

- Freshness: v1.51.0 tagged 2026-07-15 per Go module proxy, ~33 days old at review time. Past the fresh-release risk window.
- Direct dependency, imported in `pkg/crier/reporters/pubsub/reporter.go:28`, `pkg/pubsub/subscriber/subscriber.go:26`, `pkg/pubsub/subscriber/server.go:27`, and test helper `test/integration/internal/fakepubsub/fakepubsub.go:25`. This is Prow's Pub/Sub job reporter and Pub/Sub-triggered-job subscriber (deserializes external push messages into ProwJob triggers) — moderate sensitivity, light import surface (3 non-test files).
- Changelog (v1/pubsub only; most upstream commits in range are for the separate unused `pubsub/v2` module):
  - `fix(pubsub): concurrent map write` (#20103, in 1.50.4) — race fix, backported from v2.
  - `fix(pubsub): check for nil concurrency control span` (#14303) — nil-pointer guard around tracing spans.
  - `feat(o11y): regenerate clients for LRO tracing` (#20107, in 1.51.0) — internal GAPIC regen, no public API surface change.
- No GitHub Security Advisories found for `cloud.google.com/go/pubsub` in this range.
- `go build ./...` succeeds at PR head (module downloads only, no compile errors).

## Findings

None — dep-only PR, no project code changed.

## Checked
- Diff scope: confirmed only go.mod/go.sum changed (`git diff --stat`), classified dep-only.
- Import surface of `cloud.google.com/go/pubsub` across the repo (excluding vendor).
- New indirect `pubsub/v2` dep is unused directly; no risk from it.
- Freshness of new release via Go module proxy `.info` endpoint.
- Upstream changelog for pubsub v1 between old and new versions via GitHub compare/commits API.
- Security advisories via GitHub GraphQL — none found.
- `go build ./...` at PR head — succeeds.

## Open questions
None.
