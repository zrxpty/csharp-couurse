---
[← К уроку M13-L03](lesson-M13-L03-routing-endpoints.md) | [⬆ К модулю M13](../README.md) | [Следующее ДЗ →](homework-M13-L04-di-lifetimes.md)
---

### Домашнее задание M13-L03: Маршрутизация, endpoint routing / Homework M13-L03: Routing, endpoint routing

**Урок / Lesson:** M13-L03
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться проектировать таблицу эндпоинтов в minimal API на ASP.NET Core 8: использовать параметры маршрута, значения по умолчанию и необязательные сегменты, ограничения (constraints), именованные эндпоинты и `LinkGenerator` для генерации ссылок, а также корректно упорядочивать специфичные и общие шаблоны. (EN) Learn to design an endpoint table in minimal API on ASP.NET Core 8: use route parameters, defaults and optional segments, constraints, named endpoints and `LinkGenerator` for link generation, and correctly order specific versus general templates.

#### Связь с уроком / Connection to the lesson
(RU) Урок M13-L03 вводит двухфазную модель endpoint routing и инструменты minimal API (`MapGet`, `MapPost`, `MapPut`, `MapDelete`), шаблоны с параметрами, значениями по умолчанию, необязательными сегментами и ограничениями, а также `LinkGenerator` и взаимодействие маршрутизации с middleware. ДЗ закрепляет каждую из этих тем на реальном API трекера задач и требует применить best practices и избежать частых ошибок из урока.
(EN) Lesson M13-L03 introduces the two-phase endpoint routing model and the minimal API toolset (`MapGet`, `MapPost`, `MapPut`, `MapDelete`), templates with parameters, defaults, optional segments and constraints, plus `LinkGenerator` and the interaction between routing and middleware. This homework reinforces every one of those topics on a realistic task-tracker API and requires applying the lesson's best practices while avoiding its common mistakes.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Представьте, что вы участвуете в разработке внутреннего сервиса «TaskHub» — лёгкого трекера задач для небольшой команды. Команда решила строить HTTP API на minimal API поверх .NET 8, потому что такой подход даёт компактный код, явную таблицу эндпоинтов и хорошую производительность. Архитектор уже зафиксировал несколько требований: маршрутизация должна быть полностью эндпоинтной (endpoint routing), каждый публичный маршрут должен иметь понятный шаблон, параметры должны валидироваться на уровне маршрута через constraints, а ссылки на ресурсы должны строиться через `LinkGenerator`, а не через конкатенацию строк.

Дополнительно заказчик хочет, чтобы API поддерживало «человеко-понятные» URL для каталога задач (с категорией и номером страницы), постраничный и фильтрованный доступ, а также служебный маршрут для «всех остальных» путей (catch-all), который будет возвращать диагностическую информацию — это поможет на раннем этапе отладки и документирования. Поскольку API будет расти, критически важно сразу заложить правильный порядок регистрации эндпоинтов: специфичные маршруты раньше общих, чтобы catch-all и шаблоны с параметрами не перекрывали целевые обработчики.

В этом задании вы поэтапно построите работающий проект, проверите ключевые маршруты вручную через `curl` или HTTP-файл, а затем проведёте рефакторинг: уберёте жёстко закодированные URL, добавите именованные эндпоинты и сгенерируете ссылки. В финале вы должны получить компилируемое приложение, в котором каждая тема урока — параметры, defaults, optional, constraints, именованные эндпоинты, `LinkGenerator`, упорядочивание, метод-зависимая маршрутизация — представлена реальным кодом.

#### Что нужно сделать (пошагово)

1. Создайте новый проект minimal API. Выполните команду `dotnet new web -n TaskHub -o TaskHub` в директории `modules/M13/homework`. Убедитесь, что в `TaskHub.csproj` указаны `TargetFramework` равный `net8.0` и `LangVersion` равный `latest` (или `12`), чтобы были доступны top-level statements, collection expressions и record-типы. Откройте сгенерированный `Program.cs` и удалите шаблонный код, оставив только `var builder = WebApplication.CreateBuilder(args);` и `var app = builder.Build();`.

2. Зарегистрируйте сервисы маршрутизации явно через `builder.Services.AddRouting();`. Хотя minimal API добавляет базовую маршрутизацию автоматически, явный вызов позволяет дальше настраивать параметры (например, `AddRouting(o => o.LowercaseUrls = true)`) и показывает осознанное владение подсистемой. Включите use routing и use endpoints: в .NET 8 для minimal API достаточно `app.MapXxx`, но если вы хотите явно управлять конвейером, вызовите `app.UseRouting();` перед регистрацией эндпоинтов и `app.UseEndpoints(...)` (в minimal API это не обязательно — просто имейте в виду различие для MVC-проектов).

3. Реализуйте набор эндпоинтов в точности по спецификации ниже (подробные шаблоны см. в разделе «Требования к решению»). Все обработчики должны возвращать `IResult` (`Results.Ok`, `Results.Created`, `Results.NotFound`, `Results.BadRequest`) вместо «голых» строк и объектов — это улучшает контроль над статус-кодами и согласуется с best practices.

4. Заведите in-memory хранилище задач. Используйте `static` список или `ConcurrentDictionary`, чтобы не усложнять ДЗ инфраструктурой базы данных. Определите `record TaskItem(int Id, string Title, string Category, int Priority, bool Done)`. Реализуйте простую логику создания идентификатора через `Interlocked.Increment` или счётчик.

