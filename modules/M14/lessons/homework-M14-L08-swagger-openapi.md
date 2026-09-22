---
[← К уроку M14-L08](lesson-M14-L08-swagger-openapi.md) | [⬆ К модулю M14](../README.md) | [Следующее ДЗ →](homework-M14-L09-cors-ratelimiting.md)
---

### Домашнее задание M14-L08: Swagger/OpenAPI, генерация документации / Homework M14-L08: Swagger/OpenAPI, documentation generation

**Урок / Lesson:** M14-L08
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться подключать Swashbuckle к ASP.NET Core 8 API, включать XML-комментарии, задавать стабильные OperationId, описывать коды ответов и параметры, валидировать OpenAPI-контракт в CI и генерировать типизированный клиент. (EN) Learn to wire Swashbuckle into an ASP.NET Core 8 API, enable XML comments, assign stable OperationIds, describe response codes and parameters, validate the OpenAPI contract in CI, and generate a typed client.

#### Связь с уроком / Connection to the lesson
(RU) Урок M14-L08 вводит индустриальный стандарт OpenAPI, показывает два основных .NET-генератора (Swashbuckle и NSwag) и связку «XML-комментарии → Swagger-документ → Swagger UI → кодогенерация клиента». ДЗ закрепляет все эти шаги на реальном мини-проекте Orders API: вы пройдёте путь от пустого `Program.cs` до CI-шага, который сравнивает `swagger.json` с эталоном.
(EN) Lesson M14-L08 introduces the OpenAPI industry standard, shows the two main .NET generators (Swashbuckle and NSwag), and the chain "XML comments → Swagger document → Swagger UI → client codegen". This homework reinforces every step on a real mini-project, Orders API: you go from an empty `Program.cs` to a CI step that diffs `swagger.json` against a baseline.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы — backend-разработчик в команде, которая отгружает внутренний Orders API для нескольких фронтенд-команд и внешних партнёров. Сейчас API работает, но его потребители жалуются: документация в Confluence устарела, имена эндпоинтов в Swagger нечитаемые (`Orders_GetAll`, `Orders_GetById`), в UI не видны ни возможные коды ошибок, ни примеры тел, а клиентские прокси ребята пишут руками и каждое изменение контракта превращается в лавину правок. Лид поручил вам навести порядок: внедрить Swashbuckle, подключить XML-комментарии, задать стабильные `OperationId`, описать все коды ответов, скрыть UI в продакшене, а в CI добавить шаг, который генерирует `swagger.json` и падает, если контракт изменился без повышения версии.

OpenAPI здесь — это не «красивая страничка», а машино-читаемый контракт, который становится источником правды: Swagger UI для ручного тестирования, Kiota или NSwag для типизированного клиента, а pipeline-проверка — как защита от случайного breaking change. После выполнения ДЗ у вас будет воспроизводимый шаблон, который можно перенести на любой другой API: одна команда `dotnet test` скажет, сломали вы контракт или нет, ещё до того, как код попадёт в main.

В уроке подчёркнуто: XML-комментарии — основа человеко-читаемой документации; `OperationId` — часть публичного контракта, его смена ломает сгенерированных клиентов; UI в продакшене нужно закрывать. Эти принципы вы и будете воплощать в коде.

