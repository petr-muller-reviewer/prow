---
pr: kubernetes-sigs/prow#867
title: "chore(deps): bump github.com/yuin/goldmark from 1.8.2 to 1.8.5"
head_sha: dc1c09615d8283e52e417c3bca2bb48612d3ea07
base: main
reviewed_at: 2026-08-25T12:01:58Z
verdict: approve
refresh_log:
  - from_sha: dc1c09615d8283e52e417c3bca2bb48612d3ea07
    to_sha: 3ea8fc6b6f74e76f2ff064edaaea244c7ee06f2d
    at: 2026-08-25T12:01:58Z
    summary: "PR merged (rebased onto main via dependabot auto-rebase before merge; head SHA changed but the PR's own diff vs its merge-base is unchanged — still only the goldmark go.mod/go.sum bump). No new comments or reviews since prior review."
---

## Summary

Dep-only PR (dependabot). Only `go.mod`/`go.sum` change. Direct dependency
`github.com/yuin/goldmark` v1.8.2 -> v1.8.5.

Since previous review: PR merged (state MERGED, head now `3ea8fc6b6f74e76f2ff064edaaea244c7ee06f2d`).
The head SHA changed only because dependabot rebased onto `main` (which had advanced with
unrelated PR #859, pubsub v2 migration) before merge; the PR's own diff against its
merge-base is identical to what was reviewed — still just the goldmark bump. No new commits,
comments, or reviews from the author/reviewers since the prior review.

- New release age: v1.8.5 tagged 2026-07-28, ~28 days old at review time. Real tagged
  release, not a pseudo-version.
- Usage: single import site, `pkg/spyglass/lenses/markdown/markdown.go`. Prow's Spyglass
  markdown lens reads a job artifact, runs it through `goldmark.New(goldmark.WithExtensions(extension.GFM))`
  (default safe mode, no `WithUnsafe()`), and injects the result as `template.HTML` (bypasses
  Go autoescaping) into the served Spyglass UI.
- Changelog v1.8.2..v1.8.5:
  - v1.8.3: fix #556, fenced code block parsing degradation on nonzero block indent. Correctness only.
  - v1.8.4: "fix: disable svg in data:image urls" (`renderer/html/html.go`) — closes an XSS
    vector where `![](data:image/svg+xml;base64,...)` could carry an embedded script/event
    handler executed by the browser. Lands directly in the renderer path Prow exercises.
  - v1.8.5: fix #568, another fenced-code-block parser fix (`parser/fcode_block.go`). Correctness only.
- Exposure: light import surface (one call site) but sensitive (untrusted-ish markdown from
  job artifacts rendered to HTML without escaping). The v1.8.4 fix is a net security
  improvement for exactly this path.

## Findings

None. Dep-only bump; no project code changed.

## Checked

- Diff scope: confirmed only `go.mod`/`go.sum` changed (`git diff --stat`).
- Direct vs indirect: no `// indirect` marker on goldmark in go.mod.
- Import surface: `grep` shows one importer, `pkg/spyglass/lenses/markdown/markdown.go`.
- Goldmark config in that file: no `WithUnsafe()`, default-safe HTML rendering.
- Changelog commits v1.8.2..v1.8.5 via `gh api compare` and per-tag `gh release view`.
- Release freshness via `proxy.golang.org` `.info` endpoint (v1.8.5: 2026-07-28T01:16:48Z).

## Open questions

None.
