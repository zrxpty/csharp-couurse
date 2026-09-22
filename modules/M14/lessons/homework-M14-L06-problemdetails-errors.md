---
[← К уроку M14-L06](lesson-M14-L06-problemdetails-errors.md) | [⬆ К модулю M14](../README.md) | [Следующее ДЗ →](homework-M14-L07-api-versioning.md)
---

### Домашнее задание M14-L06: ProblemDetails, обработка ошибок API / Homework M14-L06: ProblemDetails, API error handling

**Урок / Lesson:** M14-L06
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться строить консистентную обработку ошибок ASP.NET Core 8 API на основе RFC 7807 ProblemDetails: реализовать иерархию доменных исключений, глобальный `IExceptionHandler`, использующий `ProblemDetailsService`, валидацию через `ValidationProblemDetails` со словарём `errors`, и единый формат `application/problem+json` для всех ошибок 4xx/5xx с `traceId` в расширениях. (EN) Learn to build consistent ASP.NET Core 8 API error handling on top of RFC 7807 ProblemDetails: implement a domain-exception hierarchy, a global `IExceptionHandler` backed by `ProblemDetailsService`, validation via `ValidationProblemDetails` with an `errors` dictionary, and a single `application/problem+json` shape for every 4xx/5xx error with a `traceId` in extensions.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит ProblemDetails как стандартный бланк ошибки и показывает, что контроллеры покрывают лишь половину ошибок — остальное перехватывает `IExceptionHandler`. ДЗ закрепляет обе части: вы построите доменные исключения со своими статусами, зарегистрируете обработчик через `ProblemDetailsService` и убедитесь, что валидация, бизнес-конфликты и необработанные сбои выглядят для клиента одинаково. (EN) The lesson introduces ProblemDetails as a standard error form and shows that controllers cover only half the errors — the rest is caught by an `IExceptionHandler`. This homework cements both halves: you will build domain exceptions with their own statuses, register a handler through `ProblemDetailsService`, and verify that validation, business conflicts, and unhandled failures all look identical to the client.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы присоединяетесь к команде, которая развивает внутренний складской API «Warehouse» на ASP.NET Core 8. Сейчас API ведёт себя хаотично: при отсутствии товара клиент получает пустой 404 без тела, при конфликте складских остатков — строку `"Conflict"` с `text/plain`, а при необработанном исключении — голый 500 со стеком вызова в JSON. Фронтенд-команда устала писать отдельный парсер под каждый эндпоинт и грозится переключиться на другого поставщика. Ваша задача — привести все ответы об ошибках к единому стандарту RFC 7807 (`application/problem+json`), чтобы любой клиент, будь то браузер, мобильное приложение или другой микросервис, мог одинаково прочитать причину сбоя.

Консистентность — это не косметика, а контракт. Когда все ошибки имеют поля `type`, `title`, `status`, `detail`, `instance` и расширение `traceId`, клиентский код сводится к одной функции-парсеру, а поддержка получает возможность связать жалобу пользователя с записью в логе по `traceId`. Особенно важна валидация: клиенту формы нужно знать не абстрактное «что-то не так», а конкретные поля — `quantity`, `sku`, `email` — с массивом сообщений по каждому полю. Именно для этого существует производный класс `ValidationProblemDetails` со словарём `errors`.

В уроке показан рекомендуемый путь .NET 8: не писать свой middleware с `app.Map`, а реализовать `IExceptionHandler` и делегировать форматирование `ProblemDetailsService`. Это даёт единый код для ручного `return Problem()`, ошибок валидации и исключений. ДЗ проведёт вас через все этапы: от доменных исключений до curl-тестов, проверяющих каждый статус-код и каждое поле ProblemDetails.

#### Что нужно сделать (пошагово)
1. Создайте проект: `dotnet new webapi -n Warehouse -o Warehouse --use-controllers --no-https`, затем `cd Warehouse`. Убедитесь, что в `Warehouse.csproj` стоит `<TargetFramework>net8.0</TargetFramework>` и `<Nullable>enable</Nullable>`. Откройте `Program.cs` — это будет точка сборки DI и конвейера.

2. Определите иерархию доменных исключений в файле `Exceptions/AppExceptions.cs`. Базовый абстрактный класс `AppException(int status, string title, string type)` хранит статус, заголовок и URI типа ошибки. От него унаследуйте `NotFoundException` (404, тип `https://warehouse.example/errors/not-found`), `ConflictException` (409, `.../conflict`), `BusinessRuleException` (422, `.../business-rule`). Каждое исключение получает `detail` через первичный конструктор. Цель — чтобы перевод исключения в HTTP-статус был предсказуемым и сосредоточенным в одном месте.

3. Реализуйте глобальный обработчик `ProblemDetailsExceptionHandler` в `Infrastructure/ProblemDetailsExceptionHandler.cs`. Класс реализует `IExceptionHandler` и принимает через DI `ILogger<ProblemDetailsExceptionHandler>` и `IProblemDetailsService`. В `TryHandleAsync` логируйте исключение с полным стеком через `logger.LogError(exception, ...)`, затем через pattern matching переведите его в кортеж `(status, title, type, detail)`: `AppException` отдаёт свои поля, `ArgumentException` → 400, всё остальное → 500 с нейтральным `detail`. Никогда не отдавайте `exception.StackTrace` или внутренние сообщения наружу — только `detail`. Заполните `ProblemDetails` (Status, Title, Type, Detail, Instance = `Request.Path`) и вызовите `problemDetailsService.TryWriteAsync(new ProblemDetailsContext { ... })`.

