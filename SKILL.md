---
name: docker-starter
description: Generate docker-compose.yml from natural language, use templates for common stacks, and manage environments with one command
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  tags:
    - docker
    - docker-compose
    - containers
    - devops
    - templates
    - infrastructure
    - environment
    - volumes
  related_skills:
    - price-tracker
    - podcast-digest
---

# Docker Starter

Generate `docker-compose.yml` files from natural language descriptions, spin up common stacks from templates, manage `.env` files, and handle health checks and volumes — all from a single command.

## Overview

This skill translates natural language requirements into production-ready Docker Compose configurations. It provides templates for common stacks (web server, database, monitoring, media server), manages environment variables securely, and offers one-command spin up/down with health check configuration.

## Commands

```yaml
commands:
  - name: generate
    description: "Generate docker-compose.yml from natural language"
    usage: "generate a web server with nginx, postgres database, and redis cache"
    args:
      - name: description
        description: "Natural language description of your stack"
        required: true
      - name: output
        description: "Output directory (default: current directory)"
        required: false

  - name: template
    description: "List or apply a pre-built stack template"
    usage: "template list | template apply <name>"
    args:
      - name: action
        description: "list or apply"
        required: true
      - name: name
        description: "Template name (web-server, database, monitoring, media-server, full-stack)"
        required: false

  - name: up
    description: "Start containers from docker-compose.yml"
    usage: "up [directory]"
    args:
      - name: directory
        description: "Directory containing docker-compose.yml (default: .)"
        required: false
      - name: detach
        description: "Run in detached mode (default: true)"
        required: false

  - name: down
    description: "Stop and remove containers"
    usage: "down [directory]"
    args:
      - name: directory
        description: "Directory containing docker-compose.yml (default: .)"
        required: false

  - name: status
    description: "Show status of containers"
    usage: "status [directory]"
    args:
      - name: directory
        description: "Directory containing docker-compose.yml (default: .)"
        required: false

  - name: env
    description: "Manage .env file — generate, update, or show variables"
    usage: "env generate | env set KEY=VALUE | env show"
    args:
      - name: action
        description: "generate, set, or show"
        required: true
      - name: values
        description: "Key=value pairs to set"
        required: false

  - name: logs
    description: "Show container logs"
    usage: "logs <service-name> [directory] [--lines 50]"
    args:
      - name: service
        description: "Service name from docker-compose.yml"
        required: true
      - name: lines
        description: "Number of lines to show (default: 50)"
        required: false
```

## Configuration

```yaml
configuration:
  templates_path: "~/.hermes/skills/docker-starter/templates"
  default_output_dir: "."
  docker_compose_version: "3.8"
  templates:
    - web-server
    - database
    - monitoring
    - media-server
    - full-stack
  health_check_defaults:
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 40s
```

## How It Works

### 1. Natural Language Generation

Describe what you need and get a `docker-compose.yml`:

```bash
# "generate a web server with nginx, postgres database, and redis cache"
# Produces:
```

```yaml
# docker-compose.yml
version: "3.8"

services:
  nginx:
    image: nginx:latest
    container_name: web_server
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./html:/usr/share/nginx/html
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    restart: unless-stopped
    networks:
      - app-network

  postgres:
    image: postgres:16-alpine
    container_name: postgres_db
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-app}
      POSTGRES_USER: ${POSTGRES_USER:-app}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD in .env}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-app}"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    container_name: redis_cache
    command: redis-server --requirepass ${REDIS_PASSWORD:?Set REDIS_PASSWORD in .env}
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
    networks:
      - app-network

volumes:
  postgres_data:
  redis_data:

networks:
  app-network:
    driver: bridge
```

### 2. Template Stacks

**Web Server (nginx + static site):**
```bash
# Usage: template apply web-server
```
```yaml
version: "3.8"
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./html:/usr/share/nginx/html
      - ./nginx/conf.d:/etc/nginx/conf.d
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
```

**Database (PostgreSQL + Adminer):**
```bash
# Usage: template apply database
```
```yaml
version: "3.8"
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-app}
      POSTGRES_USER: ${POSTGRES_USER:-app}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Set in .env}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-app}"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped

  adminer:
    image: adminer:latest
    ports:
      - "8080:8080"
    depends_on:
      - postgres
    restart: unless-stopped

volumes:
  postgres_data:
```

**Monitoring (Grafana + Prometheus):**
```bash
# Usage: template apply monitoring
```
```yaml
version: "3.8"
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9090"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin}
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
```

**Media Server (Plex + Sonarr + Radarr):**
```bash
# Usage: template apply media-server
```
```yaml
version: "3.8"
services:
  plex:
    image: lscr.io/linuxserver/plex:latest
    environment:
      - PUID=${PUID:-1000}
      - PGID=${PGID:-1000}
      - VERSION=docker
    volumes:
      - plex_config:/config
      - ${MEDIA_PATH:?Set MEDIA_PATH in .env}:/data
    ports:
      - "32400:32400"
    restart: unless-stopped

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    environment:
      - PUID=${PUID:-1000}
      - PGID=${PGID:-1000}
    volumes:
      - sonarr_config:/config
      - ${MEDIA_PATH:?Set MEDIA_PATH in .env}:/data
    ports:
      - "8989:8989"
    restart: unless-stopped

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    environment:
      - PUID=${PUID:-1000}
      - PGID=${PGID:-1000}
    volumes:
      - radarr_config:/config
      - ${MEDIA_PATH:?Set MEDIA_PATH in .env}:/data
    ports:
      - "7878:7878"
    restart: unless-stopped

volumes:
  plex_config:
  sonarr_config:
  radarr_config:
```

