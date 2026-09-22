---
[← К уроку M15-L05](lesson-M15-L05-testing-di-dbcontext.md) | [⬆ К модулю M15](../README.md) | [Следующее ДЗ →](homework-M15-L06-webapplicationfactory.md)
---

### Домашнее задание M15-L05: Тестирование DI, замена DbContext (in-memory, SQLite) / Homework M15-L05: Testing DI, replacing DbContext (in-memory, SQLite)

**Урок / Lesson:** M15-L05
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться осознанно выбирать и подменять провайдер `DbContext` в тестах — InMemory для быстрых smoke-тестов и SQLite in-memory для проверок ограничений и транзакций, — собирать изолированные DI-контейнеры на каждый тест, корректно встраивать тестовый контекст в `WebApplicationFactory` через `RemoveAll` + повторную регистрацию, и применять transactional test pattern для чистоты базы между тестами. (EN) Learn to deliberately choose and replace the `DbContext` provider in tests — InMemory for fast smoke tests and SQLite in-memory for constraint and transaction checks — build isolated DI containers per test, properly wire the test context into `WebApplicationFactory` through `RemoveAll` plus re-registration, and apply the transactional test pattern to keep the database clean between tests.

#### Связь с уроком / Connection to the lesson
(RU) Урок M15-L05 разбирает два провайдера EF Core для тестов (InMemory и SQLite in-memory), три способа их подключения (ручной DI, `SqliteFactory`, `WebApplicationFactory`) и паттерны изоляции состояния. ДЗ закрепляет все эти темы на практической задаче: вы построите мини-каталог товаров с уникальным SKU и напишете тесты трёх видов, чтобы увидеть разницу в поведении провайдеров на нарушениях ограничений. (EN) Lesson M15-L05 covers two EF Core test providers (InMemory and SQLite in-memory), three ways to wire them (manual DI, `SqliteFactory`, `WebApplicationFactory`), and state-isolation patterns. This homework cements all of them through a practical task: you will build a small product catalog with a unique SKU and write three kinds of tests to observe how the providers differ when constraints are violated.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы присоединились к команде, разрабатывающей внутренний сервис каталога товаров на ASP.NET Core 8 + EF Core 8. Сервис хранит продукты с полями `Id`, `Sku`, `Name`, `Price`; бизнес требует, чтобы `Sku` был уникальным, а `Name` — непустым и не длиннее 200 символов. Сейчас в репозитории есть пара ручных тестов, которые ходят в общий разделяемый `DbContext`, зарегистрированный через `AddDbContext` с реальной строкой соединения к SQL Server. Из-за этого набор тестов работает несколько минут, иногда падает без видимых причин (потому что параллельные тесты ломают данные друг друга), а проверка уникальности SKU вообще не срабатывает — потому что разработчики использовали InMemory и не заметили, что он не уважает ограничения.

Вам поручают навести порядок: спроектировать модель и контекст с реальными ограничениями, написать три набора тестов, каждый из которых демонстрирует своё поведение провайдера, и подготовить инфраструктуру для будущих интеграционных тестов через `WebApplicationFactory`. Цель — не «сделать зелёным» любой ценой, а показать команде разницу между быстрыми smoke-тестами на InMemory и строгими проверками на SQLite, а также зафиксировать best practices изоляции состояния, чтобы плавающие ошибки ушли навсегда. По ходу работы вы столкнётесь с тем, что InMemory молча проглатывает дубликаты SKU, что SQLite требует живого `SqliteConnection` и предварительного `EnsureCreated()`, а в `WebApplicationFactory` нельзя просто «перерегистрировать» контекст — нужно сначала убрать продакшен-регистрацию через `RemoveAll`. Каждый из этих уроков вы должны зафиксировать в тестах и комментариях.

#### Что нужно сделать (пошагово)
1. Создайте solution с тремя проектами. Выполните команды:
   ```bash
   dotnet new sln -n Catalog.Tests
   dotnet new web -n Catalog.Api -o src/Catalog.Api
   dotnet new xunit -n Catalog.Tests -o tests/Catalog.Tests
   dotnet sln add src/Catalog.Api tests/Catalog.Tests
   dotnet add src/Catalog.Api package Microsoft.EntityFrameworkCore.Sqlite --version 8.0.*
   dotnet add src/Catalog.Api package Microsoft.EntityFrameworkCore.InMemory --version 8.0.*
   dotnet add tests/Catalog.Tests reference src/Catalog.Api
   dotnet add tests/Catalog.Tests package Microsoft.AspNetCore.Mvc.Testing --version 8.0.*
   dotnet add tests/Catalog.Tests package Microsoft.Data.Sqlite --version 8.0.*
   dotnet add tests/Catalog.Tests package Microsoft.EntityFrameworkCore.Sqlite --version 8.0.*
   dotnet add tests/Catalog.Tests package Microsoft.EntityFrameworkCore.InMemory --version 8.0.*
   ```
   После каждой команды `dotnet add package` ожидайте строку вида `info : PackageReference ... added`; если видите `error`, проверьте версию .NET SDK (`dotnet --version` должно показать `8.x`).

2. В `src/Catalog.Api` создайте файлы `Product.cs`, `AppDbContext.cs`, `ProductsController.cs` и `Program.cs`. `Product` — `sealed record Product(int Id, string Sku, string Name, decimal Price);`. В `AppDbContext` настройте уникальный индекс по `Sku` и обязательное поле `Name` с `HasMaxLength(200)` — именно эти ограничения должны сработать в SQLite. Контроллер должен принимать `AppDbContext` через primary constructor и реализовать два метода: `CreateAsync` (POST, возвращает `Created`) и `GetAllAsync` (GET, возвращает список).

3. В `Program.cs` зарегистрируйте `AppDbContext` через `AddDbContext` с `UseSqlite` и строкой соединения из конфигурации, добавьте контроллеры, настройте минимальный routing (`MapControllers`). Сделайте `Program` классом `public partial class Program` — это нужно, чтобы `WebApplicationFactory<Program>` видел точку входа из тестового проекта.

4. В `tests/Catalog.Tests` создайте три файла инфраструктуры:
   - `InMemoryFactory.cs` — ручной `ServiceCollection`, регистрирующий `AppDbContext` через `UseInMemoryDatabase($"tests-{Guid.NewGuid()}")`. Реализует `IDisposable`, освобождает `ServiceProvider`.
   - `SqliteFactory.cs` — открывает `SqliteConnection("DataSource=:memory:")`, держит его в поле, регистрирует контекст через `UseSqlite(_connection)`, вызывает `EnsureCreated()` в конструкторе. В `Dispose` закрывает соединение последним.
   - `TestWebAppFactory.cs` — наследник `WebApplicationFactory<Program>`, в `ConfigureWebHost` устанавливает среду `"Testing"`, вызывает `RemoveAll<DbContextOptions<AppDbContext>>()` и `RemoveAll<AppDbContext>()`, затем регистрирует контекст с тестовым `SqliteConnection`, и один раз строит схему через временный scope.