5. Зарегистрируйте именованные эндпоинты для «самоссылок». Эндпоинт `GET /tasks/{id:int}` назовите `"GetTaskById"` через `.WithName(...)`. В ответе этого эндпоинта верните объект с полем `self`, значение которого получено через `linkGenerator.GetUriByName(ctx, "GetTaskById", new { id })`. Убедитесь, что параметр `HttpContext ctx` и `LinkGenerator linkGenerator` инжектируются в обработчик через DI minimal API.

6. Реализуйте каталог с defaults и optional: маршрут `GET /catalog/{category=general}/{page:int=1}` должен возвращать список задач указанной категории с номером страницы. Проверьте три случая: `GET /catalog` (category=general, page=1), `GET /catalog/dev` (category=dev, page=1), `GET /catalog/dev/3` (category=dev, page=3).

7. Реализуйте catch-all диагностический маршрут `GET /debug/{*path}` ПОСЛЕ всех специфичных маршрутов. Он должен возвращать JSON с полем `path` и значением из сегмента catch-all, а также HTTP-методом и базовым путём. Убедитесь, что если вы зарегистрируете его раньше `/tasks/{id:int}`, то он начнёт «съедать» запросы к задачам — это и есть демонстрация частой ошибки из урока.

8. Реализуйте методы-aware маршруты. Для пути `/tasks` зарегистрируйте `MapGet` (список) и `MapPost` (создание). Для `/tasks/{id:int}` — `MapGet`, `MapPut`, `MapDelete`. Покажите, что один и тот же путь обрабатывается разными методами, и что `GET /tasks/abc` (где `abc` не целое) не попадает в обработчик `{id:int}` и падает в catch-all или 404.

9. Запустите приложение: `dotnet run --project TaskHub`. Откройте второй терминал и протестируйте маршруты с помощью `curl`: `curl http://localhost:5000/tasks`, `curl http://localhost:5000/tasks/1`, `curl -X POST http://localhost:5000/tasks -H "Content-Type: application/json" -d '{"title":"Test","category":"dev","priority":2}'`, `curl http://localhost:5000/catalog/dev/3`, `curl http://localhost:5000/debug/any/nested/path`. Зафиксируйте ожидаемые ответы в комментарии в `Program.cs` или в отдельном `requests.http`.

10. Проведите рефакторинг ссылок. Найдите в коде все места, где URL строится как `$"/tasks/{id}"` или через конкатенацию, и замените их на `LinkGenerator.GetPathByName`/`GetUriByName`. Проверьте, что `Results.Created` в POST использует сгенерированный путь, а не жёстко закодированный.

11. (Бонус) Добавьте constraint `:regex` для проверки категории: `GET /catalog/{category:regex(^[[a-z]]+$)}/{page:int=1}`. Убедитесь, что `GET /catalog/Dev` (заглавная буква) не совпадает и попадает в catch-all или 404, что демонстрирует раннюю валидацию.

#### Требования к решению

Решение должно представлять собой компилируемый проект .NET 8 с одним файлом `Program.cs` (top-level statements) и, при необходимости, вспомогательными record-типами. Используйте C# 12: collection expressions для инициализации списков (`TaskItem[] seed = [..]`), pattern matching в обработчиках где это уместно, raw string literals для длинных JSON-ответов если нужно. Все эндпоинты должны быть зарегистрированы через методы `Map*` minimal API; использование `MapControllerRoute` и атрибутов НЕ требуется для этого ДЗ (это тема для MVC-уроков), но в разделе «Углубление» можно опционально показать параллельную реализацию.

Каждый параметр маршрута, который должен быть типизирован, обязан иметь constraint: `:int` для идентификаторов, `:alpha` или `:regex` для строковых категорий, `:maxlength` для слагов. Необязательные сегменты и значения по умолчанию должны быть объявлены через синтаксис `{x?}` и `{x=default}` соответственно — не через отдельные перегрузки маршрутов. Все ссылки на ресурсы строятся исключительно через `LinkGenerator`; наличие в коде строк вида `$"/tasks/{id}"` в ответах или `Location`-заголовках считается дефектом (кроме seed-данных и тестовых URL в комментариях).

Catch-all маршрут `{*path}` должен быть зарегистрирован последним среди GET-маршрутов. Именованные эндпоинты должны иметь осмысленные имена (`GetTaskById`, `ListTasks`, `CreateTask`), и эти имена должны использоваться при генерации ссылок. Обработчики должны возвращать `IResult`, а не строку или объект напрямую, чтобы кодstatus-коды были явными. Приложение должно запускаться без ошибок и отвечать на все тестовые запросы корректными статус-кодами (200, 201, 404).

#### Тонкости и подводные камни

Главная тонкость endpoint routing, на которую указывает урок, — это двухфазность: сопоставление эндпоинта и его выполнение разделены. Практически это значит, что `UseRouting()` выбирает `Endpoint`, а более поздние middleware (например, авторизация, CORS, endpoint-aware middleware) могут читать выбранную конечную точку через `HttpContext.GetEndpoint()`. Если вы зарегистрируете авторизацию до `UseRouting`, она не увидит эндпоинт и будет работать неправильно — это типичная ошибка при расширении ДЗ в сторону безопасности.

