---
issue: kubernetes-sigs/prow#502
title: "Enhancement: Allow combining `run_if_changed` and `skip_if_only_changed` for more flexible job triggering"
state: open
labels: kind/feature, area/plugins
main_sha: e601a1ffafd7d8d3a781238a4c5f4233d6248f68
triaged_at: 2026-07-27T00:20:48Z
verdict: wontfix
---

## Verdict

Close as wontfix. The proposal is well-formed and in-scope, but two maintainers (petr-muller, BenTheElder) already gave substantive, unrebutted technical objections in-thread: combining the two fields reintroduces a "silent job never triggers" footgun that the current mutual-exclusivity check exists to prevent, and the same outcome is generally achievable today with `skip_if_only_changed` alone. No new argument has appeared since 2025-10-07; the issue has only been kept alive by stale-bot cycling.

## What the issue reports

- `run_if_changed` and `skip_if_only_changed` are mutually exclusive on a job, enforced by `validateTriggering` in `pkg/config/config.go`.
- Author wants to combine both with AND semantics: run if `run_if_changed` matches something, unless everything also falls under `skip_if_only_changed`.
- Motivating case: matching broad file types (`*.yaml`, `*.go`, etc.) via `run_if_changed` while excluding a specific directory (`konflux/`) that also contains those file types.
- Follow-up discussion established Go's RE2 engine has no lookahead/lookbehind, so a single-regex "match X except under dir Y" is awkward — directory enumeration is the only single-regex workaround.
- Two maintainers rejected the proposal on risk grounds rather than technical infeasibility.

## Analysis

### Probable outcome / maintainer position

`petr-muller` (2025-08-05): the example given is already achievable with `run_if_changed` alone; the only real use case (using `skip_if_only_changed` to carve an exception out of `run_if_changed`) is a footgun, since a misconfiguration between the two fields could silently make a job never trigger — exactly what the current check prevents.

`BenTheElder` (2025-10-07), independently: agrees combination would make the feature "even more of a footgun"; `skip_if_only_changed` alone is the safer, easier-to-audit primitive, and `run_if_changed` is already prone to misuse (e.g. missing `go.sum`/`go.work`/`Makefile` as inputs) without adding combinatorial surface.

No comment since 2025-10-07 has addressed this objection; the thread since is only `k8s-triage-robot` stale/unstale cycling (3 rounds, most recently unstaled 2026-07-06).

### Related code

`pkg/config/config.go:3121-3131`
```go
func validateTriggering(job Presubmit) error {
	...
	if job.RunIfChanged != "" && job.SkipIfOnlyChanged != "" {
		return fmt.Errorf("job %s declares run_if_changed and skip_if_only_changed, which are mutually exclusive", job.Name)
	}
	...
}
```
This is the exact check the issue asks to remove.

`pkg/config/jobs.go:374-386` — `RegexpChangeMatcher` struct holds a single compiled regex `reChanges`, populated from whichever of `RunIfChanged`/`SkipIfOnlyChanged` is set (`setChangeRegexes`, `pkg/config/config.go:3312-3327`). Supporting combination would require splitting this into two fields.

`pkg/config/jobs.go:461-472` — `RunsAgainstChanges`, the OR-style matcher that would need to become an AND across two separate regexes:
```go
func (cm RegexpChangeMatcher) RunsAgainstChanges(changes []string) bool {
	for _, change := range changes {
		if cm.RunIfChanged != "" && cm.reChanges.MatchString(change) {
			return true
		} else if cm.SkipIfOnlyChanged != "" && !cm.reChanges.MatchString(change) {
			return true
		}
	}
	return false
}
```

`pkg/config/jobs.go:401-415` — `Brancher.ShouldRun`, a structurally similar allow/deny pair (`Branches`/`SkipBranches`) that *does* combine, with `SkipBranches` given precedence over `Branches`. Internal precedent for a combined mechanism, though its semantics (precedence override) differ from the issue's proposed AND-combination.

### Effort if ever reconsidered

Mechanically small (Level 1-2: ~3-5 files, straightforward struct/logic split, backward compatible, existing test patterns in `pkg/config/jobs_test.go` / `pkg/config/config_test.go` to extend). The blocker is not implementation difficulty — it's that the design itself was already rejected by two maintainers on risk grounds. A PR implementing this would need to overturn that recorded position to be mergeable.

