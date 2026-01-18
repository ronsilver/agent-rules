---
trigger: glob
globs: ["Dockerfile", "docker-compose*.yml", ".dockerignore"]
---

# Docker Best Practices

## Validation Tools - MANDATORY

```bash
# Lint Dockerfile
hadolint Dockerfile

# Scan for vulnerabilities
docker scout cves <image>
# Or: trivy image <image>
```

## Security - NON-NEGOTIABLE

### 1. Non-Root User - MANDATORY

Containers **MUST NOT** run as root:

```dockerfile
# Alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Debian/Ubuntu
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser

# Distroless (already non-root)
FROM gcr.io/distroless/static-debian12:nonroot
```

### 2. Base Image Selection

| Use Case | Recommended Base | Size |
|----------|-----------------|------|
| Go binaries | `gcr.io/distroless/static` | ~2MB |
| Go with libc | `gcr.io/distroless/base` | ~20MB |
| Python | `python:3.12-slim` | ~150MB |
| Node.js | `node:20-slim` | ~200MB |
| General minimal | `alpine:3.19` | ~7MB |

**Rules:**
- ✅ Pin specific versions: `python:3.12.1-slim`
- ✅ Use SHA for production: `python@sha256:abc123...`
- ❌ **NEVER** use `latest` tag
- ❌ **NEVER** use full images in production (`python:3.12` = 1GB+)

### 3. Minimal Attack Surface

```dockerfile
# ❌ Bad - Full image with unnecessary tools
FROM python:3.12
RUN pip install flask

# ✅ Good - Slim image, no cache
FROM python:3.12-slim
RUN pip install --no-cache-dir flask
```

### 4. Read-Only Filesystem

```dockerfile
# In Dockerfile
RUN chmod -R a-w /app

# Or at runtime
docker run --read-only --tmpfs /tmp myapp
```

## Multi-Stage Builds - MANDATORY

Separate build and runtime environments:

```dockerfile
# ============ Build Stage ============
FROM golang:1.23-alpine AS builder

WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server ./cmd/server

# ============ Runtime Stage ============
FROM gcr.io/distroless/static-debian12:nonroot

COPY --from=builder /app/server /server

EXPOSE 8080
ENTRYPOINT ["/server"]
```

**Python Example:**
```dockerfile
# Build stage
FROM python:3.12-slim AS builder

WORKDIR /app
RUN pip install --no-cache-dir poetry
COPY pyproject.toml poetry.lock ./
RUN poetry export -f requirements.txt > requirements.txt

# Runtime stage
FROM python:3.12-slim

WORKDIR /app
COPY --from=builder /app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ ./src/
USER nobody
CMD ["python", "-m", "src.main"]
```

## Layer Optimization

### Order: Least → Most Frequently Changed

```dockerfile
# 1. Base image (rarely changes)
FROM node:20-slim

# 2. System dependencies (rarely changes)
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# 3. Application dependencies (changes occasionally)
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production

# 4. Application code (changes frequently)
COPY src/ ./src/

# 5. Runtime config
USER node
EXPOSE 3000
CMD ["node", "src/index.js"]
```

### Combine RUN Commands

```dockerfile
# ❌ Bad - Multiple layers
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# ✅ Good - Single layer, cleanup in same layer
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*
```

## .dockerignore - MANDATORY

```dockerignore
# Git
.git
.gitignore

# Dependencies (rebuild in container)
node_modules
venv
__pycache__
*.pyc

# Secrets (NEVER include)
.env
.env.*
*.pem
*.key
secrets/

# Build artifacts
dist
build
*.egg-info

# IDE/Editor
.idea
.vscode
*.swp

# Documentation
README.md
docs/

# Docker files (not needed in context)
Dockerfile*
docker-compose*
.dockerignore
```

## Health Checks - MANDATORY

```dockerfile
# HTTP health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# For distroless (no curl)
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD ["/healthcheck"]  # Include a static binary
```

## Resource Limits

Always set in production:

```yaml
# docker-compose.yml
services:
  app:
    image: myapp:latest
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 128M
```

```bash
# docker run
docker run --memory=512m --cpus=1.0 myapp
```

## Secrets Management

```dockerfile
# ❌ NEVER - Secrets in image
ENV API_KEY=sk-12345
COPY .env /app/.env

# ✅ Runtime secrets
# Pass at runtime: docker run -e API_KEY=$API_KEY myapp
# Or use Docker secrets / external secrets manager
```

**Docker Compose Secrets:**
```yaml
services:
  app:
    secrets:
      - db_password
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

## Common Hadolint Rules

| Rule | Issue | Fix |
|------|-------|-----|
| DL3006 | Missing image tag | `FROM alpine:3.19` |
| DL3007 | Using `latest` | Pin specific version |
| DL3008 | apt without version | `apt-get install pkg=1.2.3` |
| DL3009 | apt lists not deleted | Add `rm -rf /var/lib/apt/lists/*` |
| DL3018 | apk without version | `apk add pkg=1.2.3` |
| DL3025 | CMD not JSON | `CMD ["node", "app.js"]` |
| DL4006 | No pipefail | `SHELL ["/bin/bash", "-o", "pipefail", "-c"]` |

## Checklist

- [ ] Base image pinned to specific version
- [ ] Multi-stage build used
- [ ] Running as non-root user
- [ ] No secrets in image
- [ ] .dockerignore exists and excludes secrets
- [ ] HEALTHCHECK defined
- [ ] hadolint passes
- [ ] Image scanned for vulnerabilities
- [ ] Resource limits configured
