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

## Followups

### Give test-created secret reloaders a shutdown path

- category: tech-debt
- where: `pkg/config/secret/reloader.go:55`, `pkg/config/secret/agent_test.go:260`
- necessity: could — prevents test-created reload goroutines from polling removed temporary files and producing later log noise.
- Mode: deliver
- why: PR #861 explicitly deferred the reload loop's process-lifetime behavior; agent-managed reloaders remain process-lived, while the test directly creates loaders backed by temporary directories.
- handoff prompt:

```text
In kubernetes-sigs/prow, following merged PR #861, "pkg/config/secret: fix TestAddWithParser flake" (merge commit 8591ce5f9cb0999d4918409c5b5139c0bd6e9bbb), add an internal shutdown path for test-created parsingSecretReloader instances in pkg/config/secret/reloader.go and use it from testAddWithParser in pkg/config/secret/agent_test.go.

Replace the un-stoppable time.Tick loop with a stoppable ticker and provide an idempotent way for the directly constructed test loader to stop its reload goroutine. Register cleanup after loader.start succeeds, so the goroutine exits before the test's temporary directory is removed. Make shutdown safe with the existing reload callback and loop.

Acceptance criteria:
- TestAddWithParser stops its directly created loader during cleanup, and that loader no longer polls its deleted temporary file or emits later stat errors.
- The existing reload and publication assertions still pass under the package's normal and race-enabled tests.
- Agent-managed Add/AddWithParser registrations retain their current process-lifetime reload behavior; no public shutdown API is added.

Scope guard: do not change secret reload intervals, the lastModTime initialization behavior, parser semantics, or unrelated agent lifecycle behavior.
```
