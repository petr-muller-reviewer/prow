---
issue: kubernetes-sigs/prow#686
title: "blunderbuss: Auto-assign approvers when PR receives LGTM label"
original_title: "Assigning Approvers after PR receives LGTM"
state: open
labels: []
verdict: legitimate
category: feature
effort: 2
triaged_at: "2026-07-26T23:35:44Z"
main_sha: 3e578e4f0ad16bb4435dcbf4c52434d9ec34667b
refresh_log:
  - timestamp: "2026-04-20T00:00:00Z"
    summary: "Initial triage"
  - timestamp: "2026-05-14T17:20:26Z"
    summary: "Maintainer BenTheElder responded: considers existing approve plugin instructions sufficient"
  - timestamp: "2026-07-26T23:35:44Z"
    summary: "Reporter asked BenTheElder to reconsider; BenTheElder pushed back further, citing reviewer-time (not author-time) as the real bottleneck and flagging spurious/premature LGTMs as a risk of auto-assignment"
---

# Issue #686: Assigning Approvers after PR receives LGTM

Reported by [NiJuFirenzia](https://github.com/NiJuFirenzia) on 2026-04-17.
[View on GitHub](https://github.com/kubernetes-sigs/prow/issues/686) |
[Full triage document](https://github.com/petr-muller/prow/blob/issue-triage-686/ISSUE-TRIAGE.md)

**Legitimate** | **Feature Request** | **Level 2 -- Moderate** | **help-wanted**

## What the issue reports

When a new PR is opened, the **blunderbuss** plugin automatically assigns reviewers from OWNERS files. Once a reviewer gives LGTM, there is no equivalent mechanism to automatically assign **approvers**. Approvers must manually discover LGTM'd PRs, adding friction to the merge workflow.

### Since previous triage:

- **Maintainer response** (2026-05-12): [BenTheElder](https://github.com/BenTheElder) commented that the approve plugin already posts clear instructions for the PR author to assign an approver themselves, with a [screenshot example from PR #716](https://github.com/kubernetes-sigs/prow/pull/716#issuecomment-4421792093). This signals the maintainer considers the current advisory-text approach sufficient and may not see a need for automatic assignment.
- **Reporter follow-up** (2026-06-16): [NiJuFirenzia](https://github.com/NiJuFirenzia) asked BenTheElder to reconsider, arguing automation would let the PR author be "more hands off" and save reviewer time.
- **Maintainer pushback, reinforced** (2026-06-17): BenTheElder [replied](https://github.com/kubernetes-sigs/prow/issues/686) that reviewer time, not PR-author time, is the actual bottleneck, and questioned whether "making it easier to put no effort into PRs" is net-helpful. He also raised a new concrete concern: "occasionally people just randomly LGTM out of nowhere" -- i.e. an automatic-assignment trigger on the `lgtm` label risks firing on spurious/premature LGTMs, not just genuine review sign-off. This strengthens (rather than reverses) the maintainer's prior skepticism and adds a design risk not previously captured under Open Design Decisions.

## The Gap

The **approve** plugin already computes the optimal set of approvers for every PR via `GetCCs()` and `GetSuggestedApprovers()`. It even renders advisory text:

> "Once this PR has been reviewed and has the lgtm label, please assign [suggested_approvers] for approval."

All the selection logic exists -- it just isn't wired to an automatic action.

### Current Data Flow

```
PR opened → Blunderbuss assigns reviewers → Reviewer comments /lgtm → LGTM label added → ??? nothing assigns approvers → Approver self-discovers PR
```

## Affected Components

- **Blunderbuss** -- `pkg/plugins/blunderbuss/blunderbuss.go` -- auto-assigns reviewers; recommended extension point
- **Approve/Owners** -- `pkg/plugins/approve/approvers/owners.go` -- `GetCCs()` set-cover algorithm for approver selection
- **LGTM** -- `pkg/plugins/lgtm/lgtm.go:348` -- adds the `lgtm` label, fires `PullRequestActionLabeled`
- **RepoOwners** -- `pkg/repoowners/repoowners.go` -- parses OWNERS files, separates reviewers vs approvers

The `PullRequestActionLabeled` event type already exists (`github/types.go:185`) but no plugin currently uses it for assignment.

## Solution Approaches

### 1. New Plugin

Standalone "approver-assigner" plugin listening for labeled events.

- Pro: Clean separation of concerns
- Con: Duplicates approver selection logic from approve plugin

### 2. Extend Blunderbuss (Recommended)

Add opt-in config to blunderbuss to handle `lgtm` label events and assign approvers.

- Pro: Reuses existing infrastructure
- Pro: Natural fit -- blunderbuss is already the "auto-assign" plugin

### 3. Extend LGTM

Add approver assignment directly in LGTM plugin after label addition.

- Pro: Direct trigger point
- Con: Mixes LGTM concern with approver assignment concern

### Why Blunderbuss?

- Already the "auto-assign" plugin with multiple trigger events (PR open, status, `/auto-cc`)
- `fallbackReviewersClient` already maps approver lists to the reviewer selection interface
- Existing infrastructure for OWNERS loading, user selection, and `RequestReview()`
- Adding `PullRequestActionLabeled` handling follows the `WaitForStatus` pattern

## Implementation Sketch

1. **Add config field** -- `AssignApproversOnLGTM bool` in `Blunderbuss` struct (`pkg/plugins/config.go`)
2. **Handle labeled events** -- Extend `handlePullRequest()` to fire on `PullRequestActionLabeled` when label is `lgtm`
3. **Select approvers** -- Load OWNERS via existing infrastructure, select approvers for changed files
4. **Assign** -- Call `RequestReview()` with selected approvers
5. **Test** -- Follow existing patterns in `blunderbuss_test.go` (fake clients, fake OWNERS)

### Open Design Decisions

1. **Assignment mechanism**: `RequestReview()` (review requests, more visible in GitHub UI) vs `AssignIssue()` (assignees)
2. **Selection algorithm**: Blunderbuss layered random selection vs approve plugin's set-cover algorithm (`GetSuggestedApprovers`)
3. **Review-based LGTM**: Should this also trigger when LGTM is added via GitHub review approval (`ReviewActsAsLgtm`)?
4. **Spurious/premature LGTM** (raised by BenTheElder, 2026-06-17): the `lgtm` label can be applied outside a genuine "ready to merge" review sign-off. Auto-assigning approvers on every label event could page an approver prematurely; any implementation should consider debouncing, requiring label persistence, or another guard against noisy triggers.

## Effort Assessment

| Factor | Rating | Detail |
|---|---|---|
| Scope | Moderate | 3-5 files, ~150-300 LOC |
| Complexity | Moderate | edge cases need handling |
| Expertise | Moderate | plugin system + OWNERS |
| Clarity | Well-defined | clear approach |
| Testing | Moderate | existing patterns apply |
| Backwards Compat | Fully compatible | opt-in only |
| Architecture Fit | Good | natural extension |
| External Deps | Well-supported | standard GitHub API |

## Proposed Augmentation Comment

Proposed title: **blunderbuss: Auto-assign approvers when PR receives LGTM label**

```
/retitle blunderbuss: Auto-assign approvers when PR receives LGTM label

The approve plugin already computes suggested approvers for each PR using a set-cover algorithm in `pkg/plugins/approve/approvers/owners.go` (`GetCCs()` and `GetSuggestedApprovers()`). It displays these suggestions in its notification comment with the text _"Once this PR has been reviewed and has the lgtm label, please assign [approvers] for approval."_ However, this is currently manual guidance — no automatic assignment happens.

The most natural place to implement this feature is in the **blunderbuss plugin** (`pkg/plugins/blunderbuss/`), which already handles auto-assignment of reviewers when a PR is opened. Blunderbuss already has infrastructure for loading OWNERS files, selecting users based on changed files, and calling `RequestReview()`. Adding an opt-in config option (e.g., `assign_approvers_on_lgtm: true`) to handle `PullRequestActionLabeled` events when the `lgtm` label is added would follow the existing pattern. The plugin's `fallbackReviewersClient` already maps approver lists to the reviewer selection interface, making the plumbing straightforward.

Key design decisions for the implementation: (1) whether to use `RequestReview()` (GitHub review requests, more visible) or `AssignIssue()` (GitHub assignees); (2) whether to reuse blunderbuss's layered random selection or the approve plugin's set-cover algorithm for choosing approvers; (3) whether this should also trigger on GitHub review approvals that act as LGTM (when `ReviewActsAsLgtm` is enabled), not just on the label event.

/area plugins
/kind feature
/help-wanted
```

## Recommended Labels

- `/area plugins` -- plugin feature, blunderbuss/approve interaction
- `/kind feature` -- new functionality
- `/help-wanted` -- Level 2: well-scoped, suitable for skilled contributors

## Recommended next steps

1. BenTheElder has now pushed back twice (2026-05-12, 2026-06-17), the second time with a substantive objection (reviewer time is the real bottleneck, not author time) plus a concrete technical risk (spurious/premature `lgtm` labels triggering assignment). This is stronger signal than the initial response that the maintainer does not want this feature built as proposed -- posting the augmentation comment as currently drafted is not recommended.
2. If pursuing this further, any comment or PR should explicitly address the spurious-LGTM concern (e.g. propose a guard/debounce) rather than just reiterate the original pitch -- otherwise it re-treads ground BenTheElder has already objected to.
3. Given two rounds of maintainer skepticism and no third-party support, the more likely outcome is this issue stays open as `help-wanted` without traction, or is closed as wontfix if BenTheElder chooses to. Consider this when deciding whether to invest further triage effort here.
