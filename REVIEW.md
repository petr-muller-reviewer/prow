---
pr: kubernetes-sigs/prow#1045
title: "fix(clonerefs): recover failed sparse merges with complete fetch"
head_sha: 66eb058c0c9e977cf24ea1c34a9140aad444b134
base: main
reviewed_at: 2026-11-01T23:17:54Z
verdict: approve
---

# Review

## Verdict

Approve with suggestions: the recovery is correctly placed, avoids the in-place retry that would silently leave a broken object store, preserves sparse-checkout semantics, and changes no config or API surface — three hardening items and one coverage fix are worth doing in or right after this PR.

Reviewed from three perspectives (code quality, maintainability, deployment risk); they converged on the same few issues, none blocking. Deployment risk is medium only because a failure path that used to fail fast now performs a full-object re-clone, which during a git-host incident happens across all sparse-checkout jobs at once; nothing breaks on upgrade and rollback is a clean image revert.

## What this PR does

- Splits `Run` into an outer driver and `runClone`, so the whole clone sequence can run twice with different fetch settings.
- On a failed sparse-checkout `git merge` that is not a conflict, removes the failed clone directory and redoes the clone with a full object fetch, keeping `sparse-checkout set` intact.
- Deliberately refuses an in-place merge retry: the merge commit already exists when git dies while writing its output, so a second `git merge` would report success over a broken object store.
- Threads the new `fullFetch` flag into both places that add `--filter=blob:none`, and records the retry as `fell_back_to_full_fetch` plus a line in the human-readable output.
- Adds a unit test for the trigger predicate and an integration test that injects the `upload-pack: not our ref` failure once and asserts that the second attempt is a full fetch with the right merged content and retained sparse paths.

## Findings

### [should-fix] Recovery only covers sparse checkout, not `blobless_fetch`
- where: `pkg/pod-utils/clone/clone.go:97-105`
- concern: the guard is `isSparseCheckoutSet(refs)`, but the mechanism being repaired (`--filter=blob:none` promoting a locally created merge blob to a lazy promisor object) is also enabled by `BloblessFetch` alone at `clone.go:296` and `clone.go:402`. Jobs with `blobless_fetch: true` and no sparse checkout would still crash with no recovery. Mirror the filter condition, or document why sparse is the only reachable trigger.
- excerpt: |
    if !record.Failed || !isSparseCheckoutSet(refs) || len(record.Commands) == 0 {
    	return false
    }

### [should-fix] Failure detection matches the rendered command string
- where: `pkg/pod-utils/clone/clone.go:101-103`
- concern: `strings.Contains(failedCommand.Command, " git merge --no-ff ")` depends on `cloneCommand.String()` rendering `PWD=<dir> <env...> git merge --no-ff <sha>` and on the merge being the last command. A `git -c … merge`, a `retryCommand` wrapper, or a post-merge step would silently disable recovery, and only the integration test would notice. A command-kind marker recorded when the command is built (surfaced on `Command`) would decouple detection from formatting.
- excerpt: |
    failedCommand := record.Commands[len(record.Commands)-1]
    if !strings.Contains(failedCommand.Command, " git merge --no-ff ") {
    	return false
    }

