---
pr: kubernetes-sigs/prow#884
title: "chore(deps): bump github.com/gorilla/sessions from 1.2.0 to 1.4.0"
head_sha: 9a989b9364b237d469412b355f1edf3294fe293e
base: main
reviewed_at: 2026-09-01T16:37:50Z
verdict: request-changes
---

## Verdict

Request changes. The direct dependency bump changes CookieStore defaults on Deck's OAuth cookie path; `--allow-insecure` produces `SameSite=None` cookies without `Secure`, which browsers reject.

## What this PR does

- Updates the direct production module `github.com/gorilla/sessions` from `v1.2.0` to `v1.4.0`.
- Regenerates the corresponding `go.sum` entries.
- Does not modify project source, tests, configuration, or vendored code.
- Pulls in releases that add FilesystemStore path traversal hardening, CookieStore `SameSite=None`/`Secure=true` defaults, and Go 1.23 `Partitioned` cookie support.

## Findings

### [blocking] Preserve OAuth cookies for insecure Deck deployments
- where: `cmd/deck/main.go:598`, `pkg/githuboauth/githuboauth.go:150-165`, `pkg/githuboauth/githuboauth.go:223-294`
- concern: Sessions v1.3 changes `CookieStore` defaults to `SameSite=None` and `Secure=true`. Deck deliberately overrides `Secure` with `!o.allowInsecure`; with `--allow-insecure`, the OAuth state and access-token cookies become `SameSite=None` without `Secure`. Modern browsers reject that combination, so the GitHub OAuth flow cannot retain its state/session on an HTTP Deck deployment. Set an explicit compatible SameSite mode when insecure mode is selected and cover the emitted cookie attributes in a regression test.
- excerpt: |
    secure := !o.allowInsecure
    // ...
    mux.Handle("/github-login", goa.HandleLogin(oauthClient, secure))
    // ...
    oauthSession.Options.Secure = secure

## Checked

- Classification: dep-only; only `go.mod` and `go.sum` changed.
- `github.com/gorilla/sessions v1.4.0` is a tagged release from 2024-08-20, not a pseudo-version; it had 742 days of soak at review time.
- Prow requires Go 1.26.4, satisfying v1.4.0's Go 1.23 requirement.
- The module is imported by three production files: `cmd/deck/main.go`, `pkg/githuboauth/githuboauth.go`, and `pkg/prstatus/prstatus.go`; it stores GitHub OAuth access tokens in signed cookie sessions.
- FilesystemStore path traversal hardening does not affect Prow, which constructs only `sessions.NewCookieStore`.
- GitHub's Go security-advisory query returned no advisories for `github.com/gorilla/sessions`.

## Open questions

- Are HTTP Deck deployments using `--allow-insecure` supported in practice? If so, this must be fixed before taking v1.4.0; if not, document that OAuth requires HTTPS and consider rejecting that flag together with GitHub OAuth.

## Dependency followups

Examined on 2026-10-21 for merged PR #884 (merge commit `2ffd24337da55d480c9ec19c4f11455ad600727c`). 1 accepted, 0 skipped. The maintainer explicitly accepted the OAuth cookie followup.

- Range examined: direct Go module `github.com/gorilla/sessions v1.2.0 → v1.4.0`, including intermediate releases v1.2.1, v1.2.2, and v1.3.0. No deprecated API or improved replacement applies to Prow's CookieStore calls in `cmd/deck/main.go:632`, `pkg/githuboauth/githuboauth.go:155`, or `pkg/prstatus/prstatus.go:190`.
- Transitive dependency check: sessions raises its requirement on `github.com/gorilla/securecookie v1.1.1 → v1.1.2`, but Prow already selected v1.1.2 before this PR. There is no effective transitive version bump; removing the old v1.1.1/go.mod checksum is cleanup.
- FilesystemStore ID encoding, deletion handling, and path hardening do not apply: Prow constructs CookieStore, not FilesystemStore.
- The new `Options.Partitioned` cookie feature has no demonstrated use case in Prow's GitHub OAuth flow. Enabling it speculatively would change cookie scoping rather than simplify an existing call site.
- The SameSite/Secure default interaction remains the existing review finding above and is accepted below as post-merge work. Acceptance does not resolve the finding.
- Sources: [v1.2.1 release](https://github.com/gorilla/sessions/releases/tag/v1.2.1), [v1.2.2 release](https://github.com/gorilla/sessions/releases/tag/v1.2.2), [v1.3.0 release](https://github.com/gorilla/sessions/releases/tag/v1.3.0), [v1.4.0 release](https://github.com/gorilla/sessions/releases/tag/v1.4.0), and [full source comparison](https://github.com/gorilla/sessions/compare/v1.2.0...v1.4.0).

### Preserve OAuth cookies with --allow-insecure

- Status: accepted by maintainer.
- Category: default-change.
- Necessity: must — restore the explicitly supported insecure OAuth mode after the merged dependency bump.
- Mode: deliver.
- Dependency: `github.com/gorilla/sessions v1.2.0 → v1.4.0`; v1.3.0 changed CookieStore defaults to `SameSite=None` and `Secure=true`.
- Where: `cmd/deck/main.go:598`, `cmd/deck/main.go:632`, `pkg/githuboauth/githuboauth.go:150-165`, `pkg/githuboauth/githuboauth.go:285-294`.
- Changelog evidence: v1.3.0 lists “fix no default samesite”; the source comparison adds `SameSite: http.SameSiteNoneMode` and `Secure: true` in `NewCookieStore`.

- Scope: check current main for an existing fix; otherwise make OAuth state and access-token cookie attributes compatible with `--allow-insecure`, preserve HTTPS behavior, and add focused handler regression tests.

```text
In kubernetes-sigs/prow, fix the OAuth cookie regression introduced by PR #884,
"chore(deps): bump github.com/gorilla/sessions from 1.2.0 to 1.4.0", merged as
2ffd24337da55d480c9ec19c4f11455ad600727c. Work from current main containing
that merge, not the review branch. First check whether this is already fixed;
if so, report the fixing commit and coverage instead of duplicating the work.

The v1.3.0 release notes list "fix no default samesite"; NewCookieStore now
defaults to SameSite=None and Secure=true. Deck constructs the store in
cmd/deck/main.go:632 and derives secure := !o.allowInsecure at line 598.
HandleLogin and HandleRedirect in pkg/githuboauth/githuboauth.go set
session.Options.Secure to that value without adjusting SameSite. With
--allow-insecure, the resulting SameSite=None cookies lack Secure and browsers
reject them, preventing OAuth state/session persistence.

Set an explicit SameSite policy compatible with insecure OAuth operation
(normally SameSite=Lax for the top-level GitHub redirect). Cover both the
oauth-session state cookie and access-token-session cookie. Preserve HTTPS
behavior, HttpOnly, and expiry settings; check compatibility with subsequent
session saves in pkg/prstatus/prstatus.go.

Acceptance criteria: focused tests exercise login and successful redirect
handlers with secure=false and secure=true, checking emitted cookie SameSite,
Secure, HttpOnly, and expiry/MaxAge attributes. No insecure cookie may combine
SameSite=None with Secure=false. Run go test ./pkg/githuboauth ./pkg/prstatus;
if Deck initialization changes, also run the relevant Deck tests. Report results.

Scope: cookie policy and focused regression coverage only. Do not enable
Partitioned cookies, redesign OAuth, bump dependencies, or change unrelated
authentication behavior.
```