4. Опишите домен в `Models/Warehouse.cs`: `record CreateOrderRequest(Guid ProductId, int Quantity, string CustomerEmail)` и потокобезопасный in-memory репозиторий на `ConcurrentDictionary` с предзагруженным товаром. Метод `OrderService.Place` должен бросать `NotFoundException` при отсутствии продукта, `ConflictException` при `Quantity` больше остатка, `BusinessRuleException` при невалидном email (без `@`).

5. Контроллер `OrdersController` (`[ApiController]`, `[Route("api/[controller]")]`) имеет `POST /api/orders`, который принимает `[FromBody] CreateOrderRequest`, дёргает `OrderService.Place` и возвращает `CreatedAtAction`. Добавьте `GET /api/orders/boom`, который бросает `InvalidOperationException` — для проверки 500-пути. Валидацию модели (атрибуты `[Required]`, `[Range(1,1000)]`) НЕ включайте — мы хотим видеть, как `ValidationProblemDetails` формируется фреймворком автоматически через `AddProblemDetails()` + `[ApiController]`.

6. Соберите `Program.cs`: `AddControllers()`, `AddProblemDetails(o => o.CustomizeProblemDetails = ctx => ctx.ProblemDetails.Extensions["traceId"] = ctx.HttpContext.TraceIdentifier)`, `AddExceptionHandler<ProblemDetailsExceptionHandler>()`, `AddSingleton<IProblemDetailsService, ProblemDetailsService>()`. В конвейере: `UseExceptionHandler()`, затем `MapControllers()`. Порядок критичен — `UseExceptionHandler` должен стоять раньше `MapControllers`, но после `UseRouting`.

7. Запустите: `dotnet run`. В другом термиле прогоните curl-тесты и зафиксируйте вывод:
   - `curl -i -X POST http://localhost:5000/api/orders -H "Content-Type: application/json" -d "{\"ProductId\":\"00000000-0000-0000-0000-000000000000\",\"Quantity\":1,\"CustomerEmail\":\"a@b.com\"}"` → 404 ProblemDetails с `type=.../not-found`.
   - `curl -i -X POST ... -d "{\"ProductId\":\"<валидный-guid>\",\"Quantity\":99999,\"CustomerEmail\":\"a@b.com\"}"` → 409, `detail` про остаток.
   - `... -d "{\"ProductId\":\"<guid>\",\"Quantity\":1,\"CustomerEmail\":\"noemail\"}"` → 422.
   - `curl -i http://localhost:5000/api/orders/boom` → 500, нейтральный `detail`, без стека.
   - Проверьте, что во всех ответах есть заголовок `Content-Type: application/problem+json` и поле `traceId` в `extensions`.

8. Сравните выводы между собой: структура полей должна быть идентичной, меняются только `status`, `title`, `type`, `detail`. Это и есть консистентность.

#### Требования к решению
- Проект на .NET 8, C# 12, с top-level statements в `Program.cs` и nullable-контекстом.
- Все ответы 4xx и 5xx возвращают `Content-Type: application/problem+json` и тело в формате ProblemDetails — никаких голых статус-кодов без тела и никаких `text/plain`.
- Иерархия `AppException` с минимум тремя наследниками, каждый со своим HTTP-статусом и URI типа ошибки.
- Глобальный обработчик реализован именно как `IExceptionHandler` (не как `app.Use(async (ctx, next) => ...)`), использует `IProblemDetailsService.TryWriteAsync` для форматирования.
- Стек вызова и внутренние сообщения исключений НЕ попадают в ответ клиенту; они только логируются на сервере с уровнем `Error`.
- Каждая ошибка содержит `traceId` в `extensions`, равный `HttpContext.TraceIdentifier`.
- Валидация модели (например, невалидный JSON или нарушение `[ApiController]`-аннотаций) автоматически оборачивается в `ValidationProblemDetails` с полем `errors` — без ручного кода.
- Ручные бизнес-проверки в сервисе идут через исключения, а не через `if (...) return BadRequest(...)` в контроллере — контроллер остаётся тонким.
- Код компилируется без предупреждений и проходит `dotnet build`. curl-тесты дают ожидаемые статусы и тела.

