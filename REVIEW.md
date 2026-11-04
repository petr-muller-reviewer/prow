---
pr: kubernetes-sigs/prow#1054
title: "fix(clonerefs): write merge results before merging partial clones"
head_sha: 40cea8c6791f0c1e52e020f093fc68582f9e1973
base: main
reviewed_at: 2026-11-04T17:29:26Z
verdict: approve
---

# Review

## Verdict

Approve with suggestions: the workaround is correct and low risk; the remaining feedback is non-blocking polish around cost gating, observability, and tracking its eventual removal.

Two reviewers and I independently removed the new `merge-tree` step and reproduced the git 2.54+ signature on git 2.56 (`fatal: git upload-pack: not our ref`), then confirmed the test passes with it restored, so the fix is real rather than a test artifact. No reviewer found a correctness, safety, or configuration problem, and deployment risk is LOW: there are no config/CRD/flag/API changes, and the behavior ships in the pinned `clonerefs` utility image, so a control-plane upgrade alone does not change job pods. Operators of jobs with `blobless_fetch` or `sparse_checkout_files` will see one extra clone-log step per merged pull and deserve a release note; rollback is safe and mixed-version fleets stay consistent.

## What this PR does

- Hoists the existing "blobless fetch or sparse checkout" condition into a `partialClone` flag, reused for the PR-head fetch filter and the new workaround.
- In partial clones, runs `git merge-tree --write-tree HEAD <pull>` before each `git merge`, so merged blobs are written by another process before `git merge` populates its object cache.
- Wraps that command in `bestEffortCommand`, which logs failures and returns empty output plus a nil error, so the step can never fail a clone or mark the record failed.
- Adds `TestRunMergesInPartialClone` (blobless, fetch-by-ref, sparse, conflict) and extends `TestCommandsForRefs` with the new step.
- Leaves non-partial clones and merge commit timestamps unchanged.

## Findings

### [should-fix] Ignored merge-tree failure leaves no evidence in the clone record

- where: `pkg/pod-utils/clone/clone.go:424-439`
- concern: `bestEffortCommand` returns empty output and a nil error, so the clone record shows a successful-looking `git merge-tree ...` entry with no output while the only failure detail is an `Info` log. On git < 2.38 (for example 2.34 in Ubuntu 22.04) every partial-clone merge logs an alarming ignored-failure line, and a genuine conflict logs merge-tree's conflict output immediately before `git merge` reports the same conflict, duplicating `CONFLICT` text for log-based alerting. Recording a short skipped/failure marker on the `Command` without setting `Failed`, plus `--quiet` (git 2.50+) or Debug-level logging, would keep the trace self-contained and quiet.
- excerpt: |
    func (c bestEffortCommand) run() (string, string, error) {
        cmd, out, err := c.runnable.run()
        if err != nil {
            logrus.WithError(err).WithFields(logrus.Fields{
                "command": cmd,
                "output":  out,
            }).Info("Ignoring failure of best-effort command")
        }
        return cmd, "", nil
    }

### [should-fix] Pre-pass runs on git versions where the regression cannot occur

- where: `pkg/pod-utils/clone/clone.go:378-386`
- concern: git 2.38-2.53 (including the 2.47.3 in kubekins-e2e) pays a second full ort merge plus the lazy blob fetches per merged pull for no benefit, and git < 2.38 fails the unsupported `--write-tree` on every merge. A version or capability gate (>= 2.54) would confine the workaround to affected installs; the trade-off is fragility if a distro backports `5946a564` or its fix.
- excerpt: |
    if partialClone {
        // Works around a regression in git 2.54 and later: git merge can ask
        // the server for a blob it just wrote, which fails the merge. Writing
        // those blobs first in a separate process avoids that. git merge
        // reports any real problem, so this step's result is ignored. Remove
        // this once clonerefs ships a fixed git.
        // https://lore.kernel.org/git/yrjh1p92qc7w.fsf@pohly-mobl1.fritz.box/
        commands = append(commands, bestEffortCommand{g.gitCommand("merge-tree", "--write-tree", "HEAD", prCheckout)})
    }

### [should-fix] Temporary workaround has no tracked removal marker

- where: `pkg/pod-utils/clone/clone.go:379-385`
- concern: The removal condition is prose only, while this repo tracks temporary code with `TODO(#NNNN):` (for example `pkg/deck/jobs/jobs.go:54`). A `TODO(#1043): remove once clonerefs ships a git with 5946a564 fixed` would make the workaround greppable so it does not outlive its cause.
- excerpt: |
    // reports any real problem, so this step's result is ignored. Remove
    // this once clonerefs ships a fixed git.

### [nit] Merge drivers execute twice per partial-clone merge

- where: `pkg/pod-utils/clone/clone.go:378-386`
- concern: `merge-tree` honours `.gitattributes` merge drivers (verified), so a repo with a side-effecting driver runs it once during the pre-pass and again during `git merge`. This is rare but a real semantic difference; documenting it next to the workaround would save a future investigation.
- excerpt: |
    commands = append(commands, bestEffortCommand{g.gitCommand("merge-tree", "--write-tree", "HEAD", prCheckout)})

