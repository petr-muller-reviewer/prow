---
pr:
  - 696
  - 697
title:
  - "register expiring tokens values in the secret agent"
  - "fix(clonerefs): remove cloneDepth=1 from sparseCheckout"
head_sha:
  - 73b371825cd2841963021bb141a173c549657dd4
  - 7b19dc1d09f9e29213e1f14e2e224594451dc175
base:
  - main
  - main
reviewed_at: "2026-04-30"
verdict:
  - approve-with-suggestions
  - approve
---

# PR #696 — register expiring tokens values in the secret agent

**Author:** droslean (Nikolaos Moraitis)
**Base:** main **Branch:** gh-app-token
**Diff:** +102 / -3

Files changed:
- `pkg/config/secret/agent.go` +31 / -3
- `pkg/config/secret/agent_test.go` +68
- `pkg/github/app_auth_roundtripper.go` +3

## Merge Gate: MERGE

All three reviewers approve. No show-stoppers found.

## Verdict: APPROVE WITH SUGGESTIONS

## Overview

The secret agent was built to censor file-based tokens. GitHub App tokens (JWTs, installation access tokens) are short-lived and minted at runtime, so they were never registered with the censorer and could leak in logs — for example in git fetch URLs.

This PR adds an `expiringTokens` map to the secret agent that stores token values keyed by expiry time. `refreshCensorer` prunes expired entries and includes live entries in the censor pattern list. Two call sites in `app_auth_roundtripper.go` wire JWT and installation token registration.

## Converging Concerns (2 items)

Issues flagged independently by multiple reviewers carry the highest confidence.

### `refreshCensorer` lock upgrade needs a comment

**Flagged by:** Code Quality, Maintainability

The method changed from `RLock` to `Lock` because it now mutates `expiringTokens` via `delete` during pruning. Correct, but non-obvious — a future maintainer could regress it back to `RLock`. A one-line comment explaining why would prevent that.

### `expiringTokens` not reinitialized in `Start`

**Flagged by:** Code Quality, Maintainability

`Start` resets `secretsMap` and `ReloadingCensorer` but leaves `expiringTokens` intact. If `Start` were ever called after tokens were registered, stale entries would survive the reset. Adding `expiringTokens = make(map[string]time.Time)` in `Start` would make it consistent.

## 1/3 Code Quality — Approve

### Concurrency model is sound (+)

The two-phase lock pattern in `addExpiringToken` (lock → add → unlock → `refreshCensorer`) mirrors the existing `setSecret` method. The lock upgrade from `RLock` to `Lock` in `refreshCensorer` is correct.

### Narrow censoring gap

Between adding a token to the map and `refreshCensorer` rebuilding the censorer, a log line could theoretically leak the token. Practically negligible and consistent with how `setSecret` already works.

### Missing end-to-end censoring test

Tests assert map state but don't verify the censorer was actually updated with the token value. A test calling `Censor` on a string containing the token would provide stronger coverage of the security-critical behavior.

### Good test coverage (+)

Table-driven tests cover: empty value, zero time, past expiry, and pruning of stale entries on add.

### Clean integration points (+)

Two one-liner additions in `app_auth_roundtripper.go` at exactly the right places — right after token creation, before use.

## 2/3 Maintainability — Approve (Low burden)

### Follows established patterns (+)

The `addExpiringToken` → `refreshCensorer` flow mirrors the existing `setSecret` → `refreshCensorer` pattern exactly. A future maintainer will immediately recognize it.

### Minimal public API surface (+)

One new exported function (`AddExpiringToken`), consistent with the package's style of thin wrappers around the singleton.

### Stale tokens only pruned on refresh

Expired tokens linger until `refreshCensorer` runs again. Given GitHub token lifetimes (~10 min JWT, ~1 hr installation) and typical API call frequency, this is unlikely to matter in practice.

### Global singleton coupling

The call to `secret.AddExpiringToken` uses the package-level singleton, making `appsRoundTripper` harder to test in isolation. This is a pre-existing pattern, not new debt from this PR.

## 3/3 Deployment Risk — Low risk

### No breaking changes (+)

No configuration schema changes, no CLI flags, no API surface changes. Purely internal implementation.

