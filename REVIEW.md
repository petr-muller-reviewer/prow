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
