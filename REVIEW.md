---
pr: kubernetes-sigs/prow#715
title: "buildlog: fail gracefully on large build logs"
head_sha: a67ffd4f8766b65bf045f825a04d8351efc42463
base: main
reviewed_at: 2026-05-14T11:02:10Z
verdict: approve
refresh_log:
  - old_sha: dccb2344479f601c873df34072271f538d24ff90
    new_sha: a67ffd4f8766b65bf045f825a04d8351efc42463
    summary: "Author addressed Size() error handling, added CSS for .log-error and .log-warning, added UI warning banner, added TestBodySizeErrorDisablesHighlight"
gate:
  decision: hold
  gated_at: 2026-06-01T16:36:31Z
  gated_head_sha: a67ffd4f8766b65bf045f825a04d8351efc42463
  reviewed_head_sha: a67ffd4f8766b65bf045f825a04d8351efc42463
---

## Gate

**Verdict: hold**

The code is correct, low-risk, and all reviewer-blocking concerns from @elmiko are addressed (he gave `/lgtm`). Two items prevent merge: (1) the PR lacks `/approve` from a `pkg/spyglass` approver — the bot says NOT APPROVED and is waiting for smg247 or michelle192837; (2) one open [should-fix] finding (error message stutter) is worth a one-line fix before approval. The stutter is cosmetic and doesn't affect correctness, so it doesn't block a maintainer who disagrees; but it's an easy win.

**Gating list:**
- **Process**: Missing `/approve` from smg247 or michelle192837 (`pkg/spyglass/OWNERS`). `mergeStateStatus: UNSTABLE` — likely CI; check before approving.
- **[should-fix] Error message stutter** (`lens.go:230`, from `REVIEW.md`): `av.Error = fmt.Sprintf("Failed to read log: %v", err)` wraps `logLinesAll`'s `"failed to read log %q: %w"` — user sees "Failed to read log: failed to read log 'build-log.txt': ...". One-line fix: change outer message to `"Error: %v"`. Not blocking on its own but was flagged should-fix and unaddressed.

**Merge risk (Area 2): none notable.**
- `LogArtifactView.Warning` is a new struct field but the struct is internal template data, not serialized or exported. No API surface change.
- Size() error now disables highlighting (previously allowed it silently). Strictly safer: trades a potential hang for a visible warning. Only affects backends where `Size()` can error.
- ReadAll() failures now surface in UI. Previously silent. Additive.
- No flags, config schema, or CRD changes.

## Findings

### Resolved

#### ~~[should-fix] Size() error silently allows highlighting~~
- where: `pkg/spyglass/lenses/buildlog/lens.go:240-243`
- resolution: Size() error now logs a `logrus.Warn` and disables highlighting conservatively. New test `TestBodySizeErrorDisablesHighlight` covers this path.

#### ~~[should-fix] Missing CSS for .log-error~~
- where: `pkg/spyglass/lenses/buildlog/buildlog.css:139-147`
- resolution: Added `.log-error` (red) and `.log-warning` (yellow) CSS rules.

### Open

### [should-fix] Error message stutter
- where: `pkg/spyglass/lenses/buildlog/lens.go:230`
- concern: Outer `fmt.Sprintf("Failed to read log: %v", err)` wraps inner `fmt.Errorf("failed to read log %q: %w", ...)` from `logLinesAll` (line 493). User sees "failed to read log" twice.
- excerpt: |
    av.Error = fmt.Sprintf("Failed to read log: %v", err)

### [nit] Callback path does not apply size guard
- where: `pkg/spyglass/lenses/buildlog/lens.go:452,469`
- concern: `loadLines` always passes `conf.highlightRegex` directly. Expanding collapsed sections in a large log still triggers regex matching. Lower practical risk (chunks, not full log) but inconsistent with `Body()` behavior.
- excerpt: |
    logLines := highlightLines(skipLines, skipRequest.StartLine, &request.Artifact, conf.highlightRegex, conf.highlightLengthMax)

### [nit] Test does not directly verify highlighting is skipped
- where: `pkg/spyglass/lenses/buildlog/lens_test.go:1186`
- concern: Asserts `"log-error"` is absent and warning is present, but does not check `"match-highlighted"` is absent. Does not directly test the intended behavior.

### [nit] No log line when size threshold triggers
- where: `pkg/spyglass/lenses/buildlog/lens.go:244-246`
- concern: The Size() error path now logs (resolved), but the normal `sz > highlightSizeThreshold` path at line 244 still has no server-side log. A `logrus.Info` with artifact name and size would help operators.

## Checked
- XSS safety: html/template auto-escapes {{.Error}} and {{.Warning}}
- Zero-value backwards compat: Error and Warning default to "", template {{if .Error}}/{{if .Warning}} skip for successful artifacts
- No config/API/deployment impact: LogArtifactView is internal template struct
- Test coverage: all new paths tested, errArtifact helper extended with sizeErr field
- Constant placement and naming: highlightSizeThreshold is clear
- Existing tests pass

## Open questions
- Is the Callback path omission intentional for this PR scope?
- The warning message hardcodes "10 MiB" — would it be better to derive from the constant?

## Since previous review

- Author force-pushed addressing review feedback from @elmiko and this review.
- Size() error path now disables highlighting conservatively and logs a warning (resolves main [should-fix]).
- Added `.log-error` and `.log-warning` CSS rules in `buildlog.css` (resolves missing CSS finding).
- Added a `Warning` field to `LogArtifactView` and renders a yellow warning banner in the UI when highlighting is skipped, with two messages: "log exceeds 10 MiB" or "unable to determine log size".
- New test `TestBodySizeErrorDisablesHighlight` and extended assertions in `TestBodyLargeLogSkipsHighlight`.

