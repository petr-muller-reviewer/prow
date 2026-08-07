---
pr: kubernetes-sigs/prow#576
title: "deck: add base path plumbing and helper APIs"
head_sha: 61dfbb14a1a2ec5bd1de73b5b1ab81cdb04d345b
base: main
reviewed_at: 2026-07-26T23:44:43Z
verdict: request-changes
---

## What this PR does
- Adds a `--base-path` flag to Deck (`cmd/deck/main.go`) so it can be served from a sub-URL.
- New `cmd/deck/basepath.go`: `normalizeBasePath`, `withBasePath` (http.Handler wrapper stripping/rejecting by prefix), `options.absolutePath`/`absolutePathIfRelative`.
- Rewrites a subset of Spyglass/job-history/PR-history link fields to be base-aware via `absolutePathIfRelative`.
- Extracts the HTTP->HTTPS redirect out of `prodOnlyMain` into standalone `withHTTPRedirect`, changing middleware order.
- Adds `deckBasePath`/`deckPath` template funcs, applied only in `base.html`, which also calls a third func `deckPathIfRelative` that is never registered.
- New TS exports in `cmd/deck/static/common/urls.ts`: `deckBasePath`, `deckURL`, `absoluteDeckURL` — not called from any existing frontend code.

## Findings

### [blocking] base.html calls deckPathIfRelative, which is never registered in the template FuncMap
- where: `cmd/deck/template/base.html:49`, `cmd/deck/templates.go:74-81` (`prepareBaseTemplate` FuncMap)
- concern: `base.html` renders `{{deckPathIfRelative (or branding.Logo $defaultLogo)}}`, but the `FuncMap` passed to `template.New(...).Funcs(...)` only defines `deckBasePath` and `deckPath` — not `deckPathIfRelative`. Go's `html/template` fails at `ParseFiles`/parse time when a template references an undefined function, and `base.html` is parsed on every call to `prepareBaseTemplate`, which every Deck page render goes through. This is not gated behind `--base-path` — it breaks page rendering for every installation, including ones that never set the new flag. This is a hard merge blocker, more severe than anything else in this review.
- excerpt: |
    <a href="{{deckPath "/"}}"
       class="logo"><img src="{{deckPathIfRelative (or branding.Logo $defaultLogo)}}?v={{deckVersion}}" ...
    // templates.go FuncMap:
    "deckBasePath":     func() string { return o.basePath },
    "deckPath":         func(p string) string { return o.absolutePath(p) },
    // no "deckPathIfRelative" entry

