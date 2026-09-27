---
pr: kubernetes-sigs/prow#972
title: "fix(approve): respect lgtm_acts_as_approve in review bodies"
head_sha: f8ded8df3c2be3e651fba2a667ae971c5858b1ae
base: main
reviewed_at: 2026-09-27T14:25:29Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-09-27T14:26:04Z
  gated_head_sha: f8ded8df3c2be3e651fba2a667ae971c5858b1ae
  reviewed_head_sha: f8ded8df3c2be3e651fba2a667ae971c5858b1ae
---

# Review

## Gate

**Decision: merge.** The PR head is unchanged since the saved review. There are no blocking or should-fix findings in `REVIEW.md`, and no substantive human review feedback on the PR. The two saved nits remain suggestions; neither affects the safety of this approval fix.

**Gating list:** None.

**Merge risk:** The change affects approval accounting for existing PRs with `/lgtm` in a review body. With `lgtm_acts_as_approve: false` (including its default), such a command no longer grants approval, so a PR approved only through that path may lose its `approved` label on reconciliation. With the option enabled, a considered changes requested review now takes precedence over `/lgtm` in the same body. Both changes are the documented intent of the PR. There is no exported API or configuration schema change, no migration, and no new operational dependency. Consider noting the possible label correction in release notes.

## Verdict

Approve with suggestions.

The implementation matches the PR's stated behavior and has no blocking correctness or deployment finding. The focused tests and `go vet` pass. Existing PRs whose only approval came from a review-body `/lgtm` may lose their `approved` label on reconciliation; a release note would help operators explain the corrected status.

## What this PR does

- Passes `lgtm_acts_as_approve` into the code that applies approval commands from comments and reviews.
- Skips `/lgtm` commands, including cancellations, when the option is disabled.
- Prevents `/lgtm` from reversing a changes requested review when review state is considered.
- Adds cases covering the reproduced false approval, approval credit, and the review-state configuration boundary.

## Findings

### [nit] Qualify the comment about changes requested reviews

- where: `pkg/plugins/approve/approve.go:597-598`
- excerpt: |
    // considered a cancel. The /lgtm command is only considered if lgtmActsAsApprove is set,
    // and never overrides a requested changes review.
- concern: The new guard applies only when `reviewActsAsApprove` is true. When review state is ignored, the new test expects `/lgtm` in a changes requested review to approve. Qualify the comment to describe that configuration boundary.

### [nit] Cover disabled `/lgtm cancel` in a review body

- where: `pkg/plugins/approve/approve_test.go:1123-1130`
- excerpt: |
    name:                "lgtm command does not supersede simultaneous changes requested review when lgtm does not act as approve",
    reviews:             []github.Review{newTestReview("Alice", "otherwise lgtm.\n/lgtm\n/ok-to-test\n/triage accepted", github.ReviewStateChangesRequested)},
    lgtmActsAsApprove:   false,
- concern: Code Quality and Maintainability both suggested an explicit `/lgtm cancel` case. The new guard skips it, but the added tests exercise only `/lgtm`; a cancellation case would protect the less obvious half of the stated behavior.

## Checked

- The PR behavior and added test cases match the stated reproduction and expected outcomes.
- An explicit `/approve` still overrides a simultaneous changes requested review; the existing test covers this.
- Existing configuration remains valid; the change adds no API calls, dependencies, permissions, or migration requirement.
- `go test ./pkg/plugins/approve/...` passed.
- `go vet ./pkg/plugins/approve/...` passed.

## Open questions

None.
