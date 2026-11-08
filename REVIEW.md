---
pr: kubernetes-sigs/prow#825
title: "announcement: notify transfer-issue consolidation under issue-management plugin"
head_sha: 6beef156d0cf270918f4bc148ec29b82cbe9f6b4
base: main
reviewed_at: 2026-11-07T23:54:58Z
verdict: merged; blocking finding resolved upstream by #702 before merge
refresh_log:
  - at: 2026-11-07T23:54:58Z
    old_sha: 6beef156d0cf270918f4bc148ec29b82cbe9f6b4
    new_sha: 6beef156d0cf270918f4bc148ec29b82cbe9f6b4
    summary: "No code changes. PR approved and merged at 2026-08-11T21:41:16Z. Blocking finding resolved upstream: #702 moved transfer-issue into issue-management and merged at 2026-08-11T09:37:17Z, so the announcement was accurate against main at merge time."
---

## What this PR does
- Adds a dated announcement to `site/content/en/docs/announcements.md` (4 added lines, docs only) telling users that the `transfer-issue` plugin has been consolidated into `issue-management` along with the link/pin commands, and that `plugins.yaml` should list `issue-management` instead of `transfer-issue`.
- States that both `/transfer-issue <repo>` and `/transfer <repo>` are supported.

Since previous review:
- No new commits: head is still `6beef156d0cf270918f4bc148ec29b82cbe9f6b4`; the docs text is unchanged.
- PR was approved by `petr-muller` at `2026-08-11T21:30:42Z` and merged at `2026-08-11T21:41:16Z` (labels `approved`, `lgtm`, `area/documentation`, `size/XS`). `upodroid` commented at `2026-08-11T21:23:46Z`: "We should merge this now as the change is active."
- Verified against `upstream/main`: #702 ("plugins: move transfer-issue to issue management") merged at `2026-08-11T09:37:17Z`, adding `/transfer-issue` and `/transfer` to `pkg/plugins/issue-management/` (`issue_management.go:42`, `transfer-issue.go`) and deleting `pkg/plugins/transfer-issue/`. The announcement is correct for main.

## Findings

None open.

## Resolved

### [blocking] announcement claims a consolidation that has not happened in code
- resolved_by: kubernetes-sigs/prow#702 (`plugins: move transfer-issue to issue management`, merged `2026-08-11T09:37:17Z`) — resolved upstream, not by a change in this PR.
- resolution note: The finding was accurate for the reviewed tree (the PR branch is based on `ef12e757d`, which predates #702), but the review only inspected the branch, not `upstream/main`. Main already contained the consolidation when the review ran at `2026-08-11T14:26:20Z`, so the "request changes" verdict was based on a stale base; the merged announcement does not mislead users.
- where: `site/content/en/docs/announcements.md:12-15`
- concern: The announcement states `transfer-issue` "has been consolidated into the new `issue-management` plugin" and instructs users to replace `transfer-issue` with `issue-management` in `plugins.yaml`. This is false: `pkg/plugins/transfer-issue/transfer-issue.go` still defines and registers `pluginName = "transfer-issue"` as its own generic comment handler, it is still imported separately in `cmd/hook/plugin-imports/plugin-imports.go:66` alongside `issue-management` (line 41), and `pkg/plugins/issue-management/` contains no transfer/pin/link handling at all (confirmed via grep, no matches). A user following this announcement's advice would lose the `/transfer-issue <repo>` and `/transfer <repo>` commands entirely, since `issue-management` does not implement them.
- excerpt: |
    - *August 11, 2026* The `transfer-issue` plugin has been consolidated into the new `issue-management`
        plugin along with `/link-issue`, `/unlink-issue`, `/pin-issue`, and `/unpin-issue` commands.
        Users should replace `transfer-issue` with `issue-management` in their `plugins.yaml`.
        Both `/transfer-issue <repo>` and the shorter `/transfer <repo>` are supported.

## Checked
- `/link-issue`, `/unlink-issue`, `/pin-issue`, `/unpin-issue` genuinely exist in `issue-management` — that part of the announcement is accurate.
- `transfer-issue` and `issue-management` are still two separate, independently-registered plugins in `cmd/hook/plugin-imports/plugin-imports.go`.
- `pkg/plugins/issue-management/` has no transfer-related code (grep for "transfer" returns nothing).
- Change surface is a single commit, 4 added lines in a docs file only.
- On current `upstream/main`, `pkg/plugins/issue-management/issue_management.go:42-153` and `pkg/plugins/issue-management/transfer-issue.go` implement `/transfer[-issue]`, and `pkg/plugins/transfer-issue/` no longer exists.
- No inline review comments and no further commits after `6beef156`; only approval, labels, merge, and one comment.

## Open questions
- Answered: yes, #702 is that follow-up, and it merged into main before this PR. No need to hold the announcement.
- The "planned vs. done" wording question is moot; the past-tense claim is accurate as of merge.

## Followups

Walked 2026-11-08 after the merge: 1 accepted (as probe), 2 skipped (stale `Package transferissue` doc comment in `pkg/plugins/issue-management/transfer-issue.go`; migration hint in `ValidatePluginsUnknown` for configs still listing `transfer-issue`).

### [tests] probe the consolidated transfer dispatch on ambiguous input
- where: `pkg/plugins/issue-management/issue_management.go:42,133-155`; tests in `pkg/plugins/issue-management/`
- necessity: could
- mode: probe
- why: #702 merged `/transfer[-issue]` into a plugin that dispatches a single command per comment. Existing tests cover the happy path, missing destination repo, non-member, multiple/no destinations, and PR/non-created events, but not the ambiguous inputs where the old separate-plugin behaviour may have changed.

```
In `kubernetes-sigs/prow`, following PR #825 ("announcement: notify transfer-issue
consolidation under issue-management plugin", merged as bce31d10c03da9de362adcd0c2a3722de0d787fa;
the code move itself was #702, commit 1f6d66b17), probe the consolidated `/transfer[-issue]`
dispatch by writing throwaway table tests.

The task: in `pkg/plugins/issue-management/`, write uncommitted tests against
`issue_management.go` (regex at line 42, `parseTransferCommand` at 133-155, dispatch in
`handleIssues`) that exercise:
1. `/transfer-issue foo bar` (multi-token destination) — record whether the reply is the
   "must only be used once and with a single destination repo" comment or the generic
   "destination repo does not exist" comment, and which destination string reaches `GetRepo`.
2. `/transfer <repo>` on a pull request — confirm the PR rejection fires for the alias and
   note that the reply text is hardcoded to name only `/transfer-issue`.
3. One comment containing both `/link-issue <issue>` and `/transfer <repo>` — confirm
   `handleIssues` runs only the link handler and silently drops the transfer (the old separate
   `transfer-issue` plugin would have handled it).
4. `/transfer-issue   repo   ` with extra whitespace/trailing text, and a comment with two
   transfer lines, to confirm trimming and multi-match rejection.

Run `go test ./pkg/plugins/issue-management/...` (or the focused test names). Triage each
failure: real implementation bug, or a wrong test expectation.

Acceptance criteria: all four behaviours exercised and each result classified with evidence.
If a real bug is found, apply only the minimal fix in non-test code (for example correcting
the reply/rejection wording or rejecting a multi-token destination) and report which case
triggered it. If no real bug is found, no files changed.

Scope guard: delete the probe tests before finishing — do not submit them. Do not refactor
the plugin, add docs, touch `pkg/plugins/config.go`, or change other plugins. Report the
failing case for any bug fixed so it can be kept as a regression test separately.
```
