[⬆ К модулю M14](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M14-L06: ProblemDetails, обработка ошибок API / ProblemDetails, API error handling

**Модуль / Module:** M14
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Когда API возвращает ошибку, клиенту важно понять не только статус-код, но и причину, и поля, которые её вызвали. Долгое время разработчики возвращали кто во что гораз: кто-то — пустой 500, кто-то — JSON с `message`, кто-то — текст в теле. В итоге каждый клиент писал собственный парсер под каждый сервис. RFC 7807 решил эту проблему, введя единый формат `application/problem+json` — **ProblemDetails**.

**ProblemDetails** — это стандартизованный документ ошибки со следующими полями:
- `type` — URI с описанием типа ошибки (по умолчанию `about:blank`);
- `title` — короткое описание;
- `status` — HTTP-статус;
- `detail` — пояснение для конкретного случая;
- `instance` — URI запроса.

Аналогия: ProblemDetails — это как стандартный бланк справки в МФЦ. Независимо от того, по какому вопросу вы пришли, бланк имеет одни и те же поля: «Тип услуги», «Статус», «Комментарий». Клиенту не нужно гадать, где искать причину — она всегда в `detail`.

В ASP.NET Core (`Microsoft.AspNetCore.Mvc`) ProblemDetails встроен «из коробки». Начиная с .NET 8 при включении `AddProblemDetails()` API автоматически оборачивает ошибки контроллеров в ProblemDetails. Для ошибок валидации используется производный класс **ValidationProblemDetails**, который добавляет поле `errors` — словарь «поле → массив сообщений». Это критично: клиенту формы нужно показать, что именно не так с `email` или `password`, а не общую фразу.

Однако контроллеры покрывают только половину ошибок. Исключения могут лететь из middleware, фильтров, фоновых сервисов, инфраструктуры. Здесь на сцену выходит **custom exception handler middleware** — компонент в конвейере `app.Map(...)` или реализация `IExceptionHandler` (рекомендуемый путь в .NET 8). Он перехватывает исключение, логирует его и преобразует в ProblemDetails с правильным статусом. Важно: не возвращайте стек вызовов наружу — это дыра в безопасности. Стек должен идти в лог, а клиенту — только `detail` и `type`.

**Консистентность ответов об ошибках (error responses consistency)** — главная цель. Согласованность означает:
1. Все ошибки (4xx и 5xx) возвращают `Content-Type: application/problem+json`.
2. Все ошибки имеют одну и ту же структуру ProblemDetails.
3. Бизнес-исключения (например, `NotFoundException`, `ConflictException`) отображаются на предсказуемые статус-коды.
4. Никогда не возвращается «голый» `500 Internal Server Error` без тела.

Чтобы добиться этого, создайте иерархию доменных исключений, зарегистрируйте `IExceptionHandler`, который их переводит в ProblemDetails, и используйте `ProblemDetailsService` для форматирования. Тогда и валидация, и исключения, и ручной `return Problem()` будут выглядеть для клиента одинаково — а единообразие и есть качество API.

#### Theory (EN)

When an API returns an error, the client needs more than just a status code: it needs the reason, and for validation failures, the specific fields that caused them. For years developers returned whatever they fancied — an empty 500, a JSON with `message`, plain text in the body. Every client ended up writing its own parser per service. RFC 7807 fixed that by introducing a single `application/problem+json` media type: **ProblemDetails**.

**ProblemDetails** is a standardized error document with these fields:
- `type` — a URI describing the error type (defaults to `about:blank`);
- `title` — a short summary;
- `status` — the HTTP status;
- `detail` — a human-readable explanation for this specific occurrence;
- `instance` — the request URI.

Analogy: ProblemDetails is like a standard form at a government office. No matter what you came for, the form has the same fields: “Service type”, “Status”, “Comment”. The client never has to guess where the reason lives — it is always in `detail`.

In ASP.NET Core (`Microsoft.AspNetCore.Mvc`) ProblemDetails is built in. Starting with .NET 8, when you call `AddProblemDetails()`, the framework automatically wraps controller errors in ProblemDetails. For validation failures there is a derived class, **ValidationProblemDetails**, which adds an `errors` dictionary mapping “field → array of messages”. This matters: a form client needs to know exactly what is wrong with `email` or `password`, not a generic phrase.

Controllers, however, cover only half the errors. Exceptions can fly out of middleware, filters, background services, infrastructure. This is where **custom exception handler middleware** enters the stage — either a delegate registered with `app.Map(...)` or, the recommended path in .NET 8, an implementation of `IExceptionHandler`. It catches the exception, logs it, and converts it to ProblemDetails with the correct status. Critical rule: never leak the stack trace to the client — that is a security hole. The stack belongs in the log; the client gets only `detail` and `type`.

**Error responses consistency** is the main goal. Consistency means:
1. All errors (4xx and 5xx) return `Content-Type: application/problem+json`.
2. All errors share the same ProblemDetails shape.
3. Business exceptions (e.g. `NotFoundException`, `ConflictException`) map to predictable status codes.
4. A bare `500 Internal Server Error` with no body is never returned.

To achieve this, design a hierarchy of domain exceptions, register an `IExceptionHandler` that translates them into ProblemDetails, and rely on `ProblemDetailsService` for formatting. Then validation, exceptions, and an explicit `return Problem()` all look identical to the client — and uniformity is what makes an API feel professional.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — ProblemDetails, обработка ошибок API
// Полный рабочий пример: доменные исключения, IExceptionHandler, контроллер, Program.cs

using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Mvc.Infrastructure;
using System.Collections.Concurrent;

// --- Доменные исключения / Domain exceptions ---
// Каждое бизнес-исключение знает свой HTTP-статус и тип / Each business exception knows its status and type
public abstract class AppException(int status, string title, string type) : Exception(title)
{
    public int Status { get; } = status;       // HTTP-статус / HTTP status
    public string Title { get; } = title;       // Короткое описание / Short summary
    public string Type { get; } = type;         // URI типа ошибки / Error type URI
}

public sealed class NotFoundException(string detail)
    : AppException(404, "Resource not found", "https://example.com/errors/not-found")
{
    public string Detail { get; } = detail;     // Конкретная причина / Specific reason
}

public sealed class ConflictException(string detail)
    : AppException(409, "Conflict", "https://example.com/errors/conflict")
{
    public string Detail { get; } = detail;
}

// --- Глобальный обработчик исключений / Global exception handler (.NET 8: IExceptionHandler) ---
public sealed class ProblemDetailsExceptionHandler(ILogger<ProblemDetailsExceptionHandler> logger,
    IProblemDetailsService problemDetailsService) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext, Exception exception, CancellationToken cancellationToken)
    {
        // Логируем ВСЕГДА, стек — в лог, не клиенту / Always log; stack goes to log, not to client
        logger.LogError(exception, "Необработанное исключение: {Message} / Unhandled exception: {Message}",
            exception.Message);

        // Транслируем доменное исключение в статус + ProblemDetails / Map domain exception to status + ProblemDetails
        var (status, title, type, detail) = exception switch
        {
            AppException ae => (ae.Status, ae.Title, ae.Type,
                ae is NotFoundException nfe ? nfe.Detail
                : ae is ConflictException ce ? ce.Detail : ae.Message),
            ArgumentException => (400, "Bad Request", "https://example.com/errors/bad-request", exception.Message),
            _ => (500, "Internal Server Error", "https://example.com/errors/internal", "Произошла ошибка / An error occurred")
        };

        httpContext.Response.StatusCode = status;

        // Используем ProblemDetailsService для единого форматирования / Use ProblemDetailsService for uniform formatting
        var problem = new ProblemDetails
        {
            Status = status,
            Title = title,
            Type = type,
            Detail = detail,
            Instance = httpContext.Request.Path
        };

        return await problemDetailsService.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = httpContext,
            ProblemDetails = problem,
            Exception = exception
        });
    }
}

