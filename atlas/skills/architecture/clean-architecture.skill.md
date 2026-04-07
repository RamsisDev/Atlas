---
name: clean-architecture
description: >
  Rules and patterns for implementing Clean Architecture (Uncle Bob) in any stack.
  Applies when structuring a project into Domain, Application, Infrastructure, and API
  layers with strict inward-only dependency flow. Use this skill whenever a task
  creates or modifies layer boundaries, project references, or cross-cutting concerns.
  Required by Dev Agents, Software Architect (D04), and Architecture Analyst (D02).
applies-to: [dev-agent, software-architect, architecture-analyst, spec-writer]
stack: none
layer: none
version: 1.0
---

# Clean Architecture

<context>
This skill applies when implementing or reviewing a project structured in concentric
layers: Domain (innermost), Application, Infrastructure, and Presentation/API
(outermost). The Dependency Rule is the single non-negotiable constraint: source-code
dependencies MUST only point inward — outer layers depend on inner layers, never the
reverse. Applies to any language or framework.
</context>

<rules>
## Dependency Rule
- Outer layers MUST depend on inner layers; inner layers MUST NOT depend on outer layers.
- Domain MUST NOT import from Application, Infrastructure, or API layers.
- Application MUST NOT import from Infrastructure or API layers.
- Infrastructure MUST depend on Application interfaces (ports), not the reverse.
- Cross-layer communication MUST go through interfaces (abstractions), not concrete types.

## Layer Responsibilities
- Domain MUST contain only: entities, value objects, domain events, aggregates, domain services, and repository interfaces.
- Application MUST contain only: use cases / command handlers / query handlers, DTOs, and application service interfaces.
- Infrastructure MUST contain only: repository implementations, ORM configurations, external service adapters, and persistence mappings.
- API/UI layer MUST contain only: controllers, minimal API endpoints, serialization models, and middleware.

## Interfaces and Abstractions
- Repository interfaces MUST be defined in Domain or Application, never in Infrastructure.
- Application services MUST depend on repository interfaces, not on concrete implementations.
- Infrastructure adapters MUST implement interfaces defined in inner layers.

## Framework Isolation
- Domain entities MUST NOT inherit from or be decorated by framework base classes (e.g., ActiveRecord, ORM base entities).
- Application use cases MUST NOT reference HTTP, database, or messaging frameworks directly.
- Framework-specific code SHALL be confined to Infrastructure and API layers.
</rules>

<patterns>
## Layer Structure (any stack)

```
src/
├── Domain/
│   ├── Entities/
│   ├── ValueObjects/
│   ├── Events/
│   └── Interfaces/        ← repository + domain service interfaces
├── Application/
│   ├── Commands/
│   ├── Queries/
│   ├── DTOs/
│   └── Interfaces/        ← external service interfaces (email, storage)
├── Infrastructure/
│   ├── Persistence/       ← EF Core / SQLAlchemy / Prisma configurations
│   ├── Repositories/      ← implements Domain.Interfaces
│   └── ExternalServices/  ← implements Application.Interfaces
└── API/
    ├── Controllers/ (or Endpoints/)
    └── Middleware/
```

## Correct Dependency Direction (C#)

```csharp
// Domain — no external imports
namespace MyApp.Domain.Interfaces;
public interface IOrderRepository
{
    Task<Order?> FindByIdAsync(OrderId id, CancellationToken ct);
    Task SaveAsync(Order order, CancellationToken ct);
}

// Application — depends on Domain interface, not Infrastructure
namespace MyApp.Application.Commands;
public class PlaceOrderHandler(IOrderRepository repo) : IRequestHandler<PlaceOrderCommand, OrderId>
{
    public async Task<OrderId> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Place(cmd.CustomerId, cmd.Items);
        await repo.SaveAsync(order, ct);
        return order.Id;
    }
}

// Infrastructure — implements Domain interface
namespace MyApp.Infrastructure.Repositories;
public class EfOrderRepository(AppDbContext db) : IOrderRepository
{
    public async Task<Order?> FindByIdAsync(OrderId id, CancellationToken ct) =>
        await db.Orders.FindAsync([id.Value], ct);
    public async Task SaveAsync(Order order, CancellationToken ct)
    {
        db.Orders.Update(order);
        await db.SaveChangesAsync(ct);
    }
}
```

## Dependency Injection Wiring (API layer only)

```csharp
// Program.cs — only the API layer knows about concrete implementations
builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();
builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(Application.AssemblyRef));
```
</patterns>

<anti-patterns>
- **Domain imports Infrastructure**: violates the Dependency Rule; domain becomes untestable without a real database.
- **Application uses `new DbContext()`**: couples use cases to a specific ORM; prevents unit testing.
- **Repository returns `IQueryable<T>`**: leaks persistence concerns (EF Core) into Application layer; callers can add arbitrary LINQ that bypasses domain logic.
- **Fat controllers**: business logic in controllers couples rules to HTTP; logic cannot be reused or tested without spinning up a web host.
- **Entity inherits `ActiveRecord` or ORM base**: framework invades Domain; switching ORMs or testing without a database becomes impossible.
- **Circular references between layers**: any bidirectional project reference is a clean architecture violation.
- **DTOs in Domain**: Domain objects must not be shaped by transport concerns; define DTOs in Application or API.
</anti-patterns>

<checklist>
- [ ] No project reference from Domain → Application, Infrastructure, or API exists.
- [ ] No project reference from Application → Infrastructure or API exists.
- [ ] All repository interfaces are declared in Domain or Application.
- [ ] No `DbContext`, HTTP client, or messaging client is imported in Domain or Application.
- [ ] Dependency injection wiring exists only in API/Infrastructure project (Composition Root).
- [ ] Domain entities have no base class from any ORM or framework.
- [ ] Controllers/endpoints contain no business logic — only orchestration of use cases.
</checklist>
