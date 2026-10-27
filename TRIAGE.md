---
issue: kubernetes-sigs/prow#1039
title: "Support opt-in GitHub teams in OWNERS and OWNERS_ALIASES"
state: open
labels: kind/feature
main_sha: 0d9737ca26be70c30ad4d017f38965d0f8ef0943
triaged_at: 2026-10-27T17:22:29Z
verdict: needs-discussion
legitimacy: LEGITIMATE
effort: 3
recommended_labels: [kind/feature, priority/important-longterm]
advice:
  advised_at: 2026-10-27T19:19:30Z
  based_on_triaged_at: 2026-10-27T17:22:29Z
---

# Triage

## Verdict

**needs-discussion** — legitimate, well-specified opt-in feature whose design must be agreed before implementation.

The request is in scope: `pkg/repoowners` is the single OWNERS entry point for approve, lgtm, verify-owners, blunderbuss, rifle, override, reward-owners and owners-label (`pkg/repoowners/repoowners.go:243-261`), and GitHub team listing already exists in the client (`pkg/github/client.go:4239-4267`). It is not implementable as-is because three constraints need maintainer decisions: login normalization erases the `@` marker, `RepoOwners` is cached per commit SHA while team membership is not commit-bound, and expansion must happen before collaborator filtering. Effort is level 3 (large): the change is contained but alters approval authority for every deployment that enables it.

## What the issue reports

- Allow `@org/team-slug` entries in `OWNERS` and `OWNERS_ALIASES`, including inside alias definitions, expanded to individual users.
- Gate it behind one global plugin-config option, `owners.enable_github_teams`, default off, with no per-repo or per-plugin override.
- Expand teams in every consumer of OWNERS data — approval, reviewer selection, ownership validation — without requiring contributors to mention teams themselves.
- Keep alias semantics: `required_reviewers` requests review from every team member rather than selecting one or requiring all of them to approve; resolve membership independently of commits via the existing GitHub client and ghproxy.
- Motivation is Knative's many repos across several orgs; the stated trade-off is that team membership changes are not recorded in a file's Git history, and the issue asks whether the opt-in approach is acceptable.

## Findings

### [cause] Login normalization erases the team marker

- detail: `NormLogins`/`NormLogin` strip one leading `@` and lowercase, so `@example/maintainers` becomes `example/maintainers` — identical to an alias or user spelled that way. Team detection must use the raw string and keep refs separate from logins until expansion.
- evidence: `pkg/github/types.go:166-168`, `pkg/repoowners/repoowners.go:726-732`

### [cause] RepoOwners is cached per commit SHA, team membership is not commit-bound

- detail: The parsed `*RepoOwners` is cached per `org/repo:base` and rebuilt only when the SHA changes. Resolving teams during parsing would freeze membership until a commit touches an OWNERS file, contradicting the issue's requirement. Resolution has to happen on each `LoadRepoOwnersSha` return, backed by its own cache; ghproxy only provides HTTP revalidation. `reward-owners` calls `LoadRepoOwnersSha` twice per event with `updateCache=false`.
- evidence: `pkg/repoowners/repoowners.go:344-426`, `pkg/repoowners/repoowners.go:308-343`, `pkg/plugins/reward-owners/reward-owners.go:122-128`

### [cause] Collaborator filtering drops team refs and non-collaborator members

- detail: `filterCollaborators` intersects approvers/reviewers/advisory approvers with `ListCollaborators` on every load. A `@org/team` ref is not a login and would be removed, so expansion must run first; even then, members who are not repo collaborators are dropped, so Knative-style peribolos teams may still need `skip_collaborators` or collaborator grants. `required_reviewers` is not collaborator-filtered.
- evidence: `pkg/repoowners/repoowners.go:784-808`, `pkg/repoowners/repoowners.go:308-343`

### [related-code] Alias parsing and single-level expansion

- where: `pkg/repoowners/repoowners.go:240-482`, `pkg/repoowners/repoowners.go:665-777`
- excerpt: |
    // ExpandAliases returns members of multiple aliases, duplicates are pruned
    func (a RepoAliases) ExpandAliases(logins sets.Set[string]) sets.Set[string] {
    	if a == nil {
    		return logins
    	}
    	// Make logins a copy of the original set to avoid modifying the original.
    	logins = logins.Union(nil)
    	for _, login := range sets.List(logins) {
    		if expanded, ok := a[github.NormLogin(login)]; ok {
    			logins.Delete(login)
    			logins = logins.Union(expanded)
    		}
    	}
    	return logins
    }
- detail: Expansion is one level only (nested aliases are not recursively expanded), and it runs in `applyConfigToPath` before any per-load post-processing.

### [related-code] Global Owners config and hook wiring

- where: `pkg/plugins/config.go:259-289`, `cmd/hook/main.go:207-225`
- excerpt: |
    	resolver := func(org, repo string) ownersconfig.Filenames {
    		return pluginAgent.Config().OwnersFilenames(org, repo)
    	}
    	ownersClient := repoowners.NewClient(gitClient, githubClient, mdYAMLEnabled, skipCollaborators, ownersDirDenylist, resolver)