// --- DTO и фейковый репозиторий / DTO and fake repository ---
public sealed record CreateProductRequest(string Name, decimal Price);

public static class ProductRepo
{
    private static readonly ConcurrentDictionary<Guid, CreateProductRequest> _items = new();

    public static void Add(Guid id, CreateProductRequest r) => _items[id] = r;

    public static CreateProductRequest Get(Guid id) =>
        _items.TryGetValue(id, out var r) ? r : throw new NotFoundException($"Product {id} not found");
}

// --- Контроллер / Controller ---
[ApiController]
[Route("api/[controller]")]
public sealed class ProductsController : ControllerBase
{
    [HttpPost]
    public IActionResult Create([FromBody] CreateProductRequest request)
    {
        // Ручная бизнес-валидация → ConflictException → 409 ProblemDetails
        // Manual business validation → ConflictException → 409 ProblemDetails
        if (request.Price < 0)
            throw new ConflictException("Price cannot be negative");

        if (string.IsNullOrWhiteSpace(request.Name))
            throw new ArgumentException("Name is required", nameof(request));

        var id = Guid.NewGuid();
        ProductRepo.Add(id, request);
        return CreatedAtAction(nameof(GetById), new { id }, new { id, request });
    }

    [HttpGet("{id:guid}")]
    public IActionResult GetById(Guid id) => Ok(ProductRepo.Get(id));

    [HttpGet("boom")]
    public IActionResult Boom() => throw new InvalidOperationException("Симулированная ошибка / Simulated failure");
}

