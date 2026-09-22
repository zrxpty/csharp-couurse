---
[← К уроку M13-L02](lesson-M13-L02-middleware-pipeline.md) | [⬆ К модулю M13](../README.md) | [Следующее ДЗ →](homework-M13-L03-routing-endpoints.md)
---

### Домашнее задание M13-L02: Middleware pipeline, Use/Run/Map, порядок / Homework M13-L02: Middleware pipeline, Use/Run/Map, order

**Урок / Lesson:** M13-L02
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться собирать конвейер middleware в ASP.NET Core 8, используя `Use`, `Run`, `Map`, `MapWhen` и `UseWhen`, корректно управлять порядком компонентов, писать собственные middleware-классы с `InvokeAsync` и отлаживать типичные ошибки порядка и ветвления. (EN) Learn to assemble the ASP.NET Core 8 middleware pipeline using `Use`, `Run`, `Map`, `MapWhen`, and `UseWhen`, correctly control component ordering, write custom middleware classes with `InvokeAsync`, and debug typical ordering and branching mistakes.

#### Связь с уроком / Connection to the lesson
(RU) Урок M13-L02 показывает, что middleware — это двунаправленная «матрёшка»: каждый компонент работает до и после `await next()`, а порядок регистрации определяет, кто первым ловит исключения, кто первым видит запрос и кто может прервать цепочку. ДЗ закрепляет это на практике: вы построите реальный конвейер с обработкой исключений, логированием, условным ветвлением по методу и пути, и собственной авторизационной проверкой, которая умеет коротко замыкать запрос.
(EN) Lesson M13-L02 shows that middleware is a bidirectional "matryoshka doll": each component runs both before and after `await next()`, and registration order decides who catches exceptions first, who sees the request first, and who can short-circuit the chain. This homework cements that idea in practice: you will build a real pipeline with exception handling, logging, conditional branching by method and path, and a custom authorization check that can short-circuit the request.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы присоединились к команде, которая разрабатывает внутренний API-шлюз на ASP.NET Core 8 (.NET 8, C# 12). Существующий код навалил всю логику в один огромный контроллер, обработка ошибок сделана «на коленке» через `try/catch` в каждом действии, логирование дублируется, а авторизация выполнена атрибутом `[Authorize]`, который то срабатывает, то нет, потому что порядок middleware в `Program.cs` перепутан. Ваш технический лид просит вас навести порядок в конвейере обработки запроса: вынести сквозные заботы (cross-cutting concerns) в middleware, исправить порядок, добавить ветвление для `/api` и `/admin`, обеспечить наблюдаемость (observability) через correlation ID и заголовки таймингов, а также написать собственный middleware-класс для проверки API-ключа, который умеет коротко замыкать запрос при отсутствии ключа.

Эта задача идеально укладывается в темы урока: `Use` для сквозных компонентов, `Run` для терминальной обработки, `Map` для изоляции префиксов URL, `MapWhen`/`UseWhen` для условного поведения, и критичность порядка (исключения → статика → маршрутизация → аутентификация → авторизация → endpoints). Вы не просто повторяете пример из урока, а строите конвейер, в котором каждая концепция применяется осмысленно и проверяется запросами через `curl` и `http-repl`.

#### Что нужно сделать (пошагово)
1. Создайте новый проект: `dotnet new web -n MiddlewareLab -o MiddlewareLab` и перейдите в папку: `cd MiddlewareLab`. Откройте `Program.cs` — здесь будет весь код (top-level statements, .NET 8 minimal hosting model).
2. Установите пакеты для логирования и тестов (опционально, но полезно): `dotnet add package Serilog.Extensions.Logging` и `dotnet add package Microsoft.AspNetCore.Mvc.Testing` для последующего интеграционного тестирования.
3. Реализуйте конвейер строго в следующем порядке: (a) `UseExceptionHandler` — первым; (b) собственный middleware `RequestLoggingMiddleware` через класс с `InvokeAsync` — измеряет длительность и пишет заголовки `X-Request-Started` и `X-Request-Ms`; (c) собственный middleware `CorrelationIdMiddleware` — генерирует или читает `X-Correlation-Id` и сохраняет его в `Items` и в заголовок ответа; (d) `UseWhen` для POST-запросов, добавляющий заголовок `X-Post-Processed: true`; (e) `Map("/api", ...)` с отдельным конвейером, в котором работает `ApiKeyMiddleware` (проверяет заголовок `X-Api-Key: secret` и при отсутствии возвращает 401 и коротко замыкает запрос), а затем терминальный `Run`, возвращающий JSON `{"status":"ok","correlationId":"..."}`; (f) `Map("/admin", ...)` с конвейером, который возвращает 403 для всех не-POST запросов; (g) финальный `app.Run` в основном конвейере, возвращающий текст «Main pipeline».
4. Создайте класс `ApiKeyMiddleware` в файле `Middleware/ApiKeyMiddleware.cs`: конструктор принимает `RequestDelegate _next`; метод `InvokeAsync(HttpContext context)` проверяет заголовок; если ключа нет или он неверный — `context.Response.StatusCode = 401; await context.Response.WriteAsync("Unauthorized"); return;` (короткое замыкание без вызова `_next`). Добавьте метод расширения `UseApiKey()` в `Middleware/ApiKeyExtensions.cs`.
5. Аналогично создайте `RequestLoggingMiddleware` и `CorrelationIdMiddleware` как отдельные классы с методами расширения. Логирование должно использовать `ILogger<RequestLoggingMiddleware>` через DI — внедрите его через конструктор (это работает, потому что middleware-классы поддерживают scoped-сервисы из `HttpContext.RequestServices`).
6. Запустите приложение: `dotnet run`. Запишите URL из вывода (например, `http://localhost:5000`).
7. Протестируйте запросами (используйте `curl` или `http-repl`):
   - `curl -i http://localhost:5000/` → ожидается «Main pipeline», заголовки `X-Request-Started`, `X-Request-Ms`, `X-Correlation-Id`.
   - `curl -i http://localhost:5000/api` без ключа → ожидается 401 «Unauthorized», заголовки логирования и correlation всё равно присутствуют (потому что они стоят раньше ветви).
   - `curl -i -H "X-Api-Key: secret" http://localhost:5000/api` → ожидается JSON `{"status":"ok","correlationId":"..."}` и заголовок `X-Api-Version: 1.0`.
   - `curl -i -X POST http://localhost:5000/` → ожидается «Main pipeline» и заголовок `X-Post-Processed: true`.
   - `curl -i http://localhost:5000/admin` (GET) → ожидается 403.
   - `curl -i -X POST http://localhost:5000/admin` → ожидается 200 и корректный ответ.
8. Намеренно сломайте порядок: поменяйте местами `CorrelationIdMiddleware` и `RequestLoggingMiddleware`, чтобы логирование стояло раньше correlation — посмотрите, как заголовок `X-Correlation-Id` перестанет появляться в логах «до next». Затем верните правильный порядок.
9. Намеренно зарегистрируйте что-нибудь после финального `app.Run(...)` и убедитесь, что это «что-то» никогда не выполняется (добавьте `app.Use(...)` после `Run` и проверьте отсутствие эффекта) — это демонстрирует, что `Run` терминальный.
10. Напишите один интеграционный тест с `WebApplicationFactory<Program>`, который проверяет, что запрос к `/api` без ключа возвращает 401, а с ключом — 200. Используйте `dotnet new xunit -n MiddlewareLab.Tests`.

#### Требования к решению
- Целевой фреймворк `net8.0`, язык C# 12: используйте top-level statements в `Program.cs`, pattern matching (например, `is null` / `is { Length: > 0 }`), collection expressions там, где строите списки заголовков, raw string literals `"""..."""` для JSON-ответов.
- Каждый собственный middleware оформлен как отдельный класс с `InvokeAsync(HttpContext context)` и конструктором `RequestDelegate next`. Метод расширения `UseXxx` живёт в статическом классе `Middleware/...Extensions.cs`.
- Порядок регистрации в `Program.cs` строго соответствует: `UseExceptionHandler` → `UseRequestLogging` → `UseCorrelationId` → `UseWhen(POST)` → `Map("/api")` → `Map("/admin")` → финальный `Run`. Любая перестановка должна быть обоснована комментарием.
- `ApiKeyMiddleware` корректно коротко замыкает запрос: при ошибке он НЕ вызывает `_next`, ставит статус 401 и пишет тело. При успехе — вызывает `await _next(context)`.
- Логирование и correlation ID работают на «выходе» (после `await next()`), как в примере урока с `Stopwatch`.
- Все компоненты-`Use` обязательно вызывают `await next()`, кроме случаев намеренного короткого замыкания (и это явно закомментировано).
- Код компилируется без предупреждений, `dotnet build` проходит чисто, `dotnet run` запускается без ошибок.
- Интеграционный тест проходит: `dotnet test` зелёный.

#### Тонкости и подводные камни
- **Порядок критичен.** Как подчёркнуто в уроке, `UseExceptionHandler` должен стоять первым — иначе исключения из более ранних middleware не будут обработаны. `UseAuthentication` до `UseAuthorization`. `UseRouting` до `UseEndpoints`. В нашем лабе нет routing/auth-сервисов, но порядок correlation → logging тоже важен для наблюдаемости.
- **`Run` терминальный.** Всё, что добавлено после `Run`, не выполнится. Не ставьте middleware «на всякий случай» после терминального обработчика — это частая ошибка из урока.
- **Запись в `Response.Body` после `await next()`.** Если downstream уже начал отправку (`Response.HasStarted == true`), повторная запись в тело или изменение заголовков выбросит `InvalidOperationException`. В логировании мы пишем только заголовок `X-Request-Ms` после `next` — это безопасно, потому что заголовки ещё не зафиксированы, если downstream не начал запись. Но если ваш downstream пишет в тело до того, как вы поставили заголовок, используйте `context.Response.Headers` до `next` или проверяйте `HasStarted`.
- **`MapWhen` vs `UseWhen`.** `MapWhen` создаёт отдельную ветвь и не возвращает запрос в основной конвейер; `UseWhen` добавляет middleware условно, но запрос возвращается в основной поток. В лабе для POST-заголовка мы используем `UseWhen`, потому что POST `/` должен дойти до финального `Run` основного конвейера. Если ошибочно взять `MapWhen`, запрос «застрянет» в ветви и не дойдёт до `app.Run`.
- **Забытый `await` перед `next()`.** Без `await` pipeline «молча ломается»: следующий middleware начнёт выполняться, но текущий не дождётся и продолжит сразу, портя порядок. Всегда `await next()`.
- **DI в middleware-классе.** Сервисы, внедряемые через конструктор middleware-класса, должны быть singleton или transient; scoped-сервисы берите через `InvokeAsync(HttpContext, IServiceProvider)` или `RequestServices`. В лабе `ILogger<T>` — singleton, его можно брать через конструктор.
- **`Map` и базовый путь.** Внутри ветви `Map("/api")` свойство `context.Request.PathBase` становится `/api`, а `Path` — остатком. Терминальный `Run` внутри ветви видит уже обрезанный путь.

#### Критерии приёмки
- [ ] Проект `MiddlewareLab` создан, `dotnet build` проходит без ошибок и предупреждений.
- [ ] `Program.cs` использует top-level statements и C# 12 (raw strings для JSON, pattern matching).
- [ ] `UseExceptionHandler` зарегистрирован первым.
- [ ] `RequestLoggingMiddleware` — отдельный класс с `InvokeAsync` и `UseRequestLogging`-расширением; ставит `X-Request-Started` до `next` и `X-Request-Ms` после.
- [ ] `CorrelationIdMiddleware` генерирует/читает `X-Correlation-Id`, пишет его в `Items` и в заголовок ответа.
- [ ] `UseWhen` для POST корректно добавляет `X-Post-Processed` и НЕ отрывает запрос от основного конвейера.
- [ ] `Map("/api")` содержит `ApiKeyMiddleware` и терминальный `Run` с JSON-ответом.
- [ ] `ApiKeyMiddleware` возвращает 401 без ключа и 200 с ключом `X-Api-Key: secret`.
- [ ] `Map("/admin")` возвращает 403 для не-POST и 200 для POST.
- [ ] Финальный `app.Run` отдаёт «Main pipeline» для корневого пути.
- [ ] Ничего не зарегистрировано после финального `Run`.
- [ ] Все `Use` вызывают `await next()` (кроме намеренного короткого замыкания, с комментарием).
- [ ] `curl`-проверки из шага 7 дают ожидаемые статусы и заголовки.
- [ ] Интеграционный тест через `WebApplicationFactory<Program>` проходит зелёным.

#### Подсказки (без прямого ответа)
- Для correlation ID используйте `Guid.NewGuid().ToString("N")` и проверяйте `context.Request.Headers["X-Correlation-Id"]` на наличие.
- В `RequestLoggingMiddleware` берите `ILogger<RequestLoggingMiddleware>` через конструктор и логируйте метод, путь и длительность на «выходе».
- Для `Map("/admin")` с проверкой метода можно использовать `MapWhen` внутри ветви, либо просто `if (context.Request.Method != "POST")` в первом middleware ветви.
- Чтобы `WebApplicationFactory<Program>` работал с top-level statements, добавьте `public partial class Program { }` в конец `Program.cs` — это делает тип видимым для тестов.
- Для интеграционного теста используйте `HttpClient client = factory.CreateClient()` и `var resp = await client.GetAsync("/api")`.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — Middleware pipeline: Use / Run / Map / UseWhen + custom middleware classes.
// Двуязычные комментарии RU+EN.

var builder = WebApplication.CreateBuilder(args);
builder.Logging.ClearProviders();
builder.Logging.AddSimpleConsole(o => o.SingleLine = true);

var app = builder.Build();

// 1) Обработка исключений — первой, как в уроке.
//    Exception handling first, exactly as in the lesson.
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        context.Response.StatusCode = 500;
        await context.Response.WriteAsync("Internal Server Error / Внутренняя ошибка");
    });
});