## Followups

### cleanup: Error message stutter
- category: cleanup
- necessity: should
- where: `pkg/spyglass/lenses/buildlog/lens.go:230`

```
In kubernetes-sigs/prow, as a followup to PR #715 "buildlog: fail gracefully on large build logs" (head a67ffd4f8766b65bf045f825a04d8351efc42463), fix the doubled error phrase in the buildlog lens.

The problem: `lens.go` around line 230 has:
  av.Error = fmt.Sprintf("Failed to read log: %v", err)
where `err` comes from `logLinesAll`, which itself returns:
  fmt.Errorf("failed to read log %q: %w", artifact.JobPath(), err)
The user sees: "Failed to read log: failed to read log 'build-log.txt': ..."

Fix: change the outer format string so the phrase isn't doubled. For example:
  av.Error = fmt.Sprintf("Error reading log: %v", err)
or simply:
  av.Error = err.Error()

Work on the main branch (not the PR branch). Only touch lens.go. No new tests needed — the existing TestBodyReadAllError already covers this path; verify the rendered string no longer contains the doubled phrase. Run `go test ./pkg/spyglass/lenses/buildlog/...` to confirm.

Out of scope: changes to logLinesAll, changes to other error paths, CSS changes.
```

### tech-debt: Callback path doesn't apply size guard
- category: tech-debt
- necessity: could
- where: `pkg/spyglass/lenses/buildlog/lens.go:459,480`

```
In kubernetes-sigs/prow, as a followup to PR #715 "buildlog: fail gracefully on large build logs" (head a67ffd4f8766b65bf045f825a04d8351efc42463), extend the highlight size guard to the Callback (loadLines) path.

Context: PR #715 added a size check in Body() that disables syntax highlighting for artifacts larger than highlightSizeThreshold (10 MiB). However, loadLines() — the Callback handler for expanding collapsed line groups — still passes conf.highlightRegex directly to highlightLines() without any size check. A user viewing a large log will see no highlights on initial load (correct), but clicking to expand a hidden section will trigger regex matching again (inconsistent).

Task: In loadLines() in pkg/spyglass/lenses/buildlog/lens.go, apply the same size guard used in Body(): call a.Size() on request.Artifact, and if Size() errors or the size exceeds highlightSizeThreshold, pass nil as the highlightRegex argument to highlightLines() calls. Use the same error handling pattern as Body() (log a warning on Size() error, silently skip highlighting when threshold exceeded).

Add a test that calls loadLines() (or exercises it via the Callback path) with a large artifact and asserts that no "match-highlighted" spans appear in the output.

Work on the main branch. Run `go test ./pkg/spyglass/lenses/buildlog/...` to confirm. Out of scope: UI changes, changes to the warning banner, changes to Body().
```

### cleanup: Warning message hardcodes "10 MiB"
- category: cleanup
- necessity: could
- where: `pkg/spyglass/lenses/buildlog/lens.go:246`

```
In kubernetes-sigs/prow, as a followup to PR #715 "buildlog: fail gracefully on large build logs" (head a67ffd4f8766b65bf045f825a04d8351efc42463), derive the warning message's size display from the highlightSizeThreshold constant rather than hardcoding "10 MiB".

The problem: lens.go line 246 has:
  av.Warning = "Syntax highlighting disabled (log exceeds 10 MiB)"
The threshold is defined as:
  highlightSizeThreshold = 10 * 1024 * 1024 // 10 MiB
If highlightSizeThreshold is ever changed, the message won't update.

Fix: Replace the hardcoded string with a derived one, e.g.:
  av.Warning = fmt.Sprintf("Syntax highlighting disabled (log exceeds %d MiB)", highlightSizeThreshold/1024/1024)

Update the corresponding test assertion in TestBodyLargeLogSkipsHighlight to match (it currently checks strings.Contains(got, "Syntax highlighting disabled"), so no change needed there — but verify the test still passes).

Work on the main branch. Only touch lens.go and, if needed, lens_test.go. Run `go test ./pkg/spyglass/lenses/buildlog/...` to confirm.

Out of scope: changes to threshold value, CSS changes, changes to the Size() error path warning.
```

### tests: Large-log test doesn't verify highlighting is actually skipped
- category: tests
- necessity: could
- where: `pkg/spyglass/lenses/buildlog/lens_test.go:1193`

```
In kubernetes-sigs/prow, as a followup to PR #715 "buildlog: fail gracefully on large build logs" (head a67ffd4f8766b65bf045f825a04d8351efc42463), strengthen TestBodyLargeLogSkipsHighlight to directly assert that highlighting was suppressed.

The current test (lens_test.go, TestBodyLargeLogSkipsHighlight) checks that a warning banner is present ("log-warning", "Syntax highlighting disabled") but does not assert that highlighted spans are absent. If the size guard were accidentally removed, the warning check would fail but there's no assertion catching that "match-highlighted" spans appear.

Fix: Add the following assertion to TestBodyLargeLogSkipsHighlight, after the existing assertions:
  if strings.Contains(got, "match-highlighted") {
      t.Error("Body() should not render highlighted spans for large logs")
  }

Work on the main branch. Only touch lens_test.go. Run `go test ./pkg/spyglass/lenses/buildlog/...` to confirm.

Out of scope: changes to lens.go, changes to other tests.
```
