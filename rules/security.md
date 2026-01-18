---
trigger: glob
globs: ["*.py", "*.js", "*.ts", "*.go", "*.java", "*.php", "*.rb", "*.cs"]
---

# Security OWASP Top 10:2025 - Best Practices

Rules based on **OWASP Top 10 2025**, **OWASP ASVS**, and **CWE Top 25**.

## 0. Zero Trust & Redaction - CRITICAL

The agent **MUST NEVER** output real secrets:

| ❌ Never Output | ✅ Use Instead |
|----------------|---------------|
| `API_KEY="sk-12345abc"` | `API_KEY="<REDACTED>"` |
| `password: "myP@ssw0rd"` | `password: "********"` |
| `AKIA...` (AWS keys) | `AWS_ACCESS_KEY_ID="<REDACTED>"` |

## Security Tooling - MANDATORY

Run before every PR:

```bash
# Secrets detection
gitleaks detect --source . -v

# Dependency vulnerabilities
trivy fs . --severity HIGH,CRITICAL
npm audit --audit-level=high     # Node
pip-audit                         # Python
govulncheck ./...                 # Go

# Static analysis (SAST)
semgrep --config=auto .
bandit -r src/                    # Python
gosec ./...                       # Go
```

## A01:2025 - Broken Access Control

**Principle: Deny by default**

```python
# ❌ Bad - Implicit allow
def get_document(doc_id, user):
    return Document.query.get(doc_id)

# ✅ Good - Explicit authorization check
def get_document(doc_id, user):
    doc = Document.query.get(doc_id)
    if not doc:
        raise NotFoundError()
    if doc.owner_id != user.id and not user.has_role("admin"):
        raise ForbiddenError("Access denied")
    return doc
```

**SSRF Prevention:**
```python
# Block internal/private networks
BLOCKED_NETWORKS = [
    "10.0.0.0/8",
    "172.16.0.0/12",
    "192.168.0.0/16",
    "169.254.169.254/32",  # AWS metadata
    "127.0.0.0/8",
]

def fetch_url(user_url):
    ip = socket.gethostbyname(urlparse(user_url).hostname)
    for network in BLOCKED_NETWORKS:
        if ipaddress.ip_address(ip) in ipaddress.ip_network(network):
            raise SecurityError("Access to internal networks blocked")
    return requests.get(user_url)
```

## A02:2025 - Security Misconfiguration

**Production Checklist:**
- [ ] Debug mode **DISABLED**
- [ ] Default credentials changed
- [ ] Unnecessary features disabled
- [ ] Error messages don't leak internals
- [ ] Security headers configured

**Required HTTP Headers:**
```python
# Flask example
@app.after_request
def add_security_headers(response):
    response.headers['Strict-Transport-Security'] = 'max-age=31536000; includeSubDomains'
    response.headers['Content-Security-Policy'] = "default-src 'self'"
    response.headers['X-Content-Type-Options'] = 'nosniff'
    response.headers['X-Frame-Options'] = 'DENY'
    response.headers['Referrer-Policy'] = 'strict-origin-when-cross-origin'
    return response
```

## A03:2025 - Supply Chain Failures

**Pin GitHub Actions by SHA:**
```yaml
# ❌ Vulnerable
uses: actions/checkout@v4

# ✅ Secure - Pinned to commit SHA
uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
```

**Lock Dependencies:**
```bash
# Generate lock files
npm ci                    # Uses package-lock.json
pip install -r requirements.txt --require-hashes
go mod verify
```

## A04:2025 - Cryptographic Failures

**Password Hashing:**
| Algorithm | Status | Parameters |
|-----------|--------|------------|
| Argon2id | ✅ Recommended | m=64MB, t=3, p=4 |
| bcrypt | ✅ Acceptable | cost ≥ 12 |
| scrypt | ✅ Acceptable | N ≥ 2^14 |
| PBKDF2-SHA256 | ⚠️ Legacy | iterations ≥ 600,000 |
| MD5, SHA1 | ❌ FORBIDDEN | Never use |