// 2) Логирование запроса — отдельный класс с InvokeAsync.
//    Request logging — dedicated class with InvokeAsync.
app.UseRequestLogging();

// 3) Correlation ID — до бизнес-логики, чтобы все логировали его.
//    Correlation ID — before business logic so everyone can log it.
app.UseCorrelationId();

// 4) Условный middleware без ветвления для POST.
//    Conditional middleware without branching for POST.
app.UseWhen(
    context => context.Request.Method == "POST",
    subApp => subApp.Use(async (context, next) =>
    {
        context.Response.Headers["X-Post-Processed"] = "true";
        await next();
    }));

// 5) Ветвь /api: проверка API-ключа + терминальный JSON.
//    Branch /api: API key check + terminal JSON.
app.Map("/api", apiApp =>
{
    apiApp.Use(async (context, next) =>
    {
        context.Response.Headers["X-Api-Version"] = "1.0";
        await next();
    });
    apiApp.UseApiKey();   // короткое замыкание при отсутствии ключа / short-circuit on missing key
    apiApp.Run(async context =>
    {
        context.Response.ContentType = "application/json";
        var correlationId = context.Items["CorrelationId"] as string ?? "unknown";
        // Raw string literal для JSON — идиома C# 11+.
        // Raw string literal for JSON — C# 11+ idiom.
        await context.Response.WriteAsync($$"""{"status":"ok","correlationId":"{{correlationId}}"}""");
    });
});

