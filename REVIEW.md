---
pr: kubernetes-sigs/prow#661
title: "Add support for git cherry-pick -x style commit messages"
head_sha: 9eb853108e6622d085265d7ac44a077633701d4e
base: main
reviewed_at: 2026-08-08T22:50:25Z
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
  - from: 9eb853108e
    to: 9eb853108e
    at: 2026-08-08T22:50:25Z
    summary: "no new commits or PR activity; independent re-read added 7 findings (2 blocking: warn-only failure path publishes un-amended commits, 64 KiB scanner cap); shell-injection blocking retracted as not reproducible; old silently-swallowed nit superseded; stale line numbers corrected; gate recorded"
gate:
  decision: do-not-merge
  gated_at: 2026-08-08T20:15:47Z
  gated_head_sha: 9eb853108e6622d085265d7ac44a077633701d4e
  reviewed_head_sha: 9eb853108e6622d085265d7ac44a077633701d4e
---

## Gate

**Verdict: do-not-merge** (gated 2026-08-08 against `9eb853108e`, unchanged since the review).

No code has landed since the review. **The feature can fail for ordinary reasons and reports success anyway** — that is the gate. When the annotation step fails, `handle` logs a `Warn` and proceeds to force-push the un-amended commits, producing a cherry-pick PR indistinguishable from a successful one. There is at least one config-independent trigger for that path (the 64 KiB scanner cap), so this is not hypothetical. The default-`true` flag is *not* the problem — it only decides how many deployments meet the problem — and flipping it to `false` on its own would fix nothing.

### Gating list

1. **Silent failure: the feature no-ops and publishes as if it worked** — `server.go:646`. `appendCherryPickMessages` (and `extractOriginalSHAs` before it) failures log `Warn` and fall through to `p.Push` at `:656`. A failed `--exec` step leaves `git rebase -i` stopped mid-flight with HEAD detached and the branch ref still at the pre-rebase tip; `Push` → `PushToNamedFork(fork, newBranch, force=true)` (`server.go:128-129`) pushes by branch name, so the **un-amended** commits are force-pushed and the requester sees a normal PR. Nothing but a log line records that the feature did nothing. **Blocks:** surface the failure (PR comment, or return the error) instead of proceeding.
2. **64 KiB scanner cap — a config-independent trigger for (1)** — `server.go:810`. `extractOriginalSHAs` uses `bufio.NewScanner` with no `Buffer()`, so `ErrTooLong` on any patch line over 64 KiB — routine for PRs touching lockfiles, minified assets, or base64 fixtures. The trailer is then silently absent. **Blocks:** `scanner.Buffer(...)`, or match the `From ` prefix without full-line buffering.

Not independently blocking, but in scope:

- `server.go:25,838-878` (REVIEW.md `blocking`) — raw `exec.Command` bypasses the `pkg/git/v2` `Interactor`. Unaddressed. This is an architectural objection, and it is the root of the hardening items below; the patch-modification approach in the original review's open question removes the whole class at once. On its own a maintainer could reasonably wave it through.
- `main.go:76` (REVIEW.md `should-fix`) — `--add-original-commit-id` defaults `true`. This is an **exposure multiplier, not a defect**: it decides whether (1) and (2) land on operators who opted in or on everyone who upgrades. Fix (1) and (2) and default-`true` is defensible with a release note; leave them and `false` merely limits the blast radius. Note the convention argument is weaker than the original review implied — `--allow-all` and `--create-issue-on-conflict` default `false`, but `--use-prow-assignments` (`main.go:73`) defaults `true`.

### Retracted from the original review

- **`[blocking]` shell injection via unquoted variables** — **not reproducible at this head.** The generated script at `server.go:851-855` does quote `"$ORIGINAL_SHA"` and `"$CURRENT_MSG"`. The only unquoted expansion left is `-f$COMMIT_NUM`, fed from `git rev-list --count` (always a bare integer), and `baseSHA` is `git rev-parse` output (hex). This finding was wrong when written — it does not gate merge, and should not be raised on the PR.

### Independent merge risk

Diff is confined to `cmd/external-plugins/cherrypicker/` (`main.go` +23, `server.go` +110, `server_test.go` +236). No repo skill applies — the two overlaid skills are review/triage workflows, not compatibility checkers.

