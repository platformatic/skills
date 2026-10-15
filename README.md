# Platformatic Watt Agent Skill

An [Agent Skill](https://agentskills.io/) for scaffolding, integrating, migrating, and deploying [Platformatic Watt 4](https://docs.platformatic.dev/docs/Overview) applications. Works with any skills-compatible agent: Claude Code, Cursor, GitHub Copilot, Gemini CLI, Junie, and [more](https://skills.sh/).

Also available as a Claude Code plugin with slash commands (`/watt`, `/kafka`, `/workflow`).

## Features

- **Framework Detection**: Detects Next.js, Nuxt, Nitro, React Router, TanStack Start, Vite, Express, Fastify, Koa, Remix, Astro, and NestJS. PHP, WordPress, and Laravel use an external capability whose Watt 4 compatibility must be checked
- **Configuration and Scaffolding**: Writes `watt.config.*`, application code, workspaces, gateways, services, and databases directly, without removed Watt generators/import commands
- **Watt 3 Migration**: Reviews configuration, listeners, environment semantics, tracing, globals, and shutdown before an authorized upgrade
- **Deployment Automation**: Generate Docker, Kubernetes, and cloud deployment configs
- **Performance Optimization**: Multi-worker SSR, distributed caching, Kubernetes tuning, CPU profiling with `wattpm pprof`, flamegraph generation with `@platformatic/flame`, and heap snapshots
- **Kafka Integration**: Event-driven microservices with kafka-hooks webhooks and migrations from KafkaJS or node-rdkafka (separate `kafka` skill)
- **Scheduled Jobs**: Cron-based task scheduling with retry support
- **CMS Integration**: Headless CMS webhooks and cache invalidation (Contentful, Sanity, Strapi)
- **Observability**: Logging (Pino), tracing (OpenTelemetry), metrics (Prometheus)

## Installation

### Any Skills-Compatible Agent

```bash
npx skills add platformatic/skills
```

Or clone the repository and point your agent at the `skills/` directory:

```bash
git clone https://github.com/platformatic/skills.git
```

### Claude Code

```bash
# As a plugin (enables slash commands)
claude --plugin-dir ./watt-skill
```

## Skills

This repository contains three skills, each a standalone `SKILL.md` following the [Agent Skills specification](https://agentskills.io/specification):

### `watt` — Platformatic Watt Integration

Detects your framework, writes Watt 4 configuration and application files, proposes compatible
dependencies, and creates deployment configs. Installation, migrations, startup, profiling,
and deployment require authorization; status/analyzer inspection does not install packages.

**Capabilities**: init/integration, create/scaffold, services/gateways/databases, Watt 3→4 migration,
deploy (Docker/Kubernetes/cloud), enterprise multi-application setup, wattpm CLI guidance,
scheduled tasks, CMS integration, observability, and CPU/heap profiling.

### `kafka` — Kafka Event-Driven Microservices

Sets up event-driven architectures with @platformatic/kafka and kafka-hooks.

**Capabilities**: producer/consumer setup, kafka-hooks webhooks, request/response patterns, consumer lag monitoring, OpenTelemetry tracing, KafkaJS migration, node-rdkafka migration.

### `workflow` — Vercel Workflow SDK with `@platformatic/world`

Helps build applications that use the Vercel Workflow SDK against a self-hosted Platformatic Workflow Service via `@platformatic/world`.

**Capabilities**: install + env-var wiring, Next.js `instrumentation.ts`, generic Node `world.start()`, optional `@platformatic/workflow-fastify` plugin, authoring workflows and steps (`'use workflow'` / `'use step'`, retries, hooks, sleeps, streams), triggering runs, standalone build, troubleshooting.

## Usage

Any skills-compatible agent will match these skills based on the `description` field in each `SKILL.md` and follow the workflow instructions. Just ask in natural language:

- *"Add Watt to this Next.js project"*
- *"Generate a Dockerfile for my Watt app"*
- *"Deploy this to Kubernetes"*
- *"Set up Kafka hooks for this service"*
- *"Migrate from KafkaJS to @platformatic/kafka"*
- *"Migrate from node-rdkafka to @platformatic/kafka"*
- *"Create a multi-service enterprise setup"*
- *"Capture a CPU profile for my Watt app with pprof"*
- *"Generate a flamegraph with @platformatic/flame"*
- *"Take a heap snapshot for my Watt app"*
- *"Add the Vercel Workflow SDK to my Next.js app with @platformatic/world"*
- *"Wire @platformatic/workflow-fastify into my Fastify server"*
- *"How do I author a workflow with 'use workflow' and 'use step'?"*

### Claude Code Slash Commands

When loaded as a Claude Code plugin, the skills also expose slash commands with argument-based routing. These are Claude Code-specific and not part of the Agent Skills standard:

```
/watt                     # auto-detect and integrate
/watt init nextjs         # integrate with framework hint
/watt create              # write a new project/application directly
/watt service             # service, gateway, or database guidance
/watt deploy docker       # generate Docker config
/watt deploy k8s          # generate Kubernetes manifests
/watt enterprise          # multi-service setup
/watt migrate             # review and migrate a Watt 3 project to Watt 4
/watt poc                 # onboard a non-Watt application for a POC
/watt cli                 # wattpm CLI reference
/watt status              # check Watt configuration health
/watt-status              # read-only status command (Claude Code)

/kafka                    # Kafka overview
/kafka hooks              # kafka-hooks webhooks
/kafka producer           # producer setup
/kafka consumer           # consumer setup
/kafka monitoring         # consumer lag monitoring
/kafka migrate kafkajs    # KafkaJS migration guide
/kafka migrate node-rdkafka  # node-rdkafka migration guide

/workflow                 # auto-detect and wire @platformatic/world
/workflow next            # Next.js instrumentation.ts setup
/workflow node            # generic Node setup with explicit world.start()
/workflow fastify         # set up @platformatic/workflow-fastify
/workflow author          # how to write 'use workflow' / 'use step' files
/workflow trigger         # how to call start() / resumeHook()
/workflow build           # workflow build --target standalone walkthrough
/workflow status          # check Workflow SDK configuration health
```

## Supported Frameworks

| Framework | Package | Detection |
|-----------|---------|-----------|
| Next.js | `@platformatic/next` | `next.config.{js,ts,mjs}` |
| Nuxt | `@platformatic/nuxt` | `nuxt.config.{ts,js,mjs}` |
| Nitro | `@platformatic/nitro` | Standalone Nitro / Nitro Vite plugin, after specific frameworks |
| React Router | `@platformatic/react-router` | `react-router.config.{ts,js}` |
| TanStack Start | `@platformatic/tanstack` | `@tanstack/react-start` in dependencies |
| Vite | `@platformatic/vite` | `vite.config.{js,ts,mjs}` (fallback for Vite apps) |
| Remix | `@platformatic/remix` | `remix.config.js` |
| Astro | `@platformatic/astro` | `astro.config.{mjs,ts}` |
| Express | `@platformatic/node` | `express` in dependencies |
| Fastify | `@platformatic/node` | `fastify` in dependencies |
| Koa | `@platformatic/node` | `koa` in dependencies |
| NestJS | `@platformatic/nest` | `nest-cli.json` or `@nestjs/core` |
| WordPress | External `@platformatic/php` (compatibility check) | `wp-config.php` |
| Laravel | External `@platformatic/php` (compatibility check) | `artisan` + `composer.json` |
| PHP | External `@platformatic/php` (compatibility check) | `composer.json` + `public/index.php` |

Several frameworks build on Vite (React Router, TanStack Start, Remix, Astro) and ship a `vite.config.*`. Detection matches the specific framework first and falls back to `@platformatic/vite` only when no more specific signal is present.

## Requirements

- **Watt 4.0.0 Node.js requirement**: v24.20.0 or higher, including CI and production images
- Compatible Watt 4 runtime/capability packages; preserve the existing package manager and lockfile
- Explicit approval before migrating an existing Watt 3 project; external PHP/addon support is not assumed

## Configuration Reference

### Executable configuration

For a new ESM package (`"type": "module"`), a single Next.js app can use `watt.config.js`:

```js
import { createNextConfig } from '@platformatic/next'

export default createNextConfig({
  server: {
    hostname: process.env.HOST || '127.0.0.1',
    port: Number(process.env.PORT || 3000)
  }
})
```

Recognized names are `watt.config.ts`, `.mts`, `.cts`, `.js`, `.mjs`, and `.cjs`, with
one per directory. `.ts`/`.js` follow the package module type; `.mts`/`.mjs` force ESM
and `.cts`/`.cjs` force CommonJS. Native TypeScript stripping accepts erasable syntax;
it does not replace framework builds or type checking.

A multi-application root uses `createWattConfig` and `applications`/`autoload`; a single
app with runtime options can use `application: { config, workers }`. Keep workers and
dependencies on application entries, and listener settings in the capability configuration.
There is no runtime `entrypoint` or root `server`; Watt does not rewrite ports or inject `PORT`.
Use helpers exported by installed packages or plain objects with `module` and their actual
schema shape. Preserve supported capability runtime wrapping and dynamic configuration source.

See [scaffolding](skills/watt/references/scaffolding.md),
[framework integration](skills/watt/references/frameworks.md), and
[Watt 3 migration](skills/watt/references/migration-v4.md) for detailed contracts.

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `PORT` | Server port | `3000` |
| `HOST` | Bind address explicitly read in this example; use `0.0.0.0` for public containers | `127.0.0.1` |
| `LOG_LEVEL` | Log level when explicitly read by runtime/application configuration | `info` |

These are example names, not automatic injected settings. Preserve existing env names
when integrating. Document names in `.env.example`, never overwrite `.env` or expose secrets.

## Project Structure

```
watt-skill/
├── skills/
│   ├── watt/
│   │   ├── SKILL.md             # Watt integration skill
│   │   └── references/
│   │       ├── frameworks/      # Framework-specific configs
│   │       ├── deployment/      # Deployment guides
│   │       ├── scaffolding.md   # Direct project/application scaffolding
│   │       ├── frameworks.md    # Detection and framework-reference index
│   │       ├── services.md      # Services, gateways, databases, and tests
│   │       ├── integration.md   # Local/remote application integration
│   │       ├── migration-v4.md  # Watt 3 to Watt 4 migration
│   │       ├── operations.md    # Operations and detailed-reference index
│   │       └── troubleshooting.md
│   ├── kafka/
│   │   ├── SKILL.md             # Kafka integration skill
│   │   └── references/
│   │       ├── kafka.md         # Kafka reference docs
│   │       ├── kafkajs-migration.md        # KafkaJS migration guide
│   │       └── node-rdkafka-migration.md   # node-rdkafka migration guide
│   └── workflow/
│       ├── SKILL.md             # Vercel Workflow SDK + @platformatic/world skill
│       └── references/
│           ├── world.md         # @platformatic/world adapter reference
│           ├── authoring.md     # 'use workflow' / 'use step' authoring guide
│           ├── fastify.md       # @platformatic/workflow-fastify plugin
│           └── troubleshooting.md          # common workflow pitfalls
├── agents/
│   └── watt-analyzer.md         # Project analysis sub-agent
├── commands/
│   └── watt-status.md           # Status check (Claude Code)
├── .claude-plugin/
│   └── plugin.json              # Claude Code plugin manifest
├── README.md
└── LICENSE
```

The `skills/` directory follows the [Agent Skills specification](https://agentskills.io/specification). The `agents/`, `commands/`, and `.claude-plugin/` directories are Claude Code-specific extensions.

## Development

### Testing Locally

```bash
# With Claude Code
claude --plugin-dir /path/to/watt-skill

# Test skill invocation
/watt
/watt-status
```

For other agents, point your agent's skill configuration at the `skills/` directory.

### Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test with your agent of choice
5. Submit a pull request

## Troubleshooting

See [troubleshooting guide](skills/watt/references/troubleshooting.md) for common issues.

### Quick Fixes

**Node.js version too old:** Use the project's existing version manager to select
Node.js 24.20.0+ and update CI/container versions consistently.

**wattpm not found:**
```bash
npm install wattpm
```

**Port already in use:**
```bash
PORT=3001 npm run dev
```

## Resources

- [Platformatic Watt Documentation](https://docs.platformatic.dev/docs/Overview)
- [Watt 4 Migration Guide](https://docs.platformatic.dev/docs/guides/migrate-v4)
- [Agent Skills Standard](https://agentskills.io/) — open format specification
- [Skills.sh Directory](https://skills.sh/) — discover and install agent skills
- [Claude Code Skills Guide](https://code.claude.com/docs/en/skills) — Claude Code-specific docs

## License

Apache-2.0 - See [LICENSE](LICENSE) for details.

## Support

- **Issues**: [GitHub Issues](https://github.com/platformatic/skills/issues)
- **Discord**: [Platformatic Community](https://discord.gg/platformatic)
