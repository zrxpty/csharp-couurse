[⬆ К модулю M13](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M13-L09: Error handling middleware, environment-specific startup / Error handling middleware, environment-specific startup

**Модуль / Module:** M13
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Обработка ошибок в ASP.NET Core — это не одна строка кода, а конвейер middleware, выстроенный в правильном порядке. Представьте себе станцию скорой помощи: первая «скорая» — middleware, которая перехватывает исключения, вторая — та, что показывает дружелюбную страницу пользователю, третья — диагностическая, для разработчика. Если расставить их неправильно, пациент (запрос) пройдёт мимо помощи.

Ключевые инструменты:

1. **UseDeveloperExceptionPage** — детальная страница с трассировкой стека, исходным кодом, заголовками и запросом. Включается ТОЛЬКО в среде `Development`. В production её включать опасно: стек и переменные могут раскрыть чувствительные данные.

2. **UseExceptionHandler** — «амортизатор» для production. Перехватывает исключения и направляет запрос на резервный путь (страницу `/Error` или лямбду-обработчик), где формируется безопасное сообщение. Можно настроить через `ExceptionHandlerOptions`, указав путь (`UseExceptionHandler("/Error")`) или кастомный обработчик.

3. **UseStatusCodePages** — обрабатывает HTTP-ответы с кодами 400–599, у которых нет тела. По умолчанию добавляет простой текстовый ответ. Варианты: `UseStatusCodePagesWithReExecute` (повторно выполняет конвейер по другому пути, например `/Error/{0}`) и `UseStatusCodePagesWithRedirects` (редиректит клиента). `ReExecute` предпочтительнее: URL в адресной строке не меняется и сохраняется контекст исходного запроса.

4. **IWebHostEnvironment** — сервис, который говорит, в какой среде работает приложение. Свойство `EnvironmentName` принимает значения `Development`, `Staging`, `Production` (но это просто строки — можно задать свои). Среда задаётся переменной `ASPNETCORE_ENVIRONMENT`. В `Program.cs` расширение `builder.Environment.IsDevelopment()` позволяет ветвить логику старта.

5. **appsettings по средам** — конфигурация грузится каскадом: `appsettings.json` → `appsettings.{Environment}.json` → переменные окружения. Это позволяет, например, держать в `appsettings.Development.json` строку подключения к локальной БД, а в `appsettings.Production.json` — к боевой. Файл среды переопределяет базовые значения.

Порядок middleware критичен. Перехватчик исключений должен стоять РАНЬШЕ маршрутизации и endpoints, иначе исключения из контроллеров не будут пойманы. `UseStatusCodePages` ставят после `UseExceptionHandler`, но до `UseRouting`. Типичный скелет:

```
Development: UseDeveloperExceptionPage
Production:  UseExceptionHandler("/Error") → UseStatusCodePagesWithReExecute
```

Глобальная обработка также дублируется фильтрами (`IExceptionFilter`) и middleware-классами, но для большинства сценариев достаточно связки `UseExceptionHandler` + страница `/Error`. Логи пишутся автоматически через `ILogger`, но в кастомном обработчике полезно логировать `exception.Handle()` с корреляционным ID.

Важно понимать разницу между исключением (500) и ошибкой по статус-коду (404, 403). `UseExceptionHandler` ловит только исключения; для 404 нужен `UseStatusCodePages`. Многие новички вешают только `UseExceptionHandler` и удивляются, почему пользователи видят пустую страницу на несуществующем URL.

Наконец, не показывайте пользователю технические детали в production. Дружелюбная страница «Что-то пошло не так, мы уже работаем над этим» с корреляционным ID — это и безопасность, и UX. Подробности оставляем для логов и для `Development`.

#### Theory (EN)

Error handling in ASP.NET Core is not a single line of code — it is a middleware pipeline arranged in the right order. Imagine a hospital emergency room: the first “ambulance” is the middleware that catches exceptions, the second renders a friendly page to the user, and the third is a diagnostic page for the developer. If you place them in the wrong order, the patient (the request) slips past all the help.

The core tools:

1. **UseDeveloperExceptionPage** — a detailed page with the stack trace, source code, headers, and request details. Enable it ONLY in the `Development` environment. Running it in production is dangerous: the stack and local variables can leak sensitive information.

2. **UseExceptionHandler** — the “shock absorber” for production. It catches exceptions and re-executes the request on a fallback path (an `/Error` page or a lambda handler) where a safe message is produced. You configure it with `ExceptionHandlerOptions`, passing either a path (`UseExceptionHandler("/Error")`) or a custom delegate.

3. **UseStatusCodePages** — handles HTTP responses with status codes 400–599 that have no body. By default it injects a plain-text response. Variants include `UseStatusCodePagesWithReExecute` (re-runs the pipeline on a different path, e.g. `/Error/{0}`) and `UseStatusCodePagesWithRedirects` (issues a client-side redirect). `ReExecute` is preferable: the URL in the browser address bar stays the same and the original request context is preserved.

4. **IWebHostEnvironment** — a service that tells the app which environment it is running in. The `EnvironmentName` property accepts `Development`, `Staging`, `Production` (these are just strings, you can invent your own). The environment is set with the `ASPNETCORE_ENVIRONMENT` variable. In `Program.cs`, the `builder.Environment.IsDevelopment()` extension lets you branch startup logic.

5. **appsettings per environment** — configuration is loaded as a cascade: `appsettings.json` → `appsettings.{Environment}.json` → environment variables. This lets you keep a local database connection string in `appsettings.Development.json` and the live one in `appsettings.Production.json`. The environment file overrides the base values.

Middleware order is critical. The exception handler must be placed BEFORE routing and endpoints, otherwise exceptions thrown from controllers will not be caught. `UseStatusCodePages` goes after `UseExceptionHandler` but before `UseRouting`. A typical skeleton looks like:

```
Development: UseDeveloperExceptionPage
Production:  UseExceptionHandler("/Error") → UseStatusCodePagesWithReExecute
```

Global handling is also duplicated by MVC filters (`IExceptionFilter`) and custom middleware classes, but for most scenarios the `UseExceptionHandler` + `/Error` page combination is enough. Logs are written automatically via `ILogger`, but in a custom handler it is useful to log the exception with a correlation ID.

It is important to understand the difference between an exception (500) and a status-code error (404, 403). `UseExceptionHandler` catches only exceptions; for 404 you need `UseStatusCodePages`. Many beginners install only `UseExceptionHandler` and are surprised when users see a blank page on a non-existent URL.

Finally, never show technical details to users in production. A friendly page — “Something went wrong, we are already on it” — with a correlation ID is both security and good UX. The details stay in the logs and in `Development`.

#### Пример кода / Code Example

```csharp
// Program.cs — C# 12 / .NET 8
// Глобальная обработка ошибок и конфигурация по средам
// Global error handling and environment-specific configuration

var builder = WebApplication.CreateBuilder(args);

// Регистрируем сервисы / Register services
builder.Services.AddControllersWithViews();
builder.Services.AddProblemDetails(); // RFC 9457 детали ошибок / RFC 9457 problem details

// Кастомный логгер-обработчик (опционально) / Custom logging handler (optional)
builder.Services.AddSingleton<IProblemDetailsService, ProblemDetailsService>();

var app = builder.Build();

// Ветвление старта по среде / Branch startup by environment
// ВАЖНО: порядок middleware имеет значение / IMPORTANT: middleware order matters
if (app.Environment.IsDevelopment())
{
    // Подробная страница для разработчика / Detailed developer page
    app.UseDeveloperExceptionPage();
}
else
{
    // В production: перехват исключений и reroute на /Error / Production: catch and reroute to /Error
    app.UseExceptionHandler("/Error");

    // Дружелюбные страницы для 4xx/5xx без тела / Friendly pages for bodyless 4xx/5xx
    // ReExecute сохраняет URL и контекст / ReExecute preserves URL and context
    app.UseStatusCodePagesWithReExecute("/Error/Status/{0}");
}

// HSTS и HTTPS redirect — только вне Development / HSTS and HTTPS redirect — outside Development only
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();

app.UseRouting();
app.UseAuthorization();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

// Endpoint для ошибок / Error endpoint
app.MapControllerRoute(
    name: "error",
    pattern: "Error/{action=Status}/{code:int}",
    defaults: new { controller = "Error" });

app.Run();

// --- Контроллер ошибок / Error controller ---
// ErrorController.cs
public class ErrorController : Controller
{
    private readonly ILogger<ErrorController> _logger;
    private readonly IWebHostEnvironment _env;

    public ErrorController(ILogger<ErrorController> logger, IWebHostEnvironment env)
    {
        _logger = logger;
        _env = env;
    }

    // Обработка необработанных исключений / Handle unhandled exceptions
    [Route("/Error")]
    [HttpGet]
    public IActionResult Index()
    {
        // IExceptionHandlerPathFeature содержит исключение и путь / Contains exception and path
        var exceptionFeature = HttpContext.Features.Get<IExceptionHandlerPathFeature>();
        var traceId = Activity.Current?.Id ?? HttpContext.TraceIdentifier;

        if (exceptionFeature is not null)
        {
            // Логируем всегда, подробности только в Dev / Log always, details in Dev only
            _logger.LogError(exceptionFeature.Error,
                "Необработанное исключение на пути {Path}. TraceId: {TraceId} / " +
                "Unhandled exception at {Path}. TraceId: {TraceId}",
                exceptionFeature.Path, traceId);
        }

        // Возвращает проблемные детали (RFC 9457) / Return problem details (RFC 9457)
        var problem = new ProblemDetails
        {
            Title = "Произошла ошибка / An error occurred",
            Status = 500,
            Detail = _env.IsDevelopment()
                ? exceptionFeature?.Error.Message // Dev: детали / Dev: details
                : null,                            // Prod: скрыть / Prod: hide
            Instance = exceptionFeature?.Path,
        };
        problem.Extensions["traceId"] = traceId;

        return StatusCode(500, problem);
    }

    // Обработка кодов состояния (404, 403 и т.д.) / Status code handling
    [Route("/Error/Status/{code:int}")]
    public IActionResult Status(int code)
    {
        _logger.LogWarning("Статус {Code} на пути {Path}. TraceId: {TraceId} / " +
            "Status {Code} at {Path}. TraceId: {TraceId}",
            code, HttpContext.Request.Path, HttpContext.TraceIdentifier);

        return code switch
        {
            404 => View("NotFound"),
            401 or 403 => View("Forbidden"),
            _ => View("Error", code),
        };
    }
}
```

#### Best Practices

- Ставьте `UseExceptionHandler` / `UseDeveloperExceptionPage` самым первым в pipeline, до `UseRouting` — иначе исключения из контроллеров не перехватятся.
- В production никогда не показывайте стек и исходный код: только корреляционный ID и дружелюбное сообщение.
- Используйте `UseStatusCodePagesWithReExecute`, а не `WithRedirects` — URL и контекст запроса сохраняются, что упрощает логирование и SEO.
- Разделяйте конфигурацию через `appsettings.{Environment}.json` и переопределяйте секреты через переменные окружения или Secret Manager (Dev) / Key Vault (Prod).
- Логируйте исключения с `TraceIdentifier`/`Activity.Current.Id`, чтобы связать сообщение пользователю с записью в логах.
- Применяйте `AddProblemDetails()` для единого RFC 9457 формата ответов об ошибках в API.

- Place `UseExceptionHandler` / `UseDeveloperExceptionPage` first in the pipeline, before `UseRouting` — otherwise controller exceptions are not caught.
- In production, never expose the stack trace or source code: only a correlation ID and a friendly message.
- Prefer `UseStatusCodePagesWithReExecute` over `WithRedirects` — the URL and request context are preserved, which simplifies logging and SEO.
- Split configuration through `appsettings.{Environment}.json` and override secrets via environment variables or Secret Manager (Dev) / Key Vault (Prod).
- Log exceptions with a `TraceIdentifier` / `Activity.Current.Id` to tie the user-facing message to a log entry.
- Use `AddProblemDetails()` for a unified RFC 9457 error response format in APIs.

#### Частые ошибки / Common Mistakes

- `UseExceptionHandler` стоит после `UseRouting` → исключения из endpoints не ловятся. Решение: ставьте обработчик первым.
- `UseDeveloperExceptionPage` включён в production → утечка стека и исходников. Решение: ветвите через `IsDevelopment()`.
- Только `UseExceptionHandler`, без `UseStatusCodePages` → пустая страница на 404. Решение: добавьте `WithReExecute`.
- Жёстко прописаны строки подключения в `appsettings.json` → невозможно менять по средам. Решение: вынесите в `appsettings.{Environment}.json` и env-vars.
- В кастомной странице `/Error` показывается `exception.Message` в production → утечка деталей. Решение: скрывайте через `IsDevelopment()`.
- Используется `WithRedirects` для 404 → меняется URL и теряется исходный путь. Решение: используйте `WithReExecute`.

- `UseExceptionHandler` placed after `UseRouting` → endpoint exceptions are not caught. Fix: put the handler first.
- `UseDeveloperExceptionPage` left on in production → stack and source leak. Fix: branch via `IsDevelopment()`.
- Only `UseExceptionHandler`, no `UseStatusCodePages` → blank page on 404. Fix: add `WithReExecute`.
- Connection strings hardcoded in `appsettings.json` → impossible to vary by environment. Fix: move to `appsettings.{Environment}.json` and env vars.
- Custom `/Error` page shows `exception.Message` in production → detail leak. Fix: gate with `IsDevelopment()`.
- `WithRedirects` used for 404 → URL changes and the original path is lost. Fix: use `WithReExecute`.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] `UseExceptionHandler` / `UseDeveloperExceptionPage` стоят до `UseRouting`.
- [ ] В production включён `UseExceptionHandler`, а developer page — выключен.
- [ ] Добавлены `UseStatusCodePages` (лучше `WithReExecute`) для 4xx/5xx без тела.
- [ ] Среда определяется через `ASPNETCORE_ENVIRONMENT` и ветвится `IsDevelopment()`.
- [ ] Созданы `appsettings.Development.json` и `appsettings.Production.json`.
- [ ] Секреты в production приходят из env-vars или Key Vault, а не из файлов.
- [ ] Кастомная страница `/Error` не раскрывает стек и `exception.Message` в production.
- [ ] Исключения логируются с `TraceIdentifier`/`Activity.Current.Id`.
- [ ] Для API подключён `AddProblemDetails()` (RFC 9457).
- [ ] HSTS и HTTPS redirect включены только вне Development.

