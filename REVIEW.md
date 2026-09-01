---
pr: kubernetes-sigs/prow#883
title: "chore(deps): bump github.com/sirupsen/logrus from 1.9.4 to 1.10.1 in /hack/tools"
head_sha: 978e94245738e11747e32474635cc49753f94122
base: main
reviewed_at: 2026-09-01T17:53:32Z
verdict: approve
---

## Verdict

Approve. This dependency-only update affects the isolated `hack/tools` module's image-build utility, not Prow's root module. The v1.10.1 release is a tagged, 13-day-old patch release; its formatter and concurrency fixes are compatible with the APIs used here, and OSV reports no advisories affecting this version.

## What this PR does

- Updates the direct `hack/tools` dependency `github.com/sirupsen/logrus` from `v1.9.4` to `v1.10.1`.
- Regenerates `hack/tools/go.sum`.
- Updates Logrus test-only transitive modules `github.com/stretchr/testify` (`v1.11.1` to `v1.12.0`) and `github.com/stretchr/objx` (`v0.5.2` to `v0.5.3`).
- Removes no-longer-required indirect test dependencies from the tools module.

## Findings

None.

## Checked

- Classification: dependency-only; only `hack/tools/go.mod` and `hack/tools/go.sum` changed.
- `v1.10.1` was released 2026-08-19, is a real tag, and is 13 days old at review time.
- The bump applies only to `sigs.k8s.io/prow/hack/tools`; root `go.mod` remains on Logrus `v1.9.4`.
- `hack/tools/prowimagebuilder/main.go:69-398` is Logrus's sole direct consumer in the bumped module. It uses level methods and `WithField(s)`/`WithError`; it does not use generic `Log*` methods, custom formatters, `Entry.HasCaller`, or raw `[]byte` fields affected by the release changes.
- Logrus `v1.10.1` requires Go 1.23; `hack/tools/go.mod` specifies Go 1.26.
- OSV's version-specific query returned no advisories for `github.com/sirupsen/logrus@v1.10.1`.
- A focused `go test` was not completed: package initialization builds `github.com/google/ko`, whose dependency build exceeded the 30-second execution limit.

## Open questions

None.
