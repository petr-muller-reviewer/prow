---
pr: kubernetes-sigs/prow#861
title: "pkg/config/secret: fix TestAddWithParser flake"
head_sha: b5a4bf7d2fe6a8c21cc565fbb0d9ca9a7bcda8c8
base: main
reviewed_at: 2026-09-04T17:25:51Z
verdict: approve
refresh_log:
  - old_sha: b5a4bf7d2fe6a8c21cc565fbb0d9ca9a7bcda8c8
    new_sha: b5a4bf7d2fe6a8c21cc565fbb0d9ca9a7bcda8c8
    summary: "No code changes; recorded approval activity and the PR merge."
---

## Summary

Test-only change to `pkg/config/secret/agent_test.go`. Fixes a genuine
race in `TestAddWithParser`: the old test synchronized on the parser
callback (which runs before `reloader.go:80` stores the parsed value),
giving no happens-before edge to the getter it then asserted on. New
test synchronizes on `reloadCensor`, which is invoked only after the
value is stored under lock (`reloader.go:44`, `:82`), so a channel send
from inside it does establish a happens-before edge per the Go memory
model. Bad-parse case is now asserted via publish-sequence ordering
instead of point-in-time sampling of the getter, eliminating the
observation-window/sleep pattern entirely. No production code changed.

Since previous review:

- No code changes (`b5a4bf7d2fe6a8c21cc565fbb0d9ca9a7bcda8c8` unchanged).
- `petr-muller` approved the PR at `2026-08-18T20:12:14Z`; the approval bot recorded it at `20:12:21Z`.
- The PR merged into `main` at `2026-08-18T20:33:19Z`.

## Findings

(none)

## Checked

- `reloader.go:44` and `:82` are indeed the only two call sites of
  `reloadCensor`, both after `p.lock.Unlock()` — the synchronization
  claim in the PR description holds.
- `writeSecret` uses `time.Now().Add(generation*time.Second)` for
  `os.Chtimes`, guaranteeing strictly increasing mtimes across
  generations regardless of filesystem timestamp granularity.
- `awaitPublish`'s repeat-skipping (`got == last && got != want`)
  correctly absorbs a poll landing between `os.WriteFile` and
  `os.Chtimes`, the one residual race.
- No exported API changed; no other test or production file references
  the removed synchronization pattern.
- Verified locally that the new `TestAddWithParserRegistersSecret` and
  the rewritten `TestAddWithParser` compile and pass; reasoning in the
  PR description about a synthetic regression (removing the `continue`
  at `reloader.go:75`) is consistent with the code as written.

## Open questions

(none — PR description already flags the deliberately-deferred `time.Tick`
leak and `lastModTime` zero-value warts as known, out-of-scope issues)
