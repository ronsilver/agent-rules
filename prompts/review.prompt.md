# Review

Review the selected code or recent changes to identify issues. Focus on **impact** and **actionability**.

## Review Scope

| Context | What to Review |
|---------|---------------|
| Selected code | Only the selected lines |
| `git diff` | Changed lines + immediate context |
| Full file | Entire file when explicitly requested |
| PR review | All changed files in the PR |

## Review Categories

### 1. Bugs & Correctness (Priority: HIGHEST)

| Issue | Example | Detection |
|-------|---------|-----------|
| Null/undefined | `user.name` without null check | Missing optional chaining or guards |
| Race conditions | Shared mutable state | `go vet -race`, async without locks |
| Off-by-one | `i <= len` instead of `i < len` | Loop boundaries, array access |
| Resource leaks | Unclosed connections/files | Missing `defer`, `finally`, `using` |
| Error swallowing | `except: pass`, `_ = err` | Empty catch blocks, ignored returns |

```python
# ❌ Bug: Resource leak
def read_file(path):
    f = open(path)
    return f.read()  # File never closed

# ✅ Fixed
def read_file(path):
    with open(path) as f:
        return f.read()
```

### 2. Security (Priority: CRITICAL)

| Vulnerability | CWE | Detection Pattern |
|--------------|-----|-------------------|
| Hardcoded secrets | CWE-798 | `password=`, `api_key=`, `token=` literals |
| SQL Injection | CWE-89 | String concatenation in queries |
| XSS | CWE-79 | Unescaped user input in HTML |
| Path Traversal | CWE-22 | User input in file paths without validation |
| Command Injection | CWE-78 | `shell=True`, backticks, `eval()` |
| SSRF | CWE-918 | User-controlled URLs without allowlist |

```go
// ❌ SQL Injection
query := "SELECT * FROM users WHERE id = " + userInput

// ✅ Parameterized
query := "SELECT * FROM users WHERE id = $1"
db.Query(query, userInput)
```

### 3. Performance (Priority: MEDIUM)

| Anti-Pattern | Impact | Solution |
|-------------|--------|----------|
| N+1 queries | O(n) DB calls | Eager loading, JOINs, batch queries |
| O(n²) loops | Slow at scale | Use maps/sets for lookups |
| Sync blocking | Thread starvation | Async I/O, worker pools |
| Missing indexes | Full table scans | Add indexes on WHERE/JOIN columns |
| Memory bloat | OOM risk | Streaming, pagination, generators |

```python
# ❌ N+1 Query
for user in users:
    orders = Order.query.filter_by(user_id=user.id).all()

# ✅ Eager Load
users = User.query.options(joinedload(User.orders)).all()
```

### 4. Maintainability (Priority: LOW)

| Code Smell | Threshold | Action |
|-----------|-----------|--------|
| Long function | >50 lines | Extract sub-functions |
| Deep nesting | >3 levels | Early returns, extract logic |
| Magic numbers | Any literal | Named constants |
| Code duplication | >10 lines repeated | Extract to function |
| Poor naming | Unclear intent | Rename with domain terms |

## Review Limits

- **Maximum issues to report**: 10 (prioritize by severity)
- **Focus on**: Issues that affect correctness, security, or significant performance
- **Skip**: Pure style preferences already covered by linters

## Report Format

```markdown
## 🔴 [CRITICAL] SQL Injection Vulnerability

**File:** `src/db/users.py:45`
**Category:** Security
**CWE:** CWE-89

**Problem:**
User input is concatenated directly into SQL query, allowing arbitrary SQL execution.

**Current Code:**
```python
query = f"SELECT * FROM users WHERE email = '{email}'"
```

**Suggested Fix:**
```python
query = "SELECT * FROM users WHERE email = %s"
cursor.execute(query, (email,))
```

**Impact:** An attacker could extract or modify all database records.
```

## Severity Levels

| Level | Icon | Criteria | Action |
|-------|------|----------|--------|
| CRITICAL | 🔴 | Security vuln, data loss, crash | **Block merge** |
| WARNING | 🟠 | Performance, tech debt, bugs | Should fix before merge |
| INFO | 🟡 | Style, suggestions, nitpicks | Optional improvement |

## Review Checklist

Before completing review, verify:

- [ ] All **CRITICAL** issues have suggested fixes
- [ ] Security issues reference CWE when applicable
- [ ] Performance issues include complexity analysis (O notation)
- [ ] No false positives from linter-covered rules
- [ ] Praise good patterns when encountered (brief)
