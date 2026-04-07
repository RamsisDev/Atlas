---
name: csharp-api
description: >
  C# .NET 8+ patterns for the API layer using Minimal APIs: endpoint definition,
  Problem Details (RFC 7807), global exception handling, OpenAPI annotations, and
  authentication/authorization middleware. Based on Microsoft ASP.NET Core Minimal
  APIs documentation and eShopOnContainers reference. Use for any task creating or
  modifying API endpoints, middleware, or error handling. Required by Dev Agents on
  API-layer tasks.
applies-to: [dev-agent]
stack: dotnet
layer: api
version: 1.0
---

# C# API Layer (Minimal APIs)

<context>
This skill applies when implementing .NET 8 Minimal API endpoints. Endpoints are
thin — they parse HTTP requests, dispatch to the Application layer via MediatR's
`ISender`, and map responses to HTTP status codes. No business logic belongs here.
Error responses MUST follow RFC 7807 Problem Details.
</context>

<rules>
## Endpoint Design
- Endpoints MUST dispatch to Application layer via `ISender.Send()` — no direct service calls or repository access.
- Endpoints MUST NOT contain business logic, validation, or domain rules.
- Each endpoint group SHOULD be defined in a dedicated class with an `MapEndpoints(IEndpointRouteBuilder)` extension method.
- Route parameters MUST be typed (e.g., `{id:guid}`) to reject malformed input at routing level.

## HTTP Semantics
- POST (create) endpoints MUST return `201 Created` with a `Location` header pointing to the new resource.
- GET endpoints MUST return `200 OK` with the resource or `404 Not Found` if absent.
- PUT/PATCH endpoints MUST return `200 OK` or `204 No Content`.
- DELETE endpoints MUST return `204 No Content` on success.
- Validation failures MUST return `422 Unprocessable Entity` with a Problem Details body.
- Unauthenticated requests MUST return `401 Unauthorized`; unauthorized (forbidden) requests MUST return `403 Forbidden`.

## Error Handling
- A global exception handler MUST be registered to convert exceptions to Problem Details responses.
- `DomainException` MUST map to `422 Unprocessable Entity` with the domain message in `detail`.
- `NotFoundException` MUST map to `404 Not Found`.
- `ValidationException` (FluentValidation) MUST map to `422 Unprocessable Entity` with per-field errors.
- Unhandled exceptions MUST return `500 Internal Server Error` with a generic message — never a stack trace.

## OpenAPI
- Every endpoint MUST be annotated with `.Produces<T>()` and `.ProducesValidationProblem()` / `.ProducesProblem()` for all response codes.
- Endpoints MUST have `.WithName()` and `.WithTags()` for grouping in Swagger UI.
- Request body types MUST be Pydantic/annotated so OpenAPI schema is generated automatically.

## Security
- Endpoints requiring authentication MUST be decorated with `.RequireAuthorization()`.
- Endpoints requiring a specific role or policy MUST use `.RequireAuthorization("PolicyName")`.
- Public endpoints MUST explicitly be decorated with `.AllowAnonymous()` for documentation clarity.
</rules>

<patterns>
## Endpoint Group (Minimal API)

```csharp
// API/Endpoints/OrdersEndpoints.cs
public static class OrdersEndpoints
{
    public static IEndpointRouteBuilder MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/orders")
            .WithTags("Orders")
            .RequireAuthorization();

        group.MapPost("/", PlaceOrder)
            .WithName("PlaceOrder")
            .Produces<Guid>(StatusCodes.Status201Created)
            .ProducesValidationProblem()
            .ProducesProblem(StatusCodes.Status401Unauthorized);

        group.MapGet("/{id:guid}", GetOrderById)
            .WithName("GetOrder")
            .Produces<OrderDetailDto>()
            .ProducesProblem(StatusCodes.Status404NotFound);

        return app;
    }

    private static async Task<IResult> PlaceOrder(
        PlaceOrderCommand cmd,
        ISender sender,
        CancellationToken ct)
    {
        var id = await sender.Send(cmd, ct);
        return Results.CreatedAtRoute("GetOrder", new { id }, id);
    }

    private static async Task<IResult> GetOrderById(
        Guid id,
        ISender sender,
        CancellationToken ct)
    {
        var dto = await sender.Send(new GetOrderByIdQuery(id), ct);
        return dto is null ? Results.NotFound() : Results.Ok(dto);
    }
}
```

