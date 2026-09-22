[⬆ К модулю M14](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M14-L08: Swagger/OpenAPI, генерация документации / Swagger/OpenAPI, documentation generation

**Модуль / Module:** M14
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Swagger/OpenAPI — это индустриальный стандарт описания REST API. OpenAPI Specification (OAS) — это машинно-читаемый документ (JSON или YAML), который описывает эндпоинты, параметры запроса, тела запросов, схемы ответов и способы аутентификации. Swagger — это исторический бренд от компании Smartbear; когда спецификацию передали в Linux Foundation под OpenAPI Initiative, документ переименовали в OpenAPI. Имена «Swagger UI» и «Swashbuckle» сохранились, но работают они уже со стандартом OpenAPI.

Аналогия: OpenAPI — это как меню в ресторане. Меню описывает, какие блюда (эндпоинты) доступны, какие ингредиенты (параметры) можно выбрать и что вы получите на тарелке (схема ответа). Swagger UI — это официант, который поможет оформить заказ и попробовать блюдо прямо в зале, до того как вы закажете его «на кухню» продакшена.

В мире .NET есть две основные библиотеки. Swashbuckle.AspNetCore — де-факто рекомендованный Microsoft генератор: он создаёт OpenAPI-документ и встраивает интерактивный Swagger UI. NSwag от Rico Surer — инструмент шире: он умеет генерировать спецификацию, хостить UI, а также генерировать C# и TypeScript клиенты из любого OpenAPI-документа, что делает его мощным all-in-one решением для full-stack генерации кода.

Чтобы подключить Swashbuckle, обычно нужно: установить NuGet-пакет `Swashbuckle.AspNetCore`; включить XML-комментарии в файле проекта через `<GenerateDocumentationFile>true</GenerateDocumentationFile>`; зарегистрировать `AddSwaggerGen` в DI-контейнере с вызовом `IncludeXmlComments`; добавить middleware `UseSwagger` и `UseSwaggerUI` в конвейер запросов.

XML-комментарии — это основа человеко-читаемой документации. Когда вы пишете `/// <summary>` над контроллерами и методами и включаете генерацию документации, компилятор создаёт XML-файл рядом со сборкой. Swashbuckle читает этот файл и встраивает описания в OpenAPI-документ. Без XML-комментариев ваш Swagger UI выглядит «голым»: имена методов без пояснений, загадочные схемы, без примеров.

OperationId — это уникальный идентификатор операции в OpenAPI-документе. Он критичен для генерации клиентов: инструменты вроде NSwag, Kiota и OpenAPI Generator создают типизированные методы, имена которых берутся прямо из OperationId. По умолчанию Swashbuckle использует формат `ControllerName_ActionName`, но это можно переопределить кастомным `IOperationFilter` или аккуратным именованием действий. Хорошие operation id следуют шаблону глагол+сущность в camelCase: `listOrders`, `createOrder`, `getOrderById`. Стабильные operation id — часть публичного контракта API: их изменение ломает сгенерированных клиентов.

Swagger UI — это интерактивная веб-страница (обычно по адресу `/swagger`), где можно раскрыть каждый эндпоинт, заполнить параметры, нажать «Try it out» и увидеть реальный HTTP-ответ. Это бесценно для отладки и онбординга внешних потребителей. В продакшене UI обычно отключают или закрывают авторизацией.

Генерация клиентов (codegen) — это автоматическое создание кода вызова API из спецификации. Вместо ручного написания `HttpClient`-обёрток вы генерируете типизированный клиент из OpenAPI JSON. Инструменты: NSwag (отлично для .NET), Microsoft Kiota (современный, от Microsoft, многоязычный), OpenAPI Generator (JVM-экосистема). Плюсы: всегда синхронно с API, меньше ручной работы. Минусы: сгенерированный код нужно версионировать и обновлять в CI, а саму спецификацию надо воспринимать как контракт.

#### Theory (EN)

Swagger/OpenAPI is the industry standard for describing REST APIs. The OpenAPI Specification (OAS) is a machine-readable document (JSON or YAML) that describes endpoints, request parameters, request bodies, response schemas, and authentication schemes. Swagger is the historical brand from Smartbear; when the specification was donated to the Linux Foundation under the OpenAPI Initiative, the document was renamed OpenAPI. The "Swagger UI" and "Swashbuckle" names stuck around, but they now operate on the OpenAPI standard.

Analogy: OpenAPI is like a restaurant menu. The menu lists the dishes available (endpoints), the ingredients you can choose (parameters), and what arrives on your plate (response schema). Swagger UI is the waiter who walks you through the menu, takes your order, and lets you taste the dish right at the table before you commit to ordering it from the production "kitchen".

In the .NET world there are two main libraries. Swashbuckle.AspNetCore is the de-facto Microsoft-recommended generator: it produces an OpenAPI document and ships an embedded interactive Swagger UI. NSwag, by Rico Suter, is broader: it can generate the spec, host Swagger UI, and also generate C# and TypeScript client code from any OpenAPI document, making it a powerful all-in-one tool for full-stack code generation.

To wire up Swashbuckle you typically: install the `Swashbuckle.AspNetCore` NuGet package; enable XML comments in the project file with `<GenerateDocumentationFile>true</GenerateDocumentationFile>`; register `AddSwaggerGen` in the DI container with an `IncludeXmlComments` call; and add the `UseSwagger` and `UseSwaggerUI` middleware to the request pipeline.

XML comments are the backbone of human-readable documentation. When you write `/// <summary>` comments above controllers and actions and enable documentation generation, the compiler emits an XML file next to your assembly. Swashbuckle reads that file and injects the descriptions into the OpenAPI document. Without XML comments your Swagger UI looks bare: method names with no explanation, cryptic schemas, no examples.

OperationId is the unique identifier of an operation inside an OpenAPI document. It is critical for client code generation: tools like NSwag, Kiota, and OpenAPI Generator produce typed methods whose names come directly from OperationId. By default Swashbuckle uses `ControllerName_ActionName`, but you can override it with a custom `IOperationFilter` or by naming your actions carefully. Good operation ids follow a verb+noun camelCase pattern: `listOrders`, `createOrder`, `getOrderById`. Stable operation ids are part of your public API contract — changing them breaks generated clients.

Swagger UI is the interactive web page (usually at `/swagger`) where you can expand each endpoint, fill in parameters, hit "Try it out", and see the real HTTP response. It is invaluable for debugging and onboarding external consumers. In production the UI is usually disabled or gated behind authentication.

Client codegen is the automatic generation of API calling code from the specification. Instead of hand-writing `HttpClient` wrappers, you generate a typed client from the OpenAPI JSON. Tools: NSwag (excellent for .NET), Microsoft Kiota (modern, Microsoft-built, multi-language), OpenAPI Generator (JVM ecosystem). Pros: always in sync with the API, less manual plumbing. Cons: generated code must be versioned and refreshed in CI, and the spec itself must be treated as a contract.

#### Пример кода / Code Example

```xml
<!-- Api.csproj — включаем генерацию XML-документации -->
<!-- Api.csproj — enable XML documentation generation -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <!-- Включаем XML-документацию для Swashbuckle / Enable XML docs for Swashbuckle -->
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <!-- Подавляем предупреждения о缺少 комментариях на публичных членах -->
    <!-- Suppress warnings about missing comments on public members -->
    <NoWarn>$(NoWarn);1591</NoWarn>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Swashbuckle.AspNetCore" Version="6.6.2" />
  </ItemGroup>
</Project>
```

```csharp
// Program.cs — .NET 8 / C# 12
// Полная настройка Swagger/OpenAPI с XML-комментариями и кастомными OperationId
// Full Swagger/OpenAPI setup with XML comments and custom OperationId

using Microsoft.OpenApi.Models;
using Swashbuckle.AspNetCore.SwaggerGen;
using System.Reflection;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();

// 1. Регистрируем Swagger с XML-комментариями и фильтром OperationId
// 1. Register Swagger with XML comments and an OperationId filter
builder.Services.AddSwaggerGen(options =>
{
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);

    // Включаем XML-комментарии — без них UI будет «голым»
    // Enable XML comments — without them the UI is bare
    if (File.Exists(xmlPath))
    {
        options.IncludeXmlComments(xmlPath, includeControllerXmlComments: true);
    }

    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Orders API",
        Version = "v1",
        Description = "Демонстрационный API для урока M14-L08 / Demo API for lesson M14-L08"
    });

    // Кастомный фильтр: берёт OperationId из атрибута
    // Custom filter: reads OperationId from an attribute
    options.OperationFilter<OperationIdFilter>();
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    // 2. Подключаем middleware генерации спецификации и UI
    // 2. Wire up the spec generator and UI middleware
    app.UseSwagger();
    app.UseSwaggerUI(c =>
    {
        c.SwaggerEndpoint("/swagger/v1/swagger.json", "Orders API v1");
        c.RoutePrefix = "swagger"; // UI на /swagger | UI at /swagger
    });
}

app.MapControllers();
app.Run();

/// <summary>
/// Фильтр операций: задаёт стабильный OperationId из атрибута <see cref="OperationIdAttribute"/>.
/// Operation filter: sets a stable OperationId from the <see cref="OperationIdAttribute"/>.
/// </summary>
public sealed class OperationIdFilter : IOperationFilter
{
    public void Apply(OpenApiOperation operation, OperationFilterContext context)
    {
        var attr = context.MethodInfo
            .GetCustomAttributes(typeof(OperationIdAttribute), inherit: true)
            .OfType<OperationIdAttribute>()
            .FirstOrDefault();

        if (attr is not null)
        {
            operation.OperationId = attr.OperationId;
        }
    }
}

/// <summary>
/// Кастомный атрибут для задания OperationId (часть публичного контракта API).
/// Custom attribute for setting OperationId (part of the public API contract).
/// </summary>
[AttributeUsage(AttributeTargets.Method, AllowMultiple = false)]
public sealed class OperationIdAttribute(string operationId) : Attribute
{
    public string OperationId { get; } = operationId;
}
```

```csharp
// Controllers/OrdersController.cs
using Microsoft.AspNetCore.Mvc;

namespace Api.Controllers;

/// <summary>
/// Управление заказами / Orders management.
/// </summary>
[ApiController]
[Route("api/v1/orders")]
public class OrdersController : ControllerBase
{
    private static readonly List<Order> _orders = new()
    {
        new Order(1, "Book", 19.99m),
        new Order(2, "Laptop", 1299.00m),
    };

    /// <summary>
    /// Получить все заказы / Get all orders.
    /// </summary>
    /// <returns>Список заказов / List of orders.</returns>
    /// <response code="200">Возвращает список заказов / Returns the list of orders.</response>
    [HttpGet]
    [OperationId("listOrders")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    public ActionResult<IEnumerable<Order>> GetAll()
        => Ok(_orders);

    /// <summary>
    /// Получить заказ по идентификатору / Get an order by id.
    /// </summary>
    /// <param name="id">Идентификатор заказа / Order identifier.</param>
    /// <response code="200">Заказ найден / Order found.</response>
    /// <response code="404">Заказ не найден / Order not found.</response>
    [HttpGet("{id:int}")]
    [OperationId("getOrderById")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public ActionResult<Order> GetById(int id)
    {
        var order = _orders.FirstOrDefault(o => o.Id == id);
        return order is null ? NotFound() : Ok(order);
    }

    /// <summary>
    /// Создать новый заказ / Create a new order.
    /// </summary>
    /// <param name="request">Данные заказа / Order payload.</param>
    /// <response code="201">Заказ создан / Order created.</response>
    /// <response code="400">Некорректные данные / Invalid payload.</response>
    [HttpPost]
    [OperationId("createOrder")]
    [ProducesResponseType(StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public ActionResult<Order> Create([FromBody] CreateOrderRequest request)
    {
        var order = new Order(_orders.Count + 1, request.Name, request.Price);
        _orders.Add(order);
        return CreatedAtAction(nameof(GetById), new { id = order.Id }, order);
    }
}

/// <summary>Модель заказа / Order model.</summary>
/// <param name="Id">Идентификатор / Identifier.</param>
/// <param name="Name">Название / Name.</param>
/// <param name="Price">Цена / Price.</param>
public sealed record Order(int Id, string Name, decimal Price);

/// <summary>Запрос на создание заказа / Create order request.</summary>
/// <param name="Name">Название / Name.</param>
/// <param name="Price">Цена / Price.</param>
public sealed record CreateOrderRequest(string Name, decimal Price);
```

```bash
# Запуск и проверка / Run and verify:
# dotnet run
# Откройте / Open:  http://localhost:<port>/swagger
```

#### Best Practices
- Включайте `<GenerateDocumentationFile>true</GenerateDocumentationFile>` и подавляйте `CS1591` только для непубличного кода — XML-комментарии должны покрывать весь публичный API контроллеров и DTO.
- Задавайте стабильные `OperationId` через атрибут или `IOperationFilter`: они часть контракта, меняйте их осторожно.
- Закрывайте Swagger UI в продакшене (`if (app.Environment.IsDevelopment())`) или защищайте авторизацией — не публикуйте внутренний API наружу.
- Версионируйте документы (`SwaggerDoc("v1", ...)`, `"v2"`) и используйте `ApiVersioning` для одновременной поддержки нескольких версий.
- Включайте `<response code="...">` и `[ProducesResponseType]`, чтобы в UI отображались все возможные ответы, включая ошибки.
- Текруйте OpenAPI JSON в CI: генерируйте его на билде и сравнивайте с эталоном, чтобы случайно не сломать контракт коммитом.

- Enable `<GenerateDocumentationFile>true</GenerateDocumentationFile>` and suppress `CS1591` only for non-public code — XML comments must cover every public API surface of controllers and DTOs.
- Set stable `OperationId` values via an attribute or an `IOperationFilter`: they are part of the contract, change them with care.
- Gate Swagger UI in production (`if (app.Environment.IsDevelopment())`) or protect it with auth — do not expose your internal API to the world.
- Version your documents (`SwaggerDoc("v1", ...)`, `"v2"`) and pair them with `ApiVersioning` to support several versions side by side.
- Add `<response code="...">` and `[ProducesResponseType]` so the UI shows every possible response, including errors.
- Track the OpenAPI JSON in CI: generate it at build time and diff it against a baseline so a careless commit cannot silently break the contract.

#### Частые ошибки / Common Mistakes
- XML-комментарии не отображаются в UI → забыли `IncludeXmlComments` или не включили `<GenerateDocumentationFile>`. Проверьте, что путь к XML-файлу существует через `File.Exists`.
- `OperationId` дублируются или меняются между релизами → генератор клиентов ломает существующий код. Задавайте id явно атрибутом и считайте их частью контракта.
- Swagger UI доступен в продакшене → утечка внутреннего API. Оборачивайте `UseSwagger`/`UseSwaggerUI` в `IsDevelopment()` или в авторизацию.
- Параметры маршрута не попадают в спецификацию → используется `[FromBody]` там, где нужен `[FromRoute]`/`[FromQuery]`, или не указан `[Route]` с шаблоном.
- Не указаны коды ответов → UI показывает только `200`, хотя метод может вернуть `404`/`400`. Добавьте `[ProducesResponseType]` и `<response>` теги.
- Сгенерированный клиент устарел → нет шага codegen в CI. Добавьте генерацию в pipeline и пиньте версию спецификации.

- XML comments do not show up in the UI → forgot `IncludeXmlComments` or did not enable `<GenerateDocumentationFile>`. Verify the XML file path with `File.Exists`.
- `OperationId` values are duplicated or change between releases → the client generator breaks existing code. Set ids explicitly with an attribute and treat them as part of the contract.
- Swagger UI is reachable in production → internal API leak. Wrap `UseSwagger`/`UseSwaggerUI` in `IsDevelopment()` or behind auth.
- Route parameters do not appear in the spec → using `[FromBody]` where `[FromRoute]`/`[FromQuery]` is needed, or missing a `[Route]` template.
- Response codes are missing → the UI shows only `200` even though the method can return `404`/`400`. Add `[ProducesResponseType]` and `<response>` tags.
- Generated client is out of date → no codegen step in CI. Add generation to your pipeline and pin the spec version.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Установлен `Swashbuckle.AspNetCore`, зарегистрированы `AddSwaggerGen`, `UseSwagger`, `UseSwaggerUI`.
- [ ] Включён `<GenerateDocumentationFile>true</GenerateDocumentationFile>`, XML-комментарии подключены через `IncludeXmlComments`.
- [ ] Каждый публичный метод контроллера имеет `<summary>`, `<param>`, `<response>` и `[ProducesResponseType]`.
- [ ] Для операций заданы стабильные `OperationId` (атрибутом или `IOperationFilter`).
- [ ] Swagger UI отключён или защищён в продакшене, доступен только в Dev.
- [ ] OpenAPI JSON генерируется в CI и сверяется с эталоном для защиты контракта.
- [ ] Рассмотрен выбор инструмента codegen (NSwag / Kiota / OpenAPI Generator) под задачу.

- [ ] `Swashbuckle.AspNetCore` installed; `AddSwaggerGen`, `UseSwagger`, `UseSwaggerUI` registered.
- [ ] `<GenerateDocumentationFile>true</GenerateDocumentationFile>` enabled; XML comments wired via `IncludeXmlComments`.
- [ ] Every public controller action has `<summary>`, `<param>`, `<response>` and `[ProducesResponseType]`.
- [ ] Operations carry stable `OperationId` values (attribute or `IOperationFilter`).
- [ ] Swagger UI is disabled or protected in production, available only in Dev.
- [ ] The OpenAPI JSON is generated in CI and diffed against a baseline to protect the contract.
- [ ] A codegen tool (NSwag / Kiota / OpenAPI Generator) has been picked to match the task.

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/aspnet/core/tutorials/web-api-help-pages-using-swagger](https://learn.microsoft.com/aspnet/core/tutorials/web-api-help-pages-using-swagger)
- [OpenAPI Specification — https://spec.openapis.org/oas/v3.1.0](https://spec.openapis.org/oas/v3.1.0)
- [Swashbuckle на GitHub — https://github.com/domaindrivendev/Swashbuckle.AspNetCore](https://github.com/domaindrivendev/Swashbuckle.AspNetCore)
- [NSwag — https://github.com/RicoSuter/NSwag](https://github.com/RicoSuter/NSwag)
- [Microsoft Kiota — https://learn.microsoft.com/openapi/kiota/](https://learn.microsoft.com/openapi/kiota/)

---

[⬆ К модулю M14](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
