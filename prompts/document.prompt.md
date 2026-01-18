# Document

Generate comprehensive documentation for the selected code or project. Focus on **"Why"** over **"What"**.

## Documentation Types

### 1. Code Comments (Docstrings)

**When to Document:**
| Scenario | Example |
|----------|---------|
| Non-obvious business logic | Tax calculation rules |
| Design decisions | Why we chose Strategy over Factory |
| Workarounds | Link to issue/ticket |
| Public APIs | Parameters, returns, exceptions |
| Complex algorithms | Time/space complexity |
| Security considerations | Why input is sanitized here |

**When NOT to Document:**
- Self-explanatory code
- Trivial getters/setters
- Comments that repeat the code
- Implementation details that may change

```python
# ❌ Bad - repeats code
# Increment counter by 1
counter += 1

# ❌ Bad - obvious
# Loop through users
for user in users:

# ✅ Good - explains WHY
# Skip first 2 rows: header + metadata per CSV spec v2.1
start_row = 2

# ✅ Good - documents edge case
# Handle legacy accounts created before 2020 migration
# which may have null email addresses (see JIRA-1234)
if user.email is None:
```

### 2. Docstring Formats by Language

#### Python (Google Style)
```python
def calculate_discount(user: User, amount: float) -> float:
    """Calculate discount based on user tier and purchase amount.

    Applies tiered discount rates based on user membership level.
    Premium users get 10%, standard users get 5%.

    Args:
        user: The user making the purchase.
        amount: The purchase amount before discount.

    Returns:
        The discounted amount.

    Raises:
        ValueError: If amount is negative.

    Example:
        >>> calculate_discount(premium_user, 100.0)
        90.0
    """
```

#### Go (GoDoc)
```go
// CalculateDiscount returns the discounted amount based on user tier.
//
// Premium users receive 10% discount, standard users receive 5%.
// Returns an error if amount is negative.
//
// Example:
//
//	discounted, err := CalculateDiscount(user, 100.0)
//	if err != nil {
//	    log.Fatal(err)
//	}
func CalculateDiscount(user *User, amount float64) (float64, error) {
```

#### TypeScript (TSDoc)
```typescript
/**
 * Calculates discount based on user tier and purchase amount.
 *
 * @param user - The user making the purchase
 * @param amount - The purchase amount before discount
 * @returns The discounted amount
 * @throws {Error} If amount is negative
 *
 * @example
 * ```typescript
 * const discounted = calculateDiscount(premiumUser, 100);
 * // Returns 90
 * ```
 */
function calculateDiscount(user: User, amount: number): number {
```

### 3. README Standard

```markdown
# Project Name

Brief description of what this project does and why it exists.

## Features

- Feature 1: Description
- Feature 2: Description

## Requirements

- Node.js >= 20
- PostgreSQL >= 15

## Installation

```bash
npm install
cp .env.example .env
npm run db:migrate
```

## Usage

```bash
# Development
npm run dev

# Production
npm run build && npm start
```

## Configuration

| Variable | Description | Required | Default |
|----------|-------------|----------|---------|
| `DATABASE_URL` | PostgreSQL connection string | Yes | - |
| `PORT` | Server port | No | `3000` |
| `LOG_LEVEL` | Logging verbosity | No | `info` |

## API Reference

See [API Documentation](./docs/api.md) for detailed endpoint documentation.

## Development

```bash
# Run tests
npm test

# Run linter
npm run lint

# Generate docs
npm run docs
```

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines.

## License

MIT
```

### 4. ADR (Architecture Decision Record)

```markdown
# ADR-001: Use PostgreSQL for Primary Database

## Status
Accepted

## Context
We need a database for storing user data and transactions.
Requirements: ACID compliance, JSON support, strong ecosystem.

## Decision
Use PostgreSQL 15+ as the primary database.

## Consequences

### Positive
- Strong ACID guarantees
- Excellent JSON/JSONB support
- Wide tooling ecosystem
- Team familiarity

### Negative
- Requires managed service or operational expertise
- Horizontal scaling requires additional setup (Citus, read replicas)

## Alternatives Considered

| Option | Pros | Cons | Decision |
|--------|------|------|----------|
| MySQL | Familiar, fast | Weaker JSON support | Rejected |
| MongoDB | Flexible schema | No ACID by default | Rejected |
| CockroachDB | Distributed | Less mature | Deferred |
```

### 5. API Documentation

```markdown
## POST /api/v1/users

Create a new user account.

### Authentication
Required: `Bearer` token with `users:write` scope

### Request Headers
| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | Yes | Bearer token |
| `Content-Type` | Yes | `application/json` |
| `X-Request-ID` | No | Correlation ID for tracing |

### Request Body
```json
{
  "email": "user@example.com",
  "name": "John Doe",
  "role": "member"
}
```

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `email` | string | Yes | Valid email format |
| `name` | string | Yes | 1-100 characters |
| `role` | string | No | `member`, `admin` |

### Responses

#### 201 Created
```json
{
  "id": "usr_abc123",
  "email": "user@example.com",
  "name": "John Doe",
  "role": "member",
  "created_at": "2024-01-15T10:30:00Z"
}
```

#### 400 Bad Request
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid email format",
    "field": "email"
  }
}
```

#### 409 Conflict
```json
{
  "error": {
    "code": "DUPLICATE_EMAIL",
    "message": "User with this email already exists"
  }
}
```
```

### 6. CHANGELOG (Keep a Changelog)

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- New `/api/v1/users/bulk` endpoint for batch operations

### Changed
- Improved error messages for validation failures

## [1.2.0] - 2024-01-15

### Added
- User role management (#123)
- Rate limiting on authentication endpoints (#125)

### Fixed
- Memory leak in connection pool (#124)

### Security
- Updated `lodash` to fix prototype pollution vulnerability

## [1.1.0] - 2024-01-01

### Added
- Initial release with user CRUD operations
```

## Documentation Tools

| Project Type | Tool | Command |
|-------------|------|---------|
| Terraform | terraform-docs | `terraform-docs markdown table .` |
| Go | godoc / pkgsite | `go doc -all` |
| Python | Sphinx / mkdocs | `mkdocs serve` |
| TypeScript | TypeDoc | `npx typedoc` |
| OpenAPI | Swagger / Redoc | `npx @redocly/cli build-docs` |

## When to Update Documentation

| Event | Action |
|-------|--------|
| New feature | Add to README, API docs, CHANGELOG |
| Breaking change | Update migration guide, bump major version |
| Bug fix | Add to CHANGELOG |
| Architectural change | Create/update ADR |
| Configuration change | Update README Configuration section |

## Instructions

1. **Analyze** existing documentation style in the project
2. **Match** the format and voice of existing docs
3. **Document** the "Why", not just the "What"
4. **Include** practical examples that can be copy-pasted
5. **Verify** accuracy of any code examples
6. **Update** related docs (README, CHANGELOG) if needed
