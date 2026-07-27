---
pr: kubernetes-sigs/prow#675
title: "trigger: add hierarchical `trusted_apps` config with global/org/repo levels"
head_sha: e3758df16ccef2a553ff8dfab99c6ceb617b0825
base: main
reviewed_at: 2026-07-27T00:19:13Z
verdict: request-changes
refresh_log:
  - old_sha: 18779e865c90b365b434733217b3fd008de07bdf
    new_sha: 55e4ec4e25ae945a4ebf5143cb93b6549c1210e0
    summary: "1-line test fix (cmpopts.IgnoreUnexported in TestMergeFrom). Hold lifted — do-not-merge/hold label removed by author on 2026-06-15."
  - old_sha: 55e4ec4e25ae945a4ebf5143cb93b6549c1210e0
    new_sha: e3758df16ccef2a553ff8dfab99c6ceb617b0825
    summary: "Full re-review (branch was rebased/recreated, old_sha not an ancestor). Global level redesigned to require explicit Repos: [\"*\"] opt-in instead of inferring from empty Repos — resolves prior blocking finding on silent trust expansion (part a). Same-level lock protection added at all three levels (global/org/repo) in resolveTrustedApps — resolves prior should-fix finding on same-level duplicates dropping locks. New finding: plugin-config-documented.yaml not regenerated, now describes obsolete empty-Repos-is-global semantics."
  - old_sha: e3758df16ccef2a553ff8dfab99c6ceb617b0825
    new_sha: e3758df16ccef2a553ff8dfab99c6ceb617b0825
    summary: "No code change. Ran a 3-perspective maintainer review (code quality, maintainability, deployment risk) plus advisor synthesis at the same head, in addition to the single-pass technical review. Surfaced new findings: mergeFrom does not scope-validate supplemental Triggers.Repos (a decentralized/supplemental config can now declare global Repos: [\"*\"] and escalate to instance-wide trust); same-level unlocked conflicts resolve by map/list iteration order rather than content; global/org/repo classification logic is reimplemented independently in three places (TriggerFor, validateTrigger, ValidateLockedTrustedApps); resolvedTrustedApps is hidden derived state on Trigger requiring cmpopts.IgnoreUnexported workarounds. All three reviewers plus the advisor independently landed on request-changes; deployment risk rated HIGH."
---

## Summary

Replaces flat `[]string` TrustedApps on `Trigger` with structured `TrustedApp` (`name`, `untrusted`, `locked`). Hierarchical resolution global -> org -> repo in `TriggerFor` via `resolveTrustedApps`; result exposed through `GetTrustedApps()`. Global level requires explicit `Repos: ["*"]` opt-in. Backward-compatible `UnmarshalJSON` (legacy string list still parses). Optional `checkconfig` warning `validate-locked-trusted-apps` via `ValidateLockedTrustedApps()`. All five call sites migrated to the accessor. A 3-perspective maintainer review (code quality / maintainability / deployment risk) plus advisor synthesis was run in addition to the single-pass technical review; all four independently concluded request-changes.

## Findings

### [blocking] plugin-config-documented.yaml not regenerated, describes obsolete semantics
- where: `pkg/plugins/plugin-config-documented.yaml:660-1078` (Repos/TrustedApps doc block)
- concern: This file is generated purely from Go doc comments via `hack/gen-prow-documented` (globs `pkg/plugins/*.go` through `genyaml.NewCommentMap`, no other input). It currently states "A trigger entry with an empty repos field is treated as the global level for trusted_apps resolution" and shows example `repos: - ""` — this text does not exist anywhere in the current Go source (verified via grep across `pkg/plugins`) and describes the pre-redesign behavior this PR replaced with the `Repos: ["*"]` opt-in. The current `Trigger.Repos` comment only says "Use `*` to define a global trigger...". Anyone configuring `trusted_apps` from this generated reference will write an inert empty-Repos entry expecting global scope. Independently flagged by all three maintainer-review perspectives (code quality, maintainability, deployment risk). Needs `hack/gen-prow-documented` rerun before merge.
- excerpt: |
    # Repos is either of the form org/repos or just org.
    # A trigger entry with an empty repos field is treated as the global level
    # for trusted_apps resolution. Trust is resolved hierarchically:
    # global (empty repos) -> org-level -> repo-level.
    repos:
      - ""

