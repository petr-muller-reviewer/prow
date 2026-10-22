---
pr: kubernetes-sigs/prow#911
title: "chore(deps): bump github.com/tektoncd/pipeline from 1.15.0 to 1.15.1"
head_sha: e5a5b4b241b11a18bcdd8aa3e65fdc56f4c86d6e
base: main
reviewed_at: 2026-08-31T17:21:06Z
verdict: approve
---

## What this PR does

- Dependabot PR bumping the direct dependency `github.com/tektoncd/pipeline` from v1.15.0 to v1.15.1 in `go.mod`/`go.sum`.
- Dep-only change: no vendor directory in this repo, no project source touched.
- `tektoncd/pipeline` is Prow's Tekton PipelineRun job-execution integration: imported by `cmd/pipeline/controller.go`, `pkg/apis/prowjobs/v1/types.go`, `pkg/config/{config,jobs}.go`, and a generated clientset/lister/informer under `pkg/pipeline/...` (32 importing files total).
- The v1.15.0→v1.15.1 diff upstream is 4 commits, all dependency bumps internal to tektoncd/pipeline (sigstore/sigstore + its kms/aws and kms/hashivault subpackages, and github/codeql-action/upload-sarif, a CI-only Action dependency) — no API or behavior changes reach the client/informer code Prow actually imports.
- New version is a real tagged release (not a pseudo-version), tag cut 2026-08-10, GitHub release page published 2026-08-27; no CVE or security-fix angle.

## Findings

None.

## Checked

- Confirmed dep-only classification: `git diff --stat` shows only `go.mod`/`go.sum` (3 lines changed total).
- Confirmed `github.com/tektoncd/pipeline` is a direct dependency (no `// indirect` marker, go.mod:58).
- Walked the upstream compare (`v1.15.0...v1.15.1`, 4 commits) — all dependency-bump commits, none touching code paths Prow imports (client/lister/informer/types packages untouched).
- Release freshness: tagged 2026-08-10 (21 days before this review), not a pseudo-version.

## Open questions

None.

## Dependency followups

No actionable opportunities identified. Accepted: 0; skipped: 0. No handoff prompts.

PR #911 merged on 2026-08-29 at `88f56c0e89c868ec4e6bf0305fe23c7efa984ae7`. Because the PR head is already an ancestor of `upstream/main`, their merge-base is the PR head itself; the original bump was examined against its parent (`69388fecc`) instead.

- **Direct module examined:** `github.com/tektoncd/pipeline` v1.15.0 → v1.15.1. This is the only module version changed in Prow's `go.mod`/`go.sum`; no indirect versions change. The [release notes](https://github.com/tektoncd/pipeline/releases/tag/v1.15.1) and [four-commit comparison](https://github.com/tektoncd/pipeline/compare/v1.15.0...v1.15.1) contain dependency updates, with no new APIs, deprecations, or behavior changes in the PipelineRun types Prow uses at `cmd/pipeline/controller.go:629`, `cmd/pipeline/controller.go:712`, `pkg/apis/prowjobs/v1/types.go:201`, and `pkg/config/config.go:2971`.
- **Upstream transitive updates screened:** `github.com/sigstore/sigstore`, `github.com/sigstore/sigstore/pkg/signature/kms/aws`, and `github.com/sigstore/sigstore/pkg/signature/kms/hashivault`, each v1.10.8 → v1.10.9. Prow imports none of these, nor the updated Vault, ttlcache, or Sigstore protobuf packages; no concrete Prow migration follows from them.
- **Shared transitive dependencies:** Tekton raises `github.com/aws/aws-sdk-go-v2` v1.41.7 → v1.43.0, its `config` module v1.32.17 → v1.32.31, and `credentials` v1.19.16 → v1.19.30. Prow already required v1.43.7, v1.32.38, and v1.19.37 respectively before this PR, including for `pkg/io/providers/aws.go:26`. Prow likewise already required newer AWS support modules, Smithy, and the `google.golang.org/genproto/googleapis/api` and `/rpc` revisions than Tekton's updated requirements. These updates unlock no additional APIs here.
- The remaining release change updates Tekton's own `github/codeql-action/upload-sarif` v4.37.1 → v4.37.5; it supplies no Go functionality to Prow.
