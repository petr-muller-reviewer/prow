---
pr: kubernetes-sigs/prow#1057
title: "Stop treating plank.job_queue_capacities as global only"
head_sha: bb5631aa3b932b3f1043e1ca46155d2f844e8a88
base: main
reviewed_at: 2026-11-04T16:23:20Z
verdict: approve
---

# Review

## Verdict
Approve. The one-line classifier fix is correct and well covered by regression tests.

`hasGlobalConfig()` builds a reference config containing only the fields that are legitimately allowed to be global; anything outside that reference marks a shard as global, and `checkconfig` rejects global content inside org/repo directories. #1051 made `plank.job_queue_capacities` mergeable from supplemental shards but did not teach this classifier about it, so a valid repo shard that declared capacities was rejected with "contains global config". Adding `Plank{JobQueueCapacities: ...}` to the reference fixes exactly that. I verified both new tests fail without the added line and pass with it, and the full `pkg/config` and `cmd/checkconfig` suites pass.

## What this PR does
- Adds `Plank{JobQueueCapacities: ...}` to the reference config in `ProwConfig.hasGlobalConfig()`.
- Adds a `checkconfig` hierarchy regression case for a repo shard carrying `job_queue_capacities`.
- Adds a `TestHasConfigFor` fuzz case asserting that capacities alone are not global, and clears capacities in the existing "any non-empty config is global" fuzz case.
- Is a follow-up to #1051, which added the actual merge logic for the field.

## Findings

### [nit] Capacities-only shard produces an error message ending in a dangling "but"
- where: `cmd/checkconfig/main.go:1444-1467`
- concern: With capacities no longer classified as global, a shard whose only content is `plank.job_queue_capacities` reaches the repo/org scope error with `isGlobal == false`, no orgs and no repos, so the message reads `... Must only contain config for repo my-org/my-repo, but` with nothing after "but". Such a shard was rejected before as well (as global), so acceptance does not regress, but this is now the natural first thing a user hits when trying the newly allowed field alone. Appending something like "contains no org- or repo-scoped config" when no other reason matched would make the message self-explanatory.
- excerpt: |
			if !isGlobal && len(targetedOrgs) == 0 && len(targetedRepos) == 1 && targetedRepos.Has(expectedTargetRepo) {
				return nil
			}

## Checked
- Verified the new tests are meaningful: deleting the added `Plank:` line makes `TestHasConfigFor/Job_queue_capacities_alone_are_not_considered_global` fail with `expected isGlobal: false, got: true`, and the new checkconfig case fail with `contains global config`.
- `GOEXPERIMENT=jsonv2 go test ./pkg/config ./cmd/checkconfig -count=1` passes (the installed Go toolchain gates `encoding/json/jsontext` behind the `jsonv2` experiment).
- `gofmt -l` is clean on all three changed files.
- The top-level global config is unaffected: `validateAdditionalConfigIsInOrgRepoDirectoryStructure` returns early at depth 0, so the classifier only governs org/repo shards.
- Runtime loading is unchanged: `loadConfig` still calls `ProwConfig.mergeFrom` for every shard, merging capacities into one plank map and rejecting duplicate queue names.
- No user-facing documentation claims the field is global-only (nothing under `site/` mentions `job_queue_capacities`), so no docs update is required.

## Open questions
- A repo shard can now declare capacities without failing hierarchy validation, but the capacity is still applied to the single global queue map: any job in another repo using the same queue name shares the limit. Is that acceptable for the shards this unblocks, or should queue names become repo-scoped before users start declaring per-repo capacities?
