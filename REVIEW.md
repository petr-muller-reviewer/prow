---
pr: kubernetes-sigs/prow#774
title: "peribolos: add internal repository visibility support"
head_sha: 98babf79b963f6375b1ed97bb529097caa850e2a
base: main
reviewed_at: 2026-07-22T14:54:24Z
verdict: approve
recommended_rereview:
  - at: 2026-09-02T16:53:20Z
    old_sha: 98babf79b963f6375b1ed97bb529097caa850e2a
    new_sha: 674a52169165592fc021626d57c951cc4654df3b
    reason: The reviewed SHA is no longer reachable; the replacement PR head substantially rewrites the peribolos visibility implementation and tests in areas with existing findings.
---

# Review: kubernetes-sigs/prow#774

Replaces `Private *bool` with `Visibility *RepoVisibility` (public/private/internal) on
`org.Repo`; backward-compatible `UnmarshalJSON` translates deprecated `private:` with a
throttled warning, rejects both-set, treats `null` as absent; `github.RepoRequest.Private`
→ `Visibility *string`; `github.Repo` gains `Visibility string` (keeps `Private bool`);
`sanitizeRepoDelta` gates only transitions to `public` behind `--allow-repo-publish`;
`PruneRepoDefaults` prunes `visibility: public` as default. Combined method: 8-angle
finder review + verification, plus a 3-perspective maintainer-review panel (code quality,
maintainability, deployment risk) synthesized by an advisor agent. All reviewers converge
on APPROVE_WITH_SUGGESTIONS / approve; no blocking findings.

## Re-review Recommended

### 2026-09-02T16:53:20Z — `98babf79b963f6375b1ed97bb529097caa850e2a` → `674a52169165592fc021626d57c951cc4654df3b`

- The original reviewed SHA is not an ancestor of the current PR head. The current PR contains
  one replacement commit, `674a52169 peribolos: add internal repository visibility support`,
  authored 2026-08-18, with 492 additions and 136 deletions across seven files.
- The replacement touches `cmd/peribolos/main.go`, `cmd/peribolos/main_test.go`,
  `pkg/config/org/{org.go,org_test.go}`, and `pkg/github/{types.go,types_test.go,client_test.go}`:
  the same implementation and test paths covered by the existing visibility findings.
- Since the prior review, the author stated that they simplified/DRYed the code and added tests;
  review-thread replies marked several comments addressed. The PR was merged on 2026-08-18.

The unreachable baseline plus substantial replacement code in the areas of the existing findings
meets the full re-review threshold. The existing findings remain preserved and have not been
revalidated against `674a52169`.

## Findings

### [should-fix] empty/unknown API visibility silently drops privacy info or causes perpetual delta
- where: `cmd/peribolos/main.go:371-377` (dumpOrgConfig), `cmd/peribolos/main.go:1105-1111` (newRepoUpdateRequest)
- concern: Old code always had `full.Private`/`current.Private` populated. New code parses
  `full.Visibility`/`current.Visibility` and on empty/unrecognized string: dump only warns and
  omits the field (a private repo dumped from an API not returning `visibility` loses its
  privacy marking; recreating from that dump yields a public repo). Update path treats empty
  `current.Visibility` as unequal to any configured value, generating a delta every run —
  including a perpetual `visibility: public` delta for legacy `private: false` configs, always
  blocked by `sanitizeRepoDelta` without `--allow-repo-publish` (spurious error every sync).
  Flagged independently by the finder review and by both the code-quality and deployment-risk
  maintainer-panel reviewers — high-confidence convergence. Trigger requires an API that omits
  `visibility` (older GHES); unlikely on github.com/modern GHES but the failure mode is
  security-flavored (confidentiality) on the create path specifically. Suggested fix: fall back
  to `Private` bool when `Visibility` string is empty, both in dump and in the update-delta
  comparison.
- excerpt: |
    if err := visibility.UnmarshalText([]byte(full.Visibility)); err != nil {
        logrus.WithField("repo", full.FullName).WithField("visibility", full.Visibility).Warn("Unknown repo visibility from GitHub API, omitting field")
    } else {
        visibilityPtr = &visibility
    }
    ...
    var visibilityPtr *string
    if repo.Visibility != nil {
        want := string(*repo.Visibility)
        if want != current.Visibility {
            visibilityPtr = &want
        }
    }

