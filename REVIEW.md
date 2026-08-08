---
pr: kubernetes-sigs/prow#661
title: "Add support for git cherry-pick -x style commit messages"
head_sha: 9eb853108e6622d085265d7ac44a077633701d4e
base: main
reviewed_at: 2026-06-09T19:57:29Z
verdict: request-changes
refresh_log:
  - from: 9eb853108e
    to: 9eb853108e
    at: 2026-05-20T12:11:39Z
    summary: initial /review:save; skeleton REVIEW.md generated from manual review history
  - from: 9eb853108e
    to: 9eb853108e
    at: 2026-06-09T19:57:29Z
    summary: "no new commits; @stmcginnis gave /lgtm Jun 5 with one nit (regex as global); nit added to findings"
gate:
  decision: do-not-merge
  gated_at: 2026-08-08T20:15:47Z
  gated_head_sha: 9eb853108e6622d085265d7ac44a077633701d4e
  reviewed_head_sha: 9eb853108e6622d085265d7ac44a077633701d4e
---

## Gate

**Verdict: do-not-merge** (gated 2026-08-08 against `9eb853108e`, unchanged since the review).

No code has landed since the review, so the blocking `os/exec` finding stands verbatim. On its own that is an architectural objection a maintainer could reasonably wave through — but it is what makes the rest of the risk possible, and the rest is what actually gates: the feature is **on by default** for every existing cherrypicker deployment, it runs an unbounded `git rebase -i` over every cherry-pick inside a cached shared clone, and **every failure mode is warn-only** — the PR is still pushed and opened as if nothing went wrong. Operators who never asked for this feature inherit its failure modes silently. The two unblockers are small and do not require abandoning the rebase approach.

### Retracted from the original review

- **`[blocking]` shell injection via unquoted variables** — **not reproducible at this head.** The generated script at `server.go:851-855` does quote `"$ORIGINAL_SHA"` and `"$CURRENT_MSG"`. The only unquoted expansion left is `-f$COMMIT_NUM`, fed from `git rev-list --count` (always a bare integer), and `baseSHA` is `git rev-parse` output (hex). This finding was wrong when written — it does not gate merge, and should not be raised on the PR.

### Gating list

- `cmd/external-plugins/cherrypicker/main.go:76` (REVIEW.md `should-fix`) — `--add-original-commit-id` defaults `true`. Not addressed. Both sibling feature flags (`--allow-all`, `--create-issue-on-conflict`) default `false`. **Blocks:** flip to `false`, or justify default-on with a release note.
- `cmd/external-plugins/cherrypicker/server.go:646` (independent) — `appendCherryPickMessages` failure logs `Warn` and falls through to `p.Push` at `:656`. A failed `--exec` step leaves `git rebase -i` stopped mid-flight with detached HEAD and `.git/rebase-merge` in the cached clone; `PushToNamedFork` pushes by branch name, so the un-amended commits are published and the requester sees a normal PR. **Blocks:** `git rebase --abort` on failure and surface it (PR comment or hard error).
- `cmd/external-plugins/cherrypicker/server.go:25,838-878` (REVIEW.md `blocking`) — raw `exec.Command` bypasses the `pkg/git/v2` `Interactor`. Not addressed. **Does not independently block** if the two above are fixed, but it is the root of findings 3-6 below and the patch-modification approach in the original review's open question removes all of them at once.

### Independent merge risk

Diff is confined to `cmd/external-plugins/cherrypicker/` (`main.go` +23, `server.go` +110, `server_test.go` +236). No repo skill applies — the two overlaid skills are review/triage workflows, not compatibility checkers.

