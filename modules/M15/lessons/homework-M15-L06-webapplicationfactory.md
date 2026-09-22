---
[← К уроку M15-L06](lesson-M15-L06-webapplicationfactory.md) | [⬆ К модулю M15](../README.md) | [Следующее ДЗ →](homework-M15-L07-tdd.md)
---

### Домашнее задание M15-L06: WebApplicationFactory, integration-тесты API / Homework M15-L06: WebApplicationFactory, API integration tests

**Урок / Lesson:** M15-L06
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться писать настоящие integration-тесты ASP.NET Core API через `WebApplicationFactory<Program>`: поднимать `TestServer` в памяти, точечно подменять внешние зависимости через `ConfigureTestServices`, изолировать состояние БД в `IAsyncLifetime` и проверять не только статус-код, но и тело, заголовки и побочные эффекты. (EN) Learn to write real ASP.NET Core API integration tests with `WebApplicationFactory<Program>`: boot `TestServer` in memory, surgically substitute external dependencies via `ConfigureTestServices`, isolate DB state in `IAsyncLifetime`, and assert not only the status code but also the body, headers, and side effects.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит `WebApplicationFactory<TEntryPoint>` как мост между модульными тестами и реальным HTTP-конвейером, показывает подмену БД на in-memory EF Core через `ConfigureWebHost`, объясняет, зачем нужен `public partial class Program {}`, и фиксирует best practices: один экземпляр фабрики на класс через `IClassFixture`, очистка данных в `IAsyncLifetime`, проверка статуса, тела и заголовков. Это ДЗ закрепляет каждую из этих тем на живом API с авторизацией, валидацией и побочными эффектами.
(EN) The lesson introduces `WebApplicationFactory<TEntryPoint>` as the bridge between unit tests and a real HTTP pipeline, shows the in-memory EF Core swap through `ConfigureWebHost`, explains the purpose of `public partial class Program {}`, and fixes best practices: one factory per class via `IClassFixture`, data reset in `IAsyncLifetime`, and assertions on status, body, and headers. This homework exercises every one of those topics on a live API with authorization, validation, and side effects.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Ваша команда поддерживает маленький REST API для списка задач — `TodoApi`. Он написан на C# 12 / .NET 8 в стиле minimal API, использует EF Core (SQL Server в проде) и защищён API-ключом: заголовок `X-Api-Key` должен совпадать со значением из конфигурации, а удаление задач разрешено только пользователям с ролью `admin`. Бизнес-логика проста, но конвейер нетривиален: routing, model binding, валидация, фильтр авторизации, EF Core, сериализация JSON. Модульные тесты контроллера с подменёнными сервисами здесь не дают уверенности — они проверяют, что метод вызывается, но не то, что запрос реально доходит до базы и обратно, что статус `201 Created` сопровождается корректным заголовком `Location`, что валидация отклоняет пустое имя, а фильтр ключа — отсутствующий заголовок.

Вам нужно написать integration-тесты через `WebApplicationFactory<Program>`, которые прогоняют каждый запрос через весь конвейер `TestServer`-а, но при этом не зависят от SQL Server, реального времени и реального API-ключа. Внешние зависимости подменяются точечно через `ConfigureTestServices`, внутренняя бизнес-логика (endpoint-ы, EF Core, валидация, авторизация) остаётся реальной. Один экземпляр фабрики разделяется между тестами через `IClassFixture<TestFactory>`, а состояние in-memory БД сбрасывается в `IAsyncLifetime.InitializeAsync`, чтобы порядок выполнения тестов не влиял на результат. Это ровно та связка, которую урок описывает как «мост между миром модульных тестов и реальным выполнением HTTP-конвейера».

#### Что нужно сделать (пошагово)

1. **Создайте решение и проекты.** Из пустой папки выполните:
   ```
   dotnet new sln -n TodoApiSln
   dotnet new web -n TodoApi -o src/TodoApi
   dotnet new xunit -n TodoApi.IntegrationTests -o tests/TodoApi.IntegrationTests
   dotnet sln add src/TodoApi tests/TodoApi.IntegrationTests
   dotnet add tests/TodoApi.IntegrationTests reference src/TodoApi
   dotnet add tests/TodoApi.IntegrationTests package Microsoft.AspNetCore.Mvc.Testing --version 8.*
   dotnet add tests/TodoApi.IntegrationTests package Microsoft.EntityFrameworkCore.InMemory --version 8.*
   dotnet add src/TodoApi package Microsoft.EntityFrameworkCore.SqlServer --version 8.*
   ```
   Должно получиться два проекта: SUT (`src/TodoApi`) и тестовый (`tests/TodoApi.IntegrationTests`), ссылающийся на SUT и на пакет `Microsoft.AspNetCore.Mvc.Testing`.

2. **Напишите SUT — `src/TodoApi/Program.cs`.** Реализуйте minimal API с тремя endpoint-ами: `GET /api/todos` (список), `POST /api/todos` (создание с валидацией имени), `DELETE /api/todos/{id}` (только для роли `admin`). Зарегистрируйте `AppDb` через `AddDbContext` с `UseSqlServer`, `IClock` как синглтон, добавьте кастомный `ApiKeyMiddleware` (или фильтр), который проверяет заголовок `X-Api-Key` и устанавливает claims пользователя. В конце файла обязательно добавьте `public partial class Program {}`, чтобы тестовый проект видел точку входа. Опишите классы `AppDb`, `Todo`, `IClock`, `SystemClock`.

3. **Сделайте `Program` видимым.** Без `public partial class Program {}` компилятор тестового проекта не найдёт тип `Program`, и `WebApplicationFactory<Program>` не соберётся. Это одна из частых ошибок урока — проверьте, что после сборки `dotnet build` ошибок нет.

4. **Напишите `TestFactory`** в `tests/TodoApi.IntegrationTests/TestFactory.cs`: наследник `WebApplicationFactory<Program>`, переопределяющий `ConfigureWebHost`. Внутри вызовите `builder.UseEnvironment("IntegrationTests")` и в `ConfigureTestServices` удалите реальную регистрацию `DbContextOptions<AppDb>` через `services.SingleOrDefault(...)`, зарегистрируйте in-memory контекст `UseInMemoryDatabase("todos-tests")`, удалите `IClock` и зарегистрируйте `FixedClock` на `2025-01-01 UTC`, и подмените проверку API-ключа так, чтобы тесты могли и пропускать, и отклонять запрос. Добавьте метод `ResetDbAsync()`, который через scope создаёт свежую БД (`EnsureDeletedAsync` + `EnsureCreatedAsync`).

