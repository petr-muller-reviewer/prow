---
pr: 549
title: "Allow filtering jobs by owner, repo and org"
head_sha: c8dcb67a645da3c36355bb97c8c1490198a82e18
base: main
head_sha: 3f1a6cb9f78ed401aea530aefa8634a91a27559e
reviewed_at: "2026-07-26T23:33:22Z"
verdict: approve-with-suggestions
state: merged
gate:
  decision: merge
  gated_at: "2026-06-30T11:57:44Z"
  gated_head_sha: 3f1a6cb9f78ed401aea530aefa8634a91a27559e
  reviewed_head_sha: c8dcb67a645da3c36355bb97c8c1490198a82e18
refresh_log:
  - from_sha: 3f1a6cb9f78ed401aea530aefa8634a91a27559e
    to_sha: 3f1a6cb9f78ed401aea530aefa8634a91a27559e
    at: "2026-07-26T23:33:22Z"
    summary: "No code changes. petr-muller approved (2026-06-30T12:01:25Z) and the PR merged (2026-06-30T12:34:11Z) by kubernetes-prow bot; no activity beyond the gate decision already recorded."
  - from_sha: c8dcb67a645da3c36355bb97c8c1490198a82e18
    to_sha: c8dcb67a645da3c36355bb97c8c1490198a82e18
    at: "2026-06-16T16:37:41Z"
    summary: "No code changes; k8s-triage-robot applied lifecycle/stale on 2026-06-09"
  - from_sha: c8dcb67a645da3c36355bb97c8c1490198a82e18
    to_sha: c8dcb67a645da3c36355bb97c8c1490198a82e18
    at: "2026-06-16T16:40:31Z"
    summary: "No code changes; petr-muller removed lifecycle/stale and asked author about addressing review"
  - from_sha: c8dcb67a645da3c36355bb97c8c1490198a82e18
    to_sha: 3f1a6cb9f78ed401aea530aefa8634a91a27559e
    at: "2026-06-25T12:52:31Z"
    summary: "PR rebased on main; second commit adds ExtraRefs support and addresses review comments"
---

# PR #549 — Allow filtering jobs by owner, repo and org

**Status: MERGED** (2026-06-30T12:34:11Z, by `app/kubernetes-prow`)

## Gate

**Decision: MERGE**

The two blocking-level findings from the original review (ExtraRefs handling, test style) have been addressed in `3f1a6cb9f`. The remaining open items (doc comment already added, ownerMatch closure still has shadow, no repo-only test case, stray blank line, no `t.Run` subtests) are nits that don't gate merge. No backward-incompatible API, config, or behavioral changes; the feature is purely additive and opt-in via query parameters. No merge risk to existing deployments.

**Findings disposition:**

- **ExtraRefs handling** (`REVIEW.md` converging, `@ivankatliarchuk`) — **addressed**. New `refsMatch` helper at `main.go:816-822` falls back to `ExtraRefs` when `Refs` is nil. Test fixtures include `extrarefs-org-match` and `extrarefs-different-org` jobs with dedicated test cases for org-only, repo-only, and org+repo ExtraRefs matching.
- **Test style: `t.Run`/`t.Fatalf`** (`REVIEW.md` converging, `@ivankatliarchuk`) — **not addressed**. Tests still use a bare `for` loop without `t.Run` subtests, and `t.Errorf` for setup errors instead of `t.Fatalf`. Does not gate merge — it's a style/robustness issue, not a correctness bug.
- **`resp.Body.Close()` in loop** (`@ivankatliarchuk`) — **borderline**. Author replaced `defer resp.Body.Close()` with an inline close-then-check pattern (`main_test.go:629-631`), but calls `io.ReadAll(resp.Body)` *after* closing the body. This reads from an already-closed body — effectively a no-op read returning empty bytes. The test still passes because `httptest.ResponseRecorder` buffers the full response, so `resp.Body` (a `bytes.Buffer`) remains readable after `Close()`. Functionally harmless in test code but logically wrong. Not a merge gate.
- **Doc comment** (`REVIEW.md` converging, `@ivankatliarchuk`) — **addressed**. Comment block added at `main.go:785-791`.
- **`ownerMatch` closure shadow** (`REVIEW.md` low) — **not addressed**. Still takes `owner` as a parameter shadowing the outer variable. Nit, does not gate.
- **Repo-only test case** (`REVIEW.md` low) — **addressed**. Test case "repo filter should match jobs with ExtraRefs" filters by `Repo: "testinfra"` alone.
- **Stray blank line** (`REVIEW.md` nit) — **not addressed**. Still present at `main_test.go:~620`. Nit.
- **`sets.New[string]`** (`@ivankatliarchuk`) — **addressed**. Explicit type parameter added.

