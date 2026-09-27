---
pr: kubernetes-sigs/prow#801
title: "Add nil guard to publisher.Commit and set GitUser in GithubOptions.GitClientFactory"
head_sha: b8efacee1b5d98e1647b21dd97608647b8c6d301
base: main
reviewed_at: 2026-07-30T12:55:12Z
verdict: request-changes
refresh_log:
  - from: b8efacee1b5d98e1647b21dd97608647b8c6d301
    to: b8efacee1b5d98e1647b21dd97608647b8c6d301
    summary: "No code changes. Maintainer ylink-lfs submitted CHANGES_REQUESTED review 2026-07-29T16:03:39Z with inline comment on publisher.go:56 re: error message lacking remediation guidance."
---

## What this PR does
- Fixes #800: `publisher.Commit` dereferenced `p.info` unconditionally, panicking with nil pointer when `GitUser` was unset.
- `pkg/git/v2/publisher.go`: adds `if p.info == nil` guard in `Commit`, returns wrapped error instead of panicking.
- `pkg/flagutil/github.go`: `GitClientFactory` now sets `opts.GitUser` (using the same `user` login as `opts.Username`, empty email) in the token/app-auth branch, root-causing the panic for the real-world construction path.
- No test changes included.
- Since previous review: no code changes (head SHA unchanged). A maintainer (`ylink-lfs`, MEMBER) submitted a CHANGES_REQUESTED review with one inline comment, escalating the tone on the error-message finding below from a style nit to a blocking ask.

## Findings

### [blocking] error message doesn't explain remedy
- where: `pkg/git/v2/publisher.go:55-57`
- concern: maintainer `ylink-lfs` (MEMBER) requested changes on 2026-07-29, commenting inline that "the returned error doesn't explain the remedy for developers." The message `"error committing %q: GitUser is not set"` tells the caller what's wrong but not how to fix it (e.g. set `GitUser` via `WithGitUser`/`ClientFactoryOpts.GitUser`, or which flag/option is responsible when reached via `GitClientFactory`). Needs a revision that names the fix, not just the symptom, before this can merge.
- excerpt: |
    if p.info == nil {
        return fmt.Errorf("error committing %q: GitUser is not set", title)
    }

### [should-fix] missing test for nil `info` guard
- where: `pkg/git/v2/publisher_test.go:28-133`
- concern: `TestPublisher_Commit` covers "info fails", "add fails", "commit fails", but has no case for `info == nil`, which is exactly the new branch being added. Should add a case with `info: nil`, `expectedErr: true`, `expectedCalls: [][]string{}`.
- excerpt: |
    p := publisher{
        executor: &e,
        info:     testCase.info,
        logger:   logrus.WithField("test", testCase.name),
    }
    actualErr := p.Commit(testCase.title, testCase.body)

### [should-fix] no test asserting GitClientFactory sets GitUser
- where: `pkg/flagutil/github.go:340-366`
- concern: the actual root-cause fix (populating `opts.GitUser` from the token/app-auth branch) has no accompanying test. A unit test asserting `opts.GitUser` is non-nil/returns expected user after this branch runs would cover the real bug, not just the defensive guard in publisher.go.
- excerpt: |
    opts.Username = func() (string, error) { return user, nil }
    opts.GitUser = func() (string, string, error) { return user, "", nil } // pass empty email, git allows this
    opts.Token = generator

### [question] empty email acceptable for all downstream consumers?
- where: `pkg/flagutil/github.go:359`
- concern: using an empty email is valid git (`--author "name <>"`), but worth confirming GitHub/downstream systems handle commits with empty-email authors as expected (e.g. commit attribution, signature/verification tooling) for every caller of `GitClientFactory` (tide, hook, moonraker, deck, cherrypicker, gerrit, generic-autobumper, sub, gangway).

### [nit] consider noreply-style email instead of empty string
- where: `pkg/flagutil/github.go:359`
- concern: a synthesized email like `<login>@users.noreply.github.com` (GitHub's own convention for bot commit authorship) would maximize compatibility with any external tooling that assumes a non-empty author email, versus the current empty string. Non-blocking follow-up.

## Checked
- Nil guard error format matches existing wrapping style in the same function (`fmt.Errorf("error committing %q: ...", title)`).
- `GitUser` is only set inside the `cookieFilePath == "" && (TokenPath != "" || AppPrivateKeyPath != "")` branch, consistent with where `Username`/`Token` are set — Gerrit path unaffected, correctly out of scope. The Gerrit path still leaves `GitUser` nil after this PR and relies solely on the new guard in `publisher.go` to fail cleanly instead of panicking — confirmed intentional defense-in-depth, not an oversight.
- All current callers of `GitClientFactory` (cmd/moonraker, cmd/tide, cmd/deck, cmd/hook, cmd/gangway, cmd/sub, cmd/external-plugins/cherrypicker, cmd/generic-autobumper/bumper, cmd/gerrit) go through this fixed path and benefit.
- Change is minimal and additive; low blast radius, no behavior change for already-correctly-configured callers. No config/CLI/API surface changes — safe, low-risk upgrade with clean rollback.
- Multi-perspective maintainer review (code quality, maintainability, deployment risk) independently converged on the same two findings below; deployment risk assessed LOW, maintenance burden assessed LOW.

## Open questions
- Can you revise the nil-guard error message in `publisher.go:56` to point the caller at the fix (e.g. mention `WithGitUser`/`ClientFactoryOpts.GitUser`), per ylink-lfs's review comment?
- Can you add a `TestPublisher_Commit` case covering `info == nil` to lock in the new guard?
- Can you add/extend a test in `pkg/flagutil/github_test.go` verifying `GitClientFactory` populates `opts.GitUser` when token/app auth is configured?