### [blocking] Repo-level trigger still silently merges with org/global trust instead of shadowing
- where: `pkg/plugins/config.go:1134-1174` (TriggerFor)
- concern: Unaddressed from previous review. A repo-level trigger that previously fully shadowed its org's `trusted_apps` (old code returned the first matching trigger outright) now has its trust list merged with org/global apps instead. An existing repo config that deliberately scoped a narrow trusted-apps list now silently inherits all org/global apps too, with no operator-facing signal. Deployment-risk perspective rates this HIGH severity: it silently widens automatic-CI-trust on upgrade. Needs a migration callout, a checkconfig signal, or an explicit way to opt out of inheritance.

### [blocking] `Repos: ["*"]` flips previously-inert entries into live global grants
- where: `pkg/plugins/config.go:1140-1163` (TriggerFor switch)
- concern: Pre-PR, no `Repos` value — including a literal `"*"` string — ever matched the old lookup (`sets.NewString(trigger.Repos...).Has(fullName)`/`.Has(org)`), so any existing config with `"*"` in a `Repos` list (copy-paste, template placeholder, accidental leftover) was a harmless no-op. Post-PR, `repos.Has("*")` makes that same entry an active global rule merged into every org and repo instance-wide. This activates purely as a side effect of upgrading the binary — no config change, no warning, no error. Flagged as the deployment risk review's primary HIGH-severity concern.

### [blocking] Supplemental/per-repo config aggregation has no scope validation, widening escalation surface
- where: `pkg/plugins/config.go:2459` (mergeFrom appends `other.Triggers` unchecked)
- concern: `mergeFrom` appends a supplemental config's `Triggers` directly onto `c.Triggers` with no validation that `Repos` values stay scoped to the org/repo that owns that supplemental file. This aggregation mechanism predates this PR, but pre-PR the worst a careless supplemental entry could do was claim trust for the specific org/repo string it listed. Post-PR, the same mechanism lets any supplemental config (e.g. a per-org self-service plugin-config file, a common decentralized-config pattern) declare `Repos: ["*"]` and grant its chosen app global, instance-wide trust. Flagged by the deployment-risk perspective as a materially larger escalation surface introduced by this PR.

### [should-fix] ValidateLockedTrustedApps still flags agreement as violation
- where: `pkg/plugins/config.go:1692-1758` (ValidateLockedTrustedApps)
- concern: Unaddressed from previous review. The check only tests name membership (`if globalLocked[app.Name] { errs = append(...) }`), not whether the lower level's value actually conflicts with the lock. A lower-level entry repeating the *same* trust value as the lock is still flagged as a violation, and no new test exercises the agreement case. Noisy for an opt-in warning. Independently flagged by the code-quality perspective as well.
- excerpt: |
    for _, app := range trigger.TrustedApps {
        if globalLocked[app.Name] {
            errs = append(errs, fmt.Errorf("trigger for org %q configures trusted app %q which is locked at global level", r, app.Name))
        }
    }

### [should-fix] `locked: true` at repo level still silently ignored without validation
- where: `pkg/plugins/config.go:1210-1218` (resolveTrustedApps repo loop), `pkg/plugins/config.go:1651-1660` (validateTrustedApps)
- concern: Unaddressed from previous review. Repo level still drops `Locked` by design, and `validateTrustedApps` still only checks for empty/duplicate names — no warning when a repo-level entry sets a meaningless `locked: true`.

### [should-fix] Global/org/repo classification logic reimplemented independently in three places
- where: `pkg/plugins/config.go` — `TriggerFor` (~1143-1160), `validateTrigger` (~1646), `ValidateLockedTrustedApps` (~1697-1758)
- concern: The rule for what counts as global (`Repos` contains `"*"`) vs. org (no slash) vs. repo (has slash) is implemented independently three times, with slightly different mechanics each time (`repos.Has("*")` via a `sets.Set`, a raw `sets.New[string](trigger.Repos...).Has("*")` call, and manual `r == "*"`/`strings.Contains(r, "/")`/`strings.SplitN` string logic). If the classification rule ever changes, all three call sites need to be kept in sync by hand with no shared helper or type enforcing it. Flagged independently by both the code-quality and maintainability perspectives.

### [should-fix] Same-level conflicting unlocked entries resolve by iteration order, not content
- where: `pkg/plugins/config.go:1185-1226` (resolveTrustedApps)
- concern: When two triggers match the same level (e.g. two global `Repos: ["*"]` blocks, or two org-level blocks for the same org) with conflicting `untrusted` values for the same app and neither is `locked`, the last one encountered in `c.Triggers` iteration order wins via plain map overwrite. Reordering the `triggers:` list (config generator, alphabetical re-sort, file-concatenation order from supplemental config dirs) can silently flip which value takes effect with zero change to the actual trust values. Flagged by the deployment-risk perspective as a fragile, hard-to-audit property for large multi-team configs.

