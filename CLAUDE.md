# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Repository**: `platformatic/skills` (https://github.com/platformatic/skills)

This is an **Agent Skill** following the [Agent Skills open standard](https://agentskills.io/) (not a Node.js application). There is no package.json, no build system, no source code, and no tests. The entire project is structured Markdown files that agents read and execute as instructions, workflows, and reference documentation.

The `skills/` directory is portable across any skills-compatible agent (Claude Code, Cursor, GitHub Copilot, Gemini CLI, etc.). The `agents/`, `commands/`, and `.claude-plugin/` directories are Claude Code-specific extensions that add slash commands and sub-agents.

The Watt skill delegates to the project's locally installed `wattpm skill` for release-matched integration, configuration, and deployment guidance.

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
- **`SKILL.md`** — Version-independent discovery entrypoint. Locates the project's installed `wattpm`, loads `wattpm skill`, and retrieves linked references through the same CLI. Version-specific workflows and references are maintained in `platformatic/platformatic` and bundled in `wattpm`, not copied here.

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

1. Add release-specific framework guidance to the canonical Watt skill in `platformatic/platformatic` on the appropriate release branch.
2. Update the supported frameworks table in `README.md` if needed.
3. Update detection priorities in `agents/watt-analyzer.md` when applicable.

### Adding a New Workflow

For Watt, maintain workflows and references in the release-matched skill in `platformatic/platformatic`. Keep this repository's Watt discovery skill independent of release-specific APIs and configuration formats. For Kafka and Workflow, maintain instructions and references in their respective skill directories. Update `README.md` when user-facing capabilities change.

## Key Watt Facts

- The installed `wattpm` release determines Node.js requirements, configuration formats, supported capabilities, and commands.
- Use the project's local installation for `wattpm skill [FILE]`; do not implicitly download another release.
- Resolve relative Markdown links against the referring document inside the bundled skill, not this repository or the working directory.
- Do not restore duplicate Watt reference files or cross-repository synchronization workflows here.

## Git Conventions

Imperative, single-line commit messages describing the change:
- `Add scheduled jobs (cron) documentation`
- `Fix installation patterns across framework guides`
- `Incorporate learnings from watt-contentful-next project`
