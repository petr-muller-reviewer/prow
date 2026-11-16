---
pr: kubernetes-sigs/prow#1028
title: "secretutil: censor the values inside JSON and INI secrets"
head_sha: 365526a1b1483fb958bbcb2e74d9541c084ca79c
base: main
reviewed_at: 2026-10-28T01:09:48Z
verdict: request-changes
gate:
  decision: do-not-merge
  gated_at: 2026-10-28T01:15:07Z
  gated_head_sha: 365526a1b1483fb958bbcb2e74d9541c084ca79c
  reviewed_head_sha: 365526a1b1483fb958bbcb2e74d9541c084ca79c
---

# Review

## Gate

Do not merge: both previously confirmed blocking defects remain unchanged.

PR #1028 is open at `365526a1b1483fb958bbcb2e74d9541c084ca79c`, the same head covered by the saved review; there are no intervening commits or code changes. Re-reading the current head confirms that extracted prefixes can suppress full-secret matches and opaque double-padded base64 credentials can register `=` for global censorship. Paginated GitHub review, inline-comment, and issue-comment reads found no substantive reviewer feedback or discussion changing those conclusions. Fix both blockers and add regression coverage before reconsidering merge; retain the original review verdict and treat the remaining suggestions as nonblocking.

### Findings disposition

- **Not addressed; blocks merge — REVIEW.md, preserve longer matches** (`pkg/secretutil/censor.go:145-149`, `177-179`): extracted values still precede subsequent original secrets, JSON map traversal still supplies arbitrary registration order, and the unchanged replacer uses ordered non-overlapping matches. The demonstrated `abcdefghi` to `XXXdefghi` credential leak remains possible.
- **Not addressed; blocks merge — REVIEW.md, padded-token detection** (`pkg/secretutil/censor.go:196`, `201-208`): any line containing `=` still passes INI detection. A token such as `YWJjZA==` still yields the extracted value `=`, which is registered under the unchanged zero minimum, altering unrelated log and artifact content.
- **Not addressed; nonblocking recommendation — REVIEW.md, complete JSON validation** (`pkg/secretutil/censor.go:162-165`): Decoder.More() remains the trailing-input check. Replace it with complete-document validation and cover extra closing delimiters; this does not independently drive the gate verdict.
- **Not addressed; nonblocking recommendation — REVIEW.md, successful JSON without strings** (`pkg/secretutil/censor.go:146-149`, `167`): a successful parse with no strings still returns nil and falls through to INI. Distinguish recognition from extraction results; this does not independently drive the gate verdict.
- **Not addressed; nonblocking recommendation — REVIEW.md, extraction documentation** (`pkg/secretutil/censor.go:94-104`): Refresh's public comment still only describes refreshing secrets, and the PR still changes only implementation and tests. Document automatic extraction, its relationship to IniFilenames, and the minimum-length tradeoff.

### Independent merge risk

- **API and configuration:** the full PR diff adds private helpers and changes Refresh behavior; no exported signature, schema, flag, environment variable, or default-value declaration changes. No coordinated rollout, permissions change, or migration requirement was found. No additional compatibility-specific repository skill applies; this pass examined the code directly and incorporates the prior deployment review.
- **Behavior and blast radius:** automatic extraction is unconditional for callers of ReloadingCensorer.Refresh/RefreshBytes, including the component secret agent, log formatter, and sidecar artifact censorship. Existing deployments using compatible structured secret content receive broader redaction without opting in; ordinary strings stored alongside credentials may be censored, and short patterns can alter serialized log metadata. The zero default minimum makes padded-token misclassification particularly disruptive. There is no new opt-in or operator-facing release note in the two-file PR diff.
- **Recovery:** reverting restores previous behavior but cannot reconstruct redacted uploaded artifacts and also restores the earlier gap in field-level protection. Raising minimum_secret_length trades reduced incidental masking for leaving short credentials unprotected, so it does not substitute for fixing the blockers.

### Gating list

1. Fix precedence so extracted prefixes cannot expose longer secrets — `pkg/secretutil/censor.go:145-149`; source: REVIEW.md blocking finding, confirmed by Code Quality and local probes.
2. Preserve opaque padded credentials instead of registering padding as INI values — `pkg/secretutil/censor.go:196`; source: REVIEW.md blocking finding, independently confirmed by Code Quality and Deployment Risk.


## Verdict

Request changes: prevent partial credential exposure and unintended redaction before merging.

The shared implementation is a sensible place to fix field-level censorship, but two reproduced defects require changes: overlapping extracted patterns can expose credential suffixes, and padded base64 tokens can cause unrelated output to be redacted. Code Quality and Deployment Risk independently identified the latter; Code Quality and Maintainability also agreed that JSON validation needs clarification. Maintenance burden and deployment risk are both medium: no configuration migration or new permissions are required, but broader extraction immediately affects component logs and uploaded artifacts, including ordinary configuration values stored alongside credentials.

## What this PR does

- Extracts string values recursively from JSON objects and arrays.
- Extracts values from payloads whose lines resemble INI syntax.
- Registers extracted values alongside complete secrets in the shared reloading censor, covering sidecar and in-process callers.
- Applies existing trimming, base64 encoding, minimum-length, and boolean handling to extracted values.
- Adds public-behavior tests for JSON and INI field censorship and whole-blob fallback.

## Findings