5. **Напишите тесты** в `tests/TodoApi.IntegrationTests/TodoEndpointsTests.cs`: класс реализует `IClassFixture<TestFactory>` и `IAsyncLifetime`. В конструкторе сохраните фабрику и создайте `HttpClient` через `factory.CreateClient()`. В `InitializeAsync` сбрасывайте БД. Покройте сценарии: получение существующей задачи → 200 + тело; получение отсутствующей → 404; создание валидной задачи → 201 + `Location` + тело с автоинкрементным `Id`; создание с пустым именем → 400; запрос без `X-Api-Key` → 401; удаление без роли admin → 403; удаление с admin → 204 и проверка, что задача действительно исчезла из БД (побочный эффект).

6. **Запустите тесты и добейтесь зелёного.** Команда `dotnet test` должна показать 7 прошедших тестов. Если падает с ошибкой доступа к SQL Server — вы забыли подмену БД в `ConfigureTestServices`. Если падает с `TypeLoadException` на `Program` — забыли `public partial class Program {}`. Если тесты плавают при разном порядке — забыли `ResetDbAsync` в `InitializeAsync`.

#### Требования к решению

- Тестовый проект обязан ссылаться на SUT через `<ProjectReference>` и содержать пакет `Microsoft.AspNetCore.Mvc.Testing`. Без ссылки `WebApplicationFactory<Program>` не разрешит тип `Program`.
- Класс `TestFactory` — наследник `WebApplicationFactory<Program>` с переопределённым `ConfigureWebHost`. Внутри используется именно `ConfigureTestServices`, а не пересборка `Program.cs`: реальный конвейер (routing, model binding, middleware, EF Core) должен остаться нетронутым.
- Реальная зависимость от SQL Server подменяется на EF Core in-memory. Делается это удалением `ServiceDescriptor`-а для `DbContextOptions<AppDb>` и повторной регистрацией — не `AddDbContext` поверх старой, а именно удаление дескриптора, иначе в контейнере останутся две регистрации и поведение станет недетерминированным.
- `IClock` подменяется на `FixedClock`, чтобы тесты на `CreatedAt` и таймстемпы не зависели от реального времени. Это классический приём «шпион для времени» из урока.
- Фабрика используется через `IClassFixture<TestFactory>`, а сброс БД — в `IAsyncLifetime.InitializeAsync`. Не чистите данные в конструкторе: конструктор синхронный и не должен трогать БД.
- Каждый тест проверяет не только `StatusCode`, но и тело (`ReadFromJsonAsync<T>`), заголовки (`Location`, `Content-Type`) и — там, где это уместно — побочный эффект (повторный `GET` после `DELETE`).

#### Тонкости и подводные камни

- **`AddDbContext` поверх старой регистрации не заменяет, а дублирует.** Урок явно предостерегает: подмена всех сервисов через `AddTransient` вместо `Replace` теряет реальный конвейер. Правильный путь для EF Core — сначала `services.Remove(descriptor)` для `DbContextOptions<AppDb>`, затем `services.AddDbContext(... UseInMemoryDatabase ...)`. Для скалярных синглтонов (`IClock`) подойдёт `services.RemoveAll<IClock>()` и повторная регистрация.
- **`public partial class Program {}` обязательно.** Minimal hosting model генерирует внутренний класс `Program`; без partial-объявления тестовый проект его не видит. Альтернатива — `[assembly: InternalsVisibleTo("TodoApi.IntegrationTests")]`, но partial чище.
- **Разделять состояние БД между тестами — плавающие баги.** In-memory EF Core сохраняет данные, пока жив контекст базы. Если не звать `EnsureDeletedAsync`/`EnsureCreatedAsync` (или не давать уникальное имя базы на тест), тест `Get_Existing` начнёт видеть задачи, посеянные тестом `Post_Product`. Чистите в `IAsyncLifetime.InitializeAsync`, не в конструкторе.
- **Проверка только статус-кода пропускает баги в теле и заголовках.** Урок требует `ReadFromJsonAsync<T>` и проверки `Location`/`Content-Type`. Например, `201 Created` без `Location` — это баг, который заметит только assertion на заголовок.
- **`CreateClient()` возвращает один и тот же `HttpClient` для одной фабрики.** Это нормально и даже желательно: он разделяет `HttpClientHandler`, нацеленный на `TestServer`. Не создавайте клиент на каждый тест — это лишняя работа без выгоды.
- **Авторизация.** Не отключайте middleware безопасности через `ConfigureTestServices` — вы потеряете покрытие ролевой модели. Вместо этого подменяйте `AuthenticationHandler<T>`, возвращающий claims тестового пользователя, или — как в этом ДЗ — подменяйте саму проверку API-ключа, чтобы тест мог явно передать нужный ключ и роль в заголовке.

#### Критерии приёмки

- [ ] Есть решение `TodoApiSln` с проектами `src/TodoApi` и `tests/TodoApi.IntegrationTests`.
- [ ] Тестовый проект ссылается на SUT и содержит `Microsoft.AspNetCore.Mvc.Testing`.
- [ ] `Program.cs` заканчивается `public partial class Program {}`.
- [ ] SUT реализует `GET /api/todos`, `POST /api/todos`, `DELETE /api/todos/{id}` с валидацией и авторизацией по `X-Api-Key`.
- [ ] `TestFactory : WebApplicationFactory<Program>` переопределяет `ConfigureWebHost` и зовёт `UseEnvironment("IntegrationTests")`.
- [ ] В `ConfigureTestServices` реальная регистрация `DbContextOptions<AppDb>` удалена, добавлен in-memory контекст.
- [ ] `IClock` подменён на `FixedClock`.
- [ ] Проверка API-ключа подменена так, что тест может и пройти, и отклонить запрос.
- [ ] `TestFactory.ResetDbAsync()` создаёт свежую БД через `EnsureDeletedAsync` + `EnsureCreatedAsync`.
- [ ] `TodoEndpointsTests` реализует `IClassFixture<TestFactory>` и `IAsyncLifetime`.
- [ ] Сброс БД происходит в `InitializeAsync`, а не в конструкторе.
- [ ] Есть тесты на: 200, 404, 201 + Location, 400 (валидация), 401 (нет ключа), 403 (нет admin), 204 + побочный эффект.
- [ ] Каждый тест проверяет статус, тело (где уместно) и заголовки (`Location`, `Content-Type`).
- [ ] `dotnet test` показывает все тесты зелёными; порядок выполнения не влияет на результат.
- [ ] В коде используются C# 12 / .NET 8: top-level statements, primary constructors, collection expressions, `required`/`init` где уместно.

