---
issue: kubernetes-sigs/prow#5
title: "Implement doc checkers"
state: open
labels: lifecycle/frozen
main_sha: 0d9737ca26be70c30ad4d017f38965d0f8ef0943
triaged_at: 2026-10-28T01:31:26Z
verdict: needs-discussion
legitimacy: LEGITIMATE
effort: 2
recommended_labels: [area/documentation, kind/feature, priority/important-longterm]
advice:
  advised_at: 2026-10-28T01:38:49Z
  based_on_triaged_at: 2026-10-28T01:31:26Z
---

# Triage

## Verdict

`needs-discussion` — a legitimate tracking issue whose remaining scope needs a maintainer decision.

The link-checking half is done: #952 made the checker actually check links and #954 repaired the dead external links it found. What remains is (1) a periodic external-link job, (2) an internal-link gate for site PRs, and (3) "other checkers" that are defined only in a Google Doc that requires login. The issue is actionable only after the maintainer decides whether #5 stays an umbrella or is split into focused issues.

## What the issue reports

- Tracking issue for automated quality checks on docs.prow.k8s.io, covering both existing and newly proposed documentation.
- Original proposal: adopt the `kubernetes/website` link-checking approach; run external link checking periodically (2x/day proposed) rather than as a presubmit.
- Status update from @dklima (2026-10-23): #952 fixed the link checker and #954 repaired the dead external links.
- Still open per that comment: the periodic external-link job and the other checkers from the design doc.
- Explicit question to maintainers: keep this as the tracking issue for both, or close it and file a separate periodic-job issue.

## Findings

### [cause] External link failures cannot fail the check
- detail: `site/.htmltest.yml` sets `IgnoreExternalBrokenLinks: true`, so htmltest reports broken external links as warnings and exits successfully. htmltest v0.17.0 has no CLI flag to override this; it is config-only. Scheduling the current target would produce a green job that checks nothing.
- evidence: `site/.htmltest.yml:39`; `hack/make-rules/verify/links.sh:53-54`

### [cause] Internal link checking has no CI owner
- detail: `verify-links` is absent from the `make verify` driver, and the `pull-prow-verify-lint` image has neither hugo nor node. Separately, every build/test presubmit skips site-only PRs, so no Prow job validates a change that only touches `site/`.
- evidence: `hack/make-rules/verify/all.sh:29-68`; `kubernetes/test-infra/config/jobs/kubernetes-sigs/prow/prow-presubmits.yaml` (`skip_if_only_changed: '^site/'`)

### [cause] "Other checkers" have no public definition
- detail: The referenced design doc requires a Google login. The only non-link checker named in the public trail is docs freshness. The issue cannot be implemented beyond link checking without maintainer input.
- evidence: https://docs.google.com/document/d/17_gyhQipmK_roYqUtwx-hTwi1tn2sd0aG6vdkBn6fKM ; kubernetes/test-infra#24821 (comment 1057524854 names "broken links, freshness, etc")

### [related-code] Link-check pipeline
- where: `hack/make-rules/verify/links.sh:41-54`
- excerpt: |
    cd site

    # The site build transforms SCSS through PostCSS, which hugo calls via npx.
    echo "Installing site dependencies."
    npm ci

    echo "Building the site."
    hugo --quiet --cleanDestinationDir

    echo "Checking links..."
    make check-broken-links HTMLTEST="${HTMLTEST}"

    echo 'PASS: No broken internal links detected'
    echo 'NOTE: broken external links are reported as warnings above, not failures.'

### [related-code] Site make targets
- where: `site/Makefile:31-38`
- excerpt: |
    check-broken-links:
    	$(HTMLTEST) -c .htmltest.yml

    check-broken-links-internal:
    	$(HTMLTEST) -s -c .htmltest.yml

### [related-code] htmltest configuration
- where: `site/.htmltest.yml:35-40`
- excerpt: |
    IgnoreDirectoryMissingTrailingSlash: true
    # Certificate problems are real breakage. Suppress single URLs through
    # IgnoreURLs above if one ever needs an exception.
    IgnoreSSLVerify: false
    IgnoreExternalBrokenLinks: true
    IgnoreAltMissing: true

