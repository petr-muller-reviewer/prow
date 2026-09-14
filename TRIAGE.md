---
issue: kubernetes-sigs/prow#939
title: "ReloadingCensorer does not censor fields extracted from JSON/INI secrets"
state: open
labels: 
main_sha: d91e0f84e0f2050b41390b28b69a05f5284ba4a2
triaged_at: 2026-09-14T22:47:55Z
verdict: needs-discussion
---

## Verdict

Keep open as `needs-discussion`. The disclosure gap is reproducible and affects Prow-owned shared code, but maintainers need to define the supported structured-format/value-extraction policy before accepting an implementation. Automatically making JSON and INI special raises the same question for YAML, TOML, XML, and other structured secrets.

## What the issue reports

- A complete JSON or INI secret file is censored when emitted verbatim.
- A job can extract and log one child value, which is not currently registered for replacement.
- The reporter supplied dummy AWS INI and JSON examples plus an external reproduction.
- The report correctly notes that sidecar-only work would not protect in-process logs.

## Findings

### [reproducibility] Extracted structured values are not registered
- detail: `Refresh` registers the payload, its trimmed representation, and base64 encodings. A child value from a valid JSON/INI document is none of those strings, so it remains visible when logged alone.
- evidence: reporter reproduction; `pkg/secretutil/censor.go:101-126`.

### [cause] Shared raw-payload registration has no structure awareness
- where: `pkg/secretutil/censor.go:101-131`
- excerpt: |
    for _, secret := range secrets {
        toEncode := []string{secret}
        ...
        addReplacement(secret)
        for _, item := range toEncode {
            encoded := base64.StdEncoding.EncodeToString([]byte(item))
            addReplacement(encoded)
        }
    }
    c.Replacer = bytereplacer.New(replacements...)
- relevance: no JSON, INI, or other child-value extraction occurs before replacements are built.

### [related-code] Process logging shares the affected censorer
- where: `pkg/config/secret/agent.go:155-172`
- excerpt: |
    for _, value := range a.secretsMap {
        secrets = append(secrets, value.getRaw())
    }
    ...
    a.ReloadingCensorer.RefreshBytes(secrets...)
- relevance: a sidecar-only change leaves process formatter/output censoring with the same raw-blob limitation.

### [related-code] Sidecar has narrow, duplicate structured extraction
- where: `pkg/sidecar/censor.go:507-545,567-619`
- excerpt: |
    secrets = append(secrets, raw)
    if info.Name() == ".dockercfg" { parser = loadDockercfgAuths }
    if info.Name() == ".dockerconfigjson" { parser = loadDockerconfigJsonAuths }
    if slices.Contains(iniFilenames, info.Name()) { parser = loadIniData }
    extra, parseErr := parser(raw)
- relevance: Docker auth fields and explicitly configured INI filenames receive special treatment; generic JSON does not. `loadIniData` already recursively collects INI key values.

### [cause] Format scope is an unresolved policy boundary
- detail: Extracting all JSON/INI values centrally fixes the supplied cases but creates an arbitrary boundary and can redact harmless configuration values. YAML, TOML, XML, and other structured formats have the same child-value property.
- evidence: `pkg/sidecar/options.go:125-132` makes current INI extraction an explicit filename opt-in; generic JSON fixture `pkg/sidecar/testdata/secrets/a/service-account.json` is not individually extracted.

### [related-issue] No duplicate or active fix found
- detail: GitHub searches for `censor secret JSON INI` and `secret censor` found no related issue or open PR beyond #939.

## Checked

- Checked upstream-main worktree revision `d91e0f84e0f2050b41390b28b69a05f5284ba4a2`.
- Confirmed `pkg/secretutil/censor_test.go:25-218` covers raw, trimmed, base64, boolean, and minimum-length behavior but no generic structured payload.
- Confirmed `pkg/sidecar/censor_test.go:151-166,309-367` covers opt-in INI and Docker extraction only.
- Confirmed the boolean exception and `minimum_secret_length` are intentional shared-censorer protections.
- Confirmed `gopkg.in/ini.v1` is already a direct dependency and `encoding/json` is used by sidecar.

## Next steps

- Decide the product contract: whole-blob-only, a bounded built-in format set, or explicit opt-in format/value extraction.
- If extraction is chosen, apply it at shared secret registration so sidecar and process logging have identical coverage; retain raw registration on parse failures.
- Define whether all string leaves or only explicit sensitive paths are registered; do not register JSON keys or numeric scalar values by default.
- Add regressions for nested JSON/arrays, INI sections, invalid documents, short/boolean values, base64, and both shared-censorer callers.
- Apply `kind/bug`, `area/pod-utilities`, `area/podutils/sidecar`, and `sig/security`; defer `help wanted` until policy is settled.

## Open questions

- Should the contract state that all values of a supported structured secret are sensitive, or require explicit value/path selection?
- Which formats are in scope, and why should unsupported formats be excluded?
- Should existing `IniFilenames` remain as compatibility configuration if extraction becomes centrally configurable?
