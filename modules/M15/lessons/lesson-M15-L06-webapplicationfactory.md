[⬆ К модулю M15](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M15-L06: WebApplicationFactory, integration-тесты API / WebApplicationFactory, API integration tests

**Модуль / Module:** M15
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Интеграционные тесты проверяют систему «изнутри наружу»: не отдельные методы, а связанные компоненты, работающие вместе. В ASP.NET Core инструментом для этого служит `WebApplicationFactory<TEntryPoint>` из пакета `Microsoft.AspNetCore.Mvc.Testing`. Он запускает приложение в памяти через `TestServer` и выдаёт настроенный `HttpClient`, который ходит по маршруту в тестовый сервер, а не по реальному сетевому порту. Аналогия: вы не открываете дверь дома ключом с улицы, а используете служебный ход внутри здания — быстрее, безопаснее, не зависит от погоды.

`TestServer` перехватывает HTTP-вызовы ещё до того, как они покидают процесс. Запрос проходит полный конвейер: routing, model binding, middleware, контроллеры или minimal endpoints, фильтры, валидацию. Поэтому end-to-end тест endpoint покрывает больше, чем модульный тест контроллера с подменёнными сервисами. Но платой за широту покрытия является скорость: тысячи интеграционных тестов работают медленнее сотен модульных, поэтому баланс важен.

Ключевая возможность — `ConfigureWebHost` через наследование от `WebApplicationFactory<Program>`. Здесь можно заменить зарегистрированные сервисы: например, реальную базу данных на in-memory EF Core контекст, внешний HTTP-клиент на стаб, email-отправщик на шпион. Метод `builder.ConfigureTestServices(s => s.Replace(...))` точечно подменяет зависимость, сохраняя всё остальное. Аналогия: вы не строите тестовый дом заново, а меняете в настоящем только плиту и счётчик.

Важно различать `Program` и `Program.cs`. С .NET 6+ используется minimal hosting model, и `WebApplicationFactory<Program>` ссылается на сгенерированный класс `Program`. Чтобы он был виден из тестового проекта, добавляют `public partial class Program {}` в конце `Program.cs` либо импортируют `InternalsVisibleTo`. Тестовый проект также обязан ссылаться на SCS-проект через `<ProjectReference>`.

Авторизация тестируется двумя путями. Либо отключают middleware-фильтры безопасности через `ConfigureTestServices`, либо подменяют `AuthenticationHandler<T>`, возвращающий клейм-тестового пользователя. Второй путь реалистичнее: проверяет и ролевую модель, и маршруты. Базу данных обычно поднимают в Docker (Testcontainers) или SQLite in-memory для скорости; реальную СУБД подключают только там, где нужен её диалект SQL.

Тестируйте статусы (`StatusCode`), тело (`ReadAsAsync<T>`), заголовки (`Location`, `Content-Type`) и побочные эффекты. Команда `client.GetAsync("/api/products/42")` даёт `HttpResponseMessage` — единый объект, через который проходит вся проверка. Размещайте фабрику в `IClassFixture<>` (xUnit) или `ICollectionFixture<>`, чтобы один экземпляр сервера обслуживался многими тестами и не пересоздавался на каждый метод. Чистите данные в `IAsyncLifetime`, чтобы тесты оставались изолированными.

Итог: `WebApplicationFactory` — мост между миром модульных тестов и реальным выполнением HTTP-конвейера. Он даёт почти полную уверенность, что endpoint работает, замены сервисов изолируют внешние зависимости, а `TestServer` исключает сетевой шум. Используйте его для критических маршрутов, а модульные тесты оставьте для бизнес-логики.

#### Theory (EN)

Integration tests verify the system from the inside out: not individual methods, but cooperating components working together. In ASP.NET Core the tool for this is `WebApplicationFactory<TEntryPoint>` from the `Microsoft.AspNetCore.Mvc.Testing` package. It boots the application in memory through `TestServer` and hands you a preconfigured `HttpClient` that routes calls into the test server instead of a real network port. Analogy: rather than unlocking the front door from the street, you use an interior service corridor — faster, safer, and weatherproof.

`TestServer` intercepts HTTP calls before they ever leave the process. The request runs the entire pipeline: routing, model binding, middleware, controllers or minimal endpoints, filters, validation. That is why an end-to-end endpoint test covers far more than a controller unit test with mocked services. The trade-off is speed: thousands of integration tests run slower than hundreds of unit tests, so balance matters.

The crucial capability is overriding `ConfigureWebHost` by subclassing `WebApplicationFactory<Program>`. Here you replace registered services — for example, swap the real database for an in-memory EF Core context, an external HTTP client for a stub, or an email sender for a spy. The `builder.ConfigureTestServices(s => s.Replace(...))` call surgically substitutes one dependency while leaving everything else intact. Analogy: you do not rebuild a test house, you only replace the stove and the meter in the real one.

It is important to distinguish `Program` from `Program.cs`. With .NET 6+ and the minimal hosting model, `WebApplicationFactory<Program>` references the generated `Program` class. To expose it to the test project you add `public partial class Program {}` at the bottom of `Program.cs` or grant `InternalsVisibleTo`. The test project must also reference the SUT via `<ProjectReference>`.

Authorization is tested two ways. Either disable security middleware filters via `ConfigureTestServices`, or substitute an `AuthenticationHandler<T>` that returns the test user's claims. The second path is more realistic: it exercises role checks and routes together. The database is usually lifted in Docker (Testcontainers) or SQLite in-memory for speed; the real DBMS is only attached where its SQL dialect matters.

Assert on statuses (`StatusCode`), bodies (`ReadAsJsonAsync<T>`), headers (`Location`, `Content-Type`), and side effects. The call `client.GetAsync("/api/products/42")` returns an `HttpResponseMessage` — a single object through which the whole verification flows. Place the factory in `IClassFixture<>` (xUnit) or `ICollectionFixture<>` so one server instance serves many tests rather than being recreated per method. Clean data in `IAsyncLifetime` so tests stay isolated.

In summary, `WebApplicationFactory` is the bridge between unit tests and a genuinely running HTTP pipeline. It gives near-complete confidence that an endpoint works, service substitutions isolate external dependencies, and `TestServer` removes network noise. Use it for critical routes and keep unit tests for business logic.

#### Пример кода / Code Example

```csharp
// === Тестируемый endpoint (SUT: src/Api/Program.cs) ===
// Endpoint under test (SUT: src/Api/Program.cs)
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Регистрируем EF Core контекст / Register EF Core context
builder.Services.AddDbContext<AppDb>(opt =>
    opt.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
builder.Services.AddSingleton<IClock, SystemClock>();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddAuthorization();

var app = builder.Build();

// GET /api/products/{id} — получить продукт / get product
app.MapGet("/api/products/{id:int}", async (int id, AppDb db) =>
{
    var product = await db.Products.FindAsync(id);
    return product is null ? Results.NotFound() : Results.Ok(product);
});

// POST /api/products — создать продукт / create product
app.MapPost("/api/products", async (Product p, AppDb db) =>
{
    db.Products.Add(p);
    await db.SaveChangesAsync();
    return Results.Created($"/api/products/{p.Id}", p);
});

app.Run();

// Делаем Program видимым для тестового проекта / Expose Program to test project
public partial class Program { }

public class AppDb(DbContextOptions<AppDb> o) : DbContext(o)
{
    public DbSet<Product> Products => Set<Product>();
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
}

public interface IClock { DateTime UtcNow { get; } }
public class SystemClock : IClock { public DateTime UtcNow => DateTime.UtcNow; }

// === Тестовый проект: tests/Api.IntegrationTests/Factory.cs ===
// Test project: tests/Api.IntegrationTests/Factory.cs
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;

namespace Api.IntegrationTests;

// Фабрика подменяет реальную SQL Server БД на in-memory EF Core
// The factory swaps the real SQL Server DB for in-memory EF Core
public class TestFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("IntegrationTests");

        builder.ConfigureTestServices(services =>
        {
            // Удаляем реальную регистрацию AppDb / Remove the real AppDb registration
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<AppDb>));
            if (descriptor is not null) services.Remove(descriptor);

            // Регистрируем in-memory для каждого теста / Register in-memory per test
            services.AddDbContext<AppDb>(opt => opt.UseInMemoryDatabase("tests"));

            // Подменяем IClock фиксированным временем / Substitute IClock with fixed time
            services.RemoveAll<IClock>();
            services.AddSingleton<IClock>(new FixedClock(
                new DateTime(2025, 1, 1, 0, 0, 0, DateTimeKind.Utc)));
        });
    }

    // Возвращает новый экземпляр БД, чтобы тесты не влияли друг на друга
    // Returns a fresh DB so tests don't affect each other
    public async Task<AppDb> ResetDbAsync()
    {
        var scope = Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDb>();
        await db.Database.EnsureDeletedAsync();
        await db.Database.EnsureCreatedAsync();
        return db;
    }
}

public sealed class FixedClock(DateTime fixedUtc) : IClock
{
    public DateTime UtcNow => fixedUtc;
}

// === Тестовый проект: tests/Api.IntegrationTests/ProductEndpointsTests.cs ===
// Test project: tests/Api.IntegrationTests/ProductEndpointsTests.cs
using System.Net;
using System.Net.Http.Json;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;

namespace Api.IntegrationTests;

public class ProductEndpointsTests : IClassFixture<TestFactory>, IAsyncLifetime
{
    private readonly TestFactory _factory;
    private readonly HttpClient _client;
    private AppDb _db = null!;

    public ProductEndpointsTests(TestFactory factory)
    {
        _factory = factory;
        _client = factory.CreateClient(); // готовый HttpClient к TestServer / HttpClient to TestServer
    }

    public async Task InitializeAsync() => _db = await _factory.ResetDbAsync();
    public Task DisposeAsync() => Task.CompletedTask;

    [Fact]
    public async Task Get_ExistingProduct_Returns200WithBody()
    {
        // Arrange — сеем данные напрямую через контекст / Seed via context directly
        _db.Products.Add(new Product { Id = 42, Name = "Keyboard" });
        await _db.SaveChangesAsync();

        // Act — HTTP-вызов проходит весь конвейер / HTTP call runs the whole pipeline
        var response = await _client.GetAsync("/api/products/42");

        // Assert
        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
        var product = await response.Content.ReadFromJsonAsync<Product>();
        Assert.NotNull(product);
        Assert.Equal("Keyboard", product!.Name);
    }

    [Fact]
    public async Task Get_MissingProduct_Returns404()
    {
        var response = await _client.GetAsync("/api/products/9999");
        Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
    }

    [Fact]
    public async Task Post_Product_Returns201WithLocation()
    {
        var response = await _client.PostAsJsonAsync("/api/products", new { Name = "Mouse" });

        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        Assert.NotNull(response.Headers.Location);

        var created = await response.Content.ReadFromJsonAsync<Product>();
        Assert.NotNull(created);
        Assert.True(created!.Id > 0);
    }
}
```

#### Best Practices
- Держите один экземпляр `WebApplicationFactory` на класс тестов через `IClassFixture` — запуск `TestServer` дорогой.
- Изолируйте внешние зависимости (БД, HTTP, email) через `ConfigureTestServices`, но оставляйте внутреннюю бизнес-логику реальной.
- Чистите данные перед каждым тестом в `IAsyncLifetime.InitializeAsync`, чтобы порядок выполнения не влиял на результат.
- Покрывайте не только счастливый путь: проверяйте 404, 400, 401, 403 и граничные входные данные.
- Use one `WebApplicationFactory` instance per test class via `IClassFixture` — booting `TestServer` is expensive.
- Isolate external dependencies (DB, HTTP, email) through `ConfigureTestServices`, but keep internal business logic real.
- Reset data before each test in `IAsyncLifetime.InitializeAsync` so execution order never affects results.
- Cover more than the happy path: assert 404, 400, 401, 403, and boundary inputs.

#### Частые ошибки / Common Mistakes
- Подмена всех сервисов через `AddTransient` вместо `Replace` → теряется реальный конвейер; используйте `services.Replace(ServiceDescriptor.Transient(...))` для точечной подмены. (RU)
- Забыли `public partial class Program {}` → `WebApplicationFactory<Program>` не компилируется в тестовом проекте; добавьте partial-класс или `InternalsVisibleTo`. (RU)
- Разделяете состояние БД между тестами → плавающие баги; чистите таблицы в `InitializeAsync` или используйте уникальное имя in-memory БД на тест. (RU)
- Тестируете только статус-код → пропускаете ошибки в теле и заголовках; добавляйте `ReadFromJsonAsync<T>` и проверки `Location`/`Content-Type`. (RU)
- Замена всех services via `AddTransient` instead of `Replace` → real pipeline lost; use `services.Replace(ServiceDescriptor.Transient(...))` for a targeted swap. (EN)
- Forgetting `public partial class Program {}` → `WebApplicationFactory<Program>` fails to compile in the test project; add the partial class or `InternalsVisibleTo`. (EN)
- Sharing DB state between tests → flaky failures; clean tables in `InitializeAsync` or give each test a unique in-memory DB name. (EN)
- Asserting only the status code → bugs in body and headers slip through; add `ReadFromJsonAsync<T>` and checks on `Location`/`Content-Type`. (EN)

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Тестовый проект ссылается на SUT и имеет `Microsoft.AspNetCore.Mvc.Testing`. (RU)
- [ ] `Program` видим из тестового проекта через partial или `InternalsVisibleTo`. (RU)
- [ ] Реализован класс-наследник `WebApplicationFactory<Program>` с `ConfigureWebHost`. (RU)
- [ ] Внешние зависимости (БД, HTTP, время) заменены в `ConfigureTestServices`. (RU)
- [ ] Фабрика используется через `IClassFixture`, данные чистятся в `IAsyncLifetime`. (RU)
- [ ] Тесты проверяют статус, тело и заголовки ответа, не только счастливый путь. (RU)
- [ ] The test project references the SUT and has `Microsoft.AspNetCore.Mvc.Testing`. (EN)
- [ ] `Program` is visible to the test project via partial or `InternalsVisibleTo`. (EN)
- [ ] A `WebApplicationFactory<Program>` subclass with `ConfigureWebHost` is implemented. (EN)
- [ ] External dependencies (DB, HTTP, clock) are substituted in `ConfigureTestServices`. (EN)
- [ ] The factory is shared via `IClassFixture` and data is reset in `IAsyncLifetime`. (EN)
- [ ] Tests assert status, body, and headers — not only the happy path. (EN)

#### Ресурсы / Resources
- [Microsoft Learn — Integration tests in ASP.NET Core — https://learn.microsoft.com/aspnet/core/test/integration-tests]

---

[⬆ К модулю M15](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