### [should-fix] repo-creation wire format change is a confidentiality hazard on older GHE
- where: `cmd/peribolos/main.go:1030-1044` (newRepoCreateRequest)
- concern: Sends `visibility: private` where the old code sent `private: true`. If a GHE
  version's repo-create endpoint ignores an unrecognized `visibility` parameter, a repo
  configured private would be silently created public — no error, a genuine confidentiality
  failure. Flagged by the deployment-risk maintainer-panel reviewer. Only a hazard for peribolos
  users on older/unverified GHE; github.com and GHE 3.x accept `visibility`.
- excerpt: |
    var visibility *string
    if definition.Visibility != nil {
        s := string(*definition.Visibility)
        visibility = &s
    }

### [question] private→internal ungated by --allow-repo-publish — intended scope?
- where: `cmd/peribolos/main.go:1145-1148` (sanitizeRepoDelta)
- concern: Guard only blocks deltas to `public`. Private→internal widens exposure from repo
  collaborators to the entire enterprise with no flag. Not a regression (internal wasn't
  expressible before) and PR description + tests declare private↔internal free, and the
  maintainer-panel deployment-risk reviewer judged this a reasonable, safe narrowing — but it's
  an exposure-widening transition the flag family exists to guard, so it deserves an explicit
  maintainer sign-off rather than silent acceptance.
