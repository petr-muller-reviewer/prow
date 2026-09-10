---
pr: kubernetes-sigs/prow#555
title: "`peribolos`: add org roles feature"
head_sha: a821885197538f50646200239bfbedd93e2e2756
base: main
reviewed_at: 2026-09-10T12:44:42Z
verdict: request-changes
---

# kubernetes-sigs/prow#555

## Findings

### [blocking] Preserve ignored teams’ role assignments
- where: `cmd/peribolos/main.go:1724-1749`
- concern: `configureTeams` excludes secret and enterprise teams when their respective ignore flags are enabled, so they cannot enter `githubTeams` or `wantSet`. `ListTeamsWithRole` still returns their existing assignments, which enter `haveSet`; the difference is then removed. A role configured for another team therefore revokes a role from a team that `--ignore-secret-teams` or `--ignore-enterprise-teams` promised not to update. The latest commit fixes this only in dump mode; pass ignored slugs to reconciliation and exclude them from `toRemove`.
- excerpt: |
    haveSet := sets.New[string]()
    for _, team := range currentTeams {
        haveSet.Insert(team.Slug)
    }
    ...
    toRemove := haveSet.Difference(wantSet)
    for teamSlug := range toRemove {
        if err := client.RemoveOrganizationRoleFromTeam(orgName, teamSlug, roleID); err != nil {

## Checked

- Current PR head `a821885197538f50646200239bfbedd93e2e2756`; 4 commits, +1696/-1 overall.
- Configured roles are now reconciled independently; roles absent from config are left untouched.
- Remote role existence is checked before role mutations, role/user comparisons are case-normalized, and indirect user assignments are preserved.
- Dumping logs and omits unavailable role data instead of failing the entire dump; it also excludes ignored teams from emitted role assignments.
- Read the focused Peribolos/config tests and the role client call sites. The role client has no HTTP-level tests, and the reconciliation tests do not cover an ignored team already assigned to a configured role.

## Open questions

- Should ignored team slugs be threaded into `configureRoleTeamAssignments`, so role reconciliation has the same ignore contract as team reconciliation and dump?
