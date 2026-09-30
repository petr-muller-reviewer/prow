---
pr: kubernetes-sigs/prow#794
title: "fix(trigger): verify [bot] suffix before trimming trusted app name"
head_sha: 0d3414e56d8e6b3967058a4fe9863fb4598d75ef
base: main
reviewed_at: 2026-07-27T00:26:18Z
verdict: approve
---

## Summary

In `TrustedUser` (`pkg/plugins/trigger/trigger.go`), the `trusted_apps` check used
`strings.TrimSuffix(user, "[bot]")` unconditionally, which is a no-op when the
suffix isn't present. A plain (non-bot) account whose literal username equalled
an entry in `trusted_apps` was therefore granted the same trust as a genuine
GitHub App bot (`<name>[bot]`) — a privilege-escalation path, since any account
could pick a username matching a configured trusted app and bypass
collaborator/org-membership checks. Fix adds `strings.HasSuffix(user, "[bot]")`
before trimming/comparing, so only genuine bot-suffixed identities can match.
Two unit tests were added covering the previously-mishandled cases. PR already
merged to main.

Independently corroborated by a 3-perspective maintainer review (code quality,
maintainability, deployment risk): all three verdicts agree the logic and
tests are correct and the change is strictly more restrictive (cannot
introduce new false-trust cases); all three independently flagged the same
formatting defect (see should-fix below).

## Findings

### [should-fix] mixed tabs/spaces in new code
- where: `pkg/plugins/trigger/trigger.go:281-282`
- concern: The two new lines use spaces (with a trailing tab) instead of pure tab indentation, inconsistent with the rest of the tab-indented file. Not `gofmt`-clean. Independently flagged by all three maintainer-review perspectives (code quality, maintainability, deployment risk) — high-confidence, low-effort fix. Risk: a future `gofmt -w`/IDE autoformat on this file will churn this hunk with unrelated diff noise in whatever PR touches it next.
- excerpt: |
    ```
            	return okResponse, nil
        	}
    ```

### [question] PR description undersells the security angle
- where: PR description / commit message
- concern: The description ("Ensure that the incoming username actually has the `[bot]` suffix before trimming it") reads as a correctness nit, but the actual effect is closing a privilege-escalation path where a non-bot account with a matching username could gain trusted-app status. Worth confirming with the author whether this was already exploited/observed, or purely found by inspection.

### [question] release note for behavioral tightening
- where: N/A (process)
- concern: Deployment-risk perspective: any installation that was (accidentally) relying on the old loose match for a non-bot account will silently lose that trust grant post-upgrade. No config migration needed, but a short release-note callout would help infrequent-upgrade operators avoid surprise `/ok-to-test` requirements.

## Checked
- Logic correctness: `HasSuffix` + `TrimSuffix` + equality now only matches genuine `[bot]`-suffixed usernames against the configured base name — correct.
- Test coverage: added cases (`github-app` no-suffix vs trusted `github-app`; `github-app[bot]suffix` vs trusted `github-app`) both correctly assert `notMember`/`notCollaborator` — coverage is adequate for the fix.
- Other callers of `TrustedUser` (`pull-request.go`, `generic-comment.go`) unaffected — signature unchanged.
- Deployment risk: no config schema/API/struct changes; change is strictly more restrictive, cannot cause new false-trust cases; rollback trivial. Risk level LOW.
- Maintainability: no new abstractions/dependencies/coupling introduced; burden LOW.

## Open questions
- Was this found via code audit or via an actual incident/report? Affects whether a CVE-style disclosure or backport to release branches is warranted.
- Should a `gofmt`-only cleanup commit be pushed for `trigger.go:281-282`?
- Should a release note be added calling out the stricter `trusted_apps` matching semantics?
