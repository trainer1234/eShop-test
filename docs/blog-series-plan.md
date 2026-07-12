# Blog Series Plan — Building API Tests with Mock / EF Core InMemory / Testcontainers / Aspire

> Planning document for a NashTech blog series demonstrating API integration testing
> strategies on the [dotnet/eShop](https://github.com/dotnet/eshop) reference app.
> Demo code and pluggable test modes: [trainer1234/eShop-test](https://github.com/trainer1234/eShop-test).

---

## Positioning

**Theme:** Not "how to use X in isolation," but **how to choose and combine test modes on a
real microservices application** — a *test fidelity ladder* running from Mock → EF Core
InMemory → Testcontainers → Aspire, plus messaging/outbox testing and benchmarks.

### What already exists on blog.nashtechglobal.com (do NOT repeat)

| Existing post | Already covers | We must avoid |
|---------------|----------------|---------------|
| [Accelerating Cloud-Native Development with .NET Aspire](https://blog.nashtechglobal.com/accelerating-cloud-native-application-development-with-net-aspire/) | Aspire intro, getting started | Another "What is Aspire?" primer |
| [ASPIRE – The Secret Weapon for .NET Cloud-Native Developers](https://blog.nashtechglobal.com/aspire-the-secret-weapon-for-net-cloud-native-developers/) | Docker Compose vs Aspire, service discovery, dashboard, K8s deploy | Another Docker Compose vs Aspire debate |
| [Integration Testing in .NET with Test Containers](https://blog.nashtechglobal.com/integration-testing-in-net-with-test-containers/) | WAF + Testcontainers + Respawn on a blog app | A "first Testcontainers project" tutorial |
| [Playwright – An Introduction](https://blog.nashtechglobal.com/playwright-an-introduction/) | E2E UI framework intro | UI testing content |

**Our differentiator:** microservices (Catalog + Ordering), pgvector semantic search,
transactional outbox + RabbitMQ, and a *pluggable* mode architecture with benchmarks.

---

## House style (from prior posts)

Reuse the structure that works in the Playwright and Maestro articles:

1. Hook + problem statement.
2. A short **"What is X?"** box for readers with no prior knowledge (2–3 paragraphs max).
3. Numbered sections/acts with a Table of Contents.
4. **Pros / cons / when-to-use** lists — honest, not just praise.
5. A **"In Action"** section with real code from eShop.
6. A **comparison table**.
7. **References** at the end.
8. A conclusion that ends with a memorable question.

---

## The series (5 posts + 1 optional benchmark)

Publish in order. Each post stands alone but links forward and back.

### Post 1 — Foundation / series opener
**Title:** *Building API Integration Tests in .NET — A Test Fidelity Ladder (Mock to Aspire)*
- The dilemma: green CI, broken staging.
- What is an integration test for APIs? Stub vs mock vs fake vs test double.
- What is `WebApplicationFactory`?
- The fidelity ladder diagram (Mock → InMemory → Testcontainers → Aspire).
- eShop as the demo app.
- Series map.
- **Demo code:** one diagram + one comparison table. No Docker needed to finish reading.
- **Draft:** see `docs/post-1-fidelity-ladder.md`.

### Post 2 — Bottom two rungs (Repository Mock + EF InMemory)
**Title:** *Building API Integration Tests in .NET — Repository Mock and EF InMemory: The Bottom Two Rungs*
- Unit-test mock (Moq/NSubstitute) vs integration-test fake — why not stub `ICatalogRepository` with NSubstitute.
- Unit-test mock vs integration-test fake (same word, different boundary).
- When repository-fake tests shine, and when they lie.
- EF Core InMemory: what it is and is not (provider limitations, pgvector caveat).
- Mock vs InMemory decision table and pros/cons.
- In Action: `ICatalogRepository`, `ConfigureRepositoryMock`, `ConfigureEfCoreInMemory`, `InMemoryCatalogContext`, seeding, persistence assertions.
- Same `CatalogApiTests` class, two mode attributes / runsettings.
- Run with `eShop.FunctionalTests.RepositoryMock.runsettings` and `eShop.FunctionalTests.EfCoreInMemory.runsettings`.
- **Draft:** see `docs/post-2-mock-and-inmemory.md`.

### Post 3 — Real database, minimal orchestration
**Title:** *Building API Integration Tests in .NET — Testcontainers on the Fidelity Ladder*
- What Testcontainers is (2 paragraphs) — any containerized dependency, not Postgres-only.
- What can you containerize? — module table (Postgres, RabbitMQ, Redis, …).
- The generic integration pattern; PostgreSQL + pgvector as eShop case study.
- Shared fixture vs per-test isolation (mention Respawn as an alternative).
- CI notes and GitHub Actions snippet.
- **Unique vs Divyesh's post:** microservices + pgvector + mode switching.
- **Draft:** see `docs/post-3-testcontainers.md`.

### Post 4 — Full-stack fidelity (Aspire in tests)
**Title:** *Using .NET Aspire Inside API Tests — WebApplicationFactory Meets DistributedApplication*
- The hybrid: WAF hosts the API; Aspire hosts the dependencies.
- What Aspire gives tests: Postgres (+ RabbitMQ), health waits, connection strings.
- What Aspire in tests does **not** replace (testing lens only).
- In Action: `CatalogAspireTestHost`, `WaitForResourceHealthyAsync`.
- Mode attribute + xUnit traits + `.runsettings`.
- Workflow: mock locally, Aspire in CI nightly.
- **Avoid** the Docker Compose vs Aspire essay (already covered); one "see also" link only.

### Post 5 — Messaging & outbox (advanced)
**Title:** *Testing EventBus and the Transactional Outbox in .NET — Spy Bus vs Real RabbitMQ*
- What the transactional outbox is (plain language).
- Why entry-assembly event discovery breaks under WAF, and the fix.
- Two messaging modes: `AspireMessagingOutbox` (CapturingEventBus) vs
  `AspireMessagingRabbitMq` (IntegrationEventCapture).
- In Action: Catalog price change + Ordering create order.
- Asserting outbox rows + published events.

### Post 6 (optional) — Benchmark & CI strategy
**Title:** *Benchmarking API Test Modes — Mock vs InMemory vs Testcontainers vs Aspire*
- Methodology: same suite, different `--settings` / traits.
- Metrics: cold start, warm run, Docker pull, CI minutes.
- Results table (publish real numbers).
- Recommended matrix (below).
- **Note:** run benchmarks before writing.

```
Local dev loop        → RepositoryMock / EfCoreInMemory
PR validation         → RepositoryMock + a subset of Testcontainers
Nightly / pre-release → Aspire (+ messaging modes)
```

---

## Reusable "Before you read" primer box (posts 2–6)

| Term | One-line explanation |
|------|----------------------|
| `WebApplicationFactory` | In-process test server for your ASP.NET Core app |
| `ConfigureTestServices` | Swap DI registrations for test doubles |
| Testcontainers | Programmatic Docker containers for tests |
| Aspire (in this series) | Orchestrates Postgres/RabbitMQ **for tests**, not your production deploy |
| Outbox | A DB table written in the same transaction as business data, then published |

---

## Per-post Table of Contents template

```
1. Introduction / The dilemma
2. What is [concept]?
3. Why do we need it?
4. Pros, cons, and when to use
5. [Project] In Action
   5.1 Setup
   5.2 Test mode configuration
   5.3 Sample test walkthrough
   5.4 Running from CLI and Visual Studio
6. Compare with other modes (table)
7. Conclusion
References
```

---

## Content calendar

| Week | Post | Effort | Depends on |
|------|------|--------|------------|
| 1 | Post 1 — Fidelity ladder | Low | this plan + Post 1 draft |
| 2 | Post 2 — Mock + InMemory | Medium | eShop Catalog mock + InMemory modes |
| 3 | Post 3 — Testcontainers | Medium | differentiate from Divyesh |
| 4 | Post 4 — Aspire in tests | High | unique angle |
| 5 | Post 5 — EventBus/outbox | High | messaging tests |
| 6 | Post 6 — Benchmark (optional) | Medium | run measurements first |

---

## Repo assets to prepare

| Asset | Purpose |
|-------|---------|
| `docs/blog-series-plan.md` | This plan |
| `docs/post-1-fidelity-ladder.md` | Post 1 full draft |
| `docs/post-2-mock-and-inmemory.md` | Post 2 full draft |
| `docs/post-3-testcontainers.md` | Post 3 full draft |
| `docs/code-snippets-per-post.md` | Copy-paste-ready snippets per post |
| Screenshots: Test Explorer traits, Aspire dashboard in a test run, Docker containers | Visual proof |
| Benchmark script (`dotnet test` × 4 runsettings, timed) | Post 7 |
| A GitHub branch/tag per post | Readers can check out matching code |

---

## One-line pitch

> *Same eShop API tests, four levels of reality — learn when to mock, when to fake the
> database, when to containerize Postgres, and when to orchestrate with Aspire, including
> how to test RabbitMQ and the transactional outbox without running everything in production.*
