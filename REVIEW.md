---
pr: kubernetes-sigs/prow#1056
title: "codegen: generate deepcopy code with the minimum Go version"
head_sha: dca32d5d58039c42c59eb725282f9532e4d72745
base: main
reviewed_at: 2026-11-04T16:52:48Z
verdict: approve
---

# Review

## Verdict

Approve with suggestions — verified end-to-end; the multi-perspective maintainer review found no blocking issues.

The fix pins deepcopy generation to the root module's declared Go version, so `encoding/json.RawMessage` resolves to the 1.26 stdlib definition instead of Go 1.27's `jsontext.Value` alias. Regenerating with that toolchain reproduces the committed output byte-for-byte, and the #1055 repro builds. Deployment risk is LOW (no config, API, or CRD changes); the shared follow-ups are hardening the version extraction and documenting the load-bearing runtime pin.

## What this PR does

- Derives `deepcopy_go_version` from the root `go.mod` (`go1.26.4`) and builds `deepcopy-gen` with that toolchain.
- Runs `deepcopy-gen` under the same `GOTOOLCHAIN`, which is required because gengo loads packages through `go list`.
- Leaves the other generators on the repo's build toolchain (`.go-version`, 1.27.3).
- Regenerates `pkg/config/zz_generated.deepcopy.go`, swapping `jsontext "encoding/json/jsontext"` / `new(jsontext.Value)` for `json "encoding/json"` / `new(json.RawMessage)`.

## Findings

### [should-fix] Fail loudly if the `go` directive cannot be parsed
- where: `hack/make-rules/update/codegen.sh:44`
- concern: The `awk` extraction yields an empty string if the directive's shape changes, and `GOTOOLCHAIN=` is accepted silently — codegen would fall back to the host toolchain and reintroduce `jsontext` with a confusing verify diff. Flagged independently by the Code Quality and Maintainability reviewers; add a non-empty/shape check with an explicit error (and consider `deepcopy_toolchain` as the name, since it holds a `GOTOOLCHAIN` value).
- excerpt: |
    deepcopy_go_version="$(awk '$1 == "go" { print "go" $2; exit }' "${REPO_ROOT}/go.mod")"
    GOTOOLCHAIN="${deepcopy_go_version}" go build -o "${deepcopygen}" k8s.io/code-generator/cmd/deepcopy-gen

