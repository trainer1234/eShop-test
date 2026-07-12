# Post 4 Draft — Building API Integration Tests in .NET: Aspire on the Fidelity Ladder

> **Status:** Draft for NashTech Blog  
> **Series:** Post 4 of 5 (+ optional benchmark post)  
> **Demo project:** [trainer1234/eShop-test](https://github.com/trainer1234/eShop-test) (eShop fork with pluggable test modes)  
> **Related:** `docs/post-1-fidelity-ladder.md`, `docs/post-3-testcontainers.md`, `docs/code-snippets-per-post.md`, `docs/multi-mode-functional-testing.md`

---

## Metadata (for publication)

**Suggested title:** Building API Integration Tests in .NET — Aspire on the Fidelity Ladder

**Tags:** Application Engineering, Quality Solutions, .NET, Integration Testing, Aspire, Docker

**Author:** Minh Kha Giai

**Estimated read time:** 17–19 minutes

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [A quick review from Posts 1–3](#2-a-quick-review-from-posts-13)
3. [What is Aspire?](#3-what-is-aspire)
4. [Two ways to use Aspire in tests](#4-two-ways-to-use-aspire-in-tests)
5. [When to climb to this rung](#5-when-to-climb-to-this-rung)
6. [The eShop hybrid model](#6-the-eshop-hybrid-model)
7. [In Action: Catalog and Ordering Aspire hosts](#7-in-action-catalog-and-ordering-aspire-hosts)
8. [Managing resources before startup](#8-managing-resources-before-startup)
9. [Running, traits, and workflow](#9-running-traits-and-workflow)
10. [Final act: Conclusion](#10-final-act-conclusion)
11. [References](#11-references)

---

## 1. Introduction

[Post 3](post-3-testcontainers.md) climbed the ladder with **Testcontainers** — imperative Docker containers you start and wire yourself. That works well for one or two dependencies. It gets repetitive when your test topology mirrors a development AppHost: multiple databases, a message broker, and a sibling .NET service like Identity.API.

The fourth rung adds **Aspire orchestration inside the test fixture** — declarative resources, built-in health notifications, and connection strings from the hosting layer.

| Rung | What you gain | What you still defer |
|------|---------------|----------------------|
| **Aspire (in tests)** | Resource graph, health waits, `AddProject` for live services | Full closed-box AppHost testing (optional separate style) |
| **Testcontainers** | Direct control per container | Manual orchestration for multi-service graphs |

**Prerequisites:** Docker running (same as Post 3). Section 3 summarizes Aspire for readers new to the topic; deeper introductions are in [References](#11-references).

---

## 2. A quick review from Posts 1–3

| Concept | Summary |
|---------|---------|
| **`WebApplicationFactory`** | Hosts the API under test in-process — `ConfigureTestServices` remains the side door for test doubles |
| **Testcontainers** | Start containers imperatively; you inject connection strings |
| **Why Aspire next** | Several wired dependencies + health coordination; optional live .NET sibling services (Identity.API) |

eShop keeps the same HTTP tests across rungs. Aspire mode changes **how dependencies are provisioned**, not the test methods themselves.

---

## 3. What is Aspire?

**.NET Aspire** is Microsoft's **local orchestration layer for cloud-native .NET apps**. You describe your system in an **AppHost** project — databases, caches, message brokers, and .NET services — as a graph of resources (`AddPostgres`, `AddRabbitMQ`, `AddProject`, `WithReference`). Running the AppHost starts those resources (typically in Docker), wires connection strings and endpoints between services, and exposes a **dashboard** for logs and health.

Think of it as: **docker-compose-like ergonomics, but modeled in C# with first-class .NET service references** — aimed at how you develop and test distributed applications, not at replacing your production deploy target (Kubernetes, Azure Container Apps, etc.).

This series uses Aspire **inside test fixtures** for the same reason teams use it in development: **less manual glue** when several dependencies must start in the right order and talk to each other.

### Benefits

| Benefit | What it means in practice |
|---------|---------------------------|
| **Declarative topology** | Dependencies and references live in code beside your solution — reviewable, refactorable |
| **Service discovery** | Connection strings and URLs flow to projects without hard-coding localhost ports |
| **Health and observability** | Dashboard and resource notifications show what is running and what failed |
| **Aligned dev and test vocabulary** | The same `AddPostgres` / `AddProject` patterns can appear in AppHost and test hosts |
| **Testing support** | `Aspire.Hosting.Testing` can launch and manipulate the AppHost for integration tests |

### Limitations

| Limitation | What to watch for |
|------------|-------------------|
| **Docker-heavy** | Most resources run as containers — same machine requirements as Testcontainers |
| **Heavier than one container** | A single Postgres in Testcontainers is simpler than pulling in Aspire hosting packages |
| **Another abstraction** | Teams must learn AppHost, resources, and how references map to configuration |
| **Not a deploy button** | Aspire orchestrates local/dev/test workflows; production pipelines still need their own story |
| **Graph drift** | Test-side `DistributedApplication` graphs (eShop hybrid) can diverge from production AppHost without discipline |

For full Aspire primers — getting started, dashboard, cloud-native positioning — see the posts in [References](#11-references). The rest of this post assumes that context and focuses on **testing on the fidelity ladder**.

---

## 4. Two ways to use Aspire in tests

Microsoft documents Aspire testing in the [Testing overview](https://aspire.dev/testing/overview/). There are two distinct shapes worth separating before you copy a sample from the wrong blog post.

### A — Closed-box: `DistributedApplicationTestingBuilder` (full AppHost)

This is **Microsoft's documented Aspire testing model** — the path described in the [Testing overview](https://aspire.dev/testing/overview/), the [write your first test](https://aspire.dev/testing/write-your-first-test/) tutorial, and the Aspire solution test-project template. The [`Aspire.Hosting.Testing`](https://www.nuget.org/packages/Aspire.Hosting.Testing) package launches your **real AppHost project** in a background thread. Tests send HTTP to services running as **separate processes** — Frontend → API → Database, matching full distributed topology.

```
Test project → AppHost process → Database, API, Frontend (all separate processes)
```

Typical setup:

```csharp
var appHost = await DistributedApplicationTestingBuilder
    .CreateAsync<Projects.MyAppHost>();

await using var app = await appHost.BuildAsync();
await app.StartAsync();

var httpClient = app.CreateHttpClient("api");
var response = await httpClient.GetAsync("/health");
```

**Strengths:** End-to-end verification across the full distributed application. Realistic networking and service discovery. Resource manipulation before startup (Section 8) is designed for this model.

**Limits:** No `ConfigureTestServices` on individual APIs — tests run out-of-process. Microsoft [documents](https://aspire.dev/testing/overview/) that mocking or replacing DI registrations is not supported here. You influence behavior through environment variables, AppHost arguments, and changes on the testing builder.

**Use when:** You want to prove the **whole AppHost graph** works together — the scenario Aspire's testing docs and templates target.

### B — Open-box hybrid: `WebApplicationFactory` + `DistributedApplication` (eShop)

eShop uses a **hybrid** aligned with Posts 1–3:

```
Test project → WebApplicationFactory (Catalog.API / Ordering.API in-process)
            → DistributedApplication in fixture (Postgres, RabbitMQ, Identity.API only)
```

- **`WebApplicationFactory<Program>`** still hosts the **system under test** in-process — same `ConfigureTestServices`, same test doubles, same debugger experience as Mock and Testcontainers modes.
- **`CatalogAspireTestHost` / `OrderingAspireTestHost`** spin up a **minimal** `DistributedApplication` that provisions **dependencies only** — not Catalog.API or Ordering.API as Aspire projects.

**Strengths:** Same test class and DI swapping as lower rungs. Aspire handles Postgres (+ optional RabbitMQ), health waits, and Ordering's live **Identity.API** via `AddProject`.

**Limits:** You maintain a test-side resource graph that should stay aligned with AppHost — not automatic parity with production AppHost unless you discipline it.

**Use when:** API integration tests are the focus and you need orchestration without giving up `WebApplicationFactory`.

This series implements **B** in the demo repo. **A** is the path to grow toward for full-stack closed-box suites — and it is where Microsoft's resource-manipulation APIs in Section 8 are most fully documented.

---

## 5. When to climb to this rung

### When to use it

- **Multiple dependencies** wired together — Ordering needs Postgres **and** a live Identity token endpoint.
- **Health coordination** — wait until resources are healthy before HTTP assertions, without hand-written retry loops on every container.
- **Team already uses Aspire in development** — test fixtures reuse familiar `AddPostgres`, `AddRabbitMQ`, `AddProject`, `WithReference` vocabulary.
- **Messaging tests** (Post 5) — RabbitMQ alongside Postgres in the same Aspire host.

### Blind spots

- **Slower cold start** than Testcontainers-only (~15–45 s vs ~10–30 s in eShop runs — measure on your hardware).
- **More packages** — `Aspire.Hosting`, hosting integrations, RabbitMQ hosting modules in test projects.
- **Two Aspire stories** — readers may conflate full AppHost testing (A) with the eShop hybrid (B). Pick deliberately.

### Aspire vs Testcontainers (this series)

| Question | Testcontainers | Aspire in tests |
|----------|----------------|-----------------|
| Who wires multiple deps? | Your code / shared helper | Aspire hosting APIs |
| Live .NET sibling API | You host it yourself | `AddProject<Identity_API>()` |
| API under test | `WebApplicationFactory` | `WebApplicationFactory` (eShop) |
| Health waits | Custom | `WaitForResourceHealthyAsync` |

---

## 6. The eShop hybrid model

End-to-end flow for Catalog Aspire mode:

```
1. CatalogApiFixture constructed (mode = Aspire)
2. InitializeAsync:
     CatalogAspireTestHost.StartAsync()
       → DistributedApplication starts Postgres (pgvector image)
       → WaitForResourceHealthyAsync("CatalogDB")
       → return connection string
3. WebApplicationFactory boots with connection string in IConfiguration
4. EnsurePostgresSeededAsync (migrations + seed)
5. Tests send HTTP via logged HttpClient
6. DisposeAsync stops DistributedApplication
```

Ordering adds **IdentityDB** + **Identity.API** as Aspire projects; the fixture injects `Identity:Url` from the running Identity endpoint.

The API under test never runs as an Aspire `AddProject` — that is intentional (see Post 1 / hybrid rationale): keep `ConfigureTestServices` and in-process debugging.

---

## 7. In Action: Catalog and Ordering Aspire hosts

### Catalog — Postgres (+ optional RabbitMQ for Post 5)

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

Start and wait for health:

```csharp
await _app.StartAsync(cancellationToken);

var resourceNotifications = _app.Services.GetRequiredService<ResourceNotificationService>();
await resourceNotifications.WaitForResourceHealthyAsync(Postgres.Resource.Name, timeout.Token);

return new CatalogAspireEndpoints(
    await Postgres.Resource.GetConnectionStringAsync(cancellationToken),
    eventBusConnectionString);
```

Connection string injection uses the **Aspire resource name** as the configuration key:

```csharp
settings[$"ConnectionStrings:{options.PostgresResourceName}"] = options.PostgresConnectionString;
```

Testcontainers mode used `ConnectionStrings:catalogdb` instead — same database, different wiring convention.

### Ordering — Postgres + Identity.API

```csharp
var appBuilder = DistributedApplication.CreateBuilder(options);
Postgres = appBuilder.AddPostgres("OrderingDB");
IdentityDB = appBuilder.AddPostgres("IdentityDB");
IdentityApi = appBuilder.AddProject<Projects.Identity_API>("identity-api")
    .WithReference(IdentityDB);
```

The fixture reads the live Identity endpoint:

```csharp
public string IdentityApiUrl => IdentityApi.GetEndpoint("http").Url;

// OrderingTestHostConfiguration:
settings["Identity:Url"] = options.IdentityApiUrl ?? "http://localhost/identity";
```

Testcontainers can provide the same fidelity by running an Identity.API container image next to Postgres. The difference is the setup: you must build or obtain the image, start it, wait for readiness, and inject its endpoint yourself. Aspire's `AddProject<Projects.Identity_API>()` starts the .NET project directly and wires its database reference and endpoint through the resource graph. The distinction is therefore **who owns the service orchestration**, not whether Testcontainers can run the service.

### Fixture switch — same tests

```csharp
[CatalogFunctionalTestMode(CatalogFunctionalTestMode.Aspire)]
public sealed class CatalogApiTests(CatalogApiTestSession session) { ... }

[OrderingFunctionalTestMode(OrderingFunctionalTestMode.Aspire)]
public sealed class OrderingApiTests(OrderingApiTestSession session) { ... }
```

`CatalogApiTestSession` lazily creates one fixture per mode — Aspire cold start is paid once per test run, not per test method.

Aspire mode still uses `ConfigureSharedExternalDependencies` for Catalog/Ordering API tests — fake AI, no-op messaging — unless you switch to messaging modes in Post 5.

---

## 8. Managing resources before startup

Aspire's testing docs emphasize something easy to miss: you can **shape the resource graph before the app fully starts** — skipping dependencies, simulating failure, or overriding configuration. That is how you move from "happy path only" toward **resilience and chaos-style** integration tests.

Microsoft covers this in [Testing overview](https://aspire.dev/testing/overview/) and [Advanced testing scenarios](https://aspire.dev/testing/advanced-scenarios/). Patterns fall into two buckets depending on which Aspire testing shape you use.

### Full AppHost testing (`DistributedApplicationTestingBuilder`)

After `CreateAsync<Projects.MyAppHost>()` and **before** `BuildAsync()` / `StartAsync()`, the testing builder exposes the AppHost's resources for mutation:

| Technique | Purpose | Example |
|-----------|---------|---------|
| **`WithExplicitStart` in AppHost** | Resource exists but does not start until a test starts it | Run API tests without Grafana/monitoring sidecars |
| **AppHost arguments** | Conditionally skip resources | `CreateAsync(["AddDatabase=false"])` — assert postgres absent |
| **`CreateResourceBuilder<T>(name)`** | Mutate a named resource before start | `WithEnvironment(...)` for feature flags on the API project |
| **Remove `WaitAnnotation`** | Start without waiting for dependencies | Test behavior when startup order is wrong or a dep is down |
| **Break a container entrypoint** | Simulate unavailable infrastructure | `CreateResourceBuilder<ContainerResource>("cache").WithEntrypoint("sleep 1d")` — health checks fail while Redis is "unavailable" ([API reference example](https://aspire.dev/reference/api/csharp/aspire.hosting/distributedapplicationbuilderextensions/methods/)) |

Example — API healthy without optional monitoring ([Advanced scenarios](https://aspire.dev/testing/advanced-scenarios/)):

```csharp
var appHost = await DistributedApplicationTestingBuilder
    .CreateAsync<Projects.MyAppHost>();

await using var app = await appHost.BuildAsync();
await app.StartAsync();

// "monitoring" marked WithExplicitStart in AppHost — not started
await app.ResourceNotifications.WaitForResourceHealthyAsync("api", cts.Token);
```

Example — simulate Redis unavailable for health-check testing:

```csharp
var appHost = await DistributedApplicationTestingBuilder.CreateAsync<Projects.MyAppHost>();

appHost.CreateResourceBuilder<ContainerResource>("cache")
    .WithEntrypoint("sleep 1d");  // container runs but does not serve Redis

await using var app = await appHost.BuildAsync();
await app.StartAsync();

var response = await httpClient.GetAsync("/health");
Assert.Equal(HttpStatusCode.ServiceUnavailable, response.StatusCode);
```

**Caution from the docs:** removing wait annotations or starting resources without dependencies is **intentional chaos** — use it to assert degraded behavior (timeouts, 503s, retry policies), not as the default happy-path setup.

For lifecycle hooks before any resource is created, see [Manage the AppHost in tests](https://aspire.dev/testing/manage-app-host/) (`DistributedApplicationFactory`, `OnBuilderCreating`).

### eShop hybrid — mutate the inline `DistributedApplication` builder

eShop does not launch the full eShop AppHost in tests. Instead, `CatalogAspireTestHost` / `OrderingAspireTestHost` call `DistributedApplication.CreateBuilder` **inside the test assembly**. The same *ideas* apply **on `appBuilder` before `Build()`**:

| Technique | How in eShop hybrid |
|-----------|---------------------|
| **Optional RabbitMQ** | `includeRabbitMq` flag — resource omitted entirely unless messaging mode |
| **`WithExplicitStart`** | Mark Postgres or Identity as explicit-start; test code chooses when to `StartAsync` on that resource |
| **Wrong image / bad credentials** | `.WithImage("postgres:invalid-tag")` — assert fixture startup fails or API returns errors |
| **Skip `WaitForResourceHealthyAsync`** | Deliberately boot `WebApplicationFactory` before Postgres is ready — assert retry or failure paths |
| **Inject bad connection string** | Pass garbage in `CatalogTestHostConfiguration` after start — test API error handling without stopping the container |

Because the API runs in `WebApplicationFactory`, chaos tests on **dependency unavailability** combine Aspire resource control (broken broker, late DB) with HTTP assertions and optional test doubles — something closed-box AppHost tests approach only via env vars and external HTTP.

> **Optional reading — chaos and failure injection**  
> eShop's repo does not ship chaos tests today. This section maps official Aspire capabilities to experiments you can add: broken cache containers (full AppHost), omitted RabbitMQ (hybrid), Identity started late (Ordering). Treat these as advanced suites — keep happy-path Aspire tests stable in CI and run failure scenarios in a separate trait or nightly job.

---

## 9. Running, traits, and workflow

### Runsettings

```xml
<ESHOP_CATALOG_FUNCTIONAL_TEST_MODE>Aspire</ESHOP_CATALOG_FUNCTIONAL_TEST_MODE>
<ESHOP_ORDERING_FUNCTIONAL_TEST_MODE>Aspire</ESHOP_ORDERING_FUNCTIONAL_TEST_MODE>
```

File: `eShop.FunctionalTests.Aspire.runsettings`

### CLI

```bash
dotnet test tests/Catalog.FunctionalTests --settings eShop.FunctionalTests.Aspire.runsettings
dotnet test tests/Ordering.FunctionalTests --filter-trait FunctionalTestMode=aspire
```

Trait values: `aspire`, `aspire-messaging-outbox`, `aspire-messaging-rabbitmq` (Post 5).

### Suggested workflow

| When | Mode |
|------|------|
| Local edit loop | Repository Mock or EF InMemory (Post 2) |
| PR validation — SQL | Testcontainers subset (Post 3) |
| Nightly / pre-release | Aspire (+ messaging traits in Post 5) |
| Resilience / chaos experiments | Aspire with resource manipulation (Section 8), separate job |

Enable the Aspire dashboard during a failing local run by setting `DisableDashboard = false` in `DistributedApplicationOptions` — useful when a resource never reaches healthy state.

### Pros and cons (this rung)

| Pros | Cons |
|------|------|
| Declarative multi-resource graph | Heavier packages and cold start |
| `WaitForResourceHealthyAsync` | Hybrid graph can drift from production AppHost |
| `AddProject` for Identity.API | Two Aspire testing models to understand |
| Same HTTP tests as lower rungs | Closed-box AppHost tests need a separate project setup |

---

## 10. Final act: Conclusion

Aspire on the fidelity ladder is **orchestration for test dependencies** — not a replacement for `WebApplicationFactory` in the eShop hybrid, and not the same as running your entire AppHost under `DistributedApplicationTestingBuilder`.

- **Testcontainers (Post 3):** imperative containers, maximum control per image.
- **Aspire (this post):** coordinated resources, health waits, live sibling services — with optional **pre-start manipulation** for failure and chaos scenarios documented by Microsoft.

Post 5 stays on this rung and adds **messaging fidelity** — outbox assertions with a spy bus or real RabbitMQ in the same Aspire host.

**Final question:** does your AppHost topology fit in a single Testcontainers helper — or do you need Aspire's graph (and possibly resource manipulation) to test it honestly?

---

## 11. References

**This series**

- [Post 1 — A Test Fidelity Ladder](post-1-fidelity-ladder.md)
- [Post 2 — Repository Mock and EF InMemory](post-2-mock-and-inmemory.md)
- [Post 3 — Testcontainers](post-3-testcontainers.md)
- `docs/multi-mode-functional-testing.md` — fixture architecture

**Demo repo**

- [trainer1234/eShop-test on GitHub](https://github.com/trainer1234/eShop-test)
- [dotnet/eShop on GitHub](https://github.com/dotnet/eshop) — upstream reference application

**Microsoft Aspire — testing**

- [Testing overview](https://aspire.dev/testing/overview/) — `DistributedApplicationTestingBuilder`, closed-box model, configuration
- [Advanced testing scenarios](https://aspire.dev/testing/advanced-scenarios/) — `WithExplicitStart`, conditional resources, wait annotation removal, env overrides
- [Manage the AppHost in tests](https://aspire.dev/testing/manage-app-host/) — `DistributedApplicationFactory`, lifecycle hooks
- [Integration tests in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests) — `WebApplicationFactory` (eShop hybrid base)

**Related NashTech posts (Aspire introductions — read these for depth)**

- [Accelerating Cloud-Native Application Development with .NET Aspire](https://blog.nashtechglobal.com/accelerating-cloud-native-application-development-with-net-aspire/)
- [ASPIRE – The Secret Weapon for .NET Cloud-Native Developers](https://blog.nashtechglobal.com/aspire-the-secret-weapon-for-net-cloud-native-developers/)
- [Build your first Aspire app](https://aspire.dev/get-started/first-app/) — Microsoft getting started