// 6) Ветвь /admin: только POST.
//    Branch /admin: POST only.
app.Map("/admin", adminApp =>
{
    adminApp.Use(async (context, next) =>
    {
        if (context.Request.Method != "POST")
        {
            context.Response.StatusCode = 403;
            await context.Response.WriteAsync("Forbidden: POST required");
            return; // короткое замыкание / short-circuit
        }
        await next();
    });
    adminApp.Run(async context =>
    {
        await context.Response.WriteAsync("Admin area OK");
    });
});

// 7) Терминальный обработчик основного конвейера.
//    Terminal handler of the main pipeline.
app.Run(async context =>
{
    await context.Response.WriteAsync("Main pipeline / Основной конвейер");
});

// app.Use(...) сюда писать бесполезно — после Run не выполняется.
// app.Use(...) here is useless — nothing runs after Run.
app.Run();

public partial class Program { } // для WebApplicationFactory / for WebApplicationFactory
```

**Разбор по строкам.** Строка `app.UseExceptionHandler` стоит первой — это прямо требование урока: обработчик исключений должен быть «почти в самом начале», чтобы ловить ошибки из всех последующих компонентов. `UseRequestLogging()` и `UseCorrelationId()` — собственные middleware-классы с методами расширения, как рекомендует best practice урока: «выносите сложную логику в отдельные классы с `InvokeAsync`». Порядок logging → correlation выбран так, чтобы correlation ID уже существовал, когда логирование пишет «выходную» строку (если correlation стоит позже, лог «до next» не будет содержать ID). `UseWhen` для POST — это именно тот случай из урока: «условное добавление middleware без создания ветви», и запрос возвращается в основной конвейер, поэтому POST `/` доходит до финального `app.Run`. `Map("/api", ...)` создаёт отдельный конвейер: внутри него `UseApiKey()` может коротко замыкать (`return` без `_next`), а терминальный `Run` отдаёт JSON. Raw string literal `$$"""..."""` с интерполяцией — идиома C# 11+, корректная для .NET 8. `Map("/admin")` демонстрирует короткое замыкание внутри ветви: первый middleware проверяет метод и при ошибке `return` без `next`. Финальный `app.Run(...)` закрывает основной конвейер; всё после него (как и комментарий ниже) — мёртвый код, что иллюстрирует терминальность `Run`. `public partial class Program { }` делает тип `Program` видимым для `WebApplicationFactory` в интеграционных тестах — обязательный трюк при top-level statements.

#### Задания на углубление (бонус)
1. Добавьте `UseStatusCodePages` или собственный middleware, который перехватывает 404 и отдаёт кастомную JSON-ошибку. Подумайте, где в порядке он должен стоять.
2. Реализуйте middleware для rate limiting «руками» (без встроенного `AddRateLimiter`): храните счётчик запросов per IP в `IMemoryCache` и возвращайте 429 при превышении. Где в порядке его ставить — до или после correlation ID?
3. Перепишите `ApiKeyMiddleware` так, чтобы ключи хранились в `IConfiguration` по префиксу `ApiKeys:`, и один ключ работал для `/api`, другой — для `/admin`. Используйте scoped-сервис `IConfiguration` через `RequestServices`.
4. Добавьте `MapWhen` для запросов с заголовком `X-Debug: true`, который добавляет детальное логирование стека запроса. Сравните поведение с `UseWhen` — почему в этом случае `MapWhen` может быть опаснее?

---

## Statement in English / Постановка на английском

#### Context and motivation
You have joined a team building an internal API gateway on ASP.NET Core 8 (.NET 8, C# 12). The existing code crammed everything into one giant controller, error handling was bolted on with `try/catch` in every action, logging is duplicated all over, and authorization relies on an `[Authorize]` attribute that sometimes works and sometimes does not — because the middleware order in `Program.cs` is wrong. Your tech lead asks you to clean up the request-handling pipeline: extract the cross-cutting concerns into middleware, fix the ordering, add branching for `/api` and `/admin`, ensure observability through a correlation ID and timing headers, and write a custom middleware class that validates an API key and can short-circuit the request when the key is missing.

This task maps cleanly onto the lesson topics: `Use` for cross-cutting components, `Run` for terminal handling, `Map` to isolate URL prefixes, `MapWhen`/`UseWhen` for conditional behavior, and the criticality of order (exceptions → static files → routing → authentication → authorization → endpoints). You are not just copying the lesson example; you are building a pipeline in which every concept is applied deliberately and verified with `curl` and `http-repl` requests.

#### What to do step by step
1. Create a new project: `dotnet new web -n MiddlewareLab -o MiddlewareLab`, then enter the folder: `cd MiddlewareLab`. Open `Program.cs` — all your code goes here (top-level statements, .NET 8 minimal hosting model).
2. Add logging and testing packages (optional but recommended): `dotnet add package Serilog.Extensions.Logging` and `dotnet add package Microsoft.AspNetCore.Mvc.Testing` for later integration testing.
3. Build the pipeline strictly in this order: (a) `UseExceptionHandler` first; (b) a custom `RequestLoggingMiddleware` class with `InvokeAsync` that measures duration and sets `X-Request-Started` and `X-Request-Ms` headers; (c) a custom `CorrelationIdMiddleware` that generates or reads `X-Correlation-Id` and stores it in `Items` and in a response header; (d) `UseWhen` for POST requests adding the header `X-Post-Processed: true`; (e) `Map("/api", ...)` with a separate pipeline that runs `ApiKeyMiddleware` (checks header `X-Api-Key: secret` and returns 401, short-circuiting, when absent) and then a terminal `Run` returning JSON `{"status":"ok","correlationId":"..."}`; (f) `Map("/admin", ...)` that returns 403 for any non-POST request; (g) a final `app.Run` in the main pipeline returning the text "Main pipeline".
4. Create the `ApiKeyMiddleware` class in `Middleware/ApiKeyMiddleware.cs`: the constructor takes a `RequestDelegate _next`; the `InvokeAsync(HttpContext context)` method checks the header; if the key is missing or wrong, set `context.Response.StatusCode = 401; await context.Response.WriteAsync("Unauthorized"); return;` (short-circuit without calling `_next`). Add an extension method `UseApiKey()` in `Middleware/ApiKeyExtensions.cs`.
5. Likewise create `RequestLoggingMiddleware` and `CorrelationIdMiddleware` as separate classes with extension methods. Logging should use `ILogger<RequestLoggingMiddleware>` through DI — inject it via the constructor (this works because middleware classes support scoped services resolved from `HttpContext.RequestServices`).
6. Run the application: `dotnet run`. Note the URL from the output (e.g., `http://localhost:5000`).
7. Test with requests (use `curl` or `http-repl`):
   - `curl -i http://localhost:5000/` → expect "Main pipeline", with `X-Request-Started`, `X-Request-Ms`, `X-Correlation-Id` headers.
   - `curl -i http://localhost:5000/api` without a key → expect 401 "Unauthorized", with logging and correlation headers still present (because they sit before the branch).
   - `curl -i -H "X-Api-Key: secret" http://localhost:5000/api` → expect JSON `{"status":"ok","correlationId":"..."}` and the `X-Api-Version: 1.0` header.
   - `curl -i -X POST http://localhost:5000/` → expect "Main pipeline" and the `X-Post-Processed: true` header.
   - `curl -i http://localhost:5000/admin` (GET) → expect 403.
   - `curl -i -X POST http://localhost:5000/admin` → expect 200 and a valid response.