### [nit] Explain why the runtime `GOTOOLCHAIN` pin is not redundant
- where: `hack/make-rules/update/codegen.sh:99-104`
- concern: The second assignment looks like a duplicate of the build-time pin, but it is load-bearing: gengo loads packages through `go/packages`, which shells out to `go list`; without it the host toolchain is used for type loading. Running the 1.26.4-built generator with `GOTOOLCHAIN=go1.27.3` fails while parsing the 1.27 stdlib (`math/rand/v2`), so a comment should keep a future cleanup from removing it. Flagged by both Code Quality and Maintainability.
- excerpt: |
    gen-deepcopy() {
      echo "Generating DeepCopy() methods..." >&2
      GOTOOLCHAIN="${deepcopy_go_version}" "$deepcopygen" \

### [nit] Say which module the pin comes from
- where: `hack/make-rules/update/codegen.sh:42-45`
- concern: The build happens after `cd hack/tools` (whose `go.mod` says `go 1.26.3`), but the `awk` deliberately reads the root `go.mod` (`go 1.26.4`) — the module being generated. The choice is correct but non-obvious; state that the root `go.mod` is intentional and that a `toolchain` directive / `.go-version` are deliberately not used.
- excerpt: |
    # Generate against the module's minimum Go version, including standard-library
    # aliases that can resolve differently in newer Go releases.
    deepcopy_go_version="$(awk '$1 == "go" { print "go" $2; exit }' "${REPO_ROOT}/go.mod")"

## Checked

- Built `deepcopy-gen` from `hack/tools` with `GOTOOLCHAIN=go1.26.4` and reran it with the script's exact arguments; `pkg/config/zz_generated.deepcopy.go` and `pkg/apis/prowjobs/v1/zz_generated.deepcopy.go` both came out byte-identical to the committed files.
- `GOTOOLCHAIN=go1.26.4 go build ./cmd/crier` (the repro from #1055) succeeds at this head.
- `git grep jsontext` at the base commit finds only the two lines this PR changes; nothing else in the tree references `encoding/json/jsontext`.
- `ProwIgnored` is `*json.RawMessage` (`pkg/config/config.go:143`, `pkg/config/inrepoconfig.go:94`), so the regenerated `new(json.RawMessage)` targets the declared type; runtime behavior is identical under both toolchains.
- `hack/tools/go.mod` declares `go 1.26.3`, so building the tools with the root module's `go1.26.4` toolchain is valid.
- Deployment review found no changes to config fields, struct tags, defaults, flags, APIs, or CRD schema; deployment risk is LOW.
- CI does run codegen verification: `pull-prow-verify-lint` runs `make verify`, which reaches `verify/codegen.sh` and reruns the pinned update script in an isolated tree. An earlier assumption that no codegen presubmit existed was wrong.
- `bash -n hack/make-rules/update/codegen.sh` passes; presubmits are green (including `verify`).

## Open questions

- `verify-codegen` covers generated output, but nothing compiles hand-written code with `go 1.26.4` (tests and builds use `.go-version`, 1.27.3). Would a cheap minimum-toolchain build check be worth adding so post-minimum stdlib usage is caught before downstream consumers hit it? Follow-up, not required for this fix.

## Followups

### [deferred-review] Fail loudly if the `go` directive cannot be parsed
- necessity: should
- mode: deliver
- where: `hack/make-rules/update/codegen.sh:44-45`
- prompt:

```text
In kubernetes-sigs/prow, following post-merge followup to PR #1056 ("codegen: generate deepcopy code with the minimum Go version", merged as a23969f42bcfccdeb82025ed62b7d0b3c53a9191), harden the toolchain pin in hack/make-rules/update/codegen.sh.

Context: line 44 currently does
  deepcopy_go_version="$(awk '$1 == "go" { print "go" $2; exit }' "${REPO_ROOT}/go.mod")"
and both line 45 and line 101 use it as GOTOOLCHAIN. If the go directive moves or changes shape, awk prints nothing, GOTOOLCHAIN becomes empty, and the Go command silently falls back to the host toolchain, reintroducing the encoding/json/jsontext leak this PR fixed.

Task:
- Rename deepcopy_go_version to deepcopy_toolchain (it holds a GOTOOLCHAIN value).
- Extract the raw version from the root go.mod's go directive into a separate variable, then validate it: non-empty and matching ^[0-9]+\.[0-9]+(\.[0-9]+)?$. On failure, print a clear ERROR to stderr naming hack/make-rules/update/codegen.sh and go.mod, and exit 1.
- Build the GOTOOLCHAIN value as "go${version}" and keep both call sites (the deepcopy-gen build and the gen-deepcopy invocation) otherwise unchanged.

Acceptance criteria:
- bash -n hack/make-rules/update/codegen.sh passes.
- A normal run still regenerates pkg/config/zz_generated.deepcopy.go and pkg/apis/prowjobs/v1/zz_generated.deepcopy.go byte-identically.
- In a scratch copy of the repo with the "go 1.26.4" line removed from go.mod, the script exits non-zero with the new error instead of proceeding.

Scope guard: do not change the toolchain values, other generators, generated files, or unrelated codegen steps. Do not publish anything to GitHub.
```

### [deferred-review] Document the pin's provenance and why the runtime `GOTOOLCHAIN` is load-bearing
- necessity: should
- mode: deliver
- where: `hack/make-rules/update/codegen.sh:42-45`, `hack/make-rules/update/codegen.sh:99-104`
- prompt:

```text
In kubernetes-sigs/prow, following post-merge followup to PR #1056 ("codegen: generate deepcopy code with the minimum Go version", merge commit a23969f42bcfccdeb82025ed62b7d0b3c53a9191), add the missing rationale comments around the deepcopy toolchain pin in hack/make-rules/update/codegen.sh.

Task:
- At the extraction (around lines 42-45), extend the comment to state that the root go.mod is read deliberately because it is the module whose code is being generated; hack/tools/go.mod (currently go 1.26.3) is not the source of truth, and neither the toolchain directive nor .go-version (1.27.3) is used.
- At the gen-deepcopy invocation (around lines 99-104), add a comment on GOTOOLCHAIN="${deepcopy_toolchain}" "$deepcopygen" explaining that it is not a duplicate of the build-time pin: gengo's parser loads packages through go/packages, which shells out to `go list`, so the runtime toolchain must match the one the binary was built with. Note that removing it reintroduces encoding/json/jsontext (see issue #1055).

Acceptance criteria:
- Comments are concise (about one sentence each) and factual.
- No behavioral change: bash -n passes and regeneration output is byte-identical.
- The comments make clear that GOTOOLCHAIN at the invocation is load-bearing.

Scope guard: comments only; do not alter the script logic, generated files, or other generators. Do not publish anything to GitHub.
```

### [docs] Document the project's Go version policy
- necessity: could
- mode: deliver
- where: `site/content/en/docs/build-test-update.md`
- prompt:

```text
In kubernetes-sigs/prow, following post-merge followup to PR #1056 ("codegen: generate deepcopy code with the minimum Go version", merge commit a23969f42bcfccdeb82025ed62b7d0b3c53a9191), document the project's Go version policy for contributors.

Context: the repo has three version sources: .go-version (1.27.3, the toolchain used by setup-go.sh for dev/CI builds and tests), the root go.mod go directive (1.26.4, the minimum supported language/stdlib version), and hack/tools/go.mod (1.26.3, the separate tools module). This split caused issue #1055, and a PR reviewer asked why the project is "back on 1.26".

Task:
- Add a short "Go version policy" subsection to site/content/en/docs/build-test-update.md, in the build/test section near "How to build and test Prow".
- Explain each of the three version sources and its purpose.
- State that make update-codegen builds and runs deepcopy-gen with the root go.mod minimum so generated code stays buildable on the minimum supported Go, that other generators use .go-version, and that generated code must not depend on post-minimum stdlib APIs.
- Keep it concise (roughly 8-15 lines) and follow the page's existing heading/style conventions.

Acceptance criteria:
- The page remains valid Hugo content and the new section renders with valid Markdown/headings.
- The explanation matches the behavior in hack/make-rules/update/codegen.sh and hack/build/setup-go.sh.
- No code changes.

Scope guard: documentation only; do not change versions, scripts, or generated files. Do not publish anything to GitHub.
```

### [cleanup] Move the proxy/sumdb exports above the tool-build block
- necessity: could
- mode: deliver
- where: `hack/make-rules/update/codegen.sh:233-235`, `hack/make-rules/update/codegen.sh:38-54`
- prompt:

```text
In kubernetes-sigs/prow, following post-merge followup to PR #1056 ("codegen: generate deepcopy code with the minimum Go version", merge commit a23969f42bcfccdeb82025ed62b7d0b3c53a9191), make toolchain and module acquisition deterministic in hack/make-rules/update/codegen.sh.

Context: GO111MODULE=on, GOPROXY=https://proxy.golang.org, and GOSUMDB=sum.golang.org are exported only near the bottom of the script (around lines 233-235), after all codegen tools are built (lines 38-54). The new pinned deepcopy-gen build may download the go1.26.4 toolchain, so it currently relies on ambient proxy configuration.

Task:
- Move the GO111MODULE/GOPROXY/GOSUMDB export block above the "build codegen tools" section, before any go build or go install runs.
- Remove the now-duplicate block at the bottom.

Acceptance criteria:
- bash -n hack/make-rules/update/codegen.sh passes.
- Running the script with GOPROXY/GOSUMDB/GO111MODULE unset in the environment still resolves modules and the pinned toolchain from the explicit settings, and regenerated output is byte-identical.
- No other behavior changes.

Scope guard: do not change generator versions, toolchain values, or generated files. Do not publish anything to GitHub.
```
