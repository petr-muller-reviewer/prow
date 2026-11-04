---
pr: kubernetes-sigs/prow#1046
title: "chore(deps): bump github.com/prometheus/common from 0.72.1 to 0.73.0 in the prometheus group across 1 directory"
head_sha: d8d8d87880d327f8af7371e202405663ab2490b0
base: main
reviewed_at: 2026-11-04T17:42:20Z
verdict: approve
---

# Review

## Verdict

Safe to take: the bump touches no code path we exercise beyond a bug-fix in Accept negotiation, and carries no advisories. Dep-only bump of `github.com/prometheus/common` v0.72.1 → v0.73.0 (`go.mod:52`, plus the `hack/tools` module where it is indirect). The three upstream commits are all `expfmt`: OpenMetrics 2.0 escaping defaults, a new `FmtOpenMetrics_2_0_0_rc` constant, and a new ABNF-based OpenMetrics 1.0 parser that pulls in `pandatix/go-abnf`. Prow uses `expfmt` only to *encode* delimited protobuf in the Pushgateway helper, and the `model` package (whose `UTF8Validation` we do call) is unchanged, so the encoder path is untouched. The one changed function on a live path — `expfmt.NegotiateAccept`, reached from `promhttp.HandlerFor` on every `/metrics` scrape — only differs for OpenMetrics formats, which prow does not advertise; for proto/text it is a leak fix with identical output. Only cosmetic follow-up: `v0.73.1` already exists and drops the stray `hashicorp/go-uuid` that go-abnf 0.5.1 drags in.

## What this PR does

- Bumps the direct `github.com/prometheus/common` requirement from v0.72.1 to v0.73.0 in the root module.
- Bumps the same module (indirect there) in the separate `hack/tools` module, keeping its lockfile in sync.
- Adds two new indirect modules to both module graphs: `github.com/pandatix/go-abnf v0.5.1` and, via go-abnf, `github.com/hashicorp/go-uuid v1.0.3`.
- Touches nothing else: `git diff --stat` is `go.mod`, `go.sum`, `hack/tools/go.mod`, `hack/tools/go.sum`, 16 insertions / 6 deletions.

## Findings

### [nit] Bump already superseded by v0.73.1, which removes the stray go-uuid dependency
- where: `go.mod:52`, `go.sum:381-382`
- concern: `go-abnf` v0.5.1 imports `hashicorp/go-uuid` (in `transition_graph.go`), so a 2022-era module with no security relevance to us enters the graph of every prow binary. Upstream common v0.73.1 (2026-10-30) bumps go-abnf to 0.5.3, which drops go-uuid entirely; its changelog contains nothing else. A follow-up bump would shrink the dependency graph for free.
- excerpt: |
    github.com/prometheus/common v0.73.0
    ...
    github.com/hashicorp/go-uuid v1.0.3 h1:2gKiV6YVmrJ1i2CKKa9obLvRieoRGviZFL26PcT/Co8=

### [nit] New small-maintainer parsing dependency is linked into all binaries
- where: `go.sum:449-450`
- concern: `expfmt/openmetrics_parse.go` (new in v0.73.0) imports `github.com/pandatix/go-abnf`, so this single-maintainer module (~16 stars, created 2022, MIT) is compiled into every prow binary — verified with `go list -deps ./cmd/hook`. It is never executed, since nothing in prow decodes OpenMetrics, so this is a supply-chain surface note rather than a functional risk.
- excerpt: |
    github.com/pandatix/go-abnf v0.5.1 h1:iyLvSu2Aq12iE0r4n948Qw21BHDuckLq/8WIXAAF+HI=

## Checked

