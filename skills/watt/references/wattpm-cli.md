# wattpm CLI Reference

The `wattpm` command is the unified CLI for Platformatic Watt. Install globally or locally:

```bash
npm install wattpm        # local (recommended)
npm install -g wattpm     # global
```

---

## Project Scaffolding

### `create` / `init` / `add`

Create a new Watt project or add applications to an existing one.

```bash
wattpm create                          # interactive scaffolding
wattpm create --module @platformatic/next  # create with specific module
wattpm init                            # alias for create
wattpm add                             # alias for create (adds to existing)
```

| Flag | Description | Default |
|------|-------------|---------|
| `-c, --config <config>` | Config file name | `watt.json` |
| `-s, --skip-dependencies` | Skip dependency installation | — |
| `-P, --package-manager <executable>` | Package manager to use | `npm` |
| `-M, --module <name>` | Module(s) to use as app generator (repeatable) | — |

---

## Build & Install

### `build`

Compile all applications in the project.

```bash
wattpm build                    # build from current directory
wattpm build ./my-project       # build from specified root
wattpm build --env .env.prod    # use custom env file
```

| Arg / Flag | Description | Default |
|------------|-------------|---------|
| `root` | Project directory | current dir |
| `-c, --config <config>` | Config file | autodetect |
| `-e, --env <path>` | Custom `.env` file path | — |

### `install`

Install dependencies for the project and all sub-applications.

```bash
wattpm install
wattpm install --production                    # production deps only
wattpm install --package-manager pnpm          # use pnpm
```

| Arg / Flag | Description | Default |
|------------|-------------|---------|
| `root` | Project directory | current dir |
| `-c, --config <config>` | Config file | autodetect |
| `-p, --production` | Production dependencies only | — |
| `-P, --package-manager <executable>` | Package manager | autodetect |

### `update`

Update all Platformatic runtime and capability packages to the latest version. Only works on packages defined with `~` or `^` syntax.

```bash
wattpm update
wattpm update --force          # force update beyond version range
```

| Arg / Flag | Description | Default |
|------------|-------------|---------|
| `root` | Project directory | current dir |
| `-c, --config <config>` | Config file | autodetect |
| `-f, --force` | Force update beyond version constraints | — |

---

## Running Applications

### `dev`

Start the application in development mode with hot reload.

```bash
wattpm dev
wattpm dev ./my-project
wattpm dev --env .env.local
```

| Arg / Flag | Description | Default |
|------------|-------------|---------|
| `root` | Project directory | current dir |
| `-c, --config <config>` | Config file | autodetect |
| `-e, --env <path>` | Custom `.env` file path | — |

### `start`

Start the application in production mode.

```bash
wattpm start
wattpm start --inspect         # enable Node.js inspector
wattpm start --env .env.prod
```

| Arg / Flag | Description | Default |
|------------|-------------|---------|
| `root` | Project directory | current dir |
| `-c, --config <config>` | Config file | autodetect |
| `-e, --env <path>` | Custom `.env` file path | — |
| `-i, --inspect` | Enable the Node.js inspector | — |

### `stop`

Stop a running application.

```bash
wattpm stop                    # stop (if only one running)
wattpm stop my-app             # stop by name
wattpm stop 12345              # stop by process ID
```

| Arg | Description | Default |
|-----|-------------|---------|
| `id` | Process ID or application name | required if multiple running |

### `restart`

Restart all applications within a running instance. Picks up application and config changes but **not** changes to the main Watt config. Apps restart in parallel; workers are replaced one at a time per app.

```bash
wattpm restart
wattpm restart my-app
```

| Arg | Description | Default |
|-----|-------------|---------|
| `id` | Process ID or application name | required if multiple running |

### `reload`

Reload a running application, picking up directory-level changes.

```bash
wattpm reload
wattpm reload my-app
```

| Arg | Description | Default |
|-----|-------------|---------|
| `id` | Process ID or application name | required if multiple running |

---

## Monitoring & Diagnostics

### `ps`

List all currently running Watt applications. No arguments.

```bash
wattpm ps
```

### `applications`

Show all sub-applications within a running Watt instance.

```bash
wattpm applications              # if only one instance running
wattpm applications my-app       # specify instance
```

| Arg | Description | Default |
|-----|-------------|---------|
| `id` | Process ID or application name | required if multiple running |

### `logs`

Stream log output from a running application. Without specifying a sub-application, streams from all.

```bash
wattpm logs                      # all logs from the running instance
wattpm logs my-app               # logs from specific instance
wattpm logs my-app api-service   # logs from a specific sub-application
```

| Arg | Description | Default |
|-----|-------------|---------|
| `id` | Process ID or application name | required if multiple running |
| `application` | Sub-application name | all |

### `env`

Display environment variables for a running application or one of its sub-applications.

```bash
wattpm env my-app
wattpm env my-app api-service
wattpm env my-app api-service --table
```

