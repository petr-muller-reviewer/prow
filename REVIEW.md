---
pr: kubernetes-sigs/prow#816
title: "[WIP] slackevents: alert on merged PRs with cncf-cla: no label"
head_sha: 8491820a2ce9bf06e5e9c3dad7e12498ef702dab
base: main
reviewed_at: 2026-07-29T12:53:38Z
verdict: request-changes
---

## Summary
Adds `plugins.CLAAlert` config (repo/org-scoped Slack channel list) and a new `PullRequestEvent` handler in `slackevents`. On merge, if the PR carries `cncf-cla: no` and repo/org has a configured `CLAAlert`, posts a Slack message to each configured channel. Includes help-provider/doc updates and 9 unit test cases. PR tagged WIP/do-not-merge.

Reviewed twice: a direct code review, and a three-perspective maintainer review (code quality, maintainability, deployment risk) with advisor synthesis. All four independent passes converge on the same primary defect below; no correctness bug or deployment-blocking risk was found by any reviewer.

## Findings

### [should-fix] Channel fanout short-circuits on first error
- where: `pkg/plugins/slackevents/slackevents.go:270-278` (in `notifyOnCLAIfMerged`)
- concern: The loop over `ca.Channels` returns immediately when `WriteMessage` errors on one channel, so later channels never get notified. Flagged independently by all three maintainer-review perspectives (code quality, maintainability, deployment risk) plus the direct review — highest-confidence finding in this PR. `notifyOnSlackIfManualMerge` elsewhere in the same file uses `utilerrors.NewAggregate` to attempt every channel and collect errors; this new code should follow that pattern, especially since CLA-compliance alerts benefit from best-effort delivery to all configured channels.
- excerpt: |
    for _, ch := range ca.Channels {
        if err := pc.SlackClient.WriteMessage(msg, ch); err != nil {
            return fmt.Errorf("failed to post CLA alert to %s: %w", ch, err)
        }
    }

### [should-fix] Alert message misattributes the merge to the PR author
- where: `pkg/plugins/slackevents/slackevents.go:262-269` (in `notifyOnCLAIfMerged`)
- concern: The message reads `"*Alert:* PR #%d merged with %q label by %s — %s"` using `pre.PullRequest.User.Login`, which is the PR **author**, not who merged it. This misleads readers into thinking `%s` performed the merge. Use `PullRequest.MergedBy.Login` if available, or reword to make clear it's the author (e.g. "opened by").
- excerpt: |
    msg := fmt.Sprintf(
        "*Alert:* PR #%d merged with %q label by %s — %s",
        pre.Number,
        labels.ClaNo,
        pre.PullRequest.User.Login,
        pre.PullRequest.HTMLURL,
    )

### [nit] getCLAAlert duplicates getMergeWarning's matching logic
- where: `pkg/plugins/slackevents/slackevents.go:281-291` (`getCLAAlert`)
- concern: Byte-for-byte structural duplicate of the existing `getMergeWarning` two-pass (exact org/repo match, then org-level fallback) logic, using `sets.NewString(...).Has(...)` rebuilt per call. Two copies now have to be kept in sync if matching semantics change; consider a shared helper.

### [nit] No observability for sent/suppressed CLA alerts
- where: `pkg/plugins/slackevents/slackevents.go:246-279` (`notifyOnCLAIfMerged`)
- concern: No log line marks "CLA alert sent" or "suppressed (no matching config)". For a compliance-sensitive alert, this makes production debugging of "why didn't we get notified" harder than necessary.

### [nit] Per-repo help text doesn't mention CLA alert config
- where: `pkg/plugins/slackevents/slackevents.go:63-77` (`helpProvider`)
- concern: `configInfo[repo.String()]` documents merge-warning status per repo but says nothing about whether a repo has `CLAAlerts` configured, unlike the parallel `MergeWarning` handling. Doc parity gap, not functional.

### [question] Missing test for WriteMessage error path
- where: `pkg/plugins/slackevents/slackevents_test.go:376-560` (`TestNotifyOnCLAIfMerged`)
- concern: 9 cases cover matching/non-matching/precedence logic well, but none exercise a failing `WriteMessage` call — exactly the path with the short-circuit bug above. Add once fanout error handling is fixed.

### [question] Case-sensitive org/repo matching in getCLAAlert
- where: `pkg/plugins/slackevents/slackevents.go:281-291` (`getCLAAlert`)
- concern: `sets.NewString(a.Repos...).Has(repo.String())` is case-sensitive; a misconfigured `Repos` entry (wrong case) silently never matches and never alerts. Matches existing repo-matching conventions elsewhere in Prow, but easy to get wrong and fails silently.

### [question] No validation that CLAAlert.Channels is non-empty
- where: `pkg/plugins/config.go:855-862` (`CLAAlert` struct)
- concern: A `CLAAlert` entry with `Repos` set but empty `Channels` would silently match and no-op (loop over zero channels). Consider config-load-time validation for parity with other Slack config, if such validation exists for `MergeWarnings`.

## Checked
- Merge/action guard (`Action != Closed || !Merged`) is correct and tested.
- Repo-exact-match-before-org-fallback precedence in `getCLAAlert` is correct and tested (`exact repo match takes precedence over org-level` case).
- Label matching iterates all labels and matches on `labels.ClaNo`, tested with label among others; `break` after first match prevents a double-send even if a duplicate label were ever present in the payload.
- Reading labels directly from `PullRequestEvent` (no extra GitHub API call) is efficient and documented as merge-strategy independent.
- `helpProvider` yaml snippet and doc `<ol>` update render correctly; `plugins.RegisterPullRequestHandler` registration is new and doesn't conflict with existing push/comment handler registrations.
- New `CLAAlerts` field is `omitempty` and purely additive — no breaking change to existing configs, no new Slack scopes/tokens required, rollback is a plain binary revert.
- `FakeClient` used in new tests is a pre-existing test helper, not duplicated.

## Open questions
- Should a failed `WriteMessage` on one channel still allow other configured channels to receive the CLA alert (aggregate errors) rather than aborting the loop?
- Should the alert message use `MergedBy.Login` instead of `User.Login`, or is the author intentionally the signal wanted here?
- Is silent no-op on case-mismatched `Repos` entries or empty `Channels` in `CLAAlert` config acceptable, or should config validation catch this?
- Given this is marked WIP, is there a design decision still pending (e.g., dedup on redelivered webhook events, shared repo-matching helper, or per-repo CLA-alert status in docs) that would change the shape of this review?