### [should-fix] `resolvedTrustedApps` is hidden derived state requiring a cmpopts workaround
- where: `pkg/plugins/config.go:599-603` (`GetTrustedApps`), `pkg/plugins/config.go:2464` (`mergeFrom`)
- concern: `Trigger` gained an unexported field populated only as a side effect of `TriggerFor`; any `Trigger` value constructed another way silently returns "no apps trusted" via `GetTrustedApps()`, indistinguishable from a real empty-trust config. This forced `cmpopts.IgnoreUnexported(Trigger{})` into both production `mergeFrom` code and `TestMergeFrom`, i.e. the hidden state leaked into diffing/merging logic project-wide. Flagged by both the code-quality and maintainability perspectives as fragile; a plain value-returning resolution function (or a distinct `ResolvedTrigger` type) would avoid derived state living inside the config struct.

### [nit] Indentation error in test loop body
- where: `pkg/plugins/config_test.go:781-782` (TestResolveTrustedApps)
- concern: Unaddressed. `got := trigger.GetTrustedApps()` has an extra leading tab relative to the preceding line. Not gofmt-clean; would fail CI formatting checks.
- excerpt: |
    trigger := tc.config.TriggerFor(tc.org, tc.repo)
        got := trigger.GetTrustedApps()

### [nit] Constant column alignment broken
- where: `cmd/checkconfig/main.go:128`
- concern: Unaddressed. `validateLockedTrustedAppsWarning` has one extra space of padding vs. its same-length neighbor, breaking the const block's gofmt alignment. Not gofmt-clean; would fail CI formatting checks.
- excerpt: |
    validateConfigUpdaterACLsWarning               = "validate-config-updater-acls"
    validateLockedTrustedAppsWarning                = "validate-locked-trusted-apps"

### [question] `Repos: ["*", "someorg"]` collapses entirely to global
- where: `pkg/plugins/config.go:1143-1160` (TriggerFor switch)
- concern: The switch checks `repos.Has("*")` first, so a trigger mixing `"*"` with a specific org/repo name is classified purely as global for bucket selection; the org/repo-specific name is never used to pick the org/repo match. Compounds the pre-existing mixed-Repos-list classification question. Likely a non-issue in practice, but worth a doc note or a `validateTrigger` warning when `Repos` contains `"*"` alongside other entries.

### [question] Mixed Repos lists get level-classified per lookup
- where: `pkg/plugins/config.go:1145-1163` (TriggerFor switch)
- concern: A trigger with `Repos: ["myorg", "myorg/myrepo"]` counts as repo-level for myorg/myrepo (repo case wins, dropping `locked` there) but org-level for sibling repos. Surprising classification; consider documenting or warning on mixed org+repo lists.

### [question] GetTrustedApps returns nil without TriggerFor
- where: `pkg/plugins/config.go:599-603`
- concern: `resolvedTrustedApps` is only populated by `TriggerFor`; a directly constructed `Trigger` returns nil even with `TrustedApps` set. Fails closed (correct security default, low risk), but a doc comment on the `TrustedApps` field warning against reading it directly would help future callers. (See also the should-fix finding above on the hidden-state design itself.)

### [question] DCO plugin's TrustedApps []string may confuse readers
- where: `pkg/plugins/config.go:897`, `pkg/plugins/dco/dco.go:301`
- concern: DCO has its own flat `TrustedApps []string`, unrelated to the trigger type; `config.TrustedApps` in dco.go looks like a missed migration at a glance. A clarifying comment on the DCO field would help. Whether DCO should adopt the hierarchical type is separate-PR scope.

### [question] Adopting object-form trusted_apps is a one-way street for binary rollback
- where: `pkg/plugins/config.go` (`TrustedApps.UnmarshalJSON`)
- concern: Once a config is edited to use the new `{name, untrusted, locked}` object form (even for a single entry), it can no longer be loaded by a pre-PR Prow binary — only the reverse (legacy strings loaded by the new binary) is supported. Should this be called out explicitly in the PR description/changelog as an operational rollback caveat?

## Since previous review (2026-06-15)

