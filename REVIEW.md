---
pr: kubernetes-sigs/prow#555
title: "`peribolos`: add org roles feature"
head_sha: db4667f980c26d1a554b7e7ec22f1cceb14e29ad
base: main
reviewed_at: 2026-08-18T20:03:23Z
verdict: request-changes
author: hoxhaeris
assignee: ivankatliarchuk
labels:
  - size/XXL
  - ok-to-test
  - area/peribolos
stats: "+1636 / -1"
review_method: "Independent re-review (Opus 5), validating a prior multi-perspective review + gate"
refresh_log:
  - previous_sha: null
    new_sha: "6ea8d3b727950679245132be9cb544b34f1ac114"
    date: "2026-06-08T14:41:19Z"
    summary: "No code changes. PR went stale, author removed lifecycle/stale. PR now needs rebase."
  - previous_sha: "6ea8d3b727950679245132be9cb544b34f1ac114"
    new_sha: "27020ccb1915241d1af5cdb258ffb28888a668a4"
    date: "2026-07-26T23:03:39Z"
    summary: "Author rebased onto current main and added a fix excluding unassigned roles from dump output. /hold and lgtm removed."
  - previous_sha: "27020ccb1915241d1af5cdb258ffb28888a668a4"
    new_sha: "27020ccb1915241d1af5cdb258ffb28888a668a4"
    date: "2026-07-31T17:10:15Z"
    summary: "No code change. Independent re-review replaced the prior finding set: two of three converging concerns did not survive scrutiny, one new confirmed defect found (ignored teams lose org roles)."
  - previous_sha: "27020ccb1915241d1af5cdb258ffb28888a668a4"
    new_sha: "db4667f980c26d1a554b7e7ec22f1cceb14e29ad"
    date: "2026-08-18T20:03:23Z"
    summary: "Mechanical rebase only: PR content (fe920dbd9/90c8bb66a/db4667f98) diffs identically against merge-base except one unrelated one-line fix in pkg/config/org/org_test.go (diff.ObjectReflectDiff -> diff.Diff, a renamed test helper pulled in by the rebase). No new commits, comments, or reviews from the PR author or reviewers. needs-rebase label cleared; mergeable: MERGEABLE. All findings below stand unchanged."
gate:
  decision: hold
  gated_at: "2026-08-18T20:16:21Z"
  gated_head_sha: "db4667f980c26d1a554b7e7ec22f1cceb14e29ad"
  reviewed_head_sha: "db4667f980c26d1a554b7e7ec22f1cceb14e29ad"
---

# kubernetes-sigs/prow#555

Adds declarative management of GitHub organization custom role *assignments* via peribolos. Roles must pre-exist in GitHub; peribolos manages only who holds them.

- New `--fix-org-roles` flag (default false) plus `roles:` section in org config.
- `configureOrgRoles` iterates all GitHub roles, syncing configured ones and stripping assignments from unconfigured ones.
- 7 new `github.Client` methods behind a new `OrganizationRolesClient` interface, embedded into `Client`.
- `--dump` emits a `roles:` section; roles with no assignments are omitted.
- Users holding a role indirectly (via team membership) are preserved; users with pending org invitations are skipped.

Since previous review (2026-08-18): head moved `27020ccb1` → `db4667f98` via a mechanical rebase onto current `main` (~140 unrelated upstream commits absorbed). The PR's own diff against its merge-base is unchanged except one forced one-line fix for a renamed test helper in `pkg/config/org/org_test.go`. No new PR activity. All findings below are unaffected.

## Gate: HOLD (2026-08-18)

No activity since the last review pass: head unchanged in substance (`db4667f98` is the same mechanical rebase already accounted for), no new commits, comments, or reviews from anyone. `ivankatliarchuk`'s `CHANGES_REQUESTED` review (2026-03-08) predates this review's own findings and all 13 of its items are addressed except the `--fix-teams` coupling, which is dispositioned as correct-as-is (see Checked). Nothing in the current gating set has moved.