5. Напишите тесты, которые осознанно демонстрируют разницу провайдеров:
   - `InMemory_Allows_Duplicate_Sku_But_Only_For_Smoke` — на InMemory создаёт два продукта с одинаковым SKU и проверяет, что оба сохраняются. В комментарии явно укажите: «InMemory не уважает уникальный индекс; этот тест — smoke, не проверка ограничения».
   - `Sqlite_Rejects_Duplicate_Sku` — на SQLite вставляет продукт с SKU `"DUP"`, затем пытается вставить второй с тем же SKU и asserts, что бросается `DbUpdateException`, а `InnerException.Message` содержит `UNIQUE`.
   - `Sqlite_Requires_Name_NotEmpty` — на SQLite пытается сохранить продукт с пустым `Name` и ожидает `DbUpdateException`.
   - `HttpClient_Creates_Product_Through_Real_Pipeline` — через `TestWebAppFactory.CreateClient()` отправляет POST на `/api/products`, ожидает `HttpStatusCode.Created` и затем GET подтверждает наличие продукта.
   - `Transactional_Test_Rolls_Back_Changes` — на SQLite открывает транзакцию, вставляет продукт, в `Dispose` делает откат, и отдельный тест verifies, что база осталась пустой (используйте `IClassFixture` или `IAsyncLifetime` для управления транзакцией).

6. Запустите тесты: `dotnet test tests/Catalog.Tests`. Ожидайте 5 пройденных тестов. Если SQLite-тесты падают с `SQLite Error 1: 'no such table'`, значит вы забыли `EnsureCreated()` или закрыли соединение раньше времени. Если интеграционный тест пытается достучаться до реальной базы — значит не сработал `RemoveAll`.

7. Добавьте в репозиторий файл `tests/Catalog.Tests/README.md` с таблицей сравнения провайдеров (3 колонки: InMemory, SQLite in-memory, реальный SQL Server) по характеристикам: «уважает ограничения», «уважает транзакции», «скорость», «изоляция между тестами», «миграции».

#### Требования к решению
Решение должно компилироваться без предупреждений на .NET 8 / C# 12, использовать top-level statements в `Program.cs`, primary constructors для контроллеров и фабрик, collection expressions и pattern matching там, где это уместно. Все тесты должны проходить параллельно (xUnit по умолчанию запускает классы параллельно), без общего состояния — каждый тест создаёт собственный контейнер DI и собственную базу. Запрещено использовать реальный SQL Server или любую внешнюю зависимость; весь тестовый запуск должен работать на чистой машине без соединений с сетью.

Код должен быть организован в три слоя: production-код (`Catalog.Api`), тестовая инфраструктура (фабрики) и сами тесты. Имена классов и методов должны быть говорящими и следовать стилю `Method_Scenario_ExpectedResult`. В комментариях RU+EN явно отмечайте, какой концепции урока соответствует каждая строка (InMemory как smoke, SQLite как строгий, transactional pattern, `RemoveAll` в `WebApplicationFactory`). Все `IDisposable`/`IAsyncDisposable` ресурсы (`SqliteConnection`, `ServiceProvider`, `HttpClient`) должны корректно освобождаться; утечек быть не должно. Решение должно продемонстрировать, что вы понимаете, ПОЧЕМУ InMemory пропускает дубликаты, а SQLite — нет, и уметь объяснить это в комментарии.

#### Тонкости и подводные камни
- **InMemory не выполняет SQL и игнорирует транзакции.** Если ваш тест проверяет уникальность через InMemory, он пройдёт даже при дубликате — это ложная уверенность. Ограничения (`HasIndex().IsUnique()`, `IsRequired()`, `HasMaxLength`) в InMemory фактически не применяются; они работают только на реляционных провайдерах. Поэтому разделите smoke-тесты (InMemory) и проверки ограничений (SQLite).
- **SQLite in-memory исчезает при закрытии соединения.** Строка `DataSource=:memory:` создаёт базу, привязанную к соединению. Как только вы закрываете `SqliteConnection`, база удаляется без следа. Поэтому держатель соединения должен жить на протяжении всей фабрики тестов — обычно это поле класса-фабрики, освобождаемое в `Dispose`. Если вы открываете и закрываете соединение на каждый `CreateContext`, схема потеряется и тест упадёт с `no such table`.
- **`EnsureCreated()` vs `Migrate()`.** `EnsureCreated()` создаёт схему по текущей модели — быстро, но не запускает миграции. `Migrate()` применяет реальную историю миграций, что полезно, если вы хотите протестировать сами миграции. В ДЗ используйте `EnsureCreated()` для скорости, но в комментарии упомяните, когда нужен `Migrate()`.
- **`WebApplicationFactory` без `RemoveAll` — типичная ловушка.** Продакшен-регистрация `AddDbContext` уже лежит в DI-контейнере. Если вы просто вызовете `AddDbContext` ещё раз с тестовым провайдером, обе регистрации останутся, и depending-on-resolution порядок может вернуть продакшен-контекст. Правильный порядок: `services.RemoveAll<DbContextOptions<AppDbContext>>(); services.RemoveAll<AppDbContext>();` и только потом новая регистрация. Не забудьте также убрать `DbContextOptions<AppDbContext>` — иначе в DI останется старая `Options`, и новая регистрация подхватит её.
- **Изоляция между тестами.** xUnit запускает разные классы тестов параллельно. Если два класса используют общий `static` контекст или общий `DbContext`, данные будут перекрываться. Используйте `IClassFixture<T>` для общих тяжёлых ресурсов (например, `WebApplicationFactory`) и новый `Guid`/новый `SqliteConnection` на каждый тест для лёгких.
- **Transactional test pattern не работает с InMemory.** Оборачивать тело теста в транзакцию и откатывать в `Dispose` можно только на SQLite/SQL Server. В InMemory `BeginTransaction` — это no-op, откат ничего не делает.
- **Primary constructor и `Dispose`.** Если фабрика использует primary constructor и хранит `_connection` как поле, убедитесь, что `Dispose` освобождает ресурсы в правильном порядке: сначала `ServiceProvider` (чтобы активные scope закрылись), потом `SqliteConnection`. Иначе можно получить `ObjectDisposedException` в параллельном тесте.

