---
[← К уроку M14-L04](lesson-M14-L04-model-binding.md) | [⬆ К модулю M14](../README.md) | [Следующее ДЗ →](homework-M14-L05-validation.md)
---

### Домашнее задание M14-L04: Model binding, [FromBody]/[FromQuery]/[FromRoute] / Homework M14-L04: Model binding, [FromBody]/[FromQuery]/[FromRoute]

**Урок / Lesson:** M14-L04
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться осознанно выбирать источник привязки для каждого параметра действия, строить отдельные DTO для тела, маршрута и строки запроса, подключать валидацию через `ModelState` и реализовать кастомный `IModelBinder` с `IModelBinderProvider` для сценария, который не покрывается стандартными атрибутами. (EN) Learn to deliberately choose a binding source for every action parameter, build dedicated DTOs for body, route and query string, wire up validation through `ModelState`, and implement a custom `IModelBinder` with `IModelBinderProvider` for a scenario that standard attributes cannot cover.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит понятие источника привязки (binding source), порядок поиска по умолчанию (form → route → query) и атрибуты `[FromQuery]`, `[FromRoute]`, `[FromBody]`, `[FromHeader]`, `[FromForm]`, `[FromServices]`. В этом задании вы примените каждый из них в реальном Web API, повторите best practices и наступите на частые ошибки, описанные в уроке: пустое тело с `[FromBody]`, конфликт имён в query и route, забытая проверка `ModelState.IsValid`, нерезегистрированный провайдер кастомного биндера.
(EN) The lesson introduces the binding source concept, the default lookup order (form → route → query) and the attributes `[FromQuery]`, `[FromRoute]`, `[FromBody]`, `[FromHeader]`, `[FromForm]`, `[FromServices]`. In this homework you will apply each of them in a real Web API, repeat the best practices and step on the common mistakes described in the lesson: empty body with `[FromBody]`, a name clash between query and route, a forgotten `ModelState.IsValid` check, and an unregistered custom binder provider.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы присоединяетесь к команде, которая разрабатывает внутренний сервис «Каталог задач» (Task Catalog) для небольшой SaaS-платформы. Сервис уже имеет каркас ASP.NET Core Web API на .NET 8, но старший разработчик просит вас навести порядок в контроллере `TasksController`: сейчас параметры действий берутся «откуда попало», потому что ни один из них не помечен атрибутом источника привязки. Из-за этого одни и те же данные иногда приходят из query, иногда из route, а баги с пустыми телами и невалидными GUID всплывают в проде.

Урок M14-L04 учит, что model binding — это «переводчик» между миром HTTP (текст, байты, заголовки) и миром C#-объектов, и что каждый параметр должен явно указывать, откуда брать значение. Вы примените эту идею на практике: пометите идентификаторы ресурсов через `[FromRoute]`, пострóите отдельный query DTO для фильтров и пагинации через `[FromQuery]`, организуете create/update DTO с `[FromBody]` и валидацией через DataAnnotations, добавите параметр из заголовка (`X-Request-Id`) и параметр из DI через `[FromServices]`. Дополнительно вы реализуете кастомный биндер `CorrelationContextBinder`, который собирает объект `CorrelationContext` из двух заголовков `X-Correlation-Id` и `X-Source`, и зарегистрируете его через `IModelBinderProvider`, как описано в разделе «Кастомные биндеры» урока.

Цель — не просто заставить код компилироваться, а сделать привязку предсказуемой, безопасной и самодокументируемой. Каждое действие должно однозначно говорить: «идентификатор я жду в URL, фильтры — в query, тело — это JSON, а контекст корреляции — в заголовках». После рефакторинга набор автотестов (curl/HTTP-репорт) должен проходить «зелёным».

