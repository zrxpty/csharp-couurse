[⬆ К модулю M14](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M14-L01: REST-принципы, ресурсы, методы, статус-коды / REST principles, resources, methods, status codes

**Модуль / Module:** M14
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

REST (Representational State Transfer) — это архитектурный стиль для построения распределённых веб-сервисов, описанный Роем Филдингом в 2000 году. REST — не протокол и не стандарт, а набор ограничений: клиент-сервер, отсутствие состояния (stateless), кэширование, единый интерфейс, многоуровневая система. Если API соблюдает эти ограничения, его называют RESTful.

Главное понятие REST — **ресурс**. Ресурс — это любой объект, к которому можно обратиться по URL: заказ, пользователь, статья, коллекция товаров. URL — это «адрес» ресурса, аналогично тому, как почтовый адрес указывает на дом. Имена ресурсов существительные, а не глаголы: `/orders/42` — хорошо, `/getOrder?id=42` — плохо. Коллекция ресурсов обозначается множественным числом: `/orders`, конкретный экземпляр — идентификатором: `/orders/42`.

Действия над ресурсами выражаются через **HTTP-методы**. Их часто сопоставляют с CRUD: GET — чтение (Read), POST — создание (Create), PUT — полная замена (Update), DELETE — удаление (Delete). PATCH — частичное обновление, когда меняется лишь несколько полей. Методы также различаются по **идемпотентности** и безопасности. Безопасные методы не меняют состояние сервера — это GET, HEAD, OPTIONS. Идемпотентные методы дают одинаковый результат при повторном вызове: GET, PUT, DELETE, HEAD. POST не идемпотентен: повторная отправка формы заказа создаёт второй заказ. PATCH формально не обязан быть идемпотентным, но обычно им является.

**Статус-коды** HTTP сообщают клиенту результат операции. Их делят на классы по первой цифре: 1xx — информационные, 2xx — успех (200 OK, 201 Created, 204 No Content), 3xx — перенаправление, 4xx — ошибка клиента (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity), 5xx — ошибка сервера (500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable). Важно выбирать точный код: 401 — «не представлены учётные данные», 403 — «учётные данные есть, но прав недостаточно». При создании ресурса возвращают 201 и заголовок `Location` с URL нового ресурса. При успешном удалении — 204 без тела.

**HATEOAS** (Hypermedia As The Engine Of Application State) — продвинутая часть REST: сервер возвращает вместе с ресурсом гипермедийные ссылки на доступные следующие действия. Например, ответ о заказе содержит ссылки «cancel», «pay», «ship». Клиент не обязан заранее знать URL — он следует по ссылкам, как пользователь веб-сайта. На практике чистый HATEOAS встречается редко: большинство API реализуют «REST-ish» уровень, где фиксированные URL и документация OpenAPI заменяют гипермедиа. Полезно знать HATEOAS как ориентир зрелости REST по модели Ричардсона: уровни 1 (ресурсы), 2 (HTTP-методы и статус-коды), 3 (HATEOAS).

Аналогия: REST-клиент — это посетитель библиотеки. URL — шифр книги на полке, HTTP-метод — действие (взять почитать, вернуть, заменить экземпляр), статус-код — ответ библиотекаря («книга у вас», «такой книги нет», «читальный зал закрыт»).

#### Theory (EN)

REST (Representational State Transfer) is an architectural style for building distributed web services, described by Roy Fielding in 2000. REST is neither a protocol nor a standard; it is a set of constraints: client-server, statelessness, caching, uniform interface, and layered system. When an API honours these constraints, it is called RESTful.

The central concept of REST is a **resource**. A resource is anything addressable by a URL: an order, a user, an article, a collection of products. The URL is the resource's "address", much like a postal address points to a house. Resource names are nouns, not verbs: `/orders/42` is good, `/getOrder?id=42` is bad. A collection is named in the plural: `/orders`; a single instance carries an identifier: `/orders/42`.

Actions on resources are expressed through **HTTP methods**, often mapped to CRUD: GET reads (Read), POST creates (Create), PUT fully replaces (Update), DELETE removes (Delete). PATCH performs a partial update when only a few fields change. Methods also differ by **idempotency** and safety. Safe methods do not change server state: GET, HEAD, OPTIONS. Idempotent methods produce the same result when called repeatedly: GET, PUT, DELETE, HEAD. POST is not idempotent: resubmitting an order form creates a second order. PATCH is not strictly required to be idempotent, though it usually is.

**HTTP status codes** tell the client what happened. They are grouped by the leading digit: 1xx informational, 2xx success (200 OK, 201 Created, 204 No Content), 3xx redirection, 4xx client error (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity), 5xx server error (500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable). Choosing the precise code matters: 401 means "no credentials supplied", 403 means "credentials present but insufficient rights". On resource creation, return 201 with a `Location` header pointing at the new resource. On successful deletion, return 204 with no body.

**HATEOAS** (Hypermedia As The Engine Of Application State) is the advanced part of REST: the server returns hypermedia links alongside the resource, indicating the actions currently available. For example, an order response includes links "cancel", "pay", "ship". The client does not need to know URLs in advance — it follows links, like a user browsing a website. In practice, pure HATEOAS is rare: most APIs settle at a "REST-ish" level where fixed URLs and OpenAPI documentation replace hypermedia. HATEOAS is still useful as a maturity reference, per the Richardson Maturity Model: level 1 (resources), level 2 (HTTP methods and status codes), level 3 (HATEOAS).

Analogy: a REST client is a library visitor. The URL is the call number on the shelf, the HTTP method is the action (borrow, return, replace a copy), and the status code is the librarian's reply ("here is the book", "no such book", "the reading room is closed").

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Минимальный REST API для управления заказами.
// C# 12 / .NET 8 — Minimal REST API for managing orders.
// Запуск / Run: dotnet run --project M14.OrdersApi
// Проверка / Test: curl http://localhost:5000/orders

using Microsoft.AspNetCore.Http.HttpResults;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// In-memory хранилище (демо). В реальном проекте — EF Core + база данных.
// In-memory store (demo only). In a real project use EF Core + a database.
var orders = new Dictionary<int, Order>();
var nextId = 1;

// GET /orders — список коллекции. Безопасный и идемпотентный.
// GET /orders — list the collection. Safe and idempotent.
app.MapGet("/orders", () =>
    Results.Ok(orders.Values.OrderBy(o => o.Id)))
    .WithName("ListOrders")
    .WithSummary("Возвращает все заказы / Returns all orders")
    .Produces<IReadOnlyCollection<Order>>(StatusCodes.Status200OK);

// GET /orders/{id} — один ресурс. 404, если ресурс не существует.
// GET /orders/{id} — single resource. 404 when it does not exist.
app.MapGet("/orders/{id:int}", (int id) =>
{
    return orders.TryGetValue(id, out var order)
        ? Results.Ok(order)
        : Results.NotFound(new ProblemDetails
        {
            Title = "Заказ не найден / Order not found",
            Status = StatusCodes.Status404NotFound,
            Detail = $"Order with id={id} does not exist."
        });
})
.WithName("GetOrder")
.Produces<Order>(StatusCodes.Status200OK)
.Produces<ProblemDetails>(StatusCodes.Status404NotFound);

// POST /orders — создание. Не идемпотентен: 201 Created + Location.
// POST /orders — creation. Not idempotent: 201 Created + Location.
app.MapPost("/orders", (CreateOrderRequest request, LinkGenerator links) =>
{
    var order = new Order(nextId++, request.Customer, request.Total);
    orders[order.Id] = order;

    // URL нового ресурса для заголовка Location.
    // URL of the new resource for the Location header.
    var location = links.GetUriByName(app, "GetOrder", new { id = order.Id })!;

    // 201 Created + Location + тело. HATEOAS-ссылки показывают следующие шаги.
    // 201 Created + Location + body. HATEOAS links show the next steps.
    return Results.Created(location, order with { Links = BuildLinks(order.Id) });
})
.WithName("CreateOrder")
.Produces<Order>(StatusCodes.Status201Created)
.Produces<ValidationProblem>(StatusCodes.Status400BadRequest);

// PUT /orders/{id} — полная замена. Идемпотентен.
// PUT /orders/{id} — full replacement. Idempotent.
app.MapPut("/orders/{id:int}", (int id, UpdateOrderRequest request) =>
{
    if (!orders.ContainsKey(id))
        return Results.NotFound(new ProblemDetails
        {
            Title = "Заказ не найден / Order not found",
            Status = StatusCodes.Status404NotFound
        });

    // PUT заменяет ресурс целиком, поэтому собираем новый экземпляр.
    // PUT replaces the resource entirely, so we build a fresh instance.
    orders[id] = new Order(id, request.Customer, request.Total);
    return Results.NoContent(); // 204 — успех без тела.
})
.WithName("ReplaceOrder")
.Produces(StatusCodes.Status204NoContent)
.Produces<ProblemDetails>(StatusCodes.Status404NotFound);

// PATCH /orders/{id} — частичное обновление. Идемпотентен в нашей реализации.
// PATCH /orders/{id} — partial update. Idempotent in this implementation.
app.MapPatch("/orders/{id:int}", (int id, PatchOrderRequest request) =>
{
    if (!orders.TryGetValue(id, out var existing))
        return Results.NotFound();

    // Применяем только переданные поля (JSON Merge Patch-стиль).
    // Apply only the supplied fields (JSON Merge Patch style).
    var updated = existing with
    {
        Customer = request.Customer ?? existing.Customer,
        Total = request.Total ?? existing.Total
    };
    orders[id] = updated;
    return Results.Ok(updated);
})
.WithName("PatchOrder");

// DELETE /orders/{id} — удаление. Идемпотентен: повторный вызов безопасен.
// DELETE /orders/{id} — removal. Idempotent: a repeat call is safe.
app.MapDelete("/orders/{id:int}", (int id) =>
{
    orders.Remove(id);
    return Results.NoContent(); // 204 даже если ресурса уже нет.
})
.WithName("DeleteOrder");

app.Run();

// HATEOAS-ссылки для заказа — обзорный пример уровня 3 по Ричардсону.
// HATEOAS links for an order — a level-3 Richardson example.
static IReadOnlyCollection<Link> BuildLinks(int id) =>
[
    new("self",     $"/orders/{id}", "GET"),
    new("replace",  $"/orders/{id}", "PUT"),
    new("patch",    $"/orders/{id}", "PATCH"),
    new("cancel",   $"/orders/{id}", "DELETE")
];

// Записи — reference types с record-семантикой (C# 12).
// Records — reference types with value semantics (C# 12).
public record Order(int Id, string Customer, decimal Total)
{
    public IReadOnlyCollection<Link>? Links { get; init; }
}

public record CreateOrderRequest(string Customer, decimal Total);
public record UpdateOrderRequest(string Customer, decimal Total);
public record PatchOrderRequest(string? Customer, decimal? Total);
public record Link(string Rel, string Href, string Method);
```

#### Best Practices

- Называйте ресурсы существительными во множественном числе: `/orders`, `/orders/{id}/items`. Глаголы оставьте для HTTP-методов.
- Возвращайте точные статус-коды: 201 для создания с `Location`, 204 для «без тела», 404 для отсутствующего ресурса, 409 для конфликта, 422 для ошибок валидации бизнес-правил.
- Делайте POST неидемпотентным осознанно; для операций, которые могут повторяться при сбое сети, выдавайте идемпотентный ключ (`Idempotency-Key`) и возвращайте тот же результат.
- Включайте версионирование API в URL или заголовок (`/api/v1/orders` или `Api-Version`), чтобы менять контракт без боли для существующих клиентов.
- Документируйте API через OpenAPI/Swagger и возвращайте `ProblemDetails` (RFC 7807) для ошибок — это единый формат для клиента.
- Разделяйте модель предметной области и DTO ответа: не возвращайте наружу внутренние поля и навигационные свойства.

- Name resources as plural nouns: `/orders`, `/orders/{id}/items`. Reserve verbs for HTTP methods.
- Return precise status codes: 201 on creation with `Location`, 204 for "no body", 404 for a missing resource, 409 for a conflict, 422 for business-rule validation errors.
- Keep POST non-idempotent deliberately; for operations that may be retried on network failure, accept an idempotency key (`Idempotency-Key`) and return the same result.
- Version the API in the URL or a header (`/api/v1/orders` or `Api-Version`) so you can evolve the contract without breaking existing clients.
- Document the API with OpenAPI/Swagger and return `ProblemDetails` (RFC 7807) for errors — a single format clients can rely on.
- Separate the domain model from the response DTO: do not leak internal fields or navigation properties.

#### Частые ошибки / Common Mistakes

- Глаголы в URL (`/createOrder`, `/deleteOrder`) → используйте существительные + HTTP-методы: `POST /orders`, `DELETE /orders/{id}`.
- Возврат `200 OK` для созданного ресурса без `Location` → возвращайте `201 Created` с заголовком `Location`.
- Универсальный `500` для любых ошибок → различайте ошибки клиента (4xx) и сервера (5xx), используйте 400/404/409/422.
- `PUT`, который только частично меняет поля → `PUT` заменяет ресурс целиком; для частичного изменения используйте `PATCH`.
- Документация, расходящаяся с кодом → генерируйте OpenAPI из маршрутов и держите её в CI.
- Состояние сессии на сервере (нарушение stateless) → храните состояние в токене/клиенте, сервер остаётся без состояния между запросами.

- Verbs in the URL (`/createOrder`, `/deleteOrder`) → use nouns plus HTTP methods: `POST /orders`, `DELETE /orders/{id}`.
- Returning `200 OK` for a created resource without `Location` → return `201 Created` with a `Location` header.
- A catch-all `500` for every error → distinguish client errors (4xx) from server errors (5xx); use 400/404/409/422.
- A `PUT` that only patches some fields → `PUT` replaces the resource entirely; use `PATCH` for partial changes.
- Documentation drifting from code → generate OpenAPI from routes and keep it checked in CI.
- Server-side session state (breaks statelessness) → keep state in a token or the client; the server must stay stateless between requests.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Имена ресурсов — существительные во множественном числе, без глаголов.
- [ ] GET безопасен и идемпотентен; POST — не идемпотентен; PUT/DELETE — идемпотентны.
- [ ] Создание возвращает 201 + `Location`; успешное удаление возвращает 204.
- [ ] Ошибки клиента используют точные коды 4xx, ошибки сервера — 5xx.
- [ ] Тело ошибок соответствует RFC 7807 `ProblemDetails`.
- [ ] `PUT` полностью заменяет ресурс, `PATCH` — частично.
- [ ] API документировано через OpenAPI и доступно в CI.
- [ ] Сервер не хранит состояние сессии между запросами (stateless).
- [ ] Я могу объяснить три уровня модели зрелости Ричардсона и роль HATEOAS.

- [ ] Resource names are plural nouns, with no verbs.
- [ ] GET is safe and idempotent; POST is not idempotent; PUT/DELETE are idempotent.
- [ ] Creation returns 201 + `Location`; successful deletion returns 204.
- [ ] Client errors use precise 4xx codes, server errors use 5xx.
- [ ] Error bodies follow RFC 7807 `ProblemDetails`.
- [ ] `PUT` fully replaces the resource, `PATCH` updates it partially.
- [ ] The API is documented with OpenAPI and available in CI.
- [ ] The server keeps no session state between requests (stateless).
- [ ] I can explain the three Richardson maturity levels and the role of HATEOAS.

#### Ресурсы / Resources

- Microsoft Learn — ASP.NET Core web API: https://learn.microsoft.com/aspnet/core/web-api/
- Roy Fielding, Architectural Styles and the Design of Network-based Software Architectures (Chapter 5): https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm
- RFC 9110 — HTTP Semantics (методы и статус-коды): https://www.rfc-editor.org/rfc/rfc9110
- RFC 7807 — Problem Details for HTTP APIs: https://www.rfc-editor.org/rfc/rfc7807
- Richardson Maturity Model (Martin Fowler): https://martinfowler.com/articles/richardsonMaturityModel.html

---

[⬆ К модулю M14](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