#### Критерии приёмки
- [ ] Solution содержит проекты `Catalog.Api` и `Catalog.Tests`, ссылка между ними настроена.
- [ ] `AppDbContext` настраивает уникальный индекс по `Sku` и обязательный `Name` с `HasMaxLength(200)`.
- [ ] `Program.cs` использует top-level statements и `public partial class Program` для `WebApplicationFactory<Program>`.
- [ ] `ProductsController` принимает `AppDbContext` через primary constructor.
- [ ] `InMemoryFactory` использует `UseInMemoryDatabase($"tests-{Guid.NewGuid()}")` и реализует `IDisposable`.
- [ ] `SqliteFactory` открывает `SqliteConnection("DataSource=:memory:")`, держит его в поле, вызывает `EnsureCreated()` в конструкторе.
- [ ] `TestWebAppFactory` вызывает `RemoveAll<DbContextOptions<AppDbContext>>()` и `RemoveAll<AppDbContext>()` перед новой регистрацией.
- [ ] `TestWebAppFactory` устанавливает среду `"Testing"` и строит схему один раз.
- [ ] Тест `InMemory_Allows_Duplicate_Sku_But_Only_For_Smoke` проходит и содержит комментарий о том, что InMemory не проверяет ограничения.
- [ ] Тест `Sqlite_Rejects_Duplicate_Sku` ожидает `DbUpdateException` с `UNIQUE` в сообщении.
- [ ] Тест `Sqlite_Requires_Name_NotEmpty` падает при пустом `Name` на SQLite.
- [ ] Интеграционный тест через `HttpClient` возвращает `HttpStatusCode.Created`.
- [ ] Есть тест, демонстрирующий transactional test pattern с откатом.
- [ ] Все тесты проходят параллельно без плавающих ошибок.
- [ ] В `tests/Catalog.Tests/README.md` есть таблица сравнения провайдеров.
- [ ] Команда `dotnet test` показывает 5+ пройденных тестов и ноль проваленных.

#### Подсказки (без прямого ответа)
- Вспомните, что `EnsureCreated()` нужно вызвать на открытой БД до первого `SaveChanges`, иначе EF не найдёт таблицы. Где именно вызывать — на конструкторе фабрики или в `ConfigureWebHost` — подскажет время жизни соединения.
- `RemoveAll<T>()` удаляет все регистрации сервиса `T` из `IServiceCollection`. Подумайте, какие ещё типы, кроме `AppDbContext`, нужно убрать, чтобы новая регистрация точно подхватилась (`DbContextOptions<AppDbContext>` — это намёк).
- В `WebApplicationFactory` метод `ConfigureWebHost` вызывается после того, как `Program.cs` уже отработал и зарегистрировал продакшен-контекст. Поэтому `RemoveAll` нужен всегда — даже если вам кажется, что вы «перекрываете» регистрацию.
- Для transactional test pattern в xUnit удобен `IAsyncLifetime`: `InitializeAsync` открывает транзакцию, `DisposeAsync` откатывает. Подумайте, какой `IDbContextTransaction` вы будете хранить между ними.
- `Product` — `sealed record`. Если вы попытаетесь менять `Id` после `Add`, это не сработает (record immutable). Используйте `with` или создавайте новый экземпляр, либо положитесь на EF, который заполнит `Id` после `SaveChanges`.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — Эталонное решение ДЗ M15-L05
// Reference solution for homework M15-L05
// Двуязычные комментарии: RU — концепция урока, EN — эквивалент / Bilingual comments

using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.Data.Sqlite;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection.Extensions;
using System.Net;
using System.Net.Http.Json;
using Xunit;

// --- Production code / Продакшен-код ---

namespace Catalog.Api;

// sealed record + primary constructor: современный C# 12
// sealed record with primary constructor: modern C# 12
public sealed record Product(int Id, string Sku, string Name, decimal Price);

public sealed class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        // Уникальный индекс по Sku — сработает только на реляционном провайдере (SQLite)
        // Unique index on Sku — only honored by a relational provider (SQLite)
        b.Entity<Product>().HasIndex(p => p.Sku).IsUnique();

        // Name обязателен и не длиннее 200 — снова реляционное ограничение
        // Name required, max 200 — again a relational constraint
        b.Entity<Product>().Property(p => p.Name).IsRequired().HasMaxLength(200);
    }
}

[ApiController]
[Route("api/products")]
public sealed class ProductsController(AppDbContext db) : ControllerBase
{
    // Primary constructor инъектирует контекст через DI / Primary ctor injects context via DI
    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateProductRequest req, CancellationToken ct)
    {
        var product = new Product(0, req.Sku, req.Name, req.Price);
        db.Products.Add(product);
        await db.SaveChangesAsync(ct); // SQLite бросит DbUpdateException при дубликате / throws on dup
        return Created($"/api/products/{product.Id}", product);
    }

    [HttpGet]
    public async Task<IActionResult> GetAll(CancellationToken ct)
    {
        // AsNoTracking для чтения — best practice из урока / AsNoTracking for reads
        var list = await db.Products.AsNoTracking().ToListAsync(ct);
        return Ok(list);
    }
}

public sealed record CreateProductRequest(string Sku, string Name, decimal Price);

// Program.cs (top-level) — должен быть public partial для WebApplicationFactory<Program>
public partial class Program
{
    public static void Main(string[] args)
    {
        var builder = WebApplication.CreateBuilder(args);
        builder.Services.AddControllers();
        // Продакшен-регистрация: реальный SQLite-файл / Production: real SQLite file
        builder.Services.AddDbContext<AppDbContext>(o =>
            o.UseSqlite(builder.Configuration.GetConnectionString("Default")));
        var app = builder.Build();
        app.MapControllers();
        app.Run();
    }
}

// --- Test infrastructure / Тестовая инфраструктура ---

namespace Catalog.Tests;

// 1) InMemory factory — быстрые smoke-тесты / fast smoke tests
public sealed class InMemoryFactory : IDisposable
{
    public IServiceProvider Services { get; }
    public InMemoryFactory()
    {
        var services = new ServiceCollection();
        // Уникальный Guid => изоляция между тестами / Unique Guid => isolation
        services.AddDbContext<AppDbContext>(o =>
            o.UseInMemoryDatabase($"tests-{Guid.NewGuid()}"));
        Services = services.BuildServiceProvider();
    }
    public AppDbContext CreateContext() => Services.GetRequiredService<AppDbContext>();
    public void Dispose() => (Services as IDisposable)?.Dispose();
}

// 2) SQLite in-memory factory — строгие проверки / strict checks
public sealed class SqliteFactory : IDisposable
{
    private readonly SqliteConnection _connection; // держатель базы / DB holder
    public IServiceProvider Services { get; }

    public SqliteFactory()
    {
        _connection = new SqliteConnection("DataSource=:memory:");
        _connection.Open(); // база живёт, пока соединение открыто / DB lives while open
        var services = new ServiceCollection();
        services.AddDbContext<AppDbContext>(o => o.UseSqlite(_connection));
        Services = services.BuildServiceProvider();
        // Схема один раз на фабрику / Schema once per factory
        using var scope = Services.CreateScope();
        using var ctx = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        ctx.Database.EnsureCreated();
    }

    public AppDbContext CreateContext() => Services.GetRequiredService<AppDbContext>();

    public void Dispose()
    {
        (Services as IDisposable)?.Dispose(); // сначала контейнеры / container first
        _connection.Dispose();                // потом соединение / connection last
    }
}

// 3) WebApplicationFactory с подменой / replaced DbContext
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
            // Сначала убираем продакшен-регистрации / Remove production registrations first
            services.RemoveAll<DbContextOptions<AppDbContext>>();
            services.RemoveAll<AppDbContext>();
            // Затем регистрируем тестовый контекст / Then register test context
            services.AddDbContext<AppDbContext>(o => o.UseSqlite(_connection));
            // Схема до первого запроса / Schema before first request
            using var sp = services.BuildServiceProvider();
            using var scope = sp.CreateScope();
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

