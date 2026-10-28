---
pr: kubernetes-sigs/prow#750
title: "feat: Add kube-api-linter and fix ProwJob API type annotations"
head_sha: bec45be57ec57e9849c0144952eb0c987e518d2e
base: main
reviewed_at: 2026-06-16T10:59:45Z
verdict: approve
refresh_log:
  - from: ce1159b9d156135d5c15054e86e800d5a7d37537
    to: bec45be57ec57e9849c0144952eb0c987e518d2e
    summary: "Author added 'make verify' commit: adds hack/tools/tmp/ to .gitignore, adds missing +optional on Refs.Auxiliary, lowercases CRD description for Auxiliary field, gofmt/go vet cleanup across unrelated files"
---

## Summary

Integrates kube-api-linter (KAL) into CI. Adds `make lint-api`, `hack/make-rules/verify/kube-api-lint.sh`, KAL config at `hack/tools/.golangci-kal.yml`, and build script `hack/tools/build-kal`. Fixes `pkg/apis/prowjobs/v1/types.go` to comply with `commentstart` (lowercase JSON-tag-prefixed comments) and `optionalorrequired` (every field marked `+optional` or `+required`). Regenerates CRD YAML and documented config files. Low deployment risk — no struct tags, validation rules, or runtime behavior changed.

## Converging concerns

Two issues were independently flagged by multiple reviewers:

1. **CRD description information loss** (Maintainability + Deployment Risk): `Spec` and `Status` field comments reduced to tautologies. The `commentstart` rule only requires the first word to match — descriptive content can be preserved.
2. **`build-kal` missing `set -o errexit`** (Code Quality + Maintainability): Every other verify script includes it. Practical impact low but masks real build errors.

## Since previous review (2026-06-16)

Author pushed one commit ("make verify", `bec45be57`):
- Added `hack/tools/tmp/` to `.gitignore` — this is the build output dir for the KAL binary
- Added missing `// +optional` marker on `Refs.Auxiliary bool` (line 1388 in types.go) — this was a field the original PR missed
- Lowercased `Auxiliary` → `auxiliary` in CRD description for that field (commentary rule compliance)
- `gofmt`/`go vet` alignment fixes in unrelated files (`markdown.go`, `storageartifact_test.go`, `tide_test.go`, etc.) — these appear to be required by `make verify` to pass

None of the [should-fix] findings from the original review were addressed. The open questions remain open.

## Findings

### [should-fix] CRD descriptions gutted for ProwJobSpec, ProwJobStatus, ResultStoreConfig
- where: `pkg/apis/prowjobs/v1/types.go:136-141`
- concern: To satisfy `commentstart`, several field comments were replaced with tautological stubs. Users reading `kubectl explain prowjob.spec` now see "spec is the ProwJob spec" instead of the previous description covering podspec, cloning, clusters, child jobs, concurrency. The `commentstart` rule only requires the comment to _start_ with the JSON tag name in lowercase — it doesn't require removing all content. Rewrite to keep descriptive text with lowercase first word. Flagged by Maintainability and Deployment Risk reviewers independently.
- excerpt: |
    // spec is the ProwJob spec
    // +optional
    Spec ProwJobSpec `json:"spec,omitempty"`
    // status is the ProwJob status
    // +optional
    Status ProwJobStatus `json:"status,omitempty"`

### [should-fix] build-kal missing set -o errexit
- where: `hack/tools/build-kal:16-17`
- concern: Has `set -o nounset` and `set -o pipefail` but omits `set -o errexit`. Every other verify script in the repo includes all three. Without it, if `go tool ... golangci-lint custom` fails, the real error is masked behind "binary not found". Flagged by Code Quality and Maintainability reviewers independently.
- excerpt: |
    set -o nounset
    set -o pipefail

### [should-fix] Questionable +optional on embedded time.Duration in Duration wrapper
- where: `pkg/apis/prowjobs/v1/types.go:464-467`
- concern: `Duration` is a wrapper whose sole purpose is to hold a `time.Duration` value. Marking the embedded field `+optional` is semantically wrong — it's the value itself, not an optional sub-field. If KAL flags embedded fields in wrapper types, a nolint directive would be more appropriate.
- excerpt: |
    type Duration struct {
    	// +optional
    	time.Duration
    }

