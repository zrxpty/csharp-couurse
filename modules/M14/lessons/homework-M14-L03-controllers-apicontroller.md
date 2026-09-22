---
[← К уроку M14-L03](lesson-M14-L03-controllers-apicontroller.md) | [⬆ К модулю M14](../README.md) | [Следующее ДЗ →](homework-M14-L04-model-binding.md)
---

### Домашнее задание M14-L03: Контроллеры, [ApiController], attribute routing / Homework M14-L03: Controllers, [ApiController], attribute routing

**Урок / Lesson:** M14-L03
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться строить тонкие, самодокументируемые Web API-контроллеры на ASP.NET Core 8: применять `[ApiController]` и `ControllerBase`, грамотно проектировать attribute routing с ограничениями, выбирать правильный тип возвращаемого значения (`ActionResult<T>` против `IActionResult`), аннотировать ответы через `[ProducesResponseType]`, обрабатывать `CancellationToken` и возвращать корректные `ProblemDetails`/`CreatedAtAction`. (EN) Learn to build thin, self-documenting Web API controllers on ASP.NET Core 8: apply `[ApiController]` and `ControllerBase`, design attribute routing with constraints, choose the right return type (`ActionResult<T>` vs `IActionResult`), annotate responses with `[ProducesResponseType]`, propagate `CancellationToken`, and return correct `ProblemDetails`/`CreatedAtAction`.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит стандартный набор .NET 8 Web API: `ControllerBase` + `[ApiController]` + attribute routing + `ActionResult<T>`. ДЗ закрепляет именно этот набор: вы построите CRUD-контроллер для сущности «курсы», повторив соглашения авто-валидации 400, вывод источника привязки, токены маршрутов `[controller]` и ограничения `{id:int}`, а также обязательные `[ProducesResponseType]` для OpenAPI.
(EN) The lesson introduces the standard .NET 8 Web API toolkit: `ControllerBase` + `[ApiController]` + attribute routing + `ActionResult<T>`. This homework cements exactly that toolkit: you will build a CRUD controller for a "courses" entity, reproducing the automatic 400 validation convention, binding source inference, the `[controller]` route token and the `{id:int}` constraint, and the mandatory `[ProducesResponseType]` annotations for OpenAPI.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы продолжаете разрабатывать учебную платформу CourseShop, которая уже имеет Minimal API-эндпоинты из предыдущего урока (M14-L02). Команда архитекторов решила, что по мере роста числа эндпоинтов Minimal API начинает терять читаемость: маршруты, модели запросов и обработчики оказываются размазаны по `Program.cs`, а единых соглашений по валидации и форматированию ошибок нет. Чтобы навести порядок и подготовить кодовую базу к Swagger-документации и внешним клиентам, принято решение перевести API каталога курсов на контроллеры с `[ApiController]`.

Этот переход — не косметический. Атрибут `[ApiController]` включает набор соглашений, которые радикально меняют стиль кода: исчезает ручная проверка `ModelState.IsValid`, сложные типы автоматически берутся из тела, а ошибки превращаются в структурированные `ProblemDetails` (RFC 7807). Attribute routing, в отличие от convention-based, «привязывает» URL напрямую к методу, что делает маршруты предсказуемыми и читаемыми — особенно когда используются токены вроде `[controller]` и ограничения вроде `{id:int}`. Правильный выбор типа возвращаемого значения (`ActionResult<T>` вместо голого `IActionResult` или конкретного типа) даёт и типобезопасность, и гибкость статус-кодов, и корректную OpenAPI-схему одновременно.

В этом задании вы создадите отдельный проект `CourseShop.Api`, подключите контроллеры через `AddControllers`, настроите `ProblemDetails` и Swagger, реализуете `CoursesController` с полным CRUD, а также сервис-заглушку `ICourseService` с in-memory хранилищем. Вы должны будете осознанно применять каждую конвенцию урока, а не просто скопировать шаблон.

#### Что нужно сделать (пошагово)

1. **Создайте решение и проект.** В каталоге `C:\projects\course\labs\M14-L03` выполните:
   ```bash
   dotnet new sln -n CourseShop
   dotnet new webapi -n CourseShop.Api -o src/CourseShop.Api --use-controllers --no-https
   dotnet sln add src/CourseShop.Api/CourseShop.Api.csproj
   ```
   Ключ `--use-controllers` включает шаблон с контроллерами (а не Minimal API). Ключ `--no-https` убирает редирект, чтобы упростить локальную отладку на `http://localhost:5000`.

2. **Включите `[ApiController]` на уровне сборки.** Откройте `Program.cs` и добавьте в начало файла (до `var builder = WebApplication.CreateBuilder(args);`):
   ```csharp
   [assembly: Microsoft.AspNetCore.Mvc.ApiController]
   ```
   Это снимает необходимость дублировать атрибут на каждом контроллере. Убедитесь, что в `builder.Services` вызвано `AddControllers()` (а не `AddControllersAsServices` отдельно — достаточно первого).

3. **Настройте ProblemDetails и Swagger.** В `Program.cs`:
   ```csharp
   builder.Services.AddControllers();
   builder.Services.AddProblemDetails();
   builder.Services.AddEndpointsApiExplorer();
   builder.Services.AddSwaggerGen();
   ```
   В конвейере (после `var app = builder.Build();`) добавьте:
   ```csharp
   if (app.Environment.IsDevelopment())
   {
       app.UseSwagger();
       app.UseSwaggerUI();
   }
   app.MapControllers();
   ```

4. **Создайте модели.** В файле `Models/CourseContracts.cs` определите:
   ```csharp
   public sealed record CreateCourseRequest(
       [property: System.ComponentModel.DataAnnotations.Required]
       string Title,
       [property: System.ComponentModel.DataAnnotations.Range(1, 500)]
       int DurationHours,
       [property: System.ComponentModel.DataAnnotations.StringLength(2000)]
       string? Description);

   public sealed record UpdateCourseRequest(string Title, int DurationHours, string? Description);
   public sealed record CourseDto(int Id, string Title, int DurationHours, string? Description);
   ```
   Обратите внимание: аннотации `Required`/`Range`/`StringLength` нужны, чтобы авто-валидация `[ApiController]` имела что проверять.

