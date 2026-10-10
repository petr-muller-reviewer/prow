---
issue: kubernetes-sigs/prow#737
title: "peribolos: add --ignore-repos flag for team-repo reconciliation"
state: open
labels: ""
main_sha: 71428b9c282ee8c9e7e9512068fccce86e7915da
triaged_at: 2026-07-21T23:36:29Z
verdict: accepted
refresh_log:
  - previous_triaged_at: 2026-05-30T13:33:41Z
    summary: Contributor volunteered to implement; another commenter suggested a companion --ignore-private-repos flag to avoid leaking private repo names via --ignore-repos values.
---

## Findings

### [cause] No per-repo escape hatch in configureTeamRepos
- detail: `configureTeamRepos` performs unconditional subtractive reconciliation. Any repo present in GitHub's team-repo list but absent from the YAML `want` map is removed (`permission = None`). There is no mechanism to declare a repo out of scope. The `--ignore-secret-teams` / `--ignore-enterprise-teams` pattern exists for teams but was never extended to repos.
- evidence: `cmd/peribolos/main.go:1495-1500`

### [related-code] configureTeamRepos — primary function to modify
- where: `cmd/peribolos/main.go:1468-1532`
- excerpt: |
    want := team.Repos
    have := map[string]github.RepoPermissionLevel{}
    // ...
    for haveRepo := range have {
        if _, wantRepo := want[haveRepo]; !wantRepo {
            actions[haveRepo] = github.None  // removal — no opt-out exists
        }
    }

### [related-code] options struct — where the new flag field goes
- where: `cmd/peribolos/main.go:46-70`
- excerpt: |
    ignoreSecretTeams     bool   // existing precedent
    ignoreEnterpriseTeams bool   // existing precedent
    // proposed: ignoreRepos flagutil.Strings

### [related-code] Flag registration — convention to follow
- where: `cmd/peribolos/main.go:80-103`
- excerpt: |
    flags.BoolVar(&o.ignoreSecretTeams, "ignore-secret-teams", false, "...")
    flags.BoolVar(&o.ignoreEnterpriseTeams, "ignore-enterprise-teams", false, "...")
    // proposed: flags.Var(&o.ignoreRepos, "ignore-repos", "...")

### [related-code] Outer call site for configureTeamRepos
- where: `cmd/peribolos/main.go:1007-1013`
- excerpt: |
    if err := configureTeamRepos(client, githubTeams, name, orgName, team); err != nil {

### [related-code] Recursive child-team call — must propagate ignore set
- where: `cmd/peribolos/main.go:1525-1527`
- excerpt: |
    if err := configureTeamRepos(client, githubTeams, childName, orgName, childTeam); err != nil {

### [related-code] TestConfigureTeamRepos — test patterns to extend
- where: `cmd/peribolos/main_test.go:2680-2888`
- excerpt: table-driven tests covering add/update/remove/child-team scenarios; no ignore cases yet

### [related-issue] kubernetes/org#4365 — prior incident motivating the request
- ref: kubernetes/org#4365
- relevance: Incomplete team-repo config caused peribolos to strip publishing bot permissions; emergency `--skip-removals` flag was added then reverted as too broad; targeted repo exclusion would have been the right fix.

### [comment] Contributor volunteered; companion flag suggested
- detail: SaaiAravindhRaja (2026-07-15) offered to implement the issue. StarMiner99 (2026-07-16) suggested a companion `--ignore-private-repos` flag, noting that passing private repo names as `--ignore-repos` values could leak them (e.g. via process args, CI logs) — cc'ing SaaiAravindhRaja.
- ref: comments on kubernetes-sigs/prow#737

## Checked

- `configureTeamRepos` does not accept an `options` struct — signature extension required; both call sites (outer loop line 1011, recursive line 1526) must be updated.
- `flagutil.Strings` / `flagutil.NewStrings()` is the right type for a multi-value string flag; `requiredAdmins` at line 54 is the precedent.
- No existing test in `TestConfigureTeamRepos` covers an ignore-repo scenario.
- No YAML schema changes to `pkg/config/org` needed for the CLI-flag approach.
- `configureTeams` (`main.go:713`) already models passing an ignore parameter through a reconciliation function — good reference for the implementor.

## Next steps

- Apply labels: `kind/feature`, `area/peribolos`, `help-wanted`.
- Comment on issue: welcome SaaiAravindhRaja's offer to implement; ask them to clarify the edge case — if a repo appears in both the YAML `want` map and `--ignore-repos`, should it be skipped entirely (no add/update) or only protected from removal? Also ask whether they intend to scope in the `--ignore-private-repos` companion flag StarMiner99 suggested, or leave it for a follow-up.
- When PR arrives: verify recursive child-team call propagates the ignore set and that `TestConfigureTeamRepos` includes cases for ignore-only, want+ignore, and child-team propagation.
- Note as non-goal for v1: YAML-level `ignore_repos` config (per-team granularity) — valid future evolution but out of scope for initial PR.

## Open questions

- When a repo is in both the YAML `want` map and `--ignore-repos`, should peribolos skip it entirely (no add/update) or only suppress removal? The issue's `have`-only filter would still *add* that repo if it's missing from GitHub — potentially surprising.
- Should `--ignore-repos` affect `--dump` output? (Likely no, but worth confirming with contributor.)
- Should `--ignore-private-repos` (StarMiner99's suggestion) be bundled into this same PR, or tracked as separate follow-up work? Bundling adds scope; separating risks the leak concern going unaddressed.
