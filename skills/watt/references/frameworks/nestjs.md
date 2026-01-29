# NestJS Integration with Platformatic Watt

## Package
```
@platformatic/node
```

## Detection
- `nest-cli.json` exists
- `package.json` contains `@nestjs/core` in dependencies

## watt.json Configuration

```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/node/3.0.0.json",
  "application": {
    "commands": {
      "development": "nest start --watch",
      "build": "nest build",
      "production": "node dist/main.js"
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

### With SWC (Faster Builds)
```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/node/3.0.0.json",
  "application": {
    "commands": {
      "development": "nest start --watch --builder swc",
      "build": "nest build --builder swc",
      "production": "node dist/main.js"
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

## Installation

```bash
npm install wattpm @platformatic/node
```

For SWC builder:
```bash
npm install @swc/cli @swc/core --save-dev
```

## Key Considerations

### Port Configuration
Update `main.ts` to use environment port:
```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const port = process.env.PORT || 3000;
  await app.listen(port, '0.0.0.0');

  console.log(`Application running on port ${port}`);
}

bootstrap();
```

### Build Requirement
NestJS MUST be built before production. The `dist/main.js` is the entry point.

### Entry Point
Always `dist/main.js` after build (standard NestJS structure).

## Health Check

Use the `@nestjs/terminus` package:

```bash
npm install @nestjs/terminus
```

```typescript
// health/health.controller.ts
import { Controller, Get } from '@nestjs/common';
import { HealthCheck, HealthCheckService, HttpHealthIndicator } from '@nestjs/terminus';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private http: HttpHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([]);
  }
}
```

```typescript
// health/health.module.ts
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { HttpModule } from '@nestjs/axios';
import { HealthController } from './health.controller';

@Module({
  imports: [TerminusModule, HttpModule],
  controllers: [HealthController],
})
export class HealthModule {}
```

## Common Patterns

### Standard main.ts
```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    logger: ['error', 'warn', 'log', 'debug', 'verbose'],
  });

  // Global validation pipe
  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,
    transform: true,
  }));

  // CORS
  app.enableCors();

  // Global prefix
  app.setGlobalPrefix('api');

  const port = process.env.PORT || 3000;
  await app.listen(port, '0.0.0.0');
}

bootstrap();
```

### With Swagger
```typescript
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('API')
    .setVersion('1.0')
    .build();
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('docs', app, document);

  const port = process.env.PORT || 3000;
  await app.listen(port, '0.0.0.0');
}

bootstrap();
```

### Graceful Shutdown
```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Enable graceful shutdown hooks
  app.enableShutdownHooks();

  const port = process.env.PORT || 3000;
  await app.listen(port, '0.0.0.0');
}

bootstrap();
```

## Monorepo Support

For NestJS monorepo, specify the project:
```json
{
  "application": {
    "commands": {
      "development": "nest start --watch my-app",
      "build": "nest build my-app",
      "production": "node dist/apps/my-app/main.js"
    }
  }
}
```

## Environment Configuration

Use `@nestjs/config`:
```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: '.env',
    }),
  ],
})
export class AppModule {}
```

Access in services:
```typescript
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class AppService {
  constructor(private configService: ConfigService) {}

  getPort(): number {
    return this.configService.get<number>('PORT', 3000);
  }
}
```

## Common Issues

### @nestjs/cli not found
Ensure `@nestjs/cli` is installed:
```bash
npm install @nestjs/cli --save-dev
```

### Build errors
Run `nest build` manually to see detailed errors. Check TypeScript configuration.

### Hot reload not working
Ensure `--watch` flag is in development command. Check `nest-cli.json`:
```json
{
  "compilerOptions": {
    "watchAssets": true
  }
}
```

### Port already in use
Ensure you're reading from `process.env.PORT` and binding to `0.0.0.0`.
