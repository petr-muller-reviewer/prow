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

## Dependency followups

Examined on 2026-10-20. No concrete opportunities identified; no items or handoff prompts recorded. PR #883 merged as `10f425fd6c7c425ba34d8ed6c2964a7384fc3e04` on 2026-08-25.

- Direct: `github.com/sirupsen/logrus` `v1.9.4` → `v1.10.1`, including intermediate `v1.10.0`. The [changelog](https://github.com/sirupsen/logrus/blob/v1.10.1/CHANGELOG.md) adds slog adapters, narrower logging interfaces, and explicit caller support, and deprecates `Entry.HasCaller` and `MutexWrap`. The sole importing file in the affected module, `hack/tools/prowimagebuilder/main.go:32`, uses ordinary fields, errors, and level methods; it has no deprecated calls, slog bridge, custom caller handling, or broad logger interface to replace. Formatter and concurrency improvements apply automatically; the byte-field formatting change requires no adjustment to its string and string-slice fields.
- Indirect: `github.com/stretchr/testify` `v1.11.1` → `v1.12.0`. The [release notes](https://github.com/stretchr/testify/releases/tag/v1.12.0) cover assertion aliases, fixes to assertions/mocks/suites, and internal vendoring of go-spew and go-difflib. No owned Go files in `hack/tools` import Testify; the repository's Testify imports belong to the separate root module, which remains on `v1.11.1`. There is no affected call site to migrate.
- Indirect: `github.com/stretchr/objx` `v0.5.2` → `v0.5.3`. The [release notes](https://github.com/stretchr/objx/releases/tag/v0.5.3) cover dependency, test-helper, documentation, and CI changes. No owned Go files import Objx; no new runtime API or call-site improvement applies.
- Removed indirect requirements: `github.com/davecgh/go-spew` `v1.1.2-0.20180830191138-d8f796af33cc` and `github.com/pmezard/go-difflib` `v1.0.1-0.20181226105442-5d4384ee4fb2`. These removals follow Testify's internal vendoring; they do not unlock a project-code change.