- **API:** none. `package main`; no exported surface, no CRD, no proto, no config-file schema change.
- **Configuration:** one new flag, `--add-original-commit-id`, defaulting `true`. Every cherrypicker instance starts annotating cherry-pick commit messages on upgrade with no opt-in. Not release-noted (no release-note block in the PR body, no `release-note*` label, and no docs in-tree mention cherrypicker flags at all).
- **Behavioral:** cherry-pick commit bodies gain the trailer for all users; additionally `--amend -m` applies `cleanup=whitespace`, so bodies are reflowed (trailing whitespace stripped, consecutive blank lines collapsed) — real `git cherry-pick -x` preserves them byte-for-byte (`server.go:853`). Minor, but always happens.
- **Hardening, config-dependent — flagged because `GIT_CONFIG_NOSYSTEM=1` was clearly meant to isolate the rebase and does not finish the job, not because these are expected failures.** A standard containerized cherrypicker has no `$HOME/.gitconfig`, so these need an unusual deployment: `GIT_CONFIG_NOSYSTEM=1` suppresses only `/etc/gitconfig`, leaving `commit.gpgsign=true` (breaks the in-script amend) and `rebase.autoSquash=true` (reorders/squashes `fixup!`/`squash!` commits, so the cherry-pick PR carries fewer commits than the source — also needs such commits in the source PR) live. `GIT_CONFIG_GLOBAL=/dev/null` closes both. Same category: no `context` timeout on the rebase subprocess (`server.go:878`), which leaks the goroutine and holds the request's `sync.Mutex` — scoped to one `cherryPickRequest{org, repo, num, targetBranch}` (`server.go:117,533`), not the repo's queue; and the 0755 temp script failing on a `noexec` `TMPDIR` with `--exec` shell-parsing its path (`server.go:872`).
- **Not a risk (checked):** the rebase does not run in shared state. `ClientFor` → `ClientForWithRepoOpts(org, repo, RepoOpts{})` (`pkg/git/v2/client_factory.go:425`) refreshes the persistent primary mirror under `cacheDir`, then makes a **fresh per-call secondary clone** in `os.MkdirTemp(*defaultTempDir(), "gitrepo")` (`:450-472`); `r.Directory()` is that temp clone, and `defer r.Clean()` (`server.go:562`) deletes it. `ShareObjectsWithPrimaryClone` is false under `RepoOpts{}`, so there are no shared alternates either. A left-behind `.git/rebase-merge` is discarded with the directory — no cache poisoning, no cross-request contamination.

### Reviewer state

@stmcginnis gave `/lgtm` twice (Apr 23, Jun 5) and dispositioned his own regex nit as "fine to leave it" (Jun 9), explicitly deferring to @petr-muller: "we'll see what others think as well." No `CHANGES_REQUESTED` on the PR. The gate is therefore driven entirely by the local review plus this pass, not by unresolved reviewer feedback.

## Findings

