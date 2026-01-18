---
name: pre-push
description: Comprehensive pre-push checks
---

# Workflow: Pre-Push

Run this checklist BEFORE pushing to remote. **Do not push code that fails validation.**

## Prerequisites

```bash
# Ensure you have the latest from remote
git fetch origin
```

## Steps

### 1. Check Repository Status

```bash
# // turbo
git status
```

**Verify:**
- [ ] No untracked files that should be committed
- [ ] No uncommitted changes (stash or commit them)
- [ ] Working directory is clean

```bash
# Check for sensitive files that might be staged
git diff --cached --name-only | grep -E '\.(env|pem|key|secret)$'
# Should return empty
```

**Sensitive Files - NEVER commit:**
- `.env`, `.env.local`, `.env.production`
- `*.pem`, `*.key`, `*.p12`
- `credentials.json`, `secrets.yaml`
- `id_rsa`, `id_ed25519`

### 2. Validate Code

Run ALL validation commands for your project type:

**Terraform:**
```bash
# // turbo
terraform fmt -check -recursive
terraform validate
tflint --recursive
```

**Go:**
```bash
# // turbo
gofmt -l . | grep -q . && echo "Run: go fmt ./..." && exit 1
go vet ./...
golangci-lint run
go test -race ./...
```

**Python:**
```bash
# // turbo
ruff format . --check
ruff check .
mypy src/
pytest -q
```

**Node/TS:**
```bash
# // turbo
npm run typecheck
npm run lint
npm test
```

**STOP**: If any command fails, fix it before continuing.

### 3. Security Check (Quick)

```bash
# // turbo
gitleaks detect --source . --no-git
# STOP if secrets detected
```

### 4. Sync with Remote

```bash
# Fetch latest changes
git fetch origin

# Rebase on main (or your target branch)
git rebase origin/main
# Or: git rebase origin/develop

# Resolve conflicts if any, then continue
# git rebase --continue
```

**If rebase has conflicts:**
1. Resolve conflicts in each file
2. `git add <resolved-files>`
3. `git rebase --continue`
4. Re-run validation after rebase

### 5. Review Commits

```bash
# View commits to be pushed
git log --oneline origin/main..HEAD
```

**Commit Message Checklist:**
- [ ] Follows Conventional Commits format
- [ ] No "WIP", "fix", "update" generic messages
- [ ] Each commit is atomic (single logical change)
- [ ] Commit messages explain "why" not just "what"

**Conventional Commits Format:**
```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

| Type | Usage |
|------|-------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Formatting, no code change |
| `refactor` | Code change, no new feature or fix |
| `test` | Adding/updating tests |
| `chore` | Build, CI, dependencies |

**Examples:**
- ✅ `feat(auth): add OAuth2 login support`
- ✅ `fix(api): handle null response from payment service`
- ❌ `fix stuff`
- ❌ `WIP`
- ❌ `update`

### 6. Squash/Fixup If Needed

```bash
# If you have messy commits, clean them up
git rebase -i origin/main

# In the editor:
# - pick (keep commit as-is)
# - squash (combine with previous)
# - fixup (combine, discard message)
# - reword (change message)
```

### 7. Push

```bash
# Push to your branch
git push origin <branch-name>

# If you rebased/squashed, force push with lease (safer)
git push origin <branch-name> --force-with-lease
```

**Force Push Safety:**
- `--force-with-lease` prevents overwriting others' work
- Never force push to `main` or `develop`
- Only force push to your own feature branches

## Pre-Push Git Hook (Optional)

Add to `.git/hooks/pre-push`:

```bash
#!/bin/bash
set -e

echo "Running pre-push checks..."

# Run linters
make lint || { echo "Lint failed"; exit 1; }

# Run tests
make test || { echo "Tests failed"; exit 1; }

# Check for secrets
gitleaks detect --source . --no-git || { echo "Secrets detected"; exit 1; }

echo "All pre-push checks passed!"
```

```bash
chmod +x .git/hooks/pre-push
```

## Common Issues

| Issue | Solution |
|-------|----------|
| `rejected: non-fast-forward` | Pull/rebase first: `git pull --rebase origin main` |
| `force-with-lease rejected` | Someone else pushed; fetch and rebase again |
| Merge conflicts | Resolve, stage, continue rebase |
| Wrong branch | `git checkout correct-branch` before pushing |

## Checklist Summary

- [ ] Working directory clean (`git status`)
- [ ] No sensitive files staged
- [ ] All validation passes (lint, test, typecheck)
- [ ] No secrets detected (gitleaks)
- [ ] Rebased on latest main/develop
- [ ] Commit messages follow Conventional Commits
- [ ] Commits are atomic and meaningful
- [ ] Ready to push!