Порядок регистрации эндпоинтов имеет значение только для конкурирующих шаблонов. Маршрут `GET /tasks/{id:int}` и `GET /tasks/priority/{level:alpha}` не конфликтуют (constraint `:int` отбрасывает строку `priority`), но если вы добавите `GET /tasks/{*rest}` слишком рано, он «перекроет» и `{id:int}`, и `priority`. Правило простое: специфичные (более длинные и с большим числом литеральных сегментов) маршруты — раньше, общие (с параметрами и catch-all) — позже.

Constraints дают раннюю валидацию, но они не заменяют полную модельную валидацию. `{id:int}` гарантирует, что в обработчик попадёт целое, но не проверяет диапазон или существование сущности — эту проверку всё равно нужно делать в коде и возвращать 404. Частая ошибка — рассчитывать, что `{id:int}` «отфильтрует» несуществующие идентификаторы; он лишь отсекает нечисловые строки. Аналогично `:maxlength(50)` ограничивает длину, но не содержимое — для содержимого нужен `:regex` или отдельная валидация.

`LinkGenerator` имеет несколько методов: `GetPathByName` возвращает только путь, `GetUriByName` — полный URI (с scheme и host из `HttpContext`). Важно передавать `HttpContext`, когда нужны хост и схема; без него генератор может вернуть `null`, если данные маршрута недостаточны. Ещё одна частая ошибка — забыть `AddRouting()` и удивляться, что `LinkGenerator` недоступен для инъекции (на практике minimal API регистрирует его сам, но явная регистрация делает намерение читаемым и даёт доступ к опциям).

Наконец, метод-зависимость: один и тот же путь может иметь несколько обработчиков для разных HTTP-методов, но два `MapGet` с одинаковым шаблоном приведут к неоднозначности (исключение при запуске в некоторых версиях или «тенению»). Следите за уникальностью пары (метод, шаблон).

#### Критерии приёмки

- [ ] Проект `TaskHub` создан, `dotnet build` проходит без предупреждений.
- [ ] `Program.cs` использует top-level statements и C# 12 (collection expressions видны в seed-данных).
- [ ] Зарегистрирован `AddRouting()` (явно), обработчики используют `MapGet`/`MapPost`/`MapPut`/`MapDelete`.
- [ ] Реализован `GET /tasks` (список) и `POST /tasks` (создание) на одном пути, разных методах.
- [ ] Реализован `GET /tasks/{id:int}` с именем `GetTaskById` и полем `self` через `LinkGenerator`.
- [ ] Реализован `GET /catalog/{category=general}/{page:int=1}` — три кейса возвращают ожидаемые значения.
- [ ] Реализован catch-all `GET /debug/{*path}`, зарегистрированный последним среди GET.
- [ ] `GET /tasks/abc` не попадает в `{id:int}` и уходит в 404 или catch-all (а не вызывает ошибку привязки).
- [ ] Все типизированные параметры имеют constraints (`:int`, `:alpha`/`:regex`, `:maxlength`).
- [ ] В коде нет жёстко закодированных URL в ответах и `Location`-заголовках (только `LinkGenerator`).
- [ ] Обработчики возвращают `IResult` (`Results.Ok`/`Created`/`NotFound`/`BadRequest`), не строки.
- [ ] Порядок регистрации: специфичные маршруты раньше общих и catch-all.
- [ ] `dotnet run` запускается, все тестовые `curl`-запросы возвращают корректные статус-коды.
- [ ] POST `/tasks` возвращает 201 с `Location`, сгенерированным через `LinkGenerator`.
- [ ] В `Program.cs` или `requests.http` зафиксированы ожидаемые ответы на ключевые маршруты.

#### Подсказки (без прямого ответа)

- Вспомните, что `WithName` вызывается на `IEndpointConventionBuilder`, который возвращает `MapGet`. Чтобы получить имя в другом месте, используйте `LinkGenerator.GetUriByName(ctx, "GetTaskById", new { id })`.
- Для seed-данных используйте collection expression: `List<TaskItem> _store = [ new(1, "Setup", "dev", 3, false), new(2, "Docs", "docs", 2, true) ];`.
- Если constraint `:int` отбрасывает строку, но у вас нет catch-all, запрос упадёт в 404 по умолчанию. Подумайте, нужен ли вам единый «fallback» для диагностики.
- `Results.Created(uri, value)` сам проставит заголовок `Location` — достаточно передать сгенерированный путь.
- Для catch-all `{*path}` значение приходит как одна строка со слешами; не пытайтесь разбивать его на сегменты вручную, если не требуется.
- Помните, что `AddRouting(o => o.LowercaseUrls = true)` приведёт к lowercase в сгенерированных URL — это может помочь или помешать, в зависимости от ваших имён эндпоинтов.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — TaskHub: endpoint routing, constraints, LinkGenerator
// Полный рабочий пример. Комментарии RU+EN.

using System.Collections.Concurrent;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Routing;
using Microsoft.Extensions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);

// Явная регистрация routing + lowercase URL для красивых ссылок.
// Explicit routing registration + lowercase URLs for nice links.
builder.Services.AddRouting(o => o.LowercaseUrls = true);

var app = builder.Build();

// In-memory хранилище и счётчик идентификаторов.
// In-memory store and id counter.
static int _nextId = 0;
int NextId() => Interlocked.Increment(ref _nextId);

// Collection expression для seed-данных (C# 12).
// Collection expression for seed data (C# 12).
ConcurrentDictionary<int, TaskItem> _store = new(new[]
{
    new TaskItem(NextId(), "Setup repo", "dev", 3, false),
    new TaskItem(NextId(), "Write README", "docs", 2, true),
    new TaskItem(NextId(), "Plan sprint", "pm", 4, false),
});

