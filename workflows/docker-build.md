---
name: docker-build
description: Build Docker image with strict validation
---

# Workflow: Docker Build

Build and validate Docker images following security and optimization best practices.

## Prerequisites

| Tool | Purpose | Install |
|------|---------|---------|
| `docker` | Build and run images | [Docker Desktop](https://docker.com/products/docker-desktop) |
| `hadolint` | Dockerfile linting | `brew install hadolint` |
| `trivy` or `docker scout` | Vulnerability scanning | `brew install trivy` |

## Steps

### 1. Lint Dockerfile
```bash
# // turbo
hadolint Dockerfile
# STOP if any errors (DL prefix = error, SC prefix = shellcheck warning)
```

**Common Hadolint Fixes:**
| Rule | Issue | Fix |
|------|-------|-----|
| DL3006 | Missing tag | Use `FROM alpine:3.19` not `FROM alpine` |
| DL3007 | Using `latest` | Pin specific version |
| DL3008 | apt without versions | Use `apt-get install pkg=version` |
| DL3018 | apk without versions | Use `apk add pkg=version` |
| DL3025 | JSON form for CMD | Use `CMD ["executable", "arg"]` |
| DL4006 | No SHELL | Add `SHELL ["/bin/bash", "-o", "pipefail", "-c"]` |

### 2. Build Image
```bash
# Standard build
docker build -t app:local .

# With build args
docker build \
  --build-arg VERSION=$(git describe --tags) \
  --build-arg COMMIT=$(git rev-parse --short HEAD) \
  -t app:local .

# Multi-platform (if needed)
docker buildx build --platform linux/amd64,linux/arm64 -t app:local .
```

**Build Optimization Tips:**
- Order Dockerfile layers: dependencies first, source code last
- Use `.dockerignore` to exclude unnecessary files
- Combine RUN commands to reduce layers

### 3. Verify Image Size
```bash
docker images app:local --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

**Size Guidelines:**
| Base Image | Target Size | Notes |
|------------|-------------|-------|
| Alpine | < 50MB | Minimal apps |
| Distroless | < 30MB | Go/static binaries |
| Slim variants | < 200MB | Python/Node apps |
| Full images | < 500MB | Dev/build images only |

**If image is too large:**
- Use multi-stage builds
- Remove build dependencies in final stage
- Use `--no-cache` for package managers
- Check for unnecessary files with `docker history app:local`

### 4. Security Scan
```bash
# Option A: Docker Scout (Docker Desktop)
docker scout cves app:local
docker scout recommendations app:local

# Option B: Trivy
trivy image --severity HIGH,CRITICAL app:local
# STOP if CRITICAL vulnerabilities found

# Option C: Grype
grype app:local
```

**Vulnerability Response:**
| Severity | Action |
|----------|--------|
| CRITICAL | **STOP** - Must fix before proceeding |
| HIGH | Should fix - update base image or packages |
| MEDIUM | Plan to fix - add to backlog |
| LOW | Acceptable risk - document decision |

### 5. Smoke Test
```bash
# Version check (if supported)
docker run --rm app:local --version

# Health endpoint (for web apps)
docker run -d --name smoke-test -p 8080:8080 app:local
sleep 5
curl -f http://localhost:8080/health || exit 1
docker stop smoke-test && docker rm smoke-test

# Basic functionality
docker run --rm app:local echo "Container starts successfully"
```

### 6. Tag for Registry (Optional)
```bash
# Tag with version
VERSION=$(git describe --tags --always)
docker tag app:local registry.example.com/app:${VERSION}
docker tag app:local registry.example.com/app:latest

# Push (requires authentication)
docker push registry.example.com/app:${VERSION}
docker push registry.example.com/app:latest
```

## Cleanup
```bash
# Remove local test images
docker rmi app:local

# Prune dangling images
docker image prune -f

# Full cleanup (careful in shared environments)
docker system prune -f
```

## Common Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `COPY failed: file not found` | Wrong context or .dockerignore | Check file exists and isn't ignored |
| `apt-get: command not found` | Wrong base image | Use debian-based image or `apk` for Alpine |
| `exec format error` | Architecture mismatch | Build for correct platform |
| `no space left on device` | Docker disk full | Run `docker system prune` |
| `unauthorized: access denied` | Registry auth | Run `docker login` |

## Report Format

```
✅ Docker build successful

Image: app:local
Size: 45.2 MB
Base: alpine:3.19
Vulnerabilities: 0 critical, 2 high (base image)
Smoke test: passed

Next: docker push registry.example.com/app:v1.2.3
```
