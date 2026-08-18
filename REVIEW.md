---
pr: kubernetes-sigs/prow#850
title: "chore(deps): bump the aws group across 1 directory with 4 updates"
head_sha: c0237b8ef4eb3b54d6cbe754d77dc9b33d340e58
base: main
reviewed_at: 2026-08-17T10:42:52Z
verdict: approve
---

## What this PR does

- Dependabot group bump of 4 direct AWS SDK v2 modules: `aws-sdk-go-v2` v1.43.4→v1.43.5, `aws-sdk-go-v2/config` v1.32.35→v1.32.36, `aws-sdk-go-v2/credentials` v1.19.34→v1.19.35, `aws-sdk-go-v2/service/s3` v1.107.0→v1.107.1.
- Lockstep bump of transitive AWS SDK submodules (`smithy-go` v1.27.6→v1.27.7, `internal/*`, `feature/ec2/imds`, `service/sso*`, `service/sts`, etc.) pulled in by the same monorepo release.
- Changes are confined to `go.mod`/`go.sum` and `hack/tools/go.mod`/`hack/tools/go.sum` — no project source changed.

## Findings

None. Dep-only PR; no project code review applicable.

## Checked

- Classification: dep-only — diffed `go.mod`/`go.sum`/`hack/tools/go.mod`/`hack/tools/go.sum` against everything else in the diff; no other files touched.
- Freshness: new release tagged at commit `a14f5f1` / 2026-08-10, 7 days old at review time — within the "note but don't block" band, no pseudo-versions involved.
- Usage: all four direct deps import only into `pkg/io/providers/aws.go` (+ its test) — the S3 storage-provider backend (`config.LoadDefaultConfig`, `credentials.NewStaticCredentialsProvider`, `s3.NewFromConfig`). Single-file blast radius.
- Changelog (`CHANGELOG.md` at the release commit, plus `gh api compare` commit ranges for all four submodules): no functional entries for `aws-sdk-go-v2` core/`config`/`credentials`/`service/s3` in the 2026-08-06/07/10 releases — just "Dependency Update" markers. Only functional change reaching our usage is the `smithy-go` v1.27.7 bump (aws/aws-sdk-go-v2#3480): moves close-body/logger/service-metadata handling from per-request middleware into codegen, an allocation optimization, behavior-preserving. No CVE/security advisory in range.

## Open questions

None.
