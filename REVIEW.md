---
pr: kubernetes-sigs/prow#1007
title: "WIP: Require rationale for /retest and /override"
head_sha: 803a824aa9190d4b04e88c7885f7ef4756405c51
base: main
reviewed_at: 2026-10-10T14:14:36Z
verdict: request-changes
gate:
  decision: do-not-merge
  gated_at: 2026-10-10T14:18:44Z
  gated_head_sha: 803a824aa9190d4b04e88c7885f7ef4756405c51
  reviewed_head_sha: 803a824aa9190d4b04e88c7885f7ef4756405c51
---

# Review

## Gate

**Verdict: do-not-merge** — the blocking rationale-bypass finding remains unchanged at the current PR head.

The PR is still open at `803a824aa9190d4b04e88c7885f7ef4756405c51`, the same SHA covered by the review, with no commits since review. No submitted GitHub reviews, inline comments, or substantive human issue comments add or resolve findings. The local blocking finding is confirmed in the current code: the preflight validates the first `/override` rationale, while the native handler processes every match. Do not merge until repeated commands are rejected or each invocation is validated before side effects.

### Gating list

- `REVIEW.md` `[blocking]` — repeated `/override` invocations bypass rationale validation for later commands (`pkg/plugins/override/override.go:324`, `pkg/plugins/rationale.go:119-140`, `pkg/plugins/override/override.go:489-506`). Fix by rejecting repeats or validating every occurrence before status or check-run mutations.

### Area 2 — Independent merge risk

- **Configuration and behavior:** `require_rationale` is an additive optional field, and enforcement is exact-repository opt-in. Existing configurations and unconfigured repositories retain their prior behavior. Invalid new policies or duplicate exact repository entries across global and supplemental config can fail validation/merge, affecting only installations enabling this feature. The new docs describe the policy and rollout.
- **Operational dependency:** `Emergency: true` triggers GitHub team membership lookups; lookup failures block that command, and each configured team lookup adds API traffic. This is limited to opted-in commands/comments using the emergency field.
- **Possible Go source-compatibility change:** The PR adds exported `PluginConfig` to exported `trigger.Client` (`pkg/plugins/trigger/trigger.go:202-209`). Downstream packages using unkeyed composite literals of this type would fail to compile; no such use was found in this repository, and the package exposes no constructor or handler using this type. Confirm whether this type is considered a supported external API before treating the field addition as harmless.
- **No other notable compatibility risk found:** the other new exported configuration and rationale fields/types are additive; no existing default changes for users without a policy were found.

## Verdict

Request changes. Two independent maintainer perspectives confirmed that the `/override` rationale check validates only the first invocation, while the native handler processes every match. A valid first rationale can therefore allow additional overrides without separate rationale; reject repeated commands or validate each invocation before processing any override.

## What this PR does

- Adds opt-in rationale policies scoped to exact repositories and the `/retest` and `/override` commands.
- Parses `Reason:` and optional `Emergency:` fields and checks a configurable minimum of non-whitespace Unicode code points.
- Adds a team-based emergency length bypass, config and handler coverage, and user documentation.

## Findings

