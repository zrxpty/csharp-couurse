[⬆ К модулю M15](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M15-L05: Тестирование DI, замена DbContext (in-memory, SQLite) / Testing DI, replacing DbContext (in-memory, SQLite)

**Модуль / Module:** M15
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Тестирование кода, который зависит от `DbContext`, — одна из самых частых задач в .NET-проектах с EF Core. Проблема проста: реальная база данных медленная, требует соединений, миграций и чистого состояния между тестами. Если каждый тест будет ходить в настоящий SQL Server, набор из сотни тестов станет работать минуты, а не секунды. Решение — заменить реальную зависимость (`DbContext`) на тестовый двойник через тот же механизм DI, что используется в продакшене.

EF Core предлагает два популярных провайдера для тестов. Первый — **InMemory provider** (`Microsoft.EntityFrameworkCore.InMemory`). Это «игрушечная» база, которая хранит данные в обычной памяти процесса. Она не поддерживает транзакции, внешние ключи и SQL-ограничения: по сути, это `Dictionary` под капотом. InMemory идеален, когда нужно проверить, что контроллер правильно вызывает репозиторий и собирает ответ, — то есть когда вы тестируете «форму вызова», а не целостность данных. Аналогия: InMemory — это черновик на салфетке; он быстрый, но не строгий.

Второй провайдер — **SQLite in-memory** (`Microsoft.Data.Sqlite` + `Microsoft.EntityFrameworkCore.Sqlite`). Он поднимает настоящую реляционную базу в оперативной памяти через строку соединения `DataSource=:memory:`. SQLite уважает транзакции, внешние ключи, уникальные индексы и типы столбцов. Это даёт реалистичное поведение: если ваш код нарушает ограничение, тест упадёт, как упал бы в SQL Server. Аналогия: SQLite in-memory — это стендовый макет двигателя: те же законы физики, только в миниатюре и без расхода топлива.

Когда провайдер выбран, его нужно «встроить» в приложение. В unit-тестах с ручным DI-контейнером вы регистрируете `DbContext` напрямую: `services.AddDbContext<AppDbContext>(o => o.UseSqlite(conn))`. В интеграционных тестах ASP.NET Core используют `WebApplicationFactory<TEntryPoint>` из пакета `Microsoft.AspNetCore.Mvc.Testing`. Этот класс поднимает всё приложение в памяти — Kestrel, middleware, DI — и позволяет через `ConfigureServices` подменить регистрацию `DbContext`, чтобы фабрика контроллеров получила тестовый контекст вместо продакшен-строки соединения.

Важное правило замены зависимостей: подменяйте только «границы» системы — базу данных, файловую систему, внешние HTTP-API, часы. Внутренние сервисы оставляйте реальными. Чем больше реального кода проходит через тест, тем выше доверие. Заменять `DbContext` оправданно всегда — это внешняя, медленная зависимость с состоянием; заменить внутренний доменный сервис часто означает, что вы тестируете не систему, а её макет.

Шаблон изоляции состояния между тестами: каждый тест создаёт новый контейнер DI и новую базу. Для SQLite это новый `SqliteConnection` + `EnsureCreated()` (или `Migrate`) перед тестом, и `Dispose` после. Для InMemory — новый `Guid`-ключ в `UseInMemoryDatabase`, чтобы именовать изолированное хранилище. Никогда не делитесь контекстом между тестами — параллельный запуск xUnit сломает данные и даст плавающие ошибки, которые невозможно воспроизвести.

Наконец, помните про **transactional test pattern**: оборачивайте тело теста в транзакцию и делайте откат в `Dispose`. Так база остаётся чистой даже при большом количестве тестов. Этот приём работает с SQLite и реальным SQL Server, но не с InMemory, потому что InMemory игнорирует транзакции. Выбор провайдера и паттерна изоляции — это компромисс между скоростью и реалистичностью: InMemory для быстрых smoke-тестов, SQLite для проверок ограничений и транзакций, Testcontainers с реальным SQL Server/PostgreSQL — для финальной уверенности перед релизом.

#### Theory (EN)

Testing code that depends on `DbContext` is one of the most common chores in a .NET project built on EF Core. The problem is simple: a real database is slow, needs connections, migrations, and clean state between tests. If every test hits a live SQL Server, a suite of a hundred tests runs in minutes, not seconds. The fix is to replace the real dependency (`DbContext`) with a test double through the same DI mechanism you already use in production.

EF Core offers two popular providers for tests. The first is the **InMemory provider** (`Microsoft.EntityFrameworkCore.InMemory`) — a “toy” database that keeps data in the process memory. It does not support transactions, foreign keys, or SQL constraints; under the hood it is little more than a `Dictionary`. InMemory is ideal when you want to verify that a controller calls the repository correctly and assembles the right response — that is, when you test the “shape of the call,” not data integrity. Analogy: InMemory is a sketch on a napkin; fast, but not strict.

The second provider is **SQLite in-memory** (`Microsoft.Data.Sqlite` + `Microsoft.EntityFrameworkCore.Sqlite`). It spins up a real relational database in RAM through the connection string `DataSource=:memory:`. SQLite honors transactions, foreign keys, unique indexes, and column types. This gives realistic behavior: if your code violates a constraint, the test fails the same way it would on SQL Server. Analogy: SQLite in-memory is a bench model of an engine — the same laws of physics apply, just in miniature and without burning fuel.

Once the provider is chosen, you must wire it into the application. In unit tests with a hand-built DI container you register `DbContext` directly: `services.AddDbContext<AppDbContext>(o => o.UseSqlite(conn))`. In ASP.NET Core integration tests you use `WebApplicationFactory<TEntryPoint>` from the `Microsoft.AspNetCore.Mvc.Testing` package. This class boots the whole app in memory — Kestrel, middleware, DI — and lets you override the `DbContext` registration through `ConfigureServices`, so the controller activator receives the test context instead of the production connection string.

A key rule of replacing dependencies: replace only the “edges” of the system — the database, the file system, external HTTP APIs, the clock. Keep internal services real. The more real code flows through the test, the higher the trust. Replacing `DbContext` is always justified — it is an external, slow, stateful dependency; replacing an internal domain service often means you are testing a mock of the system rather than the system itself.

The state-isolation pattern between tests: every test gets a fresh DI container and a fresh database. For SQLite that means a new `SqliteConnection` plus `EnsureCreated()` (or `Migrate`) before the test and `Dispose` after it. For InMemory that means a new `Guid` key in `UseInMemoryDatabase` to name an isolated store. Never share a context across tests — xUnit’s parallel runner will corrupt data and produce flaky failures that are impossible to reproduce.

Finally, remember the **transactional test pattern**: wrap the body of the test in a transaction and roll it back in `Dispose`. The database then stays clean even with thousands of tests. This works with SQLite and a real SQL Server, but not with InMemory, because InMemory ignores transactions. Choosing a provider and an isolation pattern is a trade-off between speed and realism: InMemory for fast smoke tests, SQLite for constraint and transaction checks, Testcontainers with a real SQL Server or PostgreSQL for final confidence before release.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Тестирование DI и замена DbContext
// Testing DI and replacing DbContext

using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.Data.Sqlite;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection.Extensions;
using System.Net;
using System.Net.Http.Json;
using Xunit;

// --- Продакшен-код / Production code ---------------------------------------

public sealed class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        // Реальные ограничения нужны для SQLite-тестов / Real constraints matter for SQLite tests
        b.Entity<Product>()
         .HasIndex(p => p.Sku)
         .IsUnique();

        b.Entity<Product>()
         .Property(p => p.Name)
         .IsRequired()
         .HasMaxLength(200);
    }
}

