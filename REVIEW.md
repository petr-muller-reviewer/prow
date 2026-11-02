---
pr: kubernetes-sigs/prow#802
title: "peribolos: add --ignore-repos flag for team repo reconciliation"
head_sha: 28dd406ae653902cd35767d540f99ce26cec1686
base: main
reviewed_at: 2026-11-01T23:20:43Z
verdict: request-changes
gate:
  decision: do-not-merge
  gated_at: 2026-11-02T12:32:36Z
  gated_head_sha: 28dd406ae653902cd35767d540f99ce26cec1686
  reviewed_head_sha: 28dd406ae653902cd35767d540f99ce26cec1686
refresh_log:
  - old_sha: fccd7841c84dca2990f854c949f3cbb66b25d202
    new_sha: fccd7841c84dca2990f854c949f3cbb66b25d202
    at: 2026-10-20T15:21:11Z
    summary: No code changes; recorded lifecycle, Tide merge-conflict status, and ok-to-test activity while the PR remains open.
  - old_sha: fccd7841c84dca2990f854c949f3cbb66b25d202
    new_sha: 28dd406ae653902cd35767d540f99ce26cec1686
    at: 2026-11-01T23:20:43Z
    summary: Rebased onto main c06373a3e and force-pushed twice on 2026-11-01. Adopted upstream's typed RepoVisibility constant and go 1.27's SplitSeq, dropped the local Repo.Visibility field (upstream now provides it). PR hunks otherwise unchanged; want-only gap and undisclosed flags remain.
---

## Gate

**Decision: do-not-merge** — both blocking findings from the review still stand unchanged at `28dd406ae`, and one of them is an access-control over-grant in the two undeclared flags.

All findings were originally raised at `fccd7841` and re-verified against `28dd406ae` (the 2026-11-01 rebase touched three style lines only). The private/internal sets are still populated exclusively from the team's current repos, so a private or internal repo newly declared for a team is still granted despite `--ignore-private-repos`/`--ignore-internal-repos`; the PR body still documents only `--ignore-repos`. Issue discussion shows the extra flags were requested (`StarMiner99`, 2026-07-16), so they are not arbitrary, but the description never says so and the flags' failure mode is silent over-granting of repository access. The rest of the PR (`--ignore-repos` itself) is sound, opt-in, and well tested.

**Gating items**
- [blocking] want-only private/internal grant gap — `cmd/peribolos/main.go:1767-1795` (`REVIEW.md`). Current code still builds `privateRepos`/`internalRepos` only from `ListTeamReposBySlug` (the "have" side) and skips those names only in the "want" loop, so a private/internal repo present in `team.Repos` but not yet granted is still added. Tests cover only the have side (`cmd/peribolos/main_test.go:4195-4244`); no want-only case exists. Disposition: blocks merge.
- [blocking] undisclosed scope beyond issue #737 — `cmd/peribolos/main.go:66-68,97-100` (`REVIEW.md`). PR body still lists only `--ignore-repos`. Partially mitigated by public issue comments (`StarMiner99` asking for `--ignore-internal-repos` at 2026-07-16T11:59Z; author reporting `--ignore-private-repos` added at 11:30Z), but reviewers still cannot see the extra surface from the description. Disposition: blocks merge until documented or split out.
- [should-fix] no logging when repos are skipped — `cmd/peribolos/main.go:1767-1782` (`REVIEW.md`). Unchanged. Not merge-blocking on its own, but it is what makes the two flags above fail invisibly.

**Merge risk (Area 2)**
- API surface: none. The diff touches only `cmd/peribolos/` (+256/−25); the `pkg/github` change was dropped in the rebase, so no exported API, CRD, or config-schema change.
- CLI surface: three new opt-in flags with no-op defaults (`flagutil.Strings{}`, `false`), so existing peribolos invocations behave identically. Blast radius is limited to operators who pass them.
- Access-control risk when passed: `--ignore-private-repos`/`--ignore-internal-repos` can still grant team access to repos they are meant to exclude, the inverse of what an operator asking for containment expects.
- GHES caveat: `github.Repo.Visibility` can be empty on older GitHub Enterprise Server releases; `--ignore-internal-repos` then silently matches nothing and peribolos may remove team permissions on internal repos the operator asked it to leave alone.
- Docs: neither new flag is documented in `site/content/en/docs/components/cli-tools/peribolos.md`; the PR changes no docs.
- External review evidence: none beyond the local review — the PR has zero submitted GitHub reviews and no inline comments. No repo-specific compatibility skill applied; none is in scope for a CLI-only diff.

