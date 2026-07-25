---
issue: kubernetes-sigs/prow#179
title: "Feature Request: exclusive label sets"
state: open
labels: kind/feature, help wanted, sig/contributor-experience, sig/release, priority/important-longterm
main_sha: 0633879af8026d056e1a5dbe1e29f5a98f6acec3
triaged_at: 2026-07-23T10:15:41Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 3
recommended_labels: [help wanted, kind/feature]
---

## Findings

### [reproducibility] Missing exclusion is trivially reproducible
- detail: Comment `/priority important-soon` on an issue that already carries `priority/backlog` (or any other `priority/*` label). Both labels end up applied simultaneously; nothing removes the prior one.
- evidence: `pkg/plugins/label/label.go:223-259` (add-labels loop), `pkg/plugins/label/label.go:261-286` (remove-labels loop) — the two loops never cross-check each other.

### [cause] No generic mutual-exclusion concept exists for label prefixes
- detail: `handleComment` computes `labelsToAdd`/`labelsToRemove` independently from regex matches and applies them with no notion of "these labels are mutually exclusive." The only prefix-grouping logic that exists (`needs-*` category handling) decides whether to add a `needs-kind`/`needs-priority` label when all labels of a category are removed — it does not prevent multiple labels of a category from coexisting.
- evidence: `pkg/plugins/label/label.go:147-312` (`handleComment`); category grouping at `label.go:206-221` and helper `labelsWithCategory` at `label.go:374-383`.

### [related-code] Working precedent exists, but hardcoded to one plugin
- where: `pkg/plugins/lifecycle/lifecycle.go:32-34,102-146`
- excerpt: |
    lifecycleLabels = []string{labels.LifecycleActive, labels.LifecycleFrozen, labels.LifecycleStale, labels.LifecycleRotten}
    ...
    if !github.HasLabel(lbl, labels) && !remove {
        for _, label := range lifecycleLabels {
            if label != lbl && github.HasLabel(label, labels) {
                gc.RemoveLabel(org, repo, number, label)
            }
        }
        gc.AddLabel(org, repo, number, lbl)
    }
- relevance: This is exactly the "exclusive set" behavior the issue requests, but fixed to one hardcoded slice, one fixed regex (`/lifecycle <x>`, not the `/priority`-style prefix syntax), and lives in a separate plugin from `label.go` — not configurable and not reusable for `priority/` or `tracked/`. Introduced by commit `83bd40f13` ("prow: allow only one lifecycle label to persist", 2018); very likely the origin of the "5-year-old TODO" the issue references.

### [related-code] Label plugin configuration model — likely template for new config
- where: `pkg/plugins/config.go:447-495`
- excerpt: |
    type Label struct {
        AdditionalLabels []string                     `json:"additional_labels,omitempty"`
        RestrictedLabels map[string][]RestrictedLabel `json:"restricted_labels,omitempty"`
    }
- relevance: `RestrictedLabels` is keyed by `"*"` (global) / org / `org/repo` and resolved via `RestrictedLabelsFor` (config.go:459-495), the established idiom for global-vs-org-vs-repo scoping in this plugin. It governs write *permission*, not exclusivity, but its merge machinery (`Label.mergeFrom`, config.go:2378-2398) is the most likely template for a new exclusivity config. Maintainer `jberkus` stated in comments the new config should be "per Prow installation rather than per repo," suggesting a simpler global-only shape may be preferred over full `RestrictedLabels`-style scoping.

### [related-code] Comparable prefix/regex + scoping pattern in another plugin
- where: `pkg/plugins/config.go:921-957`, plugin at `pkg/plugins/require-matching-label/require-matching-label.go`
- excerpt: |
    type RequireMatchingLabel struct {
        Org, Repo, Branch string
        PRs, Issues       bool
        Regexp            string
        Re                *regexp.Regexp
        MissingLabel      string
        MissingComment    string
        GracePeriod       time.Duration
    }
- relevance: Same regexp/prefix-matching idea in the opposite direction (require at least one matching label, vs. forbid more than one). Useful reference for a flat, org/repo/branch-scoped config list with a compiled `Regexp` field.

### [related-issue] Origin of this request
- ref: kubernetes/test-infra#6095
- relevance: This issue was originally filed there at the request of the release team before the `kubernetes-sigs/prow` repo split; no code or discussion beyond the original request is expected there.

## Checked

- Whether `lifecycle`/`priority`/`tracked` already have any partial exclusivity implementation elsewhere in the codebase beyond the hardcoded `lifecycle` plugin — none found.
- `blunderbuss` and `sigmention` for prefix-based label logic — neither has any (`grep -n "prefix\|Prefix"` on both returned nothing).
- Whether `tracked/*` is currently a recognized label command prefix in Prow — it is not (no regex, no constant); it's only reachable today via the generic `/label tracked/foo` path if an admin adds it to `AdditionalLabels`.
- Whether `lifecycle` is among the built-in prefixes handled by `label.go` — it is not; it's handled entirely by the separate `lifecycle` plugin.
- Broader codebase search for "exclusive"/"restricted label set" concepts — only unrelated "mutually exclusive" config-validation comments elsewhere (branch protection Include/Exclude, decoration config), nothing about labels.

## Next steps

- Get `@MadhavJivrajani`/`@cblecker` to weigh in on the open config-scope question raised in `jberkus`'s 2026-07-15 comment: global-only exclusive-label config vs per-org/per-repo scoping like `RestrictedLabels`.
- Once scope is settled, point `anshulchikhale30-p` (currently engaged on the issue) at `pkg/plugins/lifecycle/lifecycle.go:102-146` and `pkg/plugins/config.go:447-495` as concrete implementation templates.
- Decide (can be deferred to PR review) whether to migrate the standalone `lifecycle` plugin onto the new generic mechanism or leave it as a separate, hardcoded implementation.
- If `tracked/*` support is meant to be in scope, decide how to add it as a recognized command prefix (built-in regex vs `AdditionalLabels`).
- Extend `TestHandleComment` (`pkg/plugins/label/label_test.go:53`) with exclusivity cases once implemented, using `TestAddLifecycleLabels` (`pkg/plugins/lifecycle/lifecycle_test.go:83`) as the test-pattern template.

## Open questions

- Should the new exclusive-label-sets config be global-only, or does it need per-org/per-repo scoping like `RestrictedLabels`?
- Should the standalone `lifecycle` plugin be migrated onto the new generic mechanism, or remain separate?
- Is `tracked/*` support actually in scope for this issue, given it isn't a recognized command prefix in Prow today?
</content>
