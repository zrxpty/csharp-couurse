[⬆ К модулю M13](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M13-L02: Middleware pipeline, Use/Run/Map, порядок / Middleware pipeline, Use/Run/Map, order

**Модуль / Module:** M13
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Конвейер middleware (промежуточных компонентов) — это сердце обработки запроса в ASP.NET Core. Каждый HTTP-запрос проходит через цепочку компонентов, и каждый компонент решает: пропустить запрос дальше или вернуть ответ немедленно. Подумайте о конвейере как о «матрёшке» — запрос входит снаружи, проходит каждый слой, доходит до ядра (например, контроллера или endpoint), а затем возвращается обратно через те же слои, но в обратном порядке. Это позволяет компонентам работать и на «входе» (до `next`), и на «выходе» (после `await next()`).

Три главных метода построения конвейера — `Use`, `Run` и `Map`. Метод `app.Use(...)` добавляет компонент, который **обязательно вызывает следующий** компонент через `await next()`; иначе цепочка обрывается. Метод `app.Run(...)` добавляет **терминальный** компонент — он не вызывает `next` и заканчивает обработку. Важно: всё, что добавлено после `Run`, не выполняется, потому что запрос дальше не пойдёт. Метод `app.Map("/path", ...)` создаёт **ветвление**: если путь запроса начинается с указанного сегмента, выполняется отдельный конвейер; иначе — основной. Это удобно для разделения логики по префиксам URL.

**Порядок критичен.** Компоненты выполняются строго в порядке добавления через `Use`/`Run`. Компонент аутентификации должен идти до авторизации; обработка исключений (`UseExceptionHandler`) — почти в самом начале, чтобы ловить ошибки из всех последующих компонентов; `UseStaticFiles` — до маршрутизации, чтобы статические файлы отдавались быстро; `UseRouting` — до `UseEndpoints`. Если перепутать порядок, можно получить странные баги: например, 401 вместо 200, потому что `UseAuthorization` отработал раньше `UseAuthentication`, или необработанное исключение, потому что обработчик стоит после источника ошибки.

**Краткий middleware** пишется как inline-делегат `app.Use(async (context, next) => { /* до */ await next(); /* после */ });` — удобно для прототипов и простой логики (логирование, таймеры, заголовки). Для production-логики лучше создавать **отдельный класс** с методом `InvokeAsync` и методом расширения `UseXxx` — это тестируется, переиспользуется и читается легче.

**Map branching** позволяет разветвлять конвейер по пути. Существует также `MapWhen` — ветвление по произвольному условию (`context => context.Request.Method == "POST"`), и `UseWhen` — условное добавление middleware **без создания ветви** (в отличие от `MapWhen`, после условного блока запрос возвращается в основной конвейер). `UseWhen` полезен, когда нужно добавить логирование или проверку только для определённых запросов, не отрывая их от общей обработки.

Каждый middleware получает `HttpContext`, через который читает запрос и пишет ответ. Писать в `Response.Body` **после** `await next()` опасно, если downstream уже начал отправку — статус и заголовки могут быть зафиксированы. Поэтому изменения ответа делайте либо до `next`, либо через буферизацию. Помните: middleware — это про «перехват» и «передачу дальше»; всё, что не передаёт дальше, становится терминальным.

#### Theory (EN)

The middleware pipeline is the heart of request handling in ASP.NET Core. Every HTTP request flows through a chain of components, and each component decides: pass the request further down, or short-circuit and return a response immediately. Think of the pipeline as a “matryoshka doll” — the request enters from the outside, passes through every layer, reaches the core (a controller, a minimal API endpoint, or a terminal middleware), and then travels back out through the same layers in reverse order. This bidirectional flow lets each component act both on the “way in” (before `await next()`) and on the “way out” (after `await next()`).

The three core methods for building the pipeline are `Use`, `Run`, and `Map`. `app.Use(...)` adds a component that **must call the next** component via `await next()`; otherwise the chain breaks. `app.Run(...)` adds a **terminal** component — it does not call `next` and ends the pipeline. Anything registered after `Run` never executes, because the request never gets there. `app.Map("/path", ...)` creates a **branch**: if the request path starts with the given segment, a separate pipeline runs; otherwise the main pipeline continues. This is handy for isolating behavior by URL prefix.

**Order is critical.** Components execute strictly in the order they are registered. Authentication must come before authorization; exception handling (`UseExceptionHandler`) should be near the very top so it can catch failures from all later components; `UseStaticFiles` goes before routing so static files are served quickly; `UseRouting` must come before `UseEndpoints`. Get the order wrong and you get subtle bugs — for example, a 401 instead of a 200 because `UseAuthorization` ran before `UseAuthentication`, or an unhandled exception because the handler sits after the code that throws.

A **short middleware** is an inline delegate: `app.Use(async (context, next) => { /* before */ await next(); /* after */ });` — great for prototypes and trivial logic (logging, timers, custom headers). For production-grade behavior, prefer a **dedicated class** with an `InvokeAsync` method and a `UseXxx` extension method: it is testable, reusable, and far easier to read.

**Map branching** lets you split the pipeline by path. There is also `MapWhen`, which branches on an arbitrary predicate (`context => context.Request.Method == "POST"`), and `UseWhen`, which conditionally adds middleware **without branching** — unlike `MapWhen`, after the conditional block the request rejoins the main pipeline. `UseWhen` is perfect when you want logging or validation only for certain requests but do not want to detach them from the rest of the pipeline.

Every middleware receives an `HttpContext`, through which it reads the request and writes the response. Writing to `Response.Body` **after** `await next()` is risky if downstream has already started sending — the status code and headers may already be committed. Change the response either before `next` or via buffering. Remember: middleware is about “intercept” and “forward”; anything that does not forward becomes terminal.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Middleware pipeline: Use / Run / Map / UseWhen
// Двуязычные комментарии RU+EN

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// 1) Обработка исключений — первой, чтобы ловить ошибки из всех слоёв.
//    Exception handling first, so it catches errors from every layer.
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        context.Response.StatusCode = 500;
        await context.Response.WriteAsync("Internal Server Error / Внутренняя ошибка");
    });
});

