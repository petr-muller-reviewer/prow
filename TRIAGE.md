---
issue: kubernetes-sigs/prow#1029
title: "peribolos: support more org settings (repo creation by type, fork policy, commit signoff)"
state: open
labels: ""
main_sha: c78b4fb59529c0abc6e1c75eb8ed3675f58ce99e
triaged_at: 2026-10-28T00:45:59Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 2
recommended_labels: [kind/feature, area/peribolos, help wanted]
advice:
  advised_at: 2026-10-28T01:01:37Z
  based_on_triaged_at: 2026-10-28T00:45:59Z
---

## Initial validation

**Assessment**: LEGITIMATE

### Analysis

The author asks Peribolos to manage five GitHub organization settings through its declarative configuration: whether members can create public, private, or internal repositories; whether they can fork private organization repositories; and whether web-based commits require signoff. The issue gives the exact API setting names, explains the current workaround (`gh api`), and offers to contribute a PR. This is a specific, actionable feature request for a component maintained in this repository.

The open issue #737 is about excluding repositories from team-repository permission reconciliation. It is also Peribolos-related but has a different behavior and configuration surface, so it is not a duplicate. The issue timeline had no cross-references, and a PR search for `1029` returned no result.

**Issue Category**: Feature Request

**Repository Scope Check**:
- Component mentioned: Peribolos
- Exists in this repo: Yes
- Relevant code paths: `pkg/config/org/org.go`, `pkg/github/types.go`, `cmd/peribolos/main.go`, `cmd/peribolos/main_test.go`, `site/content/en/docs/components/cli-tools/peribolos.md`

**Information Completeness**:
- Sufficient detail provided: Yes
- Missing information: The author has not specified whether the YAML field names should match GitHub's API names exactly; the existing Peribolos convention uses those names, so this can be resolved consistently during implementation.

### Recommendation

Keep open and continue triage. The requested settings are organization-level fields on the GitHub organization API, and Peribolos already supports declaratively managing organization metadata. The requested work extends an existing capability rather than asking for general support or changes outside Prow.

**Suggested Action**:
- Keep open and continue triage.

## Research findings

### How it works today

Peribolos models organization settings as optional pointers in `org.Metadata` (`pkg/config/org/org.go:38-49`), while the GitHub API response/request type is `github.Organization` (`pkg/github/types.go:1216-1234`). `dumpOrgConfig` copies supported settings from `GetOrg` into the YAML config (`cmd/peribolos/main.go:228-245`). During apply, `configureOrgMeta` compares each non-nil desired field and calls `EditOrg` only when something changed (`cmd/peribolos/main.go:1090-1120`); `EditOrg` sends a `PATCH /orgs/{org}` with the organization struct (`pkg/github/client.go:1660-1679`).

Only `members_can_create_repositories` among the requested settings is represented in those paths today. The current Peribolos guide documents that field and says omitted YAML fields remain unmanaged (`site/content/en/docs/components/cli-tools/peribolos.md:25-33, 91`). GitHub documents the requested fields as inputs to its Update an organization endpoint. Internal repository creation is only available to orgs associated with GitHub Enterprise Cloud or GitHub Enterprise Server 2.20+, and web commit signoff applies to commits made through GitHub's web interface ([GitHub REST API: Update an organization](https://docs.github.com/en/rest/orgs/orgs?apiVersion=2022-11-28#update-an-organization)).

### Root cause

The remaining settings are absent from both the Peribolos YAML metadata struct and the GitHub organization model, so dump cannot emit them and apply cannot compare or send them. The lack is visible alongside the existing `MembersCanCreateRepositories` field in `pkg/config/org/org.go:45-49` and `pkg/github/types.go:1230-1234`. Confidence is high; this is a capability gap rather than a failure in an existing code path.

### Tests and docs

**Partial** — `TestConfigureOrgMeta` covers diffs for current organization settings, including the existing repository-creation bool (`cmd/peribolos/main_test.go:1882-2026`), and `TestDumpOrgConfig` verifies settings are copied into dumped config (`cmd/peribolos/main_test.go:2054-2149`). Neither covers the five requested API properties. The Peribolos guide has an organization settings example and unmanaged-field rule, but only lists the legacy repository-creation field (`site/content/en/docs/components/cli-tools/peribolos.md:24-34, 91`). GitHub's API supports these request fields; the internal-repository setting has the stated Enterprise eligibility constraint.

### Proposed approach

**Recommended**: Extend the existing organization metadata path — add the five optional YAML settings and corresponding GitHub organization API fields, then wire them through dump and apply. Preserve the current `members_can_create_repositories` behavior for backward compatibility. Since `EditOrg` sends the `github.Organization` value as its PATCH body, model the new API fields so absent response fields stay omitted rather than becoming an accidental `false`; that matters for GitHub versions or org plans where a setting may not be reported. Update the Peribolos example and unmanaged-field description.
- Trade-offs: Slightly more care is needed for optional API response fields than for a plain bool, particularly for internal repo creation; setting it to true is limited by GitHub's Enterprise eligibility.
- Backwards compatibility: Additive YAML fields remain unmanaged when omitted; retain the existing field and its behavior.
- Testing needed: Cover each setting's false/true reconciliation, no-change behavior, absent API values, dump output, and YAML serialization; document the GitHub web-only scope of commit signoff and internal-repo eligibility.