#### Тонкости и подводные камни
- **Порядок middleware**: `UseExceptionHandler()` должен стоять в конвейере раньше `MapControllers()`, иначе исключения контроллеров не будут перехвачены. Но он не должен стоять раньше `UseRouting`, если вы рассчитываете на `Request.Path` в `Instance` — маршрут должен быть уже сопоставлен.
- **`AddProblemDetails()` одного вызова достаточно** — он настраивает `ProblemDetailsService` и кастомизацию. Не регистрируйте `ProblemDetailsService` вручную как синглтон, если уже вызвали `AddProblemDetails()` — это создаст две разные инстанции; урок регистрирует `IProblemDetailsService` явно, но в реальных проектах достаточно `AddProblemDetails()`, и вы должны понимать разницу.
- **Не отдавайте стек наружу**: соблазн положить `exception.StackTrace` в `detail` велик при отладке, но это дыра в безопасности. Стек — только в лог, клиенту — нейтральный текст.
- **`ValidationProblemDetails` формируется фреймворком** автоматически, когда включён `AddProblemDetails()` и контроллер помечен `[ApiController]`. Не пытайтесь вручную строить словарь `errors` для ошибок атрибутов — это дублирует работу и ломает консистентность.
- **Pattern matching в обработчике**: используйте `switch` выражение над `exception`, как в уроке. Это централизует перевод и делает добавление новых доменных исключений тривиальным — достаточно добавить новую ветку.
- **`Instance = httpContext.Request.Path`** даёт клиенту понять, какой эндпоинт упал. Не кладите туда весь URL с query-string, если он содержит чувствительные данные.
- **`TryWriteAsync` возвращает `bool`**: если вернётся `false`, ответ не записан — стоит вернуть `false` из `TryHandleAsync`, чтобы следующий обработчик в цепочке попробовал. В простом сценарии можно возвращать `true`.
- **Логируйте всегда**, даже для 4xx бизнес-исключений — с уровнем `Warning` для бизнес-ошибок и `Error` для 500. Это помогает корреляции через `traceId`.
- **Не используйте `BadRequest("text")`** для валидации — это отдаёт строку, а не ProblemDetails, и ломает единообразие.

#### Критерии приёмки
- [ ] Проект `Warehouse` собирается командой `dotnet build` без ошибок и предупреждений.
- [ ] В `Program.cs` вызван `AddProblemDetails(...)` и `AddExceptionHandler<ProblemDetailsExceptionHandler>()`.
- [ ] В конвейере вызван `UseExceptionHandler()` раньше `MapControllers()`.
- [ ] Базовый класс `AppException` хранит `Status`, `Title`, `Type` через первичный конструктор.
- [ ] Есть минимум три наследника: `NotFoundException` (404), `ConflictException` (409), `BusinessRuleException` (422).
- [ ] `ProblemDetailsExceptionHandler` реализует `IExceptionHandler` и использует `IProblemDetailsService.TryWriteAsync`.
- [ ] Стек-трейс и внутренние сообщения НЕ появляются в теле ответа ни в одном из curl-тестов.
- [ ] Все 4xx/5xx ответы содержат заголовок `Content-Type: application/problem+json`.
- [ ] Каждое тело ошибки содержит поля `type`, `title`, `status`, `detail`, `instance`.
- [ ] В `extensions` каждой ошибки присутствует `traceId`, совпадающий с `HttpContext.TraceIdentifier`.
- [ ] POST с несуществующим `ProductId` → 404, `type=.../not-found`.
- [ ] POST с `Quantity` больше остатка → 409, `type=.../conflict`.
- [ ] POST с невалидным email → 422, `type=.../business-rule`.
- [ ] GET `/api/orders/boom` → 500, нейтральный `detail`, без стека.
- [ ] Ошибка валидации модели (невалидный JSON или пропуск `[Required]`) → 400 с `ValidationProblemDetails` и полем `errors`.

#### Подсказки (без прямого ответа)
- Подумайте, почему `IExceptionHandler` лучше, чем inline middleware: где живёт логика, как тестируется, как цепочка обработчиков расширяется.
- Вспомните аналогию урока про «стандартный бланк МФЦ» — какие поля у бланка всегда одинаковые, какие меняются от случая к случаю.
- Для перевода исключения в кортеж используйте `switch`-выражение с `pattern matching` — это покажет читателю, что статус-код определяется типом исключения, а не `if/else` лестницей.
- `traceId` — это мост между клиентом и логом: проверьте, что после запроса вы можете найти запись в логе по этому идентификатору.
- Если `Content-Type` упорно получается `application/json` вместо `application/problem+json`, проверьте, что вызвали `AddProblemDetails()` и что ответ идёт через `ProblemDetailsService`, а не через ручную сериализацию.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — Эталонное решение ДЗ M14-L06
// Warehouse: доменные исключения, IExceptionHandler, ProblemDetailsService, ValidationProblemDetails
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Mvc;
using System.Collections.Concurrent;

// --- Доменные исключения / Domain exceptions ---
// Базовый класс хранит статус, заголовок и тип URI / Base class holds status, title and type URI
public abstract class AppException(int status, string title, string type) : Exception(title)
{
    public int Status { get; } = status;
    public string Title { get; } = title;
    public string Type { get; } = type;
}

public sealed class NotFoundException(string detail)
    : AppException(404, "Resource not found", "https://warehouse.example/errors/not-found")
{
    public string Detail { get; } = detail;
}

public sealed class ConflictException(string detail)
    : AppException(409, "Conflict", "https://warehouse.example/errors/conflict")
{
    public string Detail { get; } = detail;
}

public sealed class BusinessRuleException(string detail)
    : AppException(422, "Business rule violation", "https://warehouse.example/errors/business-rule")
{
    public string Detail { get; } = detail;
}