5. **Реализуйте сервис.** В `Services/ICourseService.cs` объявите интерфейс, а в `Services/InMemoryCourseService.cs` — реализацию с `ConcurrentDictionary<int, CourseDto>` и счётчиком-генератором `Id`. Все методы должны принимать `CancellationToken` и пробрасывать его в `Task.Delay` (для эмуляции I/O) или `ThrowIfCancellationRequested`.

6. **Зарегистрируйте сервис в DI:**
   ```csharp
   builder.Services.AddSingleton<ICourseService, InMemoryCourseService>();
   ```

7. **Напишите контроллер.** В `Controllers/CoursesController.cs` создайте класс, наследующий `ControllerBase`, с атрибутом `[Route("api/[controller]")]`. Реализуйте методы:
   - `GET /api/courses` — пагинация через `[FromQuery] int page = 1, [FromQuery] int pageSize = 20`;
   - `GET /api/courses/{id:int}` — возврат `ActionResult<CourseDto>` с `NotFound()` для отсутствующего;
   - `POST /api/courses` — `[FromBody] CreateCourseRequest`, возврат `CreatedAtAction(nameof(GetById), new { id = created.Id }, created)`;
   - `PUT /api/courses/{id:int}` — `IActionResult`, возврат `NoContent()` или `NotFound()`;
   - `DELETE /api/courses/{id:int}` — `IActionResult`, возврат `NoContent()` или `NotFound()`.

   Каждый метод снабдите `[ProducesResponseType]` с корректными кодами и типами. Не пишите `if (!ModelState.IsValid)` — это уже делает `[ApiController]`.

8. **Запустите и проверьте.**
   ```bash
   dotnet run --project src/CourseShop.Api
   ```
   Откройте `http://localhost:5000/swagger` и прогоните сценарии:
   - POST с пустым `Title` → ожидайте 400 и JSON `ProblemDetails` с полем `errors`;
   - POST валидный → ожидайте 201 с заголовком `Location: http://localhost:5000/api/courses/1`;
   - GET по новому `id` → 200;
   - GET по `id=999` → 404;
   - PUT по несуществующему `id` → 404;
   - DELETE существующего → 204;
   - GET `/api/courses/abc` → 404 от роутинга (ограничение `int` не сработает, маршрут не сопоставится).

9. **Проверьте отмену.** Запустите запрос GET с задержкой (`Task.Delay(500, ct)` в сервисе) и прервите его в Swagger/Postman — должна прийти отмена без исключения в логе сервера.

#### Требования к решению

Решение должно представлять собой компилируемый проект .NET 8 (целевой фреймворк `net8.0`) с рабочим `dotnet run`. Контроллер обязан наследовать `ControllerBase`, а не `Controller` — наличие `Controller` в Web API считается ошибкой архитектуры, поскольку тянет Razor-инфраструктуру. Атрибут `[ApiController]` должен быть включён либо на сборке (`[assembly: ApiController]` в `Program.cs`), либо на самом классе; в решении выберите один подход и используйте его консистентно.

Все маршруты должны использовать attribute routing с токеном `[controller]` на классе и относительными шаблонами на методах — дублирование префикса `api/courses` в шаблонах методов запрещено. Параметр `{id}` обязан иметь ограничение `{id:int}`, чтобы отсечь нечисловые значения на уровне роутинга, а не валидацией. Типы возвращаемых значений: `ActionResult<T>` для действий с телом ответа (GET one, POST, GET list) и `IActionResult` для действий без тела (PUT, DELETE — `NoContent`).

Каждое действие должно иметь хотя бы один `[ProducesResponseType]` с реальным статус-кодом и типом, где есть тело. Запрещена ручная проверка `ModelState.IsValid` — её наличие в коде при включённом `[ApiController]` трактуется как непонимание конвенции. Все асинхронные методы контроллера должны принимать `CancellationToken ct = default` и пробрасывать его в вызовы сервиса. Бизнес-логика (поиск, создание, обновление, удаление) должна жить в `ICourseService`; контроллер не должен содержать `ConcurrentDictionary` или циклов по коллекции. Для действия POST обязательно возвращать `CreatedAtAction` с именем действия GET-by-id, чтобы клиенту приходил заголовок `Location`.

#### Тонкости и подводные камни

- **`Controller` против `ControllerBase`.** Если случайно унаследоваться от `Controller`, вы получите поддержку Razor и методов `View()`/`ViewBag`, которые в Web API не нужны и лишь увеличивают поверхность атаки. Проверьте `: ControllerBase` явно.
- **Двойная валидация.** Многие новички пишут `if (!ModelState.IsValid) return BadRequest(ModelState);` прямо под `[ApiController]`. Это не только избыточно — фреймворк уже возвращает `ValidationProblemDetails` на 400, — но и хуже: ваш ручной ответ не пройдёт через `ProblemDetails`-фабрику и не будет единообразным с остальными ошибками. Удалите эту проверку.
- **Дублирование маршрутов.** Шаблон вида `[Route("api/courses")]` на классе и `[HttpGet("api/courses/{id}")]` на методе даст физический путь `api/courses/api/courses/{id}` — частый баг. Используйте `[Route("api/[controller]")]` + `[HttpGet("{id:int}")]`.
- **`CreatedAtAction` и имя действия.** Если передать неверное `nameof`, на этапе вызова POST (а не компиляции) будет брошен `InvalidOperationException` «No route matches the supplied values». Имя должно совпадать с методом, на который указывает ссылка, а параметры `new { id = ... }` — с шаблоном маршрута этого метода.
- **`[FromQuery]` для примитивов.** При включённом `[ApiController]` вывод источника работает: сложные типы идут из тела, примитивы — из query/route. Но явный `[FromQuery]` на пагинации улучшает читаемость и OpenAPI-схему — оставляйте его.
- **Ограничение `{id:int}` и 404.** Запрос `/api/courses/abc` не попадёт в действие `GetById` вообще: роутинг не сопоставит маршрут, и фреймворк вернёт 404 на уровне конвейера. Это нормально и предпочтительнее, чем ловить `abc` в действии и пытаться его парсить.
- **`CancellationToken` и DI-синглтон.** Сервис зарегистрирован как `Singleton`, но `CancellationToken` — это per-request. Передавайте его как аргумент метода, а не храните в поле сервиса.
- **`StatusCodes` вместо магических чисел.** Используйте `StatusCodes.Status201Created` и т.п., а не `201` — это даёт читаемость и защиту от опечаток.