- detail: `plugins.Owners` is deployment-global, matching the requested `enable_github_teams` shape; the existing resolver functions are the precedent for threading another option. Only hook constructs `repoowners.NewClient` in-tree, but the constructor is exported, so its signature should stay compatible for out-of-tree consumers.

### [related-code] Team membership API already exists

- where: `pkg/github/client.go:4239-4267`
- excerpt: |
    	path := fmt.Sprintf("/orgs/%s/teams/%s/members", org, teamSlug)
    	var teamMembers []TeamMember
    	err := c.readPaginatedResultsWithValues(
    		path,
    		url.Values{
    			"per_page": []string{"100"},
    			"role":     []string{role},
    		},
- detail: `github.Client` embeds `TeamClient`, so repoowners' existing client can call this without a new dependency. The endpoint returns direct members only, so nested teams are out of scope unless recursed explicitly.

### [related-code] verify-owners re-parses changed OWNERS files and trust-checks owner strings

- where: `pkg/plugins/verify-owners/verify-owners.go:484-516`, `pkg/plugins/verify-owners/verify-owners.go:534-599`
- excerpt: |
    	// If OWNERS_ALIASES file was modified, check if newly added owners are trusted.
    	if ownerAliasesModified && !skipTrustedUserCheck {
    		allOwners := sets.List(repoAliases.ExpandAllAliases())
    		for _, owner := range allOwners {
    			nonTrustedUsers, err = checkIfTrustedUser(ghc, log, triggerConfig, owner, patch, filenames.OwnersAliases, org, repo, nonTrustedUsers, trustedUsers, repoAliases)
- detail: A team ref would reach `trigger.TrustedUser` → `IsMember`, which returns `false, nil` on 404 (`pkg/github/client.go:1474-1498`), so it is reported as an untrusted owner rather than crashing. Team refs need explicit skip-in-trust-check plus existence/permission validation.

### [related-code] required_reviewers fan-out

- where: `pkg/reviewer/reviewer.go:109`
- excerpt: |
    		requiredReviewers.Insert(rc.RequiredReviewers(file.Filename).UnsortedList()...)
- detail: Every entry becomes an individual review request, matching the issue's "include every member" requirement; a large team produces a large review-request batch.

### [related-code] Test fakes and coverage gap

- where: `pkg/repoowners/repoowners_test.go:117-145`, `pkg/github/fakegithub/fakegithub.go:823-849`
- detail: The repoowners fake implements only `ListCollaborators` and `GetRef`, so widening the internal client interface touches it. `fakegithub.ListTeamMembersBySlug` is hardcoded to `admins`/`leads` and returns `nil, nil` for unknown teams; it needs arbitrary org/team data. verify-owners has no team cases.
- evidence: `pkg/repoowners/repoowners_test.go:117-145`

### [related-code] Docs currently say teams are unsupported

- where: `site/content/en/docs/components/plugins/approve/approvers/_index.md:42`, `pkg/plugins/plugin-config-documented.yaml:499-522`
- excerpt: |
    Note that items in the OWNERS files can be GitHub usernames, or aliases defined in OWNERS_ALIASES files. An OWNERS_ALIASES file is another co-existed file that delivers a mechanism for defining groups. However, GitHub Team names are not supported. We do not use them because there is no audit log for changes to the GitHub Teams. This way we have an audit log.
- detail: The approvers doc must be rewritten, and the plugin config reference is generated by `hack/gen-prow-documented` and diffed by `make verify-codegen`, so adding the option requires regenerating it.

### [related-issue] Original tracker closed on the repo move

- ref: kubernetes/test-infra#10065
- relevance: Same request (teams in OWNERS); closed with an invitation to continue in kubernetes-sigs/prow, which this issue does.

### [related-issue] Team targets in /cc and /assign

- ref: kubernetes-sigs/prow#145
- relevance: Closed feature request for team targets in the assign/cc commands; different code path, but shows demand and the historical preference for per-member expansion over GitHub team review requests.

### [related-issue] Team mentions in /cc regressed

- ref: kubernetes-sigs/prow#204
- relevance: Context for how team handling has broken before; argues for explicit config gating, not a precedent for OWNERS behavior.

## Checked

- Searched `pkg/repoowners` for existing team handling: none; only peribolos and org-config code deals with teams.
- Searched open and closed issues for "OWNERS team" and "OWNERS_ALIASES": only this issue plus unrelated verify-owners bugs (#360, #514).
- Checked the issue timeline for cross-referenced issues/PRs (`--paginate`, `cross-referenced` events): none.
- Checked whether OWNERS consumers bypass the `RepoOwner` interface: only verify-owners re-parses changed files; all others use the shared client.
- Checked in-tree `repoowners.NewClient` callers: only `cmd/hook/main.go`, so the option can be threaded without touching other binaries.
- Checked `IsMember` and `trigger.TrustedUser` with a non-user string: `404` maps to `false, nil`, so a team ref is flagged untrusted rather than causing an error.
- Checked plugin config doc generation: `hack/gen-prow-documented` output is verified by `hack/make-rules/verify/codegen.sh`.
- Checked whether team review requests could replace expansion: they cannot, because approve matches comment authors and `required_reviewers` wants individuals.

## Next steps

- Maintainer reply on the proposal in the issue: config shape, fail-closed behavior, nested teams, and the collaborator-filter interaction. This is the blocking decision; no implementation should start before it.
- Apply `priority/important-longterm` and link kubernetes/test-infra#10065 for history.
- Recommended implementation shape if accepted: detect `@org/slug` on raw entries at parse time, keep refs distinct from logins, and expand in `LoadRepoOwnersSha` after the cache lookup and before `filterCollaborators`, using a Client-owned resolver with a short-TTL `org/team` cache over `ListTeamMembersBySlug(org, slug, RoleAll)`. Restrict refs to the repo's org, fail closed (empty set + error log) on unreadable teams, and add the option to `plugins.Owners` without breaking the exported constructor signature.
- Update `verify-owners` (skip team refs in the trust check; validate team existence/permissions via the existing `invalid-owners` path) and regenerate the plugin config reference; rewrite the approvers doc caveat and document the required org members-read permission, API budget without ghproxy, direct-members-only behavior, collaborator filtering, and loss of a Git-audited roster.
- Add tests in `pkg/repoowners` for ref detection, refs inside aliases, expansion ordering, TTL cache behavior, disabled-flag no-op, and unreadable teams; extend the fake GitHub team data; add verify-owners team cases.
- Ask the author to confirm bot permissions and whether Knative team members are repo collaborators (determines whether `skip_collaborators` is needed), and whether nested teams are in scope.
- Artifacts are local only; no issue comment, label, or state change has been made.

## Advice

Advice built on the triage of 2026-10-27T17:22:29Z. No change since then: issue is still OPEN, still labeled only `kind/feature`, no assignees or milestone, no new comments, and no cross-referenced or connected PRs. Nothing is in flight, so there is no PR to review and no close/link action available.

Note: the repo has no `needs-triage`/`needs-discussion` label; its triage labels are `triage/accepted`, `triage/needs-information`, `triage/duplicate`, `triage/unresolved`, `triage/not-reproducible`. Since the verdict is `needs-discussion`, none of them fits, so state stays in `TRIAGE.md`. (Canonical role labels are unmapped for this repo; `/setup-matt-pocock-skills` sets that up.)

### 1. Add tracking and area labels

Rationale: the request is a long-standing downstream need with no deployment breakage, and Prow sits under SIG Testing; neither label is currently applied.

```bash
gh issue edit 1039 --repo kubernetes-sigs/prow --add-label "sig/testing,priority/important-longterm"
```

### 2. Ask the area owners to decide the three design questions

Rationale: `pkg/repoowners` has no OWNERS file, so the root approvers are the decision makers (`OWNERS`: cblecker, petr-muller, smg247); cblecker and smg247 are also the most recent committers to the package. The issue explicitly asks whether the opt-in approach is acceptable, and findings [cause] 1-3 each require a maintainer answer before implementation.

```bash
gh issue comment 1039 --repo kubernetes-sigs/prow --body "> *This was generated by AI during triage.*

@cblecker @smg247 @petr-muller — flagging the root approvers since pkg/repoowners has no OWNERS file. Triage of this proposal found three decisions that need maintainer agreement before implementation:

1. Ref syntax: login normalization strips the @ (github.NormLogin), so @org/team must be detected on the raw string and kept separate from aliases. OK to special-case this in the parsing/expansion path?
2. Resolution point: RepoOwners is cached per commit SHA, but team membership is not commit-bound. Proposed: expand on each LoadRepoOwnersSha call, after the cache lookup and before filterCollaborators, with a short-TTL org/team cache. Agree with load-time expansion?
3. Collaborator filtering: the existing filter drops non-collaborator members, so peribolos-managed teams may still need skip_collaborators. Should team members be required to be repo collaborators?

Also assumed: nested teams are out of scope (the API returns direct members only) and an unreadable/unknown team fails closed (empty member set + error log). Are those acceptable defaults?"
```

### 3. Route to the documented discussion venue if the issue thread goes quiet

Rationale: `CONTRIBUTING.md` documents Slack `#sig-testing` (https://kubernetes.slack.com/messages/sig-testing) and the kubernetes-sig-testing mailing list (https://groups.google.com/forum/#!forum/kubernetes-sig-testing) as the contact channels; no `gh` command applies, this is a manual post.

## Open questions

- Should non-collaborator team members become approvers/reviewers when `skip_collaborators` is off, or is "team members must also be collaborators" acceptable? This decides whether Knative must enable `skip_collaborators`.
- Are nested teams in scope? `ListTeamMembersBySlug` returns direct members only.
- On an unreadable or unknown team, should Prow fail closed (empty set + log) with `verify-owners` attaching `invalid-owners`, or refuse to load OWNERS at all?
- Should `verify-owners` treat a team ref as trusted when the team exists in the repo's org, instead of checking members individually?
- Is the option strictly global, with no per-org/per-repo override, as proposed?
- Which bot permissions can deployments realistically grant (classic token `read:org`, GitHub App Members read)? This fixes the documented default and the failure mode.
