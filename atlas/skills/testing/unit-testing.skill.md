---
name: unit-testing
description: >
  Rules and patterns for writing effective unit tests following Khorikov's four pillars:
  protection against regression, resistance to refactoring, fast feedback, and maintainability.
  Covers AAA structure, naming conventions, mock usage rules, and the distinction between
  the London (mockist) and Chicago (classicist) schools. Required by Dev Agents writing
  tests and QA agents reviewing test quality.
applies-to: [dev-agent, qa-agent, qa-test-subagent]
stack: none
layer: Testing
version: 1.0
---

# Unit Testing

<context>
This skill applies when writing or reviewing unit tests for any layer of the application.
A unit test verifies a single unit of behavior (not necessarily a single class) in
isolation from the filesystem, network, and database. It runs in milliseconds and
gives immediate, deterministic feedback. Source: Khorikov — "Unit Testing: Principles,
Practices, and Patterns."
</context>

<rules>
## The Four Pillars (Khorikov)
- Tests MUST provide protection against regression: they MUST exercise real code paths and verify outcomes that matter.
- Tests MUST be resistant to refactoring: a test MUST NOT fail when the production code is refactored without changing behavior.
- Tests MUST give fast feedback: a unit test MUST complete in under 100ms; tests requiring I/O are integration tests.
- Tests MUST be maintainable: the test code MUST be as readable and simple as possible; duplication in tests is acceptable.

## Structure — AAA
- Every test MUST follow Arrange → Act → Assert order.
- A test MUST have exactly one logical assertion (may have multiple `assert` calls verifying the same outcome).
- Arrange MUST NOT contain assertions; Act MUST be a single method call or operation.

## Naming
- Test names MUST follow the pattern: `MethodName_Scenario_ExpectedBehavior` (or `Given_When_Then`).
- Test names MUST describe behavior observable by a consumer, not implementation details.
- Avoid names like `Test1`, `TestHappyPath`, or `ShouldWork` — these convey no information.

## Mocking (London vs. Chicago)
- Mocks SHOULD only be used for dependencies that cross a process boundary: databases, file systems, message queues, HTTP clients, clocks.
- Domain logic and pure functions MUST NOT be mocked — test them directly (Chicago/classicist approach).
- A test that mocks the class under test is always wrong.
- Prefer fakes (in-memory implementations) over mocks for repositories and stores — they verify behavior rather than interactions.
- Mocks SHOULD be verified only for commands (void methods with side effects), not for queries.

## Test Isolation
- Each test MUST be independent — order of execution MUST NOT affect results.
- Shared mutable state between tests MUST NOT exist; use fresh instances per test.
- Tests MUST NOT depend on environment variables, system clock, or external services unless using explicit test doubles.

## Coverage
- Every public method's happy path MUST have at least one test.
- Every domain invariant violation (exception path) MUST have a corresponding test.
- Code coverage SHOULD exceed 80% for domain and application layers; 100% coverage is not a goal if it produces low-value tests.
</rules>

<patterns>
## AAA + Naming (C# / xUnit)

```csharp
public class OrderTests
{
    [Fact]
    public void Confirm_WhenOrderIsPending_ChangesStatusToConfirmed()
    {
        // Arrange
        var order = Order.Place(CustomerId.New(), [new OrderLine(ProductId.New(), 1, new Money(10, "USD"))]);

        // Act
        order.Confirm();

        // Assert
        Assert.Equal(OrderStatus.Confirmed, order.Status);
    }

    [Fact]
    public void Confirm_WhenOrderIsAlreadyConfirmed_ThrowsDomainException()
    {
        // Arrange
        var order = Order.Place(CustomerId.New(), [new OrderLine(ProductId.New(), 1, new Money(10, "USD"))]);
        order.Confirm();

        // Act & Assert
        Assert.Throws<DomainException>(() => order.Confirm());
    }
}
```

## Fake Repository (preferred over Mock)

```csharp
public class InMemoryOrderRepository : IOrderRepository
{
    private readonly Dictionary<OrderId, Order> _store = new();

    public Task<Order?> FindByIdAsync(OrderId id, CancellationToken ct) =>
        Task.FromResult(_store.TryGetValue(id, out var o) ? o : null);

    public Task SaveAsync(Order order, CancellationToken ct)
    {
        _store[order.Id] = order;
        return Task.CompletedTask;
    }
}

// Usage in test
[Fact]
public async Task PlaceOrder_ValidData_PersistsOrder()
{
    var repo = new InMemoryOrderRepository();
    var handler = new PlaceOrderHandler(repo, NullPublisher.Instance);

    var id = await handler.Handle(new PlaceOrderCommand(Guid.NewGuid(), _validLines), default);

    var order = await repo.FindByIdAsync(new OrderId(id), default);
    Assert.NotNull(order);
    Assert.Equal(OrderStatus.Pending, order.Status);
}
```

## Parameterized tests (xUnit Theory)

```csharp
[Theory]
[InlineData(0)]
[InlineData(-1)]
[InlineData(-100)]
public void Money_NegativeAmount_ThrowsDomainException(decimal amount)
{
    Assert.Throws<DomainException>(() => new Money(amount, "USD"));
}
```

## Python (pytest)

```python
import pytest
from domain.order import Order, OrderStatus
from domain.exceptions import DomainException

def make_order():
    return Order.place(customer_id="cust-1", items=[{"product_id": "p-1", "qty": 1, "price": 10.00}])

def test_confirm_pending_order_changes_status():
    # Arrange
    order = make_order()
    # Act
    order.confirm()
    # Assert
    assert order.status == OrderStatus.CONFIRMED

def test_confirm_already_confirmed_raises():
    order = make_order()
    order.confirm()
    with pytest.raises(DomainException, match="Only pending"):
        order.confirm()

@pytest.mark.parametrize("amount", [0, -1, -100])
def test_money_negative_amount_raises(amount):
    from domain.value_objects import Money
    with pytest.raises(DomainException):
        Money(amount=amount, currency="USD")
```
</patterns>

<anti-patterns>
- **Test implementation details**: `assert mock_repo.called_with(order)` — coupling to internal calls; the test breaks on harmless refactors.
- **Multiple unrelated assertions in one test**: verifying both status change and event dispatch in a single test — when it fails, unclear which behavior broke.
- **Shared state between tests (class-level fields)**: one test's setup bleeds into another; execution order starts mattering.
- **Mocking the class under test**: `mock_order_service = MagicMock(spec=OrderService)` then calling the mock — tests nothing about production code.
- **Tests with `Thread.Sleep` / `asyncio.sleep`**: slow, flaky, and indicate the code under test is not properly abstracted for testing.
- **Vague test names**: `test_order()`, `test_should_work()` — impossible to diagnose a failure without reading the test body.
- **Logic in tests** (if/else, loops for assertions): tests must be straight-line code; conditional logic hides branches that are never exercised.
- **Testing private methods directly**: tests private implementation rather than observable behavior; refactoring breaks tests for no behavioral change.
</anti-patterns>

<checklist>
- [ ] All tests follow Arrange → Act → Assert with a single logical Act.
- [ ] Test names follow `MethodName_Scenario_ExpectedBehavior` convention.
- [ ] Mocks used only for out-of-process dependencies (DB, HTTP, filesystem, clock).
- [ ] Domain logic tested without mocks (direct instantiation of domain objects).
- [ ] Each aggregate invariant has a test for the violation path.
- [ ] No shared mutable state between tests.
- [ ] No `Thread.Sleep` or equivalent in unit tests.
- [ ] Tests run in < 100ms each (if not, reclassify as integration tests).
</checklist>
