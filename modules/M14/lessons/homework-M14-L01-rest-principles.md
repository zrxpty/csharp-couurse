---
[← К уроку M14-L01](lesson-M14-L01-rest-principles.md) | [⬆ К модулю M14](../README.md) | [Следующее ДЗ →](homework-M14-L02-minimal-api.md)
---

### Домашнее задание M14-L01: REST-принципы, ресурсы, методы, статус-коды / Homework M14-L01: REST principles, resources, methods, status codes

**Урок / Lesson:** M14-L01
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Спроектировать и реализовать RESTful-минимальный API для каталога товаров на C# 12 / .NET 8, корректно подобрав HTTP-методы, статус-коды, заголовок `Location`, тело ошибок RFC 7807 и HATEOAS-ссылки, и закрепить понятия ресурса, идемпотентности, безопасности и модели зрелости Ричардсона. (EN) Design and implement a RESTful minimal API for a product catalog on C# 12 / .NET 8, choosing HTTP methods, status codes, the `Location` header, RFC 7807 error bodies and HATEOAS links correctly, and reinforce the concepts of resource, idempotency, safety and the Richardson Maturity Model.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит ограничения REST (клиент-сервер, stateless, кэширование, единый интерфейс), понятие ресурса-существительного во множественном числе, соответствие HTTP-методов CRUD, свойства безопасности и идемпотентности, классы статус-кодов и HATEOAS по модели Ричардсона. Это ДЗ заставляет применить каждую из этих концепций на коде каталога товаров, опираясь на тот же набор best practices и избегая тех же частых ошибок (глаголы в URL, 200 вместо 201+`Location`, универсальный 500, PUT вместо PATCH, состояние сессии на сервере).
(EN) The lesson introduces the REST constraints (client-server, stateless, caching, uniform interface), the noun-in-plural resource concept, the mapping of HTTP methods to CRUD, safety and idempotency properties, status-code classes and HATEOAS per the Richardson model. This homework makes you apply every one of those concepts in product-catalog code, leaning on the same best practices and avoiding the same common mistakes (verbs in URLs, 200 instead of 201+`Location`, a catch-all 500, PUT where PATCH belongs, server-side session state).

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы начинаете разработку бэкенда каталога товаров для небольшого интернет-магазина. Команда выбрала REST как архитектурный стиль, потому что он даёт единый интерфейс, хорошо сочетается с HTTP-кэшированием, работает поверх стандартного HTTP и не привязывает вас к конкретному RPC-протоколу. Ваша задача — спроектировать первый ресурс «товар» (product) так, чтобы он соблюдал ограничения REST из урока: клиент-сервер, отсутствие состояния (stateless), кэширование, единый интерфейс, многоуровневая система. Каждый товар имеет идентификатор, название, цену, количество на складе и категорию.

Каталог должен поддерживать полный набор CRUD-операций, а также частичное обновление (например, изменение только цены). Критически важно правильно выбрать HTTP-методы и статус-коды: клиентское приложение (мобильное и веб) будет полагаться на них для корректной обработки ответов — показывать ли созданный товар, сообщать ли об ошибке валидации, повторять ли запрос. Ошибки в дизайне контракта на этом этапе обойдутся дорого: их придётся править одновременно в сервере, в клиентах и в сгенерированной OpenAPI-документации, а нарушение идемпотентности PUT/DELETE приведёт к дубликатам или потерям данных при повторах сети. Поэтому вы с самого начала закладываете версионирование (`/api/v1/products`), ProblemDetails (RFC 7807) для ошибок и HATEOAS-ссылки, чтобы клиенты могли следовать по ссылкам, а не зашивать URL. ДЗ также проверяет, что вы понимаете модель зрелости Ричардсона: уровень 1 (ресурсы), уровень 2 (методы и статус-коды), уровень 3 (HATEOAS).

#### Что нужно сделать (пошагово)
1. Создайте проект Minimal API:
   ```
   dotnet new web -n M14.ProductsApi -o M14.ProductsApi
   ```
   перейдите в папку и добавьте пакеты:
   ```
   dotnet add package Microsoft.AspNetCore.OpenApi
   dotnet add package Swashbuckle.AspNetCore
   ```
   Первый даёт генерацию OpenAPI из маршрутов, второй — опциональный Swagger UI.

2. Задайте модель товара как record C# 12:
   ```csharp
   public record Product(int Id, string Name, decimal Price, int Stock, string Category);
   ```
   Спланируйте DTO: `CreateProductRequest`, `UpdateProductRequest` (полная замена), `PatchProductRequest` с nullable-полями (стиль JSON Merge Patch). Разделение доменной модели и DTO — best practice из урока: наружу не должны попадать внутренние поля, а входные запросы не должны позволять клиенту задавать `Id`.

3. Зарегистрируйте in-memory хранилище `Dictionary<int, Product>` и счётчик `nextId`. В реальном проекте тут был бы EF Core + база данных, но для урока достаточно памяти — главное, что сервер остаётся stateless между запросами.

