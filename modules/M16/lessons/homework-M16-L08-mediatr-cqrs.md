---
[← К уроку M16-L08](lesson-M16-L08-mediatr-cqrs.md) | [⬆ К модулю M16](../README.md) | [Следующее ДЗ →](homework-M16-L09-ddd-overview.md)
---

### Домашнее задание M16-L08: MediatR, CQRS (обзор) / Homework M16-L08: MediatR, CQRS (overview)

**Урок / Lesson:** M16-L08
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться применять MediatR как транспорт для CQRS: спроектировать команды и запросы как `IRequest`, реализовать обработчики, добавить хотя бы один конвейерное поведение (`IPipelineBehavior`), сделать контроллеры тонкими и обосновать, нужен ли CQRS в заданном сценарии. (EN) Learn to use MediatR as a CQRS transport: design commands and queries as `IRequest`, implement handlers, add at least one pipeline behavior (`IPipelineBehavior`), keep controllers thin, and justify whether CQRS is warranted for the given scenario.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит паттерн Mediator, библиотеку MediatR (`IRequest`, `IRequestHandler`, `IMediator.Send`) и архитектурный стиль CQRS, а также конвейерные поведения. ДЗ закрепляет всё это на практическом мини-проекте: вы построите разделённые стороны записи и чтения и реальное сквозное поведение, в точности следуя best practices и избегая частых ошибок, перечисленных в уроке.
(EN) The lesson introduces the Mediator pattern, the MediatR library (`IRequest`, `IRequestHandler`, `IMediator.Send`), the CQRS architectural style, and pipeline behaviors. This homework reinforces all of it on a practical mini-project: you will build separate write and read sides plus a real cross-cutting behavior, strictly following the lesson's best practices and avoiding the common mistakes it lists.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы — backend-разработчик в компании «TaskFlow», которая строит внутренний трекер задач. Сейчас за всё отвечает один «толстый» сервис `TaskService`: он и создаёт задачи, и меняет их статус, и отдаёт списки с фильтрами, и логирует, и валидирует. Сервис разросся до 800 строк, его тяжело тестировать, а любое изменение валидации приходится копировать в три места. На code review команда устала повторять одни и те же замечания: «почему бизнес-логика в контроллере», «почему возвращаем доменную сущность наружу», «где cancellation token».

На ретроспективе принято решение: для модуля задач ввести CQRS через MediatR. Команды (`CreateTaskCommand`, `ChangeTaskStatusCommand`) будут изменять состояние и возвращать минимум (идентификатор или ничего). Запросы (`GetTaskByIdQuery`, `ListTasksQuery`) будут читать данные и возвращать DTO, оптимизированные под чтение, минуя доменную модель на стороне чтения. Сквозную логику — логирование и валидацию — вынесут в конвейерные поведения, чтобы она жила в одном месте и применялась ко всем запросам автоматически. Контроллеры должны стать «тонкими»: принять вход, вызвать `IMediator.Send`, вернуть HTTP-результат.

Этот сценарий выбран не случайно: он достаточно мал, чтобы уложиться в 90–120 минут, но достаточно богат, чтобы проявить все ключевые идеи урока — разделение моделей, иммутабельные `record`-команды, DI-регистрацию обработчиков и поведений по сборке, корректную работу с `CancellationToken`, недопущение «божественного обработчика» и честную оценку того, нужен ли здесь CQRS вообще. По итогам вы должны уметь объяснить, где CQRS приносит пользу, а где — лишь лишние типы и файлы.

