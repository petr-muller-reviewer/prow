---
pr: kubernetes-sigs/prow#758
title: "Fix expiring token handling gaps in secret agent"
head_sha: 188a2e4d130ef7f172e2e170602865e3a6f231d7
base: main
reviewed_at: 2026-07-27T00:26:40Z
verdict: approve
---

## What this PR does
- `Start()` now resets `expiringTokens` alongside `secretsMap`/`ReloadingCensorer`, so a restart no longer leaves stale expiring tokens live (`pkg/config/secret/agent.go:49`).
- `getSecrets()` now inserts `expiringTokens` keys, so the legacy one-shot `CensoringFormatter` path (built via `logrusutil.NewCensoringFormatter`, used in `pkg/pod-utils/clone/clone.go`) censors expiring tokens too — previously excluded entirely (`pkg/config/secret/agent.go:198-205`).
- Adds `TestStartResetsExpiringTokens`, `TestExpiringTokenCensoring` (active + expired sub-tests), `TestGetSecretsIncludesExpiringTokens`.
- Followup to #696 (`73b371825`), which introduced expiring token support.
- Net security effect: closes a redaction gap — active expiring tokens (e.g. GitHub App installation tokens) were previously left unredacted in logs written through the legacy `CensoringFormatter` path; no case regresses from redacted to unredacted.

## Findings

### [nit] getSecrets() does not filter already-expired tokens
- where: `pkg/config/secret/agent.go:198-205`
- concern: `getSecrets()` inserts every key in `a.expiringTokens` unconditionally, without the `exp.After(now)` check that `refreshCensorer()` already applies (agent.go:160). Expiry pruning only happens lazily inside `refreshCensorer()` (triggered by `Add`/`setSecret`/`addExpiringToken`), so an expired token can still appear in `getSecrets()`'s result until the next write prunes it. Harmless today since the only consumer, `logrusutil.NewCensoringFormatter`, snapshots the set once at construction (worst case: over-censoring, never under-censoring) — would matter if `getSecrets` is ever wired into a live/reloading path.
- excerpt: |
    for token := range a.expiringTokens {
        secrets.Insert(token)
    }

### [nit] Start does not explicitly re-trigger refreshCensorer after resetting expiringTokens
- where: `pkg/config/secret/agent.go:47-53`
- concern: `Start` resets `expiringTokens` to a fresh map but does not call `refreshCensorer`/`RefreshBytes` directly — it relies on the `Add` loop triggering `setSecret` -> `refreshCensorer` for each path. If `paths` is empty, this is still fine because `ReloadingCensorer` was just re-constructed via `secretutil.NewCensorer()` two lines above, but the implicit dependency is non-obvious to a future reader who might expect `Start` to explicitly refresh state itself.
- excerpt: |
    a.secretsMap = make(map[string]secretReloader, len(paths))
    a.expiringTokens = make(map[string]time.Time)
    a.ReloadingCensorer = secretutil.NewCensorer()

### [question] Worth a comment explaining the restart-leak rationale?
- where: `pkg/config/secret/agent.go:49`
- concern: The reason `expiringTokens` must be reset alongside `secretsMap` (preventing stale tokens from a previous `Start` from surviving a restart) is only documented in the PR description, not in code. A one-line comment would make the intent self-evident two years from now.

### [question] Release note for the redaction-gap fix?
- where: n/a (operational)
- concern: This fix closes a secret-leakage gap in the legacy `CensoringFormatter` log path — active expiring tokens (e.g. GitHub App installation tokens) were previously unredacted there. Worth a release-note mention so security-conscious operators know logs predating this fix may contain unredacted expiring tokens. Not a merge blocker.

## Checked
- `Start()` reset ordering and zero-value safety of `agent{}` literals used directly in tests (`sync.RWMutex` zero value is valid).
- `NewCensoringFormatter` call sites (`clone.go`, `agent_test.go`, `logrusutil_test.go`) confirm `getSecrets()` is a one-shot snapshot, not a live/reloading path — makes the expiry-pruning nit above low-risk.
- New tests actually exercise the fixed gaps (reset, censoring of active vs. expired token, `getSecrets` inclusion) rather than just re-testing existing behavior; they assert on `Censor()` output content, not just absence of error.
- No lock-ordering or deadlock changes; `Start()`/`addExpiringToken` locking pattern unchanged.
- No exported API, config schema (`json`/`yaml` tags), CLI flags, or RBAC/permission changes — purely internal unexported-state fix, no deployment/upgrade risk.
- Cross-checked via a separate 3-perspective maintainer review (code quality / maintainability / deployment risk) — all three independently reached APPROVE with no critical or blocking issues; findings above consolidate their nits.

## Open questions
- Should `getSecrets()` (or `refreshCensorer`) prune expired entries eagerly on every read, in case a future caller wires it into a live-refresh path instead of a one-shot snapshot?
- Would a short comment on the `expiringTokens` reset line in `Start` be worth adding for future readers?
- Should a release note flag that logs predating this fix may contain unredacted expiring tokens?