#### Критерии приёмки

- [ ] Проект `CourseShop.Api` компилируется без warning-ов и запускается через `dotnet run`.
- [ ] В `Program.cs` включён `[assembly: ApiController]` (либо атрибут стоит на каждом контроллере).
- [ ] Вызвано `AddControllers()` и `AddProblemDetails()`; Swagger подключён и доступен на `/swagger`.
- [ ] `CoursesController` наследует `ControllerBase` (не `Controller`).
- [ ] На классе стоит `[Route("api/[controller]")]`, шаблоны методов относительны и не дублируют префикс.
- [ ] Везде, где есть `id`, используется ограничение `{id:int}`.
- [ ] GET list возвращает `ActionResult<IEnumerable<CourseDto>>` с пагинацией через `[FromQuery]`.
- [ ] GET one возвращает `ActionResult<CourseDto>` и `NotFound()` для отсутствующего курса.
- [ ] POST возвращает `CreatedAtAction(nameof(GetById), ...)` со статусом 201 и заголовком `Location`.
- [ ] PUT/DELETE возвращают `IActionResult` (`NoContent`/`NotFound`).
- [ ] Каждое действие аннотировано `[ProducesResponseType]` с корректными статусами и типами.
- [ ] В коде нет ручной проверки `ModelState.IsValid`.
- [ ] Все асинхронные методы принимают `CancellationToken ct = default` и пробрасывают его в сервис.
- [ ] Бизнес-логика вынесена в `ICourseService`/`InMemoryCourseService`; контроллер тонкий.
- [ ] Невалидный POST (пустой `Title`) возвращает 400 и `ProblemDetails` с полем `errors`.
- [ ] GET `/api/courses/abc` возвращает 404 от роутинга, не доходя до действия.

#### Подсказки (без прямого ответа)

- Вспомните аналогию из урока: `IActionResult` — «коробка с сюрпризом», конкретный тип — «прозрачный пакет», `ActionResult<T>` — «коробка с этикеткой». Для действий с телом выбирайте «этикетку».
- Для генерации `Id` в in-memory сервисе подойдёт `Interlocked.Increment(ref _nextId)`.
- Чтобы `CreatedAtAction` не падал, убедитесь, что `nameof(GetById)` указывает на метод с шаблоном `{id:int}`, и что в `new { id = created.Id }` имя параметра совпадает с именем сегмента маршрута.
- Для проверки отмены в сервисе вызывайте `ct.ThrowIfCancellationRequested()` перед каждой «тяжёлой» операцией.
- Swagger покажет коды ответов только тогда, когда вы явно повесите `[ProducesResponseType]` — иначе в UI будет только «200» по умолчанию.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8
// Program.cs — точка входа с top-level statements.
// Program.cs — entry point with top-level statements.

[assembly: Microsoft.AspNetCore.Mvc.ApiController] // Включаем соглашения на уровне сборки.
                                                 // Enable conventions at assembly level.

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();          // Регистрируем контроллеры.
builder.Services.AddProblemDetails();       // Единый формат ошибок RFC 7807.
builder.Services.AddEndpointsApiExplorer(); // Метаданные для Swagger.
builder.Services.AddSwaggerGen();           // Генерация OpenAPI.

// In-memory сервис — Singleton: состояние живёт всё время процесса.
// In-memory service — Singleton: state lives for the process lifetime.
builder.Services.AddSingleton<ICourseService, InMemoryCourseService>();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.MapControllers(); // Включаем attribute routing для контроллеров.
app.Run();
```

```csharp
// Controllers/CoursesController.cs
using Microsoft.AspNetCore.Mvc;
using CourseShop.Api.Models;
using CourseShop.Api.Services;

namespace CourseShop.Api.Controllers;

