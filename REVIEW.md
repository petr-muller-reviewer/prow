---
pr: kubernetes-sigs/prow#555
title: "`peribolos`: add org roles feature"
head_sha: a0ba73401af9961d586043b59d15881b92d222fa
base: main
reviewed_at: 2026-10-05T22:28:07Z
verdict: request-changes
---

# Review of kubernetes-sigs/prow#555

## Verdict

Request changes.

This opt-in feature preserves existing configuration behavior, and its role reconciler is mostly clear and well tested. However, a remote-role failure can occur after unrelated organization mutations, and accepted `@`-prefixed users are submitted to the API unnormalized. Role endpoint contract coverage is also needed before relying on this privileged API integration broadly.

## What this PR does

- Adds organization custom-role declarations for teams and users to peribolos configuration.
- Adds GitHub client methods to list roles and reconcile direct role assignments.
- Includes roles in dumps and reconciliation behind `--fix-org-roles`.
- Adds validation, dump behavior, client error classification, tests, and operator documentation for the feature.

## Findings

### [blocking] Preflight remote roles before mutable reconciliation

- where: `cmd/peribolos/main.go:1170-1259, 1809-1831`
- concern: Role API access and configured role-name validation happen after metadata, membership, repository, team, and team-membership reconciliation. A missing role, unavailable endpoint, or insufficient permission can leave those earlier changes applied while no roles are reconciled. List and validate configured roles before any mutable subsystem runs.
- excerpt: |
    githubTeams, ignoredTeamSlugs, ignoredSecretTeamSlugs, err := configureTeams(...)
    ...
    } else if err := configureOrgRoles(client, orgName, orgConfig, githubTeams, ...); err != nil {
        return fmt.Errorf("failed to configure %s organization roles: %w", orgName, err)
    }
    ...
    roles, err := client.ListOrganizationRoles(orgName)
    if err != nil {
        return fmt.Errorf("failed to list organization roles: %w", err)
    }

### [blocking] Submit normalized usernames to the role API

- where: `cmd/peribolos/main.go:1963-2009`
- concern: `github.NormLogin` accepts `@alice` for comparison, but the assignment call passes `originalUser`, producing an API path containing `@alice` rather than the account handle. Pass `normalizedUser` (or a display-preserving value with the prefix removed) and add an `@`-prefixed regression test.
- excerpt: |
    wantMap[github.NormLogin(user)] = user
    ...
    for normalizedUser := range toAdd {
        originalUser := wantMap[normalizedUser]
        ...
        if err := client.AssignOrganizationRoleToUser(orgName, originalUser, roleID); err != nil {

### [should-fix] Avoid role API access for an org with no roles

- where: `cmd/peribolos/main.go:1809-1817`
- concern: With global `--fix-org-roles`, an org that declares no roles still calls `ListOrganizationRoles`. An unavailable or unauthorized role endpoint can therefore fail an otherwise no-op pass. Return when `len(orgConfig.Roles) == 0` before making the API call.
- excerpt: |
    roles, err := client.ListOrganizationRoles(orgName)
    if err != nil {
        return fmt.Errorf("failed to list organization roles: %w", err)
    }

    if len(roles) == 0 && len(orgConfig.Roles) == 0 {
        return nil
    }

### [should-fix] Do not infer authoritative membership from nonempty lists

- where: `pkg/config/org/org.go:190-205`
- concern: Any nonempty `members` or `admins` list is treated as the complete organization membership, even when membership reconciliation is not enabled or the config deliberately describes only a subset. That rejects role users who already belong to the organization. Gate this validation on an authoritative membership mode, or validate against GitHub's actual membership.
- excerpt: |
    validateUsers := len(c.Members) > 0 || len(c.Admins) > 0
    ...
    if !validateUsers {
        continue
    }
    for _, user := range role.Users {
        if !availableUsers[github.NormLogin(user)] {

### [should-fix] Add REST endpoint-contract tests for role operations

- where: `pkg/github/client.go:5522-5640`, `pkg/github/client_test.go`
- concern: The seven new role client methods have no endpoint-specific HTTP tests. Reconciler tests using a local fake and the generic paginated-error test cannot catch an incorrect method, path, expected status, pagination, or `{total_count, roles}` wrapper decode. Add `httptest` coverage for the list and assignment endpoints.
- excerpt: |
    path := fmt.Sprintf("/orgs/%s/organization-roles", org)
    ...
    return &orgRolesResponse{}
    ...
    path: fmt.Sprintf("/orgs/%s/organization-roles/users/%s/%d", org, user, roleID),

### [should-fix] Make the standard fake model role state

- where: `pkg/github/fakegithub/fakegithub.go:1459-1487`
- concern: The exported fake implements every new role method as a nil/no-op stub. Consumers cannot model current assignments or assert mutations, which weakens integration testing for the new public client surface. Store role and assignment state, or keep clients that do not need roles on a narrower interface.
- excerpt: |
    func (f *FakeClient) ListOrganizationRoles(org string) ([]github.OrganizationRole, error) {
        return nil, nil
    }

    func (f *FakeClient) AssignOrganizationRoleToUser(org, user string, roleID int) error {
        return nil
    }

## Checked

- Reviewed merged commit `a0ba73401af9961d586043b59d15881b92d222fa` against parent `7626c76a396f367698b87f0650c8c5df14ae6c4e`.
- Confirmed that the new `roles` field is optional and `--fix-org-roles` defaults off, preserving existing configuration behavior.
- Confirmed the final change adds dump failure classification, secret-team log protection, direct-versus-indirect assignment handling, paginated status-error coverage, and public documentation.
- Confirmed dump now adds one role-list request and two paginated assignment-list requests per organization role; this may affect API use and latency in large organizations.

## Open questions

- Are organization-role APIs available on every supported GitHub Enterprise version, and are the required token permissions documented beside `--fix-org-roles`?
- Are there downstream implementations of the exported `github.Client` interface that need migration guidance after it gained organization-role methods?