#### Что нужно сделать (пошагово)
1. Создайте новый проект: `dotnet new webapi -n OrdersApi -o OrdersApi --use-controllers`. Перейдите в каталог: `cd OrdersApi`. Убедитесь, что `TargetFramework` равен `net8.0`, а `Nullable` и `ImplicitUsings` включены.
2. Установите пакет Swashbuckle: `dotnet add package Swashbuckle.AspNetCore --version 6.6.2`. В уроке упоминается именно этот генератор как Microsoft-рекомендованный.
3. Откройте `OrdersApi.csproj` и добавьте в существующий `<PropertyGroup>` две строки: `<GenerateDocumentationFile>true</GenerateDocumentationFile>` и `<NoWarn>$(NoWarn);1591</NoWarn>`. Первая заставляет компилятор эммитить XML-файл комментариев рядом со сборкой, вторая подавляет предупреждение CS1591 о пропущенных комментариях на непубличных членах.
4. Замените содержимое `Program.cs` на конфигурацию с `AddSwaggerGen`: зарегистрируйте `AddControllers` и `AddEndpointsApiExplorer`, вызовите `AddSwaggerGen`, внутри которого через `Assembly.GetExecutingAssembly().GetName().Name` сформируйте имя XML-файла (`OrdersApi.xml`), соберите полный путь через `Path.Combine(AppContext.BaseDirectory, xmlFile)` и, если `File.Exists(xmlPath)` истинно, вызовите `options.IncludeXmlComments(xmlPath, includeControllerXmlComments: true)`. Задайте `SwaggerDoc("v1", new OpenApiInfo { Title = "Orders API", Version = "v1", Description = "..." })`. Зарегистрируйте кастомный `OperationIdFilter` через `options.OperationFilter<OperationIdFilter>()`.
5. В middleware-секции оберните `UseSwagger` и `UseSwaggerUI` в `if (app.Environment.IsDevelopment())`. В `UseSwaggerUI` задайте `c.SwaggerEndpoint("/swagger/v1/swagger.json", "Orders API v1")` и `c.RoutePrefix = "swagger"`. Это закроет UI в продакшене — ровно то, чего требует best practice из урока.
6. Создайте папку `Controllers` и файл `OrdersController.cs`. Контроллер должен иметь атрибуты `[ApiController]` и `[Route("api/v1/orders")]`. Реализуйте три действия: `GetAll` (`[HttpGet]`), `GetById(int id)` (`[HttpGet("{id:int}")]`), `Create([FromBody] CreateOrderRequest request)` (`[HttpPost]`). Над каждым действием добавьте `/// <summary>` с двуязычным описанием, `/// <param>` для параметров и `/// <response code="...">` для каждого кода. На каждом действии поставьте атрибут `[OperationId("listOrders")]` / `[OperationId("getOrderById")]` / `[OperationId("createOrder")]` — именно эти имена станут публичным контрактом для будущих клиентов.
7. Добавьте `[ProducesResponseType(StatusCodes.Status200OK)]` на `GetAll`, `[ProducesResponseType(StatusCodes.Status200OK)]` и `[ProducesResponseType(StatusCodes.Status404NotFound)]` на `GetById`, `[ProducesResponseType(StatusCodes.Status201Created)]` и `[ProducesResponseType(StatusCodes.Status400BadRequest)]` на `Create`. В `Create` возвращайте `CreatedAtAction(nameof(GetById), new { id = order.Id }, order)`.
8. Опишите модели `Order(int Id, string Name, decimal Price)` и `CreateOrderRequest(string Name, decimal Price)` как `sealed record` с XML-комментариями через `/// <param>` над каждой записью — так описания попадут в схему в Swagger UI.
9. Реализуйте `OperationIdAttribute` (primary constructor `OperationIdAttribute(string operationId)`, `AttributeUsage(AttributeTargets.Method)`) и `OperationIdFilter : IOperationFilter`, который в `Apply` читает атрибут через `context.MethodInfo.GetCustomAttributes(typeof(OperationIdAttribute), inherit: true).OfType<OperationIdAttribute>().FirstOrDefault()` и, если атрибут найден, присваивает `operation.OperationId = attr.OperationId`.
10. Запустите: `dotnet run`. Откройте `http://localhost:<port>/swagger`. Проверьте: описания на русском и английском видны под именами действий, в каждом эндпоинте развёрнуты коды `200/201/404/400`, в схемах `Order` и `CreateOrderRequest` заполнены описания полей, а OperationId равны `listOrders`, `getOrderById`, `createOrder`. Скачайте `swagger.json` через ссылку в UI и убедитесь, что поле `operationId` присутствует у каждой операции.
11. Добавьте CI-проверку контракта: создайте консольный проект `ContractVerifier` (или xUnit-тест), который через `SwaggerDocumentSerializer` или ручной вызов `WebApplicationFactory<Program>` получает `swagger.json`, сохраняет его в `baselines/swagger.v1.json` (если файла нет — создаёт) и сравнивает сериализованный JSON с эталоном, падая при отличиях. Упрощённый вариант — в тесте вызвать `/swagger/v1/swagger.json` и сравнить с файлом `baselines/swagger.v1.json`, коммитимым в репозиторий. Запустите `dotnet test` — первый прогон создаст baseline, второй должен пройти зелёным.

#### Требования к решению
- Целевой фреймворк `net8.0`, язык C# 12: используйте top-level statements в `Program.cs`, primary constructor в `OperationIdAttribute`, pattern matching (`is not null`), `sealed record` для DTO. Collection expressions и raw string literals применяйте там, где они повышают читаемость (например, многострочное описание в `OpenApiInfo.Description`).
- Пакет `Swashbuckle.AspNetCore` версии 6.6.2 (или новее в рамках 6.x). Все публичные действия контроллера покрыты XML-комментариями `<summary>`, `<param>`, `<response>`.
- `OperationId` заданы явно через атрибут и фильтр, имена соответствуют шаблону verb+noun в camelCase: `listOrders`, `getOrderById`, `createOrder`.
- Все возможные коды ответов описаны через `[ProducesResponseType]` и `<response>`: `200`, `201`, `400`, `404`.
- `UseSwagger`/`UseSwaggerUI` вызываются только в `IsDevelopment()`. В продакшене UI недоступен.
- Параметры маршрута корректны: `id` — `[FromRoute]` (через шаблон `{id:int}`), тело — `[FromBody]`. В спецификации они должны появиться в нужных секциях `parameters` и `requestBody`.
- Контракт проверяется в CI: тест или скрипт сравнивает `swagger.json` с эталоном и падает при расхождении. Baseline-файл закоммичен.
- Код компилируется без предупреждений (кроме подавленного CS1591). `dotnet build` выходит с кодом 0.

