---
pr: kubernetes-sigs/prow#1008
title: "peribolos: add end-of-run change report"
head_sha: 570bc96db9af8d5921df6d0a9ac9a326a45aa66a
base: main
reviewed_at: 2026-11-23T17:31:44Z
verdict: approve
refresh_log:
  - at: 2026-11-23T17:31:44Z
    old_sha: 570bc96db9af8d5921df6d0a9ac9a326a45aa66a
    new_sha: 570bc96db9af8d5921df6d0a9ac9a326a45aa66a
    summary: "Full re-review of the same head, with each saved finding and earlier GitHub thread checked."
---

# Review

## Verdict

**Approve.** The maintainer discussion identified follow-ups for report delivery, completeness, and coverage, plus a separate team reconciliation bug. The report path collision is optional hardening.

## What this PR does

- Adds an opt-in `--report-path` flag for a versioned JSON file describing applied changes in confirm mode and a best-effort plan in dry-run.
- Reports org metadata, memberships, teams, team repo permissions, repositories, collaborators, and org-role assignments with per-action totals.
- Records changes after successful calls and emits a partial report when an org aborts.
- Since the previous local review at 2026-11-23 17:25:48 UTC, the head remains `570bc96db` and the PR remains open. GitHub shows no new commits, comments, reviews, or issue events in that interval. [PR #1008](https://github.com/kubernetes-sigs/prow/pull/1008)

## Resolved

### Earlier local findings

- **`none` recorded as an add or update — fixed.** An existing collaborator, pending invitation, or team-repo grant is recorded as a removal when its deletion succeeds. An absent grant produces no entry. The focused tests cover existing and absent direct grants in both paths. `cmd/peribolos/main.go:1681-1689`, `cmd/peribolos/main.go:1947-1951`, `cmd/peribolos/report_test.go:556-652`.
- **Empty permission recorded without an API call — fixed.** The planning switches skip empty or unsupported values before scheduling a change. The null-input completeness question is captured in Followups. `cmd/peribolos/main.go:1691-1700`, `cmd/peribolos/main.go:1944-1958`.
- **Dry-run docs claimed parity for existing objects — fixed in the user docs.** The narrower accuracy claim covers rename and cascading-delete examples as well as new objects. `site/content/en/docs/components/cli-tools/peribolos.md:196-202`.
- **Silent partial reports — partially fixed.** An aborted org and the dry-run team-member error path set `complete: false`. The repo invitation listing and swallowed org membership 404 still need the same signal. `cmd/peribolos/main.go:241-244`, `cmd/peribolos/main.go:1825-1834`.
- **Forward compatibility for unknown kinds, actions, and summary keys — fixed.** Both the user docs and the schema comment require tolerance for new enum values and summary keys. `site/content/en/docs/components/cli-tools/peribolos.md:234-239`, `cmd/peribolos/report.go:214-227`.
- **Code comments, help, and tests promised mode parity — fixed.** The `change` comment and flag help call dry-run best-effort. The tests explain why an all-success fake cannot validate live parity. `cmd/peribolos/report.go:50-67`, `cmd/peribolos/main.go:106`, `cmd/peribolos/report_test.go:411-419`.

- **Misspelled permission accepted by the CLI — refuted.** The parser rejects `rwite` before report creation. YAML null is accepted and remains relevant to the completeness follow-up. `pkg/github/types.go:570-578`.
- **Report file permissions — dropped.** The observed mode of `0644` follows the process umask of `022`. The maintainer declined this hardening. `cmd/peribolos/main.go:219`.

### Earlier GitHub review threads

- **Inconsistent recording after mutation failures — fixed at the existing recorder sites.** The batch paths and membership paths now record after checking the call result. The independently identified invitation-cleanup gap is an omitted recording site. `cmd/peribolos/report.go:150-161`, `cmd/peribolos/main.go:1792-1796`, `cmd/peribolos/main.go:1998-2002`. [Recording rule thread](https://github.com/kubernetes-sigs/prow/pull/1008#discussion_r4260862502)
- **Squash merge fields and internal tracker — fixed as a comment change.** The comment names the pre-existing `Defined()` limitation and removes the internal reference. A squash-title/message-only delta still reaches neither the API nor the report. `cmd/peribolos/main.go:1455-1460`, `pkg/github/types.go:488-517`. [Squash field thread](https://github.com/kubernetes-sigs/prow/pull/1008#discussion_r4260862999)
- **Inconsistent identifiers — fixed for logins and documented for teams.** Logins now use their normalized form across kinds. Team entries still use config or display names and `org_role_team` still uses a slug, so joining those entries requires a mapping. The reviewer asked for at least this documentation. `cmd/peribolos/report.go:73-92`, `site/content/en/docs/components/cli-tools/peribolos.md:227-228`. [Identifier thread](https://github.com/kubernetes-sigs/prow/pull/1008#discussion_r4260863893)
- **Late destination errors and the `--dump` combination — fixed as requested.** Opening the file precedes reconciliation and flags reject `--dump` with `--report-path`. Opening early exposes the aliasing problem documented in Followups. `cmd/peribolos/main.go:146-148`, `cmd/peribolos/main.go:211-230`. [Destination thread](https://github.com/kubernetes-sigs/prow/pull/1008#discussion_r4260866314)
- **Mixed JSON types for `before` and `after` — fixed.** Both fields are scalar strings, with a separate `changed_fields` array. `cmd/peribolos/report.go:95-107`. [Field type thread](https://github.com/kubernetes-sigs/prow/pull/1008#discussion_r4260866732)
- **Versioning rules only in a Go comment — fixed.** The user guide covers the flag, an example, the kinds and their identifiers, and the consumer compatibility rules. `site/content/en/docs/components/cli-tools/peribolos.md:190-239`. [Documentation thread](https://github.com/kubernetes-sigs/prow/pull/1008#discussion_r4260868023)
- **Initial dry-run divergence examples — addressed.** All four examples are documented, including the synthetic default branch and pending-role cases. `site/content/en/docs/components/cli-tools/peribolos.md:196-202`. [First dry-run thread](https://github.com/kubernetes-sigs/prow/pull/1008#discussion_r4451018044)
- **Follow-up dry-run divergence and silent skips — partially fixed.** The document adds existing-object divergences and the team failure sets `complete: false`. Two swallowed-error paths still report completion, as described in Followups. `site/content/en/docs/components/cli-tools/peribolos.md:196-204`, `cmd/peribolos/main.go:1825-1834`. [Follow-up dry-run thread](https://github.com/kubernetes-sigs/prow/pull/1008#discussion_r4462967850)
- **`none` classification and empty-permission phantom entries — fixed.** The reporter now distinguishes removing an existing grant from a converged configuration, and an empty permission schedules no change. [Permission thread](https://github.com/kubernetes-sigs/prow/pull/1008#discussion_r4462968978)
- **Unknown `kind`, `action`, and summary values — fixed.** The published contract and the schema comment specify how consumers should handle them. [Schema thread](https://github.com/kubernetes-sigs/prow/pull/1008#discussion_r4462969171)

## Checked

- Reviewed the full five-file diff from merge base `0edcac618455b9e7716a08be480d6984ef996eb1` to `570bc96db9af8d5921df6d0a9ac9a326a45aa66a` (+1404/-118), pinned against GitHub's current base `b8bd4f20e`.
- `go test ./cmd/peribolos/ -count=1`, `go test ./pkg/github/ -run '^TestRepoPermissionLevel$' -count=1`, and `git diff --check 0edcac618455...HEAD` passed.
- Ran local CLI checks for aliased config and token paths, output mode, report write failure, and misspelled versus null YAML permissions. The API reconciliation findings follow from inspection of the call paths and test fakes.
- The GitHub lint, unit, race, integration, image-build, and CLA checks were green. Tide was pending.

## Standards

No documented standards violations found in `CONTRIBUTING.md` or `.golangci.yml`. The new permission allowlists repeat the GitHub type registry and each mutation dispatch switch. If those lists drift, neither execution switch rejects an unhandled permission, so it can fall through to recording with a nil error. An explicit executed-and-succeeded check would protect the recording contract. `pkg/github/types.go:556-563`, `cmd/peribolos/main.go:1695-1700`, `cmd/peribolos/main.go:1749-1795`, `cmd/peribolos/main.go:1953-1958`, `cmd/peribolos/main.go:1983-2001`.

## Spec

Reviewed against the [PR description](https://github.com/kubernetes-sigs/prow/pull/1008) and the amended user guide. The follow-ups apply the guide's best-effort dry-run contract. The opt-in claim also has two intentional reconciliation exceptions from the feedback commits. Explicit `none` can now skip an unnecessary delete even with the report off, and an empty team description alone skips a PATCH that cannot send that value. `cmd/peribolos/main.go:1867-1875`, `cmd/peribolos/main.go:1947-1951`.

## Followups

### Report errors should exit nonzero

A requested report can fail during write or close after reconciliation. A local run
targeting `/dev/full` logged ENOSPC and exited zero. Return a nonzero status and
identify which operation failed. Preserve both errors when reconciliation and report
delivery fail. `cmd/peribolos/main.go:241-273`.

### Mark skipped reconciliation incomplete

Mark the report incomplete when listing repository invitations fails or an org
membership update returns a swallowed 404. With null YAML permissions, Peribolos warns
and leaves any existing grant alone. Decide whether this skip should also mark the
report incomplete. If useful, clarify the user guide so it names the conditions that
set `complete: false` and distinguishes them from ordinary dry-run discrepancies.
`cmd/peribolos/main.go:813-823`, `cmd/peribolos/main.go:1642-1648`,
`cmd/peribolos/main.go:1691-1700`, `cmd/peribolos/main.go:1953-1958`,
`site/content/en/docs/components/cli-tools/peribolos.md:196-204`.

### Account for successful failed-invitation cleanup

Peribolos can successfully delete a failed invitation before attempting a reinvite.
The deletion is omitted from the report. A failed reinvite can leave the changes array
empty despite the deletion. Record the deletion as invitation cleanup or explicitly
scope that operation out of the documented coverage. `cmd/peribolos/main.go:802-823`.

### Preserve a nested team's configured parent on metadata edits

Track this as an independent reconciliation bug. The existing team PATCH sets
`parent_team_id: null` on a description-only edit when the current parent matches the
configured parent. This can detach the team while the report lists only
`description`. Send the configured parent ID on the PATCH and test that request.
`cmd/peribolos/main.go:1877-1908`, `pkg/github/client.go:4072-4080`.

### Optionally guard against an output path that aliases an input

I reproduced the overwrite with each of a temporary config and a temporary token.
The destination must resolve to an input file for this to happen. As optional
hardening, compare file identities before truncating the destination.
`cmd/peribolos/main.go:201-224`.