8. Deliberately break the order: swap `CorrelationIdMiddleware` and `RequestLoggingMiddleware` so logging runs before correlation — observe that the `X-Correlation-Id` header no longer shows up in the "before next" logs. Then restore the correct order.
9. Deliberately register something after the final `app.Run(...)` and confirm it never executes (add an `app.Use(...)` after `Run` and verify there is no effect) — this demonstrates that `Run` is terminal.
10. Write one integration test with `WebApplicationFactory<Program>` verifying that a request to `/api` without a key returns 401, and with the key returns 200. Use `dotnet new xunit -n MiddlewareLab.Tests`.

#### Requirements
- Target framework `net8.0`, language C# 12: use top-level statements in `Program.cs`, pattern matching (e.g., `is null` / `is { Length: > 0 }`), collection expressions where you assemble header lists, and raw string literals `"""..."""` for JSON responses.
- Each custom middleware is a separate class with `InvokeAsync(HttpContext context)` and a constructor taking `RequestDelegate next`. The `UseXxx` extension method lives in a static `Middleware/...Extensions.cs` class.
- Registration order in `Program.cs` strictly follows: `UseExceptionHandler` → `UseRequestLogging` → `UseCorrelationId` → `UseWhen(POST)` → `Map("/api")` → `Map("/admin")` → final `Run`. Any permutation must be justified with a comment.
- `ApiKeyMiddleware` correctly short-circuits: on failure it does NOT call `_next`, sets status 401, and writes the body. On success it calls `await _next(context)`.
- Logging and correlation ID work on the "way out" (after `await next()`), exactly like the `Stopwatch` example in the lesson.
- All `Use` components always call `await next()`, except for deliberate short-circuits (which must be explicitly commented).
- The code compiles without warnings, `dotnet build` is clean, and `dotnet run` starts without errors.
- The integration test passes: `dotnet test` is green.