- [ ] `UseExceptionHandler` / `UseDeveloperExceptionPage` are placed before `UseRouting`.
- [ ] In production `UseExceptionHandler` is on and the developer page is off.
- [ ] `UseStatusCodePages` (preferably `WithReExecute`) is added for bodyless 4xx/5xx.
- [ ] The environment is set via `ASPNETCORE_ENVIRONMENT` and branched with `IsDevelopment()`.
- [ ] `appsettings.Development.json` and `appsettings.Production.json` exist.
- [ ] Production secrets come from env vars or Key Vault, not from files.
- [ ] The custom `/Error` page does not expose the stack or `exception.Message` in production.
- [ ] Exceptions are logged with a `TraceIdentifier` / `Activity.Current.Id`.
- [ ] `AddProblemDetails()` (RFC 9457) is registered for APIs.
- [ ] HSTS and HTTPS redirect are enabled only outside Development.

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/aspnet/core/fundamentals/error-handling](https://learn.microsoft.com/aspnet/core/fundamentals/error-handling)
- [Microsoft Learn — Use multiple environments in ASP.NET Core — https://learn.microsoft.com/aspnet/core/fundamentals/environments](https://learn.microsoft.com/aspnet/core/fundamentals/environments)
- [Microsoft Learn — Status code pages — https://learn.microsoft.com/aspnet/core/fundamentals/error-handling#usestatuscodepages](https://learn.microsoft.com/aspnet/core/fundamentals/error-handling#usestatuscodepages)
- [RFC 9457 — Problem Details for HTTP APIs — https://datatracker.ietf.org/doc/html/rfc9457](https://datatracker.ietf.org/doc/html/rfc9457)

---

[⬆ К модулю M13](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
