# Refactor

Refactor the selected code while **maintaining existing behavior**. Focus on improving readability, maintainability, and testability.

## When to Refactor

| Trigger | Threshold | Action |
|---------|-----------|--------|
| Long function | >50 lines | Extract sub-functions |
| Deep nesting | >3 levels | Early returns, extract logic |
| Duplicated code | >10 lines | Extract shared function |
| God class | >300 lines | Split by responsibility |
| High cyclomatic complexity | >10 | Simplify conditionals |
| Feature envy | Frequent external calls | Move method to owner |

**When NOT to refactor:**
- Code under active development (wait for stability)
- No test coverage (add tests first)
- Working legacy code with no changes planned
- Performance-critical sections (measure first)

## Code Smells & Solutions

### 1. Long Method → Extract Function

```python
# ❌ Before: 60+ lines doing multiple things
def process_order(order):
    # Validate order (15 lines)
    if not order.items:
        raise ValueError("Empty order")
    for item in order.items:
        if item.quantity <= 0:
            raise ValueError("Invalid quantity")
    # ... more validation

    # Calculate totals (20 lines)
    subtotal = sum(item.price * item.quantity for item in order.items)
    tax = subtotal * 0.1
    # ... more calculations

    # Process payment (25 lines)
    # ... payment logic

# ✅ After: Clear, focused functions
def process_order(order):
    validate_order(order)
    totals = calculate_totals(order)
    process_payment(order, totals)

def validate_order(order):
    if not order.items:
        raise ValueError("Empty order")
    for item in order.items:
        if item.quantity <= 0:
            raise ValueError("Invalid quantity")

def calculate_totals(order):
    subtotal = sum(item.price * item.quantity for item in order.items)
    return OrderTotals(subtotal=subtotal, tax=subtotal * TAX_RATE)
```

### 2. Deep Nesting → Early Returns

```python
# ❌ Before: Deep nesting
def process_user(user):
    if user:
        if user.is_active:
            if user.has_permission("admin"):
                if user.department == "engineering":
                    return perform_action(user)
    return None

# ✅ After: Guard clauses
def process_user(user):
    if not user:
        return None
    if not user.is_active:
        return None
    if not user.has_permission("admin"):
        return None
    if user.department != "engineering":
        return None

    return perform_action(user)
```

### 3. Magic Numbers → Named Constants

```python
# ❌ Before
if response.status_code == 429:
    time.sleep(60)
    retry_count = 3

# ✅ After
HTTP_TOO_MANY_REQUESTS = 429
RATE_LIMIT_DELAY_SECONDS = 60
MAX_RETRY_ATTEMPTS = 3

if response.status_code == HTTP_TOO_MANY_REQUESTS:
    time.sleep(RATE_LIMIT_DELAY_SECONDS)
    retry_count = MAX_RETRY_ATTEMPTS
```

### 4. Conditional Complexity → Strategy/Polymorphism

```python
# ❌ Before: Growing if/elif chain
def calculate_shipping(order):
    if order.shipping_type == "standard":
        return order.weight * 0.5
    elif order.shipping_type == "express":
        return order.weight * 1.5 + 10
    elif order.shipping_type == "overnight":
        return order.weight * 3.0 + 25
    # ... more types

# ✅ After: Strategy pattern
SHIPPING_STRATEGIES = {
    "standard": lambda w: w * 0.5,
    "express": lambda w: w * 1.5 + 10,
    "overnight": lambda w: w * 3.0 + 25,
}

def calculate_shipping(order):
    strategy = SHIPPING_STRATEGIES.get(order.shipping_type)
    if not strategy:
        raise ValueError(f"Unknown shipping type: {order.shipping_type}")
    return strategy(order.weight)
```

### 5. Primitive Obsession → Value Objects

```python
# ❌ Before: Primitives everywhere
def create_user(email: str, phone: str, zip_code: str):
    if "@" not in email:
        raise ValueError("Invalid email")
    if len(phone) != 10:
        raise ValueError("Invalid phone")
    # ...

# ✅ After: Value objects with validation
@dataclass(frozen=True)
class Email:
    value: str

    def __post_init__(self):
        if "@" not in self.value:
            raise ValueError(f"Invalid email: {self.value}")

@dataclass(frozen=True)
class Phone:
    value: str

    def __post_init__(self):
        if len(self.value) != 10:
            raise ValueError(f"Invalid phone: {self.value}")

def create_user(email: Email, phone: Phone, zip_code: ZipCode):
    # Validation already done by value objects
    pass
```

## Language-Specific Refactorings

### Go
| Pattern | Refactoring |
|---------|-------------|
| Error handling | Use `errors.Join()`, `fmt.Errorf("...: %w", err)` |
| Multiple returns | Use named result struct |
| Interface bloat | Split into smaller interfaces |

### Python
| Pattern | Refactoring |
|---------|-------------|
| Dict access | Use `dataclass` or `TypedDict` |
| Optional handling | Use `if x is not None` or `x or default` |
| List comprehension | Extract to function if >1 line |

### TypeScript
| Pattern | Refactoring |
|---------|-------------|
| Type assertions | Use type guards |
| Callback hell | Use async/await |
| Optional chaining | Replace nested ifs with `?.` and `??` |

## Constraints - NON-NEGOTIABLE

| Rule | Description |
|------|-------------|
| ✅ **Same Behavior** | Same inputs must produce same outputs |
| ✅ **Tests Pass** | All existing tests must pass after refactor |
| ✅ **Public API Stable** | Do not break external interfaces |
| ✅ **Incremental** | One refactor at a time, verify after each |
| ❌ **No New Dependencies** | Unless absolutely necessary and justified |
| ❌ **No Premature Optimization** | Clarity over micro-optimization |

## Refactoring Process

### 1. Pre-Refactor Checklist
- [ ] Tests exist and pass
- [ ] Understand current behavior
- [ ] Identify specific smells to address
- [ ] Plan incremental steps

### 2. Execute Refactoring
```
For each refactoring step:
  1. Make ONE change
  2. Run tests
  3. If tests fail → Revert and analyze
  4. If tests pass → Commit (optional) and continue
```

### 3. Measure Improvement

| Metric | Before | After | Target |
|--------|--------|-------|--------|
| Lines per function | ? | ? | <50 |
| Cyclomatic complexity | ? | ? | <10 |
| Nesting depth | ? | ? | <3 |
| Test coverage | ? | ? | Maintained |

## Report Format

```markdown
## Refactoring Summary

**Target:** `src/services/order_service.py`
**Smells Identified:** Long Method, Magic Numbers, Deep Nesting

### Changes Made

1. **Extract Function**: `process_order()` → `validate_order()` + `calculate_totals()` + `process_payment()`
   - Lines: 85 → 25 (main function)

2. **Named Constants**: Replaced 5 magic numbers with constants
   - `429` → `HTTP_TOO_MANY_REQUESTS`
   - `60` → `RATE_LIMIT_DELAY_SECONDS`

3. **Early Returns**: Reduced nesting from 4 levels to 1

### Verification
- ✅ All 47 tests pass
- ✅ Coverage maintained at 82%
- ✅ No public API changes
```
