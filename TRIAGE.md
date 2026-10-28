---
issue: kubernetes-sigs/prow#676
title: "ci: add plugin for Go dependency license check"
state: open
labels: lifecycle/stale
main_sha: 3617afe983fa4df61a494fc2a60300367372de3a
triaged_at: 2026-10-28T01:08:31Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 3
recommended_labels: [kind/feature, area/dependency]
advice:
  advised_at: 2026-10-28T01:15:18Z
  based_on_triaged_at: 2026-10-28T01:08:31Z
---

## Initial validation

**Assessment**: LEGITIMATE

### Analysis

This is a feature request for automated Go dependency license policy checks, prompted by an incompatible dependency entering `kubernetes-sigs/external-dns`. It is a concrete CI problem and the requested Prow integration is related to this repository's plugins and ProwJob machinery. The issue itself acknowledges uncertainty about whether an in-tree plugin is the right home. Maintainer discussion agrees that a webhook plugin is not an obvious fit for the license analysis itself and suggests a CI job or a separately maintained external plugin.

The issue is open and currently has only `lifecycle/stale`. There is no cross-reference to a Prow pull request or issue in its timeline. The issue links external-dns PR #5955 as the motivating dependency change and external-dns PR #6334 as a repository-specific CI implementation; the latter is merged. Those examples validate the underlying need but do not settle whether Prow should ship a general-purpose checker.

**Issue Category**: Feature Request

**Repository Scope Check**:
- Component mentioned: Hook plugins / CI integration
- Exists in this repo: Yes — Prow has internal and external webhook plugins, and ProwJob orchestration.
- Relevant code paths: `pkg/hook/server.go`; `site/content/en/docs/components/plugins/_index.md`; `site/content/en/docs/life-of-a-prow-job.md`

**Information Completeness**:
- Sufficient detail provided: Partially. The desired check (Go module dependency license policy) and a concrete motivation are identified.
- Missing information: Whether the request is for a reusable first-party Prow feature or a job for specific repositories; policy ownership and configuration; treatment of unknown licenses, indirect/test/tool dependencies, and multiple `go.mod` files; and expected behavior when the check cannot classify a license.

### Recommendation

Keep open for maintainer direction on ownership and scope. The use case is legitimate, but the repository's existing source-header check is distinct from dependency-license analysis, and the proposed implementation location is unsettled. The merged external-dns workflow is a useful repository-specific precedent, not evidence that a generic Prow plugin should own license policy.

**Suggested Action**:
- Keep open and continue triage; ask whether the requested outcome is a reusable Prow integration or guidance/job configuration for individual repositories.

## Research findings

### How it works today

Prow plugins consume GitHub webhook events. The plugin documentation describes them as Hook sub-components, configurable per repository (`site/content/en/docs/components/plugins/_index.md:8-11`). Internal pull-request handlers are invoked from Hook's event path (`pkg/hook/events.go:164-193`); external plugins receive forwarded webhook payloads (`site/content/en/docs/components/plugins/_index.md:28-38`, `pkg/hook/server.go:223-249`). The docs call out long-running operations and isolating plugins that execute PR code as use cases for external plugins (`site/content/en/docs/components/plugins/_index.md:31-36`).

ProwJobs are a separate CI execution path: trigger selects configured presubmit jobs, creates a ProwJob, and prow-controller-manager runs it as a pod (`site/content/en/docs/life-of-a-prow-job.md:53-57,87-91`). Prow's repository contains three Go modules (`go.mod`, `hack/tools/go.mod`, `site/go.mod`). Its current boilerplate verifier checks source-file copyright headers (`hack/boilerplate/verify_boilerplate.py:17`), and the dependency verifier regenerates/synchronizes Go dependencies (`hack/make-rules/verify/go-deps.sh:58-85`); neither checks dependency-license policy.

### Root cause

The reported gap is in repositories' CI policy: changing a Go module dependency does not automatically evaluate that dependency's license against a project-owned allowlist. Prow's header boilerplate check covers a different concern, and the Prow repository has no dependency-license scanner or policy configuration. Confidence is high about the distinction and current code paths; the issue does not define whether it seeks a reusable Prow product feature or help with the external-dns use case. The reporter's external-dns PR #6334 is now merged and adds `.github/workflows/license-check.yml` plus `.licenserc.yaml`, so it addresses that repository's example.

### Tests and docs

Coverage for dependency-license policy is missing in this repository. Existing verification covers source boilerplate and dependency-file consistency, not license identification or allowlist enforcement. Prow's plugin docs describe webhook integration and recommend isolating plugins that execute PR code; the ProwJob lifecycle docs describe running configured checks in pods. A new check would need coverage for allowed, disallowed, and unclassified licenses, as well as the intended module-file set (this checkout has root, `hack/tools`, and `site` modules).

### Proposed approach

**Recommended**: Repository-configured ProwJob check — run an established Go license scanner as a presubmit job, with the allowlist owned alongside each repository's CI policy. For Kubernetes-wide reuse, provide job configuration or a documented template in the CI configuration repository (`kubernetes/test-infra`), rather than placing dependency classification and policy inside Hook. This follows Prow's existing job execution model and keeps the check isolated with the PR checkout.
- Trade-offs: Each repository must opt in and maintain its policy; a shared template can reduce setup duplication but still needs clear handling of unknown licenses and tool maintenance.
- Backwards compatibility: Additive when enabled per repository; turning it on can intentionally fail PRs that introduce disallowed or unclassified licenses.
- Testing needed: Validate allowed/disallowed/unknown results, all intended Go module files, and that the ProwJob reports a failing check on violations.

