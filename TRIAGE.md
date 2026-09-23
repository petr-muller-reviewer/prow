---
issue: kubernetes-sigs/prow#797
title: "Feature Request: PR Dependencies"
state: closed
labels: kind/feature
main_sha: e601a1ffafd7d8d3a781238a4c5f4233d6248f68
triaged_at: 2026-09-23T14:18:55Z
verdict: wontfix
refresh_log:
  - previous_triaged_at: 2026-07-27T00:19:19Z
    summary: "Incorporated VannTen's September 17 clarification that GitHub stacked PRs do not support fork-plus-PR workflows."
---

## What the issue reports

The issue requests a `/depend #NNN` command to hold a PR until its prerequisite merges, with a companion removal command. It proposes first-class support for stacked PRs rather than the current `/hold` plus explanatory-comment workflow.

Since previous triage:
- On 2026-09-17, VannTen noted that GitHub stacked PRs require all stack branches, including the base, to be in one repository, so they do not cover fork-plus-PR workflows.

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

### [refresh] Fork workflow clarification
- detail: VannTen commented on 2026-09-17 that GitHub-native stacked PRs are unusable for the fork-plus-PR workflow because every stack branch, including the base, must be in the same repository.
- relevance: This weakens the original fallback rationale for fork contributors, but does not resolve the documented Prow reliability and reconciliation costs or add an implementation proposal.

## Checked
- Whether Tide needs code changes to support a new blocking label — no, `missingLabels` is generic config (`pkg/tide/status.go:209-216`).
- Whether any existing plugin already does cross-PR lookup/mutation on merge events — no (7 `PullRequestHandler` plugins checked).
- Whether a PR or related issue already implements or tracks this — none found.
- Whether the issue's own closure rationale (maintainer `Prucek`'s design writeup, author `tallclair`'s agreement to close) holds up against the current codebase — yes, independently confirmed.

## Next steps
- No change to the `wontfix` verdict is warranted by the clarification alone; the issue remains closed.
- Optional: apply `wontfix` label for tracker hygiene (currently only has `kind/feature`).
- If fork-based stacked PRs need Prow support, file a new, narrowly-scoped design issue that specifies persistence, dependency topology, and reconciliation requirements.

## Open questions
- None for #797; any renewed proposal should answer the scoped design questions above.