```python
# ✅ Correct - Argon2id
from argon2 import PasswordHasher
ph = PasswordHasher()
hash = ph.hash(password)
ph.verify(hash, password)

# ✅ Correct - bcrypt
import bcrypt
hash = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))
bcrypt.checkpw(password.encode(), hash)
```

**Encryption:**
| Use Case | Algorithm |
|----------|-----------|
| Symmetric encryption | AES-256-GCM |
| Alternative | ChaCha20-Poly1305 |
| Key derivation | HKDF |
| TLS | 1.2 or 1.3 only |

## A05:2025 - Injection

### SQL Injection (CWE-89)
```python
# ❌ VULNERABLE
query = f"SELECT * FROM users WHERE email = '{email}'"

# ✅ SAFE - Parameterized query
query = "SELECT * FROM users WHERE email = %s"
cursor.execute(query, (email,))

# ✅ SAFE - ORM
user = User.query.filter_by(email=email).first()
```

### Command Injection (CWE-78)
```python
# ❌ VULNERABLE
os.system(f"convert {user_filename} output.png")

# ✅ SAFE - No shell, list arguments
subprocess.run(["convert", user_filename, "output.png"], shell=False)
```

### XSS (CWE-79)
```javascript
// ❌ VULNERABLE
element.innerHTML = userInput;

// ✅ SAFE - Text content (auto-escaped)
element.textContent = userInput;

// ✅ SAFE - Sanitization library
import DOMPurify from 'dompurify';
element.innerHTML = DOMPurify.sanitize(userInput);
```

## A07:2025 - Authentication Failures

**Session Security:**
```python
# Secure session cookie settings
response.set_cookie(
    "session_id",
    value=session_token,
    httponly=True,       # No JavaScript access
    secure=True,         # HTTPS only
    samesite="Lax",      # CSRF protection
    max_age=3600,        # 1 hour expiry
    path="/",
    domain=".example.com"
)
```

**Password Requirements:**
- Minimum 12 characters
- Check against breached password lists (HaveIBeenPwned API)
- No complexity rules (NIST 800-63B)
- Rate limit login attempts

## A09:2025 - Logging & Monitoring

**Log Security Events:**
```python
# Events to log
logger.info("login_success", extra={"user_id": user.id, "ip": request.ip})
logger.warning("login_failed", extra={"email": email, "ip": request.ip})
logger.warning("access_denied", extra={"user_id": user.id, "resource": resource})
logger.critical("privilege_escalation_attempt", extra={"user_id": user.id})
```

**Never Log:**
- Passwords (even hashed)
- Session tokens / API keys
- Credit card numbers
- SSN / National IDs
- Full request bodies (may contain secrets)

```python
# ❌ Bad - Logging sensitive data
logger.info(f"User login: {username}, password: {password}")

# ✅ Good - Sanitized
logger.info(f"User login: {username}")
```

## Input Validation - MANDATORY

```python
# Python with Pydantic
from pydantic import BaseModel, EmailStr, constr

class UserCreate(BaseModel):
    email: EmailStr
    name: constr(min_length=1, max_length=100)
    age: int = Field(ge=0, le=150)

# Automatic validation
@app.post("/users")
def create_user(user: UserCreate):
    # user is already validated
    pass
```

```typescript
// TypeScript with Zod
import { z } from 'zod';

const UserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  age: z.number().int().min(0).max(150),
});

// Validate
const user = UserSchema.parse(requestBody);
```

## Security Code Review Checklist

- [ ] **Input Validation**: All user inputs validated with strict schemas
- [ ] **SQL Injection**: Only parameterized queries used
- [ ] **XSS**: Output properly escaped/sanitized
- [ ] **Authentication**: Passwords hashed with Argon2id/bcrypt
- [ ] **Authorization**: Access control on every endpoint
- [ ] **Secrets**: No hardcoded credentials, using env vars
- [ ] **Logging**: Security events logged, no sensitive data
- [ ] **Dependencies**: Scanned for vulnerabilities
- [ ] **Headers**: Security headers configured
- [ ] **HTTPS**: TLS 1.2+ enforced
