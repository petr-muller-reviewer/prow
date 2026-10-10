---
issue: kubernetes-sigs/prow#500
title: "cherrypicker: add a flag to support `git cherry-pick -x` style commit messages"
state: open
labels: kind/feature, area/plugins
main_sha: 52c1eeb13bd2f1a241b2314b5ca06ed55ab17b2e
triaged_at: 2026-10-10T14:39:56Z
verdict: accepted
refresh_log:
  - timestamp: 2026-06-02T23:43:12Z
    summary: Initial triage completed
  - timestamp: 2026-06-30T14:13:29Z
    summary: Label changed stale→rotten (k8s-triage-robot), no substantive activity
  - timestamp: 2026-10-10T14:39:56Z
    summary: Refreshed since 2026-06-30; petr-muller removed lifecycle/rotten, with no new PR references
advice:
  advised_at: 2026-10-10T14:42:00Z
  based_on_triaged_at: 2026-10-10T14:39:56Z
---

## What the issue reports
- The cherrypicker currently omits source commit SHAs from cherry-pick commit messages.
- The request is for a flag that adds `git cherry-pick -x`-style source trailers.

Since previous triage:
- On 2026-08-08, petr-muller commented `/remove-lifecycle rotten`; the current labels are `kind/feature` and `area/plugins`.
- The issue remains open. There are no other new comments or cross-referenced PRs since 2026-06-30.

## Findings

### [cause] No post-apply commit message modification in cherry-pick flow
- detail: The cherrypicker applies PR patches via `git am --3way` and pushes immediately. Original commit SHAs are present in the patch file's `From <SHA>` headers but are never injected into the resulting commit messages. There is no step between `Am()` and `Push()` to modify messages.
- evidence: `cmd/external-plugins/cherrypicker/server.go:616-635`

### [related-code] Cherry-pick apply and push gap
- where: `cmd/external-plugins/cherrypicker/server.go:616-635`
- excerpt: |
    if err := r.Am(localPath); err != nil {
        // ... error handling, conflict issue creation ...
        return utilerrors.NewAggregate(errs)
    }
    // Push the new branch
    if err := p.Push(r, newBranch, true); err != nil {

### [related-code] getPatch fetches PR patch from GitHub API
- where: `cmd/external-plugins/cherrypicker/server.go:750-762`
- excerpt: |
    func (s *Server) getPatch(org, repo, targetBranch string, num int) (string, error) {
        patch, err := s.ghc.GetPullRequestPatch(org, repo, num)
        // writes to /tmp/<org>_<repo>_<num>_<branch>.patch

### [related-code] Am() implementation
- where: `pkg/git/v2/interactor.go:425-439`
- excerpt: |
    func (i *interactor) Am(path string) error {
        out, err := i.executor.Run("am", "--3way", path)

### [related-code] Flag pattern to follow
- where: `cmd/external-plugins/cherrypicker/main.go:74`
- excerpt: |
    fs.BoolVar(&o.issueOnConflict, "create-issue-on-conflict", false,
        "Create a GitHub issue and assign it to the requestor on cherrypick conflict.")

### [related-code] Interactor interface
- where: `pkg/git/v2/interactor.go:33-79`
- excerpt: |
    type Interactor interface {
        // No methods for commit message amendment, rebase, or log listing.
        // Am(path string) error is the only patch-apply method.

### [related-issue] Third-party cherrypicker usage
- ref: kubernetes-sigs/prow#113
- relevance: Tracks third-party use of the cherrypicker plugin. BenTheElder noted core Kubernetes doesn't use it; xmudrii confirmed active use by Kubernetes subprojects. Plugin has downstream users but limited sig-testing maintenance bandwidth.

### [related-pr] Author's example of desired output
- ref: kubevirt/kubevirt#15073
- relevance: Manually created cherry-picks showing the desired `(cherry picked from commit <SHA>)` trailer in each commit message.

### [related-pr] Author's example of current output
- ref: kubevirt/kubevirt#15076
- relevance: Cherrypicker-created PR showing current behavior — commit messages lack original commit SHAs.

## Checked
- `Interactor` interface has no commit-amend or rebase methods, but patch-modification approach avoids needing them
- No existing configuration surface for cherry-pick commit message formatting
- `getPatch` returns raw patch bytes from GitHub, suitable for pre-processing before `Am()`
- PR #661 by AaruniAggarwal explicitly addresses this request; it remains open, was updated 2026-09-29, and changes the cherrypicker flag, server, and tests. Its review decision is unset and mergeability is unknown.
- Upstream main has advanced since `main_sha`; PR #9404 adds kind-label propagation to cherry-pick PR bodies but does not change the patch-apply/push path relevant to this request.

## Next steps
- Review/track PR #661 before assigning duplicate implementation work. If it stalls or closes without merging, check with AaruniAggarwal; if inactive, remove the assignment and add `help wanted`.
- Maintainer implementation guidance in the 2026-02-23 comment is solid reference for any contributor
- Recommended approach: modify patch file before `Am()` to inject `(cherry picked from commit <SHA>)` trailers — avoids interactor interface changes

## Advice
- Review the existing open PR #661 before assigning duplicate work. It explicitly implements the requested `-x`-style messages, was updated 2026-09-29, and has no recorded review decision; `mergeable` is currently `UNKNOWN`.

  ```sh
  gh pr view 661 --repo kubernetes-sigs/prow
  gh pr diff 661 --repo kubernetes-sigs/prow
  ```

- Keep AaruniAggarwal assigned while PR #661 is active. If the PR stalls or closes without merging, follow up with the author; if they are no longer working on it, unassign and add `help wanted`.

  ```sh
  gh issue comment 500 --repo kubernetes-sigs/prow --body $'> *This was generated by AI during triage.*\n\nHi @AaruniAggarwal, are you still planning to work on PR #661 for this issue?'
  gh issue edit 500 --repo kubernetes-sigs/prow --remove-assignee AaruniAggarwal --add-label "help wanted"
  ```

## Open questions
- Should the `(cherry picked from commit <SHA>)` trailer be default or opt-in via flag? Maintainer leans default.
- Full SHA or abbreviated? `git cherry-pick -x` uses full SHA.
- For multi-commit PRs: individual commit SHAs (preferred, matches `-x`) or PR number?