### [related-code] No link job in CI
- where: `kubernetes/test-infra/config/jobs/kubernetes-sigs/prow/prow-periodics.yaml:1-9`
- excerpt: |
    periodics:
    - name: ci-prow-continuous-test-verify
      cluster: k8s-infra-prow-build
      decorate: true
      extra_refs:
      - org: kubernetes-sigs
        repo: prow
        base_ref: main
      interval: 1h

### [related-pr] #952 made the checker check links
- ref: kubernetes-sigs/prow#952
- relevance: Merged 2026-09-27; fixed the per-file htmltest invocation, pinned htmltest v0.17.0, added `verify-links`, and explicitly left CI jobs and external-link strictness as follow-up.

### [related-pr] #954 repaired dead external links
- ref: kubernetes-sigs/prow#954
- relevance: Merged 2026-09-27; fixed the 16 dead external links found by the repaired checker.

### [related-issue] Documentation Revamp
- ref: kubernetes/test-infra#24821
- relevance: Closed predecessor; defines the docs site goal and names broken-link and freshness checks as intended Prow jobs.

### [related-issue] Containerized site tooling
- ref: kubernetes-sigs/prow#29
- relevance: Closed; also asked for a containerized htmltest, which is relevant to the CI image problem.

### [related-issue] Design doc is login-gated
- ref: https://docs.google.com/document/d/17_gyhQipmK_roYqUtwx-hTwi1tn2sd0aG6vdkBn6fKM
- relevance: Cannot be read without a Google session, so the intended list of additional checkers is not recoverable from the issue.

## Resolved

### [related-pr] Link checker did not check links
- ref: kubernetes-sigs/prow#952
- relevance: Resolved 2026-09-27; the checker now runs htmltest once over the built `public/` tree.

### [related-pr] Sixteen dead external links in site content
- ref: kubernetes-sigs/prow#954
- relevance: Resolved 2026-09-27; the links were repaired.

## Checked

- Searched open issues for "docs checker" and "link checker": only #5 matches; no duplicate or successor issue.
- Read the issue timeline cross-references: #29, #952, #954 only.
- Confirmed `.github/workflows/` does not exist in this repo.
- Confirmed `hack/make-rules/verify/all.sh` has no `links` entry (go lints, go fmt, file perms, spelling, codegen, boilerplate, go deps).
- Read all three `kubernetes/test-infra` job files for `kubernetes-sigs/prow`: no link-checking job.
- Counted frontmatter across 94 Markdown files: `title` 94, `weight` 93, `description` 91, `date` 0, `lastmod` 0.
- Checked htmltest v0.17.0 CLI help and option table: only `-s/--skip-external`; `IgnoreExternalBrokenLinks` is config-only.
- Attempted to export the design doc as text: redirects to a Google login page.
- Confirmed Hugo has `enableGitInfo = true` (`site/config.toml:10`), so `.Lastmod` is derivable for a freshness checker.

## Next steps

- Maintainer decision: keep #5 as the umbrella for the remaining work, or close it after opening focused issues.
- Land the strict external config first: `site/.htmltest-external.yml` with `IgnoreExternalBrokenLinks: false` plus a `verify-links-external` make/verify target.
- Add a periodic external-link job (~2x/day) in `kubernetes/test-infra`, or a GitHub Actions `schedule:` workflow if maintainers prefer to keep docs tooling here.
- Add an internal-link presubmit with `run_if_changed: '^site/'`.
- Choose or publish a CI image containing hugo extended, node and Go; #952 explicitly leaves this choice to maintainers.
- Decide whether to add a `curl --compressed` checker for `docs.github.com`, `developer.github.com`, `help.github.com` and Slack, which htmltest cannot validate.
- Ask for the design-doc checker list to be made public or pasted into the issue; file one issue per accepted checker.
- Document `make verify-links` and its local prerequisites in contributor docs.

## Advice

Live state at advice time: issue open, only `lifecycle/frozen`, no assignee, no milestone; no comments or linked work since `triaged_at`; #952 and #954 remain merged. No PR exists for the remaining work.

### 1. Answer @dklima and settle the scope in-thread (recommended)