// --- Глобальный обработчик / Global handler (.NET 8 IExceptionHandler) ---
public sealed class ProblemDetailsExceptionHandler(
    ILogger<ProblemDetailsExceptionHandler> logger,
    IProblemDetailsService problemDetailsService) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext, Exception exception, CancellationToken cancellationToken)
    {
        // Логируем всегда: бизнес — Warning, остальное — Error / Always log: business -> Warning, else Error
        if (exception is AppException)
            logger.LogWarning(exception, "Business exception: {Message}", exception.Message);
        else
            logger.LogError(exception, "Unhandled exception: {Message}", exception.Message);

        // Pattern matching переводит тип исключения в кортеж / Pattern matching maps exception to tuple
        var (status, title, type, detail) = exception switch
        {
            NotFoundException nfe => (nfe.Status, nfe.Title, nfe.Type, nfe.Detail),
            ConflictException ce => (ce.Status, ce.Title, ce.Type, ce.Detail),
            BusinessRuleException bre => (bre.Status, bre.Title, bre.Type, bre.Detail),
            AppException ae => (ae.Status, ae.Title, ae.Type, ae.Message),
            ArgumentException => (400, "Bad Request", "https://warehouse.example/errors/bad-request", exception.Message),
            // Стек НЕ уходит клиенту — только нейтральный detail / Stack never leaks — neutral detail only
            _ => (500, "Internal Server Error", "https://warehouse.example/errors/internal", "An unexpected error occurred")
        };

        httpContext.Response.StatusCode = status;

        var problem = new ProblemDetails
        {
            Status = status,
            Title = title,
            Type = type,
            Detail = detail,
            Instance = httpContext.Request.Path
        };

        // Делегируем форматирование ProblemDetailsService — это даёт traceId и единый вид / Delegate formatting
        return await problemDetailsService.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = httpContext,
            ProblemDetails = problem,
            Exception = exception
        });
    }
}

// --- Домен / Domain ---
public sealed record CreateOrderRequest(Guid ProductId, int Quantity, string CustomerEmail);

public sealed record Product(Guid Id, string Sku, int Stock);

public static class Warehouse
{
    public static readonly ConcurrentDictionary<Guid, Product> Products = new();
}

public sealed class OrderService
{
    public Guid Place(CreateOrderRequest r)
    {
        if (!Warehouse.Products.TryGetValue(r.ProductId, out var product))
            throw new NotFoundException($"Product {r.ProductId} not found");

        if (r.Quantity > product.Stock)
            throw new ConflictException($"Only {product.Stock} items left in stock");

        if (!r.CustomerEmail.Contains('@'))
            throw new BusinessRuleException("CustomerEmail must be a valid email address");

        return Guid.NewGuid();
    }
}

// --- Контроллер / Controller ---
[ApiController]
[Route("api/[controller]")]
public sealed class OrdersController(OrderService orders) : ControllerBase
{
    [HttpPost]
    public IActionResult Create([FromBody] CreateOrderRequest request)
    {
        var id = orders.Place(request);     // исключение улетает в IExceptionHandler / exception flies to handler
        return CreatedAtAction(nameof(GetById), new { id }, new { id });
    }

    [HttpGet("{id:guid}")]
    public IActionResult GetById(Guid id) => Ok(new { id });

    [HttpGet("boom")]   // для проверки 500-пути / to exercise the 500 path
    public IActionResult Boom() => throw new InvalidOperationException("Simulated failure");
}

// --- Program.cs ---
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddSingleton<OrderService>();

// Один вызов настраивает ProblemDetailsService + кастомизацию с traceId / Single call configures both
builder.Services.AddProblemDetails(o =>
    o.CustomizeProblemDetails = ctx =>
        ctx.ProblemDetails.Extensions["traceId"] = ctx.HttpContext.TraceIdentifier);

builder.Services.AddExceptionHandler<ProblemDetailsExceptionHandler>();

var app = builder.Build();

app.UseExceptionHandler();   // ДО MapControllers, но после неявного UseRouting / before MapControllers
app.MapControllers();

// Предзагрузка товара / seed product
Warehouse.Products[Guid.Empty] = new Product(Guid.Empty, "DEMO-1", 5);

