---
pr: kubernetes-sigs/prow#1068
title: "cherrypicker: allow repo collaborators to request cherry-picks via --allow-collaborators"
head_sha: c4a5a0dbb09394c98bc1bc8e0de7b546880a9f43
base: main
reviewed_at: 2026-11-13T17:32:11Z
verdict: approve
---

# Review

## Verdict
Approve with suggestions: the change is correct, default-off, and well-tested, with no blocking issues from code quality, maintainability, or deployment risk.

Authorization is centralized in a documented `isAuthorized` helper and applied consistently to the open-PR comment, merged-PR comment, label-added, and PR-closed reprocessing paths. With the flag off, behavior and API calls are unchanged; with it on, non-member collaborators are accepted and the rejection text adapts. The remaining work is non-blocking: document the flag's scope, avoid silently dropping a queued request on a transient lookup error, and keep the two authorization implementations from drifting. The design questions the author raises (read-only signal, instance-wide scope) are recorded below as questions rather than merge blockers.

## What this PR does
- Adds an opt-in `--allow-collaborators` flag (default `false`) that authorizes repo collaborators in addition to org members; `--allow-all` still short-circuits first.
- Adds `IsCollaborator` to the plugin-local `githubClient` interface and centralizes the member-or-collaborator decision in `isAuthorized`, applied to every inbound authorization site.
- In the PR-closed batch path, keeps the bulk `ListOrgMembers` fetch and adds a memoized per-user `IsCollaborator` fallback that fails closed without aborting the batch.
- Adapts the rejection comment to say "org members or repo collaborators" when the flag is enabled, and updates the plugin help text.
- Adds table-driven tests covering authorization, API-call short-circuiting, error injection, batch semantics, and label-requester fallback.

## Findings