#### Pitfalls
- **Order is critical.** As the lesson stresses, `UseExceptionHandler` must come first — otherwise exceptions from earlier middleware are not handled. `UseAuthentication` before `UseAuthorization`. `UseRouting` before `UseEndpoints`. Our lab has no routing/auth services, but even the correlation → logging order matters for observability.
- **`Run` is terminal.** Anything registered after `Run` never executes. Do not add middleware "just in case" after the terminal handler — this is a common mistake from the lesson.
- **Writing to `Response.Body` after `await next()`.** If downstream has already started sending (`Response.HasStarted == true`), writing to the body again or changing headers throws `InvalidOperationException`. In logging we set only the `X-Request-Ms` header after `next` — this is safe because the headers are not yet committed, provided downstream has not started writing. But if your downstream writes to the body before you set the header, set headers before `next` or check `HasStarted`.
- **`MapWhen` vs `UseWhen`.** `MapWhen` creates a separate branch and does not return the request to the main pipeline; `UseWhen` adds middleware conditionally but the request rejoins the main flow. In the lab we use `UseWhen` for the POST header because POST `/` must reach the final `Run` of the main pipeline. If you mistakenly use `MapWhen`, the request gets stuck in the branch and never reaches `app.Run`.
- **Forgotten `await` before `next()`.** Without `await`, the pipeline silently breaks: the next middleware starts running, but the current one does not wait and continues immediately, corrupting the order. Always `await next()`.
- **DI in a middleware class.** Services injected through the middleware constructor must be singleton or transient; scoped services should be taken through `InvokeAsync(HttpContext, IServiceProvider)` or `RequestServices`. In the lab `ILogger<T>` is a singleton, so it is fine via the constructor.
- **`Map` and the base path.** Inside a `Map("/api")` branch, `context.Request.PathBase` becomes `/api` and `Path` is the remainder. The terminal `Run` inside the branch sees the trimmed path.

