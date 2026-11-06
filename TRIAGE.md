---
issue: kubernetes-sigs/prow#1043
title: "clonerefs: blobless+sparse-checkout PR merge fails with 'upload-pack: not our ref' and SIGSEGV after the 2026-10-23 image bump"
state: closed
labels: ""
main_sha: 091a7d73761d32baf5b1313f9a517d8c222fc5aa
triaged_at: 2026-11-04T17:25:20Z
verdict: resolved
legitimacy: LEGITIMATE
effort: 3
recommended_labels: [kind/bug, kind/regression, priority/important-soon]
refresh_log:
  - at: 2026-11-04T17:25:20Z
    since: 2026-10-31T20:16:26Z
    summary: "issue closed as completed citing merged #1054 (merge-tree pre-write workaround); #1045 closed in favour of it; main re-anchored to 091a7d737"
---

# Triage

## Verdict

**accepted → resolved** — a real regression in the `clonerefs` init container, caused by the git shipped in the 2026-10-23 base-image bump, not by any `clonerefs` code change.

Legitimate on every axis: the component lives in this repo (`cmd/clonerefs`, `pkg/pod-utils/clone`), the report carries the exact commands, image tag, blob SHAs, two independent repos and `clone-records.json` links, and the failure is isolated to this repo's own image bump (#1036) — the same PR head passed hours earlier and no clone-path commit landed in between. Root cause: sparse checkout implies `--filter=blob:none`, so git may lazily fetch an object it just created; the git 2.54.0 in Alpine 3.24.3 asks the origin for a merge-result blob that only exists locally, gets `upload-pack: not our ref`, and segfaults instead of failing cleanly. The pre-bump image (git 2.52.0) and the reporter's exact recipe under git 2.56.0 both succeed. Effort is Level 3: the patch is small, but it is the clone path every job's init container uses and the underlying defect is external. The base-image decision (`alpine:3.23.7` now vs `edge`/`3.25.0` later) and the recovery-guard question (#1045 was closed in favour of the merged workaround) are no longer blocking; what remains is shipping a fixed git so the workaround can be removed.

Resolved on 2026-11-04: #1054 shipped a client-side workaround for partial clones and the issue was closed as completed — see `## Resolution`. The verdict is now `resolved`; the base-image and workaround-removal follow-ups remain open.

## Resolution

Resolved by #1054 "fix(clonerefs): write merge results before merging partial clones", merged 2026-11-04T02:06:40Z as `5bf71c288` (change `40cea8c67`). Issue closed 2026-11-04T16:31:41Z by petr-muller as COMPLETED with `/close — I believe this is resolved with #1054`.

**Classification: sufficient** — as a workaround for the triaged cause, not as a git fix.

- Matches the recorded cause: for partial clones (`BloblessFetch` or `SparseCheckoutFiles`) it runs `git merge-tree --write-tree HEAD <pull>` before `git merge`, so the merged blobs are written by a separate process and are on disk before `git merge` fills its loose-object cache — exactly the "merge writes a blob, the cache misses it, a later bulk prefetch asks the server" mechanism in the findings. The step's result is swallowed by `bestEffortCommand`; `git merge` still reports real problems.
- Broader than the triaged remedy and cheaper than #1045: it also covers blobless full checkouts and needs no re-clone, which is why #1045's full-fetch retry was dropped in its favour.
- Verified against the cases this triage cited: `kubernetes/kubernetes#143721`, `kubevirt/hyperconverged-cluster-operator#4681` and `Azure/ARO-HCP#7828` served without their merged blobs — stock clonerefs fails on all three, the branch passes; `TestRunMergesInPartialClone` fails on git 2.54.0 without the step and passes with it (2.47.3 and 2.52.0 unaffected).
- Not fixed: git 2.54.0 is still the pod-utils base image (`.ko.yaml` pin unchanged on `091a7d737`) and the segfault itself is not root-caused. The PR states it is "a workaround, to be removed once clonerefs ships a fixed git".

