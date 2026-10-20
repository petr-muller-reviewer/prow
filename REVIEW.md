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

## Dependency followups

Examined PR #880, merged as `9c9e83d5cbb8f4980fd453d421bfdb4c73b8f304`. Accepted: 0; skipped: 1. No handoff prompts.

- `github.com/sirupsen/logrus v1.9.4 → v1.10.1` (direct): reviewed v1.10.0 and v1.10.1 changelogs against Prow usage. No calls to deprecated `Entry.HasCaller` or `MutexWrap`; no existing `log/slog` integration to simplify. Performance and concurrency fixes apply without call-site changes.
- `github.com/stretchr/testify v1.11.1 → v1.12.0` (direct, test-only): five files import `assert`; no uses of the changed assertion-function types, ordered assertions, `PanicsWithError`, `mock`, or `suite` APIs that warrant migration. Diagnostic improvements apply automatically.
- `github.com/stretchr/objx v0.5.2 → v0.5.3` (transitive): no Prow imports; changes concern upstream tests, dependency maintenance, and CI, with no application API opportunity.

### Skipped: narrow the cherrypicker helper's logger

- category: api-upgrade
- necessity: could
- dependency: `github.com/sirupsen/logrus v1.9.4 → v1.10.1`; new `WarnLogger` interface introduced in v1.10.0.
- where: `cmd/external-plugins/cherrypicker/server.go:735-750`, `getPusherAndOrg`.
- proposed scope: replace the helper's `logrus.FieldLogger` parameter with `logrus.WarnLogger`, since its only logging call is `Warnf`; verify cherrypicker tests.
- decision: skipped by user; no implementation requested.
- source: [Logrus v1.10.0 release notes](https://github.com/sirupsen/logrus/releases/tag/v1.10.0).
