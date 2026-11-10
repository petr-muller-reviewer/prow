---
issue: kubernetes-sigs/prow#872
title: "Git fetch will fail when tag was moved"
state: open
labels:
main_sha: 5765df26189ddc41cb9c975e89d613f6c26282ef
triaged_at: 2026-09-02T11:31:17Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 1
recommended_labels: [kind/bug, area/pod-utils, good-first-issue]
---

## Initial validation

**Assessment**: LEGITIMATE

This is a specific, reproducible bug in Prow's `clonerefs` utility, not a configuration request. The report includes the exact failed command, stderr, an affected public job, and source location. `clonerefs` and its clone-command construction are maintained here.

**Issue category**: Bug

**Repository scope check**:
- Component: `clonerefs` / pod utilities.
- Exists in this repo: Yes — `cmd/clonerefs/main.go`, `pkg/clonerefs`, and `pkg/pod-utils/clone`.
- Relevant code: `pkg/pod-utils/clone/clone.go:201-284`; `pkg/clonerefs/run.go:40-158`.

**Information completeness**: sufficient. A preexisting workspace holding the old tag, plus a remote that retargets it, reproduces the failure.

**Recommendation**: Keep open and fix as a localized clonerefs bug. No information request is needed.

## Research findings

### Current implementation

- `cmd/clonerefs/main.go:27-40` loads options and exits fatally if `Options.Run` returns an error.
- `pkg/clonerefs/run.go:40-158` prepares auth/environment, calls `clone.Run` in parallel for configured `prowapi.Refs`, writes clone records, and fails when configured to do so and any record failed.
- `pkg/pod-utils/clone/clone.go:39-125` runs base-reference commands, then pulls/submodules, recording the first failure.
- `pkg/pod-utils/clone/clone.go:227-304` initializes Git, performs an all-tags fetch except for `SkipFetchHead` or sparse checkout, then fetches and checks out `BaseSHA`/`BaseRef`.

The all-tags fetch is additive. The later targeted fetch resolves the checked-out ref; sparse checkout skips the all-tags fetch and passes `--no-tags` to the targeted fetch.

### Root cause and reproduction

Git rejects an existing local tag when `git fetch <uri> --tags --prune` advertises the same tag name at another object. Prow requests every tag without `--force` or a forced tag refspec, so a moved `hcpctl-nightly` tag stops cloning before the targeted base-ref fetch. Each fetch goes through `retryCommand`, adding 64.5 seconds of backoff before this deterministic failure is recorded.

### Related code and tests

- `pkg/pod-utils/clone/clone.go:201-220,390-412`: retries every fetch without classifying deterministic errors.
- `pkg/pod-utils/clone/clone_test.go:62-864`: table tests assert generated commands for regular, auth, depth, blobless, multihead, and sparse cases; non-sparse cases include `--tags --prune` (first at `73-99`).
- `pkg/pod-utils/clone/clone_test.go:1024-1079`: tests retry counts with a fake runner, not Git tag updates.
- `test/integration/test/pod-utils_test.go:47+`: tests decorated-pod cloning via fake Git, but not tag retargeting.
- `site/content/en/docs/components/pod-utilities/clonerefs.md:8-72`: documents records/options, not tag-fetch semantics.

The filter-aware construction was touched by `6d00893891`; sparse checkout was later made to skip tag fetch in `43e2154a6`. No related PR or duplicate issue was found.

### Proposed solutions

#### Approach 1: Force the existing all-tags fetch (recommended)

Add `--force` (or an equivalent forced `refs/tags/*` refspec) to the all-tags fetch. This retains tags and pruning for ordinary jobs while refreshing intentionally moved tags. Update command-plan expectations and add a real-Git moved-tag regression test. It is low risk, with the intentional behavior that a job sees the remote tag rather than retaining a stale local tag.

#### Approach 2: Remove or make all-tag fetch opt-in

Rely only on targeted base-ref fetching. This avoids collision and transfer, but changes implicit tag availability for ordinary job steps and has higher compatibility risk.

#### Approach 3: Avoid retrying deterministic Git errors

This improves latency but requires error classification and does not allow moved tags, so it is separate follow-up work.

**Recommendation**: Approach 1, the smallest change matching the report and existing tag-availability behavior.

## Effort assessment

**Effort level**: 1 — Easy change

The preferred fix is a local argument change plus existing table-test updates; a behavioral test is the only added design work.