// --- Program.cs ---
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
// Включаем ProblemDetails для единообразия всех ошибок / Enable ProblemDetails for uniform errors
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = ctx =>
    {
        // Добавляем traceId ко всем ошибкам — полезно для поддержки / Add traceId to all errors — useful for support
        ctx.ProblemDetails.Extensions["traceId"] = ctx.HttpContext.TraceIdentifier;
    };
});

// Регистрируем наш обработчик / Register our handler
builder.Services.AddExceptionHandler<ProblemDetailsExceptionHandler>();
builder.Services.AddSingleton<IProblemDetailsService, ProblemDetailsService>();

var app = builder.Build();

app.UseExceptionHandler();   // Подключает IExceptionHandler / Wires up IExceptionHandler
app.MapControllers();

app.Run();
```

#### Best Practices

- Возвращайте ошибки через `Problem()` или исключения, а не через «голые» статус-коды без тела. / Return errors via `Problem()` or exceptions, never bare status codes with no body.
- Логируйте исключения с полным стеком на сервере; клиенту отдавайте только `detail` и `type`. / Log exceptions with full stack on the server; give the client only `detail` and `type`.
- Используйте `ValidationProblemDetails` для ошибок валидации — он несёт словарь `errors` по полям. / Use `ValidationProblemDetails` for validation errors — it carries the per-field `errors` dictionary.
- Заведите иерархию доменных исключений, каждое со своим статусом, чтобы перевод был предсказуемым. / Maintain a domain-exception hierarchy, each with its own status, so mapping is predictable.
- Добавляйте `traceId` в `extensions`, чтобы связать ошибку клиента с записью в логе. / Add `traceId` to `extensions` to tie a client error to a log entry.
- Включайте `AddProblemDetails()` один раз в `Program.cs`, чтобы все ошибки форматировались одинаково. / Enable `AddProblemDetails()` once in `Program.cs` so all errors format the same way.

#### Частые ошибки / Common Mistakes

- Возврат `500` с пустым телом или без `Content-Type: application/problem+json` → всегда включайте `AddProblemDetails()` и `UseExceptionHandler()`. (RU)
- Возвращение стек-трейса и внутренних сообщений клиенту → логируйте стек на сервере, наружу отдавайте только `detail`. (RU)
- Использование одного `Exception` для всех ошибок без разделения статусов → создайте иерархию `AppException` с разными статусами. (RU)
- Ручной возврат `BadRequest("text")` для ошибок валидации → используйте `ValidationProblem()` или `BadRequest(ModelState)`, чтобы получить `errors`. (RU)
- Перехват исключений в каждом контроллере через `try/catch` → вынесите обработку в `IExceptionHandler` и логику в одном месте. (RU)

- Returning `500` with an empty body or without `Content-Type: application/problem+json` → always enable `AddProblemDetails()` and `UseExceptionHandler()`. (EN)
- Leaking the stack trace and internal messages to the client → log the stack on the server, return only `detail`. (EN)
- Using a single `Exception` for all errors with no status separation → build an `AppException` hierarchy with distinct statuses. (EN)
- Returning `BadRequest("text")` for validation errors → use `ValidationProblem()` or `BadRequest(ModelState)` to get the `errors` dictionary. (EN)
- Wrapping every controller in `try/catch` → centralize handling in an `IExceptionHandler` so logic lives in one place. (EN)

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Включён `AddProblemDetails()` и вызван `UseExceptionHandler()` в `Program.cs`. (RU)
- [ ] Все ошибки (4xx и 5xx) возвращают `application/problem+json`. (RU)
- [ ] Создан `IExceptionHandler`, переводящий исключения в ProblemDetails. (RU)
- [ ] Доменные исключения имеют предсказуемые статусы (404, 409, 400). (RU)
- [ ] Стек-трейс логируется, но не уходит клиенту. (RU)
- [ ] Ошибки валидации используют `ValidationProblemDetails` с полем `errors`. (RU)
- [ ] Добавлен `traceId` в `extensions` для связи с логом. (RU)

- [ ] `AddProblemDetails()` is enabled and `UseExceptionHandler()` is called in `Program.cs`. (EN)
- [ ] All errors (4xx and 5xx) return `application/problem+json`. (EN)
- [ ] An `IExceptionHandler` is implemented that maps exceptions to ProblemDetails. (EN)
- [ ] Domain exceptions have predictable statuses (404, 409, 400). (EN)
- [ ] The stack trace is logged but never sent to the client. (EN)
- [ ] Validation errors use `ValidationProblemDetails` with the `errors` field. (EN)
- [ ] A `traceId` is added to `extensions` to correlate with logs. (EN)

#### Ресурсы / Resources

- [Microsoft Learn — Handle errors in ASP.NET Core web APIs — https://learn.microsoft.com/aspnet/core/web-api/handle-errors](https://learn.microsoft.com/aspnet/core/web-api/handle-errors)

---

[⬆ К модулю M14](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
