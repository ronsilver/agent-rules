---
name: validate
description: Validate current project code
---

# Workflow: Validate

Execute validation tools based on project type. **STOP on first failure**.

## Prerequisites

Verify required tools are installed before running:

| Project | Required Tools | Install Command |
|---------|---------------|-----------------|
| Terraform | `terraform`, `tflint` | `brew install terraform tflint` |
| Go | `go`, `golangci-lint` | `go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest` |
| Python | `ruff`, `mypy`, `pytest` | `pip install ruff mypy pytest` |
| Node.js | `node`, `npm` | Use nvm or official installer |
| Docker | `hadolint` | `brew install hadolint` |
| Helm | `helm`, `kubeval` | `brew install helm kubeval` |

## Detection

| File Marker | Project Type | Priority |
|-------------|--------------|----------|
| `*.tf` | Terraform | 1 |
| `go.mod` | Go | 2 |
| `package.json` + `tsconfig.json` | TypeScript | 3 |
| `package.json` | Node.js | 4 |
| `pyproject.toml` / `requirements.txt` | Python | 5 |
| `Dockerfile` | Docker | 6 |
| `Chart.yaml` | Helm | 7 |

**Multi-project**: Run validation for ALL detected types in priority order.

## Commands

### Terraform
```bash
# // turbo
terraform fmt -recursive -check
# STOP if not formatted

terraform init -backend=false
# STOP if init fails

terraform validate
# STOP if invalid

# Optional: TFLint (if .tflint.hcl exists)
tflint --init && tflint --recursive
```

### Go
```bash
# // turbo
gofmt -l .
# STOP if files need formatting (non-empty output)

# // turbo
go vet ./...
# STOP if vet finds issues

golangci-lint run
# STOP if linter fails

go test -race ./...
# STOP if tests fail
```

### Python
```bash
# // turbo
ruff format . --check
# STOP if not formatted

# // turbo
ruff check .
# STOP if lint fails

mypy src/ --ignore-missing-imports
# STOP if type errors (exit code != 0)

pytest --tb=short -q
# STOP if tests fail
```

### Node.js / TypeScript
```bash
# // turbo
npm run typecheck
# Or: npx tsc --noEmit
# STOP if type errors

npm run lint
# Or: npx eslint .
# STOP if lint fails

npm test
# STOP if tests fail
```

### Docker
```bash
hadolint Dockerfile
# STOP if issues found

# Optional: Security scan
docker scout quickview .
# Or: trivy fs .
```

### Helm
```bash
helm lint ./chart
# STOP if lint fails

helm template ./chart > /dev/null
# STOP if template fails
```

## Execution Steps

1. **Detect** project type(s) from marker files
2. **Verify** required tools are installed
3. **Execute** commands in order shown above
4. **STOP** immediately on first failure (exit code != 0)
5. **Report** results with actionable details

## Common Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `terraform init required` | Backend not initialized | Run `terraform init -backend=false` |
| `golangci-lint: not found` | Tool not installed | `go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest` |
| `ruff: not found` | Tool not installed | `pip install ruff` |
| `Cannot find module` | Missing dependencies | Run `npm install` first |
| `mypy: No module named` | Missing stubs | `pip install types-<package>` or add to `mypy.ini` |
| `EACCES permission denied` | npm permissions | Use `nvm` or fix npm prefix |

## Report Format

### Success
```
✅ Validation passed

Project: Go + Docker
- go fmt: ✓
- go vet: ✓
- golangci-lint: ✓
- go test: ✓ (42 tests, 78% coverage)
- hadolint: ✓
```

### Failure
```
❌ Validation failed

Step: golangci-lint run
Exit code: 1

Error:
  internal/handler/user.go:45:9: Error return value of `db.Close` is not checked (errcheck)

Fix:
  Add error handling: `if err := db.Close(); err != nil { ... }`
```