app.Run();
```

Разбор по строкам. Базовый `AppException` использует первичный конструктор C# 12 — статус, заголовок и тип задаются один раз и неизменны, что гарантирует предсказуемость перевода (концепция урока: «каждое бизнес-исключение знает свой статус»). Наследники лишь уточняют `detail` под конкретный случай. `ProblemDetailsExceptionHandler` реализует именно `IExceptionHandler` — рекомендованный в .NET 8 путь, а не inline middleware: фреймворк сам подставляет его в конвейер через `UseExceptionHandler()`. Логирование разнесено по уровням: бизнес-ошибки — `Warning`, потому что это ожидаемое поведение пользователя, а не сбой системы; всё остальное — `Error` с полным стеком в лог. Pattern matching в `switch`-выражении централизует перевод типов в кортеж статуса — добавить новое исключение можно одной строкой, не трогая контроллеры. Критично: ветка `_ => (500, ...)` отдаёт нейтральный `detail`, а не `exception.Message` или стек, — это закрывает дыру безопасности из раздела «частые ошибки». `Instance = httpContext.Request.Path` показывает клиенту упавший эндпоинт. Делегирование `problemDetailsService.TryWriteAsync` гарантирует, что кастомизация с `traceId` применится и к исключениям, и к ручному `Problem()`, и к валидации — это и есть единообразие. Контроллер остаётся тонким: он не ловит исключения и не возвращает `BadRequest("text")`, потому что вся ответственность за форматирование переложена на обработчик и `ProblemDetailsService`. `AddProblemDetails()` один вызов решает и форматирование, и `traceId` — отдельная регистрация `ProblemDetailsService` не нужна. Порядок `UseExceptionHandler()` → `MapControllers()` обязателен: иначе исключения контроллеров уходят в дефолтный 500 без тела.

#### Задания на углубление (бонус)
1. Добавьте `ValidationProblemDetails` для ручной бизнес-валидации: создайте метод расширения, который по словарю `IDictionary<string, string[]>` строит `ValidationProblemDetails` и пишет его через `ProblemDetailsService`, чтобы ошибки нескольких полей сразу выглядели как `errors`.
2. Реализуйте второй `IExceptionHandler` для логики rate-limiting, который возвращает 429 с заголовком `Retry-After`, и убедитесь, что порядок регистрации обработчиков влияет на то, кто первый перехватит исключение.
3. Добавьте корреляционный `Correlation-Id` заголовок из запроса в `extensions` (с генерацией, если заголовка нет) и проверьте, что он попадает и в лог, и в ProblemDetails.
4. Покройте обработчик юнит-тестами на `WebApplicationFactory`, проверяя, что каждый тип исключения даёт правильный статус и что стек отсутствует в теле.

---

## Statement in English / Постановка на английском

#### Context & motivation
You are joining a team that maintains an internal warehouse API called «Warehouse» on ASP.NET Core 8. Right now the API behaves chaotically: when a product is missing, the client gets an empty 404 with no body; when stock conflicts arise, it returns the string `"Conflict"` as `text/plain`; and on an unhandled exception it returns a bare 500 with the full stack trace embedded in JSON. The front-end team is tired of writing a separate parser for every endpoint and is threatening to switch vendors. Your job is to bring every error response under the single RFC 7807 standard (`application/problem+json`) so that any client — a browser, a mobile app, or another microservice — can read the reason for a failure the same way.

Consistency is not cosmetics, it is a contract. When every error carries `type`, `title`, `status`, `detail`, `instance` and a `traceId` extension, the client code collapses to a single parser function, and support can correlate a user complaint with a log entry via `traceId`. Validation is especially important: a form client needs to know not a generic «something is wrong», but the specific fields — `quantity`, `sku`, `email` — with an array of messages per field. That is exactly why the derived class `ValidationProblemDetails` exists, carrying an `errors` dictionary.

The lesson shows the recommended .NET 8 path: instead of hand-rolling a middleware with `app.Map`, implement `IExceptionHandler` and delegate formatting to `ProblemDetailsService`. This yields a single code path for an explicit `return Problem()`, validation failures, and exceptions. This homework walks you through every stage: from domain exceptions to curl tests that verify each status code and every ProblemDetails field.

#### What to do step by step
1. Create the project: `dotnet new webapi -n Warehouse -o Warehouse --use-controllers --no-https`, then `cd Warehouse`. Confirm that `Warehouse.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and `<Nullable>enable</Nullable>`. Open `Program.cs` — this will be your DI and pipeline assembly point.

2. Define the domain-exception hierarchy in `Exceptions/AppExceptions.cs`. The abstract base `AppException(int status, string title, string type)` stores the status, the title, and the error-type URI. Derive `NotFoundException` (404, type `https://warehouse.example/errors/not-found`), `ConflictException` (409, `.../conflict`), and `BusinessRuleException` (422, `.../business-rule`). Each exception receives its `detail` through a primary constructor. The goal is to make the exception-to-status mapping predictable and concentrated in one place.

3. Implement the global handler `ProblemDetailsExceptionHandler` in `Infrastructure/ProblemDetailsExceptionHandler.cs`. The class implements `IExceptionHandler` and takes `ILogger<ProblemDetailsExceptionHandler>` and `IProblemDetailsService` via DI. In `TryHandleAsync`, log the exception with the full stack via `logger.LogError(exception, ...)`, then use pattern matching to translate it into a `(status, title, type, detail)` tuple: `AppException` yields its own fields, `ArgumentException` → 400, everything else → 500 with a neutral `detail`. Never send `exception.StackTrace` or internal messages to the client — only `detail`. Populate `ProblemDetails` (Status, Title, Type, Detail, Instance = `Request.Path`) and call `problemDetailsService.TryWriteAsync(new ProblemDetailsContext { ... })`.

4. Describe the domain in `Models/Warehouse.cs`: a `record CreateOrderRequest(Guid ProductId, int Quantity, string CustomerEmail)` and a thread-safe in-memory repository on `ConcurrentDictionary` with a preloaded product. The `OrderService.Place` method should throw `NotFoundException` when the product is absent, `ConflictException` when `Quantity` exceeds stock, and `BusinessRuleException` for an invalid email (no `@`).

5. The `OrdersController` (`[ApiController]`, `[Route("api/[controller]")]`) exposes `POST /api/orders`, which accepts `[FromBody] CreateOrderRequest`, calls `OrderService.Place`, and returns `CreatedAtAction`. Add `GET /api/orders/boom` that throws `InvalidOperationException` — to exercise the 500 path. Do NOT add model-validation attributes yet (`[Required]`, `[Range(1,1000)]`) — we want to see how `ValidationProblemDetails` is produced automatically by the framework through `AddProblemDetails()` plus `[ApiController]`.