| Arg / Flag | Description | Default |
|------------|-------------|---------|
| `id` | Process ID or application name | required if multiple running |
| `application` | Sub-application name | — |
| `-t, --table` | Display in tabular format | — |

### `config`

Display the configuration of a running application or sub-application.

```bash
wattpm config my-app
wattpm config my-app api-service
```

| Arg | Description | Default |
|-----|-------------|---------|
| `id` | Process ID or application name | required if multiple running |
| `application` | Sub-application name | — |

---

## Testing & Injection

### `inject`

Send an HTTP request to a running application and print the response. Defaults to the runtime entrypoint if no sub-application is specified.

```bash
# Simple GET
wattpm inject my-app --path /health

# POST with data
wattpm inject my-app api-service --method POST --path /users \
  --header "Content-Type: application/json" \
  --data '{"name": "Alice"}'

# POST from file, save response
wattpm inject my-app --method POST --path /upload \
  --data-file payload.json --output response.json

# Include response headers
wattpm inject my-app --path /api --full-output
```

| Arg / Flag | Description | Default |
|------------|-------------|---------|
| `id` | Process ID or application name | required if multiple running |
| `application` | Sub-application name | entrypoint |
| `-m, --method <value>` | HTTP method | `GET` |
| `-p, --path <value>` | Request path | `/` |
| `-H, --header <value>` | Request header (repeatable) | — |
| `-d, --data <value>` | Request body string | — |
| `-D, --data-file <path>` | Read request body from file | — |
| `-o, --output <path>` | Write response to file | — |
| `-f, --full-output` | Include response headers | `false` |

---

## External Applications

### `import`

Add an external resource (local folder or URL) as an application. Supports GitHub shorthand (`$USER/$REPOSITORY`). When invoked without arguments, fixes missing Platformatic dependencies in local apps.

```bash
# Import from GitHub (shorthand)
wattpm import . platformatic/acme-base --id base-app

# Import from URL
wattpm import . https://github.com/org/repo.git --branch develop

# Import to custom path
wattpm import . platformatic/ui-kit --path web/ui

# Fix missing dependencies (no arguments)
wattpm import
```

| Arg / Flag | Description | Default |
|------------|-------------|---------|
| `root` | Project directory | current dir |
| `url` | GitHub shorthand or URL | — |
| `-c, --config <config>` | Config file | autodetect |
| `-i, --id <value>` | Application ID | URL basename |
| `-p, --path <value>` | Destination path | app ID |
| `-H, --http` | Use HTTP URL for GitHub expansion | — |
| `-b, --branch <branch>` | Branch to clone | `main` |
| `-s, --skip-dependencies` | Skip dependency install (no-arg mode) | — |
| `-P, --package-manager <executable>` | Package manager (no-arg mode) | — |

### `resolve`

Clone and configure all external applications that have a `url` field defined with the path as an environment variable. After cloning, sets the relative path in `.env`.

```bash
wattpm resolve
wattpm resolve --username deploy --password secret  # for private repos
wattpm resolve --skip-dependencies
```

| Arg / Flag | Description | Default |
|------------|-------------|---------|
| `root` | Project directory | current dir |
| `-c, --config <config>` | Config file | autodetect |
| `-u, --username <value>` | Username for HTTP URLs | — |
| `-p, --password <value>` | Password for HTTP URLs | — |
| `-s, --skip-dependencies` | Skip dependency install | — |
| `-P, --package-manager <executable>` | Package manager | autodetect |

---

## Configuration Management

### `patch-config`

Apply a patch file to runtime and application configurations using [JSON Patch](https://jsonpatch.com/) format.

```bash
wattpm patch-config . patches/production.js
```

The patch file's default export must be a function receiving `runtime` and `applications`, returning an object with `runtime` and `applications` keys containing JSON Patch operations.

**Example patch file (`patches/production.js`):**
```javascript
export default function (runtime, applications) {
  return {
    runtime: [
      { op: 'replace', path: '/logger/level', value: 'warn' }
    ],
    applications: {
      'api-service': [
        { op: 'add', path: '/env/NODE_ENV', value: 'production' }
      ]
    }
  }
}
```

| Arg / Flag | Description | Default |
|------------|-------------|---------|
| `root` | Project directory | current dir |
| `patch` | Patch file path | — |
| `-c, --config <config>` | Config file | autodetect |

---

## Administration

### `admin`

Launch the Watt administration UI (`watt-admin`) via `npx`.

```bash
wattpm admin                   # launch admin UI
wattpm admin latest            # use the latest published version
wattpm admin --package-manager pnpm
```

| Arg / Flag | Description | Default |
|------------|-------------|---------|
| `latest` | Use the latest released version | — |
| `-P, --package-manager <executable>` | Package manager | autodetect |

---

## Utility

### `version`

Output the current Watt version.

```bash
wattpm version
wattpm --version
```

### `help`

Display help for Watt or a specific command.

```bash
wattpm help
wattpm help inject
wattpm help resolve
```
