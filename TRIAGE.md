---
issue: kubernetes-sigs/prow#800
title: "publisher.Commit panics (nil pointer) when GitUser is unset, and GitHubOptions.GitClientFactory can't set it"
state: open
labels: 
main_sha: e601a1ffafd7d8d3a781238a4c5f4233d6248f68
triaged_at: 2026-07-27T00:05:00Z
verdict: accepted
---

## Verdict

Accepted: legitimate, precisely-diagnosed bug (nil-pointer panic) plus a small, well-scoped usability gap, both confirmed against current `main`. Effort level 1 (good-first-issue); author has volunteered a PR.

## What the issue reports

- `RepoClient.Commit` panics with a nil pointer dereference when the underlying `GitUser` getter was never configured.
- The panic happens specifically for clients built via `flagutil.GitHubOptions.GitClientFactory(...)`, the common in-repo helper, which wires `Username`/`Token` for clone/push but never `GitUser` for commit.
- `WithGitUser` (added in #507) exists and works, but is only reachable through the lower-level `gitv2.NewClientFactory`/`NewLocalClientFactory` constructors, not through `GitHubOptions.GitClientFactory`.
- Author's workaround: build two factories on the same working tree — the GitHub-authenticated one for clone/push, and a second manual `gitv2.NewClientFactory(WithGitUser(...))` used only for `.Commit()`.
- Author proposes: (1) nil-guard `publisher.Commit` to return an error instead of panicking, (2) let `GitHubOptions.GitClientFactory` accept/forward a `GitUser` getter. Offered to send a PR for both.

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

## Checked

- All file/line references in the issue body verified against current `main` (HEAD `e601a1ffafd7d8d3a781238a4c5f4233d6248f68`) — accurate.
- Repo-wide grep of `GitHubOptions.GitClientFactory(` callers (`cmd/deck`, `cmd/external-plugins/cherrypicker`, `cmd/gerrit`, `cmd/generic-autobumper/bumper`, `cmd/hook`, `cmd/gangway`, `cmd/moonraker`, `cmd/sub`, `cmd/tide`, plus `pkg/pubsub/subscriber` tests) — none call `.Commit(...)`; no observed production impact yet, but blocks any new adopter (like the reporter's plugin).
- `pkg/git/v2/publisher_test.go` — existing `TestPublisher_Commit` table covers success/`add`-fails/`info`-fails/`commit`-fails, but no nil-`info` case; confirms the panic path is untested.
- `pkg/flagutil/github_test.go` — no test exercises `GitClientFactory` at all.
- Other `publisher`/`clientFactory` methods (`PushToFork`, `PushToNamedFork`, `PushToCentral`) get dependencies via resolver funcs always wired at construction time — `Commit`/`info` is the sole nil-dereference risk in this package.

## Next steps

- Comment on the issue inviting the author to submit the proposed PR (nil-guard on `Commit` + optional `GitUser` plumbing through `GitHubOptions`).
- Request the PR include: a `publisher_test.go` regression case for `info == nil` (expect error, not panic), and new `github_test.go` coverage asserting `GitClientFactory` forwards `GitUser` when configured.
- Apply labels manually: `kind/bug`, `area/git`, `good-first-issue`.

## Open questions

- What's the preferred shape for exposing the `GitUser` getter on `GitHubOptions` — a plain field, a setter method, or a functional-option-style addition? (Not blocking — can be settled in PR review.)
