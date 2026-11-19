---
pr: kubernetes-sigs/prow#1008
title: "peribolos: add end-of-run change report"
head_sha: 617e3a3cc2613c38a3db13e0ea74d648b506623d
base: main
reviewed_at: 2026-11-04T17:51:41Z
verdict: request-changes
refresh_log:
  - at: 2026-11-03T11:35:20Z
    old_sha: c341b7f065b84442df6dd360f1695992b69ac38a
    new_sha: 617e3a3cc2613c38a3db13e0ea74d648b506623d
    summary: "Docs-only commit documenting that the dry-run report is a best-effort plan; added should-fix for code comments still claiming dry-run/confirm parity. Blocking `none` classification finding unchanged."
  - at: 2026-11-04T17:51:41Z
    old_sha: 617e3a3cc2613c38a3db13e0ea74d648b506623d
    new_sha: 617e3a3cc2613c38a3db13e0ea74d648b506623d
    summary: "No code changes. Incorporated @cblecker's 2026-11-04 review: added should-fix findings for the dry-run doc overclaim on existing objects, swallowed team-member listing errors, and empty-permission false `add`; extended the unknown-value and stale-comment findings; blocking `none` finding independently corroborated by @cblecker."
gate:
  decision: do-not-merge
  gated_at: 2026-11-04T17:54:08Z
  gated_head_sha: 617e3a3cc2613c38a3db13e0ea74d648b506623d
  reviewed_head_sha: 617e3a3cc2613c38a3db13e0ea74d648b506623d
---

# Review

## Gate

**Decision: do-not-merge** — the blocking `none`-classification defect is unchanged at `617e3a3cc`, and `@cblecker` independently confirmed it in his 2026-11-04 review.

Nothing in `cmd/peribolos` has changed since `c341b7f`; the only later commit (`617e3a3cc`) touches documentation. All seven gating findings from the refreshed review still stand in the current code, and `@cblecker`'s second round (three inline comments, two COMMENTED reviews on 2026-11-04) re-raised four of them and added the empty-permission case. The blast radius is narrow — everything is behind the opt-in `--report-path` flag — but the feature exists to produce an accurate record, and as written a converged config can never yield an empty report, which defeats the purpose for the primary use case.

Gating list:

- **Not addressed — `REVIEW.md` (blocking), `cmd/peribolos/main.go:1685-1693` and `1912-1915`:** `github.None` is executed as a removal by the switch but recorded as `add`/`update` by the classification branches. `@cblecker` confirmed the same at `main.go:1692` (converged config keeps reporting changes).
- **Not addressed — `REVIEW.md` + `@cblecker` (should-fix), `cmd/peribolos/main.go:1715-1766`:** a permission matching no `case` (e.g. `alice:` parsing to an empty level) leaves `err` nil and still reaches `report.record`, so a change is reported with no API call. Same shape in `configureTeamRepos`.
- **Not addressed — `REVIEW.md` (should-fix), `cmd/peribolos/main.go:218-220`:** there is still no guard against `--report-path` resolving to `--config-path`; `os.Create` truncates the config after it was loaded.
- **Not addressed — `REVIEW.md` + `@cblecker` (should-fix), `site/content/en/docs/components/cli-tools/peribolos.md:234-235` and `cmd/peribolos/report.go:193-196`:** the compatibility rule still only covers unrecognized property *names*, not unrecognized `kind`/`action` values or new `summary` keys.
- **Not addressed — `REVIEW.md` + `@cblecker` (should-fix), `site/content/en/docs/components/cli-tools/peribolos.md:196`:** the docs claim the dry-run plan matches for existing objects; cascaded `team_member`/`org_role_team` removes and renamed-repo aborts contradict it.
- **Not addressed — `REVIEW.md` + `@cblecker` (should-fix), `cmd/peribolos/main.go:1795-1801`:** dry-run downgrades team-member listing errors to warnings, so a 403/5xx silently omits a team from the report while the run exits 0; no completeness signal exists.
- **Not addressed — `REVIEW.md` + `@cblecker` (should-fix), `cmd/peribolos/report.go:50-57`, `cmd/peribolos/report_test.go:346-347`, `cmd/peribolos/main.go:106`:** code comments and flag help still assert dry-run/confirm parity.

Dispositioned, not gating:

- First-round `@cblecker` inline comments (2026-10-13) are addressed in `c341b7f`: recording after a successful call, documented repo-field limitations, normalized identifiers, report-path validation and partial-failure output, typed before/after values, CLI documentation.
- `@petr-muller` APPROVED at `c341b7f` on 2026-10-27 with a hold; the LGTM was withdrawn when `617e3a3cc` landed and does not cover the unaddressed blocking item.

Independent merge risk:

