# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Repository**: `platformatic/skills` (https://github.com/platformatic/skills)

This is an **Agent Skill** following the [Agent Skills open standard](https://agentskills.io/),
not a Node.js application. Skills are Markdown instructions/reference documentation;
`scripts/prepare-watt-context.mjs` prepares immutable reconciliation snapshots. There is
no package.json or application build/test system.

The `skills/` directory is portable across any skills-compatible agent (Claude Code, Cursor, GitHub Copilot, Gemini CLI, etc.). The `agents/`, `commands/`, and `.claude-plugin/` directories are Claude Code-specific extensions that add slash commands and sub-agents.

The Watt skill covers Watt 4.0.0 direct scaffolding, executable configuration, integration,
Watt 3 migration, services/gateways/databases, and operations. External PHP compatibility
must be checked before integration. Installation, execution, migration, and deployment
require authorization; read-only status/analyzer inspection must not download packages.

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
- **`SKILL.md`** — Main contract/router for integration, scaffolding, migration, and operations.
  Workflows read only relevant references on demand, never bulk-load them.
- **`references/`** — Knowledge base loaded lazily by workflows:
  - Core guides: scaffolding, frameworks index, services, integration, migration-v4, operations
  - `frameworks/` — Per-framework Watt 4 configs (nextjs, nuxt, nitro, react-router, tanstack, vite, express, fastify, koa, remix, astro, nestjs) and external PHP compatibility
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

1. Create `skills/watt/references/frameworks/{framework}.md` with package, signals,
   verified Watt 4 helper/config, authorized installation, lifecycle, and compatibility details
2. Add combined configuration/dependency detection to `references/frameworks.md`;
   specific frameworks take precedence over shared Vite/Nitro infrastructure
3. Link the detailed guide from the framework index
4. Update the supported frameworks table in `README.md`
5. Update detection priorities in `agents/watt-analyzer.md`

### Adding a New Workflow

1. Add a routing entry to the Command Router table in `SKILL.md`
2. Keep the shared contract in `SKILL.md` and detailed steps in a focused reference
3. Link large workflows from the router and relevant reference indexes
4. Update `README.md` with the new command

## Key Watt Facts

These facts must stay accurate across all documentation files:

- **Watt 4.0.0 Node.js requirement**: v24.20.0+, including CI/production
- **Configuration**: one `watt.config.{ts,mts,cts,js,mjs,cjs}` per directory;
  module system follows extension/package type. No competing legacy configuration
- **Helpers/plain objects**: inspect installed exports/schema; plain objects identify
  `module` and use the exact installed version when stamping `$schema`
- **Environment**: explicit `process.env` expressions/conversions, not legacy placeholders
- **Internal service URLs in app code**: `http://{service-id}.plt.local`
- **Gateway**: `@platformatic/gateway`, `gateway.applications`, internal application IDs;
  do not require Composer's legacy origin syntax
- **Listeners/workers**: capability-owned `server`, no runtime `entrypoint`/root `server`;
  workers/dependencies belong to orchestration application entries, IDs need not match paths
- **Standard scripts**: `wattpm dev`, `wattpm build`, `wattpm start`
- **Install**: compatible Watt 4 `wattpm` and capability with the existing package manager,
  only with permission. No removed scaffolding/import commands
- **Utilities**: `wattpm-utils install`, `update`, `resolve`, `migrate`; no CLI `config`/`patch-config`
- **Tracing/cleanup**: `tracing`, `@platformatic/tracing`, typed globals, `registerCloseCallback()`
- **Compatibility**: preserve valid legacy framework details, but do not assume external PHP/addon support

## Git Conventions

Imperative, single-line commit messages describing the change:
- `Add scheduled jobs (cron) documentation`
- `Fix installation patterns across framework guides`
- `Incorporate learnings from watt-contentful-next project`