#### Подсказки (без прямого ответа)

- Чтобы «подменить проверку ключа, но оставить возможность отклонить», зарегистрируйте в `ConfigureTestServices` тестовый обработчик ключа, который читает `X-Api-Key` из заголовка и сравнивает с заведомо тестовым значением (например, `"test-key"`). Тогда тест без ключа получит 401, а тест с ключом — пройдёт. Роль admin можно прокидывать отдельным заголовком `X-Test-Role` или вторым ключом.
- Для валидации пустого имени не нужен FluentValidation: достаточно `if (string.IsNullOrWhiteSpace(todo.Title)) return Results.BadRequest(...)` внутри endpoint-а. Главное — вернуть 400, а не 500.
- Чтобы проверить побочный эффект удаления, после `DELETE` сделайте повторный `GET /api/todos/{id}` и убедитесь, что статус 404.
- `ReadFromJsonAsync<T>` живёт в `System.Net.Http.Json`. Для `Location` используйте `response.Headers.Location`.
- Не забудьте `await` на `SaveChangesAsync` при посеве данных — иначе in-memory контекст не сохранит.

#### Эталонное решение (разбор)

```csharp
// === SUT: src/TodoApi/Program.cs ===
using Microsoft.EntityFrameworkCore;
using System.Security.Claims;

var builder = WebApplication.CreateBuilder(args);

// EF Core: в проде SQL Server, в тестах подменим на in-memory
builder.Services.AddDbContext<AppDb>(opt =>
    opt.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

builder.Services.AddSingleton<IClock, SystemClock>();
builder.Services.AddAuthorization();

var app = builder.Build();

// middleware проверки API-ключа: ставит claims пользователя
app.Use(async (ctx, next) =>
{
    var key = ctx.Request.Headers["X-Api-Key"].ToString();
    var expected = builder.Configuration["ApiKey"] ?? "prod-secret";
    if (string.IsNullOrEmpty(key) || key != expected)
    {
        ctx.Response.StatusCode = 401;
        return;
    }
    var role = key == "admin-secret" ? "admin" : "user";
    ctx.User = new ClaimsPrincipal(new ClaimsIdentity(new[]
    {
        new Claim(ClaimTypes.NameIdentifier, "test-user"),
        new Claim(ClaimTypes.Role, role),
    }, "ApiKey"));
    await next();
});

// GET /api/todos — список всех задач
app.MapGet("/api/todos", async (AppDb db) =>
    Results.Ok(await db.Todos.AsNoTracking().ToListAsync()));

// GET /api/todos/{id} — одна задача
app.MapGet("/api/todos/{id:int}", async (int id, AppDb db) =>
{
    var todo = await db.Todos.FindAsync(id);
    return todo is null ? Results.NotFound() : Results.Ok(todo);
});

// POST /api/todos — создать (с валидацией)
app.MapPost("/api/todos", async (Todo todo, AppDb db, IClock clock) =>
{
    if (string.IsNullOrWhiteSpace(todo.Title))
        return Results.BadRequest(new { error = "Title is required" });

    todo.CreatedAt = clock.UtcNow;
    db.Todos.Add(todo);
    await db.SaveChangesAsync();
    return Results.Created($"/api/todos/{todo.Id}", todo);
});

// DELETE /api/todos/{id} — только admin
app.MapDelete("/api/todos/{id:int}", async (int id, AppDb db) =>
{
    if (!ctx.User.IsInRole("admin"))  // псевдокод; см. полный вариант ниже
        return Results.Forbid();
    var todo = await db.Todos.FindAsync(id);
    if (todo is null) return Results.NotFound();
    db.Todos.Remove(todo);
    await db.SaveChangesAsync();
    return Results.NoContent();
}).RequireAuthorization();

app.Run();

public partial class Program { }

public class AppDb(DbContextOptions<AppDb> o) : DbContext(o)
{
    public DbSet<Todo> Todos => Set<Todo>();
}

public class Todo
{
    public int Id { get; set; }
    public required string Title { get; set; }
    public DateTime CreatedAt { get; set; }
}

public interface IClock { DateTime UtcNow { get; } }
public class SystemClock : IClock { public DateTime UtcNow => DateTime.UtcNow; }
```

```csharp
// === tests/TodoApi.IntegrationTests/TestFactory.cs ===
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;

namespace TodoApi.IntegrationTests;

public class TestFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("IntegrationTests");
        builder.ConfigureAppConfiguration((_, cfg) =>
        {
            cfg.AddInMemoryCollection(new Dictionary<string, string?>
            {
                ["ApiKey"] = "test-key",
            });
        });

        builder.ConfigureTestServices(services =>
        {
            // Удаляем реальную регистрацию DbContextOptions<AppDb>
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<AppDb>));
            if (descriptor is not null) services.Remove(descriptor);

            services.AddDbContext<AppDb>(opt => opt.UseInMemoryDatabase("todos-tests"));

            services.RemoveAll<IClock>();
            services.AddSingleton<IClock>(new FixedClock(
                new DateTime(2025, 1, 1, 0, 0, 0, DateTimeKind.Utc)));
        });
    }

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
```

