---
pr: kubernetes-sigs/prow#641
title: "invalidcommitmsg: add separate handling for fixup!/amend! commits"
head_sha: a1e2956b2a958f452fce70895be497d1dae788dd
base: main
reviewed_at: 2026-07-27T00:26:50Z
verdict: request-changes
---

## Summary

Extends `invalidcommitmsg` plugin to detect `fixup!`/`amend!` commits, applying a new
`do-not-merge/fixup-commits` label + comment, independent of the existing issue-closing-keyword
check. Adds new `plugins.InvalidCommitMsg` config type (`repos`, `disable_fixup_check`) with
org/repo priority resolution (`InvalidCommitMsgFor`) and validation. The new check is
enabled-by-default (opt-out only). PR is CLOSED on GitHub; maintainer (petr-muller) asked to
split behavior-only change from new config surface into separate PRs, author agreed and opened
#656 (gated/disabled-by-default) for the behavior-only part. Multi-perspective review (code
quality, maintainability, deployment risk) converges on: this PR should not merge as-is, and the
already-agreed close/split is the right outcome. Advisor synthesis recommendation: CLOSE.

## Findings

### [blocking] validateRepoDupes emits misleading "welcome" plugin error for InvalidCommitMsg
- where: `pkg/plugins/config.go:1598-1600,1614-1621`
- concern: `Configuration.Validate()` calls the shared `validateRepoDupes(c.InvalidCommitMsg)`, but that helper hardcodes plugin-name text in its error strings (`"The repo %q is duplicated in the 'welcome' plugin configuration."`). A duplicate `repos` entry under `invalid_commit_msg` produces an error pointing at the wrong plugin (`welcome`), actively misleading anyone debugging a config validation failure. Untested for this new call site — no test exercises `validateRepoDupes` with duplicate `InvalidCommitMsg` repos.
- excerpt: |
    if err := validateRepoDupes(c.InvalidCommitMsg); err != nil {
        return err
    }
    // validateRepoDupes error text: "The repo %q is duplicated in the 'welcome' plugin configuration."