// --- Tests / Тесты ---

public sealed class ProviderComparisonTests
{
    [Fact]
    public void InMemory_Allows_Duplicate_Sku_But_Only_For_Smoke()
    {
        // InMemory НЕ уважает уникальный индекс — это smoke-тест формы вызова
        // InMemory does NOT honor unique index — this is a call-shape smoke test
        using var factory = new InMemoryFactory();
        using var ctx = factory.CreateContext();

        ctx.Products.Add(new Product(0, "DUP", "A", 1));
        ctx.SaveChanges();
        ctx.Products.Add(new Product(0, "DUP", "B", 2));
        ctx.SaveChanges(); // InMemory молча сохраняет — false confidence / silently saves

        Assert.Equal(2, ctx.Products.Count());
    }

    [Fact]
    public void Sqlite_Rejects_Duplicate_Sku()
    {
        using var factory = new SqliteFactory();
        using var ctx = factory.CreateContext();

        ctx.Products.Add(new Product(0, "DUP", "A", 1));
        ctx.SaveChanges();

        ctx.Products.Add(new Product(0, "DUP", "B", 2));
        var ex = Assert.Throws<DbUpdateException>(() => ctx.SaveChanges());
        Assert.Contains("UNIQUE", ex.InnerException?.Message ?? ex.Message);
    }

    [Fact]
    public void Sqlite_Requires_Name_NotEmpty()
    {
        using var factory = new SqliteFactory();
        using var ctx = factory.CreateContext();

        ctx.Products.Add(new Product(0, "S1", "", 1));
        Assert.Throws<DbUpdateException>(() => ctx.SaveChanges());
    }
}

public sealed class WebPipelineTests : IClassFixture<TestWebAppFactory>
{
    private readonly TestWebAppFactory _factory;
    public WebPipelineTests(TestWebAppFactory factory) => _factory = factory;

    [Fact]
    public async Task HttpClient_Creates_Product_Through_Real_Pipeline()
    {
        var client = _factory.CreateClient();
        var resp = await client.PostAsJsonAsync("/api/products",
            new { sku = "SKU-1", name = "Keyboard", price = 49.9m });
        Assert.Equal(HttpStatusCode.Created, resp.StatusCode);

        // GET подтверждает, что продукт действительно сохранился / GET confirms persistence
        var all = await client.GetFromJsonAsync<List<ProductDto>>("/api/products");
        Assert.Contains(all!, p => p.Sku == "SKU-1");
    }
}

public sealed record ProductDto(int Id, string Sku, string Name, decimal Price);

// 5) Transactional test pattern / Транзакционный паттерн
public sealed class TransactionalTests : IAsyncLifetime
{
    private SqliteFactory? _factory;
    private IDbContextTransaction? _tx;
    private AppDbContext? _ctx;

    public Task InitializeAsync()
    {
        _factory = new SqliteFactory();
        _ctx = _factory.CreateContext();
        _tx = _ctx.Database.BeginTransaction(); // откатим в Dispose / will roll back
        return Task.CompletedTask;
    }

    [Fact]
    public async Task Insert_Rolls_Back_After_Dispose()
    {
        _ctx!.Products.Add(new Product(0, "T1", "Tmp", 1));
        await _ctx.SaveChangesAsync(); // внутри транзакции / inside tx
        Assert.Equal(1, await _ctx.Products.CountAsync()); // виден в этой сессии
    }

