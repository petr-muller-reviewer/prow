---
pr: kubernetes-sigs/prow#675
title: "trigger: add hierarchical `trusted_apps` config with global/org/repo levels"
head_sha: 55e4ec4e25ae945a4ebf5143cb93b6549c1210e0
base: main
reviewed_at: 2026-06-15T17:02:56Z
verdict: request-changes
refresh_log:
  - old_sha: 18779e865c90b365b434733217b3fd008de07bdf
    new_sha: 55e4ec4e25ae945a4ebf5143cb93b6549c1210e0
    summary: "1-line test fix (cmpopts.IgnoreUnexported in TestMergeFrom). Hold lifted — do-not-merge/hold label removed by author on 2026-06-15."
---

## Summary

Replaces flat `[]string` TrustedApps on `Trigger` with structured `TrustedApp` (`name`, `untrusted`, `locked`). Hierarchical resolution global -> org -> repo in `TriggerFor` via `resolveTrustedApps`; result exposed through `GetTrustedApps()`. Backward-compatible `UnmarshalJSON` (legacy string list still parses). Optional `checkconfig` warning `validate-locked-trusted-apps` via `ValidateLockedTrustedApps()`. All five call sites migrated to the accessor.

## Findings

### [blocking] Existing configs silently expand trust on upgrade
- where: `pkg/plugins/config.go:1139-1177` (TriggerFor)
- concern: Two behavior changes for unmodified existing configs. (a) Triggers with empty `Repos` were previously inert — old `TriggerFor` set lookups never matched an empty set, and no validation rejects such entries — but now become the global level and grant trust instance-wide; a forgotten/mistaken entry activates on upgrade. (b) A repo-level trigger previously shadowed the org trigger entirely; now trusted apps merge across levels, so a repo that deliberately defined a narrow list also inherits all org/global apps. Trusted apps gate automatic CI execution on PRs, so both broaden trust implicitly. Needs at least a prominent release-note/migration callout; ideally the global level requires explicit opt-in rather than reinterpreting previously-dead entries.
- excerpt: |
    case len(trigger.Repos) == 0:
        globalApps = append(globalApps, trigger.TrustedApps...)

### [should-fix] Same-level duplicates silently drop locks
- where: `pkg/plugins/config.go:1188-1204` (resolveTrustedApps)
- concern: The lock check only protects against lower levels; within a level, entries overwrite unconditionally. Two global triggers both naming app1 — first `{locked: true}`, second `{untrusted: true}` — the second wins and the lock vanishes. `validateTrustedApps` only dedupes within a single trigger's list; cross-trigger same-level duplicates are unvalidated. `locked` is a security guarantee and is defeatable by an unflagged duplicate. Either honor locks within a level or validate cross-trigger duplicates per level.
- excerpt: |
    for _, app := range globalApps {
        state[app.Name] = &trustedAppState{
            trusted: !app.Untrusted,
            locked:  app.Locked,
        }
    }

### [should-fix] ValidateLockedTrustedApps flags agreement as violation
- where: `pkg/plugins/config.go:~1690-1760` (ValidateLockedTrustedApps)
- concern: Checks only name membership, not whether the lower level conflicts with the locked value. Global locks app1 trusted; org also lists app1 trusted (same value) -> flagged. Noisy for a warning users opt into. Also self-flagging edge: a single trigger with `Repos: ["myorg", "myorg/myrepo"]` and a locked app populates orgLocked from itself, then its own repo entry is flagged against its own lock. Compare effective trust values instead of name presence.

### [should-fix] `locked: true` at repo level silently ignored without validation
- where: `pkg/plugins/config.go:1205-1213` (resolveTrustedApps repo loop), `pkg/plugins/config.go:~1655` (validateTrustedApps)
- concern: Repo level drops `Locked` by design (commented), but `validateTrustedApps` neither errors nor warns when a repo-level entry sets it. User gets no feedback that the field is meaningless there.
- excerpt: |
    // locked is not set here: repo is the lowest level, so there is nothing to lock against.
    state[app.Name] = &trustedAppState{trusted: !app.Untrusted}

### [nit] Indentation error in test loop body
- where: `pkg/plugins/config_test.go` (TestResolveTrustedApps, ~749)
- concern: `got := trigger.GetTrustedApps()` has an extra leading tab relative to surrounding lines.
- excerpt: |
    trigger := tc.config.TriggerFor(tc.org, tc.repo)
        got := trigger.GetTrustedApps()

