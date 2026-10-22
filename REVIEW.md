---
pr: kubernetes-sigs/prow#900
title: "chore(deps): bump github.com/GoogleCloudPlatform/testgrid from 0.0.123 to 0.0.176"
head_sha: 6913b7d77120370d6757e7f0e5ea77b7d7891a76
base: main
reviewed_at: 2026-09-01T00:50:49Z
verdict: approve
---

## Verdict

Approve. This dependency-only update is adequately soaked, remains API-compatible with Prow's usage, and passed the repository's unit, integration, image-build, lint, and race-detector checks.

## What this PR does

- Updates the direct `github.com/GoogleCloudPlatform/testgrid` dependency from `v0.0.123` to `v0.0.176`.
- Updates indirect dependencies `bitbucket.org/creachadair/stringset` from `v0.0.9` to `v0.0.11` and `github.com/fvbommel/sortorder` from `v1.0.1` to `v1.1.0`.
- Removes obsolete `go.sum` entries produced by dependency resolution.
- Changes no Prow source code.

## Findings

No findings.

## Checked

- `v0.0.176` is a tagged release from 2026-08-10, giving it 22 days of soak time at review.
- Testgrid is a direct dependency imported by 20 Prow files across 11 packages; 14 imports are in production files.
- Prow exercises Testgrid metadata models, JUnit parsing, configuration protobufs, and GCS utilities in artifact upload, reporting, Deck, and Spyglass paths.
- The 889-commit upstream range primarily adds configuration and ResultStore capabilities, extends JUnit metadata parsing, and updates GCS/config internals; the APIs Prow exercises remain compatible.
- The configuration protobuf changes are additive for Prow's usage.
- Prow does not directly import either updated indirect dependency.
- GitHub CI passed unit, integration, image-build, lint, and nonblocking race-detector jobs for head `6913b7d77120370d6757e7f0e5ea77b7d7891a76`.

## Open questions

None.

## Dependency followups

Examined at: 2026-10-22T16:15:39Z. Accepted: 2. Skipped: 0. Both items are Mode: deliver.

PR #900 was merged on 2026-08-26 at `1078c911f25784c44cd6b200706af4e7794eebe2`. Execute these followups from the merged default branch (`main`), rechecking the current implementation before changing it.

### Ranges and exposure examined

