---
pr: kubernetes-sigs/prow#530
title: "github: enable dry-run mode when using apps auth"
head_sha: 37d005f253997a71541cce48faaf7dbe8272601c
base: main
reviewed_at: 2026-09-28T21:23:22Z
verdict: approve
refresh_log:
  - from_sha: 37d005f253997a71541cce48faaf7dbe8272601c
    to_sha: 37d005f253997a71541cce48faaf7dbe8272601c
    summary: "No code changes. Title updated to reflect pkg/github scope. Approved by kaovilai (2026-05-27) and petr-muller (2026-06-15)."
  - from_sha: 37d005f253997a71541cce48faaf7dbe8272601c
    to_sha: 37d005f253997a71541cce48faaf7dbe8272601c
    summary: "No code changes. PR merged on 2026-06-15; two reviewer replies and two comment-only reviews recorded."
---

# Review: kubernetes-sigs/prow#530

## What this PR does

- Introduces `allowInDryRun bool` field on the internal request struct in `pkg/github/client.go`
- Sets `allowInDryRun: true` on the single GitHub Apps token acquisition request (`POST /app/installations/{id}/access_tokens`)
- Enforces a hardcoded allowlist check: logs a security error and skips `allowInDryRun` if the URL does not match the expected Apps token endpoint
- Previously, GitHub Apps auth failed entirely in dry-run mode because all POSTs were blocked; this carve-out allows token acquisition while mutations remain blocked
- Adds 409 test lines including `TestAllowInDryRunEnforcesAllowlist` covering the security boundary

Since previous review:

- No new commits or diff; the PR was merged at `37d005f253997a71541cce48faaf7dbe8272601c` on 2026-06-15.
- Reviewer discussion closed the panic suggestion as low value, and recorded [#756](https://github.com/kubernetes-sigs/prow/pull/756) as a follow-up for the dry-run condition. The follow-up was merged separately on 2026-06-16.

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
- follow-up discussion: petr-muller replied on 2026-06-15 that a panic would make the logic hard to test and its value is not significant; no change was requested.

### [nit] URL pattern matching is fragile and untested for boundary cases
- where: `pkg/github/client.go` (allowlist check using `strings.Contains` / `strings.HasSuffix`)
- concern: Pattern is not a package-level constant. No test verifies that a near-miss URL (e.g. `/apps/installations/{id}/tokens`) is correctly blocked. If GitHub changes the endpoint path, the allowlist silently stops working.

### [nit] Dry-run condition readability
- where: `pkg/github/client.go:932-934`
- concern: The compound boolean involving `allowInDryRun` and `c.dry` lacks explicit parentheses; intent requires careful reading.
- follow-up: petr-muller filed [#756](https://github.com/kubernetes-sigs/prow/pull/756) to centralize the dry-run bypass logic; it was merged separately on 2026-06-16.

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
- 2026-06-15 12:59 UTC: PR merged at `37d005f253997a71541cce48faaf7dbe8272601c`.
- 2026-06-15 13:52 UTC: petr-muller submitted two comment-only reviews. In inline replies, he declined the panic suggestion as hard to test and low value, and linked follow-up PR #756 for the dry-run condition.
- 2026-06-16 19:21 UTC: follow-up PR #756 merged separately.

## Followups

### Block GraphQL mutations in dry-run mode

- category: safety
- necessity: must — dry-run clients can currently send a GraphQL mutation after Apps authentication obtains an installation token.
- where: `pkg/github/client.go` (`MutateWithGitHubAppsSupport` and the GraphQL Apps transport); `pkg/plugins/transfer-issue/transfer-issue.go` is a concrete caller.

```text
In kubernetes-sigs/prow, following PR #530, "github: enable dry-run mode when using apps auth" (merged as da30d8b3fec2bdb5c5e923f85df17faa04520d6f), prevent Client.MutateWithGitHubAppsSupport from sending GraphQL mutations when the GitHub client is in dry-run mode. The REST dry-run guard in pkg/github/client.go does not cover this GraphQL entry point; Apps authentication can now obtain an installation token in dry-run mode, and pkg/plugins/transfer-issue/transfer-issue.go calls the mutation method.

Add focused tests that show a dry-run GraphQL mutation sends no mutation request with either Apps auth or PAT auth, while read-only GraphQL queries and non-dry-run mutations still work. Keep the change at the shared GitHub client boundary; plugin-specific policy and unrelated GraphQL behavior are out of scope.
```

### Make the GitHub Apps dry-run constructor honor its contract

- category: cleanup
- necessity: should — the exported constructor promises a dry-run client but configures `DryRun: false`.
- where: `pkg/github/client.go` (`NewAppsAuthDryRunClientWithFields`); `pkg/github/client_test.go`.

```text
In kubernetes-sigs/prow, following PR #530, "github: enable dry-run mode when using apps auth" (merged as da30d8b3fec2bdb5c5e923f85df17faa04520d6f), fix NewAppsAuthDryRunClientWithFields in pkg/github/client.go so it actually constructs a dry-run client. The current implementation passes DryRun: false despite its name and comment. Peribolos uses GitHubClient(!confirm), so its behavior is separate from this helper.

Add a focused constructor-level test in pkg/github/client_test.go that proves Apps token acquisition succeeds and an ordinary REST mutation sends no request. Make the test exercise the exported constructor rather than manually setting c.dry. Leave the Peribolos flag path and unrelated constructors out of scope.
```