// 1) Health endpoint.
app.MapGet("/", () => Results.Ok(new { service = "TaskHub", status = "ok" }))
   .WithName("Health");

// 2) List tasks (с optional-фильтром по категории через query — НЕ маршрут).
app.MapGet("/tasks", (string? category, HttpContext ctx, LinkGenerator linker) =>
{
    // Pattern matching для фильтрации (C# 12).
    // Pattern matching for filtering (C# 12).
    IEnumerable<TaskItem> items = _store.Values;
    if (category is not null)
        items = items.Where(t => t.Category == category);

    var self = linker.GetUriByName(ctx, "ListTasks", values: null);
    return Results.Ok(new { count = items.Count(), self, items = items.ToArray() });
}).WithName("ListTasks");

// 3) Get task by id — именованный эндпоинт + самоссылка через LinkGenerator.
app.MapGet("/tasks/{id:int}", (int id, HttpContext ctx, LinkGenerator linker) =>
{
    if (!_store.TryGetValue(id, out var task))
        return Results.NotFound(new { error = $"Task {id} not found" });

    var self = linker.GetUriByName(ctx, "GetTaskById", new { id });
    return Results.Ok(new { task, self });
}).WithName("GetTaskById");

// 4) Create task — POST, Location строится через LinkGenerator.
app.MapPost("/tasks", (TaskItem input, HttpContext ctx, LinkGenerator linker) =>
{
    if (string.IsNullOrWhiteSpace(input.Title))
        return Results.BadRequest(new { error = "Title is required" });

    var created = input with { Id = NextId(), Done = false };
    _store[created.Id] = created;

    var location = linker.GetUriByName(ctx, "GetTaskById", new { id = created.Id });
    return Results.Created(location, created);
}).WithName("CreateTask");

// 5) Update + delete на том же пути, разные HTTP-методы.
app.MapPut("/tasks/{id:int}", (int id, TaskItem input) =>
{
    if (!_store.TryGetValue(id, out var existing))
        return Results.NotFound(new { error = $"Task {id} not found" });

    var updated = existing with { Title = input.Title, Category = input.Category, Priority = input.Priority, Done = input.Done };
    _store[id] = updated;
    return Results.Ok(updated);
}).WithName("UpdateTask");

app.MapDelete("/tasks/{id:int}", (int id) =>
{
    if (!_store.TryRemove(id, out _))
        return Results.NotFound(new { error = $"Task {id} not found" });

    return Results.NoContent();
}).WithName("DeleteTask");

// 6) Каталог с defaults и optional + constraint на категорию.
//    GET /catalog             -> category=general, page=1
//    GET /catalog/dev         -> category=dev, page=1
//    GET /catalog/dev/3       -> category=dev, page=3
app.MapGet("/catalog/{category:regex(^[[a-z]]+$)=general}/{page:int=1}",
    (string category, int page) =>
{
    var items = _store.Values.Where(t => t.Category == category).ToArray();
    return Results.Ok(new { category, page, count = items.Length, items });
}).WithName("Catalog");

// 7) Catch-all — регистрируется ПОСЛЕДНИМ, чтобы не перекрывать специфичные маршруты.
app.MapGet("/debug/{*path}", (string path, HttpContext ctx) =>
    Results.Ok(new { path, method = ctx.Request.Method, basePath = ctx.Request.PathBase.Value }))
   .WithName("DebugCatchAll");

app.Run();

public record TaskItem(int Id, string Title, string Category, int Priority, bool Done);
```

Разбор по строкам. Строка `builder.Services.AddRouting(o => o.LowercaseUrls = true)` делает две вещи: явно показывает, что приложение осознанно использует подсистему маршрутизации, и включает lowercase-генерацию URL — это best practice для стабильных внешних ссылок. Хранилище `ConcurrentDictionary` и счётчик через `Interlocked.Increment` потокобезопасны, что важно, потому что minimal API обрабатывает запросы параллельно; обычный `List<T>` со счётчиком мог бы дать гонки идентификаторов.

Collection expression `new[] { ... }` для инициализации `ConcurrentDictionary` использует seed-данные — это новая возможность C# 12, которая делает инициализацию читаемой. `WithName("GetTaskById")` присваивает эндпоинту имя; именно это имя потом используется в `linker.GetUriByName(ctx, "GetTaskById", new { id })`. Заметьте, что `GetUriByName` принимает `HttpContext` — это даёт генератору доступ к scheme, host и path base, так что результат — полный URI вроде `http://localhost:5000/tasks/1`. Без `ctx` пришлось бы передавать `LinkOptions` и `HostString` вручную.

Constraint `:int` в `/tasks/{id:int}` — это ранняя валидация на уровне маршрута: запрос `GET /tasks/abc` просто не совпадает с этим эндпоинтом и идёт дальше по таблице, попадая в catch-all `debug` (если путь начинается с `/debug`) или возвращая 404. Это иллюстрирует ключевую мысль урока: constraints отсекают неподходящие запросы до обработчика. Constraint `:regex(^[[a-z]]+$)` в каталоге дополнительно гарантирует, что категория состоит только из строчных латинских букв; `GET /catalog/Dev` не совпадёт.