#### Тонкости и подводные камни
- XML-комментарии не отображаются в UI — самая частая ошибка. Причины: забыли `<GenerateDocumentationFile>true</GenerateDocumentationFile>`, забыли `IncludeXmlComments`, либо путь к XML-файлу вычислен неверно. Имя XML-файла совпадает с именем сборки (`OrdersApi.xml`, не `OrdersApi.csproj.xml`), лежит в `AppContext.BaseDirectory`. Всегда оборачивайте `IncludeXmlComments` в `File.Exists(xmlPath)` — на машинах без ребилда файла может не быть.
- `OperationId` по умолчанию у Swashbuckle — `ControllerName_ActionName`, например `Orders_GetAll`. Это нестабильно: переименование контроллера ломает всех клиентов. Явный атрибут + фильтр фиксирует контракт. Никогда не меняйте `OperationId` между минорными релизами — это breaking change для сгенерированных клиентов NSwag/Kiota.
- Swagger UI в продакшене — утечка внутреннего API. Обязательно `if (app.Environment.IsDevelopment())`. Если UI нужен в проде для партнёров — закрывайте авторизацией, а не публикуйте открыто.
- Параметры не попадают в спецификацию, если перепутаны источники привязки. `[FromBody]` для `id` создаст `requestBody` вместо `parameters`; `[FromRoute]` для тела — ошибка. Шаблон маршрута `{id:int}` важен: без `:int` constraint параметр всё равно попадёт в spec, но потеряется тип-ограничение.
- Коды ответов. Без `[ProducesResponseType]` UI показывает только дефолтный `200`. Добавляйте каждый реальный код: `404` для «не найдено», `400` для невалидного тела, `201` для создания. Дублируйте текстом `<response code="404">Заказ не найден</response>` — это улучшает UX UI.
- CI-проверка контракта. Сравнивать `swagger.json` «как есть» нельзя — Swashbuckle может менять порядок ключей. Сериализуйте через `JsonSerializer.Serialize` с `WriteIndented = true` и стабильным `JsonSerializerOptions`, либо нормализуйте через `Microsoft.OpenApi.Readers` и `OpenApiSpecWriter`. Закоммитьте baseline в репозиторий, и любой PR, меняющий контракт, упадёт в CI до мержа.
- `NoWarn` для CS1591 — подавляйте только если понимаете, что часть публичного кода документировать не нужно. Иначе вы «слепнете» к пропущенным комментариям. Лучше документировать всё публичное.
- Codegen: NSwag хорош для .NET-клиентов, Kiota — современный Microsoft-инструмент с поддержкой многих языков, OpenAPI Generator — JVM-экосистема. Выбор зависит от стека потребителя. Сгенерированный код тоже версионнируется и обновляется в CI, не руками.

#### Критерии приёмки
- [ ] Проект `OrdersApi` создан, `Swashbuckle.AspNetCore` 6.6.2 установлен.
- [ ] В `.csproj` включены `GenerateDocumentationFile` и подавление `CS1591`.
- [ ] `Program.cs` регистрирует `AddSwaggerGen` с `IncludeXmlComments` через `File.Exists`.
- [ ] `SwaggerDoc("v1", ...)` задан с `Title`, `Version`, `Description`.
- [ ] `OperationIdFilter` зарегистрирован через `options.OperationFilter<OperationIdFilter>()`.
- [ ] `UseSwagger` и `UseSwaggerUI` обёрнуты в `IsDevelopment()`, `RoutePrefix = "swagger"`.
- [ ] `OrdersController` имеет `[ApiController]`, `[Route("api/v1/orders")]`, три действия.
- [ ] Каждое действие покрыто `<summary>`, `<param>`, `<response>` и `[ProducesResponseType]`.
- [ ] `OperationId` равны `listOrders`, `getOrderById`, `createOrder` (проверено в `swagger.json`).
- [ ] `Order` и `CreateOrderRequest` — `sealed record` с XML-комментариями на поля.
- [ ] `Create` возвращает `CreatedAtAction` с `201`.
- [ ] В UI видны коды `200/201/400/404` и описания полей схем.
- [ ] CI-тест/скрипт сравнивает `swagger.json` с `baselines/swagger.v1.json`, падает при расхождении.
- [ ] `dotnet build` без предупреждений, `dotnet test` зелёный.
- [ ] В отчёте описан выбор codegen-инструмента (NSwag/Kiota/OpenAPI Generator) под сценарий.

#### Подсказки (без прямого ответа)
- Имя XML-файла — это имя сборки плюс `.xml`, а не имя проекта. Проверьте через `Assembly.GetExecutingAssembly().GetName().Name`.
- Для фильтра операций используйте `IOperationFilter.Apply(OpenApiOperation, OperationFilterContext)` и ищите атрибут через отражение на `context.MethodInfo`.
- В тестах контракта удобен `WebApplicationFactory<Program>` из `Microsoft.AspNetCore.Mvc.Testing` — он поднимает `TestServer` и позволяет дернуть `/swagger/v1/swagger.json` по HTTP.
- Нормализация JSON перед сравнением спасает от «плавающих» порядка ключей. Подумайте о `JsonSerializerOptions { WriteIndented = true, DefaultIgnoreCondition = WhenWritingNull }`.
- Для codegen-сравнения вспомните аналогию из урока: OpenAPI — это меню, а Swagger UI — официант. Какой инструмент будет вашим «шеф-поваром», генерирующим клиента?

#### Эталонное решение (разбор)
```csharp
// OrdersApi/Program.cs — .NET 8 / C# 12
// Полная настройка Swagger/OpenAPI с XML-комментариями, стабильным OperationId и защитой UI в продакшене
// Full Swagger/OpenAPI setup with XML comments, stable OperationId, and production UI gating

using Microsoft.OpenApi.Models;
using Swashbuckle.AspNetCore.SwaggerGen;
using System.Reflection;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();

// 1. Swagger + XML-комментарии + фильтр OperationId / Swagger + XML comments + OperationId filter
builder.Services.AddSwaggerGen(options =>
{
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);

    // Включаем XML-комментарии — без них UI «голый» / Enable XML comments or the UI is bare
    if (File.Exists(xmlPath))
    {
        options.IncludeXmlComments(xmlPath, includeControllerXmlComments: true);
    }

    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Orders API",
        Version = "v1",
        Description = """
            Демонстрационный API для урока M14-L08.
            Demo API for lesson M14-L08.
            """
    });

    // Стабильный OperationId — часть публичного контракта / Stable OperationId is part of the contract
    options.OperationFilter<OperationIdFilter>();
});

var app = builder.Build();

// 2. UI только в Dev — не публикуем внутренний API / Dev-only UI — do not expose the internal API
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(c =>
    {
        c.SwaggerEndpoint("/swagger/v1/swagger.json", "Orders API v1");
        c.RoutePrefix = "swagger";
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
// OrdersApi/Controllers/OrdersController.cs
using Microsoft.AspNetCore.Mvc;

namespace OrdersApi.Controllers;

/// <summary>
/// Управление заказами / Orders management.
/// </summary>
[ApiController]
[Route("api/v1/orders")]
public sealed class OrdersController : ControllerBase
{
    private static readonly List<Order> _orders =
    [
        new(1, "Book", 19.99m),
        new(2, "Laptop", 1299.00m),
    ];

    /// <summary>
    /// Получить все заказы / Get all orders.
    /// </summary>
    /// <returns>Список заказов / List of orders.</returns>
    /// <response code="200">Возвращает список заказов / Returns the list of orders.</response>
    [HttpGet]
    [OperationId("listOrders")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    public ActionResult<IEnumerable<Order>> GetAll() => Ok(_orders);

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

```csharp
// OrdersApi.Tests/ContractTests.cs — защита контракта в CI / contract protection in CI
using System.Net.Http.Json;
using System.Text.Json;
using Microsoft.AspNetCore.Mvc.Testing;

