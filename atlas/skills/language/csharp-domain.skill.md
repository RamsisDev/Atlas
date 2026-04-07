---
name: csharp-domain
description: >
  C# .NET 8+ patterns for implementing the Domain layer: entities, value objects,
  aggregate roots, domain events, and domain services following Microsoft's .NET
  Architecture Guides (eShopOnContainers) and Khorikov's DDD in Practice. Use this
  skill for any task that creates or modifies domain model classes. Required by Dev
  Agents working on Domain-layer tasks.
applies-to: [dev-agent]
stack: dotnet
layer: domain
version: 1.0
---

# C# Domain Layer

<context>
This skill applies when implementing C# classes in the Domain layer of a Clean
Architecture project. It covers the concrete .NET 8 implementation of DDD patterns:
entity base class with domain events, value object with structural equality, aggregate
root lifecycle, and domain service interfaces. All code in this layer MUST be
framework-free — no EF Core, no ASP.NET, no MediatR imports.
</context>

<rules>
## Framework Freedom
- Domain classes MUST NOT import Entity Framework Core, ASP.NET, MediatR, or any infrastructure library.
- Domain classes MUST NOT inherit from any ORM base class.
- Domain project MUST reference only: `System.*` BCL namespaces and other Domain projects.

## Entities and Aggregate Roots
- Entities MUST have a strongly-typed ID (e.g., `OrderId`, `CustomerId`) — not a raw `Guid` or `int`.
- Strongly-typed IDs MUST be readonly record structs with a single `Value` property.
- Public setters on entity properties MUST NOT exist — all mutations go through domain methods.
- Aggregate roots MUST inherit from `AggregateRoot<TId>` which collects domain events.
- Aggregate roots MUST expose a private parameterless constructor for ORM reconstitution (marked with a comment).

## Value Objects
- Value objects MUST be implemented as `sealed record` or as a class that overrides `Equals` and `GetHashCode`.
- Value objects MUST validate all invariants in their constructor and throw `DomainException` on violation.
- Value objects MUST be immutable — no mutation methods; produce new instances for modifications.

## Domain Events
- Domain events MUST implement `IDomainEvent` (marker interface).
- Domain events MUST be named in past tense and as `record` types: `OrderPlaced`, `PaymentFailed`.
- Domain events MUST be raised inside the aggregate via `Raise()` and stored until dispatched by the Application layer.

## Domain Exceptions
- Domain rule violations MUST throw `DomainException` (a custom exception in the Domain project).
- `DomainException` MUST NOT wrap infrastructure exceptions — those are handled in Infrastructure.
- Exception messages MUST describe the business rule that was violated, not a technical error.
</rules>

<patterns>
## Strongly-Typed ID

```csharp
public readonly record struct OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.NewGuid());
    public static OrderId From(Guid value) => new(value);
    public override string ToString() => Value.ToString();
}
```

## AggregateRoot Base Class

```csharp
public abstract class AggregateRoot<TId> where TId : notnull
{
    private readonly List<IDomainEvent> _domainEvents = [];

    public TId Id { get; protected set; } = default!;
    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void Raise(IDomainEvent domainEvent) => _domainEvents.Add(domainEvent);
    public void ClearDomainEvents() => _domainEvents.Clear();
}
```

## Aggregate Root Implementation

```csharp
public sealed class Order : AggregateRoot<OrderId>
{
    private readonly List<OrderLine> _lines = [];

    public CustomerId CustomerId { get; private set; } = default!;
    public OrderStatus Status { get; private set; }
    public Money Total => _lines.Aggregate(Money.Zero("USD"), (sum, l) => sum.Add(l.Subtotal));
    public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();

    private Order() { } // ORM reconstitution

    public static Order Place(CustomerId customerId, IEnumerable<OrderLine> lines)
    {
        var lineList = lines.ToList();
        if (lineList.Count == 0)
            throw new DomainException("An order must have at least one line.");

        var order = new Order
        {
            Id = OrderId.New(),
            CustomerId = customerId,
            Status = OrderStatus.Pending
        };
        foreach (var line in lineList) order._lines.Add(line);
        order.Raise(new OrderPlaced(order.Id, customerId, order.Total));
        return order;
    }

    public void Confirm()
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException($"Cannot confirm an order with status '{Status}'.");
        Status = OrderStatus.Confirmed;
        Raise(new OrderConfirmed(Id));
    }

    public void Cancel(string reason)
    {
        if (Status == OrderStatus.Shipped)
            throw new DomainException("Cannot cancel an order that has already been shipped.");
        Status = OrderStatus.Cancelled;
        Raise(new OrderCancelled(Id, reason));
    }
}
```

## Value Object (Money)

```csharp
public sealed class Money : IEquatable<Money>
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0) throw new DomainException("Amount cannot be negative.");
        if (string.IsNullOrWhiteSpace(currency)) throw new DomainException("Currency is required.");
        Amount = Math.Round(amount, 2, MidpointRounding.AwayFromZero);
        Currency = currency.ToUpperInvariant();
    }

    public static Money Zero(string currency) => new(0, currency);

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException($"Cannot add {Currency} and {other.Currency}.");
        return new Money(Amount + other.Amount, Currency);
    }

    public bool Equals(Money? other) => other is not null && Amount == other.Amount && Currency == other.Currency;
    public override bool Equals(object? obj) => Equals(obj as Money);
    public override int GetHashCode() => HashCode.Combine(Amount, Currency);
    public override string ToString() => $"{Amount:F2} {Currency}";
}
```

## Domain Event

```csharp
public interface IDomainEvent { }

public record OrderPlaced(OrderId OrderId, CustomerId CustomerId, Money Total) : IDomainEvent;
public record OrderConfirmed(OrderId OrderId) : IDomainEvent;
public record OrderCancelled(OrderId OrderId, string Reason) : IDomainEvent;
```

## Domain Exception

```csharp
public sealed class DomainException : Exception
{
    public DomainException(string message) : base(message) { }
}
```
</patterns>

<anti-patterns>
- **`public set` on entity properties**: `public OrderStatus Status { get; set; }` — bypasses invariant enforcement; any code can put the entity in an invalid state.
- **Raw `Guid` as entity ID**: `public Guid Id { get; set; }` — type-unsafe; `OrderId` can be accidentally passed where `CustomerId` is expected at compile time.
- **EF Core `DbContext` import in Domain**: violates Clean Architecture's Dependency Rule — Domain becomes untestable without a database.
- **Domain events dispatched synchronously**: calling `EmailService.Send()` inside `Order.Confirm()` — couples domain to infrastructure; infrastructure can fail and rollback the domain event.
- **Mutable value objects**: `money.Amount = 10` — immutability is the core contract of value objects; mutation breaks referential transparency.
- **Throwing `Exception` instead of `DomainException`**: callers cannot distinguish domain rule violations from infrastructure errors.
- **Business logic in domain event handlers**: domain event handlers are infrastructure concerns (sending emails, updating read models) — no business rules there.
</anti-patterns>

<checklist>
- [ ] No EF Core, ASP.NET, or MediatR imports anywhere in the Domain project.
- [ ] All entity IDs are strongly-typed record structs.
- [ ] No public property setters on entities or aggregate roots.
- [ ] All aggregate roots inherit `AggregateRoot<TId>` and collect domain events.
- [ ] All value objects validate invariants in constructor and throw `DomainException`.
- [ ] Domain events are `record` types named in past tense.
- [ ] All domain rule violations throw `DomainException`, not `Exception` or `ArgumentException`.
- [ ] Private parameterless constructor exists on aggregate roots for ORM reconstitution.
</checklist>
