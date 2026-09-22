[⬆ К модулю M14](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M14-L04: Model binding, [FromBody]/[FromQuery]/[FromRoute] / Model binding, [FromBody]/[FromQuery]/[FromRoute]

**Модуль / Module:** M14
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Model binding — это «переводчик» между миром HTTP-запроса (текст, байты, заголовки) и миром C#-объектов (сильно типизированные классы). Представьте официанта в ресторане: гость делает заказ на непонятном языке (query string, JSON в теле, параметр в URL), а официант превращает этот заказ в понятную кухне записку — объект C#, который понимает ваш контроллер. Без model binding вам пришлось бы вручную парсить `Request.QueryString`, читать `Request.Body` через `StreamReader` и приводить строки к `int`, `DateTime`, `Guid` — утомительно и ошибкоопасно.

ASP.NET Core делает это автоматически через **источники привязки (binding sources)**. Каждый параметр метода действия контроллера по умолчанию пытается найти данные в нескольких местах в строгом порядке: сначала form values, затем route values, затем query string. Если параметр помечен атрибутом-указателем, binder ищет значение только в указанном месте — это снижает неоднозначность и ошибки безопасности.

Основные атрибуты:
- `[FromQuery]` — берёт значение из строки запроса (`?name=Ivan&age=30`). Идеально для фильтров, пагинации, сортировки.
- `[FromRoute]` — берёт значение из маршрута (`/api/users/42`). Идеально для идентификаторов ресурса.
- `[FromBody]` — десериализует тело запроса (обычно JSON) в объект. Используется один раз на метод, потому что тело — поток, который можно прочитать лишь однажды.
- `[FromHeader]` — берёт значение из HTTP-заголовка. Полезно для `Authorization`, `Accept-Language`, `X-Request-Id`.
- `[FromForm]` — берёт значения из отправленной формы (multipart/form-data или url-encoded).
- `[FromServices]` — берёт сервис из DI-контейнера, а не из запроса.

Правила хорошего тона: для простых типов (`int`, `string`, `Guid`) явно указывайте источник, чтобы избежать сюрпризов; для сложных DTO в POST/PUT используйте `[FromBody]`; для запросов-фильтров со множеством параметров создавайте отдельный query DTO и помечайте его `[FromQuery]`. Не смешивайте `[FromBody]` с другим источником тела.

**Валидация** идёт рука об руку с привязкой. После того как binder построил объект, MVC запускает валидаторы: DataAnnotations (`[Required]`, `[Range]`, `[StringLength]`), `IValidatableObject`, а также FluentValidation при подключении. Результат накапливается в `ModelState.IsValid`. В .NET 8 с minimal APIs используется `IValidatableObject` и `Validator` вручную, а в контроллерах MVC достаточно проверить `ModelState.IsValid` перед работой.

**Кастомные биндеры.** Когда встроенной логики не хватает (например, вы хотите привязывать сложный объект из нескольких заголовков или десериализовать специфический формат), реализуйте `IModelBinder` и зарегистрируйте его через `IModelBinderProvider` в `MvcOptions.ModelBinderProviders`. Это продвинутый сценарий — приберегите его для случаев, когда стандартные атрибуты реально не справляются.

Аналогия: стандартная привязка — это «автоматическая коробка передач»: работает для 95% случаев. Кастомный биндер — «ручная коробка»: даёт контроль, но требует мастерства. Сначала убедитесь, что автомат действительно не справляется, потом переходите на ручное управление.

#### Theory (EN)

Model binding is the "translator" between the HTTP request world (text, bytes, headers) and the C# object world (strongly typed classes). Picture a waiter in a restaurant: a guest places an order in an unknown language (query string, JSON body, URL parameter), and the waiter turns that order into a note the kitchen understands — a C# object your controller can work with. Without model binding you would manually parse `Request.QueryString`, read `Request.Body` through a `StreamReader`, and convert strings to `int`, `DateTime`, `Guid` — tedious and error-prone.

ASP.NET Core does this automatically through **binding sources**. Every controller action parameter, by default, tries to find data in several places in a strict order: form values first, then route values, then query string. If a parameter is decorated with a source attribute, the binder looks for the value only in the specified place — this reduces ambiguity and security mistakes.

Main attributes:
- `[FromQuery]` — takes the value from the query string (`?name=Ivan&age=30`). Perfect for filters, pagination, sorting.
- `[FromRoute]` — takes the value from the route (`/api/users/42`). Perfect for resource identifiers.
- `[FromBody]` — deserializes the request body (usually JSON) into an object. Used at most once per method, because the body is a stream that can be read only once.
- `[FromHeader]` — takes the value from an HTTP header. Useful for `Authorization`, `Accept-Language`, `X-Request-Id`.
- `[FromForm]` — takes values from a submitted form (multipart/form-data or url-encoded).
- `[FromServices]` — resolves a service from the DI container, not from the request.

Good-taste rules: for simple types (`int`, `string`, `Guid`) always state the source explicitly to avoid surprises; for complex DTOs in POST/PUT use `[FromBody]`; for filter requests with many parameters create a dedicated query DTO and mark it `[FromQuery]`. Do not mix `[FromBody]` with another body source.

**Validation** goes hand in hand with binding. After the binder builds the object, MVC runs validators: DataAnnotations (`[Required]`, `[Range]`, `[StringLength]`), `IValidatableObject`, and FluentValidation when wired up. The result accumulates in `ModelState.IsValid`. In .NET 8 minimal APIs you use `IValidatableObject` and `Validator` manually, while in MVC controllers a simple `ModelState.IsValid` check before work is enough.

**Custom binders.** When built-in logic is not enough (for example, you want to bind a complex object from several headers or deserialize a specific format), implement `IModelBinder` and register it via `IModelBinderProvider` in `MvcOptions.ModelBinderProviders`. This is an advanced scenario — save it for cases where standard attributes genuinely cannot cope.

Analogy: standard binding is an "automatic transmission" — works for 95% of cases. A custom binder is a "manual transmission" — gives control but demands skill. First prove the automatic really cannot handle your case, then switch to manual.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8+, ASP.NET Core MVC/Web API
// Рабочий пример: источники привязки, валидация, кастомный биндер.

using System.ComponentModel.DataAnnotations;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.ModelBinding;
using Microsoft.Extensions.DependencyInjection;

// 1) DTO для создания пользователя — приходит в теле (JSON) / DTO for user creation — comes in body (JSON)
public record CreateUserDto(
    [Required] string Name,
    [EmailAddress] string Email,
    [Range(18, 120)] int Age);

// 2) Query DTO для фильтрации списка / Query DTO for list filtering
public record UserQueryDto(
    [FromQuery(Name = "q")] string? SearchTerm,           // ?q=ivan / ?q=ivan
    [FromQuery(Name = "page")] int Page = 1,                // ?page=2 (значение по умолчанию / default)
    [FromQuery(Name = "size")] [Range(1, 100)] int PageSize = 20);

// 3) Контроллер с разными источниками привязки / Controller with different binding sources
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    // GET api/users/42 — идентификатор из маршрута / id from route
    [HttpGet("{id:guid}")]
    public IActionResult GetById([FromRoute] Guid id)
        => Ok(new { id, name = "Demo User" });

    // GET api/users?q=ivan&page=2 — фильтр из query / filter from query
    [HttpGet]
    public IActionResult List([FromQuery] UserQueryDto query)
        => Ok(new { query.SearchTerm, query.Page, query.PageSize });

    // POST api/users — тело запроса (JSON) / request body (JSON)
    [HttpPost]
    public IActionResult Create([FromBody] CreateUserDto dto)
    {
        // ModelState проверяется автоматически в [ApiController], но для наглядности:
        // ModelState is checked automatically with [ApiController], shown for clarity:
        if (!ModelState.IsValid)
            return BadRequest(ModelState);

        return CreatedAtAction(nameof(GetById), new { id = Guid.NewGuid() }, dto);
    }

    // GET api/users/me — значение из заголовка / value from header
    [HttpGet("me")]
    public IActionResult Me([FromHeader(Name = "X-User-Id")] Guid userId)
        => Ok(new { userId });
}

