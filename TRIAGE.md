---
issue: kubernetes-sigs/prow#994
title: "branchprotector: proposing changes to current setup vs. switching to branch rulesets"
state: open
labels: kind/support, area/branchprotector
main_sha: 0edcac618455b9e7716a08be480d6984ef996eb1
triaged_at: 2026-10-09T10:40:27Z
verdict: accepted
advice:
  advised_at: 2026-10-09T12:27:45Z
  based_on_triaged_at: 2026-10-09T10:40:27Z
---
# Triage

## Verdict

**accepted** — This is a valid support and direction question for Prow's branchprotector. A maintainer has clarified that small config-to-API additions are acceptable while Prow eventually adopts rulesets. The broader rulesets design remains a separate effort tracked in #846.

## Resolution

No implementation has merged yet. Issue #994 remains open, and PR #995 was open at head `5ddea54f47dbf78bbfcf31cf8c7bb6f108e34423` when checked on 2026-10-09.

## What the issue reports

- The reporter wants branchprotector to support GitHub's `require_last_push_approval` review protection setting.
- They ask whether Prow should continue extending classic branch protection support or move toward GitHub rulesets.
- The reporter opened PR #995 for the setting.

## Re-triage recommended

Revisit after PR #995 merges or closes, or if maintainers decide to start a rulesets design. The issue's latest activity is the maintainer response on 2026-10-07.

## Findings

### [cause] Current config and API models omit last-push approval

- detail: The checked-out branchprotector policy and GitHub request/response types do not represent `require_last_push_approval`, so Prow cannot configure or reconcile it.
- evidence: `pkg/config/branch_protection.go:88-98`, `cmd/branchprotector/request.go:133-153`, `pkg/github/types.go:638-699`, `cmd/branchprotector/protect.go:650-663`.

### [related-code] Review policy is converted and reconciled through classic branch protection

- where: `pkg/config/branch_protection.go:88-98`; `cmd/branchprotector/request.go:133-153`; `cmd/branchprotector/protect.go:650-663`.
- excerpt: |
    type ReviewPolicy struct {
        DismissStale *bool `json:"dismiss_stale_reviews,omitempty"`
        RequireOwners *bool `json:"require_code_owner_reviews,omitempty"`
        Approvals *int `json:"required_approving_review_count,omitempty"`
        BypassRestrictions *BypassRestrictions `json:"bypass_pull_request_allowances,omitempty"`
    }

### [related-pr] PR #995 adds the requested field and Apps bypass handling

- ref: kubernetes-sigs/prow#995
- relevance: The open PR adds `require_last_push_approval` through config, request generation, state comparison, tests, and docs. Its current diff also adds Apps support for bypass pull request allowances. The current head is `5ddea54f47dbf78bbfcf31cf8c7bb6f108e34423` (8 files; 109 additions and 18 deletions).

### [related-issue] Rulesets support has a separate design discussion

- ref: kubernetes-sigs/prow#846
- relevance: This issue tracks broader rulesets support spanning branchprotector and Tide, and its maintainer discussion calls for a design discussion before implementation.

### [related-issue] Maintainer answered the immediate direction question

- ref: kubernetes-sigs/prow#994
- relevance: On 2026-10-07, maintainer Petr Muller said he is reluctant to add more branch-protection logic because rulesets are the eventual direction, but is comfortable with simple config-to-API wiring like this proposal and would review the PR. [Comment](https://github.com/kubernetes-sigs/prow/issues/994#issuecomment-6037268076).

## Resolved

- The immediate question of whether a small branch-protection setting is acceptable has been answered: a maintainer supports evaluating this simple additive change while treating rulesets as the future direction.

## Checked

- Issue #994 is open with labels `kind/support` and `area/branchprotector`; its latest comment is Petr Muller's response from 2026-10-07T11:48:44Z.
- PR #995 is open at head `5ddea54f47dbf78bbfcf31cf8c7bb6f108e34423`; inspected its current diff and file summary.
- Read issue #846 and its maintainer discussion on rulesets scope and design.
- Inspected the branchprotector policy, request generation, reconciliation comparison, GitHub types, and component docs.
- Checked GitHub's [ruleset management](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets) and [branch-protection conversion](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/converting-branch-protections-to-rulesets) docs.

## Next steps

- Review PR #995 on its merits, including both the requested field and its current Apps bypass changes.
- Keep the broader rulesets design discussion in #846; link #994 there if maintainers want a single design thread.
- After PR #995 is resolved, decide whether to close #994 as answered or retain it to track the larger rulesets question.

## Advice

1. **Review PR #995.** It remains open and mergeable, with no review decision recorded. Its current scope includes both the requested setting and Apps support for bypass allowances.

   ```sh
   gh pr view 995 --repo kubernetes-sigs/prow --web
   ```

2. **Leave the issue labels and milestone unchanged for now.** `kind/support` and `area/branchprotector` are already applied; no milestone is set, and the linked PR is in flight.

3. **After PR #995 merges, close #994 if #846 will remain the home for the broader rulesets discussion.** Do not close it while the PR is still under review.

   ```sh
   gh issue close 994 --repo kubernetes-sigs/prow --comment "> *This was generated by AI during triage.*

   PR #995 addresses the immediate setting request. The broader GitHub rulesets design is tracked in #846."
   ```

## Open questions

- Should #994 remain open after PR #995 is resolved, or should the remaining rulesets question be consolidated into #846?
- Is there maintainer interest in starting the broader rulesets design now, and who would own it?