    public Task DisposeAsync()
    {
        _tx?.Rollback(); // откат — база пуста для следующего теста / rollback leaves DB empty
        _ctx?.Dispose();
        _factory?.Dispose();
        return Task.CompletedTask;
    }
}
```

**Разбор по строкам.** `Product` объявлен как `sealed record` с позиционными параметрами — это современный C# 12, дающий иммутабельность и value equality, что удобно для DTO и тестовых assertions. `AppDbContext` использует синтаксис primary constructor `(DbContextOptions<AppDbContext> options) : DbContext(options)` — это эквивалент классического поля, но короче; концепция из урока «DI и замена DbContext». В `OnModelCreating` мы настраиваем `HasIndex(p => p.Sku).IsUnique()` и `IsRequired().HasMaxLength(200)` — эти ограничения почти дословно повторяют пример из урока, и именно они позволяют SQLite-тестам поймать нарушение, которое InMemory пропустит. Это ключевой учебный момент: модель описывает ограничения, но их enforcement зависит от провайдера.

`ProductsController` через primary constructor получает `AppDbContext` из DI — это «граница системы», которую мы будем подменять в тестах, тогда как внутренняя логика контроллера остаётся реальной (best practice из урока: «заменяйте только границы»). `Program` помечен `public partial class Program`, чтобы `WebApplicationFactory<Program>` из тестового проекта смог увидеть тип точки входа — это типичная ловушка, без `partial` компилятор тестов не увидит класс.

`InMemoryFactory` регистрирует контекст с `UseInMemoryDatabase($"tests-{Guid.NewGuid()}")` — уникальное имя на каждую фабрику даёт изоляцию между тестами при параллельном запуске xUnit (концепция «изолированное хранилище на тест» из урока). `SqliteFactory` открывает `SqliteConnection("DataSource=:memory:")` и держит его в приватном поле — потому что in-memory база SQLite живёт только пока открыто соединение (частая ошибка из урока: закрыли соединение → база исчезла). В конструкторе же вызывается `EnsureCreated()` через временный scope — это создаёт схему по модели один раз. В `Dispose` мы сначала освобождаем `ServiceProvider` (чтобы закрыть активные scope и контексты), и только потом `SqliteConnection` — обратный порядок может вызвать `ObjectDisposedException` в параллельном тесте.

`TestWebAppFactory` — самая тонкая часть. В `ConfigureWebHost` мы устанавливаем среду `"Testing"`, затем вызываем `RemoveAll<DbContextOptions<AppDbContext>>()` и `RemoveAll<AppDbContext>()`. Это критично: `Program.Main` уже зарегистрировал продакшен-контекст, и простое добавление новой регистрации оставит обе в контейнере — типичная ошибка из урока. Удаляем именно `DbContextOptions<T>` тоже, потому что `AddDbContext` регистрирует не только сам контекст, но и его `Options`-объект. После очистки регистрируем тестовый контекст с тем же `_connection` и один раз строим схему через временный `BuildServiceProvider()` — это безопасно, потому что мы не оставляем этот временный провайдер в DI, а только используем его для `EnsureCreated`.

В тестах `InMemory_Allows_Duplicate_Sku_But_Only_For_Smoke` сознательно демонстрирует ложную уверенность: InMemory сохраняет оба продукта с одинаковым SKU, и в комментарии это явно отмечено. `Sqlite_Rejects_Duplicate_Sku` ожидает `DbUpdateException` и проверяет, что `InnerException.Message` содержит `UNIQUE` — это доказывает, что сработал именно реляционный индекс, а не какая-то случайная ошибка. `Sqlite_Requires_Name_NotEmpty` проверяет обязательное поле. Интеграционный тест через `HttpClient` проходит полный путь: DI → controller → `AppDbContext` → SQLite, что соответствует best practice «чем больше реального кода, тем выше доверие». `TransactionalTests` реализует `IAsyncLifetime`: в `InitializeAsync` открывает транзакцию, в теле теста вставляет запись, в `DisposeAsync` откатывает — так база остаётся чистой для следующего теста (transactional test pattern из урока, который не работает с InMemory, но работает с SQLite).

#### Задания на углубление (бонус)
1. Перепишите тесты на `Migrate()` вместо `EnsureCreated()` и добавьте первую EF-миграцию. Убедитесь, что миграция действительно создаёт индекс, запустив `dotnet ef migrations list` и проверив сгенерированный SQL.
2. Добавьте тест через `Testcontainers.MsSql` (или PostgreSQL), который поднимает реальный контейнер Docker и проверяет те же ограничения. Сравните скорость запуска с SQLite in-memory — выпишите числа в `README.md`.
3. Реализуйте паттерн «shared database fixture» через `ICollectionFixture<T>` и сравните его с per-test isolation по скорости и надёжности. Объясните в комментарии, когда shared fixture оправдан.
4. Добавьте тест, который проверяет поведение InMemory при явном вызове `BeginTransaction` — покажите, что откат не откатывает изменения, и зафиксируйте это в `README.md` как антипаттерн.

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you have joined a team building an internal product catalog service on ASP.NET Core 8 with EF Core 8. The service stores products with the fields `Id`, `Sku`, `Name`, and `Price`; the business requires `Sku` to be unique and `Name` to be non-empty and at most 200 characters long. Right now the repository contains a handful of hand-written tests that share a single `DbContext` registered through `AddDbContext` against a real SQL Server connection string. Because of that, the test suite takes several minutes to run, occasionally fails for no apparent reason (because parallel tests corrupt each other’s data), and the SKU uniqueness assertion never fires — because the developers used InMemory and did not realize it does not honor constraints.

You are tasked with putting things in order: design the model and context with real constraints, write three families of tests that each demonstrate a distinct provider behavior, and prepare the infrastructure for future integration tests through `WebApplicationFactory`. The goal is not to “go green at any cost” but to show the team the difference between fast smoke tests on InMemory and strict checks on SQLite, and to lock down the state-isolation best practices so that flaky failures disappear forever. Along the way you will discover that InMemory silently accepts duplicate SKUs, that SQLite requires a live `SqliteConnection` and a prior `EnsureCreated()`, and that inside `WebApplicationFactory` you cannot just “re-register” the context — you must first remove the production registration through `RemoveAll`. Each of those lessons must be captured in tests and comments.

#### What to do step by step
1. Create a solution with three projects. Run the commands:
   ```bash
   dotnet new sln -n Catalog.Tests
   dotnet new web -n Catalog.Api -o src/Catalog.Api
   dotnet new xunit -n Catalog.Tests -o tests/Catalog.Tests
   dotnet sln add src/Catalog.Api tests/Catalog.Tests
   dotnet add src/Catalog.Api package Microsoft.EntityFrameworkCore.Sqlite --version 8.0.*
   dotnet add src/Catalog.Api package Microsoft.EntityFrameworkCore.InMemory --version 8.0.*
   dotnet add tests/Catalog.Tests reference src/Catalog.Api
   dotnet add tests/Catalog.Tests package Microsoft.AspNetCore.Mvc.Testing --version 8.0.*
   dotnet add tests/Catalog.Tests package Microsoft.Data.Sqlite --version 8.0.*
   dotnet add tests/Catalog.Tests package Microsoft.EntityFrameworkCore.Sqlite --version 8.0.*
   dotnet add tests/Catalog.Tests package Microsoft.EntityFrameworkCore.InMemory --version 8.0.*
   ```
   After each `dotnet add package` expect a line like `info : PackageReference ... added`; if you see `error`, check the SDK version (`dotnet --version` should print `8.x`).

2. In `src/Catalog.Api` create `Product.cs`, `AppDbContext.cs`, `ProductsController.cs`, and `Program.cs`. `Product` is `sealed record Product(int Id, string Sku, string Name, decimal Price);`. In `AppDbContext` configure a unique index on `Sku` and a required `Name` with `HasMaxLength(200)` — these are the constraints that SQLite must enforce. The controller takes `AppDbContext` through a primary constructor and exposes two actions: `CreateAsync` (POST, returns `Created`) and `GetAllAsync` (GET, returns the list).

3. In `Program.cs` register `AppDbContext` via `AddDbContext` with `UseSqlite` and a connection string from configuration, add controllers, and set up minimal routing (`MapControllers`). Make `Program` a `public partial class Program` so that `WebApplicationFactory<Program>` can see the entry point from the test project.

4. In `tests/Catalog.Tests` create three infrastructure files:
   - `InMemoryFactory.cs` — a hand-built `ServiceCollection` registering `AppDbContext` via `UseInMemoryDatabase($"tests-{Guid.NewGuid()}")`. Implements `IDisposable`, disposes the `ServiceProvider`.
   - `SqliteFactory.cs` — opens `SqliteConnection("DataSource=:memory:")`, keeps it in a field, registers the context through `UseSqlite(_connection)`, and calls `EnsureCreated()` in the constructor. In `Dispose` closes the connection last.
   - `TestWebAppFactory.cs` — a subclass of `WebApplicationFactory<Program>`; in `ConfigureWebHost` set the environment to `"Testing"`, call `RemoveAll<DbContextOptions<AppDbContext>>()` and `RemoveAll<AppDbContext>()`, then register the context with the test `SqliteConnection`, and build the schema once through a temporary scope.

5. Write tests that deliberately demonstrate the provider difference:
   - `InMemory_Allows_Duplicate_Sku_But_Only_For_Smoke` — on InMemory create two products with the same SKU and assert both are saved. In a comment state explicitly: “InMemory does not honor the unique index; this is a smoke test, not a constraint check.”
   - `Sqlite_Rejects_Duplicate_Sku` — on SQLite insert a product with SKU `"DUP"`, then attempt a second one with the same SKU and assert that a `DbUpdateException` is thrown and `InnerException.Message` contains `UNIQUE`.
   - `Sqlite_Requires_Name_NotEmpty` — on SQLite try to save a product with an empty `Name` and expect `DbUpdateException`.
   - `HttpClient_Creates_Product_Through_Real_Pipeline` — through `TestWebAppFactory.CreateClient()` send a POST to `/api/products`, expect `HttpStatusCode.Created`, then issue a GET to confirm the product is present.
   - `Transactional_Test_Rolls_Back_Changes` — on SQLite open a transaction, insert a product, roll back in `Dispose`, and use a separate assertion to verify the database stays empty (use `IClassFixture` or `IAsyncLifetime` to manage the transaction).

6. Run the tests: `dotnet test tests/Catalog.Tests`. Expect 5 passing tests. If the SQLite tests fail with `SQLite Error 1: 'no such table'`, you forgot `EnsureCreated()` or closed the connection too early. If the integration test tries to reach a real database, `RemoveAll` did not take effect.

7. Add `tests/Catalog.Tests/README.md` with a comparison table of providers (three columns: InMemory, SQLite in-memory, real SQL Server) across the characteristics: “honors constraints”, “honors transactions”, “speed”, “isolation between tests”, “migrations”.

#### Requirements
The solution must compile without warnings on .NET 8 / C# 12, use top-level statements in `Program.cs`, primary constructors for controllers and factories, and collection expressions and pattern matching where appropriate. All tests must pass in parallel (xUnit runs classes in parallel by default) with no shared state — each test builds its own DI container and its own database. Using a real SQL Server or any external dependency is forbidden; the whole test run must work on a clean machine with no network connections.

The code must be organized in three layers: production code (`Catalog.Api`), test infrastructure (the factories), and the tests themselves. Class and method names must be self-describing and follow the `Method_Scenario_ExpectedResult` style. In RU+EN comments explicitly mark which lesson concept each line corresponds to (InMemory as smoke, SQLite as strict, transactional pattern, `RemoveAll` in `WebApplicationFactory`). Every `IDisposable`/`IAsyncDisposable` resource (`SqliteConnection`, `ServiceProvider`, `HttpClient`) must be released correctly; no leaks are allowed. The solution must demonstrate that you understand WHY InMemory accepts duplicates and SQLite does not, and can explain it in a comment.

#### Pitfalls
- **InMemory runs no SQL and ignores transactions.** If your test asserts uniqueness through InMemory, it passes even on duplicates — that is false confidence. Constraints (`HasIndex().IsUnique()`, `IsRequired()`, `HasMaxLength`) are effectively not enforced by InMemory; they only work on relational providers. So separate smoke tests (InMemory) from constraint checks (SQLite).
- **SQLite in-memory vanishes when the connection closes.** The string `DataSource=:memory:` creates a database bound to the connection. As soon as you close the `SqliteConnection`, the database is gone without a trace. Therefore the connection holder must live for the entire test factory lifetime — usually a field on the factory class, disposed in `Dispose`. If you open and close the connection on every `CreateContext`, the schema is lost and the test fails with `no such table`.
- **`EnsureCreated()` vs `Migrate()`.** `EnsureCreated()` builds the schema from the current model — fast, but it does not run migrations. `Migrate()` applies the real migration history, which is useful when you want to test the migrations themselves. In this homework use `EnsureCreated()` for speed, but mention in a comment when `Migrate()` is needed.
- **`WebApplicationFactory` without `RemoveAll` — a classic trap.** The production `AddDbContext` registration is already in the DI container. If you simply call `AddDbContext` again with the test provider, both registrations remain, and depending on resolution order the production context may be returned. The correct order is `services.RemoveAll<DbContextOptions<AppDbContext>>(); services.RemoveAll<AppDbContext>();` and only then the new registration. Do not forget to remove `DbContextOptions<AppDbContext>` too — otherwise the old `Options` object lingers in DI and the new registration picks it up.
- **Isolation between tests.** xUnit runs different test classes in parallel. If two classes share a `static` context or a shared `DbContext`, the data overlaps. Use `IClassFixture<T>` for heavy shared resources (such as `WebApplicationFactory`) and a fresh `Guid` or a fresh `SqliteConnection` per test for light ones.
- **Transactional test pattern does not work with InMemory.** Wrapping the test body in a transaction and rolling back in `Dispose` works only on SQLite/SQL Server. In InMemory `BeginTransaction` is a no-op, and rollback does nothing.
- **Primary constructor and `Dispose`.** If the factory uses a primary constructor and stores `_connection` as a field, make sure `Dispose` releases resources in the right order: first the `ServiceProvider` (so active scopes close), then the `SqliteConnection`. Otherwise you can get an `ObjectDisposedException` in a parallel test.

#### Acceptance criteria
- [ ] The solution contains the `Catalog.Api` and `Catalog.Tests` projects, with the project reference configured.
- [ ] `AppDbContext` configures a unique index on `Sku` and a required `Name` with `HasMaxLength(200)`.
- [ ] `Program.cs` uses top-level statements and a `public partial class Program` for `WebApplicationFactory<Program>`.
- [ ] `ProductsController` takes `AppDbContext` through a primary constructor.
- [ ] `InMemoryFactory` uses `UseInMemoryDatabase($"tests-{Guid.NewGuid()}")` and implements `IDisposable`.
- [ ] `SqliteFactory` opens `SqliteConnection("DataSource=:memory:")`, keeps it in a field, and calls `EnsureCreated()` in the constructor.
- [ ] `TestWebAppFactory` calls `RemoveAll<DbContextOptions<AppDbContext>>()` and `RemoveAll<AppDbContext>()` before the new registration.
- [ ] `TestWebAppFactory` sets the environment to `"Testing"` and builds the schema once.
- [ ] The `InMemory_Allows_Duplicate_Sku_But_Only_For_Smoke` test passes and contains a comment noting that InMemory does not enforce constraints.
- [ ] The `Sqlite_Rejects_Duplicate_Sku` test expects a `DbUpdateException` with `UNIQUE` in the message.
- [ ] The `Sqlite_Requires_Name_NotEmpty` test fails on an empty `Name` under SQLite.
- [ ] The integration test through `HttpClient` returns `HttpStatusCode.Created`.
- [ ] A test demonstrates the transactional test pattern with a rollback.
- [ ] All tests pass in parallel without flaky failures.
- [ ] `tests/Catalog.Tests/README.md` contains the provider comparison table.
- [ ] `dotnet test` reports 5+ passed tests and zero failures.

#### Hints
- Recall that `EnsureCreated()` must be called against an open database before the first `SaveChanges`, otherwise EF finds no tables. Where exactly to call it — in the factory constructor or in `ConfigureWebHost` — is decided by the connection lifetime.
- `RemoveAll<T>()` removes every registration of the service `T` from `IServiceCollection`. Think about which other types besides `AppDbContext` need to be removed so the new registration actually takes over (`DbContextOptions<AppDbContext>` is a hint).
- In `WebApplicationFactory` the `ConfigureWebHost` method runs after `Program.cs` has already executed and registered the production context. So `RemoveAll` is always needed — even when you think you are “overriding” the registration.
- For the transactional test pattern in xUnit `IAsyncLifetime` is convenient: `InitializeAsync` opens the transaction, `DisposeAsync` rolls it back. Decide which `IDbContextTransaction` you keep between them.
- `Product` is a `sealed record`. If you try to mutate `Id` after `Add`, it will not work (records are immutable). Use `with`, create a new instance, or rely on EF to populate `Id` after `SaveChanges`.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — Reference solution for homework M15-L05
// Bilingual comments: EN explains the lesson concept, RU mirrors it

using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.Data.Sqlite;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection.Extensions;
using System.Net;
using System.Net.Http.Json;
using Xunit;

// --- Production code ---

namespace Catalog.Api;

// sealed record with primary constructor: modern C# 12
public sealed record Product(int Id, string Sku, string Name, decimal Price);

public sealed class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        // Unique index on Sku — only honored by a relational provider (SQLite)
        b.Entity<Product>().HasIndex(p => p.Sku).IsUnique();
        // Name required, max 200 — again a relational constraint
        b.Entity<Product>().Property(p => p.Name).IsRequired().HasMaxLength(200);
    }
}

[ApiController]
[Route("api/products")]
public sealed class ProductsController(AppDbContext db) : ControllerBase
{
    // Primary ctor injects the context via DI
    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateProductRequest req, CancellationToken ct)
    {
        var product = new Product(0, req.Sku, req.Name, req.Price);
        db.Products.Add(product);
        await db.SaveChangesAsync(ct); // SQLite throws DbUpdateException on duplicate SKU
        return Created($"/api/products/{product.Id}", product);
    }

    [HttpGet]
    public async Task<IActionResult> GetAll(CancellationToken ct)
    {
        // AsNoTracking for reads — lesson best practice
        var list = await db.Products.AsNoTracking().ToListAsync(ct);
        return Ok(list);
    }
}

public sealed record CreateProductRequest(string Sku, string Name, decimal Price);

// Program.cs (top-level) — must be public partial for WebApplicationFactory<Program>
public partial class Program
{
    public static void Main(string[] args)
    {
        var builder = WebApplication.CreateBuilder(args);
        builder.Services.AddControllers();
        // Production registration: real SQLite file
        builder.Services.AddDbContext<AppDbContext>(o =>
            o.UseSqlite(builder.Configuration.GetConnectionString("Default")));
        var app = builder.Build();
        app.MapControllers();
        app.Run();
    }
}

// --- Test infrastructure ---

namespace Catalog.Tests;

// 1) InMemory factory — fast smoke tests
public sealed class InMemoryFactory : IDisposable
{
    public IServiceProvider Services { get; }
    public InMemoryFactory()
    {
        var services = new ServiceCollection();
        // Unique Guid => isolation between tests
        services.AddDbContext<AppDbContext>(o =>
            o.UseInMemoryDatabase($"tests-{Guid.NewGuid()}"));
        Services = services.BuildServiceProvider();
    }
    public AppDbContext CreateContext() => Services.GetRequiredService<AppDbContext>();
    public void Dispose() => (Services as IDisposable)?.Dispose();
}

// 2) SQLite in-memory factory — strict checks
public sealed class SqliteFactory : IDisposable
{
    private readonly SqliteConnection _connection; // DB holder
    public IServiceProvider Services { get; }

    public SqliteFactory()
    {
        _connection = new SqliteConnection("DataSource=:memory:");
        _connection.Open(); // DB lives while connection is open
        var services = new ServiceCollection();
        services.AddDbContext<AppDbContext>(o => o.UseSqlite(_connection));
        Services = services.BuildServiceProvider();
        // Schema once per factory
        using var scope = Services.CreateScope();
        using var ctx = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        ctx.Database.EnsureCreated();
    }

    public AppDbContext CreateContext() => Services.GetRequiredService<AppDbContext>();

    public void Dispose()
    {
        (Services as IDisposable)?.Dispose(); // containers first
        _connection.Dispose();                // connection last
    }
}

// 3) WebApplicationFactory with replaced DbContext
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
            // Remove production registrations first
            services.RemoveAll<DbContextOptions<AppDbContext>>();
            services.RemoveAll<AppDbContext>();
            // Then register the test context
            services.AddDbContext<AppDbContext>(o => o.UseSqlite(_connection));
            // Schema before the first request
            using var sp = services.BuildServiceProvider();
            using var scope = sp.CreateScope();
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

// --- Tests ---

public sealed class ProviderComparisonTests
{
    [Fact]
    public void InMemory_Allows_Duplicate_Sku_But_Only_For_Smoke()
    {
        // InMemory does NOT honor the unique index — this is a call-shape smoke test
        using var factory = new InMemoryFactory();
        using var ctx = factory.CreateContext();

        ctx.Products.Add(new Product(0, "DUP", "A", 1));
        ctx.SaveChanges();
        ctx.Products.Add(new Product(0, "DUP", "B", 2));
        ctx.SaveChanges(); // InMemory silently saves — false confidence

        Assert.Equal(2, ctx.Products.Count());
    }

    [Fact]
    public void Sqlite_Rejects_Duplicate_Sku()
    {
        using var factory = new SqliteFactory();
        using var ctx = factory.CreateContext();

        ctx.Products.Add(new Product(0, "DUP", "A", 1));
        ctx.SaveChanges();

        ctx.Products.Add(new Product(0, "DUP", "B", 2));
        var ex = Assert.Throws<DbUpdateException>(() => ctx.SaveChanges());
        Assert.Contains("UNIQUE", ex.InnerException?.Message ?? ex.Message);
    }

    [Fact]
    public void Sqlite_Requires_Name_NotEmpty()
    {
        using var factory = new SqliteFactory();
        using var ctx = factory.CreateContext();

        ctx.Products.Add(new Product(0, "S1", "", 1));
        Assert.Throws<DbUpdateException>(() => ctx.SaveChanges());
    }
}

public sealed class WebPipelineTests : IClassFixture<TestWebAppFactory>
{
    private readonly TestWebAppFactory _factory;
    public WebPipelineTests(TestWebAppFactory factory) => _factory = factory;

    [Fact]
    public async Task HttpClient_Creates_Product_Through_Real_Pipeline()
    {
        var client = _factory.CreateClient();
        var resp = await client.PostAsJsonAsync("/api/products",
            new { sku = "SKU-1", name = "Keyboard", price = 49.9m });
        Assert.Equal(HttpStatusCode.Created, resp.StatusCode);

        // GET confirms the product was actually persisted
        var all = await client.GetFromJsonAsync<List<ProductDto>>("/api/products");
        Assert.Contains(all!, p => p.Sku == "SKU-1");
    }
}

public sealed record ProductDto(int Id, string Sku, string Name, decimal Price);

// 5) Transactional test pattern
public sealed class TransactionalTests : IAsyncLifetime
{
    private SqliteFactory? _factory;
    private IDbContextTransaction? _tx;
    private AppDbContext? _ctx;

    public Task InitializeAsync()
    {
        _factory = new SqliteFactory();
        _ctx = _factory.CreateContext();
        _tx = _ctx.Database.BeginTransaction(); // will roll back
        return Task.CompletedTask;
    }

    [Fact]
    public async Task Insert_Rolls_Back_After_Dispose()
    {
        _ctx!.Products.Add(new Product(0, "T1", "Tmp", 1));
        await _ctx.SaveChangesAsync(); // inside the transaction
        Assert.Equal(1, await _ctx.Products.CountAsync()); // visible in this session
    }

    public Task DisposeAsync()
    {
        _tx?.Rollback(); // rollback — DB empty for the next test
        _ctx?.Dispose();
        _factory?.Dispose();
        return Task.CompletedTask;
    }
}
```

