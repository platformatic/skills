# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Repository**: `platformatic/skills` (https://github.com/platformatic/skills)

This is an **Agent Skill** following the [Agent Skills open standard](https://agentskills.io/) (not a Node.js application). There is no package.json, no build system, no source code, and no tests. The entire project is structured Markdown files that agents read and execute as instructions, workflows, and reference documentation.

The `skills/` directory is portable across any skills-compatible agent (Claude Code, Cursor, GitHub Copilot, Gemini CLI, etc.). The `agents/`, `commands/`, and `.claude-plugin/` directories are Claude Code-specific extensions that add slash commands and sub-agents.

The skill gives agents the knowledge to help users integrate [Platformatic Watt](https://docs.platformatic.dev/docs/Overview) into Node.js and PHP projects — framework detection, `watt.json` generation, dependency installation, deployment configs, and more.

## Testing

No automated tests. Test manually by loading the plugin and invoking skill commands:

```bash
claude --plugin-dir /path/to/watt-skill

# Then invoke:
/watt
/watt init nextjs
/watt deploy docker
/watt status
```

## Architecture

The plugin exposes three component types, each a Markdown file with YAML frontmatter:

### Skills (`skills/watt/`)
- **`SKILL.md`** — Main entrypoint. Contains a command router that maps user input (e.g. `init`, `deploy docker`) to inline workflows. Workflows read reference files on demand — they never bulk-load all references.
- **`references/`** — Knowledge base loaded lazily by workflows:
  - `frameworks/` — Per-framework watt.json configs (nextjs, nuxt, react-router, tanstack, vite, express, fastify, koa, remix, astro, nestjs, php)
  - `deployment/` — Docker, Kubernetes, cloud deployment guides
  - Topic files: enterprise, scheduler, cms-integration, observability, performance, poc-checklist, troubleshooting, wattpm-cli

### Skills (`skills/kafka/`)
- **`SKILL.md`** — Kafka integration skill. Routes kafka-related commands (hooks, producer, consumer, monitoring, tracing, migrations) to workflows that reference the kafka knowledge base.
- **`references/kafka.md`** — Full Kafka reference: @platformatic/kafka client, kafka-hooks webhooks, consumer lag monitoring, OpenTelemetry instrumentation, Docker Compose setup.
- **`references/kafkajs-migration.md`** — KafkaJS to @platformatic/kafka migration guide: API mapping, code examples, migration checklist.
- **`references/node-rdkafka-migration.md`** — node-rdkafka to @platformatic/kafka migration guide: librdkafka configuration mapping, producer/consumer API examples, stream migration, metadata/admin migration, error handling, shutdown, and verification checklist.

### Skills (`skills/workflow/`)
- **`SKILL.md`** — Vercel Workflow SDK skill. Routes commands (init, next, node, fastify, author, trigger, build, status, troubleshoot) for building apps that use the Vercel Workflow SDK with `@platformatic/world`. Scope is client-side only — it does not cover deploying the Workflow Service.
- **`references/world.md`** — `@platformatic/world` reference: install, env vars (`WORKFLOW_TARGET_WORLD`, `PLT_WORLD_SERVICE_URL`, etc.), `createWorld()` / `createPlatformaticWorld()`, Next.js `instrumentation.ts`, manual `world.start()`, triggering runs, SDK compatibility (4.2.x / 5.0.0-beta.x).
- **`references/authoring.md`** — Authoring guide for the Vercel SDK itself: `'use workflow'` / `'use step'` directives, retries, hooks (human-in-the-loop + webhook), sleeps, streams, determinism rules, file layout, build targets.
- **`references/fastify.md`** — Optional `@platformatic/workflow-fastify` plugin: install, options (`buildDir`, `register`), standalone-build extension handling, when not to use the plugin.
- **`references/troubleshooting.md`** — Common pitfalls: runs stuck in `pending`, missing env vars, missing standalone build, `CorruptedEventLogError`, cursor-missing warning, transform not applied, hook never resumes.

### Agents (`agents/`)
- **`watt-analyzer.md`** — Read-only sub-agent for framework detection. Only has `Read, Glob, Grep` tools.

### Commands (`commands/`)
- **`watt-status.md`** — `/watt-status` slash command for health checking Watt setup.

## Editing Guidelines

### Frontmatter Format

Every skill/agent/command file requires YAML frontmatter:

```yaml
---
name: skill-name
description: |
  Description of when this skill is invoked.
allowed-tools: Read, Glob, Grep, Bash, Write, Edit
argument-hint: "[optional] [args]"
---
```

Read-only agents should only declare: `allowed-tools: Read, Glob, Grep`

### Adding a New Framework

1. Create `skills/watt/references/frameworks/{framework}.md` with: package name, detection method, watt.json config, installation steps, key considerations, common issues
2. Add detection entry to `SKILL.md` Priority 1 (config files) or Priority 2 (dependencies) tables
3. Add a Read instruction for the new reference file in `SKILL.md`'s Integration Workflow
4. Update the supported frameworks table in `README.md`
5. Update detection priorities in `agents/watt-analyzer.md`

### Adding a New Workflow

1. Add a routing entry to the Command Router table in `SKILL.md`
2. Add a new `## Workflow Name` section to `SKILL.md` with step-by-step instructions
3. For large workflows, create a reference file in `skills/watt/references/` and Read it from `SKILL.md`
4. Update `README.md` with the new command

## Key Watt Facts

These facts must stay accurate across all documentation files:

- **Node.js requirement**: v22.19.0+
- **watt.json schema URL pattern**: `https://schemas.platformatic.dev/@platformatic/{package}/3.0.0.json` (packages: `next`, `nuxt`, `react-router`, `tanstack`, `vite`, `node`, `remix`, `astro`, `php`)
- **Environment variables in watt.json**: `{VAR_NAME}` (curly braces, no dollar sign)
- **Internal service URLs in app code**: `http://{service-id}.plt.local`
- **Composer service origins**: `internal://{service-id}`
- **Worker naming**: directory name only (e.g. `"storefront"`, not `"web/storefront"`)
- **Standard scripts**: `wattpm dev`, `wattpm build`, `wattpm start`
- **Install**: `npm install wattpm` (plus framework-specific `@platformatic/*` package)

## Git Conventions

Imperative, single-line commit messages describing the change:
- `Add scheduled jobs (cron) documentation`
- `Fix installation patterns across framework guides`
- `Incorporate learnings from watt-contentful-next project`
