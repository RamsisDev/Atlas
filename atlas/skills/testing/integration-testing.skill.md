---
name: integration-testing
description: >
  Rules and patterns for writing integration tests that verify behavior across multiple
  components including real databases, HTTP layers, and external service stubs. Covers
  WebApplicationFactory for .NET, Testcontainers for database lifecycle, and FastAPI
  TestClient for Python. Required by QA agents and Dev Agents writing API or repository
  tests that need real infrastructure components.
applies-to: [dev-agent, qa-agent, qa-test-subagent, integration-agent]
stack: none
layer: Testing
version: 1.0
---

# Integration Testing

<context>
This skill applies when writing tests that cross a process boundary (real HTTP server,
real database, real message broker) or that verify multiple components working together.
Integration tests are slower than unit tests but verify that components integrate
correctly. They use real infrastructure managed by Testcontainers or an in-process test
server (WebApplicationFactory / FastAPI TestClient).
</context>

<rules>
## Scope Definition
- Integration tests MUST test at least two components together (e.g., HTTP handler + application service + repository + database).
- Integration tests MUST NOT test domain logic in isolation — that is the responsibility of unit tests.
- Each integration test MUST be independent: it MUST NOT depend on state created by another test.

## Database Isolation
- Each test or test class MUST start with a known database state; shared mutable state between tests MUST NOT exist.
- Database state SHOULD be reset between tests using transactions (rolled back after each test) or by truncating tables in a setup method.
- Production databases MUST NOT be used for integration tests; use Testcontainers or a dedicated test database.
- Database migrations MUST be applied to the test database before the test suite runs.

## Test Server
- HTTP integration tests MUST use an in-process test server (WebApplicationFactory for .NET, TestClient for FastAPI).
- The test server MUST configure the same dependency injection as production, with only external dependencies swapped (real DB via Testcontainers, stubbed third-party HTTP clients).
- The test MUST make HTTP requests and assert HTTP responses — it MUST NOT call application services directly, bypassing the HTTP layer.

## External Services
- Real external HTTP services (payment gateways, email providers) MUST be replaced with stubs (WireMock, httpretty, respx) in integration tests.
- Stub servers MUST verify that the application makes the expected outbound requests (assert request was made with correct shape).
- Message brokers in tests SHOULD use an in-memory or containerized broker; production brokers MUST NOT receive test messages.

## Data Setup
- Test data MUST be created programmatically (via application APIs or database seed helpers), not by importing production data dumps.
- Test data helpers MUST clean up after themselves or be created inside a transaction that is rolled back.
- Hardcoded IDs in tests MUST be unique per test run or generated dynamically to avoid collisions.
</rules>

<patterns>
## .NET — WebApplicationFactory + Testcontainers

```csharp
public class OrdersApiTests : IAsyncLifetime
{
    private readonly PostgreSqlContainer _db = new PostgreSqlBuilder().Build();
    private WebApplicationFactory<Program> _factory = null!;
    private HttpClient _client = null!;

    public async Task InitializeAsync()
    {
        await _db.StartAsync();
        _factory = new WebApplicationFactory<Program>().WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Replace real DB connection with Testcontainers connection string
                services.RemoveAll<DbContextOptions<AppDbContext>>();
                services.AddDbContext<AppDbContext>(opts =>
                    opts.UseNpgsql(_db.GetConnectionString()));
            });
        });
        _client = _factory.CreateClient();

        // Apply migrations
        using var scope = _factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();
    }

    public async Task DisposeAsync()
    {
        await _factory.DisposeAsync();
        await _db.DisposeAsync();
    }

    [Fact]
    public async Task POST_Orders_ValidPayload_Returns201WithLocation()
    {
        var payload = new { customerId = Guid.NewGuid(), lines = new[] { new { productId = Guid.NewGuid(), quantity = 2, unitPrice = 19.99 } } };

        var response = await _client.PostAsJsonAsync("/orders", payload);

        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        Assert.NotNull(response.Headers.Location);
    }
}
```

## Python — FastAPI TestClient + SQLAlchemy + Testcontainers

```python
import pytest
from fastapi.testclient import TestClient
from testcontainers.postgres import PostgresContainer
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.main import app
from app.database import get_db, Base

@pytest.fixture(scope="session")
def pg_container():
    with PostgresContainer("postgres:16") as pg:
        yield pg

@pytest.fixture(scope="session")
def db_engine(pg_container):
    engine = create_engine(pg_container.get_connection_url())
    Base.metadata.create_all(engine)
    return engine

@pytest.fixture()
def client(db_engine):
    TestingSession = sessionmaker(bind=db_engine)
    session = TestingSession()
    session.begin_nested()  # savepoint — rolled back after test

    def override_db():
        yield session

    app.dependency_overrides[get_db] = override_db
    with TestClient(app) as c:
        yield c
    session.rollback()
    session.close()

def test_place_order_returns_201(client):
    payload = {"customer_id": "cust-1", "items": [{"product_id": "p-1", "quantity": 1, "unit_price": 9.99}]}
    response = client.post("/orders", json=payload)
    assert response.status_code == 201
    assert "id" in response.json()
```

## Stubbing outbound HTTP (Python / respx)

```python
import respx
import httpx

@respx.mock
def test_place_order_calls_payment_service(client):
    respx.post("https://payments.internal/charge").mock(
        return_value=httpx.Response(200, json={"charge_id": "ch_123"})
    )
    response = client.post("/orders", json=valid_order_payload)
    assert response.status_code == 201
    assert respx.calls.call_count == 1
    assert respx.calls[0].request.url == "https://payments.internal/charge"
```
</patterns>

<anti-patterns>
- **Tests sharing database state**: test B relies on data created by test A — execution order dependency causes random failures.
- **Using production database**: integration tests write real data or call real external services — billing charges, data corruption, flaky CI.
- **Calling application services directly in HTTP tests**: bypasses middleware, authentication, routing — important integration points are untested.
- **No migration applied before tests**: testing against a stale schema — production bugs slip through because the DB structure doesn't match.
- **Real external HTTP calls**: tests fail when the third-party API is down; test suite cannot run offline; non-deterministic results.
- **Hardcoded IDs**: `customer_id = "00000000-0000-0000-0000-000000000001"` reused across tests — tests pollute each other's data when not using transactions.
- **Testcontainer per test method** (not per session): starting a new container for each test adds minutes to the suite; share the container, isolate state via transactions.
</anti-patterns>

<checklist>
- [ ] Each test starts from a known, isolated database state (transaction rollback or truncate).
- [ ] Testcontainers or a dedicated test database used — never production.
- [ ] Migrations applied before the test suite runs.
- [ ] HTTP tests use an in-process test server (WebApplicationFactory / TestClient).
- [ ] All outbound HTTP calls to real third parties are stubbed (WireMock / respx / httpretty).
- [ ] Tests make requests and assert responses — not calling services directly.
- [ ] Test container created once per session (not per test method).
- [ ] No test-to-test state sharing via class-level mutable fields.
</checklist>