public sealed record Product(int Id, string Sku, string Name, decimal Price);

public sealed class ProductController(AppDbContext db)
{
    // Создаём продукт и сразу сохраняем / Create a product and persist immediately
    public async Task<Product> CreateAsync(string sku, string name, decimal price, CancellationToken ct)
    {
        var product = new Product(0, sku, name, price);
        db.Products.Add(product);
        await db.SaveChangesAsync(ct); // SQLite бросит исключение при дубликате SKU
        return product;
    }
}

// --- 1) InMemory-провайдер в ручном DI / InMemory provider with manual DI --

public sealed class InMemoryFactory : IDisposable
{
    public IServiceProvider Services { get; }

    public InMemoryFactory()
    {
        var services = new ServiceCollection();
        // Уникальное имя => изоляция между тестами / Unique name => isolation between tests
        services.AddDbContext<AppDbContext>(o =>
            o.UseInMemoryDatabase($"tests-{Guid.NewGuid()}"));
        Services = services.BuildServiceProvider();
    }

    public AppDbContext CreateContext() =>
        Services.GetRequiredService<AppDbContext>();

    public void Dispose() => (Services as IDisposable)?.Dispose();
}

// --- 2) SQLite in-memory + транзакционный тест / SQLite in-memory + transactional test

public sealed class SqliteFactory : IDisposable
{
    private readonly SqliteConnection _connection;