// 4) Кастомный биндер: собирает объект из двух заголовков / Custom binder: builds object from two headers
public record TenantContext(Guid TenantId, string Region);

public class TenantContextBinder : IModelBinder
{
    public Task BindModelAsync(ModelBindingContext bindingContext)
    {
        var req = bindingContext.HttpContext.Request;
        var tenantRaw = req.Headers["X-Tenant-Id"].FirstOrDefault();
        var region = req.Headers["X-Region"].FirstOrDefault();

        if (!Guid.TryParse(tenantRaw, out var tenantId) || string.IsNullOrWhiteSpace(region))
        {
            bindingContext.Result = ModelBindingResult.Failed();
            return Task.CompletedTask;
        }

        bindingContext.Result = ModelBindingResult.Success(new TenantContext(tenantId, region));
        return Task.CompletedTask;
    }
}

// 5) Провайдер, связывающий тип TenantContext с биндером / Provider linking TenantContext to the binder
public class TenantContextBinderProvider : IModelBinderProvider
{
    public IModelBinder? GetBinder(ModelBinderProviderContext context)
        => context.Metadata.ModelType == typeof(TenantContext)
            ? new TenantContextBinder()
            : null;
}

// 6) Регистрация в Program.cs / Registration in Program.cs
// builder.Services.AddControllers(options =>
// {
//     options.ModelBinderProviders.Insert(0, new TenantContextBinderProvider());
// });

