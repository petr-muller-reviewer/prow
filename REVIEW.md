---
pr: kubernetes-sigs/prow#1008
title: "peribolos: add end-of-run change report"
head_sha: f2adb95bae4f760973642b96f364a8b1c2111191
base: main
reviewed_at: 2026-10-12T17:32:58Z
verdict: request-changes
---

# Review

## Verdict

Request changes: a reconciliation failure prevents the promised report from being written, even when earlier steps already applied changes.

The report covers the requested mutation categories and provides stable, censored JSON output on a successful run. However, partial application is a normal failure mode for a reconciler, and the absence of a report in that case loses the audit information this feature is intended to provide.

## What this PR does

- Adds an opt-in `--report-path` flag and writes a versioned JSON report at the end of a successful run.
- Records adds, updates, and removals for organization metadata, memberships, teams, repositories, collaborators, and organization role assignments.
- Captures role/permission or changed-field descriptors while censoring serialized output.
- Sorts changes and initializes all action counts so output is stable and easy to consume.

## Findings

### [should-fix] Write the report after partial reconciliation failures
- where: `cmd/peribolos/main.go:208-215`
- concern: `logrus.Fatalf` exits as soon as any organization reconciliation returns an error, so control never reaches `emitReport`. If earlier reconciliation steps or organizations already applied changes, `--report-path` is left without an artifact and the run loses its audit trail. Emit the accumulated report on failure as well, with failure semantics made clear to consumers.
- excerpt: |
    for name, orgcfg := range cfg.Orgs {
        if err := configureOrg(o, githubClient, name, orgcfg, report); err != nil {
            logrus.Fatalf("Configuration failed: %v", err)
        }
    }
    logrus.Info("Finished syncing configuration.")

    emitReport(o, report)

## Checked

- Compared the PR head with `upstream/main` and checked the PR description as the available spec source.
- Reviewed the reporting paths for each requested change category, the deterministic sort and summary, and the secret censoring call.
- `CONTRIBUTING.md` has no repository-specific code standards relevant to this change; no standards violations found.
- Did not run tests.

## Open questions

- Should the report include a run status or reconciliation error details so consumers can distinguish a complete report from a partial one?
