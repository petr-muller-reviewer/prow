---
issue: kubernetes-sigs/prow#1040
title: "Peribolos: Managing repository/org permissions for a GitHub Enterprise teams without managing members"
state: open
labels: kind/feature, area/peribolos
main_sha: c78b4fb59529c0abc6e1c75eb8ed3675f58ce99e
triaged_at: 2026-11-17T19:17:24Z
verdict: accepted
refresh_log:
  - at: 2026-11-17T19:17:24Z
    since: 2026-10-27T23:48:35Z
    summary: "Maintainer endorsement of a dedicated flag, kind/feature + area/peribolos applied, and reporter hmhngx implementing --ignore-enterprise-team-members; no PR or code change yet."
advice:
  advised_at: 2026-10-28T00:46:06Z
  based_on_triaged_at: 2026-10-27T23:48:35Z
---

# Triage

## Verdict

**Accepted — feature request.** Peribolos has code for identifying enterprise teams, but its current all-or-nothing ignore behavior cannot preserve those teams as permission targets while leaving their membership under enterprise management. Effort estimate: Level 2 (Moderate); the change needs explicit rules across team lifecycle, membership, and permission reconciliation.

## What the issue reports

- The reporter uses Peribolos to manage permissions across organizations in an on-prem GitHub Enterprise Server environment.
- Enterprise teams are managed centrally, while Peribolos should manage their repository and organization access within each org.
- `--fix-team-members` also processes enterprise teams whose configured member lists are empty, which can make Peribolos try to remove enterprise-managed members.
- `--ignore-enterprise-teams` avoids reconciling those teams but also prevents their configured permissions from being managed.

Since previous triage:

- petr-muller (2026-10-28T00:59:59Z) endorsed the idea of a dedicated flag and applied `/kind feature` `/area peribolos`.
- hmhngx (2026-11-17T06:06:53Z) reports implementing `--ignore-enterprise-team-members` so Peribolos can manage enterprise-team repository permissions without reconciling membership — the strategy the triage recommended.
- No PR or code change was linked by 2026-11-17.

## Findings

### [cause] Enterprise-team filtering and membership reconciliation are coupled

- detail: With `--ignore-enterprise-teams`, `configureTeams` filters enterprise teams out of the resolved team map, so later team permission configuration cannot use them. Without that flag, configured teams enter `configureTeamAndMembers`; when `--fix-team-members` is enabled, `configureTeamMembers` reconciles configured members and maintainers, including empty lists. This matches the reported gap; the exact GHES API error was not independently reproduced.
- evidence: `cmd/peribolos/main.go:928-960, 1241-1260, 1623-1659, 2057-2127`

### [related-code] Enterprise filtering, membership updates, and permission reconciliation

- where: `cmd/peribolos/main.go:948-952, 1241-1259, 1643-1647`
- excerpt: |
    if ignoreEnterpriseTeams && t.Type == github.TeamTypeEnterprise {
        logrus.Infof("Skipping enterprise team %s(%s) — managed at the enterprise level", t.Slug, t.Name)
        ignoredTeamSlugs.Insert(t.Slug)
        continue
    }

    if !opt.fixTeamMembers {
        logrus.Infof("Skipping %s member configuration", name)
    } else if err = configureTeamMembers(client, orgName, gt, team, opt.ignoreInvitees); err != nil {
        if opt.confirm {
            return fmt.Errorf("failed to update %s members: %w", name, err)
        }

### [related-code] Enterprise teams are identified by their API type

- where: `pkg/github/types.go:1171-1187`
- excerpt: |
    // TeamTypeEnterprise identifies teams managed at the enterprise level.
    TeamTypeEnterprise = "enterprise"

### [related-pr] Existing all-or-nothing enterprise-team ignore behavior

- ref: kubernetes-sigs/prow#710
- relevance: Merged PR #710 added `--ignore-enterprise-teams` to avoid team and member mutations; that option filters enterprise teams out of all team reconciliation, including permission targets.

### [related-pr] Enterprise membership handling in dumps

- ref: kubernetes-sigs/prow#977
- relevance: Merged PR #977 improved `--dump --ignore-enterprise-teams` handling for members whose org membership comes only from enterprise teams; it does not provide permission-only reconciliation.

## Checked

- Checked issue #1040's metadata on 2026-11-17: still open, labels now `kind/feature` and `area/peribolos` (applied 2026-10-28T01:00:02Z by kubernetes-prow[bot] after petr-muller's `/kind feature` `/area peribolos` comment).
- Queried the issue timeline for cross-references through 2026-11-17; none were returned.
- Searched PRs by hmhngx, for `ignore-enterprise-team-members`, and for open PRs with `enterprise` in the title; no implementation PR existed yet as of 2026-11-17.
- Fetched `upstream/main` on 2026-11-17: 49 commits ahead of `main_sha` but none touching `cmd/peribolos` or `pkg/github/types.go`, so the anchored analysis still holds.
- Searched issues for `peribolos enterprise teams`, `peribolos members permissions`, `enterprise team members`, and `enterprise teams`. No duplicate was confirmed; issue #737 concerns a separate `--ignore-repos` feature.
- Searched PRs for `enterprise team peribolos`; no active PR for this behavior was found. Read merged PRs #710 and #977.
- Read the relevant team configuration, enterprise-team filtering, membership reconciliation, repository/org-role permission paths, tests, and Peribolos documentation.
- Existing tests cover all-or-nothing enterprise-team filtering and ordinary member/maintainer diffs, but not keeping an enterprise team as a permission target while skipping its membership updates (`cmd/peribolos/main_test.go:1279-1315, 1597-1710`).

## Next steps

- Confirm the GHES version and whether its API supports the desired repository and organization-role assignments for enterprise teams.
- Agree whether team metadata should also remain outside Peribolos management.
- If confirmed, implement an opt-in permissions-only mode that protects enterprise teams from lifecycle and membership mutations while keeping configured teams available as permission targets; add tests and docs.
- Suggested labels: `kind/feature`, `area/peribolos`, `help wanted`. The first two were applied on 2026-10-28 by kubernetes-prow[bot]; `help wanted` is still unapplied.
- Track hmhngx's `--ignore-enterprise-team-members` implementation once a PR opens: it should keep enterprise teams as permission targets while skipping `configureTeamMembers` (and any team-metadata mutation), and should ship tests plus docs.

## Advice

Note (2026-11-17 refresh): partially superseded. `kind/feature` and `area/peribolos` are now applied; `help wanted` is still not, and the reporter reports an implementation in progress (comment of 2026-11-17).

The issue is still open and unlabeled; there are no new comments or linked PRs since triage. The existing next steps still cover the GHES/API and metadata questions, so there is no new disposition action to take.

1. Apply the suggested discovery labels. All three labels exist in the repository and none is currently applied.

   ```sh
   gh issue edit 1040 --repo kubernetes-sigs/prow \
     --add-label "kind/feature" \
     --add-label "area/peribolos" \
     --add-label "help wanted"
   ```

No milestone, assignee, or implementation PR is indicated by the current issue state.

## Open questions

- Which GHES version is in use, and what API response occurs when Peribolos attempts enterprise-team membership changes?
- Should the requested mode also skip team metadata edits, or only member and maintainer reconciliation?
- Should the behavior be enabled by a CLI option or a per-team config field?