4. Реализуйте маршруты:
   - `GET /api/v1/products` — список коллекции, 200 OK, сортировка по `Id`. Безопасный и идемпотентный.
   - `GET /api/v1/products/{id:int}` — один ресурс; 404 + ProblemDetails, если ресурса нет.
   - `POST /api/v1/products` — создание; валидация (имя непустое, цена > 0, stock >= 0); при ошибке 400 + ValidationProblem; при успехе 201 Created + заголовок `Location` (через `LinkGenerator.GetUriByName`) + тело с HATEOAS-ссылками.
   - `PUT /api/v1/products/{id:int}` — полная замена; 404 если нет; 204 No Content без тела.
   - `PATCH /api/v1/products/{id:int}` — частичное обновление только переданных полей; 200 OK с обновлённым телом.
   - `DELETE /api/v1/products/{id:int}` — удаление; 204 даже если ресурса уже нет (идемпотентность).

5. Добавьте функцию `BuildLinks(int id)`, возвращающую коллекцию `Link` (self, replace, patch, delete) через collection expression C# 12: `[ new("self", ...), ... ]`.

6. Запустите: `dotnet run --project M14.ProductsApi`. Откройте `/openapi/v1.json` или Swagger UI и убедитесь, что маршруты и статус-коды документированы и не расходятся с кодом.

7. Протестируйте curl-запросами:
   - `curl -i http://localhost:5000/api/v1/products` → 200, JSON-массив.
   - `curl -i -X POST http://localhost:5000/api/v1/products -H "Content-Type: application/json" -d "{\"name\":\"Mug\",\"price\":12.5,\"stock\":100,\"category\":\"Kitchen\"}"` → 201, проверьте заголовок `Location`.
   - `curl -i http://localhost:5000/api/v1/products/1` → 200.
   - `curl -i http://localhost:5000/api/v1/products/999` → 404 + ProblemDetails.
   - `curl -i -X PATCH http://localhost:5000/api/v1/products/1 -H "Content-Type: application/json" -d "{\"price\":15.0}"` → 200, цена изменилась, остальные поля на месте.
   - `curl -i -X DELETE http://localhost:5000/api/v1/products/1` → 204, без тела.
   - Повторите DELETE → снова 204 (проверка идемпотентности).

8. Убедитесь, что сервер не хранит состояние сессии между запросами (stateless): каждый запрос самодостаточен, никакой кэш состояния конкретного клиента в памяти сервера не живёт между вызовами.

#### Требования к решению
- Проект собирается под .NET 8 (`dotnet build` без ошибок и предупреждений).
- Все маршруты используют существительные во множественном числе и версию `/api/v1/`.
- Используются точные статус-коды: 200, 201 + `Location`, 204, 400 (ValidationProblem), 404 (ProblemDetails), опционально 409 для конфликта (например, дубль имени).
- `PUT` полностью заменяет ресурс, `PATCH` — частично (только переданные поля).
- `DELETE` идемпотентен: повторный вызов возвращает 204.
- Тела ошибок соответствуют RFC 7807 (`ProblemDetails` / `ValidationProblem`).
- HATEOAS-ссылки присутствуют в ответах создания и получения одного ресурса.
- Доменная модель `Product` отделена от DTO запросов.
- OpenAPI-документ генерируется и содержит корректные коды ответов (`.Produces<T>(...)`).
- Код использует возможности C# 12: record, `with`-выражения, collection expressions, pattern matching, top-level statements, nullable-аннотации.
- Сервер stateless: нет сессий, нет кэширования состояния клиента между запросами.
- Ответы — UTF-8 JSON с `application/json`.

#### Тонкости и подводные камни
- Не пишите глаголы в URL (`/getProduct`, `/deleteProduct`) — это нарушение единого интерфейса. Существительное во множественном числе + HTTP-метод.
- Не возвращайте `200 OK` для созданного ресурса без `Location`. Правильно — 201 Created с заголовком `Location`, как требует урок.
- Не используйте универсальный 500 для ошибок валидации или «не найдено»: различайте 4xx (ошибка клиента) и 5xx (ошибка сервера). Для клиента — 400/404/409/422.
- Не делайте `PUT`, который меняет только часть полей — это ломает идемпотентность и семантику. `PUT` = полная замена; для частичного изменения — `PATCH`.
- Помните, что `PATCH` с `null` означает «поле не передано» (JSON Merge Patch). В нашей упрощённой модели nullable-свойство, равное `null`, трактуется как «не менять», а переданное значение — как новое. Различайте «не передано» и «явно обнулено», если ваша модель это поддерживает.
- Не храните состояние сессии на сервере — это нарушает stateless. Каждое состояние должно приезжать с запросом (токен, тело, заголовок).
- Не возвращайте внутренние поля доменной модели наружу — используйте DTO.
- Следите, чтобы OpenAPI-документация не расходилась с кодом: генерируйте её из маршрутов через `.Produces`, не пишите вручную и держите проверку в CI.
- Помните: `POST` не идемпотентен — повторная отправка создаст второй товар. В реальном проекте добавили бы заголовок `Idempotency-Key` и возвращали бы тот же результат при повторе.
- `DELETE` возвращайте 204 без тела, даже если ресурса уже нет — это сохраняет идемпотентность и не пугает клиента 404 при повторе после сбоя сети.
- 401 и 403 — разные вещи: 401 — «учётные данные не представлены», 403 — «представлены, но прав недостаточно». В этом ДЗ аутентификации нет, но держите различие в голове.
- `Location` стройте через `LinkGenerator.GetUriByName`, а не конкатенацией строк — это устойчиво к смене хоста/порта и версии маршрута.

