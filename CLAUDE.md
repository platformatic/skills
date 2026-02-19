# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Claude Code plugin** (not a Node.js application). There is no package.json, no build system, no source code, and no tests. The entire project is structured Markdown files that Claude Code reads and executes as instructions, workflows, and reference documentation.

The plugin gives Claude Code the knowledge to help users integrate [Platformatic Watt](https://docs.platformatic.dev/docs/Overview) into Node.js and PHP projects — framework detection, `watt.json` generation, dependency installation, deployment configs, and more.

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
  - `frameworks/` — Per-framework watt.json configs (nextjs, express, fastify, koa, remix, astro, nestjs, php)
  - `deployment/` — Docker, Kubernetes, cloud deployment guides
  - Topic files: enterprise, scheduler, cms-integration, observability, performance, poc-checklist, troubleshooting, wattpm-cli

### Skills (`skills/kafka/`)
- **`SKILL.md`** — Kafka integration skill. Routes kafka-related commands (hooks, producer, consumer, monitoring, tracing) to workflows that reference the kafka knowledge base.
- **`references/kafka.md`** — Full Kafka reference: @platformatic/kafka client, kafka-hooks webhooks, consumer lag monitoring, OpenTelemetry instrumentation, Docker Compose setup.
- **`references/migration.md`** — KafkaJS to @platformatic/kafka migration guide: API mapping, code examples, migration checklist.

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
- **watt.json schema URL pattern**: `https://schemas.platformatic.dev/@platformatic/{package}/3.0.0.json` (packages: `next`, `node`, `remix`, `astro`, `php`)
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