#### Что нужно сделать (пошагово)
1. Создайте новый проект, если его ещё нет: `dotnet new webapi -n TaskCatalog -o TaskCatalog --use-controllers` в корне репозитория курса. Убедитесь, что целевая платформа — `.NET 8` и что в `TaskCatalog.csproj` есть `<Nullable>enable</Nullable>` и `<ImplicitUsings>enable</ImplicitUsings>`. Запустите один раз `dotnet build` и убедитесь, что сборка проходит без ошибок.
2. В папке `Models` создайте DTO. Сначала тело для создания задачи: `record CreateTaskDto([Required] string Title, [StringLength(500)] string Description, [Range(1, 5)] int Priority, [Required] Guid AssigneeId)`. Затем тело для обновления: `record UpdateTaskDto([StringLength(500)] string? Description, [Range(1, 5)] int? Priority)`. Обратите внимание, что в update все поля необязательные — это идиома PATCH-подобного обновления.
3. Создайте query DTO для фильтров списка: `record TaskQueryDto([FromQuery(Name = "q")] string? Search, [FromQuery(Name = "status")] string? Status, [FromQuery(Name = "page")] int Page = 1, [FromQuery(Name = "size")] [Range(1, 100)] int PageSize = 20, [FromQuery(Name = "sort")] string? Sort)`. Имена в `[FromQuery(Name = ...)]` отличаются от свойств — это типичный приём для коротких URL.
4. Создайте record для контекста корреляции: `record CorrelationContext(Guid CorrelationId, string Source)`. Этот объект будет собираться из двух заголовков и не имеет стандартного атрибута источника — именно поэтому нужен кастомный биндер.
5. Реализуйте `CorrelationContextBinder : IModelBinder`. В `BindModelAsync` прочитайте `req.Headers["X-Correlation-Id"]` и `req.Headers["X-Source"]`, распарсите GUID через `Guid.TryParse`, проверьте, что `Source` не пустой. Если данные некорректны — установите `bindingContext.Result = ModelBindingResult.Failed()` и верните `Task.CompletedTask`. Иначе — `ModelBindingResult.Success(new CorrelationContext(...))`. Повторите логику из примера урока с `TenantContextBinder`, но для двух ваших заголовков.
6. Реализуйте `CorrelationContextBinderProvider : IModelBinderProvider`, который возвращает `CorrelationContextBinder`, если `context.Metadata.ModelType == typeof(CorrelationContext)`, иначе `null`. Без провайдера биндер не будет вызван — это частая ошибка из урока.
7. В `Program.cs` зарегистрируйте провайдер: `builder.Services.AddControllers(o => o.ModelBinderProviders.Insert(0, new CorrelationContextBinderProvider()));`. Используйте именно `Insert(0, ...)`, чтобы ваш провайдер проверялся раньше стандартных, как в примере урока.
8. Реализуйте `TasksController` с атрибутом `[ApiController]` и маршрутом `api/[controller]`. Действия: `GET /{id:guid}` — получает `Guid id` через `[FromRoute]`; `GET /` — получает `TaskQueryDto query` через `[FromQuery]`; `POST /` — получает `CreateTaskDto dto` через `[FromBody]` и проверяет `ModelState.IsValid` (для наглядности, хотя `[ApiController]` сделает это автоматически); `PUT /{id:guid}` — получает `Guid id` через `[FromRoute]` и `UpdateTaskDto dto` через `[FromBody]`; `GET /correlated` — получает `CorrelationContext ctx` через `[ModelBinder(typeof(CorrelationContextBinder))]` и `ILogger<TasksController> logger` через `[FromServices]`, а также `Guid requestId` через `[FromHeader(Name = "X-Request-Id")]`.
9. Запустите сервис: `dotnet run --project TaskCatalog`. По умолчанию Kestrel слушает `http://localhost:5000` или `https://localhost:5001` (порт из `launchSettings.json`).
10. Проверьте эндпоинты через `curl`. Примеры: `curl "http://localhost:5000/api/tasks?q=login&page=2&size=10"` должен вернуть JSON с полями `search`, `page`, `pageSize`; `curl -X POST http://localhost:5000/api/tasks -H "Content-Type: application/json" -d '{"title":"Fix bug","description":"...","priority":3,"assigneeId":"00000000-0000-0000-0000-000000000000"}'` должен вернуть 201; запрос без `title` должен вернуть 400 с описанием ошибки валидации; `curl -H "X-Correlation-Id: 11111111-1111-1111-1111-111111111111" -H "X-Source: web" http://localhost:5000/api/tasks/correlated` должен вернуть контекст корреляции.
11. Добавьте单元-тест или хотя бы один `curl`-скрипт `tests/smoke.sh`, который прогоняет все сценарии и печатает `OK`/`FAIL`. Зафиксируйте ожидаемые HTTP-коды в комментариях скрипта.

#### Требования к решению
- Целевая платформа — `.NET 8`, язык — `C# 12`. Используйте top-level statements в `Program.cs`, record-типы для DTO, коллекционные выражения и pattern matching там, где это уместно. Включите `<Nullable>enable</Nullable>`.
- Каждый параметр действия должен иметь явный атрибут источника привязки. Ни один параметр не должен полагаться на порядок поиска по умолчанию (form → route → query). Для простых типов (`Guid`, `int`, `string`) используйте `[FromRoute]` или `[FromHeader]`; для сложных DTO — `[FromBody]` или `[FromQuery]`.
- На один метод — не более одного параметра с `[FromBody]`. Урок прямо предупреждает: тело — это поток, читаемый один раз.
- Все DTO для тела должны иметь DataAnnotations для обязательной валидации: `[Required]`, `[StringLength]`, `[Range]`, `[EmailAddress]` там, где нужно. Контроллер должен проверять `ModelState.IsValid` явно в POST/PUT, даже если `[ApiController]` автоматически возвращает 400, — это требование для наглядности.
- Кастомный биндер должен реализовывать `IModelBinder` и регистрироваться через `IModelBinderProvider` в `Program.cs`. Без регистрации провайдера биндер не вызывается — это частая ошибка урока.
- Имена свойств query DTO должны отличаться от имён в URL через `[FromQuery(Name = "...")]`, чтобы продемонстрировать маппинг коротких имён (`q`, `page`, `size`) к читаемым свойствам (`Search`, `Page`, `PageSize`).
- Маршруты должны использовать ограничение `{id:guid}` для идентификаторов — это убирает неоднозначность и сразу возвращает 404 для не-GUID строк.

#### Тонкости и подводные камни
- **Пустое тело с `[FromBody]` на простом типе.** Если поставить `[FromBody] int x` и прислать пустое тело, ASP.NET Core вернёт 400, потому что тело нельзя десериализовать в `int`. Для простых значений используйте `[FromQuery]` или `[FromRoute]` — это правило из best practices урока.
- **Конфликт имён параметра в query и route.** Если маршрут `{id}` и одновременно в query есть `?id=...`, binder по умолчанию может взять значение из неожиданного места. Явный `[FromRoute]` или `[FromQuery]` решает проблему однозначно.
- **`[FromBody]` без конструктора или сеттеров.** Если DTO — это класс с приватными свойствами, JSON-десериализатор вернёт `null` или объект с дефолтами. Используйте `record` с позиционными параметрами или класс с публичными свойствами — урок упоминает это в частых ошибках.
- **Забытый `[ApiController]`.** Без него `ModelState.IsValid` не проверяется автоматически, и невалидные данные уходят в бизнес-логику. С `[ApiController]` невалидная модель автоматически даёт 400, но явная проверка всё равно полезна для читаемости и для MVC-представлений.
- **Кастомный биндер не зарегистрирован.** Если вы реализовали `IModelBinder`, но не добавили провайдер в `ModelBinderProviders`, биндер никогда не вызовется, а параметр останется `null`. Урок явно предупреждает об этом.
- **`Insert(0, ...)` vs `Add(...)`.** Стандартные провайдеры проверяются по порядку. Если ваш провайдер добавить в конец, стандартный провайдер для сложных типов может перехватить привязку раньше. `Insert(0, ...)` гарантирует приоритет.
- **`Guid.TryParse` и культура.** Парсинг GUID не зависит от культуры, но парсинг чисел и дат — зависит. Если в биндере будете парсить `DateTime` или `decimal`, используйте `TryParse` с `CultureInfo.InvariantCulture`.
- **`ModelBindingResult.Failed()` vs `Success(null)`.** `Failed()` означает «значение не найдено», `Success(null)` — «значение найдено, но оно null». Для обязательных заголовков используйте `Failed()`, чтобы фреймворк знал, что привязка не удалась.

