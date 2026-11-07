---
pr: kubernetes-sigs/prow#860
title: "chore(deps): bump github.com/andygrunwald/go-gerrit from 0.0.0-20210709065208-9d38b0be0268 to 1.1.1"
head_sha: 5d971769fe538f1fecbe2ef78735de5e6601762e
base: main
reviewed_at: 2026-08-18T20:17:12Z
verdict: approve
---

## What this PR does

- Bumps direct dependency `github.com/andygrunwald/go-gerrit` from an untagged pseudo-version (`v0.0.0-20210709065208-9d38b0be0268`) to the real tagged release `v1.1.1` (released 2025-12-05, upstream tag `refs/tags/v1.1.1`).
- Adapts `pkg/gerrit/client/client.go` to upstream's breaking change #153 (all `gerritX` service interfaces gained a leading `context.Context` param) by threading `context.Background()` through each call site.
- Adapts `SetReview` to upstream's breaking change #159 (`ReviewInput.Labels` changed from `map[string]string` to `map[string]int`) by converting caller-supplied `"+1"/"0"/"-1"` strings via `strconv.Atoi`.
- Fixes query-string separators from `+` to whitespace in `pkg/gerrit/client/client.go` (`queryChangesForProjectWithoutMetrics`) and `pkg/tide/gerrit.go` (`gerritDefaultQueryParam`/`gerritQueryParam`), since the new go-gerrit version percent-encodes literal `+` instead of treating it as the historical Gerrit term separator.
- Adds `TestQueryTermsReachGerritSeparated` (httptest-backed, asserts the literal query string Gerrit receives) and `TestSetReviewLabelVotes` (table test covering nil labels, valid conversion, and unparseable vote) plus updates existing string-builder tests to the new separator.

## Findings

None.

