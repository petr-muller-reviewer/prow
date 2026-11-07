---
pr: kubernetes-sigs/prow#825
title: "announcement: notify transfer-issue consolidation under issue-management plugin"
head_sha: 6beef156d0cf270918f4bc148ec29b82cbe9f6b4
base: main
reviewed_at: 2026-08-11T14:26:20Z
verdict: request-changes
---

## Findings

### [blocking] announcement claims a consolidation that has not happened in code
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

## Open questions
- Is there a follow-up PR (referenced as #702 in the commit message) that actually merges `transfer-issue` into `issue-management`, and should this announcement be held until that lands?
- If the consolidation is only planned, not done, should the announcement be reworded to describe planned/future behavior instead of past tense ("has been consolidated")?
