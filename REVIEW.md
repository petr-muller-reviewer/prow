---
pr: kubernetes-sigs/prow#905
title: "chore(deps): bump github.com/golangci/golangci-lint/v2 from 2.12.2 to 2.13.1 in /hack/tools"
head_sha: 72c049d2b66a42a7a998e07246cd38bce3e9c754
base: main
reviewed_at: 2026-08-30T17:39:07Z
verdict: approve
---

## What this PR does

- Bumps `github.com/golangci/golangci-lint/v2` from `v2.12.2` to `v2.13.1` in `hack/tools/go.mod`/`go.sum` (direct dependency of the separate build-tooling module, plus the usual transitive churn of golangci-lint's own dependencies).
- Narrows the `staticcheck` exclude regex in `.golangci.yml:41` from a message-specific match to a looser `SA1019: .*TrustedOrg is deprecated` pattern, to keep matching after upstream reworded the deprecation string.
- Migrates `cmd/deck/main.go:1364-1370` from the deprecated `httputil.ReverseProxy.Director` field to `.Rewrite`, newly flagged by the updated linter set.
- Adds a documented `//nolint:staticcheck` on `pkg/io/credentials.go:34` for `CredentialsFile` (SA1019), also newly surfaced by the bump.

## Findings

None.

## Checked

- Freshness: `v2.13.1` tagged 2026-08-20 (verified via `proxy.golang.org` and `gh release view`) — 10 days old, real tag (not a pseudo-version). Acceptable soak time for a dev-only tool.
- Usage/exposure: `golangci-lint/v2` is a direct dependency of `hack/tools/go.mod`, referenced only via the tool-pinning pattern in `hack/tools/tools.go` (`grep -rln "github.com/golangci/golangci-lint" --include='*.go' .` → only that file). Build/CI tooling only, not linked into any shipped binary — no production exposure.
- Changelog (`v2.12.2`→`v2.13.0`→`v2.13.1`): almost entirely bot bumps of golangci-lint's own internal linter deps, plus minor fixes (package-facts caching, `funcorder`/`gomoddirectives` field additions, go1.27 support, a new `goconst` option). No CVEs, no security-relevant changes.
- `cmd/deck/main.go` `Director`→`Rewrite` migration: `grep -rn "Director:" --include='*.go' .` returns no remaining hits; the new code sets `pr.Out.URL`/`.ContentLength`/`.Body`, which is the doc-recommended replacement and semantically equivalent to the old full-replacement `Director` body.
- `pkg/io/credentials.go` nolint: justification in the comment (operator-supplied flag, not untrusted input; no single replacement API covers all credential types accepted here) is sound, not a blind suppression.
- `.golangci.yml` regex narrowing still scopes only to `TrustedOrg`-related `SA1019` findings — no risk of over-broadly suppressing unrelated staticcheck hits.

## Open questions

None.

## Dependency followups

Examined on 2026-10-16T00:19:00Z. No concrete opportunities identified; no items or handoff prompts recorded.

PR #905 merged on 2026-08-26 at `7f1a03797ffbc75d8c54bc1adba0a045ef91669e`. Scope: the Go tooling dependency bump and the capabilities it adds at PR head `72c049d2b66a42a7a998e07246cd38bce3e9c754`. Since the merged head is already an ancestor of `upstream/main`, the ordinary merge-base is the PR head itself; the dependency inventory instead uses GitHub's recorded PR base `9a587d94024f159171ab0d71c8482cf2b064bc16` to avoid an empty post-merge diff. The local review commit is excluded.

### Changelogs cross-referenced with Prow usage

- `github.com/golangci/golangci-lint/v2` **v2.12.2 → v2.13.1** (direct): reviewed both intervening releases, [v2.13.0](https://github.com/golangci/golangci-lint/releases/tag/v2.13.0) and [v2.13.1](https://github.com/golangci/golangci-lint/releases/tag/v2.13.1), and the [curated changelog](https://golangci-lint.run/docs/product/changelog/#v2130). Prow pins the command in `hack/tools/tools.go:25` and runs it through `hack/make-rules/verify/golangci-lint.sh:37`. New settings for goconst, gomoddirectives, exhaustruct and other disabled linters have no existing configuration to migrate; cache fixes apply automatically. Go 1.27 support alone provides no concrete followup for this Go 1.26 codebase.
- `golang.org/x/tools` **v0.48.0 → v0.49.0** (indirect, used through enabled govet/modernize): read the [commit range](https://github.com/golang/tools/compare/v0.48.0...v0.49.0). The new [reflecttypeassert analyzer](https://github.com/golang/tools/commit/cd7d17b664fc3769ed34b444294ccb2eb2e3310e) targets two-result assertions on `reflect.Value.Interface()`. The only immediate assertion in Prow is the single-result assertion in `pkg/github/client_test.go:3438`, which the analyzer deliberately excludes to preserve panic behavior; the calls in `pkg/genyaml/populate_struct.go` do not perform such assertions. The new [slicesclip analyzer](https://github.com/golang/tools/commit/63ec6f9ccb03c2a50c7e30d4e0ad29bf301e6d36) targets `s[:len(s):len(s)]`; no matching capacity-clipping expressions were found. Both analyzers are enabled automatically through modernize's suite; `.golangci.yml:24` only disables `omitzero`. Remaining changes fix analyzer behavior or documentation, with no matching suppression to remove.
- `honnef.co/go/tools` **v0.7.0 → v0.8.0** (indirect, enabled staticcheck/unused): read [Staticcheck 2026.2 release notes](https://github.com/dominikh/go-tools/releases/tag/2026.2). SA9010 is covered by the existing `checks: all` configuration. SA1019 struct-initializer detection and diagnostic formatting are already addressed in this PR's ReverseProxy migration, CredentialsFile suppression and TrustedOrg exclusion. SA5011's removal does not require adding the suggested nilness replacement: govet's nilness analyzer is already explicitly enabled in `.golangci.yml:10`. Existing staticcheck suppressions concern real deprecations, not the generic/interface false positives improved by this release; none provides a concrete cleanup candidate.
- `github.com/stretchr/testify` **v1.12.0 → v1.12.1** (indirect in tooling, imported by five Prow test files): read the [release](https://github.com/stretchr/testify/releases/tag/v1.12.1) and [complete three-commit range](https://github.com/stretchr/testify/compare/v1.12.0...v1.12.1). This patch changes Testify's internal YAML dependency; it introduces no assertion API to adopt. Prow's separate root module already selects v1.12.1 at `go.mod:58`, unchanged by this PR.
- `go.yaml.in/yaml/v3` **v3.0.4 → v3.0.5** (indirect in tooling, four Prow import sites): read the [complete ten-commit range](https://github.com/yaml/go-yaml/compare/v3.0.4...v3.0.5) because no GitHub release entry exists for this tag. Changes retract invalid historical tags and update upstream tests/workflows; no new application API or behavior to adopt. Prow already selects v3.0.5 at `go.mod:62`, unchanged by this PR.

### Complete tooling manifest inventory

The following entries were inventoried from `hack/tools/go.mod`. Except for the direct command dependency, every changed entry is indirect. Pure transitive packages with no Prow imports were deprioritized rather than having every individual changelog read; analyzer packages exposed by the enabled linters were examined above. Adding a module path is distinguished from upgrading an existing path.

- `4d63.com/gocheckcompilerdirectives`: `v1.3.0` → `v1.4.0` (indirect).
- `charm.land/lipgloss/v2`: `v2.0.3` → `v2.0.6` (indirect).
- `dev.gaijin.team/go/exhaustruct/v5`: `absent` → `v5.0.3` (indirect).
- `dev.gaijin.team/go/golib`: `v0.6.0` → `v0.8.1` (indirect).
- `github.com/Abirdcfly/dupword`: `v0.1.7` → `v0.1.8` (indirect).
- `github.com/AlwxSin/noinlineerr`: `v1.0.5` → `v1.0.6` (indirect).
- `github.com/Antonboom/errname`: `v1.1.1` → `v1.1.2` (indirect).
- `github.com/Antonboom/nilnil`: `v1.1.1` → `v1.1.2` (indirect).
- `github.com/ClickHouse/clickhouse-go-linter`: `v1.2.0` → `v1.2.1` (indirect).
- `github.com/alecthomas/chroma/v2`: `v2.24.1` → `v2.27.0` (indirect).
- `github.com/bombsimon/wsl/v5`: `v5.8.0` → `v5.9.0` (indirect).
- `github.com/butuzov/mirror`: `v1.3.0` → `v1.3.3` (indirect).
- `github.com/charmbracelet/ultraviolet`: `v0.0.0-20251205161215-1948445e3318` → `v0.0.0-20260811164956-006e29f97886` (indirect).
- `github.com/charmbracelet/x/ansi`: `v0.11.7` → `v0.11.8` (indirect).
- `github.com/dlclark/regexp2`: `v1.12.0` → `removed` (indirect).
- `github.com/dlclark/regexp2/v2`: `absent` → `v2.2.1` (indirect).
- `github.com/felixge/httpsnoop`: `v1.0.4` → `v1.1.0` (indirect).
- `github.com/firefart/nonamedreturns`: `v1.0.6` → `v1.0.8` (indirect).
- `github.com/ghostiam/protogetter`: `v0.3.20` → `v0.3.21` (indirect).
- `github.com/go-critic/go-critic`: `v0.14.3` → `v0.14.4` (indirect).
- `github.com/golangci/gofmt`: `v0.0.0-20250106114630-d62b90e6713d` → `v0.0.0-20260820135601-e84e05053792` (indirect).
- `github.com/golangci/golangci-lint/v2`: `v2.12.2` → `v2.13.1` (direct).
- `github.com/jgautheron/goconst`: `v1.10.0` → `v1.11.0` (indirect).
- `github.com/kisielk/errcheck`: `v1.10.0` → `v1.20.0` (indirect).
- `github.com/ldez/gomoddirectives`: `v0.8.0` → `v0.9.0` (indirect).
- `github.com/lucasb-eyer/go-colorful`: `v1.4.0` → `v1.4.1` (indirect).
- `github.com/mattn/go-colorable`: `v0.1.14` → `v0.1.15` (indirect).
- `github.com/mattn/go-runewidth`: `v0.0.23` → `v0.0.24` (indirect).
- `github.com/nunnatsa/ginkgolinter`: `v0.23.0` → `v0.24.0` (indirect).
- `github.com/pelletier/go-toml/v2`: `v2.3.1` → `v2.4.3` (indirect).
- `github.com/raeperd/recvcheck`: `v0.2.0` → `v0.3.0` (indirect).
- `github.com/rogpeppe/go-internal`: `v1.14.1` → `v1.16.0` (indirect).
- `github.com/santhosh-tekuri/jsonschema/v6`: `v6.0.2` → `v6.0.3` (indirect).
- `github.com/securego/gosec/v2`: `v2.26.1` → `v2.28.0` (indirect).
- `github.com/stretchr/testify`: `v1.12.0` → `v1.12.1` (indirect).
- `github.com/uudashr/iface`: `v1.4.2` → `v1.5.0` (indirect).
- `go.augendre.info/fatcontext`: `v0.9.0` → `v0.10.0` (indirect).
- `go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp`: `v0.68.0` → `v0.69.0` (indirect).
- `go.yaml.in/yaml/v3`: `v3.0.4` → `v3.0.5` (indirect).
- `golang.org/x/exp/typeparams`: `v0.0.0-20260209203927-2842357ff358` → `v0.0.0-20260811152304-ee035b5b010f` (indirect).
- `golang.org/x/mod`: `v0.38.0` → `v0.40.0` (indirect).
- `golang.org/x/tools`: `v0.48.0` → `v0.49.0` (indirect).
- `google.golang.org/genproto/googleapis/rpc`: `v0.0.0-20260523011958-0a33c5d7ca68` → `v0.0.0-20260706201446-f0a921348800` (indirect).
- `honnef.co/go/tools`: `v0.7.0` → `v0.8.0` (indirect).
- `mvdan.cc/gofumpt`: `v0.9.2` → `v0.11.0` (indirect).
- `mvdan.cc/unparam`: `v0.0.0-20251027182757-5beb8c8f8f15` → `v0.0.0-20260818115549-3f964bcb5673` (indirect).

### Additional go.sum changes

The following eight modules change only in `hack/tools/go.sum`, not in its explicit requirements. These are checksum inventory changes and do not by themselves establish a change to Prow's runtime dependency selection. Of these modules, Prow directly imports only Cloud Auth and the Google API client: their root-module versions are already v0.23.2 and v0.293.0 respectively (`go.mod:6`, `go.mod:72`), newer than the tooling checksum ranges below. Their existing call sites, including `pkg/io/credentials.go:22` and `pkg/io/opener.go:39`, therefore gain no newly available runtime API from this PR. The other six modules have no direct Prow imports and were deprioritized.

- `cloud.google.com/go/auth`: `v0.20.0` → `v0.21.0` (transitive checksum entries).
- `github.com/go-quicktest/qt`: `v1.101.0` → `v1.102.0` (transitive checksum entries).
- `github.com/google/pprof`: `v0.0.0-20260402051712-545e8a4df936` → `v0.0.0-20260709232956-b9395ee17fa0` (transitive checksum entries).
- `github.com/googleapis/enterprise-certificate-proxy`: `v0.3.16` → `v0.3.18` (transitive checksum entries).
- `github.com/googleapis/gax-go/v2`: `v2.22.0` → `v2.23.0` (transitive checksum entries).
- `github.com/onsi/ginkgo/v2`: `v2.28.2` → `v2.32.0` (transitive checksum entries).
- `github.com/onsi/gomega`: `v1.40.0` → `v1.42.1` (transitive checksum entries).
- `google.golang.org/api`: `v0.283.0` → `v0.288.0` (transitive checksum entries).
