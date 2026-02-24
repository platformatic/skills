# Platformatic Watt Agent Skill

An [Agent Skill](https://agentskills.io/) for integrating and deploying [Platformatic Watt](https://docs.platformatic.dev/docs/Overview) in any Node.js project. Works with any skills-compatible agent: Claude Code, Cursor, GitHub Copilot, Gemini CLI, Junie, and [more](https://skills.sh/).

Also available as a Claude Code plugin with slash commands (`/watt`, `/kafka`).

## Features

- **Automatic Framework Detection**: Detects Next.js, Express, Fastify, Koa, Remix, Astro, NestJS, WordPress, Laravel, and PHP
- **Configuration Generation**: Creates optimized `watt.json` for your framework
- **Deployment Automation**: Generate Docker, Kubernetes, and cloud deployment configs
- **Performance Optimization**: Multi-worker SSR, distributed caching, and Kubernetes tuning
- **Kafka Integration**: Event-driven microservices with kafka-hooks webhooks (separate `kafka` skill)
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

This repository contains two skills, each a standalone `SKILL.md` following the [Agent Skills specification](https://agentskills.io/specification):

### `watt` — Platformatic Watt Integration

Detects your framework, generates `watt.json`, installs dependencies, and creates deployment configs.

**Capabilities**: init, deploy (Docker/Kubernetes/cloud), migrate, enterprise multi-service setup, wattpm CLI guidance, scheduled jobs, CMS integration, observability.

### `kafka` — Kafka Event-Driven Microservices

Sets up event-driven architectures with @platformatic/kafka and kafka-hooks.

**Capabilities**: producer/consumer setup, kafka-hooks webhooks, request/response patterns, consumer lag monitoring, OpenTelemetry tracing, KafkaJS migration.

## Usage

Any skills-compatible agent will match these skills based on the `description` field in each `SKILL.md` and follow the workflow instructions. Just ask in natural language:

- *"Add Watt to this Next.js project"*
- *"Generate a Dockerfile for my Watt app"*
- *"Deploy this to Kubernetes"*
- *"Set up Kafka hooks for this service"*
- *"Migrate from KafkaJS to @platformatic/kafka"*
- *"Create a multi-service enterprise setup"*

### Claude Code Slash Commands

When loaded as a Claude Code plugin, the skills also expose slash commands with argument-based routing. These are Claude Code-specific and not part of the Agent Skills standard:

```
/watt                     # auto-detect and integrate
/watt init nextjs         # integrate with framework hint
/watt deploy docker       # generate Docker config
/watt deploy k8s          # generate Kubernetes manifests
/watt enterprise          # multi-service setup
/watt migrate             # migrate existing app
/watt cli                 # wattpm CLI reference
/watt status              # check Watt configuration health

/kafka                    # Kafka overview
/kafka hooks              # kafka-hooks webhooks
/kafka producer           # producer setup
/kafka consumer           # consumer setup
/kafka monitoring         # consumer lag monitoring
/kafka migrate            # KafkaJS migration guide
```

## Supported Frameworks

| Framework | Package | Detection |
|-----------|---------|-----------|
| Next.js | `@platformatic/next` | `next.config.{js,ts,mjs}` |
| Remix | `@platformatic/remix` | `remix.config.js` |
| Astro | `@platformatic/astro` | `astro.config.{mjs,ts}` |
| Express | `@platformatic/node` | `express` in dependencies |
| Fastify | `@platformatic/node` | `fastify` in dependencies |
| Koa | `@platformatic/node` | `koa` in dependencies |
| NestJS | `@platformatic/node` | `nest-cli.json` or `@nestjs/core` |
| WordPress | `@platformatic/php` | `wp-config.php` |
| Laravel | `@platformatic/php` | `artisan` + `composer.json` |
| PHP | `@platformatic/php` | `composer.json` + `public/index.php` |

## Requirements

- **Node.js**: v22.19.0 or higher

## Configuration Reference

### watt.json Structure

```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/{package}/3.0.0.json",
  "application": {
    "commands": {
      "development": "your-dev-command",
      "build": "your-build-command",
      "production": "your-start-command"
    }
  },
  "runtime": {
    "logger": {
      "level": "{PLT_SERVER_LOGGER_LEVEL}"
    },
    "server": {
      "hostname": "{PLT_SERVER_HOSTNAME}",
      "port": "{PORT}"
    }
  }
}
```

### Runtime Placement Rule

- **Single-config application schema** (`@platformatic/node`, `@platformatic/next`, `@platformatic/remix`, `@platformatic/astro`, `@platformatic/php`): use a `runtime` block for runtime settings (`logger`, `server`, `workers`, telemetry, etc.).
- **Root multi-app orchestrator schema** (`watt` / `@platformatic/runtime`): keep runtime settings in the root-level `runtime` block.

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `PORT` | Server port | `3000` |
| `PLT_SERVER_HOSTNAME` | Bind address | `0.0.0.0` |
| `PLT_SERVER_LOGGER_LEVEL` | Log level | `info` |

## Project Structure

```
watt-skill/
├── skills/
│   ├── watt/
│   │   ├── SKILL.md             # Watt integration skill
│   │   └── references/
│   │       ├── frameworks/      # Framework-specific configs
│   │       ├── deployment/      # Deployment guides
│   │       └── troubleshooting.md
│   └── kafka/
│       ├── SKILL.md             # Kafka integration skill
│       └── references/
│           ├── kafka.md         # Kafka reference docs
│           └── migration.md     # KafkaJS migration guide
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

**Node.js version too old:**
```bash
nvm install 22 && nvm use 22
```

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
- [Watt 3 Announcement](https://blog.platformatic.dev/introducing-watt-3)
- [Agent Skills Standard](https://agentskills.io/) — open format specification
- [Skills.sh Directory](https://skills.sh/) — discover and install agent skills
- [Claude Code Skills Guide](https://code.claude.com/docs/en/skills) — Claude Code-specific docs

## License

Apache-2.0 - See [LICENSE](LICENSE) for details.

## Support

- **Issues**: [GitHub Issues](https://github.com/platformatic/skills/issues)
- **Discord**: [Platformatic Community](https://discord.gg/platformatic)