#### Что нужно сделать (пошагово)
1. Создайте solution и проект. Выполните `dotnet new web -n TaskFlow.Api -o TaskFlow.Api`, затем `dotnet new sln -n TaskFlow` и `dotnet sln add TaskFlow.Api/TaskFlow.Api.csproj`. Перейдите в папку проекта и добавьте MediatR: `dotnet add package MediatR --version 12.*`. Убедитесь, что в `TaskFlow.Api.csproj` целевой фреймворк — `net8.0`.
2. Разбейте код по папкам, отражающим CQRS: `Tasks/Commands`, `Tasks/Queries`, `Tasks/Behaviors`, `Tasks/Domain`, `Tasks/Infrastructure`, `Tasks/Dtos`. Команды — в `Commands`, запросы — в `Queries`; не смешивайте их в одном файле.
3. Опишите доменную модель и контракты инфраструктуры. В `Tasks/Domain` создайте `record TaskItem(Guid Id, Guid AssigneeId, string Title, string Description, TaskStatus Status, DateTime CreatedAt)` и `enum TaskStatus { New, InProgress, Done, Cancelled }`. В `Tasks/Infrastructure` опишите `ITaskRepository` (методы `AddAsync`, `UpdateAsync`, `GetByIdAsync`) и `ITaskReadModel` (метод `GetSummaryAsync`, `ListAsync`). Репозиторий — сторона записи; read-model — сторона чтения. Сделайте in-memory реализации для обоих, чтобы пример запускался без БД.
4. Реализуйте команду `CreateTaskCommand`. Это `sealed record CreateTaskCommand(Guid AssigneeId, string Title, string Description) : IRequest<Guid>`. Поля `Title` и `Description` пометьте атрибутами валидации (`[Required]`, `[StringLength(200, MinimumLength = 3)]`), чтобы конвейерное поведение валидации могло их проверить через `Validator.TryValidateObject`. Обработчик `CreateTaskHandler` должен реализовывать `IRequestHandler<CreateTaskCommand, Guid>`, создавать сущность, сохранять её через `ITaskRepository.AddAsync`, логировать создание через `ILogger` и возвращать `Guid`.
5. Реализуйте команду `ChangeTaskStatusCommand`. Это `sealed record ChangeTaskStatusCommand(Guid TaskId, TaskStatus NewStatus) : IRequest` (без возвращаемого значения — это `IRequest<Unit>`). Обработчик должен загружать задачу, проверять допустимость перехода статуса через `pattern matching` (например, из `Done` нельзя перейти в `InProgress`), бросать `InvalidOperationException` при недопустимом переходе, сохранять изменения и логировать результат.
6. Реализуйте запрос `GetTaskByIdQuery`. Это `sealed record GetTaskByIdQuery(Guid TaskId) : IRequest<TaskSummaryDto?>`. Обработчик читает из `ITaskReadModel.GetSummaryAsync` и возвращает DTO `TaskSummaryDto(Guid Id, string Title, TaskStatus Status, DateTime CreatedAt)` либо `null`. Запрос **не должен** ничего писать и **не должен** возвращать доменную сущность `TaskItem`.
7. Реализуйте запрос `ListTasksQuery` с фильтром. Это `sealed record ListTasksQuery(TaskStatus? StatusFilter, int Limit = 50) : IRequest<IReadOnlyList<TaskSummaryDto>>`. Обработчик делегирует чтение в `ITaskReadModel.ListAsync`. Проверьте, что `Limit` ограничен сверху (например, 200) — это бизнес-правило чтения.
8. Реализуйте конвейерное поведение `LoggingBehavior<TRequest, TResponse>`. Оно реализует `IPipelineBehavior<TRequest, TResponse>`, логирует тип запроса и длительность обработки через `Stopwatch`, всегда прокидывает `CancellationToken` в `next(ct)`. Используйте primary constructor `LoggingBehavior(ILogger<LoggingBehavior<TRequest,TResponse>> log)`.
9. Реализуйте конвейерное поведение `ValidationBehavior<TRequest, TResponse>` через `System.ComponentModel.DataAnnotations`, как в уроке: `Validator.TryValidateObject(req, ctx, results, validateAllProperties: true)`, при ошибке — `throw new ValidationException(...)`. На тип `TRequest` наложите ограничение `where TRequest : notnull`.
10. Зарегистрируйте всё в `Program.cs`: `builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssemblyContaining<CreateTaskHandler>());`, затем `builder.Services.AddScoped(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));` и `builder.Services.AddScoped(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));`. Порядок регистрации определяет порядок выполнения: сначала логирование, потом валидация. Зарегистрируйте репозитории и read-model как `Singleton` (in-memory) или `Scoped`.
11. Опишите тонкие эндпоинты через `app.MapGroup("/tasks")`. `POST /tasks` принимает `CreateTaskCommand` и вызывает `mediator.Send`, возвращая `Results.Created`. `PUT /tasks/{id}/status` принимает `ChangeTaskStatusCommand`. `GET /tasks/{id}` и `GET /tasks` соответствуют запросам. В каждом эндпоинте принимайте `CancellationToken ct` из параметров маршрута и передавайте в `Send`.
12. Запустите и проверьте: `dotnet run`. Выполните `curl -X POST http://localhost:5000/tasks -H "Content-Type: application/json" -d '{"assigneeId":"...","title":"Demo","description":"..."}'` — ожидается `201` с `id`. Проверьте `GET /tasks/{id}`. Отправьте команду с пустым `Title` — ожидается `400` от `ValidationBehavior`. Просмотрите логи: `LoggingBehavior` должен напечатать тип запроса и время.
13. Напишите unit-тесты (xUnit + NSubstitute или Moq): мок `ITaskRepository` и `ITaskReadModel`, проверка создания задачи, проверка отказа при недопустимом переходе статуса, проверка, что запрос не вызывает `AddAsync` у репозитория (контракт CQRS на стороне чтения). Тесты на поведение `ValidationBehavior` для невалидной и валидной команды.

#### Требования к решению
- Проект на C# 12 / .NET 8, целевой фреймворк `net8.0`; используйте top-level statements в `Program.cs`, primary constructors, `record` для команд/запросов/DTO, `pattern matching` для переходов статусов.
- Все команды и запросы — `sealed record`, оканчивающиеся на `Command`/`Query`; обработчики — `internal sealed class`, реализующие `IRequestHandler<,>`.
- Команды возвращают минимум (`Guid` или `Unit`), запросы возвращают DTO/read-model-проекции, а не `TaskItem`.
- Контроллеры/эндпоинты тонкие: только приём входа, вызов `IMediator.Send`, формирование HTTP-ответа; никакой бизнес-логики в них.
- Минимум два `IPipelineBehavior`: `LoggingBehavior` и `ValidationBehavior`; оба зарегистрированы в DI как open generic.
- Все вызовы `Send` и `Handle` прокидывают `CancellationToken`.
- Сторона записи (`ITaskRepository`) и сторона чтения (`ITaskReadModel`) разделены; запросы не вызывают методы записи.
- Обоснование «нужен ли CQRS» оформлено коротким комментарием в начале `Program.cs` или отдельным `README.md`: где CQRS оправдан, где был бы избыточен.
- Код компилируется без warning, `dotnet build` зелёный; `dotnet run` поднимает сервер; тесты зелёные.
- Используйте `InternalsVisibleTo` для тестовой сборки, если обработчики `internal sealed`.

#### Тонкости и подводные камни
- `IRequest` без аргумента — это `IRequest<Unit>`; команда «без возвращаемого значения» возвращает `Unit`, обработчик реализует `IRequestHandler<TRequest, Unit>` и возвращает `Unit.Value`. Не делайте команду «void» через сторонние обходные пути.
- Конвейерное поведение, не зарегистрированное в DI как `typeof(IPipelineBehavior<,>)`, **молча не вызывается** — это самая частая ошибка. Проверяйте через лог: если `LoggingBehavior` не пишет в лог до обработчика, регистрация неверная.
- Порядок выполнения behaviours совпадает с порядком регистрации: первым зарегистрирован — внешним в цепочке. Логирование обычно внешнее, валидация — внутреннее, чтобы логировать и валидные, и невалидные попытки.
- `Validator.TryValidateObject` проверяет только атрибуты `[Required]`, `[StringLength]` и т.п.; без атрибутов на `record` валидация ничего не найдёт и пропустит пустые значения. Не путайте с FluentValidation — это отдельная библиотека, упомянутая в уроке как альтернатива.
- На `TRequest` в `ValidationBehavior` ставьте `where TRequest : notnull` — иначе компилятор может выдать nullable-предупреждение, а `Validator.TryValidateObject` требует непустой объект.
- Запросы, возвращающие `null` (`TaskSummaryDto?`), корректно обрабатывайте в эндпоинте через `pattern matching`: `dto is null ? Results.NotFound() : Results.Ok(dto)`.
- Не возвращайте доменную сущность `TaskItem` из запросов — это нарушает инкапсуляцию и связывает API с внутренностями. Возвращайте только DTO.
- «Божественный» обработчик, обслуживающий несколько `IRequest`, невозможен в MediatR по контракту (один `IRequest` — один `IRequestHandler`), но частая ошибка — вынести общую логику в один метод сервисного класса и дёргать его из десяти обработчиков. Лучше — поведение или доменный сервис с ясной ответственностью.
- `CancellationToken` в ASP.NET Core: привязанный параметр `ct` в минимальном API автоматически отменяется при разрыве соединения клиентом. Если не прокидывать его в `Send` и в репозиторий, обработка продолжится после ухода клиента — расход ресурсов впустую.
- `AddMediatR` с `RegisterServicesFromAssemblyContaining<T>` сканирует сборку; обработчики и поведения в другой сборке не подхватятся автоматически — укажите нужную сборку явно или вынесите в общий extension-метод.
- Не применяйте CQRS «по умолчанию»: для простого CRUD на одной таблице без боли это избыточно. В ДЗ CQRS оправдан учебной целью и наличием разных моделей чтения/записи; в реальном проекте такое решение надо обосновать нагрузкой или сложностью.