Rationale: their 2026-10-23 comment asks the exact open question, so a decision comment is the disposition-changing action. Keeping #5 as the umbrella, agreeing on the periodic job, and asking for the strict external target first is consistent with the triage findings.

```
gh issue comment 5 --repo kubernetes-sigs/prow --body '> *This was generated by AI during triage.*

@dklima @listx @petr-muller — keeping #5 open as the tracking issue for the remaining doc-checker work.

Agreed on the periodic external-link job. The prerequisite is a strict external target: `site/.htmltest-external.yml` with `IgnoreExternalBrokenLinks: false`, plus a `verify-links-external` make/verify rule. Without that, scheduling the current target produces a green job that checks nothing.

@dklima, can you open that config/target PR first? Then we can decide the job location (test-infra periodic vs. a GitHub Actions workflow) and the hugo+node+Go image.

For the other checkers: the design doc requires login. Can you paste the remaining list here so we can file focused issues?'
```

### 2. Apply the triage labels

Rationale: the issue currently carries only `lifecycle/frozen`; `area/documentation` and `kind/feature` make it discoverable, and `priority/important-longterm` fits quality infrastructure that is not urgent.

```
gh issue edit 5 --repo kubernetes-sigs/prow \
  --add-label "area/documentation" \
  --add-label "kind/feature" \
  --add-label "priority/important-longterm"
```

### 3. Split and close #5 (alternative)

Rationale: if the umbrella should not stay open, create focused issues and close #5 with cross-references. The first body targets the strict external config; the second the internal presubmit; the third the undefined checker list.

```
gh issue create --repo kubernetes-sigs/prow \
  --title "Add a periodic external link check for docs.prow.k8s.io" \
  --label area/documentation --label kind/feature \
  --body 'Track the periodic external link check from #5: add site/.htmltest-external.yml with IgnoreExternalBrokenLinks: false plus a verify-links-external target, then run it ~2x/day. See TRIAGE.md findings.'

gh issue create --repo kubernetes-sigs/prow \
  --title "Gate site PRs on internal link checks" \
  --label area/documentation --label kind/feature \
  --body 'Track the internal-link presubmit from #5: gate site/** changes on check-broken-links-internal. See TRIAGE.md findings.'

gh issue create --repo kubernetes-sigs/prow \
  --title "Define the remaining documentation checkers" \
  --label area/documentation --label kind/feature \
  --body 'Track the undefined doc checkers from #5. The design doc is login-gated; freshness is the only publicly named candidate. Enumerate the wanted checkers first.'

gh issue close 5 --repo kubernetes-sigs/prow --comment '> *This was generated by AI during triage.*

Split into #<external>, #<internal>, #<checkers>; closing this umbrella.'
```

### 4. Escalate to SIG Testing if no decision

Rationale: SIG Testing is the documented venue for Prow discussion (`CONTRIBUTING.md:25-26`), and the root `OWNERS` approvers (`cblecker`, `petr-muller`, `matthyx`, `droslean`, `krzyzacy`, `smg247`) are the people who can decide.

- Slack: https://kubernetes.slack.com/messages/sig-testing
- Mailing list: https://groups.google.com/forum/#!forum/kubernetes-sig-testing

No `gh` command carries this out; raise it in the SIG Testing channel or on the list, then record the outcome here.

### 5. Assign @dklima once they take the first PR

Rationale: they volunteered to open the periodic-job PR; assignment is appropriate only after they confirm they are taking it.

```
gh issue edit 5 --repo kubernetes-sigs/prow --add-assignee dklima
```

## Open questions

- @dklima: should #5 stay open as the tracking issue for the periodic job and other checkers, or should we close it and open a focused issue for the periodic job?
- Is the design doc still the source of truth for "other checkers"? It requires login — can the remaining list be pasted here?
- For the periodic job, do we prefer `kubernetes/test-infra` job config or a GitHub Actions workflow in this repo?
- Who can supply or publish the hugo+node+Go image, and is 2x/day still the right cadence?
- Should internal-link checking gate every PR or only `site/**` changes?
- What should a periodic failure do — TestGrid only, open an issue, or notify a channel?
- Should `lifecycle/frozen` stay once the issue is split or narrowed?