#### Критерии приёмки
- [ ] Проект `TaskCatalog` собирается командой `dotnet build` без ошибок и предупреждений.
- [ ] Каждый параметр действия имеет явный атрибут источника привязки (`[FromRoute]`, `[FromQuery]`, `[FromBody]`, `[FromHeader]`, `[FromServices]` или `[ModelBinder]`).
- [ ] `CreateTaskDto` и `UpdateTaskDto` — record-типы с DataAnnotations (`[Required]`, `[StringLength]`, `[Range]`).
- [ ] `TaskQueryDto` помечен `[FromQuery]` на каждом свойстве, с переименованием через `Name = "q"`, `"page"`, `"size"`, `"status"`, `"sort"`.
- [ ] `POST /api/tasks` с пустым `title` возвращает 400 и описание ошибки валидации в теле ответа.
- [ ] `POST /api/tasks` с корректным телом возвращает 201 с заголовком `Location`.
- [ ] `GET /api/tasks/{id:guid}` с не-GUID строкой возвращает 404 (благодаря ограничению маршрута).
- [ ] `GET /api/tasks?q=...&page=2&size=10` корректно парсит query в `TaskQueryDto` и возвращает значения в ответе.
- [ ] `GET /api/tasks/correlated` без заголовков `X-Correlation-Id` и `X-Source` возвращает 400 или не вызывает действия с невалидным контекстом.
- [ ] `CorrelationContextBinder` реализует `IModelBinder`, `CorrelationContextBinderProvider` реализует `IModelBinderProvider`.
- [ ] Провайдер зарегистрирован в `Program.cs` через `Insert(0, ...)` в `AddControllers`.
- [ ] В действиях используется `[FromServices]` для `ILogger<TasksController>`, а не `new Logger()`.
- [ ] Smoke-тест `tests/smoke.sh` проходит все сценарии и печатает `OK` для каждого.
- [ ] Код использует возможности C# 12 (record, top-level statements, при необходимости — pattern matching и коллекционные выражения).
- [ ] В `Program.cs` включена регистрация контроллеров и Swagger/OpenAPI для удобной ручной проверки.

#### Подсказки (без прямого ответа)
- Вспомните порядок источников по умолчанию: form → route → query. Подумайте, что произойдёт, если убрать все атрибуты — и почему явные атрибуты делают код безопаснее.
- Для `CorrelationContextBinder` посмотрите на `TenantContextBinder` из урока: структура одинакова, отличаются только имена заголовков и логика валидации.
- Если `ModelState` кажется пустым, проверьте, что у вас стоит `[ApiController]` — без него ошибки валидации не накапливаются автоматически в нужном виде.
- Для `X-Request-Id` используйте `Guid.TryParse` в действии или сделайте тип параметра `Guid?`, чтобы обрабатывать отсутствие заголовка.
- Не забудьте `Content-Type: application/json` в `curl` для POST/PUT — иначе тело не десериализуется.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — TaskCatalog/Program.cs
// Регистрация сервисов, контроллеров и кастомного биндера / Service, controller and binder registration

using Microsoft.AspNetCore.Mvc;
using TaskCatalog.Binders;
using TaskCatalog.Controllers;
using TaskCatalog.Models;

var builder = WebApplication.CreateBuilder(args);

// Регистрируем контроллеры и вставляем провайдер кастомного биндера на первое место.
// Register controllers and insert the custom binder provider at position 0.
builder.Services.AddControllers(options =>
{
    options.ModelBinderProviders.Insert(0, new CorrelationContextBinderProvider());
});

// Swagger для ручной проверки эндпоинтов / Swagger for manual endpoint testing.
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.MapControllers();
app.Run();
```

```csharp
// TaskCatalog/Models/Dtos.cs — DTO для тела и query / Body and query DTOs

using System.ComponentModel.DataAnnotations;
using Microsoft.AspNetCore.Mvc;

// Тело для создания задачи / Body for task creation
public record CreateTaskDto(
    [Required(ErrorMessage = "Title is required.")] string Title,
    [StringLength(500, ErrorMessage = "Description must be under 500 chars.")] string Description,
    [Range(1, 5, ErrorMessage = "Priority must be between 1 and 5.")] int Priority,
    [Required(ErrorMessage = "AssigneeId is required.")] Guid AssigneeId);

// Тело для обновления (PATCH-подобное) / Body for update (PATCH-like)
public record UpdateTaskDto(
    [StringLength(500)] string? Description,
    [Range(1, 5)] int? Priority);

// Query DTO для фильтров списка / Query DTO for list filters
// Имена в URL короткие (q, page, size), имена свойств читаемые / Short URL names, readable properties
public record TaskQueryDto(
    [FromQuery(Name = "q")] string? Search,
    [FromQuery(Name = "status")] string? Status,
    [FromQuery(Name = "page")] int Page = 1,
    [FromQuery(Name = "size")] [Range(1, 100)] int PageSize = 20,
    [FromQuery(Name = "sort")] string? Sort);