- Direct: `github.com/GoogleCloudPlatform/testgrid v0.0.123 → v0.0.176`. Examined all 53 intermediate release notes, the 889-commit comparison, and source changes against Prow's metadata, JUnit, configuration, GCS-path, and ResultStore usage. The original PR parent `2677f075a487a800c949d06e24305b1d7301c456` identifies the manifest diff; the merge-base with current `upstream/main` already includes this merged PR.
- Indirect: `bitbucket.org/creachadair/stringset v0.0.9 → v0.0.11`. Compared the versioned source and module metadata. Prow has no imports; Testgrid's bump removes a dependency on the unavailable `github.com/creachadair/staticfile` repository. No Prow migration identified.
- Indirect: `github.com/fvbommel/sortorder v1.0.1 → v1.1.0`. Compared the versioned source and README. The new capability is case-insensitive natural ordering in `casefolded`; Prow has no imports, and its Testgrid GCS-path usage does not call the sorting API. No Prow migration identified.
- Testgrid's other changes chiefly affect its updater, tabulator, API, and frontend. No new deprecation affecting a Prow call site or regression introduced by this bump was identified. `metadata.Finished.DeprecatedRevision` was already deprecated before this range.
- Sources: [complete Testgrid comparison](https://github.com/GoogleCloudPlatform/testgrid/compare/v0.0.123...v0.0.176), [v0.0.166 release](https://github.com/GoogleCloudPlatform/testgrid/releases/tag/v0.0.166), [JUnit metadata change c268e49](https://github.com/GoogleCloudPlatform/testgrid/commit/c268e49dbcf42b840a1483478d5193806b63e8c3), [stringset dependency rationale](https://github.com/GoogleCloudPlatform/testgrid/commit/1efa84c2d2).

### 1. Classify status-only skipped JUnit tests correctly

- Decision: accepted.
- Category: api-upgrade.
- Necessity: should — a supported upstream fixture with `status="skipped"` and no `<skipped>` element is currently displayed as passed. This behavior predates the bump; it is not a regression introduced by PR #900.
- Unlocked by: `github.com/GoogleCloudPlatform/testgrid v0.0.123 → v0.0.176`; `Result.Status` was added in v0.0.166.
- Changelog: “Update junit.go to allow more data to be extracted.” Commit `c268e49` adds the status attribute and a status-only skipped fixture.
- Where: `pkg/spyglass/lenses/junit/lens.go:218` (`JunitResult.Status`), `classifyTests`, and `pkg/spyglass/lenses/junit/lens_test.go`.

```text
Mode: deliver

In kubernetes-sigs/prow, PR #900 — "chore(deps): bump github.com/GoogleCloudPlatform/testgrid from 0.0.123 to 0.0.176" — bumped github.com/GoogleCloudPlatform/testgrid v0.0.123 → v0.0.176 and merged at 1078c911f25784c44cd6b200706af4e7794eebe2. Work from the merged default branch, main, and first check whether this followup has already been addressed.

Use the newly available junit.Result.Status in pkg/spyglass/lenses/junit/lens.go to classify <testcase status="skipped"> as skipped when it has no explicit outcome element. Testgrid v0.0.166's changelog says "Update junit.go to allow more data to be extracted"; commit c268e49dbcf42b840a1483478d5193806b63e8c3 adds Result.Status and a fixture for exactly this input. JunitResult.Status currently examines only Skipped, Failure, and Errored, so it classifies that input as passed. Access the embedded field as jr.Result.Status because jr.Status is the wrapper method.

Acceptance: honor the exact "skipped" attribute only when Skipped, Failure, and Errored are all absent; preserve existing explicit-element precedence and missing/unknown-attribute behavior. Add focused tests in pkg/spyglass/lenses/junit/lens_test.go for parsing through junit.Parse, default and configured-group counts, conflicting explicit elements, and repeated runs, preserving current rerun classification rules. Run go test ./pkg/spyglass/lenses/junit.

Scope: change only this classification behavior and its tests; do not add speculative mappings for other status strings, change the Testgrid dependency, or redesign grouping or rendering.
```

### 2. Let JUnit groups use suite-level properties

- Decision: accepted.
- Category: new-feature.
- Necessity: could — enables existing grouping to use metadata emitted once on a suite rather than requiring each testcase to repeat it.
- Unlocked by: `github.com/GoogleCloudPlatform/testgrid v0.0.123 → v0.0.176`; `Suite.Properties` was added in v0.0.166.
- Changelog: “Update junit.go to allow more data to be extracted.” The upstream commit explains: “Properties is available for testsuite, not for result in all prow jobs I checked.”
- Where: `pkg/spyglass/lenses/junit/lens.go:158` (`selectorPredicate.matches`), `lens.go:393` (recursive suite traversal), `GroupConfig`, `lens_test.go`, and `site/content/en/docs/spyglass/lenses/junit.md`.

```text
Mode: deliver

In kubernetes-sigs/prow, PR #900 — "chore(deps): bump github.com/GoogleCloudPlatform/testgrid from 0.0.123 to 0.0.176" — bumped github.com/GoogleCloudPlatform/testgrid v0.0.123 → v0.0.176 and merged at 1078c911f25784c44cd6b200706af4e7794eebe2. Work from the merged default branch, main, and first check whether this followup has already been addressed.

Add an opt-in way for JUnit lens groups to include enclosing suite properties. Testgrid v0.0.166 adds junit.Suite.Properties in commit c268e49dbcf42b840a1483478d5193806b63e8c3 ("Update junit.go to allow more data to be extracted"). Its rationale says: "Properties is available for testsuite, not for result in all prow jobs I checked." In pkg/spyglass/lenses/junit/lens.go, getJvd recursively visits suites but retains only junit.Result data; selectorPredicate.matches reads only Result.Properties.

Expose an optional include_suite_properties boolean on each GroupConfig, defaulting to false. For an opted-in group, match against effective properties inherited through enclosing suites: testcase values override suite values with the same name, and nearer suites override ancestors. Preserve testcase-only behavior for other groups. Keep inherited metadata separate from the original parsed testcase properties so one group's option cannot affect another group.

Acceptance: existing configurations retain their behavior; opted-in groups match suite-only properties, including nested suites, with the documented precedence and existing first-matching-group order. Add tests in pkg/spyglass/lenses/junit/lens_test.go for absent properties, conflicts, sibling isolation, mixed opted-in and ordinary groups, repeated tests, and group counts. Document the option and a suite-property XML example in site/content/en/docs/spyglass/lenses/junit.md. Run go test ./pkg/spyglass/lenses/junit.

Scope: limit changes to optional property inheritance for grouping, documentation, and focused tests; preserve selector syntax, status classification, deduplication rules, and unconfigured rendering.
```
