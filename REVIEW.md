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

## Dependency followups

Examined on 2026-10-20: no improvement opportunities identified; no items or handoff prompts.

- Scope: PR #892, merged as `f678c3d2cfa3d13e672584882585f4d62398866f`; direct Go dependency `google.golang.org/protobuf` in `hack/tools` changed from `v1.36.12-0.20260120151049-f2248ac996af` to `v1.36.12`. No transitive versions changed; the upstream module's `go.mod` is identical between these versions.
- Reused the reviewed three-commit upstream changelog: numeric descriptor defaults gain hexadecimal/octal parsing, protobuf/protoc edition and test metadata are refreshed, and the version is tagged. No new deprecation or replacement API calls for a migration in Prow.
- Usage cross-check: `hack/tools/tools.go:35` tracks `protoc-gen-go`, but `hack/make-rules/update/codegen.sh:50` explicitly installs `protoc-gen-go@v1.32.0`, independently of this module pin. The only project proto, `pkg/gangway/gangway.proto:1`, uses proto3; no explicit numeric defaults or affected descriptor/edition API call sites were found in project code. Neither upstream change supplies a concrete simplification or feature adoption for this PR's usage.