// Контекст корреляции — собирается из двух заголовков / Correlation context from two headers
public record CorrelationContext(Guid CorrelationId, string Source);
```

```csharp
// TaskCatalog/Binders/CorrelationContextBinder.cs — кастомный биндер / custom binder

using Microsoft.AspNetCore.Mvc.ModelBinding;

public class CorrelationContextBinder : IModelBinder
{
    public Task BindModelAsync(ModelBindingContext bindingContext)
    {
        // Защита от null-контекста / Null-context guard
        if (bindingContext is null) throw new ArgumentNullException(nameof(bindingContext));

        var req = bindingContext.HttpContext.Request;
        var correlationRaw = req.Headers["X-Correlation-Id"].FirstOrDefault();
        var source = req.Headers["X-Source"].FirstOrDefault();

        // Если GUID не парсится или источник пуст — привязка не удалась / If GUID fails or source empty — fail
        if (!Guid.TryParse(correlationRaw, out var correlationId)
            || string.IsNullOrWhiteSpace(source))
        {
            bindingContext.Result = ModelBindingResult.Failed();
            return Task.CompletedTask;
        }

        bindingContext.Result = ModelBindingResult.Success(
            new CorrelationContext(correlationId, source));
        return Task.CompletedTask;
    }
}

// Провайдер: связывает тип CorrelationContext с биндером / Provider: links CorrelationContext type to binder
public class CorrelationContextBinderProvider : IModelBinderProvider
{
    public IModelBinder? GetBinder(ModelBinderProviderContext context)
        => context.Metadata.ModelType == typeof(CorrelationContext)
            ? new CorrelationContextBinder()
            : null;
}
```

```csharp
// TaskCatalog/Controllers/TasksController.cs

using Microsoft.AspNetCore.Mvc;

namespace TaskCatalog.Controllers;

[ApiController]                                  // автоматическая валидация ModelState / auto ModelState validation
[Route("api/[controller]")]
public class TasksController : ControllerBase
{
    private readonly ILogger<TasksController> _logger;

    // ILogger внедряется через DI — в действии используем [FromServices] для демонстрации / DI logger
    public TasksController(ILogger<TasksController> logger) => _logger = logger;

    // GET api/tasks/{id:guid} — идентификатор из маршрута / id from route
    [HttpGet("{id:guid}")]
    public IActionResult GetById([FromRoute] Guid id)
        => Ok(new { id, title = "Demo task", status = "open" });

    // GET api/tasks?q=...&page=2&size=10 — фильтры из query / filters from query
    [HttpGet]
    public IActionResult List([FromQuery] TaskQueryDto query)
        => Ok(new { query.Search, query.Status, query.Page, query.PageSize, query.Sort });

    // POST api/tasks — тело запроса (JSON) / request body (JSON)
    [HttpPost]
    public IActionResult Create([FromBody] CreateTaskDto dto)
    {
        // ModelState проверяется автоматически с [ApiController], но явно — для наглядности
        // ModelState is checked automatically with [ApiController], explicit here for clarity
        if (!ModelState.IsValid)
            return BadRequest(ModelState);

        var id = Guid.NewGuid();
        return CreatedAtAction(nameof(GetById), new { id }, new { id, dto.Title, dto.Priority });
    }

    // PUT api/tasks/{id:guid} — id из маршрута, тело из JSON / id from route, body from JSON
    [HttpPut("{id:guid}")]
    public IActionResult Update([FromRoute] Guid id, [FromBody] UpdateTaskDto dto)
    {
        if (!ModelState.IsValid)
            return BadRequest(ModelState);

        return NoContent();
    }