namespace OrdersApi.Tests;

public sealed class ContractTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    public ContractTests(WebApplicationFactory<Program> factory) => _factory = factory;

    [Fact]
    public async Task SwaggerJson_Matches_Baseline()
    {
        var client = _factory.CreateClient();
        var json = await client.GetStringAsync("/swagger/v1/swagger.json");

        var normalized = JsonSerializer.Serialize(
            JsonDocument.Parse(json).RootElement,
            new JsonSerializerOptions { WriteIndented = true });

        var baselinePath = "baselines/swagger.v1.json";
        if (!File.Exists(baselinePath))
        {
            Directory.CreateDirectory("baselines");
            await File.WriteAllTextAsync(baselinePath, normalized);
            throw new Xunit.Sdk.XunitException("Baseline created; re-run the test.");
        }

        var baseline = await File.ReadAllTextAsync(baselinePath);
        Assert.Equal(baseline, normalized);
    }
}
```

Разбор по строкам. `Assembly.GetExecutingAssembly().GetName().Name` даёт имя сборки (`OrdersApi`), к которому компилятор прицепляет `.xml` — именно этот файл эммитится при `<GenerateDocumentationFile>true</GenerateDocumentationFile>`. `Path.Combine(AppContext.BaseDirectory, xmlFile)` строит путь к `bin/Debug/net8.0/OrdersApi.xml`; `File.Exists` защищает от падения на чистом клоне без билда. `IncludeXmlComments(xmlPath, includeControllerXmlComments: true)` вторым параметром включает комментарии на уровне контроллера — без `true` вы получите только комментарии действий. `SwaggerDoc("v1", new OpenApiInfo {...})` создаёт документ версии v1; `Description` через raw string literal `"""..."""` (C# 12) позволяет писать многострочный текст без экранирования кавычек. `OperationFilter<OperationIdFilter>()` регистрирует фильтр, который Swashbuckle вызывает для каждой операции: в `Apply` мы через отражение ищем `[OperationId]` на методе и перезаписываем дефолтный `ControllerName_ActionName` на стабильный идентификатор. `if (app.Environment.IsDevelopment())` прячет UI в продакшене — ровно тот best practice, что в уроке выделен жирным. `OperationIdAttribute(string operationId)` использует primary constructor C# 12 — короче, чем классический конструктор с полем. В контроллере `_orders` инициализируется collection expression `[]` (C# 12) — эквивалент `new List<Order> { ... }`, но лаконичнее. `[HttpGet("{id:int}")]` задаёт route constraint, который Swashbuckle транслирует в тип параметра `integer` в spec. `[ProducesResponseType]` наполняет секцию `responses` — без него UI показал бы только дефолтный 200. `CreatedAtAction(nameof(GetById), ...)` возвращает `201 Created` с заголовком `Location`, что соответствует REST-конвенции. В тесте `WebApplicationFactory<Program>` поднимает `TestServer` без реального порта, `GetStringAsync("/swagger/v1/swagger.json")` получает документ, нормализация через `JsonDocument` + `WriteIndented = true` убирает плавающие отличия в порядке ключей, а сравнение с `baselines/swagger.v1.json` делает контракт частью репозитория: любое изменение падает в CI до мержа. Эти шаги покрывают все ключевые концепции урока: XML-комментарии, стабильный OperationId, коды ответов, защиту UI и CI-защиту контракта.

#### Задания на углубление (бонус)
1. Добавьте вторую версию документа `SwaggerDoc("v2", ...)` и контроллер `OrdersV2Controller` с маршрутом `api/v2/orders`, в котором поле `Price` заменено на `Amount`. Настройте `UseSwaggerUI` с двумя `SwaggerEndpoint`. Сравните, как в spec отражаются два разных контракта.
2. Подключите Microsoft Kiota и сгенерируйте типизированный клиент из `swagger.json`: `kiota generate -l CSharp -n OrdersApi.Client -d http://localhost:<port>/swagger/v1/swagger.json -o ./generated`. Напишите консольный проект, который вызывает `listOrders` и `createOrder` через сгенерированный клиент. Объясните, почему стабильный `OperationId` критичен.
3. Добавьте схему безопасности в Swagger: `options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme { Type = SecuritySchemeType.Http, Scheme = "bearer", BearerFormat = "JWT" })` и соответствующий `SecurityRequirement`. Защитите один из эндпоинтов `[Authorize]` и покажите в UI кнопку Authorize.
4. В CI-тесте добавьте проверку через `Microsoft.OpenApi.Readers.OpenApiStreamReader` — прочитанный документ должен быть валиден по OpenAPI 3.0, а нормализованный текст через `OpenApiSpecWriter` должен совпадать с baseline. Сравните устойчивость двух подходов (raw JSON vs OpenAPI-модель).

