# Security Check

Analyze code for security vulnerabilities based on **OWASP Top 10:2025** and **CWE Top 25**.

## Analysis Categories

### 1. Secrets & Credentials (CWE-798)

**Detection Patterns:**
```regex
# API Keys & Tokens
(?i)(api[_-]?key|apikey|secret[_-]?key|access[_-]?token)\s*[:=]\s*["'][a-zA-Z0-9_\-]{16,}["']

# AWS Keys
AKIA[0-9A-Z]{16}
(?i)aws[_-]?secret[_-]?access[_-]?key\s*[:=]\s*["'][^"']{40}["']

# Passwords
(?i)(password|passwd|pwd)\s*[:=]\s*["'][^"']+["']

# Private Keys
-----BEGIN (RSA |EC |OPENSSH )?PRIVATE KEY-----

# JWT Tokens
eyJ[a-zA-Z0-9_-]*\.eyJ[a-zA-Z0-9_-]*\.[a-zA-Z0-9_-]*

# Connection Strings
(?i)(mongodb|postgres|mysql|redis):\/\/[^:]+:[^@]+@
```

**Tools by Language:**
| Language | Tool | Command |
|----------|------|---------|
| All | Gitleaks | `gitleaks detect --source .` |
| All | TruffleHog | `trufflehog filesystem .` |
| All | git-secrets | `git secrets --scan` |

**False Positives to Ignore:**
- Example/placeholder values: `YOUR_API_KEY`, `xxx`, `changeme`
- Test fixtures with fake credentials
- Documentation examples

### 2. Injection Vulnerabilities

| Type | CWE | Language | Vulnerable Pattern | Safe Pattern |
|------|-----|----------|-------------------|--------------|
| SQL Injection | CWE-89 | Python | `f"SELECT * FROM users WHERE id={id}"` | `cursor.execute("SELECT * FROM users WHERE id=%s", (id,))` |
| SQL Injection | CWE-89 | Go | `"SELECT * FROM users WHERE id=" + id` | `db.Query("SELECT * FROM users WHERE id=$1", id)` |
| SQL Injection | CWE-89 | Node | `` `SELECT * FROM users WHERE id=${id}` `` | `db.query("SELECT * FROM users WHERE id=$1", [id])` |
| Command Injection | CWE-78 | Python | `os.system(f"ls {user_input}")` | `subprocess.run(["ls", user_input], shell=False)` |
| Command Injection | CWE-78 | Go | `exec.Command("sh", "-c", userInput)` | `exec.Command("ls", userInput)` |
| XSS | CWE-79 | JS | `innerHTML = userInput` | `textContent = userInput` or sanitize |
| Path Traversal | CWE-22 | All | `open(user_path)` | `os.path.realpath()` + validate prefix |
| SSRF | CWE-918 | All | `requests.get(user_url)` | Allowlist domains + block internal IPs |

### 3. Authentication & Session (CWE-287, CWE-384)

**Password Storage:**
| Algorithm | Status | Notes |
|-----------|--------|-------|
| Argon2id | ✅ Recommended | Use for new systems |
| bcrypt | ✅ Acceptable | cost >= 12 |
| scrypt | ✅ Acceptable | N >= 2^14 |
| PBKDF2 | ⚠️ Legacy | iterations >= 600,000 |
| SHA256/MD5 | ❌ Forbidden | Never for passwords |

**Session Security:**
```python
# ✅ Secure cookie settings
response.set_cookie(
    "session",
    value=token,
    httponly=True,      # Prevents XSS access
    secure=True,        # HTTPS only
    samesite="Lax",     # CSRF protection
    max_age=3600        # 1 hour expiry
)
```

### 4. Sensitive Data Exposure (CWE-200, CWE-532)

**Log Sanitization - MANDATORY:**
```python
# ❌ Logging sensitive data
logger.info(f"User login: {username}, password: {password}")
logger.info(f"API response: {api_response}")  # May contain tokens

# ✅ Sanitized logging
logger.info(f"User login: {username}")
logger.info(f"API response: status={response.status_code}")
```

**Fields to NEVER log:**
- Passwords, tokens, API keys
- Credit card numbers (PCI-DSS)
- SSN, national IDs (PII)
- Health information (HIPAA)
- Full request/response bodies

### 5. Security Headers

**Required Headers:**
```
Strict-Transport-Security: max-age=31536000; includeSubDomains
Content-Security-Policy: default-src 'self'; script-src 'self'
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), camera=(), microphone=()
```

### 6. Supply Chain Security

**Dependency Scanning:**
```bash
# Node.js
npm audit --audit-level=high
npx better-npm-audit audit

# Python
pip-audit
safety check

# Go
govulncheck ./...

# General (all languages)
trivy fs .
snyk test
```

**GitHub Actions - Pin by SHA:**
```yaml
# ❌ Vulnerable to supply chain attacks
uses: actions/checkout@v4

# ✅ Pinned to specific commit
uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
```

## Severity Classification

| Severity | CVSS | Examples | SLA |
|----------|------|----------|-----|
| 🔴 CRITICAL | 9.0-10.0 | RCE, Auth bypass, SQL injection | Fix immediately |
| 🟠 HIGH | 7.0-8.9 | XSS, SSRF, Sensitive data exposure | Fix within 7 days |
| 🟡 MEDIUM | 4.0-6.9 | CSRF, Information disclosure | Fix within 30 days |
| 🟢 LOW | 0.1-3.9 | Minor info leak, Clickjacking | Fix within 90 days |

## Report Format

```markdown
## 🔴 [CRITICAL] SQL Injection in User Authentication

**File:** `src/auth/login.py:47`
**CWE:** CWE-89 (SQL Injection)
**CVSS:** 9.8 (Critical)

**Vulnerability:**
User-supplied email is concatenated directly into SQL query without sanitization.

**Vulnerable Code:**
```python
query = f"SELECT * FROM users WHERE email = '{email}'"
cursor.execute(query)
```

**Proof of Concept:**
```
email: ' OR '1'='1' --
```

**Impact:**
- Authentication bypass
- Full database read access
- Potential data exfiltration

**Remediation:**
```python
query = "SELECT * FROM users WHERE email = %s"
cursor.execute(query, (email,))
```

**References:**
- https://owasp.org/Top10/A03_2021-Injection/
- https://cwe.mitre.org/data/definitions/89.html
```

## Security Checklist

- [ ] No hardcoded secrets (run `gitleaks detect`)
- [ ] All user inputs validated and sanitized
- [ ] Parameterized queries used (no string concatenation)
- [ ] Passwords hashed with Argon2id/bcrypt
- [ ] Sessions use HttpOnly + Secure + SameSite cookies
- [ ] No sensitive data in logs
- [ ] Security headers configured
- [ ] Dependencies scanned for vulnerabilities
- [ ] GitHub Actions pinned by SHA