#### Acceptance criteria
- [ ] Project `MiddlewareLab` is created, `dotnet build` passes with no errors or warnings.
- [ ] `Program.cs` uses top-level statements and C# 12 (raw strings for JSON, pattern matching).
- [ ] `UseExceptionHandler` is registered first.
- [ ] `RequestLoggingMiddleware` is a separate class with `InvokeAsync` and a `UseRequestLogging` extension; it sets `X-Request-Started` before `next` and `X-Request-Ms` after.
- [ ] `CorrelationIdMiddleware` generates/reads `X-Correlation-Id`, stores it in `Items` and in a response header.
- [ ] `UseWhen` for POST correctly adds `X-Post-Processed` and does NOT detach the request from the main pipeline.
- [ ] `Map("/api")` contains `ApiKeyMiddleware` and a terminal `Run` returning JSON.
- [ ] `ApiKeyMiddleware` returns 401 without the key and 200 with the key `X-Api-Key: secret`.
- [ ] `Map("/admin")` returns 403 for non-POST and 200 for POST.
- [ ] The final `app.Run` returns "Main pipeline" for the root path.
- [ ] Nothing is registered after the final `Run`.
- [ ] All `Use` components call `await next()` (except deliberate short-circuits, with a comment).
- [ ] The `curl` checks from step 7 produce the expected statuses and headers.
- [ ] The integration test through `WebApplicationFactory<Program>` is green.

