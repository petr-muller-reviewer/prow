---
issue: kubernetes-sigs/prow#910
title: "mistakenly failed to remove `approved` after requirements changed"
state: open
labels:
main_sha: 5765df26189ddc41cb9c975e89d613f6c26282ef
triaged_at: 2026-09-01T00:28:53Z
verdict: accepted
---

## Findings

### [reproducibility] Reproducible via linked external PR
- detail: Not reproducible in this repo directly, but concretely evidenced by `kubernetes/enhancements#5923`: PR opened as a sig-network KEP (author self-approves via `keps/sig-network/OWNERS`), then repushed with the KEP moved to `keps/sig-architecture/` (author not an approver there). The bot's APPROVALNOTIFIER comment still read "This pull-request has been approved by: danwinship (Author self-approved)" plus "Approval requirements bypassed by manually added approval," and the `approved` label was not removed.
- evidence: kubernetes/enhancements#5923#issuecomment-5453924514 (quoted in kubernetes-sigs/prow#910 issue body).

### [cause] `ManuallyApproved()` bypass is file-set- and time-agnostic
- detail: `Approvers.IsApproved()` = `RequirementsMet() || ManuallyApproved()`. `RequirementsMet()`/`AreFilesApproved()` correctly recompute against the *current* file set each `handle()` run (self-approval is OWNERS-scoped, confirmed by reading `UnapprovedFiles`), so it correctly went `false` once files moved to `keps/sig-architecture/`. But `ManuallyApproved()` overrode that regardless, because it only checks whether the *most recent* `labeled` GitHub event for the `approved` label had a non-bot actor — with no correlation to which file set was in effect when that event happened, and no expiry when later pushes change the file set entirely.
- evidence: `pkg/plugins/approve/approvers/owners.go:556-563` (`RequirementsMet`, `IsApproved`, and doc comment "If a human manually added the approved label, this returns true, ignoring normal approval rules").

### [cause] Two candidate failure modes, undetermined which applies
- detail: (a) `WasLabelAddedByHuman`/`BotUserChecker` may be misidentifying the actual bot/GitHub App identity used on `kubernetes/enhancements` as non-bot, making a genuinely-automated label add look "manual" — a narrow bot-identity bug. (b) Alternatively, a human really did once force the label (a legitimate, intentional override per current design), and the bug is that this override has no bound tied to the file set it was granted for — a broader semantics/design gap. Distinguishing requires the actual GitHub issue-events timeline for `kubernetes/enhancements#5923`, which was not available during this triage.
- evidence: `pkg/github/client.go:3157-3176` (`WasLabelAddedByHuman`), `pkg/github/client.go:1309-1325` (`BotUserChecker`).

### [related-code] Where the bypass is computed and cached
- where: `pkg/plugins/approve/approve.go:508-527`
- excerpt: |
    func humanAddedApproved(ghc githubClient, log *logrus.Entry, org, repo string, number int, hasLabel bool) func() bool {
    	findOut := func() bool {
    		if !hasLabel {
    			return false
    		}
    		humanApproved, err := ghc.WasLabelAddedByHuman(org, repo, number, labels.Approved)
    		...
    		return humanApproved
    	}
    	var cache *bool
    	return func() bool {
    		if cache == nil {
    			val := findOut()
    			cache = &val
    		}
    		return *cache
    	}
    }

### [related-code] Where label add/remove decision is made
- where: `pkg/plugins/approve/approve.go:490-505`
- excerpt: |
    if !approversHandler.IsApproved() {
    	if hasApprovedLabel {
    		ghc.RemoveLabel(pr.org, pr.repo, pr.number, labels.Approved)
    	}
    } else if !hasApprovedLabel {
    	ghc.AddLabel(pr.org, pr.repo, pr.number, labels.Approved)
    }

### [related-code] Actor-vs-bot comparison
- where: `pkg/github/client.go:3157-3176`
- excerpt: |
    func (c *client) WasLabelAddedByHuman(org, repo string, number int, label string) (bool, error) {
    	isBot, err := c.BotUserChecker()
    	...
    	events, err := c.ListIssueEvents(org, repo, number)
    	var lastAdded ListedIssueEvent
    	for _, event := range events {
    		if event.Event != IssueActionLabeled || event.Label.Name != label {
    			continue
    		}
    		lastAdded = event
    	}
    	if lastAdded.Actor.Login == "" || isBot(lastAdded.Actor.Login) {
    		return false, nil
    	}
    	return true, nil
    }

### [related-pr] Similar bug class in same plugin
- ref: kubernetes-sigs/prow#707
- relevance: "approve: fix silent approval bypass when PR exceeds GitHub file list API limit" — a different mechanism (pagination truncation vs. label-history staleness), but the same family of bug: the `approve` plugin's label state silently diverging from the true current-file-set approval state. Useful precedent for the rigor (tests, review) expected on a fix here.

## Checked

- Confirmed self-approval (`AddAuthorSelfApprover`) is correctly OWNERS-scoped via `AreFilesApproved`/`UnapprovedFiles` — ruled out "self-approval is a blanket bypass" as the cause.
- Searched for existing duplicate issues/PRs (`gh search issues/prs`) — none found beyond the unrelated-but-similar #707.
- Read `pkg/plugins/approve/approve_test.go` and `pkg/plugins/approve/approvers/approvers_test.go` — no test currently exercises "label added by actor X, then file set changes to unrelated Y"; this is a genuine coverage gap, not intentionally-accepted-and-tested behavior.

## Next steps

- Pull the GitHub issue-events timeline for `kubernetes/enhancements#5923` (`gh api repos/kubernetes/enhancements/issues/5923/events`) to determine the actor login of the `labeled` event(s) for `approved`, disambiguating bot-misidentification from a genuine manual override.
- Get maintainer consensus on intended semantics of `ManuallyApproved`: permanent override, or scoped/expiring when the file set changes materially.
- Once confirmed, open a PR touching `pkg/plugins/approve/approvers/owners.go`, `pkg/plugins/approve/approve.go`, and/or `pkg/github/client.go`, with new test coverage for the "label added, then unrelated file-set change" scenario.
- Apply `kind/bug` and an appropriate `area/` label (verify exact label name against this repo's label set — not confirmed during this triage).

## Open questions

- Is `ManuallyApproved`'s "once true, always true regardless of later file changes" behavior intentional (per the `IsApproved` doc comment), or should it only bypass requirements for the diff state at the time the label was added?
- What GitHub identity performs Prow's bot actions against `kubernetes/enhancements`, and does `BotUserChecker` correctly recognize it there?
- Should any change to the set of OWNERS-protected files touched by a PR force a fresh `RequirementsMet` re-evaluation and label removal, with `ManuallyApproved` only ever acting as a one-time, non-persistent override?
