---
issue: kubernetes-sigs/prow#662
title: "Switch From logrus To slog"
state: open
labels: kind/cleanup, area/dependency, lifecycle/stale
main_sha: e601a1ffafd7d8d3a781238a4c5f4233d6248f68
triaged_at: 2026-07-27T00:35:40Z
verdict: accepted
---

## Verdict

Accepted as a legitimate, well-scoped cleanup proposal, but effort-rated Very Large (level 4): 376 files import `sirupsen/logrus`, and migration requires breaking changes to four public interfaces (`crier.ReportClient`, `WithFields(logrus.Fields)` on `github`/`bugzilla`/`jira`/`repoowners`, `plugins.Agent.Logger`) that are consumed by code outside this repo. Recommend tracking as an epic with sequenced sub-issues rather than a single PR.

## What the issue reports

- Proposes migrating Prow's logging from `sirupsen/logrus` (unmaintained upstream, 6 years without a major release) to stdlib `log/slog`.
- Not a bug report — a cleanup/dependency-modernization proposal.
- Already labeled `kind/cleanup`, `area/dependency`; picked up `lifecycle/stale` from k8s-triage-robot on 2026-06-27 due to inactivity.
- A prior triage comment (petr-muller, 2026-03-29) already scoped the problem in detail; this triage independently verified that scoping against current code and found it slightly understated.

## Findings

### [related-code] logrusutil package
- where: `pkg/logrusutil/logrusutil.go:37-155` (only source file; `logrusutil_test.go` alongside)
- excerpt: |
    ComponentInit()  // :60, sugar over Init(), sets DefaultFieldsFormatter{PrintLineNumber: true, DefaultFields: {"component": <binary name>}}; called from 37 cmd/* entry points
    Init()           // :45, configures logrus formatter/ReportCaller; wires controller-runtime logger via log.SetLogger(logrusr.New(logrus.StandardLogger())) at :56
    DefaultFieldsFormatter  // :37-86, renames "level" -> "severity" at :74-75 (comment: GCP log collection expects "severity"), injects component name
    CensoringFormatter      // :88-125, scrubs secrets from message and raw formatter output via secretutil.Censorer
    ThrottledWarnf          // :127-155, rate-limits logrus.Warnf via throttleCheck/throttleLock

### [related-code] crier.ReportClient interface
- where: `pkg/crier/controller.go:38-47`
- excerpt: |
    Report(ctx, log *logrus.Entry, pj) ...
    ShouldReport(ctx, log *logrus.Entry, pj) ...
- detail: implemented by 7 reporter packages: `pkg/crier/reporters/{pubsub,slack,gcs,gcs/kubernetes,gerrit,github,resultstore}/reporter.go`.

### [related-code] WithFields(logrus.Fields) interface methods
- where: `pkg/github/client.go:299` (interface), `:387` (impl); `pkg/bugzilla/client.go:97`/`:204`; `pkg/jira/jira.go:96`/`:642`; `pkg/repoowners/repoowners.go:148`/`:178`
- detail: four independent public interfaces all expose `logrus.Fields` directly in their method signatures.

### [related-code] plugins.Agent.Logger
- where: `pkg/plugins/plugins.go:209`
- excerpt: |
    type Agent struct {
        ...
        Logger *logrus.Entry
        ...
    }
- detail: highest blast-radius exposure — referenced in ~54 files under `pkg/plugins/` (`pc.Logger`, `.Logger.With(...)`), and consumed by plugin implementations that can live outside this repo. This is the sharpest edge of the whole migration.

### [related-code] bombsimon/logrusr/v4 bridge
- where: `go.mod:20` (v4.1.0), single usage at `pkg/logrusutil/logrusutil.go:25,56`
- detail: bridges logrus to logr for controller-runtime. Replaceable by `logr.FromSlogHandler()` (available in `go-logr/logr` since v1.4) once `Init()` no longer needs a logrus-backed `log.SetLogger` — an early, low-risk win that drops a dependency without touching the four public interfaces.

### [cause] why this is hard
- detail: not algorithmic difficulty — structural. The GCP `severity`-field rename and secret-censoring behavior must be preserved exactly, and the four public interfaces need coordinated, atomic conversion across every implementer/caller, including code outside this repo (plugin implementations). Effort is dominated by breaking-change surface area, not line count.

### [related-issue] prior triage comment
- ref: kubernetes-sigs/prow#662 (comment by petr-muller, 2026-03-29)
- relevance: did the initial scoping (~250 files, `ComponentInit` 31+ call sites, same four interfaces) that this triage verified against current code and found modestly larger (376 files, 37 call sites). Full detail at `https://github.com/petr-muller/prow/blob/issue-triage-662/ISSUE-TRIAGE.md`.

## Checked

- Verified logrus file-count and `ComponentInit()` call-site counts against current code: 376 files (`grep -rl "sirupsen/logrus"`), 37 call sites — both higher than the prior triage comment's ~250/31+.
- Confirmed `bombsimon/logrusr/v4` bridge is still in use, single call site, replaceable by `logr.FromSlogHandler()`.
- Searched for any existing partial slog migration (`grep -rl "log/slog"`) — none found; greenfield.
- Reviewed `pkg/logrusutil/logrusutil_test.go` — covers only `CensoringFormatter` (4 test functions); `DefaultFieldsFormatter` and `ThrottledWarnf` have zero test coverage today.
- Effort-assessed at level 4 (Very Large): scope (376 files), breaking changes to 4 public interfaces including one consumed by external plugin code, and required expertise in Prow's plugin architecture and production log-collection requirements all point to level 4 despite the underlying goal being architecturally uncontroversial.

## Next steps

- Comment on the issue proposing decomposition into sub-issues: (a) add missing `logrusutil` tests for `DefaultFieldsFormatter`/`ThrottledWarnf`, (b) replace the `logrusr` bridge with `logr.FromSlogHandler()`, (c) one sub-issue per public interface (`crier.ReportClient`, the four `WithFields` methods, `plugins.Agent.Logger`).
- Resolve the `logrus.Fields` -> slog-attrs-vs-shim design question explicitly (issue comment or short design note) before the bulk internal-file conversion starts.
- Loudly flag the `plugins.Agent.Logger` breaking change for anyone maintaining plugin implementations outside this repo.
- Consider removing/resetting `lifecycle/stale` once an epic/sub-issue plan is posted — the staleness reflects lack of scoping, not lack of validity.
- Recommended labels going forward: keep `kind/cleanup`, `area/dependency`; do not add `good-first-issue` or `help-wanted` as-is.

## Open questions

- Should `logrus.Fields`-shaped call sites convert to idiomatic slog `Attr`/`...any` pairs, or route through a compatibility shim? This determines how mechanical the internal-file conversion can be.
- Is there existing precedent/policy for handling breaking changes to interfaces consumed by external plugin implementations (deprecation window, major-version signal, etc.)?
