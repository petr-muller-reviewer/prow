---
pr: kubernetes-sigs/prow#1037
title: "peribolos: support org-level repo defaults"
head_sha: 2930af9fa960fe74d13efd67837b3191f9abedc7
base: main
reviewed_at: 2026-11-07T22:19:44Z
verdict: approve
refresh_log:
  - at: 2026-11-07T22:19:44Z
    old_sha: 80e1e26595f5c055bb0ef8ee42b9f5b998649060
    new_sha: 2930af9fa960fe74d13efd67837b3191f9abedc7
    summary: "Incorporated 2930af9fa: reverts the Go 1.27 requirement, nested-literal flattening, and unrelated Jira/rifle/lint/module changes; no repository reconciliation behavior change."
gate:
  decision: merge
  gated_at: 2026-11-07T22:25:29Z
  gated_head_sha: 2930af9fa960fe74d13efd67837b3191f9abedc7
  reviewed_head_sha: 2930af9fa960fe74d13efd67837b3191f9abedc7
---

# Review

## Gate

**Verdict: merge** at `2930af9fa960fe74d13efd67837b3191f9abedc7`.

Re-gated at the same head, which is now also the refresh baseline: no commits or GitHub activity since the previous gate. Every substantive reviewer concern is resolved in the current code — Go 1.26.4 retained, nested `RepoSettings` literals, unrelated Jira/rifle/lint/module changes dropped, archived and squash behavior verified and accepted by the reviewer. The one remaining change is a documented source-compatibility caveat for Go importers; the deployed YAML format is unchanged. No unacceptable independent merge risk.

### Findings disposition

- **Addressed — Go minimum and unrelated API changes**, raised by @cblecker on 2026-11-05T21:14: verified `go.mod:3` is `go 1.26.4`; `cmd/peribolos/main.go:465-483` uses nested `RepoSettings` literals; the full diff against merge base `091a7d73` touches 8 files and includes no `pkg/jira`, `pkg/plugins/rifle` or `.golangci.yml` changes.
- **Addressed — archived and squash behavior**, raised by @cblecker on 2026-11-03/04: the reviewer said these "look good to me" on 2026-11-05T21:14. Current code ignores defaults for archived repos with unset archival (`cmd/peribolos/main.go:1494-1503`), validates effective squash pairs on create and update (`1476`, `1506`), and omits squash settings when disabled (`1295`, `1392`).
- **Acceptably dispositioned — keyed `org.Repo` literals**, raised by @cblecker on 2026-11-04: `pkg/config/org/org.go:63-64` embeds `RepoSettings`, so Go 1.26 importers using promoted fields in keyed literals must nest them. The PR description gives the exact migration and states the minimum Go version and flat YAML format are unchanged. Source compatibility caveat only, not an unresolved blocker.
- **Addressed — compatibility claim and docs**, raised by @cblecker: the description now states that effective-pair validation and the `Defined()` fix affect all configs, and `peribolos.md:107-124` documents scope, overrides, dump limits and deployment ordering.
- The two local maintenance suggestions below are optional and do not gate merge.

### Independent merge risk

- **Go source API:** consumers constructing `org.Repo` with keyed defaultable fields on Go 1.26 must nest those fields in `RepoSettings`. This affects source importers, not prebuilt deployments; migration is documented in the PR description. Include it in release notes when publishing.
- **Existing configuration behavior:** `pkg/github/types.go:491-495` activates squash-only updates. Invalid effective combinations return an error at `cmd/peribolos/main.go:1506-1509` and skip that repository's remaining delta. This affects existing configs even without defaults, as the description states. Audit these settings on upgrade; this is an intentional validation/fix, not a merge blocker.
- **Opt-in rollout:** defaults can update every listed repository, and null does not opt a repository out. Deploy the binary before migrating settings; retain explicit configuration for rollback. `peribolos.md:124` documents the ordering.
- No new credentials, permissions, external endpoints, or background workers. No additional repository-specific compatibility skill applied.

### Gating list

None. No unresolved blocking or substantive should-fix feedback remains.

### Evidence and limits

Read all paginated PR reviews, inline comments and issue comments again at this re-gate; the newest is the author's 2026-11-05T22:23 report that Go 1.26.4 and the nested literals are restored. PR head is unchanged at `2930af9fa` since the previous gate, so the review delta is empty; inspected the full PR diff against merge base `091a7d73761d32baf5b1313f9a517d8c222fc5aa` (8 files, +1248/−101) and the current code through `git show`. The local checkout's files remain at `80e1e2659`; tests were not re-run at `2930af9fa` for this gate (the author reports race and JSON v2 tests pass on Go 1.26.4), and no live GitHub write was performed.

## Verdict

Approve with suggestions. No merge-blocking code findings.

The three-perspective maintainer review of `80e1e26595f5c055bb0ef8ee42b9f5b998649060` found sound code quality and low maintenance burden. Deployment risk is moderate because squash-only settings now take effect for existing configurations and defaults can update many listed repositories. Race and JSON v2 compatibility tests passed at that head. The review recommended an upgrade note and narrowing the compatibility claim, with optional completeness-test and diagnostic improvements; the gate above separately evaluates the newer head and the author's subsequent changes.

## What this PR does

- Shares nine repository settings through an optional org-level `repo_defaults` block; listed repositories inherit unset values and retain explicit overrides, including `false`.
- Embeds the shared settings in `org.Repo`, preserves flat JSON/YAML and legacy `private` parsing, and rejects unsupported default keys.
- Validates effective squash title/message pairs before repository writes, omits these settings when squash merging is disabled, and recognizes squash-only update requests.
- Ignores inherited defaults on archived repositories with unset archival while continuing to check explicit per-repository settings.
- Requires Go 1.26.4 and uses nested `RepoSettings` literals; the unrelated Jira, rifle, lint and module changes from earlier revisions were reverted.