#### Критерии приёмки
- [ ] Проект `M14.ProductsApi` собирается под .NET 8 без ошибок.
- [ ] Все URL начинаются с `/api/v1/products`, имя ресурса — существительное во множественном числе.
- [ ] `GET /api/v1/products` возвращает 200 и JSON-массив товаров.
- [ ] `GET /api/v1/products/{id}` возвращает 200 для существующего и 404 + ProblemDetails для отсутствующего.
- [ ] `POST` возвращает 201 + заголовок `Location` + тело с HATEOAS-ссылками.
- [ ] Невалидный `POST` возвращает 400 + ValidationProblem.
- [ ] `PUT` возвращает 204 и полностью заменяет ресурс; 404 если ресурса нет.
- [ ] `PATCH` меняет только переданные поля и возвращает 200.
- [ ] `DELETE` возвращает 204; повторный `DELETE` тоже 204 (идемпотентность).
- [ ] Тела всех ошибок соответствуют RFC 7807 ProblemDetails.
- [ ] OpenAPI-документ содержит корректные статус-коды для каждого маршрута.
- [ ] Код использует C# 12: records, `with`, collection expressions, top-level statements, pattern matching.
- [ ] Доменная модель отделена от DTO запросов.
- [ ] Сервер stateless — нет состояния сессии между запросами.
- [ ] Я могу объяснить соответствие методов CRUD и свойства безопасности/идемпотентности и три уровня модели Ричардсона.

#### Подсказки (без прямого ответа)
- Используйте `LinkGenerator.GetUriByName(app, "GetProduct", new { id })` для построения `Location`.
- Для HATEOAS примените collection expression: `[ new Link("self", ...), new Link("replace", ...) ]`.
- Валидацию удобно записать через pattern matching: `request is not { Name.Length: > 0, Price: > 0, Stock: >= 0 }`.
- Для 404 используйте `Results.NotFound(new ProblemDetails { Title = ..., Status = ..., Detail = ... })`.
- Не забудьте `.WithName(...)` на каждом маршруте — это имя нужно `LinkGenerator`.
- Для 400 используйте `Results.BadRequest(new ValidationProblemDetails(errors))`.
- `PUT` собирает `new Product(id, ...)` с нуля, а не `existing with { ... }` — это и есть «полная замена».

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — Каталог товаров, минимальный REST API.
// C# 12 / .NET 8 — Product catalog, minimal REST API.
// Запуск / Run: dotnet run --project M14.ProductsApi
// Проверка / Test: curl -i http://localhost:5000/api/v1/products

using Microsoft.AspNetCore.Http.HttpResults;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);

// Регистрируем генератор OpenAPI и инфраструктуру ProblemDetails (RFC 7807).
builder.Services.AddOpenApi();
builder.Services.AddProblemDetails();

var app = builder.Build();

// In-memory хранилище (демо). В реальном проекте — EF Core + база данных.
var products = new Dictionary<int, Product>();
var nextId = 1;

// GET /api/v1/products — коллекция. Безопасный и идемпотентный.
app.MapGet("/api/v1/products", () =>
        Results.Ok(products.Values.OrderBy(p => p.Id)))
    .WithName("ListProducts")
    .WithSummary("Возвращает все товары / Returns all products")
    .Produces<IReadOnlyCollection<Product>>(StatusCodes.Status200OK);

// GET /api/v1/products/{id} — один ресурс. 404 + ProblemDetails, если нет.
app.MapGet("/api/v1/products/{id:int}", (int id) =>
    {
        // Pattern matching + тернарный: ресурс или ProblemDetails.
        return products.TryGetValue(id, out var product)
            ? Results.Ok(product with { Links = BuildLinks(id) })
            : Results.NotFound(new ProblemDetails
            {
                Title = "Товар не найден / Product not found",
                Status = StatusCodes.Status404NotFound,
                Detail = $"Product with id={id} does not exist."
            });
    })
    .WithName("GetProduct")
    .Produces<Product>(StatusCodes.Status200OK)
    .Produces<ProblemDetails>(StatusCodes.Status404NotFound);

// POST /api/v1/products — создание. Не идемпотентен: 201 Created + Location.
app.MapPost("/api/v1/products", (CreateProductRequest request, LinkGenerator links) =>
    {
        // Валидация бизнес-правил через pattern matching C# 12.
        if (request is not { Name.Length: > 0, Price: > 0, Stock: >= 0 })
        {
            return Results.BadRequest(new ValidationProblemDetails(
                new Dictionary<string, string[]>
                {
                    ["request"] = ["Name must be non-empty, Price > 0, Stock >= 0."]
                })
            { Status = StatusCodes.Status400BadRequest });
        }

        var product = new Product(nextId++, request.Name, request.Price, request.Stock, request.Category);
        products[product.Id] = product;

        // URL нового ресурса для заголовка Location (без зашитых строк).
        var location = links.GetUriByName(app, "GetProduct", new { id = product.Id })!;

        // 201 Created + Location + тело с HATEOAS-ссылками (уровень 3 Ричардсона).
        return Results.Created(location, product with { Links = BuildLinks(product.Id) });
    })
    .WithName("CreateProduct")
    .Produces<Product>(StatusCodes.Status201Created)
    .Produces<ValidationProblemDetails>(StatusCodes.Status400BadRequest);