    // GET api/tasks/correlated — контекст из двух заголовков + requestId из заголовка + логгер из DI
    // GET api/tasks/correlated — context from two headers + requestId from header + logger from DI
    [HttpGet("correlated")]
    public IActionResult Correlated(
        [ModelBinder(typeof(CorrelationContextBinder))] CorrelationContext ctx,
        [FromHeader(Name = "X-Request-Id")] Guid? requestId,
        [FromServices] ILogger<TasksController> logger)
    {
        logger.LogInformation("Correlation {CorrelationId} from {Source}, request {RequestId}",
            ctx.CorrelationId, ctx.Source, requestId);

        return Ok(new { ctx.CorrelationId, ctx.Source, requestId });
    }
}
```

Разбор по строкам. В `Program.cs` мы вызываем `AddControllers` с лямбдой, которая настраивает `MvcOptions`, и вставляем `CorrelationContextBinderProvider` через `Insert(0, ...)` — это гарантирует, что наш провайдер проверяется раньше стандартных, как требует урок. `AddSwaggerGen` добавляет UI для ручной проверки. В `Dtos.cs` каждый DTO — это record с позиционными параметрами и DataAnnotations: урок подчёркивает, что record с публичными свойствами необходим для JSON-десериализации через `[FromBody]`. `TaskQueryDto` использует `[FromQuery(Name = "q")]` — это приём маппинга коротких URL-имён к читаемым свойствам, упомянутый в best practices. `CorrelationContext` — это тип, для которого нет стандартного атрибута, поэтому нужен кастомный биндер.

В `CorrelationContextBinder` мы повторяем структуру `TenantContextBinder` из урока: читаем заголовки, парсим GUID через `TryParse`, при неудаче вызываем `ModelBindingResult.Failed()`, при успехе — `Success(new CorrelationContext(...))`. Провайдер проверяет `ModelType` и возвращает биндер только для `CorrelationContext`, иначе `null` — фреймворк продолжит искать подходящий провайдер. Без регистрации в `Program.cs` биндер бы не вызвался, и параметр остался бы `null` — это частая ошибка урока.

В `TasksController` атрибут `[ApiController]` включает автоматическую валидацию `ModelState` и автоматический 400 при ошибках; мы всё равно проверяем `ModelState.IsValid` явно для наглядности, как требует best practice. `[FromRoute] Guid id` берёт идентификатор строго из маршрута, а ограничение `{id:guid}` в шаблоне гарантирует 404 для не-GUID строк. `[FromQuery] TaskQueryDto` собирает все фильтры в один объект — это рекомендуемый паттерн для запросов со множеством параметров. `[FromBody]` используется ровно один раз на метод — урок прямо предупреждает, что тело читается один раз. `Correlated` демонстрирует сразу три источника: кастомный биндер для `CorrelationContext`, `[FromHeader]` для `X-Request-Id` и `[FromServices]` для логгера. Последний показывает, что DI-зависимости тоже можно получать как параметры действия, а не только через конструктор. Все эти элементы напрямую закрепляют темы урока: источники привязки, порядок поиска, валидация и кастомные биндеры.

#### Задания на углубление (бонус)
1. Добавьте `[FromForm]` действие `POST /api/tasks/import`, которое принимает `IFormFile` и строковое поле `format` через `[FromForm]`. Продемонстрируйте, что form values — первый источник в порядке поиска по умолчанию.
2. Реализуйте `IValidatableObject` для `CreateTaskDto`: если `Priority == 5`, то `Description` должен быть не короче 20 символов (бизнес-правило). Верните `ValidationResult` через `Validate(ValidationContext)`.
3. Добавьте второй кастомный биндер `SortOrderBinder`, который парсит строку `sort=created_at:desc` в объект `SortOrder(string Field, bool Descending)`. Зарегистрируйте оба провайдера и проверьте, что порядок `Insert` не ломает их работу.
4. Подключите FluentValidation через пакет `FluentValidation.AspNetCore` и напишите валидатор для `UpdateTaskDto`, который запрещает одновременно пустые `Description` и `Priority` (хотя бы одно поле должно изменяться).

---

## Statement in English / Постановка на английском

#### Context & motivation
You are joining a team that builds an internal "Task Catalog" service for a small SaaS platform. The service already has an ASP.NET Core Web API skeleton on .NET 8, but the senior developer asks you to clean up the `TasksController`: right now its action parameters are bound "from wherever", because none of them is decorated with a binding source attribute. As a result, the same piece of data sometimes arrives from the query string, sometimes from the route, and bugs with empty bodies and invalid GUIDs keep surfacing in production.

Lesson M14-L04 teaches that model binding is the "translator" between the HTTP world (text, bytes, headers) and the C# object world, and that every parameter must explicitly state where to take its value from. You will apply that idea in practice: you will mark resource identifiers with `[FromRoute]`, build a dedicated query DTO for filters and pagination with `[FromQuery]`, design create/update DTOs with `[FromBody]` and DataAnnotations validation, add a parameter from a header (`X-Request-Id`) and a parameter from DI through `[FromServices]`. On top of that, you will implement a custom binder `CorrelationContextBinder` that builds a `CorrelationContext` object from two headers `X-Correlation-Id` and `X-Source`, and register it through an `IModelBinderProvider`, exactly as described in the "Custom binders" section of the lesson.

The goal is not merely to make the code compile, but to make binding predictable, secure and self-documenting. Every action should say unambiguously: "I expect the identifier in the URL, the filters in the query string, the body as JSON, and the correlation context in the headers." After the refactor, a smoke test (curl/HTTP report) must pass green.

#### What to do step by step
1. Create a new project if one does not exist yet: run `dotnet new webapi -n TaskCatalog -o TaskCatalog --use-controllers` at the root of the course repository. Make sure the target framework is `.NET 8` and that `TaskCatalog.csproj` contains `<Nullable>enable</Nullable>` and `<ImplicitUsings>enable</ImplicitUsings>`. Run `dotnet build` once and confirm the build is clean.
2. In the `Models` folder, create the DTOs. First the body for creating a task: `record CreateTaskDto([Required] string Title, [StringLength(500)] string Description, [Range(1, 5)] int Priority, [Required] Guid AssigneeId)`. Then the body for updating: `record UpdateTaskDto([StringLength(500)] string? Description, [Range(1, 5)] int? Priority)`. Notice that in the update DTO every field is optional — that is the PATCH-like update idiom.
3. Create a query DTO for list filters: `record TaskQueryDto([FromQuery(Name = "q")] string? Search, [FromQuery(Name = "status")] string? Status, [FromQuery(Name = "page")] int Page = 1, [FromQuery(Name = "size")] [Range(1, 100)] int PageSize = 20, [FromQuery(Name = "sort")] string? Sort)`. The names in `[FromQuery(Name = ...)]` differ from the properties — this is a typical trick for short URLs.
4. Create a record for the correlation context: `record CorrelationContext(Guid CorrelationId, string Source)`. This object is assembled from two headers and has no standard source attribute — that is exactly why a custom binder is needed.
5. Implement `CorrelationContextBinder : IModelBinder`. In `BindModelAsync` read `req.Headers["X-Correlation-Id"]` and `req.Headers["X-Source"]`, parse the GUID with `Guid.TryParse`, check that `Source` is not empty. If the data is invalid, set `bindingContext.Result = ModelBindingResult.Failed()` and return `Task.CompletedTask`. Otherwise, call `ModelBindingResult.Success(new CorrelationContext(...))`. Mirror the logic from the lesson's `TenantContextBinder`, but for your two headers.
6. Implement `CorrelationContextBinderProvider : IModelBinderProvider` that returns `CorrelationContextBinder` when `context.Metadata.ModelType == typeof(CorrelationContext)`, and `null` otherwise. Without the provider the binder is never called — this is a common mistake from the lesson.
7. In `Program.cs` register the provider: `builder.Services.AddControllers(o => o.ModelBinderProviders.Insert(0, new CorrelationContextBinderProvider()));`. Use `Insert(0, ...)` so your provider is checked before the standard ones, as in the lesson example.
8. Implement `TasksController` with the `[ApiController]` attribute and the route `api/[controller]`. Actions: `GET /{id:guid}` takes `Guid id` via `[FromRoute]`; `GET /` takes `TaskQueryDto query` via `[FromQuery]`; `POST /` takes `CreateTaskDto dto` via `[FromBody]` and checks `ModelState.IsValid` (for clarity, although `[ApiController]` does it automatically); `PUT /{id:guid}` takes `Guid id` via `[FromRoute]` and `UpdateTaskDto dto` via `[FromBody]`; `GET /correlated` takes `CorrelationContext ctx` via `[ModelBinder(typeof(CorrelationContextBinder))]`, `ILogger<TasksController> logger` via `[FromServices]`, and `Guid requestId` via `[FromHeader(Name = "X-Request-Id")]`.
9. Run the service: `dotnet run --project TaskCatalog`. By default Kestrel listens on `http://localhost:5000` or `https://localhost:5001` (the port comes from `launchSettings.json`).
10. Verify the endpoints with `curl`. Examples: `curl "http://localhost:5000/api/tasks?q=login&page=2&size=10"` should return JSON with `search`, `page`, `pageSize` fields; `curl -X POST http://localhost:5000/api/tasks -H "Content-Type: application/json" -d '{"title":"Fix bug","description":"...","priority":3,"assigneeId":"00000000-0000-0000-0000-000000000000"}'` should return 201; a request without `title` should return 400 with a validation error body; `curl -H "X-Correlation-Id: 11111111-1111-1111-1111-111111111111" -H "X-Source: web" http://localhost:5000/api/tasks/correlated` should return the correlation context.
11. Add a unit test or at least one `curl` script `tests/smoke.sh` that runs all the scenarios and prints `OK`/`FAIL`. Record the expected HTTP codes in comments inside the script.