- **Scope**: Small — mainly `pkg/pod-utils/clone/clone.go` and `clone_test.go`, under 100 LOC; level 1.
- **Complexity**: Simple — Git behavior and the failing invocation are explicit; level 1.
- **Required expertise**: Minimal — Go table tests and basic Git fetch semantics; level 1.
- **Clarity**: Well-defined — retain tag fetching but permit a moved tag; level 1.
- **Testing**: Simple/partial — existing tests cover variants; real-Git retarget scenario is missing; level 1.
- **Backward compatibility**: Intentional minor change — remote tag replaces stale local tag; level 1.
- **Architectural alignment**: Good fit in clone command construction; level 1.
- **External dependencies**: Well-supported Git behavior; level 1.

### Recommended labels

- [x] `kind/bug`: incorrect clonerefs behavior.
- [x] `area/pod-utils`: affected code is `pkg/pod-utils/clone`.
- [x] `good-first-issue`: isolated change, clear tests, and no deep Prow expertise required.

### Guidance for contributors

Follow `TestCommandsForRefs` in `pkg/pod-utils/clone/clone_test.go`; preserve sparse and `SkipFetchHead` exceptions. A behavioral test should fetch a tag, retarget it in a local/bare remote, then verify forced tag fetch succeeds and updates it.

## Briefing summary

Completed the seven-slide maintainer briefing: the issue is a legitimate, localized clonerefs bug; the all-tags prefetch lacks force semantics; generic retry adds avoidable delay; forcing only that prefetch is preferred; and the work is level 1 with `kind/bug`, `area/pod-utils`, and `good-first-issue` recommended.

## Findings

### [reproducibility] Explicit tag fetch rejects a moved local tag
- detail: The reported `git fetch <repo> --tags --prune` exits nonzero when the remote tag moved and a local tag of that name points elsewhere.
- evidence: kubernetes-sigs/prow#872; `pkg/pod-utils/clone/clone.go:257-264`.

### [cause] clonerefs fetches all tags without force semantics
- detail: `commandsForBaseRef` emits an all-tags fetch before the targeted base-ref fetch, but supplies neither `--force` nor a forced tag refspec.
- evidence: `pkg/pod-utils/clone/clone.go:257-284`.

### [cause] Generic retry delays a deterministic failure
- detail: Fetch retries total 64.5 seconds, then `Run` records failure before checkout.
- evidence: `pkg/pod-utils/clone/clone.go:39-125`, `pkg/pod-utils/clone/clone.go:201-220`, `pkg/pod-utils/clone/clone.go:390-412`.

### [related-code] Targeted and sparse paths do not require tag prefetch
- where: `pkg/pod-utils/clone/clone.go:257-284`
- excerpt: |
    skipTagFetch := refs.SkipFetchHead || sparseCheckoutSet
    if !skipTagFetch {
        fetchArgs = append(fetchArgs, g.repositoryURI, "--tags", "--prune")
    }
    if sparseCheckoutSet {
        fetchArgs = append(fetchArgs, "--no-tags")
    }
    fetchArgs = append(fetchArgs, g.repositoryURI, fetchRef)

### [related-code] Tests cover command construction but not tag retargeting
- where: `pkg/pod-utils/clone/clone_test.go:62-864`, `test/integration/test/pod-utils_test.go:47+`
- detail: Existing tests assert invocations and normal decorated clones; none retarget a tag after an earlier fetch.

### [related-issue] Git default protects tags but Prow can override deliberately
- ref: kubernetes-sigs/prow#872
- relevance: A maintainer notes tags are generally immutable; Prow may explicitly select different behavior for an all-tags refresh.

## Checked
- Issue report, comment, state, labels, events, and linked work via GitHub API.
- `cmd/clonerefs`, `pkg/clonerefs`, `pkg/pod-utils/clone`, CRD/docs, tests, and command history.
- No fix PR or duplicate issue found.

## Next steps
- Complete the mandatory briefing, then apply recommended labels manually if present in repository taxonomy.
- Force only the all-tags invocation while retaining `--tags --prune`.
- Update table tests and add moved-tag regression coverage if practical.
- Treat deterministic-error retry classification as separate follow-up.

## Open questions
- Is all-tag availability an intentional compatibility contract, or could it later become opt-in?
- Should force semantics be configurable, or apply to every fetched tag?
