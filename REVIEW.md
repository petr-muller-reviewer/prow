---
pr: kubernetes-sigs/prow#728
title: "chore: upgrade gopkg.in/robfig/cron.v2 to github.com/robfig/cron/v3"
head_sha: 519165993bc0d7827b83fe27f9e5506be654cf4d
base: main
reviewed_at: 2026-05-26T16:17:25Z
verdict: approve
---

## Summary

Upgrades the cron dependency from the unmaintained `gopkg.in/robfig/cron.v2` to `github.com/robfig/cron/v3`. Changes touch `go.mod`/`go.sum`, `pkg/config/config.go` (validation), `pkg/cron/cron.go` (scheduler), and `pkg/config/config_test.go` (cosmetic cleanup + new test cases).

## Findings

### [should-fix] cronParser duplicated across two packages
- where: `pkg/config/config.go:66-69`, `pkg/cron/cron.go:33-36`
- concern: Identical `cronParser` vars defined independently in both packages. If flags diverge, `config.validatePeriodics()` could accept strings that `pkg/cron` rejects at runtime. The parser in `pkg/cron` should be exported and imported by `pkg/config`.
- excerpt: |
    // pkg/config/config.go:66
    var cronParser = cron.NewParser(
        cron.SecondOptional | cron.Minute | cron.Hour | cron.Dom | cron.Month | cron.Dow | cron.Descriptor,
    )

    // pkg/cron/cron.go:33 — identical
    var cronParser = cron.NewParser(
        cron.SecondOptional | cron.Minute | cron.Hour | cron.Dom | cron.Month | cron.Dow | cron.Descriptor,
    )

### [nit] TZ=UTC string prefix could use cron.WithLocation instead
- where: `pkg/cron/cron.go:156`
- concern: `"TZ=UTC "+cron` works via the Descriptor flag, but v3 provides `cron.WithLocation(time.UTC)` as a first-class option that is clearer about intent. Not broken, just a minor idiomatic improvement opportunity.
- excerpt: |
    id, err := c.cronAgent.AddFunc("TZ=UTC "+cron, func() {

### [nit] Cosmetic test changes inflate diff
- where: `pkg/config/config_test.go` (bulk of the diff)
- concern: Changes like `var testCases = []` → `testCases := []`, blank line removals, and struct literal brace reformatting are unrelated to the dependency upgrade. Not wrong, but makes the upgrade harder to review in isolation.
- excerpt: |
    -	var testCases = []struct {
    +	testCases := []struct {

## Checked
- Parser flags (`SecondOptional | Minute | Hour | Dom | Month | Dow | Descriptor`) preserve v2 behavior for both standard 5-field and optional-second 6-field expressions
- `@every` descriptor handling in `pkg/cron/cron.go:174` (`strings.HasPrefix(cron, "@every")`) still works — Descriptor flag enables it
- `cron.WithParser(cronParser)` usage in `cron.New()` is correct v3 API
- Error message in test updated to match v3 format (`"expected 5 to 6 fields, found 1: [hello]"`)
- No remaining references to `gopkg.in/robfig/cron.v2` anywhere in the codebase
- Two new test cases for 5-field and 6-field cron validation are correct

## Open questions
- Would the author be open to exporting the parser from `pkg/cron` and importing it in `pkg/config` to avoid the duplication?
- Was the cosmetic test cleanup intentional, or did it come from running a formatter? Either way it's fine, but knowing helps with future review scope.
