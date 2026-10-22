---
pr: kubernetes-sigs/prow#896
title: "chore(deps): bump github.com/clarketm/json from 1.13.4 to 1.17.1"
head_sha: 6098ff515ed14120f0c0613762b0a9b1578c37e3
base: main
reviewed_at: 2026-09-01T09:34:11Z
verdict: approve
---

## Verdict

Approve. This is a direct but narrowly used JSON encoder bump; the upstream changes are well-soaked and the only affected package tests successfully.

## What this PR does

- Updates `github.com/clarketm/json` from `v1.13.4` to `v1.17.1` in `go.mod` and `go.sum`.
- Retains the existing direct dependency used by `pkg/genyaml`.
- Picks up JSON decoder nesting-depth protection, encoder cycle detection, and encoding fixes.
- Adds zero-value struct `omitempty` behavior used by this project's custom JSON fork.

## Findings

No findings.

## Checked

- Dependency-only diff: `go.mod` and `go.sum`; no project source changes.
- `v1.17.1` is a non-pseudo tagged version released 2021-09-14, with ample soak time.
- Direct import surface is one package: `pkg/genyaml`; `marshal` uses `json.Marshal` at `pkg/genyaml/genyaml.go:164`, and diagnostics use `json.MarshalIndent` at `pkg/genyaml/genyaml.go:460`.
- Upstream range contains decoder nesting-depth protection, cycle detection, corrected encoding behavior, and zero-value struct `omitempty` support; no published repository advisory was returned.
- `go test ./pkg/genyaml` passed.
- PR CI checks passed before merge.

## Open questions

None.

## Dependency followups

No concrete followup opportunities identified for PR #896, merged as `2677f075a487a800c949d06e24305b1d7301c456`.

- Examined Go module `github.com/clarketm/json` (direct), `v1.13.4` → `v1.17.1`. The manifest and checksum diff changes no other module versions, so there are no transitive bumps to investigate.
- Reviewed all 27 commits in the [upstream version range](https://github.com/clarketm/json/compare/v1.13.4...v1.17.1), including intermediate changes and their reverts, plus the [v1.17.1 feature documentation](https://github.com/clarketm/json/blob/v1.17.1/README.md). No GitHub release notes or separate changelog were available.
- Cross-referenced the only project import, `pkg/genyaml/genyaml.go:82`, and its two calls: `json.Marshal` at line 166 and `json.MarshalIndent` at line 460. Cycle detection and encoding corrections take effect through those existing calls without a migration. Decoder changes do not apply: `jsonToYaml` deliberately uses YAML decoding to preserve numeric types (`pkg/genyaml/genyaml.go:183`). The proposed `errors.Is` APIs were reverted within the range.
- The new feature, “Support zero values of structs with omitempty,” does not replace `PopulateStruct`: that helper deliberately creates nonempty examples so fields remain visible in generated documentation (`pkg/genyaml/populate_struct.go:57-71`). Removing its placeholder strings or recursive population would undermine that purpose. No deprecated calls, replacement APIs, or concrete redundant workaround were identified.

No items or handoff prompts recorded. This was changelog and source inspection; no tests were run for this followup pass.
