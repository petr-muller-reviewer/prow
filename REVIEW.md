---
pr: kubernetes-sigs/prow#1000
title: "github: skip backoff after the final REST API attempt"
head_sha: 8f4ec6a50c593d5bba8b63f991539624baeae52a
base: main
reviewed_at: 2026-10-08T12:57:04Z
verdict: approve
---

# Review

## Verdict

Approve. The change checks whether another retry-loop iteration remains before applying the 5xx backoff. It preserves backoff between retryable 5xx responses and returns the final response without sleeping after the last attempt. The added cases cover a single attempt, exhausted retries, and success after retries; no findings were identified.

## What this PR does

- Skips exponential backoff after the final configured REST API attempt receives a 5xx response.
- Keeps the existing backoff between 5xx responses when another attempt remains.
- Records sleep durations in the test clock so tests can assert the retry timing.
- Covers single-attempt failure, exhausted retries, and success on a later attempt.

## Findings

None.

## Checked

- `pkg/github/client.go:1245-1249`: the guard permits sleeping only when `retries + 1` is less than `maxRetries`.
- The final 5xx response remains available to the caller after retries are exhausted.
- The test cases assert attempt counts, final response status and body, and sleep durations.
- `CONTRIBUTING.md` contains no local code-specific rule relevant to this patch.
- `git diff --check upstream/main...HEAD` passed.

## Open questions

None.