// PUT /api/v1/products/{id} — полная замена. Идемпотентен.
app.MapPut("/api/v1/products/{id:int}", (int id, UpdateProductRequest request) =>
    {
        if (!products.ContainsKey(id))
            return Results.NotFound(new ProblemDetails
            {
                Title = "Товар не найден / Product not found",
                Status = StatusCodes.Status404NotFound
            });

        // PUT заменяет ресурс целиком — собираем новый экземпляр с нуля.
        products[id] = new Product(id, request.Name, request.Price, request.Stock, request.Category);
        return Results.NoContent(); // 204 — успех без тела.
    })
    .WithName("ReplaceProduct")
    .Produces(StatusCodes.Status204NoContent)
    .Produces<ProblemDetails>(StatusCodes.Status404NotFound);

// PATCH /api/v1/products/{id} — частичное обновление. Идемпотентен в этой реализации.
app.MapPatch("/api/v1/products/{id:int}", (int id, PatchProductRequest request) =>
    {
        if (!products.TryGetValue(id, out var existing))
            return Results.NotFound();

        // JSON Merge Patch: null => «не менять», иначе — новое значение.
        var updated = existing with
        {
            Name = request.Name ?? existing.Name,
            Price = request.Price ?? existing.Price,
            Stock = request.Stock ?? existing.Stock,
            Category = request.Category ?? existing.Category
        };
        products[id] = updated;
        return Results.Ok(updated with { Links = BuildLinks(id) });
    })
    .WithName("PatchProduct");

// DELETE /api/v1/products/{id} — удаление. Идемпотентен.
app.MapDelete("/api/v1/products/{id:int}", (int id) =>
    {
        products.Remove(id); // молча, даже если уже удалён.
        return Results.NoContent(); // 204 без тела.
    })
    .WithName("DeleteProduct");

app.Run();

// HATEOAS-ссылки — collection expression C# 12.
static IReadOnlyCollection<Link> BuildLinks(int id) =>
[
    new("self",    $"/api/v1/products/{id}", "GET"),
    new("replace", $"/api/v1/products/{id}", "PUT"),
    new("patch",   $"/api/v1/products/{id}", "PATCH"),
    new("delete",  $"/api/v1/products/{id}", "DELETE")
];

// Доменная модель (record C# 12, value-семантика).
public record Product(int Id, string Name, decimal Price, int Stock, string Category)
{
    public IReadOnlyCollection<Link>? Links { get; init; }
}

