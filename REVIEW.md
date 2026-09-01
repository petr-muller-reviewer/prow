---
pr: kubernetes-sigs/prow#889
title: "chore(deps): bump google.golang.org/protobuf from 1.36.12-0.20260120151049-f2248ac996af to 1.36.12"
head_sha: 5d768d2bd1ab1f50fb3b0a61fe08bf8452f5e07b
base: main
reviewed_at: 2026-09-01T16:37:29Z
verdict: approve
---

## Verdict

Approve. This dependency-only change replaces an untagged pseudo-version with the tagged `v1.36.12` release, published 2026-08-10 (22 days of soak). The limited runtime changes do not overlap Prow's direct production use, and all PR CI checks passed.

## What this PR does

- Updates direct production dependency `google.golang.org/protobuf` from `v1.36.12-0.20260120151049-f2248ac996af` to `v1.36.12` in `go.mod`.
- Replaces the matching module checksums in `go.sum`.
- Moves from an untagged January pseudo-version to the August tagged release.

## Findings

None.

## Checked

- Classification: only `go.mod` and `go.sum` change; no Prow source changes.
- Provenance and freshness: Go proxy resolves `v1.36.12` to upstream tag `refs/tags/v1.36.12`; release published 2026-08-10.
- Upstream delta after commit `f2248ac996af`: protobuf type metadata updates and support for hexadecimal/octal numeric defaults in descriptor parsing; no published security advisory.
- Usage: 12 Prow Go files import protobuf, primarily generated Gangway types and ResultStore timestamp/duration/wrapper construction. The only direct import of an affected encoder is example-only `prototext.Unmarshal` in `pkg/examples/gangway/main.go:25`.
- CI: unit, integration, image-build, lint, and nonblocking race-detector checks succeeded.

## Open questions

None.