### Zero operator action required (+)

Rolling upgrades fully supported. Old and new instances coexist safely. Rollback is trivial.

### Only observable effect is improved security (+)

Tokens that previously leaked in plain text in logs will now be censored.

### Lock contention negligible

The `RLock` → `Lock` promotion adds minor contention, but the critical section is short (map iteration + deletion).

### Memory bounded naturally

Map size bounded by number of active (non-expired) tokens, proportional to distinct installations. Pruned on every `refreshCensorer` call.

## Gate: Non-Blocking Observations

### Double-lock pattern in `addExpiringToken`

Theoretically allows a pruning goroutine to remove a just-inserted expired token between the two lock acquisitions. Benign — expired tokens should not be censored anyway.

### Lock upgrade from `RLock` to `Lock` in `refreshCensorer`

Changes contention profile for all callers. Practical impact negligible given frequency and duration of these operations.

## Suggestions (non-blocking)

1. **Comment on `refreshCensorer` lock upgrade** — Add a brief comment explaining that the full write lock is required because the method now prunes expired entries from `expiringTokens`. Protects against a future "optimization" that downgrades it back to `RLock`.

2. **Initialize `expiringTokens` in `Start`** — Add `s.expiringTokens = make(map[string]time.Time)` alongside the existing field resets for consistency and correctness.

3. **End-to-end censoring test** — Add a test that calls `Censor` on a string containing the registered token and asserts it gets redacted. Current tests verify map state but not the actual censoring behavior.

## PR Comment

This looks good to me. All aspects — code quality, maintainability, and deployment risk — check out. The concurrency model is correct, the tests cover the important edge cases, and the change closes a real security gap with minimal complexity. Two minor suggestions for future maintainability: (1) add a comment explaining why `refreshCensorer` needs a full write lock now that it prunes `expiringTokens`, and (2) initialize `expiringTokens` in `Start` for consistency with the other field resets. Neither is blocking.

---

# PR #697 — fix(clonerefs): remove cloneDepth=1 from sparseCheckout

**Author:** Prucek (Peter Rúček)
**Base:** main **Branch:** fix-sparse-checkout-depth
**Diff:** +3 / -6 (Bug Fix)

Files changed:
- `pkg/pod-utils/clone/clone.go` -3
- `pkg/pod-utils/clone/clone_test.go` +3 / -3

## Verdict: APPROVE

## Overview

Sparse checkout with `--depth 1 --filter=blob:none` creates a shallow clone that lacks enough history to find a common ancestor when merging the PR head commit, causing "unrelated histories" failures.

This PR removes the automatic `cloneDepth = 1` default that was forced when sparse checkout was enabled and `CloneDepth` was not explicitly set. The `--filter=blob:none` flag is preserved, so blob fetching remains optimized.

## Findings

### Root cause is clear and fix is correct (+)

`--depth 1` + `--filter=blob:none` + sparse checkout = no common ancestor for merge. Removing the forced shallow depth lets git fetch enough history.

### Minimal, safe change (+)

3 lines removed from production code, 3 test lines updated to match. Users who explicitly set `CloneDepth > 0` still get `--depth N` as before.

### Tests updated correctly (+)

All three sparse checkout test cases now expect `fetch` without `--depth 1`, consistent with the code change.

### Performance tradeoff

Without `--depth 1`, sparse checkout fetches will download more commit history. For repos with very long histories this could be slower, but correctness trumps speed — a clone that can't merge is useless. `--filter=blob:none` still avoids blob downloads.

## PR Comment

Clean bug fix. The root cause is well-understood — `--depth 1` with sparse checkout prevents git from finding a common ancestor for the merge. Removing the forced shallow depth is the right fix, and `--filter=blob:none` is preserved to keep blob fetching optimized. Tests updated correctly. LGTM.

---

## Followups

### 1. Initialize `expiringTokens` in `Start` (deferred-review, should)

**Where:** `pkg/config/secret/agent.go:48`

`Start` resets `secretsMap` and `ReloadingCensorer` but leaves `expiringTokens` intact. If `Start` is called after tokens have been registered, stale entries survive the reset. Converging concern from two review perspectives.