// DTO запросов отделены от доменной модели — best practice урока.
public record CreateProductRequest(string Name, decimal Price, int Stock, string Category);
public record UpdateProductRequest(string Name, decimal Price, int Stock, string Category);
public record PatchProductRequest(string? Name, decimal? Price, int? Stock, string? Category);
public record Link(string Rel, string Href, string Method);
```

Разбор по строкам. `var builder = WebApplication.CreateBuilder(args);` — точка входа Minimal API через top-level statements: нет `static void Main` и ручного класса `Program` (его генерирует компилятор). Это идиома C# 12 / .NET 8. `builder.Services.AddOpenApi();` и `AddProblemDetails()` регистрируют генератор OpenAPI и инфраструктуру ошибок RFC 7807 — без явной регистрации ASP.NET Core всё равно вернёт ProblemDetails по умолчанию, но явный вызов гарантирует единый формат во всём API.

`var products = new Dictionary<int, Product>();` — in-memory хранилище; урок подчёркивает, что реальный проект использует EF Core, а счётчик `nextId` имитирует генерацию ключей базой. `app.MapGet("/api/v1/products", ...)` — коллекция, существительное во множественном числе, версия в URL. Это best practice урока: версионирование позволяет менять контракт, не ломая клиентов. GET безопасен и идемпотентен — повторный запрос не меняет состояние сервера и возвращает тот же результат.

`.WithName("GetProduct")` — имя маршрута нужно `LinkGenerator.GetUriByName`, чтобы строить `Location` без зашитых URL; это реализация единого интерфейса и подготовка к HATEOAS. `Results.NotFound(new ProblemDetails { ... })` — 404 с телом RFC 7807; урок предостерегает от универсального 500, здесь выбран точный 4xx для клиента. `Results.Created(location, ...)` — 201 Created + заголовок Location + тело; урок настаивает, что создание возвращает 201 и Location, а не «голый» 200. Тело содержит HATEOAS-ссылки `BuildLinks` — уровень 3 модели Ричардсона.

Валидация `request is not { Name.Length: > 0, Price: > 0, Stock: >= 0 }` — pattern matching C# 12, компактнее каскада `if`. При ошибке — 400 + ValidationProblem (расширение ProblemDetails со словарём ошибок по полям). `app.MapPut` строит `new Product(id, ...)` с нуля — это и есть семантика полной замены; урок строго разделяет PUT (полная замена) и PATCH (частичная). Возвращаем 204 No Content без тела. `app.MapPatch` использует `with`-выражение record: `existing with { Name = request.Name ?? existing.Name, ... }` — JSON Merge Patch, где `null` означает «не менять»; реализация идемпотентна, как отмечено в уроке. `app.MapDelete` молча вызывает `Remove` и возвращает 204 даже при отсутствии ресурса — идемпотентность, повтор безопасен. `BuildLinks` использует collection expression `[ ... ]` — новая возможность C# 12. Records дают value-семантику и работающий `with`; DTO отделены от доменной модели, поэтому клиент не может задать `Id` и не видит внутренних полей.

#### Задания на углубление (бонус)
1. Добавьте поддержку заголовка `Idempotency-Key` для `POST`: сохраняйте ключ → результат на несколько минут; при повторе возвращайте сохранённый ответ с тем же статусом и `Location`. Так `POST` остаётся неидемпотентным по семантике, но безопасным при повторах сети.
2. Реализуйте подресурс `/api/v1/products/{id}/reviews` (коллекция отзывов) — продемонстрируйте вложенные ресурсы и переход по HATEOAS от товара к его отзывам.
3. Добавьте фильтрацию, пагинацию и сортировку через query-параметры (`?category=Kitchen&page=1&size=20`) и обсудите, что уместнее вернуть: 200 с метаданными или 206 Partial Content.
4. Покройте все маршруты интеграционными тестами на `WebApplicationFactory`, проверяя статус-коды и тела ошибок, и сравните поведение `PUT` vs `PATCH` при одинаковых входных данных.

---

## Statement in English / Постановка на английском

#### Context & motivation
You are starting the backend for a small online shop's product catalog. The team has chosen REST as the architectural style because it provides a uniform interface, plays well with HTTP caching, rides on standard HTTP, and does not lock you into a specific RPC protocol. Your job is to design the first resource — a "product" — so that it honours the REST constraints from the lesson: client-server, statelessness, caching, a uniform interface, and a layered system. Each product has an identifier, a name, a price, a stock count, and a category.

The catalog must support the full CRUD set plus a partial update (for example, changing only the price). It is critical to choose HTTP methods and status codes precisely: the clients (mobile and web) will rely on them to decide whether to show a created product, surface a validation error, or retry the request. Contract mistakes made now are expensive — they have to be fixed simultaneously on the server, in the clients, and in the generated OpenAPI document, and a broken PUT/DELETE idempotency contract will cause duplicates or data loss on network retries. That is why you start with versioning (`/api/v1/products`), RFC 7807 ProblemDetails for errors, and HATEOAS links so clients can follow links rather than hard-coding URLs. The homework also checks that you understand the Richardson Maturity Model: level 1 (resources), level 2 (HTTP methods and status codes), level 3 (HATEOAS).

#### What to do step by step
1. Create a Minimal API project:
   ```
   dotnet new web -n M14.ProductsApi -o M14.ProductsApi
   ```
   move into the folder and add the packages:
   ```
   dotnet add package Microsoft.AspNetCore.OpenApi
   dotnet add package Swashbuckle.AspNetCore
   ```
   The first package gives you OpenAPI generation straight from the routes; the second adds an optional Swagger UI.

2. Model the product as a C# 12 record:
   ```csharp
   public record Product(int Id, string Name, decimal Price, int Stock, string Category);
   ```
   Plan your DTOs: `CreateProductRequest`, `UpdateProductRequest` (full replacement), `PatchProductRequest` with nullable fields (JSON Merge Patch style). Separating the domain model from the DTOs is a lesson best practice: internal fields must not leak out, and incoming requests must not let a client set the `Id`.

3. Register an in-memory store `Dictionary<int, Product>` and a `nextId` counter. A real project would use EF Core plus a database, but memory is enough for the lesson — the key point is that the server stays stateless between requests.

4. Implement the routes:
   - `GET /api/v1/products` — list the collection, 200 OK, sorted by `Id`. Safe and idempotent.
   - `GET /api/v1/products/{id:int}` — single resource; 404 + ProblemDetails when missing.
   - `POST /api/v1/products` — create; validate (non-empty name, price > 0, stock >= 0); on failure return 400 + ValidationProblem; on success return 201 Created + a `Location` header (via `LinkGenerator.GetUriByName`) + a body with HATEOAS links.
   - `PUT /api/v1/products/{id:int}` — full replacement; 404 if missing; 204 No Content with no body.
   - `PATCH /api/v1/products/{id:int}` — apply only the supplied fields; 200 OK with the updated body.
   - `DELETE /api/v1/products/{id:int}` — remove; 204 even if the resource is already gone (idempotency).

5. Add a `BuildLinks(int id)` helper returning a collection of `Link` records (self, replace, patch, delete) using a C# 12 collection expression: `[ new("self", ...), ... ]`.

6. Run: `dotnet run --project M14.ProductsApi`. Open `/openapi/v1.json` or the Swagger UI and verify the routes and status codes are documented and do not drift from the code.

7. Test with curl:
   - `curl -i http://localhost:5000/api/v1/products` → 200, JSON array.
   - `curl -i -X POST http://localhost:5000/api/v1/products -H "Content-Type: application/json" -d "{\"name\":\"Mug\",\"price\":12.5,\"stock\":100,\"category\":\"Kitchen\"}"` → 201, check the `Location` header.
   - `curl -i http://localhost:5000/api/v1/products/1` → 200.
   - `curl -i http://localhost:5000/api/v1/products/999` → 404 + ProblemDetails.
   - `curl -i -X PATCH http://localhost:5000/api/v1/products/1 -H "Content-Type: application/json" -d "{\"price\":15.0}"` → 200, price changed, other fields intact.
   - `curl -i -X DELETE http://localhost:5000/api/v1/products/1` → 204, no body.
   - Repeat DELETE → 204 again (verify idempotency).