#### Критерии приёмки
- [ ] Проект `TaskFlow.Api` на `net8.0` собирается без ошибок и warning.
- [ ] Установлен пакет MediatR 12.* и зарегистрирован через `AddMediatR` + `RegisterServicesFromAssemblyContaining`.
- [ ] `CreateTaskCommand` и `ChangeTaskStatusCommand` — `sealed record`, реализующие `IRequest<Guid>` и `IRequest` соответственно.
- [ ] `GetTaskByIdQuery` и `ListTasksQuery` — `sealed record`, реализующие `IRequest<TaskSummaryDto?>` и `IRequest<IReadOnlyList<TaskSummaryDto>>`.
- [ ] Каждый `IRequest` имеет ровно один `IRequestHandler`; нет «божественного» обработчика.
- [ ] Команды возвращают минимум (`Guid`/`Unit`); запросы возвращают DTO, не доменную сущность `TaskItem`.
- [ ] `LoggingBehavior` и `ValidationBehavior` реализуют `IPipelineBehavior<,>` и зарегистрированы как open generic в DI.
- [ ] `ValidationBehavior` действительно отклоняет команду с пустым `Title` (`400`/`ValidationException`).
- [ ] `LoggingBehavior` пишет в лог тип запроса и время выполнения (видно в консоли `dotnet run`).
- [ ] Все вызовы `IMediator.Send` и `Handle` прокидывают `CancellationToken`.
- [ ] Эндпоинты тонкие: только приём, `Send`, формирование ответа; бизнес-логика — в обработчиках и домене.
- [ ] Сторона записи (`ITaskRepository`) и сторона чтения (`ITaskReadModel`) разделены; запросы не вызывают `AddAsync`/`UpdateAsync`.
- [ ] Переход статуса реализован через `pattern matching`; недопустимый переход бросает `InvalidOperationException`.
- [ ] Unit-тесты покрывают: создание, отказ перехода статуса, контракт «запрос не пишет».
- [ ] В `Program.cs` или `README.md` дано краткое обоснование, где CQRS оправдан, а где был бы избыточен.

#### Подсказки (без прямого ответа)
- Вспомните из урока, что MediatR находит обработчик по типу запроса через DI — значит, достаточно правильно зарегистрировать сборку.
- Для команды без возвращаемого значения посмотрите в уроке разницу между `IRequest<T>` и `IRequest` (он же `IRequest<Unit>`).
- Для поведения валидации используйте шаблон из урока с `Validator.TryValidateObject` и `where TRequest : notnull`.
- Чтобы проверить, что запросы ничего не пишут, в тесте мокните `ITaskRepository` и убедитесь, что `AddAsync` не вызывается.
- Порядок behaviours = порядок регистрации; подумайте, что должно идти раньше — логирование или валидация.
- Для `MapGroup` и минимального API параметр `CancellationToken ct` берётся из маршрута/запроса автоматически — просто объявите его в сигнатуре.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — MediatR как транспорт для CQRS в модуле задач.
// Регистрация в Program.cs:
//   builder.Services.AddMediatR(cfg =>
//       cfg.RegisterServicesFromAssemblyContaining<CreateTaskHandler>());
//   builder.Services.AddScoped(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
//   builder.Services.AddScoped(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
//
// Обоснование CQRS: стороны записи и чтения различаются по форме данных
// (запись — доменная сущность + правила перехода; чтение — плоские summary),
// поэтому разделение оправдано. Для простого CRUD на одной таблице CQRS был бы избыточен.

using System.ComponentModel.DataAnnotations;
using MediatR;
using System.Diagnostics;

namespace TaskFlow.Api.Tasks;

// ── Домен и инфраструктура (in-memory для учебного запуска) ───────────
public enum TaskStatus { New, InProgress, Done, Cancelled }

public sealed record TaskItem(
    Guid Id, Guid AssigneeId, string Title, string Description,
    TaskStatus Status, DateTime CreatedAt);

public interface ITaskRepository
{
    Task AddAsync(TaskItem task, CancellationToken ct);
    Task UpdateAsync(TaskItem task, CancellationToken ct);
    Task<TaskItem?> GetByIdAsync(Guid id, CancellationToken ct);
}

public interface ITaskReadModel
{
    Task<TaskSummaryDto?> GetSummaryAsync(Guid id, CancellationToken ct);
    Task<IReadOnlyList<TaskSummaryDto>> ListAsync(TaskStatus? filter, int limit, CancellationToken ct);
}

public sealed record TaskSummaryDto(Guid Id, string Title, TaskStatus Status, DateTime CreatedAt);

// ── КОМАНДЫ (сторона записи) ──────────────────────────────────────────
public sealed record CreateTaskCommand(
    Guid AssigneeId,
    [Required] [StringLength(200, MinimumLength = 3)] string Title,
    [Required] string Description) : IRequest<Guid>;

internal sealed class CreateTaskHandler(ITaskRepository repo, ILogger<CreateTaskHandler> log)
    : IRequestHandler<CreateTaskCommand, Guid>
{
    public async Task<Guid> Handle(CreateTaskCommand cmd, CancellationToken ct)
    {
        var task = new TaskItem(
            Id: Guid.NewGuid(),
            AssigneeId: cmd.AssigneeId,
            Title: cmd.Title,
            Description: cmd.Description,
            Status: TaskStatus.New,
            CreatedAt: DateTime.UtcNow);

        await repo.AddAsync(task, ct);
        log.LogInformation("Task {TaskId} created for {Assignee}", task.Id, task.AssigneeId);
        return task.Id; // Команда возвращает минимум — только идентификатор.
    }
}

