---
pr: kubernetes-sigs/prow#803
title: "append the release note if there is no release note block"
head_sha: 94c6c56b65a3e5e0d9db39c77c2e8d6baebfb52a
base: main
reviewed_at: 2026-09-27T15:38:26Z
verdict: approve
gate:
  decision: hold
  gated_at: 2026-09-27T15:38:36Z
  gated_head_sha: 94c6c56b65a3e5e0d9db39c77c2e8d6baebfb52a
  reviewed_head_sha: 94c6c56b65a3e5e0d9db39c77c2e8d6baebfb52a
refresh_log:
  - from_sha: 94c6c56b65a3e5e0d9db39c77c2e8d6baebfb52a
    to_sha: 94c6c56b65a3e5e0d9db39c77c2e8d6baebfb52a
    summary: no code changes; author posted an assign comment (@petr-muller, @cblecker), no new reviews or inline comments.
  - from_sha: 94c6c56b65a3e5e0d9db39c77c2e8d6baebfb52a
    to_sha: 94c6c56b65a3e5e0d9db39c77c2e8d6baebfb52a
    summary: no code changes; @cblecker left two inline review comments (COMMENTED review) — a new correctness finding on the pre-existing splice path hardcoding `\r\n`, and a test-coverage suggestion overlapping the existing narrow-coverage finding.
---

# Review

## Gate

**Decision: hold.** The PR head is unchanged since the saved review, and the
append path still adds an extra blank line to bodies with mixed line endings.
The new CRLF branch also has no test. Resolve the separator behavior and add
coverage for CRLF bodies, or explicitly accept those two points before merge.

Gating items:

- `REVIEW.md`, `pkg/plugins/releasenote/releasenote.go:511-523`: the append
  path detects CRLF anywhere in the body but checks only CRLF suffixes. A
  body with CRLF earlier and LF at the end gets an extra blank line and
  mixed line endings. **Not addressed; hold for a fix or explicit acceptance.**
- `REVIEW.md` and @cblecker, `pkg/plugins/releasenote/releasenote_test.go:767-800`:
  both new tests use LF-only bodies; the CRLF append branch remains untested.
  **Not addressed; add a CRLF case or explicitly accept the gap.**
- @cblecker, `pkg/plugins/releasenote/releasenote.go:528`: the existing-block
  splice still inserts CRLF into LF bodies. The code is unchanged, and this
  behavior predates the PR. **Not addressed; confirm whether to handle it in
  this PR or defer it.**

Merge risk: no exported API, configuration, schema, or deployment changes.
The functional change is limited to an org member invoking
`/release-note-edit` on a PR body with no release-note block: the command now
appends a block instead of returning an error. Existing deployments need no
migration. The command also rejects comments containing multiple recognized
release-note blocks; confirm that stricter input rule is intended.

## Verdict

Approve with non-blocking findings. The change is confined to the release
note command handler and its tests. The latest review confirmed that the
append path can add an extra blank line when a PR body mixes line endings;
the earlier findings remain recorded below.

## What this PR does

- `/release-note-edit` appends a release-note block when the PR body has none.
- The append path attempts to preserve the body's line endings and leave a
  blank line before the new block.
- The command now checks that its comment contains exactly one release-note
  block, rather than checking the PR body for one.
- Two tests cover appending to a non-empty body and to an empty body.

Since previous review:

- No code changes; the reviewed PR head remains `94c6c56b6`.
- A fresh read-only review corroborated the mixed line-ending finding in the
  append path. The earlier inline comments and findings remain below.

## Findings

### [should-fix] mixed line-ending handling in the append path is unsound
- where: `pkg/plugins/releasenote/releasenote.go:511-524`
- concern: `lineEnding` is inferred from whether `\r\n` appears *anywhere* in
  `ic.Issue.Body`, but the suffix checks (`HasSuffix(body, lineEnding+lineEnding)`,
  `HasSuffix(body, lineEnding)`) assume that global guess matches the body's
  *trailing* line ending. A body with inconsistent line endings (e.g. `\r\n`
  in the middle, bare `\n` at the end, plausible after manual GitHub edits)
  falls through all suffix branches to `separator = lineEnding+lineEnding`,
  producing a mixed-style seam right before the new block (e.g.
  `...text\n\r\n\r\n\`\`\`release-note`). Flagged independently by the code
  quality, maintainability, and deployment-risk reviewers. Not a functional
  break (GitHub renders either way), but a real correctness gap in the new
  logic.
- excerpt: |
    lineEnding := "\n"
    if strings.Contains(ic.Issue.Body, "\r\n") {
        lineEnding = "\r\n"
    }
    separator := lineEnding + lineEnding
    if ic.Issue.Body == "" {
        separator = ""
    } else if strings.HasSuffix(ic.Issue.Body, lineEnding+lineEnding) {
        separator = ""
    } else if strings.HasSuffix(ic.Issue.Body, lineEnding) {
        separator = lineEnding
    }