Defaults и optional реализованы через синтаксис шаблона: `{category:regex(...)=general}` задаёт значение по умолчанию `general`, а `{page:int=1}` — значение по умолчанию `1`. Оба сегмента можно опустить, и маршрут всё равно совпадёт — это именно то, что требует урок от «необязательных сегментов и значений по умолчанию». Catch-all `{*path}` зарегистрирован последним: если бы он шёл раньше `/tasks/{id:int}`, то запрос `GET /tasks/1` мог бы совпасть с `debug` (если бы шаблон был `/debug/{*path}` — нет, не мог бы, но если бы catch-all был корневым `/{{*rest}}`, то точно перекрыл бы). Этот порядок — материализация правила «специфичные раньше общих».

`Results.Created(location, created)` автоматически ставит статус 201 и заголовок `Location`, причём `location` построен через `LinkGenerator`, а не конкатенацией — это устраняет частую ошибку из урока. Pattern matching `if (category is not null)` и `with`-выражения в record-апдейте — это современные возможности C# 12, которые делают код лаконичным и типобезопасным. Наконец, метод-зависимость видна на пути `/tasks`: `MapGet` и `MapPost` сосуществуют, не конфликтуя, потому что у них разные HTTP-глаголы — ровно как описано в уроке.

#### Задания на углубление (бонус)

1. Добавьте эндпоинт `GET /tasks/{id:int}/comments/{commentId:int}` и постройте на него ссылку из `GET /tasks/{id:int}` через `LinkGenerator` (имя `GetComment`). Продемонстрируйте вложенные параметры маршрута.
2. Реализуйте ту же таблицу эндпоинтов через MVC-контроллеры с attribute routing (`[Route("tasks")]`, `[HttpGet("{id:int}")]`) и зарегистрируйте `MapControllers()`. Сравните объём кода и читаемость.
3. Добавьте endpoint-aware middleware, который через `HttpContext.GetEndpoint()` читает имя выбранного эндпоинта и логирует его; зарегистрируйте его между `UseRouting()` и эндпоинтами. Покажите, что без `UseRouting()` имя равно `null`.
4. Сконфигурируйте `AddRouting(o => o.ConstraintMap.Add("slug", typeof(SlugConstraint)))` с собственным `IRouteConstraint`, который разрешает только строки вида `kebab-case`, и примените `{slug:slug}` в маршруте. Это потребует реализации `IRouting`-инфраструктуры.

---

## Statement in English / Постановка на английском

#### Context & motivation

Imagine you are part of a team building an internal service called "TaskHub" — a lightweight task tracker for a small company. The team has decided to build the HTTP API on minimal API over .NET 8, because that approach yields compact code, an explicit endpoint table, and good performance. The architect has already pinned down several requirements: routing must be fully endpoint-based (endpoint routing), every public route must have a clear template, parameters must be validated at the route level through constraints, and links to resources must be built through `LinkGenerator` rather than string concatenation.

Additionally, the customer wants the API to support "human-readable" URLs for a task catalog (with a category and a page number), paginated and filtered access, and a service route for "everything else" (a catch-all) that returns diagnostic information — useful for early debugging and documentation. Because the API will grow, it is critically important to lock in the correct endpoint registration order right now: specific routes before general ones, so that catch-all and parameterized templates do not shadow the intended handlers.

In this assignment you will, step by step, build a working project, test the key routes manually with `curl` or an HTTP file, and then perform a refactor: remove hard-coded URLs, add named endpoints, and generate links. By the end you should have a compiling application in which every lesson topic — parameters, defaults, optional segments, constraints, named endpoints, `LinkGenerator`, ordering, method-aware routing — is represented by real code.

#### What to do step by step

1. Create a new minimal API project. Run `dotnet new web -n TaskHub -o TaskHub` inside the `modules/M13/homework` directory. Make sure `TaskHub.csproj` sets `TargetFramework` to `net8.0` and `LangVersion` to `latest` (or `12`), so that top-level statements, collection expressions, and records are available. Open the generated `Program.cs`, delete the boilerplate, and keep only `var builder = WebApplication.CreateBuilder(args);` and `var app = builder.Build();`.

2. Register the routing services explicitly via `builder.Services.AddRouting();`. Although minimal API adds basic routing automatically, an explicit call lets you configure options later (for example `AddRouting(o => o.LowercaseUrls = true)`) and shows that you own the subsystem deliberately. Understand the difference between use routing and use endpoints: in .NET 8 minimal API only needs `app.MapXxx`, but if you want to control the pipeline explicitly you would call `app.UseRouting();` before registering endpoints and `app.UseEndpoints(...)` — this matters more for MVC projects.

3. Implement the set of endpoints exactly per the specification below (detailed templates are in the "Requirements" section). All handlers must return `IResult` (`Results.Ok`, `Results.Created`, `Results.NotFound`, `Results.BadRequest`) instead of bare strings and objects — this gives explicit control over status codes and matches best practices.

4. Set up an in-memory task store. Use a `static` list or a `ConcurrentDictionary` so the homework does not drag in database infrastructure. Define `record TaskItem(int Id, string Title, string Category, int Priority, bool Done)`. Implement a simple id-generation strategy with `Interlocked.Increment` or a counter.

5. Register named endpoints for "self links". Name the `GET /tasks/{id:int}` endpoint `"GetTaskById"` via `.WithName(...)`. In that endpoint's response, return an object whose `self` field is produced by `linkGenerator.GetUriByName(ctx, "GetTaskById", new { id })`. Make sure `HttpContext ctx` and `LinkGenerator linkGenerator` are injected into the handler by minimal API's DI.