public sealed record ChangeTaskStatusCommand(Guid TaskId, TaskStatus NewStatus) : IRequest;

internal sealed class ChangeTaskStatusHandler(ITaskRepository repo, ILogger<ChangeTaskStatusHandler> log)
    : IRequestHandler<ChangeTaskStatusCommand, Unit>
{
    public async Task<Unit> Handle(ChangeTaskStatusCommand cmd, CancellationToken ct)
    {
        var task = await repo.GetByIdAsync(cmd.TaskId, ct)
            ?? throw new InvalidOperationException("Task not found.");

        // Переход статуса через pattern matching — бизнес-правило домена.
        task = (task.Status, cmd.NewStatus) switch
        {
            (TaskStatus.New, TaskStatus.InProgress) => task with { Status = TaskStatus.InProgress },
            (TaskStatus.InProgress, TaskStatus.Done) => task with { Status = TaskStatus.Done },
            (TaskStatus.New or TaskStatus.InProgress, TaskStatus.Cancelled) => task with { Status = TaskStatus.Cancelled },
            _ => throw new InvalidOperationException($"Invalid transition {task.Status} -> {cmd.NewStatus}")
        };

        await repo.UpdateAsync(task, ct);
        log.LogInformation("Task {TaskId} -> {Status}", task.Id, task.Status);
        return Unit.Value; // IRequest без результата возвращает Unit.
    }
}

// ── ЗАПРОСЫ (сторона чтения) ──────────────────────────────────────────
public sealed record GetTaskByIdQuery(Guid TaskId) : IRequest<TaskSummaryDto?>;

internal sealed class GetTaskByIdHandler(ITaskReadModel readModel)
    : IRequestHandler<GetTaskByIdQuery, TaskSummaryDto?>
{
    public Task<TaskSummaryDto?> Handle(GetTaskByIdQuery q, CancellationToken ct)
        => readModel.GetSummaryAsync(q.TaskId, ct); // Только чтение, DTO, не доменная сущность.
}

public sealed record ListTasksQuery(TaskStatus? StatusFilter, int Limit = 50)
    : IRequest<IReadOnlyList<TaskSummaryDto>>;

internal sealed class ListTasksHandler(ITaskReadModel readModel)
    : IRequestHandler<ListTasksQuery, IReadOnlyList<TaskSummaryDto>>
{
    public Task<IReadOnlyList<TaskSummaryDto>> Handle(ListTasksQuery q, CancellationToken ct)
        => readModel.ListAsync(q.StatusFilter, Math.Min(q.Limit, 200), ct); // Ограничение чтения.
}

// ── КОНВЕЙЕРНЫЕ ПОВЕДЕНИЯ ────────────────────────────────────────────
public sealed class LoggingBehavior<TRequest, TResponse>(ILogger<LoggingBehavior<TRequest, TResponse>> log)
    : IPipelineBehavior<TRequest, TResponse> where TRequest : notnull
{
    public async Task<TResponse> Handle(TRequest req, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var sw = Stopwatch.StartNew();
        log.LogInformation("Handling {Request}", typeof(TRequest).Name);
        try { return await next(ct); }
        finally { log.LogInformation("Handled {Request} in {Ms} ms", typeof(TRequest).Name, sw.ElapsedMilliseconds); }
    }
}