**Line-by-line walk-through.** `Product` is declared as a `sealed record` with positional parameters — modern C# 12 that gives immutability and value equality, convenient for DTOs and test assertions. `AppDbContext` uses the primary-constructor syntax `(DbContextOptions<AppDbContext> options) : DbContext(options)` — equivalent to a classic field but shorter; this is the “DI and replacing DbContext” concept from the lesson. In `OnModelCreating` we configure `HasIndex(p => p.Sku).IsUnique()` and `IsRequired().HasMaxLength(200)` — these constraints almost verbatim mirror the lesson example, and it is exactly what lets the SQLite tests catch a violation that InMemory will skip. That is the key teaching point: the model describes the constraints, but their enforcement depends on the provider.

`ProductsController` receives `AppDbContext` through a primary constructor from DI — this is the “edge of the system” that we will replace in tests, while the controller’s internal logic stays real (lesson best practice: “replace only the edges”). `Program` is marked `public partial class Program` so that `WebApplicationFactory<Program>` from the test project can see the entry-point type — a classic trap; without `partial` the test compiler cannot see the class.

`InMemoryFactory` registers the context with `UseInMemoryDatabase($"tests-{Guid.NewGuid()}")` — a unique name per factory gives isolation between tests under xUnit’s parallel runner (the “isolated store per test” concept from the lesson). `SqliteFactory` opens `SqliteConnection("DataSource=:memory:")` and keeps it in a private field — because the SQLite in-memory database lives only while the connection is open (a common mistake from the lesson: closed the connection → the database vanished). In the same constructor we call `EnsureCreated()` through a temporary scope — that builds the schema from the model once. In `Dispose` we release the `ServiceProvider` first (to close active scopes and contexts) and only then the `SqliteConnection` — the opposite order can raise `ObjectDisposedException` in a parallel test.