## Global Exception Handler (Problem Details)

```csharp
// API/Middleware/GlobalExceptionHandler.cs
public sealed class GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext ctx, Exception exception, CancellationToken ct)
    {
        var (status, title, detail) = exception switch
        {
            ValidationException ve  => (422, "Validation Failed",   string.Join("; ", ve.Errors.Select(e => e.ErrorMessage))),
            DomainException de      => (422, "Business Rule Violation", de.Message),
            NotFoundException nfe   => (404, "Not Found",           nfe.Message),
            _                       => (500, "Internal Server Error", "An unexpected error occurred.")
        };

        if (status == 500)
            logger.LogError(exception, "Unhandled exception");

        var problem = new ProblemDetails
        {
            Status = status,
            Title  = title,
            Detail = detail,
            Type   = $"https://tools.ietf.org/html/rfc9110#section-15.{(status == 404 ? "5.5" : "5.1")}"
        };

        ctx.Response.StatusCode = status;
        await ctx.Response.WriteAsJsonAsync(problem, ct);
        return true;
    }
}

// Registration in Program.cs
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();
// ...
app.UseExceptionHandler();
```

## Program.cs Structure

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddApplication()          // MediatR + validators (Application project)
    .AddInfrastructure(builder.Configuration)  // EF Core + repositories
    .AddAuthentication(...)
    .AddAuthorization(...)
    .AddExceptionHandler<GlobalExceptionHandler>()
    .AddProblemDetails()
    .AddEndpointsApiExplorer()
    .AddSwaggerGen();

var app = builder.Build();

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.UseExceptionHandler();

app.MapOrderEndpoints();
app.MapCustomerEndpoints();

if (app.Environment.IsDevelopment())
    app.UseSwaggerUI();

// Apply migrations
using (var scope = app.Services.CreateScope())
    await scope.ServiceProvider.GetRequiredService<AppDbContext>().Database.MigrateAsync();

app.Run();
```
</patterns>

<anti-patterns>
- **Business logic in endpoints**: `if (order.Total > 500) ApplyDiscount()` inside a route handler — logic belongs in domain or application; endpoints only orchestrate.
- **Direct repository access in endpoints**: `_repo.FindByIdAsync(id)` bypasses the Application layer; always go through `ISender`.
- **Generic 500 for validation errors**: returning 500 for `ValidationException` — clients cannot distinguish user error from server failure; use 422 with Problem Details.
- **Stack traces in error responses**: `"detail": "NullReferenceException at OrdersEndpoints.GetOrder line 42"` — exposes internals; always use generic messages for 500 errors.
- **Missing `Location` header on POST**: returning `201 Created` without a `Location` header violates REST conventions and breaks hypermedia clients.
- **No `.Produces<T>()` annotations**: OpenAPI schema omits response types; API consumers cannot generate correct client code.
- **`.RequireAuthorization()` forgotten**: endpoints default to anonymous; a single missing attribute exposes a protected resource.
</anti-patterns>

<checklist>
- [ ] All endpoints dispatch via `ISender.Send()` — no direct service or repository calls.
- [ ] POST endpoints return `201 Created` with `Location` header.
- [ ] GET endpoints return `200 OK` or `404 Not Found`.
- [ ] Global exception handler registered and converts `DomainException` → 422, `NotFoundException` → 404.
- [ ] No stack traces exposed in HTTP responses.
- [ ] All endpoints have `.Produces<T>()` and `.ProducesProblem()` annotations.
- [ ] Protected endpoints have `.RequireAuthorization()`.
- [ ] Public endpoints explicitly have `.AllowAnonymous()`.
</checklist>