public sealed class ValidationBehavior<TRequest, TResponse>(ILogger<ValidationBehavior<TRequest, TResponse>> log)
    : IPipelineBehavior<TRequest, TResponse> where TRequest : notnull
{
    public async Task<TResponse> Handle(TRequest req, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var ctx = new ValidationContext(req);
        var results = new List<ValidationResult>();
        if (!Validator.TryValidateObject(req, ctx, results, validateAllProperties: true))
        {
            var errors = string.Join("; ", results.Select(r => r.ErrorMessage));
            throw new ValidationException($"Validation failed for {typeof(TRequest).Name}: {errors}");
        }
        log.LogDebug("Validated {Request}", typeof(TRequest).Name);
        return await next(ct);
    }
}
```

Разбор по строкам. `CreateTaskCommand` — иммутабельный `record` с атрибутами валидации; это и контракт, и транспорт, и DTO запроса одновременно, как рекомендует урок. `IRequest<Guid>` явно говорит, что команда возвращает минимум — идентификатор. Обработчик `internal sealed` с primary constructor принимает зависимости (`ITaskRepository`, `ILogger`) — MediatR сам создаст его через DI. Внутри `Handle` создаётся доменная сущность, сохраняется через репозиторий (сторона записи), логируется событие и возвращается `Guid`. Никакой логики в контроллере не остаётся.

`ChangeTaskStatusCommand` — `IRequest` без аргумента, то есть `IRequest<Unit>`; обработчик возвращает `Unit.Value`. Переход статуса реализован через `pattern matching` с `switch` по кортежу `(текущий статус, новый статус)`: урок прямо поощряет pattern matching для бизнес-правил. Недопустимый переход бросает `InvalidOperationException`, что легко тестируется. `task with { Status = ... }` использует семантику `record` — немутабельное обновление.

Запросы `GetTaskByIdQuery` и `ListTasksQuery` возвращают DTO `TaskSummaryDto` и `IReadOnlyList<...>` — никогда `TaskItem`. Обработчики тонкие: делегируют в `ITaskReadModel`, который физически читает из проекции, а не из доменного репозитория. Это и есть разделение моделей CQRS из урока. `Math.Min(q.Limit, 200)` защищает чтение от перегрузки — бизнес-правило read-side.

`LoggingBehavior` и `ValidationBehavior` — `IPipelineBehavior<,>` с `where TRequest : notnull`. `LoggingBehavior` оборачивает `next(ct)` в `try/finally`, чтобы залогировать время и в случае успеха, и в случае исключения. `ValidationBehavior` использует `Validator.TryValidateObject` с `validateAllProperties: true`, как в примере урока; при ошибке бросает `ValidationException` с человекочитаемым списком ошибок. Оба зарегистрированы в DI как `typeof(IPipelineBehavior<,>)` — без этого они бы молча не вызвались, что урок прямо называет частой ошибкой. Порядок регистрации (сначала логирование, потом валидация) даёт цепочку «лог → валидация → обработчик». Все `Handle` и `next` прокидывают `CancellationToken`, как требует best practice урока.

#### Задания на углубление (бонус)
1. Замените `DataAnnotations` на FluentValidation: добавьте пакет `FluentValidation.DependencyInjectionExtensions`, опишите `CreateTaskCommandValidator`, зарегистрируйте валидаторы по сборке и перепишите `ValidationBehavior` на `IEnumerable<IValidator<TRequest>>`. Сравните выразительность.
2. Добавьте кэширующее поведение `CachingBehavior` для запросов: для `GetTaskByIdQuery` кэшируйте результат в `IMemoryCache` на 30 секунд, инвалидация — по команде `ChangeTaskStatusCommand`. Подумайте, как отличить команду от запроса в generic-поведении (маркерный интерфейс `IQuery`).
3. Введите доменное событие `TaskCreatedEvent : INotification` и обработчик `TaskCreatedHandler`, который пишет уведомление в лог. Отправьте событие через `IMediator.Publish` из `CreateTaskHandler` и сравните с прямым вызовом сервиса.
4. Напишите интеграционный тест на конвейер: через `WebApplicationFactory` отправьте невалидную команду и убедитесь, что `LoggingBehavior` отработал ДО `ValidationBehavior` (по порядку записей в логе).

---

## Statement in English / Постановка на английском

#### Context & motivation
You are a backend developer at a company called TaskFlow that builds an internal task tracker. Today a single fat `TaskService` is responsible for everything: creating tasks, changing their status, returning filtered lists, logging, and validation. The service has grown to 800 lines, it is painful to test, and every change to validation must be copied into three places. On code review the team keeps repeating the same remarks: "why is business logic in the controller", "why are we returning a domain entity to the outside world", "where is the cancellation token".

At the retrospective the team decides to introduce CQRS backed by MediatR for the tasks module. Commands (`CreateTaskCommand`, `ChangeTaskStatusCommand`) will mutate state and return the minimum (an identifier, or nothing). Queries (`GetTaskByIdQuery`, `ListTasksQuery`) will read data and return DTOs optimized for reading, bypassing the domain model on the read side. Cross-cutting logic — logging and validation — will be moved into pipeline behaviors so that it lives in one place and applies to every request automatically. Controllers must become thin: accept input, call `IMediator.Send`, shape the HTTP response.

This scenario is not accidental: it is small enough to fit into 90–120 minutes, yet rich enough to exercise every key idea of the lesson — model separation, immutable `record` commands, assembly-based DI registration of handlers and behaviors, correct `CancellationToken` handling, avoiding the god-handler, and the honest assessment of whether CQRS is even needed. By the end you should be able to explain where CQRS pays off and where it merely adds types and files.

#### What to do step by step
1. Create the solution and project. Run `dotnet new web -n TaskFlow.Api -o TaskFlow.Api`, then `dotnet new sln -n TaskFlow` and `dotnet sln add TaskFlow.Api/TaskFlow.Api.csproj`. Inside the project folder add MediatR: `dotnet add package MediatR --version 12.*`. Confirm that `TaskFlow.Api.csproj` targets `net8.0`.
2. Split the code into folders that mirror CQRS: `Tasks/Commands`, `Tasks/Queries`, `Tasks/Behaviors`, `Tasks/Domain`, `Tasks/Infrastructure`, `Tasks/Dtos`. Commands live in `Commands`, queries in `Queries`; do not mix them in one file.
3. Describe the domain model and infrastructure contracts. In `Tasks/Domain` create `record TaskItem(Guid Id, Guid AssigneeId, string Title, string Description, TaskStatus Status, DateTime CreatedAt)` and `enum TaskStatus { New, InProgress, Done, Cancelled }`. In `Tasks/Infrastructure` declare `ITaskRepository` (`AddAsync`, `UpdateAsync`, `GetByIdAsync`) and `ITaskReadModel` (`GetSummaryAsync`, `ListAsync`). The repository is the write side; the read model is the read side. Provide in-memory implementations for both so the sample runs without a database.
4. Implement `CreateTaskCommand`. It is a `sealed record CreateTaskCommand(Guid AssigneeId, string Title, string Description) : IRequest<Guid>`. Mark `Title` and `Description` with validation attributes (`[Required]`, `[StringLength(200, MinimumLength = 3)]`) so the validation behavior can check them through `Validator.TryValidateObject`. The handler `CreateTaskHandler` implements `IRequestHandler<CreateTaskCommand, Guid>`, builds the entity, persists it through `ITaskRepository.AddAsync`, logs creation via `ILogger`, and returns the `Guid`.
5. Implement `ChangeTaskStatusCommand`. It is a `sealed record ChangeTaskStatusCommand(Guid TaskId, TaskStatus NewStatus) : IRequest` (no return value — that is `IRequest<Unit>`). The handler must load the task, validate the status transition using `pattern matching` (for example, `Done` cannot become `InProgress`), throw `InvalidOperationException` on an invalid transition, persist the change, and log the result.
6. Implement `GetTaskByIdQuery`. It is `sealed record GetTaskByIdQuery(Guid TaskId) : IRequest<TaskSummaryDto?>`. The handler reads from `ITaskReadModel.GetSummaryAsync` and returns either `TaskSummaryDto(Guid Id, string Title, TaskStatus Status, DateTime CreatedAt)` or `null`. The query must not write anything and must not return the `TaskItem` domain entity.
7. Implement `ListTasksQuery` with a filter. It is `sealed record ListTasksQuery(TaskStatus? StatusFilter, int Limit = 50) : IRequest<IReadOnlyList<TaskSummaryDto>>`. The handler delegates to `ITaskReadModel.ListAsync`. Enforce an upper bound on `Limit` (for example, 200) — that is a read-side business rule.
8. Implement `LoggingBehavior<TRequest, TResponse>` as an `IPipelineBehavior`. Log the request type and the handling duration using `Stopwatch`, always propagate `CancellationToken` to `next(ct)`. Use a primary constructor `LoggingBehavior(ILogger<LoggingBehavior<TRequest,TResponse>> log)`.
9. Implement `ValidationBehavior<TRequest, TResponse>` via `System.ComponentModel.DataAnnotations`, exactly as in the lesson: `Validator.TryValidateObject(req, ctx, results, validateAllProperties: true)`, and on failure `throw new ValidationException(...)`. Constrain the type with `where TRequest : notnull`.
10. Register everything in `Program.cs`: `builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssemblyContaining<CreateTaskHandler>());`, then `builder.Services.AddScoped(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));` and `builder.Services.AddScoped(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));`. Registration order defines execution order: logging first, then validation. Register repositories and the read model as `Singleton` (in-memory) or `Scoped`.
11. Expose thin endpoints through `app.MapGroup("/tasks")`. `POST /tasks` accepts `CreateTaskCommand`, calls `mediator.Send`, and returns `Results.Created`. `PUT /tasks/{id}/status` accepts `ChangeTaskStatusCommand`. `GET /tasks/{id}` and `GET /tasks` map to the queries. In each endpoint accept a `CancellationToken ct` parameter and forward it to `Send`.
12. Run and verify: `dotnet run`. Send `curl -X POST http://localhost:5000/tasks -H "Content-Type: application/json" -d '{"assigneeId":"...","title":"Demo","description":"..."}'` — expect `201` with an `id`. Verify `GET /tasks/{id}`. Send a command with an empty `Title` — expect `400` from `ValidationBehavior`. Inspect the logs: `LoggingBehavior` should print the request type and the elapsed time.
13. Write unit tests (xUnit + NSubstitute or Moq): mock `ITaskRepository` and `ITaskReadModel`, verify task creation, verify that an invalid status transition is rejected, and verify that a query never calls `AddAsync` on the repository (the CQRS contract on the read side). Add tests for `ValidationBehavior` covering both valid and invalid commands.

