---
pr: kubernetes-sigs/prow#748
title: "Fix malformed pagination URL when resp.Request.URL differs from paged Path"
head_sha: 75a124c10beae7c575276b7b757b5910f8f526f5
base: main
reviewed_at: 2026-06-10T17:05:07Z
verdict: request-changes
---

Production fix in `readPaginatedResultsWithValuesWithContext` (pkg/github/client.go:1960) is correct; request-changes is solely about the redirect test not reproducing the bug. A pending GitHub review with these findings (incl. suggestion blocks) was created at review time: pullrequestreview-4469867265.

Bug: GHE prefix was computed as `TrimSuffix(resp.Request.URL.RequestURI(), pagedPath)`. After an HTTP redirect, `resp.Request.URL` is the final URL, suffix doesn't match, `prefix` becomes the whole response URI, and `TrimPrefix` mangles the next Link into e.g. `&page=2`, concatenated as `c.bases[hostIndex]+path` (client.go:981) → `https://api.github.com&page=2` → fatal URL-parse error.

Fix: compare paths only (`resp.Request.URL.Path` vs query-stripped `pagedPath`), plus fallback to full Link `RequestURI()` when the trimmed result doesn't start with `/`. Known accepted limitation (documented in PR body): GHE+redirect falls back to doubled `/api/v3` → 404 instead of crash.

## Findings

### [blocking] Redirect test passes against pre-fix code — not a regression test
- where: `pkg/github/client_test.go:1446-1448`
- concern: Verified empirically: with `git checkout HEAD~1 -- pkg/github/client.go`, both new tests PASS. The old code only breaks when the next Link literally extends the redirected response's RequestURI (GitHub's real behavior: current URL + `&page=N`). The test's Link drops `protected=false`, so the old `TrimPrefix` is a no-op and old code accidentally works. Fix: echo `r.URL.RawQuery`. Verified: with that change the test fails on pre-fix code with `parse "https://127.0.0.1:<port>&page=2": invalid port` and passes with the fix.
- excerpt: |
    w.Header().Set("Link", fmt.Sprintf(
        `<https://%s/repos/new-org/new-repo/branches?per_page=100&page=2>; rel="next"`,
        r.Host))
- suggested: |
    w.Header().Set("Link", fmt.Sprintf(
        `<https://%s/repos/new-org/new-repo/branches?%s&page=2>; rel="next"`,
        r.Host, r.URL.RawQuery))

### [nit] t.Fatalf called from HTTP handler goroutine
- where: `pkg/github/client_test.go:1390` (also 1394, 1454)
- concern: `FailNow` is only valid from the test goroutine; from an httptest handler it can hang or fail to stop the test. Use `t.Errorf` (the redirect test's `default:` branch at 1457-1459 does it right). Pre-existing tests in the file share the flaw — not a blocker.
- excerpt: |
    default:
        t.Fatalf("Unexpected request: %s", r.URL.String())

### [nit] Redirect test lacks request-count assertion
- where: `pkg/github/client_test.go:1484`
- concern: `TestReadPaginatedResultsWithValuesSamePathPagination` asserts `requestCount == 2`; the redirect test could assert 3 (redirect + page1 + page2) to catch looping/short-circuiting.

### [nit] Rewritten comment lost concrete examples
- where: `pkg/github/client.go:1992-2002`
- concern: New comment explains the mechanism but dropped the old worked examples (`<ghe-url>/api/v3`, `api.github.com/repositories/22/...` next-link shape) that showed why the prefix dance exists. Keep one concrete example.

### [question] SamePathPagination test also passes on old code
- where: `pkg/github/client_test.go:1376-1427`
- concern: Fine as a guard for the previously-uncovered same-path+query case (existing `TestReadPaginatedResults` Links carry no query strings), but it is not a reproducer either — don't mistake it for one.

## Checked
- Traced new prefix logic through: github.com same-path, github.com `/repositories/<id>/...` next-link shape, GHE with and without query strings, redirect on plain GitHub, GHE+redirect (falls into documented 404 limitation). All correct.
- Fallback `pagedPath[0] != '/'` cannot misfire on legit GHE links (`/api/v3...` trims to `/...`); fires correctly on redirect-mangled prefixes.
- URL construction is plain string concat `c.bases[hostIndex]+path` (client.go:981) — confirms the reported failure mode.
- Existing pagination tests (`TestReadPaginatedResults` incl. `/api/v3` case) pass with new logic; full `-run TestReadPaginated` green.
- Doubled-prefix 404 (GHE+redirect fallback) additionally burns `max404Retries` with backoff before surfacing — noted, acceptable vs crash.
- Decoded-vs-raw path mismatch (`resp.Request.URL.Path` is decoded, `pagedPath` is raw): checked all `readPaginatedResults*` call sites; none use percent-encoded path segments (escaping only in query params, client.go:3651,5187). Theoretical only; `EscapedPath()` would harden.
- `values.Encode()` sorts keys, so iteration-2+ prefix computation stays consistent.

## Open questions
- None beyond findings; the GHE+redirect limitation is explicitly acknowledged in the PR description and acceptable.