#### Requirements
- Target framework `.NET 8`, language `C# 12`. Use top-level statements in `Program.cs`, record types for DTOs, collection expressions and pattern matching where appropriate. Enable `<Nullable>enable</Nullable>`.
- Every action parameter must have an explicit binding source attribute. No parameter may rely on the default lookup order (form → route → query). For simple types (`Guid`, `int`, `string`) use `[FromRoute]` or `[FromHeader]`; for complex DTOs use `[FromBody]` or `[FromQuery]`.
- At most one parameter per method may carry `[FromBody]`. The lesson warns directly: the body is a stream that can be read only once.
- All body DTOs must carry DataAnnotations for baseline validation: `[Required]`, `[StringLength]`, `[Range]`, `[EmailAddress]` where applicable. The controller must check `ModelState.IsValid` explicitly in POST/PUT, even though `[ApiController]` automatically returns 400 — this is required for clarity.
- The custom binder must implement `IModelBinder` and be registered through an `IModelBinderProvider` in `Program.cs`. Without provider registration the binder never runs — this is a common lesson mistake.
- Query DTO property names must differ from the URL names through `[FromQuery(Name = "...")]`, to demonstrate the mapping of short names (`q`, `page`, `size`) to readable properties (`Search`, `Page`, `PageSize`).
- Routes must use the `{id:guid}` constraint for identifiers — this removes ambiguity and immediately returns 404 for non-GUID strings.

#### Pitfalls
- **Empty body with `[FromBody]` on a simple type.** If you put `[FromBody] int x` and send an empty body, ASP.NET Core returns 400, because the body cannot be deserialized into `int`. For simple values use `[FromQuery]` or `[FromRoute]` — this rule comes from the lesson best practices.
- **Name clash between query and route.** If the route is `{id}` and the query also has `?id=...`, the binder may pick the value from an unexpected place by default. An explicit `[FromRoute]` or `[FromQuery]` resolves the ambiguity.
- **`[FromBody]` without a constructor or setters.** If the DTO is a class with private properties, the JSON deserializer returns `null` or an object with defaults. Use a `record` with positional parameters or a class with public properties — the lesson mentions this in common mistakes.
- **Forgotten `[ApiController]`.** Without it `ModelState.IsValid` is not checked automatically and invalid data reaches business logic. With `[ApiController]` an invalid model automatically yields 400, but an explicit check is still useful for readability and for MVC views.
- **Custom binder not registered.** If you implemented `IModelBinder` but never added the provider to `ModelBinderProviders`, the binder is never invoked and the parameter stays `null`. The lesson explicitly warns about this.
- **`Insert(0, ...)` vs `Add(...)`.** Standard providers are checked in order. If you add your provider at the end, the standard complex-type provider may capture the binding first. `Insert(0, ...)` guarantees priority.
- **`Guid.TryParse` and culture.** GUID parsing is culture-independent, but parsing numbers and dates is not. If your binder parses `DateTime` or `decimal`, use `TryParse` with `CultureInfo.InvariantCulture`.
- **`ModelBindingResult.Failed()` vs `Success(null)`.** `Failed()` means "no value found", `Success(null)` means "value found, but it is null". For mandatory headers use `Failed()` so the framework knows the binding failed.

