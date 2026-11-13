---
pr: kubernetes-sigs/prow#1052
title: "peribolos: support members and commit signoff org settings"
head_sha: 9eb63ba114c7b5dfe71f3d8c1d0242213f673244
base: main
reviewed_at: 2026-11-04T13:13:20Z
verdict: approve
refresh_log:
  - at: 2026-11-04T13:13:20Z
    from_sha: 9d87cb7376064c590601de2a52aab7ca0c4eb701
    to_sha: 9eb63ba114c7b5dfe71f3d8c1d0242213f673244
    summary: "Rebased+squashed onto 43ed23400; all prior findings and cblecker's five inline comments addressed (gofmt, clearing the internal field on non-enterprise orgs, skip warning, nil-branch tests)."
---

# Review

## Verdict

All prior blocking findings are fixed and covered by tests; nothing left that should hold up a merge.

The delta since the last review is small and targeted: the non-enterprise path now clears
`MembersCanCreateInternalRepositories` before `EditOrg`, the silent skip became a warning, the two
missing nil-branch tests were added, and `gofmt` is clean. The only remarks left are cosmetic (commit
message wording) and a documentation question about the plan-based gate.

## What this PR does

- Adds `members_can_create_public_repositories`, `members_can_create_private_repositories`,
  `members_can_fork_private_repositories` and `web_commit_signoff_required` to peribolos: config
  schema, `--dump`, and `--fix-org` reconciliation.
- Adds `members_can_create_internal_repositories`, gated on `plan.name == "enterprise"` for both dump
  and apply, because the orgs/update API rejects the key (true *or* false) for other orgs with 422.
- Extends the `github.Organization` response/PATCH model with those fields; the enterprise-only one
  is `*bool` + `omitempty`, so it can be left out of the PATCH.
- Documents the new keys in the peribolos site page.
- Most of the `main_test.go` delta is mechanical re-alignment of expected struct literals after the
  longest field name changed; the substantive additions are one case per new field, plus enterprise /
  non-enterprise / API-omits-it cases for the internal setting in the dump test.

Since previous review:

- The PR was rebased onto `43ed23400` and squashed into a single commit, `9eb63ba11` (carries the
  `Signed-off-by` the second of the two old commits was missing).
- `configureOrgMeta` now nils `MembersCanCreateInternalRepositories` on non-enterprise plans and logs
  a warning when a configured value is skipped, with a regression test for the clear-on-other-change
  case.
- Two `TestConfigureOrgMeta` cases were added for the enterprise nil-`have` branch (set it, and
  both-nil means no change); `gofmt` was applied to both type files.

## Findings

### [nit] squashed commit message omits `members_can_fork_private_repositories`

- where: commit `9eb63ba11`
- concern: the message lists `members_can_create_public_repositories`,
  `members_can_create_private_repositories`, `members_can_create_internal_repositories` and
  `web_commit_signoff_required`, but the change also adds `members_can_fork_private_repositories`
  (schema, dump, apply, docs, tests). The PR description does list it, so this is cosmetic only.

### [question] `plan.name == "enterprise"` is still the capability probe

- where: `cmd/peribolos/main.go:248`, `cmd/peribolos/main.go:1124-1139`
- concern: the gate is unchanged, so an org whose `plan` is unpopulated (cblecker's point about apps
  without the Organization plan permission) is never managed here. The new warning at least makes that
  visible, with the org and the observed plan name, which is a reasonable outcome - but it means an
  enterprise org can be silently dumped *without* the field too. Fine to keep, provided the behaviour
  is documented.
- excerpt: |
    	} else {
    		if want.MembersCanCreateInternalRepositories != nil {
    			logrus.Warnf("ignoring members_can_create_internal_repositories for org %s: plan %q is not %q", orgName, cur.Plan.Name, github.EnterprisePlan)
    		}
    		cur.MembersCanCreateInternalRepositories = nil
    	}

## Resolved

- **gofmt not applied** - fixed in `9eb63ba11`; `gofmt -s -l cmd/peribolos pkg/config/org pkg/github`
  is clean.
- **Non-enterprise orgs echoed the internal-repository key in every org PATCH** - fixed exactly as
  suggested: the new `else` branch sets `cur.MembersCanCreateInternalRepositories = nil`, and the case
  "members can create internal cleared on non-enterprise org when other fields change" asserts the
  struct handed to `EditOrg` no longer carries it.
- **Silently ignoring the internal setting on a non-enterprise org** - now warns with org and plan;
  matches what cblecker asked for.
- **The nil branch was awkward and untested** - kept, but now covered: "set on enterprise org with nil
  have" (`change: true`) and "nil want and have on enterprise org means no change" (`change: false`).
- **Second commit missing `Signed-off-by`** - moot: the history is now one squashed commit that
  carries it.

## Checked

- `go build` and `go test ./cmd/peribolos/ ./pkg/config/org/` pass at `9eb63ba11`.
- `gofmt -s -l` clean across `cmd/peribolos`, `pkg/config/org`, `pkg/github`.
- `nil` means "leave alone" and `true`/`false` means "enforce" for all four new settings, matching the
  existing `members_can_create_repositories` handling; no existing field changed semantics.
- `--dump` output for the four unconditional fields, plus the enterprise / non-enterprise / API-omits-it
  cases for the internal one, are asserted in `TestDumpOrgConfig`.
- The non-enterprise clearing path, the enterprise set path, and the both-nil no-op path all have
  direct test cases now.
- cblecker's review (`2026-11-04T02:33Z`, five inline comments) raised the same PATCH-echo, gofmt and
  nil-branch points, plus the plan-permission angle and the `members_can_create_repositories`
  interaction; each has an author reply and a code or documentation change except the last, which was
  answered with an experiment (the legacy flag is an aggregate alias that sets the public/private
  values).
- Adding fields to `github.Organization` also changes webhook payload structs (`GenericEvent.Org`, etc.)
  - harmless, purely additive decode targets.
- No golden/config fixtures elsewhere in the repo carry `members_can_create_repositories`, so replacing
  the test expectations is the only fixture churn needed.

## Open questions

- Worth one sentence in the docs next to `members_can_create_internal_repositories: false # enterprise
  only`: that the setting is skipped (with a warning) when the token cannot see the org plan, and that
  GitHub treats `members_can_create_repositories: false` as setting the public/private sub-settings to
  `false` too - the example now shows all four keys side by side with nothing stopping a contradictory
  combination.