**What unblocks merge**: drop or split out `--ignore-private-repos`/`--ignore-internal-repos` and merge `--ignore-repos` alone; or fix want-side detection, add a want-only test, document all three flags in the PR body/docs, and log skipped repos.

## Verdict

Request changes — `--ignore-repos` is correct and well tested, but the two undeclared private/internal flags can still grant the access they exist to prevent.

Implements #737 correctly: `--ignore-repos` excludes named repos
(case-insensitive, comma-separated + repeatable) from team-repo permission
reconciliation, propagated to child teams, and matches the linked issue. The
same PR also adds `--ignore-private-repos` and `--ignore-internal-repos`,
undisclosed in the issue/PR description, and those two flags have a confirmed
correctness bug: they only protect repos the team already has, not new grants.

Reviewed via `/review` (single-pass) and `/muller-maintainer-review`
(3 parallel specialist reviewers + advisor synthesis) — the private/internal
"add" gap and the undisclosed scope were independently identified by every
reviewer across both passes.

## What this PR does

- Adds `--ignore-repos` to peribolos: comma-separated, repeatable, case-insensitive repo names excluded from team-repo reconciliation on both the add and remove sides, inherited by child teams (fixes #737).
- Adds `ignoredRepos()` to split/trim/lowercase the flag values into a `sets.Set[string]` and threads it through `configureTeamRepos` recursion.
- Adds two undeclared flags, `--ignore-private-repos` and `--ignore-internal-repos`, which build skip-sets from the repos the team already has and use them to avoid updates and removals.
- Adds ~236 lines of tests in `main_test.go` covering flag parsing, case-insensitive matching, top-level team behaviour, and child-team propagation.

Since previous review:
- Code changed (`fccd7841` → `28dd406ae`) but only by rebasing onto current `main` (`c06373a3e`, 2026-11-01). Two force-pushes at 09:01:12Z and 09:52:17Z; the PR's own hunks changed in exactly three lines.
- `strings.Split` → `strings.SplitSeq` in `ignoredRepos` (go 1.27 style, `main.go:172`), and `repo.Visibility == "internal"` → `github.RepoVisibilityInternal` (`main.go:1773`), using the typed `RepoVisibility` enum upstream now carries.
- The PR's one-line `Visibility` field on `pkg/github/types.go` was dropped: `main` already added `RepoVisibility` plus a documented `Repo.Visibility` field, so the public-API part of the undisclosed scope is gone; the two undisclosed flags remain.
- Bot activity only: `kubernetes-prow[bot]` posted the `APPROVALNOTIFIER` "NOT APPROVED" note at 2026-11-01T09:52:52Z (author self-approval does not count). No new reviews or inline comments. PR still open, still `lifecycle/stale`.

## Findings

### [blocking] --ignore-private-repos/--ignore-internal-repos don't prevent new grants
- where: `cmd/peribolos/main.go:1767-1795`
- concern: `privateRepos`/`internalRepos` sets are populated only from repos `ListTeamReposBySlug` currently returns for the team (the "have" side). The "want" loop checks `privateRepos.Has(repoName) || internalRepos.Has(repoName)` to skip updates/removals, but a private/internal repo present in `team.Repos` (want) that the team does **not yet** have is never added to those sets — so peribolos will still **add** it despite the flag. Example: `org.yaml` declares `team-x` should get `Write` on private repo `secret-repo`, `team-x` doesn't have it yet — `--ignore-private-repos` will not stop the grant. Contradicts the PR description's "skip ignored repos on both sides of the diff" claim for these two flags (true only for `--ignore-repos`, which is name-based and doesn't depend on current association). Independently flagged by all 3 specialist reviewers (code-quality, maintainability, deployment-risk) in the maintainer-review pass. No test exercises this "want-only" case. Unchanged by the rebase.
- excerpt: |
    for wantRepo, wantPermission := range want {
        repoName := strings.ToLower(wantRepo)
        if ignoreRepos.Has(repoName) || privateRepos.Has(repoName) || internalRepos.Has(repoName) {
            continue
        }

### [blocking] undisclosed scope expansion beyond linked issue
- where: `cmd/peribolos/main.go:66-68,97-100`
- concern: Issue #737 and the PR body describe only `--ignore-repos`. The diff still also adds `--ignore-private-repos` and `--ignore-internal-repos`, neither disclosed in the issue or the PR description (re-read 2026-11-01: body lists only `--ignore-repos`), and both carry finding #1 above. All 3 maintainer-review specialists flagged this independently as a reviewability/governance concern. Should be split into a separate PR or the description updated to justify it. Note the public-API half of this finding is now moot: the rebase dropped the PR's own `pkg/github/types.go` change because upstream landed `RepoVisibility`.
- excerpt: |
    ignoreRepos           flagutil.Strings
    ignorePrivateRepos    bool
    ignoreInternalRepos   bool

### [should-fix] no logging when repos are skipped by ignore flags
- where: `cmd/peribolos/main.go:1767-1782`
- concern: Ignored/private/internal repos are silently excluded from reconciliation with no `logrus` line, unlike other reconciliation actions in this file. Reduces operator visibility when auditing dry-run (`--confirm=false`) output to understand why a repo present in config isn't being managed.

### [nit] ignoredRepos manual comma-splitting undocumented
- where: `cmd/peribolos/main.go:169-180`
- concern: `flagutil.Strings.Set` does not split on commas itself (confirmed in `pkg/flagutil/strings.go`) — it appends the raw arg once per `--flag=value` occurrence. `ignoredRepos` correctly splits/trims/lowercases itself, but duplicates logic that could largely reuse the existing (non-splitting) `Strings.StringSet()` helper. A short comment explaining the manual split would help future readers.
- excerpt: |
    func ignoredRepos(values []string) sets.Set[string] {
        repos := sets.Set[string]{}
        for _, value := range values {
            for repo := range strings.SplitSeq(value, ",") {

### [nit] configureTeamRepos growing to 8 parameters
- where: `cmd/peribolos/main.go:1753`
- concern: 3 new ignore-related params threaded through unchanged on every recursive child-team call. Consider bundling `ignoreRepos`/`ignorePrivateRepos`/`ignoreInternalRepos` into a small struct if more ignore knobs are added later.

### [nit] cmp.Diff argument order reversed in new test
- where: `cmd/peribolos/main_test.go:201`
- concern: `cmp.Diff(actual, expected)` reverses the conventional `cmp.Diff(want, got)` order used elsewhere; harmless here (sets compare symmetrically) but inconsistent with typical diff framing.

### [question] have map keying vs. case-insensitive ignore matching
- where: `cmd/peribolos/main.go:1780-1789`
- concern: `have[repo.Name]` is still keyed by original case while the new ignore matching is case-insensitive. Pre-existing exact-case want/have matching predates this PR — intentional, or worth revisiting given the new case-insensitive semantics layered on top?

## Resolved

- **[should-fix] no typed constant for "internal" visibility literal** — fixed in `28dd406ae`: the check is now `repo.Visibility == github.RepoVisibilityInternal` (`cmd/peribolos/main.go:1773`), using the typed enum upstream added (`pkg/github/types.go:322-333`).
- **[nit] new Visibility field lacks doc comment** — obsolete: the rebase dropped the PR's `Visibility string` field entirely; upstream's `RepoVisibility` type, constants, and `Repo.Visibility RepoVisibility` field are all documented (`pkg/github/types.go:322-333,370`), including the `RepoVisibilityInternal` case.

## Checked
- `go build ./cmd/peribolos/...` succeeds at `28dd406ae` (Go 1.26.10 local, `go.mod` says 1.26.4)
- `go test ./cmd/peribolos/...` passes at `28dd406ae`
- the only code differences vs. the previously reviewed head are the 3 lines listed under "Since previous review" — the private/internal logic is byte-for-byte identical after rebase
- `--ignore-repos` matching is case-insensitive consistently in both the have-building and want-diffing loops
- flag naming follows existing `--ignore-*` convention (`--ignore-secret-teams`, `--ignore-enterprise-teams`)
- no markdown docs reference these flags at `28dd406ae` (`git grep` over `site/`), so none needed updating
- the rebase pulled in large unrelated `main` peribolos changes (`--fix-org-roles`, enterprise-team dump filtering, org-role diffing); the PR's hunks do not touch those paths and the package still builds/tests green, but the PR has not been re-examined against that new code beyond compilation
- all new flags default to no-op values (empty `flagutil.Strings`, `false` booleans) — no behavior change for operators who don't pass them; safe rollback, no config migration needed

## Open questions
- Why were `--ignore-private-repos`/`--ignore-internal-repos` added when issue #737 only requested `--ignore-repos`? Would you be open to splitting them into a follow-up PR?
- Given the "want-only" private/internal repo gap, do you want to fix it (fetch/verify visibility before adding), or explicitly document it as a known limitation with a test that pins the current behavior?
