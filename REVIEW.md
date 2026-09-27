---
pr: kubernetes-sigs/prow#952
title: "fix(site): make the docs link checker check links"
head_sha: eff2b2dd20c14a7a31c0e477074e992ed8fa337d
base: main
reviewed_at: 2026-09-27T15:42:41Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-09-27T15:39:50Z
  gated_head_sha: eff2b2dd20c14a7a31c0e477074e992ed8fa337d
  reviewed_head_sha: eff2b2dd20c14a7a31c0e477074e992ed8fa337d
---

# Review

## Gate

Merge. The PR head is unchanged since the saved review, no substantive reviewer objection is outstanding, and the diff affects documentation verification rather than deployed Prow behavior.

- Gating findings: none. The earlier concern about `verify-links` running an external scan was reconsidered during review; the full scan is the documented behavior of the manual command, and the internal-only target is available for CI.
- Reviewer feedback: Prucek approved on 2026-09-22; no inline review comments or changes-requested reviews were submitted.
- Merge risk: no exported API, CRD, runtime configuration, or deployed behavior changes. The altered htmltest configuration affects users who run the documentation link check; it is the intended behavior of this PR.

## Verdict

Approve.

The change makes htmltest scan the built site from its root, so absolute internal links are checked. I found no actionable defect in the PR diff. The full external scan in `verify-links` is an explicit choice for the manual command; the separate internal-only target is available for a future CI job.

## What this PR does

- Builds the documentation site and scans its generated tree once with htmltest.
- Resolves absolute links from `public` and removes broad internal-link ignores.
- Pins htmltest in `hack/tools` and exposes a top-level `verify-links` target.
- Adds an internal-only htmltest target and excludes generated site output from unrelated checks.

## Findings

No actionable findings.

## Checked

- Compared the PR head with its merge base and inspected the complete change and relevant call paths.
- `DirectoryPath: public` lets the checker resolve site-root links, and the old per-file wrapper is removed.
- `check-broken-links-internal` passes `-s` to htmltest; `verify-links` deliberately invokes the full scan and treats broken external links as warnings.
- The generated `site/public` and `site/resources` directories are excluded from the spelling and shell-permission checks.
- Dependency vet: `github.com/wjdp/htmltest v0.17.0` is the sole new direct module and is used only by the documentation verifier. Its [latest tagged release](https://github.com/wjdp/htmltest/releases/tag/v0.17.0) dates to 2022-11-04; the new options in that release are not used here.
- The five newly required indirect modules (`checkmail`, `docopt-go`, `mergo`, `govcr.v2`, and `yaml.v2`) are in htmltest's import chain. `docopt-go` is an untagged 2018 commit; the dependency surface is limited to the verifier binary, not deployed Prow packages.
- OSV returned no advisories for the six added module versions. `govulncheck -C hack/tools -scan symbol github.com/wjdp/htmltest` reported no vulnerabilities. There is no older version to compare because these modules are additions.
- The PR reports a passing site scan and `make verify`; I did not rerun the site build because Hugo is unavailable in this worktree environment.

## Open questions

- When CI integration is added, will it call `check-broken-links-internal` so external network failures do not affect the gate?

## Followups

### Add an internal-link presubmit

- category: CI
- necessity: should
- where: `site/Makefile:37-38`, `hack/make-rules/verify/links.sh:41-51`, and the Prow job configuration in `kubernetes/test-infra`
- why followup: PR #952 supplies the deterministic internal-only target but deliberately leaves CI integration until a suitable image with Hugo Extended, Node, and Go is chosen.

```text
In kubernetes-sigs/prow, following PR #952, "fix(site): make the docs link checker check links", add a presubmit that verifies links in the generated documentation site without requesting external URLs. Work from the merged default branch after #952 lands. The checker targets are in site/Makefile:33-38; the current local setup and build steps are in hack/make-rules/verify/links.sh:20-51. Locate the Prow job configuration for kubernetes-sigs/prow in kubernetes/test-infra and add the job there. Choose or build an image containing Hugo Extended, Node, and Go, build the site and the pinned htmltest binary, then run make -C site check-broken-links-internal. Validate the job configuration and demonstrate that a missing internal target fails while an unavailable external URL does not affect the presubmit. Keep the full external-link scan, periodic jobs, and documentation link repairs out of scope.
```

### Triage the unresolved external URLs after PR #954

- category: docs
- necessity: could
- where: `site/content/en/docs/overview/_index.md`, `private-deck.md`, `metadata-artifacts.md`, `components/cli-tools/generic-autobumper.md`, and `announcements.md`
- why followup: PR #954 already repairs the original 17 broken links reported alongside #952, but documents ten further URLs whose replacement or removal needs a maintainer decision.

```text
In kubernetes-sigs/prow, following PR #952, "fix(site): make the docs link checker check links", and the related PR #954, "docs: repair the dead links in the site content", resolve the ten additional external URLs listed in #954's "Links that need a maintainer decision" section. Work from the merged default branch after #954 lands so its 17 link repairs are not duplicated. Inspect site/content/en/docs/overview/_index.md (the Jetstack and KubeSphere adopter URLs), private-deck.md (two oss-prow.knative.dev example URLs), metadata-artifacts.md (two inaccessible kubernetes-jenkins artifacts and one expired demo), components/cli-tools/generic-autobumper.md (the oncall.json artifact and upstreamURLBase example), and announcements.md (one expired demo). Verify each URL and its context, then use a working replacement, remove a misleading link, or make an intentionally historical or access-restricted example clear to readers. Ask maintainers for a choice when no accurate replacement can be established. Acceptance: each of the ten URLs has an explicit disposition, all new destinations are checked, and the site link check shows no new broken links from these edits. Do not change htmltest policy, CI jobs, or the 17 repairs already covered by #954.
```