8. Confirm the server keeps no session state between requests (stateless): each request is self-contained, and no per-client state lives in server memory between calls.

#### Requirements
- The project builds on .NET 8 (`dotnet build` with no errors or warnings).
- Every route uses a plural noun and the `/api/v1/` version prefix.
- Precise status codes are used: 200, 201 + `Location`, 204, 400 (ValidationProblem), 404 (ProblemDetails), optionally 409 for a conflict (e.g. a duplicate name).
- PUT fully replaces the resource; PATCH updates only the supplied fields.
- DELETE is idempotent: a repeated call returns 204.
- Error bodies follow RFC 7807 (ProblemDetails / ValidationProblem).
- HATEOAS links are present in creation and single-resource responses.
- The domain model `Product` is separated from request DTOs.
- The OpenAPI document is generated and lists correct response codes (`.Produces<T>(...)`).
- The code uses C# 12 features: records, `with` expressions, collection expressions, pattern matching, top-level statements, nullable annotations.
- The server is stateless: no sessions, no per-client state cached between requests.
- Responses are UTF-8 JSON with `application/json`.

#### Pitfalls
- Do not put verbs in the URL (`/getProduct`, `/deleteProduct`) — that breaks the uniform interface. Use a plural noun plus an HTTP method.
- Do not return `200 OK` for a created resource without `Location`. Return 201 Created with the header, as the lesson insists.
- Do not use a catch-all 500 for validation or "not found" errors: separate 4xx (client) from 5xx (server). Use 400/404/409/422 for client errors.
- Do not write a PUT that updates only some fields — it breaks idempotency and semantics. PUT is a full replacement; use PATCH for partial changes.
- Remember that PATCH with `null` means "field omitted" (JSON Merge Patch). In this simplified model a nullable property that is `null` is treated as "do not change", while a supplied value replaces the old one. Distinguish "omitted" from "explicitly nulled" if your model supports it.
- Do not store session state on the server — that violates statelessness. State must travel with the request (token, body, header).
- Do not return internal domain fields to the outside world — use DTOs.
- Keep the OpenAPI document in sync with code: generate it from routes via `.Produces`, never hand-write it, and keep a check in CI.
- Remember POST is not idempotent — resubmitting creates a second product. A real project would add an `Idempotency-Key` header and return the same result on retry.
- Return 204 with no body for DELETE, even if the resource is already gone — this preserves idempotency and avoids scaring the client with a 404 after a network retry.
- 401 and 403 are different: 401 means "no credentials supplied", 403 means "credentials present but insufficient rights". There is no auth in this homework, but keep the distinction in mind.
- Build `Location` through `LinkGenerator.GetUriByName`, not string concatenation — it survives changes of host, port and route version.

#### Acceptance criteria
- [ ] The `M14.ProductsApi` project builds on .NET 8 with no errors.
- [ ] Every URL starts with `/api/v1/products`; the resource name is a plural noun.
- [ ] `GET /api/v1/products` returns 200 and a JSON array of products.
- [ ] `GET /api/v1/products/{id}` returns 200 for an existing product and 404 + ProblemDetails for a missing one.
- [ ] `POST` returns 201 + a `Location` header + a body with HATEOAS links.
- [ ] An invalid `POST` returns 400 + ValidationProblem.
- [ ] `PUT` returns 204 and fully replaces the resource; 404 when missing.
- [ ] `PATCH` changes only the supplied fields and returns 200.
- [ ] `DELETE` returns 204; a repeated `DELETE` also returns 204 (idempotency).
- [ ] All error bodies follow RFC 7807 ProblemDetails.
- [ ] The OpenAPI document lists correct status codes for every route.
- [ ] The code uses C# 12: records, `with`, collection expressions, top-level statements, pattern matching.
- [ ] The domain model is separated from request DTOs.
- [ ] The server is stateless — no session state between requests.
- [ ] I can explain the CRUD-to-method mapping, the safety/idempotency properties, and the three Richardson maturity levels.

#### Hints (no direct answer)
- Use `LinkGenerator.GetUriByName(app, "GetProduct", new { id })` to build `Location`.
- For HATEOAS use a collection expression: `[ new Link("self", ...), new Link("replace", ...) ]`.
- Express validation with pattern matching: `request is not { Name.Length: > 0, Price: > 0, Stock: >= 0 }`.
- For 404 use `Results.NotFound(new ProblemDetails { Title = ..., Status = ..., Detail = ... })`.
- Remember `.WithName(...)` on every route — `LinkGenerator` needs the name.
- For 400 use `Results.BadRequest(new ValidationProblemDetails(errors))`.
- `PUT` builds `new Product(id, ...)` from scratch rather than `existing with { ... }` — that is the "full replacement" semantics.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — Product catalog, minimal REST API.
// Run: dotnet run --project M14.ProductsApi
// Test: curl -i http://localhost:5000/api/v1/products

using Microsoft.AspNetCore.Http.HttpResults;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);

// Register the OpenAPI generator and the RFC 7807 ProblemDetails infrastructure.
builder.Services.AddOpenApi();
builder.Services.AddProblemDetails();

