[⬆ К модулю M14](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M14-L02: Minimal API, MapGet/MapPost, groups / Minimal API, MapGet/MapPost, groups

**Модуль / Module:** M14
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Minimal API появился в .NET 6 как упрощённый способ построения HTTP-эндпоинтов без тяжёлой инфраструктуры контроллеров MVC. Вместо того чтобы писать класс-контроллер с атрибутами `[ApiController]` и `[Route]`, вы описываете маршруты прямо на `WebApplication`, вызывая методы `MapGet`, `MapPost`, `MapPut`, `MapDelete`. Это похоже на переход от большого ресторана с официантами-контроллерами к фуд-корту, где каждый прилавок сам принимает заказ и сразу отдаёт блюдо — меньше слоёв, быстрее обслуживание, но блюда всё равно качественные.

Каждый метод `Map*` принимает шаблон маршрута (например `"/todos/{id}"`) и делегат-обработчик. Обработчик — это обычная лямбда или метод, параметры которого автоматически связываются с запросом: параметры маршрута (`{id}`), строка запроса, тело (`[FromBody]` не нужен — компилятор сам понимает, что сложный тип берётся из тела), заголовки и сервисы из DI. Возвращать можно объект (он сериализуется в JSON), `IResult` или `TypedResults`. `IResult` — это абстракция результата HTTP-ответа: `Results.Ok(item)`, `Results.NotFound()`, `Results.BadRequest(error)` и так далее. Каждый такой вызов конструирует объект, который уже в точке расширения знает свой статус-код и тело.

В .NET 7 появился `TypedResults` — то же самое, но с типизированным возвращаемым значением. Если обработчик объявлен как `Results<Ok<Todo>, NotFound>` (конкретный `Results`-union), компилятор и OpenAPI-генератор точно знают возможные формы ответа. Это даёт корректную схему Swagger, проверку типов компилятором и подсказки в IDE. Рекомендуется использовать именно `TypedResults` вместе с union-типом `Results<>` — это «лучший из обоих миров»: лаконичность Minimal API и строгая типизация.

Route Groups (`MapGroup`) позволяют сгруппировать эндпоинты под общим префиксом и общими middleware/политиками. Вы пишете `var group = app.MapGroup("/api/todos");` и далее `group.MapGet("/", ...)` — маршрут станет `/api/todos/`. На группу можно навесить `WithTags`, `RequireAuthorization`, `WithOpenApi`, `AddEndpointFilter` — все они применятся к каждому эндпоинту группы. Это аналог базового маршрута в контроллерах, но гибче: группы можно вкладывать друг в друга и переиспользовать через методы расширения, вынося регистрацию группы в отдельный статический класс, например `TodoEndpoints.Map(api)`.

Endpoint filters (`AddEndpointFilter`) — это аналог middleware, но scoped к конкретному эндпоинту или группе: валидация модели, логирование, обработка исключений, проверка авторизации. Они выполняются до обработчика и могут как прервать цепочку, вернув свой `IResult`, так и вызвать `next(context)` и обработать ответ.

Важно понимать: Minimal API — это не «только для маленьких проектов». С группами, фильтрами, статическими методами-обработчиками и `TypedResults` он отлично масштабируется. Контроллеры остаются полезны для сложных сценариев с модель-биндингом, форматтерами и сложным наследованием, но для большинства REST-API Minimal API — это современный дефолт.

#### Theory (EN)

Minimal API was introduced in .NET 6 as a lightweight way to build HTTP endpoints without the heavy MVC controller infrastructure. Instead of writing a controller class decorated with `[ApiController]` and `[Route]`, you describe routes directly on a `WebApplication` by calling `MapGet`, `MapPost`, `MapPut`, and `MapDelete`. Think of it as moving from a full-service restaurant where waiters (controllers) carry orders back and forth, to a food court where each counter takes the order and hands you the dish directly — fewer layers, faster service, but the food is still properly prepared.

Each `Map*` method takes a route template (for example `"/todos/{id}"`) and a handler delegate. The handler is an ordinary lambda or method whose parameters are automatically bound from the request: route parameters (`{id}`), query string, body (no `[FromBody]` needed — the compiler infers that a complex type comes from the body), headers, and DI services. You can return a plain object (serialized to JSON), an `IResult`, or a `TypedResults` value. `IResult` is an abstraction over an HTTP response: `Results.Ok(item)`, `Results.NotFound()`, `Results.BadRequest(error)` and so on. Each call constructs an object that already knows its status code and body at the point of creation.

In .NET 7 Microsoft added `TypedResults` — the same factory methods, but with a strongly typed return value. If a handler is declared as `Results<Ok<Todo>, NotFound>` (the concrete `Results` union), the compiler and the OpenAPI generator know exactly which response shapes are possible. This yields a correct Swagger schema, compile-time type checking, and IntelliSense. The recommended pattern is to combine `TypedResults` with the `Results<>` union type — the best of both worlds: Minimal API’s brevity plus strict typing.

Route Groups (`MapGroup`) let you cluster endpoints under a common prefix and shared middleware or policies. You write `var group = app.MapGroup("/api/todos");` and then `group.MapGet("/", ...)` — the final route becomes `/api/todos/`. On a group you can call `WithTags`, `RequireAuthorization`, `WithOpenApi`, and `AddEndpointFilter`, and each applies to every endpoint in the group. This is analogous to a base route on controllers, but more flexible: groups can nest inside each other and be reused through extension methods, moving group registration into a static class such as `TodoEndpoints.Map(api)`.

Endpoint filters (`AddEndpointFilter`) are like middleware scoped to a single endpoint or group: model validation, logging, exception handling, authorization checks. They run before the handler and can either short-circuit the chain by returning their own `IResult`, or call `next(context)` and post-process the response.

A key insight: Minimal API is not “only for tiny projects.” With groups, filters, static handler methods, and `TypedResults`, it scales well. Controllers remain useful for complex scenarios involving custom model binding, formatters, or deep inheritance, but for most REST APIs Minimal API is the modern default.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8+ — Minimal API с группами маршрутов, TypedResults и фильтром
// Minimal API with route groups, TypedResults, and an endpoint filter

using Microsoft.AspNetCore.Http.HttpResults;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Routing;

var builder = WebApplication.CreateBuilder(args);

// Регистрируем сервис в DI / Register the service in DI
builder.Services.AddSingleton<TodoStore>();

var app = builder.Build();

// Группа маршрутов с общим префиксом и тегом для OpenAPI
// Route group with shared prefix and OpenAPI tag
var todos = app.MapGroup("/api/todos")
               .WithTags("Todos")
               .RequireAuthorization(); // требует авторизации для всей группы / requires auth for the whole group

// GET /api/todos — вернуть все задачи / return all todos
todos.MapGet("/", (TodoStore store) =>
{
    return TypedResults.Ok(store.GetAll());
});

// GET /api/todos/{id} — задача по идентификатору / todo by id
// Типизированный union Results<> даёт корректную схему OpenAPI
// The typed Results<> union yields a correct OpenAPI schema
todos.MapGet("/{id:int}", async Task<Results<Ok<Todo>, NotFound>> (int id, TodoStore store) =>
{
    var todo = await store.FindAsync(id);
    return todo is null
        ? TypedResults.NotFound()
        : TypedResults.Ok(todo);
});

// POST /api/todos — создать задачу / create a todo
// Простой фильтр валидации: прерывает цепочку при пустом заголовке
// Simple validation filter: short-circuits when the title is empty
todos.MapPost("/", (Todo todo, TodoStore store) =>
{
    var created = store.Add(todo);
    return TypedResults.Created($"/api/todos/{created.Id}", created);
})
.AddEndpointFilter(async (context, next) =>
{
    // context.Arguments[0] — первый параметр обработчика (Todo)
    // context.Arguments[0] is the handler's first argument (Todo)
    var arg = (Todo)context.Arguments[0]!;
    if (string.IsNullOrWhiteSpace(arg.Title))
    {
        return TypedResults.BadRequest("Title is required / Заголовок обязателен");
    }
    // Передаём управление дальше / Pass control down the chain
    return await next(context);
});

// PUT /api/todos/{id} — обновить задачу / update a todo
todos.MapPut("/{id:int}", (int id, Todo todo, TodoStore store) =>
{
    if (!store.Update(id, todo))
    {
        return Results.NotFound();
    }
    return Results.NoContent();
});

// DELETE /api/todos/{id} — удалить задачу / delete a todo
todos.MapDelete("/{id:int}", (int id, TodoStore store) =>
{
    store.Remove(id);
    return TypedResults.NoContent();
});

app.Run();

// --- Модели и «репозиторий» для примера / Models and a sample "repository" ---

public record Todo(int Id, string Title, bool Done);

public sealed class TodoStore
{
    private readonly Dictionary<int, Todo> _items = new();
    private int _seq = 0;

    public IEnumerable<Todo> GetAll() => _items.Values;

    public Task<Todo?> FindAsync(int id) =>
        Task.FromResult(_items.TryGetValue(id, out var t) ? t : null);

    public Todo Add(Todo todo)
    {
        var created = todo with { Id = ++_seq };
        _items[created.Id] = created;
        return created;
    }

    public bool Update(int id, Todo todo)
    {
        if (!_items.ContainsKey(id)) return false;
        _items[id] = todo with { Id = id };
        return true;
    }

    public void Remove(int id) => _items.Remove(id);
}
```

#### Best Practices

- Выносите регистрацию группы эндпоинтов в статические методы расширения (например `public static class TodoEndpoints { public static void Map(this IEndpointRouteBuilder app) {...} }`), чтобы `Program.cs` оставался чистым.
- Используйте `TypedResults` вместе с union-типом `Results<Ok<T>, NotFound, BadRequest>` — это даёт строгую типизацию и корректную OpenAPI-схему.
- Группируйте родственные эндпоинты через `MapGroup` и навешивайте общие политики (`RequireAuthorization`, `WithTags`, фильтры) на всю группу, а не на каждый эндпоинт.
- Применяйте `AddEndpointFilter` для сквозной логики (валидация, логирование, обработка исключений), а не дублируйте её в каждом обработчике.
- Делайте обработчики статическими методами, а не замыканиями с захваченными сервисами — так их проще тестировать и они быстрее работают.

- Extract endpoint group registration into static extension methods (e.g. `public static class TodoEndpoints { public static void Map(this IEndpointRouteBuilder app) {...} }`) to keep `Program.cs` clean.
- Use `TypedResults` together with the `Results<Ok<T>, NotFound, BadRequest>` union — this gives strong typing and a correct OpenAPI schema.
- Group related endpoints via `MapGroup` and apply shared policies (`RequireAuthorization`, `WithTags`, filters) to the whole group rather than to each endpoint.
- Use `AddEndpointFilter` for cross-cutting logic (validation, logging, exception handling) instead of duplicating it in every handler.
- Make handlers static methods instead of closures capturing services — they are easier to test and run faster.

#### Частые ошибки / Common Mistakes

- Возврат `null` из обработчика вместо `Results.NotFound()` → клиент получает 200 OK с пустым телом и не понимает, что объект не найдён. Всегда возвращайте явный `IResult` со статус-кодом.
- Путаница между `Results` (нетипизированный) и `TypedResults` (типизированный) — если объявили возвращаемый тип как `Results<Ok<T>, NotFound>`, используйте `TypedResults.Ok`/`TypedResults.NotFound`, иначе теряется типизация.
- Забыли `.WithTags("...")` на группе → в Swagger эндпоинты свалятся в кучу без категории и документация станет нечитаемой.
- Параметр маршрута `{id}` без констрейнта → возможны неожиданные строки в обработчике, который ждёт `int`. Используйте `{id:int}`.
- Захват DI-сервиса в замыкание `app.MapGet("/", () => _service.Get())` вместо параметра обработчика → ломает тестирование и область видимости сервиса. Передавайте сервис как параметр.
- Навешивание `RequireAuthorization` на отдельный эндпоинт, хотя он уже в защищённой группе → дублирование и путаница в политиках.

- Returning `null` from a handler instead of `Results.NotFound()` → the client gets 200 OK with an empty body and has no idea the object is missing. Always return an explicit `IResult` with a status code.
- Mixing up `Results` (untyped) and `TypedResults` (typed) — if you declared the return type as `Results<Ok<T>, NotFound>`, use `TypedResults.Ok` / `TypedResults.NotFound`, otherwise you lose the typing.
- Forgetting `.WithTags("...")` on a group → in Swagger the endpoints pile up without a category and the docs become unreadable.
- A route parameter `{id}` without a constraint → unexpected strings can reach a handler that expects an `int`. Use `{id:int}`.
- Capturing a DI service in a closure `app.MapGet("/", () => _service.Get())` instead of as a handler parameter → breaks testing and service lifetime scoping. Pass the service as a parameter.
- Adding `RequireAuthorization` to a single endpoint that already lives in a protected group → duplication and confusing policies.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Все CRUD-эндпоинты (GET, POST, PUT, DELETE) описаны через методы `Map*`.
- [ ] Эндпоинты сгруппированы через `MapGroup` с общим префиксом и тегом.
- [ ] Возвращаемые значения используют `TypedResults` и union-тип `Results<>` для корректной OpenAPI-схемы.
- [ ] Параметры маршрута имеют констрейнты (`{id:int}`), где это уместно.
- [ ] Сервисы получены через параметры обработчика, а не через замыкания.
- [ ] Добавлен хотя бы один `AddEndpointFilter` для валидации или логирования.
- [ ] Авторизация применена на уровне группы, а не дублирована на каждом эндпоинте.
- [ ] Регистрация группы вынесена в статический метод расширения, `Program.cs` остаётся чистым.
- [ ] Обработчики в случае «не найдено» возвращают `NotFound()`, а не `null`.

- [ ] All CRUD endpoints (GET, POST, PUT, DELETE) are declared via `Map*` methods.
- [ ] Endpoints are grouped via `MapGroup` with a shared prefix and tag.
- [ ] Return values use `TypedResults` and the `Results<>` union type for a correct OpenAPI schema.
- [ ] Route parameters carry constraints (`{id:int}`) where appropriate.
- [ ] Services are obtained through handler parameters, not closures.
- [ ] At least one `AddEndpointFilter` is added for validation or logging.
- [ ] Authorization is applied at the group level rather than duplicated on each endpoint.
- [ ] Group registration is extracted into a static extension method so `Program.cs` stays clean.
- [ ] “Not found” cases return `NotFound()` rather than `null`.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/)
- [Minimal APIs overview — https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/overview](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/overview)
- [Route groups — https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/route-handlers](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/route-handlers)
- [TypedResults — https://learn.microsoft.com/dotnet/api/microsoft.aspnetcore.http.typedresults](https://learn.microsoft.com/dotnet/api/microsoft.aspnetcore.http.typedresults)

---

[⬆ К модулю M14](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
