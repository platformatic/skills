---
name: watt
description: |
  Integrate, scaffold, configure, migrate, diagnose, profile, and deploy Platformatic Watt applications.
  Use for adding Watt, watt.json or watt.config.* configuration, multi-application workspaces,
  services, databases, gateways, wattpm CLI commands, observability, scheduled jobs, CMS integration,
  performance profiling, Docker, Kubernetes, and cloud deployment.
  Covers Next.js, Nuxt, Nitro, React Router, TanStack Start, Vite, Express, Fastify, Koa,
  Remix, Astro, NestJS, PHP, WordPress, and Laravel according to the installed release's capabilities.
argument-hint: "[init|create|migrate|deploy|status] [framework-hint]"
allowed-tools: Read, Glob, Grep, Bash, Write, Edit
compatibility: Requires Node.js and a project-local Watt 3.74.0 or newer installation with the skill command, plus an agent capable of running local commands.
---

# Platformatic Watt

Use the skill bundled with the project's installed `wattpm`. It contains the workflows
and references for that exact release. Do not choose configuration formats, APIs, Node.js
requirements, or scaffolding commands from memory or assume that the project uses the
latest Watt major.

## Locate the project's installation

1. Read the project's instructions, package manifests, lockfile, and workspace definitions.
   Identify the application or workspace package relevant to the user's request.
2. Locate its installed `wattpm` package and read its `package.json` version, Node.js
   requirements, and `bin` entry. Account for workspace hoisting and package-manager
   layouts; do not assume that the repository root owns the relevant installation.
   The `skill` command is supported from Watt 3.74.0 onward. Compare the installed version
   using semantic version ordering, not string ordering. If it is below 3.74.0, abort the
   request and explain that this skill requires Watt 3.74.0 or newer.
3. Resolve the CLI entrypoint from that installed package's `bin` field, or use its
   verified local binary. Run it from the target project's directory and reuse the same
   installation for every skill and reference request. If multiple installations are
   relevant and the target is ambiguous, ask which project to work on.

Do not use a global `wattpm` or an unqualified `npx wattpm` invocation that could download
a different version. In the examples below, `"<local-wattpm-bin>"` is a placeholder for
the absolute CLI entrypoint discovered above, not a literal path.

## Load the release-matched instructions

```bash
node "<local-wattpm-bin>" skill
```

Read the returned Markdown and follow its workflow for the user's request. Preserve
any supplied skill arguments as task context; `skill` accepts a documentation file,
not workflow arguments such as `init` or `deploy`.

The command is `skill` (singular). Its default document is `SKILL.md`. Documentation
access is local and does not require copying files into the project or downloading docs.

## Follow references through the same CLI

Read only the references needed for the current task:

```bash
node "<local-wattpm-bin>" skill references/frameworks.md
node "<local-wattpm-bin>" skill references/frameworks/nextjs.md
```

These paths illustrate the interface; use the links actually returned by the installed
release rather than assuming that every release contains the same files.

The returned Markdown belongs to the installed Watt skill bundle, not this discovery
skill's directory or the project's working directory. Track the bundle-relative path
of each document you read:

1. Resolve each relative Markdown link against the directory of the document containing it.
2. Normalize `.` and `..`, retaining a path inside the bundle. Remove any `#fragment`
   from the file argument, then locate that section in the returned document.
3. Pass the resulting bundle-relative path to the same local `wattpm skill` command.
   Never pass an absolute filesystem path to `skill`.
4. For a link containing only `#fragment`, use the current document. Treat external URLs
   as external documentation, not CLI file arguments, and check their version applicability.

For example, `frameworks/nextjs.md` linked from `references/frameworks.md` resolves to
`references/frameworks/nextjs.md`. A link to `../deployment/docker.md` from that Next.js
guide resolves to `references/deployment/docker.md`.

If a reference is unavailable, report the installed version and missing path. Do not
silently substitute instructions from another release.

## Missing installations, older releases, and migrations

- If dependencies are declared but not installed, use the project's package manager and
  lockfile to restore them when dependency installation is authorized, then resolve the CLI again.
- For a new project without `wattpm`, establish the intended Watt release from the user's
  request and project constraints. Ask if the target is unclear. Install that release with
  the project's package manager when authorized, then load its bundled skill.
- If the installed version is below 3.74.0 or the installed CLI does not support `skill`, abort
  the request and explain that this skill requires Watt 3.74.0 or newer with the `skill` command.
  Do not upgrade the installation, substitute external documentation, or continue the
  requested workflow as a fallback.
- If the command fails for another reason, report that failure rather than treating it
  as evidence that `skill` is unsupported.
- For a major-version migration, distinguish the installed source version from the intended
  target version. Obtain a migration guide explicitly covering that transition; the current
  release's instructions alone do not establish the target release's behavior.
