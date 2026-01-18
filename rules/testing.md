---
trigger: glob
globs: ["*_test.go", "*_test.py", "*.test.ts", "test_*.py", "conftest.py"]
---

# Testing Best Practices

## Philosophy

### Test-Driven Development (TDD)
For complex logic: **Red** → **Green** → **Refactor**

1. **Red**: Write a failing test that defines expected behavior
2. **Green**: Write minimal code to make the test pass
3. **Refactor**: Improve code while keeping tests green

### Test Pyramid

| Level | Proportion | Speed | Scope | Dependencies |
|-------|------------|-------|-------|--------------|
| **Unit** | 70% | < 100ms | Single function/class | Mocked |
| **Integration** | 20% | < 5s | Multiple components | Real (containerized) |
| **E2E** | 10% | < 30s | Full user flow | Real |

## Coverage Requirements - MANDATORY

| Code Type | Target | Enforcement |
|-----------|--------|-------------|
| Critical business logic | 90% | Block merge if below |
| Public API/interfaces | 80% | Block merge if below |
| Overall project | 70% | Warn if below |
| Utilities/helpers | 60% | Advisory |

**Measure with:**
- Go: `go test -coverprofile=coverage.out ./...`
- Python: `pytest --cov=src --cov-fail-under=70`
- Node: `vitest run --coverage`

## Naming Convention - MANDATORY

**Format:** `test_<unit>_<scenario>_<expected_result>`

| Language | Good | Bad |
|----------|------|-----|
| Python | `test_calculate_discount_premium_user_returns_10_percent` | `test_discount` |
| Go | `TestCalculateDiscount_PremiumUser_Returns10Percent` | `TestDiscount` |
| TypeScript | `it('returns 10% discount for premium user')` | `it('works')` |

## AAA Pattern - MANDATORY

Every test follows **Arrange-Act-Assert**:

```python
def test_transfer_sufficient_balance_succeeds():
    # Arrange - Set up test data and dependencies
    source = Account(balance=100)
    target = Account(balance=0)
    service = TransferService()

    # Act - Execute the code under test (ONE action)
    result = service.transfer(source, target, amount=50)

    # Assert - Verify the outcome
    assert result.success is True
    assert source.balance == 50
    assert target.balance == 50
```

## Test Doubles

| Type | Purpose | When to Use |
|------|---------|-------------|
| **Stub** | Returns canned answers | External API responses |
| **Mock** | Verifies interactions | Check if method was called |
| **Spy** | Records calls, uses real impl | Partial mocking |
| **Fake** | Working implementation | In-memory database |

```python
# Stub - Returns fixed data
user_repo = Mock()
user_repo.find_by_id.return_value = User(id=1, name="Test")

# Mock - Verify interaction
email_service = Mock()
service.create_user(data)
email_service.send_welcome.assert_called_once()

# Fake - In-memory implementation
class FakeUserRepository:
    def __init__(self):
        self.users = {}

    def save(self, user):
        self.users[user.id] = user
```

## Unit Test Guidelines

| Requirement | Reason |
|-------------|--------|
| **Isolated** | No network, filesystem, or DB calls |
| **Fast** | < 100ms per test |
| **Deterministic** | Same result every run |
| **Independent** | No shared state between tests |

**Avoid Non-Determinism:**
```python
# ❌ Bad - Non-deterministic
def test_random_selection():
    result = select_random_item(items)
    assert result in items

# ✅ Good - Seeded random
def test_random_selection():
    random.seed(42)
    result = select_random_item(items)
    assert result == items[3]

# ❌ Bad - Time-dependent
def test_token_expiry():
    token = create_token()
    assert not token.is_expired()

# ✅ Good - Frozen time
@freeze_time("2024-01-15 12:00:00")
def test_token_expiry():
    token = create_token(expires_in=3600)
    assert not token.is_expired()
```

## Integration Tests

**Use testcontainers for real dependencies:**

```python
# Python with testcontainers
@pytest.fixture(scope="module")
def postgres():
    with PostgresContainer("postgres:16") as pg:
        yield pg.get_connection_url()

def test_user_repository(postgres):
    repo = UserRepository(postgres)
    user = repo.save(User(name="Test"))
    assert repo.find_by_id(user.id).name == "Test"
```

```go
// Go with testcontainers
func TestUserRepository(t *testing.T) {
    ctx := context.Background()
    postgres, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: testcontainers.ContainerRequest{
            Image:        "postgres:16",
            ExposedPorts: []string{"5432/tcp"},
            WaitingFor:   wait.ForListeningPort("5432/tcp"),
        },
        Started: true,
    })
    defer postgres.Terminate(ctx)

    // Test with real database
}
```

**Cleanup state after each test:**
- Truncate tables
- Use transactions and rollback
- Use unique test data per test

## Table-Driven Tests

**Recommended for multiple scenarios:**

```go
func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name    string
        email   string
        wantErr bool
    }{
        {"valid email", "user@example.com", false},
        {"missing @", "userexample.com", true},
        {"missing domain", "user@", true},
        {"empty", "", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := ValidateEmail(tt.email)
            if (err != nil) != tt.wantErr {
                t.Errorf("ValidateEmail(%q) error = %v, wantErr %v", tt.email, err, tt.wantErr)
            }
        })
    }
}
```

```python
@pytest.mark.parametrize("email,valid", [
    ("user@example.com", True),
    ("userexample.com", False),
    ("user@", False),
    ("", False),
])
def test_validate_email(email, valid):
    assert validate_email(email) == valid
```

## Anti-Patterns to Avoid

| Anti-Pattern | Problem | Solution |
|-------------|---------|----------|
| Testing implementation | Brittle tests | Test behavior/output |
| Shared mutable state | Flaky tests | Fresh setup per test |
| Sleep/time delays | Slow, unreliable | Use mocks, async waits |
| Testing private methods | Over-specification | Test through public API |
| Multiple unrelated assertions | Unclear failures | One logical concept per test |

## CI/CD Integration

Tests **MUST** run in CI pipeline:

```yaml
# GitHub Actions
- name: Run tests
  run: |
    go test -race -coverprofile=coverage.out ./...

- name: Check coverage
  run: |
    coverage=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | tr -d '%')
    if (( $(echo "$coverage < 70" | bc -l) )); then
      echo "Coverage $coverage% is below 70%"
      exit 1
    fi
```

**Pipeline fails if:**
- Any test fails
- Coverage drops below threshold
- Flaky tests detected (run 3x)
