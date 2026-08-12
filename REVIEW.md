---
pr: kubernetes-sigs/prow#827
title: "chore(deps): bump github.com/andygrunwald/go-gerrit from 0.0.0-20210709065208-9d38b0be0268 to 1.1.1"
head_sha: c8be0c7f4f2d2f9c5fc3b2428e8af932a565e2e3
base: main
reviewed_at: 2026-08-12T11:46:43Z
verdict: request-changes
---

## What this PR does

- Bumps direct dependency `github.com/andygrunwald/go-gerrit` from a 2021 pseudo-version (`v0.0.0-20210709065208-9d38b0be0268`) to the first real tagged releases, landing on `v1.1.1`.
- Dep-only change: `go.mod`/`go.sum` only, no project code touched.
- Spans ~92 upstream commits / 4+ years of history, including the library's first ever tagged releases (v1.0.0, v1.1.0, v1.1.1).

## Findings

### [blocking] query `+`-join relies on removed go-gerrit URL-encoding special-case
- where: `pkg/gerrit/client/client.go:682`
- concern: |
    go-gerrit removed its special-casing of literal `+` in query values (upstream commit `a7b2c28fbf9b684a5f521276e939f432f7b8a6df`, "don't treat + as a special character for queries", included in v1.0.0+). Previously `addOptions` substituted a placeholder for `+` before calling `url.Values.Encode()` and restored it afterward, so a raw unescaped `+` reached Gerrit and was interpreted (via standard form-decoding) as a space joining ANDed search predicates. That protection is gone as of v1.1.1.

    `queryChangesForProjectWithoutMetrics` builds every project query with exactly this pattern, joining predicates with `+`. Verified locally that `url.Values.Encode()` now percent-encodes a literal `+` to `%2B`; Gerrit will decode `%2B` back to a literal `+` character in the query text (not a space), turning `"status:open+project:foo"` into one malformed token instead of two ANDed predicates. This is the only `+`-join site in the codebase (checked `pkg/gerrit/client/*.go` and `pkg/gerrit/adapter/*.go`), but it's on the path every Gerrit adapter poll loop calls. If correct, this bump silently breaks Gerrit change discovery for every configured Gerrit project (empty results or a 400 from Gerrit).
- excerpt: |
    opt.Query = append(opt.Query, strings.Join(append(additionalFilters, "project:"+project), "+"))

## Checked
- Release freshness: v1.1.1 released 2025-12-05 per proxy.golang.org — ~8 months old, adequate soak time, real tag (not a pseudo-version, unlike the old pin).
- Import surface: direct dep, used in `pkg/gerrit/client`, `pkg/gerrit/adapter`, `pkg/gerrit/fakegerrit`, `pkg/tide/{gerrit,codereview}.go`, `pkg/crier/reporters/gerrit`, plus integration tests — core to prow's Gerrit integration, not incidental.
- Grepped for other API surfaces changed upstream in this range (`CreateChangeDeprecated` removal, `DeleteDraftChange`→`DeleteChange` rename, `MoveInput.KeepAllLabels`→`KeepAllVotes` rename, new `ChangeInput`/`SubmitInput`/`CherryPickInput` fields) — no usages of the renamed/removed symbols in prow code, so those are non-issues here.
- go.sum diff is consistent with the go.mod version bump (h1/go.mod hashes only for the one module).

## Open questions
- Has this been tested against a real (or fake) Gerrit instance with a multi-predicate query (e.g. `additionalFilters` non-empty, or the `project:` clause combined with another filter) to confirm change discovery still works after the `+`-encoding behavior change?
- Would it be safer to change `pkg/gerrit/client/client.go:682` to join with `" "` (space) instead of `"+"` before taking this bump — matching go-gerrit's own updated example (`"change:249244+status:merged"` → `"change:249244 status:merged"`) and working correctly under both old and new library versions?
