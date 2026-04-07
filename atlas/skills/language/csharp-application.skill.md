---
name: csharp-application
description: >
  C# .NET 8+ patterns for the Application layer using MediatR CQRS: command and query
  handlers, DTOs, FluentValidation pipeline behaviors, and domain event dispatch.
  Based on Microsoft .NET Architecture Guides and Jason Taylor's Clean Architecture
  template. Use for any task creating or modifying use cases, application services,
  or command/query handlers. Required by Dev Agents on Application-layer tasks.
applies-to: [dev-agent]
stack: dotnet
layer: application
version: 1.0
---

# C# Application Layer

<context>
This skill applies when implementing the Application layer in a .NET 8 Clean Architecture
project. It covers MediatR-based CQRS handlers, FluentValidation pipeline behaviors,
DTO definitions, and domain event dispatch. The Application layer orchestrates the domain
— it MUST NOT contain business logic and MUST NOT import Infrastructure.
</context>

<rules>
## Layer Boundaries
- Application classes MUST NOT import from Infrastructure or API projects.
- Application handlers MUST use repository interfaces from Domain — never concrete implementations.
- Application layer MUST NOT contain business logic; logic belongs in Domain entities/services.
- DTOs MUST be defined in the Application layer; they MUST NOT be shared with the Domain layer.

## Commands and Queries (MediatR)
- Commands MUST implement `IRequest<TResponse>` where `TResponse` is an ID type or `Unit`.
- Queries MUST implement `IRequest<TResponse>` where `TResponse` is a DTO or `IEnumerable<DTO>`.
- Handlers MUST implement `IRequestHandler<TRequest, TResponse>`.
- Command handlers MUST NOT return domain entities — return IDs or `Unit` only.
- Query handlers MUST NOT load domain aggregates — read from the database directly (Dapper, raw EF projections).

## Validation Pipeline
- Every command MUST have a corresponding `AbstractValidator<TCommand>` registered as a pipeline behavior.
- Validation MUST occur before the handler runs — use `ValidationBehavior<TRequest, TResponse>` pipeline behavior.
- Handlers MUST assume input is already validated; no duplicate validation inside handlers.

## Domain Event Dispatch
- After saving the aggregate, command handlers MUST publish all domain events via `IPublisher`.
- Domain events MUST be published AFTER the aggregate is persisted — not before.
- Handlers MUST call `aggregate.ClearDomainEvents()` after publishing.

## Error Handling
- Handlers MUST NOT catch `DomainException` — let it propagate to the API layer for mapping.
- Handlers MUST catch infrastructure exceptions (DbException, TimeoutException) and wrap them in application-layer exceptions if needed.
- `NotFoundException` MUST be thrown when a requested aggregate does not exist.
</rules>

<patterns>
## Command + Validator + Handler

```csharp
// Command
public record PlaceOrderCommand(Guid CustomerId, List<OrderLineDto> Lines) : IRequest<Guid>;

// DTO
public record OrderLineDto(Guid ProductId, int Quantity, decimal UnitPrice);

// Validator (FluentValidation)
public class PlaceOrderCommandValidator : AbstractValidator<PlaceOrderCommand>
{
    public PlaceOrderCommandValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Lines).NotEmpty().WithMessage("Order must have at least one line.");
        RuleForEach(x => x.Lines).ChildRules(line =>
        {
            line.RuleFor(l => l.Quantity).GreaterThan(0);
            line.RuleFor(l => l.UnitPrice).GreaterThan(0);
        });
    }
}

// Handler
public class PlaceOrderHandler(
    IOrderRepository repo,
    IPublisher publisher) : IRequestHandler<PlaceOrderCommand, Guid>
{
    public async Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var lines = cmd.Lines.Select(l =>
            new OrderLine(new ProductId(l.ProductId), l.Quantity, new Money(l.UnitPrice, "USD")));

        var order = Order.Place(new CustomerId(cmd.CustomerId), lines);
        await repo.SaveAsync(order, ct);

        foreach (var evt in order.DomainEvents)
            await publisher.Publish(evt, ct);
        order.ClearDomainEvents();

        return order.Id.Value;
    }
}
```

## Query Handler (reads directly from DB — Dapper)

```csharp
// Query
public record GetOrderByIdQuery(Guid OrderId) : IRequest<OrderDetailDto?>;

// DTO
public record OrderDetailDto(Guid Id, string Status, decimal Total, string CustomerName, DateTime CreatedAt);

// Handler — bypasses domain, reads directly
public class GetOrderByIdHandler(IDbConnection db) : IRequestHandler<GetOrderByIdQuery, OrderDetailDto?>
{
    public async Task<OrderDetailDto?> Handle(GetOrderByIdQuery qry, CancellationToken ct)
    {
        const string sql = """
            SELECT o.id, o.status, o.total, c.full_name AS customer_name, o.created_at
            FROM orders o
            JOIN customers c ON c.id = o.customer_id
            WHERE o.id = @OrderId
            """;
        return await db.QuerySingleOrDefaultAsync<OrderDetailDto>(sql, new { qry.OrderId });
    }
}
```

## ValidationBehavior Pipeline (register once)

```csharp
public class ValidationBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse> where TRequest : notnull
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (!validators.Any()) return await next();

        var context = new ValidationContext<TRequest>(request);
        var failures = validators
            .Select(v => v.Validate(context))
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count > 0)
            throw new ValidationException(failures);

        return await next();
    }
}

// Registration in DI (Program.cs or Infrastructure/DependencyInjection.cs)
builder.Services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(Application.AssemblyRef);
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
});
builder.Services.AddValidatorsFromAssembly(Application.AssemblyRef);
```

## NotFoundException (application-layer exception)

```csharp
public sealed class NotFoundException(string entityName, object id)
    : Exception($"{entityName} with ID '{id}' was not found.");

// Usage in handler
var order = await repo.FindByIdAsync(new OrderId(qry.OrderId), ct)
    ?? throw new NotFoundException(nameof(Order), qry.OrderId);
```
</patterns>

<anti-patterns>
- **Business logic in handler**: `if (order.Total > 1000 && customer.IsVip) discount = 0.15m` — domain logic in application layer; put invariants in the domain entity.
- **Returning domain entity from query handler**: `public Task<Order> Handle(...)` — exposes domain internals to API layer; return a DTO.
- **Loading aggregate in query handler**: `var order = await repo.FindByIdAsync(...)` for a list query — aggregates are designed for writes; read directly from DB for queries.
- **Publishing events before saving**: emitting domain events before `await repo.SaveAsync()` — if save fails, events were already published but state not persisted (consistency violation).
- **Catching `DomainException` in handler**: swallowing domain violations prevents the API layer from mapping them correctly to HTTP status codes.
- **Validation inside handler**: duplicating `if (cmd.Lines.Count == 0) throw ...` in the handler when a `FluentValidation` validator should handle this in the pipeline.
</anti-patterns>

<checklist>
- [ ] No imports from Infrastructure or API in Application project references.
- [ ] All commands return `Guid` (ID) or `Unit` — never a domain entity.
- [ ] All queries return DTOs — never domain aggregates.
- [ ] Every command has a corresponding FluentValidation validator.
- [ ] ValidationBehavior is registered in the MediatR pipeline.
- [ ] Domain events published after `SaveAsync`, then cleared from the aggregate.
- [ ] `NotFoundException` thrown for missing aggregates — not null returned.
- [ ] No business logic (domain rules) in command or query handlers.
</checklist>
