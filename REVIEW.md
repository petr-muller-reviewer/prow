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