## What the issue reports

- Since the 2026-10-23 utility-image rebuild, jobs whose refs set `sparse_checkout_files` fail in the `clonerefs` init container before any test runs — the same PR head that passed earlier the same day.
- The merge succeeds (`Merge made by the 'ort' strategy.`), then git tries to fetch a blob it just created, gets `fatal: remote error: upload-pack: not our ref <sha>`, and dies with `signal: segmentation fault (core dumped)`.
- Trigger is a merge producing new content for an out-of-cone (skip-worktree) path, i.e. the same file changed on the PR and on the base since the merge base; trivial merges pass, so failures look intermittent and track base-branch activity rather than the PR.
- Affects at least Azure/ARO-HCP (#7828, #7662) and kubevirt/hyperconverged-cluster-operator (#4681); the reporter says `sparse_checkout_files` is used by presubmits across many OpenShift repos (origin, os, api, oc, hive, quay, stolostron/ocm, medik8s/common).
- Asks: roll back the git/clonerefs image, say whether dropping `sparse_checkout_files` is the recommended interim mitigation, and point at the right upstream tracking location.

**Since previous triage (2026-10-31):** the issue was closed 2026-11-04 as completed; #1054 merged a partial-clone workaround (pre-writing merge results with `merge-tree --write-tree`); #1045 was closed in favour of it; no affected repo has reported recurrence since.


## Findings

### [reproducibility] exact recipe fails only on the new base image, passes on 2.56.0
- detail: The reporter's recipe (ARO-HCP base `4ce838f7293c…`, PR head `e006fbbf7f45…`, 13 sparse paths, `--filter=blob:none`, `git merge --no-ff`) exits 0 with git 2.51.1 on a workstation and, verified during triage, with git 2.56.0 here: diffstat printed, merge-result blob present and readable locally, no remote contact. It fails only inside the `v20261023-*` image.
- evidence: `git merge --no-ff e006fbbf7f45ef2398c29613d7d256b8c6bb4261` run locally, exit 0; failing runs https://storage.googleapis.com/test-platform-results-public/pr-logs/pull/Azure_ARO-HCP/7828/pull-ci-Azure-ARO-HCP-main-test-unit/2115807042240253952/clone-records.json and https://storage.googleapis.com/test-platform-results-public/pr-logs/pull/kubevirt_hyperconverged-cluster-operator/4681/pull-ci-kubevirt-hyperconverged-cluster-operator-main-hco-e2e-upgrade-prev-operator-sdk-sno-azure/2115671286276427776/clone-records.json

### [reproducibility] same PR head passed before the bump on a trivial merge
- detail: ARO-HCP PR #7828 head `e006fbbf` passed `pull-ci-Azure-ARO-HCP-main-e2e-parallel` earlier on 2026-10-23; the later `test-unit` run of the same head failed. Consistent with "needs overlapping changes on both sides", not with PR content.
- evidence: https://prow.ci.openshift.org/view/gs/test-platform-results-public/pr-logs/pull/Azure_ARO-HCP/7828/pull-ci-Azure-ARO-HCP-main-e2e-parallel/2115727005218508800

### [cause] git regression pinned to 5946a564, mechanism confirmed
- detail: `git merge` in a partial clone writes the merged blob but its loose-object cache does not see the write, so a later bulk prefetch asks the promisor remote for it; the server rejects the whole fetch (`upload-pack: not our ref`). Bisected to git `5946a564` by the author of #1054, with an upstream report on the git mailing list. The segfault on the rejected fetch is still not root-caused.
- evidence: kubernetes-sigs/prow#1054 description and https://lore.kernel.org/git/yrjh1p92qc7w.fsf@pohly-mobl1.fritz.box/T/#u; `TestRunMergesInPartialClone` fails only on git 2.54.0 (passes on 2.47.3 and 2.52.0 without the workaround)

### [cause] the base image itself is brand new, and no newer stable Alpine exists yet
- detail: `alpine:3.24.3` was pushed 2026-10-23T05:24Z — the same day test-infra#38144 merged (20:02Z) and #1036 landed (21:29Z) — so this is a same-day point-release regression, not stale drift. The 3.24 line started 2026-06-10; the replaced image used `3.23.4` (2026-04-15). `3.23.7` was published the same day and still ships git 2.52.0. There is no `3.25-stable` branch, no `3.25`/`4.x` tag, `latest` is still 3.24.3, and Alpine cuts release branches each May and November.
- evidence: Docker Hub `library/alpine` tag push times; aports branches `3.24-stable`, `master` only; kubernetes/test-infra `images/git/Dockerfile` (`FROM alpine:3.24.3`)

### [cause] sparse checkout implies a blobless partial clone with the PR URL as promisor remote
- detail: `sparseCheckoutFiles` forces `--filter=blob:none` on both the base and the PR fetch, so every sparse job is a partial clone; the merge result of a content merge is a brand-new object that no promisor remote can serve.
- evidence: `pkg/pod-utils/clone/clone.go:253-255`, `pkg/pod-utils/clone/clone.go:355-360`

### [cause] no retry or fallback in the clone path
- detail: `Run` executes the generated argv in order, records each command in `Record.Commands`, and aborts on the first failure — no error classification, no alternative fetch strategy, no notion of a recoverable failure.
- evidence: `pkg/pod-utils/clone/clone.go:79-105`

### [cause] the bump arrived automatically from a repo prow does not control
- detail: test-infra rebuilds `gcr.io/k8s-staging-test-infra/git` from `images/git/Dockerfile` and republishes `:latest`; the hourly `ci-prow-autobump-base-images` job runs `generic-autobumper --config=hack/autobump.yaml` (`targetVersion: latest`, prefix `gcr.io/k8s-staging-test-infra`, `extraFiles: [.ko.yaml]`), which produced #1036. A hand-edited `.ko.yaml` pin is therefore undone on the next run unless `targetVersion` is pinned to an explicit tag — which applies to every staging image, not just `git`.
- evidence: `hack/autobump.yaml`, `.ko.yaml`, kubernetes/test-infra `config/jobs/kubernetes/sig-k8s-infra/trusted/sig-k8s-infra-prow.yaml` (`ci-prow-autobump-base-images`)

### [cause] no end-to-end coverage of blobless+sparse+overlapping-history merges
- detail: Sparse coverage is argv assertions only; the integration suite never exercises a sparse clone. Nothing in the repo would have caught this (it only manifests on a specific git version), and nothing documents the partial-clone coupling or a minimum git version — `SparseCheckoutFiles` is described solely as "limits the working tree to only the listed paths".
- evidence: `pkg/pod-utils/clone/clone_test.go:610-690` (argv only), zero occurrences of "sparse" in `test/integration/test/pod-utils_test.go`, `pkg/apis/prowjobs/v1/types.go` field comment

### [related-code] sparse-checkout command builders
- where: `pkg/pod-utils/clone/clone.go:242-244`, `:253-255`, `:257`, `:272-290`, `:296-308`, `:348-385`
- excerpt: |
    sparseCheckoutSet := isSparseCheckoutSet(refs)
    if sparseCheckoutSet {
    	commands = append(commands, g.gitCommand("sparse-checkout", "init"))
    }
    ...
    if (refs.BloblessFetch != nil && *refs.BloblessFetch) || sparseCheckoutSet {
    	filterArgs = append(filterArgs, "--filter=blob:none")
    }
    ...
    gitMergeCommand := g.gitCommand("merge", "--no-ff", prCheckout)

### [related-code] no clone-path change in the regression window
- where: commits `43e2154a6` ("feat(clonerefs): add sparse-checkout mode"), `7b19dc1d0` ("fix(clonerefs): remove cloneDepth=1 from sparseCheckout")
- excerpt: |
    git log -- pkg/pod-utils/clone/clone.go shows nothing between those commits and the 2026-10-23 bump, so the behaviour change in the window is external.

### [related-code] merge-tree pre-write step in the clone command list
- where: `pkg/pod-utils/clone/clone.go:378-386` (`40cea8c67`), `pkg/pod-utils/clone/clone.go:421-437` (`bestEffortCommand`)
- excerpt: |
    if partialClone {
    	// Works around a regression in git 2.54 and later: git merge can ask
    	// the server for a blob it just wrote, which fails the merge. ...
    	commands = append(commands, bestEffortCommand{g.gitCommand("merge-tree", "--write-tree", "HEAD", prCheckout)})
    }

### [related-code] base image pin and previous tag
- where: `.ko.yaml:5-40` (`baseImageOverrides`), `.prow-images.yaml:31` (`cmd/clonerefs`), `Makefile:126` (`push-images`)
- excerpt: |
    sigs.k8s.io/prow/cmd/clonerefs: gcr.io/k8s-staging-test-infra/git:v20261023-d6356df2e8
    (previous, pre-regression: gcr.io/k8s-staging-test-infra/git:v20260512-1f34ef0df3)

### [related-pr] #1054 merged the workaround that resolves this issue
- ref: kubernetes-sigs/prow#1054
- relevance: Merged 2026-11-04T02:06:40Z as `5bf71c288` (change `40cea8c67`), +167/−2 across `pkg/pod-utils/clone/clone.go` and `clone_test.go`. Runs `merge-tree --write-tree` before `git merge` for blobless or sparse clones and ignores its result; adds `TestRunMergesInPartialClone`. Self-described workaround, removable once a fixed git ships.

### [related-pr] #1045 recovers failed sparse merges with a complete fetch
- ref: kubernetes-sigs/prow#1045
- relevance: Open at `66eb058c0c9e`, CI green (unit, integration, lint, image build), mergeable, no review decision. Extracts `runClone`, adds `shouldFallBackToFullFetch` (sparse set, last command is `git merge --no-ff`, output has no `CONFLICT`/`Automatic merge failed`), removes the failed clone dir, re-runs with `g.fullFetch`, keeps the failed commands and sets `Record.FellBackToFullFetch` (rendered as `# Sparse checkout merge failed, retried with a full object fetch`). Its integration test injects the failure with a `git` shim because a working git cannot reproduce it. Closed 2026-11-04T08:06:31Z by its author (`/close in favor of https://github.com/kubernetes-sigs/prow/pull/1054`) after cblecker suggested #1054 covers it; never merged.

### [related-issue] OpenShift CI tracks the same failure as DPTP-5469
- ref: private JIRA DPTP-5469 (author comment 2026-10-29T19:53:44Z)
- relevance: Their mitigation ask is pinning the `/pod-utils` utility image tag on OpenShift build clusters; they note any Prow instance on the new base image is equally affected.

## Resolved

### [cause] git regression between 2.52.0 and 2.54.0, apparently fixed by 2.55+
- resolved: 2026-11-04 — the regression is pinned to git `5946a564` and the failure mechanism is confirmed (see the `[cause] git regression pinned to 5946a564` finding), so the "exact commit unidentified / inference only" caveat no longer applies.
- prior detail: Alpine 3.23.x ships git 2.52.0 (pre-bump image), Alpine 3.24.3 ships git 2.54.0 (post-bump image), Alpine `edge`/aports `master` ships git 2.56.0; evidence pointed at the promisor lazy-fetch path treating a locally written merge-result blob as missing, then segfaulting on the rejected fetch.

## Checked

- Duplicate search: `gh issue list --search "sparse blobless merge"` and `"clonerefs sparse"` return only #1043; no older duplicate.
- Timeline (paginated), as of the 2026-10-31 triage: one `cross-referenced` event (#1045) and one author comment (DPTP-5469); no maintainer response, no labels.
- `git log -- pkg/pod-utils/clone/clone.go`: no clone-path commit between the sparse feature/`cloneDepth` fix and the bump.
- `.ko.yaml` history: previous pin `v20260512-1f34ef0df3` (2026-05-12), so a rollback target exists.
- Local reproduction with the reporter's exact SHAs under git 2.56.0: passes. Could not reproduce the failure — no container runtime and no zlib/curl dev headers to build git 2.52/2.54, so "broken on 2.54.0" rests on the reporter's image plus the Alpine package mapping.
- Base image age: `alpine:3.24.3` pushed 2026-10-23T05:24Z; `3.24.0` 2026-06-10; `3.23.4` 2026-04-15; `3.23.7` 2026-10-23 (still git 2.52.0).
- Alpine release line: no `3.25-stable` branch, no `3.25`/`4.x` tags, `latest` = 3.24.3, `edge` is the only newer stream; release branches each May and November.
- Test coverage: zero occurrences of "sparse" in `test/integration/test/pod-utils_test.go`; sparse unit tests assert argv only.
- Label set: `kubernetes-sigs/prow` has no `area/pod-utils`; `kind/bug`, `kind/regression`, `priority/important-soon` all exist.
- Refresh at 2026-11-04: timeline since 2026-10-31T20:16:26Z shows `cross-referenced` #1054 (2026-11-04T01:21:41Z), the maintainer's `/close` comment (16:31:34Z) and the `closed` event (16:31:41Z), plus the bot's close comment; no other activity.
- #1045 ended CLOSED on 2026-11-04T08:06:31Z (author self-close in favour of #1054), never merged.
- Upstream main moved from `c06373a3e` to `091a7d737`; the only clone-path change in that range is `40cea8c67` (#1054). `.ko.yaml` still pins `gcr.io/k8s-staging-test-infra/git:v20261023-d6356df2e8` (git 2.54.0), so the workaround is still load-bearing.
- No post-merge comment from ARO-HCP, kubevirt or OpenShift CI claiming the failure persists.
- Briefed the maintainer in slide form on 2026-10-31 (overview, legitimacy, root cause, technical detail, solution, effort, recommendations) plus follow-ups on how a base-image bump propagates, the age of `alpine:3.24.3`, available Alpine versions, and whether a fixed git alone is enough. Maintainer subsequently closed the issue as resolved via #1054 (2026-11-04).

## Next steps

- Nothing blocking: the issue is closed and the workaround merged. Watch the previously failing jobs (ARO-HCP `test-unit`, kubevirt HCO) for a repeat of `upload-pack: not our ref`; if one shows up, this needs a full re-triage rather than another workaround.
- Still open long-term: move the pod-utils base image off git 2.54.0 (`alpine:3.25` when the branch is cut, or `edge`), then delete the `merge-tree --write-tree` step and its comment in `commandsForPullRefs` (`clone.go:378-386`) and drop the `bestEffortCommand` shim if nothing else uses it; keep `TestRunMergesInPartialClone` as the regression test.
- Consider an integration test that runs in the clonerefs image: CI's `kubekins-e2e` carries git 2.47.3, so `TestRunMergesInPartialClone` passes there with or without the workaround and cannot catch its removal.
- Check the cost: the step runs for every pull merge in every blobless or sparse job (a second merge computation per PR), which is new overhead on large repos.
- Optionally answer the reporter's ask #3 on the closed issue: the upstream tracking is the git mailing-list thread linked from #1054, and the base-image decision lives in kubernetes/test-infra `images/git/Dockerfile`.
- Labels: the issue is closed with none applied; `kind/bug` and `kind/regression` remain unset if the project wants bookkeeping.

## Open questions

- When will a fixed git land in the pod-utils base image so the `merge-tree --write-tree` workaround can be removed, and should prow track that with its own issue?
- Is the extra `merge-tree --write-tree` per pull merge acceptable on large repos long term, or does it need a cheaper equivalent once git is fixed?
- Should `clonerefs` surface a clearer diagnostic when a promisor fetch of a locally created object fails, given the segfault itself is still unexplained upstream?