### [should-fix] splice path (existing single block) still hardcodes `\r\n`, inconsistent with the new adaptive append path
- where: `pkg/plugins/releasenote/releasenote.go:528`
- concern: raised by @cblecker on 2026-08-07. The new append branch (511-523,
  above) infers `lineEnding` from the body and uses it consistently; the
  pre-existing `else` splice branch (for editing an already-present single
  block) unconditionally injects `\r\n` around the replacement note
  regardless of the body's actual line-ending style. For a body using LF
  line endings (e.g. a PR body created via the API rather than GitHub's web
  UI), editing an existing release-note block now produces mixed CRLF/LF
  line endings around the note content. This behavior predates this PR (see
  Checked — the splice branch is "functionally unchanged, only re-indented"),
  but the PR's own new logic right above it highlights the inconsistency and
  would be the natural place to fix both paths together by extracting a
  shared `lineEnding` variable before the `if`/`else`.
- excerpt: |
    replaced := append(b[:i[2]], append([]byte("\r\n"+strings.TrimSpace(newNote)+"\r\n"), b[i[3]:]...)...)

### [should-fix] append-branch test coverage is narrow
- where: `pkg/plugins/releasenote/releasenote_test.go:767-800`
- concern: only "body without a block, no trailing newline" and "empty body"
  are tested. Untested: body already ending in a blank line (separator ""
  via the non-empty branch), body ending in exactly one line ending
  (separator = lineEnding), and a body using `\r\n` line endings (the whole
  CRLF-detection branch is never exercised). A future refactor of this logic
  has no test to catch a regression in those branches. @cblecker made the
  same observation on 2026-08-07, specifically suggesting a CRLF-body test
  case (e.g. `Body: "Top\r\nBelow"`) to exercise the
  `strings.Contains(ic.Issue.Body, "\r\n")` branch.
- excerpt: |
    expectedNote: "Top\nBelow\n\n```release-note\nThe new note\n```\n",
    ...
    expectedNote: "```release-note\nThe new note\n```\n",

### [nit] pluginHelp description is stale
- where: `pkg/plugins/releasenote/releasenote.go:92-97`
- concern: `Description: "Replaces the release note block in the top level comment with the provided one."` no longer fully describes behavior — it now also appends a block when none exists.
- excerpt: |
    pluginHelp.AddCommand(pluginhelp.Command{
        Usage:       "/release-note-edit",
        Description: "Replaces the release note block in the top level comment with the provided one.",
        WhoCanUse:   "Org Members.",
        Examples:    []string{"/release-note-edit\r\n```release-note\r\nThe new release note\r\n```"},
    })

### [nit] append/separator logic should be extracted
- where: `pkg/plugins/releasenote/releasenote.go:508-524`
- concern: the four-branch separator computation is inlined in `editReleaseNote`. Extracting it into a small pure helper (e.g. `appendReleaseNoteBlock(body, note string) string`) would make it directly unit-testable and easier to verify by inspection. Raised by the maintainability reviewer.

### [nit] no comment explaining the semantic shift from "strict edit" to "edit-or-create"
- where: `pkg/plugins/releasenote/releasenote.go:507-524`
- concern: `/release-note-edit` previously only edited an existing block and errored otherwise; it now silently creates one. That rationale currently lives only in the PR description (linking to an external kubernetes/kubernetes PR comment), not in the code.

### [question] does the new comment-multiplicity check narrow previously accepted inputs?
- where: `pkg/plugins/releasenote/releasenote.go:496-501`
- concern: the new `FindAllStringSubmatchIndex(ic.Comment.Body, -1) != 1` check is stricter than the prior implicit behavior of `getReleaseNote` (which just took the first regex match via `FindStringSubmatch`). If any existing user relied on `/release-note-edit` comments containing more than one code block (only the first being treated as the release note), those would now be rejected. Worth confirming this is intentional and calling it out as a minor behavior tightening if so. Raised by the deployment-risk reviewer.

## Checked
- `go test ./pkg/plugins/releasenote -run '^Test_editReleaseNote$' -count=1`
  passed on 2026-09-27. The new tests do not cover mixed line endings.
- Confirmed the old "single release note block" check (`len(i) != 4` on `FindStringSubmatchIndex(ic.Issue.Body)`) could never actually detect multiple blocks — `FindStringSubmatchIndex` returns a fixed-length slice or nil, so the check was equivalent to "no match in issue body." The pre-existing "multiple release note blocks" test only passed incidentally because its fixture left `Issue.Body` empty. The new `FindAllStringSubmatchIndex(ic.Comment.Body, -1) != 1` check is a genuine correctness fix, independently confirmed by all three specialist reviewers.
- Hand-traced both new test cases ("append to body without release note block", "append to empty body") against the separator logic — both produce the expected output.
- `newNote` is already validated non-empty before the append branch runs (early return above for an empty note).
- The appended block (fenced with `` ```release-note `` and a trailing line ending) matches `noteMatcherRE`, so subsequent plugin runs/label determination pick it up correctly.
- No config structs, YAML/JSON tags, CLI flags, or plugin registration are touched — change is isolated to one command handler. Rollback is a plain binary revert with no persisted-state migration.
- The pre-existing "one release-note block" splice path (`else` branch) is functionally unchanged from before this PR, only re-indented.

## Open questions
- Should the CRLF-vs-LF detection and blank-line-suffix branches get explicit test coverage before merge, given they're the actual new logic being introduced?
- Is the `pluginHelp` description meant to be updated in a follow-up, or is it intentionally left generic?
- Does the new "exactly one block in the comment" check reject any previously-accepted `/release-note-edit` comment shape? If so, worth a one-line callout in the PR description/release notes.
