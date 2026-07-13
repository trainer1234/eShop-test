# Code Snippets per Blog Post — eShop Multi-Mode Functional Testing

> Copy-paste-ready snippets extracted from the merged eShop codebase  
> (PR #1: `feat/logging_integration-test-mode`, commit `7b58e4f`)  
> Use with `docs/blog-series-plan.md` and `docs/post-1-fidelity-ladder.md`.

---

## Post 1 — Fidelity ladder (concepts only)

No code required beyond the comparison table in the draft. Optional: link readers to the mode enum.

**File:** `tests/Catalog.FunctionalTests/Fixture/CatalogFunctionalTestMode.cs`

```csharp
public enum CatalogFunctionalTestMode
{
    Aspire,
    AspireMessagingOutbox,
    AspireMessagingRabbitMq,
    RepositoryMock,
    EfCoreInMemory,
    Testcontainers
}
```

**Run a quick smoke test (any mode via runsettings):**

```bash
dotnet test tests/Catalog.FunctionalTests --settings eShop.FunctionalTests.RepositoryMock.runsettings
dotnet test tests/Catalog.FunctionalTests --filter-trait FunctionalTestMode=aspire
```

---

## Post 2 — Repository Mock

### Why production needed a repository abstraction

**File:** `src/Catalog.API/Infrastructure/Repositories/ICatalogRepository.cs`

```csharp
public interface ICatalogRepository
{
    Task<PaginatedItems<CatalogItem>> GetItemsAsync(
        int pageIndex, int pageSize, string? name, int? type, int? brand,
        CancellationToken cancellationToken = default);
    Task<CatalogItem?> GetItemByIdAsync(int id, bool includeBrand = false,
        CancellationToken cancellationToken = default);
    Task AddAsync(CatalogItem item, CancellationToken cancellationToken = default);
    void Remove(CatalogItem item);
    Task SaveChangesAsync(CancellationToken cancellationToken = default);
    // ... GetBrandsAsync, GetTypesAsync, semantic search, etc.
}
```

**File:** `src/Catalog.API/Extensions/Extensions.cs` — registration:

```csharp
builder.Services.AddScoped<ICatalogRepository, CatalogRepository>();
```

### Swapping the repository in tests

**File:** `tests/Catalog.FunctionalTests/Configuration/CatalogTestServiceConfiguration.cs`

```csharp
public static void ConfigureRepositoryMock(IServiceCollection services, CatalogRepositoryMockStore store)
{
    RemoveSharedExternalServices(services);
    services.RemoveAll<ICatalogRepository>();
    services.RemoveAll<CatalogContext>();
    services.RemoveAll<DbContextOptions<CatalogContext>>();

    services.AddSingleton(store);
    services.AddSingleton<ICatalogRepository, InMemoryCatalogRepository>();
    AddSharedTestDoubles(services);
}
```

### Fixture wiring for mock mode

**File:** `tests/Catalog.FunctionalTests/Fixture/CatalogApiFixture.cs` (excerpt)

```csharp
case CatalogFunctionalTestMode.RepositoryMock:
    builder.UseEnvironment("Build");
    builder.ConfigureTestServices(services =>
        CatalogTestServiceConfiguration.ConfigureRepositoryMock(services, _repositoryMockStore));
    break;
```

```csharp
case CatalogFunctionalTestMode.RepositoryMock:
    await _repositoryMockStore.ResetAsync();
    _ = Services;
    break;
```

### Test class annotation

**File:** `tests/Catalog.FunctionalTests/CatalogApiTests.cs` (pattern — change mode for Post 2 demo)

```csharp
[FlushTestLogs]
[CatalogFunctionalTestMode(CatalogFunctionalTestMode.RepositoryMock)]  // Post 2: use RepositoryMock
public sealed class CatalogApiTests(CatalogApiTestSession session)
{
    private Task<CatalogApiTestHost> CreateHostAsync(
        ApiVersion apiVersion, [CallerMemberName] string testMethod = "")
    {
        var handler = new ApiVersionHandler(new QueryStringApiVersionWriter(), apiVersion);
        return _session.CreateHostAsync(GetType(), testMethod, handler);
    }
}
```

### Persistence assertion without a database

**File:** `tests/Catalog.FunctionalTests/Fixture/CatalogApiFixture.cs`

```csharp
public async Task<CatalogItem?> LoadPersistedCatalogItemAsync(int id)
{
    return _mode switch
    {
        CatalogFunctionalTestMode.RepositoryMock => _repositoryMockStore.GetItemById(id),
        // ... other modes
        _ => throw new ArgumentOutOfRangeException()
    };
}
```

**Usage in test** (`CatalogApiTests.cs`):

```csharp
var persistedItem = await host.Fixture.LoadPersistedCatalogItemAsync(id);
Assert.NotNull(persistedItem);
Assert.Equal(bodyContent.Name, persistedItem.Name);
```

### Runsettings (no Docker)

**File:** `eShop.FunctionalTests.RepositoryMock.runsettings`

```xml
<EnvironmentVariables>
  <ESHOP_CATALOG_FUNCTIONAL_TEST_MODE>RepositoryMock</ESHOP_CATALOG_FUNCTIONAL_TEST_MODE>
  <ESHOP_ORDERING_FUNCTIONAL_TEST_MODE>RepositoryMock</ESHOP_ORDERING_FUNCTIONAL_TEST_MODE>
</EnvironmentVariables>
```

---

## Post 3 — EF Core InMemory

### InMemory DI configuration

**File:** `tests/Catalog.FunctionalTests/Configuration/CatalogTestServiceConfiguration.cs`

```csharp
public static void ConfigureEfCoreInMemory(IServiceCollection services)
{
    RemoveSharedExternalServices(services);
    services.RemoveAll<ICatalogRepository>();
    services.RemoveAll<CatalogContext>();
    services.RemoveAll<InMemoryCatalogContext>();
    services.RemoveAll<DbContextOptions<CatalogContext>>();
    services.RemoveAll<DbContextOptions<InMemoryCatalogContext>>();

    services.AddDbContext<InMemoryCatalogContext>(options =>
        options.UseInMemoryDatabase(
            InMemoryDatabaseOptions.DatabaseName,
            InMemoryDatabaseOptions.DatabaseRoot));
    services.AddScoped<CatalogContext>(sp => sp.GetRequiredService<InMemoryCatalogContext>());
    services.AddScoped<ICatalogRepository, CatalogRepository>();  // production repository
    AddSharedTestDoubles(services);
}
```

### CatalogContext constructor (enables test substitution)

**File:** `src/Catalog.API/Infrastructure/CatalogContext.cs`

```csharp
[SetsRequiredMembers]
public CatalogContext(DbContextOptions options, IConfiguration configuration) : base(options)
{
}
```

Note: accepts `DbContextOptions` (non-generic) so `InMemoryCatalogContext` can substitute.

### Fixture init — seed InMemory database

**File:** `tests/Catalog.FunctionalTests/Fixture/CatalogApiFixture.cs`

```csharp
case CatalogFunctionalTestMode.EfCoreInMemory:
    _ = Services;
    await CatalogDatabaseHelper.EnsureInMemorySeededAsync(Services);
    break;
```

### Run

```bash
dotnet test tests/Catalog.FunctionalTests --settings eShop.FunctionalTests.EfCoreInMemory.runsettings
dotnet test tests/Catalog.FunctionalTests --filter-trait FunctionalTestMode=inmemory
```

---

## Post 4 — Testcontainers

### Testcontainers host

**File:** `tests/Catalog.FunctionalTests/Infrastructure/CatalogTestcontainersHost.cs`

```csharp
internal sealed class CatalogTestcontainersHost : IAsyncDisposable
{
    private PostgreSqlContainer? _postgresContainer;

    public async Task<string> StartAsync()
    {
        _postgresContainer = new PostgreSqlBuilder("ankane/pgvector:latest")
            .WithDatabase("CatalogDB")
            .WithUsername("postgres")
            .WithPassword("postgres")
            .WithCleanUp(true)
            .Build();

        await _postgresContainer.StartAsync();
        return _postgresContainer.GetConnectionString();
    }

    public async ValueTask DisposeAsync()
    {
        if (_postgresContainer is not null)
            await _postgresContainer.DisposeAsync();
    }
}
```

### Fixture wiring

**File:** `tests/Catalog.FunctionalTests/Fixture/CatalogApiFixture.cs`

```csharp
else if (_mode == CatalogFunctionalTestMode.Testcontainers)
{
    _testcontainersHost = new CatalogTestcontainersHost();
}

// InitializeAsync:
case CatalogFunctionalTestMode.Testcontainers:
    _postgresConnectionString = await _testcontainersHost!.StartAsync();
    _ = Services;
    await CatalogDatabaseHelper.EnsurePostgresSeededAsync(Services);
    break;
```

### Run

```bash
dotnet test tests/Catalog.FunctionalTests --settings eShop.FunctionalTests.Testcontainers.runsettings
dotnet test tests/Catalog.FunctionalTests --filter-trait FunctionalTestMode=testcontainers
```

---

## Post 5 — Aspire in tests

### The hybrid model

- **`WebApplicationFactory<Program>`** hosts Catalog.API (the system under test).
- **`CatalogAspireTestHost`** (Aspire `DistributedApplication`) hosts PostgreSQL (+ optional RabbitMQ).

**File:** `tests/Catalog.FunctionalTests/Infrastructure/CatalogAspireTestHost.cs`

```csharp
public CatalogAspireTestHost(Assembly testAssembly, bool includeRabbitMq = false)
{
    var options = new DistributedApplicationOptions
    {
        AssemblyName = testAssembly.FullName,
        DisableDashboard = true
    };

    var appBuilder = DistributedApplication.CreateBuilder(options);
    Postgres = appBuilder.AddPostgres("CatalogDB")
        .WithImage("ankane/pgvector")
        .WithImageTag("latest");

    if (includeRabbitMq)
        EventBus = appBuilder.AddRabbitMQ("eventbus");

    _app = appBuilder.Build();
}
```

```csharp
public async Task<CatalogAspireEndpoints> StartAsync(bool waitForEventBus, ...)
{
    await _app.StartAsync(cancellationToken);

    var resourceNotifications = _app.Services.GetRequiredService<ResourceNotificationService>();
    await resourceNotifications.WaitForResourceHealthyAsync(Postgres.Resource.Name, timeout.Token);

    // ... optional RabbitMQ health wait ...

    return new CatalogAspireEndpoints(
        await Postgres.Resource.GetConnectionStringAsync(cancellationToken),
        eventBusConnectionString);
}
```

### Session — lazy fixture per mode

**File:** `tests/Catalog.FunctionalTests/Fixture/CatalogApiTestSession.cs`

```csharp
public sealed class CatalogApiTestSession : IAsyncDisposable
{
    private readonly ConcurrentDictionary<CatalogFunctionalTestMode, Lazy<Task<CatalogApiFixture>>> _fixtures = new();

    public async Task<CatalogApiTestHost> CreateHostAsync(
        Type testClass, string testMethodName, DelegatingHandler? handler = null, ...)
    {
        var mode = CatalogFunctionalTestModeAttribute.Resolve(testClass, testMethodName, bindingFlags);
        var effectiveMode = FunctionalTestModeReader.ReadOverrideFromEnvironment() ?? mode;
        var fixture = await GetOrCreateFixtureAsync(effectiveMode);
        var client = fixture.CreateLoggedClient(handler!);
        return new CatalogApiTestHost(fixture, client);
    }
}
```

### Mode attribute + xUnit traits

**File:** `tests/Catalog.FunctionalTests/Fixture/CatalogFunctionalTestModeAttribute.cs`

```csharp
[CatalogFunctionalTestMode(CatalogFunctionalTestMode.Aspire)]
public sealed class CatalogApiTests(...) { ... }

// Trait for filtering:
public IReadOnlyCollection<KeyValuePair<string, string>> GetTraits() =>
[
    new(FunctionalTestModeTrait.Name, FunctionalTestModeTrait.ToTraitValue(Mode))
];
// FunctionalTestMode=aspire | mock | inmemory | testcontainers | ...
```

### Assembly fixture registration

**File:** `tests/Catalog.FunctionalTests/AssemblyInfo.cs`

```csharp
[assembly: Xunit.CaptureConsole]
[assembly: Xunit.CaptureTrace]
[assembly: Xunit.AssemblyFixture(typeof(eShop.Catalog.FunctionalTests.CatalogApiTestSession))]
```

### Run

```bash
dotnet test tests/Catalog.FunctionalTests --settings eShop.FunctionalTests.Aspire.runsettings
dotnet test tests/Catalog.FunctionalTests --filter-trait FunctionalTestMode=aspire
```

---

## Post 6 — EventBus & transactional outbox

### Event type discovery fix (required for WAF test host)

**File:** `src/IntegrationEventLogEF/Services/IntegrationEventLogService.cs`

```csharp
private static Type[] GetIntegrationEventTypes()
{
    return AppDomain.CurrentDomain.GetAssemblies()
        .Where(static assembly => !assembly.IsDynamic)
        .SelectMany(static assembly =>
        {
            try { return assembly.GetTypes(); }
            catch (ReflectionTypeLoadException ex)
            {
                return ex.Types.Where(static type => type is not null)!;
            }
        })
        .Where(static type => type is not null
            && typeof(IntegrationEvent).IsAssignableFrom(type)
            && !type.IsAbstract)
        .Distinct()
        .ToArray();
}
```

### Spy bus mode — outbox + CapturingEventBus

**File:** `tests/Catalog.FunctionalTests/Configuration/CatalogMessagingTestServiceConfiguration.cs`

```csharp
public static void ConfigureOutboxWithSpyBus(IServiceCollection services)
{
    RemoveRabbitMqInfrastructure(services);

    services.AddSingleton<CapturingEventBus>();
    services.AddSingleton<IEventBus>(sp => sp.GetRequiredService<CapturingEventBus>());
    services.RemoveAll<ICatalogAI>();
    services.AddSingleton<ICatalogAI, FakeCatalogAI>();
}
```

**File:** `tests/Testing.Common/Messaging/CapturingEventBus.cs`

```csharp
public sealed class CapturingEventBus : IEventBus
{
    private readonly ConcurrentQueue<IntegrationEvent> _published = new();
    public IReadOnlyCollection<IntegrationEvent> Published => _published.ToArray();

    public Task PublishAsync(IntegrationEvent @event)
    {
        _published.Enqueue(@event);
        return Task.CompletedTask;
    }
}
```

### Outbox assertion helper

**File:** `tests/Testing.Common/Messaging/OutboxAssertions.cs`

```csharp
public static async Task<IReadOnlyList<IntegrationEventLogEntry>> GetPublishedEventsAsync<TContext>(
    IServiceProvider services,
    CancellationToken cancellationToken = default,
    params string[] eventTypeShortNames)
    where TContext : DbContext
{
    await using var scope = services.CreateAsyncScope();
    var context = scope.ServiceProvider.GetRequiredService<TContext>();

    var query = context.Set<IntegrationEventLogEntry>()
        .AsNoTracking()
        .Where(entry => entry.State == EventStateEnum.Published);
    // ... filter by eventTypeShortNames ...
    return await query.OrderBy(entry => entry.CreationTime).ToListAsync(cancellationToken);
}
```

### Catalog messaging test (outbox + spy)

**File:** `tests/Catalog.FunctionalTests/CatalogMessagingTests.cs`

```csharp
[Fact]
[CatalogFunctionalTestMode(CatalogFunctionalTestMode.AspireMessagingOutbox)]
public async Task UpdateCatalogItem_PublishesPriceChangedEventToOutboxAndSpyBus()
{
    var host = await _session.CreateHostAsync(GetType(), nameof(...), handler);
    var capturingBus = host.Fixture.Services.GetRequiredService<CapturingEventBus>();
    capturingBus.Reset();

    // ... PUT price update ...

    var outboxEvents = await OutboxAssertions.GetPublishedEventsAsync<CatalogContext>(
        host.Fixture.Services, cancellationToken,
        nameof(ProductPriceChangedIntegrationEvent));

    Assert.Single(outboxEvents);
    Assert.Contains(capturingBus.Published, @event =>
        @event is ProductPriceChangedIntegrationEvent priceChanged
        && priceChanged.ProductId == itemToUpdate.Id);
}
```

### RabbitMQ capture mode

**File:** `tests/Testing.Common/Messaging/IntegrationEventCapture.cs`

```csharp
public static async Task WaitForCountAsync(int expectedCount, TimeSpan timeout, ...)
{
    while (!timeoutSource.IsCancellationRequested)
    {
        if (Events.Count >= expectedCount) return;
        await Task.Delay(100, timeoutSource.Token);
    }
    throw new TimeoutException($"Timed out waiting for {expectedCount} events. Captured {Events.Count}.");
}
```

### Run messaging tests

```bash
dotnet test tests/Catalog.FunctionalTests --settings eShop.FunctionalTests.AspireMessagingOutbox.runsettings
dotnet test tests/Catalog.FunctionalTests --filter-trait FunctionalTestMode=aspire-messaging-outbox
```

---

## Shared utilities (mention in any post)

**Test logging** — `tests/Testing.Common/TestLogging.cs`:

```csharp
public static HttpClient CreateLoggedClient<TEntryPoint>(
    this WebApplicationFactory<TEntryPoint> factory, DelegatingHandler innerHandler)
    where TEntryPoint : class
    => factory.CreateDefaultClient(innerHandler, new HttpTrafficLoggingHandler());
```

**README quick reference** — `tests/README.md` documents all modes and trait filters.