## Checked
- Dependency freshness: v1.1.1 tagged 2025-12-05, ~8.5 months old at review time — well past soak-time concerns, not a pseudo-version.
- Import surface: 6 non-vendor importers of `github.com/andygrunwald/go-gerrit`; only `pkg/gerrit/client/client.go` calls the low-level `gerritX` interfaces whose signatures changed. Other importers (`pkg/gerrit/adapter/{trigger,adapter}.go`, `pkg/gerrit/fakegerrit/fakegerrit.go`, `pkg/crier/reporters/gerrit/reporter.go`, `pkg/tide/codereview.go`, `test/integration/cmd/fakegerritserver/main.go`) go through the higher-level `Client` wrapper whose exported method signatures are unchanged, so they needed no updates.
- Upstream changelog: 92 commits between old and new version; the only two breaking changes (#153 context support, #159 label votes as ints) are exactly the two adaptations made in this PR. No CVEs or security-relevant fixes in the range.
- `strconv.Atoi` correctly parses `"+1"` (leading `+` sign is valid), so the label-vote conversion handles the existing caller format without a translation layer.
- Query-separator fix is verified end-to-end: `TestQueryTermsReachGerritSeparated` spins up a real `httptest.Server` and asserts the raw query string Gerrit receives (`"(branch:foo OR branch:bar) project:myproject"`), not just the string-builder helper output.
- No other call sites of the changed low-level interfaces exist outside `pkg/gerrit/client/client.go` and its test.

## Open questions

None.

## Dependency followups

Bump examined: `github.com/andygrunwald/go-gerrit` `v0.0.0-20210709065208-9d38b0be0268` -> `v1.1.1` (direct; the only module in the PR diff, and `go.mod`/`go.sum` show no transitive version changes). Current `main` has since moved to `go-gerrit v1.2.0`; the context API and the cited line numbers are unchanged there.

Accepted 1, skipped 0.

### Thread request contexts through the Gerrit client

- category: api-upgrade
- module: `github.com/andygrunwald/go-gerrit` `v0.0.0-20210709065208-9d38b0be0268` -> `v1.1.1` (upstream #153, "Add golang context support")
- where: `pkg/gerrit/client/client.go:317,397,426,443,460,505,523,554,571,605,721,843`; consumers `pkg/crier/reporters/gerrit/reporter.go:272`, `pkg/gerrit/adapter/adapter.go:187,366`, `pkg/tide/gerrit.go:97`
- necessity: should — `context.Background()` is the stopgap introduced by the bump; the crier reporter's 10s `newCtx` (`reporter.go:318`) bounds only the k8s `List` at `:342`, not its Gerrit `GetChange`/`ChangeExist`/`SetReview` calls (`reporter.go:396-457`), so a hung Gerrit API call blocks `Report` indefinitely. Before v1.1.1 go-gerrit had no ctx parameters at all.
- handoff prompt:

```text
In kubernetes-sigs/prow, now that PR #860 (merged as 89129737dc106ed8bcd73a8689e143ccea5ff67c) bumped
github.com/andygrunwald/go-gerrit from v0.0.0-20210709065208-9d38b0be0268 to v1.1.1 (upstream #153,
"Add golang context support"), migrate the Gerrit client off the context.Background() stopgap.

What and where
- pkg/gerrit/client/client.go has 11 context.Background() sites: NewClient construction at :317,
  the logrus.WithContext at :397, and the go-gerrit calls at :426 (GetChange), :443 (SubmitChange),
  :460 (ChangeExist), :505 (SetReview), :523 (GetBranch), :554 (GetAccount), :571 (GetMergeable),
  :605 (ListChangeComments), :721 (QueryChanges), :843 (GetRelatedChanges).
- Consumers: pkg/crier/reporters/gerrit/reporter.go (Report at :272 builds newCtx with a 10s
  timeout at :318 but only uses it for pjclientset.List at :342; GetChange/ChangeExist/SetReview
  at :396-457 take no ctx), pkg/gerrit/adapter/adapter.go (NewController at :187 receives a ctx but
  only forwards it to client.NewSyncTime; Sync workers at :366 and processSingleProject at :290 have
  none), pkg/tide/gerrit.go (NewGerritController at :97 creates ctx := context.Background() at :118;
  GerritProvider methods at :197-433 take no ctx).
- Test fakes that implement the changed signatures: pkg/gerrit/client/client_test.go,
  pkg/gerrit/adapter/adapter_test.go, pkg/tide/gerrit_test.go, pkg/crier/reporters/gerrit/reporter_test.go.

Task
1. Add ctx context.Context as the first parameter to the exported Client methods in
   pkg/gerrit/client/client.go (QueryChanges, QueryChangesForProject, GetChange, SubmitChange,
   ChangeExist, SetReview, GetBranchRevision, Account, GetMergeableInfo, HasRelatedChanges) and to
   the internal helpers that make the calls (injectPatchsetMessages,
   queryChangesForProjectWithoutMetrics, QueryChangesForInstance, queryAllChanges as needed).
   Replace each context.Background() with the passed ctx. NewClient at :317 may keep
   context.Background() (it is the auth probe run at handler construction) - if kept, add a comment
   saying so.
2. Crier reporter: pass its existing newCtx to the Gerrit calls in Report (:396-457) so the 10s
   deadline actually applies; use the ctx passed to ShouldReport where it calls the Gerrit client.
3. Gerrit adapter: store the ctx given to NewController on Controller and pass it to the Sync
   worker goroutines, processSingleProject, processChange, triggerJobs and the other paths that
   call Client methods.
4. Tide: store a ctx on GerritProvider (extend newGerritProvider at :167 and pass the controller ctx
   from NewGerritController) and use it for Client calls. Do not change Tide's shared provider
   interface.
5. Update the test fakes to the new signatures.

Acceptance criteria
- go build ./... passes, and go test ./pkg/gerrit/... ./pkg/tide/... ./pkg/crier/reporters/gerrit/... passes.
- pkg/gerrit/client/client.go contains no context.Background() except, at most, the documented
  NewClient construction.
- A reporter test with a fake Gerrit client asserts the context handed to GetChange/SetReview
  carries the 10s deadline established in Report (e.g. by checking Deadline()/Done() in the fake).
- Queries, votes and messages are unchanged; this is a plumbing change only.

Scope guard
- Do not change Tide's shared provider interface or any GitHub code path.
- Do not alter exported method semantics beyond adding the ctx parameter, and do not refactor
  unrelated code.
- Repo main may already be on go-gerrit v1.2.0; the context API is identical there, so make the
  change against current main.
```