**Merge risk (Area 2):** None. The change is purely additive — new optional query parameters on an existing internal HTTP endpoint. No exported Go API changes, no config/flag changes, no CRD/schema changes, no behavioral change when no filter parameters are supplied. Existing Prow deployments are unaffected on upgrade.

**Gating list:** Empty — nothing blocks merge. All blocking/should-fix items are addressed. Remaining items are nits.

---

**Author:** @rikatz | **+231 / -25** | **size/L** | **Files: 2**

**What it does:** Adds server-side filtering to the `/prowjobs.js` Deck endpoint via `?org=`, `?repo=`, and `?owner=` query parameters. Merges the filtering and omit logic into a single loop over jobs, building a `finalJobs` slice with only matching entries.

**Motivation:** Large Prow instances return huge JSON payloads. Server-side filtering reduces response size when callers only care about a specific org, repo, or PR author.

**Verdict: Approve with suggestions**

**Since previous review (2026-04-12):**
- 2026-06-09: `k8s-triage-robot` applied `lifecycle/stale` — 90 days of inactivity
- 2026-06-16: `petr-muller` commented `/remove-lifecycle stale` and asked @rikatz to address the review
- 2026-06-16: `rikatz` responded, committed to addressing comments by EOW
- 2026-06-18: `rikatz` replied to inline review comments (closure simplification, docs, test style, ExtraRefs approach), pushed second commit `3f1a6cb9f` ("add extrarefs to filter, address comments"), confirmed all findings addressed
- Code: `c8dcb67a` → `3f1a6cb9f` (PR rebased on main + ExtraRefs fix; PR-specific delta is in `cmd/deck/main.go` and `cmd/deck/main_test.go`)

---

## Reviewer Perspectives

### Code Quality — APPROVE

- No critical issues — filtering logic is correct with consistent nil-safety checks on `Spec.Refs`
- Clean single-loop merge of filter + omit logic avoids a second pass
- `ownerMatch` closure shadows outer `owner` variable — could simplify signature
- Test uses `t.Errorf` where existing tests use `t.Fatalf` for setup errors
- `defer resp.Body.Close()` in loop accumulates until function exit
- Missing test case for repo-only filter

### Maintainability — COMMENT (Burden: LOW)

- `ExtraRefs` gap is the main concern — periodic jobs silently excluded when filtering by org/repo
- Codebase precedent exists for `ExtraRefs` fallback in `cmd/deck/rerun.go:171-172`
- `ownerMatch` inline per-request is untestable in isolation
- New query parameters are undocumented
- Change is cleanly revertible, table-driven tests are idiomatic

### Deployment Risk — LOW RISK

- Purely additive — no config, CLI, RBAC, or schema changes
- Without filters, behavior is identical to before
- No action required from operators; drop-in upgrade
- Performance neutral (same linear scan as existing omit logic)
- Comprehensive tests cover no-filter, single-filter, and combined-filter scenarios

---

## Converging Concerns

Issues independently flagged by 2+ reviewers:

1. ~~**`ExtraRefs` not considered during filtering**~~ — **resolved in `3f1a6cb9f`**; verify implementation matches `rerun.go:171-172` pattern.

2. **Tests should use `t.Run` subtests and `t.Fatalf`** — fixes `defer` scoping, improves failure isolation, matches existing `TestHandleProwJobs` pattern. *(Flagged by: Code Quality, Maintainability)*