#### Acceptance criteria
- [ ] The `TaskCatalog` project builds with `dotnet build` without errors or warnings.
- [ ] Every action parameter has an explicit binding source attribute (`[FromRoute]`, `[FromQuery]`, `[FromBody]`, `[FromHeader]`, `[FromServices]` or `[ModelBinder]`).
- [ ] `CreateTaskDto` and `UpdateTaskDto` are record types with DataAnnotations (`[Required]`, `[StringLength]`, `[Range]`).
- [ ] `TaskQueryDto` is decorated with `[FromQuery]` on every property, with renaming via `Name = "q"`, `"page"`, `"size"`, `"status"`, `"sort"`.
- [ ] `POST /api/tasks` with an empty `title` returns 400 and a validation error description in the response body.
- [ ] `POST /api/tasks` with a valid body returns 201 with a `Location` header.
- [ ] `GET /api/tasks/{id:guid}` with a non-GUID string returns 404 (thanks to the route constraint).
- [ ] `GET /api/tasks?q=...&page=2&size=10` correctly parses the query into `TaskQueryDto` and returns the values in the response.
- [ ] `GET /api/tasks/correlated` without the `X-Correlation-Id` and `X-Source` headers returns 400 or does not invoke the action with an invalid context.
- [ ] `CorrelationContextBinder` implements `IModelBinder`, `CorrelationContextBinderProvider` implements `IModelBinderProvider`.
- [ ] The provider is registered in `Program.cs` via `Insert(0, ...)` inside `AddControllers`.
- [ ] Actions use `[FromServices]` for `ILogger<TasksController>` instead of `new Logger()`.
- [ ] The smoke test `tests/smoke.sh` runs all scenarios and prints `OK` for each.
- [ ] The code uses C# 12 features (records, top-level statements, pattern matching and collection expressions where needed).
- [ ] `Program.cs` registers controllers and Swagger/OpenAPI for convenient manual testing.

#### Hints (no direct answer)
- Recall the default source order: form → route → query. Think about what would happen if you removed all attributes — and why explicit attributes make the code safer.
- For `CorrelationContextBinder` look at the lesson's `TenantContextBinder`: the structure is identical, only the header names and the validation logic differ.
- If `ModelState` looks empty, check that you have `[ApiController]` — without it validation errors are not accumulated automatically in the right shape.
- For `X-Request-Id` use `Guid.TryParse` inside the action or make the parameter `Guid?` to handle a missing header.
- Do not forget `Content-Type: application/json` in `curl` for POST/PUT — otherwise the body is not deserialized.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — TaskCatalog/Program.cs
// Registration of services, controllers and the custom binder

using Microsoft.AspNetCore.Mvc;
using TaskCatalog.Binders;
using TaskCatalog.Controllers;
using TaskCatalog.Models;

var builder = WebApplication.CreateBuilder(args);

// Register controllers and insert the custom binder provider at position 0.
builder.Services.AddControllers(options =>
{
    options.ModelBinderProviders.Insert(0, new CorrelationContextBinderProvider());
});

// Swagger for manual endpoint testing.
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.MapControllers();
app.Run();
```

```csharp
// TaskCatalog/Models/Dtos.cs — body and query DTOs

using System.ComponentModel.DataAnnotations;
using Microsoft.AspNetCore.Mvc;

// Body for task creation
public record CreateTaskDto(
    [Required(ErrorMessage = "Title is required.")] string Title,
    [StringLength(500, ErrorMessage = "Description must be under 500 chars.")] string Description,
    [Range(1, 5, ErrorMessage = "Priority must be between 1 and 5.")] int Priority,
    [Required(ErrorMessage = "AssigneeId is required.")] Guid AssigneeId);

// Body for update (PATCH-like)
public record UpdateTaskDto(
    [StringLength(500)] string? Description,
    [Range(1, 5)] int? Priority);

// Query DTO for list filters
// Short URL names (q, page, size), readable property names
public record TaskQueryDto(
    [FromQuery(Name = "q")] string? Search,
    [FromQuery(Name = "status")] string? Status,
    [FromQuery(Name = "page")] int Page = 1,
    [FromQuery(Name = "size")] [Range(1, 100)] int PageSize = 20,
    [FromQuery(Name = "sort")] string? Sort);

// Correlation context assembled from two headers
public record CorrelationContext(Guid CorrelationId, string Source);
```

```csharp
// TaskCatalog/Binders/CorrelationContextBinder.cs — custom binder

using Microsoft.AspNetCore.Mvc.ModelBinding;

public class CorrelationContextBinder : IModelBinder
{
    public Task BindModelAsync(ModelBindingContext bindingContext)
    {
        // Null-context guard
        if (bindingContext is null) throw new ArgumentNullException(nameof(bindingContext));

        var req = bindingContext.HttpContext.Request;
        var correlationRaw = req.Headers["X-Correlation-Id"].FirstOrDefault();
        var source = req.Headers["X-Source"].FirstOrDefault();

        // If the GUID fails to parse or the source is empty — fail the binding
        if (!Guid.TryParse(correlationRaw, out var correlationId)
            || string.IsNullOrWhiteSpace(source))
        {
            bindingContext.Result = ModelBindingResult.Failed();
            return Task.CompletedTask;
        }

        bindingContext.Result = ModelBindingResult.Success(
            new CorrelationContext(correlationId, source));
        return Task.CompletedTask;
    }
}

// Provider: links the CorrelationContext type to the binder
public class CorrelationContextBinderProvider : IModelBinderProvider
{
    public IModelBinder? GetBinder(ModelBinderProviderContext context)
        => context.Metadata.ModelType == typeof(CorrelationContext)
            ? new CorrelationContextBinder()
            : null;
}
```

```csharp
// TaskCatalog/Controllers/TasksController.cs

using Microsoft.AspNetCore.Mvc;

namespace TaskCatalog.Controllers;

[ApiController]                                  // automatic ModelState validation
[Route("api/[controller]")]
public class TasksController : ControllerBase
{
    private readonly ILogger<TasksController> _logger;

    // ILogger is injected via DI — in the action we use [FromServices] for demonstration
    public TasksController(ILogger<TasksController> logger) => _logger = logger;

    // GET api/tasks/{id:guid} — id from route
    [HttpGet("{id:guid}")]
    public IActionResult GetById([FromRoute] Guid id)
        => Ok(new { id, title = "Demo task", status = "open" });

    // GET api/tasks?q=...&page=2&size=10 — filters from query
    [HttpGet]
    public IActionResult List([FromQuery] TaskQueryDto query)
        => Ok(new { query.Search, query.Status, query.Page, query.PageSize, query.Sort });

