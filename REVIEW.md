---
pr: kubernetes-sigs/prow#774
title: "peribolos: add internal repository visibility support"
head_sha: 674a52169165592fc021626d57c951cc4654df3b
base: main
reviewed_at: 2026-10-19T10:51:42Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-10-19T10:52:18Z
  gated_head_sha: 674a52169165592fc021626d57c951cc4654df3b
  reviewed_head_sha: 674a52169165592fc021626d57c951cc4654df3b
---

# Review

## Gate

Merge — no unresolved substantive finding warrants a hold. The PR is already merged; this is a retrospective assessment of its final head.

The PR head is unchanged from the completed review. Paginated GitHub reviews and comment threads confirm that the YAML-path test, dry-run Private state, duplicated conversions, pointer helpers, and publication-flag help text were addressed. No changes-requested review remains to assess. The remaining older-GHES concern was explicitly accepted in discussion rather than fixed; no affected supported server version was identified. Local affected-package tests passed in the preceding review. @hoxhaeris also reported 15 end-to-end scenarios passing against an enterprise organization on 2026-07-15; that is reviewer-reported evidence from an earlier head, not a new final-head integration run.

### Findings disposition

- Borderline, accepted: @hoxhaeris raised absent API visibility in [the publication-guard thread](https://github.com/kubernetes-sigs/prow/pull/774#discussion_r3587727455). At cmd/peribolos/main.go:1098 an absent visibility still produces a delta, and the dump at lines 371-378 omits it. The author [explicitly declined an old-GHES fallback](https://github.com/kubernetes-sigs/prow/pull/774#discussion_r3788273750). Fine to merge with the compatibility caveat; no supported deployment failure was established.
- The local migration/documentation nit remains open and non-blocking.

### Independent merge risks

- Configuration/rollback: consumers of new dumps or visibility configs must upgrade before adoption; older binaries ignore visibility. Restore legacy private settings before rollback. Affects peribolos operators adopting the new field; legacy configs still parse on the new binary.
- Behavior: legacy private: true now means specifically private and can narrow an existing internal repository to private (pkg/config/org/org.go:107-111). Explicit private-to-internal transitions broaden enterprise access without an extra flag; transitions to public remain gated (cmd/peribolos/main.go:1126-1129).
- Go API: org.Repo.Private is replaced by Visibility, requiring external Go callers constructing org.Repo to migrate. Only peribolos imports this config package in-tree. github.RepoRequest.Private is retained, so the earlier claimed break in the general GitHub request API is resolved. No downstream caller failure was demonstrated; mention the config-type migration in release guidance.
- Legacy server APIs: no Private fallback exists for missing visibility or legacy create parameters. This is the accepted conditional limitation above, with no newly confirmed blast radius.
- No additional specialized compatibility skill applies to this diff; the gate used direct inspection of current code and prior review evidence.

### Gating list

None. Migration documentation is recommended, not required to merge.

## Verdict

Approve with suggestions — no required changes remain.

The defect-first review and three maintainer perspectives found no actionable defects in the final seven-file diff. Code quality is sound, maintenance burden and deployment risk are low, and tests passed for all three affected packages. Documenting visibility configuration, migration, access changes, and rollback is a non-blocking improvement recommended by both Maintainability and Deployment Risk. Older GHES compatibility remains an explicitly accepted limitation; no live GHES integration was tested. This assessment completes the re-review recommended on 2026-09-02. Local HEAD 63031cc5bea92e5a2d8d7a80a52529a1e54c2b43 is an artifact-only review commit atop the reviewed PR head recorded above.

## What this PR does

- Lets peribolos configure public, private, and enterprise-internal repository visibility.
- Converts legacy private booleans during config loading, warns about deprecation, and rejects conflicting or invalid visibility settings.
- Sends visibility through organization repository create/update requests while retaining Private for user repository creation.
- Requires --allow-repo-publish for transitions to public; permits private/internal transitions without that flag.
- Dumps non-public visibility and keeps dry-run Private state consistent with explicit visibility.

## Findings

### [nit] Document visibility migration, enterprise access, and rollback
- where: `pkg/config/org/org.go:101-111`; `cmd/peribolos/main.go:1126-1129`
- concern: Add a visibility: internal configuration example and explain that legacy private remains accepted but deprecated. Legacy private: true now reconciles internal repositories to private; private-to-internal intentionally broadens enterprise access without another flag. Earlier binaries ignore visibility, so rollback requires restoring legacy configuration. This documentation suggestion is non-blocking.
- excerpt: |
    v := github.RepoVisibilityPublic
    if *alias.Private {
        v = github.RepoVisibilityPrivate
    }
    alias.Visibility = &v

## Resolved

- API compatibility: the final RepoRequest retains Private alongside Visibility; the earlier downstream compile-break concern no longer applies.
- Shared type and helpers: RepoVisibility now lives in pkg/github, request fields use that type, and generic helpers replace duplicated comparison/default logic.
- Decoder complexity: UnmarshalJSON now uses a single embedded alias and legacy Private pointer; the raw-message second parse and dead null assignment are gone.
- Dry-run state: ToRepo derives Private from explicit Visibility, resolving the earlier inconsistent-state concern.
- Pointer helpers: tests use Go's new(expression) syntax instead of bespoke helpers.
- Older GHES behavior: the author explicitly declined a fallback for sufficiently old APIs in the [2026-08-15 review reply](https://github.com/kubernetes-sigs/prow/pull/774#discussion_r3788273750). Empty-visibility and unsupported-create-field scenarios remain compatibility limitations, not verified failures on an identified supported deployment.
- Private-to-internal policy: the PR explicitly intends this transition to be ungated and tests it. It is not an unintended regression.
- Prior suggestions about diagnostic context and deprecation sunset tracking were reassessed as non-actionable nits under this defect-first review.

## Checked

- Complete final diff: 674a52169165592fc021626d57c951cc4654df3b against its parent; seven files.
- YAML loading invokes Repo.UnmarshalJSON; legacy true/false mapping, null handling, conflicting fields, and invalid values are covered.
- RepoVisibility validates public/private/internal at JSON decoding boundaries.
- Repository updates omit unchanged visibility and preserve public-transition gating.
- Dump coverage preserves private/internal visibility and prunes public defaults.
- User repository creation still supports the Private request field; organization creation serializes Visibility.
- ToRepo applies visibility precedence and derives the matching Private boolean.
- go test ./pkg/config/org ./pkg/github ./cmd/peribolos — passed, exit status 0.
- No live GitHub Enterprise integration tests were run.
- Maintainer panel: Code Quality APPROVE; Maintainability APPROVE with LOW burden; Deployment Risk LOW. Final synthesis APPROVE_WITH_SUGGESTIONS.

## Open questions

None.

## Followups

Post-merge followup assessment for PR #774, merged as e78b8229bbcd26293b8303fc20fe0c054e903a30. Accepted: 0. Skipped: 2. No handoff prompts.

- Skipped — docs: document repository visibility, legacy private migration, enterprise access changes, publication safeguards, dump output, and rollback in site/content/en/docs/components/cli-tools/peribolos.md. Necessity: should. This is the existing non-blocking documentation suggestion; no work accepted.
- Skipped — compatibility: derive public/private from Private when API response visibility is empty in dump and update-comparison paths, with focused regression tests. Necessity: could. Older-GHES support was explicitly deferred, and no affected supported deployment was demonstrated; no work accepted.
