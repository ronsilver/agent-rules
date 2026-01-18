---
name: pr-review
description: Review a Pull Request
---

# Workflow: PR Review

Systematically review Pull Requests for correctness, security, and quality.

## Prerequisites

| Tool | Purpose | Install |
|------|---------|---------|
| `gh` | GitHub CLI | `brew install gh` |
| `git` | Version control | Pre-installed |

```bash
# Authenticate if needed
gh auth login
```

## Steps

### 1. Fetch PR Context

```bash
# View PR summary
gh pr view <pr-number>

# View changed files
gh pr diff <pr-number> --name-only

# View full diff
gh pr diff <pr-number>

# Check CI status
gh pr checks <pr-number>
```

**Gather Context:**
- What problem does this PR solve?
- Is there a linked issue?
- What's the scope of changes?

### 2. Review Checklist

#### Functionality (REQUIRED)
- [ ] Does the code do what the PR description says?
- [ ] Are edge cases handled?
- [ ] Does it break existing functionality?

#### Tests (REQUIRED)
- [ ] Are there tests for new functionality?
- [ ] Do all tests pass in CI?
- [ ] Is test coverage maintained/improved?

#### Security (REQUIRED)
- [ ] No hardcoded secrets or credentials?
- [ ] User inputs validated and sanitized?
- [ ] No SQL injection, XSS, or command injection?
- [ ] Sensitive data not logged?

#### Code Quality (RECOMMENDED)
- [ ] Follows project conventions?
- [ ] No unnecessary complexity?
- [ ] Clear naming and structure?
- [ ] No code duplication?

#### Documentation (RECOMMENDED)
- [ ] Public APIs documented?
- [ ] README updated if needed?
- [ ] Breaking changes noted?

### 3. Local Verification

```bash
# Checkout PR branch
gh pr checkout <pr-number>

# Run validation
make lint      # Or: npm run lint
make test      # Or: npm test

# Test manually if needed
make run       # Or: npm start
```

### 4. Provide Feedback

**Comment Types:**

| Prefix | Meaning | Action Required |
|--------|---------|-----------------|
| `blocking:` | Must fix before merge | Yes |
| `suggestion:` | Recommended improvement | No |
| `question:` | Need clarification | Maybe |
| `nit:` | Minor style preference | No |
| `praise:` | Good pattern to highlight | No |

**Comment Format:**
```markdown
**blocking:** SQL Injection vulnerability

`src/db/users.py:45`

User input is concatenated directly into query:
```python
query = f"SELECT * FROM users WHERE id = {user_id}"
```

**Suggested fix:**
```python
query = "SELECT * FROM users WHERE id = %s"
cursor.execute(query, (user_id,))
```
```

### 5. Submit Review

```bash
# Approve
gh pr review <pr-number> --approve --body "LGTM! Clean implementation."

# Request changes
gh pr review <pr-number> --request-changes --body "See comments for required fixes."

# Comment only (no approval/rejection)
gh pr review <pr-number> --comment --body "Left some suggestions, overall looks good."
```

## Review Decision Matrix

| Condition | Action |
|-----------|--------|
| All checks pass, no issues | **Approve** |
| Minor suggestions only | **Approve** with comments |
| Questions need answers | **Comment**, wait for response |
| Security issues | **Request Changes** |
| Missing tests for new code | **Request Changes** |
| Broken functionality | **Request Changes** |

## Review Best Practices

**DO:**
- Review within 24 hours if possible
- Be specific and actionable in feedback
- Suggest concrete fixes, not just problems
- Acknowledge good patterns
- Ask questions if unclear

**DON'T:**
- Nitpick style issues covered by linters
- Bikeshed on naming unless truly unclear
- Block for minor preferences
- Approve without reviewing all files
- Ignore failing CI checks

## Common Review Comments

| Issue | Comment |
|-------|---------|
| Missing error handling | `blocking:` Add error handling for `X.method()` |
| No tests | `blocking:` Please add tests for the new `calculateDiscount` function |
| Hardcoded value | `suggestion:` Extract `3600` to a named constant like `SESSION_TIMEOUT_SECONDS` |
| Complex function | `suggestion:` Consider extracting lines 45-60 into a separate function |
| Good pattern | `praise:` Nice use of early returns here, much cleaner! |

## Report Format

```
PR #123: Add user authentication

Files changed: 8
Lines: +245 / -32
CI Status: ✓ All checks passed

Review Summary:
- Functionality: ✓
- Tests: ✓ (12 new tests)
- Security: ✓
- Quality: 2 suggestions

Decision: Approved with comments
```
