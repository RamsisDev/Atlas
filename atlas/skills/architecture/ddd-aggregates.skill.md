---
name: ddd-aggregates
description: >
  Rules and patterns for designing Domain-Driven Design aggregates, aggregate roots,
  value objects, and domain events following Vernon's IDDD and Evans' DDD Reference.
  Use this skill when implementing or reviewing domain entities, enforcing invariants,
  or designing transactional boundaries. Required by Domain-layer Dev Agents, Spec
  Writers (D01), and the Domain Expert agent.
applies-to: [dev-agent, spec-writer, domain-expert, architecture-analyst]
stack: none
layer: domain
version: 1.0
---

# DDD Aggregates

<context>
This skill applies when designing or implementing the Domain layer of a DDD project.
It covers Aggregate, Aggregate Root, Entity, Value Object, Repository interface, and
Domain Event patterns. The core constraint is transactional consistency: each aggregate
is the unit of consistency and the boundary of a single transaction.
</context>

<rules>
## Aggregate Boundaries
- Each business transaction MUST modify at most one aggregate.
- Aggregates MUST reference other aggregates by identity (ID) only, never by object reference.
- Aggregate size SHOULD be kept small — model invariants first, then expand only if necessary.
- An aggregate MUST enforce all its own invariants; no external service may do so on its behalf.

## Aggregate Root
- Every aggregate MUST have exactly one Aggregate Root — the only entry point for external modifications.
- External code MUST NOT hold a reference to any non-root entity within an aggregate.
- The Aggregate Root MUST control the lifecycle of all child entities.
- State-changing behavior MUST be expressed as methods on the Aggregate Root, not as property setters.

## Entities
- Entities MUST be identified by a unique, stable identity (ID), not by their attribute values.
- Entity identity MUST be set at construction and MUST NOT change over the entity's lifetime.
- Entities SHOULD encapsulate state; public setters on mutable fields SHALL NOT exist.

## Value Objects
- Value Objects MUST be immutable — no setters, no mutation methods.
- Value Objects MUST implement structural equality (equals by value, not by reference).
- Conceptually distinct primitives (Money, Email, OrderId) SHOULD be wrapped in Value Objects, not left as raw primitives.
- Value Objects SHOULD be validated in their constructor; an invalid value object MUST NOT be constructed.

## Domain Events
- Significant state changes in an aggregate MUST raise a Domain Event.
- Domain Events MUST be named in the past tense (e.g., `OrderPlaced`, `PaymentFailed`).
- Domain Events MUST be collected inside the aggregate root and dispatched by the Application layer after saving.
- Domain Events MUST NOT trigger side effects directly inside the aggregate.

## Repository Interface
- Repository interfaces MUST be defined in the Domain layer.
- Each aggregate root SHOULD have exactly one repository.
- Repositories MUST NOT expose query methods that return partial aggregate state (no projections).
- Repositories MUST return a fully-reconstituted aggregate or null/Option — never a partially-loaded object.
</rules>

<patterns>
## Value Object (C#)

```csharp
public sealed class Money : IEquatable<Money>
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0) throw new DomainException("Amount cannot be negative.");
        if (string.IsNullOrWhiteSpace(currency)) throw new DomainException("Currency is required.");
        Amount = amount;
        Currency = currency.ToUpperInvariant();
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency) throw new DomainException("Cannot add different currencies.");
        return new Money(Amount + other.Amount, Currency);
    }

    public bool Equals(Money? other) => other is not null && Amount == other.Amount && Currency == other.Currency;
    public override bool Equals(object? obj) => Equals(obj as Money);
    public override int GetHashCode() => HashCode.Combine(Amount, Currency);
}
```

## Aggregate Root with Domain Events (C#)

```csharp
public class Order : AggregateRoot<OrderId>
{
    private readonly List<OrderLine> _lines = [];
    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();

    private Order() { } // EF Core / reconstitution

    public static Order Place(CustomerId customerId, IEnumerable<OrderLine> lines)
    {
        var order = new Order
        {
            Id = OrderId.New(),
            CustomerId = customerId,
            Status = OrderStatus.Pending
        };
        foreach (var line in lines) order._lines.Add(line);
        order.Raise(new OrderPlaced(order.Id, customerId));
        return order;
    }

    public void Confirm()
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("Only pending orders can be confirmed.");
        Status = OrderStatus.Confirmed;
        Raise(new OrderConfirmed(Id));
    }
}

// AggregateRoot base — collects domain events
public abstract class AggregateRoot<TId>
{
    private readonly List<IDomainEvent> _events = [];
    public TId Id { get; protected set; } = default!;
    public IReadOnlyList<IDomainEvent> DomainEvents => _events.AsReadOnly();
    protected void Raise(IDomainEvent evt) => _events.Add(evt);
    public void ClearEvents() => _events.Clear();
}
```

## Repository Interface (Domain layer)

```csharp
// Lives in Domain.Interfaces — no EF Core imports here
public interface IOrderRepository
{
    Task<Order?> FindByIdAsync(OrderId id, CancellationToken ct);
    Task SaveAsync(Order order, CancellationToken ct);
}
```

## Cross-Aggregate Reference by ID

```csharp
// CORRECT — Order references Customer by ID only
public class Order : AggregateRoot<OrderId>
{
    public CustomerId CustomerId { get; private set; }  // ID only
}

// WRONG — object reference to another aggregate
public class Order : AggregateRoot<OrderId>
{
    public Customer Customer { get; private set; }  // violates aggregate boundary
}
```
</patterns>

<anti-patterns>
- **Anemic domain model**: aggregates with only public getters/setters and no behavior; business logic lives in application services — domain becomes a data bag, invariants are never enforced.
- **Large aggregates**: loading the entire `Customer` with all their `Orders` and `Addresses` in a single transaction causes lock contention and performance issues.
- **Public setters on entities**: bypasses invariant enforcement; any code can put the aggregate in an invalid state.
- **Domain events dispatched synchronously inside aggregate**: couples domain logic to side effects (email sending, external calls) that may fail.
- **Repository returns IQueryable**: leaks persistence concerns into the domain; callers can filter partial state, bypassing aggregate reconstitution rules.
- **Raw primitive obsession**: using `string email`, `decimal amount`, `Guid orderId` instead of `Email`, `Money`, `OrderId` — misuse at callsites becomes a runtime bug.
- **Constructor-less value object mutation**: creating a modified copy through property mutation instead of factory methods — breaks immutability guarantee.
</anti-patterns>

<checklist>
- [ ] Each aggregate has a single, clearly identified Aggregate Root.
- [ ] No cross-aggregate object references exist (only ID references).
- [ ] Value Objects are immutable and validate in constructor.
- [ ] All state-changing operations are methods (not public setters).
- [ ] Domain Events are raised inside the aggregate and collected, not dispatched.
- [ ] Repository interface is declared in the Domain layer with no infrastructure imports.
- [ ] Each aggregate invariant has a corresponding unit test that asserts the exception on violation.
</checklist>