### [blocking] annotation failure is swallowed and the un-amended commits are published anyway
- where: `cmd/external-plugins/cherrypicker/server.go:638-656`
- concern: Both `extractOriginalSHAs` and `appendCherryPickMessages` failures log `Warn` and fall through to `p.Push(r, newBranch, true)` at `:656`. A failed `--exec` step leaves `git rebase -i` stopped mid-flight with HEAD detached and the branch ref still at the pre-rebase tip, so `Push` → `PushToNamedFork(fork, newBranch, force=true)` (`:128-129`) force-pushes the **un-amended** commits. The requester gets a cherry-pick PR indistinguishable from a successful one; only a log line records that the feature did nothing. This is what converts every other defect below from visible breakage into quiet wrong output.
- precedent: the package being bypassed already handles this correctly for the same operation — `mergeRebase` (`pkg/git/v2/interactor.go:368-396`) runs `rebase --abort`, restores the pre-rebase HEAD via `Checkout(headRev)`, and propagates the outcome; `Am` (`:428-435`) does the same with `am --abort`.
- fix: abort the rebase and surface the failure (PR comment, or return the error) instead of proceeding to push.
- note: supersedes the earlier `[nit] feature failures silently swallowed`, which under-rated this.
- excerpt: |
    if err := appendCherryPickMessages(r.Directory(), originalSHAs); err != nil {
        logger.WithError(err).Warn("Failed to append cherry-pick messages")
    } else {
    ...
    if err := p.Push(r, newBranch, true); err != nil {

### [blocking] 64 KiB line cap silently disables the feature
- where: `cmd/external-plugins/cherrypicker/server.go:810`
- concern: `bufio.NewScanner` with no `Buffer()` call fails with `bufio.ErrTooLong` on any line over 64 KiB. GitHub `.patch` bodies routinely contain such lines — single-line JSON/lockfiles, minified assets, base64 fixtures, embedded SVG. `scanner.Err()` then returns non-nil, `extractOriginalSHAs` returns an error, and the caller only warns — so cherry-picks of any PR touching a long-line file lose the trailer with no user-visible signal. This is a config-independent trigger for the finding above; no unusual deployment required.
- fix: `scanner.Buffer(make([]byte, 0, 64*1024), maxPatchLine)`, or match the `From ` prefix without full-line buffering.
- excerpt: |
    scanner := bufio.NewScanner(file)

### [should-fix] rebase inherits ambient git configuration
- where: `cmd/external-plugins/cherrypicker/server.go:879`
- concern: `GIT_CONFIG_NOSYSTEM=1` suppresses only `/etc/gitconfig`; `$HOME/.gitconfig` and `$XDG_CONFIG_HOME/git/config` still apply. Two concrete misbehaviors: `commit.gpgsign=true` makes the in-script `git commit --amend` fail with no signing key in the pod, breaking the trailer on every cherry-pick (→ blocking finding above); `rebase.autoSquash=true` makes `git rebase -i` reorder and squash any commit whose subject starts with `fixup!`/`squash!`, so the cherry-pick PR silently carries fewer commits than the source PR.
- severity note: hardening, not an expected failure — a standard containerized cherrypicker has no `$HOME/.gitconfig`, and the autosquash case additionally needs such commits in the source PR. Flagged because `GIT_CONFIG_NOSYSTEM` was clearly meant to isolate the rebase and does not finish the job.
- fix: `GIT_CONFIG_GLOBAL=/dev/null`, or explicit `-c commit.gpgsign=false --no-autosquash --no-autostash --no-rebase-merges`.
- excerpt: |
    cmd.Env = append(os.Environ(), origEnv, "GIT_SEQUENCE_EDITOR=true", "GIT_CONFIG_NOSYSTEM=1")

### [should-fix] unbounded subprocess, no context timeout
- where: `cmd/external-plugins/cherrypicker/server.go:878`
- concern: `exec.Command` with no `context.WithTimeout`. A wedged `git` (index.lock contention, a hung hook, an editor invocation with no tty) leaks the goroutine and holds the request's `sync.Mutex` forever, blocking re-requests of that cherry-pick. Scope is one `cherryPickRequest{org, repo, num, targetBranch}` (`:117,533`), not the repo's whole queue.
- fix: `exec.CommandContext` with a bounded deadline.

### [nit] temp script needs an executable filesystem and its path is shell-parsed
- where: `cmd/external-plugins/cherrypicker/server.go:859-878`
- concern: `git rebase --exec <cmd>` runs `<cmd>` through `sh -c`. The 0755 script in `os.CreateTemp("")` therefore fails to run if `TMPDIR` is mounted `noexec` (a common hardening for pod `emptyDir` scratch), and any whitespace or shell metacharacter in `TMPDIR` breaks the command line. Passing the logic as an inline `sh -c` string with the interpreter explicit avoids both.

### [nit] `--amend -m` normalizes the original commit message
- where: `cmd/external-plugins/cherrypicker/server.go:853`
- concern: `-m` applies `cleanup=whitespace`, which strips trailing whitespace and collapses consecutive blank lines. The cherry-picked commit body therefore differs from the source commit beyond the added trailer — a message with deliberate double blank lines between paragraphs gets reflowed. Real `git cherry-pick -x` preserves the body byte-for-byte. Minor, but it happens on every cherry-pick.
- fix: `--amend --file=- --cleanup=verbatim`, or `--no-edit` plus a trailer append.

### [nit] new git tests are not hermetic
- where: `cmd/external-plugins/cherrypicker/server_test.go:1563,1616`
- concern: the `run` helper builds `cmd.Env` from bare `os.Environ()` with no `GIT_CONFIG_GLOBAL`/`GIT_CONFIG_NOSYSTEM`, and `appendCherryPickMessages` under test sets only `GIT_CONFIG_NOSYSTEM`. On a developer machine with `commit.gpgsign=true`, `core.hooksPath`, or `rebase.autoSquash` in `~/.gitconfig`, `TestAppendCherryPickMessages_SingleCommit` and `_MultiCommit` fail for reasons unrelated to the code.
- excerpt: |
    cmd.Env = append(os.Environ(),
        "GIT_SEQUENCE_EDITOR=true",
        "GIT_EDITOR=true",
    )

### [blocking] os/exec bypasses pkg/git/v2 abstraction layer
- where: `cmd/external-plugins/cherrypicker/server.go:25,839,878`
- concern: Two raw `exec.Command("git", ...)` calls added — one for `git rev-parse HEAD~N` (`:839`) and one for `git rebase -i --exec` (`:878`). At the PR's base (`f65e1b3c3`) there is no `exec.Command` anywhere under `cmd/external-plugins/cherrypicker/`, tests included: every git operation goes through the `Interactor` (`r.Am()`, `r.CheckoutNewBranch()`, `r.Config()`, `PushToNamedFork`) backed by `pkg/git/v2/executor.go`, which provides logging, credential censoring, and consistent error handling. The plugin already holds a `RepoClient` for this exact repo (`r`, from `s.gc.ClientFor` at `:557`) and then reaches around it with `exec.Command("git", "-C", r.Directory(), ...)`.
- scope note: Prow does shell out to git elsewhere (`cmd/generic-autobumper/bumper/bumper.go:354,468`, `pkg/clonerefs/run.go:204`, `pkg/pod-utils/clone/clone.go:423`), but those run where no client factory is available — a standalone CLI, or inside a test pod. This is the opposite case. The claim is "this plugin has a client in hand and bypasses it," not "Prow never shells out to git."
- how much is actually bespoke: `rev-parse` is a drop-in — `Interactor.RevParse(commitlike)` (`pkg/git/v2/interactor.go:45,218`) takes an arbitrary commitlike, so `r.RevParse(fmt.Sprintf("HEAD~%d", numCommits))` works today (it returns output untrimmed, so keep the existing `TrimSpace`). `rebase -i --exec` genuinely is not covered: there is no exported generic `Run` — `executor.Run(args...)` (`pkg/git/v2/executor.go:29-31`) is unexported and held as a private field — and the closest existing helper, `mergeRebase` (`:368`), is unexported and semantically a merge strategy. Adding a `Rebase(...)` method to a shared interface for one external plugin is a real design ask, which is the strongest argument for the current shape.
- note: @stmcginnis gave /lgtm without flagging this. Unaddressed. Never discussed anywhere in PR #661 — zero mentions of the interactor, the abstraction, or `os/exec` across all inline comments, issue comments, and reviews.
- excerpt: |
    baseCmd := exec.Command("git", "-C", repoPath, "rev-parse", fmt.Sprintf("HEAD~%d", numCommits))
    ...
    cmd := exec.Command("git", "-C", repoPath, "rebase", "-i", baseSHA, "--exec", tmpPath)

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

### [nit] tests use os/exec instead of localgit helpers
- where: `cmd/external-plugins/cherrypicker/server_test.go:1540-1695`
- concern: `TestAppendCherryPickMessages_SingleCommit` and `TestAppendCherryPickMessages_MultiCommit` set up git repos via `exec.Command("git", ...)` directly. The rest of the test file uses `localgit.Clients` from `pkg/git/localgit`. Minor style inconsistency; not wrong, just diverges from project convention.

### [question] would patch-modification approach be acceptable?
- where: `cmd/external-plugins/cherrypicker/server.go:618` (the r.Am() call site)
- concern: The mbox patch already contains `From <sha> Mon Sep 17 00:00:00 2001` headers before `r.Am()` is called. Inserting `(cherry picked from commit <sha>)` just above each block's `---` separator — pure Go string manipulation on a file already on disk — removes both `exec.Command` calls, the temp script, the ambient-config sensitivity, the missing timeout, the `HEAD~N` coupling, and the `cleanup=whitespace` reflow at once, and makes the tests pure string fixtures. It also fixes the top blocking finding structurally: the failure happens *before* anything is applied or pushed, so degrading is a genuine no-op rather than a half-finished rebase whose commits get published.
- provenance: this is @dhiller's second option from #500 ("modify the patch before application so that each part contains the original commit id"). Never discussed in PR #661 — zero mentions of the patch, mbox, `format-patch`, or `git am` across the whole conversation. The @petr-muller guidance of 2026-02-23 that scoped the work pointed at the other option (extract SHAs from the patch, amend the applied commits), which is what the author built.
- disposition: raise as a follow-up, not a precondition for merge — both blocking findings are fixable inside the current rebase design, and the author has been through eight review rounds on the approach they were directed to.
- known edges if pursued: a bare `---` inside a commit body would be taken as the separator (`git am` shares this ambiguity); base64/quoted-printable transfer-encoded bodies are not line-editable and need a guard or a decode step (the rebase approach is immune here).

## Resolved
Entries no longer active. Kept as history; do not raise on the PR.

### retracted: shell injection via unquoted variables in generated exec script
- retracted_at: 2026-08-08
- where: `cmd/external-plugins/cherrypicker/server.go:846-857`
- reason: not reproducible at this head. The generated script quotes both `"$ORIGINAL_SHA"` and `"$CURRENT_MSG"`. The only unquoted expansion is `-f$COMMIT_NUM`, fed from `git rev-list --count` (always a bare integer); `baseSHA` is `git rev-parse` output (hex). The finding was wrong when written, against this same SHA.
- original concern: Three injection vectors claimed: (1) `$ORIGINAL_SHA` unquoted in `if [ -n $ORIGINAL_SHA ]`; (2) `$CURRENT_MSG` unquoted in the `git commit --amend -m` argument; (3) `baseSHA` interpolated into the script body via `fmt.Sprintf` without validation.

### superseded: feature failures silently swallowed
- superseded_at: 2026-08-08
- superseded_by: `[blocking] annotation failure is swallowed and the un-amended commits are published anyway`
- where: `cmd/external-plugins/cherrypicker/server.go:638-653`
- reason: rated `nit` on the grounds of missing visibility only. It is worse than that — the un-amended commits are force-pushed and the PR opens looking successful.

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