### [blocking] Validate or reject every `/override` invocation
- where: `pkg/plugins/override/override.go:324-342`; `pkg/plugins/rationale.go:119-140`; `pkg/plugins/override/override.go:489-506`
- concern: The preflight calls rationale validation once, and the parser selects the first matching command block. The native handler then processes every `/override` match, so a valid first reason can authorize subsequent overrides with no rationale. Reject repeated invocations or validate each before any status or check-run mutation.
- excerpt: |
    if settings, required := pluginConfig.RequireRationaleFor(org, repo, "override"); required && len(overrideRe.FindAllStringSubmatch(e.Body, -1)) > 0 {
        context := plugins.RationaleContext{
            Command:     "/override",
            Actor:       e.User.Login,
            Org:         org,
            Repo:        repo,
            PullRequest: e.Number,
        }
        if _, err := plugins.ValidateRationale(e.Body, context, settings, oc, logger); err != nil {

### [nit] Keep rationale and native command matchers aligned
- where: `pkg/plugins/rationale.go:36-45`
- concern: The rationale parser duplicates the `/retest` and `/override` matchers and relies on comments to keep them synchronized with the native handlers. Consider sharing the matchers or adding tests that catch syntax drift.
- excerpt: |
    var rationaleCommands = map[string]rationaleCommand{
        "retest": {
            // Keep this syntax in sync with pjutil.RetestRe.
            matcher: regexp.MustCompile(`^/retest\s*$`),
        },
        "override": {
            // Keep this syntax in sync with override.overrideRe.
            matcher: regexp.MustCompile(`(?i)^/override( ([^\r\n]+))?[\r\n]?$`),
        },
    }

### [nit] Reduce duplicated rationale rejection flow
- where: `pkg/plugins/trigger/generic-comment.go:83-91`; `pkg/plugins/override/override.go:332-342`
- concern: Both integrations repeat the validation failure log, response construction, and comment creation. Consider extracting the shared rejection flow if more protected commands are added. This is a judgment-call smell, not a documented repository-standard violation.
- excerpt: |
    if _, err := plugins.ValidateRationale(body, context, settings, teamClient, logger); err != nil {
        logger.WithError(err).WithFields(...).Warn("Blocking command because rationale validation failed")
        response := plugins.RationaleErrorComment(context.Command, noAction, settings, err)
        return createComment(plugins.FormatResponseRaw(body, htmlURL, actor, response))
    }

### [question] Should emergency reasons be copied into logs?
- where: `pkg/plugins/rationale.go:221-230`
- concern: The emergency path logs the entire user-provided rationale. Please confirm that Prow log access and retention are appropriate for arbitrary PR comment text, or consider logging the bypass event without the full reason.
- excerpt: |
    logger.WithFields(logrus.Fields{
        "event":        "rationale_emergency_bypass",
        "actor":        rationaleContext.Actor,
        "repo":         rationaleContext.Org + "/" + rationaleContext.Repo,
        "pull_request": rationaleContext.PullRequest,
        "command":      rationaleContext.Command,
        "reason":       rationale.Reason,
        "team_matched": team,
    }).Info("Emergency rationale bypass used")

### [question] Can emergency lookups avoid unnecessary API load?
- where: `pkg/plugins/rationale.go:196-220`; `pkg/plugins/trigger/generic-comment.go:74-83`
- concern: An `Emergency: true` comment triggers team-membership lookups before the existing trust or command-authorization checks. Consider checking existing authorization first or using a direct membership check to reduce API work; lookup errors currently block the command.
- excerpt: |
    if rationale.Emergency {
        team, authorized, err := authorizedRationaleBypassTeam(teamClient, rationaleContext, settings.EmergencyBypass)
        if err != nil {
            return Rationale{}, &RationaleValidationError{Kind: RationaleErrorValidation, Cause: err}
        }
        if !authorized {
            return Rationale{}, &RationaleValidationError{Kind: RationaleErrorEmergencyBypassDenied}
        }
    }

## Checked

- The policy is opt-in and scoped to exact repositories; unconfigured repositories preserve existing behavior.
- Emergency membership bypasses rationale length only; existing `/override` authorization and `/retest` trust checks remain in place.
- Team lookup and rationale validation errors fail closed.
- There are no breaking changes for existing repositories unless they enable a policy. Duplicate exact repository entries across global and supplemental config cause the merge to fail.
- Reviewers found test coverage for parser edge cases, Unicode counting, and team lookup failures. Tests were not run during this review.
- The review helper for fetching referenced issues was absent, so the PR description was used as the spec source.

## Open questions

- Should emergency bypass logs retain the full free-text reason, and what log access or retention expectations should operators follow?
- Can team membership checks run after existing trust and command-authorization checks, or use a more targeted API call?
