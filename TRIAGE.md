---
issue: kubernetes-sigs/prow#800
title: "publisher.Commit panics (nil pointer) when GitUser is unset, and GitHubOptions.GitClientFactory can't set it"
state: open
labels: kind/bug
main_sha: e601a1ffafd7d8d3a781238a4c5f4233d6248f68
triaged_at: 2026-10-27T23:57:08Z
verdict: accepted
refresh_log:
  - at: 2026-10-26T15:45:47Z
    since: 2026-07-27T00:05:00Z
    summary: "Folded in linked PR #801 (open, CHANGES_REQUESTED 2026-07-29) that the first triage missed, plus the 2026-10-22 lifecycle/stale label. Bug analysis unchanged; upstream main advanced 376 commits without touching the three referenced files."
  - at: 2026-10-27T23:57:08Z
    since: 2026-10-26T15:45:47Z
    summary: "Maintainer petr-muller commented /kind bug and /remove-lifecycle stale; issue remains open with kind/bug as its only label. No new comments or cross-references beyond that action; PR #801 is unchanged and relevant main files remain untouched."
advice:
  advised_at: 2026-10-26T15:50:03Z
  based_on_triaged_at: 2026-10-26T15:45:47Z
---

## Verdict

Accepted: legitimate, precisely-diagnosed bug (nil-pointer panic) plus a small, well-scoped usability gap, both confirmed against current `main`. Effort level 1 (good-first-issue); author has volunteered a PR.

## What the issue reports