- excerpt: |
    if delta.Visibility != nil && *delta.Visibility == string(org.RepoVisibilityPublic) && !opt.allowRepoPublish {

### [question] pkg/github.RepoRequest API break for downstream vendors
- where: `pkg/github/types.go:382-390` (RepoRequest)
- concern: `Private *bool` removed, replaced with `Visibility *string`. In-tree only peribolos
  is affected, but Prow is widely vendored — any downstream tool constructing
  `RepoCreateRequest`/`RepoUpdateRequest` directly gets a compile break on next update. Flagged
  by the deployment-risk maintainer-panel reviewer; worth a release-note callout, not a code
  change.
- excerpt: |
    Visibility               *string `json:"visibility,omitempty"`

### [nit] both-set config error and deprecation warning lack repo/org identification
- where: `pkg/config/org/org.go:138-140` (UnmarshalJSON conflict check), `pkg/config/org/org.go:147-148` (deprecation warning)
- concern: "may not specify both 'private' and 'visibility'" aborts the whole config parse with
  no pointer to the offending repo entry (the unmarshaler doesn't know the map key); the
  deprecation warning likewise names neither org nor repo. In a config with hundreds of repos,
  this hurts debuggability. Flagged by both code-quality and deployment-risk maintainer-panel
  reviewers.
- excerpt: |
    if hasPrivate && hasVisibility {
        return fmt.Errorf("repo config may not specify both 'private' and 'visibility'")
    }

### [nit] visibility delta hand-rolls the setString helper defined in the same function
- where: `cmd/peribolos/main.go:1105-1111`; conversion also duplicated in `newRepoCreateRequest` (`main.go:1033-1037`) and `dumpOrgConfig` (`main.go:371-377`)
- concern: `setString` (main.go:1093) already does nil-check + not-equal + return-pointer logic.
  Three ad-hoc `*RepoVisibility → *string` conversion blocks instead of one shared helper.
  Flagged by both code-quality and maintainability maintainer-panel reviewers.
- excerpt: |
    setString := func(current string, want *string) *string {
        if want != nil && *want != current {
            return want
        }
        return nil
    }

### [nit] UnmarshalJSON double-parses payload; dead null assignment
- where: `pkg/config/org/org.go:116-158`
- concern: Second unmarshal into `map[string]json.RawMessage` plus manual `== "null"` string
  checks exist only to detect presence of two keys; a probe struct
  `struct{ Private, Visibility *json.RawMessage }` does the same in one small pass.
  `alias.Visibility = nil` in the visibility-null branch is dead: encoding/json sets a pointer
  field to nil for JSON null and never invokes UnmarshalText.
- excerpt: |
    rawVisibility, hasVisibility := raw["visibility"]
    if hasVisibility && string(rawVisibility) == "null" {
        hasVisibility = false
        alias.Visibility = nil
    }

### [nit] no sunset marker on the private-compat shim
- where: `pkg/config/org/org.go:110-158`
- concern: `UnmarshalJSON`'s private-translation block and the `privateFieldDeprecationWarningLast`
  package global have no TODO or tracking-issue reference; deprecations without a marker tend to
  become permanent. The alias-type pattern means it deletes cleanly whenever removed. Flagged by
  the maintainability maintainer-panel reviewer.
- excerpt: |
    var privateFieldDeprecationWarningLast time.Time

### [nit] github.Repo carries unsynced dual state (Private bool / Visibility string)
- where: `pkg/github/types.go:332-336`; ToRepo at `pkg/github/types.go:427`
- concern: `RepoRequest.ToRepo()` sets only `Visibility`, leaving `Private` at zero value for a
  private/internal request. Verified REFUTED as currently observable — `configureRepos` in
  cmd/peribolos only reads `.Visibility` from CreateRepo/UpdateRepo results, and
  branchprotector reads `Private` only from live API GetRepo responses (always populated), so no
  code path observes the divergence today. Latent inconsistency only; a one-line comment
  relating the two fields would prevent future confusion (flagged by code-quality and
  maintainability maintainer-panel reviewers).
- excerpt: |
    setString(&repo.Visibility, r.Visibility)

### [nit] visibility enum lives in pkg/config/org; github wire type is untyped *string
- where: `pkg/github/types.go:385`; symptom at `cmd/peribolos/main.go:1145`
- concern: `RepoRequest.Visibility *string` drops type safety at the API boundary, and
  `sanitizeRepoDelta` compares a github-layer value against
  `string(org.RepoVisibilityPublic)`, coupling that layer to the org config enum. Defining
  `RepoVisibility` in `pkg/github` (which `pkg/config/org` already imports) would fix both.

### [nit] bespoke pointer helpers duplicate ptr.To
- where: `cmd/peribolos/main_test.go:37`, `pkg/config/org/org_test.go:219`
- concern: Repo depends on `k8s.io/utils/ptr` and uses `ptr.To` widely; `visibilityPtr` and
  `strPtr` are one-off reimplementations.

## Checked

- Config loading goes through sigs.k8s.io/yaml (YAML→JSON), so custom `Repo.UnmarshalJSON` IS
  honored; no gopkg.in/yaml bypass anywhere; pkg/config/org imported only by cmd/peribolos.
- Marshal round-trip for dump correct: RepoVisibility is string kind; PruneRepoDefaults prunes
  only `public`, so `private`/`internal` survive dumps.
- No stale references to removed `org.Repo.Private` / `RepoRequest.Private`; branchprotector
  reads `github.Repo.Private` from live API responses, still populated.
- `RepoRequest.Defined()` updated to include Visibility.
- No extra GitHub mutations on the happy path: update request omits visibility when equal,
  matching old setBool omit-when-equal behavior.
- Both-set (`private` + `visibility`) hard error is stated PR intent and tested.
- ThrottledWarnf once-per-5-min adequate for a single-run CLI (one warning per run).
- No repo CLAUDE.md conventions apply; peribolos site docs don't document `private`/`visibility`
  fields, so no stale docs.
- UnmarshalJSON test coverage thorough (null handling, both-set, typo rejection, migration paths).
- sanitizeRepoDelta policy change (public-only gating, free private↔internal) is behavior-tested
  including internal→public gating.
- No new flags, RBAC, scopes, or endpoints; visibility rides existing repo GET/PATCH/POST calls.
- Rollback to older peribolos after adopting `visibility:` configs (esp. `internal`) doesn't
  fail — sigs.k8s.io/yaml is non-strict, field is silently ignored — but stops reconciling
  visibility with zero error signal (deployment-risk maintainer-panel finding).

## Open questions

- Should exposure-widening private→internal require `--allow-repo-publish` (or a dedicated
  flag)? PR declares it free — confirm deliberate given the flag family's purpose.
- Does the minimum GHES version prow supports always populate `visibility` on the repos API and
  accept it on create/edit? Old create path sent `private: true`, which every API version honors.
- Is the `pkg/github.RepoRequest` API break (Private → Visibility) called out in release notes
  for downstream vendors of the Go module?
