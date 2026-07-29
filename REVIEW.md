---
pr: kubernetes-sigs/prow#790
title: "chore(deps): bump golang.org/x/net from 0.54.0 to 0.55.0"
head_sha: 76779feb42f72049a56522b05a9a61f631577bed
base: main
reviewed_at: 2026-07-29T00:50:53Z
verdict: approve
refresh_log:
  - from_sha: 77e537af301492918ffbbec9095f87f01b0720d5
    to_sha: 76779feb42f72049a56522b05a9a61f631577bed
    summary: >
      PR rebased onto a much newer main. x/text, x/crypto, x/sys, x/term were bumped to
      the previously-reviewed target versions by other merged PRs in the meantime, so
      this PR's own diff shrank to a single-module bump: x/net v0.54.0 -> v0.55.0. All 5
      CVE fixes previously identified in x/net/html remain within this narrower range.
      Author (petr-muller) left /lgtm on 2026-07-27.
---

## Summary

Dep-only PR (`go.mod`/`go.sum` only, no project code). Bumps:

- `golang.org/x/net` v0.54.0 -> v0.55.0 (direct)

Since previous review: PR was rebased onto a newer `main`, which had already absorbed
`x/text` v0.37.0, `x/crypto` v0.52.0, and other `x/*` bumps via separate merged PRs
(#806, #807). As a result this PR's own scope narrowed from a 5-module group bump to
just `x/net`. The CVE fixes previously found (CVE-2026-27136, CVE-2026-42506,
CVE-2026-42502, CVE-2026-25680, CVE-2026-25681) are all still delivered within the
v0.54.0..v0.55.0 range, so the prior analysis and verdict stand.

## Findings

No findings — dependency-only change, no project code to review.

## Checked

- Freshness: v0.55.0 tagged 2026-05-22, real tagged release, ~68 days old as of this refresh. No pseudo-versions.
- `x/net` usage unchanged: `pkg/githuboauth/githuboauth.go` (`x/net/xsrftoken`), `pkg/spyglass/lenses/html/html.go` (`x/net/html`, tokenizer-only via `html.NewTokenizer`, no `html.Parse`/`html.Render` calls).
- Re-verified the CVE fixes (CVE-2026-27136, CVE-2026-42506, CVE-2026-42502, CVE-2026-25680, CVE-2026-25681) are within `v0.54.0..v0.55.0`, not already landed at `v0.54.0` — confirmed via `gh api repos/golang/net/compare/v0.54.0...v0.55.0`. All five `html:` fix commits are present in that range. Exposure verdict unchanged: not on Prow's code path since `extractDocumentDetails` doesn't re-serialize via `html.Render`.
- `x/text`/`x/crypto`/`x/sys`/`x/term` are no longer part of this PR's diff (already on `main` via #806/#807 and other merges) — dropped from scope, no longer relevant to this review.
- Current base is `f2a00ef5199fea8239738bbfc59830f7c92db809` (post-rebase); `go.mod`/`go.sum` diff against it is exactly the `x/net` bump, nothing else.

## Open questions

None.
