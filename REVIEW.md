---
pr: kubernetes-sigs/prow#892
title: "chore(deps): bump google.golang.org/protobuf from 1.36.12-0.20260120151049-f2248ac996af to 1.36.12 in /hack/tools"
head_sha: 9ec3e553adbe7a2a3bc456be0646aabb039a4ad0
base: main
reviewed_at: 2026-09-01T16:55:49Z
verdict: approve
---

## Verdict

Approve. This is a tool-only, direct dependency update from an untagged pseudo-version to its corresponding 22-day-old tagged release. The relevant upstream changes do not affect Prow's use of `protoc-gen-go`.

## What this PR does

- Updates `google.golang.org/protobuf` in the nested `hack/tools` Go module.
- Replaces pseudo-version `v1.36.12-0.20260120151049-f2248ac996af` with tagged `v1.36.12`.
- Updates only the associated two `go.sum` checksums.
- Leaves Prow application code and the root Go module unchanged.

## Findings

None.

## Checked

- Classification: dep-only; only `hack/tools/go.mod` and `hack/tools/go.sum` change.
- `google.golang.org/protobuf` is direct in `hack/tools/go.mod:12`; its sole tools-module use is blank-importing `google.golang.org/protobuf/cmd/protoc-gen-go` at `hack/tools/tools.go:35`.
- The release is tagged, dated 2026-08-10, and was 22 days old at review; the previous version was an untagged pseudo-version.
- The upstream range contains the release commit, numeric descriptor-default parsing that accepts hexadecimal/octal literals, and protobuf/protoc edition/test metadata updates. No stated security fix or API incompatibility affects this generator-only use.
- `go mod verify` in `hack/tools` passed.

## Open questions

None.