- No exported API surface: every code change is in `package main` under `cmd/peribolos` (`main.go`, new `report.go`, tests). Blast radius is the peribolos binary only; nothing imports these symbols.
- `--report-path` is new and opt-in. With the flag unset, `report` stays nil and every record path is a no-op, so existing invocations behave identically; the parameter threading is behavior-preserving (`else if` → early-return refactors on the same paths).
- Opt-in report-file creation happens before reconciliation, so a bad path fails fast by design (`Fatalf`); `--report-path` cannot be combined with `--dump`.
- Write/close failures are logged but do not change the exit status, so automation can observe success with a truncated or missing report.
- Rolling back to a binary without the flag requires removing `--report-path` from invocations.
- No Kubernetes API/CRD, proto, config-schema, or wire-format change; docs change is limited to `site/content/en/docs/components/cli-tools/peribolos.md`. No repository compatibility skill applies to this CLI-only change.

## Verdict

Request changes: configured `none` permissions can be reported as adds or updates even though reconciliation removes access.

This makes the report inaccurate for a valid configuration value and undermines its purpose as a record of applied changes. `@cblecker` independently reported the same defect on 2026-11-04 (`cmd/peribolos/main.go:1692`), noting that a converged config would never produce an empty report. `@petr-muller` approved at `c341b7f` on 2026-10-27 (with `/hold`), but the LGTM was removed when `617e3a3cc` was pushed, and the second review round found further accuracy problems in the dry-run documentation and error handling. The other perspectives found useful follow-up improvements, but no additional merge blockers.

## What this PR does

- Adds an opt-in `--report-path` flag for a versioned JSON change report.
- Reports organization metadata, memberships, teams, repositories, collaborators, team repository permissions, and organization role assignments.
- Records successful mutations in confirm mode and planned mutations in dry-run mode, with stable ordering and action totals.
- Avoids including changed field values in the report and censors the serialized output.

Since previous review (no code changes; `617e3a3cc` unchanged):

- `@cblecker` (2026-11-04, three inline comments and two COMMENTED reviews) extended the dry-run critique to existing objects and to silently skipped objects: cascaded `team_member`/`org_role_team` removes, renamed repos via `previously` with `--fix-collaborators`, and listing errors swallowed by `configureTeamAndMembers` in dry-run.
- He also independently raised the blocking `none` classification defect (`main.go:1692`), an empty-permission false `add`, and the unknown-`kind`/`action` consumer rule.
- No author response or new commits since; the last push remains `617e3a3cc` from 2026-11-03.

## Findings

### [blocking] Classify `none` permissions as removals
- where: `cmd/peribolos/main.go:1685-1693`
- concern: `github.None` is a valid permission and the execution switch treats it as removal, but these branches classify the action by whether the collaborator exists. The same mismatch exists for team repository permissions at `cmd/peribolos/main.go:1912-1915`; classify `none` as remove and cover both cases with tests. `@cblecker` adds a concrete symptom (`main.go:1692`): because the report says `add`/`update`, a converged config keeps producing non-empty reports on every run.
- excerpt: |
    if currentInfo, exists := currentCollaborators.collaborators[normalizedUser]; exists {
        actions[wantUser] = plannedChange{perm: wantPermission, change: change{Kind: kindCollaborator, Org: orgName, Scope: repoName, Target: normalizedUser, Action: actionUpdate, Before: string(currentInfo.permission), After: string(wantPermission)}}
    } else {
        actions[wantUser] = plannedChange{perm: wantPermission, change: change{Kind: kindCollaborator, Org: orgName, Scope: repoName, Target: normalizedUser, Action: actionAdd, After: string(wantPermission)}}
    }

### [should-fix] Document handling for unknown kind and action values
- where: `site/content/en/docs/components/cli-tools/peribolos.md:234-235`
- concern: The schema policy allows adding `kind` and `action` values without a version bump but only tells consumers to ignore unknown property names. Consumers also need to tolerate unrecognized values in these known fields — and unrecognized keys in `summary` — to make the compatibility promise reliable. The same sentence is missing from the `reportSchemaVersion` comment at `cmd/peribolos/report.go:193-196`.
- excerpt: |
    * Backward-compatible changes (a new field, or a new `kind`/`action` value) do **not** bump the version. Consumers **must** ignore properties with names they do not recognize so they keep working against newer peribolos versions.

