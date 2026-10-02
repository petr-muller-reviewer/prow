---
pr: kubernetes-sigs/prow#583
title: "`peribolos`: Add Repository Fork Management Support"
head_sha: 1082b0f155e55439f237d07edc039c92bdd65b74
base: main
reviewed_at: 2026-09-29T16:10:39Z
verdict: request-changes
gate:
  decision: do-not-merge
  gated_at: 2026-10-01T12:46:52Z
  gated_head_sha: 1082b0f155e55439f237d07edc039c92bdd65b74
  reviewed_head_sha: 1082b0f155e55439f237d07edc039c92bdd65b74
refresh_log:
  - at: 2026-09-29T16:10:39Z
    old_sha: 8c2cfc86ec0f0efff8cc6b0dce1aef9c49084d8f
    new_sha: 1082b0f155e55439f237d07edc039c92bdd65b74
    summary: "Three-line comment about fork-list propagation; no behavior change or findings resolved."
---

## Gate

**Decision: do-not-merge.** The PR head is unchanged since the saved review, and three blocking fork-reconciliation failures remain. Team access can be removed from a valid fork or changed on a repository rejected as the wrong fork; malformed fork configuration can still permit repository and collaborator writes before the run fails. Resolve these before merging. The remaining should-fix findings also lack a disposition.

Gating findings:

- **Not addressed — local `REVIEW.md`, `cmd/peribolos/main.go:1154-1159,1899-1930` (blocks merge):** Team reconciliation receives neither the config-to-actual fork-name map nor `conflictedForks`. A renamed fork can lose its existing team permission, while a same-name wrong repository can receive team access changes. Map names and suppress actions for rejected fork entries.
- **Not addressed — local `REVIEW.md`, `cmd/peribolos/main.go:1106-1132,1490-1497` (blocks merge):** Fork validation returns an error without conflict information, but repository and collaborator stages still run. Reject the invalid configuration before dependent writes, or exclude its entries from all downstream stages.
- **Not addressed — local `REVIEW.md`, `cmd/peribolos/main.go:1591-1599,1382-1384` (should fix):** A fork found under a `previously` name is mapped to that old name and deliberately not renamed. This also leaves the earlier reviewer request for an explicit configured fork name unresolved for such forks.
- **Not addressed — local `REVIEW.md`, `cmd/peribolos/main.go:1472-1475,1106-1112` (should fix):** `fork.from` still accepts extra slash segments; fork creation still precedes case-insensitive repository-name collision validation. Correct both validation paths before accepting fork changes.

Independent merge risk:

- **Existing deployments with unchanged configs:** No new fork API call or fork reconciliation occurs without `fork:` entries (`cmd/peribolos/main.go:1500-1502`); the reported runtime failures are confined to feature adoption.
- **Dump-to-apply behavior:** `--dump` now emits `fork:` for existing forks (`cmd/peribolos/main.go:479-480`). Adopting a newly dumped config opts those repositories into fork reconciliation on an ordinary `--fix-repos` run because `--fix-forks` inherits that flag (`main.go:112-123`). Document this rollout effect.
- **Go source compatibility:** Adding `CreateForkInOrg` to exported `github.RepositoryClient` (`pkg/github/client.go:208`) requires out-of-tree implementations to add the method. This is a compile-time integration risk, not a runtime break for unchanged peribolos configs. The earlier `org.Repo` metadata-field extraction was removed, so its keyed literals remain compatible.

## Verdict

**Request changes.** Fork resolution is not carried through to team repository permissions: a renamed fork can lose access, and a rejected same-name repository can still receive access changes. Invalid fork configuration can also leave repository or collaborator writes in its wake. The remaining findings concern rename handling, source validation, and validation order.

## What this PR does

- Adds `fork:` configuration to peribolos repository entries and creates organization forks through GitHub's forks API.
- Finds existing forks by their upstream parent, waits for new forks to become available, and passes actual fork names to repository and collaborator reconciliation.
- Adds `--fix-forks`, defaulting to the `--fix-repos` setting unless explicitly set.
- Includes a fork's upstream in `--dump` output and adds fork client and reconciliation tests.

Since previous review:

- Added a comment at `cmd/peribolos/main.go:1419-1421` noting that a just-created fork may be visible to `GetRepo` before `GetRepos`; metadata then waits until a later run. No executable code changed.

## Findings