`TestWebAppFactory` is the most delicate part. In `ConfigureWebHost` we set the environment to `"Testing"`, then call `RemoveAll<DbContextOptions<AppDbContext>>()` and `RemoveAll<AppDbContext>()`. This is critical: `Program.Main` has already registered the production context, and simply adding a new registration leaves both in the container — the classic lesson mistake. We remove `DbContextOptions<T>` too because `AddDbContext` registers not only the context itself but also its `Options` object. After cleanup we register the test context with the same `_connection` and build the schema once through a temporary `BuildServiceProvider()` — this is safe because we do not leave that temporary provider in DI, we only use it for `EnsureCreated`.

In the tests, `InMemory_Allows_Duplicate_Sku_But_Only_For_Smoke` deliberately demonstrates the false confidence: InMemory saves both products with the same SKU, and the comment says so explicitly. `Sqlite_Rejects_Duplicate_Sku` expects a `DbUpdateException` and asserts that `InnerException.Message` contains `UNIQUE` — proving that the relational index fired, not some random error. `Sqlite_Requires_Name_NotEmpty` checks the required field. The integration test through `HttpClient` exercises the full path: DI → controller → `AppDbContext` → SQLite, matching the best practice “the more real code flows, the higher the trust.” `TransactionalTests` implements `IAsyncLifetime`: `InitializeAsync` opens a transaction, the test body inserts a row, `DisposeAsync` rolls back — so the database stays clean for the next test (the transactional test pattern from the lesson, which does not work with InMemory but does with SQLite).

