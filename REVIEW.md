---
pr: kubernetes-sigs/prow#555
title: "`peribolos`: add org roles feature"
head_sha: f46e6a6c1e428b22dee10711af765936b50308b4
base: main
reviewed_at: 2026-10-02T08:52:46Z
verdict: request-changes
refresh_log:
  - old_sha: f46e6a6c1e428b22dee10711af765936b50308b4
    new_sha: f46e6a6c1e428b22dee10711af765936b50308b4
    at: 2026-10-02T08:52:46Z
    summary: Incorporated one review and four inline comments; no code changed.
---

# Review of kubernetes-sigs/prow#555

## Verdict

Request changes.

The code-quality and deployment reviewers independently found partial-application and incomplete-dump risks; code quality and maintainability both found mode-dependent validation and HTTP-test gaps. Role reconciliation is opt-in and preserves ignored and inherited grants, but enabling it also forces potentially destructive team reconciliation. These confirmed issues, plus incorrect `@`-prefixed user assignments, warrant changes before merge.

## What this PR does

- Adds per-organization role declarations for team and user assignments to peribolos configuration.
- Adds GitHub client methods for listing roles and managing direct assignments.
- Adds role assignments to configuration dumps and reconciliation behind `--fix-org-roles`.
- Adds validation and reconciliation tests, including ignored teams and inherited assignments.

Since previous review:

- No code changed; `cblecker` submitted a commented review with four inline comments.
- The reviewer prioritized distinguishing expected role-API unavailability from transient or malformed dump failures, and requested public role documentation.

## Findings

### [blocking] Provide a safe role-only path without team reconciliation

- where: `cmd/peribolos/main.go:155-156`
- concern: `--fix-org-roles` requires `--fix-teams`, and `configureOrg` then runs `configureTeams` before roles. Even a user-only role configuration can create or delete undeclared teams managed elsewhere, or fail at the removal cap. Allow role-only management without destructive team reconciliation, resolving team slugs read-only when needed.
- excerpt: |
    if o.fixOrgRoles && !o.fixTeams {
        return fmt.Errorf("--fix-org-roles requires --fix-teams")
    }

### [blocking] Validate remote role names and access before other mutations

- where: `cmd/peribolos/main.go:1804-1809`
- concern: `configureOrg` can update org metadata, members, repositories, and teams before this remote role check runs. A misspelled role or role-API permission failure therefore aborts after earlier changes have been applied. Preflight role existence and access before reconciliation starts.
- excerpt: |
    // Validate all configured roles exist in GitHub BEFORE any mutations
    for roleName := range orgConfig.Roles {
        if _, ok := githubRolesByName[strings.ToLower(roleName)]; !ok {
            return fmt.Errorf("role %q does not exist in organization %s - create the role in GitHub before assigning it", roleName, orgName)
        }
    }

### [blocking] Normalize role usernames before calling the GitHub API