- Classification: dep-only. All four changed files are manifests/lockfiles; no project source, config, or docs touched.
- Freshness: v0.73.0 tagged 2026-10-27T07:36Z, published 2026-10-28 — 8 days old at review time. Real tag, not a pseudo-version. No open issues in `prometheus/common` referencing 0.73.
- Advisories: OSV has no entries for `github.com/prometheus/common` at either v0.72.1 or v0.73.0, so nothing is fixed or inherited.
- `govulncheck -scan module`: identical at base (`c06373a3e`) and head — 4 findings, none in `prometheus/*`: `GO-2023-1901` (tektoncd/pipeline, via `cmd/pipeline`), `GO-2026-5932` (x/crypto, via go-git), `GO-2026-6860`/`GO-2026-6866` (x/net/html, via `pkg/spyglass/lenses/html`). Pre-existing and unrelated to this bump.
- Scoped symbol-level scan of the three importing packages at head: `x/net/html` is imported but no vulnerable symbol is called (`function: null`); x/crypto is module-level only. The process was OOM-killed (exit 137) after emitting findings, so treat reachability as indicative. No base-side symbol scan; the module-level findings are byte-identical on both sides and involve no module that this PR changes.
- Import surface: only `pkg/metrics/push.go` imports common — `expfmt.NewEncoder`/`expfmt.NewFormat(expfmt.TypeProtoDelim)` at `push.go:67` and `push.go:89`, `model.UTF8Validation.IsValidLabelName` at `push.go:52`. Everything else in `pkg/metrics` reaches common indirectly via client_golang.
- Encoder path: `expfmt.NewEncoder`'s `TypeOpenMetrics` branch and `init()` registration table changed; the `TypeProtoDelim` branch, which prow uses, is untouched. The `model` package has no changes in this range, so `UTF8Validation.IsValidLabelName` is unchanged.
- `expfmt.NegotiateAccept` is live via `promhttp.HandlerFor(..., promhttp.HandlerOpts{})` at `pkg/metrics/metrics.go:64`, but prow advertises `DefaultAcceptedFormats()` (proto delimited/text/compact; `EnableOpenMetrics` is false), and those lists are unchanged. The escaping change only applies when the matched format is OpenMetrics 2.0, which is never offered; for the formats prow offers the change is the fix for a cross-clause escaping-parameter leak, with identical output.
- Build and test: `GOEXPERIMENT=jsonv2 go build ./pkg/metrics/...` and `GOEXPERIMENT=jsonv2 go test ./pkg/metrics/...` both pass. (`GOEXPERIMENT=jsonv2` is a pre-existing repo requirement from `pkg/config/zz_generated.deepcopy.go`, not caused by this bump.)
- `go mod tidy -diff` reports no diff in either the root or `hack/tools` module, and go.sum verification succeeded during the build.
- Toolchain: common's `go` directive is `1.26.0` in both versions; prow has `go 1.26.4`. No toolchain movement.

## Open questions

- Would you prefer to roll this straight to v0.73.1 rather than land v0.73.0 and then chase it? The only delta upstream is go-abnf 0.5.1 → 0.5.3, which removes `hashicorp/go-uuid` from the graph.

## Dependency followups

Examined `github.com/prometheus/common` v0.72.1 → v0.73.0 (direct), plus the transitively added `github.com/pandatix/go-abnf` v0.5.1 and `github.com/hashicorp/go-uuid` v1.0.3. No deprecations apply: our only call sites already use the current forms (`model.UTF8Validation.IsValidLabelName` at `pkg/metrics/push.go:52`, `expfmt.NewEncoder`/`expfmt.NewFormat` at `push.go:67` and `push.go:89`), and the deprecated `Negotiate`/`NegotiateIncludingOpenMetrics` helpers and `model` legacy-validation functions are not called from prow. The new ABNF-based OpenMetrics decoder and the OpenMetrics 2.0 format constants have no consumer in this repo. One opportunity recorded.

- **simplification: bump prometheus/common to v0.73.1 to drop hashicorp/go-uuid** — module: `github.com/prometheus/common` v0.73.0 → v0.73.1 — where: `go.mod:52`, `hack/tools/go.mod:269`, `go.sum:381-382` — necessity: could.

```
In kubernetes-sigs/prow, now that PR #1046 ("chore(deps): bump github.com/prometheus/common from 0.72.1 to 0.73.0", merged as d6d463db09f58ee49e552a676d29dc7ba2415cb7) has taken github.com/prometheus/common to v0.73.0, bump it once more to v0.73.1.

Why: v0.73.1's only change is bumping github.com/pandatix/go-abnf from 0.5.1 to 0.5.3, and go-abnf 0.5.3 dropped its dependency on github.com/hashicorp/go-uuid. In the current tree go.sum:381-382 carries github.com/hashicorp/go-uuid v1.0.3 and go.sum:449-450 carries github.com/pandatix/go-abnf v0.5.1; neither module is imported anywhere in prow's own code.

Task:
- In the root module, update the direct requirement at go.mod:52 to github.com/prometheus/common v0.73.1 and run `go mod tidy`.
- In the hack/tools module, update the indirect requirement at hack/tools/go.mod:269 to v0.73.1 and run `go mod tidy` there too.
- Do not hand-edit go.sum; let `go mod tidy` regenerate it.

Acceptance criteria:
- `grep -n "hashicorp/go-uuid" go.sum hack/tools/go.sum` returns nothing, and `github.com/pandatix/go-abnf` is at v0.5.3 in both go.sum files.
- `GOEXPERIMENT=jsonv2 go build ./pkg/metrics/...` and `GOEXPERIMENT=jsonv2 go test ./pkg/metrics/...` pass (GOEXPERIMENT=jsonv2 is a pre-existing repo requirement from pkg/config/zz_generated.deepcopy.go).
- `GOEXPERIMENT=jsonv2 go list -deps ./cmd/hook | grep -E "go-abnf|go-uuid"` reports go-abnf but no go-uuid.
- `go mod tidy -diff` reports no diff in either module.

Scope guard: dependency manifests and lockfiles only. Do not change pkg/metrics or any other source, do not touch unrelated dependency versions that `go mod tidy` does not require, and do not investigate the OpenMetrics parser that go-abnf backs.
```