6. Implement a catalog with defaults and optional: the route `GET /catalog/{category=general}/{page:int=1}` should return the list of tasks for the given category with the page number. Verify three cases: `GET /catalog` (category=general, page=1), `GET /catalog/dev` (category=dev, page=1), `GET /catalog/dev/3` (category=dev, page=3).

7. Implement a catch-all diagnostic route `GET /debug/{*path}` AFTER all specific routes. It should return JSON with a `path` field holding the catch-all segment, plus the HTTP method and the path base. Confirm that if you register it before `/tasks/{id:int}` it starts "eating" task requests — that is the lesson's common mistake made visible.

8. Implement method-aware routes. On the path `/tasks` register `MapGet` (list) and `MapPost` (create). On `/tasks/{id:int}` register `MapGet`, `MapPut`, `MapDelete`. Show that the same path is handled by different verbs, and that `GET /tasks/abc` (where `abc` is not an integer) does not reach the `{id:int}` handler and falls into the catch-all or 404.

9. Run the app: `dotnet run --project TaskHub`. In a second terminal, test the routes with `curl`: `curl http://localhost:5000/tasks`, `curl http://localhost:5000/tasks/1`, `curl -X POST http://localhost:5000/tasks -H "Content-Type: application/json" -d '{"title":"Test","category":"dev","priority":2}'`, `curl http://localhost:5000/catalog/dev/3`, `curl http://localhost:5000/debug/any/nested/path`. Record the expected responses in a comment in `Program.cs` or in a separate `requests.http`.

10. Refactor the links. Find every place in the code where a URL is built as `$"/tasks/{id}"` or via concatenation, and replace it with `LinkGenerator.GetPathByName`/`GetUriByName`. Verify that `Results.Created` in POST uses the generated path, not a hard-coded one.

11. (Bonus) Add a `:regex` constraint for the category: `GET /catalog/{category:regex(^[[a-z]]+$)}/{page:int=1}`. Confirm that `GET /catalog/Dev` (uppercase) does not match and falls into the catch-all or 404, demonstrating early validation.

#### Requirements

The solution is a compilable .NET 8 project with a single `Program.cs` (top-level statements) and, if needed, auxiliary record types. Use C# 12: collection expressions to initialize lists (`TaskItem[] seed = [..]`), pattern matching in handlers where appropriate, raw string literals for long JSON payloads if necessary. All endpoints are registered through `Map*` minimal API methods; `MapControllerRoute` and attributes are NOT required for this homework (that belongs to MVC lessons), but the "Going deeper" section may optionally show a parallel implementation.

Every route parameter that should be typed must carry a constraint: `:int` for identifiers, `:alpha` or `:regex` for string categories, `:maxlength` for slugs. Optional segments and defaults must be declared with the `{x?}` and `{x=default}` syntax respectively — not via separate route overloads. All resource links are built exclusively through `LinkGenerator`; the presence of strings like `$"/tasks/{id}"` in responses or `Location` headers is considered a defect (except for seed data and test URLs in comments).

The catch-all route `{*path}` must be registered last among GET routes. Named endpoints must have meaningful names (`GetTaskById`, `ListTasks`, `CreateTask`), and those names must be used when generating links. Handlers must return `IResult` rather than a string or object directly, so that status codes are explicit. The app must start without errors and respond to all test requests with the correct status codes (200, 201, 404).

#### Pitfalls

The main subtlety of endpoint routing highlighted by the lesson is its two-phase nature: endpoint matching and endpoint execution are separated. Practically, this means `UseRouting()` selects the `Endpoint`, and later middleware (authorization, CORS, endpoint-aware middleware) can read the selected endpoint through `HttpContext.GetEndpoint()`. If you register authorization before `UseRouting`, it will not see the endpoint and will behave incorrectly — a typical mistake when extending the homework toward security.

Registration order matters only for competing templates. The route `GET /tasks/{id:int}` and `GET /tasks/priority/{level:alpha}` do not conflict (the `:int` constraint rejects the literal `priority`), but if you add `GET /tasks/{*rest}` too early it will shadow both `{id:int}` and `priority`. The rule is simple: specific (longer, more literal) routes go first, general (parameterized and catch-all) routes go later.

Constraints provide early validation, but they do not replace full model validation. `{id:int}` guarantees that the handler receives an integer, but it does not check the range or the existence of the entity — that check still belongs in code and returns 404. A frequent mistake is to expect `{id:int}` to "filter out" non-existent ids; it only rejects non-numeric strings. Likewise `:maxlength(50)` limits length but not content — for content you need `:regex` or separate validation.

`LinkGenerator` exposes several methods: `GetPathByName` returns only the path, `GetUriByName` returns a full URI (with scheme and host taken from `HttpContext`). It is important to pass `HttpContext` when host and scheme are needed; without it the generator may return `null` if the route data is insufficient. Another common mistake is to forget `AddRouting()` and be surprised that `LinkGenerator` is not available for injection (in practice minimal API registers it itself, but the explicit registration makes the intent readable and gives access to options).

Finally, method awareness: the same path may have several handlers for different HTTP verbs, but two `MapGet` calls with the same template will lead to ambiguity (a startup exception in some versions, or shadowing). Watch out for uniqueness of the (verb, template) pair.

#### Acceptance criteria

