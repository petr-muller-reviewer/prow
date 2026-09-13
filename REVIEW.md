---
pr: kubernetes-sigs/prow#936
title: "chore(deps): bump github.com/sirupsen/logrus from 1.10.1 to 1.10.2 in /hack/tools"
head_sha: a3137535b212f6a89ebad6dd088cf941a69b637b
base: main
reviewed_at: 2026-09-13T21:58:39Z
verdict: approve
---

## Verdict

Approve. This is a direct dependency update confined to the `hack/tools` Go module. Logrus v1.10.2 is a tagged release from 2026-08-25 with no functional or API changes; it only updates its test dependency and removes legacy `gopkg.in/yaml.v3` from Logrus's dependency graph.

## What this PR does

- Updates `github.com/sirupsen/logrus` from `v1.10.1` to `v1.10.2` in `hack/tools/go.mod`.
- Updates the corresponding two `go.sum` entries.
- Leaves Prow source, generated code, and the root Go module unchanged.
- Limits the upgraded module's direct in-repository consumer to `hack/tools/prowimagebuilder/main.go`.

## Findings

No findings.

## Checked

- `git diff --check` is clean.
- `go test ./prowimagebuilder` in `hack/tools` completed without failures.
- Go proxy metadata identifies `v1.10.2` as tagged release `refs/tags/v1.10.2`, published 2026-08-25.
- Upstream release notes and commit range show only the `github.com/stretchr/testify` v1.12.1 update; no functional or security behavior changes.
- `github.com/sirupsen/logrus` is direct in `hack/tools/go.mod` and imported there only by `hack/tools/prowimagebuilder/main.go:32`.

## Open questions

None.
