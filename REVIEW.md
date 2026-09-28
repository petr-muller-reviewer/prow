---
pr: kubernetes-sigs/prow#682
title: "Migrate webhook validation from HMAC-SHA1 to HMAC-SHA256"
head_sha: 63a2a247cb5f9f5e8eee839006ac89a6d7f7e264
base: main
reviewed_at: 2026-09-28T17:49:21Z
verdict: approve
---

# Review

## Verdict

Approve with suggestions — no blocking findings.

The validator and request header use the SHA-256 format documented by GitHub, and the focused package tests passed. Direct GitHub deliveries with a configured secret remain compatible. Custom senders and relays that provide only the SHA-1 header will receive HTTP 403; operators should account for that when upgrading. The PR is already merged.

## What this PR does

- Computes and validates webhook HMACs with SHA-256 instead of SHA-1.
- Reads `X-Hub-Signature-256` when validating incoming requests.
- Updates the phony sender and webhook test fixtures to use the new header and digest.
- Adds GitHub's published SHA-256 test vector for signature generation.

## Findings

### [should-fix] Document the header requirement for custom webhook senders

- where: `pkg/github/webhooks.go:49-52`
- concern: Requests from custom senders or relays that supply only `X-Hub-Signature` now receive HTTP 403. Document the `X-Hub-Signature-256` requirement for operators using those integrations; direct GitHub deliveries with a configured secret already include the new header.
- excerpt: |
    sig := r.Header.Get("X-Hub-Signature-256")
    if sig == "" {
        responseHTTPError(w, http.StatusForbidden, "403 Forbidden: Missing X-Hub-Signature-256")
        return "", "", nil, false, http.StatusForbidden
    }

### [nit] Lock in rejection of the legacy header with a test

- where: `pkg/github/hmac_test.go:46-101`
- concern: Add an explicit case for a valid `sha1=` signature, and ideally a webhook request carrying only `X-Hub-Signature`. This would record the intentional compatibility boundary in the tests.
- excerpt: |
    func TestValidatePayload(t *testing.T) {
        var testcases = []struct {
            name           string
            payload        string
            sig            string
            tokenGenerator func() []byte
            valid          bool
        }{

## Checked

- Compared the changed header, `sha256=` prefix, and test vector with [GitHub's webhook validation documentation](https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries).
- Confirmed `pkg/hook/server.go` passes the original request headers through to external plugins.
- Searched the checkout for remaining Go references to `X-Hub-Signature`, `sha1=`, `PayloadSignature`, and `ValidatePayload`; found no missed webhook signing or validation call sites.
- Confirmed no configuration, HMAC secret format, dependency, or RBAC change. New phony sends only SHA-256 and therefore cannot target an older Hook binary.
- Ran `go test ./pkg/github ./pkg/hook ./pkg/githubeventserver ./pkg/phony`; all four packages passed or compiled. Standalone external plugin tests were not run.

## Open questions

None.

## Followups

None accepted.
