---
pr: kubernetes-sigs/prow#816
title: "slackevents: alert on merged PRs with cncf-cla: no label"
head_sha: 8491820a2ce9bf06e5e9c3dad7e12498ef702dab
base: main
reviewed_at: 2026-10-15T22:26:38Z
verdict: request-changes
gate:
  decision: hold
  gated_at: 2026-10-15T23:08:12Z
  gated_head_sha: 8491820a2ce9bf06e5e9c3dad7e12498ef702dab
  reviewed_head_sha: 8491820a2ce9bf06e5e9c3dad7e12498ef702dab
refresh_log:
  - old_sha: 8491820a2ce9bf06e5e9c3dad7e12498ef702dab
    new_sha: 8491820a2ce9bf06e5e9c3dad7e12498ef702dab
    summary: "No code changes; incorporated local Slack test report, WIP removal, and new PR comments and labels."
  - old_sha: 8491820a2ce9bf06e5e9c3dad7e12498ef702dab
    new_sha: 8491820a2ce9bf06e5e9c3dad7e12498ef702dab
    summary: "No code changes; recorded the author's October 15 review reminder."
---

## Gate

**hold** — fix incomplete channel delivery and misleading merge attribution before merging.

The current PR head is unchanged from the reviewed head, `8491820a2ce9bf06e5e9c3dad7e12498ef702dab`. Both should-fix findings from `REVIEW.md` remain present in the current code. There are no submitted GitHub reviews or inline comments adding further gating concerns. The author's local Slack delivery report demonstrates the successful single-channel path, but does not resolve either finding. The full PR diff introduces optional configuration and an opt-in notification path without an identified breaking change for existing deployments.

Gating list and findings disposition:

- **not-addressed — Channel fanout short-circuits on first error** (source: `REVIEW.md`): `pkg/plugins/slackevents/slackevents.go:275-278` returns as soon as one `WriteMessage` call fails, skipping every later configured channel. This gates merge for deployments enabling CLA alerts with multiple channels. Attempt every channel and aggregate errors; add a test where an earlier channel fails and a later channel still receives its message (`pkg/plugins/slackevents/slackevents_test.go:374`).
- **not-addressed — Alert message misattributes the merge to the PR author** (source: `REVIEW.md`): `pkg/plugins/slackevents/slackevents.go:268-273` formats "merged ... by" with `PullRequest.User.Login`, the author. This gates merge because every enabled alert can name the wrong merger. Reword the message to identify the author explicitly (for example, "opened by") and update the expected message in the test. The current `github.PullRequest` type does not expose `MergedBy`, so the earlier suggestion cannot be applied directly without extending that type.

Evidence correction: the earlier fanout finding incorrectly claimed that `notifyOnSlackIfManualMerge` already aggregates errors. Its current loop at `pkg/plugins/slackevents/slackevents.go:173-176` also returns on the first error. The CLA delivery defect stands independently of that inaccurate precedent.

Independent merge risk:

- **API/config:** `pkg/plugins/config.go:629-635` adds optional `Slack.CLAAlerts` (`claalerts,omitempty`), and `pkg/plugins/config.go:855-862` adds `CLAAlert`. No existing exported fields, signatures, configuration keys, or defaults are removed or changed. Existing deployments without matching CLA alert configuration return before making a Slack call (`pkg/plugins/slackevents/slackevents.go:255-260`); no coordinated rollout or migration is indicated.
- **Behavior:** `pkg/plugins/slackevents/slackevents.go:58` registers a new pull-request handler. Slack output requires a closed, merged PR, matching explicit repo/org configuration, and the `cncf-cla: no` label. The delivery and attribution risks above affect those opted-in repositories and channels. Existing push and comment behavior is unchanged by the diff.
- **Configuration pitfalls, non-gating:** matching is case-sensitive (`pkg/plugins/slackevents/slackevents.go:287-298`), and empty channel lists silently send nothing (`pkg/plugins/config.go:861`). These affect newly configured CLA alerts; they do not change behavior for existing configurations. No relevant compatibility-specific repository skill was available; this assessment used the full PR diff and current source.

## What this PR does
Adds `plugins.CLAAlert` config (repo/org-scoped Slack channel list) and a new `PullRequestEvent` handler in `slackevents`. On merge, if the PR carries `cncf-cla: no` and repo/org has a configured `CLAAlert`, posts a Slack message to each configured channel. Includes help-provider/doc updates and 9 unit test cases.

Reviewed twice: a direct code review, and a three-perspective maintainer review (code quality, maintainability, deployment risk) with advisor synthesis. All four independent passes converge on the same primary defect below; no correctness bug or deployment-blocking risk was found by any reviewer.

Since previous review:
- No code changes: `8491820a2ce9bf06e5e9c3dad7e12498ef702dab` remains the PR head.
- On 2026-09-29, AaruniAggarwal removed `[WIP]` from the title; the `do-not-merge/work-in-progress` label was removed. The author reported a successful local merge test and showed the bot's `WriteMessage` log and Slack alert. This confirms the basic delivery path but does not test multi-channel failure handling or merge attribution when author and merger differ.
- The author requested a review on 2026-09-29. On 2026-09-30, petr-muller commented `/ok-to-test`, and the bot added `ok-to-test` and removed `needs-ok-to-test`. There are no new inline comments or submitted reviews.

Since previous review (2026-09-30):
- No code or PR state changes. On 2026-10-15 at 08:20 UTC, AaruniAggarwal [reminded petr-muller to review](https://github.com/kubernetes-sigs/prow/pull/816#issuecomment-6187720889). No new inline comments, submitted reviews, or label changes followed.

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