6. Assemble `Program.cs`: `AddControllers()`, `AddProblemDetails(o => o.CustomizeProblemDetails = ctx => ctx.ProblemDetails.Extensions["traceId"] = ctx.HttpContext.TraceIdentifier)`, `AddExceptionHandler<ProblemDetailsExceptionHandler>()`. In the pipeline: `UseExceptionHandler()`, then `MapControllers()`. Order is critical — `UseExceptionHandler` must come before `MapControllers`, but after `UseRouting`.

7. Run it: `dotnet run`. In another terminal, run the curl tests and record the output:
   - `curl -i -X POST http://localhost:5000/api/orders -H "Content-Type: application/json" -d "{\"ProductId\":\"00000000-0000-0000-0000-000000000000\",\"Quantity\":1,\"CustomerEmail\":\"a@b.com\"}"` → 404 ProblemDetails with `type=.../not-found`.
   - `curl -i -X POST ... -d "{\"ProductId\":\"<valid-guid>\",\"Quantity\":99999,\"CustomerEmail\":\"a@b.com\"}"` → 409, `detail` about stock.
   - `... -d "{\"ProductId\":\"<guid>\",\"Quantity\":1,\"CustomerEmail\":\"noemail\"}"` → 422.
   - `curl -i http://localhost:5000/api/orders/boom` → 500, neutral `detail`, no stack.
   - Verify every response carries the `Content-Type: application/problem+json` header and a `traceId` in `extensions`.

8. Compare the outputs side by side: the field structure must be identical, only `status`, `title`, `type`, `detail` change. That is consistency.

#### Requirements
- The project targets .NET 8, C# 12, with top-level statements in `Program.cs` and a nullable context enabled.
- Every 4xx and 5xx response returns `Content-Type: application/problem+json` with a ProblemDetails body — no bare status codes without a body, no `text/plain`.
- The `AppException` hierarchy has at least three descendants, each with its own HTTP status and error-type URI.
- The global handler is implemented as an `IExceptionHandler` (not as `app.Use(async (ctx, next) => ...)`), and it uses `IProblemDetailsService.TryWriteAsync` for formatting.
- The stack trace and internal exception messages never reach the client; they are only logged on the server at the `Error` level.
- Every error includes a `traceId` in `extensions`, equal to `HttpContext.TraceIdentifier`.
- Model validation (for example, malformed JSON or a `[ApiController]`-annotation violation) is automatically wrapped in `ValidationProblemDetails` with an `errors` field — no manual code.
- Manual business checks in the service go through exceptions, not through `if (...) return BadRequest(...)` in the controller — the controller stays thin.
- The code compiles without warnings and passes `dotnet build`. The curl tests produce the expected statuses and bodies.

#### Pitfalls
- **Middleware order**: `UseExceptionHandler()` must appear in the pipeline before `MapControllers()`, otherwise controller exceptions will not be intercepted. But it should not precede `UseRouting` if you rely on `Request.Path` for `Instance` — the route must already be matched.
- **A single `AddProblemDetails()` call is enough** — it configures both `ProblemDetailsService` and the customization. Do not register `ProblemDetailsService` manually as a singleton if you already called `AddProblemDetails()` — that creates two distinct instances; the lesson registers `IProblemDetailsService` explicitly, but in real projects `AddProblemDetails()` alone suffices, and you should understand the difference.
- **Never leak the stack**: the temptation to put `exception.StackTrace` into `detail` is strong during debugging, but it is a security hole. The stack belongs in the log; the client gets neutral text.
- **`ValidationProblemDetails` is produced by the framework** automatically once `AddProblemDetails()` is enabled and the controller is decorated with `[ApiController]`. Do not hand-build the `errors` dictionary for attribute failures — that duplicates work and breaks consistency.
- **Pattern matching in the handler**: use a `switch` expression over `exception`, exactly as in the lesson. It centralizes the mapping and makes adding a new domain exception trivial — just a new arm.
- **`Instance = httpContext.Request.Path`** tells the client which endpoint failed. Do not put the full URL with the query string there if it carries sensitive data.
- **`TryWriteAsync` returns a `bool`**: if it returns `false`, the response was not written — consider returning `false` from `TryHandleAsync` so the next handler in the chain can try. In a simple scenario returning `true` is fine.
- **Always log**, even for 4xx business exceptions — `Warning` for business errors, `Error` for 500s. This helps correlation through `traceId`.
- **Do not use `BadRequest("text")`** for validation — it returns a string instead of a ProblemDetails and breaks uniformity.