### [should-fix] Any non-conflict merge failure pays for a full re-clone
- where: `pkg/pod-utils/clone/clone.go:97-110`
- concern: the predicate is "merge failed and does not look like a conflict", so unrelated causes (network/auth failure during git's lazy fetch, unrelated histories, missing committer identity) also trigger a complete fetch of a potentially huge repo before failing again, with no jitter or backoff. On a systemic git-host degradation that becomes a full re-clone of every sparse-checkout job across all tenants at once. Consider narrowing to the known signature (`not our ref`, promisor/lazy-fetch errors) and/or a small jittered delay.
- excerpt: |
    return !mergeConflictDetected(failedCommand)

### [should-fix] Recursive delete is keyed on a config-derived path
- where: `pkg/pod-utils/clone/clone.go:79-84`
- concern: `PathForRefs` honours `Refs.PathAlias`, which is not validated (`path.Join(root, "src", "../foo")` resolves outside `root/src`, and `../../..` escapes `root` entirely; verified). Clones run concurrently under one `SrcRoot`, so a pathological alias lets one failed clone delete a sibling clone's directory. Assert the target is a strict descendant of `<dir>/src` (or that it equals `gitCtx.cloneDir`) before removing.
- excerpt: |
    if err := os.RemoveAll(PathForRefs(dir, refs)); err != nil {
    	logrus.WithError(err).Error("Cannot remove failed clone directory")

### [should-fix] Regression test does not run in unit CI and does not reproduce blobless
- where: `test/integration/test/pod-utils_test.go:631`
- concern: the test needs no cluster but lives in a package whose `TestMain` (`overall_test.go:28`) exits 0 unless `-run-integration-test` is set, so it never runs under plain `go test ./...`. Related fidelity note: the fixture uses a local path as `CloneURI` and git logs `warning: filtering not recognized by server, ignoring`, so attempt 1 is not actually blobless; the shim simulates the failure. A clone-package variant (skipped when `git` is absent) would give this signal on every PR.
- excerpt: |
    func TestClonerefsRecoversFromPostMergeFailure(t *testing.T) {

### [nit] Retry boundary is invisible in records and logs
- where: `pkg/pod-utils/clone/clone.go:78-87`, `pkg/pod-utils/clone/format.go:31-33`
- concern: the two attempts' `Commands` are concatenated with no separator, so `clone-records.json` reads as one clone that ran `git init` twice, and the banner prints above the first attempt rather than at the seam. A synthetic `Command{Command: "retry with full object fetch"}` makes incident triage straightforward.
- excerpt: |
    record = runClone(refs, dir, gitUserName, gitUserEmail, cookiePath, env, user, token, true)
    record.Commands = append(failedCommands, record.Commands...)
    record.FellBackToFullFetch = true

### [nit] Rationale and issue link absent from the code
- where: `pkg/pod-utils/clone/clone.go:95-96`
- concern: the key design decision — must re-clone because git may already have created the merge commit before failing — lives only in the PR body, and nothing in `pkg/` references #1043. A future maintainer will read this as "just retry the clone" and may simplify it into an in-place re-fetch/merge. A comment with the issue link and the condition for retiring the workaround is cheap insurance.
- excerpt: |
    // shouldFallBackToFullFetch reports whether a sparse merge failed for a reason
    // that fetching all objects in a fresh repository might resolve.

### [nit] No unit coverage of the `fullFetch` command lists
- where: `pkg/pod-utils/clone/clone_test.go:1114`
- concern: `TestShouldFallBackToFullFetch` covers the predicate, but nothing asserts that `fullFetch` drops `--filter=blob:none` from both `commandsForBaseRef` and `commandsForPullRefs` while leaving `--depth`, `sparse-checkout set` and `submodule update` alone. The existing `TestCommandsForRefs` table makes a `fullFetch: true` case cheap. The retry-also-fails and `RemoveAll`-failure paths are also unexercised.
- excerpt: |
    func TestShouldFallBackToFullFetch(t *testing.T) {

### [nit] `FellBackToFullFetch` is set even when the retry fails
- where: `pkg/pod-utils/clone/clone.go:87`
- concern: the field means "a full-fetch retry was attempted", but the log line "Sparse checkout merge failed, retried with a full object fetch" does not say whether it helped. Either set it only on a successful retry or make the wording/field doc explicit.
- excerpt: |
    record.FellBackToFullFetch = true

### [nit] New table test lacks `t.Run`
- where: `pkg/pod-utils/clone/clone_test.go:1163`
- concern: every other table test in this file (`clone_test.go:858`, `:913`, `:1001`, `:1067`, `:1105`) uses `t.Run`, so a failure here is harder to locate in CI.
- excerpt: |
    for _, testCase := range testCases {
    	got := shouldFallBackToFullFetch(testCase.refs, testCase.record)

### [nit] Duplicated blobless-filter predicate
- where: `pkg/pod-utils/clone/clone.go:296`, `pkg/pod-utils/clone/clone.go:402`
- concern: `(BloblessFetch || sparse)` was already duplicated; this PR adds `!g.fullFetch &&` to both copies. One helper returning the filter args would keep the base and pull fetch sites from drifting when a third fetch is added.
- excerpt: |
    if !g.fullFetch && ((refs.BloblessFetch != nil && *refs.BloblessFetch) || sparseCheckoutSet) {

### [question] Does conflict detection survive a non-English git locale?
- where: `pkg/pod-utils/clone/clone.go:108-110`
- concern: `mergeConflictDetected` matches literal `CONFLICT` / `Automatic merge failed`, which git translates when a locale with gettext data is active. Worst case is an undetected conflict that triggers a wasted full re-clone and then fails anyway, so this is a cost issue, not a correctness one — but using the merge exit status would be locale-proof.
- excerpt: |
    return strings.Contains(command.Output, "CONFLICT") || strings.Contains(command.Output, "Automatic merge failed")

### [question] Does the integration shim survive git object-layout changes?
- where: `test/integration/test/pod-utils_test.go:668-680`
- concern: the injected failure locates the merged blob at `.git/objects/xx/yyyy…` and fails the test if git wrote it into a pack (which is reportedly what older git does on some merge paths). It passes with the local git 2.56, but it is a harness portability risk rather than a product assertion; setting `gc.auto=0` or tolerating a packed object would harden it.
- excerpt: |
    object=".git/objects/$(echo "$blob" | cut -c1-2)/$(echo "$blob" | cut -c3-)"
    if [ ! -f "$object" ]; then
      echo "expected a loose merge-result blob at $object" >&2

### [question] What is the plan for the underlying git regression?
- where: `pkg/pod-utils/clone/clone.go:95-110`
- concern: this works around a git behaviour change in the 2026-10-23 base image, and #1043 also asks for a rollback or pin. The recovery makes the regression silent (and expensive) rather than visible. Is there a tracked follow-up to pin or roll back `gcr.io/k8s-staging-test-infra/git` and/or report upstream, and should the fallback rate be queryable (e.g. via `fell_back_to_full_fetch`) so the workaround can be retired?

## Checked

- No config, API, flag or CRD surface change: `Refs` fields and JSON tags are untouched, so existing `sparse_checkout_files` / `blobless_fetch` / `path_alias` configs parse and behave identically.
- The additive `fell_back_to_full_fetch` field is `omitempty` and both readers (`pkg/initupload/run.go:87`, `pkg/crier/reporters/gcs/reporter.go:142`) use non-strict `json.Unmarshal`, so clonerefs/initupload/crier version skew is safe; rollback is a clean image revert with no persisted state.
- `runClone` is a behaviour-preserving extraction: `Refs` is set for both attempts, `FinalSHA` comes from the winning attempt, and the early returns for user/token generation are unchanged.
- `fullFetch` is consulted at exactly the two sites that add `--filter=blob:none`, so `--depth`, `sparse-checkout init/set`, branch handling and `submodule update` are unchanged.
- A successful fallback leaves `Failed == false` while the stale failing `Command` remains in `Commands`; the only success decision is `record.Failed` in `pkg/clonerefs/run.go:175`, so records and logs stay consistent.
- Merge conflicts still fail the job: `CONFLICT` / `Automatic merge failed` output is recognised and covered by a unit test case.
- Total `Duration` spans both attempts (single `startTime` in `Run`), and per-command durations are retained for each attempt; token censoring applies per command across both runs.
- A fresh repository is the correct recovery: retrying `git merge` in place would hit "Already up to date" once the merge commit exists, leaving the object store broken.
- Exactly one retry, no loop; removal failure aborts with a recorded diagnostic command.
- Tests run locally with the module toolchain (`GOTOOLCHAIN=go1.27.3`): `go test ./pkg/pod-utils/clone/... ./pkg/clonerefs/...`, `go test ./test/integration/test/ -run TestClonerefsRecoversFromPostMergeFailure -args -run-integration-test`, `go build ./...`, `go vet` on the touched packages, `gofmt -l` clean, `git diff --check` clean.

## Open questions

- Does `blobless_fetch: true` without `sparse_checkout_files` reproduce the crash? If yes, should the predicate key off the filter actually being applied rather than off sparse checkout?
- Should the retry be narrowed to the known failure signature so unrelated non-conflict merge failures do not pay for a full blob fetch on very large repos?
- Should `FellBackToFullFetch` mean "retried" or "retried successfully"?
- Is there a tracked follow-up for the git base-image pin/rollback and an upstream report, so this recovery path can eventually be removed?
