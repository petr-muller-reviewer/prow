---
pr: kubernetes-sigs/prow#743
title: "fix(autobumper): use HEAD refspec when pushing in App auth mode"
head_sha: 34f6aa69e8fb92f6adc471257d21b822f1c8ce24
base: main
reviewed_at: 2026-07-26T23:50:09Z
verdict: approve
---

## Summary

One-line fix in `cmd/generic-autobumper/bumper/bumper.go`. `processGitHubAppAuth` called
`repoClient.PushToCentral(o.HeadBranchName, true)`, but `PushToCentral(branch string, ...)`
(`pkg/git/v2/publisher.go:96`) passes `branch` straight through as the ref argument to
`git push <remote> <branch>`. Since no local branch named `autobump` (default
`o.HeadBranchName`) is ever created — commits land on the checked-out branch, usually `main` —
git fails with `src refspec autobump does not match any`. Fix changes the call to
`PushToCentral("HEAD:"+o.HeadBranchName, true)`, i.e. `git push <remote> HEAD:autobump`, which
pushes the current commit regardless of local branch naming. Regression was introduced in #716.
Matches the explicit-refspec approach requested in review comments on this PR, which was then
approved and merged.

## Findings

### [should-fix] no regression test for processGitHubAppAuth's push refspec
- where: `cmd/generic-autobumper/bumper/bumper.go:390-393`
- concern: No test exercises `processGitHubAppAuth`'s call into `PushToCentral`, so nothing
  would have caught the original #716 regression or would catch a future reintroduction. Existing
  `pkg/git/v2/publisher_test.go` only covers `PushToCentral` genericially (it treats `branch` as
  an opaque string), not this call site.
- excerpt: |
    logrus.WithField("branch", o.HeadBranchName).Info("Pushing branch directly to upstream repo")
    if err := repoClient.PushToCentral("HEAD:"+o.HeadBranchName, true); err != nil {

## Checked
- `PushToCentral` implementation (`pkg/git/v2/publisher.go:96`) confirms `branch` arg is appended
  verbatim to `git push <remote> <branch>` args, so `"HEAD:"+o.HeadBranchName` is a valid,
  standard refspec — fix is correct.
- Other `o.HeadBranchName` usage, e.g. `MinimalGitPush` call at `bumper.go:297` (fork-based push
  path), is unaffected — that path checks out/creates the branch locally first.
- No security implications; only the push refspec target changed, no credential handling touched.

## Open questions
(none)

## Followups

### Avoid unchanged-tree pushes in App auth mode
- category: cleanup
- necessity: could — avoids unnecessary CI runs and review churn when a repeated bump produces the same file tree.
- where: `cmd/generic-autobumper/bumper/bumper.go:390-393`; compare with `MinimalGitPush` in the same file.
- why followup: PR #743 fixed the App auth push refspec; aligning its unchanged-tree behavior with the fork path is separate work.

```text
In kubernetes-sigs/prow, following merged PR #743, "fix(autobumper): use HEAD refspec when pushing in App auth mode" (merge commit 539ecaca1ce9f8aeeefbfd016be10d5c02876f6c), update the App auth push path in cmd/generic-autobumper/bumper/bumper.go. Before force-pushing HEAD:<HeadBranchName> to the central repository, compare the proposed commit's tree with the existing remote branch's tree and skip the push when they match. Use the existing MinimalGitPush behavior in that file as a guide, while keeping the App auth path's explicit HEAD refspec.

Add focused tests for a remote branch that does not yet exist, an existing branch with the same tree, and an existing branch with a changed tree. The new branch and changed-tree cases must push HEAD:<HeadBranchName>; the same-tree case must avoid a push. Preserve PR creation or update behavior and leave the fork and Gerrit flows unchanged. Limit refactoring to what is needed to share or test the tree comparison.
```
