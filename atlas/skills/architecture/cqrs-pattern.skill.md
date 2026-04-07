---
name: cqrs-pattern
description: >
  Rules and patterns for implementing Command Query Responsibility Segregation (CQRS)
  at the Application layer. Covers command/query separation, handler structure, read
  model projections, and when CQRS adds value versus when it is over-engineering.
  Use when implementing use cases via MediatR or equivalent dispatcher. Required by
  Dev Agents working on Application layer and the Architecture Analyst (D02).
applies-to: [dev-agent, architecture-analyst, spec-writer]
stack: none
layer: application
version: 1.0
---

# CQRS Pattern

<context>
This skill applies when structuring Application-layer use cases. CQRS separates
operations that change state (Commands) from operations that read state (Queries).
Commands go through the domain model and enforce invariants; Queries bypass the
domain and read directly from a read-optimized store. Apply at the Application layer —
not necessarily at the infrastructure level (event sourcing is not required).
</context>

<rules>
## Command / Query Separation
- A Command MUST change state and MUST NOT return domain data (may return an ID or void).
- A Query MUST return data and MUST NOT change state — no side effects.
- A single handler MUST NOT act as both a command and a query handler.
- Commands MUST be named in the imperative (e.g., `PlaceOrderCommand`, `CancelSubscriptionCommand`).
- Queries MUST be named as questions or data requests (e.g., `GetOrderByIdQuery`, `ListActiveUsersQuery`).

## Command Handlers
- Command handlers MUST validate the command before touching the domain.
- Command handlers MUST load the aggregate via its repository, call the domain method, then save.
- Command handlers MUST dispatch domain events after the aggregate is persisted.
- Command handlers MUST NOT contain business logic — all logic belongs in the domain model.

## Query Handlers
- Query handlers SHOULD bypass the domain model and read directly from the database (raw SQL, views, projections).
- Query handlers MUST return DTOs or view models, never domain entities.
- Query handlers MUST NOT trigger writes or side effects.
- Query reads SHOULD be optimized independently of the write model (separate projections, denormalized views).

## Dispatcher
- Commands and Queries MUST be dispatched through a single mediator/bus interface (e.g., `ISender`).
- API endpoints MUST NOT instantiate handlers directly — dispatch only through the mediator.
- The mediator interface MUST be defined in Application; no infrastructure imports in the dispatch call.
</rules>

<patterns>
## Command + Handler (C# / MediatR)

```csharp
// Command — imperative name, no return data except ID
public record PlaceOrderCommand(Guid CustomerId, List<OrderLineDto> Lines) : IRequest<Guid>;

// Handler — load → domain method → save → dispatch events
public class PlaceOrderHandler(
    IOrderRepository repo,
    IPublisher publisher) : IRequestHandler<PlaceOrderCommand, Guid>
{
    public async Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var lines = cmd.Lines.Select(l => new OrderLine(l.ProductId, l.Quantity, l.UnitPrice));
        var order = Order.Place(new CustomerId(cmd.CustomerId), lines);
        await repo.SaveAsync(order, ct);
        foreach (var evt in order.DomainEvents)
            await publisher.Publish(evt, ct);
        order.ClearEvents();
        return order.Id.Value;
    }
}
```

## Query + Handler with direct read (C# / Dapper)

```csharp
// Query — question-shaped name, returns DTO
public record GetOrderByIdQuery(Guid OrderId) : IRequest<OrderDetailDto?>;

// Handler — bypasses domain, reads from DB directly
public class GetOrderByIdHandler(IDbConnection db) : IRequestHandler<GetOrderByIdQuery, OrderDetailDto?>
{
    public async Task<OrderDetailDto?> Handle(GetOrderByIdQuery qry, CancellationToken ct)
    {
        const string sql = """
            SELECT o.id, o.status, o.created_at, c.name AS customer_name
            FROM orders o JOIN customers c ON c.id = o.customer_id
            WHERE o.id = @Id
            """;
        return await db.QuerySingleOrDefaultAsync<OrderDetailDto>(sql, new { Id = qry.OrderId });
    }
}
```

## API endpoint dispatching via mediator

```csharp
// Minimal API — endpoint only dispatches, no logic
app.MapPost("/orders", async (PlaceOrderCommand cmd, ISender sender, CancellationToken ct) =>
{
    var id = await sender.Send(cmd, ct);
    return Results.CreatedAtRoute("GetOrder", new { id }, null);
});

app.MapGet("/orders/{id:guid}", async (Guid id, ISender sender, CancellationToken ct) =>
{
    var dto = await sender.Send(new GetOrderByIdQuery(id), ct);
    return dto is null ? Results.NotFound() : Results.Ok(dto);
}).WithName("GetOrder");
```
</patterns>

<anti-patterns>
- **Command returns domain entity**: couples the caller to the write model; use an ID or void return instead.
- **Query handler loads an aggregate**: loading a full aggregate for a list view wastes resources and couples the read path to domain complexity; read directly from the database.
- **Business logic in command handler**: the handler should orchestrate (load → call → save), not decide; domain rules belong in the aggregate.
- **Single "service" class with both read and write methods**: violates CQS; reads and writes have different optimization axes and should evolve independently.
- **Applying CQRS to a CRUD app**: for simple create/read/update/delete with no complex invariants, CQRS adds ceremony without benefit. Use plain services until the domain justifies it.
- **Event sourcing required for CQRS**: CQRS does not require event sourcing; a separate read model can be as simple as a denormalized SQL view.
</anti-patterns>

<checklist>
- [ ] Every command is named in the imperative and returns void or an ID.
- [ ] Every query is named as a data request and returns a DTO, not a domain entity.
- [ ] No command handler reads data for the response (reads only to load the aggregate).
- [ ] No query handler calls domain model methods or writes to the database.
- [ ] API endpoints dispatch via mediator — no direct `new Handler()` instantiation.
- [ ] Domain events are dispatched after persistence, inside the command handler.
- [ ] Read models (DTOs) are not reused as command inputs or domain objects.
</checklist>
