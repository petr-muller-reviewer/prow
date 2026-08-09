---
pr: kubernetes-sigs/prow#102
title: "Update trusted author handling for DCO"
head_sha: 8569602168cf34cb0deadf507652ff711f596785
base: main
reviewed_at: 2026-07-26T23:44:08Z
verdict: request-changes
---

## What this PR does
- Splits `SkipDCOCheckForMembers` and `SkipDCOCheckForCollaborators` so they're handled as independent booleans instead of being tied together via a single `trigger.TrustedUser` call.
- Moves `TrustedApps` matching out of `trigger.TrustedUser` into an inline check in `dco.go`'s `filterTrustedUsers`, so trusted apps are honored regardless of the two skip booleans.
- Zeroes `TrustedOrg` before calling `trigger.TrustedUser` when `SkipDCOCheckForMembers` is false, to stop that call's secondary-org membership check from granting trust in collaborator-only configs.
- Updates doc comments and `plugin-config-documented.yaml`; adds test cases for trusted apps and the member/collaborator split.

## Findings

### [blocking] Member/collaborator separation is incomplete: org members still bypass DCO with collaborator-only config
- where: `pkg/plugins/dco/dco.go:139-156` (calls into `pkg/plugins/trigger/trigger.go:260-265`)
- concern: `trigger.TrustedUser` checks `ghc.IsMember(org, user)` unconditionally as its first branch, gated by neither `onlyOrgMembers` nor the `trustedOrg` parameter — those only affect the collaborator check and the *secondary*-org check, respectively. When `SkipDCOCheckForCollaborators=true` and `SkipDCOCheckForMembers=false`, `filterTrustedUsers` zeroes `trustedOrg` but still calls `trigger.TrustedUser(gc, ..., trustedOrg="", ..., org, repo)`, which still trusts any member of the PR's own org via that first unconditional `IsMember(org, user)` check. This contradicts the PR's stated goal of separating the two booleans: a collaborator-only config still silently skips DCO for plain org members.
- excerpt: |
    if !trustedResponse.IsTrusted && (config.SkipDCOCheckForMembers || config.SkipDCOCheckForCollaborators) {
        trustedOrg := config.TrustedOrg
        // If SkipDCOCheckforMembers is disabled, make sure the trusted org is empty
        if !config.SkipDCOCheckForMembers {
            trustedOrg = ""
        }
        var err error
        trustedResponse, err = trigger.TrustedUser(gc, !config.SkipDCOCheckForCollaborators, config.TrustedApps, trustedOrg, commit.Author.Login, org, repo)

### [should-fix] Inline TrustedApps check is looser than trigger.go's — matches non-bot usernames
- where: `pkg/plugins/dco/dco.go:135-141`
- concern: `trigger.go`'s existing app match (`trigger.go:279-283`) requires `strings.HasSuffix(user, "[bot]")` before trimming and comparing. This new inline copy omits that guard, so `strings.TrimSuffix` is a no-op for non-bot logins, meaning any plain user account whose username literally equals a configured `TrustedApps` entry is also trusted. Diverges from the documented semantics ("usernames of each GitHub App without [bot] suffix") and from the stricter existing implementation — a DCO-bypass vector if a human account name collides with a configured app name.
- excerpt: |
    for _, trustedApp := range config.TrustedApps {
        if tUser := strings.TrimSuffix(commit.Author.Login, "[bot]"); tUser == trustedApp {
            trustedResponse.IsTrusted = true
            break
        }
    }

### [should-fix] New test doesn't actually exercise the member-check-disabled scenario
- where: `pkg/plugins/dco/dco_test.go:259-291`, harness at `pkg/plugins/dco/dco_test.go:683-693`
- concern: Test `"should fail if an user is member of the trusted org but skip member check is false"` is meant to prove that a member is *not* trusted when `SkipDCOCheckForMembers=false`. But the `PullRequestEvent` has no `Repo` set, so `org` passed through to `IsMember` is `""`, while `fc.OrgMembers` is only populated under `"kubernetes"` — so the primary-org membership check trivially returns false regardless of the logic under test. The test only proves the secondary-org-check zeroing works, not that plain org members are excluded, so it doesn't catch the blocking finding above.

### [nit] Comment typo
- where: `pkg/plugins/dco/dco.go:150`
- concern: `SkipDCOCheckforMembers` should be `SkipDCOCheckForMembers` (lowercase `f`).

### [nit] Duplicated trusted-app matching logic
- where: `pkg/plugins/dco/dco.go:135-141` vs `pkg/plugins/trigger/trigger.go:279-283`
- concern: Two independent implementations of "is this login a trusted app" now exist with different behavior (see should-fix above). Consider extracting a shared helper to prevent further drift.

### [nit] Undocumented API surface change and silent behavior change
- where: `pkg/plugins/dco/dco.go:128` (signature), overall PR
- concern: `filterTrustedUsers` now takes the whole `plugins.Dco` struct instead of individual fields — reasonable but not called out as intentional. Separately, `TrustedApps` now takes effect even when neither skip-boolean is set (previously it didn't) — a legitimate fix per the PR description, but a silent, user-visible config semantics change for existing repos with no release-note-style callout.

### [nit] Stale error message and loop-scoped variable reuse
- where: `pkg/plugins/dco/dco.go:131`, `pkg/plugins/dco/dco.go:155`
- concern: `trustedResponse` is declared once outside the loop and reset field-by-field each iteration rather than declared fresh with `:=` inside the loop — works but reads as an unnecessary micro-optimization. The pre-existing error message `"Error checking is member trusted"` (capitalized, grammatically off) is worth tightening to `fmt.Errorf("error checking if user is trusted: %w", err)` while these lines are already touched.

## Checked
- `handle()` call site update to pass full `plugins.Dco` config into `filterTrustedUsers` — consistent with new signature.
- Doc-comment and `plugin-config-documented.yaml` wording changes are consistent with the new decoupled TrustedApps behavior (removal of "By default, this option is ignored").
- New test `"should succeed from a trusted app"` and `"should fail dco check as one unsigned commit is not from the trusted app"` correctly exercise the new inline TrustedApps branch (module the HasSuffix gap noted above).
- `helpProvider`'s example config snippet update (adding `TrustedApps`) is cosmetic/documentation only.

## Open questions
- For collaborator-only configs (`SkipDCOCheckForCollaborators=true`, `SkipDCOCheckForMembers=false`), is it intentional that org members still bypass DCO via `trigger.TrustedUser`'s unconditional primary-org `IsMember` check, or should `filterTrustedUsers` avoid calling `trigger.TrustedUser` at all in that case (checking only `IsCollaborator` directly)?
- Should the inline `TrustedApps` match require the `[bot]` suffix (matching `trigger.go`'s stricter check) to avoid trusting a plain user account that happens to share a name with a configured app?
- `TrustedApps` now applies even with both skip-booleans false — is this behavior change intentional and worth a release note for operators who already set `trusted_apps` without either flag?
