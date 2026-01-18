# Test

Generate comprehensive tests for the selected code following the **Test Pyramid** and **AAA pattern**.

## When to Write Tests

| Scenario | Test Type | Priority |
|----------|-----------|----------|
| Business logic | Unit | HIGH |
| Public API/Interface | Unit + Integration | HIGH |
| Data transformations | Unit | HIGH |
| External integrations | Integration | MEDIUM |
| User workflows | E2E | LOW |

**When NOT to test:**
- Trivial getters/setters with no logic
- Framework/library code (already tested)
- Generated code (protobuf, OpenAPI)
- Simple delegation methods

## Coverage Requirements

| Test Category | What to Cover |
|--------------|---------------|
| **Happy Path** | Normal flow, valid inputs, main use cases |
| **Edge Cases** | Boundaries (0, -1, MAX), empty collections, null/undefined |
| **Error Cases** | Invalid inputs, validation errors, expected exceptions |
| **Concurrency** | Race conditions, deadlocks (if applicable) |

### Boundary Value Analysis

```python
# For a function that accepts age 0-120
def test_age_boundaries():
    # Valid boundaries
    assert is_valid_age(0) == True
    assert is_valid_age(120) == True

    # Invalid boundaries
    assert is_valid_age(-1) == False
    assert is_valid_age(121) == False
```

## Test Doubles

| Type | Purpose | When to Use |
|------|---------|-------------|
| **Stub** | Returns canned data | External API responses |
| **Mock** | Verifies interactions | Checking method was called |
| **Spy** | Records calls, delegates to real | Partial mocking |
| **Fake** | Working implementation | In-memory database |

```python
# Stub - Returns fixed data
@pytest.fixture
def stub_user_repo():
    repo = Mock()
    repo.find_by_id.return_value = User(id=1, name="Test")
    return repo

# Mock - Verify interaction
def test_sends_welcome_email(mock_email_service):
    service.create_user(user_data)
    mock_email_service.send.assert_called_once_with(
        to="user@test.com",
        template="welcome"
    )

# Fake - In-memory implementation
class FakeUserRepository:
    def __init__(self):
        self.users = {}

    def save(self, user):
        self.users[user.id] = user

    def find_by_id(self, id):
        return self.users.get(id)
```

## Frameworks & Commands

| Language | Framework | Run | Coverage |
|----------|-----------|-----|----------|
| Go | testing + testify | `go test ./... -v` | `go test -cover ./...` |
| Python | pytest | `pytest -v` | `pytest --cov=src` |
| TypeScript | vitest | `npm test` | `npm run test:coverage` |
| Terraform | terraform test | `terraform test` | N/A |

## Test Structure

### Naming Convention

**Format:** `test_<function>_<scenario>_<expected>`

| Language | Example |
|----------|---------|
| Python | `test_calculate_discount_premium_user_returns_10_percent` |
| Go | `TestCalculateDiscount_PremiumUser_Returns10Percent` |
| TypeScript | `it('should return 10% discount for premium user')` |

### AAA Pattern - MANDATORY

```python
def test_transfer_sufficient_balance_succeeds():
    # Arrange - Setup test data and dependencies
    source = Account(balance=100)
    target = Account(balance=0)

    # Act - Execute the function under test
    result = transfer(source, target, amount=50)

    # Assert - Verify the outcome
    assert result.success == True
    assert source.balance == 50
    assert target.balance == 50
```

### Table-Driven Tests (Recommended)

```go
func TestCalculateDiscount(t *testing.T) {
    tests := []struct {
        name     string
        userTier string
        amount   float64
        want     float64
    }{
        {"premium user gets 10%", "premium", 100, 90},
        {"standard user gets 5%", "standard", 100, 95},
        {"no discount for basic", "basic", 100, 100},
        {"zero amount", "premium", 0, 0},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := CalculateDiscount(tt.userTier, tt.amount)
            assert.Equal(t, tt.want, got)
        })
    }
}
```

```python
@pytest.mark.parametrize("user_tier,amount,expected", [
    ("premium", 100, 90),
    ("standard", 100, 95),
    ("basic", 100, 100),
    ("premium", 0, 0),
])
def test_calculate_discount(user_tier, amount, expected):
    result = calculate_discount(user_tier, amount)
    assert result == expected
```

## Anti-Patterns to Avoid

| Anti-Pattern | Problem | Solution |
|-------------|---------|----------|
| Testing implementation | Brittle tests | Test behavior/output |
| Shared mutable state | Flaky tests | Fresh setup per test |
| Sleep/time delays | Slow, unreliable | Use async/await, mocks |
| Testing private methods | Over-specification | Test through public API |
| Multiple assertions | Unclear failures | One logical assert per test |
| Hardcoded test data | Hard to maintain | Use fixtures/factories |

```python
# ❌ Anti-pattern: Testing implementation details
def test_user_service():
    service.create_user(data)
    assert service._internal_cache["user_1"] is not None  # Private!

# ✅ Test behavior
def test_user_service():
    service.create_user(data)
    user = service.get_user("user_1")
    assert user.name == "Test"
```

## Test Independence

```python
# ✅ Each test is independent
class TestUserService:
    @pytest.fixture(autouse=True)
    def setup(self):
        self.db = FakeDatabase()
        self.service = UserService(self.db)

    def test_create_user(self):
        # Fresh service instance for each test
        self.service.create(user_data)
        assert self.db.count() == 1

    def test_delete_user(self):
        # Not affected by test_create_user
        self.service.create(user_data)
        self.service.delete(user_id)
        assert self.db.count() == 0
```

## Instructions

1. **Detect** existing test framework and patterns in the codebase
2. **Follow** existing naming conventions and file structure
3. **Generate** tests using AAA pattern
4. **Include** happy path, edge cases, and error cases
5. **Use** table-driven tests for multiple scenarios
6. **Mock** only external dependencies (DB, APIs, filesystem)
7. **Verify** tests are independent (no shared state)