    // POST api/tasks — request body (JSON)
    [HttpPost]
    public IActionResult Create([FromBody] CreateTaskDto dto)
    {
        // ModelState is checked automatically with [ApiController], explicit here for clarity
        if (!ModelState.IsValid)
            return BadRequest(ModelState);

        var id = Guid.NewGuid();
        return CreatedAtAction(nameof(GetById), new { id }, new { id, dto.Title, dto.Priority });
    }

    // PUT api/tasks/{id:guid} — id from route, body from JSON
    [HttpPut("{id:guid}")]
    public IActionResult Update([FromRoute] Guid id, [FromBody] UpdateTaskDto dto)
    {
        if (!ModelState.IsValid)
            return BadRequest(ModelState);

        return NoContent();
    }

    // GET api/tasks/correlated — context from two headers + requestId from header + logger from DI
    [HttpGet("correlated")]
    public IActionResult Correlated(
        [ModelBinder(typeof(CorrelationContextBinder))] CorrelationContext ctx,
        [FromHeader(Name = "X-Request-Id")] Guid? requestId,
        [FromServices] ILogger<TasksController> logger)
    {
        logger.LogInformation("Correlation {CorrelationId} from {Source}, request {RequestId}",
            ctx.CorrelationId, ctx.Source, requestId);

        return Ok(new { ctx.CorrelationId, ctx.Source, requestId });
    }
}
```

Line-by-line walk-through. In `Program.cs` we call `AddControllers` with a lambda that configures `MvcOptions` and inserts `CorrelationContextBinderProvider` through `Insert(0, ...)` — this guarantees our provider is checked before the standard ones, as the lesson requires. `AddSwaggerGen` adds a UI for manual testing. In `Dtos.cs` every DTO is a record with positional parameters and DataAnnotations: the lesson stresses that a record with public properties is required for JSON deserialization through `[FromBody]`. `TaskQueryDto` uses `[FromQuery(Name = "q")]` — this is the technique for mapping short URL names to readable properties mentioned in best practices. `CorrelationContext` is a type that has no standard attribute, so a custom binder is required.

In `CorrelationContextBinder` we mirror the structure of the lesson's `TenantContextBinder`: we read the headers, parse the GUID with `TryParse`, call `ModelBindingResult.Failed()` on failure and `Success(new CorrelationContext(...))` on success. The provider checks `ModelType` and returns the binder only for `CorrelationContext`, otherwise `null` — the framework continues looking for a suitable provider. Without registration in `Program.cs` the binder would never be called and the parameter would stay `null` — this is the common lesson mistake.

In `TasksController` the `[ApiController]` attribute enables automatic `ModelState` validation and an automatic 400 on errors; we still check `ModelState.IsValid` explicitly for clarity, as the best practice demands. `[FromRoute] Guid id` takes the identifier strictly from the route, and the `{id:guid}` constraint in the template guarantees 404 for non-GUID strings. `[FromQuery] TaskQueryDto` gathers all filters into one object — this is the recommended pattern for requests with many parameters. `[FromBody]` is used exactly once per method — the lesson warns directly that the body is read once. `Correlated` demonstrates three sources at once: the custom binder for `CorrelationContext`, `[FromHeader]` for `X-Request-Id` and `[FromServices]` for the logger. The last one shows that DI dependencies can also be obtained as action parameters, not only through the constructor. All these elements reinforce the lesson topics directly: binding sources, lookup order, validation and custom binders.

#### Going deeper (bonus)
1. Add a `[FromForm]` action `POST /api/tasks/import` that accepts an `IFormFile` and a string field `format` through `[FromForm]`. Demonstrate that form values are the first source in the default lookup order.
2. Implement `IValidatableObject` on `CreateTaskDto`: if `Priority == 5`, then `Description` must be at least 20 characters long (a business rule). Return a `ValidationResult` through `Validate(ValidationContext)`.
3. Add a second custom binder `SortOrderBinder` that parses the string `sort=created_at:desc` into a `SortOrder(string Field, bool Descending)` object. Register both providers and verify that the `Insert` order does not break their work.
4. Wire up FluentValidation via the `FluentValidation.AspNetCore` package and write a validator for `UpdateTaskDto` that forbids both `Description` and `Priority` being empty at the same time (at least one field must change).

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `TaskCatalog` собирается без ошибок и предупреждений.
- [ ] Каждый параметр действия помечен атрибутом источника привязки.
- [ ] DTO для тела и query созданы как record-типы с DataAnnotations.
- [ ] Кастомный биндер `CorrelationContextBinder` реализован и зарегистрирован через провайдер.
- [ ] Smoke-тест `tests/smoke.sh` проходит все сценарии.
- [ ] Использованы возможности C# 12 (top-level statements, records, при необходимости — pattern matching).
- [ ] The `TaskCatalog` project builds without errors or warnings.
- [ ] Every action parameter is decorated with a binding source attribute.
- [ ] Body and query DTOs are record types with DataAnnotations.
- [ ] The custom binder `CorrelationContextBinder` is implemented and registered via a provider.
- [ ] The smoke test `tests/smoke.sh` passes all scenarios.
- [ ] C# 12 features are used (top-level statements, records, pattern matching where needed).

#### Ресурсы / Resources
- [Microsoft Learn — Model binding in ASP.NET Core](https://learn.microsoft.com/aspnet/core/mvc/models/model-binding)
- [Microsoft Learn — Custom model binders](https://learn.microsoft.com/aspnet/core/mvc/models/custom-model-binding)
- [Microsoft Learn — Validation in ASP.NET Core](https://learn.microsoft.com/aspnet/core/mvc/models/validation)
- [Microsoft Learn — `[ApiController]` attribute](https://learn.microsoft.com/aspnet/core/web-api/)
- [FluentValidation — official docs](https://docs.fluentvalidation.net/)