### [nit] Constant column alignment broken
- where: `cmd/checkconfig/main.go:128`
- concern: `validateLockedTrustedAppsWarning` padding breaks the column alignment of the surrounding const block.
- excerpt: |
    validateConfigUpdaterACLsWarning               = "validate-config-updater-acls"
    validateLockedTrustedAppsWarning                = "validate-locked-trusted-apps"

### [question] Mixed Repos lists get level-classified per lookup
- where: `pkg/plugins/config.go:1145-1163` (TriggerFor switch)
- concern: A trigger with `Repos: ["myorg", "myorg/myrepo"]` counts as repo-level for myorg/myrepo (repo case wins in the switch, so its `locked` is dropped there) but org-level for sibling repos. Surprising classification; consider documenting or warning on mixed org+repo lists.

### [question] GetTrustedApps returns nil without TriggerFor
- where: `pkg/plugins/config.go:599-603`
- concern: `resolvedTrustedApps` is only populated by `TriggerFor`; a directly constructed `Trigger` returns nil even with `TrustedApps` set. Fails closed (no apps trusted) — correct security default, so low risk — but a doc comment on the `TrustedApps` field warning against reading it directly would help future callers.

### [question] DCO plugin's TrustedApps []string may confuse readers
- where: `pkg/plugins/config.go:897`, `pkg/plugins/dco/dco.go:301`
- concern: DCO has its own flat `TrustedApps []string`, unrelated to the trigger type; `config.TrustedApps` in dco.go looks like a missed migration at a glance. A clarifying comment on the DCO field would help. Whether DCO should adopt the hierarchical type is separate-PR scope.

## Since previous review (2026-06-09)

- One new commit `55e4ec4e2`: adds `cmpopts.IgnoreUnexported(Trigger{})` to the `TestMergeFrom` assertion (was already present in the `mergeFrom` production code; this aligns the test). 1 file, 1 line changed.
- Author ran `/retest` (2026-06-15) and `/hold cancel` (2026-06-15) — the `do-not-merge/hold` label is now removed. PR is merge-eligible pending LGTM.
- No review comments or formal reviews since last review. The earlier COMMENTED review by jmguzik (2026-04-15) predates our review.
- No findings from our review were addressed by the new commit; all blocking/should-fix items remain open.

## Checked

- All call sites migrated to `GetTrustedApps()`: generic-comment.go, pull-request.go, verify-owners.go, welcome.go, retitle.go. No remaining direct reads of the trigger field outside config.go (dco.go:301 is DCO's own field).
- `UnmarshalJSON` handles legacy string, new object, and mixed arrays; YAML path verified via sigs.k8s.io/yaml JSON conversion (TestTrustedAppsUnmarshalYAML).
- `sort.Strings(result)` makes resolution deterministic despite map iteration.
- First-match-wins semantics for non-trusted-apps trigger fields (OnlyOrgMembers, TrustedOrg, etc.) preserved exactly by the single-pass rewrite; no-match path still calls SetDefaults.
- `cmpopts.IgnoreUnexported(Trigger{})` correctly added in both `mergeFrom` (config.go:2464) and TestMergeFrom; cmpopts already imported in the test file. `append(config.DefaultDiffOpts, ...)` is safe today (literal has len==cap), though `slices.Clone` would be more defensive.
- `ValidateLockedTrustedApps` wired as an optional checkconfig warning, not a hard error — right call for upgrade safety.
- Test coverage: UnmarshalJSON/YAML, resolveTrustedApps (locks, untrusted, inheritance, three levels), validateTrustedApps, GetTrustedApps nil behavior, ValidateLockedTrustedApps aggregate counts.

## Open questions

- Should the global level require explicit opt-in (distinct config key) instead of reinterpreting previously-inert empty-Repos triggers?
- Is merge-across-levels intended to apply to existing repo triggers that previously shadowed org trusted_apps, and should release notes call out the trust expansion?
- Should locks be honored (or duplicates rejected) within the same level, given two same-level triggers can currently drop a lock silently?
- Should ValidateLockedTrustedApps compare effective trust values rather than flagging any name overlap (which flags agreement and can self-flag mixed Repos triggers)?
- Should `locked: true` on a repo-level entry be a validation error?
- Should DCO's separate `TrustedApps []string` eventually adopt the hierarchical type?
