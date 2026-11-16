---
issue: kubernetes-sigs/prow#376
title: "Cannot run prow at a subpath"
state: open
labels: kind/bug, lifecycle/rotten, area/deck
main_sha: e601a1ffafd7d8d3a781238a4c5f4233d6248f68
triaged_at: 2026-10-28T00:44:38Z
verdict: accepted
refresh_log:
  - since: 2026-07-27T00:38:17Z
    summary: "Issue became rotten and related PR #576 was closed by stale automation without merging."
---

## What the issue reports

The reporter cannot run Deck beneath a URL subpath; the existing findings trace this to root-relative routes and URL construction.

Since previous triage:
- On 2026-10-17, the stale bot changed the issue label from `lifecycle/stale` to `lifecycle/rotten`; the issue remains open.
- Related PR #576 was marked rotten on 2026-08-07 and closed by automation on 2026-10-17 without merging. The bot says it needs a rebase; it remains an unmerged candidate for the identified base-path work.
- No new cross-references were added to issue #376.

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
- relevance: Near-complete candidate fix by tsj-30 — adds `--base-path` flag, `cmd/deck/basepath.go` (mux-wrapping middleware, `normalizeBasePath`/`stripBasePath`/`absolutePath`), template `deckPath`/`deckBasePath` exposure via `window.prowBasePath`, and frontend helpers in `cmd/deck/static/common/urls.ts`. At the previous triage it was open and stalled, with CI failures since the 2026-04-08 push and previously requested tests/type changes still absent from the reviewed diff. Since then it was marked rotten on 2026-08-07 and closed by automation on 2026-10-17 without merging; the close comment says it needs a rebase. The approach still addresses the findings here, but the PR is no longer active.

## Checked
- Searched `cmd/deck/` and `pkg/deck/` for existing base-path/prefix/subpath config — none found.
- Searched `*_test.go` under `cmd/deck` for base-path/subpath/URL-prefix coverage, before and after PR #576 — none found; no `basepath_test.go` in PR #576's diff despite being requested and agreed to in review.
- Reviewed PR #576's full review/comment history and `statusCheckRollup` (all failing checks dated 2026-04-08, matching the last commit).
- Confirmed canonical repo is `kubernetes-sigs/prow` (`upstream`), current worktree `origin` is `petr-muller/prow`.

## Next steps
- Check whether @tsj-30 intends to rebase and reopen PR #576; otherwise identify a new owner for the base-path fix.
- If the PR resumes, review its rebased diff and rerun the relevant CI, including coverage for `normalizeBasePath` and `stripBasePath`.
- Issue #376 remains open but is now labeled `lifecycle/rotten`; keep its disposition tied to whether the base-path work resumes or is taken over.

## Open questions
- Will @tsj-30 reopen and rebase PR #576, or should someone else take over?
- After rebasing, are the previously failing CI checks and requested base-path tests addressed?
