---
pr: kubernetes-sigs/prow#772
title: "git: add SSH commit signing support to the git client factory"
head_sha: 6ac6774906e85c753c3903ceb98e8d3a691449ce
base: main
reviewed_at: 2026-06-29T15:19:47Z
verdict: approve
gate:
  decision: hold
  gated_at: 2026-06-29T15:22:12Z
  gated_head_sha: 6ac6774906e85c753c3903ceb98e8d3a691449ce
  reviewed_head_sha: 6ac6774906e85c753c3903ceb98e8d3a691449ce
---

## Gate

**Decision: hold** — PR has `lgtm` but is missing `approved`. The required approver (`droslean` per `pkg/OWNERS`) requested architectural changes via `/hold`, those changes were made (signing moved from cherrypicker into the git client factory), and `/unhold` was applied by `jmguzik` with `/lgtm`. But `droslean` has not re-engaged to confirm the rework addresses the concern or to `/approve`.

### Prior findings disposition

All review findings from REVIEW.md are **not addressed** in the current head (no commits since review):

- **[should-fix] ClientFromDir bypasses signing** (`client_factory.go:435-438`) — still returns `c.bootstrapClients(...)` without signing config. Latent gap, no production callers. Does not gate merge on its own.
- **[should-fix] Generic error message** (`client_factory.go:504`) — still wraps all three config calls with the same message. Polish, does not gate merge.
- **[nit] No startup validation, no log line, placement debt** — unchanged. None gate merge.

droslean's hold concern (move signing from cherrypicker into git client factory) — **addressed** by the current architecture. `jmguzik` confirmed with `/lgtm` + `/unhold`.

### Merge risk (Area 2)

No notable merge risk. All changes are additive and opt-in:
- `NewLocalClientFactory` signature adds `...ClientFactoryOpt` — backwards compatible (variadic).
- New exported `SigningKeyPath` field on `GitHubOptions` and `ClientFactoryOpts` — additive, no json/yaml tags.
- New exported `WithSigningKeyPath` func — additive.
- Feature is opt-in via `--git-signing-key-path`, empty default preserves current behavior.

### Gating list

1. Missing `approved` label — `droslean` is the required approver per `pkg/OWNERS` and has not `/approve`d after the rework.

## Summary

Adds `--git-signing-key-path` flag to `GitHubOptions`. When set, every repo cloned via `ClientForWithRepoOpts` gets `gpg.format=ssh`, `user.signingkey`, and `commit.gpgsign=true`. Primary consumer is cherrypicker (`git am` path). Opt-in, no behavioral change without the flag. Three independent reviewer perspectives (code quality, maintainability, deployment risk) all converge on approve.

## Findings

### [should-fix] ClientFromDir does not apply signing configuration
- where: `pkg/git/v2/client_factory.go:435-438`
- concern: `ClientFromDir` is part of the public `ClientFactory` interface but does not apply signing config, unlike `ClientForWithRepoOpts`. No production callers today, but a future caller expecting signed commits will silently get unsigned ones.
- excerpt: |
    func (c *clientFactory) ClientFromDir(org, repo, dir string) (RepoClient, error) {
        return c.bootstrapClients(org, repo, dir)
    }

### [should-fix] Generic error message for three distinct git config calls
- where: `pkg/git/v2/client_factory.go:497-506`
- concern: All three config calls (`gpg.format`, `user.signingkey`, `commit.gpgsign`) wrap errors with the same message `"failed to configure commit signing"`. Include the config key: `fmt.Errorf("failed to configure commit signing (%s): %%w", args[0], err)`. Flagged independently by code quality and maintainability reviewers.
- excerpt: |
    if err := repoClient.Config(args...); err != nil {
        return nil, fmt.Errorf("failed to configure commit signing: %w", err)
    }

### [nit] No early validation of signing key path
- where: `pkg/flagutil/github.go:151`
- concern: `SigningKeyPath` is not checked in `Validate()`. A typo lets the component start but fail at commit time with a cryptic git error. An `os.Stat` check would surface misconfiguration at startup. Consistent with how `AppPrivateKeyPath` is handled (also no stat), so not blocking. Flagged by all three reviewers.
- excerpt: |
    fs.StringVar(&o.SigningKeyPath, "git-signing-key-path", "", "Path to an SSH private key for signing git commits.")

### [nit] No log line when signing is enabled
- where: `pkg/git/v2/client_factory.go:494`
- concern: Other factory-level configurations produce log output. A debug-level log when `signingKeyPath` is non-empty would help operators confirm signing is active without inspecting individual repo configs.

### [nit] Signing support deepens existing GitClientFactory placement debt
- where: `pkg/flagutil/github.go:51`
- concern: Existing `TODO(chaodaiG)` at line 338 acknowledges `GitClientFactory` belongs outside `github.go`. Gerrit adapter instantiates empty `GitHubOptions{}` just for git client access. Adding `SigningKeyPath` here is pragmatic but further entangles git concerns with GitHub-specific types. Not blocking; noting for the eventual refactor.

### [question] Should ClientFromDir also apply signing config?
- where: `pkg/git/v2/client_factory.go:435`
- concern: Is `ClientFromDir` intentionally a "raw" client that skips factory-level config, or should it apply signing for interface completeness?

### [question] Key format validation
- where: `pkg/git/v2/client_factory.go:499`
- concern: Passing a GPG key path while `gpg.format=ssh` is hardcoded would fail at commit time with a confusing error. Worth a doc note or comment?

## Checked
- `NewLocalClientFactory` signature change is backwards-compatible (variadic opts, existing callers unaffected)
- `ClientFor` delegates to `ClientForWithRepoOpts`, so signing config applies for both entry points
- `Apply` method handles `SigningKeyPath` consistently with other string fields (`Host`, `CookieFilePath`)
- Flag naming (`--git-signing-key-path`) is appropriate; distinguished from `--github-*` flags since this is a git concern
- Test correctly exercises `git am` path matching cherrypicker's production usage
- Signing config applied to secondary clone only (correct; cache is bare/mirror for fetching)
- No interface breakage from `NewLocalClientFactory` signature change
- Feature is entirely opt-in, zero behavioral change without the flag
- Per-clone config, no global git config side effects

## Open questions
- Should `ClientFromDir` also apply signing config for interface completeness, or is it intentionally a "raw" client?
- Would an `os.Stat` check on the key path at `Validate()` time be welcome, or is deferred validation preferred here?
- Is a GPG-vs-SSH key format mismatch worth guarding against or documenting?
