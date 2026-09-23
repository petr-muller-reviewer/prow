---
issue: kubernetes-sigs/prow#797
title: "Feature Request: PR Dependencies"
state: closed
labels: kind/feature
main_sha: e601a1ffafd7d8d3a781238a4c5f4233d6248f68
triaged_at: 2026-07-27T00:19:19Z
verdict: wontfix
---

## Findings

### [reproducibility] N/A - feature request, not a bug
- detail: Issue requests a new `/depend #NNN` command so PR B auto-blocks on PR A's merge, auto-releasing when A merges. No reproduction applicable.

### [cause] Auto-unblock-on-merge has no precedent in the plugin codebase
- detail: The "block" half (new `do-not-merge/has-unmerged-dependency` label + Tide's existing generic `missingLabels` filter) needs zero new machinery. The "auto-unblock when dependency merges" half requires a `PullRequestHandler` that looks up and mutates *other* PRs on a merge event, plus periodic reconciliation to survive missed webhooks. No existing plugin does cross-PR lookup/mutation on events.
- evidence: grep for `GetPullRequests`/`Search(` across `pkg/plugins/*` returned nothing; checked `branchcleaner`, `bugzilla`, `releasenote`, `milestoneapplier`, `lgtm`, `reward-owners`, `updateconfig` — all `PullRequestHandler` registrants — none search/mutate other PRs.

### [related-code] `/hold` plugin - precedent for the "block" half only
- where: `pkg/plugins/hold/hold.go:41-108`
- excerpt: |
    RegisterGenericCommentHandler(pluginName, handleGenericComment, helpProvider)
    // matches /hold and /hold cancel via regex, then AddLabel/RemoveLabel(labels.Hold)
- relevance: Purely comment-driven label add/remove; no `PullRequestHandler`, no merge-event reaction, no reconciliation. Simplest possible design in this space and the closest existing analog to the proposed plugin's "block" half.

### [related-code] Tide's blocking-label filter is already generic
- where: `pkg/tide/status.go:209-216`, config field `pkg/config/tide.go:550,583` (`missingLabels`)
- relevance: Confirms the issue commenter's claim that a new blocking label requires zero Tide code changes — `missingLabels` is a plain string list.

### [related-code] `do-not-merge/*` label convention
- where: `pkg/labels/labels.go:22-54`
- relevance: Existing labels (`BlockedPaths`, `CpUnapproved`, `Hold`, `InvalidOwners`, `MergeCommits`, `WorkInProgress`) establish the naming convention a new `do-not-merge/has-unmerged-dependency` label would fit.

### [related-issue] None found
- relevance: No other issue in the tracker references `/depend` or PR-dependency tracking.

### [related-pr] None found
- relevance: No implementation was attempted; issue closed without a PR.

## Checked
- Whether Tide needs code changes to support a new blocking label — no, `missingLabels` is generic config (`pkg/tide/status.go:209-216`).
- Whether any existing plugin already does cross-PR lookup/mutation on merge events — no (7 `PullRequestHandler` plugins checked).
- Whether a PR or related issue already implements or tracks this — none found.
- Whether the issue's own closure rationale (maintainer `Prucek`'s design writeup, author `tallclair`'s agreement to close) holds up against the current codebase — yes, independently confirmed.

## Next steps
- No action required; issue already closed with a sound, community-agreed rationale.
- Optional: apply `wontfix` label for tracker hygiene (currently only has `kind/feature`).
- If GitHub-native stacked PRs later prove insufficient, file a new, narrowly-scoped issue rather than reopening #797.

## Open questions
- None — the original author and a maintainer reached explicit agreement in-thread.