**Alternative**: External plugin — a separately deployed service could receive PR webhooks and coordinate a standardized check. This may suit centralized policy and reporting, but adds a service to operate and needs careful isolation because it examines untrusted PR content; Hook's docs explicitly identify PR-code isolation as an external-plugin use case.

## Effort assessment

**Effort Level**: 3 - Large

The request is not yet a bounded implementation: maintainers need to choose whether Prow should own a reusable scanner integration or whether this belongs in repository-level ProwJob configuration, and define who owns the allowlist and unknown-license behavior.

- **Solution certainty**: significant uncertainty — the need is clear, but the product boundary, policy model, and treatment of multiple module files are unresolved.
- **Blast radius**: multiple components — a first-party integration could affect Hook/plugin configuration and external service or ProwJob execution; reusable CI guidance would also involve `kubernetes/test-infra`.
- **Expertise**: Prow plugin and ProwJob architecture, license-scanner behavior, and safe handling of untrusted PR contents.
- **Scope estimate**: Medium — design decision plus either a reusable CI job/template and documentation, or a service/plugin with configuration, tests, and deployment guidance.

**Recommended labels**: `kind/feature` (new CI capability), `area/dependency` (Go dependency policy). No priority or contributor-level label is recommended until scope is agreed.

**Guidance for contributors**: Read the Hook/external-plugin docs and ProwJob lifecycle first. Get maintainer agreement on the execution boundary and policy inputs before implementing a scanner or adding a Prow dependency. If the scope is narrowed to a repository-level job using an already selected scanner and policy, reassess the effort level.

## Briefing summary

Briefed the maintainer on the request and its status, the legitimate but unresolved product boundary, the absence of dependency-license checks in Prow's current verification, the ProwJob and external-plugin options, and the resulting level 3 effort assessment. Recommended keeping the request open until maintainers decide whether to build a reusable Prow integration; the external-dns example is already addressed by its merged repository workflow.

Briefed maintainer on: 2026-10-28T01:08:31Z

Key questions asked:
- No technical questions were asked; the maintainer chose to continue through each briefing checkpoint.

Maintainer decision:
No implementation or disposition decision indicated.

## Findings

- [related-issue] `kubernetes-sigs/external-dns#5955` — cited by the reporter as an incompatible dependency merged without a license check; it is the motivating example, not a duplicate.
- [related-pr] `kubernetes-sigs/external-dns#6334` — the repository-specific dependency license check workflow linked by the reporter; merged, but it does not implement a Prow feature.
- [related-code] `site/content/en/docs/components/plugins/_index.md:8-38`, `pkg/hook/events.go:164-193`, `pkg/hook/server.go:223-249` — internal plugins handle Hook webhooks; external plugins receive forwarded payloads and are suited to isolated/slow work.
- [related-code] `site/content/en/docs/life-of-a-prow-job.md:53-57,87-91` — configured presubmit jobs execute as ProwJobs in pods, providing the closest existing CI mechanism.
- [cause] `hack/boilerplate/verify_boilerplate.py:17`, `hack/make-rules/verify/go-deps.sh:58-85` — current verification checks source headers and dependency-file consistency, not dependency licenses.

## Checked

- Read issue #676 and its discussion; confirmed the issue is open, labeled `lifecycle/stale`, and has no Prow issue/PR cross-references.
- Searched Prow issues for `go license` and `dependency license`; #777 from the first query is unrelated, and the second query returned only #676.
- Confirmed Prow's existing `verify-boilerplate`/header checking is separate from checking third-party dependency licenses.
- Confirmed the external-dns PR #6334 is merged and only changes that repository's `.github/workflows/license-check.yml` and `.licenserc.yaml`.

## Next steps

- Ask maintainers to decide whether a reusable Prow implementation is desired; if so, define whether it should be a ProwJob pattern, external plugin, or in-tree plugin.
- If the desired outcome is only for external-dns, track the already merged external-dns workflow there instead of adding a generic Prow plugin.
- Consider `kind/feature` and `area/dependency` after scope is confirmed; remove `lifecycle/stale` only if active work resumes.

## Advice

No change since triage: issue #676 remains open with `lifecycle/stale`; it has no assignees or milestone, and there are no new comments or Prow cross-references. No Prow PR is in flight. The cited external-dns workflow remains merged.

1. **Add the feature and area labels.** These labels are available and are not currently applied; no priority or contributor-level label is justified while the ownership and scope remain unsettled.

   ```sh
   gh issue edit 676 --repo kubernetes-sigs/prow --add-label "kind/feature,area/dependency"
   ```

2. **Ask whether the reusable request is still needed.** The only concrete use case cited was external-dns, whose workflow has since merged. If a shared Prow capability is still wanted, the issue needs a target implementation home and an owner for license policy.

   ```sh
   gh issue comment 676 --repo kubernetes-sigs/prow --body "$(cat <<'EOF'
   > *This was generated by AI during triage.*

   The external-dns workflow you cited in #6334 has since merged. Does that resolve your motivating use case, or are you still requesting reusable license-check support for Kubernetes repositories? If shared support is still wanted, would you prefer a ProwJob template in kubernetes/test-infra or a centrally operated plugin, and who should own the allowlist and unknown-license policy?
   EOF
   )"
   ```

   Leave `lifecycle/stale` in place unless discussion or active work resumes. No milestone or assignee is recommended until maintainers agree on scope.

## Open questions

- Should Prow own a reusable license-check integration, or should this remain a repository-specific CI job/tooling concern?
- What license allowlist, policy owner, module/workspace coverage, and unknown-license failure behavior should the check use?
- If a shared implementation is wanted, should it be a ProwJob template in `kubernetes/test-infra` or a Prow external plugin with a separately operated service?
