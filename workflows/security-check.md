---
name: security-check
description: Deep security analysis
---

# Workflow: Security Check

Comprehensive security analysis covering secrets, dependencies, code vulnerabilities, and infrastructure.

## Prerequisites

| Tool | Purpose | Install |
|------|---------|---------|
| `gitleaks` | Secret detection | `brew install gitleaks` |
| `trivy` | Vulnerability scanner | `brew install trivy` |
| `semgrep` | SAST scanner | `brew install semgrep` |
| `govulncheck` | Go vulnerabilities | `go install golang.org/x/vuln/cmd/govulncheck@latest` |

## 1. Secrets & Credentials Detection

### Automated Scanning
```bash
# // turbo
gitleaks detect --source . --verbose
# STOP if secrets found (exit code != 0)

# Scan git history (slower but thorough)
gitleaks detect --source . --log-opts="--all"
```

### Manual Patterns to Search
```bash
# API Keys & Tokens
grep -rE "(api[_-]?key|apikey|secret[_-]?key|access[_-]?token)\s*[:=]\s*[\"'][^\"']{16,}" .

# AWS Keys
grep -rE "AKIA[0-9A-Z]{16}" .

# Private Keys
grep -rE "BEGIN (RSA |EC |OPENSSH )?PRIVATE KEY" .

# Connection Strings with passwords
grep -rE "(mongodb|postgres|mysql|redis)://[^:]+:[^@]+@" .
```

### False Positive Handling
| Pattern | Likely False Positive |
|---------|----------------------|
| `YOUR_API_KEY`, `xxx`, `changeme` | Placeholder values |
| In `*_test.go`, `test_*.py` | Test fixtures |
| In `*.example`, `*.sample` | Documentation |
| In `node_modules/`, `vendor/` | Dependencies |

## 2. Dependency Vulnerabilities (SCA)

### By Language
```bash
# Node.js
npm audit --audit-level=high
# STOP if high/critical vulnerabilities

# Python
pip-audit
# Or: safety check

# Go
govulncheck ./...
# STOP if vulnerabilities found

# Ruby
bundle audit check --update

# Java/Kotlin
./gradlew dependencyCheckAnalyze
```

### Universal Scanner
```bash
# Trivy - works for all languages
trivy fs . --severity HIGH,CRITICAL
# STOP if HIGH or CRITICAL found

# Snyk (requires auth)
snyk test
```

### Vulnerability Response

| Severity | CVSS | Action | SLA |
|----------|------|--------|-----|
| CRITICAL | 9.0-10.0 | **STOP** - Fix immediately | 24 hours |
| HIGH | 7.0-8.9 | Fix before merge | 7 days |
| MEDIUM | 4.0-6.9 | Plan to fix | 30 days |
| LOW | 0.1-3.9 | Accept risk or fix | 90 days |

## 3. Static Application Security Testing (SAST)

```bash
# Semgrep - Multi-language SAST
semgrep --config=auto .
# Or specific rulesets:
semgrep --config=p/security-audit .
semgrep --config=p/owasp-top-ten .

# Language-specific
bandit -r . -f json          # Python
gosec ./...                   # Go
npm run lint:security         # Node (eslint-plugin-security)
```

### Key Vulnerabilities to Detect

| Vulnerability | CWE | Detection Pattern |
|--------------|-----|-------------------|
| SQL Injection | CWE-89 | String concatenation in queries |
| Command Injection | CWE-78 | `shell=True`, `exec()`, backticks |
| Path Traversal | CWE-22 | User input in file paths |
| XSS | CWE-79 | Unescaped output, `innerHTML` |
| SSRF | CWE-918 | User-controlled URLs |
| Insecure Deserialization | CWE-502 | `pickle.loads()`, `eval()` |

## 4. Infrastructure Security (IaC)

### Terraform
```bash
# // turbo
trivy config . --severity HIGH,CRITICAL
# Or: tfsec .
# STOP if HIGH/CRITICAL issues

# Check for common misconfigs
grep -rE 'cidr_blocks.*0\.0\.0\.0/0' *.tf         # Open to world
grep -rE 'encrypted\s*=\s*false' *.tf              # Unencrypted resources
grep -rE 'publicly_accessible\s*=\s*true' *.tf     # Public databases
```

### Kubernetes
```bash
# // turbo
trivy config . --severity HIGH,CRITICAL

# Manual checks
grep -rE 'runAsUser:\s*0' .                        # Running as root
grep -rE 'runAsNonRoot:\s*false' .                 # Allowing root
grep -rE 'privileged:\s*true' .                    # Privileged containers
grep -rE 'allowPrivilegeEscalation:\s*true' .      # Privilege escalation
grep -rE 'readOnlyRootFilesystem:\s*false' .       # Writable filesystem
```

### Docker
```bash
hadolint Dockerfile

# Image scanning
trivy image app:latest --severity HIGH,CRITICAL
# STOP if CRITICAL vulnerabilities
```

## 5. GitHub Actions Security

```bash
# Check for unpinned actions
grep -rE 'uses:\s*[^@]+@(main|master|v\d+)' .github/workflows/

# Should be pinned by SHA:
# uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
```

## 6. Report Format

### Critical Finding
```markdown
## 🔴 CRITICAL: Hardcoded AWS Credentials

**File:** `config/settings.py:23`
**Tool:** gitleaks
**CWE:** CWE-798 (Hardcoded Credentials)

**Finding:**
```python
AWS_ACCESS_KEY_ID = "AKIAIOSFODNN7EXAMPLE"
AWS_SECRET_ACCESS_KEY = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

**Impact:**
- Full access to AWS account
- Potential data breach
- Compliance violation

**Remediation:**
1. Immediately rotate the exposed credentials
2. Use environment variables or secrets manager
3. Add to `.gitignore` and `.gitleaks.toml`

**References:**
- https://cwe.mitre.org/data/definitions/798.html
```

### Summary Report
```
🔒 Security Scan Results

Scanned: 2024-01-15 10:30:00 UTC
Repository: myapp

| Category | Critical | High | Medium | Low |
|----------|----------|------|--------|-----|
| Secrets | 0 | 0 | 0 | 0 |
| Dependencies | 0 | 2 | 5 | 12 |
| SAST | 0 | 1 | 3 | 8 |
| IaC | 0 | 0 | 1 | 2 |

Status: ⚠️ HIGH issues found - review required
Action: Fix 2 high-severity dependency vulnerabilities before merge
```

## Checklist

- [ ] Secrets scan passed (gitleaks)
- [ ] No CRITICAL/HIGH dependency vulnerabilities
- [ ] SAST scan completed (semgrep/bandit)
- [ ] IaC scan passed (trivy config)
- [ ] Docker images scanned
- [ ] GitHub Actions pinned by SHA