    public IServiceProvider Services { get; }

    public SqliteFactory()
    {
        // Один держатель соединения = одна живая БД в памяти / One holder = one live in-memory DB
        _connection = new SqliteConnection("DataSource=:memory:");
        _connection.Open();

        var services = new ServiceCollection();
        services.AddDbContext<AppDbContext>(o => o.UseSqlite(_connection));
        Services = services.BuildServiceProvider();

        // Создаём схему один раз / Build the schema once
        using (var scope = Services.CreateScope())
        using (var ctx = scope.ServiceProvider.GetRequiredService<AppDbContext>())
        {
            ctx.Database.EnsureCreated();
        }
    }

    public AppDbContext CreateContext() =>
        Services.GetRequiredService<AppDbContext>();

    public void Dispose()
    {
        _connection.Dispose();
        (Services as IDisposable)?.Dispose();
    }
}

// --- 3) WebApplicationFactory с подменой DbContext / Replace DbContext in WebApplicationFactory

public sealed class TestWebAppFactory : WebApplicationFactory<Program>
{
    private readonly SqliteConnection _connection;

    public TestWebAppFactory()
    {
        _connection = new SqliteConnection("DataSource=:memory:");
        _connection.Open();
    }

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");

        builder.ConfigureServices(services =>
        {
            // Убираем продакшен-регистрацию / Remove the production registration
            services.RemoveAll<DbContextOptions<AppDbContext>>();
            services.RemoveAll<AppDbContext>();

            services.AddDbContext<AppDbContext>(o => o.UseSqlite(_connection));

            // Создаём схему до первого запроса / Build the schema before the first request
            using var scope = services.BuildServiceProvider().CreateScope();
            using var ctx = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            ctx.Database.EnsureCreated();
        });
    }

    protected override void Dispose(bool disposing)
    {
        if (disposing) _connection.Dispose();
        base.Dispose(disposing);
    }
}

// --- 4) Тесты / Tests ------------------------------------------------------

public sealed class ProductTests : IClassFixture<TestWebAppFactory>
{
    private readonly TestWebAppFactory _factory;

    public ProductTests(TestWebAppFactory factory) => _factory = factory;

    [Fact]
    public async Task HttpClient_Creates_Product_Through_Real_Pipeline()
    {
        // Полный путь: DI -> controller -> AppDbContext -> SQLite / Full real path
        var client = _factory.CreateClient();

        var response = await client.PostAsJsonAsync(
            "/api/products",
            new { sku = "SKU-1", name = "Keyboard", price = 49.9m });

        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
    }

    [Fact]
    public void Sqlite_Rejects_Duplicate_Sku()
    {
        // SQLite уважает уникальный индекс / SQLite honors the unique index
        using var factory = new SqliteFactory();
        using var ctx = factory.CreateContext();

        ctx.Products.Add(new Product(0, "DUP", "A", 1));
        ctx.SaveChanges();

        ctx.Products.Add(new Product(0, "DUP", "B", 2));
        var ex = Assert.Throws<DbUpdateException>(() => ctx.SaveChanges());
        Assert.Contains("UNIQUE", ex.InnerException?.Message ?? ex.Message);
    }

