---
pr: kubernetes-sigs/prow#799
title: "feat(label): support configurable exclusive label sets"
head_sha: bd9671c139db78a6d381e40b2c3ff2ca4deaf108
base: main
reviewed_at: 2026-07-26T23:25:04Z
verdict: request-changes
---

## Summary

Adds `ExclusiveLabelPrefixes` to the `label` plugin config. When a label matching a configured prefix is added, existing labels sharing that prefix are queued for removal. Touches `pkg/plugins/config.go` (field + merge), `pkg/plugins/label/label.go` (core logic), tests, generated docs.

Confirmed via independent multi-perspective review (code-quality, maintainability, deployment-risk reviewers + advisor synthesis): all three independently converged on the hardcoded-defaults issue as the primary blocker, and code-quality independently rated the case-sensitivity bug as critical.

## Findings

### [blocking] hardcoded default prefixes change behavior for all existing installations without opt-in
- where: `pkg/plugins/label/label.go:41` and `pkg/plugins/label/label.go:206`
- concern: `defaultExclusiveLabelPrefixes = []string{"priority/", "lifecycle/", "tracked/"}` is unconditionally merged into the exclusive set for every repo running the label plugin, even with `exclusive_label_prefixes` unset. This makes `priority/*`, `lifecycle/*`, `tracked/*` mutually exclusive for every existing upstream user with no config change and no mention in the PR description. `lifecycle/*` in particular is already managed by the separate `lifecycle` plugin, which has its own exclusivity handling — this can race or conflict with it. No test covers this default-prefix path at all (the only added test explicitly passes `exclusiveLabelPrefixes: []string{"priority/"}`). This should default to empty and be purely opt-in.
- excerpt: |
    defaultExclusiveLabelPrefixes = []string{"priority/", "lifecycle/", "tracked/"}
    ...
    exclusiveLabelPrefixes := uniqueExclusiveLabelPrefixes(append(defaultExclusiveLabelPrefixes, config.ExclusiveLabelPrefixes...))

### [blocking] silent fleet-wide behavior change: `/priority` now auto-removes existing labels with zero config change
- where: `pkg/plugins/label/label.go:41,206`
- concern: since `priority` is one of the label plugin's always-active regex categories (`labelRegex` includes `priority`), the hardcoded default above means any `/priority X` command on any repo running the label plugin starts silently removing that issue/PR's existing `priority/*` label the moment this ships — no operator opts in, no restart-time signal, and it's not documented as a default anywhere (`plugin-config-documented.yaml` only shows the new field as empty). This is a one-way change: rollback reverts the binary but does not restore labels already removed by the new logic while it was live. Independently confirmed by the deployment-risk reviewer as HIGH risk.
- excerpt: |
    defaultExclusiveLabelPrefixes = []string{"priority/", "lifecycle/", "tracked/"}

### [should-fix] duplicates existing `lifecycle` plugin's own exclusivity logic for `lifecycle/*`
- where: `pkg/plugins/label/label.go:41` vs `pkg/plugins/lifecycle/lifecycle.go`
- concern: the separate `lifecycle` plugin already implements exclusivity for `lifecycle/{active,frozen,stale,rotten}` via its own `/lifecycle` command and an exact-enum match. This PR's default `lifecycle/` prefix creates a second, divergent implementation of the same invariant (prefix-match vs exact-enum, different case-handling), reachable via `/label lifecycle/...` if that prefix is whitelisted through `AdditionalLabels`/`RestrictedLabels`. Two independent sources of truth for the same label family is a likely source of future incident-debugging confusion ("why was this label removed — which plugin did it?").
- excerpt: |
    defaultExclusiveLabelPrefixes = []string{"priority/", "lifecycle/", "tracked/"}

### [question] where does the `tracked/` default prefix come from?
- where: `pkg/plugins/label/label.go:41`
- concern: `tracked/` appears nowhere else in the codebase — no constant, no plugin, no docs, no mention in the PR description. Its presence in a shared default list applied to every Prow installation looks like it encodes one org's private labeling convention rather than a general-purpose default.
- excerpt: |
    defaultExclusiveLabelPrefixes = []string{"priority/", "lifecycle/", "tracked/"}

### [should-fix] stale label snapshot misses conflicts within a single comment
- where: `pkg/plugins/label/label.go:378-394` (`conflictingExclusiveLabels`), called from `label.go:262`
- concern: `conflictingExclusiveLabels` is evaluated against `issueLabels`, fetched once before the add-loop and never updated as the loop adds labels. If one comment adds two labels sharing an exclusive prefix (e.g. two `/label` commands hitting the same configured prefix), neither is flagged as conflicting with the other, so both survive — breaking the "mutually exclusive" guarantee.
- excerpt: |
    for _, labelToAdd := range labelsToAdd {
        ...
        if err := gc.AddLabel(org, repo, e.Number, labelToAdd); err != nil {
            ...
            continue
        }
        labelsToRemove = append(labelsToRemove, conflictingExclusiveLabels(issueLabels, labelToAdd, exclusiveLabelPrefixes)...)
    }

### [should-fix] case-sensitive equality check inconsistent with rest of file
- where: `pkg/plugins/label/label.go:388`
- concern: `existingLabel == labelToAdd` is a plain string comparison, while the rest of the codebase uses `strings.EqualFold` for label comparisons (see `github.HasLabel`). `labelToAdd` is already lowercased by the regex-extraction helpers, but `issueLabels` come straight from the GitHub API with arbitrary casing. A label that already exists with different casing than `labelToAdd` would fail this equality check and could be queued for removal of the very label just added.
- excerpt: |
    if existingLabel == labelToAdd || !hasExclusiveLabelPrefix(existingLabel, []string{matchingPrefix}) {
        continue
    }