## Effort assessment

**Effort Level**: 2 - Moderate

The requested fields and the GitHub endpoint are identified, and Peribolos has an established dump/apply pattern. The change spans the YAML and API models, reconciliation and dump paths, tests, and documentation; the internal-repository setting also needs to preserve GitHub's Enterprise eligibility behavior.

- **Solution certainty**: clear — each requested setting is an organization API field, with an existing metadata pattern to extend.
- **Blast radius**: localized — Peribolos org metadata reconciliation and dump; existing optional-field behavior should remain intact.
- **Expertise**: Go structs and JSON/YAML serialization, plus GitHub organization setting semantics.
- **Scope estimate**: Medium — a few metadata/model/wiring changes, table-driven apply and dump coverage, and a docs update.

**Recommended labels**: `kind/feature` (new supported capability), `area/peribolos` (component ownership), `help wanted` (moderate contributor-sized work).

**Guidance for contributors**: Start with `org.Metadata`, `github.Organization`, `dumpOrgConfig`, and `configureOrgMeta`. Preserve omission semantics for unconfigured fields and verify what GitHub returns for the internal-repository setting on organizations without Enterprise eligibility; document that commit signoff covers web-based commits.

## Briefing summary

The briefing covered the issue's legitimacy as a Peribolos feature request, the missing YAML/API model fields that prevent dump and apply from handling the settings, and the existing metadata path to extend. The recommended approach keeps the five new settings optional, preserves the legacy repository-creation field, and tests dump/apply behavior and unsupported or absent API values. Effort was assessed as Level 2 (Moderate), with `kind/feature`, `area/peribolos`, and `help wanted` recommended.

Briefed maintainer on: 2026-10-28 00:45 UTC

Key questions asked:
- None; no substantive questions were raised during the briefing checkpoints.

Maintainer decision:
No implementation or label decision was stated; recommendation is to keep the issue open and invite a PR.

## Findings

- [related-issue] #737 (open): adds a proposed `--ignore-repos` option for team-repository reconciliation; it concerns a separate Peribolos feature and is not a duplicate.
- [cause] The five requested properties are not modeled in Peribolos org metadata or `github.Organization`, so current dump/apply code cannot serialize or reconcile them.
- [related-code] `pkg/config/org/org.go:38-49` and `pkg/github/types.go:1216-1234` define the current org metadata/API types; `cmd/peribolos/main.go:228-245, 1090-1120` contains the dump and apply paths; `pkg/github/client.go:1660-1679` sends the org PATCH.

## Checked

- Read issue #1029 and its empty comment list; the issue is open and has no labels or assignee.
- Checked the issue timeline for cross-references; none were returned.
- Searched repository issues for Peribolos organization settings, `members_can_create`, and commit signoff. No matching prior request appeared beyond #1029; #737 is a distinct Peribolos request.
- Searched PRs for `1029`; no result was returned.
- Reviewed GitHub's Update an organization docs for the requested API fields; internal repo creation requires an Enterprise-associated organization, and web commit signoff is limited to commits made through GitHub's web interface.

## Next steps

- Keep the issue open and invite a PR for the five settings through the metadata/API models, dump/apply paths, tests, and Peribolos guide.
- A maintainer can apply `kind/feature`, `area/peribolos`, and `help wanted` if desired.

## Advice

Current state is unchanged since triage: #1029 is open, unlabeled, unassigned, and has no milestone, comments, or cross-referenced PR/issues. The issue author offered to contribute a PR; no implementation is in flight in the issue timeline.

1. **Add the recommended labels.** The issue has no labels, and all three labels below exist in `kubernetes-sigs/prow`. `help wanted` fits the Level 2 effort; `good first issue` is not recommended.

   ```sh
   gh issue edit 1029 --repo kubernetes-sigs/prow --add-label "kind/feature,area/peribolos,help wanted"
   ```

2. **Invite the offered contribution with implementation scope.** This reiterates the triage guidance and highlights the API compatibility details.

   ```sh
   gh issue comment 1029 --repo kubernetes-sigs/prow --body "Thanks for offering to contribute this. A PR adding the five settings through Peribolos org metadata, dump, and apply would be welcome. Please preserve the existing members_can_create_repositories setting and leave omitted fields unmanaged; cover the Enterprise restriction for internal repository creation and that commit signoff applies to GitHub web commits in tests and docs."
   ```

No milestone or project assignment is indicated yet: the request names no release target or urgency. There is no linked PR to review or link.

## Open questions

- Confirm implementation behavior when GitHub does not return an optional setting on an organization or GitHub Enterprise Server version that does not support it.
