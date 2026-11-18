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

## Followups

Post-merge followups for merged PR #555 (`f82256c4c4328dfc2ed2be653c64cf321765b04b`, merged 2026-10-03). Recorded 2026-11-18 from the unresolved findings above and inspection of the merged code. Accepted: 3; skipped: 4.

### Preflight org-role reads before mutable reconciliation

- category: `preflight`
- necessity: `must`
- mode: `deliver`
- where: `cmd/peribolos/main.go:1179`, `cmd/peribolos/main.go:1258`, `cmd/peribolos/main.go:1809-1836`
- why: This is the unresolved blocking finding. Role-name validation and every organization-role read still happen after metadata, membership, repository, collaborator, team, and team-membership reconciliation, so a missing role or unavailable role endpoint can abort the run after those changes were applied. An org with no configured roles also still calls `ListOrganizationRoles`, because the empty-config return follows the API call.

```text
In kubernetes-sigs/prow, following merged PR #555 — "`peribolos`: add org roles feature" (merge commit f82256c4c4328dfc2ed2be653c64cf321765b04b) — fix the organization-role preflight ordering in cmd/peribolos/main.go.

What and where: `configureOrg` currently validates role names only inside `configureOrgRoles` (around lines 1179, 1258, 1809-1836), after metadata, invitations, membership, repositories, collaborators, teams, and team membership have already mutated state. `configureOrgRoles` also calls `ListOrganizationRoles` before checking whether `orgConfig.Roles` is empty, so a no-role org still performs a role API request.

Task:
- When `--fix-org-roles` is set, perform all organization-role reads and validation before the first mutating subsystem in `configureOrg`.
- List organization roles and build the case-insensitive name-to-role map before `configureOrgMeta`; validate every configured role name against that map before any mutation.
- When `len(orgConfig.Roles) == 0`, skip all organization-role API calls entirely.
- Prefetch `ListTeamsWithRole` and `ListUsersWithRole` for every configured role before mutations, then thread those snapshots into `configureOrgRoles`, `configureRoleTeamAssignments`, and `configureRoleUserAssignments` so no role read can fail after a mutation has started. Prefetching must not require the team-name-to-slug map produced by `configureTeams`; that map is still needed only for the later mutation phase.
- Preserve existing role policy: only configured roles are managed, indirect assignments are left alone, ignored secret/enterprise teams are untouched, and invitees are skipped.

Acceptance criteria:
- With `--fix-org-roles`, no `ListOrganizationRoles`, `ListTeamsWithRole`, or `ListUsersWithRole` call occurs after the first mutating client call. Add a unit test using the existing fake client(s) that makes the second role's assignment read fail and asserts no team or user assignment/removal call was made.
- An org with `roles: {}` makes zero organization-role API calls when `--fix-org-roles` is set. Add a test.
- `go test ./cmd/peribolos ./pkg/github ./pkg/config/org` passes.

Scope guard: do not make writes transactional, change role-management semantics, or alter dump behavior.
```

### Probe org-role client endpoint contracts

- category: `tests`
- necessity: `should`
- mode: `probe`
- where: `pkg/github/client.go:5522-5640`, `pkg/github/client_test.go`
- why: This is the unresolved should-fix finding. The existing generic paginated-error test cannot catch a wrong method, path, expected status, pagination behavior, or `{total_count, roles}` wrapper decode in the seven new role methods.

```text
In kubernetes-sigs/prow, following merged PR #555 — "`peribolos`: add org roles feature" (merge commit f82256c4c4328dfc2ed2be653c64cf321765b04b) — write and run a throwaway probe of the new organization-role client methods in pkg/github/client.go (around lines 5522-5640), then delete the probe tests. The goal is to find real client bugs, not to add permanent tests.

Use an httptest server and the existing `getClient(ts.URL)`-style helper to exercise these behaviours:
1. `ListOrganizationRoles`: GET `/orgs/{org}/organization-roles`, decode the `{total_count, roles}` wrapper, and follow Link-header pagination across at least two pages.
2. `ListTeamsWithRole` and `ListUsersWithRole`: GET `/orgs/{org}/organization-roles/{roleID}/teams` and `/users`, including paginated array decoding.
3. `AssignOrganizationRoleToTeam` and `RemoveOrganizationRoleFromTeam`: PUT and DELETE `/orgs/{org}/organization-roles/teams/{slug}/{roleID}`, expecting 204.
4. `AssignOrganizationRoleToUser` and `RemoveOrganizationRoleFromUser`: PUT and DELETE `/orgs/{org}/organization-roles/users/{login}/{roleID}`, expecting 204.
5. Non-2xx responses: 403 and 404 from the list calls must satisfy `IsForbidden` and `IsNotFound` respectively; an unexpected 500 must not be classified as either.

For each failure, decide whether it is a real implementation bug (wrong method, path, status handling, wrapper decode, pagination, or error classification) or a wrong test expectation. If you find a real bug, apply the minimal fix and report the failing case. If you find none, leave the repository unchanged.

Acceptance criteria: every listed behaviour was exercised; each failure is classified with evidence; any real bug has a minimal fix and a reported reproduction; all probe tests are deleted before finishing.

Scope guard: do not add permanent tests, refactor the client, change role reconciliation, or fix style or unrelated issues.
```

### Gate role-user validation on authoritative membership

- category: `design`
- necessity: `should`
- mode: `deliver`
- where: `pkg/config/org/org.go:191-205`, `cmd/peribolos/main.go:1179`
- why: This is the unresolved should-fix finding. `ValidateRoles` treats any nonempty `members` or `admins` list as the complete organization membership, so a valid role user is rejected when the config lists only a subset or membership is managed elsewhere.

```text
In kubernetes-sigs/prow, following merged PR #555 — "`peribolos`: add org roles feature" (merge commit f82256c4c4328dfc2ed2be653c64cf321765b04b) — make organization-role user validation depend on authoritative membership instead of the mere presence of a nonempty list.

What and where: `pkg/config/org/org.go` `ValidateRoles` currently sets `validateUsers := len(c.Members) > 0 || len(c.Admins) > 0` and rejects role users who are not in that config list (around lines 191-205). It is called from `configureOrg` in `cmd/peribolos/main.go` (around line 1179).

Task:
- Pass an explicit authoritative-membership signal into `ValidateRoles` from `configureOrg`, true only when `--fix-org-members` is enabled and the config declares membership; alternatively, move the user-reference check to the caller where the options are known.
- Keep role-name collision and team-reference validation unchanged.
- Add a regression test with a nonempty member list that is not authoritative (for example `members: [alice]` and role user `bob`, with membership reconciliation off) and assert validation succeeds.
- Add a test that still rejects an undefined role user when membership reconciliation is on.
- Update all call sites and existing tests.

Acceptance criteria: `go test ./pkg/config/org ./cmd/peribolos` passes; a valid role user is not rejected when peribolos is not reconciling org membership; undefined users are still rejected when membership is authoritative.

Scope guard: do not change role assignment or reconciliation behavior, the team checks in `ValidateRoles`, or dump behavior. Do not add a membership API lookup unless you deliberately choose that alternative and document the extra request.
```
