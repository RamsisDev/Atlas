---
name: csharp-infrastructure
description: >
  C# .NET 8+ patterns for the Infrastructure layer: EF Core DbContext configuration,
  IEntityTypeConfiguration for aggregate mapping, repository implementations, and
  database migration management. Based on Microsoft .NET Architecture Guides (eShopOnContainers).
  Use for any task creating or modifying repositories, database configurations, or
  persistence mappings. Required by Dev Agents on Infrastructure-layer tasks.
applies-to: [dev-agent]
stack: dotnet
layer: infrastructure
version: 1.0
---

# C# Infrastructure Layer

<context>
This skill applies when implementing the Infrastructure layer in a .NET 8 Clean
Architecture project. Covers EF Core 8 DbContext setup, entity configuration via
IEntityTypeConfiguration, repository implementations, and strongly-typed ID
conversions. This layer knows about Domain entities and Application interfaces —
it implements them. It MUST NOT be referenced by Application or Domain.
</context>

<rules>
## DbContext Design
- DbContext MUST inherit from `DbContext` (or `IdentityDbContext` if using Identity) — never from any domain base class.
- DbContext MUST NOT expose `DbSet<T>` publicly for types that are not aggregate roots — child entities are accessed through the aggregate.
- DbContext MUST apply all configurations via `modelBuilder.ApplyConfigurationsFromAssembly()` in `OnModelCreating`.
- DbContext MUST NOT contain business logic or domain rules.

## Entity Configuration
- Every aggregate root MUST have a dedicated `IEntityTypeConfiguration<TAggregateRoot>` class.
- Strongly-typed IDs MUST be converted using `ValueConverter<TId, Guid>` (or appropriate primitive).
- Value objects MUST be mapped as `OwnsOne` (single table) or as a separate table with `OwnsMany`.
- Shadow properties MUST be used for infrastructure concerns (e.g., `RowVersion`, `CreatedAt`) that the domain does not need to know about.
- Concurrency tokens SHOULD be configured for aggregates that require optimistic concurrency.

## Repositories
- Repository classes MUST implement the interface defined in the Domain layer.
- Repository implementations MUST use `DbContext` directly — no Unit of Work wrapper unless explicitly required.
- Repositories MUST NOT expose `IQueryable<T>` — return fully-reconstituted aggregates or null.
- `FindByIdAsync` MUST eager-load all child collections needed to reconstitute the aggregate completely.
- `SaveAsync` MUST call `dbContext.SaveChangesAsync()` — one save per aggregate per transaction.

## Migrations
- All schema changes MUST go through EF Core migrations — no manual DDL scripts.
- Migration names MUST be descriptive: `AddOrderStatusColumn`, not `Migration20240401`.
- Migrations MUST be reviewed for data loss (dropping columns, changing types) before applying.
- `database.MigrateAsync()` MUST be called at startup (or via a dedicated migration service), not `EnsureCreated`.
</rules>

<patterns>
## DbContext

