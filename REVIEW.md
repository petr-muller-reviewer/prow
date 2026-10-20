---
pr: kubernetes-sigs/prow#880
title: "chore(deps): bump github.com/sirupsen/logrus from 1.9.4 to 1.10.1"
head_sha: 5c172fb819b326db98ab2204c5854ff10fd47116
base: main
reviewed_at: 2026-09-01T18:03:10Z
verdict: approve
---

## Verdict

Approve. This is a dependency-only update with no project-source changes. Logrus v1.10.1 is compatible with Prow's Go 1.26.4 baseline, uses the production JSON formatter, and includes relevant concurrency and reentrant-logging fixes. The accompanying Testify update is test-only.

## What this PR does

- Updates direct dependency `github.com/sirupsen/logrus` from `v1.9.4` to `v1.10.1` in `go.mod`.
- Updates direct test dependency `github.com/stretchr/testify` from `v1.11.1` to `v1.12.0` and its indirect `github.com/stretchr/objx` dependency from `v0.5.2` to `v0.5.3`.
- Regenerates the corresponding module checksums in `go.sum`.

## Findings

No findings.

## Checked

- PR diff: only `go.mod` and `go.sum` changed; no project code changed.
- `github.com/sirupsen/logrus v1.10.1`: tagged 2026-08-19; requires Go 1.23, satisfied by `go 1.26.4` in `go.mod:3`.
- Prow production logging defaults to `logrus.JSONFormatter` through `pkg/logrusutil/logrusutil.go:44-55`; the release's `TextFormatter` output changes do not affect normal production logs.
- Prow has a reentrant-logging deadlock regression test at `pkg/logrusutil/logrusutil_test.go:150-157`; Logrus v1.10 includes reentrant logging deadlock and race fixes.
- `github.com/stretchr/testify v1.12.0` is imported only by five `*_test.go` files; no production Go file imports it.
- `git diff --check HEAD^ HEAD` and `go test ./pkg/logrusutil ./pkg/config/secret` pass.
- Logrus v1.10.2, released after this PR, contains only a test-dependency update and no functional changes.

## Open questions

None.