3. **New API parameters undocumented** — `org`, `repo`, `owner` query params have no comments or docs. *(Flagged by: Maintainability, Deployment Risk)*

---

## Detailed Findings

### [2/3 converging] Tests should use `t.Run` subtests and `t.Fatalf` — `main_test.go:559-610`

### [2/3 converging] Tests should use `t.Run` subtests and `t.Fatalf` — `main_test.go:559-610`

The test iterates over cases without `t.Run(tc.Name, ...)`. This causes three issues:

- `defer resp.Body.Close()` accumulates — all response bodies stay open until function exit (resource leak)
- After `json.Unmarshal` fails, `t.Errorf` continues execution against a zero-value struct
- Inconsistent with the existing `TestHandleProwJobs` which uses `t.Run`

Wrapping in `t.Run` fixes all three. Use `t.Fatalf` for setup errors (request creation, unmarshal).

**Suggested comment:**
> Could you wrap each test case in `t.Run(tc.Name, func(t *testing.T) { ... })`? This matches the pattern in `TestHandleProwJobs` above and fixes the `defer` scoping issue. Also, setup errors (request creation, unmarshal) should use `t.Fatalf` since continuing past them produces misleading results.

---

### [2/3 converging] New API parameters undocumented — `main.go:790-792`

The `org`, `repo`, and `owner` query parameters are undocumented. Adding a brief comment block on the handler listing supported parameters would help future maintainers and API consumers.

**Suggested comment:**
> nit: Consider adding a brief comment documenting the supported query parameters for this endpoint (org, repo, owner, omit) so future maintainers and API consumers can discover them.

---

### [low] `ownerMatch` closure: simplify or extract — `main.go:794-804`

The `ownerMatch` closure takes `owner` as a parameter even though it's only ever called with the outer variable. Two options:

- **Simplify:** remove the `owner` parameter and capture it from the closure scope
- **Extract:** move to a package-level function (avoids per-request allocation, enables isolated testing)

**Suggested comment:**
> nit: `ownerMatch` doesn't close over anything from the handler scope (the `owner` parameter shadows the outer variable). Consider either simplifying the signature to `func(job prowapi.ProwJob) bool` (capturing `owner` from the closure) or extracting it as a package-level function.

---

### [low] Missing test case for repo-only filter — `main_test.go:529-552`

The `repo` filter is only tested in combination with `org` and `owner`. A test case with just `?repo=pizzacontroller` (no org, no owner) would verify the cross-org matching behavior and improve coverage of the independent filter paths.

**Suggested comment:**
> Could you add a test case for filtering by `repo` alone? It would match across all orgs, which is worth verifying is intentional and tested.

---

### [nit] Stray blank line after `for` statement — `main_test.go:~562`

Remove the blank line immediately after `for _, tc := range testCases {`.

---

## Maintainer Recommendation

LGTM — approving this. The filtering logic is correct, the change is fully backward compatible, and test coverage is solid. Two suggestions worth considering: (1) use `t.Run` subtests and `t.Fatalf` for setup errors in the new test to match existing test style, and (2) consider a follow-up to handle `ExtraRefs` so periodic jobs are not silently excluded when filtering by org/repo. Neither is blocking.

---

## Resolved Findings

### [resolved] `ExtraRefs` not considered in org/repo/owner filtering — `main.go:808-814`

Addressed in commit `3f1a6cb9f`. Author confirmed: "filtering will also use extraRefs in case refs is null (for periodic jobs)". Implementation not re-read in this refresh — verify approach matches the `rerun.go:171-172` pattern before approving.

---

## Review Checklist

- [x] **[converging]** `ExtraRefs` handling — addressed in `3f1a6cb9f`; verify implementation
- [ ] **[converging]** Tests refactored to use `t.Run` subtests with `t.Fatalf` for setup errors
- [ ] **[converging]** Document new `org`, `repo`, `owner` query parameters
- [ ] `ownerMatch` simplified or extracted to package level
- [ ] Test case added for repo-only filter
- [ ] Stray blank line removed
- [ ] CI passes after changes
