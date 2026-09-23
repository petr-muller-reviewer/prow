---
pr: kubernetes-sigs/prow#555
title: "`peribolos`: add org roles feature"
head_sha: a821885197538f50646200239bfbedd93e2e2756
base: main
reviewed_at: 2026-09-23T19:22:09Z
verdict: request-changes
---

# Review of kubernetes-sigs/prow#555

## Verdict

Request changes.

Role reconciliation can revoke custom-role grants from teams explicitly excluded by the ignore flags. Three other paths can produce incomplete dumps, reject valid configuration, or leave earlier mutations applied after a role-name error.

## What this PR does

- Adds organization custom-role assignments for teams and users to the peribolos configuration.
- Adds GitHub client methods to list roles and assignments and to add or remove assignments.
- Adds role assignments to configuration dumps and a `--fix-org-roles` reconciliation option.
- Adds configuration validation, tests, and peribolos documentation for the feature.

## Findings

### [blocking] Preserve role grants on ignored teams

- where: `cmd/peribolos/main.go:1724-1749`
- concern: With `--ignore-secret-teams` or `--ignore-enterprise-teams`, `configureTeams` excludes those teams from the desired configuration, but role reconciliation still includes their existing grants in `haveSet` and removes everything in `haveSet.Difference(wantSet)`. Running role reconciliation can therefore revoke grants from teams the operator explicitly asked peribolos to ignore. Exclude ignored team slugs from removal and cover both flags in tests.
- excerpt: |
    haveSet := sets.New[string]()
    for _, team := range currentTeams {
        haveSet.Insert(team.Slug)
    }
    toRemove := haveSet.Difference(wantSet)
    for teamSlug := range toRemove {
        if err := client.RemoveOrganizationRoleFromTeam(orgName, teamSlug, roleID); err != nil {

### [should-fix] Fail when a dump cannot read role assignments

- where: `cmd/peribolos/main.go:420-440`
- concern: Failures listing roles or their team/user assignments are logged as warnings while the dump succeeds with roles omitted. A pipeline can capture this incomplete YAML as a backup without detecting the lost assignments. Return an error, or provide an explicit machine-detectable incomplete result.
- excerpt: |
    roles, err := client.ListOrganizationRoles(orgName)
    if err != nil {
        logrus.WithError(err).Warn("Failed to list organization roles; omitting roles from dump")
        roles = nil
    }

### [should-fix] Validate role users according to `--fix-org-members`

- where: `pkg/config/org/org.go:186-200`
- concern: Any nonempty admins or members list makes validation treat that list as the complete organization membership set. When membership management is disabled, a valid role user managed elsewhere is rejected merely because the configuration lists some other members. Apply this check only when org membership is being managed, or validate against actual membership.
- excerpt: |
    validateUsers := len(c.Members) > 0 || len(c.Admins) > 0
    if !validateUsers {
        continue
    }
    for _, user := range role.Users {
        if !availableUsers[github.NormLogin(user)] {
            errors = append(errors, fmt.Sprintf("role %q references user %q who is not an org member", roleName, user))
        }
    }

### [should-fix] Validate configured role names before earlier mutations

- where: `cmd/peribolos/main.go:1660-1664`
- concern: The remote role-existence check occurs in `configureOrgRoles`, after organization, repository, and team reconciliation. A misspelled role can therefore cause an error only after those earlier changes have been applied. Move the check into pre-mutation validation.
- excerpt: |
    // Validate all configured roles exist in GitHub BEFORE any mutations
    for roleName := range orgConfig.Roles {
        if _, ok := githubRolesByName[strings.ToLower(roleName)]; !ok {
            return fmt.Errorf("role %q does not exist in organization %s - create the role in GitHub before assigning it", roleName, orgName)
        }
    }

## Checked

- Reviewed the full PR diff at `a821885197538f50646200239bfbedd93e2e2756`, including configuration, dump, reconciliation, client methods, tests, and documentation.
- Compared the new REST endpoint shapes with GitHub's official API documentation.
- Ran `go test ./cmd/peribolos ./pkg/config/org ./pkg/github` successfully.
- Existing tests do not cover ignored teams with configured roles or HTTP responses for the new client methods.

## Open questions

- Is the expansion of the exported `github.Client` interface intended to require downstream fake and wrapper implementations to add these methods?