// 2) Краткий inline-middleware: логирование времени запроса.
//    Short inline middleware: log request duration.
app.Use(async (context, next) =>
{
    var sw = System.Diagnostics.Stopwatch.StartNew();
    // ДО next — работаем на "входе".
    // Before next — act on the way in.
    context.Response.Headers["X-Request-Started"] = DateTime.UtcNow.ToString("O");
    await next();                              // передаём дальше / forward
    sw.Stop();
    // ПОСЛЕ next — работаем на "выходе".
    // After next — act on the way out.
    context.Response.Headers["X-Request-Ms"] = sw.ElapsedMilliseconds.ToString();
});

// 3) Условный middleware без ветвления: добавляем заголовок только для POST.
//    Conditional middleware without branching: add header only for POST.
app.UseWhen(context => context.Request.Method == "POST",
    subApp => subApp.Use(async (context, next) =>
    {
        context.Response.Headers["X-Post-Processed"] = "true";
        await next();
    }));

// 4) Ветвление по пути: /api работает в отдельном конвейере.
//    Branch by path: /api runs in a separate pipeline.
app.Map("/api", apiApp =>
{
    apiApp.Use(async (context, next) =>
    {
        context.Response.Headers["X-Api-Version"] = "1.0";
        await next();
    });

    // Терминальный обработчик внутри ветви.
    // Terminal handler inside the branch.
    apiApp.Run(async context =>
    {
        context.Response.ContentType = "application/json";
        await context.Response.WriteAsync("""{"status":"ok"}""");
    });
});

// 5) Терминальный middleware в основном конвейере.
//    Terminal middleware in the main pipeline.
app.Run(async context =>
{
    await context.Response.WriteAsync("Main pipeline endpoint / Конец основного конвейера");
});

app.Run(); // запуск приложения / start the app
```

#### Best Practices
- Регистрируйте middleware строго в порядке: ExceptionHandler → StaticFiles → Routing → Authentication → Authorization → Endpoints.
- Выносите сложную логику в отдельные классы с `InvokeAsync` и методами расширения `UseXxx` для тестируемости.
- Всегда вызывайте `await next()` в `Use`, если не хотите прервать конвейер; помните, что пропуск `next` делает middleware терминальным.
- Не пишите в тело ответа после `await next()`, если заголовки уже зафиксированы — используйте буферизацию.
- Use `UseWhen` for conditional behavior that should rejoin the main pipeline; use `Map`/`MapWhen` only when you genuinely want a separate branch.
- Keep middleware thin — each component should do one job (logging, auth, headers, routing) and delegate the rest to `next`.
- Add correlation IDs early in the pipeline so every downstream component can log them.

#### Частые ошибки / Common Mistakes
- `UseAuthorization` стоит раньше `UseAuthentication` → сначала `UseAuthentication`, затем `UseAuthorization` (RU).
- Регистрация middleware после `Run` → ничего не выполнится; не ставьте ничего после терминального компонента (RU).
- Запись в `Response.Body` после `await next()` → проверяйте `Response.HasStarted` или буферизуйте ответ (RU).
- Использование `MapWhen` вместо `UseWhen` и потеря продолжения основного конвейера → используйте `UseWhen`, если запрос должен вернуться в основной поток (RU).
- Putting `UseRouting` after `UseEndpoints` → endpoints won't be resolved; `UseRouting` must precede `UseEndpoints` (EN).
- Forgetting `await` before `next()` → pipeline silently breaks; always `await next()` (EN).
- Catching exceptions with try/catch around `next` but still writing to a started response → check `Response.HasStarted` before writing (EN).
- Overusing inline `app.Use` for complex logic → extract to a dedicated middleware class for clarity and testing (EN).

#### Чек-лист самопроверки / Self-check Checklist
- [ ] `UseExceptionHandler` зарегистрирован первым (RU).
- [ ] `UseAuthentication` идёт до `UseAuthorization` (RU).
- [ ] `UseRouting` идёт до `UseEndpoints` (RU).
- [ ] Каждый `Use` вызывает `await next()`, если не является терминальным (RU).
- [ ] `Map`/`MapWhen` используются только для реального ветвления; `UseWhen` — для условного добавления (RU).
- [ ] Сложная логика вынесена в отдельные классы middleware (RU).
- [ ] No middleware is registered after a `Run` terminal (EN).
- [ ] Response is not written after `next()` unless `HasStarted` is false (EN).
- [ ] Order reflects: exceptions → static files → routing → auth → endpoints (EN).
- [ ] Inline middleware is reserved for short, single-purpose logic (EN).
- [ ] Branches via `Map` return a coherent response and do not fall through unexpectedly (EN).

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/aspnet/core/fundamentals/middleware/](https://learn.microsoft.com/aspnet/core/fundamentals/middleware/)

---

[⬆ К модулю M13](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