### [nit] single-element slice allocation to reuse multi-prefix helper
- where: `pkg/plugins/label/label.go:388`
- concern: `hasExclusiveLabelPrefix(existingLabel, []string{matchingPrefix})` wraps one string in a slice per existing label just to call the multi-prefix helper. A direct `strings.HasPrefix(strings.ToLower(existingLabel), matchingPrefix)` would avoid the per-call allocation and read more directly.
- excerpt: |
    if existingLabel == labelToAdd || !hasExclusiveLabelPrefix(existingLabel, []string{matchingPrefix}) {

### [nit] append onto package-level slice is fragile
- where: `pkg/plugins/label/label.go:206`
- concern: `append(defaultExclusiveLabelPrefixes, config.ExclusiveLabelPrefixes...)` appends onto a shared package `var`. It's safe today only because `len(defaultExclusiveLabelPrefixes) == cap(...)` forces reallocation; a future change that pre-allocates capacity on that var would introduce a data race across concurrent `handleComment` calls. Prefer copying into a fresh slice first.
- excerpt: |
    defaultExclusiveLabelPrefixes = []string{"priority/", "lifecycle/", "tracked/"}

### [nit] mergeFrom doesn't dedup ExclusiveLabelPrefixes
- where: `pkg/plugins/config.go:2385-2389` (`Label.mergeFrom`)
- concern: `l.ExclusiveLabelPrefixes = append(l.ExclusiveLabelPrefixes, other.ExclusiveLabelPrefixes...)` just appends without dedup, unlike the runtime `uniqueExclusiveLabelPrefixes` dedup in label.go. Functionally harmless since runtime dedup happens later, but inconsistent with how other merge fields in this function are handled.
- excerpt: |
    l.ExclusiveLabelPrefixes = append(l.ExclusiveLabelPrefixes, other.ExclusiveLabelPrefixes...)

### [nit] no validation that configured prefixes end in a separator
- where: `pkg/plugins/config.go:458-461`, `pkg/plugins/label/label.go` (`exclusiveLabelPrefix`/`hasExclusiveLabelPrefix`)
- concern: prefixes are matched with plain `strings.HasPrefix`. A configured prefix missing the trailing `/` (e.g. `track` instead of `tracked/`) would also match unrelated labels like `trackless/foo`. Worth documenting the expectation or validating/normalizing at config-load time.
- excerpt: |
    if prefix != "" && strings.HasPrefix(label, prefix) {

### [nit] no log line distinguishes exclusivity-triggered removal from explicit /remove-*
- where: `pkg/plugins/label/label.go:262-290`
- concern: labels queued via `conflictingExclusiveLabels` are merged into the same generic `labelsToRemove` path as explicit `/remove-*` commands, with no distinguishing log field. During an incident ("why was `priority/backlog` removed from my PR?") the audit trail only shows a generic `RemoveLabel` call.
- excerpt: |
    labelsToRemove = append(labelsToRemove, conflictingExclusiveLabels(issueLabels, labelToAdd, exclusiveLabelPrefixes)...)

### [question] was the `continue` after failed AddLabel intentional and worth calling out?
- where: `pkg/plugins/label/label.go:258-262`
- concern: The PR adds a `continue` in the `AddLabel` error branch so conflict-removal is skipped when the add itself failed — a correct, welcome fix, but it's not mentioned in the PR description as an intentional behavior change alongside the new feature.
- excerpt: |
    if err := gc.AddLabel(org, repo, e.Number, labelToAdd); err != nil {
        log.WithError(err).WithField("label", labelToAdd).Error("GitHub failed to add the label")
        continue
    }

## Checked
- `Label.mergeFrom` correctly appends `ExclusiveLabelPrefixes` (aside from dedup nit above).
- `plugin-config-documented.yaml` regenerated correctly for the new field (though it doesn't reveal the hardcoded defaults — see blocking finding).
- Permission/restricted-label gating (`canUserSetLabel`) is unaffected — new removal path goes through the same remove-loop with the same checks; confirmed independently by deployment-risk review — no permission-bypass risk, risk is purely behavioral/data-loss.
- Added positive-path test (`TestHandleComment` "Exclusive label prefixes remove conflicting labels") correctly exercises configured-prefix removal end-to-end.
- No security concerns — no new bypass of existing label permission checks.
- Config schema change is additive/`omitempty`-tagged; existing YAML/JSON configs continue to parse without error (no breaking change at the config-schema level).

## Open questions
- Why hardcode `priority/`, `lifecycle/`, `tracked/` as always-on defaults rather than making the feature purely opt-in via `exclusive_label_prefixes`? This changes behavior for every existing repo using the label plugin without any config change.
- Is the interaction with the separate `lifecycle` plugin's own exclusivity handling for `lifecycle/*` labels considered? Could the two conflict or double-remove?
- Should `conflictingExclusiveLabels` use the running set of labels-added-so-far rather than the pre-comment snapshot, to catch conflicts between two labels added in the same comment?
- Where does the `tracked/` prefix convention come from — is it specific to one downstream user's setup rather than a general-purpose default?
- If the hardcoded defaults are kept in some form, will there be a release note / migration announcement so operators can audit exposure before upgrading?
