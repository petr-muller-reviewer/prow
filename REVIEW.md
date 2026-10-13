---
pr: kubernetes-sigs/prow#862
title: "fix(githuboauth): fall back to fresh session on undecodable cookie"
head_sha: 92a43670c34520975735e794c651e1ce7114ca35
base: main
reviewed_at: 2026-10-13T12:47:09Z
verdict: approve
refresh_log:
  - from: 0be13d0bf259ef5239d32f6701cad0815074eff3
    to: 92a43670c34520975735e794c651e1ce7114ca35
    summary: "Author addressed review-comment thread on line 160/185: added explanatory comments at the three fallback sites, and made GetLogin return the original decode error again instead of falling through."
gate:
  decision: merge
  gated_at: 2026-10-13T12:49:46Z
  gated_head_sha: 92a43670c34520975735e794c651e1ce7114ca35
  reviewed_head_sha: 92a43670c34520975735e794c651e1ce7114ca35
---

# Review

## Gate

Merge — prior substantive feedback is addressed, and the stale-cookie recovery introduces no unacceptable deployment risk.

Gated PR #862 at `92a43670c34520975735e794c651e1ce7114ca35`, unchanged from the saved review; the PR remains open. Read all submitted reviews, inline comment threads, and issue comments. Both GitHub reviews are COMMENTED; neither requests changes. The two substantive threads are addressed in the current code. The local review has no active findings, and its reassessed store-misconfiguration concern is acceptably dispositioned because cookie-save errors still return HTTP 500. The existing successful package test result applies to this unchanged head.

Findings disposition: no unaddressed or borderline substantive concerns. Direct stale-cookie logout test coverage remains an optional improvement, not a merge condition.

Gating list:

- Addressed: @petr-muller's request to explain continuing after decode errors is documented at `pkg/githuboauth/githuboauth.go:159,202,290` ([review thread](https://github.com/kubernetes-sigs/prow/pull/862#discussion_r3815055144)).
- Addressed: @petr-muller's concern about replacing `GetLogin`'s decode error with a generic missing-token error is resolved by the early return at `pkg/githuboauth/githuboauth.go:183`; @psalajova acknowledged the concern on 2026-08-20 ([review thread](https://github.com/kubernetes-sigs/prow/pull/862#discussion_r3815078074)).
- Acceptably dispositioned: the historical `REVIEW.md` store-misconfiguration concern is not a blocker. Save failures still invoke `serverErrorAndPrint` at `pkg/githuboauth/githuboauth.go:166,208,296`, and Deck rejects an empty signing secret at `cmd/deck/main.go:629-630`.
- Security boundary preserved: callback state-cookie decoding still fails closed at `pkg/githuboauth/githuboauth.go:245-248`; state validation still precedes token exchange. No blockers remain.

Independent merge risk:

- API/configuration compatibility: the complete PR touches only the OAuth implementation and its tests (183 insertions, 6 deletions). No exported signature, configuration field, flag, default, Kubernetes schema, or dependency changes; no coordinated rollout or migration is required.
- Behavior: deployments using these GitHub OAuth handlers automatically recover from stale client cookies during login, token replacement, and logout instead of returning HTTP 500 for the decode error. Valid sessions and cookie wire formats remain compatible. Callback state validation and `GetLogin`'s authentication failure behavior remain unchanged. This is the intended fix, documented in the PR description, and is not opt-in.
- Operations: stale-cookie failures now emit warnings at `pkg/githuboauth/githuboauth.go:160,182,203,291`. Operators may see warnings after signing-key rotation while stale-cookie HTTP 500s decrease; persistent save failures still emit errors and return HTTP 500. An operator-facing release note would be useful, but is not required to merge.
- Assessment: low risk. No relevant compatibility-specific repository skill was available in the session catalog; the API, configuration, and behavior pass was performed directly.


## Verdict

Approve. No actionable defects found.

Reviewed the complete PR diff against merge base `e78b8229bbcd26293b8303fc20fe0c054e903a30`, the surrounding handlers, Deck's callers, and the pinned session/cookie implementations. The fallback allows stale cookies to be replaced without weakening callback state validation or suppressing cookie-save failures. Package tests passed. Direct coverage of stale-cookie logout remains a test gap.

## What this PR does

- Lets `HandleLogin` replace an undecodable `oauth-session` cookie and begin a new OAuth flow.
- Lets `HandleRedirect` replace an undecodable `access-token-session` cookie after validating OAuth state and exchanging the authorization code.
- Lets `HandleLogout` expire an undecodable token cookie; adds warning logs for decode failures, including `GetLogin`, which still returns the original error.
- Adds three regression tests covering stale token replacement, login with a stale state cookie, and rejection of a stale state cookie on callback.

The PR changes two files: `pkg/githuboauth/githuboauth.go` and `pkg/githuboauth/githuboauth_test.go`, with 183 insertions and 6 deletions. This review covers PR head `92a43670c34520975735e794c651e1ce7114ca35`; checkout HEAD `d034cf6e07f1ba7c73262d5a3cf2ab99a107460f` adds only the previously saved review artifacts.

## Findings

No findings.

## Resolved

### [should-fix] Decode-failure fallback conflates benign key rotation with persistent misconfiguration — reassessed 2026-10-13

- where: `pkg/githuboauth/githuboauth.go:158-168,201-212,289-301`
- resolution: the earlier claim that persistent store misconfiguration degrades silently is not supported by these paths. All three fallback handlers still check `Save` and call `serverErrorAndPrint` on encoding failure. In securecookie v1.1.2, missing signing keys or a stored codec initialization error also make encoding fail. Deck rejects an empty signing secret before constructing the store. `GetLogin` still returns its decode error. No introduced defect was demonstrated.

### [nit] Fallback pattern duplicated across 4 call sites instead of centralized — reassessed 2026-10-13

- where: `pkg/githuboauth/githuboauth.go:158-161,181-184,201-204,289-292`
- resolution: there are three fallback handlers; `GetLogin` only logs and returns. A helper is an optional refactoring, and the current explicit branches do not introduce a meaningful maintainability defect.

### [nit] Log message wording inconsistent across the four sites — reassessed 2026-10-13

- where: `pkg/githuboauth/githuboauth.go:160,182,203,291`
- resolution: the messages distinguish new-session creation, login lookup failure, and logout. The prior claim that all four paths continue with an empty session was incorrect because `GetLogin` returns early. No actionable defect remains.

### [nit] Test-fixture duplication across 3 new tests — reassessed 2026-10-13

- where: `pkg/githuboauth/githuboauth_test.go:373-548`
- resolution: a fixture helper could reduce repetition, but the duplicated setup does not obscure these tests or demonstrate a regression. Retained as historical context rather than an active finding.

### [question] Alerting impact of removing the 500 path — reassessed 2026-10-13

- where: `pkg/githuboauth/githuboauth.go:158-168,201-212,289-301`
- resolution: stale client cookies now produce warnings and recover, which is the intended behavior. Cookie-save failures still produce error logs and HTTP 500. No affected alert was identified, so this is not an actionable finding.

### [nit] Undocumented reliance on gorilla/sessions non-nil-session invariant — resolved 2026-08-25
- where: `pkg/githuboauth/githuboauth.go:159,181,203,290` (line numbers shifted after the fix)
- resolution: author added a one-line comment at each of the three remaining fallback sites (`HandleLogin`, `HandleLogout`, `HandleRedirect`) documenting that `New`/`Get` always return a usable session and a decode error just means the existing cookie is invalid, e.g. `// New always returns a session; a decode error only means the existing cookie is invalid (e.g. stale signing key). Continue with this empty session so we can overwrite it.` Prompted by petr-muller's review comment on line 160.

### [nit] GetLogin's differing observable behavior is untested — resolved 2026-08-25
- where: `pkg/githuboauth/githuboauth.go:178-187`
- resolution: not just tested — the underlying behavior difference is gone. `GetLogin` now does `if err != nil { ...; return "", err }`, restoring the exact pre-PR early return. Prompted by petr-muller's review comment noting the type assertion would fail on an empty session anyway, just surfacing a more generic error; author agreed. `GetLogin`'s read path never touches `session.Values` after a decode error again, closing the corresponding [question] below.

### [question] Downstream string-matching on GetLogin's error text — resolved 2026-08-25
- where: `pkg/githuboauth/githuboauth.go:178-187`
- resolution: moot — `GetLogin` returns the original securecookie decode error again on a decode failure, identical to pre-PR, so no downstream string changes.

## Checked

- gorilla/sessions v1.2.0 `CookieStore.New` creates a non-nil session, initializes its values and copies store options before decoding. `CookieStore.Get` returns that session through the registry for the constant, valid cookie names used here.
- gorilla/securecookie v1.1.2 verifies the cookie MAC before deserializing values, so a cookie signed with the old key cannot supply trusted session contents. The login and redirect paths explicitly set the new state or token before saving.
- `HandleRedirect` still returns immediately when the OAuth state cookie cannot be decoded, when its state value is missing, or when state comparison fails. The new stale-state callback test expects HTTP 500.
- `GetLogin` still returns its decode error before reading token values or consulting the user identifier. Deck's `isAllowedToRerun` maps that error to HTTP 401.
- The three fallback handlers still check `Save` and return HTTP 500 on failure. The fallback does not hide encoding failures.
- `go test -mod=readonly ./pkg/githuboauth` passed on the reviewed source tree.
- Earlier observations about missing returns in unrelated redirect-error paths are pre-existing and outside this PR.

## Open questions

- Could you add a stale-cookie logout test that checks the token cookie is expired and the handler redirects? Existing logout tests do not exercise the newly changed decode-error branch; this is non-blocking.

## Followups

PR #862 is still open at `92a43670c34520975735e794c651e1ce7114ca35`; there is no merge commit yet. All three candidates were accepted by the maintainer. Implement these followups against the default branch after the PR lands.

Accepted: 3. Skipped: 0.

### Cover remaining stale-cookie recovery paths

- category: tests
- necessity: should
- why followup: The changed logout error branch lacks direct coverage, and stale-cookie login currently asserts only HTTP 302. This extends regression coverage without changing the approved behavior.

```text
In kubernetes-sigs/prow, following PR #862 — "fix(githuboauth): fall back to fresh session on undecodable cookie" — extend recovery-path tests in pkg/githuboauth/githuboauth_test.go. The PR was still open when this followup was recorded, at head 92a43670c34520975735e794c651e1ce7114ca35; implement against the default branch after it lands.

Add tests using real gorilla CookieStore instances and the existing OAuth mocks:
- Supply a token-session cookie signed with an old key to HandleLogout. Assert HTTP 302 and that the returned access-token-session cookie expires the browser cookie. Include the existing login cookie and assert its expiration as well.
- Strengthen TestHandleLoginWithStaleCookie: decode the replacement oauth-session cookie with the current signing key, validate its state token, and assert that its state matches the state parameter in the authorization redirect.
- Exercise a deterministic CookieStore encoding failure after entering a decode-error fallback. Assert HTTP 500 and no successful redirect, preserving the distinction between recoverable stale cookies and failure to save the replacement session.

Acceptance criteria: these assertions exercise the intended error paths, existing tests retain their coverage, and `go test -mod=readonly ./pkg/githuboauth` passes.

Scope: tests only; preserve production behavior and dependencies. Do not add unrelated OAuth changes or a new testing framework.
```

### Stop login after redirect URL parsing fails

- category: bug fix
- necessity: should
- why followup: A neighbouring pre-existing error path writes HTTP 500 but falls through to a nil-client call. It is separate from the stale-cookie change.

```text
In kubernetes-sigs/prow, following PR #862 — "fix(githuboauth): fall back to fresh session on undecodable cookie" — fix the pre-existing error fallthrough in Agent.HandleLogin in pkg/githuboauth/githuboauth.go:169-173. The PR was still open when this followup was recorded, at head 92a43670c34520975735e794c651e1ce7114ca35; implement against the default branch after it lands.

When client.WithFinalRedirectURL(destPage) returns an error, HandleLogin calls serverErrorAndPrint but continues to newClient.AuthCodeURL. The concrete OAuth client returns nil on redirect URL parsing failure, so this causes a panic. Deck's configuration check only requires a nonempty redirect URL.

Return immediately after reporting the WithFinalRedirectURL error. Add a regression test in pkg/githuboauth/githuboauth_test.go using a malformed configured redirect URL or an OAuthClient that returns nil and an error. Assert HTTP 500, no panic, and no authorization redirect; if using a mock, ensure AuthCodeURL is never called.

Acceptance criteria: the regression test exercises the original failure path, successful login behavior remains covered, and `go test -mod=readonly ./pkg/githuboauth` passes.

Scope: this HandleLogin error branch and its test only. Preserve configuration validation, other handlers, dependencies, and successful OAuth behavior.
```

### Share the stale-cookie test fixture

- category: cleanup
- necessity: could
- why followup: Three tests repeat store setup, cookie serialization, and extraction. A small helper will make the accepted coverage extension easier to maintain.

```text
In kubernetes-sigs/prow, following PR #862 — "fix(githuboauth): fall back to fresh session on undecodable cookie" — extract the repeated stale-cookie fixture setup in pkg/githuboauth/githuboauth_test.go:373-548. The PR was still open when this followup was recorded, at head 92a43670c34520975735e794c651e1ce7114ca35; implement against the default branch after it lands.

Add a private test helper that calls t.Helper(), constructs stores with distinct old/current signing keys, creates a session with a supplied cookie name and values, saves it using the old store, checks all setup errors, and returns the resulting cookie and the current-key store needed by the caller.

Use it in TestHandleRedirectWithStaleCookie, TestHandleLoginWithStaleCookie, and TestHandleRedirectWithStaleOAuthSessionCookie. Reuse it in any related recovery tests already present, but do not require another followup to be implemented first. Keep handler setup and scenario assertions explicit in each test.

Acceptance criteria: the helper produces a real old-key cookie that cannot be decoded by the current store; all existing assertions, including replacement-cookie decoding and callback state rejection, remain intact; and `go test -mod=readonly ./pkg/githuboauth` passes.

Scope: test fixtures only. Preserve production code and dependencies, and avoid a general fixture framework.
```
