---
pr: kubernetes-sigs/prow#577
title: "updateconfig: Add ACL support for ConfigMap updates"
head_sha: 63cbfe6405c75233925dc73adce431eedb34a5bc
base: main
reviewed_at: 2026-06-22T16:51:14Z
verdict: request-changes
---

## Findings

### [blocking] Config-bootstrapper silently blocked by allowed_repos
- where: `cmd/config-bootstrapper/main.go:149` + `pkg/plugins/config.go:603-631`
- concern: config-bootstrapper passes `""` as the repo argument. `IsAllowed("")` returns false when `AllowedRepos` is non-empty because empty string matches no entry. The bootstrapper silently skips those ConfigMaps (Debug-level log only). The comment on line 148 is misleading. Fix: add `if repo == "" { return true }` at the top of `IsAllowed`.

### [should-fix] Trailing-slash entries pass validation but silently never match
- where: `cmd/checkconfig/main.go:1552-1566`
- concern: `validateRepoName("kubernetes/")` passes (SplitN yields `["kubernetes", ""]`, satisfies all checks). At runtime this silently behaves as org-level entry. Add `len(s) == 2 && s[1] == ""` check.

### [should-fix] Non-deterministic test due to map iteration order
- where: `cmd/checkconfig/main_test.go` "multiple maps with errors" test case
- concern: Asserts exact error string containing errors from two map keys. Go map iteration is non-deterministic; test will flake. Assert each error substring independently or sort before comparing.

### [nit] ACL denials logged at Debug level only
- where: `pkg/plugins/updateconfig/updateconfig.go:314-320`
- concern: Operators won't see ACL denials unless debug logging is enabled. Info or Warn level would surface misconfigurations faster.

### [nit] Whitespace in ACL entries not validated
- where: `cmd/checkconfig/main.go:1552-1566`
- concern: Entries like `" kubernetes"` pass validation but never match at runtime. TrimSpace or explicit check would catch YAML copy-paste errors.

### [nit] Value receiver copies full ConfigMapSpec struct
- where: `pkg/plugins/config.go:603`
- concern: Pointer receiver would avoid copying scalar fields. Slices/maps are reference types so no deep copy, but pointer receiver is cleaner.

### [nit] FilterChanges now has 6 positional parameters
- where: `pkg/plugins/updateconfig/updateconfig.go:284`
- concern: Becoming harder to read at call sites. Not worth changing now; consider an options struct if another param is added.

### [nit] No test for trailing-slash entry behavior
- where: `pkg/plugins/config_test.go`, `cmd/checkconfig/main_test.go`
- concern: No test case for entries like `"kubernetes/"`. Would document intended behavior or catch the proposed validation fix.

### [question] Are there external consumers of FilterChanges?
- where: `pkg/plugins/updateconfig/updateconfig.go:284`
- concern: Adding `repo` parameter is a breaking change to an exported function. Both in-tree call sites updated. External consumers get a compile error (correct failure mode), but worth confirming none are known.

## Checked
- Deny-before-allow precedence is correct default-safe ordering
- 29 test cases across IsAllowed (14), FilterChangesWithACLs (7), validateConfigUpdaterACLs (8)
- Fully backwards-compatible: omitempty tags, zero-value defaults preserve "allow all"
- Reuses config.NewOrgRepo for org extraction
- checkconfig validation registered as default warning
- Generated config docs updated
- Rollback safe: remove ACL fields from YAML + deploy prior binary

## Open questions
- Was config-bootstrapper behavior with allowed_repos intentional? Should docs warn that bootstrapper won't update ACL-protected ConfigMaps?
- Are there known external consumers of FilterChanges affected by the signature change?

## Followups

### bug-fix: Make IsAllowed bypass ACLs for empty repo (config-bootstrapper)
- category: bug-fix
- necessity: must
- where: `pkg/plugins/config.go:603`, `cmd/config-bootstrapper/main.go:149-150`, `pkg/plugins/config_test.go`
- prompt:
```
In kubernetes-sigs/prow, following the merge of PR #577 ("updateconfig: Add ACL support for ConfigMap updates", merge commit e821f0db6d3151be5dc9bf950cfc63446043e72f), there is a bug in the new ACL logic.

The config-bootstrapper (cmd/config-bootstrapper/main.go) passes "" as the repo argument to FilterChanges. The IsAllowed method on ConfigMapSpec (pkg/plugins/config.go) returns false when AllowedRepos is non-empty and repo is "", because the empty string matches no entry. This silently blocks the bootstrapper from updating ConfigMaps that have allowed_repos configured.

Fix this:
1. In pkg/plugins/config.go, in the IsAllowed method, add an early return at the top: if repo == "" { return true }. This treats empty repo as a non-PR context that bypasses ACL checks.
2. In cmd/config-bootstrapper/main.go around line 149-150, update the misleading comment. The current comment says "ACL restrictions won't apply in this mode unless explicitly configured" — replace it with something accurate like "Pass empty repo to bypass ACL checks, since config-bootstrapper works with local files, not PRs."
3. In pkg/plugins/config_test.go, in TestConfigMapSpecIsAllowed: update the existing test case "empty repo with allowed repos - denied" to expect true instead of false, and rename it to reflect the new behavior (e.g., "empty repo with allowed repos - allowed (bypass)"). Keep the "empty repo with no ACLs - allowed" case as-is.

Acceptance criteria: go test ./pkg/plugins/... and go test ./cmd/config-bootstrapper/... pass. The IsAllowed method returns true for empty repo regardless of ACL configuration.

Out of scope: changing the FilterChanges signature, adding a skipACL parameter, modifying log levels, or any other ACL behavioral changes.
```

