---
pr: kubernetes-sigs/prow#885
title: "chore(deps): bump github.com/gorilla/mux from 1.8.0 to 1.8.1"
head_sha: c8e0de9ac9b57e39d150c58e3d44278278e71ed4
base: main
reviewed_at: 2026-09-01T17:44:53Z
verdict: approve
---

## Verdict

Approve. This is a dependency-only patch updating a long-established tagged release. The only behavioral router change relevant to Prow fixes 404-versus-405 selection when query matching fails; Prow's sole query-matcher use has no competing route pattern. The module is used only by integration-test fake HTTP servers.

## What this PR does

- Updates the direct Go dependency `github.com/gorilla/mux` from `v1.8.0` to `v1.8.1`.
- Updates the corresponding module checksums.
- Does not modify Prow source, tests, configuration, or generated artifacts.

## Findings

None.

## Checked

- `go.mod` and `go.sum`: the patch contains only the `github.com/gorilla/mux v1.8.0` to `v1.8.1` dependency update.
- Provenance and freshness: `v1.8.1` is a normal upstream tag from 2023-10-18, 1,049 days old at review time; it is not a pseudo-version.
- Upstream delta: substantive library changes add `Route.GetVarNames()` and correct method-mismatch handling after a query mismatch; remaining changes are documentation, tests, formatting, or CI/project maintenance.
- Usage: mux is directly required but imported solely by `test/integration/cmd/fakegitserver/main.go`, `test/integration/cmd/fakeghserver/main.go`, and `test/integration/cmd/fakegerritserver/main.go`.
- Exposure: these routes serve integration-test fakes, not Prow production handlers. The fake GitHub server's only query constraint is `Queries("per_page", "{page}")`; it has no overlapping route whose method-mismatch behavior could be affected.
- Security: an OSV package query returned no vulnerability records for `github.com/gorilla/mux`.
- Verification: selected fake-server package builds were started, but dependency compilation did not complete within the review execution window; no completed test result is recorded.

## Open questions

None.
