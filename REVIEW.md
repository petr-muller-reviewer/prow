---
pr: kubernetes-sigs/prow#661
title: "Add support for git cherry-pick -x style commit messages"
head_sha: 1d35794f7752d9a169c08478ae6b5125368075fb
base: main
reviewed_at: 2026-09-29T17:23:02Z
verdict: request-changes
gate:
  decision: do-not-merge
  gated_at: 2026-09-29T17:48:12Z
  gated_head_sha: 1d35794f7752d9a169c08478ae6b5125368075fb
  reviewed_head_sha: 1d35794f7752d9a169c08478ae6b5125368075fb
refresh_log:
  - old_sha: 05052e23ba428d40595917f28f2236f488b5b3a5
    new_sha: 1d35794f7752d9a169c08478ae6b5125368075fb
    summary: "Reviewed explicit Git identity arguments and updated helper tests; long-line patch finding remains open."
---

# Review

## Gate

**Decision: do-not-merge.** At PR head `1d35794f7752d9a169c08478ae6b5125368075fb`, the saved blocking finding remains unchanged. Enabling the new flag makes a valid patch with a line over `bufio.Scanner`'s default 64 KiB limit fail after `git am`, so the cherry-pick branch is never pushed. Handle long patch lines and add a regression test before reconsidering merge.

### Gating finding

- **Not addressed — `REVIEW.md`, “SHA extraction rejects valid long-line patches”:** `cmd/external-plugins/cherrypicker/server.go:850-860` still uses an unconfigured `bufio.Scanner` and returns its token-too-long error. This blocks merge.

### Other review feedback

- The substantive GitHub comments about the default, error path, rollback, timeout, deprecated `filter-branch`, and local Git test helpers are addressed in the current code. The rebase still logs raw subprocess output internally at `cmd/external-plugins/cherrypicker/server.go:944-950`, while the requester receives a generic error at lines 658-664; this local-only subprocess does not independently gate merge, but deployments with hooks should avoid emitting secrets to its output.

### Independent merge risk

- The diff changes no exported Go API, Kubernetes API, CRD, or config-file schema. `--add-original-commit-id` defaults to `false` at `cmd/external-plugins/cherrypicker/main.go:76`, so existing deployments retain their current behavior.
- Only operators enabling the flag incur a rebase/amend per commit, new commit hashes and bodies, and sensitivity to Git hooks and global configuration. The flag and these operational effects are not yet documented in the cherrypicker documentation; document them before rollout.

## Verdict

Request changes. SHA extraction still uses `bufio.Scanner` without a larger buffer, so an otherwise valid patch with a line over 64 KiB prevents the enabled cherry-pick from being pushed.

## What this PR does

- Adds `--add-original-commit-id`, disabled by default.
- Extracts source commit SHAs from the downloaded mbox patch.
- Rewrites applied commit messages with `git cherry-pick -x`-style trailers before pushing.
- Aborts and restores the local checkout if rewriting fails.

Since previous review:

- The author changed the rewrite helper to receive the already configured Git name and email from `Server.handle`, avoiding separate `git config` reads, and updated its tests. The changed Go files total 7 additions and 29 deletions.
- The PR branch removed previously committed `REVIEW.md` and `REVIEW.html` artifacts. AaruniAggarwal commented on 2026-09-29 at 17:21:02Z that the review comments had been addressed; there were no new inline comments or submitted reviews.

## Findings

### [blocking] SHA extraction rejects valid long-line patches
- where: `cmd/external-plugins/cherrypicker/server.go:850-860`
- concern: `bufio.Scanner` has a 64 KiB maximum token size by default. A valid GitHub mbox patch containing a long generated, minified, or lockfile line makes `extractOriginalSHAs` fail; with `--add-original-commit-id`, the handler then refuses to push an otherwise successful cherry-pick branch.
- excerpt: |
    scanner := bufio.NewScanner(file)
    for scanner.Scan() {
        line := scanner.Text()
        if matches := fromPattern.FindStringSubmatch(line); matches != nil {
            shas = append(shas, matches[1])
        }
    }
    if err := scanner.Err(); err != nil {
        return nil, fmt.Errorf("error reading patch file: %w", err)
    }

## Checked

- The new option defaults to false, preserving existing cherrypicker behavior unless explicitly enabled.
- The updated helper receives the same name and email that `Server.handle` configures on the repository before `git am`; the helper tests pass explicit values.
- Rebase failures are aborted and reset to the pre-rebase HEAD before returning, and no branch is pushed after an extraction/rewrite failure.
- Commit messages are written through a temporary file rather than interpolated into the generated shell script.
- Unit coverage exercises empty input, invalid history, single- and multi-commit trailer ordering, and rebase rollback.

## Open questions

- Can the follow-up add a handler-level test for an enabled multi-commit mbox patch, plus regression coverage for a patch line larger than 64 KiB?
- Should the cherrypicker documentation describe `--add-original-commit-id`, its default-disabled rollout, and the changed commit hashes/messages when enabled?
