---
[← К уроку M13-L09](lesson-M13-L09-error-handling-environments.md) | [⬆ К модулю M13](../README.md) | [Следующее ДЗ →]()
---

### Домашнее задание M13-L09: Error handling middleware, environment-specific startup / Homework M13-L09: Error handling middleware, environment-specific startup

**Урок / Lesson:** M13-L09
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться выстраивать корректный конвейер middleware для обработки ошибок в ASP.NET Core 8, ветвить запуск приложения по средам (Development / Staging / Production), разделять конфигурацию через каскад `appsettings.{Environment}.json` и переменные окружения, а также применять RFC 9457 Problem Details для единообразных ответов об ошибках. (EN) Learn to build a correct error-handling middleware pipeline in ASP.NET Core 8, branch application startup by environment (Development / Staging / Production), split configuration through the `appsettings.{Environment}.json` cascade and environment variables, and apply RFC 9457 Problem Details for uniform error responses.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит пять ключевых инструментов: `UseDeveloperExceptionPage`, `UseExceptionHandler`, `UseStatusCodePages` (с акцентом на `WithReExecute`), `IWebHostEnvironment` и каскад `appsettings` по средам. Домашнее задание требует собрать всё это в одном работающем проекте, воспроизвести типичный скелет `Program.cs` из урока, реализовать `ErrorController` с извлечением `IExceptionHandlerPathFeature`, и намеренно нарушить порядок middleware, чтобы на себе прочувствовать частые ошибки из чек-листа урока.
(EN) The lesson introduces five core tools: `UseDeveloperExceptionPage`, `UseExceptionHandler`, `UseStatusCodePages` (with emphasis on `WithReExecute`), `IWebHostEnvironment`, and the per-environment `appsettings` cascade. The homework asks you to combine all of them in one runnable project, reproduce the typical `Program.cs` skeleton from the lesson, implement an `ErrorController` that extracts `IExceptionHandlerPathFeature`, and deliberately break the middleware order so you personally feel the common mistakes from the lesson checklist.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы присоединились к команде, которая поддерживает внутренний портал на ASP.NET Core 8 (.NET 8, C# 12). Пользователи жалуются на две повторяющиеся проблемы. Во-первых, при любом сбое в контроллере они видят пустую белую страницу без единого объяснения, а в логах_support-инженеры не могут связать жалобу клиента с конкретной записью, потому что нигде не фигурирует корреляционный идентификатор. Во-вторых, при переходе на несуществующий URL (например, опечатка в ссылке из рассылки) браузер тоже показывает пустоту, хотя сервер возвращает 404. Параллельно разработчики страдают от того, что локально у них работает строка подключения к локальной базе, а на боевом сервере приложение упорно ходит в неё же, потому что всё захардкожено в едином `appsettings.json`.

Ревью кодовой базы выявляет сразу несколько проблем из чек-листа урока: `UseExceptionHandler` стоит после `UseRouting` и поэтому не ловит исключения из endpoints; `UseDeveloperExceptionPage` включён всегда, даже в production, что грозит утечкой стека и исходников; `UseStatusCodePages` отсутствует вовсе, отсюда пустые страницы на 404; секреты лежат в файлах, а не в переменных окружения или Key Vault; страница `/Error` радостно показывает `exception.Message` всем посетителям. Ваша задача — за одно домашнее задание привести обработку ошибок и старт приложения в соответствие с best practices урока, подготовив проект, который ведёт себя по-разному в Development, Staging и Production, но при этом остаётся безопасным и информативным в каждой среде. Это не учебная абстракция: ровно так выглядит типичный технический долг в реальных ASP.NET Core проектах, и именно поэтому урок настаивает на строгом порядке middleware и ветвлении по `IWebHostEnvironment`.

#### Что нужно сделать (пошагово)

1. Создайте новый проект MVC с помощью команды `dotnet new mvc -n ErrorHandlingPortal -o ErrorHandlingPortal -f net8.0`. Перейдите в каталог: `cd ErrorHandlingPortal`. Убедитесь, что сборка проходит: `dotnet build`. Ожидаемый вывод — `Build succeeded` с 0 ошибок и 0 предупреждений.

2. Откройте `Program.cs` и приведите его к скелету из урока. Зарегистрируйте сервисы: `builder.Services.AddControllersWithViews();` и обязательно `builder.Services.AddProblemDetails();` для RFC 9457. После `var app = builder.Build();` добавьте ветвление по среде: в `Development` включите `app.UseDeveloperExceptionPage();`, в остальных средах — `app.UseExceptionHandler("/Error");` и `app.UseStatusCodePagesWithReExecute("/Error/Status/{0}");`. Вне Development добавьте `app.UseHsts();`. Затем通用-часть: `UseHttpsRedirection`, `UseStaticFiles`, `UseRouting`, `UseAuthorization`, `MapControllerRoute` с шаблоном по умолчанию и отдельный маршрут для ошибок `pattern: "Error/{action=Status}/{code:int}"` с `defaults: new { controller = "Error" }`.

3. Создайте контроллер `ErrorController.cs` в каталоге `Controllers`. Реализуйте два действия: `Index` (маршрут `/Error`) для необработанных исключений и `Status(int code)` (маршрут `/Error/Status/{code:int}`) для статус-кодов. В `Index` получите `IExceptionHandlerPathFeature` через `HttpContext.Features.Get<IExceptionHandlerPathFeature>()`, вычислите `traceId = Activity.Current?.Id ?? HttpContext.TraceIdentifier`, залогируйте ошибку через `ILogger.LogError` с путём и traceId. Верните `ProblemDetails` с `Title`, `Status = 500`, `Detail`, который зависит от `IWebHostEnvironment.IsDevelopment()` (в Dev — `exception.Message`, в Prod — `null`), и `Extensions["traceId"] = traceId`.

4. Добавьте представление `Views/Error/NotFound.cshtml`, `Views/Error/Forbidden.cshtml` и `Views/Error/Error.cshtml`. На каждой странице выведите дружелюбное сообщение, корреляционный ID (через `ViewData["TraceId"]` или `Context.TraceIdentifier`) и НЕ выводите стек или `Exception.Message` в production-режиме представления.

5. Подготовьте конфигурацию по средам. Создайте `appsettings.Development.json` с локальной строкой подключения ` "ConnectionStrings": { "Default": "Server=(localdb)\\MSSQLLocalDB;Database=Portal_Dev;Trusted_Connection=True;" }` и `appsettings.Production.json` с placeholder-строкой `Server=PROD_DB_HOST;Database=Portal_Prod;User Id=${DB_USER};Password=${DB_PASSWORD};`. В базовом `appsettings.json` оставьте только общие настройки (logging, allowed hosts), а `ConnectionStrings:Default` уберите — пусть его переопределяет среда.

6. Запустите приложение в трёх средах и задокументируйте поведение. Для Development: `dotnet run` (по умолчанию `ASPNETCORE_ENVIRONMENT=Development`). Для Staging: в PowerShell `$env:ASPNETCORE_ENVIRONMENT="Staging"; dotnet run`. Для Production: `$env:ASPNETCORE_ENVIRONMENT="Production"; dotnet run`. Для каждой среды откройте `https://localhost:5xxx/`, `https://localhost:5xxx/Home/Throw` (создайте действие, которое бросает `InvalidOperationException`) и `https://localhost:5xxx/ThisPageDoesNotExist`. Запишите в отчёт: какой HTTP-код, какое тело, меняется ли URL в адресной строке, виден ли стек.

7. Намеренно сломайте порядок middleware в отдельной ветке или копии: поместите `UseExceptionHandler` после `UseRouting` и убедитесь, что исключение из `Home/Throw` больше не перехватывается (вы получаете пустой 500 или Kestrel-ответ). Затем верните правильный порядок и убедитесь, что обработка снова работает. Это упражнение — ключевая точка урока: порядок middleware критичен, и прочувствовать его на себе важнее, чем просто прочитать.

8. Включите проблемные детали в API-сценарии: добавьте контроллер `ApiController` с маршрутом `[Route("api/[controller]")]` и действием, возвращающим `throw new ApplicationException("boom")`. Убедитесь, что благодаря `AddProblemDetails()` ответ приходит в формате RFC 9457 (`application/problem+json`), а не в виде голой строки.

#### Требования к решению

Решение обязано быть проектом на .NET 8 (целевой фреймворк `net8.0`), использующим C# 12: top-level statements в `Program.cs`, pattern matching (включая `switch` выражения и `or`-patterns для 401/403), collection expressions там, где они уместны (например, при формировании списка логов), и raw string literals для многострочных шаблонов сообщений, если такие понадобятся. Все middleware обработки ошибок должны располагаться строго до `UseRouting`; `UseStatusCodePagesWithReExecute` — строго после `UseExceptionHandler`, но до `UseRouting`. Ветвление по среде выполняется через `IWebHostEnvironment` и методы `IsDevelopment()`, `IsStaging()`, `IsProduction()` (или прямые сравнения `EnvironmentName`).

Конфигурация должна загружаться каскадом: базовые значения в `appsettings.json`, переопределения в `appsettings.{Environment}.json`, секреты в production — только через переменные окружения (или имитацию Key Vault), никогда не в файлах. Контроллер ошибок обязан извлекать `IExceptionHandlerPathFeature`, логировать исключение с `TraceIdentifier`/`Activity.Current?.Id`, и не раскрывать стек или `exception.Message` вне Development. Все представления ошибок должны показывать корреляционный ID и дружелюбное сообщение, но не технические детали. В API-части должен быть подключён `AddProblemDetails()`. Код должен компилироваться без предупреждений, а приложение — запускаться во всех трёх средах без ручных правок кода между запусками.

#### Тонкости и подводные камни

Главная тонкость, на которую бьёт урок, — порядок middleware. `UseExceptionHandler` ловит только те исключения, которые возникают «ниже» него в конвейере. Если он стоит после `UseRouting` и `MapControllerRoute`, то исключение из контроллера до него не доходит: оно выбрасывается уже внутри endpoint-инвокера, который сам по себе не оборачивает вызов в try/catch для передачи наверх. Поэтому обработчик исключений всегда должен быть самым первым после `Build()`. Вторая тонкость — `UseDeveloperExceptionPage` в production. Если вы забудете ветвление и оставите её включённой, любой посетитель боевого сайта при сбое увидит стек, исходный код с номерами строк, заголовки запроса и иногда значения локальных переменных. Это и утечка информации, и плохой UX.

Третья подводная камень — разница между 500 (исключение) и 4xx (статус-код без тела). `UseExceptionHandler` реагирует только на исключения; на 404 от несуществующего маршрута он не сработает, потому что никакое исключение не выбрасывается — просто возвращается ответ с пустым телом. Поэтому без `UseStatusCodePagesWithReExecute` пользователь видит пустую страницу. Четвёртая тонкость — выбор между `WithReExecute` и `WithRedirects`. `WithRedirects` выполняет клиентский 302-редирект, из-за чего в адресной строке появляется `/Error/Status/404`, а исходный путь `/ThisPageDoesNotExist` теряется, и в логах вы не увидите, по какому URL ходил пользователь. `WithReExecute` заново прогоняет конвейер по пути `/Error/Status/{0}`, но исходный `Request.Path` и `Query` сохраняются в `IStatusCodeReExecuteFeature`, что критично для аналитики и SEO. Пятая тонкость — `ProblemDetails` и `AddProblemDetails()`: это не просто удобство, а контракт RFC 9457, единый для клиентов API, с полями `type`, `title`, `status`, `detail`, `instance`. Наконец, не путайте `IWebHostEnvironment` (старый, но всё ещё валидный для веб-сценариев) с `IHostEnvironment` — для проверки среды в `Program.cs` подходят оба, но `IWebHostEnvironment` даёт ещё `WebRootPath`, нужный для статических файлов.

#### Критерии приёмки

- [ ] Проект собирается командой `dotnet build` без ошибок и предупреждений.
- [ ] Целевой фреймворк — `net8.0`, используется C# 12 (top-level statements, switch-выражения, collection expressions/raw strings где уместно).
- [ ] В `Program.cs` зарегистрированы `AddControllersWithViews()` и `AddProblemDetails()`.
- [ ] `UseExceptionHandler` / `UseDeveloperExceptionPage` стоят ДО `UseRouting`.
- [ ] Ветвление по среде выполнено через `app.Environment.IsDevelopment()` (и/или `IsStaging`/`IsProduction`).
- [ ] В Development включён `UseDeveloperExceptionPage`, в остальных средах — `UseExceptionHandler("/Error")` + `UseStatusCodePagesWithReExecute("/Error/Status/{0}")`.
- [ ] `UseHsts` включён только вне Development; `UseHttpsRedirection` присутствует всегда.
- [ ] Создан `ErrorController` с действиями `Index` (маршрут `/Error`) и `Status(int code)`.
- [ ] `ErrorController.Index` извлекает `IExceptionHandlerPathFeature` и логирует ошибку с `TraceIdentifier`/`Activity.Current?.Id`.
- [ ] В `Index` `Detail` поля `ProblemDetails` зависит от `IsDevelopment()`: в Dev — сообщение, в Prod — `null`.
- [ ] `Status(int code)` использует switch-выражение с `404`, `401 or 403` и default-веткой.
- [ ] Созданы представления `NotFound`, `Forbidden`, `Error`, не раскрывающие стек и `exception.Message` в production.
- [ ] Созданы `appsettings.Development.json` и `appsettings.Production.json` с разными строками подключения.
- [ ] Базовый `appsettings.json` не содержит production-секретов.
- [ ] Документировано поведение в трёх средах (Development/Staging/Production) для трёх URL: `/`, `/Home/Throw`, `/ThisPageDoesNotExist`.
- [ ] Намеренно нарушенный порядок middleware (`UseExceptionHandler` после `UseRouting`) продемонстрирован и описан в отчёте.
- [ ] API-контроллер возвращает `application/problem+json` благодаря `AddProblemDetails()`.

#### Подсказки (без прямого ответа)

- Вспомните метафору урока про станцию скорой помощи: первая «скорая» — перехватчик исключений, и она должна стоять у входа, а не в конце коридора.
- Если 404 даёт пустую страницу, спросите себя: какое middleware отвечает за статус-коды без тела? И почему URL не должен меняться?
- Чтобы не потерять исходный путь при обработке 404, посмотрите в сторону `IStatusCodeReExecuteFeature` и `OriginalPath`.
- Для корреляции пользователя с логом используйте то, что ASP.NET Core уже генерирует на каждый запрос — посмотрите на `HttpContext.TraceIdentifier` и `Activity.Current?.Id`.
- Если в production всё ещё виден стек, проверьте: точно ли `Detail` зависит от `IsDevelopment()`, и точно ли представление не дёргает `Model.Error.Message`?

#### Эталонное решение (разбор)

```csharp
// Program.cs — C# 12 / .NET 8
// Глобальная обработка ошибок и старт по средам / Global error handling and environment-specific startup
var builder = WebApplication.CreateBuilder(args);

// Сервисы MVC + RFC 9457 Problem Details / MVC services + RFC 9457 Problem Details
builder.Services.AddControllersWithViews();
builder.Services.AddProblemDetails();

var app = builder.Build();

// 1) Перехватчик исключений — САМЫЙ ПЕРВЫЙ / Exception handler — VERY FIRST
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage(); // Dev: подробный стек / Dev: detailed stack
}
else
{
    app.UseExceptionHandler("/Error");               // Prod: reroute to /Error
    app.UseStatusCodePagesWithReExecute("/Error/Status/{0}"); // 4xx/5xx без тела / bodyless 4xx/5xx
}

// 2) HSTS только вне Development / HSTS only outside Development
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

// Маршрут для статус-кодов / Route for status codes
app.MapControllerRoute(
    name: "error",
    pattern: "Error/{action=Status}/{code:int}",
    defaults: new { controller = "Error" });

app.Run();
```

```csharp
// Controllers/ErrorController.cs — C# 12 / .NET 8
public class ErrorController : Controller
{
    private readonly ILogger<ErrorController> _logger;
    private readonly IWebHostEnvironment _env;

    public ErrorController(ILogger<ErrorController> logger, IWebHostEnvironment env)
    {
        _logger = logger;
        _env = env;
    }

    // Необработанные исключения → 500 / Unhandled exceptions → 500
    [Route("/Error")]
    [HttpGet]
    public IActionResult Index()
    {
        var feature = HttpContext.Features.Get<IExceptionHandlerPathFeature>();
        var traceId = Activity.Current?.Id ?? HttpContext.TraceIdentifier;

        if (feature is not null)
        {
            _logger.LogError(feature.Error,
                "Необработанное исключение на пути {Path}. TraceId: {TraceId} / " +
                "Unhandled exception at {Path}. TraceId: {TraceId}",
                feature.Path, traceId);
        }

        var problem = new ProblemDetails
        {
            Title = "Произошла ошибка / An error occurred",
            Status = 500,
            // Dev: детали; Prod: скрыть / Dev: details; Prod: hide
            Detail = _env.IsDevelopment() ? feature?.Error.Message : null,
            Instance = feature?.Path,
        };
        problem.Extensions["traceId"] = traceId;

        return StatusCode(500, problem);
    }

    // Статус-коды 4xx/5xx → дружелюбные страницы / Status codes → friendly pages
    [Route("/Error/Status/{code:int}")]
    public IActionResult Status(int code)
    {
        var reexecute = HttpContext.Features.Get<IStatusCodeReExecuteFeature>();
        _logger.LogWarning(
            "Статус {Code} на пути {OriginalPath}. TraceId: {TraceId} / " +
            "Status {Code} at {OriginalPath}. TraceId: {TraceId}",
            code, reexecute?.OriginalPath ?? HttpContext.Request.Path, HttpContext.TraceIdentifier);

        ViewData["TraceId"] = HttpContext.TraceIdentifier;
        ViewData["Code"] = code;

        return code switch
        {
            404 => View("NotFound"),
            401 or 403 => View("Forbidden"),
            _ => View("Error"),
        };
    }
}
```

Разбор по строкам. Первый блок (`Program.cs`) воспроизводит скелет урока практически дословно, потому что именно порядок — главное. Строка `if (app.Environment.IsDevelopment())` реализует четвёртый инструмент урока — ветвление по `IWebHostEnvironment`. В ветке Development стоит первый инструмент (`UseDeveloperExceptionPage`), в else — второй и третий (`UseExceptionHandler` + `UseStatusCodePagesWithReExecute`). Путь `"/Error/Status/{0}"` — это плейсхолдер, куда подставляется числовой код; `WithReExecute` предпочтительнее `WithRedirects`, потому что сохраняет URL и контекст (это best practice из урока и ответ на частую ошибку «URL меняется, исходный путь теряется»). Блок `if (!app.Environment.IsDevelopment()) app.UseHsts();` реализует последний пункт чек-листа — HSTS только вне Development. Важно, что `UseRouting` идёт строго после всех обработчиков ошибок — это та самая частая ошибка («обработчик после routing не ловит исключения»), которую вы должны прочувствовать в шаге 7. Маршрут `error` позволяет `Status(int code)` принимать код из URL.

Второй блок (`ErrorController`) применяет сразу несколько концепций. `IExceptionHandlerPathFeature` — это feature-интерфейс, который `UseExceptionHandler` кладёт в `HttpContext.Features` после перехвата; из него мы берём `Error` (исключение) и `Path` (где оно случилось). `Activity.Current?.Id ?? HttpContext.TraceIdentifier` — рекомендованный в уроке способ получить корреляционный ID: `Activity.Current` существует, когда включена distributed tracing, иначе fallback на `TraceIdentifier`. Логирование через `ILogger.LogError` с двумя парами плейсхолдеров `{Path}` и `{TraceId}` — структурное, его потом легко искать в логах. Конструкция `Detail = _env.IsDevelopment() ? feature?.Error.Message : null` — прямая реализация best practice «не показывайте стек и `exception.Message` в production»; именно этот тернарный оператор закрывает частую ошибку «в кастомной странице `/Error` показывается `exception.Message` в production». `problem.Extensions["traceId"] = traceId` добавляет корреляционный ID в RFC 9457-ответ, связывая пользователя с логом. В `Status` мы дополнительно достаём `IStatusCodeReExecuteFeature`, чтобы залогировать `OriginalPath` — исходный URL, по которому ходил пользователь до re-execute; без этого в логах был бы только `/Error/Status/404`, что бесполезно для аналитики. Switch-выражение с `401 or 403` — это C# 12 pattern matching, рекомендованный в задании формат; он лаконичнее каскада `if` и хорошо читается.

#### Задания на углубление (бонус)

1. Добавьте Staging-среду как «промежуточную»: в ней должен работать `UseExceptionHandler`, но `Detail` должен содержать сокращённое сообщение (например, только тип исключения, без сообщения), чтобы QA видели больше, чем пользователи, но меньше, чем разработчики. Реализуйте через собственное расширение `IsStaging()` или сравнение `EnvironmentName == "Staging"`.
2. Реализуйте собственный middleware-класс `CorrelationIdMiddleware`, который генерирует или читает заголовок `X-Correlation-Id` и кладёт его в `Activity.Current` и в `HttpContext.Items`, чтобы `ErrorController` использовал единый ID на всём пути запроса.
3. Подключите `IProblemDetailsService` (как в примере урока) и кастомный `IProblemDetailsWriter`, который для API-контроллеров всегда возвращает `application/problem+json`, а для браузерных запросов (по заголовку `Accept: text/html`) рендерит HTML-страницу ошибки.
4. Добавьте интеграционный тест с `WebApplicationFactory<Program>`, который запускает приложение в `Production`-среде, обращается к `/Home/Throw` и проверяет, что ответ содержит `traceId` и НЕ содержит `exception.Message`.

---

## Statement in English / Постановка на английском

#### Context & motivation

You have joined a team maintaining an internal portal built on ASP.NET Core 8 (.NET 8, C# 12). Users complain about two recurring problems. First, whenever a controller throws, they see a blank white page with no explanation, and the support engineers cannot tie a customer complaint to a specific log entry because no correlation identifier appears anywhere. Second, when somebody hits a non-existent URL (for example a typo in a link from a newsletter), the browser again shows nothing, even though the server returns a 404. In parallel, the developers suffer because locally they have a connection string to a local database, but on the production server the application keeps connecting to that same local string, because everything is hardcoded in a single `appsettings.json`.

A code review reveals several problems straight from the lesson checklist: `UseExceptionHandler` sits after `UseRouting`, so it never catches endpoint exceptions; `UseDeveloperExceptionPage` is enabled everywhere, including production, which risks leaking the stack and source code; `UseStatusCodePages` is missing entirely, hence the blank 404 pages; secrets live in files rather than in environment variables or Key Vault; and the `/Error` page happily shows `exception.Message` to every visitor. Your task in this single homework is to bring error handling and startup in line with the lesson's best practices, producing a project that behaves differently in Development, Staging and Production while remaining safe and informative in each environment. This is not an academic abstraction: it is exactly what typical technical debt looks like in real ASP.NET Core projects, and that is precisely why the lesson insists on a strict middleware order and on branching through `IWebHostEnvironment`.

#### What to do step by step

1. Create a new MVC project with the command `dotnet new mvc -n ErrorHandlingPortal -o ErrorHandlingPortal -f net8.0`. Move into the directory: `cd ErrorHandlingPortal`. Verify the build: `dotnet build`. The expected output is `Build succeeded` with 0 errors and 0 warnings.

2. Open `Program.cs` and bring it to the skeleton from the lesson. Register services: `builder.Services.AddControllersWithViews();` and, mandatory, `builder.Services.AddProblemDetails();` for RFC 9457. After `var app = builder.Build();`, add environment branching: in `Development` enable `app.UseDeveloperExceptionPage();`, in all other environments add `app.UseExceptionHandler("/Error");` and `app.UseStatusCodePagesWithReExecute("/Error/Status/{0}");`. Outside Development, add `app.UseHsts();`. Then the common part: `UseHttpsRedirection`, `UseStaticFiles`, `UseRouting`, `UseAuthorization`, `MapControllerRoute` with the default pattern, and a separate route for errors with `pattern: "Error/{action=Status}/{code:int}"` and `defaults: new { controller = "Error" }`.

3. Create an `ErrorController.cs` in the `Controllers` folder. Implement two actions: `Index` (route `/Error`) for unhandled exceptions and `Status(int code)` (route `/Error/Status/{code:int}`) for status codes. In `Index`, obtain `IExceptionHandlerPathFeature` via `HttpContext.Features.Get<IExceptionHandlerPathFeature>()`, compute `traceId = Activity.Current?.Id ?? HttpContext.TraceIdentifier`, and log the error through `ILogger.LogError` with the path and traceId. Return a `ProblemDetails` with `Title`, `Status = 500`, a `Detail` that depends on `IWebHostEnvironment.IsDevelopment()` (Dev — `exception.Message`, Prod — `null`), and `Extensions["traceId"] = traceId`.

4. Add views `Views/Error/NotFound.cshtml`, `Views/Error/Forbidden.cshtml`, and `Views/Error/Error.cshtml`. Each page must show a friendly message and the correlation ID (through `ViewData["TraceId"]` or `Context.TraceIdentifier`) and must NOT show the stack or `Exception.Message` in the production-oriented view.

5. Prepare per-environment configuration. Create `appsettings.Development.json` with a local connection string `"ConnectionStrings": { "Default": "Server=(localdb)\\MSSQLLocalDB;Database=Portal_Dev;Trusted_Connection=True;" }` and `appsettings.Production.json` with a placeholder string `Server=PROD_DB_HOST;Database=Portal_Prod;User Id=${DB_USER};Password=${DB_PASSWORD};`. In the base `appsettings.json`, keep only common settings (logging, allowed hosts) and remove `ConnectionStrings:Default` — let the environment override it.

6. Run the application in three environments and document the behavior. For Development: `dotnet run` (defaults to `ASPNETCORE_ENVIRONMENT=Development`). For Staging: in PowerShell `$env:ASPNETCORE_ENVIRONMENT="Staging"; dotnet run`. For Production: `$env:ASPNETCORE_ENVIRONMENT="Production"; dotnet run`. For each environment open `https://localhost:5xxx/`, `https://localhost:5xxx/Home/Throw` (create an action that throws `InvalidOperationException`), and `https://localhost:5xxx/ThisPageDoesNotExist`. Record in a report: the HTTP code, the body, whether the URL changes in the address bar, and whether the stack is visible.

7. Deliberately break the middleware order in a separate branch or copy: place `UseExceptionHandler` after `UseRouting` and confirm that the exception from `Home/Throw` is no longer caught (you get a blank 500 or a raw Kestrel response). Then restore the correct order and confirm handling works again. This exercise is the heart of the lesson: middleware order is critical, and feeling it personally matters more than just reading about it.

8. Enable problem details in an API scenario: add an `ApiController` with route `[Route("api/[controller]")]` and an action that does `throw new ApplicationException("boom")`. Confirm that, thanks to `AddProblemDetails()`, the response comes back in RFC 9457 format (`application/problem+json`) rather than as a bare string.

#### Requirements

The solution must be a .NET 8 project (target framework `net8.0`) using C# 12: top-level statements in `Program.cs`, pattern matching (including switch expressions and `or`-patterns for 401/403), collection expressions where appropriate (for example when assembling a list of log entries), and raw string literals for multi-line message templates where they help. All error-handling middleware must sit strictly before `UseRouting`; `UseStatusCodePagesWithReExecute` must sit strictly after `UseExceptionHandler` but before `UseRouting`. Environment branching must go through `IWebHostEnvironment` and the methods `IsDevelopment()`, `IsStaging()`, `IsProduction()` (or direct `EnvironmentName` comparisons).

Configuration must load as a cascade: base values in `appsettings.json`, overrides in `appsettings.{Environment}.json`, and production secrets only through environment variables (or a Key Vault simulation) — never in files. The error controller must extract `IExceptionHandlerPathFeature`, log the exception with `TraceIdentifier`/`Activity.Current?.Id`, and never expose the stack or `exception.Message` outside Development. All error views must show the correlation ID and a friendly message, but no technical details. The API part must have `AddProblemDetails()` registered. The code must compile without warnings, and the application must start in all three environments without manual code edits between runs.

#### Pitfalls

The main pitfall the lesson hammers on is middleware order. `UseExceptionHandler` only catches exceptions thrown “below” it in the pipeline. If it sits after `UseRouting` and `MapControllerRoute`, the controller exception never reaches it: it is thrown inside the endpoint invoker, which itself does not wrap the call in try/catch to propagate upward. That is why the exception handler must always be the very first thing after `Build()`. The second pitfall is `UseDeveloperExceptionPage` in production. If you forget the branching and leave it on, any visitor to the live site will see the stack trace, source code with line numbers, request headers, and sometimes local variable values — an information leak and a poor UX.

The third pitfall is the difference between a 500 (an exception) and a 4xx (a status code with no body). `UseExceptionHandler` reacts only to exceptions; on a 404 from a non-existent route it does nothing, because no exception is thrown — a response with an empty body simply comes back. Without `UseStatusCodePagesWithReExecute`, the user therefore sees a blank page. The fourth pitfall is the choice between `WithReExecute` and `WithRedirects`. `WithRedirects` issues a client-side 302, so the address bar changes to `/Error/Status/404` and the original path `/ThisPageDoesNotExist` is lost; in the logs you will not see which URL the user actually visited. `WithReExecute` re-runs the pipeline on `/Error/Status/{0}` but keeps the original `Request.Path` and `Query` inside `IStatusCodeReExecuteFeature`, which is critical for analytics and SEO. The fifth pitfall is `ProblemDetails` and `AddProblemDetails()`: this is not just convenience but an RFC 9457 contract, uniform for API clients, with the fields `type`, `title`, `status`, `detail`, `instance`. Finally, do not confuse `IWebHostEnvironment` (older, but still valid for web scenarios) with `IHostEnvironment` — both work for environment checks in `Program.cs`, but `IWebHostEnvironment` also exposes `WebRootPath`, which you need for static files.

#### Acceptance criteria

- [ ] The project builds with `dotnet build` with no errors and no warnings.
- [ ] The target framework is `net8.0` and the code uses C# 12 (top-level statements, switch expressions, collection expressions / raw strings where appropriate).
- [ ] `Program.cs` registers `AddControllersWithViews()` and `AddProblemDetails()`.
- [ ] `UseExceptionHandler` / `UseDeveloperExceptionPage` are placed BEFORE `UseRouting`.
- [ ] Environment branching uses `app.Environment.IsDevelopment()` (and/or `IsStaging`/`IsProduction`).
- [ ] Development enables `UseDeveloperExceptionPage`; all other environments enable `UseExceptionHandler("/Error")` + `UseStatusCodePagesWithReExecute("/Error/Status/{0}")`.
- [ ] `UseHsts` is enabled only outside Development; `UseHttpsRedirection` is always present.
- [ ] An `ErrorController` exists with actions `Index` (route `/Error`) and `Status(int code)`.
- [ ] `ErrorController.Index` extracts `IExceptionHandlerPathFeature` and logs the error with `TraceIdentifier`/`Activity.Current?.Id`.
- [ ] In `Index`, the `Detail` field of `ProblemDetails` depends on `IsDevelopment()`: Dev — the message, Prod — `null`.
- [ ] `Status(int code)` uses a switch expression with `404`, `401 or 403`, and a default branch.
- [ ] Views `NotFound`, `Forbidden`, `Error` exist and do not expose the stack or `exception.Message` in production.
- [ ] `appsettings.Development.json` and `appsettings.Production.json` exist with different connection strings.
- [ ] The base `appsettings.json` contains no production secrets.
- [ ] Behavior in three environments (Development/Staging/Production) is documented for three URLs: `/`, `/Home/Throw`, `/ThisPageDoesNotExist`.
- [ ] A deliberately broken middleware order (`UseExceptionHandler` after `UseRouting`) is demonstrated and described in the report.
- [ ] The API controller returns `application/problem+json` thanks to `AddProblemDetails()`.

#### Hints

- Recall the lesson's emergency-room metaphor: the first “ambulance” is the exception catcher, and it must stand at the entrance, not at the end of the corridor.
- If a 404 gives a blank page, ask yourself: which middleware is responsible for bodyless status codes, and why should the URL not change?
- To avoid losing the original path on a 404, look toward `IStatusCodeReExecuteFeature` and `OriginalPath`.
- For user-to-log correlation, use what ASP.NET Core already generates per request — look at `HttpContext.TraceIdentifier` and `Activity.Current?.Id`.
- If the stack still shows in production, double-check: does `Detail` really depend on `IsDevelopment()`, and does the view really not pull `Model.Error.Message`?

#### Reference solution walk-through

```csharp
// Program.cs — C# 12 / .NET 8
// Global error handling and environment-specific startup
var builder = WebApplication.CreateBuilder(args);

// MVC services + RFC 9457 Problem Details
builder.Services.AddControllersWithViews();
builder.Services.AddProblemDetails();

var app = builder.Build();

// 1) Exception handler — VERY FIRST in the pipeline
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage(); // Dev: detailed stack
}
else
{
    app.UseExceptionHandler("/Error");               // Prod: reroute to /Error
    app.UseStatusCodePagesWithReExecute("/Error/Status/{0}"); // bodyless 4xx/5xx
}

// 2) HSTS only outside Development
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

// Route for status codes
app.MapControllerRoute(
    name: "error",
    pattern: "Error/{action=Status}/{code:int}",
    defaults: new { controller = "Error" });

app.Run();
```

```csharp
// Controllers/ErrorController.cs — C# 12 / .NET 8
public class ErrorController : Controller
{
    private readonly ILogger<ErrorController> _logger;
    private readonly IWebHostEnvironment _env;

    public ErrorController(ILogger<ErrorController> logger, IWebHostEnvironment env)
    {
        _logger = logger;
        _env = env;
    }

    // Unhandled exceptions → 500
    [Route("/Error")]
    [HttpGet]
    public IActionResult Index()
    {
        var feature = HttpContext.Features.Get<IExceptionHandlerPathFeature>();
        var traceId = Activity.Current?.Id ?? HttpContext.TraceIdentifier;

        if (feature is not null)
        {
            _logger.LogError(feature.Error,
                "Unhandled exception at {Path}. TraceId: {TraceId}",
                feature.Path, traceId);
        }

        var problem = new ProblemDetails
        {
            Title = "An error occurred",
            Status = 500,
            // Dev: details; Prod: hide
            Detail = _env.IsDevelopment() ? feature?.Error.Message : null,
            Instance = feature?.Path,
        };
        problem.Extensions["traceId"] = traceId;

        return StatusCode(500, problem);
    }

    // Status codes 4xx/5xx → friendly pages
    [Route("/Error/Status/{code:int}")]
    public IActionResult Status(int code)
    {
        var reexecute = HttpContext.Features.Get<IStatusCodeReExecuteFeature>();
        _logger.LogWarning(
            "Status {Code} at {OriginalPath}. TraceId: {TraceId}",
            code, reexecute?.OriginalPath ?? HttpContext.Request.Path, HttpContext.TraceIdentifier);

        ViewData["TraceId"] = HttpContext.TraceIdentifier;
        ViewData["Code"] = code;

        return code switch
        {
            404 => View("NotFound"),
            401 or 403 => View("Forbidden"),
            _ => View("Error"),
        };
    }
}
```

Walk-through. The first block (`Program.cs`) reproduces the lesson skeleton almost verbatim, because the order is the whole point. The line `if (app.Environment.IsDevelopment())` implements the fourth lesson tool — branching through `IWebHostEnvironment`. The Development branch holds the first tool (`UseDeveloperExceptionPage`); the else branch holds the second and third (`UseExceptionHandler` + `UseStatusCodePagesWithReExecute`). The path `"/Error/Status/{0}"` is a placeholder where the numeric status code is substituted; `WithReExecute` is preferred over `WithRedirects` because it preserves the URL and the context — this is the lesson best practice and the answer to the common mistake “the URL changes and the original path is lost.” The block `if (!app.Environment.IsDevelopment()) app.UseHsts();` implements the last checklist item — HSTS only outside Development. Crucially, `UseRouting` comes strictly after all error handlers — this is exactly the common mistake (“the handler after routing does not catch exceptions”) that you are asked to feel in step 7. The `error` route lets `Status(int code)` receive the code from the URL.

The second block (`ErrorController`) applies several concepts at once. `IExceptionHandlerPathFeature` is a feature interface that `UseExceptionHandler` puts into `HttpContext.Features` after catching; from it we take `Error` (the exception) and `Path` (where it happened). `Activity.Current?.Id ?? HttpContext.TraceIdentifier` is the lesson-recommended way to obtain a correlation ID: `Activity.Current` exists when distributed tracing is on, otherwise we fall back to `TraceIdentifier`. Logging through `ILogger.LogError` with the placeholder pairs `{Path}` and `{TraceId}` is structured logging — easy to search later. The construct `Detail = _env.IsDevelopment() ? feature?.Error.Message : null` is a direct implementation of the best practice “never expose the stack or `exception.Message` in production”; this ternary closes the common mistake “the custom `/Error` page shows `exception.Message` in production.” The line `problem.Extensions["traceId"] = traceId` adds the correlation ID to the RFC 9457 response, linking the user to the log entry. In `Status` we additionally pull `IStatusCodeReExecuteFeature` so we can log `OriginalPath` — the URL the user actually visited before the re-execute; without it the log would only show `/Error/Status/404`, which is useless for analytics. The switch expression with `401 or 403` is C# 12 pattern matching, the format recommended in the assignment; it is more concise than a cascade of `if` statements and reads well.

#### Going deeper (bonus)

1. Add a Staging environment as an “intermediate” tier: it should run `UseExceptionHandler`, but `Detail` should carry a shortened message (for example only the exception type, without the message), so QA see more than users but less than developers. Implement it through a custom `IsStaging()` extension or an `EnvironmentName == "Staging"` comparison.
2. Implement a custom `CorrelationIdMiddleware` class that generates or reads an `X-Correlation-Id` header and stores it in `Activity.Current` and `HttpContext.Items`, so `ErrorController` uses a single ID across the whole request path.
3. Wire up `IProblemDetailsService` (as in the lesson example) and a custom `IProblemDetailsWriter` that always returns `application/problem+json` for API controllers but renders an HTML error page for browser requests (by the `Accept: text/html` header).
4. Add an integration test with `WebApplicationFactory<Program>` that runs the app in the `Production` environment, hits `/Home/Throw`, and asserts that the response contains `traceId` and does NOT contain `exception.Message`.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `ErrorHandlingPortal` собирается без ошибок и предупреждений (RU).
- [ ] В `Program.cs` реализован скелет из урока с правильным порядком middleware (RU).
- [ ] `ErrorController` содержит `Index` и `Status`, извлекает `IExceptionHandlerPathFeature` (RU).
- [ ] Созданы `appsettings.Development.json` и `appsettings.Production.json` (RU).
- [ ] Документировано поведение в трёх средах и продемонстрирован нарушенный порядок middleware (RU).
- [ ] The `ErrorHandlingPortal` project builds with no errors or warnings (EN).
- [ ] `Program.cs` reproduces the lesson skeleton with the correct middleware order (EN).
- [ ] `ErrorController` has `Index` and `Status`, extracts `IExceptionHandlerPathFeature` (EN).
- [ ] `appsettings.Development.json` and `appsettings.Production.json` are created (EN).
- [ ] Behavior in three environments is documented and a broken middleware order is demonstrated (EN).

#### Ресурсы / Resources
- [Microsoft Learn — Handle errors in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/error-handling)
- [Microsoft Learn — Use multiple environments in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/environments)
- [Microsoft Learn — Status code pages](https://learn.microsoft.com/aspnet/core/fundamentals/error-handling#usestatuscodepages)
- [Microsoft Learn — Problem Details](https://learn.microsoft.com/aspnet/core/web-api/handle-errors#problem-details)
- [RFC 9457 — Problem Details for HTTP APIs](https://datatracker.ietf.org/doc/html/rfc9457)
- [Урок M13-L09](lesson-M13-L09-error-handling-environments.md)