#### Requirements
- C# 12 / .NET 8 project targeting `net8.0`; use top-level statements in `Program.cs`, primary constructors, `record` for commands/queries/DTOs, and `pattern matching` for status transitions.
- All commands and queries are `sealed record` types suffixed with `Command`/`Query`; handlers are `internal sealed class` implementing `IRequestHandler<,>`.
- Commands return the minimum (`Guid` or `Unit`); queries return DTO/read-model projections, never `TaskItem`.
- Controllers/endpoints are thin: only input handling, `IMediator.Send`, and HTTP shaping; no business logic inside them.
- At least two `IPipelineBehavior` types — `LoggingBehavior` and `ValidationBehavior` — both registered in DI as open generics.
- Every `Send` and `Handle` call propagates `CancellationToken`.
- The write side (`ITaskRepository`) and the read side (`ITaskReadModel`) are separated; queries never call write methods.
- A short justification of "is CQRS needed here" is placed as a comment at the top of `Program.cs` or in a `README.md`: where CQRS pays off and where it would be overkill.
- The code compiles without warnings, `dotnet build` is green, `dotnet run` starts the server, and the tests are green.
- Use `InternalsVisibleTo` for the test assembly when handlers are `internal sealed`.

#### Pitfalls
- `IRequest` with no argument is `IRequest<Unit>`; a "void" command returns `Unit`, and its handler implements `IRequestHandler<TRequest, Unit>` and returns `Unit.Value`. Do not invent side workarounds for void commands.
- A pipeline behavior that is not registered in DI as `typeof(IPipelineBehavior<,>)` is **silently never invoked** — this is the most common mistake. Verify via logs: if `LoggingBehavior` does not write before the handler runs, registration is wrong.
- Execution order of behaviors equals registration order: the first registered behavior is the outermost in the chain. Logging is usually outer, validation inner, so both valid and invalid attempts are logged.
- `Validator.TryValidateObject` only checks attributes such as `[Required]` and `[StringLength]`; without attributes on the `record`, validation finds nothing and lets empty values through. Do not confuse this with FluentValidation — a separate library mentioned in the lesson as an alternative.
- Put `where TRequest : notnull` on `ValidationBehavior` — otherwise the compiler may emit a nullable warning and `Validator.TryValidateObject` requires a non-null object.
- For queries returning `null` (`TaskSummaryDto?`), handle the result in the endpoint via `pattern matching`: `dto is null ? Results.NotFound() : Results.Ok(dto)`.
- Never return the domain entity `TaskItem` from queries — it leaks internals and couples the API to implementation. Return DTOs only.
- A "god-handler" serving several `IRequest` types is impossible by MediatR contract (one `IRequest` — one `IRequestHandler`), but a common smell is to extract shared logic into one service method called from ten handlers. Prefer a behavior or a domain service with a clear responsibility.
- `CancellationToken` in ASP.NET Core: a bound `ct` parameter in minimal APIs is automatically cancelled when the client disconnects. If you do not propagate it to `Send` and to the repository, work continues after the client is gone — wasted resources.
- `AddMediatR` with `RegisterServicesFromAssemblyContaining<T>` scans one assembly; handlers and behaviors in another assembly are not picked up automatically — reference the right assembly or factor a shared extension method.
- Do not adopt CQRS "by default": for simple CRUD over a single table with no pain it is overkill. In this homework CQRS is justified by the educational goal and by genuinely different read/write shapes; in a real project you must justify it by load or complexity.

#### Acceptance criteria
- [ ] `TaskFlow.Api` targets `net8.0` and builds without errors or warnings.
- [ ] MediatR 12.* is installed and registered via `AddMediatR` + `RegisterServicesFromAssemblyContaining`.
- [ ] `CreateTaskCommand` and `ChangeTaskStatusCommand` are `sealed record` types implementing `IRequest<Guid>` and `IRequest` respectively.
- [ ] `GetTaskByIdQuery` and `ListTasksQuery` are `sealed record` types implementing `IRequest<TaskSummaryDto?>` and `IRequest<IReadOnlyList<TaskSummaryDto>>`.
- [ ] Each `IRequest` has exactly one `IRequestHandler`; no god-handler exists.
- [ ] Commands return the minimum (`Guid`/`Unit`); queries return DTOs, never the `TaskItem` domain entity.
- [ ] `LoggingBehavior` and `ValidationBehavior` implement `IPipelineBehavior<,>` and are registered as open generics in DI.
- [ ] `ValidationBehavior` actually rejects a command with an empty `Title` (`400`/`ValidationException`).
- [ ] `LoggingBehavior` writes the request type and the elapsed time to the log (visible in the `dotnet run` console).
- [ ] Every `IMediator.Send` and `Handle` propagates `CancellationToken`.
- [ ] Endpoints are thin: only input, `Send`, and response shaping; business logic lives in handlers and the domain.
- [ ] The write side (`ITaskRepository`) and the read side (`ITaskReadModel`) are separated; queries never call `AddAsync`/`UpdateAsync`.
- [ ] The status transition is implemented with `pattern matching`; an invalid transition throws `InvalidOperationException`.
- [ ] Unit tests cover: creation, status-transition rejection, and the "query never writes" contract.
- [ ] `Program.cs` or `README.md` contains a brief justification of where CQRS is warranted and where it would be overkill.

