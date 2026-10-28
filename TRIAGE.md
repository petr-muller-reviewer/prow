---
issue: kubernetes-sigs/prow#376
title: "Cannot run prow at a subpath"
state: open
labels: kind/bug, lifecycle/stale, area/deck
main_sha: e601a1ffafd7d8d3a781238a4c5f4233d6248f68
triaged_at: 2026-07-27T00:38:17Z
verdict: accepted
---

## Findings

### [reproducibility] Confirmed by code inspection, no repro steps needed
- detail: Deck has no base-path concept anywhere in routing, templates, or frontend JS. Any subpath ingress deployment (e.g. `nginx.ingress.kubernetes.io/rewrite-target` mapping `/prow/?(.*)` to Deck) will 404 on static assets and mis-navigate, exactly as the reporter describes.
- evidence: full-repo search for `BasePath`/`base-path`/`subpath`/`prefix` under `cmd/deck/` and `pkg/deck/` at `main_sha` returns no config/flag hits — only unrelated GCS-prefix and TS `Prefix` fields.

### [cause] Root-relative routing and inconsistent absolute/relative URL construction
- detail: `cmd/deck/main.go` registers every route on a single `http.NewServeMux()` with literal root-relative paths. `cmd/deck/template/base.html` emits absolute asset/nav URLs (`/static/...`, `/pr`, `/tide`), while the inline data-fetch `<script src>` tags in `index.html`, `pr.html`, `tide.html`, `command-help.html`, `plugins.html` happen to use relative paths (no leading slash). Under a subpath, the absolute URLs break; only the accidentally-relative ones survive. No central frontend URL-builder exists either — call sites in `static/pr/pr.ts`, `static/common/rerun.ts`, `static/common/abort.ts`, `static/spyglass/spyglass.ts`, `static/prow/prow.ts` each construct URLs independently, several via `window.location.origin` + hardcoded absolute paths.
- evidence: `cmd/deck/main.go:314-704`, `cmd/deck/template/base.html:30-65`, `cmd/deck/template/index.html:5`, `cmd/deck/static/pr/pr.ts:113`, `cmd/deck/static/prow/prow.ts:525,663,760`

### [related-code] Deck mux route registration
- where: `cmd/deck/main.go:314-704`
- excerpt: |
    mux.Handle("/static/", http.StripPrefix("/static", staticHandlerFromDir(o.staticFilesLocation))) // main.go:316
    mux.HandleFunc("/", ...)                                                                          // main.go:435
    mux.Handle("/prowjobs.js", ...)                                                                   // main.go:455

### [related-code] Absolute asset/nav paths in base template
- where: `cmd/deck/template/base.html:30-65`
- excerpt: |
    href="/static/style.css" ... href="/pr" ... href="/tide"

### [related-pr] deck: add base path plumbing and helper APIs
- ref: kubernetes-sigs/prow#576
- relevance: Near-complete candidate fix by tsj-30 — adds `--base-path` flag, `cmd/deck/basepath.go` (mux-wrapping middleware, `normalizeBasePath`/`stripBasePath`/`absolutePath`), template `deckPath`/`deckBasePath` exposure via `window.prowBasePath`, and frontend helpers in `cmd/deck/static/common/urls.ts`. Currently OPEN and stalled: CI red (`pull-prow-integration`, `pull-prow-unit-test`, `pull-prow-unit-test-race-detector-nonblocking`, `pull-prow-verify-lint` all FAILURE) since the last push on 2026-04-08, no activity since other than an automated stale-bot comment on 2026-07-08. Maintainer (petr-muller) posted review + `/hold cancel` on 2026-01-24; author (tsj-30) acknowledged all feedback on 2026-02-01 (including adding unit tests for `stripBasePath`/`normalizeBasePath` and narrowing `absolutePath`'s receiver to a custom `basePath` string type) but neither change appears in the current diff (`cmd/deck/basepath.go` has no matching `basepath_test.go`).

## Checked
- Searched `cmd/deck/` and `pkg/deck/` for existing base-path/prefix/subpath config — none found.
- Searched `*_test.go` under `cmd/deck` for base-path/subpath/URL-prefix coverage, before and after PR #576 — none found; no `basepath_test.go` in PR #576's diff despite being requested and agreed to in review.
- Reviewed PR #576's full review/comment history and `statusCheckRollup` (all failing checks dated 2026-04-08, matching the last commit).
- Confirmed canonical repo is `kubernetes-sigs/prow` (`upstream`), current worktree `origin` is `petr-muller/prow`.

## Next steps
- Re-review PR #576's current head commit; CI has been red for ~3.5 months since 2026-04-08 — confirm whether failures are unchanged or new.
- Confirm whether the unit tests for `stripBasePath`/`normalizeBasePath` and the `basePath` custom-type refactor (agreed to by tsj-30 on 2026-02-01) were ever pushed; current diff suggests not.
- Ping @tsj-30 on PR #576 to check if they are still working it given the multi-month gap, or offer to take over/close if abandoned.
- Do not open separate good-first-issue/help-wanted work on #376 itself while PR #576 is active — its disposition is tied to the PR.

## Open questions
- Is @tsj-30 still actively working PR #576, or has it effectively stalled?
- Were the CI failures from 2026-04-08 ever investigated, or do they still block as-is?