#### Going deeper (bonus)
1. Rewrite the tests to use `Migrate()` instead of `EnsureCreated()` and add the first EF migration. Confirm that the migration actually creates the index by running `dotnet ef migrations list` and inspecting the generated SQL.
2. Add a test through `Testcontainers.MsSql` (or PostgreSQL) that spins up a real Docker container and checks the same constraints. Compare the startup time with SQLite in-memory — write the numbers into `README.md`.
3. Implement a shared-database fixture pattern through `ICollectionFixture<T>` and compare it with per-test isolation on speed and reliability. Explain in a comment when a shared fixture is justified.
4. Add a test that checks InMemory’s behavior on an explicit `BeginTransaction` call — show that rollback does not undo changes, and capture this in `README.md` as an anti-pattern.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Solution собирается командой `dotnet build` без ошибок и предупреждений.
- [ ] Все пакеты установлены с версиями `8.0.*`.
- [ ] `AppDbContext` содержит уникальный индекс по `Sku` и обязательный `Name`.
- [ ] `ProductsController` использует primary constructor и принимает `AppDbContext` из DI.
- [ ] `InMemoryFactory` создаёт изолированную БД через `Guid`.
- [ ] `SqliteFactory` держит `SqliteConnection` открытым и вызывает `EnsureCreated()`.
- [ ] `TestWebAppFactory` использует `RemoveAll` для очистки продакшен-регистрации.
- [ ] Тест `InMemory_Allows_Duplicate_Sku_But_Only_For_Smoke` проходит и комментирует ложную уверенность.
- [ ] Тест `Sqlite_Rejects_Duplicate_Sku` падает с `UNIQUE` в сообщении.
- [ ] Интеграционный тест через `HttpClient` возвращает `Created`.
- [ ] Транзакционный тест корректно откатывает изменения.
- [ ] Все тесты проходят параллельно без плавающих ошибок.
- [ ] В `README.md` есть таблица сравнения провайдеров.
- [ ] Код использует C# 12 (top-level statements, primary constructors, sealed records).
- [ ] Solution solution builds with `dotnet build` with no errors or warnings.
- [ ] All packages installed with `8.0.*` versions.
- [ ] `AppDbContext` contains a unique index on `Sku` and a required `Name`.
- [ ] `ProductsController` uses a primary constructor and receives `AppDbContext` from DI.
- [ ] `InMemoryFactory` creates an isolated DB via `Guid`.
- [ ] `SqliteFactory` keeps the `SqliteConnection` open and calls `EnsureCreated()`.
- [ ] `TestWebAppFactory` uses `RemoveAll` to clear the production registration.
- [ ] The `InMemory_Allows_Duplicate_Sku_But_Only_For_Smoke` test passes and comments on the false confidence.
- [ ] The `Sqlite_Rejects_Duplicate_Sku` test fails with `UNIQUE` in the message.
- [ ] The integration test through `HttpClient` returns `Created`.
- [ ] The transactional test rolls back changes correctly.
- [ ] All tests pass in parallel without flaky failures.
- [ ] `README.md` contains the provider comparison table.
- [ ] The code uses C# 12 (top-level statements, primary constructors, sealed records).

#### Ресурсы / Resources
- [Microsoft Learn — EF Core testing](https://learn.microsoft.com/ef/core/testing/)
- [EF Core InMemory provider](https://learn.microsoft.com/ef/core/providers/in-memory/)
- [SQLite in-memory testing — testing without the database](https://learn.microsoft.com/ef/core/testing/testing-without-the-database)
- [WebApplicationFactory — integration tests in ASP.NET Core](https://learn.microsoft.com/aspnet/core/test/integration-tests)
- [Testcontainers for .NET](https://dotnet.testcontainers.org/)
- [xUnit parallel test execution](https://xunit.net/docs/running-tests-in-parallel)