#### Hints (no direct answer)
- Recall from the lesson that MediatR resolves the handler by request type via DI — so registering the assembly correctly is enough.
- For a command with no return value, look at the lesson's distinction between `IRequest<T>` and `IRequest` (the latter is `IRequest<Unit>`).
- For the validation behavior, follow the lesson's template with `Validator.TryValidateObject` and `where TRequest : notnull`.
- To assert that queries never write, mock `ITaskRepository` in a test and verify `AddAsync` is never called.
- Behavior order equals registration order; decide whether logging or validation should run first.
- With `MapGroup` and minimal APIs a `CancellationToken ct` parameter is bound from the request automatically — just declare it in the signature.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — MediatR as a CQRS transport for the tasks module.
// Registration in Program.cs:
//   builder.Services.AddMediatR(cfg =>
//       cfg.RegisterServicesFromAssemblyContaining<CreateTaskHandler>());
//   builder.Services.AddScoped(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
//   builder.Services.AddScoped(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
//
// CQRS justification: the write and read sides differ in data shape
// (write: domain entity + transition rules; read: flat summary projections),
// so the split pays off. For plain CRUD over one table CQRS would be overkill.

using System.ComponentModel.DataAnnotations;
using MediatR;
using System.Diagnostics;

namespace TaskFlow.Api.Tasks;

// ── Domain and infrastructure (in-memory for a runnable sample) ──────
public enum TaskStatus { New, InProgress, Done, Cancelled }

public sealed record TaskItem(
    Guid Id, Guid AssigneeId, string Title, string Description,
    TaskStatus Status, DateTime CreatedAt);

public interface ITaskRepository
{
    Task AddAsync(TaskItem task, CancellationToken ct);
    Task UpdateAsync(TaskItem task, CancellationToken ct);
    Task<TaskItem?> GetByIdAsync(Guid id, CancellationToken ct);
}

public interface ITaskReadModel
{
    Task<TaskSummaryDto?> GetSummaryAsync(Guid id, CancellationToken ct);
    Task<IReadOnlyList<TaskSummaryDto>> ListAsync(TaskStatus? filter, int limit, CancellationToken ct);
}

public sealed record TaskSummaryDto(Guid Id, string Title, TaskStatus Status, DateTime CreatedAt);

// ── COMMANDS (write side) ────────────────────────────────────────────
public sealed record CreateTaskCommand(
    Guid AssigneeId,
    [Required] [StringLength(200, MinimumLength = 3)] string Title,
    [Required] string Description) : IRequest<Guid>;

internal sealed class CreateTaskHandler(ITaskRepository repo, ILogger<CreateTaskHandler> log)
    : IRequestHandler<CreateTaskCommand, Guid>
{
    public async Task<Guid> Handle(CreateTaskCommand cmd, CancellationToken ct)
    {
        var task = new TaskItem(
            Id: Guid.NewGuid(),
            AssigneeId: cmd.AssigneeId,
            Title: cmd.Title,
            Description: cmd.Description,
            Status: TaskStatus.New,
            CreatedAt: DateTime.UtcNow);

        await repo.AddAsync(task, ct);
        log.LogInformation("Task {TaskId} created for {Assignee}", task.Id, task.AssigneeId);
        return task.Id; // Command returns the minimum — just the identifier.
    }
}

public sealed record ChangeTaskStatusCommand(Guid TaskId, TaskStatus NewStatus) : IRequest;

internal sealed class ChangeTaskStatusHandler(ITaskRepository repo, ILogger<ChangeTaskStatusHandler> log)
    : IRequestHandler<ChangeTaskStatusCommand, Unit>
{
    public async Task<Unit> Handle(ChangeTaskStatusCommand cmd, CancellationToken ct)
    {
        var task = await repo.GetByIdAsync(cmd.TaskId, ct)
            ?? throw new InvalidOperationException("Task not found.");

        // Status transition via pattern matching — a domain business rule.
        task = (task.Status, cmd.NewStatus) switch
        {
            (TaskStatus.New, TaskStatus.InProgress) => task with { Status = TaskStatus.InProgress },
            (TaskStatus.InProgress, TaskStatus.Done) => task with { Status = TaskStatus.Done },
            (TaskStatus.New or TaskStatus.InProgress, TaskStatus.Cancelled) => task with { Status = TaskStatus.Cancelled },
            _ => throw new InvalidOperationException($"Invalid transition {task.Status} -> {cmd.NewStatus}")
        };

        await repo.UpdateAsync(task, ct);
        log.LogInformation("Task {TaskId} -> {Status}", task.Id, task.Status);
        return Unit.Value; // IRequest with no result returns Unit.
    }
}

// ── QUERIES (read side) ──────────────────────────────────────────────
public sealed record GetTaskByIdQuery(Guid TaskId) : IRequest<TaskSummaryDto?>;

internal sealed class GetTaskByIdHandler(ITaskReadModel readModel)
    : IRequestHandler<GetTaskByIdQuery, TaskSummaryDto?>
{
    public Task<TaskSummaryDto?> Handle(GetTaskByIdQuery q, CancellationToken ct)
        => readModel.GetSummaryAsync(q.TaskId, ct); // Read-only, DTO, not a domain entity.
}

public sealed record ListTasksQuery(TaskStatus? StatusFilter, int Limit = 50)
    : IRequest<IReadOnlyList<TaskSummaryDto>>;

internal sealed class ListTasksHandler(ITaskReadModel readModel)
    : IRequestHandler<ListTasksQuery, IReadOnlyList<TaskSummaryDto>>
{
    public Task<IReadOnlyList<TaskSummaryDto>> Handle(ListTasksQuery q, CancellationToken ct)
        => readModel.ListAsync(q.StatusFilter, Math.Min(q.Limit, 200), ct); // Read-side bound.
}