- Branch was force-pushed/rebased: old head `55e4ec4e2` is not an ancestor of new head `e3758df16` (same two commit messages, recreated hashes, rebased onto current `main`). Diff between trees: `pkg/plugins/config.go` (36 lines), `pkg/plugins/config_test.go` (41 lines), plus the unchanged rest of the PR.
- Global level redesigned: now requires explicit `Repos: ["*"]` instead of inferring global from empty `Repos`. Resolves prior `[blocking]` finding "Existing configs silently expand trust on upgrade" (part a) — an empty-Repos trigger stays inert, matching its old (dead) behavior. Part (b) of that finding (merge-across-levels for previously-shadowing repo triggers) remains unaddressed — tracked as its own blocking finding above.
- Same-level lock protection added to all three levels (`if s, exists := state[app.Name]; exists && s.locked { continue }` in each of the global/org/repo loops of `resolveTrustedApps`). Resolves prior `[should-fix]` finding "Same-level duplicates silently drop locks". New test case `"same-level duplicate cannot override lock"` covers it.
- `plugin-config-documented.yaml` not regenerated after the `Repos` doc comment changed, so it still documents the old empty-Repos-is-global semantics (blocking finding above).
- `cmpopts.IgnoreUnexported(Trigger{})` test fix survived the rebase (folded into the recreated commit).
- CI: one `pull-prow-integration` test failure notice posted 2026-06-16 (author has not yet re-run `/retest` since). No new PR reviews or inline comments since last review; labels unchanged (`approved`, no hold).
- Ran a 3-perspective maintainer review (code quality, maintainability, deployment risk) plus advisor synthesis, same head. Surfaced the `"*"`-activation risk, the supplemental-config escalation gap, the triplicated classification logic, the order-dependent same-level conflict resolution, and the hidden-state design concern — all new findings not previously captured, on top of confirming the earlier single-pass findings. All three reviewers and the advisor independently concluded request-changes; deployment risk rated HIGH.

## Checked

- All call sites migrated to `GetTrustedApps()`: generic-comment.go, pull-request.go, verify-owners.go, welcome.go, retitle.go. No remaining direct reads of the trigger field outside config.go (dco.go:301 is DCO's own field).
- `UnmarshalJSON` handles legacy string, new object, and mixed arrays; YAML path verified via sigs.k8s.io/yaml JSON conversion (TestTrustedAppsUnmarshalYAML).
- `sort.Strings(result)` makes resolution deterministic despite map iteration.
- First-match-wins semantics for non-trusted-apps trigger fields (OnlyOrgMembers, TrustedOrg, etc.) preserved exactly by the single-pass rewrite; no-match path still calls SetDefaults. Note: this first-match-wins semantics is inconsistent with TrustedApps' merge-across-levels semantics — see should-fix findings above.
- `cmpopts.IgnoreUnexported(Trigger{})` correctly present in both `mergeFrom` (config.go) and TestMergeFrom; cmpopts already imported in the test file. `append(config.DefaultDiffOpts, ...)` is safe today (literal has len==cap), though `slices.Clone` would be more defensive.
- `ValidateLockedTrustedApps` wired as an optional checkconfig warning, not a hard error — right call for upgrade safety, though deployment-risk perspective notes this means operators get no automatic signal unless they opt in.
- Test coverage: UnmarshalJSON/YAML, resolveTrustedApps (locks, untrusted, inheritance, three levels, same-level duplicate-lock), validateTrustedApps, GetTrustedApps nil behavior, ValidateLockedTrustedApps aggregate counts.
- Rollback: legacy `[]string` config remains loadable by the new binary; the reverse (new object-form config on an old binary) is not — see open question above.

## Open questions

- Can `hack/gen-prow-documented` be rerun before merge so `plugin-config-documented.yaml` reflects the `Repos: ["*"]` opt-in?
- Is merge-across-levels intended to apply to existing repo triggers that previously shadowed org trusted_apps, and should release notes call out the trust expansion?
- Should existing configs be scanned/warned for stray `Repos: ["*"]` entries before upgrade, given they silently activate?
- Should supplemental/per-repo plugin config files be restricted from declaring a global (`Repos: ["*"]`) trigger, or scoped to the org/repo they belong to?
- Should `ValidateLockedTrustedApps` compare effective trust values rather than flagging any name overlap (which flags agreement)?
- Should `locked: true` on a repo-level entry be a validation error?
- Should a trigger mixing `"*"` with specific org/repo names in `Repos` be rejected or warned on?
- Should same-level unlocked conflicts error out instead of silently resolving by list order?
- Should the global/org/repo classification rule be extracted into one shared helper instead of three independent implementations?
- Should DCO's separate `TrustedApps []string` eventually adopt the hierarchical type?