### [nit] Partial-clone predicate is spelled out in three places

- where: `pkg/pod-utils/clone/clone.go:242,253-255,357`
- concern: The "blobless fetch or sparse checkout" concept appears as `sparseCheckoutSet` at 242, again in the base-ref `filterArgs` condition at 253, and again in `partialClone` at 357. Adding another partial mode means finding all three; extracting `isPartialClone(refs)` next to `isSparseCheckoutSet` keeps the concept in one place.
- excerpt: |
    partialClone := (refs.BloblessFetch != nil && *refs.BloblessFetch) || isSparseCheckoutSet(refs)

### [nit] bestEffortCommand contract and multi-pull interleaving are untested

- where: `pkg/pod-utils/clone/clone_test.go:884-1013`
- concern: `bestEffortCommand`'s failure-to-nil-error contract has no direct unit test (unlike `retryCommand`, exercised through `TestGitFetch`), and `TestCommandsForRefs` covers only one pull per partial-clone case, so the `merge-tree`/`merge` interleaving across multiple pulls is not pinned. The real clonerefs-image integration case the PR mentions is also still missing.
- excerpt: |
    // TestRunMergesInPartialClone runs real git against a local server. Git 2.54 and
    // later fail the merge by asking the server for the merged blob when two things
    // hold: the merged blob shares its first OID byte with a blob the merge looked up
    // earlier, which is what seed 9 achieves, and the same lazy fetch needs a blob
    // that really is missing, which is what added.txt is for.

### [question] Regression fixture depends on ort's content hash

- where: `pkg/pod-utils/clone/clone_test.go:878-1007`
- concern: The hardcoded `seed 9` fixture and the `merged[:2] != baseBlob[:2]` self-check mean any change in ort's merge output or the fixture content fails with "fixture no longer triggers the bug" for reasons unrelated to this code. Is that deliberate as a coverage canary, or would computing a colliding seed at runtime (and skipping where the bug cannot occur) age better?
- excerpt: |
    merged, baseBlob := git(clone, "rev-parse", "HEAD:"+file), git(server, "rev-parse", mergeBase+":"+file)
    if merged[:2] != baseBlob[:2] {
        t.Errorf("fixture no longer triggers the bug: merged blob %s and merge-base blob %s have different first OID bytes", merged, baseBlob)
    }

## Checked

- Reproduced the regression on git 2.56.0: with the new step removed, the blobless and sparse cases fail with `fatal: git upload-pack: not our ref`; restored, all cases pass. The Code Quality reviewer reproduced the same independently.
- `git merge-tree --write-tree` uses the same merge machinery as `git merge` (three-way content merges, rename detection, recursive merge-base consolidation) and honours `.gitattributes` merge drivers (verified in a scratch repo).
- No config, CRD, flag, webhook, or API changes; jobs without `blobless_fetch`/`sparse_checkout_files` get the identical command list, and machine consumers of the clone record read only `FinalSHA`.
- `bestEffortCommand` cannot fail a clone and leaves `record.Failed` false; the conflict case proves `git merge` still reports real failures.
- The step writes objects only: no HEAD, index, worktree, or reflog changes, so `FinalSHA` and the deterministic fake merge timestamps are preserved; `TestCommandsForRefs` pins the command ordering.
- `go test ./pkg/pod-utils/clone/ ./pkg/clonerefs/`, `go vet`, and `gofmt -l` are clean; the new test passes repeatedly with `-count=2` and is isolated from host git config (`GIT_CONFIG_GLOBAL=/dev/null`, `GIT_CONFIG_NOSYSTEM=1`).
- No follow-up change to this code is on `main` after the merge.

## Open questions

- #1045 takes the fallback-to-full-fetch approach for the same failure; is it being closed in favor of this PR now that this is merged?
- The kubevirt merge that segfaults on stock clonerefs: is it tracked as a follow-up, and is it confirmed to be the same `5946a564` cache regression rather than a separate memory bug this workaround happens to sidestep?
- Is a clonerefs-image integration test planned (the PR says one can follow), and would a release note for `blobless_fetch`/`sparse_checkout_files` users be worth adding?

## Followups

Recorded by `/review:followup` after PR #1054 merged as `5bf71c2882c9513292618ff13dd6e20973e4a1e6`. Accepted 3, skipped 4 (removal TODO, git-version gating, docs/release note, SIGSEGV root-cause).

### Make the ignored pre-pass failure visible and quiet

- mode: deliver
- where: `pkg/pod-utils/clone/clone.go:424-439`
- necessity: should