#### Acceptance criteria
- [ ] The `Warehouse` project builds with `dotnet build` without errors or warnings.
- [ ] `Program.cs` calls `AddProblemDetails(...)` and `AddExceptionHandler<ProblemDetailsExceptionHandler>()`.
- [ ] The pipeline calls `UseExceptionHandler()` before `MapControllers()`.
- [ ] The base `AppException` stores `Status`, `Title`, `Type` through a primary constructor.
- [ ] There are at least three descendants: `NotFoundException` (404), `ConflictException` (409), `BusinessRuleException` (422).
- [ ] `ProblemDetailsExceptionHandler` implements `IExceptionHandler` and uses `IProblemDetailsService.TryWriteAsync`.
- [ ] The stack trace and internal messages never appear in any curl-test response body.
- [ ] Every 4xx/5xx response carries the `Content-Type: application/problem+json` header.
- [ ] Every error body contains the `type`, `title`, `status`, `detail`, `instance` fields.
- [ ] A `traceId` is present in the `extensions` of every error, matching `HttpContext.TraceIdentifier`.
- [ ] A POST with a non-existent `ProductId` → 404, `type=.../not-found`.
- [ ] A POST with `Quantity` exceeding stock → 409, `type=.../conflict`.
- [ ] A POST with an invalid email → 422, `type=.../business-rule`.
- [ ] `GET /api/orders/boom` → 500, neutral `detail`, no stack.
- [ ] A model-validation failure (malformed JSON or a missing `[Required]`) → 400 with `ValidationProblemDetails` and the `errors` field.

#### Hints (no direct answer)
- Reflect on why `IExceptionHandler` is better than an inline middleware: where the logic lives, how it is tested, how a chain of handlers extends.
- Recall the lesson’s «government form» analogy — which fields of the form are always the same, which change case by case.
- Use a `switch` expression with pattern matching to map the exception to a tuple — it shows the reader that the status is determined by the exception type, not by an `if/else` ladder.
- `traceId` is the bridge between the client and the log: verify that after a request you can locate the log entry by this identifier.
- If `Content-Type` stubbornly comes back as `application/json` instead of `application/problem+json`, check that `AddProblemDetails()` is called and that the response goes through `ProblemDetailsService`, not through manual serialization.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — Reference solution for homework M14-L06
// Warehouse: domain exceptions, IExceptionHandler, ProblemDetailsService, ValidationProblemDetails
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Mvc;
using System.Collections.Concurrent;

// --- Domain exceptions ---
// Base class stores status, title and type URI once and immutably
public abstract class AppException(int status, string title, string type) : Exception(title)
{
    public int Status { get; } = status;
    public string Title { get; } = title;
    public string Type { get; } = type;
}

public sealed class NotFoundException(string detail)
    : AppException(404, "Resource not found", "https://warehouse.example/errors/not-found")
{
    public string Detail { get; } = detail;
}

public sealed class ConflictException(string detail)
    : AppException(409, "Conflict", "https://warehouse.example/errors/conflict")
{
    public string Detail { get; } = detail;
}

public sealed class BusinessRuleException(string detail)
    : AppException(422, "Business rule violation", "https://warehouse.example/errors/business-rule")
{
    public string Detail { get; } = detail;
}

// --- Global handler (.NET 8 IExceptionHandler) ---
public sealed class ProblemDetailsExceptionHandler(
    ILogger<ProblemDetailsExceptionHandler> logger,
    IProblemDetailsService problemDetailsService) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext, Exception exception, CancellationToken cancellationToken)
    {
        // Always log: business -> Warning, everything else -> Error
        if (exception is AppException)
            logger.LogWarning(exception, "Business exception: {Message}", exception.Message);
        else
            logger.LogError(exception, "Unhandled exception: {Message}", exception.Message);

        // Pattern matching maps the exception type to a tuple
        var (status, title, type, detail) = exception switch
        {
            NotFoundException nfe => (nfe.Status, nfe.Title, nfe.Type, nfe.Detail),
            ConflictException ce => (ce.Status, ce.Title, ce.Type, ce.Detail),
            BusinessRuleException bre => (bre.Status, bre.Title, bre.Type, bre.Detail),
            AppException ae => (ae.Status, ae.Title, ae.Type, ae.Message),
            ArgumentException => (400, "Bad Request", "https://warehouse.example/errors/bad-request", exception.Message),
            // Stack never leaks to the client — neutral detail only
            _ => (500, "Internal Server Error", "https://warehouse.example/errors/internal", "An unexpected error occurred")
        };

        httpContext.Response.StatusCode = status;

        var problem = new ProblemDetails
        {
            Status = status,
            Title = title,
            Type = type,
            Detail = detail,
            Instance = httpContext.Request.Path
        };

        // Delegate formatting to ProblemDetailsService — this applies traceId and a uniform shape
        return await problemDetailsService.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = httpContext,
            ProblemDetails = problem,
            Exception = exception
        });
    }
}

// --- Domain ---
public sealed record CreateOrderRequest(Guid ProductId, int Quantity, string CustomerEmail);
public sealed record Product(Guid Id, string Sku, int Stock);

public static class Warehouse
{
    public static readonly ConcurrentDictionary<Guid, Product> Products = new();
}

public sealed class OrderService
{
    public Guid Place(CreateOrderRequest r)
    {
        if (!Warehouse.Products.TryGetValue(r.ProductId, out var product))
            throw new NotFoundException($"Product {r.ProductId} not found");

        if (r.Quantity > product.Stock)
            throw new ConflictException($"Only {product.Stock} items left in stock");

        if (!r.CustomerEmail.Contains('@'))
            throw new BusinessRuleException("CustomerEmail must be a valid email address");

        return Guid.NewGuid();
    }
}

