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

## Dependency followups

Examined on 2026-10-20: no actionable opportunities identified; no items or handoff prompts recorded.

- Context: PR #881 merged on 2026-08-25 as `9ce9de565a570827be6c9e55ac540a1351a6796f`. Compared PR head `764a1e54b78bbb53c72aa72fe07bbe53c0291850` with its parent `c55704ffb` because the merged PR head is already an ancestor of `upstream/main`; their merge-base would hide the original bump.
- Module examined: Go direct production dependency `cloud.google.com/go/secretmanager`, `v1.20.0` → `v1.21.0`. Read the full intervening changelog entry and compared the published module sources. The [release notes](https://github.com/googleapis/google-cloud-go/releases/tag/secretmanager/v1.21.0) describe the change as “Update API sources and regenerate.”
- New APIs `EnableManagedRotation` and `RotateSecret` rotate Cloud SQL database passwords. They cannot replace Prow's certificate/key generation and version upload in `cmd/webhook-server/helpers.go:171-178` and `cmd/webhook-server/secretmanager/secretmanager.go:110-124`.
- Generated additions include secret-type metadata and managed-rotation status fields. No concrete simplification or correctness benefit was identified for Prow's existing create, list, get, update-label, add-version, or access-version calls in `cmd/webhook-server/secretmanager/secretmanager.go:59-174`.
- No newly deprecated API used by Prow or relevant default change was found. The stronger `Close()` documentation repeats an existing cleanup requirement rather than introducing a new API or migration opportunity.
- Transitive scope: no indirect module version changes are introduced by this PR. Although the dependency's own minimum requirements increase, every new requirement is already met or exceeded by Prow's unchanged root requirements before the bump. Those upstream requirement increases therefore unlock no additional transitive features here.
