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

## Dependency followups

No opportunities identified. Examined direct Go dependency `google.golang.org/protobuf`, `v1.36.12-0.20260120151049-f2248ac996af` → `v1.36.12`, for PR #889 (merged 2026-08-25 at `796b00398130fcba87242a54299c1bb68f3c7c17`). The manifest/checksum diff contains no transitive version changes.

- The [actual upstream commit range](https://github.com/protocolbuffers/protobuf-go/compare/f2248ac996af...v1.36.12) adds hexadecimal/octal numeric-default parsing and updates protobuf type metadata, generated descriptors, and upstream tests. Prow has no dynamic-descriptor construction, numeric-default workaround, or use of the newly added descriptor fields to migrate or simplify.
- Existing Gangway generated message/runtime use and ResultStore timestamp/duration/wrapper construction need no call-site changes. No newly deprecated API used by Prow was identified in this range.
- The [release notes](https://github.com/protocolbuffers/protobuf-go/releases/tag/v1.36.12) also describe changes since `v1.36.11` that were already present in the old pseudo-version, including prototext recursion limits and protojson fixes. These are not newly unlocked by this PR; the example's `prototext.Unmarshal` call at `pkg/examples/gangway/main.go:69` therefore provides no followup opportunity for this bump.

No items or handoff prompts recorded.