#### Hints (without giving the answer away)
- For the correlation ID, use `Guid.NewGuid().ToString("N")` and check whether `context.Request.Headers["X-Correlation-Id"]` already has a value.
- In `RequestLoggingMiddleware`, take `ILogger<RequestLoggingMiddleware>` via the constructor and log the method, path, and duration on the "way out".
- For `Map("/admin")` with a method check, you can use `MapWhen` inside the branch, or simply `if (context.Request.Method != "POST")` in the first middleware of the branch.
- To make `WebApplicationFactory<Program>` work with top-level statements, add `public partial class Program { }` at the end of `Program.cs` — this exposes the type to tests.
- For the integration test, use `HttpClient client = factory.CreateClient()` and `var resp = await client.GetAsync("/api")`.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — Middleware pipeline: Use / Run / Map / UseWhen + custom middleware classes.
// Bilingual comments RU+EN.

var builder = WebApplication.CreateBuilder(args);
builder.Logging.ClearProviders();
builder.Logging.AddSimpleConsole(o => o.SingleLine = true);

var app = builder.Build();

// 1) Exception handling first, exactly as in the lesson.
//    Обработка исключений первой, как в уроке.
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        context.Response.StatusCode = 500;
        await context.Response.WriteAsync("Internal Server Error / Внутренняя ошибка");
    });
});

// 2) Request logging — dedicated class with InvokeAsync.
//    Логирование запроса — отдельный класс с InvokeAsync.
app.UseRequestLogging();

// 3) Correlation ID — before business logic so everyone can log it.
//    Correlation ID — до бизнес-логики, чтобы все логировали его.
app.UseCorrelationId();

// 4) Conditional middleware without branching for POST.
//    Условный middleware без ветвления для POST.
app.UseWhen(
    context => context.Request.Method == "POST",
    subApp => subApp.Use(async (context, next) =>
    {
        context.Response.Headers["X-Post-Processed"] = "true";
        await next();
    }));

// 5) Branch /api: API key check + terminal JSON.
//    Ветвь /api: проверка ключа + терминальный JSON.
app.Map("/api", apiApp =>
{
    apiApp.Use(async (context, next) =>
    {
        context.Response.Headers["X-Api-Version"] = "1.0";
        await next();
    });
    apiApp.UseApiKey();   // short-circuit on missing key / короткое замыкание при отсутствии ключа
    apiApp.Run(async context =>
    {
        context.Response.ContentType = "application/json";
        var correlationId = context.Items["CorrelationId"] as string ?? "unknown";
        // Raw string literal for JSON — C# 11+ idiom.
        // Raw string literal для JSON — идиома C# 11+.
        await context.Response.WriteAsync($$"""{"status":"ok","correlationId":"{{correlationId}}"}""");
    });
});

// 6) Branch /admin: POST only.
//    Ветвь /admin: только POST.
app.Map("/admin", adminApp =>
{
    adminApp.Use(async (context, next) =>
    {
        if (context.Request.Method != "POST")
        {
            context.Response.StatusCode = 403;
            await context.Response.WriteAsync("Forbidden: POST required");
            return; // short-circuit / короткое замыкание
        }
        await next();
    });
    adminApp.Run(async context =>
    {
        await context.Response.WriteAsync("Admin area OK");
    });
});

