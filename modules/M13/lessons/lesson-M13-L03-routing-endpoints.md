[⬆ К модулю M13](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M13-L03: Маршрутизация, endpoint routing / Routing, endpoint routing

**Модуль / Module:** M13
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Маршрутизация в ASP.NET Core — это процесс сопоставления входящего HTTP-запроса с конкретной точкой обработки (endpoint). Начиная с ASP.NET Core 3.0 появилась модель **endpoint routing**, которая разделяет маршрутизацию на два этапа: сначала запрос «привязывается» к эндпоинту на этапе маршрутизации, а уже затем выполняется в конвейере middleware. Это похоже на работу почтового сортировочного центра: письмо (запрос) сначала определяется по индексу и адресу (маршрут), а потом передаётся конкретному курьеру (делегат эндпоинта).

В минимальных API (minimal APIs) маршруты создаются методами `MapGet`, `MapPost`, `MapPut`, `MapDelete` и др. Каждый вызов регистрирует эндпоинт с шаблоном маршрута и делегатом-обработчиком. Например, `app.MapGet("/hello", () => "Hi!");` создаёт эндпоинт, отвечающий на `GET /hello`. Шаблон маршрута может содержать **параметры** в фигурных скобках: `/products/{id}`. Значения параметров автоматически извлекаются из URL и привязываются к аргументам обработчика.

Шаблоны поддерживают значения по умолчанию и необязательные сегменты: `{category=books}` задаёт значение по умолчанию, `{id?}` делает сегмент необязательным. Для валидации значений применяются **constraints** (ограничения): `{id:int}` требует целое число, `{name:alpha}` — только буквы, `{slug:maxlength(50)}` — строка до 50 символов. Если ограничение не выполнено, эндпоинт считается несовпадающим, и запрос идёт дальше по таблице маршрутов (или падает в 404).

Под капотом работает подсистема `IRouting` и реализация `LinkGenerator`. Интерфейс `IRouter` из старой модели практически не используется в новом коде — современная маршрутизация строится вокруг `EndpointDataSource`, `MatcherPolicy` и `LinkGenerator`. Последний позволяет генерировать URL по имени эндпоинта и значениям маршрута, что удобно для построения ссылок и редиректов: `linkGenerator.GetPathByName("GetProduct", new { id = 42 })`.

Для MVC и Razor Pages используется `MapControllerRoute`, который регистрирует шаблоны контроллеров по умолчанию. Классический шаблон `{controller=Home}/{action=Index}/{id?}` сопоставляет сегменты URL с именами контроллера и действия. Атрибуты `[Route]`, `[HttpGet]`, `[HttpPost]` на контроллерах и методах добавляют **attribute routing**, который работает параллельно с convention-based маршрутизацией. В современном приложении обычно выбирают один подход: либо minimal API с `Map*`, либо контроллеры с `MapControllerRoute`/атрибутами.

Важное правило endpoint routing: порядок регистрации эндпоинтов имеет значение только при одинаковых шаблонах; более специфичные шаблоны должны идти раньше общих. Маршрутизация не выполняет авторизацию и валидацию модели — этим занимаются другие middleware, которые обращаются к выбранному `Endpoint` через `HttpContext.GetEndpoint()`. Также маршрутизация работает с HTTP-методами: `MapGet` отвечает только на GET, а `MapPost` — на POST, поэтому один и тот же путь может иметь разные обработчики для разных методов.

#### Theory (EN)

Routing in ASP.NET Core is the process of matching an incoming HTTP request to a specific handling point called an **endpoint**. Since ASP.NET Core 3.0, the framework uses **endpoint routing**, which splits the work into two phases: first the request is matched to an endpoint during the routing phase, and only then the endpoint's delegate executes later in the middleware pipeline. Think of it like a postal sorting hub: a letter (the request) is first identified by its zip code and address (the route), and only afterwards is it handed to a specific courier (the endpoint delegate).

In minimal APIs, routes are created with `MapGet`, `MapPost`, `MapPut`, `MapDelete`, and similar methods. Each call registers an endpoint with a route template and a handler delegate. For example, `app.MapGet("/hello", () => "Hi!");` creates an endpoint that responds to `GET /hello`. A route template can include **parameters** in curly braces, such as `/products/{id}`. Parameter values are extracted from the URL automatically and bound to the handler's arguments by name.

Templates support defaults and optional segments: `{category=books}` provides a default value, and `{id?}` makes the segment optional. To validate values, you apply **constraints**: `{id:int}` requires an integer, `{name:alpha}` allows only letters, `{slug:maxlength(50)}` limits the string to 50 characters. When a constraint fails, the endpoint is treated as non-matching and the request continues through the route table (or falls through to a 404).

Under the hood, the system is built on `IRouting` infrastructure and the `LinkGenerator` service. The old `IRouter` interface is rarely used in modern code — current routing revolves around `EndpointDataSource`, `MatcherPolicy`, and `LinkGenerator`. The link generator lets you build URLs by endpoint name and route values, which is handy for links and redirects: `linkGenerator.GetPathByName("GetProduct", new { id = 42 })`.

For MVC and Razor Pages, `MapControllerRoute` registers controller templates. The classic pattern `{controller=Home}/{action=Index}/{id?}` maps URL segments to controller and action names. Attributes such as `[Route]`, `[HttpGet]`, and `[HttpPost]` on controllers and methods add **attribute routing**, which runs side by side with convention-based routing. In a modern app you usually pick one approach: either minimal API with `Map*` methods, or controllers with `MapControllerRoute` and attributes.

A key rule of endpoint routing: the registration order matters only for identical templates — more specific templates should be registered before general ones. Routing itself does not perform authorization or model validation; those are handled by other middleware that inspects the selected `Endpoint` via `HttpContext.GetEndpoint()`. Routing is also method-aware: `MapGet` responds only to GET, `MapPost` only to POST, so the same path can have different handlers for different HTTP verbs.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8+ — Minimal API + endpoint routing + link generation
// Минимальный API + маршрутизация эндпоинтов + генерация ссылок
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Routing;
using Microsoft.Extensions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);