// ── PIPELINE BEHAVIORS ───────────────────────────────────────────────
public sealed class LoggingBehavior<TRequest, TResponse>(ILogger<LoggingBehavior<TRequest, TResponse>> log)
    : IPipelineBehavior<TRequest, TResponse> where TRequest : notnull
{
    public async Task<TResponse> Handle(TRequest req, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var sw = Stopwatch.StartNew();
        log.LogInformation("Handling {Request}", typeof(TRequest).Name);
        try { return await next(ct); }
        finally { log.LogInformation("Handled {Request} in {Ms} ms", typeof(TRequest).Name, sw.ElapsedMilliseconds); }
    }
}

public sealed class ValidationBehavior<TRequest, TResponse>(ILogger<ValidationBehavior<TRequest, TResponse>> log)
    : IPipelineBehavior<TRequest, TResponse> where TRequest : notnull
{
    public async Task<TResponse> Handle(TRequest req, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var ctx = new ValidationContext(req);
        var results = new List<ValidationResult>();
        if (!Validator.TryValidateObject(req, ctx, results, validateAllProperties: true))
        {
            var errors = string.Join("; ", results.Select(r => r.ErrorMessage));
            throw new ValidationException($"Validation failed for {typeof(TRequest).Name}: {errors}");
        }
        log.LogDebug("Validated {Request}", typeof(TRequest).Name);
        return await next(ct);
    }
}
```

Walk-through, line by line. `CreateTaskCommand` is an immutable `record` carrying validation attributes; it serves simultaneously as the contract, the transport, and the request DTO, exactly as the lesson recommends. `IRequest<Guid>` explicitly states that the command returns the minimum — an identifier. The handler is `internal sealed` with a primary constructor that receives dependencies (`ITaskRepository`, `ILogger`); MediatR instantiates it through DI. Inside `Handle`, the domain entity is constructed, persisted through the repository (write side), the event is logged, and the `Guid` is returned. No logic remains in the controller.

`ChangeTaskStatusCommand` is an `IRequest` with no argument, which is `IRequest<Unit>`; the handler returns `Unit.Value`. The transition is implemented with `pattern matching` over a `(current status, new status)` tuple — the lesson explicitly encourages pattern matching for business rules. An invalid transition throws `InvalidOperationException`, which is trivial to test. `task with { Status = ... }` leverages `record` semantics for non-destructive mutation.

The queries `GetTaskByIdQuery` and `ListTasksQuery` return the `TaskSummaryDto` DTO and `IReadOnlyList<...>` — never `TaskItem`. The handlers are thin: they delegate to `ITaskReadModel`, which physically reads from a projection rather than from the domain repository. This is exactly the model separation from the lesson. `Math.Min(q.Limit, 200)` protects the read side from overload — a read-side business rule.

`LoggingBehavior` and `ValidationBehavior` are `IPipelineBehavior<,>` with `where TRequest : notnull`. `LoggingBehavior` wraps `next(ct)` in a `try/finally` so the elapsed time is logged both on success and on exception. `ValidationBehavior` uses `Validator.TryValidateObject` with `validateAllProperties: true`, matching the lesson example; on failure it throws a `ValidationException` with a human-readable list of errors. Both are registered in DI as `typeof(IPipelineBehavior<,>)` — without that they would silently never run, a mistake the lesson explicitly calls out. Registration order (logging first, then validation) produces the chain "log → validation → handler". Every `Handle` and `next` propagates `CancellationToken`, as required by the lesson's best practices.

#### Going deeper (bonus)
1. Replace `DataAnnotations` with FluentValidation: add `FluentValidation.DependencyInjectionExtensions`, write `CreateTaskCommandValidator`, register validators by assembly, and rewrite `ValidationBehavior` on top of `IEnumerable<IValidator<TRequest>>`. Compare expressiveness.
2. Add a `CachingBehavior` for queries: cache `GetTaskByIdQuery` results in `IMemoryCache` for 30 seconds, invalidate on `ChangeTaskStatusCommand`. Think about how to tell a command from a query inside a generic behavior (a marker interface `IQuery`).
3. Introduce a domain event `TaskCreatedEvent : INotification` with a `TaskCreatedHandler` that logs a notification. Publish it via `IMediator.Publish` from `CreateTaskHandler` and compare with a direct service call.
4. Write an integration test for the pipeline: through `WebApplicationFactory`, send an invalid command and assert that `LoggingBehavior` ran BEFORE `ValidationBehavior` (by the order of log entries).

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект собирается без warning и запускается через `dotnet run`. / The project builds without warnings and runs via `dotnet run`.
- [ ] MediatR установлен и зарегистрирован по сборке. / MediatR is installed and registered by assembly.
- [ ] Минимум две команды и два запроса как `sealed record : IRequest`. / At least two commands and two queries as `sealed record : IRequest`.
- [ ] Каждый запрос имеет ровно один обработчик. / Each request has exactly one handler.
- [ ] Два `IPipelineBehavior` зарегистрированы как open generic. / Two `IPipelineBehavior` types registered as open generics.
- [ ] `CancellationToken` прокидывается во все `Send`/`Handle`. / `CancellationToken` is propagated to every `Send`/`Handle`.
- [ ] Контроллеры тонкие, без бизнес-логики. / Controllers are thin, with no business logic.
- [ ] Запросы возвращают DTO, не доменные сущности. / Queries return DTOs, not domain entities.
- [ ] Стороны записи и чтения разделены. / Write and read sides are separated.
- [ ] Unit-тесты зелёные, покрывают ключевые контракты. / Unit tests are green and cover the key contracts.
- [ ] Есть обоснование «нужен ли CQRS». / A "is CQRS needed" justification is present.

#### Ресурсы / Resources
- MediatR Wiki — https://github.com/jbogard/MediatR/wiki
- MediatR (NuGet) — https://www.nuget.org/packages/MediatR
- CQRS on Microsoft Learn — https://learn.microsoft.com/azure/architecture/patterns/cqrs
- Mediator pattern (Refactoring Guru) — https://refactoring.guru/design-patterns/mediator
- Minimal APIs in ASP.NET Core — https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis
- FluentValidation — https://docs.fluentvalidation.net/
- DataAnnotations validation — https://learn.microsoft.com/dotnet/api/system.componentmodel.dataannotations.validator
