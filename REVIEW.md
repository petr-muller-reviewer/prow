---
pr: kubernetes-sigs/prow#1009
title: "Allow sharding Tide context policies in supplemental config"
head_sha: e7d18e7ad8ef2a600ba9aa778cd600e53993d418
base: main
reviewed_at: 2026-10-14T15:20:04Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-10-14T15:56:03Z
  gated_head_sha: e7d18e7ad8ef2a600ba9aa778cd600e53993d418
  reviewed_head_sha: e7d18e7ad8ef2a600ba9aa778cd600e53993d418
---

# Review

## Gate

**Decision: merge.** The PR head is unchanged since review, and no GitHub review submissions or substantive inline comments were found. The only local finding is a non-blocking diagnostic suggestion; it does not create merge risk.

### Findings disposition

- No unresolved gating findings. The local nit about including the earlier shard path in boolean-conflict errors is not a correctness or operational blocker.

### Gating list

- None.

### Independent merge-risk pass

- No exported Go API, CLI flag, or existing configuration field was removed or changed. Existing main-config behavior remains unchanged unless an operator opts into supplemental `tide.context_options`.
- Config impact is scoped to repositories and branches named by the supplemental policies. Operators should pass the supplemental-config flags to Tide and `checkconfig`, then run `checkconfig`; the new behavior is documented.
- Rolling back to an older binary requires removing `tide.context_options` from supplemental files, since the older loader rejects that field there. This affects only deployments that opt into the new configuration.

## Verdict

Approve with a non-blocking suggestion. The three perspectives found no correctness issues or breaking changes, and deployment risk is low. Consider including the earlier shard path in same-scope boolean-conflict diagnostics to make conflicts easier to locate across large shard sets.

## What this PR does

- Allows Tide context policies in supplemental Prow config files.
- Combines context lists and rejects conflicting explicit booleans at matching policy scopes.
- Extends config scope detection so `checkconfig` can validate org and repo shard placement.
- Documents the shard structure, merge behavior, and operational flags.

## Findings

### [nit] Include the earlier shard in conflict diagnostics
- where: `pkg/config/tide.go:408-409`
- concern: When two supplemental files set different values for a boolean at the same scope, the error names the scope and field, but not the earlier file that supplied the conflicting value. Including both shard paths would make this easier to diagnose in large shard sets.
- excerpt: |
    if field.a != nil && field.b != nil && *field.a != *field.b {
        errs = append(errs, fmt.Errorf("%s: conflicting values for %s", scope, field.name))
    }

## Checked

- Code quality review found no actionable correctness, safety, or performance issues.
- The merge path preserves existing context policies, unions lists, rejects same-scope boolean conflicts, and leaves hierarchical inheritance in the existing resolver.
- Deployment review found the change opt-in and backward-compatible for existing main-config YAML; operators adding supplemental policies should run `checkconfig`.
- Rolling back to a binary without this change requires removing `tide.context_options` from supplemental files.
- The PR description reports targeted tests and lint passing; tests were not rerun during this review.

## Open questions

None.