// --- Controller ---
[ApiController]
[Route("api/[controller]")]
public sealed class OrdersController(OrderService orders) : ControllerBase
{
    [HttpPost]
    public IActionResult Create([FromBody] CreateOrderRequest request)
    {
        var id = orders.Place(request);     // the exception flies to IExceptionHandler
        return CreatedAtAction(nameof(GetById), new { id }, new { id });
    }

    [HttpGet("{id:guid}")]
    public IActionResult GetById(Guid id) => Ok(new { id });

    [HttpGet("boom")]   // to exercise the 500 path
    public IActionResult Boom() => throw new InvalidOperationException("Simulated failure");
}

// --- Program.cs ---
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddSingleton<OrderService>();

// A single call configures ProblemDetailsService plus the traceId customization
builder.Services.AddProblemDetails(o =>
    o.CustomizeProblemDetails = ctx =>
        ctx.ProblemDetails.Extensions["traceId"] = ctx.HttpContext.TraceIdentifier);

builder.Services.AddExceptionHandler<ProblemDetailsExceptionHandler>();

var app = builder.Build();

app.UseExceptionHandler();   // before MapControllers, after the implicit UseRouting
app.MapControllers();

// Seed product
Warehouse.Products[Guid.Empty] = new Product(Guid.Empty, "DEMO-1", 5);

app.Run();
```

Line-by-line walk-through. The base `AppException` uses a C# 12 primary constructor — the status, title, and type are set once and are immutable, which guarantees a predictable mapping (the lesson’s concept: «each business exception knows its status»). Descendants merely refine `detail` for the specific case. `ProblemDetailsExceptionHandler` implements `IExceptionHandler` exactly — the path recommended in .NET 8, not an inline middleware: the framework wires it into the pipeline through `UseExceptionHandler()`. Logging is split by level: business errors go to `Warning`, because they are expected user behavior, not a system fault; everything else goes to `Error` with the full stack in the log. Pattern matching in the `switch` expression centralizes the type-to-tuple mapping — adding a new exception is a single line, with no controller changes. Critically, the `_ => (500, ...)` arm returns a neutral `detail`, not `exception.Message` or the stack — this closes the security hole from the «common mistakes» section. `Instance = httpContext.Request.Path` shows the client which endpoint failed. Delegating to `problemDetailsService.TryWriteAsync` guarantees that the `traceId` customization applies to exceptions, an explicit `Problem()`, and validation alike — this is consistency. The controller stays thin: it does not catch exceptions and does not return `BadRequest("text")`, because all formatting responsibility is pushed to the handler and `ProblemDetailsService`. A single `AddProblemDetails()` call handles both formatting and `traceId` — a separate `ProblemDetailsService` registration is unnecessary. The `UseExceptionHandler()` → `MapControllers()` order is mandatory: otherwise controller exceptions fall through to a default 500 with no body.

#### Going deeper (bonus)
1. Add `ValidationProblemDetails` for manual business validation: write an extension method that builds a `ValidationProblemDetails` from an `IDictionary<string, string[]>` and writes it through `ProblemDetailsService`, so multi-field errors surface as an `errors` dictionary.
2. Implement a second `IExceptionHandler` for rate-limiting that returns 429 with a `Retry-After` header, and confirm that handler registration order determines who intercepts the exception first.
3. Add a `Correlation-Id` request header into `extensions` (generating one if absent) and verify it lands both in the log and in the ProblemDetails.
4. Cover the handler with `WebApplicationFactory`-based unit tests, asserting each exception type yields the correct status and that the stack is absent from the body.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `Warehouse` собирается без ошибок и предупреждений. (RU)
- [ ] В `Program.cs` есть `AddProblemDetails()` и `AddExceptionHandler<>()`. (RU)
- [ ] `UseExceptionHandler()` вызван раньше `MapControllers()`. (RU)
- [ ] Реализован `IExceptionHandler` через `ProblemDetailsService.TryWriteAsync`. (RU)
- [ ] Стек-трейс не утекает клиенту ни в одном сценарии. (RU)
- [ ] Все 4xx/5xx ответы имеют `application/problem+json` и `traceId`. (RU)
- [ ] Прогнаны все curl-тесты, выводы зафиксированы. (RU)
- [ ] The `Warehouse` project builds without errors or warnings. (EN)
- [ ] `Program.cs` contains `AddProblemDetails()` and `AddExceptionHandler<>()`. (EN)
- [ ] `UseExceptionHandler()` is called before `MapControllers()`. (EN)
- [ ] An `IExceptionHandler` is implemented via `ProblemDetailsService.TryWriteAsync`. (EN)
- [ ] The stack trace never leaks to the client in any scenario. (EN)
- [ ] All 4xx/5xx responses carry `application/problem+json` and a `traceId`. (EN)
- [ ] All curl tests are run and their outputs are recorded. (EN)

#### Ресурсы / Resources
- [Microsoft Learn — Handle errors in ASP.NET Core web APIs](https://learn.microsoft.com/aspnet/core/web-api/handle-errors)
- [RFC 7807 — Problem Details for HTTP APIs](https://datatracker.ietf.org/doc/html/rfc7807)
- [Microsoft Learn — ProblemDetails and IExceptionHandler in .NET 8](https://learn.microsoft.com/aspnet/core/fundamentals/error-handling)
- [Microsoft Learn — Model validation in ASP.NET Core](https://learn.microsoft.com/aspnet/core/mvc/models/validation)