- where: `cmd/peribolos/main.go:1965-1974`
- concern: `ValidateRoles` accepts `@alice` because `github.NormLogin` strips the prefix, but this call sends the original `@alice` as the API username. Send the normalized handle and test an `@`-prefixed role user.
- excerpt: |
    toAdd := wantSet.Difference(haveSet)
    for normalizedUser := range toAdd {
        originalUser := wantMap[normalizedUser]
        // Skip users who have pending org invitations - they must accept before we can assign roles
        if invitees.Has(normalizedUser) {
            logrus.Infof("Waiting for %s to accept org invitation before assigning role %s", originalUser, roleName)
            continue
        }
        if err := client.AssignOrganizationRoleToUser(orgName, originalUser, roleID); err != nil {

### [blocking] Make incomplete role dumps detectable

- where: `cmd/peribolos/main.go:493-513`
- concern: Every failure is treated as expected best-effort behavior, including timeouts, exhausted rate limits, decode errors, and persistent 5xx responses. `--dump` then exits successfully with role grants missing. Limit the best-effort path to the intentional 403/404 cases, return other errors, and include the organization in warnings; apply the same policy to per-role assignment listing.
- excerpt: |
    roles, err := client.ListOrganizationRoles(orgName)
    if err != nil {
        logrus.WithError(err).Warn("Failed to list organization roles; omitting roles from dump")
        roles = nil
    }

### [blocking] Validate role users according to membership-management mode

- where: `pkg/config/org/org.go:190-205`
- concern: When `--fix-org-members` is off, a config listing some admins or members need not list every existing org member. This validation nevertheless rejects a role user managed elsewhere whenever those lists are nonempty. Pass the membership-management mode into validation or check actual org membership.
- excerpt: |
    validateUsers := len(c.Members) > 0 || len(c.Admins) > 0
    var errors []string
    for roleName, role := range c.Roles {
        for _, teamName := range role.Teams {
            if !availableTeams[strings.ToLower(teamName)] {
                errors = append(errors, fmt.Sprintf("role %q references undefined team %q", roleName, teamName))
            }
        }
        if !validateUsers {
            continue
        }

### [blocking] Skip the role API when an org declares no roles

- where: `cmd/peribolos/main.go:1789-1795`
- concern: `--fix-org-roles` is global, but even an org with no `roles` calls `ListOrganizationRoles` before the empty-config check. A permission or availability error can fail that org despite there being no role work to do. Return before the API call when `orgConfig.Roles` is empty.
- excerpt: |
    roles, err := client.ListOrganizationRoles(orgName)
    if err != nil {
        return fmt.Errorf("failed to list organization roles: %w", err)
    }

    if len(roles) == 0 && len(orgConfig.Roles) == 0 {
        return nil
    }

### [blocking] Test the new role client methods at the HTTP boundary

- where: `pkg/github/client.go:5512-5529`
- concern: The seven new REST methods have no dedicated HTTP-level tests, while peribolos tests use fakes. Endpoint paths, verbs, response decoding, pagination, and status handling could regress unnoticed. Add focused HTTP-server tests for the role methods before relying on them to change access grants.
- excerpt: |
    func (c *client) ListOrganizationRoles(org string) ([]OrganizationRole, error) {
        c.log("ListOrganizationRoles", org)
        if c.fake {
            return nil, nil
        }

        path := fmt.Sprintf("/orgs/%s/organization-roles", org)
        var roles []OrganizationRole
        err := c.readPaginatedResults(

### [blocking] Document the operator contract for organization roles

- where: `site/content/en/docs/components/cli-tools/peribolos.md:21-76`
- concern: The public guide has no role example or explanation of the flag dependency, per-role opt-in management, inherited and ignored assignments, role-API permissions, or dump behavior. Add these rules alongside the existing YAML example before presenting this as a supported configuration surface.
- excerpt: |
    Peribolos allows the org settings, teams and memberships to be declared in a yaml file. GitHub is then updated to match the declared configuration.

### [should-fix] Make the shared fake's role behavior observable

- where: `pkg/github/fakegithub/fakegithub.go:1459-1474`
- concern: All seven new methods on the shared `FakeClient` silently return empty data or success without recording assignments, so consumers using the fake cannot observe role changes or inject role-list errors. Model role state and operations, or reject unsupported calls explicitly.
- excerpt: |
    func (f *FakeClient) ListOrganizationRoles(org string) ([]github.OrganizationRole, error) {
        return nil, nil
    }

    func (f *FakeClient) AssignOrganizationRoleToTeam(org, teamSlug string, roleID int) error {
        return nil
    }

### [nit] Centralize organization-role assignment states

- where: `pkg/github/types.go:1845`
- concern: The `"indirect"` string and the rule that `mixed` counts as direct are repeated in dump and reconciliation paths. Define a named assignment type with constants and a helper such as `IsDirect()` so the API contract and its interpretation stay in one place.
- excerpt: |
    type OrganizationRoleAssignment struct {
        ID         int    `json:"id"`
        Login      string `json:"login,omitempty"`
        Slug       string `json:"slug,omitempty"`
        Assignment string `json:"assignment,omitempty"` // "direct", "indirect", or "mixed"
    }

### [question] Account for out-of-tree `github.Client` implementations

- where: `pkg/github/client.go:287-295`
- concern: Embedding `OrganizationRolesClient` expands the exported `github.Client` interface by seven methods, which is source-breaking for any out-of-tree implementations or fakes. Do downstream consumers exist, and should this dependency be narrowed or accompanied by release guidance? Their presence is unverified in this worktree.
- excerpt: |
    type Client interface {
        PullRequestClient
        RepositoryClient
        CommitClient
        IssueClient
        CommentClient
        OrganizationClient
        OrganizationRolesClient
        TeamClient

## Checked

- All three maintainer perspectives examined the PR at `f46e6a6c1e428b22dee10711af765936b50308b4`; the local product tree matches that head apart from review artifacts.
- `go test ./cmd/peribolos ./pkg/config/org ./pkg/github` passed in the code-quality review.
- Existing configuration remains compatible: `roles` is optional and `--fix-org-roles` defaults off.
- Role reconciliation preserves intentionally ignored teams and indirect assignments; team renames update the slug used later.
- The role feature adds GitHub role-API permission needs and per-org/per-role API calls when enabled.
- New review activity was incorporated: `cblecker` submitted a COMMENTED review on 2026-10-02 at 00:10 UTC; the author also asked about the stale `needs-rebase` label on 2026-10-01.

## Open questions

- Are there out-of-tree `github.Client` implementations that need migration guidance for the expanded interface?