```
In `kubernetes-sigs/prow`, following the merge of PR #696 ("register expiring tokens values in the secret agent", merge commit 7dfdc18c180f0f6ba26d905b10b110ca3c8ae343):

The `Start` method in `pkg/config/secret/agent.go` resets `secretsMap` and `ReloadingCensorer` but does not reinitialize `expiringTokens`. Add `a.expiringTokens = make(map[string]time.Time)` in `Start` alongside the existing field resets (currently around line 48).

Also add a test in `pkg/config/secret/agent_test.go` that:
1. Creates an agent with a pre-populated `expiringTokens` map.
2. Calls `Start` on it.
3. Asserts `expiringTokens` is empty after the reset.

Acceptance criteria: the new line is present in `Start`, the test passes, and `go test ./pkg/config/secret/...` passes with the race detector (`-race`).

Out of scope: any other changes to the secret agent, refactoring the `Start` method, or changing how `expiringTokens` works.
```

### 2. End-to-end censoring test for expiring tokens (tests, should)

**Where:** `pkg/config/secret/agent_test.go`

The existing `TestAddExpiringToken` tests verify map state but never call `Censor` to confirm the token is actually redacted. The security-critical behavior is untested at the integration level.

```
In `kubernetes-sigs/prow`, following the merge of PR #696 ("register expiring tokens values in the secret agent", merge commit 7dfdc18c180f0f6ba26d905b10b110ca3c8ae343):

Add an integration-level test in `pkg/config/secret/agent_test.go` that verifies the censorer actually redacts expiring tokens. The test should:

1. Create an `agent` with an initialized `expiringTokens` map and `ReloadingCensorer` (same setup as the existing `TestAddExpiringToken` tests).
2. Call `addExpiringToken` with a known token value and a future expiry.
3. Call `Censor` on a byte slice containing the token value (e.g., `[]byte("https://x-access-token:ghs_secret@github.com/org/repo")`).
4. Assert the token value is replaced with the censoring placeholder (check `secretutil` for the placeholder pattern — it uses `XXXXXX` repeated to match the original length).
5. Also test that after simulating expiry (register a token with a past expiry, call `refreshCensorer`), `Censor` no longer redacts the expired token value.

Follow the existing table-driven test style in the file.

Acceptance criteria: the new test passes, and `go test ./pkg/config/secret/... -race` passes.

Out of scope: changes to production code, changes to the censoring mechanism itself, or tests for the `app_auth_roundtripper.go` call sites.
```

### 3. `getSecrets()` doesn't include expiring tokens (consistency, could)

**Where:** `pkg/config/secret/agent.go:193`

`getSecrets()` iterates only `secretsMap` and returns file-based secrets. Expiring tokens from `expiringTokens` are excluded. This method feeds `NewCensoringFormatter` (a legacy censoring path used in `clone.go` and tests). No current caller is affected, but the inconsistency between the two censoring paths is a trap for future callers.

```
In `kubernetes-sigs/prow`, following the merge of PR #696 ("register expiring tokens values in the secret agent", merge commit 7dfdc18c180f0f6ba26d905b10b110ca3c8ae343):

The `getSecrets()` method in `pkg/config/secret/agent.go` (around line 193) iterates only `secretsMap` and returns file-based secrets as a `sets.Set[string]`. It does not include values from the `expiringTokens` map. This creates an inconsistency: `refreshCensorer` includes both sources when rebuilding the censor pattern list, but `getSecrets()` only returns one.

Fix `getSecrets()` to also iterate `expiringTokens` and insert those values into the returned set. The method already holds `RLock`, which is sufficient since it only reads the map.

Add a test case in `pkg/config/secret/agent_test.go` that:
1. Creates an agent with both a file-based secret in `secretsMap` and a token in `expiringTokens`.
2. Calls `getSecrets()`.
3. Asserts the returned set contains both the file-based secret value and the expiring token value.

Acceptance criteria: `getSecrets()` returns both sources, the test passes, and `go test ./pkg/config/secret/... -race` passes.

Out of scope: changing `NewCensoringFormatter`, modifying `clone.go`, or any refactoring of the censoring architecture.
```