var app = builder.Build();

// In-memory store (demo). Use EF Core + a database in real projects.
var products = new Dictionary<int, Product>();
var nextId = 1;

// GET /api/v1/products — collection. Safe and idempotent.
app.MapGet("/api/v1/products", () =>
        Results.Ok(products.Values.OrderBy(p => p.Id)))
    .WithName("ListProducts")
    .WithSummary("Returns all products")
    .Produces<IReadOnlyCollection<Product>>(StatusCodes.Status200OK);

// GET /api/v1/products/{id} — single resource. 404 + ProblemDetails if missing.
app.MapGet("/api/v1/products/{id:int}", (int id) =>
    {
        // Pattern matching + ternary: the resource or ProblemDetails.
        return products.TryGetValue(id, out var product)
            ? Results.Ok(product with { Links = BuildLinks(id) })
            : Results.NotFound(new ProblemDetails
            {
                Title = "Product not found",
                Status = StatusCodes.Status404NotFound,
                Detail = $"Product with id={id} does not exist."
            });
    })
    .WithName("GetProduct")
    .Produces<Product>(StatusCodes.Status200OK)
    .Produces<ProblemDetails>(StatusCodes.Status404NotFound);

// POST /api/v1/products — create. Not idempotent: 201 Created + Location.
app.MapPost("/api/v1/products", (CreateProductRequest request, LinkGenerator links) =>
    {
        // Business-rule validation via C# 12 pattern matching.
        if (request is not { Name.Length: > 0, Price: > 0, Stock: >= 0 })
        {
            return Results.BadRequest(new ValidationProblemDetails(
                new Dictionary<string, string[]>
                {
                    ["request"] = ["Name must be non-empty, Price > 0, Stock >= 0."]
                })
            { Status = StatusCodes.Status400BadRequest });
        }

        var product = new Product(nextId++, request.Name, request.Price, request.Stock, request.Category);
        products[product.Id] = product;

        // URL of the new resource for the Location header (no hard-coded strings).
        var location = links.GetUriByName(app, "GetProduct", new { id = product.Id })!;

        // 201 Created + Location + body with HATEOAS links (Richardson level 3).
        return Results.Created(location, product with { Links = BuildLinks(product.Id) });
    })
    .WithName("CreateProduct")
    .Produces<Product>(StatusCodes.Status201Created)
    .Produces<ValidationProblemDetails>(StatusCodes.Status400BadRequest);

// PUT /api/v1/products/{id} — full replacement. Idempotent.
app.MapPut("/api/v1/products/{id:int}", (int id, UpdateProductRequest request) =>
    {
        if (!products.ContainsKey(id))
            return Results.NotFound(new ProblemDetails
            {
                Title = "Product not found",
                Status = StatusCodes.Status404NotFound
            });

        // PUT replaces the resource entirely — build a fresh instance from scratch.
        products[id] = new Product(id, request.Name, request.Price, request.Stock, request.Category);
        return Results.NoContent(); // 204 — success with no body.
    })
    .WithName("ReplaceProduct")
    .Produces(StatusCodes.Status204NoContent)
    .Produces<ProblemDetails>(StatusCodes.Status404NotFound);

// PATCH /api/v1/products/{id} — partial update. Idempotent in this implementation.
app.MapPatch("/api/v1/products/{id:int}", (int id, PatchProductRequest request) =>
    {
        if (!products.TryGetValue(id, out var existing))
            return Results.NotFound();

        // JSON Merge Patch: null means "leave as is", otherwise use the new value.
        var updated = existing with
        {
            Name = request.Name ?? existing.Name,
            Price = request.Price ?? existing.Price,
            Stock = request.Stock ?? existing.Stock,
            Category = request.Category ?? existing.Category
        };
        products[id] = updated;
        return Results.Ok(updated with { Links = BuildLinks(id) });
    })
    .WithName("PatchProduct");

// DELETE /api/v1/products/{id} — remove. Idempotent.
app.MapDelete("/api/v1/products/{id:int}", (int id) =>
    {
        products.Remove(id); // silently, even if already removed.
        return Results.NoContent(); // 204 with no body.
    })
    .WithName("DeleteProduct");

app.Run();

// HATEOAS links — C# 12 collection expression.
static IReadOnlyCollection<Link> BuildLinks(int id) =>
[
    new("self",    $"/api/v1/products/{id}", "GET"),
    new("replace", $"/api/v1/products/{id}", "PUT"),
    new("patch",   $"/api/v1/products/{id}", "PATCH"),
    new("delete",  $"/api/v1/products/{id}", "DELETE")
];

// Domain model (C# 12 record, value semantics).
public record Product(int Id, string Name, decimal Price, int Stock, string Category)
{
    public IReadOnlyCollection<Link>? Links { get; init; }
}