### Related issues

- `#151` — referenced by the issue author as prior related discussion on file-change-based job triggering.

## Findings

### [cause] Design proposal declined by maintainer consensus
- detail: Not a bug or missing-info case; a well-formed feature request that two maintainers independently rejected on footgun/risk grounds, with no rebuttal since 2025-10-07.
- evidence: issue comments by petr-muller (2025-08-05) and BenTheElder (2025-10-07).

### [related-code] Mutual-exclusivity validation
- where: `pkg/config/config.go:3121-3131`
- excerpt: |
    if job.RunIfChanged != "" && job.SkipIfOnlyChanged != "" {
        return fmt.Errorf("job %s declares run_if_changed and skip_if_only_changed, which are mutually exclusive", job.Name)
    }

### [related-code] Duplicate check paired with AlwaysRun
- where: `pkg/config/config.go:3105-3117`
- excerpt: |
    if job.AlwaysRun {
        if job.RunIfChanged != "" { ... mutually exclusive ... }
        if job.SkipIfOnlyChanged != "" { ... mutually exclusive ... }
    }

### [related-code] RegexpChangeMatcher struct — single regex field
- where: `pkg/config/jobs.go:374-386`
- excerpt: |
    type RegexpChangeMatcher struct {
        RunIfChanged string `json:"run_if_changed,omitempty"`
        SkipIfOnlyChanged string `json:"skip_if_only_changed,omitempty"`
        reChanges *CopyableRegexp
    }

### [related-code] RunsAgainstChanges matching logic
- where: `pkg/config/jobs.go:461-472`
- excerpt: |
    func (cm RegexpChangeMatcher) RunsAgainstChanges(changes []string) bool {
        for _, change := range changes {
            if cm.RunIfChanged != "" && cm.reChanges.MatchString(change) {
                return true
            } else if cm.SkipIfOnlyChanged != "" && !cm.reChanges.MatchString(change) {
                return true
            }
        }
        return false
    }

### [related-code] Brancher.ShouldRun — internal precedent for combining allow/deny
- where: `pkg/config/jobs.go:401-415`
- excerpt: |
    func (br Brancher) ShouldRun(branch string) bool {
        if br.RunsAgainstAllBranch() { return true }
        if len(br.SkipBranches) != 0 && br.reSkip.MatchString(branch) { return false }
        if len(br.Branches) == 0 || br.re.MatchString(branch) { return true }
        return false
    }

### [related-issue] Prior related discussion
- ref: kubernetes-sigs/prow#151
- relevance: referenced by the issue author as related prior discussion on file-change-based job triggering.

## Checked

- Confirmed `validateTriggering` and `RegexpChangeMatcher` in current `pkg/config` match what the issue describes, at HEAD `e601a1ff`.
- Read full comment thread (13 comments): maintainer objections, author's real-world use case (`konflux/` exclusion), RE2 lookahead/lookbehind limitation discussion, repeated stale-bot cycling.
- Searched for a linked/implementing PR — none exists.
- Checked `Brancher.ShouldRun` as a structural precedent for combined allow/deny matching.

## Next steps

- Post closing comment (below) and `/close` the issue.
- If a future report shows a concrete case that `skip_if_only_changed` alone truly cannot express, file it as a new issue rather than reopening this one.

## Open questions

- None requiring author input — the technical objection is already fully articulated in-thread by two maintainers.

## Suggested closing comment

```
Closing as wontfix. Two maintainers (@petr-muller, @BenTheElder) raised the same core concern above: combining run_if_changed and skip_if_only_changed reintroduces a footgun that the current mutual-exclusivity check exists to prevent — a misconfiguration between the two fields would silently cause jobs to never trigger, and the same outcome is generally achievable today with skip_if_only_changed alone.

If you have a use case that genuinely can't be expressed with either field alone (not just the docs-carve-out case discussed here, which RE2's lack of lookahead/lookbehind makes awkward but isn't unique to this proposal), please open a new issue describing it — happy to reconsider with a concrete example that shows the existing primitives are insufficient.

/close
```
