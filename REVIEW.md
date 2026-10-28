---
pr: kubernetes-sigs/prow#766
title: "handle github teams lookup correctly with github apps"
head_sha: b6a76dd31dbbcd5c7292052f4400d138820c2bbc
base: main
reviewed_at: 2026-07-26T22:55:58Z
verdict: request-changes
refresh_log:
  - from: b6a76dd31dbbcd5c7292052f4400d138820c2bbc
    to: b6a76dd31dbbcd5c7292052f4400d138820c2bbc
    summary: "No code changes. Incorporated new discussion: matthyx suggested the ListTeamMembersbyID comment-typo fix inline; upodroid and matthyx opened a broader design thread on deprecating/reworking GitHubTeamIDs vs GitHubTeamSlugs."
---

Fixes GitHub Apps auth failure for team-member-by-ID lookups by changing the API path from `/teams/{id}/members` (no org in URL, so `extractOrgFromContext` returned empty) to `/orgs/{org}/team/{id}/members`. Renames `ListTeamMembers` to `ListTeamMembersByID`, removes deprecation warning, updates all callers/interfaces/fakes/tests. Drive-by doc cleanup in peribolos.md.

Since previous review:
- No new commits (head still `b6a76dd3`).
- matthyx posted an inline suggestion on `pkg/github/fakegithub/fakegithub.go` fixing the `ListTeamMembersbyID` comment typo (matches the existing "Comment typo and stale references in fakegithub" nit below) — not yet pushed by the author.
- upodroid and matthyx opened a design discussion on the issue thread about the empty-org problem: whether to deprecate/remove `GitHubTeamIDs` in favor of `GitHubTeamSlugs`, or keep `GitHubTeamIDs` working via this PR's org-scoped path with a hard error on empty org, plus a deprecation warning and doc update — this bears directly on the unresolved "Empty org produces malformed URL" blocking finding below.

## Findings

### [blocking] Undocumented API path
- where: `pkg/github/client.go:3909`
- concern: `/orgs/{org}/team/{id}/members` does not match any documented GitHub REST API endpoint. Documented patterns are `/orgs/{org}/teams/{slug}/members` (plural, slug), `/organizations/{org_id}/team/{id}/members` (numeric org ID under `/organizations/`), and legacy `/teams/{id}/members`. This path mixes `/orgs/{org_name}` with singular `/team/{numeric_id}`. Author validated with `gh api` but GitHub could break this undocumented route without notice.
- excerpt: |
    path := fmt.Sprintf("/orgs/%s/team/%d/members", org, id)

### [blocking] Empty org produces malformed URL
- where: `pkg/github/client.go:3909`
- concern: Callers like `RerunAuthConfig.IsAuthorized` (types.go:314) can pass empty `org` in periodic job contexts (visible in the original stack trace as `{0x0, 0x0}`). New path produces `/orgs//team/{id}/members`. Raised by reviewers alvaroaleman and matthyx in PR comments, remains unaddressed.
- update (2026-07-26): matthyx proposed a concrete path forward on the issue thread — keep `GitHubTeamIDs` working via this PR's org-scoped route with a hard error when org is empty, log a deprecation warning, document `github_team_slugs` as the replacement, and drop the field in a future major. upodroid is weighing this against outright removing `github_team_ids`. Still unresolved/unimplemented.
- excerpt: |
    path := fmt.Sprintf("/orgs/%s/team/%d/members", org, id)

### [should-fix] org omitted from duration log
- where: `pkg/github/client.go:3903`
- concern: `ListTeamMembersByID` logs `c.log("ListTeamMembersByID", id, role)` without `org`. Sibling `ListTeamMembersBySlug` logs `c.log("ListTeamMembersBySlug", org, teamSlug, role)`. Since org is now in the API path, omitting it hinders debugging auth failures.
- excerpt: |
    durationLogger := c.log("ListTeamMembersByID", id, role)

### [nit] Comment typo and stale references in fakegithub
- where: `pkg/github/fakegithub/fakegithub.go:741`
- concern: Comment reads `ListTeamMembersbyID` (lowercase b), function is `ListTeamMembersByID` (capital B). Also lines 723 and 759 still reference old `ListTeamMembers` name.
- update (2026-07-26): matthyx posted an inline suggested-change fixing this exact typo; not yet applied by the author.
- excerpt: |
    // ListTeamMembersbyID return a fake team with a single "sig-lead" GitHub teammember

### [nit] Misleading comment about team_id/team_slug
- where: `pkg/github/client.go:3901`
- concern: Comment says "the API accepts both team_id and team_slug" which reads as if this method accepts either form. It only takes `int` id; `ListTeamMembersBySlug` handles slugs.
- excerpt: |
    // Note: the API accepts both team_id and team_slug, but not fully documented

### [nit] Stale doc link
- where: `pkg/github/client.go:3899`
- concern: Points to old `developer.github.com` domain. Sibling `ListTeamMembersBySlug` uses current `docs.github.com`.
- excerpt: |
    // https://developer.github.com/v3/teams/members/#list-team-members

### [nit] Inconsistent Accept headers
- where: `pkg/github/client.go:3917`
- concern: `ListTeamMembersByID` uses `application/vnd.github+json` while `ListTeamMembersBySlug` uses `application/vnd.github.v3+json`. Functionally equivalent but inconsistent.

### [question] Interface rename as breaking change
- where: `pkg/github/client.go:214`
- concern: `TeamClient` interface rename from `ListTeamMembers` to `ListTeamMembersByID` is a compile-time break for out-of-tree implementations. Should be in release notes. Consider whether a backward-compatible wrapper is warranted.

## Checked
- All `TeamClient` interface implementations updated (real, fake, plugin-local interfaces)
- Test updated to match new path `/orgs/orgName/team/1/members`
- `TeamHasMember` delegates to renamed method correctly
- No remaining compile-breaking references to old `ListTeamMembers` name
- Peribolos doc changes are formatting-only plus log output update
- Error message repo reference update (test-infra to kubernetes-sigs/prow) is correct
- Deprecation warning removal is appropriate — method is rehabilitated, not sunset

## Open questions
- Can you confirm `/orgs/{org_name}/team/{numeric_id}/members` works against production GitHub API? How was it verified?
- How should the empty-org case be handled — return an error, fall back to legacy `/teams/{id}/members`, or require callers to always supply org? (matthyx's proposal: hard error on empty org + deprecation warning + future removal — does upodroid want to go further and drop `GitHubTeamIDs` outright instead?)
- Duration logger at `client.go:3903` omits `org` — intentional or should it match `ListTeamMembersBySlug`?