- **API:** none. `package main`; no exported surface, no CRD, no proto, no config-file schema change.
- **Configuration:** one new flag, `--add-original-commit-id`, defaulting `true`. Blast radius: every cherrypicker instance in every Prow deployment starts rewriting cherry-pick commit messages on upgrade with no opt-in. Not gated, not release-noted.
- **Behavioral:** cherry-pick commit bodies change shape (trailer appended) for all users; `--amend -m` applies `cleanup=whitespace`, so bodies are additionally reflowed (trailing whitespace stripped, consecutive blank lines collapsed) — real `git cherry-pick -x` preserves them byte-for-byte (`server.go:853`).
- **Environment sensitivity:** `GIT_CONFIG_NOSYSTEM=1` only suppresses `/etc/gitconfig`; `$HOME/.gitconfig` still applies (`server.go:878`). `commit.gpgsign=true` breaks the in-script amend on every cherry-pick; `rebase.autoSquash=true` makes `rebase -i` reorder and squash any `fixup!`/`squash!` commit, so the cherry-pick PR would silently carry fewer commits than the source PR. Recommend `GIT_CONFIG_GLOBAL=/dev/null`.
- **Reliability:** `extractOriginalSHAs` uses `bufio.NewScanner` with no `Buffer()` (`server.go:810`) — `ErrTooLong` on any patch line over 64 KiB (lockfiles, minified assets, base64 fixtures), which silently disables the trailer for those PRs. The rebase subprocess has no `context` timeout and runs while holding the per-org/repo `lock.Lock()`, so a wedged `git` stalls that repo's cherry-pick queue indefinitely (`server.go:878`). The 0755 temp script also fails on a `noexec` `TMPDIR`, and `--exec` shell-parses its path (`server.go:872`).

### Reviewer state

@stmcginnis gave `/lgtm` twice (Apr 23, Jun 5) and dispositioned his own regex nit as "fine to leave it" (Jun 9), explicitly deferring to @petr-muller: "we'll see what others think as well." No `CHANGES_REQUESTED` on the PR. The gate is therefore driven entirely by the local review plus this pass, not by unresolved reviewer feedback.

## Findings

### [blocking] os/exec bypasses pkg/git/v2 abstraction layer
- where: `cmd/external-plugins/cherrypicker/server.go:25,796-833`
- concern: Two raw `exec.Command("git", ...)` calls added — one for `git rev-parse HEAD~N` and one for `git rebase -i --exec`. Every other git operation in this file goes through the `Interactor` interface (`r.Am()`, `r.CheckoutNewBranch()`, etc.) backed by `pkg/git/v2/executor.go`, which provides logging, credential censoring, and consistent error handling. These calls bypass all of that. The patch-modification approach (modify the mbox file before `r.Am()`) would require zero additional git commands.
- note: @stmcginnis gave /lgtm without flagging this. Unaddressed.
- excerpt: |
    baseCmd := exec.Command("git", "-C", repoPath, "rev-parse", fmt.Sprintf("HEAD~%d", numCommits))
    ...
    cmd := exec.Command("git", "-C", repoPath, "rebase", "-i", baseSHA, "--exec", tmpPath)

### [blocking] shell injection via unquoted variables in generated exec script
- where: `cmd/external-plugins/cherrypicker/server.go:800-812`
- concern: Three injection vectors in the generated shell script: (1) `$ORIGINAL_SHA` unquoted in `if [ -n $ORIGINAL_SHA ]` — word-splits on whitespace; (2) `$CURRENT_MSG` unquoted in the `git commit --amend -m` argument — a commit message containing backticks, `$()`, or double quotes executes as shell code; (3) `baseSHA` from `git rev-parse` output interpolated into the script body via `fmt.Sprintf` without any validation. The SHA regex in `extractOriginalSHAs` only protects the `ORIGINAL_SHAS` env var, not `baseSHA` (a different value) or `CURRENT_MSG`.
- note: @stmcginnis gave /lgtm without flagging this. Unaddressed.
- excerpt: |
    ORIGINAL_SHA=$(echo "$ORIGINAL_SHAS" | cut -d',' -f$COMMIT_NUM)
    if [ -n $ORIGINAL_SHA ]; then
        CURRENT_MSG=$(git log -1 --pretty=%%B)
        git commit --amend -m "$CURRENT_MSG

    (cherry picked from commit $ORIGINAL_SHA)"
    fi

### [should-fix] flag defaults to true — silent behavior change for existing deployments
- where: `cmd/external-plugins/cherrypicker/main.go:76`
- concern: `fs.BoolVar(&o.addOriginalCommitID, "add-original-commit-id", true, ...)`. Every existing cherrypicker deployment begins rewriting cherry-pick commit messages on upgrade without any opt-in. All other new-feature flags in this file (`--allow-all`, `--create-issue-on-conflict`) default to `false`.
- note: @stmcginnis gave /lgtm without flagging this. Unaddressed.
- excerpt: |
    fs.BoolVar(&o.addOriginalCommitID, "add-original-commit-id", true, "Add original commit ID...")