---

## Statement in English / Постановка на английском

#### Context & motivation
You are a backend engineer on a team that ships an internal Orders API consumed by several frontend teams and external partners. The API works, but the consumers are unhappy: the Confluence docs are stale, the endpoint names in Swagger are unreadable (`Orders_GetAll`, `Orders_GetById`), the UI shows neither possible error codes nor request body examples, and client proxies are hand-written, so every contract tweak turns into an avalanche of edits. Your lead asks you to clean this up: bring in Swashbuckle, wire up XML comments, assign stable `OperationId` values, describe every response code, hide the UI in production, and add a CI step that generates `swagger.json` and fails when the contract changes without a version bump.

OpenAPI here is not "a pretty page"; it is a machine-readable contract that becomes the source of truth: Swagger UI for manual exploration, Kiota or NSwag for a typed client, and a pipeline check as a guard against accidental breaking changes. Once you are done, you will have a reproducible template you can lift onto any other API: a single `dotnet test` run tells you whether you broke the contract before the code ever reaches main.

The lesson is explicit about three principles: XML comments are the backbone of human-readable docs; `OperationId` is part of the public contract and changing it breaks generated clients; the UI must be gated in production. You will embody all three in code.

#### What to do (step by step)
1. Create a new project: `dotnet new webapi -n OrdersApi -o OrdersApi --use-controllers`. Move into the folder: `cd OrdersApi`. Confirm `TargetFramework` is `net8.0` and `Nullable` / `ImplicitUsings` are enabled.
2. Install Swashbuckle: `dotnet add package Swashbuckle.AspNetCore --version 6.6.2`. The lesson names this package as the Microsoft-recommended generator.
3. Open `OrdersApi.csproj` and add to the existing `<PropertyGroup>`: `<GenerateDocumentationFile>true</GenerateDocumentationFile>` and `<NoWarn>$(NoWarn);1591</NoWarn>`. The first line makes the compiler emit the XML comments file next to the assembly; the second suppresses the CS1591 warning about missing comments on non-public members.
4. Replace `Program.cs` with a `AddSwaggerGen` configuration: register `AddControllers` and `AddEndpointsApiExplorer`, call `AddSwaggerGen`, inside it build the XML file name via `Assembly.GetExecutingAssembly().GetName().Name` (so you get `OrdersApi.xml`), compose the full path with `Path.Combine(AppContext.BaseDirectory, xmlFile)`, and when `File.Exists(xmlPath)` is true call `options.IncludeXmlComments(xmlPath, includeControllerXmlComments: true)`. Define `SwaggerDoc("v1", new OpenApiInfo { Title = "Orders API", Version = "v1", Description = "..." })`. Register a custom `OperationIdFilter` via `options.OperationFilter<OperationIdFilter>()`.
5. In the middleware section wrap `UseSwagger` and `UseSwaggerUI` in `if (app.Environment.IsDevelopment())`. Inside `UseSwaggerUI` set `c.SwaggerEndpoint("/swagger/v1/swagger.json", "Orders API v1")` and `c.RoutePrefix = "swagger"`. This hides the UI in production — exactly what the lesson's best practice demands.
6. Create a `Controllers` folder and `OrdersController.cs`. The controller carries `[ApiController]` and `[Route("api/v1/orders")]`. Implement three actions: `GetAll` (`[HttpGet]`), `GetById(int id)` (`[HttpGet("{id:int}")]`), `Create([FromBody] CreateOrderRequest request)` (`[HttpPost]`). Above every action add a `/// <summary>` with a bilingual description, `/// <param>` for parameters, and `/// <response code="...">` for every status code. Decorate every action with `[OperationId("listOrders")]` / `[OperationId("getOrderById")]` / `[OperationId("createOrder")]` — these names become the public contract for future clients.
7. Add `[ProducesResponseType(StatusCodes.Status200OK)]` to `GetAll`; `[ProducesResponseType(StatusCodes.Status200OK)]` and `[ProducesResponseType(StatusCodes.Status404NotFound)]` to `GetById`; `[ProducesResponseType(StatusCodes.Status201Created)]` and `[ProducesResponseType(StatusCodes.Status400BadRequest)]` to `Create`. In `Create` return `CreatedAtAction(nameof(GetById), new { id = order.Id }, order)`.
8. Describe models `Order(int Id, string Name, decimal Price)` and `CreateOrderRequest(string Name, decimal Price)` as `sealed record` with `/// <param>` comments above each record — the descriptions land in the schema shown by Swagger UI.
9. Implement `OperationIdAttribute` (primary constructor `OperationIdAttribute(string operationId)`, `AttributeUsage(AttributeTargets.Method)`) and `OperationIdFilter : IOperationFilter`, whose `Apply` reads the attribute via `context.MethodInfo.GetCustomAttributes(typeof(OperationIdAttribute), inherit: true).OfType<OperationIdAttribute>().FirstOrDefault()` and, when present, assigns `operation.OperationId = attr.OperationId`.
10. Run: `dotnet run`. Open `http://localhost:<port>/swagger`. Verify: bilingual descriptions appear under the action names, each endpoint expands the `200/201/404/400` codes, the `Order` and `CreateOrderRequest` schemas show field descriptions, and the OperationId values are `listOrders`, `getOrderById`, `createOrder`. Download `swagger.json` via the UI link and confirm that `operationId` is present on every operation.
11. Add a contract CI check: create a console project `ContractVerifier` (or an xUnit test) that obtains `swagger.json` through `SwaggerDocumentSerializer` or a manual `WebApplicationFactory<Program>` call, stores it as `baselines/swagger.v1.json` (creating the file on the first run), and compares the serialized JSON against the baseline, failing on any diff. A simpler variant: in the test, hit `/swagger/v1/swagger.json` and diff against the committed `baselines/swagger.v1.json`. Run `dotnet test` — the first run creates the baseline, the second should pass green.

