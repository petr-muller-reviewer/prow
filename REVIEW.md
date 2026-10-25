---
pr: kubernetes-sigs/prow#843
title: "fix(deck): replace gorilla/csrf with net/http.CrossOriginProtection"
head_sha: 7953d2b1d708f6f44bad88ac423223ca64fae42c
base: main
reviewed_at: 2026-08-27T11:07:01Z
verdict: needs-discussion
refresh_log:
  - old_head: 43b33864a72e472ca89f9f038506a9fe97239eea
    new_head: 7953d2b1d708f6f44bad88ac423223ca64fae42c
    summary: "PR merged (squash/rebase-merge to main). Only change vs previous review: csrf.md doc nit (kubectl -> curl example) addressing BenTheElder's inline comment; main_test.go delta is an unrelated apimachinery-bump rebase artifact, not part of this PR's own change. cmd/deck/main.go unchanged."
---

## What this PR does
- Removes the `gorilla/csrf` dependency from deck and replaces it with Go 1.25's stdlib `net/http.CrossOriginProtection`.
- Updates all call sites of `createAbortProwJobIcon`, `createRerunProwJobIcon`, `prepareBaseTemplate`, `renderSpyglass` for the new function signatures (no more `csrfToken`/`X-CSRF-Token` threading).
- Drops the 32-byte cookie-secret validation that was specific to gorilla/csrf's token requirement; the OAuth session-cookie path keeps its own independent validation.
- Removes the now-obsolete `--rerun-creates-job` fatal precondition requiring a 32-byte cookie secret, since CSRF protection is now always-on regardless of cookie-secret config.
- Updates `csrf.md` and `github-oauth-setup.md` docs; cleans `go.mod`/`go.sum` of the `gorilla/csrf` dependency.

Since previous review:
- `site/content/en/docs/components/core/deck/csrf.md`: doc example changed `kubectl` -> `curl` per BenTheElder's review comment (non-browser API client example was misleading since kubectl doesn't talk to deck).
- PR merged 2026-08-26 with approvals from BenTheElder, cblecker (self-approve), michelle192837; `/hold` placed by BenTheElder for the doc nit was cancelled by cblecker after the fix landed.
- The should-fix finding below (no trusted origins / no escape hatch) was not addressed before merge and was not raised by other reviewers.

## Findings

### [should-fix] CrossOriginProtection constructed with no trusted origins and no escape hatch
- where: `cmd/deck/main.go:467`
- concern: `http.NewCrossOriginProtection()` is constructed with no trusted origins configured, and the diff drops the old escape-hatch flag the gorilla/csrf setup had. If deck sits behind a reverse proxy/ingress where the Host header doesn't match the public origin, or is hit by an old client lacking `Sec-Fetch-Site` support with a mismatched `Origin` header, legitimate same-origin POSTs (rerun/abort) will be rejected with 403. There is no flag in this diff to register a trusted origin as a workaround for such deployments.
- excerpt: |
    http.NewCrossOriginProtection()

## Checked
- All call sites of `createAbortProwJobIcon`/`createRerunProwJobIcon`/`prepareBaseTemplate`/`renderSpyglass` updated consistently for the new signatures; no leftover `csrfToken`/`X-CSRF-Token` references anywhere.
- Removed 32-byte cookie-secret validation was specific to gorilla/csrf's token requirement; OAuth session-cookie path's independent validation untouched.
- Removal of `--rerun-creates-job` fatal precondition is correct, not an unsafe removal — CSRF protection is now always-on.
- Docs (`csrf.md`, `github-oauth-setup.md`) updated accurately.
- `go.mod`/`go.sum` cleanly drop `gorilla/csrf` with no dangling references.

## Open questions
- Should `CrossOriginProtection` be configured with `AddTrustedOrigin` for deployments behind a reverse proxy/ingress with a mismatched Host, given the old insecure-style escape hatch was dropped without replacement?
