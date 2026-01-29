# PHP Integration with Platformatic Watt

Watt can run PHP applications (Laravel, WordPress, and custom PHP) directly inside the Node.js runtime using the `@platformatic/php` stackable.

## Package
```
@platformatic/php
```

## How It Works

The PHP stackable uses `@platformatic/php-node`, a Rust-based native module that embeds a multi-threaded PHP runtime directly inside Node.js. PHP runs in the Node worker pool, shares cache, auth, and rate-limits with adjacent JS services.

## Detection

- `composer.json` exists
- `artisan` file exists (Laravel)
- `wp-config.php` exists (WordPress)
- `public/index.php` exists (generic PHP)

## Requirements

- Node.js >= 22.14.0
- PHP binary dependencies installed (see [php-node repository](https://github.com/platformatic/php-node))
- For WordPress/Laravel with database: MySQL/MariaDB

---

## WordPress Configuration

### watt.json for WordPress

```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/php/1.0.0.json",
  "module": "@platformatic/php",
  "docroot": "./public",
  "application": {
    "basePath": "/"
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

### Directory Structure

```
wordpress-app/
├── watt.json
├── public/
│   ├── wp-config.php
│   ├── wp-content/
│   ├── wp-admin/
│   ├── wp-includes/
│   └── index.php
└── .env
```

### Environment Variables

```
PLT_SERVER_HOSTNAME=0.0.0.0
PLT_SERVER_LOGGER_LEVEL=info
PORT=3000
PLT_WP_TYPESCRIPT=false

# WordPress Database
DB_HOST=localhost
DB_NAME=wordpress
DB_USER=root
DB_PASSWORD=password
```

### wp-config.php Setup

Ensure `wp-config.php` reads from environment:
```php
define('DB_NAME', getenv('DB_NAME') ?: 'wordpress');
define('DB_USER', getenv('DB_USER') ?: 'root');
define('DB_PASSWORD', getenv('DB_PASSWORD') ?: '');
define('DB_HOST', getenv('DB_HOST') ?: 'localhost');
```

---

## Laravel Configuration

### watt.json for Laravel

```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/php/1.0.0.json",
  "module": "@platformatic/php",
  "docroot": "./public",
  "rewriter": {
    "rules": [
      {
        "match": "^/(?!index\\.php)(.*)$",
        "rewrite": "/index.php/$1"
      }
    ]
  },
  "application": {
    "basePath": "/"
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

### Rewriter Configuration

The `rewriter` implements Laravel's front controller pattern, routing all requests through `index.php`:

```json
{
  "rewriter": {
    "rules": [
      {
        "match": "^/(?!index\\.php)(.*)$",
        "rewrite": "/index.php/$1"
      }
    ]
  }
}
```

### Directory Structure

```
laravel-app/
├── watt.json
├── artisan
├── composer.json
├── public/
│   └── index.php
├── app/
├── routes/
├── resources/
└── .env
```

---

## Generic PHP Configuration

### watt.json for Custom PHP

```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/php/1.0.0.json",
  "module": "@platformatic/php",
  "docroot": "./public",
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

Any PHP files in the `docroot` directory are automatically routed.

---

## Installation

### Create New PHP Service

```bash
npx wattpm@latest create --module=@platformatic/php
```

### Add to Existing Project

```bash
npm install wattpm @platformatic/php
```

### Install PHP Dependencies

Follow the [php-node installation guide](https://github.com/platformatic/php-node) for your platform.

---

## Multi-Service Setup (WordPress + Next.js)

For headless WordPress with a JavaScript frontend:

### Root watt.json

```json
{
  "$schema": "https://schemas.platformatic.dev/watt/3.0.0.json",
  "runtime": {
    "logger": {
      "level": "{PLT_SERVER_LOGGER_LEVEL}"
    },
    "server": {
      "hostname": "{PLT_SERVER_HOSTNAME}",
      "port": "{PORT}"
    }
  },
  "services": [
    {
      "id": "frontend",
      "path": "./web/frontend"
    },
    {
      "id": "wordpress",
      "path": "./web/wp"
    },
    {
      "id": "composer",
      "path": "./web/composer"
    }
  ]
}
```

### Inter-Service Communication

PHP services can call adjacent JS services with intra-process RPC using internal hostnames:
```
http://frontend.plt.local
http://wordpress.plt.local
```

---

## Docker Setup for WordPress

### docker-compose.yml

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - PLT_SERVER_HOSTNAME=0.0.0.0
      - PORT=3000
      - DB_HOST=db
      - DB_NAME=wordpress
      - DB_USER=wordpress
      - DB_PASSWORD=wordpress
    depends_on:
      db:
        condition: service_healthy

  db:
    image: mysql:8
    environment:
      - MYSQL_ROOT_PASSWORD=root
      - MYSQL_DATABASE=wordpress
      - MYSQL_USER=wordpress
      - MYSQL_PASSWORD=wordpress
    volumes:
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  mysql_data:
```

---

## Common Issues

### PHP Binary Not Found

Install PHP dependencies per your platform:
- macOS: `brew install php`
- Ubuntu: `apt install php php-mysql php-xml php-mbstring`
- See [php-node docs](https://github.com/platformatic/php-node) for full requirements

### Database Connection Failed

1. Ensure MySQL is running
2. Check environment variables are set
3. Verify database credentials in wp-config.php or .env

### WordPress Plugins Not Loading

Ensure `wp-content/plugins/` is in the correct location within your `docroot`.

### Laravel Routes Not Working

Verify the `rewriter` rules are correctly configured to route through `index.php`.

---

## Resources

- [Platformatic PHP Stackable](https://github.com/platformatic/php)
- [php-node Repository](https://github.com/platformatic/php-node)
- [WordPress + Next.js Example](https://github.com/platformatic/watt-next-wordpress)
- [Laravel Example](https://github.com/platformatic/watt-next-laravel)
- [npm Package](https://www.npmjs.com/package/@platformatic/php)