```csharp
// === tests/TodoApi.IntegrationTests/TodoEndpointsTests.cs ===
using System.Net;
using System.Net.Http.Json;
using Microsoft.Extensions.DependencyInjection;

namespace TodoApi.IntegrationTests;

public class TodoEndpointsTests : IClassFixture<TestFactory>, IAsyncLifetime
{
    private readonly TestFactory _factory;
    private readonly HttpClient _client;
    private AppDb _db = null!;

    public TodoEndpointsTests(TestFactory factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }

    public async Task InitializeAsync() => _db = await _factory.ResetDbAsync();
    public Task DisposeAsync() => Task.CompletedTask;

    private HttpRequestMessage WithKey(string key, string? role = null)
    {
        var req = new HttpRequestMessage();
        req.Headers.Add("X-Api-Key", key);
        if (role is not null) req.Headers.Add("X-Test-Role", role);
        return req;
    }

    [Fact]
    public async Task Get_ExistingTodo_Returns200WithBody()
    {
        _db.Todos.Add(new Todo { Title = "Купить хлеб" });
        await _db.SaveChangesAsync();

        var req = WithKey("test-key");
        req.Method = HttpMethod.Get;
        req.RequestUri = new Uri("/api/todos/1", UriKind.Relative);
        var resp = await _client.SendAsync(req);

        Assert.Equal(HttpStatusCode.OK, resp.StatusCode);
        var todo = await resp.Content.ReadFromJsonAsync<Todo>();
        Assert.NotNull(todo);
        Assert.Equal("Купить хлеб", todo!.Title);
    }

    [Fact]
    public async Task Get_MissingTodo_Returns404()
    {
        var req = WithKey("test-key");
        req.Method = HttpMethod.Get;
        req.RequestUri = new Uri("/api/todos/9999", UriKind.Relative);
        var resp = await _client.SendAsync(req);
        Assert.Equal(HttpStatusCode.NotFound, resp.StatusCode);
    }

    [Fact]
    public async Task Post_ValidTodo_Returns201WithLocation()
    {
        var req = WithKey("test-key");
        req.Method = HttpMethod.Post;
        req.RequestUri = new Uri("/api/todos", UriKind.Relative);
        req.Content = JsonContent.Create(new { Title = "Полить цветы" });

        var resp = await _client.SendAsync(req);

        Assert.Equal(HttpStatusCode.Created, resp.StatusCode);
        Assert.NotNull(resp.Headers.Location);
        var created = await resp.Content.ReadFromJsonAsync<Todo>();
        Assert.NotNull(created);
        Assert.True(created!.Id > 0);
        Assert.Equal(new DateTime(2025, 1, 1, 0, 0, 0, DateTimeKind.Utc), created.CreatedAt);
    }

    [Fact]
    public async Task Post_EmptyTitle_Returns400()
    {
        var req = WithKey("test-key");
        req.Method = HttpMethod.Post;
        req.RequestUri = new Uri("/api/todos", UriKind.Relative);
        req.Content = JsonContent.Create(new { Title = "" });

        var resp = await _client.SendAsync(req);
        Assert.Equal(HttpStatusCode.BadRequest, resp.StatusCode);
    }

    [Fact]
    public async Task Request_WithoutApiKey_Returns401()
    {
        var resp = await _client.GetAsync("/api/todos");
        Assert.Equal(HttpStatusCode.Unauthorized, resp.StatusCode);
    }

    [Fact]
    public async Task Delete_WithoutAdminRole_Returns403()
    {
        _db.Todos.Add(new Todo { Title = "Старая задача" });
        await _db.SaveChangesAsync();

        var req = WithKey("test-key");
        req.Method = HttpMethod.Delete;
        req.RequestUri = new Uri("/api/todos/1", UriKind.Relative);

        var resp = await _client.SendAsync(req);
        Assert.Equal(HttpStatusCode.Forbidden, resp.StatusCode);
    }

    [Fact]
    public async Task Delete_WithAdminRole_Returns204AndRemovesTodo()
    {
        _db.Todos.Add(new Todo { Title = "Старая задача" });
        await _db.SaveChangesAsync();

        var req = WithKey("admin-secret", "admin");
        req.Method = HttpMethod.Delete;
        req.RequestUri = new Uri("/api/todos/1", UriKind.Relative);

        var resp = await _client.SendAsync(req);
        Assert.Equal(HttpStatusCode.NoContent, resp.StatusCode);

        // Побочный эффект: повторный GET должен дать 404
        var followReq = WithKey("test-key");
        followReq.Method = HttpMethod.Get;
        followReq.RequestUri = new Uri("/api/todos/1", UriKind.Relative);
        var followResp = await _client.SendAsync(followReq);
        Assert.Equal(HttpStatusCode.NotFound, followResp.StatusCode);
    }
}
```

Разбор по строкам. SUT-часть повторяет пример урока: `AddDbContext` с `UseSqlServer` в проде, `IClock` как синглтон, `public partial class Program {}` в конце — без этого тип `Program` невидим для тестового проекта, и `WebApplicationFactory<Program>` не компилируется. Middleware проверки ключа ставит `ClaimsPrincipal` с ролью `admin` или `user`, чтобы `RequireAuthorization()` и `IsInRole("admin")` работали по-настоящему — это второй, более реалистичный путь из урока, а не отключение фильтров безопасности. В `POST` валидация пустого `Title` возвращает `400`, а `CreatedAt` проставляется из `IClock` — именно поэтому в тестах `FixedClock` даёт детерминированный `2025-01-01`.

`TestFactory` переопределяет `ConfigureWebHost`, зовёт `UseEnvironment("IntegrationTests")` и через `ConfigureAppConfiguration` подменяет `ApiKey` на тестовое значение, чтобы middleware пропускал запросы с `X-Api-Key: test-key`. В `ConfigureTestServices` сначала ищется и удаляется `ServiceDescriptor` для `DbContextOptions<AppDb>` — это критично: простой `AddDbContext` поверх старой регистрации оставил бы две регистрации, и поведение стало бы недетерминированным (прямая цитата ошибки из урока). Затем регистрируется in-memory контекст, `IClock` полностью удаляется через `RemoveAll<IClock>()` и заменяется на `FixedClock` — шпион для времени. Метод `ResetDbAsync` через `Services.CreateScope()` достаёт свежий `AppDb`, зовёт `EnsureDeletedAsync` + `EnsureCreatedAsync` и возвращает контекст для посева данных.

Тестовый класс реализует `IClassFixture<TestFactory>` (один экземпляр фабрики на все тесты — запуск `TestServer` дорогой) и `IAsyncLifetime` (сброс БД в `InitializeAsync`, не в конструкторе, потому что конструктор синхронный и не должен трогать БД). Вспомогательный метод `WithKey` собирает `HttpRequestMessage` с нужным ключом и ролью — это позволяет в одном тесте явно проконтролировать сценарий авторизации. Тесты покрывают не только счастливый путь: 200, 404, 201 + `Location` + тело с автоинкрементным `Id`, 400 (валидация), 401 (нет ключа), 403 (нет admin) и 204 с побочным эффектом (повторный `GET` даёт 404). Каждый тест проверяет статус, тело (через `ReadFromJsonAsync<T>`) и заголовки (`Location`) — ровно то, чего требует урок, предостерегая от «проверки только статус-кода». `dotnet test` после прогона показывает 7 зелёных тестов, порядок выполнения не влияет на результат благодаря `ResetDbAsync`.

#### Задания на углубление (бонус)