- [ ] The `TaskHub` project is created, `dotnet build` succeeds without warnings.
- [ ] `Program.cs` uses top-level statements and C# 12 (collection expressions visible in seed data).
- [ ] `AddRouting()` is registered explicitly; handlers use `MapGet`/`MapPost`/`MapPut`/`MapDelete`.
- [ ] `GET /tasks` (list) and `POST /tasks` (create) are implemented on the same path with different verbs.
- [ ] `GET /tasks/{id:int}` is implemented with the name `GetTaskById` and a `self` field built via `LinkGenerator`.
- [ ] `GET /catalog/{category=general}/{page:int=1}` is implemented — three cases return the expected values.
- [ ] The catch-all `GET /debug/{*path}` is implemented and registered last among GET routes.
- [ ] `GET /tasks/abc` does not reach `{id:int}` and falls into 404 or the catch-all (not a binding exception).
- [ ] Every typed parameter carries a constraint (`:int`, `:alpha`/`:regex`, `:maxlength`).
- [ ] No hard-coded URLs appear in responses or `Location` headers (only `LinkGenerator`).
- [ ] Handlers return `IResult` (`Results.Ok`/`Created`/`NotFound`/`BadRequest`), not strings.
- [ ] Registration order: specific routes before general and catch-all ones.
- [ ] `dotnet run` starts; all test `curl` requests return the correct status codes.
- [ ] POST `/tasks` returns 201 with a `Location` built via `LinkGenerator`.
- [ ] Expected responses for key routes are recorded in `Program.cs` or `requests.http`.

#### Hints (no direct answer)

- Remember that `WithName` is called on the `IEndpointConventionBuilder` returned by `MapGet`. To resolve that name elsewhere, use `LinkGenerator.GetUriByName(ctx, "GetTaskById", new { id })`.
- For seed data use a collection expression: `List<TaskItem> _store = [ new(1, "Setup", "dev", 3, false), new(2, "Docs", "docs", 2, true) ];`.
- If the `:int` constraint rejects a string and you have no catch-all, the request falls into a default 404. Decide whether you need a single diagnostic "fallback".
- `Results.Created(uri, value)` sets the `Location` header itself — just pass the generated path.
- For catch-all `{*path}` the value arrives as a single string with slashes; do not split it manually unless required.
- Keep in mind that `AddRouting(o => o.LowercaseUrls = true)` lowercases generated URLs — that may help or hinder depending on your endpoint names.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — TaskHub: endpoint routing, constraints, LinkGenerator
// Full working example. EN comments.

using System.Collections.Concurrent;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Routing;
using Microsoft.Extensions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);

// Explicit routing registration + lowercase URLs for nice links.
builder.Services.AddRouting(o => o.LowercaseUrls = true);

var app = builder.Build();

// In-memory store and id counter.
static int _nextId = 0;
int NextId() => Interlocked.Increment(ref _nextId);

// Collection expression for seed data (C# 12).
ConcurrentDictionary<int, TaskItem> _store = new(new[]
{
    new TaskItem(NextId(), "Setup repo", "dev", 3, false),
    new TaskItem(NextId(), "Write README", "docs", 2, true),
    new TaskItem(NextId(), "Plan sprint", "pm", 4, false),
});

// 1) Health endpoint.
app.MapGet("/", () => Results.Ok(new { service = "TaskHub", status = "ok" }))
   .WithName("Health");

// 2) List tasks (with optional category filter via query, NOT route).
app.MapGet("/tasks", (string? category, HttpContext ctx, LinkGenerator linker) =>
{
    // Pattern matching for filtering (C# 12).
    IEnumerable<TaskItem> items = _store.Values;
    if (category is not null)
        items = items.Where(t => t.Category == category);

    var self = linker.GetUriByName(ctx, "ListTasks", values: null);
    return Results.Ok(new { count = items.Count(), self, items = items.ToArray() });
}).WithName("ListTasks");

// 3) Get task by id — named endpoint + self link via LinkGenerator.
app.MapGet("/tasks/{id:int}", (int id, HttpContext ctx, LinkGenerator linker) =>
{
    if (!_store.TryGetValue(id, out var task))
        return Results.NotFound(new { error = $"Task {id} not found" });

    var self = linker.GetUriByName(ctx, "GetTaskById", new { id });
    return Results.Ok(new { task, self });
}).WithName("GetTaskById");

// 4) Create task — POST, Location built via LinkGenerator.
app.MapPost("/tasks", (TaskItem input, HttpContext ctx, LinkGenerator linker) =>
{
    if (string.IsNullOrWhiteSpace(input.Title))
        return Results.BadRequest(new { error = "Title is required" });

    var created = input with { Id = NextId(), Done = false };
    _store[created.Id] = created;

    var location = linker.GetUriByName(ctx, "GetTaskById", new { id = created.Id });
    return Results.Created(location, created);
}).WithName("CreateTask");

// 5) Update + delete on the same path, different HTTP verbs.
app.MapPut("/tasks/{id:int}", (int id, TaskItem input) =>
{
    if (!_store.TryGetValue(id, out var existing))
        return Results.NotFound(new { error = $"Task {id} not found" });

    var updated = existing with { Title = input.Title, Category = input.Category, Priority = input.Priority, Done = input.Done };
    _store[id] = updated;
    return Results.Ok(updated);
}).WithName("UpdateTask");

app.MapDelete("/tasks/{id:int}", (int id) =>
{
    if (!_store.TryRemove(id, out _))
        return Results.NotFound(new { error = $"Task {id} not found" });

    return Results.NoContent();
}).WithName("DeleteTask");