#### Requirements
- Target framework `net8.0`, language C# 12: top-level statements in `Program.cs`, primary constructor in `OperationIdAttribute`, pattern matching (`is not null`), `sealed record` for DTOs. Use collection expressions and raw string literals where they improve readability (for instance, a multiline `OpenApiInfo.Description`).
- Package `Swashbuckle.AspNetCore` version 6.6.2 (or a newer 6.x release). Every public controller action is covered by `<summary>`, `<param>`, `<response>` XML comments.
- `OperationId` values are set explicitly via attribute and filter, following the verb+noun camelCase pattern: `listOrders`, `getOrderById`, `createOrder`.
- Every possible response code is described with `[ProducesResponseType]` and `<response>`: `200`, `201`, `400`, `404`.
- `UseSwagger` / `UseSwaggerUI` run only inside `IsDevelopment()`. In production the UI is unreachable.
- Route parameters are correctly bound: `id` is `[FromRoute]` (via the `{id:int}` template), the body is `[FromBody`. In the spec they must appear under the correct `parameters` and `requestBody` sections.
- The contract is checked in CI: a test or script diffs `swagger.json` against a baseline and fails on mismatch. The baseline file is committed.
- The code compiles without warnings (besides the suppressed CS1591). `dotnet build` exits with code 0.

#### Pitfalls
- XML comments do not show up in the UI — the most common mistake. Causes: missing `<GenerateDocumentationFile>true</GenerateDocumentationFile>`, missing `IncludeXmlComments`, or a wrong XML file path. The XML file name matches the assembly name (`OrdersApi.xml`, not `OrdersApi.csproj.xml`) and lives in `AppContext.BaseDirectory`. Always guard `IncludeXmlComments` with `File.Exists(xmlPath)` — on a fresh clone without a rebuild the file may be absent.
- The default `OperationId` from Swashbuckle is `ControllerName_ActionName`, for example `Orders_GetAll`. That is unstable: renaming the controller breaks every client. An explicit attribute plus filter fixes the contract. Never change an `OperationId` between minor releases — it is a breaking change for NSwag/Kiota generated clients.
- Swagger UI in production is an internal API leak. Always use `if (app.Environment.IsDevelopment())`. If partners need the UI in production, gate it behind authentication rather than publishing it openly.
- Parameters vanish from the spec when binding sources are mixed up. `[FromBody]` for `id` produces a `requestBody` instead of a `parameters` entry; `[FromRoute]` for a body is an error. The `{id:int}` route constraint matters: without `:int` the parameter still appears, but the integer constraint is lost.
- Response codes. Without `[ProducesResponseType]` the UI shows only the default `200`. Add every real code: `404` for "not found", `400` for an invalid body, `201` for creation. Mirror them with `<response code="404">Order not found</response>` to improve the UI's UX.
- CI contract check. You cannot diff `swagger.json` as a raw string — Swashbuckle may reorder keys. Serialize through `JsonSerializer.Serialize` with `WriteIndented = true` and stable `JsonSerializerOptions`, or normalize through `Microsoft.OpenApi.Readers` and `OpenApiSpecWriter`. Commit the baseline into the repo so any PR that changes the contract fails in CI before merge.
- `NoWarn` for CS1591 — suppress it only when you knowingly leave parts of the public surface undocumented. Otherwise you go blind to missing comments. Document everything public instead.
- Codegen: NSwag is great for .NET clients, Kiota is Microsoft's modern multi-language tool, OpenAPI Generator belongs to the JVM ecosystem. The choice depends on the consumer's stack. Generated code is also versioned and refreshed in CI, not by hand.

#### Acceptance criteria
- [ ] Project `OrdersApi` created; `Swashbuckle.AspNetCore` 6.6.2 installed.
- [ ] `.csproj` enables `GenerateDocumentationFile` and suppresses `CS1591`.
- [ ] `Program.cs` registers `AddSwaggerGen` with `IncludeXmlComments` guarded by `File.Exists`.
- [ ] `SwaggerDoc("v1", ...)` is set with `Title`, `Version`, `Description`.
- [ ] `OperationIdFilter` is registered via `options.OperationFilter<OperationIdFilter>()`.
- [ ] `UseSwagger` and `UseSwaggerUI` are wrapped in `IsDevelopment()`, with `RoutePrefix = "swagger"`.
- [ ] `OrdersController` carries `[ApiController]`, `[Route("api/v1/orders")]`, three actions.
- [ ] Every action has `<summary>`, `<param>`, `<response>` and `[ProducesResponseType]`.
- [ ] `OperationId` values are `listOrders`, `getOrderById`, `createOrder` (verified in `swagger.json`).
- [ ] `Order` and `CreateOrderRequest` are `sealed record` with XML comments on fields.
- [ ] `Create` returns `CreatedAtAction` with `201`.
- [ ] The UI shows `200/201/400/404` codes and field descriptions in schemas.
- [ ] A CI test/script diffs `swagger.json` against `baselines/swagger.v1.json` and fails on mismatch.
- [ ] `dotnet build` is warning-free, `dotnet test` is green.
- [ ] The report explains the codegen tool choice (NSwag/Kiota/OpenAPI Generator) for the scenario.