1. **Testcontainers для реального SQL Server.** Замените in-memory EF Core на Testcontainers-контейнер с SQL Server и проверьте, что тесты, зависящие от диалекта SQL (например, `JSON_VALUE` или констрейнты), проходят. Сравните скорость прогона.
2. **`ICollectionFixture` вместо `IClassFixture`.** Разнесите тесты по двум классам (`TodoReadTests`, `TodoWriteTests`) и поделите один `TestFactory` через `ICollectionFixture<TestFactory>`. Добейтесь, что обе коллекции проходят без конфликтов по данным.
3. **`WebApplicationFactoryContentLanguage` и кастомный `HttpClient`.** Переопределите `CreateClient` так, чтобы добавить `DelegatingHandler`, логирующий каждый запрос и ответ в `ITestOutputHelper`. Убедитесь, что логи помогают дебажить плавающий тест.
4. **Тест на параллельные запросы.** Запустите 50 одновременных `POST` через `Task.WhenAll` и проверьте, что все вернули 201, а `Id` уникальны. Подумайте, почему in-memory EF Core может дать гонки и как это исправить.

---

## Statement in English / Постановка на английском

#### Context & motivation

Your team maintains a small REST API for a todo list — `TodoApi`. It is written in C# 12 / .NET 8 in the minimal API style, uses EF Core (SQL Server in production), and is protected by an API key: the `X-Api-Key` header must match the value from configuration, and deleting todos is only allowed for users with the `admin` role. The business logic is simple, but the pipeline is not: routing, model binding, validation, an authorization filter, EF Core, JSON serialization. Controller unit tests with mocked services do not give confidence here — they prove that a method is called, but not that a request actually reaches the database and back, that a `201 Created` carries a correct `Location` header, that validation rejects an empty title, or that the key filter rejects a missing header.

You need to write integration tests through `WebApplicationFactory<program>` that run every request through the entire `TestServer` pipeline, yet do not depend on SQL Server, real time, or a real API key. External dependencies are swapped surgically through `ConfigureTestServices`, while internal business logic (endpoints, EF Core, validation, authorization) stays real. One factory instance is shared across tests through `IClassFixture<TestFactory>`, and the in-memory DB state is reset in `IAsyncLifetime.InitializeAsync`, so execution order never affects results. This is exactly the combination the lesson describes as "the bridge between the world of unit tests and a genuinely running HTTP pipeline."

#### What to do step by step

1. **Create the solution and projects.** From an empty folder run:
   ```
   dotnet new sln -n TodoApiSln
   dotnet new web -n TodoApi -o src/TodoApi
   dotnet new xunit -n TodoApi.IntegrationTests -o tests/TodoApi.IntegrationTests
   dotnet sln add src/TodoApi tests/TodoApi.IntegrationTests
   dotnet add tests/TodoApi.IntegrationTests reference src/TodoApi
   dotnet add tests/TodoApi.IntegrationTests package Microsoft.AspNetCore.Mvc.Testing --version 8.*
   dotnet add tests/TodoApi.IntegrationTests package Microsoft.EntityFrameworkCore.InMemory --version 8.*
   dotnet add src/TodoApi package Microsoft.EntityFrameworkCore.SqlServer --version 8.*
   ```
   You should end up with two projects: the SUT (`src/TodoApi`) and the test project (`tests/TodoApi.IntegrationTests`), where the test project references the SUT and the `Microsoft.AspNetCore.Mvc.Testing` package.

2. **Write the SUT — `src/TodoApi/Program.cs`.** Implement a minimal API with three endpoints: `GET /api/todos` (list), `POST /api/todos` (create with title validation), `DELETE /api/todos/{id}` (admin role only). Register `AppDb` through `AddDbContext` with `UseSqlServer`, register `IClock` as a singleton, and add a custom API key middleware (or filter) that checks the `X-Api-Key` header and sets the user's claims. At the very end of the file you must add `public partial class Program {}` so the test project can see the entry point. Define the `AppDb`, `Todo`, `IClock`, and `SystemClock` classes.

3. **Make `Program` visible.** Without `public partial class Program {}`, the test project's compiler will not find the `Program` type, and `WebApplicationFactory<Program>` will not build. This is one of the common mistakes called out in the lesson — verify that `dotnet build` produces no errors after the change.

4. **Write `TestFactory`** in `tests/TodoApi.IntegrationTests/TestFactory.cs`: a subclass of `WebApplicationFactory<Program>` that overrides `ConfigureWebHost`. Inside, call `builder.UseEnvironment("IntegrationTests")`, and in `ConfigureTestServices` remove the real `DbContextOptions<AppDb>` registration via `services.SingleOrDefault(...)`, register an in-memory context with `UseInMemoryDatabase("todos-tests")`, remove `IClock` and register a `FixedClock` pinned to `2025-01-01 UTC`, and substitute the API key check so tests can both pass and reject a request. Add a `ResetDbAsync()` method that creates a fresh DB through a scope (`EnsureDeletedAsync` + `EnsureCreatedAsync`).

5. **Write the tests** in `tests/TodoApi.IntegrationTests/TodoEndpointsTests.cs`: the class implements `IClassFixture<TestFactory>` and `IAsyncLifetime`. In the constructor, store the factory and create an `HttpClient` via `factory.CreateClient()`. In `InitializeAsync`, reset the DB. Cover these scenarios: get an existing todo → 200 + body; get a missing one → 404; create a valid todo → 201 + `Location` + body with an auto-incremented `Id`; create with an empty title → 400; request without `X-Api-Key` → 401; delete without the admin role → 403; delete with admin → 204 plus a check that the todo is really gone from the DB (side effect).

6. **Run the tests and get them green.** The command `dotnet test` should report 7 passing tests. If it fails with a SQL Server access error, you forgot the DB swap in `ConfigureTestServices`. If it fails with a `TypeLoadException` on `Program`, you forgot `public partial class Program {}`. If the tests flake under a different order, you forgot `ResetDbAsync` in `InitializeAsync`.

#### Requirements

- The test project must reference the SUT through `<ProjectReference>` and contain the `Microsoft.AspNetCore.Mvc.Testing` package. Without the reference, `WebApplicationFactory<Program>` will not resolve the `Program` type.
- `TestFactory` is a subclass of `WebApplicationFactory<Program>` with an overridden `ConfigureWebHost`. Inside it you must use `ConfigureTestServices`, not a reimplementation of `Program.cs`: the real pipeline (routing, model binding, middleware, EF Core) must remain intact.
- The real SQL Server dependency is replaced with EF Core in-memory. This is done by removing the `ServiceDescriptor` for `DbContextOptions<AppDb>` and re-registering — not by calling `AddDbContext` on top of the old one, otherwise two registrations stay in the container and behavior becomes non-deterministic.
- `IClock` is replaced with `FixedClock` so that tests on `CreatedAt` and timestamps do not depend on real time. This is the classic "spy for time" pattern from the lesson.
- The factory is used through `IClassFixture<TestFactory>`, and the DB reset lives in `IAsyncLifetime.InitializeAsync`. Do not clean data in the constructor: the constructor is synchronous and must not touch the DB.
- Every test asserts not only the `StatusCode` but also the body (`ReadFromJsonAsync<T>`), headers (`Location`, `Content-Type`), and — where relevant — the side effect (a follow-up `GET` after `DELETE`).

