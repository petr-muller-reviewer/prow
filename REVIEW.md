---
pr: kubernetes-sigs/prow#802
title: "peribolos: add --ignore-repos flag for team repo reconciliation"
head_sha: fccd7841c84dca2990f854c949f3cbb66b25d202
base: main
reviewed_at: 2026-07-22T14:34:09Z
verdict: request-changes
---

## Summary

Implements #737: `--ignore-repos` flag for peribolos, excluding named repos
(case-insensitive, comma-separated + repeatable) from team-repo permission
reconciliation, propagated to child teams. That part is correct, well-tested,
and matches the linked issue.

The PR also adds `--ignore-private-repos` and `--ignore-internal-repos` (new
`github.Repo.Visibility` field), undisclosed in the issue/PR description, and
these two flags have a confirmed correctness bug: they only protect repos the
team already has, not new grants.

Reviewed via `/review` (single-pass) and `/muller-maintainer-review`
(3 parallel specialist reviewers + advisor synthesis) — the private/internal
"add" gap and the undisclosed scope were independently identified by every
reviewer across both passes.

## Findings

### [blocking] --ignore-private-repos/--ignore-internal-repos don't prevent new grants
- where: `cmd/peribolos/main.go:1498-1523`
- concern: `privateRepos`/`internalRepos` sets are populated only from repos `ListTeamReposBySlug` currently returns for the team (the "have" side). The "want" loop checks `privateRepos.Has(repoName) || internalRepos.Has(repoName)` to skip updates/removals, but a private/internal repo present in `team.Repos` (want) that the team does **not yet** have is never added to those sets — so peribolos will still **add** it despite the flag. Example: `org.yaml` declares `team-x` should get `Write` on private repo `secret-repo`, `team-x` doesn't have it yet — `--ignore-private-repos` will not stop the grant. Contradicts the PR description's "skip ignored repos on both sides of the diff" claim for these two flags (true only for `--ignore-repos`, which is name-based and doesn't depend on current association). Independently flagged by all 3 specialist reviewers (code-quality, maintainability, deployment-risk) in the maintainer-review pass. No test exercises this "want-only" case.
- excerpt: |
    for wantRepo, wantPermission := range want {
        repoName := strings.ToLower(wantRepo)
        if ignoreRepos.Has(repoName) || privateRepos.Has(repoName) || internalRepos.Has(repoName) {
            continue
        }

### [blocking] undisclosed scope expansion beyond linked issue
- where: `cmd/peribolos/main.go:65-67,98-99`, `pkg/github/types.go:335`
- concern: Issue #737 and the PR body describe only `--ignore-repos`. The diff also adds `--ignore-private-repos`, `--ignore-internal-repos`, and a new `Visibility` field on the public `github.Repo` type, none of which are disclosed. All 3 maintainer-review specialists flagged this independently as a reviewability/governance concern. Should be split into a separate PR or the description updated to justify it, especially given finding #1 above.
- excerpt: |
    ignoreRepos           flagutil.Strings
    ignorePrivateRepos    bool
    ignoreInternalRepos   bool

### [should-fix] no typed constant for "internal" visibility literal
- where: `cmd/peribolos/main.go:1510`
- concern: `repo.Visibility == "internal"` is a bare string comparison with no constant/enum defined in `pkg/github`, unlike other GitHub API enums in the package. A typo or GitHub API change would silently break the feature with no compiler assistance.
- excerpt: |
    if ignoreInternalRepos && repo.Visibility == "internal" {

### [should-fix] no logging when repos are skipped by ignore flags
- where: `cmd/peribolos/main.go:1503-1516`
- concern: Ignored/private/internal repos are silently excluded from reconciliation with no `logrus` line, unlike other reconciliation actions in this file. Reduces operator visibility when auditing dry-run (`--confirm=false`) output to understand why a repo present in config isn't being managed.

### [nit] ignoredRepos manual comma-splitting undocumented
- where: `cmd/peribolos/main.go:163-174`
- concern: `flagutil.Strings.Set` does not split on commas itself (confirmed in `pkg/flagutil/strings.go`) — it appends the raw arg once per `--flag=value` occurrence. `ignoredRepos` correctly splits/trims/lowercases itself, but duplicates logic that could largely reuse the existing (non-splitting) `Strings.StringSet()` helper. A short comment explaining the manual split would help future readers.
- excerpt: |
    func ignoredRepos(values []string) sets.Set[string] {
        repos := sets.Set[string]{}
        for _, value := range values {
            for _, repo := range strings.Split(value, ",") {

### [nit] configureTeamRepos growing to 8 parameters
- where: `cmd/peribolos/main.go:1490`
- concern: 3 new ignore-related params threaded through unchanged on every recursive child-team call. Consider bundling `ignoreRepos`/`ignorePrivateRepos`/`ignoreInternalRepos` into a small struct if more ignore knobs are added later.

### [nit] new Visibility field lacks doc comment
- where: `pkg/github/types.go:335`
- concern: `Visibility string` has three-way semantics (`public`/`private`/`internal`) but no comment. Also worth noting it may be empty on older GitHub Enterprise Server versions that predate this API field — in which case `--ignore-internal-repos` fails safe (never matches) but silently, which could confuse operators expecting it to work.
- excerpt: |
    Private       bool   `json:"private"`
    Visibility    string `json:"visibility"`

### [nit] cmp.Diff argument order reversed in new test
- where: `cmd/peribolos/main_test.go:196`
- concern: `cmp.Diff(actual, expected)` reverses the conventional `cmp.Diff(want, got)` order used elsewhere; harmless here (sets compare symmetrically) but inconsistent with typical diff framing.

### [question] have map keying vs. case-insensitive ignore matching
- where: `cmd/peribolos/main.go:1516`
- concern: `have[repo.Name]` is still keyed by original case while the new ignore matching is case-insensitive. Pre-existing exact-case want/have matching predates this PR — intentional, or worth revisiting given the new case-insensitive semantics layered on top?

## Checked
- `go build ./cmd/peribolos/... ./pkg/github/...` succeeds
- `go test ./cmd/peribolos/...` passes, including new tests for ignoredRepos, ignore-repos/private/internal in configureTeamRepos, and child-team propagation
- `--ignore-repos` matching is case-insensitive consistently in both the have-building and want-diffing loops
- flag naming follows existing `--ignore-*` convention (`--ignore-secret-teams`, `--ignore-enterprise-teams`)
- no markdown docs reference these flags, so none needed updating
- new `Visibility` field is additive-only on `github.Repo`; grepped other consumers (pluginhelp/hook, plugins/blunderbuss, plugins/lgtm, plugins/project, github/fakegithub, branchprotector, checkconfig) — none reference `.Visibility`, no deserialization risk for existing payloads
- all new flags default to no-op values (empty `flagutil.Strings`, `false` booleans) — no behavior change for operators who don't pass them; safe rollback, no config migration needed

## Open questions
- Why were `--ignore-private-repos`/`--ignore-internal-repos` added when issue #737 only requested `--ignore-repos`? Would you be open to splitting them into a follow-up PR?
- Given the "want-only" private/internal repo gap, do you want to fix it (fetch/verify visibility before adding), or explicitly document it as a known limitation with a test that pins the current behavior?