#### Hints (no direct answer)
- The XML file name is the assembly name plus `.xml`, not the project name. Verify it with `Assembly.GetExecutingAssembly().GetName().Name`.
- For the operation filter implement `IOperationFilter.Apply(OpenApiOperation, OperationFilterContext)` and look up the attribute via reflection on `context.MethodInfo`.
- For contract tests, `WebApplicationFactory<Program>` from `Microsoft.AspNetCore.Mvc.Testing` spins up a `TestServer` and lets you hit `/swagger/v1/swagger.json` over HTTP.
- JSON normalization before comparison saves you from floating key order. Consider `JsonSerializerOptions { WriteIndented = true, DefaultIgnoreCondition = WhenWritingNull }`.
- For the codegen comparison recall the lesson's analogy: OpenAPI is the menu, Swagger UI is the waiter. Which tool will be your "chef" that cooks the client?

#### Reference solution walk-through
```csharp
// OrdersApi/Program.cs — .NET 8 / C# 12
// Full Swagger/OpenAPI setup with XML comments, stable OperationId, and production UI gating

using Microsoft.OpenApi.Models;
using Swashbuckle.AspNetCore.SwaggerGen;
using System.Reflection;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();

// 1. Swagger + XML comments + OperationId filter
builder.Services.AddSwaggerGen(options =>
{
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);

    // Enable XML comments or the UI is bare
    if (File.Exists(xmlPath))
    {
        options.IncludeXmlComments(xmlPath, includeControllerXmlComments: true);
    }

    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Orders API",
        Version = "v1",
        Description = """
            Demo API for lesson M14-L08.
            Демонстрационный API для урока M14-L08.
            """
    });

    // Stable OperationId is part of the contract
    options.OperationFilter<OperationIdFilter>();
});

var app = builder.Build();

// 2. Dev-only UI — do not expose the internal API
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(c =>
    {
        c.SwaggerEndpoint("/swagger/v1/swagger.json", "Orders API v1");
        c.RoutePrefix = "swagger";
    });
}

app.MapControllers();
app.Run();

/// <summary>
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
/// Custom attribute for setting OperationId (part of the public API contract).
/// </summary>
[AttributeUsage(AttributeTargets.Method, AllowMultiple = false)]
public sealed class OperationIdAttribute(string operationId) : Attribute
{
    public string OperationId { get; } = operationId;
}
```

```csharp
// OrdersApi/Controllers/OrdersController.cs
using Microsoft.AspNetCore.Mvc;

namespace OrdersApi.Controllers;

/// <summary>
/// Orders management / Управление заказами.
/// </summary>
[ApiController]
[Route("api/v1/orders")]
public sealed class OrdersController : ControllerBase
{
    private static readonly List<Order> _orders =
    [
        new(1, "Book", 19.99m),
        new(2, "Laptop", 1299.00m),
    ];

    /// <summary>
    /// Get all orders / Получить все заказы.
    /// </summary>
    /// <returns>List of orders / Список заказов.</returns>
    /// <response code="200">Returns the list of orders / Возвращает список заказов.</response>
    [HttpGet]
    [OperationId("listOrders")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    public ActionResult<IEnumerable<Order>> GetAll() => Ok(_orders);

    /// <summary>
    /// Get an order by id / Получить заказ по идентификатору.
    /// </summary>
    /// <param name="id">Order identifier / Идентификатор заказа.</param>
    /// <response code="200">Order found / Заказ найден.</response>
    /// <response code="404">Order not found / Заказ не найден.</response>
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
    /// Create a new order / Создать новый заказ.
    /// </summary>
    /// <param name="request">Order payload / Данные заказа.</param>
    /// <response code="201">Order created / Заказ создан.</response>
    /// <response code="400">Invalid payload / Некорректные данные.</response>
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

/// <summary>Order model / Модель заказа.</summary>
/// <param name="Id">Identifier / Идентификатор.</param>
/// <param name="Name">Name / Название.</param>
/// <param name="Price">Price / Цена.</param>
public sealed record Order(int Id, string Name, decimal Price);

/// <summary>Create order request / Запрос на создание заказа.</summary>
/// <param name="Name">Name / Название.</param>
/// <param name="Price">Price / Цена.</param>
public sealed record CreateOrderRequest(string Name, decimal Price);
```

```csharp
// OrdersApi.Tests/ContractTests.cs — contract protection in CI
using System.Net.Http.Json;
using System.Text.Json;
using Microsoft.AspNetCore.Mvc.Testing;

namespace OrdersApi.Tests;

public sealed class ContractTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    public ContractTests(WebApplicationFactory<Program> factory) => _factory = factory;

    [Fact]
    public async Task SwaggerJson_Matches_Baseline()
    {
        var client = _factory.CreateClient();
        var json = await client.GetStringAsync("/swagger/v1/swagger.json");

        var normalized = JsonSerializer.Serialize(
            JsonDocument.Parse(json).RootElement,
            new JsonSerializerOptions { WriteIndented = true });

        var baselinePath = "baselines/swagger.v1.json";
        if (!File.Exists(baselinePath))
        {
            Directory.CreateDirectory("baselines");
            await File.WriteAllTextAsync(baselinePath, normalized);
            throw new Xunit.Sdk.XunitException("Baseline created; re-run the test.");
        }

        var baseline = await File.ReadAllTextAsync(baselinePath);
        Assert.Equal(baseline, normalized);
    }
}
```