### [should-fix] Authorization policy is implemented twice
- where: `cmd/external-plugins/cherrypicker/server.go:474-516`, `cmd/external-plugins/cherrypicker/server.go:600-619`
- concern: `handlePullRequestClosed` reimplements the org-member-then-collaborator rule inline instead of reusing `isAuthorized`. A future policy change (e.g. `HasPermission`, write-level gating, caching) must be made in both places or the paths silently diverge. Extract a per-event authorizer that supports a bulk member preload plus memoized lookups, or add a cross-reference comment. Flagged by Code Quality and Maintainability.
- excerpt: |
    authorized = func(user string) bool {
        if user == "" {
            return false
        }
        if memberLogins.Has(user) {
            return true
        }
        if !s.allowCollaborators {
            return false
        }

### [should-fix] A transient collaborator lookup silently drops a queued cherry-pick
- where: `cmd/external-plugins/cherrypicker/server.go:503-513`
- concern: A failed `IsCollaborator` call is cached as `false` and the requester is deleted from the batch with only a `Warn` log and no user-visible feedback. A transient GitHub error at merge time therefore loses a valid request. Distinguish an error from "not a collaborator" (do not memoize the failure), and either retry/post a "could not verify authorization" comment or add a metric. Flagged by all three reviewers.
- excerpt: |
    ok, err := s.ghc.IsCollaborator(org, repo, user)
    if err != nil {
        log.WithError(err).WithField("user", user).Warn("Failed to check collaborator status, treating as unauthorized.")
        ok = false
    }
    collaborators[user] = ok

### [should-fix] Document the flag's scope and semantics
- where: `cmd/external-plugins/cherrypicker/main.go:75`
- concern: The flag is instance-wide, explicitly includes read-only collaborators, and is ignored with `--allow-all`. Enabling it for one repo broadens every repo the instance serves, and a read-only collaborator can make the bot push a branch and open a PR. Add this to `site/content/en/docs/components/external-plugins/cherrypicker.md` alongside the token-scope notes; a misconfigured token surfaces a 404 as "not a collaborator" rather than an error. Flagged by Deployment Risk.
- excerpt: |
    fs.BoolVar(&o.allowCollaborators, "allow-collaborators", false, "Allow repo collaborators (any access level, including read-only) in addition to org members to use automated cherrypicks. Ignored when --allow-all is set.")

### [nit] Label-added path authorizes the PR author even when the labeler is the requester
- where: `cmd/external-plugins/cherrypicker/server.go:350-361`
- concern: The loop eagerly calls `isAuthorized` for both `prAuthor` and `labeler.Login`, but `labelRequester` never consults the author when the labeler is an authorized non-bot. With `--allow-collaborators` this adds a lookup, and a transient error for the irrelevant author aborts the whole event. A lazy, memoizing predicate (as in the closed-PR path) would avoid both.
- excerpt: |
    for _, user := range []string{prAuthor, labeler.Login} {
        if _, done := authorized[user]; done || user == "" {
            continue
        }
        ok, err := s.isAuthorized(org, repo, user)
        if err != nil {
            return log, err
        }
        authorized[user] = ok
    }

### [nit] Minor readability issues in the rejection/template plumbing
- where: `cmd/external-plugins/cherrypicker/server.go:48`, `cmd/external-plugins/cherrypicker/server.go:582`
- concern: `notAuthorizedMessageTemplate` now has five positional verbs with `org` repeated three times, guarded only by an "in order" comment; indexed verbs (`%[1]s`) would make edits safer. In `labelRequester`, the `isAuthorized` parameter shadows the `(*Server).isAuthorized` method and invites misreads.
- excerpt: |
    var notAuthorizedMessageTemplate = "only [%s](https://github.com/orgs/%s/people) %s may request cherry picks. ..."
    ...
    func (s *Server) labelRequester(prAuthor string, labeler github.User, isAuthorized func(string) bool) (string, bool) {

### [nit] Call-count assertions pin the lookup strategy
- where: `cmd/external-plugins/cherrypicker/server_test.go:1487`
- concern: Exact `IsMember`/`IsCollaborator` call counts assert the current implementation rather than behavior and will break on a benign batching or `ListCollaborators` refactor. An upper bound (at most one lookup per user) preserves the intent; keep zero-call assertions only where they encode a real constraint such as `--allow-all` needing no GitHub access.
- excerpt: |
    if ghc.memberCalls[tc.user] != tc.wantMemberCalls {
        t.Errorf("IsMember(%q) called %d times, want %d", tc.user, ghc.memberCalls[tc.user], tc.wantMemberCalls)
    }

### [question] Signal strength: should read-only collaborators qualify?
- where: `cmd/external-plugins/cherrypicker/server.go:600-618`, `cmd/external-plugins/cherrypicker/main.go:75`
- concern: Bare `IsCollaborator` includes read-only outside collaborators. That is fine for backports of already-merged PRs and matches lgtm, but if the intent is "trusted contributors" rather than "any reader", `HasPermission` (push/admin) is the stricter signal. This is the author's open question 3 and remains a maintainer policy call.
- excerpt: |
    collaborator, err := s.ghc.IsCollaborator(org, repo, user)
    if err != nil {
        return false, fmt.Errorf("failed to check collaborator status for %q: %w", user, err)
    }
    return collaborator, nil

### [question] Instance-wide flag vs per-repo configuration
- where: `cmd/external-plugins/cherrypicker/main.go:75`
- concern: The rationale is repo-scoped, but the flag applies to every repo the instance serves. Is the server-wide flag the intended long-term shape, or is a follow-up for per-repo config (like lgtm's `Owners.SkipCollaborators`, which requires wiring `plugins.Configuration`) expected? This is the author's open question 2.
- excerpt: |
    fs.BoolVar(&o.allowCollaborators, "allow-collaborators", false, "Allow repo collaborators ... to use automated cherrypicks. Ignored when --allow-all is set.")

### [question] Should a membership-check error fall through to the collaborator check?
- where: `cmd/external-plugins/cherrypicker/server.go:604-613`
- concern: `isAuthorized` only consults `IsCollaborator` when `IsMember` returns `false`, not when it errors (the real client returns an error on a 302, e.g. private org membership). If that path fires, enabling the flag does not rescue those users. PLAUSIBLE: depends on how often the 302 path occurs in practice; worth confirming the intent.
- excerpt: |
    member, err := s.ghc.IsMember(org, user)
    if err != nil {
        return false, fmt.Errorf("failed to check org membership for %q: %w", user, err)
    }
    if member {
        return true, nil
    }

## Checked
- Flag defaults to `false`: `IsCollaborator` is never reached and behavior/API calls match the previous code (tested).
- `--allow-all` still short-circuits before any GitHub call (tested by the `allow-all` cases).
- All inbound authorization sites are covered: open-PR comment, merged-PR comment, label-added, and PR-closed reprocessing. `handlePullRequest` only dispatches `labeled` and `closed`, so there are no other entry points.
- The closed-PR path keeps the bulk `ListOrgMembers` fetch, memoizes collaborator lookups per event, and a failed lookup does not abort the rest of the batch (tested).
- `githubClient.IsCollaborator(org, repo, user)` matches `pkg/github.Client` and lgtm's usage; the interface is plugin-local, so no external implementors break.
- No config-schema, manifest, API, or serialization changes; no breaking changes, and rollback is removing the flag.
- `gofmt -l`, `go vet`, `go build`, and `go test ./cmd/external-plugins/cherrypicker/...` are clean at the PR head; the Code Quality reviewer also ran `-race`.

## Open questions
- Should the policy be bare `IsCollaborator` (read-only included), or gate on write/admin via `HasPermission`?
- Is the instance-wide flag the intended long-term shape, or should per-repo config follow?
- Is the `IsMember`-error short-circuit (no collaborator fallback) acceptable, given private org membership can return 302?
