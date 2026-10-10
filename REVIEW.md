---
pr: kubernetes-sigs/prow#965
title: "config: reject partial job config directory walks"
head_sha: c5a30d39221c6adec51b9ba153425a01f78a457d
base: main
reviewed_at: 2026-09-23T09:38:18Z
verdict: request-changes
---

## Verdict

Request changes. Making every directory-walk callback error fatal means a transient checkout/config-map race can make a component fail its initial config load and exit. Prow binaries conventionally treat failure to start the config agent as fatal, so this breaks (or crash-loops) a deployment instead of retaining the existing availability behavior.

## What this PR does

- Makes `ReadJobConfig` delegate to an injectable directory walker.
- Collects errors supplied to the walk callback instead of only logging and discarding them.
- Returns the aggregated error after walking, rather than a partially populated configuration.
- Adds a regression test for a valid job file followed by a simulated disappearing checkout entry.

## Findings

### [blocking] Do not make a transient walk race fatal at initial startup
- where: `pkg/config/config.go:1731-1736`
- concern: `filepath.Walk` reports an entry disappearing during a git-sync/config-map update through this callback. The new aggregate makes `Agent.Start` return that error (`pkg/config/agent.go:346-348`); consumers such as sinker immediately call `Fatal` on it (`cmd/sinker/main.go:114-116`). A restart during the update therefore cannot start until a later restart, turning a normally transient filesystem race into deployment downtime/crash-looping. Preserve an availability-safe retry/last-known-good startup path, or restrict rejection to errors that prove the configured job config is invalid rather than temporarily inconsistent.
- excerpt: |
    if err != nil {
        errs = append(errs, fmt.Errorf("walking path %q: %w", path, err))
        return nil
    }

## Checked

- `pkg/config/config.go:1731-1784`: callback errors are now wrapped, retained, and included with the walk return value in `utilerrors.NewAggregate`.
- `pkg/config/agent.go:346-348`: an initial `Load` error is returned before the agent has a stored configuration.
- `pkg/flagutil/config/config.go:94-95` and `cmd/sinker/main.go:114-116`: callers propagate that initial load error to a fatal process exit.
- `pkg/config/config_test.go:3692-3713`: the added test models precisely the disappearing-entry race that now prevents startup.
- `git diff --check`: clean.

## Open questions

- Is the intended operational policy really to require a pod restart after a git-sync/config-map update races its first config load? If so, it needs an explicit startup retry mechanism before this behavior can be safe.
