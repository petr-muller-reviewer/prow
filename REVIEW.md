---
pr: kubernetes-sigs/prow#1008
title: "peribolos: add end-of-run change report"
head_sha: 617e3a3cc2613c38a3db13e0ea74d648b506623d
base: main
reviewed_at: 2026-11-03T11:35:20Z
verdict: request-changes
refresh_log:
  - at: 2026-11-03T11:35:20Z
    old_sha: c341b7f065b84442df6dd360f1695992b69ac38a
    new_sha: 617e3a3cc2613c38a3db13e0ea74d648b506623d
    summary: "Docs-only commit documenting that the dry-run report is a best-effort plan; added should-fix for code comments still claiming dry-run/confirm parity. Blocking `none` classification finding unchanged."
gate:
  decision: do-not-merge
  gated_at: 2026-10-15T23:18:51Z
  gated_head_sha: c341b7f065b84442df6dd360f1695992b69ac38a
  reviewed_head_sha: c341b7f065b84442df6dd360f1695992b69ac38a
---

# Review

## Gate

**Decision: do-not-merge** — a confirmed blocking report-classification defect remains at the current PR head.

This gate was evaluated at `c341b7f065b84442df6dd360f1695992b69ac38a`. One docs-only commit (`617e3a3cc`) has landed since; it does not touch `cmd/peribolos`, so the decision and the gating list below are unchanged. The six substantive inline comments from `@cblecker` appear addressed in `c341b7f`: recording follows successful calls, repo-field limitations are documented, identifiers are normalized/documented, report-path validation and partial-failure output are implemented, before/after values are typed, and CLI documentation was added. The local review's blocking `none` permission issue remains, along with three should-fix findings.

Gating list:

- **Not addressed — `REVIEW.md` (blocking), `cmd/peribolos/main.go:1685-1693` and `1912-1915`:** `github.None` is executed as removal but may be recorded as an add/update. Fix action classification and add regression coverage before merge.
- **Not addressed — `REVIEW.md` (should-fix), `site/content/en/docs/components/cli-tools/peribolos.md:234-235`:** the schema says new `kind`/`action` values are compatible but does not tell consumers how to handle unknown values.
- **Not addressed — `REVIEW.md` (should-fix), `cmd/peribolos/main.go:218-220`:** `--report-path` matching `--config-path` truncates the configuration after it has been read. Reject the alias to avoid source-file loss.

Independent merge risk:

- No exported Go API or configuration schema is changed. Existing runs are unaffected unless they opt into the new, release-noted `--report-path` flag.
- For users enabling reporting, write/close errors are logged but do not change the exit status, so automation may see success without a valid report. Rolling back to an older binary also requires removing the new flag.
- No repository compatibility skill specifically applies to this CLI/report change; compatibility was assessed directly from the diff and current call paths.

## Verdict

Request changes: configured `none` permissions can be reported as adds or updates even though reconciliation removes access.

This makes the report inaccurate for a valid configuration value and undermines its purpose as a record of applied changes. The other perspectives found useful follow-up improvements, but no additional merge blockers. `@petr-muller` approved at `c341b7f` on 2026-10-27 (with `/hold`); that LGTM was removed automatically by the bot when `617e3a3cc` was pushed. The blocking finding is in `cmd/peribolos` and is untouched by the new docs commit, so this verdict stands.

## What this PR does

- Adds an opt-in `--report-path` flag for a versioned JSON change report.
- Reports organization metadata, memberships, teams, repositories, collaborators, team repository permissions, and organization role assignments.
- Records successful mutations in confirm mode and planned mutations in dry-run mode, with stable ordering and action totals.
- Avoids including changed field values in the report and censors the serialized output.

Since previous review (`c341b7f` → `617e3a3cc`):