// 7) Terminal handler of the main pipeline.
//    Терминальный обработчик основного конвейера.
app.Run(async context =>
{
    await context.Response.WriteAsync("Main pipeline / Основной конвейер");
});

// app.Use(...) here is useless — nothing runs after Run.
// app.Use(...) сюда писать бесполезно — после Run не выполняется.
app.Run();

public partial class Program { } // for WebApplicationFactory / для WebApplicationFactory
```

**Line-by-line walk-through.** The `app.UseExceptionHandler` line is first — directly required by the lesson: the exception handler must sit "near the very top" to catch failures from all later components. `UseRequestLogging()` and `UseCorrelationId()` are custom middleware classes with extension methods, exactly as the best practice recommends: "extract complex logic into dedicated classes with `InvokeAsync`". The logging → correlation order is chosen so that the correlation ID already exists when logging writes its "way out" line (if correlation came later, the "before next" log would not contain the ID). `UseWhen` for POST is precisely the case from the lesson: "conditional middleware without branching", and the request rejoins the main pipeline, so POST `/` reaches the final `app.Run`. `Map("/api", ...)` creates a separate pipeline: inside it, `UseApiKey()` can short-circuit (`return` without `_next`), and the terminal `Run` returns JSON. The raw string literal `$$"""..."""` with interpolation is a C# 11+ idiom, valid on .NET 8. `Map("/admin")` demonstrates short-circuiting inside a branch: the first middleware checks the method and, on failure, `return`s without calling `next`. The final `app.Run(...)` closes the main pipeline; anything after it (as the comment below shows) is dead code, illustrating the terminal nature of `Run`. `public partial class Program { }` makes the `Program` type visible to `WebApplicationFactory` in integration tests — a mandatory trick with top-level statements.

#### Going deeper (bonus)
1. Add `UseStatusCodePages` or a custom middleware that intercepts 404 and returns a custom JSON error. Think about where in the order it should sit.
2. Implement a rate-limiting middleware by hand (without the built-in `AddRateLimiter`): keep a per-IP request counter in `IMemoryCache` and return 429 when the limit is exceeded. Where in the order should it go — before or after the correlation ID?
3. Rewrite `ApiKeyMiddleware` so keys are stored in `IConfiguration` under the `ApiKeys:` prefix, with one key for `/api` and another for `/admin`. Use the scoped `IConfiguration` service through `RequestServices`.
4. Add a `MapWhen` for requests carrying the `X-Debug: true` header that adds detailed request-stack logging. Compare its behavior with `UseWhen` — why is `MapWhen` more dangerous in this case?

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `MiddlewareLab` собирается без ошибок (RU).
- [ ] `Program.cs` использует top-level statements и C# 12 (RU).
- [ ] Все пять собственных middleware реализованы как классы с `InvokeAsync` (RU).
- [ ] Порядок в `Program.cs` соответствует требованию (RU).
- [ ] `curl`-проверки дают ожидаемые результаты (RU).
- [ ] Интеграционный тест через `WebApplicationFactory<Program>` зелёный (RU).
- [ ] Файл `homework-M13-L02-middleware-pipeline.md` присутствует в репозитории (RU).
- [ ] The `MiddlewareLab` project builds cleanly (EN).
- [ ] `Program.cs` uses top-level statements and C# 12 (EN).
- [ ] All five custom middleware are implemented as classes with `InvokeAsync` (EN).
- [ ] The order in `Program.cs` matches the requirement (EN).
- [ ] `curl` checks produce the expected results (EN).
- [ ] The integration test through `WebApplicationFactory<Program>` is green (EN).
- [ ] The `homework-M13-L02-middleware-pipeline.md` file is in the repository (EN).

#### Ресурсы / Resources
- [Microsoft Learn — ASP.NET Core Middleware](https://learn.microsoft.com/aspnet/core/fundamentals/middleware/)
- [Microsoft Learn — Write custom middleware](https://learn.microsoft.com/aspnet/core/fundamentals/middleware/write)
- [Microsoft Learn — Map and MapWhen](https://learn.microsoft.com/aspnet/core/fundamentals/middleware/#branch-the-pipeline)
- [Microsoft Learn — WebApplicationFactory integration tests](https://learn.microsoft.com/aspnet/core/test/integration-tests)
- [Урок M13-L02 / Lesson M13-L02](lesson-M13-L02-middleware-pipeline.md)
