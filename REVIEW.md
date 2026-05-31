---
pr: kubernetes-sigs/prow#703
title: "chore(deps): bump github.com/tektoncd/pipeline from 1.6.1 to 1.6.2"
head_sha: 94450c020a8c5c598b81f97535849526b0636c07
base: main
reviewed_at: 2026-05-31T00:58:10Z
verdict: approve
---

## Summary

Dependabot patch bump of `github.com/tektoncd/pipeline` v1.6.1 to v1.6.2 (security release, five CVEs). Only `go.mod` and `go.sum` touched. No Prow code changes. No transitive dependency changes.

## Findings

None.

## Checked

- API compatibility: Prow uses only `pkg/apis/pipeline/v1` types; CVE fixes are in resolver/admission code, not API structs
- Transitive deps: `go.sum` diff confined to direct module hash only
- Semver: patch bump within v1.6.x LTS line
- No `replace` directives for this module
- Deployment: no config/CLI/API/behavioral changes; zero operator action; trivial rollback

## Open questions

None.

## Deployment notes

- This bump updates imported Go types only. Does not remediate the CVEs in a running installation. Operators must upgrade Tekton Pipeline controller to v1.6.2 independently.