### [blocking] Preserve longer secret matches when adding extracted values
- where: `pkg/secretutil/censor.go:145-149`
- concern: CONFIRMED by code inspection and temporary probes: extracted values precede subsequent original secrets, and the replacer prioritizes registration order; Refresh with `{"password":"abc"}` followed by `abcdefghi` produces `XXXdefghi`, exposing a previously protected suffix (P1). Overlapping values within a single JSON object are also registered in map iteration order: `secret-extra` can become `XXXXXX-extra`. Ensure shorter patterns cannot suppress complete matches, including original, extracted, and encoded patterns, and add regression coverage; Code Quality identified this blocker.
- excerpt: |
    expanded = append(expanded, secret)
    if values := jsonStringValues(secret); values != nil {
        expanded = append(expanded, values...)
    } else {
        expanded = append(expanded, iniValues(secret)...)

### [blocking] Avoid interpreting base64 padding as an INI value
- where: `pkg/secretutil/censor.go:194-196`
- concern: CONFIRMED: Contains("=") classifies an opaque token such as `YWJjZA==` as INI, whose parsed value is `=`; the default minimum length of zero then makes unrelated `result=ok` become `resultXok` in component logs and sidecar artifacts (P2). Code Quality and Deployment Risk independently reproduced this regression. Tighten detection to preserve opaque padded credentials and add a regression test; raising minimum_secret_length is not a general fix because it also leaves genuinely short credentials unprotected.
- excerpt: |
    case line == "", strings.HasPrefix(line, "#"), strings.HasPrefix(line, ";"):
    case strings.HasPrefix(line, "[") && strings.HasSuffix(line, "]"):
    case strings.Contains(line, "="):

### [should-fix] Validate the complete JSON document
- where: `pkg/secretutil/censor.go:162-165`
- concern: Nonblocking suggestion shared by Code Quality and Maintainability: Decoder.More() does not establish end-of-input, so malformed `{"a":"secret"}}` can still register `secret`, contrary to the whole-blob fallback contract. Use json.Unmarshal or require a subsequent decode to return io.EOF, and cover trailing closing delimiters in the tests.
- excerpt: |
    decoder := json.NewDecoder(bytes.NewReader([]byte(trimmed)))
    var parsed interface{}
    if err := decoder.Decode(&parsed); err != nil || decoder.More() {
        return nil
    }

### [should-fix] Distinguish recognized JSON without strings from parsing failure
- where: `pkg/secretutil/censor.go:146-149`
- concern: Maintainability identified a parser-contract ambiguity: valid JSON without string values returns nil and falls through to INI parsing. Its temporary probe showed that `{"a=b":1}` registers `b":1}` as a secret even though the JSON document contains no string values. As a nonblocking improvement, return a separate recognition flag or a non-nil empty slice for successfully parsed documents, and add a public-behavior regression case.
- excerpt: |
    if values := jsonStringValues(secret); values != nil {
        expanded = append(expanded, values...)
    } else {
        expanded = append(expanded, iniValues(secret)...)

### [should-fix] Document automatic extraction and its operational effects
- where: `pkg/secretutil/censor.go:104-105`
- concern: Maintainability and Deployment Risk recommend documenting that every compatible secret is now expanded automatically, including ordinary configuration values and short strings, and explaining the relationship to sidecar's existing IniFilenames selection. Short extracted values can also alter serialized log metadata because censorship runs after formatting; this is an intentional broadening of extraction rather than another parsing regression. Clarify the behavior at Refresh and in operator-facing documentation, including the tradeoff when increasing minimum_secret_length.
- excerpt: |
    for _, secret := range expandStructuredSecrets(secrets) {
        toEncode := []string{secret}

## Checked

- Inspected the complete PR diff against merge base `6c4d88e3b73e434d7acc03395ec337fa2b3f1fad`, resolved using `upstream/main`.
- Confirmed the local tracked checkout matches PR head `365526a1b1483fb958bbcb2e74d9541c084ca79c`, excluding review artifacts.
- Read PR #1028 and issue #939 to establish the intended extraction and fallback behavior.
- Completed independent Code Quality, Maintainability, and Deployment Risk reviews, followed by a maintainer walkthrough and synthesis.
- Verified both required changes against the cited code, registration order, byte replacer's ordered non-overlapping matching contract, and sidecar's default minimum length of zero.
- Temporary public-API probes reproduced both required findings; reviewers additionally reproduced overlapping JSON values and the JSON parser-boundary cases. Probe files were removed afterward.
- Passed `go test ./pkg/secretutil ./pkg/config/secret ./pkg/logrusutil`; the Code Quality reviewer also passed `go test ./pkg/secretutil`.
- Checked sidecar, secret-agent, and log formatter call sites, including post-format censorship and existing filename-based INI extraction.
- Existing trimming, base64, minimum-length, boolean exclusion, length-preserving replacements, and locking remain in use; parsing runs outside the censorer lock.
- No configuration schema, API, CLI, RBAC, new service dependency, migration, or deployment ordering change was identified. The INI library was already a project dependency.
- Generic extraction is intentionally broad; masking nonsecret configuration fields is not a separate blocker.
- Rollback restores prior extraction behavior but cannot reconstruct already-redacted artifacts; it also restores the previous gap in field-level protection.
- Existing tests omit overlapping patterns, opaque padded tokens, and several JSON boundary cases. Sidecar integration tests and representative-scale refresh benchmarks were not run during this review.

## Open questions

None.
