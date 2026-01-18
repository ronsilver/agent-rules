# Validate

Validate the current project using appropriate tools to ensure code quality, correctness, and security.

## Project Detection

| Marker File | Project Type | Priority |
|-------------|--------------|----------|
| `*.tf` | Terraform | 1 |
| `go.mod` | Go | 2 |
| `pyproject.toml` / `requirements.txt` | Python | 3 |
| `package.json` + `tsconfig.json` | TypeScript | 4 |
| `package.json` (no tsconfig) | Node.js | 5 |
| `*.sh` | Bash | 6 |
| `Dockerfile` | Docker | 7 |
| `Chart.yaml` | Helm | 8 |

**Multi-language projects**: Run validation for ALL detected types, in priority order.

## Validation Commands

### Terraform
```bash
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
tflint --recursive          # If .tflint.hcl exists
```

### Go
```bash
go fmt ./...
go vet ./...
golangci-lint run           # Requires .golangci.yml
go test -race ./...
```

### Python
```bash
ruff format . --check
ruff check .
mypy src/                   # Or mypy . if no src/
pytest --tb=short
```

### TypeScript / Node.js
```bash
npm run typecheck           # Or: npx tsc --noEmit
npm run lint                # Or: npx eslint .
npm test
```

### Bash
```bash
shfmt -d *.sh               # Format check
shellcheck *.sh
```

### Docker
```bash
hadolint Dockerfile
docker scout quickview .    # Or: trivy fs .
```

### Helm
```bash
helm lint ./chart
helm template ./chart > /dev/null
```

## Execution Flow

1. **Detect** project type(s) based on marker files
2. **Verify prerequisites** - Check required tools are installed
3. **Execute** validation commands in order
4. **STOP immediately** if any command fails (exit code != 0)
5. **Report** results with actionable details

## Report Format

### Success
```
✅ Validation passed

Summary:
- Terraform: fmt ✓, validate ✓
- Go: fmt ✓, vet ✓, lint ✓, test ✓ (coverage: 78%)
```

### Failure
```
❌ Validation failed

Error: golangci-lint found issues

File: internal/handler/user.go:45
Rule: errcheck
Issue: Error return value not checked

Suggested fix:
  result, err := db.Query(...)
  if err != nil {
      return fmt.Errorf("query failed: %w", err)
  }
```

### Warnings
```
⚠️ Validation passed with warnings

Warning: pytest collected 0 tests
Action: Verify test files match pattern test_*.py or *_test.py
```

## Common Issues & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `golangci-lint: command not found` | Tool not installed | `go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest` |
| `ruff: command not found` | Tool not installed | `pip install ruff` |
| `terraform init required` | Backend not initialized | Run `terraform init -backend=false` |
| `no tests found` | Wrong test naming | Use `test_*.py` (Python) or `*_test.go` (Go) |
| `type errors in node_modules` | Missing excludes | Add `"exclude": ["node_modules"]` to tsconfig.json |

## Prerequisites

Ensure these tools are installed before running validation:

| Project | Required Tools |
|---------|---------------|
| Terraform | `terraform`, `tflint` |
| Go | `go`, `golangci-lint` |
| Python | `python`, `ruff`, `mypy`, `pytest` |
| Node/TS | `node`, `npm`, `tsc` |
| Bash | `shfmt`, `shellcheck` |
| Docker | `hadolint`, `docker` or `trivy` |
| Helm | `helm`, `kubeval` |