### [should-fix] Prevent the report from truncating the configuration file
- where: `cmd/peribolos/main.go:218-220`
- concern: If `--report-path` resolves to the same file as `--config-path`, `os.Create` truncates the configuration after it has been loaded. Rejecting that path would prevent accidental loss of the source config.
- excerpt: |
    if o.reportPath != "" {
        reportFile, err = os.Create(o.reportPath)
        if err != nil {

### [should-fix] Dry-run docs still overclaim parity for existing objects
- where: `site/content/en/docs/components/cli-tools/peribolos.md:196`
- concern: The new text says "For existing objects the plan matches what `--confirm` would apply". That does not hold, because later reads still observe state the skipped mutations would have changed: removing an org member cascades recorded `team_member` removes and a deleted team that holds a role cascades an `org_role_team` remove, neither of which `--confirm` would perform; a repo renamed via `previously` with `--fix-collaborators` fails the listing and aborts the org. The caveat should cover existing objects too, or the claim should be removed.
- excerpt: |
    In dry-run every mutating call is a no-op that succeeds, so the report is a best-effort *plan* rather than a guaranteed account of what `--confirm` would do. For existing objects the plan matches what `--confirm` would apply, but for objects the run would create it can diverge, because dry-run still reads current state from GitHub while the skipped mutations return synthesized results:

### [should-fix] Report can be silently incomplete when team listing fails
- where: `cmd/peribolos/main.go:1795-1801`
- concern: `configureTeamAndMembers` downgrades any `configureTeamMembers` error to a warning in dry-run, so a 403/5xx on an existing team drops that team's members and child teams from the report while the run exits 0. The report is presented as a plan, so consumers need a way to tell a complete plan from a partial one — e.g. a top-level `complete`/`error` field (suggested by `@cblecker` on 2026-11-04). Note that this swallow predates the PR, but the report raises its impact.
- excerpt: |
    } else if err = configureTeamMembers(client, orgName, name, gt, team, opt.ignoreInvitees, report); err != nil {
        if opt.confirm {
            return fmt.Errorf("failed to update %s members: %w", name, err)
        }
        logrus.WithError(err).Warnf("failed to update %s members: %s", name, err)
        return nil
    }

### [should-fix] Empty permission records an add that never happens
- where: `cmd/peribolos/main.go:1715-1766`
- concern: A config entry like `alice:` parses to an empty `RepoPermissionLevel`, which matches no case in the execution switch, so `err` stays nil and `report.record(pa.change)` still records an `add` even though no API call was made. Recording in a `default` case (or validating the permission during parsing) would keep the report honest. Related to the blocking `none` classification finding: both record actions that were not applied.
- excerpt: |
            if err != nil {
                updateErrors = append(updateErrors, fmt.Errorf("failed to update %s/%s collaborator %s to %s: %w", orgName, repoName, user, permission, err))
            } else {
                report.record(pa.change)
            }
        }

### [should-fix] Code comments still promise dry-run/confirm parity
- where: `cmd/peribolos/report.go:50-57`
- concern: The `change` type comment says "in dry-run ... every planned change is recorded; in confirm mode a call that fails records nothing, so the report lists exactly the mutations that took effect" — true of the recording rule, but it reads as a guarantee that the dry-run report equals what `--confirm` would apply. The same overstatement appears at `cmd/peribolos/report_test.go:346-347` ("the dry-run report is the same") and in the `--report-path` flag help at `cmd/peribolos/main.go:106` ("all planned (dry-run) changes"). The new docs text says otherwise for objects the run would create, so these should point at that caveat.
- excerpt: |
    // Recording rule, applied at every site: a change is recorded only after its
    // mutating API call returns a nil error, never before. In dry-run the GitHub
    // client is a no-op that returns nil without calling the API, so every planned
    // change is recorded; in confirm mode a call that fails records nothing, so the
    // report lists exactly the mutations that took effect. See recordOnSuccess,
    // which enforces this for the plain-call sites.

## Resolved

### [should-fix] Write the report after partial reconciliation failures
- where: `cmd/peribolos/main.go:228-243`
- resolution: The run now saves the reconciliation error, emits the accumulated report, and only then exits with the error. This preserves the report when earlier operations succeeded before a later failure.

## Checked

- Reviewed the full PR diff from merge base `0edcac618455b9e7716a08be480d6984ef996eb1` to head `617e3a3cc2613c38a3db13e0ea74d648b506623d`; no changes since the last refresh.
- Confirmed `github.None` is accepted by `RepoPermissionLevel` and is handled as a removal by the mutation switch; unchanged by `617e3a3cc`.
- Confirmed at `cmd/peribolos/main.go:1715-1759` that a permission matching no `case` leaves `err` nil and still reaches `report.record`, so an empty permission yields a reported `add` with no API call.
- Confirmed at `cmd/peribolos/main.go:1795-1801` that dry-run (non-`--confirm`) downgrades `configureTeamMembers` errors to a warning and returns nil, so the surrounding team and its children are simply missing from the report.
- Re-read `site/content/en/docs/components/cli-tools/peribolos.md:191-236` at `617e3a3cc`; the new dry-run caveat describes real behavior except for the "existing objects match" sentence.
- Checked report recording paths, serialization, deterministic ordering, secret censoring, and the documented JSON schema.
- Existing invocations are unaffected unless they opt into `--report-path`; rolling back to a binary without the flag requires removing it from the invocation.
- Tests were not run.

## Open questions

- Should report write or close failures make the run exit nonzero when `--report-path` is requested? They are currently logged while the process can still exit successfully.
- Would a top-level completeness indicator (`complete`/`error` with counts of skipped objects) be preferable to enumerating every dry-run divergence in prose?
