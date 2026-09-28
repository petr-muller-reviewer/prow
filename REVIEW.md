---
pr: kubernetes-sigs/prow#763
title: "log more info about failing webhooks"
head_sha: c1f22a6dd39c5186a7196b319430ead41b0a526e
base: main
reviewed_at: 2026-06-22T14:35:49Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-06-22T14:35:49Z
  gated_head_sha: c1f22a6dd39c5186a7196b319430ead41b0a526e
  reviewed_head_sha: c1f22a6dd39c5186a7196b319430ead41b0a526e
---

## Gate

**Decision: merge**

The PR is correct, all CI checks pass, and the one material risk (exported `ValidatePayload` signature change) does not meet the bar for blocking. @Prucek approved with no findings. The missing `approved` label is a process gate (needs `/approve` from a `pkg/OWNERS` approver, suggested: @droslean), not a code concern.

- (process) Missing `approved` label.
- (suggestion, non-blocking) PR description should acknowledge the `ValidatePayload` API signature change.

## What this PR does

- Changes `ValidatePayload` return from `bool` to `(bool, string)`, adding a reason string for each failure path.
- Surfaces the reason in the HTTP 403 response via `ValidateWebhook`, e.g. `"403 Forbidden: Invalid X-Hub-Signature-256 - misconfigured webhook secret from organization/repo: foobar/foobar"`.
- Updates a stale GitHub docs URL in `types.go`.
- Updates tests to destructure the new return value.

## Findings

### [should-fix] Exported API signature change not acknowledged
- where: `pkg/github/hmac.go:42`
- concern: `ValidatePayload` is exported. Changing from `bool` to `(bool, string)` is a compile-time break for any external importer calling this function. The blast radius is small (`ValidateWebhook` is the common entry point and is unchanged), but the PR description should acknowledge the API change.
- excerpt: |
    func ValidatePayload(payload []byte, sig string, tokenGenerator func() []byte) (bool, string) {

### [should-fix] No test assertions on message return values
- where: `pkg/github/hmac_test.go:97-102`
- concern: Tests destructure `(valid, msg)` but only assert on `valid`. The diagnostic strings (the whole point of the PR) are untested. Adding an `expectedMsg` field to the table-driven tests would prevent silent regressions. No test case exercises the `"misconfigured webhook secret"` path (valid JSON, valid-format signature, wrong HMAC key).
- excerpt: |
    valid, msg := ValidatePayload([]byte(tc.payload), tc.sig, tc.tokenGenerator)
    if valid != tc.valid {
        t.Errorf("Wrong validation for the test %q: expected %t but got %t, message: %s", tc.name, tc.valid, valid, msg)
    }

### [nit] Error message wording slightly misleading
- where: `pkg/github/hmac.go:78`
- concern: `"misconfigured webhook secret from organization/repo: "` implies the webhook secret is misconfigured, but the mismatch could also mean the webhook points at the wrong Prow instance or the HMAC config on the Prow side is stale. `"webhook signature mismatch for organization/repo: "` would be more neutral.
- excerpt: |
    return false, "misconfigured webhook secret from organization/repo: " + orgRepo

### [nit] Diagnostic message in HTTP body vs server-side logging
- where: `pkg/github/webhooks.go:67`
- concern: The detailed message (including attacker-controlled `orgRepo` from the unauthenticated payload) is returned in the HTTP response body. `responseHTTPError` logs at Debug level, which is often disabled in production. Logging at Warning/Info and keeping the HTTP body generic would be slightly cleaner, though the current approach is not unsafe (`text/plain`, no injection vector, reflected data is what the sender provided).
- excerpt: |
    responseHTTPError(w, http.StatusForbidden, "403 Forbidden: Invalid X-Hub-Signature-256 - "+message)

## Checked

- All in-tree callers of `ValidatePayload` updated (only `webhooks.go`; five callers of `ValidateWebhook` are unaffected).
- No injection risk from `orgRepo` in HTTP response (`http.Error` sets `text/plain`).
- HTTP status code (403) unchanged; response prefix `"403 Forbidden: Invalid X-Hub-Signature-256"` preserved for tooling compatibility.
- No config, flag, or schema changes. Zero operator action required for upgrade.
- CI passes: unit tests, lint, integration, race detector, image build, CLA.
- `types.go` URL fix verified: old URL redirects to the new one.

## Open questions

- Should the PR description mention the `ValidatePayload` signature change for downstream consumers?
- Would you consider adding a test case with a valid-format signature but wrong HMAC key, asserting on the `"misconfigured webhook secret"` message?
