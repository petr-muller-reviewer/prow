---
issue: kubernetes-sigs/prow#500
title: "cherrypicker: add a flag to support `git cherry-pick -x` style commit messages"
state: open
labels: kind/feature, lifecycle/stale, area/plugins
main_sha: 52c1eeb13bd2f1a241b2314b5ca06ed55ab17b2e
triaged_at: 2026-06-02T23:43:12Z
verdict: accepted
---

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
- No other issues or PRs address this feature request
- Contributor AaruniAggarwal self-assigned 2026-03-01 with no PR submitted as of triage time

## Next steps
- Check whether AaruniAggarwal is still working on this (no activity since 2026-03-01)
- `/remove-lifecycle stale` to keep the issue alive
- If contributor inactive, re-label as `help-wanted`
- Maintainer implementation guidance in the 2026-02-23 comment is solid reference for any contributor
- Recommended approach: modify patch file before `Am()` to inject `(cherry picked from commit <SHA>)` trailers — avoids interactor interface changes

## Open questions
- Should the `(cherry picked from commit <SHA>)` trailer be default or opt-in via flag? Maintainer leans default.
- Full SHA or abbreviated? `git cherry-pick -x` uses full SHA.
- For multi-commit PRs: individual commit SHAs (preferred, matches `-x`) or PR number?