### [should-fix] Dubious +optional/+required markers on ProwJobList embedded types
- where: `pkg/apis/prowjobs/v1/types.go:1465-1473`
- concern: `TypeMeta` is marked `+optional` and `ListMeta`/`Items` are marked `+required`. These are structural fields on a list type, not user-facing optional/required API fields. Consider a nolint directive or excluding the list type from KAL scope.
- excerpt: |
    type ProwJobList struct {
    	// +optional
    	metav1.TypeMeta `json:",inline"`
    	// +required
    	metav1.ListMeta `json:"metadata"`
    	// +required
    	Items []ProwJob `json:"items"`
    }

### [nit] .custom-gcl.yaml version coupling with go.mod
- where: `hack/tools/.custom-gcl.yaml:1`
- concern: Pins `version: v2.12.2` for golangci-lint separately from the version in `go.mod`. When golangci-lint is bumped in `go.mod`, this file must be updated in lockstep. No comment warns about the coupling.

### [nit] build-kal duplicate "Build complete" message
- where: `hack/tools/build-kal:29,33-34`
- concern: Lines 29 and 33-34 both print "Build complete" on success. The second one (with the "Run with:" hint) is redundant.

### [nit] build-kal redundant -modfile flag
- where: `hack/tools/build-kal:26`
- concern: The subshell already `cd`s into `hack/tools/` where `go.mod` lives. The `-modfile "${SCRIPT_DIR}/go.mod"` flag is redundant.
- excerpt: |
    (cd "${SCRIPT_DIR}" && go tool -modfile "${SCRIPT_DIR}/go.mod" golangci-lint custom)

### [nit] KAL pinned to a pseudo-version
- where: `hack/tools/go.mod:21`, `hack/tools/.custom-gcl.yaml:6`
- concern: `sigs.k8s.io/kube-api-linter v0.0.0-20260206102632-39e3d06a2850` is a pre-release pseudo-version. If a tagged release exists, prefer it.

### [nit] Trailing blank line in kube-api-lint.sh
- where: `hack/make-rules/verify/kube-api-lint.sh:37`
- concern: File ends with an extra blank line.

### [question] ResultStoreConfig.ProjectID marker mismatch
- where: `pkg/apis/prowjobs/v1/types.go:509`
- concern: The prose says "Required to upload results to ResultStore" but the marker is `// +optional`. Pre-existing condition, but this PR had the opportunity to clarify it.

## Checked

- KAL config structure and enabled/disabled linter selection — conservative, well-documented with good TODO comments
- Makefile integration (`lint-api` target, `clean` additions) — correct
- `verify/all.sh` integration with env-var gate `VERIFY_KUBE_API_LINT` — follows existing pattern, can bypass via env var
- `+required` / `+optional` marker correctness on regular struct fields (ProwJobSpec, ProwJobStatus, Refs, Pull, etc.) — correct throughout
- `+kubebuilder:validation:Required` to `+required` migration — correct, produces identical CRD output
- Regenerated CRD YAML and documented config YAMLs — consistent with types.go changes
- `plugin-config-documented.yaml` includes new `invalid_commit_msg` section — from prior main commits picked up during regeneration, not from this PR
- `go.sum` changes — only adds the KAL module entry
- `tools.go` blank import — follows existing pattern
- No `json:` or `yaml:` struct tags modified — wire-format compatibility preserved
- No validation rules, default values, or required/optional field semantics change in generated CRD
- KAL dependency scoped to `hack/tools` module only — cannot affect main binary or runtime
- Deployment risk is LOW — CRD update is safe, no config migration needed, rollback trivial

## Open questions

- Why were descriptive comments stripped rather than reworded to start with the JSON tag? The `commentstart` rule only requires the first word to match — the rest of the description could have been preserved.
- Is `+optional` on embedded `time.Duration` (in the `Duration` wrapper) actually required by KAL, or was it added preemptively? If KAL flags it, would a nolint directive be more appropriate?
- Is `kube-api-linter` available as a tagged release? The pseudo-version makes it harder to track which version of the rules the codebase conforms to.
- Should `ResultStoreConfig.ProjectID` be `+required` given the prose says it is required for ResultStore functionality?
