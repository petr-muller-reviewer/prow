---
issue: kubernetes-sigs/prow#376
title: "Cannot run prow at a subpath"
state: closed
labels: kind/bug, lifecycle/rotten, area/deck
main_sha: e601a1ffafd7d8d3a781238a4c5f4233d6248f68
triaged_at: 2026-11-16T22:40:23Z
verdict: accepted
refresh_log:
  - since: 2026-07-27T00:38:17Z
    summary: "Issue became rotten and related PR #576 was closed by stale automation without merging."
  - since: 2026-10-28T00:44:38Z
    summary: "Rot lifecycle closed the issue as NOT_PLANNED (bot-only, no fix); no human comments, cross-references, or label changes, and upstream/main still has no base-path support."
---

## What the issue reports

The reporter cannot run Deck beneath a URL subpath; the existing findings trace this to root-relative routes and URL construction.

Since previous triage:
- On 2026-10-17, the stale bot changed the issue label from `lifecycle/stale` to `lifecycle/rotten`; the issue remains open.
- Related PR #576 was marked rotten on 2026-08-07 and closed by automation on 2026-10-17 without merging. The bot says it needs a rebase; it remains an unmerged candidate for the identified base-path work.
- No new cross-references were added to issue #376.

Since the 2026-10-28 triage:
- On 2026-11-16 at 00:19:58Z, `k8s-triage-robot` commented the standard rot notice ending in `/close not-planned` (issuecomment-6903848855); `kubernetes-prow[bot]` then closed the issue at 2026-11-16T00:20:04Z with `stateReason: NOT_PLANNED`.
- No human comments, no new cross-references, and no label changes since `triaged_at`: labels are still `kind/bug`, `lifecycle/rotten`, `area/deck`, and `closedByPullRequestsReferences` is empty.
- The close is lifecycle automation only — no maintainer decision and no fix. The code-level analysis below is unchanged and still holds on current `upstream/main`.

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
- Re-checked `upstream/main` on 2026-11-16 for `base-path`/`basapath` under `cmd/deck/` and `pkg/deck/` — still no hits, 421 commits after `main_sha` (latest `86b2c4908`).
- Confirmed the only post-`triaged_at` activity on #376 is the rot close: two bot comments, one `closed` timeline event, zero cross-references, zero label events.

## Next steps
- Issue #376 is now closed as `NOT_PLANNED` by the rot bot, not by a maintainer. Decide whether to reopen it (`/reopen`) because the bug is real, or leave it closed and track the base-path work elsewhere — reopening only makes sense with an active owner, since it will go rotten again in ~150 days without activity.
- Base-path support is still unowned: PR #576 is closed unmerged, no newer PR references #376, and `upstream/main` has no base-path code. If someone picks it up, reuse PR #576's approach (flag + mux middleware + `deckPath` template helper + frontend URL helper) and add the `normalizeBasePath`/`stripBasePath` tests that review asked for.
- If the issue stays closed, no further triage cycles are needed; the analysis above applies unchanged whenever a maintainer revives the work.

## Open questions
- Is sig-testing/prow interested in a subpath deployment enough to fund an owner, given the issue was auto-closed for inactivity rather than resolved?
- Would a reopened #376 need a fresh design proposal (e.g. whether base path belongs in Deck config or is derived from `X-Forwarded-Prefix`), or is a rebased PR #576 sufficient?
