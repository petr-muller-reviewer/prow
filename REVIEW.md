---
pr: kubernetes-sigs/prow#530
title: "github: enable dry-run mode when using apps auth"
head_sha: 37d005f253997a71541cce48faaf7dbe8272601c
base: main
reviewed_at: 2026-06-15T12:16:44Z
verdict: approve
refresh_log:
  - from_sha: 37d005f253997a71541cce48faaf7dbe8272601c
    to_sha: 37d005f253997a71541cce48faaf7dbe8272601c
    summary: "No code changes. Title updated to reflect pkg/github scope. Approved by kaovilai (2026-05-27) and petr-muller (2026-06-15)."
---

# Review: kubernetes-sigs/prow#530

## What this PR does

- Introduces `allowInDryRun bool` field on the internal request struct in `pkg/github/client.go`
- Sets `allowInDryRun: true` on the single GitHub Apps token acquisition request (`POST /app/installations/{id}/access_tokens`)
- Enforces a hardcoded allowlist check: logs a security error and skips `allowInDryRun` if the URL does not match the expected Apps token endpoint
- Previously, GitHub Apps auth failed entirely in dry-run mode because all POSTs were blocked; this carve-out allows token acquisition while mutations remain blocked
- Adds 409 test lines including `TestAllowInDryRunEnforcesAllowlist` covering the security boundary

## Findings

### [resolved] Title and description misstate scope of change
- where: `pkg/github/client.go` (shared infrastructure, not Peribolos code)
- concern: The change lives in the shared GitHub client used by every Prow component. Any component running `--dry-run` with GitHub Apps auth is affected, not just Peribolos. The PR title and description do not disclose this. Reviewers and future readers will underestimate the blast radius.
- resolution: PR title updated from "`Peribolos`: enable dry-run mode for GitHub Apps" to "github: enable dry-run mode when using apps auth" reflecting the pkg/github scope.

### [question] Behavioral change for non-Peribolos components in dry-run + GH Apps
- where: `pkg/github/client.go:932-950` (request execution path)
- concern: Before this PR, any Prow component using dry-run + GH Apps would fail immediately at token acquisition. After, they acquire a token and proceed until the first blocked mutation. The code path between successful auth and the first mutation call is untested for components other than Peribolos. Are there observable side effects (API reads, state, metrics, logging) in that window for other components?

### [nit] Allowlist violation does not fail loudly in test/dev builds
- where: `pkg/github/client.go:950`
- concern: Misuse of `allowInDryRun: true` on a non-allowed endpoint logs an error and silently falls back. In production this is safe; in development or test builds a panic would surface mistakes earlier.

### [nit] URL pattern matching is fragile and untested for boundary cases
- where: `pkg/github/client.go` (allowlist check using `strings.Contains` / `strings.HasSuffix`)
- concern: Pattern is not a package-level constant. No test verifies that a near-miss URL (e.g. `/apps/installations/{id}/tokens`) is correctly blocked. If GitHub changes the endpoint path, the allowlist silently stops working.

### [nit] Dry-run condition readability
- where: `pkg/github/client.go:932-934`
- concern: The compound boolean involving `allowInDryRun` and `c.dry` lacks explicit parentheses; intent requires careful reading.

## Checked

- Defense in depth: both `allowInDryRun` flag and hardcoded allowlist check must pass for the carve-out to apply
- `TestAllowInDryRunEnforcesAllowlist` correctly exercises the security boundary
- Backward compatibility: PAT users see no behavioral change; all mutation endpoints remain blocked in dry-run
- Only one call site sets `allowInDryRun: true`; the 96 other request creation sites are unaffected
- No API, configuration schema, or CLI flag changes

## Open questions

- Does the PR description need to state explicitly that this affects all Prow components, not just Peribolos? Other maintainers own those components and may want to know. *(Title now updated to reflect pkg/github scope; description update less critical.)*
- Have components other than Peribolos been tested or considered when running `--dry-run` with GH Apps auth? What happens between successful token acquisition and the first blocked mutation in e.g. Tide, Sinker, or Crier?

## Activity since first review

- 2026-05-27: kaovilai reviewed and commented "Works for me."
- 2026-06-15: petr-muller approved
- 2026-06-15: PR title updated from "`Peribolos`: enable dry-run mode for GitHub Apps" to "github: enable dry-run mode when using apps auth" (addresses scope concern)
- 2026-06-15: petr-muller comment "Let's move forward with this one"