### tests: Fix non-deterministic checkconfig ACL validation test
- category: tests
- necessity: should
- where: `cmd/checkconfig/main_test.go` "multiple maps with errors" test case
- prompt:
```
In kubernetes-sigs/prow, following the merge of PR #577 ("updateconfig: Add ACL support for ConfigMap updates", merge commit e821f0db6d3151be5dc9bf950cfc63446043e72f), there is a flaky test in cmd/checkconfig/main_test.go.

The test case "multiple maps with errors" in TestValidateConfigUpdaterACLs asserts an exact error string that contains errors from iterating a Go map (pcfg.ConfigUpdater.Maps with keys "config.yaml" and "plugins.yaml"). Go map iteration order is non-deterministic, so the two error substrings can appear in either order, making the test flake.

Fix this by changing the assertion approach. Instead of comparing the full aggregated error string, assert that each expected error substring is present independently. For example, check that the error string contains both `config_updater.maps["config.yaml"].allowed_repos: org/repo cannot be empty` and `config_updater.maps["plugins.yaml"].denied_repos: you cannot set a repo without an org` as substrings, regardless of order. Alternatively, split the error string and sort before comparing.

Keep the test's coverage intent intact — it should still verify that errors from multiple map entries are all reported.

Acceptance criteria: go test ./cmd/checkconfig/... passes reliably (run it multiple times to confirm no flake). The test still catches the same validation errors.

Out of scope: changing the validation logic itself, modifying other test cases, or any functional changes.
```

### validation: Reject trailing-slash and whitespace entries in validateRepoName
- category: validation
- necessity: should
- where: `cmd/checkconfig/main.go:1552-1566`, `cmd/checkconfig/main_test.go`, `pkg/plugins/config_test.go`
- prompt:
```
In kubernetes-sigs/prow, following the merge of PR #577 ("updateconfig: Add ACL support for ConfigMap updates", merge commit e821f0db6d3151be5dc9bf950cfc63446043e72f), the validateRepoName function in cmd/checkconfig/main.go accepts entries that will silently misbehave at runtime.

Two gaps:
1. Trailing-slash entries like "kubernetes/" pass validation (SplitN yields ["kubernetes", ""], which satisfies all current checks). At runtime, this silently behaves as an org-level entry, which is confusing.
2. Entries with leading/trailing whitespace like " kubernetes" or "kubernetes/ test-infra" pass validation but will never match anything at runtime, because IsAllowed does exact string comparison.

Fix this in validateRepoName (cmd/checkconfig/main.go):
1. After the existing `len(s) > 2` check, add: if len(s) == 2 && s[1] == "" { return errors.New("repo name cannot be empty after org") } — this catches trailing-slash entries.
2. Add a whitespace check: if strings.TrimSpace(repo) != repo { return errors.New("org/repo must not contain leading or trailing whitespace") } — put this near the top, before the split.

Then add test cases:
- In TestValidateConfigUpdaterACLs (cmd/checkconfig/main_test.go): add cases for "kubernetes/" in allowed_repos (expect error about empty repo after org) and " kubernetes" in denied_repos (expect error about whitespace).
- In TestConfigMapSpecIsAllowed (pkg/plugins/config_test.go): add a case for "kubernetes/" in AllowedRepos with repo "kubernetes/test-infra" to document runtime behavior (it would match via org, but the point is that checkconfig now rejects it before it gets to runtime).

Acceptance criteria: go test ./cmd/checkconfig/... and go test ./pkg/plugins/... pass. validateRepoName rejects "kubernetes/", " kubernetes", and "kubernetes " with clear error messages.

Out of scope: changing IsAllowed logic, normalizing inputs at runtime (e.g., trimming whitespace in IsAllowed), or modifying any other validation functions.
```