### [blocking] Map fork names before reconciling team repository permissions
- where: `cmd/peribolos/main.go:1154-1159`
- concern: `configureForks` can map a config key to an existing fork with a different name, but `configureTeamRepos` still receives the original `team.Repos` keys. If the team currently has access to the actual fork, reconciliation attempts a grant on the absent config-key name and removes the team's permission on the real fork. Pass the mapping through team reconciliation before computing its permission delta.
- excerpt: |
    if err := configureTeamRepos(client, githubTeams, name, orgName, team); err != nil {
        return fmt.Errorf("failed to configure %s team %s repos: %w", orgName, name, err)
    }

### [blocking] Skip team permission writes for rejected fork entries
- where: `cmd/peribolos/main.go:1126-1129,1154-1159`
- concern: A non-fork or wrong-upstream repository occupying a configured fork name is recorded in `conflictedForks`, which protects repository metadata and collaborators. Team repository reconciliation does not receive that set, so a team entry for the name can still grant or change access on the repository that the fork check rejected. Exclude conflicted fork entries from team permission actions.
- excerpt: |
    if conflictedForks.Has(repoName) {
        logrus.WithField("repo", repoName).Debug("skipping collaborators for repo flagged as a fork conflict")
        continue
    }

### [blocking] Stop downstream writes when fork configuration fails validation
- where: `cmd/peribolos/main.go:1490-1495`
- concern: Invalid `fork.from` returns an error and nil conflict information, but `configureOrg` defers the error and still runs `configureRepos` and collaborators. For an existing non-fork at that config key, malformed fork configuration can therefore change its metadata or access before the run fails. Stop before downstream reconciliation or mark invalid fork entries as unavailable to every dependent stage.
- excerpt: |
    if len(validationErrors) > 0 {
        sort.Slice(validationErrors, func(i, j int) bool {
            return validationErrors[i].Error() < validationErrors[j].Error()
        })
        return nil, nil, utilerrors.NewAggregate(validationErrors)
    }

### [should-fix] Honor `previously` when locating and renaming a fork
- where: `cmd/peribolos/main.go:1591-1599`
- concern: When a fork exists under a name listed in `Repo.Previously`, the parent index finds it but maps the config key to that old name. `configureRepos` then deliberately updates using the old name, so changing the config key with `previously` never performs the requested rename. Rename the matched fork or report the mismatch explicitly.
- excerpt: |
    if actualName, ok := forkParentIndex()[expectedUpstream]; ok {
        forkNames[repoName] = actualName
        repoLogger.WithField("actual_name", actualName).Info("fork of upstream already exists with different name")
        continue
    }

### [should-fix] Reject extra path segments in `fork.from`
- where: `cmd/peribolos/main.go:1472-1475`
- concern: `strings.SplitN(..., "/", 2)` accepts `owner/repo/extra` and treats `repo/extra` as a repository name. The create call then builds a malformed GitHub API path rather than rejecting the input as invalid configuration. Require exactly two nonempty segments.
- excerpt: |
    parts := strings.SplitN(repoCfg.Fork.From, "/", 2)
    if len(parts) != 2 || parts[0] == "" || parts[1] == "" {
        validationErrors = append(validationErrors, fmt.Errorf("invalid fork from format %q for repo %s, expected 'owner/repo'", repoCfg.Fork.From, repoName))
        continue
    }

### [should-fix] Validate repository-name collisions before creating forks
- where: `cmd/peribolos/main.go:1106-1109`
- concern: `configureForks` runs before `configureRepos`, where `validateRepos` detects case-insensitive duplicate repo names. With two fork config keys such as `Fork` and `fork` pointing to different upstreams, the first fork can be created before the invalid repository configuration is rejected. Run the repository-name validation before any fork creation, including fork-only runs.
- excerpt: |
    if !opt.fixForks {
        logrus.Info("Skipping repository forks configuration")
    } else {
        forkNames, conflictedForks, forkErr = configureForks(client, orgName, orgConfig)
    }

## Checked

- The reviewed PR head is `1082b0f155e55439f237d07edc039c92bdd65b74`; the only change since the prior review is a three-line comment in `cmd/peribolos/main.go`.
- The fork API client sends the target organization, requested name, and `default_branch_only`; its GitHub App token scope is the target organization.
- Newly created forks are polled for readiness, with retries limited to 404 responses. Existing same-name forks are checked against their parent before use.
- `conflictedForks` prevents repository metadata and collaborator writes for detected same-name conflicts; the findings above concern the remaining stages and early validation errors.
- The earlier `RepoMetadata` extraction is gone, preserving the previous `org.Repo` field shape apart from the new `Fork` field.
- The added tests cover fork creation, basic existing-fork matching, invalid formats without extra segments, and repository metadata/conflict paths. I did not run tests at this head during the read-only review.

## Open questions

- Should an existing correctly sourced fork under a different name be renamed to the config key, or should the mismatch be reported as a conflict when `previously` does not authorize the rename?