// Регистрируем сервис генерации ссылок / Register the link generator
builder.Services.AddRouting();

var app = builder.Build();

// 1) Простой эндпоинт GET / Simple GET endpoint
app.MapGet("/", () => "Hello, routing! / Привет, маршрутизация!");

// 2) Параметр маршрута / Route parameter
//    GET /products/42  ->  id = 42
app.MapGet("/products/{id:int}", (int id) =>
    $"Product id = {id} / Товар id = {id}");

// 3) Параметр по умолчанию + необязательный / Default + optional
//    GET /catalog          -> category=books, page=1
//    GET /catalog/toys/5   -> category=toys,  page=5
app.MapGet("/catalog/{category=books}/{page:int=1}",
    (string category, int page) =>
        $"Category={category}, Page={page} / Категория={category}, Стр={page}");

// 4) Именованный эндпоинт + генерация ссылки / Named endpoint + link gen
app.MapGet("/orders/{id:int}", (int id, LinkGenerator linker, HttpContext ctx) =>
{
    var url = linker.GetUriByName(ctx, "GetOrder", new { id });
    return Results.Ok(new { id, self = url });
}).WithName("GetOrder");

// 5) POST с привязкой тела / POST with body binding
//    POST /orders  { "customer": "Alice" }
app.MapPost("/orders", (Order order) =>
    Results.Created($"/orders/{order.Id}", order));

// 6) Маршруты контроллеров (MVC) / Controller routes (MVC)
// app.MapControllerRoute(
//     name: "default",
//     pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();

public record Order(int Id, string Customer);
```

#### Best Practices
- Регистрируйте более специфичные маршруты раньше общих, чтобы избежать ложных совпадений.
- Используйте constraints (`:int`, `:guid`, `:regex(...)`) для ранней валидации параметров вместо проверок в обработчике.
- Называйте эндпоинты через `WithName`, чтобы генерировать ссылки через `LinkGenerator` без жёстко закодированных URL.
- Выбирайте один стиль маршрутизации в приложении: minimal API (`Map*`) или контроллеры (attribute/convention routing).
- Register more specific routes before general ones to avoid accidental matches.
- Use constraints (`:int`, `:guid`, `:regex(...)`) for early parameter validation instead of in-handler checks.
- Name endpoints with `WithName` and build URLs through `LinkGenerator` rather than hard-coded strings.
- Pick one routing style per app: minimal API (`Map*`) or controllers (attribute/convention routing).

#### Частые ошибки / Common Mistakes
- Одинаковые шаблоны для GET и POST без различения методов → используйте `MapGet`/`MapPost` отдельно для каждого метода.
- `{id}` без `:int` принимает произвольные строки и ломает привязку → добавляйте constraints для типобезопасности.
- Жёстко закодированные URL в ответах (`"/orders/" + id`) → генерируйте ссылки через `LinkGenerator.GetUriByName`.
- Порядок: общий шаблон `{*slug}` раньше `/products/{id}` перекрывает нужный маршрут → ставьте специфичные маршруты первыми.
- Same template for GET and POST without method distinction → use `MapGet`/`MapPost` separately per verb.
- `{id}` without `:int` accepts arbitrary strings and breaks binding → add constraints for type safety.
- Hard-coded URLs in responses (`"/orders/" + id`) → build links via `LinkGenerator.GetUriByName`.
- Order: a catch-all `{*slug}` registered before `/products/{id}` shadows the intended route → put specific routes first.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Все эндпоинты используют корректные HTTP-методы (`MapGet`, `MapPost` и т.д.).
- [ ] Параметры маршрута защищены constraints (`:int`, `:alpha`, `:maxlength`).
- [ ] Необязательные сегменты и значения по умолчанию заданы через `{x?}` и `{x=default}`.
- [ ] Ссылки строятся через `LinkGenerator`, а не конкатенацией строк.
- [ ] Для MVC зарегистрирован `MapControllerRoute` или используется attribute routing.
- [ ] All endpoints use the correct HTTP verbs (`MapGet`, `MapPost`, etc.).
- [ ] Route parameters are guarded with constraints (`:int`, `:alpha`, `:maxlength`).
- [ ] Optional segments and defaults are declared via `{x?}` and `{x=default}`.
- [ ] Links are built through `LinkGenerator`, not string concatenation.
- [ ] MVC apps register `MapControllerRoute` or rely on attribute routing.

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/aspnet/core/fundamentals/routing]

---

[⬆ К модулю M13](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
