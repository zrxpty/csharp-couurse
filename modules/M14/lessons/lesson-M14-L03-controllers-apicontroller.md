[⬆ К модулю M14](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M14-L03: Контроллеры, [ApiController], attribute routing / Controllers, [ApiController], attribute routing

**Модуль / Module:** M14
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

В ASP.NET Core Web API контроллер — это класс, который принимает входящие HTTP-запросы, выполняет бизнес-логику (чаще всего делегируя её сервисам) и возвращает HTTP-ответ. Контроллеры — это «парк-консьержи» вашего приложения: они не варят кофе сами, но они знают, куда отправить гостя и как ему ответить.

Базовым классом для API-контроллеров служит `ControllerBase` (а не `Controller`). `Controller` добавляет поддержку представлений (Razor), которая Web API ни к чему. `ControllerBase` предоставляет доступ к `HttpContext`, `Request`, `Response`, `User`, `ModelState` и куче вспомогательных методов вроде `Ok()`, `NotFound()`, `BadRequest()`.

Атрибут `[ApiController]`, который вешается на класс (или на сборку через `[assembly: ApiController]`), включает набор соглашений, сильно сокращающих шаблонный код:

1. **Обязательная аннотация маршрута** — каждый контроллер должен иметь `[Route]` или `[RouteBase]`. Это явное управление, а не магия.
2. **Автоматическая HTTP 400-валидация** — если модель не проходит `ModelState.IsValid`, фреймворк сам вернёт `ProblemDetails` с кодом 400. Не нужно писать `if (!ModelState.IsValid) return BadRequest(...)`.
3. **Вывод источника привязки** — сложные типы по умолчанию берутся из тела (`[FromBody]`), а примитивы — из маршрута/запроса. `[FromQuery]`/`[FromRoute]` больше не обязательно писать для простых случаев.
4. **Вывод multipart/form-data** для `IFormFile`.
5. **Подробный ProblemDetails** вместо пустого ответа при ошибках.

Attribute routing — это способ сопоставления URL с действиями через атрибуты `[Route]`, `[HttpGet]`, `[HttpPost]`, `[HttpPut]`, `[HttpDelete]`, `[HttpPatch]`. В отличие от convention-based routing, маршрут «привязан» к конкретному методу, что даёт читаемость и предсказуемость. Например:

```csharp
[Route("api/products/{id:int}")]
[HttpGet]
public ActionResult<Product> Get(int id) { ... }
```

Токен `{id:int}` накладывает ограничение: только целые числа. Существуют ограничения `min`, `max`, `length`, `regex` и другие. Маршруты могут объединяться: `[Route("api/[controller]")]` на классе + `[HttpGet("{id}")]` на методе даст `api/products/5`.

Что возвращать из действия? Три основных варианта:

- **Конкретный тип** (`IEnumerable<Product>`) — прост и удобен, но не передаёт статус-код.
- **`IActionResult`** — гибкость: можно вернуть `Ok`, `NotFound`, `CreatedAtAction`. Но тип данных теряется, и Swagger/OpenAPI не знает структуру ответа без дополнительных атрибутов.
- **`ActionResult<T>`** — компромисс: одновременно явно описывает тип данных и позволяет вернуть любой `IActionResult`. Используется чаще всего в современных API.

Аналогия: `IActionResult` — это «коробка с сюрпризом», получатель не знает, что внутри, пока не откроет. Конкретный тип — «прозрачный пакет», всё видно, но нельзя подсунуть ничего другого. `ActionResult<T>` — «коробка с этикеткой»: получатель видит тип, но при необходимости туда можно положить любой код ответа.

Сочетание `[ApiController]` + `ControllerBase` + attribute routing + `ActionResult<T>` — де-факто стандартный набор для Web API в .NET 8. Он убирает boilerplate, делает маршруты читаемыми, а ответы — самодокументируемыми.

#### Theory (EN)

In ASP.NET Core Web API, a controller is a class that receives incoming HTTP requests, runs business logic (usually by delegating to services), and returns an HTTP response. Controllers are the "lobby concierges" of your application: they don't brew the coffee themselves, but they know where to send each guest and how to reply.

The base class for API controllers is `ControllerBase`, not `Controller`. `Controller` adds Razor view support, which a Web API does not need. `ControllerBase` gives you access to `HttpContext`, `Request`, `Response`, `User`, `ModelState`, and helper methods such as `Ok()`, `NotFound()`, and `BadRequest()`.

The `[ApiController]` attribute, applied to a class (or to an assembly via `[assembly: ApiController]`), enables a set of conventions that dramatically reduce boilerplate:

1. **Attribute routing requirement** — every controller must have a `[Route]`. This is explicit, convention-driven control rather than magic.
2. **Automatic HTTP 400 validation** — if a model fails `ModelState.IsValid`, the framework itself returns a `ProblemDetails` response with status 400. No need to write `if (!ModelState.IsValid) return BadRequest(...)`.
3. **Binding source inference** — complex types come from the body by default (`[FromBody]`), primitives come from route/query. You no longer have to write `[FromQuery]`/`[FromRoute]` for trivial cases.
4. **multipart/form-data inference** for `IFormFile`.
5. **Detailed ProblemDetails** instead of empty error responses.

Attribute routing is a way to map URLs to actions using attributes such as `[Route]`, `[HttpGet]`, `[HttpPost]`, `[HttpPut]`, `[HttpDelete]`, and `[HttpPatch]`. Unlike convention-based routing, the route is attached to a specific method, which makes the code more readable and predictable. For example:

```csharp
[Route("api/products/{id:int}")]
[HttpGet]
public ActionResult<Product> Get(int id) { ... }
```

The `{id:int}` token applies a constraint: only integers are accepted. Other constraints include `min`, `max`, `length`, `regex`, and more. Routes can be combined: `[Route("api/[controller]")]` on the class plus `[HttpGet("{id}")]` on the method yields `api/products/5`.

What should an action return? There are three main options:

- **Specific type** (`IEnumerable<Product>`) — simple and convenient, but it cannot carry an arbitrary status code.
- **`IActionResult`** — flexible: you can return `Ok`, `NotFound`, `CreatedAtAction`. However, the data type is lost, and Swagger/OpenAPI cannot infer the response shape without extra attributes.
- **`ActionResult<T>`** — the compromise: it explicitly documents the data type while still allowing any `IActionResult`. This is the most common choice in modern APIs.

Analogy: `IActionResult` is a "surprise box" — the recipient does not know what is inside until they open it. A specific type is a "transparent bag" — everything is visible, but you cannot slip in anything else. `ActionResult<T>` is a "labeled box": the recipient sees the type, but when needed you can still put any status code in it.

The combination of `[ApiController]` + `ControllerBase` + attribute routing + `ActionResult<T>` is the de facto standard toolkit for Web API in .NET 8. It removes boilerplate, makes routes readable, and makes responses self-documenting.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8+
// Полный пример: контроллер товаров с CRUD и типизированными ответами.
// Full example: products controller with CRUD and typed responses.

using Microsoft.AspNetCore.Mvc;

namespace CourseShop.Api.Controllers;

// [ApiController] включает соглашения: авто-валидация 400,
// вывод источника привязки, обязательный атрибут [Route].
// [ApiController] enables conventions: automatic 400 validation,
// binding source inference, mandatory [Route] attribute.
[ApiController]
[Route("api/[controller]")] // /api/products
public class ProductsController : ControllerBase
{
    private readonly IProductService _service;
    private readonly ILogger<ProductsController> _logger;

    // Внедрение зависимостей через primary constructor (C# 12).
    // Dependency injection via primary constructor (C# 12).
    public ProductsController(IProductService service, ILogger<ProductsController> logger)
    {
        _service = service;
        _logger = logger;
    }

    // GET /api/products — список с пагинацией.
    // GET /api/products — paginated list.
    [HttpGet]
    [ProducesResponseType(typeof(IEnumerable<ProductDto>), StatusCodes.Status200OK)]
    public async Task<ActionResult<IEnumerable<ProductDto>>> GetAll(
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20,
        CancellationToken ct = default)
    {
        var items = await _service.GetPageAsync(page, pageSize, ct);
        return Ok(items); // ActionResult<T>: и тип, и статус.
                          // ActionResult<T>: both type and status.
    }

    // GET /api/products/{id} — один товар.
    // GET /api/products/{id} — single product.
    [HttpGet("{id:int}", Name = "GetProductById")]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ProductDto>> GetById(int id, CancellationToken ct = default)
    {
        var product = await _service.GetByIdAsync(id, ct);
        if (product is null)
        {
            _logger.LogWarning("Product {ProductId} not found", id);
            return NotFound(); // 404
        }

        return product; // неявный 200 OK — допустимо для ActionResult<T>.
                        // implicit 200 OK — allowed for ActionResult<T>.
    }

    // POST /api/products — создание.
    // POST /api/products — create.
    [HttpPost]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<ProductDto>> Create(
        [FromBody] CreateProductRequest request,
        CancellationToken ct = default)
    {
        // ModelState здесь проверять не нужно: [ApiController] сам вернёт 400.
        // No need to check ModelState here: [ApiController] returns 400 automatically.
        var created = await _service.CreateAsync(request, ct);

        // CreatedAtAction: 201 + Location: /api/products/42.
        // CreatedAtAction: 201 + Location header.
        return CreatedAtAction(nameof(GetById), new { id = created.Id }, created);
    }

    // PUT /api/products/{id} — полное обновление (идемпотентное).
    // PUT /api/products/{id} — full update (idempotent).
    [HttpPut("{id:int}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Update(
        int id,
        [FromBody] UpdateProductRequest request,
        CancellationToken ct = default)
    {
        // Здесь IActionResult уместен: тела у ответа нет, только статус.
        // IActionResult fits here: no body, only status.
        var success = await _service.UpdateAsync(id, request, ct);
        return success ? NoContent() : NotFound();
    }

    // DELETE /api/products/{id} — удаление.
    // DELETE /api/products/{id} — delete.
    [HttpDelete("{id:int}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(int id, CancellationToken ct = default)
    {
        var success = await _service.DeleteAsync(id, ct);
        return success ? NoContent() : NotFound();
    }
}

// Модель запроса с аннотациями данных для авто-валидации.
// Request model with data annotations for automatic validation.
public sealed record CreateProductRequest(
    [property: System.ComponentModel.DataAnnotations.Required]
    string Name,

    [property: System.ComponentModel.DataAnnotations.Range(0, 1_000_000)]
    decimal Price,

    [property: System.ComponentModel.DataAnnotations.StringLength(500)]
    string? Description);

public sealed record UpdateProductRequest(string Name, decimal Price, string? Description);

public sealed record ProductDto(int Id, string Name, decimal Price, string? Description);

public interface IProductService
{
    Task<IEnumerable<ProductDto>> GetPageAsync(int page, int pageSize, CancellationToken ct);
    Task<ProductDto?> GetByIdAsync(int id, CancellationToken ct);
    Task<ProductDto> CreateAsync(CreateProductRequest request, CancellationToken ct);
    Task<bool> UpdateAsync(int id, UpdateProductRequest request, CancellationToken ct);
    Task<bool> DeleteAsync(int id, CancellationToken ct);
}
```

#### Best Practices

- Старайтесь наследоваться от `ControllerBase`, а не `Controller`, в Web API — это убирает лишний вес и сближает код с соглашениями `[ApiController]`.
- Включайте `[ApiController]` на уровне сборки через `[assembly: ApiController]` в `Program.cs`, чтобы не дублировать атрибут на каждом контроллере.
- Используйте `ActionResult<T>` по умолчанию; `IActionResult` оставляйте для действий без тела ответа (`NoContent`, `Accepted`).
- Применяйте `[ProducesResponseType]` для каждого действия — это даёт корректную OpenAPI-документацию и помогает клиенту.
- Добавляйте `CancellationToken ct = default` в сигнатуры асинхронных действий и пробрасывайте его в сервисы и EF Core.
- Держите контроллеры тонкими: валидация входа, вызов сервиса, формирование ответа. Бизнес-логику уводите в сервисы.

- Inherit from `ControllerBase` rather than `Controller` in Web API — it removes unnecessary weight and aligns with `[ApiController]` conventions.
- Enable `[ApiController]` at the assembly level with `[assembly: ApiController]` in `Program.cs` to avoid repeating the attribute on every controller.
- Prefer `ActionResult<T>` by default; reserve `IActionResult` for actions without a response body (`NoContent`, `Accepted`).
- Apply `[ProducesResponseType]` to each action to produce correct OpenAPI documentation and help clients.
- Add `CancellationToken ct = default` to asynchronous action signatures and pass it down to services and EF Core.
- Keep controllers thin: validate input, call a service, shape the response. Push business logic into services.

#### Частые ошибки / Common Mistakes

- Наследование от `Controller` в Web API → используйте `ControllerBase`, чтобы не тянуть Razor-инфраструктуру.
- Ручная проверка `ModelState.IsValid` при включённом `[ApiController]` → это уже сделано фреймворком; не дублируйте.
- Возврат сырого объекта без `[ProducesResponseType]` → Swagger не покажет реальные коды ответов; аннотируйте действие.
- Маршруты вроде `[Route("api/products")]` + `[HttpGet("api/products/{id}")]` → дублирование. Используйте токен `[controller]` на классе и относительный шаблон на методе.
- Смешивание бизнес-логики с доступом к данным внутри контроллера → выносите в сервисы и репозитории.
- Игнорирование `CancellationToken` → долгие запросы нельзя отменить, ресурсы утекают.
- Использование `string`-id без ограничений маршрута → добавляйте `{id:int}` или `{id:guid}`, чтобы отсечь мусор на уровне роутинга.

- Inheriting from `Controller` in a Web API → use `ControllerBase` to avoid dragging in Razor infrastructure.
- Manual `ModelState.IsValid` checks with `[ApiController]` enabled → the framework already does this; do not duplicate it.
- Returning a raw object without `[ProducesResponseType]` → Swagger will not show real response codes; annotate the action.
- Routes like `[Route("api/products")]` plus `[HttpGet("api/products/{id}")]` → duplication. Use the `[controller]` token on the class and a relative template on the method.
- Mixing business logic with data access inside a controller → move it to services and repositories.
- Ignoring `CancellationToken` → long-running requests cannot be cancelled and resources leak.
- Using a string `id` without route constraints → add `{id:int}` or `{id:guid}` to filter junk at the routing layer.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Контроллер наследует `ControllerBase`, а не `Controller`.
- [ ] На классе стоит `[ApiController]` (или на сборке).
- [ ] Маршрут задан через `[Route("api/[controller]")]` и не дублирует префикс на методах.
- [ ] Действия аннотированы `[HttpGet]`/`[HttpPost]`/`[HttpPut]`/`[HttpDelete]` с шаблонами и ограничениями (`{id:int}`).
- [ ] Используется `ActionResult<T>` для действий с телом и `IActionResult` для действий без тела.
- [ ] Расставлены `[ProducesResponseType]` с корректными кодами и типами.
- [ ] Нет ручной проверки `ModelState.IsValid` (она избыточна с `[ApiController]`).
- [ ] Асинхронные методы принимают `CancellationToken` и пробрасывают его дальше.
- [ ] Бизнес-логика вынесена в сервисы; контроллер остаётся тонким.
- [ ] После `Create` возвращается `CreatedAtAction` со ссылкой на новый ресурс.

- [ ] The controller inherits from `ControllerBase`, not `Controller`.
- [ ] The class has `[ApiController]` (or the assembly does).
- [ ] The route is set via `[Route("api/[controller]")]` and the prefix is not duplicated on methods.
- [ ] Actions are annotated with `[HttpGet]`/`[HttpPost]`/`[HttpPut]`/`[HttpDelete]` with templates and constraints (`{id:int}`).
- [ ] `ActionResult<T>` is used for actions with a body and `IActionResult` for actions without one.
- [ ] `[ProducesResponseType]` is present with correct codes and types.
- [ ] There is no manual `ModelState.IsValid` check (redundant with `[ApiController]`).
- [ ] Asynchronous methods accept `CancellationToken` and pass it along.
- [ ] Business logic lives in services; the controller stays thin.
- [ ] After `Create`, `CreatedAtAction` returns a link to the new resource.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/aspnet/core/web-api/action-return-types](https://learn.microsoft.com/aspnet/core/web-api/action-return-types)

---

[⬆ К модулю M14](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