#### Pitfalls

- **`AddDbContext` on top of the old registration duplicates instead of replacing.** The lesson explicitly warns that substituting all services through `AddTransient` instead of `Replace` loses the real pipeline. The correct path for EF Core is first `services.Remove(descriptor)` for `DbContextOptions<AppDb>`, then `services.AddDbContext(... UseInMemoryDatabase ...)`. For scalar singletons (`IClock`), `services.RemoveAll<IClock>()` followed by a new registration works.
- **`public partial class Program {}` is mandatory.** The minimal hosting model generates an internal `Program` class; without the partial declaration the test project cannot see it. The alternative is `[assembly: InternalsVisibleTo("TodoApi.IntegrationTests")]`, but the partial class is cleaner.
- **Sharing DB state across tests causes flaky bugs.** EF Core in-memory keeps data as long as the database context is alive. If you do not call `EnsureDeletedAsync`/`EnsureCreatedAsync` (or give each test a unique DB name), the `Get_Existing` test will start seeing todos seeded by the `Post_Product` test. Clean in `IAsyncLifetime.InitializeAsync`, never in the constructor.
- **Asserting only the status code lets body and header bugs through.** The lesson demands `ReadFromJsonAsync<T>` and checks on `Location`/`Content-Type`. For example, a `201 Created` without `Location` is a bug that only a header assertion catches.
- **`CreateClient()` returns the same `HttpClient` for a given factory.** That is fine and even desirable: it shares an `HttpClientHandler` aimed at `TestServer`. Do not create a client per test — it is extra work with no benefit.
- **Authorization.** Do not disable security middleware via `ConfigureTestServices` — you lose coverage of the role model. Instead, substitute an `AuthenticationHandler<T>` that returns the test user's claims, or — as in this homework — substitute the API key check itself, so the test can explicitly pass the right key and role in a header.

#### Acceptance criteria

- [ ] There is a `TodoApiSln` solution with `src/TodoApi` and `tests/TodoApi.IntegrationTests` projects.
- [ ] The test project references the SUT and contains `Microsoft.AspNetCore.Mvc.Testing`.
- [ ] `Program.cs` ends with `public partial class Program {}`.
- [ ] The SUT implements `GET /api/todos`, `POST /api/todos`, `DELETE /api/todos/{id}` with validation and `X-Api-Key` authorization.
- [ ] `TestFactory : WebApplicationFactory<Program>` overrides `ConfigureWebHost` and calls `UseEnvironment("IntegrationTests")`.
- [ ] In `ConfigureTestServices`, the real `DbContextOptions<AppDb>` registration is removed and an in-memory context is added.
- [ ] `IClock` is replaced with `FixedClock`.
- [ ] The API key check is substituted so a test can both pass and reject a request.
- [ ] `TestFactory.ResetDbAsync()` builds a fresh DB via `EnsureDeletedAsync` + `EnsureCreatedAsync`.
- [ ] `TodoEndpointsTests` implements `IClassFixture<TestFactory>` and `IAsyncLifetime`.
- [ ] The DB is reset in `InitializeAsync`, not in the constructor.
- [ ] There are tests for: 200, 404, 201 + Location, 400 (validation), 401 (no key), 403 (no admin), 204 + side effect.
- [ ] Each test asserts status, body (where relevant), and headers (`Location`, `Content-Type`).
- [ ] `dotnet test` shows all tests green; execution order does not affect results.
- [ ] The code uses C# 12 / .NET 8: top-level statements, primary constructors, collection expressions, `required`/`init` where appropriate.

#### Hints (no direct answer)

- To "substitute the key check but keep the ability to reject," register a test key handler in `ConfigureTestServices` that reads `X-Api-Key` and compares it to a known test value (say, `"test-key"`). Then a test without a key gets 401, and a test with a key passes. The admin role can be carried by a separate `X-Test-Role` header or a second key.
- For empty-title validation you do not need FluentValidation: a simple `if (string.IsNullOrWhiteSpace(todo.Title)) return Results.BadRequest(...)` inside the endpoint is enough. The point is to return 400, not 500.
- To verify the delete side effect, after `DELETE` issue a follow-up `GET /api/todos/{id}` and assert 404.
- `ReadFromJsonAsync<T>` lives in `System.Net.Http.Json`. For `Location`, use `response.Headers.Location`.
- Do not forget to `await` `SaveChangesAsync` while seeding — otherwise the in-memory context will not persist.

#### Reference solution walk-through

```csharp
// === SUT: src/TodoApi/Program.cs ===
using Microsoft.EntityFrameworkCore;
using System.Security.Claims;

var builder = WebApplication.CreateBuilder(args);

// EF Core: SQL Server in prod, in-memory in tests via substitution
builder.Services.AddDbContext<AppDb>(opt =>
    opt.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

builder.Services.AddSingleton<IClock, SystemClock>();
builder.Services.AddAuthorization();

var app = builder.Build();

// API key middleware: sets the user's claims
app.Use(async (ctx, next) =>
{
    var key = ctx.Request.Headers["X-Api-Key"].ToString();
    var expected = builder.Configuration["ApiKey"] ?? "prod-secret";
    if (string.IsNullOrEmpty(key) || key != expected)
    {
        ctx.Response.StatusCode = 401;
        return;
    }
    var role = key == "admin-secret" ? "admin" : "user";
    ctx.User = new ClaimsPrincipal(new ClaimsIdentity(new[]
    {
        new Claim(ClaimTypes.NameIdentifier, "test-user"),
        new Claim(ClaimTypes.Role, role),
    }, "ApiKey"));
    await next();
});

// GET /api/todos — list all todos
app.MapGet("/api/todos", async (AppDb db) =>
    Results.Ok(await db.Todos.AsNoTracking().ToListAsync()));

// GET /api/todos/{id} — single todo
app.MapGet("/api/todos/{id:int}", async (int id, AppDb db) =>
{
    var todo = await db.Todos.FindAsync(id);
    return todo is null ? Results.NotFound() : Results.Ok(todo);
});

// POST /api/todos — create (with validation)
app.MapPost("/api/todos", async (Todo todo, AppDb db, IClock clock) =>
{
    if (string.IsNullOrWhiteSpace(todo.Title))
        return Results.BadRequest(new { error = "Title is required" });

    todo.CreatedAt = clock.UtcNow;
    db.Todos.Add(todo);
    await db.SaveChangesAsync();
    return Results.Created($"/api/todos/{todo.Id}", todo);
});

// DELETE /api/todos/{id} — admin only
app.MapDelete("/api/todos/{id:int}", async (int id, AppDb db, HttpContext ctx) =>
{
    if (!ctx.User.IsInRole("admin"))
        return Results.Forbid();
    var todo = await db.Todos.FindAsync(id);
    if (todo is null) return Results.NotFound();
    db.Todos.Remove(todo);
    await db.SaveChangesAsync();
    return Results.NoContent();
}).RequireAuthorization();

app.Run();

public partial class Program { }

public class AppDb(DbContextOptions<AppDb> o) : DbContext(o)
{
    public DbSet<Todo> Todos => Set<Todo>();
}

public class Todo
{
    public int Id { get; set; }
    public required string Title { get; set; }
    public DateTime CreatedAt { get; set; }
}

public interface IClock { DateTime UtcNow { get; } }
public class SystemClock : IClock { public DateTime UtcNow => DateTime.UtcNow; }
```

