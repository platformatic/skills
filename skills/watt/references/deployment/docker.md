# Docker Deployment for Platformatic Watt

## Dockerfile (Multi-stage Build)

Create a `Dockerfile` in the project root:

```dockerfile
# Build stage
FROM node:22-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install all dependencies (including devDependencies for build)
RUN npm ci

# Copy source code
COPY . .

# Build the application
RUN npm run build

# Production stage
FROM node:22-alpine AS production

WORKDIR /app

# Create non-root user for security
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 watt

# Copy package files
COPY package*.json ./

# Install only production dependencies
RUN npm ci --only=production && npm cache clean --force

# Copy built application from builder stage
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/watt.json ./

# Copy public assets if they exist
COPY --from=builder /app/public ./public

# Set ownership
RUN chown -R watt:nodejs /app

# Switch to non-root user
USER watt

# Set environment variables
ENV NODE_ENV=production
ENV PLT_SERVER_HOSTNAME=0.0.0.0
ENV PORT=3000

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

# Start command
CMD ["npm", "start"]
```

## .dockerignore

Create `.dockerignore`:

```
# Dependencies
node_modules
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Build output (rebuilt in container)
dist
build
.next
.nuxt

# Git
.git
.gitignore

# Environment (secrets should be injected at runtime)
.env
.env.*
!.env.example

# IDE
.idea
.vscode
*.swp
*.swo

# Documentation
*.md
!README.md

# Docker
Dockerfile*
docker-compose*
.dockerignore

# Testing
coverage
.nyc_output
__tests__
*.test.*
*.spec.*

# Misc
.DS_Store
*.log
tmp
temp
```

## Build Commands

```bash
# Build image
docker build -t my-watt-app:latest .

# Build with specific tag
docker build -t my-watt-app:v1.0.0 .

# Build with build args
docker build \
  --build-arg NODE_ENV=production \
  -t my-watt-app:latest .
```

## Run Commands

```bash
# Run container
docker run -d \
  -p 3000:3000 \
  -e PLT_SERVER_LOGGER_LEVEL=info \
  --name my-watt-app \
  my-watt-app:latest

# Run with environment file
docker run -d \
  -p 3000:3000 \
  --env-file .env.production \
  --name my-watt-app \
  my-watt-app:latest

# Run with memory limits
docker run -d \
  -p 3000:3000 \
  --memory=512m \
  --cpus=1 \
  --name my-watt-app \
  my-watt-app:latest

# View logs
docker logs -f my-watt-app

# Stop container
docker stop my-watt-app

# Remove container
docker rm my-watt-app
```

## Docker Compose (Development)

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      target: builder  # Use builder stage for dev
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - PLT_SERVER_LOGGER_LEVEL=debug
      - PLT_SERVER_HOSTNAME=0.0.0.0
      - PORT=3000
    volumes:
      - .:/app
      - /app/node_modules  # Preserve container's node_modules
    command: npm run dev
    restart: unless-stopped
```

## Docker Compose (Production)

Create `docker-compose.prod.yml`:

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      target: production
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - PLT_SERVER_LOGGER_LEVEL=info
      - PLT_SERVER_HOSTNAME=0.0.0.0
      - PORT=3000
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 10s

  # Optional: Nginx reverse proxy
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - app
    restart: unless-stopped
```

## Docker Compose with Database

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://user:password@db:5432/myapp
      - PLT_SERVER_HOSTNAME=0.0.0.0
      - PORT=3000
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d myapp"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # Optional: Redis for caching/sessions
  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

## Production Considerations

### Security
- Use specific Node.js version tags (not `latest`)
- Run as non-root user
- Don't include secrets in image
- Scan images for vulnerabilities: `docker scout cve my-watt-app`

### Performance
- Set memory limits: `--memory=512m`
- Set CPU limits: `--cpus=1`
- Use multi-stage builds to reduce image size
- Leverage build cache

### Reliability
- Configure restart policy: `--restart=unless-stopped`
- Add health checks
- Use proper logging driver

### Image Registry

```bash
# Tag for registry
docker tag my-watt-app:latest registry.example.com/my-watt-app:latest

# Push to registry
docker push registry.example.com/my-watt-app:latest

# Pull from registry
docker pull registry.example.com/my-watt-app:latest
```

## Framework-Specific Dockerfiles

### Next.js
```dockerfile
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine AS production
WORKDIR /app
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/watt.json ./
USER nextjs
ENV NODE_ENV=production PORT=3000
EXPOSE 3000
CMD ["node", "server.js"]
```

### NestJS
```dockerfile
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/watt.json ./
ENV NODE_ENV=production PORT=3000
EXPOSE 3000
CMD ["node", "dist/main.js"]
```
