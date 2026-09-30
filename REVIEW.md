---
pr: kubernetes-sigs/prow#981
title: "fix(hack): keep codegen verification read-only"
head_sha: b579745c003f1c9ba9438679b1f91588d6a6eea2
base: main
reviewed_at: 2026-09-30T16:25:40Z
verdict: approve
refresh_log:
  - old_sha: f9ecdd47f6ba8aa8920a693775c8e2a958435a8f
    new_sha: b579745c003f1c9ba9438679b1f91588d6a6eea2
    summary: "Filtered deleted source and output paths before tar; derived proto exclusions from output_paths; incorporated reviewer discussion."
---

# Review

## Verdict

Approve. I found no actionable regressions in the revised verifier. It generates in a temporary source tree, compares the updater's output locations, and removes that tree on exit.

## What this PR does

- Runs `update/codegen.sh` from a temporary copy of the working tree so verification does not overwrite checkout files.
- Includes tracked, untracked, and locally ignored files needed for whole-directory comparisons.
- Reuses a copied protoc cache and compares generated API, client, config, gangway, pipeline, plugin, CRD, and proto outputs.
- Removes the temporary tree when generation, comparison, or verification exits.

Since previous review:

- Commit `b579745c0` changed `hack/make-rules/verify/codegen.sh` (+18/−7): skip deleted paths before both tar copies, and derive proto exclusions from `output_paths`.
- On September 27, cblecker commented that deleted tracked paths break tar and that the proto exclusion list duplicated `output_paths`.
- On September 30, petr-muller replied to both threads that the changes were addressed and requested another look in a PR comment.

## Findings

None.

## Checked

- Compared the full PR diff at `f9ecdd47f6ba8aa8920a693775c8e2a958435a8f` with its merge base on `main`.
- Checked the updater's output paths against the verifier's comparisons.
- Confirmed the revised copy includes untracked files, addressing the earlier false failure for files added before staging.
- `bash -n` and `git diff --check` passed. Full code generation was not run.
- Reviewed the 18 additions and 7 deletions in `b579745c0`: existence checks retain symlinks, skip deleted paths in both copies, and the nested loop skips proto files already covered by `output_paths`. `bash -n` and `git diff --check` passed for this commit; full code generation was not run.

## Open questions

None.