// 6) Catalog with defaults and optional + regex constraint on category.
//    GET /catalog             -> category=general, page=1
//    GET /catalog/dev         -> category=dev, page=1
//    GET /catalog/dev/3       -> category=dev, page=3
app.MapGet("/catalog/{category:regex(^[[a-z]]+$)=general}/{page:int=1}",
    (string category, int page) =>
{
    var items = _store.Values.Where(t => t.Category == category).ToArray();
    return Results.Ok(new { category, page, count = items.Length, items });
}).WithName("Catalog");

// 7) Catch-all — registered LAST so it does not shadow specific routes.
app.MapGet("/debug/{*path}", (string path, HttpContext ctx) =>
    Results.Ok(new { path, method = ctx.Request.Method, basePath = ctx.Request.PathBase.Value }))
   .WithName("DebugCatchAll");

app.Run();

public record TaskItem(int Id, string Title, string Category, int Priority, bool Done);
```

Line-by-line walk-through. The line `builder.Services.AddRouting(o => o.LowercaseUrls = true)` does two things: it states explicitly that the app deliberately uses the routing subsystem, and it turns on lowercase URL generation — a best practice for stable external links. The `ConcurrentDictionary` store plus an `Interlocked.Increment` counter are thread-safe, which matters because minimal API serves requests concurrently; a plain `List<T>` with a counter could produce id races.

The collection expression `new[] { ... }` initializing the `ConcurrentDictionary` uses seed data — a C# 12 feature that makes initialization readable. `WithName("GetTaskById")` assigns the endpoint a name; that name is later consumed by `linker.GetUriByName(ctx, "GetTaskById", new { id })`. Note that `GetUriByName` takes `HttpContext`, which gives the generator access to scheme, host, and path base, so the result is a full URI such as `http://localhost:5000/tasks/1`. Without `ctx` you would have to pass `LinkOptions` and a `HostString` manually.

The `:int` constraint on `/tasks/{id:int}` is early route-level validation: a `GET /tasks/abc` request simply does not match this endpoint and continues through the table, hitting the `debug` catch-all (if the path started with `/debug`) or returning 404. This illustrates the lesson's key point: constraints reject unsuitable requests before the handler runs. The `:regex(^[[a-z]]+$)` constraint on the catalog additionally guarantees that the category consists only of lowercase Latin letters; `GET /catalog/Dev` will not match.

Defaults and optional are expressed through template syntax: `{category:regex(...)=general}` sets the default value `general`, and `{page:int=1}` sets the default `1`. Both segments can be omitted and the route still matches — exactly what the lesson requires of "optional segments and defaults". The catch-all `{*path}` is registered last: had it come before `/tasks/{id:int}`, a root catch-all `/{{*rest}}` would have shadowed it. This order is the materialization of the "specific before general" rule.

`Results.Created(location, created)` automatically sets status 201 and the `Location` header, where `location` is built through `LinkGenerator` rather than concatenation — eliminating the lesson's common mistake. Pattern matching (`if (category is not null)`) and `with` expressions in the record update are modern C# 12 features that keep the code concise and type-safe. Finally, method awareness is visible on the `/tasks` path: `MapGet` and `MapPost` coexist without conflict because they use different HTTP verbs — exactly as described in the lesson.

#### Going deeper (bonus)

1. Add an endpoint `GET /tasks/{id:int}/comments/{commentId:int}` and build a link to it from `GET /tasks/{id:int}` via `LinkGenerator` (name `GetComment`). Demonstrate nested route parameters.
2. Reimplement the same endpoint table with MVC controllers using attribute routing (`[Route("tasks")]`, `[HttpGet("{id:int}")]`) and register `MapControllers()`. Compare code volume and readability.
3. Add an endpoint-aware middleware that reads the selected endpoint's name through `HttpContext.GetEndpoint()` and logs it; register it between `UseRouting()` and the endpoints. Show that without `UseRouting()` the name is `null`.
4. Configure `AddRouting(o => o.ConstraintMap.Add("slug", typeof(SlugConstraint)))` with a custom `IRouteConstraint` that only allows `kebab-case` strings, and apply `{slug:slug}` in a route. This will require touching the `IRouting` infrastructure.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект собирается и запускается без ошибок.
- [ ] (RU) Все эндпоинты реализованы согласно спецификации и названы.
- [ ] (RU) Ссылки строятся только через `LinkGenerator`.
- [ ] (RU) Записаны ожидаемые ответы на ключевые маршруты.
- [ ] (RU) Catch-all зарегистрирован последним; порядок маршрутов корректен.
- [ ] (EN) The project builds and runs without errors.
- [ ] (EN) All endpoints are implemented per the spec and named.
- [ ] (EN) Links are built only through `LinkGenerator`.
- [ ] (EN) Expected responses for key routes are recorded.
- [ ] (EN) The catch-all is registered last; the route order is correct.

#### Ресурсы / Resources
- [Microsoft Learn — Routing in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/routing)
- [Microsoft Learn — URL generation with LinkGenerator](https://learn.microsoft.com/aspnet/core/fundamentals/routing#url-generation)
- [Microsoft Learn — Route constraints reference](https://learn.microsoft.com/aspnet/core/fundamentals/routing#route-constraint-reference)
- [Microsoft Learn — Minimal APIs overview](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis)

---

[← К уроку M13-L03](lesson-M13-L03-routing-endpoints.md) | [⬆ К модулю M13](../README.md) | [Следующее ДЗ →](homework-M13-L04-di-lifetimes.md)