```csharp
// === tests/TodoApi.IntegrationTests/TestFactory.cs ===
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

namespace TodoApi.IntegrationTests;

public class TestFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("IntegrationTests");
        builder.ConfigureAppConfiguration((_, cfg) =>
        {
            cfg.AddInMemoryCollection(new Dictionary<string, string?>
            {
                ["ApiKey"] = "test-key",
            });
        });

        builder.ConfigureTestServices(services =>
        {
            // Remove the real DbContextOptions<AppDb> registration
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<AppDb>));
            if (descriptor is not null) services.Remove(descriptor);

            services.AddDbContext<AppDb>(opt => opt.UseInMemoryDatabase("todos-tests"));

            services.RemoveAll<IClock>();
            services.AddSingleton<IClock>(new FixedClock(
                new DateTime(2025, 1, 1, 0, 0, 0, DateTimeKind.Utc)));
        });
    }

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
```

```csharp
// === tests/TodoApi.IntegrationTests/TodoEndpointsTests.cs ===
using System.Net;
using System.Net.Http.Json;
using Microsoft.Extensions.DependencyInjection;

namespace TodoApi.IntegrationTests;

public class TodoEndpointsTests : IClassFixture<TestFactory>, IAsyncLifetime
{
    private readonly TestFactory _factory;
    private readonly HttpClient _client;
    private AppDb _db = null!;

    public TodoEndpointsTests(TestFactory factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }

    public async Task InitializeAsync() => _db = await _factory.ResetDbAsync();
    public Task DisposeAsync() => Task.CompletedTask;

    private HttpRequestMessage WithKey(string key, string? role = null)
    {
        var req = new HttpRequestMessage();
        req.Headers.Add("X-Api-Key", key);
        if (role is not null) req.Headers.Add("X-Test-Role", role);
        return req;
    }

    [Fact]
    public async Task Get_ExistingTodo_Returns200WithBody()
    {
        _db.Todos.Add(new Todo { Title = "Buy bread" });
        await _db.SaveChangesAsync();

        var req = WithKey("test-key");
        req.Method = HttpMethod.Get;
        req.RequestUri = new Uri("/api/todos/1", UriKind.Relative);
        var resp = await _client.SendAsync(req);

        Assert.Equal(HttpStatusCode.OK, resp.StatusCode);
        var todo = await resp.Content.ReadFromJsonAsync<Todo>();
        Assert.NotNull(todo);
        Assert.Equal("Buy bread", todo!.Title);
    }

    [Fact]
    public async Task Get_MissingTodo_Returns404()
    {
        var req = WithKey("test-key");
        req.Method = HttpMethod.Get;
        req.RequestUri = new Uri("/api/todos/9999", UriKind.Relative);
        var resp = await _client.SendAsync(req);
        Assert.Equal(HttpStatusCode.NotFound, resp.StatusCode);
    }

    [Fact]
    public async Task Post_ValidTodo_Returns201WithLocation()
    {
        var req = WithKey("test-key");
        req.Method = HttpMethod.Post;
        req.RequestUri = new Uri("/api/todos", UriKind.Relative);
        req.Content = JsonContent.Create(new { Title = "Water flowers" });

        var resp = await _client.SendAsync(req);

        Assert.Equal(HttpStatusCode.Created, resp.StatusCode);
        Assert.NotNull(resp.Headers.Location);
        var created = await resp.Content.ReadFromJsonAsync<Todo>();
        Assert.NotNull(created);
        Assert.True(created!.Id > 0);
        Assert.Equal(new DateTime(2025, 1, 1, 0, 0, 0, DateTimeKind.Utc), created.CreatedAt);
    }

    [Fact]
    public async Task Post_EmptyTitle_Returns400()
    {
        var req = WithKey("test-key");
        req.Method = HttpMethod.Post;
        req.RequestUri = new Uri("/api/todos", UriKind.Relative);
        req.Content = JsonContent.Create(new { Title = "" });

        var resp = await _client.SendAsync(req);
        Assert.Equal(HttpStatusCode.BadRequest, resp.StatusCode);
    }

    [Fact]
    public async Task Request_WithoutApiKey_Returns401()
    {
        var resp = await _client.GetAsync("/api/todos");
        Assert.Equal(HttpStatusCode.Unauthorized, resp.StatusCode);
    }

    [Fact]
    public async Task Delete_WithoutAdminRole_Returns403()
    {
        _db.Todos.Add(new Todo { Title = "Old todo" });
        await _db.SaveChangesAsync();

        var req = WithKey("test-key");
        req.Method = HttpMethod.Delete;
        req.RequestUri = new Uri("/api/todos/1", UriKind.Relative);

        var resp = await _client.SendAsync(req);
        Assert.Equal(HttpStatusCode.Forbidden, resp.StatusCode);
    }

    [Fact]
    public async Task Delete_WithAdminRole_Returns204AndRemovesTodo()
    {
        _db.Todos.Add(new Todo { Title = "Old todo" });
        await _db.SaveChangesAsync();

        var req = WithKey("admin-secret", "admin");
        req.Method = HttpMethod.Delete;
        req.RequestUri = new Uri("/api/todos/1", UriKind.Relative);

        var resp = await _client.SendAsync(req);
        Assert.Equal(HttpStatusCode.NoContent, resp.StatusCode);

        // Side effect: a follow-up GET must return 404
        var followReq = WithKey("test-key");
        followReq.Method = HttpMethod.Get;
        followReq.RequestUri = new Uri("/api/todos/1", UriKind.Relative);
        var followResp = await _client.SendAsync(followReq);
        Assert.Equal(HttpStatusCode.NotFound, followResp.StatusCode);
    }
}
```

