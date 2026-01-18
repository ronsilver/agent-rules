---
trigger: glob
globs: ["*.py", "*.js", "*.ts", "*.go", "*.java", "*.cs", "*.rb"]
---

# SOLID Principles - Best Practices

## Summary

| Principle | Description | Benefit |
|-----------|-------------|---------|
| **S** - Single Responsibility | One class = One reason to change | Maintainability |
| **O** - Open/Closed | Open for extension, closed for modification | Extensibility |
| **L** - Liskov Substitution | Subtypes must be substitutable | Correctness |
| **I** - Interface Segregation | Small, specific interfaces | Flexibility |
| **D** - Dependency Inversion | Depend on abstractions, not concretions | Testability |

## S - Single Responsibility Principle (SRP)
**"A class should have one, and only one, reason to change."**

```python
# ❌ Bad - God Class (3 responsibilities)
class UserService:
    def create_user(self, data):
        # Validation logic
        if not data.get("email"):
            raise ValueError("Email required")
        # Persistence logic
        self.db.save(data)
        # Notification logic
        self.smtp.send_welcome_email(data["email"])

# ✅ Good - Single responsibility each
class UserValidator:
    def validate(self, data: dict) -> None: ...

class UserRepository:
    def save(self, user: User) -> User: ...

class NotificationService:
    def send_welcome(self, email: str) -> None: ...

class UserService:
    def __init__(self, validator, repo, notifier):
        self.validator = validator
        self.repo = repo
        self.notifier = notifier

    def create_user(self, data):
        self.validator.validate(data)
        user = self.repo.save(User(**data))
        self.notifier.send_welcome(user.email)
        return user
```

```go
// Go example
// ❌ Bad - Mixed concerns
type OrderHandler struct{}
func (h *OrderHandler) Handle(w http.ResponseWriter, r *http.Request) {
    // Parse request, validate, save to DB, send email, write response
}

// ✅ Good - Separated
type OrderValidator struct{}
type OrderRepository struct{}
type OrderNotifier struct{}
type OrderHandler struct {
    validator *OrderValidator
    repo      *OrderRepository
    notifier  *OrderNotifier
}
```

## O - Open/Closed Principle (OCP)
**"Open for extension, closed for modification."**

Use **Strategy Pattern** or **Interfaces** to add behavior without modifying existing code.

```python
# ❌ Bad - Must modify for every new payment type
def process_payment(payment_type: str, amount: float):
    if payment_type == 'credit':
        return process_credit(amount)
    elif payment_type == 'paypal':
        return process_paypal(amount)
    elif payment_type == 'crypto':  # New type = code change
        return process_crypto(amount)

# ✅ Good - Open for extension via new classes
from abc import ABC, abstractmethod

class PaymentProcessor(ABC):
    @abstractmethod
    def process(self, amount: float) -> bool: ...

class CreditCardProcessor(PaymentProcessor):
    def process(self, amount: float) -> bool: ...

class PayPalProcessor(PaymentProcessor):
    def process(self, amount: float) -> bool: ...

# Adding crypto = new class, no modification to existing code
class CryptoProcessor(PaymentProcessor):
    def process(self, amount: float) -> bool: ...

def process_payment(processor: PaymentProcessor, amount: float):
    return processor.process(amount)  # Works for ANY processor
```

## L - Liskov Substitution Principle (LSP)
**"Subtypes must be substitutable for their base types."**

```python
# ❌ Bad - Violates LSP (Square can't substitute Rectangle)
class Rectangle:
    def set_width(self, w): self.width = w
    def set_height(self, h): self.height = h

class Square(Rectangle):
    def set_width(self, w):
        self.width = w
        self.height = w  # Side effect! Breaks expectations

# ✅ Good - Separate types or use composition
class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...

class Rectangle(Shape):
    def __init__(self, width: float, height: float): ...

class Square(Shape):
    def __init__(self, side: float): ...
```

**LSP Violations:**
- Throwing `NotImplementedError` in subclass methods
- Returning different types than parent
- Requiring additional preconditions
- Having side effects parent doesn't have

## I - Interface Segregation Principle (ISP)
**"Clients should not be forced to depend on interfaces they do not use."**

```go
// ❌ Bad - Fat interface forces unnecessary implementations
type Worker interface {
    Work()
    Eat()
    Sleep()
    TakeVacation()
}

type Robot struct{}  // Robots don't eat, sleep, or vacation!

// ✅ Good - Segregated interfaces
type Workable interface { Work() }
type Eatable interface { Eat() }
type Sleepable interface { Sleep() }

type Human struct{}
func (h Human) Work() { ... }
func (h Human) Eat() { ... }
func (h Human) Sleep() { ... }

type Robot struct{}
func (r Robot) Work() { ... }  // Only implements what it needs
```

```python
# Python - Use Protocol for structural typing
from typing import Protocol

class Readable(Protocol):
    def read(self) -> bytes: ...

class Writable(Protocol):
    def write(self, data: bytes) -> None: ...

# Functions declare exactly what they need
def copy_data(src: Readable, dst: Writable) -> None:
    dst.write(src.read())
```

## D - Dependency Inversion Principle (DIP)
**"Depend on abstractions, not concretions."**

```python
# ❌ Bad - High-level depends on low-level concrete class
class OrderService:
    def __init__(self):
        self.db = MySQLDatabase()  # Tight coupling!
        self.mailer = SMTPMailer()  # Can't test without real SMTP!

# ✅ Good - Depend on abstractions, inject dependencies
from abc import ABC, abstractmethod

class Database(ABC):
    @abstractmethod
    def save(self, entity) -> None: ...

class Mailer(ABC):
    @abstractmethod
    def send(self, to: str, subject: str, body: str) -> None: ...

class OrderService:
    def __init__(self, db: Database, mailer: Mailer):
        self.db = db
        self.mailer = mailer

# Production
service = OrderService(MySQLDatabase(), SMTPMailer())

# Testing - Easy to mock!
service = OrderService(FakeDatabase(), FakeMailer())
```

```go
// Go - Use interfaces at consumer side
type Repository interface {
    Save(ctx context.Context, user *User) error
}

type UserService struct {
    repo Repository  // Accepts any Repository implementation
}

func NewUserService(repo Repository) *UserService {
    return &UserService{repo: repo}
}
```

## Refactoring Triggers

| Code Smell | SOLID Violation | Refactoring |
|------------|-----------------|-------------|
| God Class (>300 lines) | SRP | Extract classes by responsibility |
| Shotgun Surgery | SRP | Group related changes together |
| Switch on type | OCP | Strategy pattern, polymorphism |
| NotImplementedError | LSP | Redesign hierarchy |
| Fat Interface | ISP | Split into smaller interfaces |
| `new` in constructor | DIP | Inject dependencies |
| Circular dependencies | All | Introduce abstraction layer |

## When NOT to Apply SOLID

- **Simple scripts**: Over-engineering for one-off scripts
- **Prototypes**: Speed matters more than architecture
- **Trivial code**: Don't abstract a single-use function
- **Performance critical**: Sometimes direct calls are faster

**Rule of thumb**: Apply SOLID when code will be maintained, extended, or tested.