**Full Stack (Nginx + Node.js + PostgreSQL + Redis):**
```bash
# Usage: template apply full-stack
```
```yaml
version: "3.8"
services:
  app:
    build: ./app
    environment:
      - DATABASE_URL=postgresql://${POSTGRES_USER:-app}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB:-app}
      - REDIS_URL=redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - app-network

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
    depends_on:
      - app
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
    networks:
      - app-network

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-app}
      POSTGRES_USER: ${POSTGRES_USER:-app}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Set in .env}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-app}"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD:-redis}
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
    networks:
      - app-network

volumes:
  postgres_data:
  redis_data:

networks:
  app-network:
    driver: bridge
```

### 3. Environment Variable Management

`.env` file generated alongside `docker-compose.yml`:

```bash
# .env (auto-generated — edit with real values)
POSTGRES_DB=app
POSTGRES_USER=app
POSTGRES_PASSWORD=CHANGE_ME_STRONG_PASSWORD
REDIS_PASSWORD=CHANGE_ME_REDIS_PASSWORD
GRAFANA_PASSWORD=admin
PUID=1000
PGID=1000
MEDIA_PATH=/path/to/media
```

### 4. One-Command Operations

```bash
# Start stack
cd /my/project && docker compose up -d

# Stop stack  
docker compose down

# Stop and remove volumes (full reset)
docker compose down -v

# Check status
docker compose ps

# View logs
docker compose logs -f nginx --tail 50
```

### 5. Natural Language Patterns

- `"generate docker compose for <description>"` — generate from description
- `"set up a <template> stack"` — apply a template
- `"docker up"` or `"start containers"` — spin up
- `"docker down"` or `"stop containers"` — spin down
- `"show container status"` — show running containers
- `"generate env file"` — create .env template
- `"set POSTGRES_PASSWORD=mysecret"` — update .env variable

## Pitfalls

- **Password security**: Never commit `.env` files to version control. Add `.env` to `.gitignore` immediately. The skill generates `.env.example` (with placeholder values) for sharing.
- **Port conflicts**: If a port is already in use, Docker will fail to start the container. Check for conflicts with `ss -tlnp | grep <port>` before starting.
- **Health check timing**: `start_period` must be long enough for the service to initialise. PostgreSQL can take 30+ seconds on first boot. Adjust `start_period` accordingly.
- **Volume data loss**: `docker compose down -v` removes volumes and **all data**. Always warn before running this. Use `down` (without `-v`) by default.
- **Docker Compose v2**: Use `docker compose` (v2, space) not `docker-compose` (v1, hyphen). The skill requires Docker Compose v2, which is included with Docker Desktop.
- **Network conflicts**: Multiple compose projects on the same host may create network name collisions. The skill uses project-specific network names (`app-network` by default).
- **Template customisation**: Templates are starting points. Always review generated files before running in production. Pay attention to image tags — `latest` is convenient but can introduce unexpected breaking changes.
- **Volume mounting on SELinux**: On SELinux-enabled systems (some Linux distros), volume mounts need `:z` or `:Z` suffix. The templates don't include this; add it if needed.
- **ARM compatibility**: Some images (Plex, etc.) may not have ARM builds. If running on a Raspberry Pi or ARM server, check multi-arch availability or use `platform: linux/amd64` with emulation.

## Verification Steps

1. **Verify Docker and Compose are installed:**
   ```bash
   docker --version
   docker compose version
   ```

2. **Test generating a stack:**
   ```bash
   mkdir -p /tmp/docker-test && cd /tmp/docker-test
   # Generate a simple web server stack
   # The skill should produce docker-compose.yml and .env
   ls docker-compose.yml .env
   ```

3. **Validate the generated compose file:**
   ```bash
   docker compose config
   # Should output the parsed YAML without errors
   ```

4. **Start and verify a template stack:**
   ```bash
   cd /tmp/docker-test
   docker compose up -d
   docker compose ps
   # All services should show "Up" or "healthy"
   ```

5. **Check health checks are working:**
   ```bash
   docker inspect --format='{{json .State.Health}}' $(docker compose ps -q nginx) | jq
   # Should show "healthy" status
   ```

6. **Test environment variable loading:**
   ```bash
   docker compose exec postgres env | grep POSTGRES
   # Should show values from .env file
   ```

7. **Clean up test stack:**
   ```bash
   docker compose down -v
   rm -rf /tmp/docker-test
   ```

8. **Verify templates directory exists:**
   ```bash
   ls ~/.hermes/skills/docker-starter/templates/ && echo "Templates OK" \
     || echo "MISSING: mkdir -p ~/.hermes/skills/docker-starter/templates"
   ```