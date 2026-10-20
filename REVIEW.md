---
pr: kubernetes-sigs/prow#881
title: "chore(deps): bump cloud.google.com/go/secretmanager from 1.20.0 to 1.21.0"
head_sha: 764a1e54b78bbb53c72aa72fe07bbe53c0291850
base: main
reviewed_at: 2026-09-01T18:02:45Z
verdict: approve
---

## Verdict

approve — This is a narrowly scoped, adequately soaked, direct dependency update. The only substantive upstream change is additive managed-secret-rotation API support; Prow does not call those APIs.

## What this PR does

- Updates `cloud.google.com/go/secretmanager` from `v1.20.0` to `v1.21.0`.
- Replaces the corresponding two module checksums in `go.sum`.
- Takes the Secret Manager client release published on 2026-07-20.
- Does not change Prow source, tests, generated code, or configuration.

## Findings

None.

## Checked

- Dependency-only classification: `go.mod` and `go.sum` are the only files changed.
- `v1.21.0` is a signed/tagged upstream release, 43 days old at review time; it is not a pseudo-version.
- The module is a direct production dependency, imported only by `cmd/webhook-server/secretmanager/secretmanager.go:23-24`.
- Prow uses existing Secret Manager create, get, list, add-version, and update operations to manage webhook-server certificate secrets; this is a sensitive but single-package integration.
- Upstream release notes and module diff show additive `EnableManagedRotation` and `RotateSecret` APIs plus generated protobuf/client support. Prow does not call either API; no security advisory or behavior change affecting its invoked APIs is listed.
- `git diff --check HEAD^ HEAD` passes.

## Open questions

None.
