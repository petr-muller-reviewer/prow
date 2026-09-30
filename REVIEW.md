---
pr: kubernetes-sigs/prow#981
title: "fix(hack): keep codegen verification read-only"
head_sha: f9ecdd47f6ba8aa8920a693775c8e2a958435a8f
base: main
reviewed_at: 2026-09-27T12:55:59Z
verdict: approve
---

# Review

## Verdict

Approve. I found no actionable regressions in the revised verifier. It generates in a temporary source tree, compares the updater's output locations, and removes that tree on exit.

## What this PR does

- Runs `update/codegen.sh` from a temporary copy of the working tree so verification does not overwrite checkout files.
- Includes tracked, untracked, and locally ignored files needed for whole-directory comparisons.
- Reuses a copied protoc cache and compares generated API, client, config, gangway, pipeline, plugin, CRD, and proto outputs.
- Removes the temporary tree when generation, comparison, or verification exits.

## Findings

None.

## Checked

- Compared the full PR diff at `f9ecdd47f6ba8aa8920a693775c8e2a958435a8f` with its merge base on `main`.
- Checked the updater's output paths against the verifier's comparisons.
- Confirmed the revised copy includes untracked files, addressing the earlier false failure for files added before staging.
- `bash -n` and `git diff --check` passed. Full code generation was not run.

## Open questions

None.
