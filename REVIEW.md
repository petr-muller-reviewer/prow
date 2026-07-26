---
pr: kubernetes-sigs/prow#756
title: "github: centralize dry-run bypass logic in isDryRunAllowed"
head_sha: 3a7df92a71e2cef2f3224cf0e8b1ffc3afbb6e8a
base: main
reviewed_at: 2026-07-26T23:34:54Z
verdict: approve
state: merged
refresh_log:
  - from: 31c8ed5d879e752dc84954a7002135fdade69f00
    to: 3a7df92a71e2cef2f3224cf0e8b1ffc3afbb6e8a
    summary: "Author removed the stale comment in requestRawWithContext (1 line deletion), addressing the nit."
  - from: 3a7df92a71e2cef2f3224cf0e8b1ffc3afbb6e8a
    to: 3a7df92a71e2cef2f3224cf0e8b1ffc3afbb6e8a
    summary: "No code changes. droslean approved 2026-06-16; PR merged."
---

## What this PR does

- Moves the `r.method == http.MethodGet` check from the inline condition in `requestRawWithContext` into `isDryRunAllowed`, making it the single source of truth for dry-run bypass decisions.
- Removes a stale `c.dry` guard from `IsAppInstalled` that was overly conservative — the function uses GET, which was never blocked by `requestRawWithContext`.
- Adds a dry-run subtest to `TestIsAppInstalled` verifying GET requests reach the server in dry-run mode.

Since previous review: author pushed one commit (`3a7df92`) removing the stale comment in `requestRawWithContext` that the nit identified. Since then, no further code changes; `droslean` approved the PR (2026-06-16), and it has since merged.

## Findings

No open findings.

## Resolved

### [nit] Stale comment in requestRawWithContext
- where: `pkg/github/client.go:967`
- concern: The comment said "block all non-GET requests except for explicitly allowed read-only operations" but the GET-vs-non-GET distinction now lives inside `isDryRunAllowed`. The comment was removed in `3a7df92`.
- resolution: Author deleted the comment entirely.

## Checked

- Behavioral equivalence of old vs new code for all request method / `allowInDryRun` / allowlist combinations: GET (allowed before and after), non-GET with `allowInDryRun=false` (blocked), non-GET matching allowlist (allowed), non-GET with `allowInDryRun=true` but not matching allowlist (logged + blocked).
- `IsAppInstalled` at line 5134 constructs a `http.MethodGet` request, confirming the removed guard was redundant.
- New dry-run test reuses existing test cases and test server, verifying requests actually reach the server (not short-circuited).
- `isDryRunAllowed` comment updated to reflect the new GET-first logic.

## Open questions

- None.
