---
issue: kubernetes-sigs/prow#468
title: "Cherrypick behaviour is different depending on PR author and whether you use a label or a comment"
state: open
labels: kind/bug, good first issue, help wanted, area/plugins
main_sha: 695a33027101f624c2bbd80f6943b47abb46d09e
triaged_at: 2026-09-23T20:39:01Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 2
recommended_labels: [kind/bug, area/plugins, help wanted]
---

# Triage

## Verdict

Accepted — review PR #943 as the candidate resolution; retain #468 until that change merges and is verified.

The cherrypicker authorizes comment requests as their commenter but historically authorizes label requests as the PR author. This rejects a request made by an authorized labeler when the author is not an organization member. Merged PR #614 only makes that rejection visible; open PR #943 addresses its cause.

## What the issue reports

- An Istio maintainer observed intermittent label-triggered cherry-picks that did not run, while an equivalent comment by the same person did.
- Prow permits only organization members to request a cherry-pick by default.
- A cherrypick label is treated as though the PR author requested it, rather than the user who added the label.
- Thus, a member can add the label to a non-member-authored PR and have the request rejected despite being authorized to issue `/cherrypick` directly.

## Findings

### [reproducibility] Member labeler and non-member author produce different results
- detail: With `--allow-all=false`, an organization member's comment is authorized by its comment author, but a cherrypick label is authorized as the PR author and can be rejected.
- evidence: Issue #468 report; `cmd/external-plugins/cherrypicker/server.go:186-307`, `342-404`, `407-487`.

### [cause] Label paths conflate the requester with the PR author
- detail: The live label handler checks and passes `pr.User.Login`; the merged-PR handler inserts each cherrypick label under `pr.User.Login` before membership filtering. The live webhook's sender is not used, and the merge-time path does not retrieve label actors.
- evidence: `cmd/external-plugins/cherrypicker/server.go:342-404`; `cmd/external-plugins/cherrypicker/server.go:457-487`.

### [related-code] Existing client support exposes issue-event actors
- where: `pkg/github/client.go:4768-4789`; `pkg/github/types.go:780-840`
- excerpt: `ListIssueEvents` retrieves GitHub issue events, whose `ListedIssueEvent` includes the event action, actor, and label.

### [related-pr] Feedback-only mitigation is merged
- ref: kubernetes-sigs/prow#614
- relevance: Merged on 2026-04-20. It posts a rejection comment when a non-member PR author causes a label request to be ignored, but keeps PR-author attribution.

### [related-pr] Open PR implements labeler attribution
- ref: kubernetes-sigs/prow#943
- relevance: Uses the webhook sender for live label events and issue-event actors after merge, preferring an authorized non-bot labeler and falling back to the author. It declares `Fixes #468`; its unit, integration, image-build, lint, EasyCLA, and Netlify checks are successful, while Tide is pending.

## Checked

- Full issue body and comments, including Craig Box's reproduction and subsequent maintainer discussion.
- Cross-references: #614 and #943 are relevant; #16601 is unrelated.
- Similar-issue searches: `cherrypick author` and `cherry-pick label`; no duplicate found.
- Current label and comment control flow in `cmd/external-plugins/cherrypicker/server.go`.
- Current label tests in `cmd/external-plugins/cherrypicker/server_test.go:880-1045` and `1353-1443`; they do not cover a member labeler on a non-member-authored PR.
- Existing issue-event client support and the full diffs for #614 and #943.

## Next steps

- Review #943's attribution and fallback logic for both live labels and labels observed at merge.
- Confirm coverage for member-labeler/non-member-author, bot labeler, unavailable event history, and label removal/readdition.
- Keep #468 open until #943 merges and the affected paths are verified.
- Apply `kind/bug`, `area/plugins`, and `help wanted` manually if the current labels need correction; Level 2 is more accurate than `good first issue` for the complete two-path authorization fix.

## Open questions

- Does the fallback to PR-author attribution on issue-events API failure behave acceptably for deployments with restricted GitHub API access or incomplete event history?
