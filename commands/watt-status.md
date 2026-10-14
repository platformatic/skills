---
name: watt-status
description: Read-only inspection of Platformatic Watt versions, configuration, and integration readiness
allowed-tools: Read, Bash, Glob, Grep
---

# Watt status check

Inspect the current project without modifying files, installing packages, or starting
applications. Follow the same contract as `/watt status` in the Watt skill. Do not dump
secrets, evaluate config modules without permission, or use `npx` to check installation.

## Checks

1. **Versions**: read manifests, lockfiles, and installed package metadata for `wattpm`
   and capability versions. Distinguish installed versions from manifest ranges. Report
   mixed majors and external PHP/addon compatibility gaps. Do not silently migrate Watt 3.
2. **Node.js**: with permission, run `node --version` using the configured shell. Watt 4.0.0
   requires 24.20.0+. For a legacy project, report its own major's requirement separately.
3. **Configuration inventory**: look for `watt.config.ts`, `.mts`, `.cts`, `.js`, `.mjs`,
   `.cjs` and legacy Watt/Platformatic JSON/YAML files at relevant project/app roots.
   Multiple recognized modules or mixed dialects need review. No config may be valid
   for an inferred application; do not automatically mark it missing/broken.
4. **Static review**: read source and package `type`; check ESM/CJS extension consistency,
   helpers or `module`, orchestration topology, listener placement, application IDs,
   worker settings, and explicit environment conversions. Root `server`/`entrypoint`
   and `services`/`web` aliases are not the Watt 4 contract.
5. **Dependencies and scripts**: inspect local availability and `dev`/`build`/`start`,
   preserving custom script names and framework build commands. For an authorized CLI
   version check use the existing local binary, not a command that can download it.
6. **Environment**: report required variable **names**, source/precedence assumptions,
   and env-example presence. `.env` is optional when deployment provides values. Never
   print credentials or recommend overwriting `.env` with defaults.
7. **Deployment**: inspect Node images, actual public listener, framework assets, health
   routes, workspace manifests, and production Watt/capability dependencies.

Static inspection is not schema/startup validation. Imports can execute arbitrary code;
ask before evaluating dynamic configuration, running tests/builds, injecting requests,
or launching processes. Do not report “Ready to run” solely from a version/syntax check.

## Output

```text
Watt Configuration Status
Node.js:          [observed version / not checked], [compatible / upgrade needed]
Watt:             [installed version / declared range only / unavailable]
Capabilities:     [compatible majors / mixed / external compatibility unverified]
Configuration:    [module paths / inferred / legacy / conflict]
Review:           [statically inspected / evaluation not performed]
Scripts/build:    [present / custom pipeline / missing output]
Public listener:  [owning application and port expression / unresolved]
Environment:      [required names only / missing documentation]
Deployment:       [findings / not inspected]
Checks performed: [actual checks]
Remaining checks: [schema evaluation, startup/build, endpoints, shutdown]
Next steps:       [focused recommendations requiring authorization where applicable]
```

Offer focused fixes but do not perform them as part of this read-only command. For a
new/inferred app use `/watt init`; for new files use `/watt create`; for Watt 3 use
`/watt migrate` after agreeing scope. Preserve lockfiles, dynamic exports, and user work.
