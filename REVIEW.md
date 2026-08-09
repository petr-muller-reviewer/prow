---
pr: kubernetes-sigs/prow#811
title: "chore(deps): bump the prometheus group across 1 directory with 2 updates"
head_sha: efa0dd249310217482638e798cb79487052c2097
base: main
reviewed_at: 2026-08-09T14:06:31Z
verdict: request-changes
refresh_log:
  - from: f8211615433853c3aa9472eb3df6be972b74b20a
    to: efa0dd249310217482638e798cb79487052c2097
    summary: "Branch rebased onto later main (picked up unrelated gopkg.in/ini.v1 bump merge, PR #814); no prometheus-bump content changed. Same lint failure persists on new SHA. New discussion: elmiko asked about the unrelated lint failures, petr-muller suggested a separate PR to fix lint then retest."
---

## Summary

Dependabot dep-bump PR. Only `go.mod`/`go.sum` in `.` and `hack/tools` changed — no project source touched. CI is currently red: `pull-prow-verify-lint` fails due to a `staticcheck` deprecation flag caused directly by this bump (see Findings). All other checks (build, integration, unit, race detector) pass.

Since previous review: the branch was rebased onto a later `main` (picking up the unrelated `gopkg.in/ini.v1` v1.67.0→v1.67.3 bump, PR #814, merged in the meantime) — the actual prometheus-group content is unchanged, and the same `pull-prow-verify-lint` failure persists on the new head SHA. `elmiko` asked in the PR thread (2026-08-07) whether the lint failures were related to this PR; `petr-muller` replied (2026-08-09) recommending a separate PR to fix the lint issues, merge it, then retest here.

Direct bumps (dependabot "prometheus" group):
- `github.com/prometheus/client_golang` v1.22.0 → v1.24.1 (main go.mod; hack/tools go.mod bumps v1.23.2 → v1.24.1, indirect there)
- `github.com/prometheus/common` v0.62.0 → v0.70.1 (main go.mod, direct; hack/tools indirect)

Remaining diff (`golang.org/x/net`, `x/sync`, `x/text`, `x/crypto`, `x/mod`, `x/sys`, `x/term`, `x/tools`, `google.golang.org/protobuf`, `prometheus/procfs`, `klauspost/compress`, `go.yaml.in/yaml/v2`) is indirect churn from `go mod tidy` following the group bump — all marked `// indirect`, none imported directly by project code.

## Dependency analysis

### github.com/prometheus/client_golang v1.22.0 → v1.24.1
- freshness: v1.24.1 released 2026-07-24, **5 days old** as of review — borderline per usual soak-time guideline. v1.24.0 (substantive release) is 9 days old (2026-07-20); v1.24.1 is a 1-commit bugfix on top (nil-URL panic fix in promhttp). Both proper tags, not pseudo-versions.
- usage: direct, heavy — imported in 40 files across nearly every component (deck, hook, tide, crier, ghproxy, gerrit, github client/ghmetrics, moonraker, sinker, plugins/updateconfig, bugzilla, jenkins, jira, pubsub, etc).
- changelog/exposure: min Go bumped to 1.25 (repo's go.mod already requires 1.25.8, fine); metric-name-validation default switched from `model.NameValidationScheme` global to always-UTF-8 (repo doesn't reference `NameValidationScheme`/`LegacyValidation` anywhere — unaffected); new opt-in `promhttp.HandlerOpts.CoalesceGather` and `name[]` filtering (not adopted here); panic-recovery added in `Gather()`; v1.24.1 fixes a promhttp panic on requests with nil URL. No CVEs. All touched code is metrics instrumentation, not a sensitive path (no auth/TLS/exec).
- take: content is safe (bugfixes/opt-in features, no breaking behavior for how prow uses it), but freshness is borderline and blast radius is large (40 files, every component's metrics). Reasonable to wait a few more days for soak.

### github.com/prometheus/common v0.62.0 → v0.70.1
- freshness: v0.70.1 released 2026-07-22, **7 days old**.
- usage: direct, light — single call site `pkg/metrics/push.go:31-32`, using `expfmt.NewEncoder`/`expfmt.NewFormat` (`pkg/metrics/push.go:67,89`) and `model`. Does not import `prometheus/common/config` (the HTTP client/transport package).
- changelog/exposure: v0.66.0 breaking change requires `expfmt.NewTextParser()` instead of zero-value `TextParser` — prow never constructs a `TextParser`, unaffected. v0.69.0 hardens `prometheus/common/config` against credential leakage on cross-host redirects — prow doesn't import that package, so no exposure either way. Various nil-pointer/allocation fixes in `expfmt`/`model` not exercised by prow's narrow usage (encoder + model only).
- take: safe — single call site using stable encoder/model APIs, no breaking changes hit prow's usage. Same soak-time caveat as client_golang (7 days old).

## Findings

### [blocking] pull-prow-verify-lint fails: staticcheck flags deprecated model.LabelName.IsValid
- where: `pkg/metrics/push.go:52`
- concern: `prometheus/common` v0.70.1 deprecates `model.LabelName.IsValid()` in favor of `ValidationScheme.IsValidLabelName()`. `pkg/metrics/push.go:52` — the only call site of `prometheus/common` in this project — still calls the deprecated method, and `staticcheck` (SA1019) fails the `pull-prow-verify-lint` job on it. This is CI-verified (job https://prow.k8s.io/view/gs/kubernetes-ci-logs/pr-logs/pull/kubernetes-sigs_prow/811/pull-prow-verify-lint/2081917474990723072), not a hypothetical from the changelog. The PR needs a follow-up commit switching to `ValidationScheme.IsValidLabelName` before it can merge.
- excerpt: |
    if !model.LabelName(ln).IsValid() {

### [nit] unrelated modernize lint findings bundled in the same failing check
- where: `cmd/checkconfig/main.go:1008`, `cmd/checkconfig/main.go:1545`, `cmd/deck/configured_jobs.go:83`, `cmd/deck/tide.go:193`
- concern: Four `modernize` (`stringscut`) findings suggesting `strings.Split(...)[0]` become `strings.Cut(...)` also show up in the same lint run. These look unrelated to the prometheus bump (no code in those files changed) and are more likely surfaced by a golangci-lint/Go toolchain version shift — but they're bundled into the same failing check and will need addressing (or suppressing) to get CI green regardless of cause.
- excerpt: |
    org := strings.SplitN(repo, "/", 2)[0]

## Checked
- Classification confirmed dep-only via `git diff --stat` against merge-base with `upstream/main` — no non-manifest files changed.
- `go.mod` `go 1.25.8` already satisfies client_golang v1.24.0's new Go 1.25 minimum.
- Grepped for `NameValidationScheme`/`LegacyValidation` (client_golang v1.24.0 behavior change) — no hits in project code.
- Grepped for `prometheus/common/config` import (v0.69.0 cross-host-redirect credential fix) — not imported; prow only uses `expfmt`/`model` from prometheus/common.
- Confirmed `expfmt.TextParser` (v0.66.0 breaking change) is never constructed directly in `pkg/metrics/push.go`.
- Ran `gh pr checks 811` and read the `pull-prow-verify-lint` build log (`gs://kubernetes-ci-logs/pr-logs/pull/kubernetes-sigs_prow/811/pull-prow-verify-lint/2081917474990723072/build-log.txt`) to confirm the CI failure and its root cause.
- Confirmed `pull-prow-image-build-test`, `pull-prow-integration`, `pull-prow-unit-test`, `pull-prow-unit-test-race-detector-nonblocking` all pass — the bump doesn't break tests or the build, only the lint gate.

## Open questions
- Both new versions are 5-7 days old (client_golang v1.24.1 released 2026-07-24, prometheus/common v0.70.1 released 2026-07-22). Given client_golang's 40-file blast radius, is it worth waiting a bit longer for soak time once the lint fix lands, or is dependabot's grouped cadence acceptable here?
- Are the 4 `modernize`/`stringscut` findings pre-existing (introduced by an unrelated toolchain bump landing around the same time) or new fallout from this PR specifically? Worth checking whether `main` already fails lint independently of this PR.