- `@cblecker` reported that a dry-run report cannot match `--confirm` for objects the run would create: a new team's members and child teams are omitted, `--fix-team-repos`/`--fix-collaborators` stop the run at a new object, a new repo with `default_branch` gets a spurious `update`, and a pending invite with an org role gets a spurious `org_role_user` add.
- `@hoxhaeris` rewrote the change-report docs to call the dry-run output a best-effort *plan* and enumerate those four divergences (`site/content/en/docs/components/cli-tools/peribolos.md:194-202`); no code changed.
- The code-side follow-through `@cblecker` also asked for is still missing: the `change` type comment in `cmd/peribolos/report.go` and the test comment in `cmd/peribolos/report_test.go` still assert dry-run/confirm parity.

## Findings

### [blocking] Classify `none` permissions as removals
- where: `cmd/peribolos/main.go:1685-1693`
- concern: `github.None` is a valid permission and the execution switch treats it as removal, but these branches classify the action by whether the collaborator exists. The same mismatch exists for team repository permissions at `cmd/peribolos/main.go:1912-1915`; classify `none` as remove and cover both cases with tests.
- excerpt: |
    if currentInfo, exists := currentCollaborators.collaborators[normalizedUser]; exists {
        actions[wantUser] = plannedChange{perm: wantPermission, change: change{Kind: kindCollaborator, Org: orgName, Scope: repoName, Target: normalizedUser, Action: actionUpdate, Before: string(currentInfo.permission), After: string(wantPermission)}}
    } else {
        actions[wantUser] = plannedChange{perm: wantPermission, change: change{Kind: kindCollaborator, Org: orgName, Scope: repoName, Target: normalizedUser, Action: actionAdd, After: string(wantPermission)}}
    }

### [should-fix] Document handling for unknown kind and action values
- where: `site/content/en/docs/components/cli-tools/peribolos.md:234-235`
- concern: The schema policy allows adding `kind` and `action` values without a version bump but only tells consumers to ignore unknown property names. Consumers also need to handle or skip unrecognized values in these known fields to make the compatibility promise reliable.
- excerpt: |
    * Backward-compatible changes (a new field, or a new `kind`/`action` value) do **not** bump the version. Consumers **must** ignore properties with names they do not recognize so they keep working against newer peribolos versions.

### [should-fix] Prevent the report from truncating the configuration file
- where: `cmd/peribolos/main.go:218-220`
- concern: If `--report-path` resolves to the same file as `--config-path`, `os.Create` truncates the configuration after it has been loaded. Rejecting that path would prevent accidental loss of the source config.
- excerpt: |
    if o.reportPath != "" {
        reportFile, err = os.Create(o.reportPath)
        if err != nil {

### [should-fix] Code comments still promise dry-run/confirm parity
- where: `cmd/peribolos/report.go:50-57`
- concern: The `change` type comment says "in dry-run ... every planned change is recorded; in confirm mode a call that fails records nothing, so the report lists exactly the mutations that took effect" — which is true of the recording rule but reads as a guarantee that the dry-run report equals what `--confirm` would apply. The same overstatement is repeated at `cmd/peribolos/report_test.go:346-347` ("the dry-run report is the same"). The new docs text says the opposite for objects the run would create, so the comments should point at that caveat.
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

- Reviewed the full PR diff from merge base `0edcac618455b9e7716a08be480d6984ef996eb1` to head `617e3a3cc2613c38a3db13e0ea74d648b506623d`.
- Confirmed `github.None` is accepted by `RepoPermissionLevel` and is handled as a removal by the mutation switch; unchanged by `617e3a3cc`.
- Checked report recording paths, serialization, deterministic ordering, secret censoring, and the documented JSON schema.
- Re-read `site/content/en/docs/components/cli-tools/peribolos.md:191-236` at `617e3a3cc`; the new dry-run caveat describes real behavior and is consistent with the reconciliation code paths cited in the review comment.
- Existing invocations are unaffected unless they opt into `--report-path`; rolling back to a binary without the flag requires removing it from the invocation.
- Tests were not run.

## Open questions

- Should report write or close failures make the run exit nonzero when `--report-path` is requested? They are currently logged while the process can still exit successfully.