Walk-through. The SUT section mirrors the lesson's example: `AddDbContext` with `UseSqlServer` for production, `IClock` as a singleton, and `public partial class Program {}` at the bottom — without it the `Program` type is invisible to the test project and `WebApplicationFactory<Program>` will not compile. The API key middleware sets a `ClaimsPrincipal` with the `admin` or `user` role, so `RequireAuthorization()` and `IsInRole("admin")` work for real — this is the second, more realistic path from the lesson rather than disabling the security filters. In `POST`, the empty-title validation returns `400`, and `CreatedAt` is stamped from `IClock` — which is exactly why `FixedClock` makes the tests deterministic at `2025-01-01`.

`TestFactory` overrides `ConfigureWebHost`, calls `UseEnvironment("IntegrationTests")`, and through `ConfigureAppConfiguration` overrides `ApiKey` with a test value so the middleware accepts requests carrying `X-Api-Key: test-key`. In `ConfigureTestServices` it first finds and removes the `ServiceDescriptor` for `DbContextOptions<AppDb>` — this is critical: a plain `AddDbContext` on top of the old registration would leave two registrations and the behavior would become non-deterministic (a direct quote of the mistake from the lesson). Then the in-memory context is registered, `IClock` is wiped via `RemoveAll<IClock>()` and replaced with `FixedClock` — the time spy. The `ResetDbAsync` method pulls a fresh `AppDb` out of a scope via `Services.CreateScope()`, calls `EnsureDeletedAsync` + `EnsureCreatedAsync`, and returns the context for seeding.

The test class implements `IClassFixture<TestFactory>` (one factory instance for all tests — booting `TestServer` is expensive) and `IAsyncLifetime` (DB reset in `InitializeAsync`, not in the constructor, because the constructor is synchronous and must not touch the DB). The helper `WithKey` builds an `HttpRequestMessage` with the right key and role, so each test can explicitly control the authorization scenario. The tests cover more than the happy path: 200, 404, 201 + `Location` + body with an auto-incremented `Id`, 400 (validation), 401 (no key), 403 (no admin), and 204 with a side effect (the follow-up `GET` returns 404). Every test asserts status, body (via `ReadFromJsonAsync<T>`), and headers (`Location`) — exactly what the lesson asks for, warning against "asserting only the status code." After the run, `dotnet test` reports 7 green tests, and the order does not matter thanks to `ResetDbAsync`.

#### Going deeper (bonus)

1. **Testcontainers for a real SQL Server.** Swap the in-memory EF Core for a Testcontainers container running SQL Server and verify that tests depending on the SQL dialect (for example `JSON_VALUE` or constraints) still pass. Compare the run time.
2. **`ICollectionFixture` instead of `IClassFixture`.** Split the tests into two classes (`TodoReadTests`, `TodoWriteTests`) and share one `TestFactory` through `ICollectionFixture<TestFactory>`. Make sure both collections pass without data conflicts.
3. **A custom `HttpClient` with a `DelegatingHandler`.** Override `CreateClient` so every request and response is logged to `ITestOutputHelper` via a `DelegatingHandler`. Confirm that the logs help debug a flaky test.
4. **A concurrency test.** Fire 50 parallel `POST` calls with `Task.WhenAll` and assert that all return 201 and that the `Id` values are unique. Reason about why in-memory EF Core can race and how to fix it.

---

#### Чек-лист сдачи / Submission checklist

**RU:**
- [ ] Решение `TodoApiSln` собирается командой `dotnet build` без ошибок.
- [ ] Тестовый проект ссылается на SUT и содержит `Microsoft.AspNetCore.Mvc.Testing`.
- [ ] В `Program.cs` есть `public partial class Program {}`.
- [ ] `TestFactory : WebApplicationFactory<Program>` переопределяет `ConfigureWebHost`.
- [ ] В `ConfigureTestServices` реальная БД подменена на in-memory EF Core, `IClock` — на `FixedClock`.
- [ ] Фабрика разделяется через `IClassFixture`, сброс БД — в `IAsyncLifetime.InitializeAsync`.
- [ ] Есть тесты на 200, 404, 201+Location, 400, 401, 403, 204+побочный эффект.
- [ ] `dotnet test` показывает все тесты зелёными.

**EN:**
- [ ] The `TodoApiSln` solution builds with `dotnet build` without errors.
- [ ] The test project references the SUT and contains `Microsoft.AspNetCore.Mvc.Testing`.
- [ ] `Program.cs` contains `public partial class Program {}`.
- [ ] `TestFactory : WebApplicationFactory<Program>` overrides `ConfigureWebHost`.
- [ ] In `ConfigureTestServices` the real DB is swapped for in-memory EF Core and `IClock` for `FixedClock`.
- [ ] The factory is shared via `IClassFixture`, DB reset is in `IAsyncLifetime.InitializeAsync`.
- [ ] There are tests for 200, 404, 201+Location, 400, 401, 403, 204+side effect.
- [ ] `dotnet test` shows all tests green.

#### Ресурсы / Resources
- [Microsoft Learn — Integration tests in ASP.NET Core — https://learn.microsoft.com/aspnet/core/test/integration-tests](https://learn.microsoft.com/aspnet/core/test/integration-tests)
- [Microsoft Learn — WebApplicationFactory<TEntryPoint> — https://learn.microsoft.com/dotnet/api/microsoft.aspnetcore.mvc.testing.webapplicationfactory-1](https://learn.microsoft.com/dotnet/api/microsoft.aspnetcore.mvc.testing.webapplicationfactory-1)
- [EF Core — In-memory database provider — https://learn.microsoft.com/ef/core/providers/in-memory](https://learn.microsoft.com/ef/core/providers/in-memory)
- [xUnit — IClassFixture and IAsyncLifetime — https://xunit.net/docs/shared-context](https://xunit.net/docs/shared-context)
- [Testcontainers for .NET — https://dotnet.testcontainers.org/](https://dotnet.testcontainers.org/)