// Request DTOs are separated from the domain model — lesson best practice.
public record CreateProductRequest(string Name, decimal Price, int Stock, string Category);
public record UpdateProductRequest(string Name, decimal Price, int Stock, string Category);
public record PatchProductRequest(string? Name, decimal? Price, int? Stock, string? Category);
public record Link(string Rel, string Href, string Method);
```

Line-by-line walk-through. `var builder = WebApplication.CreateBuilder(args);` is the Minimal API entry point via top-level statements: there is no hand-written `static void Main` and no manual `Program` class — the compiler synthesises them. This is the C# 12 / .NET 8 idiom. `builder.Services.AddOpenApi();` and `AddProblemDetails()` register the OpenAPI generator and the RFC 7807 error infrastructure; ASP.NET Core returns ProblemDetails by default anyway, but the explicit call guarantees a single error format across the whole API.

`var products = new Dictionary<int, Product>();` is the in-memory store — the lesson stresses that a real project uses EF Core, and `nextId` merely mimics database key generation. `app.MapGet("/api/v1/products", ...)` is the collection: a plural noun with a version prefix in the URL. Versioning is a lesson best practice because it lets you evolve the contract without breaking existing clients. GET is safe and idempotent — a repeated request does not mutate server state and yields the same result.

`.WithName("GetProduct")` gives the route a name that `LinkGenerator.GetUriByName` uses to build `Location` without hard-coded strings; this is the uniform interface in action and a step toward HATEOAS. `Results.NotFound(new ProblemDetails { ... })` returns 404 with an RFC 7807 body — the lesson warns against a catch-all 500, so a precise 4xx is used for a client error. `Results.Created(location, ...)` returns 201 Created plus the `Location` header plus a body; the lesson insists that creation returns 201 and `Location`, not a bare 200. The body carries HATEOAS links from `BuildLinks` — Richardson level 3.

Validation `request is not { Name.Length: > 0, Price: > 0, Stock: >= 0 }` uses C# 12 pattern matching and is more compact than a chain of `if` checks. On failure it returns 400 + ValidationProblem, which extends ProblemDetails with a per-field error dictionary. `app.MapPut` builds `new Product(id, ...)` from scratch — that is the full-replacement semantics; the lesson strictly separates PUT (full replace) from PATCH (partial). It returns 204 No Content with no body. `app.MapPatch` uses a record `with` expression: `existing with { Name = request.Name ?? existing.Name, ... }` — JSON Merge Patch, where `null` means "do not change"; the implementation is idempotent, as the lesson notes PATCH usually is. `app.MapDelete` silently calls `Remove` and returns 204 even when the resource is absent — idempotency, so a retry is safe. `BuildLinks` uses a collection expression `[ ... ]`, new in C# 12. Records give value semantics and a working `with`; DTOs are separated from the domain model, so a client cannot set `Id` and never sees internal fields.

#### Going deeper (bonus)
1. Add an `Idempotency-Key` header for `POST`: store key → result for a few minutes; on retry return the stored response with the same status and `Location`. POST stays non-idempotent in semantics but safe across network retries.
2. Implement a sub-resource `/api/v1/products/{id}/reviews` (a review collection) — demonstrate nested resources and a HATEOAS transition from a product to its reviews.
3. Add filtering, pagination and sorting via query parameters (`?category=Kitchen&page=1&size=20`) and discuss what to return: 200 with metadata or 206 Partial Content.
4. Cover every route with integration tests on `WebApplicationFactory`, asserting status codes and error bodies, and compare PUT vs PATCH behaviour on identical input.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект собирается под .NET 8 без ошибок и предупреждений.
- [ ] (RU) Все маршруты — `/api/v1/products`, существительное во множественном числе.
- [ ] (RU) Реализованы GET (коллекция + один), POST, PUT, PATCH, DELETE с корректными статус-кодами.
- [ ] (RU) Создание возвращает 201 + `Location` + HATEOAS; удаление — 204 (идемпотентно).
- [ ] (RU) Ошибки — RFC 7807 ProblemDetails / ValidationProblem.
- [ ] (RU) OpenAPI-документ сгенерирован и не расходится с кодом.
- [ ] (RU) Код использует C# 12 (records, `with`, collection expressions, pattern matching, top-level statements).
- [ ] (RU) Доменная модель отделена от DTO; сервер stateless.
- [ ] (EN) The project builds on .NET 8 with no errors or warnings.
- [ ] (EN) All routes are `/api/v1/products`, plural noun.
- [ ] (EN) GET (collection + single), POST, PUT, PATCH, DELETE are implemented with correct status codes.
- [ ] (EN) Creation returns 201 + `Location` + HATEOAS; deletion returns 204 (idempotent).
- [ ] (EN) Errors follow RFC 7807 ProblemDetails / ValidationProblem.
- [ ] (EN) The OpenAPI document is generated and in sync with the code.
- [ ] (EN) The code uses C# 12 (records, `with`, collection expressions, pattern matching, top-level statements).
- [ ] (EN) The domain model is separated from DTOs; the server is stateless.

#### Ресурсы / Resources
- Microsoft Learn — ASP.NET Core web API: https://learn.microsoft.com/aspnet/core/web-api/
- Microsoft Learn — Minimal APIs overview: https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/overview
- RFC 9110 — HTTP Semantics (methods and status codes): https://www.rfc-editor.org/rfc/rfc9110
- RFC 7807 — Problem Details for HTTP APIs: https://www.rfc-editor.org/rfc/rfc7807
- Roy Fielding, Architectural Styles and the Design of Network-based Software Architectures (Chapter 5): https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm
- Richardson Maturity Model (Martin Fowler): https://martinfowler.com/articles/richardsonMaturityModel.html
