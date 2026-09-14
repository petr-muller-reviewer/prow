---
pr: 624
title: "`tide` does not respect required contexts from Github Rulesets"
issue_url: https://github.com/kubernetes-sigs/prow/issues/624
reviewed_at: 2026-07-26T23:35:19Z
verdict: LEGITIMATE
effort: 3
labels: kind/feature, area/tide, lifecycle/rotten
refresh_log:
  - at: 2026-05-03
    summary: Initial triage
  - at: 2026-06-01T00:31:21Z
    summary: "k8s-triage-robot applied lifecycle/stale on 2026-05-31 (90d inactivity); no human activity, analysis unchanged"
  - at: 2026-07-26T23:35:19Z
    summary: "k8s-triage-robot applied lifecycle/rotten on 2026-06-30 (30d since stale); no human activity, analysis unchanged"
---

# Triage: Issue #624 — Tide + GitHub Rulesets

**Issue**: [github.com/kubernetes-sigs/prow/issues/624](https://github.com/kubernetes-sigs/prow/issues/624)
**Reported by**: oliver-goetz — 2026-03-02
**Assessment**: LEGITIMATE | Level 3 | kind/feature | area/tide

## TL;DR

Tide can infer required status checks from branch protection via `from-branch-protection`, but has **zero support** for GitHub Rulesets. The GitHub client has no Ruleset types, methods, or HTTP calls. The fix is a parallel `from-rulesets` config option. Rated **Level 3** due to polymorphic API types, layering complexity, coexistence with branch protection, and interface-wide changes.

## How `from-branch-protection` Works Today

1. Tide sync loop calls `GetTideContextPolicy()` — `pkg/config/tide.go:903-954`
2. If `FromBranchProtection` is `true` (line 920), calls `GetBranchProtection()`
3. Extracts `RequiredStatusChecks.Contexts` (plain string list) and adds to required set
4. Prow presubmit job names added via `BranchRequirements()`
5. Final `TideContextPolicy` used by `contextChecker` to gate merges

## Key Code Locations

| Component | File | Lines |
|---|---|---|
| `TideContextPolicy` struct | `pkg/config/tide.go` | 159-175 |
| `TideContextPolicyOptions` | `pkg/config/tide.go` | 189-194 |
| `GetTideContextPolicy()` | `pkg/config/tide.go` | 903-954 |
| Branch protection fetch | `pkg/config/tide.go` | 920-930 |
| Config merging | `pkg/config/tide.go` | 852-898 |
| `RepositoryClient` interface | `pkg/github/client.go` | 165-202 |
| Branch protection HTTP calls | `pkg/github/client.go` | 2727-2806 |
| `BranchProtection` type | `pkg/github/types.go` | 552 |
| Override plugin (also needs update) | `pkg/plugins/override/override.go` | 445-453 |
| Context checker | `pkg/tide/tide.go` | 961-984 |

## GitHub Rulesets API

### The Golden Endpoint

`GET /repos/{owner}/{repo}/rules/branches/{branch}`

- Returns **pre-aggregated** effective rules for a branch
- Includes org-level rulesets automatically
- Only returns `active` enforcement — `evaluate`/`disabled` excluded
- Resolves conditions — no fnmatch matching needed
- Only needs `Metadata` (read) permission — less than branch protection's `Administration` (read)
- Paginated (max 100 per page)

### Why It's Complex

- **Polymorphic responses** — mixed rule types with `type` discriminator, each with different `parameters`
- **Layering** — up to 150 rulesets per branch (75 repo + 75 org), union with most-restrictive-wins
- **Coexistence** — branch protection + Rulesets can be active simultaneously during migration
- **`integration_id`** — optional field pinning checks to a specific GitHub App (ignore for "what's required")
- **Interface-wide changes** — `RepositoryClient` extension touches all implementations/mocks

### Data Model Comparison

Branch protection (current, flat):
```json
{
  "required_status_checks": {
    "strict": true,
    "contexts": ["ci/test", "ci/lint"]
  }
}
```

Rulesets (polymorphic, nested):
```json
{
  "type": "required_status_checks",
  "parameters": {
    "required_status_checks": [
      {"context": "ci/test", "integration_id": 15368},
      {"context": "ci/lint"}
    ]
  }
}
```

## Proposed Solution

Parallel `from-rulesets` config option:

1. Add Ruleset types to `pkg/github/types.go` (polymorphic rule structs)
2. Add `GetBranchRules()` to `RepositoryClient` interface + implement with pagination
3. Add `FromRulesets` field to `TideContextPolicy`
4. New block in `GetTideContextPolicy()` alongside branch protection block (line 920-930)
5. Update override plugin for consistency
6. Update all fake/mock `RepositoryClient` implementations

**Backwards compatible** — opt-in, default off. Both options independently enableable.
**Estimated**: 8-12 files, 300-500 LOC, 3-5 days.

## Effort Assessment: Level 3

| Factor | Assessment | Indication |
|---|---|---|
| Scope | Moderate-Large (8-12 files incl. all mocks) | Level 2-3 |
| Complexity | High (polymorphic types, layering, coexistence, pagination) | Level 3 |
| Required Expertise | Deep (Rulesets API + RepositoryClient + Tide + override) | Level 3 |
| Clarity | Some uncertainty (coexistence design, integration_id handling) | Level 2-3 |
| Testing | Moderate-Complex (polymorphic deserialization, coexistence scenarios) | Level 2-3 |
| Backwards Compat | Fully compatible — new opt-in option | Level 1-2 |
| Architecture | Good fit with new polymorphic handling needed | Level 2-3 |
| External Deps | Stable API, complex response model | Level 2-3 |

**Labels**: `area/tide`, `kind/feature`, `lifecycle/rotten` (current). No difficulty label — Level 3, experienced contributors self-select.

### Since previous triage (2026-05-03)

- 2026-05-31: `k8s-triage-robot` applied `lifecycle/stale` after 90 days of inactivity. No human comments or linked PRs. Analysis and recommendations unchanged. To keep open, post `/remove-lifecycle stale`.

### Since previous triage (2026-06-01)

- 2026-06-30: `k8s-triage-robot` applied `lifecycle/rotten` after a further 30 days of inactivity (bot comment only; `lifecycle/stale` removed, `lifecycle/rotten` added). No human comments or linked PRs. Analysis and recommendations unchanged. Issue will be auto-closed after another 30 days of inactivity unless someone posts `/remove-lifecycle rotten`.

## Key Gotchas for Contributors

- Use `rules/branches/{branch}` endpoint — do NOT fetch all rulesets and do condition matching manually
- `integration_id` constrains *who* can satisfy a check, not *what* is required — extract `context` strings only
- When both `from-branch-protection` and `from-rulesets` are enabled, take the union of required contexts
- GitHub limitation: path-filtered workflows as required status checks cause permanent "Pending" — not solvable in Prow
- Rules endpoint needs `Metadata` (read) only — less than branch protection's `Administration` (read)

## Draft Comment

Tide currently supports inferring required contexts from GitHub branch protection rules via the `from-branch-protection` config option in `tide.context_options` (implemented in `pkg/config/tide.go`). When enabled, Tide calls `GetBranchProtection()` to fetch `RequiredStatusChecks.Contexts` and adds them to the set of required contexts before allowing a merge. However, Prow's GitHub client has no Ruleset API support at all — no types, no interface methods, no HTTP calls — so there's no way to extend this to Rulesets without building that foundation first.

The most natural approach would be a parallel `from-rulesets` config option that mirrors `from-branch-protection`. GitHub provides a helpful `GET /repos/{owner}/{repo}/rules/branches/{branch}` endpoint that returns pre-aggregated effective rules for a branch (including org-level rulesets, already filtered to active enforcement). The implementation would need: (1) new Ruleset types in `pkg/github/types.go` — note the API uses polymorphic rule objects with a `type` discriminator, more complex than branch protection's flat structure, (2) a `GetBranchRules()` method on the `RepositoryClient` interface in `pkg/github/client.go` with pagination support, (3) a `FromRulesets` field on `TideContextPolicy`, and (4) a new block in `GetTideContextPolicy()` alongside the existing branch protection block at `pkg/config/tide.go:920-930`. A key design consideration is coexistence: during migration, repos may have both branch protection and Rulesets active (GitHub layers both, union with most-restrictive-wins), so both `from-branch-protection` and `from-rulesets` should be independently enableable. The override plugin (`pkg/plugins/override/override.go`) also fetches branch protection for validation and should ideally gain Ruleset awareness for consistency.

## Links

- [Issue #624](https://github.com/kubernetes-sigs/prow/issues/624)
- [Full triage document](https://github.com/petr-muller/prow/blob/issue-triage-624/ISSUE-TRIAGE.md)
- [GitHub Rulesets REST API docs](https://docs.github.com/en/rest/repos/rules)
