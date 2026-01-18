---
name: fix-pr-comments
description: Systematically address PR review comments
---

# Workflow: Fix PR Comments

Systematically process and resolve PR review comments with clear commit history.

## Prerequisites

```bash
# Ensure you're on the PR branch
gh pr checkout <pr-number>

# Fetch latest
git fetch origin
git rebase origin/main
```

## Steps

### 1. Fetch and List Comments

```bash
# View PR summary and comments
gh pr view <pr-number> --comments

# View the diff for context
gh pr diff <pr-number>

# List only unresolved review threads (if available)
gh api repos/:owner/:repo/pulls/<pr-number>/reviews
```

### 2. Triage Comments by Priority

| Priority | Type | Icon | Action |
|----------|------|------|--------|
| 1 | Security vulnerability | 🔴 | Fix immediately, no exceptions |
| 2 | Bug/Error | 🔴 | Fix immediately |
| 3 | Breaking change | 🟠 | Fix or justify |
| 4 | Performance issue | 🟠 | Fix if significant |
| 5 | Style/Convention | 🟡 | Apply if improves clarity |
| 6 | Documentation | 🟢 | Apply if relevant |
| 7 | Nitpick/Preference | ⚪ | Discuss or accept |
| 8 | False positive | ⚪ | Explain with evidence |

**Process order:** High priority (🔴) → Medium (🟠) → Low (🟡🟢⚪)

### 3. Process Each Comment

#### If Comment is Valid

1. **Understand** the issue completely
2. **Fix** the code
3. **Verify** fix works (run tests)
4. **Commit** with clear reference

**Commit Strategy Options:**

**Option A: Fixup commits (recommended for small fixes)**
```bash
# Create fixup commit referencing original
git commit --fixup=<original-commit-sha>

# Before final push, squash fixups
git rebase -i --autosquash origin/main
```

**Option B: Separate commits (for significant changes)**
```bash
git commit -m "fix(scope): address review - description

Addresses review comment about XYZ.
- Changed A to B
- Added validation for C"
```

**Option C: Amend existing commit (if comment applies to most recent)**
```bash
git add <files>
git commit --amend --no-edit
# Note: Requires force push
```

#### If Comment is Invalid/Disagree

1. **Acknowledge** the reviewer's perspective
2. **Explain** your reasoning with evidence
3. **Cite** documentation, standards, or benchmarks
4. **Suggest** alternative if applicable

**Response Template:**
```markdown
Thanks for the feedback! I considered this approach, but chose the current implementation because:

1. [Reason with evidence]
2. [Reference to documentation/standard]

Alternative considered: [X], but [reason it wasn't chosen].

Happy to discuss further or reconsider if you see issues with this reasoning.
```

### 4. Batch Related Fixes

Group related comments into single commits when logical:

```bash
# Multiple style fixes in same file
git add src/utils.py
git commit -m "style(utils): address review comments

- Rename variable per review
- Add docstring per review
- Fix import order"
```

### 5. Verify All Changes

```bash
# Run full validation
# // turbo
make lint   # Or: npm run lint
make test   # Or: npm test

# STOP if any check fails - fix before continuing
```

### 6. Interactive Rebase (Clean History)

If you have multiple fixup commits:

```bash
# Squash fixup commits into their targets
git rebase -i --autosquash origin/main

# Review the rebase plan, save and exit
# This combines fixup commits with their originals
```

### 7. Push Changes

```bash
# If you rebased/amended, force push with lease
git push origin <branch> --force-with-lease

# If only added new commits
git push origin <branch>
```

### 8. Notify Reviewer

```bash
# Add summary comment
gh pr comment <pr-number> --body "Review comments addressed:

✅ Fixed SQL injection vulnerability (commit abc123)
✅ Added input validation (commit def456)
✅ Updated documentation (commit ghi789)
💬 Responded to naming suggestion - see thread

Ready for re-review!"
```

**Or resolve individual threads:**
```bash
# Mark conversation as resolved (via GitHub UI or API)
gh api repos/:owner/:repo/pulls/<pr-number>/reviews/<review-id>/dismiss
```

## Comment Response Examples

### Security Issue
```markdown
**Comment:** SQL injection vulnerability in query

**Fix commit:** `fix(db): use parameterized query for user lookup`
```python
# Before
query = f"SELECT * FROM users WHERE id = {user_id}"

# After
query = "SELECT * FROM users WHERE id = %s"
cursor.execute(query, (user_id,))
```
```

### Disagreement with Evidence
```markdown
**Comment:** Should use `map()` instead of list comprehension

**Response:** I prefer list comprehension here because:
1. Python style guide (PEP 8) recommends list comprehensions for simple transformations
2. Benchmark shows equivalent performance for this size
3. More readable for this specific case

However, happy to change if there's a project convention I'm missing.
```

### Requesting Clarification
```markdown
**Comment:** This doesn't handle edge case X

**Response:** Could you clarify the expected behavior for case X?

Current handling:
- Input: `null` → Returns empty list
- Input: `[]` → Returns empty list

Is there a specific scenario you're thinking of?
```

## Common Issues

| Issue | Solution |
|-------|----------|
| Reviewer hasn't re-reviewed | Politely ping after 24-48h |
| New comments after fixes | Process new comments, push again |
| Conflicting reviewer opinions | Escalate to team lead or discuss in thread |
| Stale branch after rebase | Force push with `--force-with-lease` |

## Checklist

- [ ] All 🔴 (critical) comments addressed
- [ ] All 🟠 (important) comments addressed or justified
- [ ] Responded to all comments (fix or explanation)
- [ ] All tests pass
- [ ] Commit history is clean (fixups squashed)
- [ ] Pushed changes
- [ ] Notified reviewer with summary