// [ApiController] наследуется от сборки, но [Route] нужен на классе.
// [ApiController] is inherited from the assembly, but [Route] is required on the class.
[Route("api/[controller]")] // /api/courses — токен [controller] = "courses".
public sealed class CoursesController(ICourseService service, ILogger<CoursesController> logger)
    : ControllerBase // Не Controller! / Not Controller!
{
    // GET /api/courses?page=1&pageSize=20
    [HttpGet]
    [ProducesResponseType(typeof(IEnumerable<CourseDto>), StatusCodes.Status200OK)]
    public async Task<ActionResult<IEnumerable<CourseDto>>> GetAll(
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20,
        CancellationToken ct = default)
    {
        var items = await service.GetPageAsync(page, pageSize, ct);
        return Ok(items); // ActionResult<T>: и тип, и статус. / Both type and status.
    }

    // GET /api/courses/{id} — ограничение int отсекает /api/courses/abc на уровне роутинга.
    // GET /api/courses/{id} — int constraint rejects /api/courses/abc at routing.
    [HttpGet("{id:int}", Name = "GetCourseById")]
    [ProducesResponseType(typeof(CourseDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<CourseDto>> GetById(int id, CancellationToken ct = default)
    {
        var course = await service.GetByIdAsync(id, ct);
        if (course is null) // pattern matching: is null.
        {
            logger.LogWarning("Course {CourseId} not found", id);
            return NotFound(); // 404.
        }
        return course; // Неявный 200 OK для ActionResult<T>. / Implicit 200 OK for ActionResult<T>.
    }

    // POST /api/courses — ModelState НЕ проверяем: [ApiController] сам вернёт 400.
    // POST /api/courses — do NOT check ModelState: [ApiController] returns 400 itself.
    [HttpPost]
    [ProducesResponseType(typeof(CourseDto), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<CourseDto>> Create(
        [FromBody] CreateCourseRequest request,
        CancellationToken ct = default)
    {
        var created = await service.CreateAsync(request, ct);
        // 201 + Location: /api/courses/42. Имя действия должно совпадать с GetById.
        // 201 + Location header. Action name must match GetById.
        return CreatedAtAction(nameof(GetById), new { id = created.Id }, created);
    }

    // PUT /api/courses/{id} — идемпотентное полное обновление, тело ответа пустое.
    // PUT /api/courses/{id} — idempotent full update, no response body.
    [HttpPut("{id:int}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Update(
        int id,
        [FromBody] UpdateCourseRequest request,
        CancellationToken ct = default)
    {
        var success = await service.UpdateAsync(id, request, ct);
        return success ? NoContent() : NotFound(); // Тернарный оператор для краткости.
                                                    // Ternary for brevity.
    }

    // DELETE /api/courses/{id} — без тела ответа, поэтому IActionResult.
    // DELETE /api/courses/{id} — no body, so IActionResult.
    [HttpDelete("{id:int}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(int id, CancellationToken ct = default)
    {
        var success = await service.DeleteAsync(id, ct);
        return success ? NoContent() : NotFound();
    }
}
```

```csharp
// Services/InMemoryCourseService.cs
using System.Collections.Concurrent;
using CourseShop.Api.Models;

namespace CourseShop.Api.Services;

public sealed class InMemoryCourseService : ICourseService
{
    private readonly ConcurrentDictionary<int, CourseDto> _store = new();
    private int _nextId;

    public Task<IEnumerable<CourseDto>> GetPageAsync(int page, int pageSize, CancellationToken ct)
    {
        ct.ThrowIfCancellationRequested();
        var items = _store.Values
            .OrderBy(c => c.Id)
            .Skip((page - 1) * pageSize)
            .Take(pageSize);
        return Task.FromResult(items);
    }

    public Task<CourseDto?> GetByIdAsync(int id, CancellationToken ct)
    {
        ct.ThrowIfCancellationRequested();
        _store.TryGetValue(id, out var course);
        return Task.FromResult(course);
    }

    public Task<CourseDto> CreateAsync(CreateCourseRequest request, CancellationToken ct)
    {
        ct.ThrowIfCancellationRequested();
        var id = Interlocked.Increment(ref _nextId); // Потокобезопасная генерация Id.
                                                      // Thread-safe Id generation.
        var dto = new CourseDto(id, request.Title, request.DurationHours, request.Description);
        _store[id] = dto;
        return Task.FromResult(dto);
    }

    public Task<bool> UpdateAsync(int id, UpdateCourseRequest request, CancellationToken ct)
    {
        ct.ThrowIfCancellationRequested();
        if (!_store.ContainsKey(id)) return Task.FromResult(false);
        _store[id] = new CourseDto(id, request.Title, request.DurationHours, request.Description);
        return Task.FromResult(true);
    }

    public Task<bool> DeleteAsync(int id, CancellationToken ct)
    {
        ct.ThrowIfCancellationRequested();
        return Task.FromResult(_store.TryRemove(id, out _));
    }
}
```

**Разбор по строкам.** `[assembly: ApiController]` в `Program.cs` включает соглашения для всех контроллеров сборки — это best practice из урока, избавляющая от дублирования атрибута на каждом классе. `AddControllers()` регистрирует инфраструктуру MVC для API (без Razor-views); `AddProblemDetails()` активирует единый RFC 7807-формат ошибок, что особенно важно для авто-валидации 400. `AddSwaggerGen` вместе с `AddEndpointsApiExplorer` даёт Swagger UI, который читает `[ProducesResponseType]`.

Контроллер использует **primary constructor** (C# 12): `CoursesController(ICourseService service, ...)` — это сокращает boilerplate по сравнению с явным полем и конструктором. `sealed` запрещает наследование, что рекомендуется для контроллеров. `[Route("api/[controller]")]` раскрывается в `api/courses` по имени класса (суффикс `Controller` отбрасывается) — это устраняет дублирование префикса. `GetAll` принимает пагинацию через `[FromQuery]` с дефолтами — явная аннотация улучшает читаемость и OpenAPI-схему, хотя `[ApiController]` и так вывел бы источник. Возврат `Ok(items)` оборачивает результат в 200; `ActionResult<IEnumerable<CourseDto>>` сохраняет тип для Swagger.

`GetById` использует pattern matching `is null` для проверки отсутствия — идиома C# 12. `NotFound()` возвращает 404 без тела; `return course` неявно конвертируется в 200 OK благодаря неявному оператору `ActionResult<T>`. Ограничение `{id:int}` в шаблоне маршрута отсекает нечисловые значения до контроллера — запрос `/api/courses/abc` получит 404 от роутинга, а не 400 от валидации. `Name = "GetCourseById"` здесь не используется в `CreatedAtAction` (там передаётся `nameof(GetById)`), но даёт маршруту человекочитаемое имя для диагностики.

`Create` НЕ содержит `ModelState.IsValid` — это ключевая конвенция урока: `[ApiController]` сам вернёт `ValidationProblemDetails` со списком ошибок по каждому полю. `CreatedAtAction(nameof(GetById), new { id = created.Id }, created)` генерирует 201 + заголовок `Location: http://localhost:5000/api/courses/1`; если бы `nameof` не совпал с именем метода GET-by-id или параметр маршрута назывался бы иначе, был бы брошен `InvalidOperationException` во время выполнения.

`Update` и `Delete` возвращают `IActionResult` (а не `ActionResult<T>`), потому что у ответа нет тела — только статус `204 NoContent` или `404 Not Found`. Урок явно рекомендует `IActionResult` для действий без тела ответа. Тернарный оператор `success ? NoContent() : NotFound()` компактен и читаем.

Сервис `InMemoryCourseService` зарегистрирован как `Singleton` и хранит данные в `ConcurrentDictionary`. `Interlocked.Increment` обеспечивает потокобезопасную генерацию `Id` при конкурентных POST. `ct.ThrowIfCancellationRequested()` в каждом методе позволяет отменять «тяжёлые» операции — это best practice урока по работе с `CancellationToken`. Контроллер остаётся тонким: вся работа с состоянием — в сервисе, контроллер лишь проверяет результат и формирует HTTP-ответ.

#### Задания на углубление (бонус)

1. **Глобальная обработка исключений.** Добавьте middleware-фильтр `IExceptionHandler` (через `builder.Services.AddExceptionHandler<GlobalExceptionHandler>()` и `app.UseExceptionHandler()`), который ловит необработанные исключения и возвращает `ProblemDetails` с кодом 500. Сравните поведение с дефолтным `AddProblemDetails()`.
2. **Версионирование.** Добавьте пакет `Asp.Versioning.Mvc` и создайте вторую версию контроллера `CoursesV2Controller` с маршрутом `api/v2/courses`, отличающуюся форматом DTO (например, поле `DurationMinutes` вместо `DurationHours`). Настройте `AddApiVersioning` с report-headers.
3. **Rate-limiting.** Подключите `Microsoft.AspNetCore.RateLimiting` и настройте лимитер «fixed window» 5 запросов в минуту на эндпоинт POST. Верните 429 с `Retry-After`.
4. **Контрактные тесты.** С помощью `Microsoft.AspNetCore.Mvc.Testing` напишите интеграционный тест, который стартует приложение в памяти, шлёт POST без `Title` и asserts, что ответ — 400 с `ProblemDetails`, содержащим `errors.Title`.

---

## Statement in English / Постановка на английском

#### Context & motivation

You continue to develop the CourseShop learning platform, which already has Minimal API endpoints from the previous lesson (M14-L02). The architecture team has decided that, as the number of endpoints grows, Minimal API is losing readability: routes, request models, and handlers end up scattered across `Program.cs`, and there are no shared conventions for validation and error shaping. To bring order to the codebase and prepare it for Swagger documentation and external clients, the team decides to migrate the course-catalog API to controllers with `[ApiController]`.

This migration is not cosmetic. The `[ApiController]` attribute enables a set of conventions that radically change the coding style: manual `ModelState.IsValid` checks disappear, complex types are automatically bound from the body, and errors become structured `ProblemDetails` (RFC 7807). Attribute routing, unlike convention-based routing, attaches a URL directly to a method, which makes routes predictable and readable — especially when tokens like `[controller]` and constraints like `{id:int}` are used. Choosing the right return type (`ActionResult<T>` instead of a bare `IActionResult` or a concrete type) gives you type safety, status-code flexibility, and a correct OpenAPI schema all at once.

In this assignment you will create a separate `CourseShop.Api` project, wire up controllers via `AddControllers`, configure `ProblemDetails` and Swagger, implement a `CoursesController` with full CRUD, and write a stub service `ICourseService` backed by an in-memory store. You must consciously apply each convention from the lesson, not just copy a template.

#### What to do step by step

1. **Create the solution and project.** In the `C:\projects\course\labs\M14-L03` directory run:
   ```bash
   dotnet new sln -n CourseShop
   dotnet new webapi -n CourseShop.Api -o src/CourseShop.Api --use-controllers --no-https
   dotnet sln add src/CourseShop.Api/CourseShop.Api.csproj
   ```
   The `--use-controllers` flag selects the controllers template (not Minimal API). The `--no-https` flag removes the HTTPS redirect to simplify local debugging on `http://localhost:5000`.

2. **Enable `[ApiController]` at the assembly level.** Open `Program.cs` and add at the top of the file (before `var builder = WebApplication.CreateBuilder(args);`):
   ```csharp
   [assembly: Microsoft.AspNetCore.Mvc.ApiController]
   ```
   This removes the need to repeat the attribute on every controller. Make sure `AddControllers()` is called in `builder.Services`.

3. **Configure ProblemDetails and Swagger.** In `Program.cs`:
   ```csharp
   builder.Services.AddControllers();
   builder.Services.AddProblemDetails();
   builder.Services.AddEndpointsApiExplorer();
   builder.Services.AddSwaggerGen();
   ```
   In the pipeline (after `var app = builder.Build();`) add:
   ```csharp
   if (app.Environment.IsDevelopment())
   {
       app.UseSwagger();
       app.UseSwaggerUI();
   }
   app.MapControllers();
   ```

4. **Create the models.** In `Models/CourseContracts.cs` define:
   ```csharp
   public sealed record CreateCourseRequest(
       [property: System.ComponentModel.DataAnnotations.Required]
       string Title,
       [property: System.ComponentModel.DataAnnotations.Range(1, 500)]
       int DurationHours,
       [property: System.ComponentModel.DataAnnotations.StringLength(2000)]
       string? Description);

   public sealed record UpdateCourseRequest(string Title, int DurationHours, string? Description);
   public sealed record CourseDto(int Id, string Title, int DurationHours, string? Description);
   ```
   Note: the `Required`/`Range`/`StringLength` annotations are needed so that the `[ApiController]` automatic validation has something to check.

5. **Implement the service.** In `Services/ICourseService.cs` declare the interface, and in `Services/InMemoryCourseService.cs` provide an implementation backed by a `ConcurrentDictionary<int, CourseDto>` and an `Id` counter. All methods must accept `CancellationToken` and propagate it to `Task.Delay` (to emulate I/O) or call `ThrowIfCancellationRequested`.

6. **Register the service in DI:**
   ```csharp
   builder.Services.AddSingleton<ICourseService, InMemoryCourseService>();
   ```

7. **Write the controller.** In `Controllers/CoursesController.cs` create a class inheriting from `ControllerBase`, with the `[Route("api/[controller]")]` attribute. Implement the methods:
   - `GET /api/courses` — pagination via `[FromQuery] int page = 1, [FromQuery] int pageSize = 20`;
   - `GET /api/courses/{id:int}` — return `ActionResult<CourseDto>` with `NotFound()` for a missing course;
   - `POST /api/courses` — `[FromBody] CreateCourseRequest`, return `CreatedAtAction(nameof(GetById), new { id = created.Id }, created)`;
   - `PUT /api/courses/{id:int}` — `IActionResult`, return `NoContent()` or `NotFound()`;
   - `DELETE /api/courses/{id:int}` — `IActionResult`, return `NoContent()` or `NotFound()`.

   Decorate each method with `[ProducesResponseType]` using correct codes and types. Do NOT write `if (!ModelState.IsValid)` — `[ApiController]` already does it.

8. **Run and verify.**
   ```bash
   dotnet run --project src/CourseShop.Api
   ```
   Open `http://localhost:5000/swagger` and run the scenarios:
   - POST with an empty `Title` → expect 400 and a `ProblemDetails` JSON with an `errors` field;
   - POST a valid payload → expect 201 with a `Location: http://localhost:5000/api/courses/1` header;
   - GET by the new `id` → 200;
   - GET with `id=999` → 404;
   - PUT against a non-existent `id` → 404;
   - DELETE of an existing course → 204;
   - GET `/api/courses/abc` → 404 from routing (the `int` constraint does not match, so the route is not matched).

9. **Verify cancellation.** Run a GET request with a delay (`Task.Delay(500, ct)` in the service) and abort it in Swagger/Postman — cancellation should arrive without an exception in the server log.

#### Requirements

The solution must be a compilable .NET 8 project (target framework `net8.0`) with a working `dotnet run`. The controller must inherit from `ControllerBase`, not `Controller` — having `Controller` in a Web API is an architecture mistake because it drags in the Razor infrastructure. The `[ApiController]` attribute must be enabled either at the assembly level (`[assembly: ApiController]` in `Program.cs`) or on the class itself; pick one approach and use it consistently.

All routes must use attribute routing with the `[controller]` token on the class and relative templates on the methods — duplicating the `api/courses` prefix in method templates is forbidden. The `id` parameter must carry the `{id:int}` constraint so non-numeric values are rejected at the routing layer rather than by validation. Return types: `ActionResult<T>` for actions with a response body (GET one, POST, GET list) and `IActionResult` for actions without a body (PUT, DELETE — `NoContent`).

Every action must have at least one `[ProducesResponseType]` with the real status code and, where there is a body, the type. Manual `ModelState.IsValid` checks are forbidden — their presence alongside `[ApiController]` shows a misunderstanding of the convention. All asynchronous controller methods must accept `CancellationToken ct = default` and pass it down to service calls. Business logic (lookup, create, update, delete) must live in `ICourseService`; the controller must not contain a `ConcurrentDictionary` or loops over the collection. The POST action must return `CreatedAtAction` with the GET-by-id action name so the client receives a `Location` header.

#### Pitfalls

- **`Controller` vs `ControllerBase`.** If you accidentally inherit from `Controller`, you gain Razor support and methods like `View()`/`ViewBag` that a Web API does not need and that only increase the attack surface. Verify `: ControllerBase` explicitly.
- **Double validation.** Many beginners write `if (!ModelState.IsValid) return BadRequest(ModelState);` right under `[ApiController]`. This is not only redundant — the framework already returns a `ValidationProblemDetails` on 400 — but worse: your manual response bypasses the `ProblemDetails` factory and is inconsistent with other errors. Remove this check.
- **Route duplication.** A template like `[Route("api/courses")]` on the class and `[HttpGet("api/courses/{id}")]` on the method yields the physical path `api/courses/api/courses/{id}` — a common bug. Use `[Route("api/[controller]")]` + `[HttpGet("{id:int}")]`.
- **`CreatedAtAction` and the action name.** If you pass the wrong `nameof`, an `InvalidOperationException` "No route matches the supplied values" is thrown at POST invocation time (not at compile time). The name must match the action you link to, and the `new { id = ... }` parameters must match that action's route template.
- **`[FromQuery]` for primitives.** With `[ApiController]` enabled, binding source inference works: complex types come from the body, primitives from query/route. But an explicit `[FromQuery]` on pagination improves readability and the OpenAPI schema — keep it.
- **The `{id:int}` constraint and 404.** A request to `/api/courses/abc` never reaches the `GetById` action: routing does not match the route, and the framework returns a 404 at the pipeline level. This is expected and preferable to catching `abc` inside the action and trying to parse it.
- **`CancellationToken` and a DI singleton.** The service is registered as `Singleton`, but `CancellationToken` is per-request. Pass it as a method argument, never store it in a service field.
- **`StatusCodes` instead of magic numbers.** Use `StatusCodes.Status201Created` etc., not `201` — this gives readability and protection from typos.

#### Acceptance criteria

- [ ] The `CourseShop.Api` project compiles without warnings and runs via `dotnet run`.
- [ ] `[assembly: ApiController]` is enabled in `Program.cs` (or the attribute is on every controller).
- [ ] `AddControllers()` and `AddProblemDetails()` are called; Swagger is wired up and reachable at `/swagger`.
- [ ] `CoursesController` inherits from `ControllerBase` (not `Controller`).
- [ ] The class has `[Route("api/[controller]")]`; method templates are relative and do not duplicate the prefix.
- [ ] Wherever `id` appears, the `{id:int}` constraint is used.
- [ ] GET list returns `ActionResult<IEnumerable<CourseDto>>` with `[FromQuery]` pagination.
- [ ] GET one returns `ActionResult<CourseDto>` and `NotFound()` for a missing course.
- [ ] POST returns `CreatedAtAction(nameof(GetById), ...)` with status 201 and a `Location` header.
- [ ] PUT/DELETE return `IActionResult` (`NoContent`/`NotFound`).
- [ ] Every action is annotated with `[ProducesResponseType]` using correct statuses and types.
- [ ] There is no manual `ModelState.IsValid` check in the code.
- [ ] All asynchronous methods accept `CancellationToken ct = default` and pass it to the service.
- [ ] Business logic is extracted into `ICourseService`/`InMemoryCourseService`; the controller stays thin.
- [ ] An invalid POST (empty `Title`) returns 400 and a `ProblemDetails` with an `errors` field.
- [ ] GET `/api/courses/abc` returns a 404 from routing, never reaching the action.

#### Hints

- Recall the analogy from the lesson: `IActionResult` is a "surprise box", a concrete type is a "transparent bag", and `ActionResult<T>` is a "labeled box". For actions with a body, choose the "labeled box".
- For `Id` generation in the in-memory service, `Interlocked.Increment(ref _nextId)` works well.
- To keep `CreatedAtAction` from throwing, make sure `nameof(GetById)` points to a method with a `{id:int}` template, and that the name in `new { id = created.Id }` matches the route segment name.
- To test cancellation in the service, call `ct.ThrowIfCancellationRequested()` before each "heavy" operation.
- Swagger shows response codes only when you explicitly attach `[ProducesResponseType]` — otherwise the UI shows only the default "200".

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8
// Program.cs — entry point with top-level statements.

[assembly: Microsoft.AspNetCore.Mvc.ApiController] // Enable conventions at assembly level.

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();          // Register controllers.
builder.Services.AddProblemDetails();       // Unified RFC 7807 error format.
builder.Services.AddEndpointsApiExplorer(); // Metadata for Swagger.
builder.Services.AddSwaggerGen();           // OpenAPI generation.

// In-memory service — Singleton: state lives for the process lifetime.
builder.Services.AddSingleton<ICourseService, InMemoryCourseService>();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.MapControllers(); // Enable attribute routing for controllers.
app.Run();
```

```csharp
// Controllers/CoursesController.cs
using Microsoft.AspNetCore.Mvc;
using CourseShop.Api.Models;
using CourseShop.Api.Services;

namespace CourseShop.Api.Controllers;

// [ApiController] is inherited from the assembly, but [Route] is required on the class.
[Route("api/[controller]")] // /api/courses — [controller] token = "courses".
public sealed class CoursesController(ICourseService service, ILogger<CoursesController> logger)
    : ControllerBase // Not Controller!
{
    // GET /api/courses?page=1&pageSize=20
    [HttpGet]
    [ProducesResponseType(typeof(IEnumerable<CourseDto>), StatusCodes.Status200OK)]
    public async Task<ActionResult<IEnumerable<CourseDto>>> GetAll(
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20,
        CancellationToken ct = default)
    {
        var items = await service.GetPageAsync(page, pageSize, ct);
        return Ok(items); // ActionResult<T>: both type and status.
    }

    // GET /api/courses/{id} — int constraint rejects /api/courses/abc at routing.
    [HttpGet("{id:int}", Name = "GetCourseById")]
    [ProducesResponseType(typeof(CourseDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<CourseDto>> GetById(int id, CancellationToken ct = default)
    {
        var course = await service.GetByIdAsync(id, ct);
        if (course is null) // pattern matching: is null.
        {
            logger.LogWarning("Course {CourseId} not found", id);
            return NotFound(); // 404.
        }
        return course; // Implicit 200 OK for ActionResult<T>.
    }

    // POST /api/courses — do NOT check ModelState: [ApiController] returns 400 itself.
    [HttpPost]
    [ProducesResponseType(typeof(CourseDto), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<CourseDto>> Create(
        [FromBody] CreateProductRequest request,
        CancellationToken ct = default)
    {
        var created = await service.CreateAsync(request, ct);
        // 201 + Location: /api/courses/42. Action name must match GetById.
        return CreatedAtAction(nameof(GetById), new { id = created.Id }, created);
    }

    // PUT /api/courses/{id} — idempotent full update, no response body.
    [HttpPut("{id:int}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Update(
        int id,
        [FromBody] UpdateCourseRequest request,
        CancellationToken ct = default)
    {
        var success = await service.UpdateAsync(id, request, ct);
        return success ? NoContent() : NotFound(); // Ternary for brevity.
    }

    // DELETE /api/courses/{id} — no body, so IActionResult.
    [HttpDelete("{id:int}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(int id, CancellationToken ct = default)
    {
        var success = await service.DeleteAsync(id, ct);
        return success ? NoContent() : NotFound();
    }
}
```

```csharp
// Services/InMemoryCourseService.cs
using System.Collections.Concurrent;
using CourseShop.Api.Models;

namespace CourseShop.Api.Services;

public sealed class InMemoryCourseService : ICourseService
{
    private readonly ConcurrentDictionary<int, CourseDto> _store = new();
    private int _nextId;

    public Task<IEnumerable<CourseDto>> GetPageAsync(int page, int pageSize, CancellationToken ct)
    {
        ct.ThrowIfCancellationRequested();
        var items = _store.Values
            .OrderBy(c => c.Id)
            .Skip((page - 1) * pageSize)
            .Take(pageSize);
        return Task.FromResult(items);
    }

    public Task<CourseDto?> GetByIdAsync(int id, CancellationToken ct)
    {
        ct.ThrowIfCancellationRequested();
        _store.TryGetValue(id, out var course);
        return Task.FromResult(course);
    }

    public Task<CourseDto> CreateAsync(CreateCourseRequest request, CancellationToken ct)
    {
        ct.ThrowIfCancellationRequested();
        var id = Interlocked.Increment(ref _nextId); // Thread-safe Id generation.
        var dto = new CourseDto(id, request.Title, request.DurationHours, request.Description);
        _store[id] = dto;
        return Task.FromResult(dto);
    }

    public Task<bool> UpdateAsync(int id, UpdateCourseRequest request, CancellationToken ct)
    {
        ct.ThrowIfCancellationRequested();
        if (!_store.ContainsKey(id)) return Task.FromResult(false);
        _store[id] = new CourseDto(id, request.Title, request.DurationHours, request.Description);
        return Task.FromResult(true);
    }

    public Task<bool> DeleteAsync(int id, CancellationToken ct)
    {
        ct.ThrowIfCancellationRequested();
        return Task.FromResult(_store.TryRemove(id, out _));
    }
}
```

**Line-by-line walk-through.** `[assembly: ApiController]` in `Program.cs` enables conventions for every controller in the assembly — this is the lesson's best practice and removes attribute duplication. `AddControllers()` registers the MVC infrastructure for APIs (without Razor views); `AddProblemDetails()` activates the unified RFC 7807 error format, which is especially important for automatic 400 validation. `AddSwaggerGen` together with `AddEndpointsApiExplorer` produces a Swagger UI that reads `[ProducesResponseType]`.

The controller uses a **primary constructor** (C# 12): `CoursesController(ICourseService service, ...)` — this removes boilerplate compared to an explicit field and constructor. `sealed` forbids inheritance, which is recommended for controllers. `[Route("api/[controller]")]` expands to `api/courses` from the class name (the `Controller` suffix is stripped) — this eliminates prefix duplication. `GetAll` takes pagination via `[FromQuery]` with defaults — the explicit annotation improves readability and the OpenAPI schema, even though `[ApiController]` would infer the source anyway. Returning `Ok(items)` wraps the result in a 200; `ActionResult<IEnumerable<CourseDto>>` preserves the type for Swagger.

`GetById` uses the `is null` pattern for the missing-entity check — a C# 12 idiom. `NotFound()` returns a 404 with no body; `return course` implicitly converts to a 200 OK thanks to the implicit `ActionResult<T>` operator. The `{id:int}` constraint in the route template rejects non-numeric values before the controller is reached — a request to `/api/courses/abc` gets a 404 from routing, not a 400 from validation. `Name = "GetCourseById"` is not used by `CreatedAtAction` (which passes `nameof(GetById)`), but it gives the route a human-readable name for diagnostics.

`Create` does NOT contain `ModelState.IsValid` — this is the key convention from the lesson: `[ApiController]` itself returns a `ValidationProblemDetails` listing per-field errors. `CreatedAtAction(nameof(GetById), new { id = created.Id }, created)` produces a 201 + a `Location: http://localhost:5000/api/courses/1` header; if `nameof` did not match the GET-by-id method's name or the route parameter were named differently, an `InvalidOperationException` would be thrown at runtime.

`Update` and `Delete` return `IActionResult` (not `ActionResult<T>`) because the response has no body — only the `204 NoContent` or `404 Not Found` status. The lesson explicitly recommends `IActionResult` for actions without a response body. The `success ? NoContent() : NotFound()` ternary is compact and readable.

The `InMemoryCourseService` is registered as `Singleton` and stores data in a `ConcurrentDictionary`. `Interlocked.Increment` provides thread-safe `Id` generation under concurrent POSTs. `ct.ThrowIfCancellationRequested()` in each method allows "heavy" operations to be cancelled — this is the lesson's best practice for `CancellationToken`. The controller stays thin: all state work lives in the service; the controller only checks the result and shapes the HTTP response.

#### Going deeper (bonus)

1. **Global exception handling.** Add an `IExceptionHandler` (via `builder.Services.AddExceptionHandler<GlobalExceptionHandler>()` and `app.UseExceptionHandler()`) that catches unhandled exceptions and returns a `ProblemDetails` with status 500. Compare its behavior with the default `AddProblemDetails()`.
2. **Versioning.** Add the `Asp.Versioning.Mvc` package and create a second version of the controller, `CoursesV2Controller`, with the route `api/v2/courses` and a different DTO shape (e.g., `DurationMinutes` instead of `DurationHours`). Configure `AddApiVersioning` with report headers.
3. **Rate limiting.** Wire up `Microsoft.AspNetCore.RateLimiting` and configure a "fixed window" limiter of 5 requests per minute on the POST endpoint. Return 429 with `Retry-After`.
4. **Contract tests.** Using `Microsoft.AspNetCore.Mvc.Testing`, write an integration test that starts the app in memory, sends a POST without `Title`, and asserts that the response is a 400 `ProblemDetails` containing `errors.Title`.

---

#### Чек-лист сдачи / Submission checklist

- [ ] (RU) Файл `homework-M14-L03-controllers-apicontroller.md` создан в `modules/M14/lessons/`.
- [ ] (RU) Проект `CourseShop.Api` компилируется и запускается через `dotnet run`.
- [ ] (RU) `CoursesController` наследует `ControllerBase` и использует `[Route("api/[controller]")]`.
- [ ] (RU) Реализованы все пять действий CRUD с корректными типами возврата и `[ProducesResponseType]`.
- [ ] (RU) В коде нет ручной проверки `ModelState.IsValid`.
- [ ] (RU) `CancellationToken` пробрасывается во все асинхронные методы.
- [ ] (RU) Скриншот Swagger UI с перечнем эндпоинтов приложен к сдаче.
- [ ] (EN) The file `homework-M14-L03-controllers-apicontroller.md` exists in `modules/M14/lessons/`.
- [ ] (EN) The `CourseShop.Api` project compiles and runs via `dotnet run`.
- [ ] (EN) `CoursesController` inherits `ControllerBase` and uses `[Route("api/[controller]")]`.
- [ ] (EN) All five CRUD actions are implemented with correct return types and `[ProducesResponseType]`.
- [ ] (EN) No manual `ModelState.IsValid` check is present.
- [ ] (EN) `CancellationToken` is propagated to every asynchronous method.
- [ ] (EN) A Swagger UI screenshot listing the endpoints is attached to the submission.

#### Ресурсы / Resources
- [Microsoft Learn — Controllers and action return types](https://learn.microsoft.com/aspnet/core/web-api/action-return-types)
- [Microsoft Learn — [ApiController] attribute](https://learn.microsoft.com/aspnet/core/web-api/?view=aspnetcore-8.0#apicontroller-attribute)
- [Microsoft Learn — Routing in ASP.NET Core](https://learn.microsoft.com/aspnet/core/mvc/controllers/routing)
- [Microsoft Learn — ProblemDetails (RFC 7807)](https://learn.microsoft.com/aspnet/core/web-api/handle-errors)
- [Microsoft Learn — CancellationToken in ASP.NET Core](https://learn.microsoft.com/aspnet/core/performance/caching/response?view=aspnetcore-8.0)
