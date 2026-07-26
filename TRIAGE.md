---
issue: kubernetes-sigs/prow#585
title: "Remove unneeded prow plugins/cmds"
state: open
labels: lifecycle/rotten
main_sha: 351e8cfd58915657bd36a50e7e86bbe972bc0739
triaged_at: 2026-07-26T00:00:00Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 3
recommended_labels: [kind/cleanup, area/plugins, help-wanted]
refresh_log:
  - previous_triaged_at: 2026-06-16T15:48:18Z
    summary: "k8s-triage-robot escalated lifecycle/stale to lifecycle/rotten (2026-07-11) after continued inactivity; no other issue activity, no linked PRs, no relevant code movement on main."
---

## Findings

### [cause] Four plugins/commands are obsolete
- detail: golint uses deprecated `golang.org/x/lint`. buildifier uses unmaintained `bazelbuild/buildtools` pinned to 2020; Prow no longer uses Bazel. dco is superseded by GitHub-native DCO enforcement. jenkins-operator has no active users.
- evidence: go.mod deps, issue discussion by upodroid, BenTheElder, petr-muller.

### [related-code] golint AddedLines() coupling
- where: `pkg/plugins/golint/golint.go:340-377`
- excerpt: |
    func AddedLines(patch string) (map[int]int, error) {
- detail: General-purpose unified diff parser. Imported by `pkg/plugins/verify-owners/verify-owners.go:39,221`. Must relocate before removing golint.

### [related-code] dco MarkdownSHAList() coupling
- where: `pkg/plugins/dco/dco.go`
- excerpt: |
    func MarkdownSHAList(...)
- detail: Formats commit SHA lists as markdown. Imported by `pkg/plugins/invalidcommitmsg/invalidcommitmsg.go:32,158,166`. Must relocate or inline before removing dco.

### [related-code] golint plugin registration
- where: `cmd/hook/plugin-imports/plugin-imports.go:35`, `pkg/hook/plugin-imports/plugin-imports.go:35`
- detail: Blank import registers golint handler. Config struct `Golint` in `pkg/plugins/config.go:80,153-158`.

### [related-code] dco plugin registration
- where: `cmd/hook/plugin-imports/plugin-imports.go:33`, `pkg/hook/plugin-imports/plugin-imports.go:33`
- detail: Blank import registers dco handler. Config struct `Dco` in `pkg/plugins/config.go`.

### [related-code] buildifier plugin registration
- where: `cmd/hook/plugin-imports/plugin-imports.go:28`, `pkg/hook/plugin-imports/plugin-imports.go:28`
- detail: Blank import registers buildifier handler. No config struct. No other code depends on it.

### [related-code] jenkins-operator config integration
- where: `pkg/config/config.go:177,2563-2576`
- excerpt: |
    JenkinsOperators []JenkinsOperator
- detail: Config field and validation for jenkins-operator. Must be deprecated/removed.

### [related-code] jenkins ProwJob API types
- where: `pkg/apis/prowjobs/v1/types.go:91,192,1141`
- excerpt: |
    JenkinsAgent ProwJobAgent = "jenkins"
    JenkinsSpec  *JenkinsSpec
    JenkinsBuildID string
- detail: Agent type enum, spec struct, build ID field. API type changes need careful deprecation.

### [related-code] deprecation announcements page
- where: `site/content/en/docs/announcements.md`
- detail: Established pattern for documenting breaking changes with deprecation timelines. Precedent: requiresig, docs-no-retest (deprecated Jan 2020, removed April 2020), autobump.

### [related-code] runtime deprecation warning utility
- where: `pkg/logrusutil/logrusutil.go:127-128`
- detail: `ThrottledWarnf()` provides throttled warning logging, used by existing config deprecation warnings.

### [related-code] checkconfig warning framework
- where: `cmd/checkconfig/main.go:79-420`
- detail: Has `--warnings`/`--exclude-warning`/`--strict` flags. No plugin-specific deprecation checks yet, but the framework supports adding them.

## Checked

- All four components confirmed present in codebase at HEAD (351e8cfd5)
- No existing PRs address any of the four removals
- No sub-issues exist for individual components
- checkconfig has a warning framework but no plugin-specific deprecation checks
- Precedent for plugin removal exists (requiresig, docs-no-retest, autobump in announcements.md)
- Confirmed AddedLines() coupling: golint -> verify-owners
- Confirmed MarkdownSHAList() coupling: dco -> invalidcommitmsg
- jenkins-operator only imported by its own cmd/ entry point

## Next steps

- Remove `/lifecycle rotten` (escalated from `lifecycle/stale` by k8s-triage-robot on 2026-07-11) -- issue is valid and actionable, and will auto-close ~30d after rotten (i.e. around 2026-08-10) if left untouched
- Post comment summarizing per-component plan, ask for maintainer alignment on deprecation process
- Consider creating separate sub-issues per component for independent assignment
- Unblock buildifier removal immediately -- zero coupling, no controversy
- Apply labels: kind/cleanup, area/plugins, help-wanted

## Since previous triage (2026-06-16)

- k8s-triage-robot escalated the issue from `lifecycle/stale` to `lifecycle/rotten` on 2026-07-11 due to continued inactivity; no human comments, no linked PRs, and no relevant upstream code changes since the prior triage.

## Open questions

- Should linters (buildifier, golint) skip phased deprecation and be removed immediately (BenTheElder's position)?
- Should hook gracefully ignore removed plugin config (petr-muller) or hard-fail (BenTheElder)?
- Is GitHub native DCO enforcement sufficient to replace the dco plugin, given the retroactive-check difference?
- Is rajibmitra (volunteered January 2026) still interested in contributing?