### [nit] fromPattern regex compiled per call — could be package-level var
- where: `cmd/external-plugins/cherrypicker/server.go:811`
- concern: `fromPattern := regexp.MustCompile(...)` inside `extractOriginalSHAs` recompiles the regex on every call. Since the pattern never changes, it could be a package-level `var`. @stmcginnis flagged this (Jun 5) and immediately said "relatively minor overhead though, so I'm not too concerned about it."
- excerpt: |
    fromPattern := regexp.MustCompile(`^From ([0-9a-f]{40}) `)

### [nit] feature failures silently swallowed
- where: `cmd/external-plugins/cherrypicker/server.go:578-594`
- concern: Both SHA extraction failure and rebase failure log `Warn` and continue. The cherry-pick PR is created without the expected commit IDs and the requester has no indication anything went wrong. At minimum, a PR comment on failure would provide visibility.
- excerpt: |
    logger.WithError(err).Warn("Failed to extract original SHAs from patch")
    ...
    logger.WithError(err).Warn("Failed to append cherry-pick messages")

### [nit] tests use os/exec instead of localgit helpers
- where: `cmd/external-plugins/cherrypicker/server_test.go:1540-1695`
- concern: `TestAppendCherryPickMessages_SingleCommit` and `TestAppendCherryPickMessages_MultiCommit` set up git repos via `exec.Command("git", ...)` directly. The rest of the test file uses `localgit.Clients` from `pkg/git/localgit`. Minor style inconsistency; not wrong, just diverges from project convention.

### [question] would patch-modification approach be acceptable?
- where: `cmd/external-plugins/cherrypicker/server.go:559` (the r.Am() call site)
- concern: The mbox patch already contains `From <sha> Mon Sep 17 00:00:00 2001` headers before `r.Am()` is called. Modifying the patch file in-place between `getPatch()` and `r.Am()` — inserting `(cherry picked from commit <sha>)` before the `diff --git` separator in each commit section — would eliminate both blocking findings: no os/exec, no shell script, no injection surface. Pure Go string manipulation, trivially testable. Wanted to surface this as a question rather than a hard blocker since the current approach does work.

## Checked
- `extractOriginalSHAs` regex `^From ([0-9a-f]{40}) ` correctly validates 40-char hex SHAs and avoids false positives from email-style `From ` headers in commit message bodies
- Temp script file is properly closed before passing path to `git rebase` — both happy path (`tmpfile.Close()`) and error path (`tmpfile.Close()` before return)
- `defer os.Remove(tmpPath)` correctly cleans up temp script after rebase
- `GIT_SEQUENCE_EDITOR=true` correctly suppresses the interactive editor for `git rebase -i`
- `GIT_CONFIG_NOSYSTEM=1` prevents interference from system git config during rebase
- Flag name `--add-original-commit-id` follows existing flag naming conventions in main.go
- `TestExtractOriginalSHAs` covers key cases: single commit, multi-commit, invalid From lines filtered by regex, empty patch returns error
- `TestAppendCherryPickMessages_MultiCommit` verifies correct SHA ordering (commit N gets SHA N)
- `TestAppendCherryPickMessages_NoGitRepo` verifies error path when repo path is invalid
- Server struct field `addOriginalCommitID bool` correctly wired through main.go to server initialization

## Open questions
- If the patch-modification approach is acceptable, it would resolve both blocking findings without requiring changes to the existing test structure — worth discussing before another round of revisions.
- If staying with the rebase approach: quoting `"$ORIGINAL_SHA"` and `"$CURRENT_MSG"` in the shell script would close most of the injection surface for the shell injection finding — would that targeted fix be acceptable?
- Why does `extractOriginalSHAs` return an error when zero SHAs are found, rather than returning an empty slice? The caller at server.go:582 checks `err != nil` before calling `appendCherryPickMessages`, so an empty slice would naturally produce a no-op. The current error forces the caller to treat "no SHAs in patch" as a warning condition even for legitimate single-file-change PRs with no `From` lines — or is the assumption that every patch from GitHub's API always has at least one?