Line-by-line walk-through. `Assembly.GetExecutingAssembly().GetName().Name` yields the assembly name (`OrdersApi`), to which the compiler appends `.xml` — that file is emitted when `<GenerateDocumentationFile>true</GenerateDocumentationFile>` is set. `Path.Combine(AppContext.BaseDirectory, xmlFile)` builds the path to `bin/Debug/net8.0/OrdersApi.xml`; the `File.Exists` guard prevents a crash on a clean clone without a prior build. `IncludeXmlComments(xmlPath, includeControllerXmlComments: true)` — the second parameter pulls in controller-level comments; without `true` you only get action-level descriptions. `SwaggerDoc("v1", new OpenApiInfo {...})` creates the v1 document; the `Description` uses a C# 12 raw string literal `"""..."""` so you can write multiline text without escaping quotes. `OperationFilter<OperationIdFilter>()` registers a filter that Swashbuckle invokes per operation: in `Apply` we use reflection to find `[OperationId]` on the method and overwrite the default `ControllerName_ActionName` with a stable identifier. `if (app.Environment.IsDevelopment())` hides the UI in production — exactly the best practice the lesson emphasizes. `OperationIdAttribute(string operationId)` uses a C# 12 primary constructor — shorter than a classic constructor with a backing field. In the controller, `_orders` is initialized with a collection expression `[]` (C# 12) — equivalent to `new List<Order> { ... }` but terser. `[HttpGet("{id:int}")]` declares a route constraint that Swashbuckle translates into an `integer` parameter type in the spec. `[ProducesResponseType]` populates the `responses` section — without it the UI would show only a default 200. `CreatedAtAction(nameof(GetById), ...)` returns `201 Created` with a `Location` header, matching the REST convention. In the test, `WebApplicationFactory<Program>` spins up a `TestServer` with no real port, `GetStringAsync("/swagger/v1/swagger.json")` fetches the document, normalization through `JsonDocument` plus `WriteIndented = true` removes floating differences in key order, and the comparison against `baselines/swagger.v1.json` makes the contract part of the repository: any change fails in CI before merge. Together these steps cover every key concept of the lesson: XML comments, stable OperationId, response codes, UI gating, and CI contract protection.

#### Going deeper (bonus)
1. Add a second document version `SwaggerDoc("v2", ...)` and an `OrdersV2Controller` at `api/v2/orders` where the `Price` field is renamed to `Amount`. Configure `UseSwaggerUI` with two `SwaggerEndpoint` entries. Compare how the two contracts surface in the spec.
2. Bring in Microsoft Kiota and generate a typed client from `swagger.json`: `kiota generate -l CSharp -n OrdersApi.Client -d http://localhost:<port>/swagger/v1/swagger.json -o ./generated`. Write a console app that calls `listOrders` and `createOrder` through the generated client. Explain why a stable `OperationId` is critical.
3. Add a security scheme to Swagger: `options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme { Type = SecuritySchemeType.Http, Scheme = "bearer", BearerFormat = "JWT" })` plus a matching `SecurityRequirement`. Protect one endpoint with `[Authorize]` and demonstrate the Authorize button in the UI.
4. In the CI test, add a check through `Microsoft.OpenApi.Readers.OpenApiStreamReader` — the parsed document must be valid against OpenAPI 3.0, and the normalized text via `OpenApiSpecWriter` must match the baseline. Compare the robustness of the two approaches (raw JSON vs OpenAPI model).

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `OrdersApi` собирается через `dotnet build` без предупреждений.
- [ ] `Swashbuckle.AspNetCore` 6.6.2 установлен и зарегистрирован (`AddSwaggerGen`, `UseSwagger`, `UseSwaggerUI`).
- [ ] XML-документация включена и подключена через `IncludeXmlComments` с проверкой `File.Exists`.
- [ ] Все публичные действия покрыты `<summary>`, `<param>`, `<response>` и `[ProducesResponseType]`.
- [ ] `OperationId` стабильны: `listOrders`, `getOrderById`, `createOrder`.
- [ ] UI закрыт в продакшене через `IsDevelopment()`.
- [ ] CI-тест сравнивает `swagger.json` с `baselines/swagger.v1.json`.
- [ ] В отчёте выбран codegen-инструмент (NSwag/Kiota/OpenAPI Generator) и обоснован.
- [ ] Project `OrdersApi` builds via `dotnet build` with no warnings.
- [ ] `Swashbuckle.AspNetCore` 6.6.2 installed and registered (`AddSwaggerGen`, `UseSwagger`, `UseSwaggerUI`).
- [ ] XML documentation enabled and wired through `IncludeXmlComments` with a `File.Exists` check.
- [ ] Every public action carries `<summary>`, `<param>`, `<response>` and `[ProducesResponseType]`.
- [ ] `OperationId` values are stable: `listOrders`, `getOrderById`, `createOrder`.
- [ ] UI is gated in production via `IsDevelopment()`.
- [ ] A CI test diffs `swagger.json` against `baselines/swagger.v1.json`.
- [ ] The report picks a codegen tool (NSwag/Kiota/OpenAPI Generator) and justifies the choice.

#### Ресурсы / Resources
- [Microsoft Learn — Get started with Swashbuckle and ASP.NET Core](https://learn.microsoft.com/aspnet/core/tutorials/web-api-help-pages-using-swagger)
- [OpenAPI Specification 3.1.0](https://spec.openapis.org/oas/v3.1.0)
- [Swashbuckle.AspNetCore on GitHub](https://github.com/domaindrivendev/Swashbuckle.AspNetCore)
- [NSwag on GitHub](https://github.com/RicoSuter/NSwag)
- [Microsoft Kiota](https://learn.microsoft.com/openapi/kiota/)
- [ASP.NET Core API integration tests with WebApplicationFactory](https://learn.microsoft.com/aspnet/core/test/integration-tests)
