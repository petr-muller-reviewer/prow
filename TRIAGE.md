---
issue: kubernetes-sigs/prow#913
title: "Expose a public API for Tide status and blocking checks per PR"
state: open
labels:
main_sha: 88f56c0e89c868ec4e6bf0305fe23c7efa984ae7
triaged_at: 2026-08-31T17:41:26Z
verdict: accepted
---

## Findings

### [cause] No per-PR structured status API exists
- detail: Tide exposes pool-shaped JSON (`/tide.js`) and a free-text GitHub commit-status description, but nothing answers "for org/repo#N, what's Tide's verdict and structured blocking reason(s)" in one call.
- evidence: `cmd/deck/main.go:553-554` (route registration), `pkg/tide/status.go:277` (`expectedStatus`, free-text only).

### [related-code] Existing pool JSON endpoints
- where: `cmd/deck/main.go:1334-1362` (`handleTidePools`), `cmd/deck/main.go:1365-1382` (`handleTideHistory`)
- excerpt: |
    mux.Handle("/tide.js", gziphandler.GzipHandler(handleTidePools(cfg, ta, logrus.WithField("handler", "/tide.js"))))
    mux.Handle("/tide-history.js", gziphandler.GzipHandler(handleTideHistory(ta, logrus.WithField("handler", "/tide-history.js"))))
- relevance: Dumps all pools (`tide.PoolForDeck`: SuccessPRs/PendingPRs/MissingPRs/BatchPending/Action/Blockers) across every org/repo/branch Tide manages. A client must fetch the whole payload and scan for one PR; no reason field is included.

### [related-code] Source of pool data
- where: `pkg/tide/tide.go:372` (`Controller.ServeHTTP`), `pkg/tide/tide.go:625` (`syncController.ServeHTTP`)
- relevance: Backing data source for `/tide.js`; a per-PR endpoint would look up the same in-memory `pools` rather than inventing new state.

### [related-code] Only existing per-PR "reason" text
- where: `pkg/tide/status.go:277-395` (`expectedStatus`)
- excerpt: |
    // statusNotInPool is a format string used when a PR is not in a tide pool.
    statusNotInPool = "Not mergeable.%s"
    ...
    return github.StatusError, fmt.Sprintf(statusNotInPool, fmt.Sprintf(" Merging is blocked by issue%s %s.", s, strings.Join(numbers, ", "))), nil
- relevance: This is the text posted as the GitHub `tide` commit-status description — the only place a per-PR "why" currently exists. It's unstructured, unversioned prose, not a stable API (this is exactly what the author calls error-prone to parse).

## Checked

- Confirmed `/tide.js` and `/tide-history.js` already exist as public, undocumented JSON endpoints — ruled out "no API exists at all"; gap is specifically per-PR + structured reason.
- Confirmed the GitHub `tide` status context description (`pkg/tide/status.go`) is the only existing per-PR reason text, and it is free text.
- Casual scan for duplicate open issues by title/body found none; not exhaustively searched against closed issues.

## Next steps

- Ask author to confirm whether `/tide.js`/`/tide-history.js` were already considered and found insufficient (sharpens issue scope).
- Apply labels: `kind/feature`, `area/tide`, `area/deck`.
- If a maintainer wants to pursue this, next concrete step is a design comment proposing the response schema/reason taxonomy before any PR is opened.

## Open questions

- Should the new API report live/in-memory state only (matches `/tide.js` staleness) or trigger a fresh evaluation for the requested PR?
- What response shape/taxonomy for "blocking reasons" is desired — free enumeration vs. fixed reason codes? This drives most of the implementation effort.
- Should this be REST-ish (`/tide/status?org=&repo=&pr=`) or follow the existing `*.js` JSON-dump naming convention?
- Any concern about making Tide's per-PR status easily scriptable/pollable from a load/abuse perspective, even though it's cheap in-memory data?