### [blocking] New templates_test.go assertions do not match what the referenced templates emit
- where: `cmd/deck/templates_test.go:47-71` vs `cmd/deck/template/index.html:4`
- concern: The test asserts the rendered body contains `href="/prow/static/prow_bundle.min.js?v="`-style base-prefixed paths, but `index.html` (unmodified by this PR) hardcodes `<script src="/static/prow_bundle.min.js?v={{deckVersion}}">` and a bare relative `prowjobs.js?...` with no leading slash — neither goes through `deckPath`. Combined with the FuncMap bug above, this test cannot pass as written; treat any reported "green CI" on this PR with suspicion until re-verified.
- excerpt: |
    expectedSnippets := []string{
        `window.prowBasePath = "\/prow";`,
        `href="/prow/static/style.css?v=`,
        ...
        `src="/prow/static/prow_bundle.min.js?v=`,

### [blocking] Only base.html is base-path-aware; every other template still hardcodes root paths
- where: `cmd/deck/template/command-help.html:3-4`, `cmd/deck/template/configured-jobs.html:3-4,10,26`, `cmd/deck/template/configured-jobs-index.html:3,10,14`, `cmd/deck/template/tide.html:4-5`, `cmd/deck/template/tide-history.html:4`, `cmd/deck/template/spyglass.html:13-14,45`, `cmd/deck/template/index.html:4`, `cmd/deck/template/plugins.html:4,6`, `cmd/deck/template/pr.html:3-5,78`, `cmd/deck/template/job-history.html:3`, `cmd/deck/template/github-login.html:12`
- concern: These templates still emit `href="/static/..."`, `src="/static/..."`, and internal links like `href="/configured-jobs/{{.Org.Name}}"` without going through `deckPath`. Since `withBasePath` 404s any request not prefixed with the configured base path, every one of these page-specific assets/links breaks as soon as `--base-path` is set to anything other than `/`. Only the shared chrome in `base.html` was migrated — and even that is currently broken by the FuncMap bug above.
- excerpt: |
    <link rel="stylesheet" href="/static/dialog-polyfill.css?v={{deckVersion}}">
    <script type="text/javascript" src="/static/configured_jobs_bundle.min.js?v={{deckVersion}}"></script>
    <a href="/configured-jobs/{{.Org.Name}}/{{.Name}}">{{.Org.Name}}/{{.Name}}</a>

### [blocking] Frontend API calls bypass the new deckURL helper entirely
- where: `cmd/deck/static/pr/pr.ts:113`, `cmd/deck/static/common/rerun.ts:27`, `cmd/deck/static/common/abort.ts:5`
- concern: `deckURL`/`deckBasePath` are added to `urls.ts` but never called from any other file. `/pr-data.js`, `/rerun`, and `/abort` are still built as absolute root paths, so under a configured base path these requests will be rejected (404) by `withBasePath`. This contradicts the PR description's claim that "APIs" are made base-path-safe.
- excerpt: |
    const url = "/pr-data.js";
    return `${location.protocol}//${location.host}/rerun?mode=${mode}&prowjob=${prowjob}`;
    const url = `${location.protocol}//${location.host}/abort?prowjob=${prowjob}`;

### [should-fix] withBasePath's RawPath handling can corrupt percent-encoded paths
- where: `cmd/deck/basepath.go`, `withBasePath` closure (approx lines 49-61)
- concern: After stripping the base path prefix, `r.URL.RawPath` is set equal to the *unescaped* trimmed `Path` rather than being trimmed independently from the original `RawPath`. If the remainder of the original path contained percent-encoded characters (e.g. a Spyglass artifact key with an encoded `%2F`), this collapses the encoding, and `EscapedPath()` will subsequently return a mis-escaped path to anything downstream that relies on it (e.g. `handleRemoteLens`'s reverse-proxy path, GCS object fetches). stdlib's `http.StripPrefix` trims `RawPath` independently for exactly this reason; mirror that instead of deriving it from `trimmed`.
- excerpt: |
    r.URL.Path = trimmed
    if r.URL.RawPath != "" {
        r.URL.RawPath = trimmed
    }

### [should-fix] HTTP->HTTPS redirect middleware reordered as unrelated, undocumented scope creep
- where: `cmd/deck/main.go:507-519` (new handler chain), `withHTTPRedirect`
- concern: Previously `redirectMux` wrapped the raw mux inside `prodOnlyMain`, so a plain-HTTP request was redirected before reaching `traceHandler`/CSRF. Now the chain is `mux -> withBasePath -> traceHandler -> CSRF -> withHTTPRedirect`, making the redirect check outermost. Plausibly an improvement (CSRF cookie logic no longer runs on requests about to be redirected), but it also means 301 redirects for `--redirect-http-to` deployments are no longer observed by `traceHandler`, so they'll disappear from `httpRequestDuration`/`httpResponseSize` metrics — a quiet dashboard/alerting shift. This rides along with an otherwise-unrelated refactor and isn't called out in the PR description or a code comment.
- excerpt: |
    var handler http.Handler = mux
    handler = withBasePath(o.basePath, handler)
    handler = traceHandler(handler)
    if csrfToken != nil {
        handler = CSRF(handler)
    }
    if o.redirectHTTPTo != "" {
        handler = withHTTPRedirect(handler, o.redirectHTTPTo)
    }

### [should-fix] No unit tests for basepath.go's core logic
- where: `cmd/deck/basepath.go` (whole file — `normalizeBasePath`, `stripBasePath`, `absolutePath`, `absolutePathIfRelative`)
- concern: This is the most intricate, edge-case-prone new code in the PR (trailing slashes, query/fragment splitting, prefix-vs-exact matching, relative-vs-absolute detection), yet there is no `basepath_test.go`. The one indirect test (`templates_test.go`) exercises template rendering, not this logic, and currently doesn't pass (see above). A table-driven test would have caught the `RawPath` issue.

### [should-fix] traceHandler observes the unstripped path under a configured base path
- where: `cmd/deck/main.go:507-511` (handler chain ordering: `traceHandler` wraps `withBasePath`)
- concern: `traceHandler`'s route simplifier was authored against unprefixed paths (e.g. `/static`, `/tide`). Since `traceHandler` runs outside `withBasePath`, it sees paths like `/prow/static/...` under a non-root base path, which will likely fall into a catch-all bucket instead of the expected route label — degrading per-endpoint metrics/tracing specifically for base-path deployments.

### [nit] Leftover tooling artifact comment
- where: `cmd/deck/templates_test.go` (end of file)
- concern: Trailing `// Made with Bob` comment looks like leftover output from an AI coding tool and should be removed before merge.
- excerpt: |
    // Made with Bob

### [nit] absolutePath("/") returns basePath without a trailing slash
- where: `cmd/deck/basepath.go`, `absolutePath`
- concern: `o.absolutePath("/")` returns `o.basePath` (e.g. `/prow`), not `/prow/`. This round-trips correctly with `stripBasePath` (exact match on `basePath` maps to `/`), but worth confirming it's intentional since some proxies/browsers can treat `/prow` vs `/prow/` differently for relative-URL resolution.
- excerpt: |
    if pathPart == "/" {
        return o.basePath + suffix
    }

### [nit] absolutePathIfRelative treats protocol-relative strings as relative
- where: `cmd/deck/basepath.go`, `absolutePathIfRelative`
- concern: Any string starting with `/` is treated as "relative" and prefixed, including protocol-relative strings like `//evil.com/x`, which would become `basePath//evil.com/x`. Currently harmless since inputs come from internal job/artifact data, not attacker-controlled redirect targets, but worth a guard if these helpers are ever reused for user-influenced values.

### [question] Was the deckPathIfRelative FuncMap gap caught by CI, or is CI actually red?
- concern: Given `base.html` cannot parse without `deckPathIfRelative` registered, either CI is failing (consistent with independent reports of red `pull-prow-unit-test`/`pull-prow-integration`/`pull-prow-verify-lint` on this PR) or there's a registration path this review missed. Needs a direct check of the PR's CI status before assuming the rest of the analysis is moot.

## Checked
- `normalizeBasePath` correctly forces a leading slash and runs through `path.Clean`, preventing `..`-based prefix escape.
- `csrf.Path(o.basePath)` correctly scopes the CSRF cookie to the base path instead of hardcoding `/`.
- `withBasePath` fails closed (404) on any request path not matching the configured prefix, rather than falling through to unprefixed routes.
- `stripBasePath` prefix-matching correctly requires `basePath + "/"` (or exact match) so a sibling path like `/prowfoo` under base path `/prow` is not falsely accepted.
- `--base-path` defaults to `/` in both `gatherOptions` and `normalizeBasePath`, and `withBasePath` is a true no-op when basePath is root — non-adopters should see no behavior change once the FuncMap bug is fixed.
- License headers on new files match repo convention (Apache 2026 boilerplate).

## Open questions
- Is CI actually green or red on this PR? The `deckPathIfRelative` gap should make `base.html` fail to parse — please confirm you've re-run `go test ./cmd/deck/...` and manually loaded a page after the latest commit.
- Was the HTTP->HTTPS redirect reordering (now outermost, ahead of CSRF/tracing) deliberate, or an incidental side effect of the refactor?
- Given `deckURL`/`deckBasePath` exist but are unused, is a follow-up planned to wire them into `pr.ts`, `rerun.ts`, `abort.ts`, `spyglass.ts`, and the other templates — or should this PR be expanded/rescoped to cover them before merge, and its description corrected to not claim full API/asset coverage?
- Is there a plan to add a table-driven test that renders every template (not just `index.html`) with a non-root base path and asserts no raw `/static` or root-relative hrefs remain?