**Gating item (blocks merge):** *Ignored teams silently lose their organization roles* — `cmd/peribolos/main.go:1710-1768` (source: this REVIEW.md, `[blocking]`). `wantSet` (config-declared teams only) is diffed against `haveSet` (every team GitHub reports holding the role), and the difference is removed. A team excluded via `--ignore-secret-teams` or `--ignore-enterprise-teams` therefore has its org role stripped despite the flag's own contract, with no way in the config schema to exempt it. Confirmed empirically against this exact code (see Findings). Not addressed, not discussed by the author. **Blocks merge** as-is: this is a live correctness defect in a code path that ships whenever `--fix-org-roles` is enabled, not a style nit.

**Should-fix items, not gating on their own:**
- *`--dump` output is not round-trippable when teams are filtered* (`main.go:425-433`) — same root cause as the blocking item; will be naturally addressed by the same fix or requires its own if the fix only touches the sync path.
- *Embedding `OrganizationRolesClient` in `Client` breaks downstream implementers* (`pkg/github/client.go:79-89,291`) — real but consistent with how this project has always grown `github.Client`; worth a release note, not a blocker.
- *`errors` variable shadows the `errors` package* (`main.go:1750`, `1817`) — cosmetic, `go vet` clean.

**Merge risk (Area 2):** No new risk beyond what the findings already capture. `--fix-org-roles` remains opt-in, default `false` — no existing deployment is affected by merging as-is. The risk is entirely inside the opt-in path: an operator who combines `--fix-org-roles` with `--ignore-secret-teams` or `--ignore-enterprise-teams` (the latter shipped on `main` in May 2026, `daa3c7a62`) silently loses role assignments on teams they explicitly asked peribolos not to touch. No CRD/API-server-facing surface; this is a CLI tool operators run themselves, so blast radius is bounded to whoever runs peribolos with that flag combination, but it is a silent permission change with no warning or dry-run signal beyond the standard `--confirm=false` mechanism (which most operators won't rerun once they trust config as reviewed). No skill in `.claude/skills/` was applicable (no CRD/schema-compat skill present); assessed directly.

**Unblocks by:** fixing the ignored-team/role-removal asymmetry (thread ignored-team slugs through to `configureRoleTeamAssignments` and exclude them from `toRemove`, or diff against the full team list on both sides). The should-fix items can ride along or follow in a fast-follow; they don't need to gate this hold.

## Findings

### [blocking] Ignored teams silently lose their organization roles
- where: `cmd/peribolos/main.go:1710-1768`
- concern: `wantSet` is built from `githubTeams` (only config-declared teams — `configureTeams` `continue`s past secret teams at `main.go:796` and enterprise teams at `main.go:792` before recording them), while `haveSet` comes from `ListTeamsWithRole`, which returns every team on GitHub holding the role. The difference is deleted, so `--fix-org-roles` revokes roles from exactly the teams the operator opted out of managing. Contradicts both flags' documented contracts at `main.go:92-93` ("Do not dump or update secret teams if set" / "Skip enterprise teams and their members during reconciliation") and partially regresses `daa3c7a62` (PR #710, May 2026, same author), which exists to stop peribolos reconciling enterprise teams. `ValidateRoles` (`pkg/config/org/org.go:98`) rejects any role naming a team absent from `c.Teams`, so the exemption is inexpressible; declaring the team instead routes it to `missing` and peribolos attempts `CreateTeam` (`main.go:855-870`). Confirmed empirically against this head — a throwaway test driving `configureOrgRoles` with one config-declared team and one ignored team produced `level=info msg="Removed role security-manager from team enterprise-sec"`.
- excerpt: |
    normalizedTeams := make(map[string]string)
    for name, team := range githubTeams {
        normalizedTeams[strings.ToLower(name)] = team.Slug
    }
    ...
    haveSet := sets.New[string]()
    for _, team := range currentTeams {
        haveSet.Insert(team.Slug)
    }
    ...
    toRemove := haveSet.Difference(wantSet)
    for teamSlug := range toRemove {
        if err := client.RemoveOrganizationRoleFromTeam(orgName, teamSlug, roleID); err != nil {

### [should-fix] `--dump` output is not round-trippable when teams are filtered
- where: `cmd/peribolos/main.go:425-433`
- concern: Same filtered-vs-unfiltered asymmetry as the blocking finding, on the dump side. A role held by a team missing from `slugToName` (enterprise teams under `--ignore-enterprise-teams`) gets the raw **slug** appended to `roles.<name>.teams`, while the `teams:` section omits that team. Re-applying the dump fails peribolos' own validation: `role validation failed: - role "security-manager" references undefined team "enterprise-sec"`. Under `--ignore-secret-teams` the team is instead dropped silently, feeding the blocking finding above. Dump→apply idempotence is a documented peribolos property. Verified by feeding a synthesized dump through `ValidateRoles`.
- excerpt: |
    for _, team := range teamsWithRole {
        // Only include teams that are in slugToName (secret teams are excluded from this map)
        if name, ok := slugToName[team.Slug]; ok {
            teamNames = append(teamNames, name)
        } else if !ignoreSecretTeams {
            teamNames = append(teamNames, team.Slug)
        }
    }

### [should-fix] Embedding `OrganizationRolesClient` in `Client` breaks downstream implementers
- where: `pkg/github/client.go:79-89,291`
- concern: Adding 7 methods to the exported `github.Client` interface is a compile-time break for any out-of-tree type satisfying it — custom fakes, throttling/caching wrappers, decorators. Safe for callers, not for implementers. In-tree `fakegithub.FakeClient` is updated so the common case is covered, but this is an upstream library whose consumers we do not control. Needs a release note; the prior gate recorded this as unqualifiedly "additive, safe", which is wrong.
- excerpt: |
    type Client interface {
        ...
        OrganizationClient
        OrganizationRolesClient
        TeamClient

### [should-fix] `errors` local variable shadows the imported `errors` package
- where: `cmd/peribolos/main.go:1750`, `cmd/peribolos/main.go:1817`
- concern: `errors` is imported at `main.go:20`. Neither function uses the package, so this compiles and `go vet` is clean, but the codebase convention elsewhere is `errs` / `allErrors` / `updateErrors`. Same pattern as `var errors []string` in `pkg/config/org/org.go:143`.
- excerpt: |
    // Teams to add
    var errors []error

### [nit] `--fix-org-roles requires --fix-teams` error does not say why
- where: `cmd/peribolos/main.go:155-157`
- concern: The dependency is genuine, not arbitrary — `configureOrg` returns early at `main.go:1065-1068` when `fixTeams` is false, making `configureOrgRoles` unreachable, and team-name→slug resolution needs `configureTeams`' output. Two prior reviewers flagged this as "overly strict"; that reading is incorrect as the code stands and the author's rebuttal is right. Only the message needs improving.
- excerpt: |
    if o.fixOrgRoles && !o.fixTeams {
        return fmt.Errorf("--fix-org-roles requires --fix-teams")
    }

### [nit] Comment omits the `mixed` assignment case
- where: `cmd/peribolos/main.go:1793-1795`
- concern: The comment explains that `indirect` users are skipped, but not that `mixed` users (role held both directly and via a team) keep their indirect grant while their direct grant is removed. Behavior is correct; the comment under-describes it.
- excerpt: |
    // Only consider DIRECT assignments when building haveMap.
    // Users with "indirect" assignment have the role via team membership and should not be
    // removed just because they're not in the users list - they keep the role through their team.

### [nit] `readPaginatedResults` used with a wrapper object
- where: `pkg/github/client.go:5433-5451`
- concern: List Organization Roles returns `{"total_count": N, "roles": [...]}`; every other `readPaginatedResults` caller in this file accumulates a bare array. It works because pagination keys off the `Link` header, but it is the sole wrapper-shaped usage and deserves a one-line comment.
- excerpt: |
    func() interface{} {
        return &orgRolesResponse{}
    },

### [nit] `findOriginalRoleName` is O(n) inside the per-role loop
- where: `cmd/peribolos/main.go:1701-1708`
- concern: Called once per configured role inside the main processing loop. A `lowerToOriginal` map built alongside `configuredRoleNames` would remove the scan. Negligible at realistic role counts.
- excerpt: |
    func findOriginalRoleName(roles map[string]org.Role, lowerName string) string {
        for name := range roles {
            if strings.ToLower(name) == lowerName {

### [nit] Two consecutive loops over `orgConfig.Roles`
- where: `cmd/peribolos/main.go:1645-1656`
- concern: The role-existence validation loop and the `configuredRoleNames` construction loop iterate the same map back to back and can be merged.

### [nit] `ValidateRoles` accumulates `[]string` rather than `[]error`
- where: `pkg/config/org/org.go:143-160`
- concern: The rest of the codebase uses `[]error` + `utilerrors.NewAggregate`. Cosmetic consistency.
- excerpt: |
    var errors []string
    ...
    return fmt.Errorf("role validation failed:\n  - %s", strings.Join(errors, "\n  - "))

### [nit] Dump test never exercises slug→name resolution
- where: `cmd/peribolos/main_test.go:2033-2090`
- concern: The "dump organization with roles" case configures no teams, so `slugToName` is always empty and the mapping that makes dumps round-trippable is untested. Adding a team would cover the property that the should-fix dump finding breaks.

### [question] Does `ListTeamsWithRole` return enterprise teams?
- concern: The blocking finding assumes GitHub reports enterprise teams among the teams assigned to an org role, which is the documented purpose of enterprise team role grants. The author's manual testing was against a test org; confirming this specific case would settle the severity.

### [question] What is the intended contract for teams peribolos does not manage?
- concern: The author's own test at `main_test.go:4734` ("remove team assignment", `githubTeams: {}`) asserts that a team unknown to the config loses the role. That is defensible for unmanaged teams, but nothing in the code distinguishes "not in config" from "explicitly ignored". Which of the two was intended determines whether the fix is to exempt ignored teams only, or to restrict role management to declared teams entirely.

## Checked
- `go build ./...`, `go vet ./cmd/peribolos/...`, `go test ./cmd/peribolos/... ./pkg/config/org/...` — all pass on `27020ccb1`.
- Diff against `merge-base(27020ccb1, upstream/main)` is purely additive (+1636/−1); `TeamTypeEnterprise`, `Team.Type`, `OrgInvitation.FailedAt`/`FailedReason` are present and used. The rebase reverted nothing; the `needs-rebase` label is stale (`gh pr view` reports `mergeable: MERGEABLE`).
- Role-existence validation runs before any mutation (`main.go:1637-1643`), and `configureOrg` validates the config at `main.go:1013-1018` before API calls.
- Removals are logged at **Info**, not Debug (`main.go:1766`, `main.go:1839`); peribolos defaults to `--log-level=info` (`main.go:104`). The prior review's "cleanup is invisible at Debug level" claim is false, and its cited `main.go:411` is the dump path's role count, not the cleanup path.
- Indirect-assignment filtering (`main.go:1795-1801`) correctly preserves users holding a role via team membership.
- Pending-invitee skipping (`main.go:1822-1826`) and the invitee-set threading from `configureOrgMembers` (`main.go:607`) work as described.
- Mutating client methods rely on `requestRawWithContext` short-circuiting non-GET in dry mode; no redundant `c.dry` guards remain.
- Case-insensitive matching for role names, team names, and logins; role-name collision detection in `ValidateRoles`.
- `ListOrganizationRoles` paginates via `readPaginatedResults`.
- All 13 items from @ivankatliarchuk's `CHANGES_REQUESTED` review are addressed in code, except the `--fix-teams` coupling, which the author declined with a justification that holds up.
- No secrets, tokens, or credentials in the diff.

## Open questions
- Should `--fix-org-roles` exempt teams excluded by `--ignore-secret-teams` / `--ignore-enterprise-teams` from role removal, or should role management be restricted to config-declared teams altogether?
- Should the dump emit anything at all for roles held by filtered-out teams, given the current raw-slug output produces a config that fails `ValidateRoles`?
- Is a release note planned for the `github.Client` interface growth, for downstream implementers?
- Was the enterprise-team + `--fix-org-roles` combination covered in the manual testing against the test org?
