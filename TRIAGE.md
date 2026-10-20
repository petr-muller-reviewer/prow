---
issue: kubernetes-sigs/prow#745
title: "[cherrypicker]: reports a false conflict although the change applies cleanly"
state: open
labels: lifecycle/stale
main_sha: 9fa65c208d31319fc1d64b01100656a8c7a52199
triaged_at: 2026-10-20T13:37:17Z
verdict: accepted
refresh_log:
  - previous: 2026-06-09T14:41:27Z
    summary: "New comment from valen-mascarenhas14 refining root cause to CRLF line-ending stripping by git am; contributor volunteered to implement fix"
  - previous: 2026-06-15T11:18:17Z
    summary: "Recorded maintainer guidance, Karmada's CRLF cleanup tracking, and lifecycle/stale label; issue remains open with no Prow fix linked"
---

## What the issue reports
- The cherrypicker reports a conflict when applying a change that the reporter can apply cleanly through other Git commands.
- The existing reproduction points to CRLF context being stripped by `git am`, making the patch fail against the target branch.

### Since previous triage
- On 2026-06-15, valen-mascarenhas14 asked maintainers for input. BenTheElder replied, “I don't use the cherry-picker plugin,” and suggested checking for CRLF in presubmit; this adds no new reproduction or Prow fix.
- On 2026-06-16, zhzhuang-zju cross-referenced [karmada-io/website#1026](https://github.com/karmada-io/website/issues/1026), which cites this issue and tracks normalizing CRLF files and adding a CRLF check. As last updated on 2026-06-30, its main and release-1.17 cleanup tasks were complete, while release-1.18 and release-1.16 remained open.
- On 2026-10-20, k8s-triage-robot added `lifecycle/stale` after the inactivity period. The issue remains open.

## Findings

### [cause] git am strips CRLF line endings, causing patch context mismatch
- detail: The actual trigger is `git am`'s default behavior of stripping CR from CRLF lines during mail processing. When files in the repository use CRLF line endings, the patch downloaded from GitHub contains CRLF context lines. `git am` normalizes these to LF, causing the patch context to no longer match the working tree content, and the apply fails. `git am --keep-cr --3way` preserves the original CRLF endings and succeeds on the same patch. This is more specific than the initial "missing base blobs" hypothesis — the blob reconstruction fails because the content has been altered by CR stripping.
- evidence: Comment by valen-mascarenhas14 on 2026-06-14 confirming local reproduction. The 6 `compatibility.md` files in `karmada-io/website` were stored with CRLF endings; `git am --keep-cr` applies the patch cleanly. See also [git am --keep-cr documentation](https://git-scm.com/docs/git-am).

### [cause] git am --3way fails when target branch lacks base blobs from patch index
- detail: Original hypothesis. `git am --3way` reconstructs the base tree using blob SHAs from the patch's `index` line. When the target branch has diverged from the source branch, those blobs may not exist locally. Now understood to be a downstream effect of the CRLF stripping — the content mismatch prevents blob lookup. Both causes may independently trigger failures; the CRLF case is confirmed for this specific report.
- evidence: `pkg/git/v2/interactor.go:428-439` — `Am()` runs `git am --3way` with no fallback.

### [reproducibility] Fully reproducible with public PR
- detail: Reporter provides exact steps using `karmada-io/website#1022` and target branch `release-1.18`. `git am --3way` fails on the patch; `git apply --3way`, `git cherry-pick`, and `git am --keep-cr --3way` all succeed on the same change.
- evidence: https://github.com/karmada-io/website/pull/1022#issuecomment-4657244157

### [related-code] Am() method — hardcoded git am --3way
- where: `pkg/git/v2/interactor.go:425-439`
- excerpt: |
    func (i *interactor) Am(path string) error {
    	i.logger.Infof("Applying patch at %s", path)
    	out, err := i.executor.Run("am", "--3way", path)
    	if err == nil {
    		return nil
    	}
    	i.logger.WithError(err).Infof("Patch apply failed with output: %s", string(out))
    	if abortOut, abortErr := i.executor.Run("am", "--abort"); abortErr != nil {
    		i.logger.WithError(abortErr).Warningf("Aborting patch apply failed with output: %s", string(abortOut))
    	}
    	return errors.New(string(bytes.TrimPrefix(out, []byte("The copy of the patch that failed is found in: .git/rebase-apply/patch"))))
    }

### [related-code] handle() error path — no fallback attempted
- where: `cmd/external-plugins/cherrypicker/server.go:616-632`
- excerpt: |
    if err := r.Am(localPath); err != nil {
    	errs := []error{fmt.Errorf("failed to `git am`: %w", err)}
    	logger.WithError(err).Warn("failed to apply PR on top of target branch")
    	resp := fmt.Sprintf("#%d failed to apply on top of branch %q:\n```\n%v\n```", num, targetBranch, err)
    	if err := s.createComment(logger, org, repo, num, comment, resp); err != nil {
    		errs = append(errs, fmt.Errorf("failed to create comment: %w", err))
    	}
    	...

### [related-code] getPatch() — downloads patch in mbox format
- where: `cmd/external-plugins/cherrypicker/server.go:750-761`
- excerpt: |
    func (s *Server) getPatch(org, repo, targetBranch string, num int) (string, error) {
    	patch, err := s.ghc.GetPullRequestPatch(org, repo, num)
    	...
    	localPath := fmt.Sprintf("/tmp/%s_%s_%d_%s.patch", org, repo, num, normalize(targetBranch))
    	...

### [related-code] GetPullRequestPatch() — GitHub API with patch media type
- where: `pkg/github/client.go:2360-2372`
- excerpt: |
    func (c *client) GetPullRequestPatch(org, repo string, number int) ([]byte, error) {
    	_, patch, err := c.requestRaw(&request{
    		accept:    "application/vnd.github.patch",
    		method:    http.MethodGet,
    		path:      fmt.Sprintf("/repos/%s/%s/pulls/%d", org, repo, number),
    		...

### [related-code] Interactor interface — Am() is the only patch application method
- where: `pkg/git/v2/interactor.go:61-62`
- excerpt: |
    // Am calls `git am`
    Am(path string) error

### [related-issue] Karmada tracks CRLF cleanup as a mitigation
- ref: karmada-io/website#1026
- relevance: The issue cites Prow #745 and tracks normalizing CRLF files plus a CRLF check. This supports the identified trigger and offers a downstream mitigation; its release-1.18 cleanup task was still open at the issue's 2026-06-30 update.

## Checked
- No existing issues or PRs in kubernetes-sigs/prow addressing this problem
- No configuration option to select alternative cherry-pick strategies
- The `Interactor` interface has no `Apply()` or `CherryPick()` method — only `Am()`
- `cherrypickapproved` and `cherrypickunapproved` in `pkg/plugins/` are separate label-management plugins, not involved in actual cherry-pick operations
- CRLF root cause confirmed by valen-mascarenhas14's local reproduction (2026-06-14)

## Next steps
- If the issue remains active, remove `lifecycle/stale` and apply `kind/bug`, `area/cherrypicker`, and `help-wanted` as appropriate.
- Reply to valen-mascarenhas14 with guidance: retry with `git am --keep-cr --3way` when plain `git am --3way` fails. Consider `git apply --3way` for failures that `--keep-cr` does not address, such as genuinely missing base blobs.
- Track the unfinished release-1.18 and release-1.16 CRLF cleanup tasks in karmada-io/website#1026 as downstream mitigation; they do not address the general Prow behavior.

## Open questions
- Is retrying with `--keep-cr` sufficient, or should the fix also include a `git apply --3way` fallback for non-CRLF failure cases?
- Should `--keep-cr` be the default instead of a retry? It's harmless on non-CRLF repos (no effect when there are no CRs to keep).
