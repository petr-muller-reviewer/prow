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