// Использование в действии / Usage in an action:
// [HttpGet("tenant-info")]
// public IActionResult TenantInfo([ModelBinder(typeof(TenantContextBinder))] TenantContext ctx)
//     => Ok(ctx);
```

#### Best Practices
- Для простых типов (`int`, `Guid`, `string`) всегда указывайте источник явно — это убирает неоднозначность и упрощает чтение кода.
- Для POST/PUT с JSON используйте `[FromBody]`, для фильтров-списков — отдельный query DTO с `[FromQuery]`, для идентификаторов ресурса — `[FromRoute]`.
- Не используйте `[FromBody]` больше одного раза на метод: тело запроса читается один раз.
- Проверяйте `ModelState.IsValid` (в `[ApiController]` это автоматически возвращает 400, но в MVC-представлениях — нет).
- Кастомные биндеры создавайте только когда стандартные атрибуты реально не справляются — иначе вы добавляете сложность без выгоды.

- For simple types (`int`, `Guid`, `string`) always state the source explicitly — it removes ambiguity and makes the code easier to read.
- For POST/PUT with JSON use `[FromBody]`, for list filters a dedicated query DTO with `[FromQuery]`, for resource identifiers `[FromRoute]`.
- Never use `[FromBody]` more than once per method: the request body is read once.
- Check `ModelState.IsValid` (with `[ApiController]` this auto-returns 400, but in MVC views it does not).
- Build custom binders only when standard attributes genuinely cannot cope — otherwise you add complexity for no gain.

#### Частые ошибки / Common Mistakes
- `[FromBody]` стоит на простом типе (`int`) и приходит пустое тело → 400. Используйте `[FromQuery]`/`[FromRoute]` для простых значений.
- Одинаковые имена параметра в query и route → binder берёт значение из неожиданного места. Указывайте атрибут источника явно.
- Забыли `[ApiController]` и не проверили `ModelState.IsValid` → невалидные данные уходят в бизнес-логику. Всегда проверяйте валидность.
- Параметр с `[FromBody]` без конструктора/сеттеров для JSON-десериализации → null. Используйте record или класс с публичными свойствами.
- Кастомный биндер не зарегистрирован в `ModelBinderProviders` → binder не вызывается. Регистрируйте провайдер в `Program.cs`.

- `[FromBody]` is on a simple type (`int`) and the body is empty → 400. Use `[FromQuery]`/`[FromRoute]` for simple values.
- Same parameter name in both query and route → the binder picks the value from an unexpected place. State the source explicitly.
- Forgot `[ApiController]` and did not check `ModelState.IsValid` → invalid data reaches business logic. Always validate.
- A `[FromBody]` parameter without a constructor/setters for JSON deserialization → null. Use a record or a class with public properties.
- A custom binder is not registered in `ModelBinderProviders` → the binder never runs. Register the provider in `Program.cs`.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я знаю порядок источников привязки по умолчанию (form → route → query).
- [ ] Я понимаю, когда использовать `[FromBody]`, `[FromQuery]`, `[FromRoute]`, `[FromHeader]`.
- [ ] Я могу объяснить, почему `[FromBody]` используется только один раз на метод.
- [ ] Я умею создавать отдельный query DTO для фильтров и помечать его `[FromQuery]`.
- [ ] Я знаю, как проверить `ModelState.IsValid` и что `[ApiController]` делает это автоматически.
- [ ] Я могу реализовать `IModelBinder` + `IModelBinderProvider` для кастомного сценария.
- [ ] Я понимаю роль валидации: DataAnnotations, `IValidatableObject`, FluentValidation.

- [ ] I know the default binding source order (form → route → query).
- [ ] I understand when to use `[FromBody]`, `[FromQuery]`, `[FromRoute]`, `[FromHeader]`.
- [ ] I can explain why `[FromBody]` is used only once per method.
- [ ] I can create a dedicated query DTO for filters and mark it `[FromQuery]`.
- [ ] I know how to check `ModelState.IsValid` and that `[ApiController]` does this automatically.
- [ ] I can implement `IModelBinder` + `IModelBinderProvider` for a custom scenario.
- [ ] I understand the role of validation: DataAnnotations, `IValidatableObject`, FluentValidation.

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/aspnet/core/mvc/models/model-binding](https://learn.microsoft.com/aspnet/core/mvc/models/model-binding)

---

[⬆ К модулю M14](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