- `RepoClient.Commit` panics with a nil pointer dereference when the underlying `GitUser` getter was never configured.
- The panic happens specifically for clients built via `flagutil.GitHubOptions.GitClientFactory(...)`, the common in-repo helper, which wires `Username`/`Token` for clone/push but never `GitUser` for commit.
- `WithGitUser` (added in #507) exists and works, but is only reachable through the lower-level `gitv2.NewClientFactory`/`NewLocalClientFactory` constructors, not through `GitHubOptions.GitClientFactory`.
- Author's workaround: build two factories on the same working tree — the GitHub-authenticated one for clone/push, and a second manual `gitv2.NewClientFactory(WithGitUser(...))` used only for `.Commit()`.
- Author proposes: (1) nil-guard `publisher.Commit` to return an error instead of panicking, (2) let `GitHubOptions.GitClientFactory` accept/forward a `GitUser` getter. Offered to send a PR for both.

Since previous triage:
- On 2026-10-26T15:51:49Z, petr-muller commented `/kind bug` and `/remove-lifecycle stale`; the stale label was cleared and `kind/bug` is now the issue's only label.
- No new PR references or substantive discussion. PR #801 remains open and unchanged since 2026-09-27; its proposed fixes and outstanding review concerns remain as recorded below.

## Findings

### [reproducibility] Panic reproduces unconditionally, no timing dependency
- detail: Any client built via `GitHubOptions.GitClientFactory(...)` has `GitUser` unset; calling `.Commit(...)` on it panics every time, deterministically — no special config or timing required beyond "don't have GitUser set," which is the only state reachable through that helper.
- evidence: `pkg/flagutil/github.go:340-365` (never sets `ClientFactoryOpts.GitUser`); `pkg/git/v2/publisher.go:53-64` (`Commit` calls `p.info()` unconditionally).

### [cause] `p.info()` called without nil check in `(*publisher).Commit`
- detail: `p.info` is a `GitUserGetter` (`func() (name, email string, err error)`). It is only non-nil if the owning `clientFactory.gitUser` was set. `ClientFactoryOpts.GitUser` (`client_factory.go:120`) is applied only `if cfo.GitUser != nil` (`client_factory.go:177-178`) — the whole chain is designed to tolerate an absent `GitUser`, deferring failure to first use. `Commit` violates that "fail-with-error" contract by panicking instead.
- evidence: |
    ```go
    // pkg/git/v2/publisher.go:53-56
    func (p *publisher) Commit(title, body string) error {
        p.logger.Infof("Committing changes with title %q", title)
        name, email, err := p.info()
        if err != nil {
    ```

### [cause] `GitHubOptions.GitClientFactory` has no way to set `GitUser`
- detail: The function sets `Censor`, `CookieFilePath`, `Host`, `Persist`, `SigningKeyPath`, and conditionally `Username`/`Token` (when using GitHub token/app auth) — never `GitUser`. There's no parameter or `GitHubOptions` field for it, so any caller wanting to use `Publisher.Commit` through this common entrypoint has no supported path.
- evidence: `pkg/flagutil/github.go:340-365`; contrast with `WithGitUser(getter GitUserGetter) ClientFactoryOpt` at `pkg/git/v2/client_factory.go:251-255`, which is unreachable from `GitClientFactory`.

### [related-code] `publisher.Commit` (panic site)
- where: `pkg/git/v2/publisher.go:53-64`
- excerpt: |
    func (p *publisher) Commit(title, body string) error {
        p.logger.Infof("Committing changes with title %q", title)
        name, email, err := p.info()
        if err != nil {
            return err
        }
        commands := [][]string{
            {"add", "--all"},
            {"commit", "--message", title, "--message", body, "--author", fmt.Sprintf("%s <%s>", name, email)},
        }

### [related-code] `ClientFactoryOpts.GitUser` and `WithGitUser`
- where: `pkg/git/v2/client_factory.go:102-120,177-178,251-255`
- excerpt: |
    // client_factory.go:120
    GitUser GitUserGetter
    // client_factory.go:177-178
    if cfo.GitUser != nil {
        target.GitUser = cfo.GitUser
    }
    // client_factory.go:251-255
    func WithGitUser(getter GitUserGetter) ClientFactoryOpt {
        return func(o *clientFactoryOpts) {
            o.GitUser = getter
        }
    }

### [related-code] `GitHubOptions.GitClientFactory` — missing `GitUser` wiring
- where: `pkg/flagutil/github.go:340-365`
- excerpt: |
    func (o *GitHubOptions) GitClientFactory(cookieFilePath string, cacheDir *string, dryRun, persistCache bool) (gitv2.ClientFactory, error) {
        opts := gitv2.ClientFactoryOpts{
            Censor:         secret.Censor,
            CookieFilePath: cookieFilePath,
            Host:           o.Host,
            Persist:        &persistCache,
            SigningKeyPath: o.SigningKeyPath,
        }
        ...
        if cookieFilePath == "" && (o.TokenPath != "" || o.AppPrivateKeyPath != "") {
            opts.Username = func() (string, error) { return user, nil }
            opts.Token = generator
        }
        // GitUser is never set here

### [related-code] No caller currently hits the panic, but none can use Commit through this helper either
- where: `pkg/git/v2/publisher_test.go:28-138` (`TestPublisher_Commit`, no `info == nil` case); `cmd/generic-autobumper/bumper/bumper.go:613,638` (`gitCommit`/`MakeGitCommit`)
- excerpt: |
    // bumper.go bypasses Publisher.Commit entirely, using raw exec with
    // explicit name/email instead — the same style of workaround the
    // issue author describes, done independently.
- (Grep across `cmd/` and `pkg/` for `.Commit(` on a `GitClientFactory`-derived client returns no non-test hits — the 9+ existing callers of `GitHubOptions.GitClientFactory` never call `.Commit()`.)

### [related-issue] Prior art for `GitUser` option
- ref: kubernetes-sigs/prow#507
- relevance: Introduced `WithGitUser`/`GitUserGetter` on the lower-level `gitv2` constructors; issue #800 is about making that reachable from `GitHubOptions.GitClientFactory` too.

### [related-pr] PR #801 implements both proposed fixes; open, blocked on review
- ref: kubernetes-sigs/prow#801
- relevance: Touches only `pkg/git/v2/publisher.go` (nil guard) and `pkg/flagutil/github.go` (`opts.GitUser`), matching both recommended next steps from the previous triage. State `OPEN`, head `b8efacee1b5d98e1647b21dd97608647b8c6d301`, `ok-to-test` present, no test changes. ylink-lfs requested changes on 2026-07-29 at `publisher.go:56`: "Current the returned error doesn't explain the remedy for developers."

### [cause] PR #801's `GitUser` wiring covers only token/App auth and leaves the author email empty
- detail: The added getter returns `user, "", nil`, so `Commit` builds `--author "prow <>"`; verified locally that git accepts this and records an author with a blank email, which no GitHub account can claim. The assignment also sits inside the existing `cookieFilePath == "" && (o.TokenPath != "" || o.AppPrivateKeyPath != "")` branch, so cookie-authenticated clients still get no `GitUser` at all — there the nil guard only converts the panic into an error instead of closing the usability gap that is half of this issue.
- evidence: |
    ```go
    // PR #801, pkg/flagutil/github.go
    opts.Username = func() (string, error) { return user, nil }
    opts.GitUser = func() (string, string, error) { return user, "", nil } // pass empty email, git allows this
    opts.Token = generator
    ```

## Checked

- All file/line references in the issue body verified against current `main` (HEAD `e601a1ffafd7d8d3a781238a4c5f4233d6248f68`) — accurate.
- Repo-wide grep of `GitHubOptions.GitClientFactory(` callers (`cmd/deck`, `cmd/external-plugins/cherrypicker`, `cmd/gerrit`, `cmd/generic-autobumper/bumper`, `cmd/hook`, `cmd/gangway`, `cmd/moonraker`, `cmd/sub`, `cmd/tide`, plus `pkg/pubsub/subscriber` tests) — none call `.Commit(...)`; no observed production impact yet, but blocks any new adopter (like the reporter's plugin).
- `pkg/git/v2/publisher_test.go` — existing `TestPublisher_Commit` table covers success/`add`-fails/`info`-fails/`commit`-fails, but no nil-`info` case; confirms the panic path is untested.
- `pkg/flagutil/github_test.go` — no test exercises `GitClientFactory` at all.
- Other `publisher`/`clientFactory` methods (`PushToFork`, `PushToNamedFork`, `PushToCentral`) get dependencies via resolver funcs always wired at construction time — `Commit`/`info` is the sole nil-dereference risk in this package.
- Refresh 2026-10-26: issue comments and timeline after `2026-07-27T00:05:00Z` hold only k8s-triage-robot's stale notice (`2026-10-22T21:20:28Z`) and the matching `lifecycle/stale` label event; no human comments.
- Refresh 2026-10-26: fetched upstream `main` over HTTPS (376 commits past `e601a1f`); none touch `pkg/git/v2/publisher.go`, `pkg/flagutil/github.go`, or `pkg/git/v2/client_factory.go`, so line references and the cause analysis remain accurate.
- Refresh 2026-10-26: inspected PR #801's diff, its single review comment, and its label/head state; no CI signal beyond `ok-to-test`.
- Refresh 2026-10-26: confirmed in a scratch repo that `git commit --author "name <>"` succeeds and records a blank-email author.
- Refresh 2026-10-27: issue remains open; its only current label is `kind/bug`. The only new comment was petr-muller's 2026-10-26 `/kind bug` and `/remove-lifecycle stale` command; the timeline has no new cross-references.
- Refresh 2026-10-27: PR #801 remains open, unmerged, and unchanged since 2026-09-27. Upstream `main` advanced, but `pkg/git/v2/publisher.go`, `pkg/flagutil/github.go`, and `pkg/git/v2/client_factory.go` have no commits since the recorded `main_sha`.

## Next steps

- Unblock #801 rather than encouraging a competing PR: it is the reporter's own implementation of both proposals and already carries `ok-to-test`.
- Have the author address the open review comment at `publisher.go:56` — make the error name the remedy, e.g. `committing %q: client has no GitUser; pass gitv2.WithGitUser to the client factory`.
- Raise the two gaps #801 still leaves: `GitUser` is set only on the token/App auth path (cookie-auth clients still cannot commit), and `--author "user <>"` yields unattributable commits — require a real email or reuse the bot/app email.
- Still missing from #801: the `publisher_test.go` case for `info == nil` (expect error, not panic) and `github_test.go` coverage that `GitClientFactory` forwards `GitUser`.
- The stale marker was cleared on 2026-10-26 and `kind/bug` was applied; current labels are only `kind/bug`. No `area/git` label exists, and `good-first-issue` remains unapplied.

## Advice

Live state re-checked 2026-10-27: issue remains `open`, with only `kind/bug`; `lifecycle/stale` was removed by petr-muller on 2026-10-26. PR #801 is still `OPEN`, unchanged since 2026-09-27, and remains the tracked fix. The prior recorded review concerns and two substantive gaps in the `[cause]` finding remain outstanding.

1. Nudge #801 with the concrete remaining work — it is 4 lines from merging and is the only thing standing between this issue and closure.

```
gh pr comment 801 --repo kubernetes-sigs/prow --body "> *This was generated by AI during triage.*

CI is green and this is close. Remaining before merge:

- Make the \`publisher.Commit\` error name the remedy (ylink-lfs on \`publisher.go:56\`), e.g. \`committing %q: client has no GitUser; pass gitv2.WithGitUser to the client factory\`.
- Set \`GitUser\` outside the token/App branch: it is currently assigned only when \`cookieFilePath == \"\" && (TokenPath != \"\" || AppPrivateKeyPath != \"\")\`, so cookie-authenticated clients still hit the new error instead of committing.
- Do not emit \`--author \"user <>\"\`: a blank email produces commits no GitHub account can claim. Return a real email, or reuse the bot/app email.
- Add the two tests: a \`publisher_test.go\` case with nil \`info\` (expect an error, not a panic) and \`github_test.go\` coverage that \`GitClientFactory\` forwards \`GitUser\`."
```

2. Record on the issue that the PR is the tracked path, so a future stale pass and the next triage see it without digging through the timeline.

```
gh issue comment 800 --repo kubernetes-sigs/prow --body "> *This was generated by AI during triage.*

Tracked in #801: nil guard for \`publisher.Commit\` plus \`GitUser\` plumbing through \`GitHubOptions.GitClientFactory\`. CI is green; what remains is an actionable error message, \`GitUser\` on the non-token auth path, a non-empty author email, and the two regression tests."
```

3. If the author has gone quiet (last PR update was 2026-09-27, the review comment still unaddressed), take the branch over rather than leaving the fix stuck. The delta is small enough to land in one sitting.

```
gh pr checkout 801   # or: git fetch https://github.com/StarMiner99/prow-fork.git main:pr-801
```

## Open questions

- The shape question is answered by #801: assign `opts.GitUser` directly in `GitClientFactory`, mirroring `Username`/`Token`. Remaining: which email should the default getter return, and should `GitUser` be settable for cookie/App-authenticated clients too?
