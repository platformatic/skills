# Platformatic Watt Skill for Claude Code

A Claude Code plugin for integrating and deploying [Platformatic Watt](https://docs.platformatic.dev/docs/Overview) in any Node.js project.

## Features

- **Automatic Framework Detection**: Detects Next.js, Express, Fastify, Koa, Remix, Astro, NestJS, WordPress, Laravel, and PHP
- **Configuration Generation**: Creates optimized `watt.json` for your framework
- **Deployment Automation**: Generate Docker, Kubernetes, and cloud deployment configs
- **Performance Optimization**: Multi-worker SSR, distributed caching, and Kubernetes tuning
- **Status Checking**: Verify your Watt setup with a single command

## Installation

### From Plugin Directory (Development)

```bash
# Clone the repository
git clone https://github.com/platformatic/watt-skill.git

# Use with Claude Code
claude --plugin-dir ./watt-skill
```

### From npm (When Published)

```bash
# Install globally
npm install -g @platformatic/watt-skill

# Or use with Claude Code
/plugin install @platformatic/watt-skill
```

## Usage

### Initialize Watt in a Project

```bash
/watt
```

This will:
1. Detect your framework (Next.js, Express, Fastify, etc.)
2. Generate appropriate `watt.json` configuration
3. Install required dependencies (`wattpm`, `@platformatic/*`)
4. Update `package.json` scripts
5. Create `.env` file with defaults

### Initialize with Framework Hint

```bash
/watt init nextjs
/watt init express
/watt init fastify
```

### Check Watt Configuration Status

```bash
/watt status
```

Outputs:
```
Watt Configuration Status
=========================
Node.js Version:  v22.19.0       [OK]
watt.json:        Found          [OK]
Configuration:    Valid JSON     [OK]
wattpm:           v3.2.0         [OK]
Scripts:          Configured     [OK]

Status: Ready to run
```

### Deploy with Docker

```bash
/watt deploy docker
```

Generates:
- `Dockerfile` (multi-stage, optimized)
- `.dockerignore`
- `docker-compose.yml` (optional)

### Deploy to Kubernetes

```bash
/watt deploy k8s
```

Generates:
- `k8s/deployment.yaml`
- `k8s/service.yaml`
- `k8s/configmap.yaml`
- `k8s/hpa.yaml` (autoscaling)

### Deploy to Cloud Platforms

```bash
/watt deploy cloud
/watt deploy fly
/watt deploy railway
```

Generates platform-specific configuration files.

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
- **Claude Code**: Latest version

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

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `PORT` | Server port | `3000` |
| `PLT_SERVER_HOSTNAME` | Bind address | `0.0.0.0` |
| `PLT_SERVER_LOGGER_LEVEL` | Log level | `info` |

## Plugin Structure

```
watt-skill/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── skills/
│   └── watt/
│       ├── SKILL.md             # Main skill
│       └── references/
│           ├── frameworks/      # Framework-specific configs
│           ├── deployment/      # Deployment guides
│           └── troubleshooting.md
├── agents/
│   └── watt-analyzer.md         # Project analysis agent
├── commands/
│   └── watt-status.md           # Status check command
├── README.md
└── LICENSE
```

## Development

### Testing Locally

```bash
# Start Claude Code with plugin
claude --plugin-dir /path/to/watt-skill

# Test skill invocation
/watt:watt
/watt:watt-status
```

### Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test with Claude Code
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
- [Claude Code Skills Guide](https://code.claude.com/docs/en/skills)

## License

Apache-2.0 - See [LICENSE](LICENSE) for details.

## Support

- **Issues**: [GitHub Issues](https://github.com/platformatic/watt-skill/issues)
- **Discord**: [Platformatic Community](https://discord.gg/platformatic)