```csharp
public sealed class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Order> Orders { get; init; } = null!;
    public DbSet<Customer> Customers { get; init; } = null!;

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

## IEntityTypeConfiguration for Aggregate Root

```csharp
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("orders");

        // Strongly-typed ID conversion
        builder.HasKey(o => o.Id);
        builder.Property(o => o.Id)
            .HasConversion(id => id.Value, value => OrderId.From(value))
            .HasColumnName("id");

        // Enum stored as string
        builder.Property(o => o.Status)
            .HasConversion<string>()
            .HasColumnName("status")
            .HasMaxLength(50);

        // Value object — owned entity (single table)
        builder.OwnsOne(o => o.ShippingAddress, addr =>
        {
            addr.Property(a => a.Street).HasColumnName("shipping_street").HasMaxLength(200);
            addr.Property(a => a.City).HasColumnName("shipping_city").HasMaxLength(100);
            addr.Property(a => a.PostalCode).HasColumnName("shipping_postal_code").HasMaxLength(20);
        });

        // Child entity collection (private field)
        builder.HasMany<OrderLine>("_lines")
            .WithOne()
            .HasForeignKey("order_id")
            .OnDelete(DeleteBehavior.Cascade);

        // Shadow property for optimistic concurrency
        builder.Property<byte[]>("RowVersion").IsRowVersion();

        // Foreign key — CustomerId as strongly-typed ID
        builder.Property(o => o.CustomerId)
            .HasConversion(id => id.Value, value => CustomerId.From(value))
            .HasColumnName("customer_id");
    }
}
```

## Repository Implementation

```csharp
public sealed class EfOrderRepository(AppDbContext db) : IOrderRepository
{
    public async Task<Order?> FindByIdAsync(OrderId id, CancellationToken ct) =>
        await db.Orders
            .Include("_lines")          // eager-load private collection
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task SaveAsync(Order order, CancellationToken ct)
    {
        if (db.Entry(order).State == EntityState.Detached)
            db.Orders.Add(order);
        else
            db.Orders.Update(order);

        await db.SaveChangesAsync(ct);
    }
}
```

## Dependency Injection Registration (Infrastructure project)

```csharp
// Infrastructure/DependencyInjection.cs
public static IServiceCollection AddInfrastructure(
    this IServiceCollection services, IConfiguration config)
{
    services.AddDbContext<AppDbContext>(opts =>
        opts.UseNpgsql(config.GetConnectionString("DefaultConnection"),
            npgsql => npgsql.MigrationsAssembly(typeof(AppDbContext).Assembly.FullName)));

    services.AddScoped<IOrderRepository, EfOrderRepository>();
    services.AddScoped<ICustomerRepository, EfCustomerRepository>();

    return services;
}
```

## Apply Migrations at Startup

```csharp
// Program.cs
using var scope = app.Services.CreateScope();
var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
await db.Database.MigrateAsync();
```
</patterns>

<anti-patterns>
- **Exposing `IQueryable<Order>` from repository**: callers bypass the aggregate boundary; LINQ queries added by Application layer leak persistence concerns upward.
- **`EnsureCreated()` instead of `MigrateAsync()`**: drops and recreates schema in some scenarios; `EnsureCreated` does not run migrations and is not production-safe.
- **Partial aggregate loading**: `FindByIdAsync` returns an `Order` without including `_lines` — aggregate is not reconstituted; domain methods fail with null reference.
- **Business logic in `OnModelCreating`**: computing values, calling services, or applying domain rules during schema configuration — `OnModelCreating` is for persistence mapping only.
- **Public `DbSet` for child entities**: `public DbSet<OrderLine> OrderLines` — allows direct queries on child entities bypassing the aggregate root, breaking the DDD aggregate boundary.
- **Manual SQL DDL in migrations**: adding raw `migrationBuilder.Sql("ALTER TABLE ...")` for schema changes that EF Core can express — migration drift from the model leads to bugs.
- **`new AppDbContext()` inside repository**: repositories should receive DbContext via constructor injection — manual instantiation bypasses scoping and transaction management.
</anti-patterns>

<checklist>
- [ ] `ApplyConfigurationsFromAssembly` used in `OnModelCreating` — no inline `entity.HasKey()` in DbContext.
- [ ] Every aggregate root has a dedicated `IEntityTypeConfiguration<T>` class.
- [ ] Strongly-typed IDs have `ValueConverter` configured.
- [ ] Value objects mapped with `OwnsOne` or `OwnsMany`.
- [ ] `FindByIdAsync` eager-loads all child collections required by domain methods.
- [ ] No `IQueryable<T>` returned from any repository method.
- [ ] `MigrateAsync()` called at startup — not `EnsureCreated()`.
- [ ] Repository registered in DI via interface — `IOrderRepository → EfOrderRepository`.
</checklist>
