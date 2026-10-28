---
pr: kubernetes-sigs/prow#102
title: "Update trusted author handling for DCO"
head_sha: 6a28e9c4eb2e0604e1684d16237165b4efaf3553
base: main
reviewed_at: 2026-10-28T01:01:07Z
verdict: request-changes
worktree_head_sha: 65b16c998fe6df4ac9aa1dbe4080ec6bc37da606
gate:
  decision: hold
  gated_at: 2026-10-28T01:01:14Z
  gated_head_sha: 6a28e9c4eb2e0604e1684d16237165b4efaf3553
  reviewed_head_sha: 6a28e9c4eb2e0604e1684d16237165b4efaf3553
---

# Review

## Gate

Hold — the trusted-app DCO bypass remains unaddressed at `6a28e9c4eb2e0604e1684d16237165b4efaf3553`.

The current PR head matches the reviewed head, with no intervening commits. The local should-fix finding still applies: the inline app comparison trusts ordinary accounts sharing a configured app name. Paginated GitHub reviews and inline-comment queries returned no entries; issue comments contain no substantive review concerns. Fix the app identity check before merging. This decision is based on the code and review evidence.

Gating list:

- **Not addressed; blocks merge** — `REVIEW.md`, `pkg/plugins/dco/dco.go:138`: require `[bot]` before comparing the configured app name, and cover both the bot login and a matching human login in tests.

Independent merge risk:

- **Behavior/configuration compatibility:** repositories already setting `trusted_apps` while leaving both skip flags false will stop exempting unrelated primary/secondary-org members and the Prow bot merely through the shared trust helper. At `pkg/plugins/dco/dco.go:144-153`, only configured apps remain exempt in that configuration. Existing unsigned commits from those other authors may start failing DCO; this follows the intended separation, but operators should receive a release note explaining the required skip flags. No release note is present in the PR diff.
- **Collaborator-only configuration:** `trusted_org` stops granting a secondary-org member exemption when `skip_dco_check_for_members` is false (`pkg/plugins/dco/dco.go:147-150`). This is intentional and documented in the updated configuration comments; primary-org membership still grants trust through the pre-existing shared helper behavior.
- **API/schema:** no exported Go signatures or configuration field names/types change; the changed helper is private. No CRD, protocol, migration, or rollout dependency is introduced. No applicable repository compatibility skill was available; this pass used direct code inspection.

## Verdict

Request changes: require the bot suffix before granting the trusted-app exemption.

The new inline app check also trusts ordinary accounts whose usernames equal a configured app name, allowing unsigned human commits to bypass DCO. The complete four-file PR diff was reviewed against merge base `4425e386ffba859f8fbfb535617559f05d37bf2f`; the DCO package tests pass, but the added app cases accept suffix-free usernames and miss this regression. The worktree HEAD includes a local review-artifact commit; `head_sha` records the actual reviewed PR head, whose code is identical.

## What this PR does

- Passes the full DCO configuration into trusted-commit filtering so the member and collaborator settings can be considered separately.
- Checks configured trusted apps before calling the shared trigger trust helper, including when both skip flags are disabled.
- Clears the secondary trusted organization when skipping DCO for members is disabled.
- Updates configuration descriptions, the help example, and tests for the revised trust handling.

## Findings

### [should-fix] Require the bot suffix before trusting an app

- where: `pkg/plugins/dco/dco.go:137-140`
- concern: With `trusted_apps: ["my-app"]`, this comparison exempts both `my-app[bot]` and the unrelated human account `my-app`, because `strings.TrimSuffix` leaves ordinary usernames unchanged. The previous `trigger.TrustedUser` check required the `[bot]` suffix, so preserve that guard to prevent unsigned human commits from bypassing DCO; update the added app tests to use bot logins and reject the matching human login.
- excerpt: |
    for _, trustedApp := range config.TrustedApps {
        if tUser := strings.TrimSuffix(commit.Author.Login, "[bot]"); tUser == trustedApp {
            trustedResponse.IsTrusted = true
            break

## Checked

- Reviewed all changes in `pkg/plugins/config.go`, `pkg/plugins/dco/dco.go`, `pkg/plugins/dco/dco_test.go`, and `pkg/plugins/plugin-config-documented.yaml`.
- Traced `handle` through `filterTrustedUsers` and compared the old implementation with `trigger.TrustedUser`, including membership, collaborator, bot-self, and app checks.
- Confirmed trusted-app filtering runs when neither skip flag is set, as intended by the PR.
- Confirmed the updated call site passes the full DCO configuration and the documented configuration matches the new app handling.
- `go test ./pkg/plugins/dco` passed.
- Primary-org members still bypass DCO in collaborator-only configurations through `trigger.TrustedUser`'s unconditional membership check. This behavior already exists in the base and is excluded from the defect findings for this change.
- The added app cases use suffix-free `my-trusted-app` logins and expect them to be trusted, leaving the human/app distinction untested.

## Open questions

- Could you require `[bot]` in the inline app check, use `my-trusted-app[bot]` in the positive cases, and add a negative case for the ordinary `my-trusted-app` account?