    [Fact]
    public async Task InMemory_Fast_Smoke_Test()
    {
        // InMemory: быстро, но без ограничений / Fast, but no constraints enforced
        using var factory = new InMemoryFactory();
        var controller = new ProductController(factory.CreateContext());

        var created = await controller.CreateAsync("S1", "Mouse", 12.5m, CancellationToken.None);

        Assert.NotEqual(0, created.Id);
        Assert.Equal("S1", created.Sku);
    }
}
```

#### Best Practices

- Заменяйте только границы системы (база, файлы, HTTP, часы); внутренние сервисы оставляйте реальными — это повышает доверие к тестам.
- Используйте InMemory только для быстрых smoke-тестов формы вызовов; для проверки ограничений и транзакций берите SQLite in-memory.
- Каждый тест создаёт новый контейнер DI и новую БД (новый `Guid` для InMemory, новый `SqliteConnection` для SQLite) — это гарантирует изоляцию при параллельном запуске xUnit.
- В `WebApplicationFactory` сначала `RemoveAll<DbContextOptions<T>>()` и `RemoveAll<T>()`, затем регистрируйте тестовый контекст — иначе останется продакшен-регистрация.
- Создавайте схему через `EnsureCreated()` для скорости или `Migrate()` для проверки миграций — выбор зависит от цели теста.
- Применяйте transactional test pattern (откат в `Dispose`) с SQLite/SQL Server, чтобы база оставалась чистой между тестами.

- Replace only the system’s edges (DB, files, HTTP, clock); keep internal services real — this raises trust in the tests.
- Use InMemory only for fast smoke tests of call shapes; use SQLite in-memory to verify constraints and transactions.
- Each test builds a fresh DI container and a fresh DB (a new `Guid` for InMemory, a new `SqliteConnection` for SQLite) — this guarantees isolation under xUnit’s parallel runner.
- In `WebApplicationFactory` call `RemoveAll<DbContextOptions<T>>()` and `RemoveAll<T>()` first, then register the test context — otherwise the production registration lingers.
- Build the schema with `EnsureCreated()` for speed or `Migrate()` to exercise migrations — the choice depends on the test’s goal.
- Apply the transactional test pattern (rollback in `Dispose`) with SQLite/SQL Server so the database stays clean between tests.

#### Частые ошибки / Common Mistakes

- Использование одного общего `DbContext` между тестами → плавающие ошибки при параллельном запуске; создавайте новый контейнер и новую БД на каждый тест.
- Проверка уникальности/внешних ключей через InMemory → тест проходит, а продакшен падает; для ограничений берите SQLite in-memory.
- Забыли вызвать `EnsureCreated()`/`Migrate()` → запросы падают с «no such table»; создавайте схему после открытия соединения.
- Подмена `AddDbContext` в `WebApplicationFactory` без `RemoveAll` → в DI остаётся продакшен-регистрация и берётся реальная база.
- Держатель `SqliteConnection` закрыли раньше времени → in-memory база исчезает; держите соединение живым на время жизни фабрики тестов.
- Проверка SQL-поведения через InMemory → ложная уверенность; InMemory не выполняет SQL и игнорирует транзакции.

- Sharing one `DbContext` across tests → flaky failures under the parallel runner; build a fresh container and DB per test.
- Asserting uniqueness/foreign keys via InMemory → test passes, production fails; use SQLite in-memory for constraints.
- Forgetting `EnsureCreated()`/`Migrate()` → requests fail with “no such table”; build the schema right after opening the connection.
- Overriding `AddDbContext` in `WebApplicationFactory` without `RemoveAll` → the production registration stays in DI and a real database is used.
- Closing the `SqliteConnection` holder too early → the in-memory database vanishes; keep the connection alive for the lifetime of the test factory.
- Asserting SQL behavior through InMemory → false confidence; InMemory runs no SQL and ignores transactions.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я выбрал провайдер осознанно: InMemory для скорости, SQLite для ограничений и транзакций.
- [ ] Каждый тест получает свой контейнер DI и свою БД, без общего состояния.
- [ ] В `WebApplicationFactory` я удаляю старую регистрацию `DbContext` перед добавлением тестовой.
- [ ] Схема создаётся через `EnsureCreated()` или `Migrate()` до первого запроса в тесте.
- [ ] `SqliteConnection` держится открытым на всё время жизни фабрики/теста.
- [ ] Я подменяю только границы системы, а внутренние сервисы остаются реальными.
- [ ] Тесты не зависят от порядка выполнения и проходят параллельно.

- [ ] I chose the provider deliberately: InMemory for speed, SQLite for constraints and transactions.
- [ ] Each test gets its own DI container and its own DB, with no shared state.
- [ ] In `WebApplicationFactory` I remove the old `DbContext` registration before adding the test one.
- [ ] The schema is built via `EnsureCreated()` or `Migrate()` before the first request in the test.
- [ ] The `SqliteConnection` stays open for the whole lifetime of the factory/test.
- [ ] I replace only the system’s edges, while internal services remain real.
- [ ] Tests do not depend on execution order and pass in parallel.

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/ef/core/testing/]
- [EF Core InMemory provider — https://learn.microsoft.com/ef/core/providers/in-memory/]
- [SQLite in-memory testing — https://learn.microsoft.com/ef/core/testing/testing-without-the-database]
- [WebApplicationFactory — https://learn.microsoft.com/aspnet/core/test/integration-tests]

---

[⬆ К модулю M15](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
