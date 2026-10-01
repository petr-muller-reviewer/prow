---
pr: kubernetes-sigs/prow#759
title: "chore(deps): bump github.com/sigstore/timestamp-authority/v2 from 2.0.3 to 2.0.6 in /hack/tools"
head_sha: c81d340d410f3c01f307bd65164df246c60305d9
base: main
reviewed_at: 2026-07-26T23:48:59Z
verdict: approve
---

## What this PR does
- Dependabot bump of `github.com/sigstore/timestamp-authority/v2` from 2.0.3 to 2.0.6 in the `hack/tools` go module.
- Pulls in a `go mod tidy`-driven refresh of unrelated transitive/indirect deps (AWS SDK v2 components, `golang.org/x/net`, `go-jose`, etc.).
- Changes are confined to `hack/tools/go.mod` and `hack/tools/go.sum` (179 insertions, 181 deletions, no hand-written code).
- v2.0.6 fixes GHSA-xm5m-wgh2-rrg3 ("Ensure correct certificate is used for TSA auth checks"); v2.0.5 fixed a chi-middleware panic and raised the default HTTP idle timeout.
- PR already merged with `lgtm`/`approved` labels.

## Findings
None.

## Checked
- Version bump target: only affects `hack/tools` (build tooling), not the main module — minimal runtime blast radius.
- Release notes/changelog for 2.0.4-2.0.6: confirms security fix (GHSA-xm5m-wgh2-rrg3) plus routine bug fixes, no breaking API changes called out.
- Transitive AWS SDK v2 bumps are all patch/minor increments consistent with a mechanical `go mod tidy`, nothing anomalous.
- Diff is fully mechanical (go.mod/go.sum only), matching expected Dependabot output.

## Open questions
None.

## Dependency followups
No actionable opportunities were identified for PR #759 (merged as `37d12bec71174fdb925833014b32133404abd775`).

- `github.com/sigstore/timestamp-authority/v2` v2.0.3→v2.0.6 (releases v2.0.4–v2.0.6): the changelog covers dependency refreshes, a chi middleware panic fix, an increased default HTTP idle timeout, and the TSA authentication certificate correctness fix (GHSA-xm5m-wgh2-rrg3). It offers no API migration or feature for our code to adopt, and `hack/tools` contains no owned Go import sites for this module.
- `github.com/sigstore/sigstore` v1.10.3→v1.10.5 (indirect): releases mention a target-name security fix and OpenBao support, but there are no matching import sites in `hack/tools`.
- The remaining indirect manifest changes likewise have no matching Go imports under `hack/tools`; references elsewhere in the repository belong to the root module and are not governed by this nested module's dependency update.
