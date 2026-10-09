---
pr: kubernetes-sigs/prow#776
title: "Fix double linking bug when key is a substring of another link"
head_sha: b566b97b9568557e32ab5c0eaea74423d5528981
base: main
reviewed_at: 2026-07-01T11:57:54Z
verdict: approve
---

## Findings

### [nit] suffix boundary case untested (converging: 2 reviewers)
- where: `pkg/plugins/jira/jira_test.go:472`
- concern: The updated code comment explicitly names two cases — prefix boundary (BC-1 inside ABC-123) and suffix boundary (AC-1 inside AC-1234) — but the new test only exercises the prefix. Every skipped replacement in the test is decided by `prefixOk=false` (preceding 'A' short-circuits `&&`); `suffixOk` is never the decisive gate. A bug in the suffix path would leave all tests green.
- excerpt: |
    issueNames: []string{"ABC-123", "BC-1", "BC-2", "BC-3"},
    // all blocked matches: preceding char 'A' -> prefixOk=false -> short-circuit

### [nit] prefix condition mixes two idioms
- where: `pkg/plugins/jira/jira.go:264`
- concern: The combined prefix check uses `strings.Contains("[/\`-", string(text[idx-1]))` (pre-existing byte-to-string set membership) alongside `isAlphanumeric(text[idx-1])` (clean predicate). A future `isSafePrefixChar(c byte) bool` helper would unify them and make the logic easier to extend.
- excerpt: |
    prefixOk := idx == 0 || (!strings.Contains("[/`-", string(text[idx-1])) && !isAlphanumeric(text[idx-1]))

## Checked
- `startingIdx = endIdx` is semantically equivalent to the removed `startingIdx = idx + len(old)` — no regression.
- `isAlphanumeric` operates on bytes; UTF-8 continuation bytes (0x80–0xBF) are outside ASCII alphanumeric range, so adjacent multibyte chars don't interfere.
- `strings.Contains("[/\`-", ...)` is NOT made redundant by `!isAlphanumeric` — those chars are non-alphanumeric, so without the explicit check they would pass `!isAlphanumeric` and allow replacement. Both checks are necessary.
- Both issue-ordering paths work: when BC-1 is processed before ABC-123, the preceding 'A' in the raw text still triggers `prefixOk=false`. Non-determinism from `UnsortedList()` is safe.
- `suffixOk` correctly guards end-of-string: `endIdx >= len(text)` short-circuits before byte read.
- Test helper refactor (adding optional `issueNames []string`, nil-fallback to `issueName` constant) does not change semantics of any existing test case.
- No config schema, API surface, or external dependency changes — deployment risk is LOW, zero operator action needed on upgrade.
- Rollback is safe: reverting restores the original bug, introduces no new failure mode.

## Open questions
- Would you add a follow-up test for the suffix boundary? `issueNames: []string{"AC-1", "AC-1234"}`, body `"AC-1234 references AC-1"`, expected `"AC-1234 references [AC-1](url)"` — closes the gap and makes the intent self-documenting.
- Consider extracting `isSafePrefixChar(c byte) bool` to wrap both `strings.Contains` and `isAlphanumeric`? Non-blocking, but would make line 264 a one-liner and give the whole concept a name.