### [blocking] Fixup check ships enabled-by-default with no pre-rollout opt-out window
- where: `pkg/plugins/invalidcommitmsg/invalidcommitmsg.go:128-131` (checkFixup derivation), config default via `InvalidCommitMsgFor` zero-value
- concern: `do-not-merge/fixup-commits` uses the standard Prow/Tide merge-blocking label prefix. Because the check is enabled by default (only `disable_fixup_check: true` opts out), every repo already running this plugin starts auto-labeling and blocking in-flight PRs with fixup/amend commits the moment this ships, with no config action needed and no way for operators to pre-stage the opt-out before the behavior activates. Any team that tolerates fixup/amend commits mid-review (autosquash workflows, squash-at-merge) is affected immediately. This is the core reason the maintainer requested this be split and gated (see #656, which makes it opt-in/disabled-by-default).
- excerpt: |
    invalidCommitCfg := cfg.InvalidCommitMsgFor(org, repo)
    checkFixup := !invalidCommitCfg.DisableFixupCheck

### [should-fix] Behavior and config surface bundled in one PR
- where: whole PR (pkg/plugins/config.go + pkg/plugins/invalidcommitmsg/invalidcommitmsg.go)
- concern: the org/repo config-resolution machinery (`InvalidCommitMsg`, `InvalidCommitMsgFor`, `validateInvalidCommitMsg`) is independently reviewable/testable/bisectable from the fixup-detection logic. Maintainer asked for a split; author agreed and opened #656 for the behavior-only half. This PR as-is conflates both concerns.

### [should-fix] Label/comment sync logic duplicated instead of shared
- where: `pkg/plugins/invalidcommitmsg/invalidcommitmsg.go:177-183,213-227`
- concern: the fixup label add/remove/prune logic is a close copy of the existing invalid-commit-message logic, including pruning fixup comments in two separate branches (on label removal, and again unconditionally before re-adding a comment). Not new dead weight, but doubles the surface area of an already-hard-to-reason-about idiom (prune-by-substring-match) rather than factoring out a shared helper. A third similar check would likely triple it.
- excerpt: |
    if hasFixupCommitMsgLabel && len(fixupCommits) == 0 {
        ...
        cp.PruneComments(func(comment github.IssueComment) bool {
            return strings.Contains(comment.Body, fixupCommitMsgCommentPruneBody)
        })
    }
    ...
    if len(fixupCommits) != 0 {
        cp.PruneComments(func(comment github.IssueComment) bool {
            return strings.Contains(comment.Body, fixupCommitMsgCommentPruneBody)
        })

### [should-fix] No pre-creation/communication path for the new label, silent AddLabel failures
- where: `pkg/plugins/invalidcommitmsg/invalidcommitmsg.go:194-198`
- concern: if a repo's label set is managed by a labels-sync tool that doesn't yet include `do-not-merge/fixup-commits`, `AddLabel` fails; the failure is only logged (`log.WithError(err).Errorf(...)`), not surfaced to PR authors/operators, so failures go unnoticed at scale until logs are checked.
- excerpt: |
    if !hasFixupCommitMsgLabel && len(fixupCommits) != 0 {
        if err := gc.AddLabel(org, repo, number, fixupCommitMsgLabel); err != nil {
            log.WithError(err).Errorf("GitHub failed to add the following label: %s", fixupCommitMsgLabel)
        }
    }

### [nit] validateInvalidCommitMsg doesn't reject empty org/repo segments
- where: `pkg/plugins/config.go:1459-1477`
- concern: checks for empty string and `>1` slash, but `"org/"` or `"/repo"` pass validation despite being malformed and would then silently never match at runtime in `InvalidCommitMsgFor` — misconfiguration fails silently rather than at config-validation time.
- excerpt: |
    if strings.Count(repo, "/") > 1 {
        errs = append(errs, fmt.Errorf(
            "error validating invalid_commit_msg config #%d: repo %q must be of form org or org/repo", i, repo))
    }

### [nit] Stray blank line before closing brace in InvalidCommitMsgFor
- where: `pkg/plugins/config.go:1039-1059`
- concern: `return &InvalidCommitMsg{}` followed by a blank line then `}` — likely to trip `golangci-lint`'s whitespace linter.
- excerpt: |
    return &InvalidCommitMsg{}

    }

### [nit] Copy-pasted "triggers" comment on InvalidCommitMsgFor
- where: `pkg/plugins/config.go:1045`
- concern: comment `// Prioritize repo level triggers over org level triggers.` is copy-pasted from `TriggerFor`/`ApproveFor` — `InvalidCommitMsgFor` has nothing to do with triggers. Confusing for future readers.

### [nit] Three near-identical org/repo priority-resolution functions, no shared generic helper
- where: `pkg/plugins/config.go` (`TriggerFor`, `DcoFor`, `InvalidCommitMsgFor`)
- concern: `InvalidCommitMsgFor` is essentially a verbatim copy of `TriggerFor`/`DcoFor` with the receiver type changed, even though `ListableRepos`/`validateRepoDupes[C]` generic machinery already exists for the dupe-check side. A generic `configFor[C ListableRepos](configs []C, org, repo string, defaultFn func() C) C` could collapse all three. Not this PR's debt to introduce, but it adds a third copy.

### [nit] strings.Split allocates full line slice for first-line-only use
- where: `pkg/plugins/invalidcommitmsg/invalidcommitmsg.go:150`
- concern: `strings.Split(msg, "\n")[0]` allocates a slice of all lines just to take the first one, in a loop over `allCommits`. `strings.SplitN(msg, "\n", 2)[0]` avoids the extra work. Negligible in practice given typical commit counts.

### [nit] FixupCommitRegex doesn't require the space after `!`
- where: `pkg/plugins/invalidcommitmsg/invalidcommitmsg.go:82`
- concern: `^(fixup!|amend!)` matches even without the trailing space git's `--autosquash` convention always produces (e.g. `fixup!nospace: ...`), slightly looser than the real fixup/amend commit format. Low impact.

### [nit] Test doesn't assert both labels added for a combined fixup+close-keyword commit
- where: `pkg/plugins/invalidcommitmsg/invalidcommitmsg_test.go:230-247`
- concern: "commit with fixup and close keyword" test case only asserts `invalidCommitMsgLabel` membership in `IssueLabelsAdded`, not `fixupCommitMsgLabel` too, even though both should be added. Harness checks membership not exact-set, so a regression dropping the fixup label on combined commits wouldn't be caught.

### [question] Should #641 simply be closed in favor of #656 + a future config-surface PR?
- concern: given the maintainer's split request was already agreed to and #656 delivers the gated behavior-only half, is there any remaining reason to keep #641 open, or should it be closed outright with the config-surface work re-proposed fresh (fixing the issues above) in a new PR?

## Checked
- `CloseIssueRegex` still matches full commit message (not subject-only); only new `FixupCommitRegex` is subject-line-scoped — earlier regression from a prior revision is fixed.
- `InvalidCommitMsgFor` two-pass repo-then-org priority resolution matches the existing `TriggerFor`/`DcoFor` convention.
- `&cfg` taken from `for _, cfg := range` loop is safe: go.mod requires Go 1.25 (Go 1.22+ per-iteration loop variable semantics).
- `getRepos()` on `InvalidCommitMsg` correctly implements `ListableRepos` used by `validateRepoDupes`, matching `Approve`/`Welcome`.
- `dco.MarkdownSHAList` reuse for fixup commit list is consistent with its existing use for the invalid-commit-message list.
- Config schema change is additive/backward-compatible (`omitempty`, new struct) — no parsing breakage for existing deployments.
- `checkFixup` being false still allows stale fixup labels to be cleaned up (removal not gated by `checkFixup`) — correct when `disable_fixup_check` is turned on after the fact.
- `disable_fixup_check` wired through and tested end-to-end (config → `handle`).
- Test coverage of fixup-only, amend-only, disabled-check, and label-removal-on-resolution cases is solid.

## Open questions
- Should #641 be closed outright in favor of #656 plus a future config-surface PR, or is further work expected on #641 itself?
- Is the double comment-prune (on label removal, and again before adding a new comment) intentional, or leftover from iterating on the label-removal fix?
- If/when the config-surface PR is resubmitted, will it fix `validateRepoDupes`'s hardcoded "welcome" plugin wording (now three call sites: Approve, Welcome, InvalidCommitMsg)?
