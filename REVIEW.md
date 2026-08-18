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
