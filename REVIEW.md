---
pr: kubernetes-sigs/prow#770
title: "trigger: expose triggering comment ID and URL in JOB_SPEC"
head_sha: 043dffcf0ea7a043e769daea76d1a2b39cee944f
base: main
reviewed_at: 2026-06-25T12:04:58Z
verdict: approve
---

## Findings

### [should-fix] RunRequestedWithLabels has 7 positional parameters
- where: `pkg/plugins/trigger/trigger.go:335-336`
- concern: Exported function now takes 7 positional args including the new `triggerComment`. Next addition should refactor to an options struct, but this PR should at minimum note the debt.
- excerpt: |
    func RunRequestedWithLabels(c Client, pr *github.PullRequest, baseSHA string, requestedJobs []config.Presubmit, eventGUID string, labels map[string]string, triggerComment *prowapi.TriggerComment) error {

### [nit] Reserved env var names deserve a release note
- where: `pkg/pod-utils/downwardapi/jobspec.go:209`
- concern: `TRIGGER_COMMENT_ID` and `TRIGGER_COMMENT_URL` are now reserved for all presubmits via `EnvForType`. Any existing job config defining these names fails config validation on upgrade. Extremely unlikely collision but should be documented.
- excerpt: |
    pullEnv := []string{PullNumberEnv, PullPullShaEnv, PullHeadRefEnv, PullTitleEnv, TriggerCommentIDEnv, TriggerCommentURLEnv}

### [question] Abstraction boundary: provenance vs. behavioral coupling
- where: `pkg/apis/prowjobs/v1/types.go:240-259`
- concern: TriggerComment introduces "how a job was triggered" into ProwJobSpec, which otherwise describes "what to run against what code". This PR keeps the right boundary (provenance only — ID and URL, not comment body), but future extensions exposing comment content would cross it. Worth an explicit note in the struct doc comment that this is provenance metadata, not a channel for trigger-time arguments.

## Checked
- All callers of `RunRequestedWithLabels` and `RunRequested` updated for new parameter
- `TriggerComment.ID` uses `int`, consistent with prow GitHub types (`IssueComment.ID`, `Review.ID`)
- Shared `triggerComment` pointer in `runRequested` loop is safe: value-only fields, never mutated, DeepCopy generated correctly
- Zero-value handling: `ID == 0` suppresses `TRIGGER_COMMENT_ID`, empty `HTMLURL` suppresses `TRIGGER_COMMENT_URL`
- `EnvForType`/`EnvForSpec` asymmetry (unconditional listing vs conditional setting) matches existing pattern for `PULL_TITLE` etc.
- CRD schema matches Go types; both production and integration CRDs updated consistently
- Tests cover full-comment case (ID + URL) and URL-only case (review body, ID=0) with explicit absent-key assertion
- `gc.CommentID` nil-check in `generic-comment.go` handles events without a dedicated comment ID
- Rollback is clean: old binaries ignore `trigger_comment` field

## Open questions
- Should the `TriggerComment` doc comment explicitly state this is provenance metadata and not intended as a channel for passing trigger-time arguments to jobs?
- Would you consider adding a release note about the two new reserved env var names for presubmits?