Since previous review (`80e1e2659` → `2930af9fa`):

- Reverted `go.mod` to Go 1.26.4 and removed the dependency churn, restoring nested `RepoSettings` literals in `cmd/peribolos/main.go:465-483` and the dump tests.
- Reverted the unrelated `pkg/jira/jira.go`, `pkg/plugins/rifle/blame.go` and `.golangci.yml` changes, so the PR no longer touches those areas.
- No changes to inheritance, archived-repository handling, squash validation, or repository request serialization; both optional nits below still apply.

## Findings

These are optional maintainer suggestions, not correctness defects or merge blockers.

### [nit] Detect omissions when future settings are added

- where: `pkg/config/org/org.go:146-178`
- concern: The current inheritance list covers every field. Optionally add a test that discovers `RepoSettings` fields so a future addition cannot be omitted from both the inheritance implementation and its handwritten fixtures.
- excerpt: |
    func (r Repo) WithDefaults(defaults *RepoDefaults) Repo {
        if defaults == nil {
            return r
        }
        if r.AllowSquashMerge == nil {
            r.AllowSquashMerge = defaults.AllowSquashMerge
        }

### [nit] Include the owning org in default-decoding errors

- where: `pkg/config/org/org.go:133-139`
- concern: Errors identify the defaults block and unsupported key, but a configuration with many orgs still requires locating the relevant block. Consider attaching the org name or configuration location when decoding.
- excerpt: |
    func (r *RepoDefaults) UnmarshalJSON(data []byte) error {
        var settings RepoSettings
        decoder := json.NewDecoder(bytes.NewReader(data))
        decoder.DisallowUnknownFields()
        if err := decoder.Decode(&settings); err != nil {
            return fmt.Errorf("repo_defaults: %w", err)
        }

## Resolved

These entries preserve earlier review history. The gate above records subsequent changes at the current PR head.

- **Create-path squash message without a compatible title:** `configureRepos` now validates against the default title/message pair before creation. Incompatible message-only creation is rejected locally with repository context; updates use current values for unspecified fields. Tests cover both paths.
- **Archived-repository skip and missing documentation:** when `archived` is unset, the delta now uses the explicit repository config without inherited defaults. Description and rename tests preserve the existing error behavior for archived repositories, and the documentation explains the distinction.
- **Squash settings while squash merging is disabled:** create and update requests omit both squash fields when squash merging will be off, preserving other requested changes.
- **Empty-title update fallback:** the title backfill was removed. Validation uses the current pair without adding an empty title to the request.
- **Unvalidated squash enum values:** effective values are now checked before create/update requests. Invalid or empty values produce local errors instead of reaching the GitHub write call.
- **Field-list completeness guard:** no present defect was demonstrated. The maintainer review retains a future-proofing test only as the optional nit above.
- **Decode errors without org context:** no correctness regression was found; the maintainer review retains additional context only as the optional diagnostic nit above.
- **Naming and pointer aliasing:** no unsafe mutation through inherited pointers was found in the call paths reviewed. The earlier naming and hypothetical mutation concerns are not retained as defects.

## Checked

- Independent Code Quality and Maintainability reviewers both recommended approval. The Deployment Risk reviewer rated risk medium and found no deployment blocker. No concern was independently raised by multiple reviewers.
- Current-head gate checked all substantive GitHub feedback against fetched commit `2930af9fa`; the Go 1.27 requirement and unrelated Jira/rifle changes described in the historical feature summary above have been removed. The PR description now explains the keyed-literal migration and the squash fixes' applicability to all configs.
- Compared all 12 changed paths against merge base `091a7d73761d32baf5b1313f9a517d8c222fc5aa`, using `upstream/main`, which is ahead of local `main`.
- `git rev-parse HEAD` returned `d6e6b92f6e8bd91e1a2ae6c70bea7e5853cec956`, a local review-artifact commit whose only changed files are `REVIEW.md` and `REVIEW.html`. Its parent `80e1e26595f5c055bb0ef8ee42b9f5b998649060` is the reviewed PR head, also confirmed through read-only GitHub metadata. The frontmatter records that PR head so later refreshes compare code revisions correctly.
- Inheritance covers nil/empty defaults, explicit true/false, null inheritance, supported fields, source-config immutability, and unlisted repositories.
- JSON/YAML round trips remain flat; legacy `private` parsing and permissive per-repository decoding are preserved, while default keys are checked strictly.
- Squash pair validation covers create/update paths, inherited and overridden combinations, partial updates, invalid and empty values, enablement, and disabled-setting omission. `RepoRequest.Defined()` recognizes either squash field.
- Archived-repository coverage includes inherited defaults, explicit description changes, previous-name renames, and matching explicit settings.
- Reviewed repository request serialization, create/update call sites, Jira error construction and type handling, and rifle directory-selection behavior.
- Passed with Go 1.27 selected through `GOTOOLCHAIN=auto`: `go test -race ./pkg/config/org ./cmd/peribolos ./pkg/github`.
- Passed with `GOTOOLCHAIN=auto GOEXPERIMENT=jsonv2`: `go test ./pkg/config/... ./pkg/github/... ./cmd/peribolos ./pkg/jira ./pkg/plugins/rifle`.
- GitHub write behavior was evaluated through code and test fakes; no live API mutation or full end-to-end GitHub test was performed. External downstream importers were not built.
- Refresh at `2930af9fa` inspected `80e1e2659..2930af9fa` (1 commit, 8 files, +195/-170 lines) and found no new GitHub comments, inline comments or reviews since `2026-11-06T10:08:27Z`. The delta is literal restructuring plus reverts; tests were not re-run at this head.

## Open questions

None.