```text
In kubernetes-sigs/prow, following PR #1054 — "fix(clonerefs): write merge results before merging partial clones" (merged as 5bf71c2882c9513292618ff13dd6e20973e4a1e6) — change bestEffortCommand in pkg/pod-utils/clone/clone.go so an ignored git merge-tree failure is visible in the clone record without failing the clone.

Context: bestEffortCommand.run currently returns cmd, "", nil on failure, so Record.Commands shows an empty-output, no-error entry while the failure detail only reaches an Info log. On git < 2.38 the unsupported --write-tree fails on every partial-clone merge; on a genuine conflict merge-tree exits 1 and logs conflict text immediately before git merge fails with the same conflict.

The task:
- Keep swallowing the error (the clone must not fail), but return a short marker such as "best-effort command failed; continuing" plus the captured output, or the captured output itself, so Command.Output records what happened. Do not set Record.Failed.
- Stop logging the full output at Info: log at Debug, and/or add --quiet to the merge-tree invocation where supported (git 2.50+). If using --quiet, verify merged blobs are still written on a clean merge.
- Update TestCommandsForRefs golden expectations and add a focused test that a failing best-effort command records evidence but leaves Record.Failed false.

Acceptance criteria:
- A failing merge-tree produces a clone-record command entry with non-empty output (or an explicit marker) and no error, and Run continues to git merge.
- Logs no longer emit merge-tree's conflict detail at Info.
- go test ./pkg/pod-utils/clone/ ./pkg/clonerefs/ passes; gofmt and go vet are clean.

Scope guard: do not change clone behavior, step ordering, or git version handling; do not touch the version-gating or predicate-refactor followups; no commits, pushes, or GitHub actions unless asked.
```

### Probe the partial-clone merge path

- mode: probe
- where: `pkg/pod-utils/clone/clone_test.go:884-1013`
- necessity: should

```text
In kubernetes-sigs/prow, following PR #1054 — "fix(clonerefs): write merge results before merging partial clones" (merged as 5bf71c2882c9513292618ff13dd6e20973e4a1e6) — write throwaway tests that probe the partial-clone merge path for real bugs, then discard them.

Context: the new pre-pass in pkg/pod-utils/clone/clone.go (git merge-tree --write-tree HEAD <pull> before each git merge, wrapped in bestEffortCommand) has no direct test, and multi-pull partial-clone interleaving is only covered for a single pull. The existing regression fixture is TestRunMergesInPartialClone in pkg/pod-utils/clone/clone_test.go.

The task: add temporary tests (uncommitted, or under a scratch path) that exercise:
1. Multiple pulls in one blobless and one sparse-checkout Refs: assert each pull gets merge-tree HEAD <pull> immediately before merge --no-ff <pull>, HEAD advances between pulls, and merge timestamps stay sequential.
2. bestEffortCommand when the wrapped command fails: assert the clone continues, Record.Failed stays false, and record what the Command entry contains (currently empty output and no error).
3. The merge-tree-unavailable path (simulate git without --write-tree, e.g. a wrapper script exiting 129 on merge-tree): assert the clone still completes and git merge reports conflicts normally.
4. Conflict interleaving: first pull conflicts after a successful pre-pass; the second pull is not attempted.
Run with go test ./pkg/pod-utils/clone/ -run <pattern> -count=1 on the available git.

Acceptance criteria:
- Every behaviour above was exercised; each failure is classified as a real implementation bug, a wrong test expectation, or an environment limitation, with evidence.
- If a real bug is found: apply only the minimal fix and report the failing case. If none: no files changed.
- Delete the probe tests before finishing.

Scope guard: do not submit the probe tests, refactor unrelated code, change production behaviour beyond a minimal bug fix, or post to GitHub / commit / push.
```

### Keep the partial-clone predicate in one place

- mode: deliver
- where: `pkg/pod-utils/clone/clone.go:242`, `:253-255`, `:357`
- necessity: could

```text
In kubernetes-sigs/prow, following PR #1054 — "fix(clonerefs): write merge results before merging partial clones" (merged as 5bf71c2882c9513292618ff13dd6e20973e4a1e6) — extract the repeated partial-clone predicate into one helper.

Context: the "blobless fetch or sparse checkout" condition appears at pkg/pod-utils/clone/clone.go:242 (sparseCheckoutSet := isSparseCheckoutSet(refs)), :253 (base-ref filterArgs), and :357 (partialClone). Keep isSparseCheckoutSet because sparse-only commands still need it.

The task:
- Add func isPartialClone(refs prowapi.Refs) bool next to isSparseCheckoutSet, returning (refs.BloblessFetch != nil && *refs.BloblessFetch) || isSparseCheckoutSet(refs).
- Use it for the base-ref filterArgs condition and the partialClone variable; leave the sparse-specific command conditions using sparseCheckoutSet.
- Add table-driven unit tests for nil/true/false BloblessFetch plus empty/non-empty SparseCheckoutFiles, including both set.

Acceptance criteria:
- No behaviour change: TestCommandsForRefs expectations stay unchanged and pass; go test ./pkg/pod-utils/clone/ ./pkg/clonerefs/ passes; gofmt and go vet are clean.
- The predicate is defined once and used at both partial-clone sites.

Scope guard: no changes to merge behaviour, logging, version gating, or docs; no commits, pushes, or GitHub actions unless asked.
```
