[⬆ К модулю M16](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M16-L08: MediatR, CQRS (обзор) / MediatR, CQRS (overview)

**Модуль / Module:** M16
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

MediatR и CQRS — это две связанные идеи, которые меняют способ организации связей между компонентами приложения. Начнём с паттерна «Посредник» (Mediator).

**Паттерн Mediator.** Представьте шумный офис, где каждый сотрудник кричит каждому другому: бухгалтер — кассиру, кассир — кладовщику, кладовщик — бухгалтеру. Скоро наступает хаос. Появляется секретарь-диспетчер: все обращения идут через него, он сам разбирается, кого позвать. Компоненты перестают знать друг о друге напрямую — они знают только посредника. В коде это значит: классы контроллеров, сервисов и бизнес-логики не ссылаются друг на друга, а отправляют «запросы» посреднику, который находит нужного обработчика.

**MediatR** — популярная .NET-библиотека, реализующая этот паттерн. Её ядро — два интерфейса: `IRequest<TResponse>` (описывает запрос с результатом) и `IRequestHandler<TRequest, TResponse>` (обрабатывает запрос). Вы отправляете запрос через `IMediator.Send(...)`, а MediatR по типу запроса находит зарегистрированный обработчик через DI. Важно: MediatR сам по себе не распределяет запросы по разным моделям — он лишь связывает «отправителя» и «обработчика» через общий интерфейс. Преимущества: слабая связанность, лёгкое тестирование (мокаем `IMediator`), точечная кросс-функциональная логика (логирование, валидация) через конвейерные поведения (pipeline behaviors).

**CQRS (Command Query Responsibility Segregation).** Идея в раздельных моделях для записи и чтения. Классическая архитектура использует одни и те же сущности и репозитории для команд (изменяют состояние) и запросов (читают данные). CQRS предлагает: команды — отдельно, запросы — отдельно. Команды изменяют состояние, ничего не возвращают (или возвращают только идентификатор); запросы никогда не изменяют состояние и оптимизированы под чтение. Часто запросы читают напрямую из денормализованных projection-таблиц, материализованных представлений или read-model, минуя доменную модель — это даёт огромный прирост производительности на чтении.

MediatR отлично подходит как транспорт для CQRS: каждый `IRequest` становится либо командой (`CreateOrderCommand`), либо запросом (`GetOrderByIdQuery`). Обработчики команд содержат бизнес-логику и пишут в БД; обработчики запросов просто читают данные и возвращают DTO. Так приложение превращается в набор небольших, одиночных, хорошо типизированных операций (single-responsibility), которые легко тестировать и развивать независимо.

**Pipeline Behaviors** — механизм MediatR, похожий на middleware в ASP.NET Core. Каждый запрос проходит через цепочку поведений до и после обработчика. Это идеальное место для сквозной логики: логирование, валидация (FluentValidation), измерение производительности, авторизация, обработка исключений, кэширование. Поведение реализует `IPipelineBehavior<TRequest, TResponse>` и регистрируется в DI — MediatR сам соберёт их в цепочку. Один `ValidationBehavior` заменяет сотни одинаковых проверок в каждом контроллере.

**Когда применять CQRS?** Не всегда. CQRS добавляет сложность: больше типов, больше файлов, возможна eventual consistency между записью и чтением. Применяйте его, когда: чтение и запись имеют сильно разные нагрузку или форму данных; команда сложная (long-running, доменная); нужна масштабируемость (read-side кэшируется, реплицируется); сложные правила домена лучше выразить через команды. Для простого CRUD на одной таблице CQRS избыточен — обычный сервис-репозиторий проще и понятнее. Главное правило: начинайте без CQRS, вводите его там, где боль от связанности или скорости чтения становится реальной.

#### Theory (EN)

MediatR and CQRS are two related ideas that reshape how components in an application talk to each other. Let us start with the Mediator pattern.

**The Mediator pattern.** Picture a noisy office where every employee shouts at every other employee: the accountant to the cashier, the cashier to the storekeeper, the storekeeper back to the accountant. Chaos follows. Then a dispatcher appears: all requests go through the dispatcher, who decides whom to call. Components stop knowing each other directly — they only know the mediator. In code this means controllers, services, and domain logic no longer reference one another; instead, they send requests to a mediator that routes each request to the proper handler.

**MediatR** is a popular .NET library implementing this pattern. Its core consists of two interfaces: `IRequest<TResponse>` (describes a request with a result) and `IRequestHandler<TRequest, TResponse>` (handles the request). You send a request through `IMediator.Send(...)`, and MediatR resolves the registered handler by request type using dependency injection. Crucially, MediatR does not impose any particular architecture — it simply decouples the sender from the handler through a shared contract. Benefits include loose coupling, easy testing (you mock `IMediator`), and a clean place for cross-cutting concerns (logging, validation) through pipeline behaviors.

**CQRS (Command Query Responsibility Seggregation).** The idea is to separate the write model from the read model. Classic architectures use the same entities and repositories for commands (which mutate state) and queries (which read data). CQRS proposes: commands live on one side, queries on another. Commands change state and return little or nothing; queries never mutate state and are optimized for reading. Queries often read straight from denormalized projection tables, materialized views, or dedicated read models, bypassing the domain model entirely — this yields a dramatic performance boost on the read side.

MediatR works beautifully as the transport for CQRS: each `IRequest` becomes either a command (`CreateOrderCommand`) or a query (`GetOrderByIdQuery`). Command handlers contain business logic and write to the database; query handlers simply read data and return DTOs. The application becomes a collection of small, single-responsibility, strongly typed operations that are easy to test and evolve independently.

**Pipeline Behaviors** are MediatR's middleware-like mechanism. Each request flows through a chain of behaviors before and after the handler. This is the ideal home for cross-cutting logic: logging, validation (FluentValidation), performance measurement, authorization, exception handling, caching. A behavior implements `IPipelineBehavior<TRequest, TResponse>` and is registered in DI; MediatR wires the chain automatically. A single `ValidationBehavior` can replace hundreds of identical checks scattered across controllers.

**When should you apply CQRS?** Not always. CQRS adds complexity: more types, more files, and sometimes eventual consistency between writes and reads. Use it when reads and writes have very different load or shape; when commands are complex (long-running, domain-heavy); when scalability matters (the read side can be cached or replicated); or when complex domain rules are best expressed as commands. For simple CRUD over a single table, CQRS is overkill — a plain service-and-repository is simpler and clearer. The guiding rule: start without CQRS, introduce it only where the pain of coupling or read performance becomes real.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — MediatR as a transport for CQRS
// Setup: dotnet add package MediatR
// Register: builder.Services.AddMediatR(cfg =>
//     cfg.RegisterServicesFromAssemblyContaining<OrderCommandHandler>());

using MediatR;
using System.ComponentModel.DataAnnotations;

// ── COMMAND (write side) ──────────────────────────────────────────
// Команда изменяет состояние и возвращает только идентификатор.
// A command changes state and returns only an identifier.
public sealed record CreateOrderCommand(
    Guid CustomerId,
    IReadOnlyList<OrderLineDto> Lines) : IRequest<Guid>;

public sealed record OrderLineDto(Guid ProductId, int Quantity, decimal Price);

// Handler contains business logic + persistence.
// Обработчик содержит бизнес-логику и запись в БД.
internal sealed class CreateOrderHandler(IOrderRepository repo, ILogger<CreateOrderHandler> log)
    : IRequestHandler<CreateOrderCommand, Guid>
{
    public async Task<Guid> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        if (cmd.Lines.Count == 0)
            throw new ArgumentException("Order must have at least one line."); // Заказ пуст / Order is empty.

        var total = cmd.Lines.Sum(l => l.Quantity * l.Price);
        var order = new Order(Guid.NewGuid(), cmd.CustomerId, total, DateTime.UtcNow);

        await repo.AddAsync(order, ct);              // Запись в хранилище / Persist.
        log.LogInformation("Order {OrderId} created for {Customer}", order.Id, order.CustomerId);

        return order.Id;                              // Команда возвращает минимум / Return the minimum.
    }
}

// ── QUERY (read side) ─────────────────────────────────────────────
// Запрос ничего не меняет и оптимизирован под чтение: сразу возвращает DTO.
// A query mutates nothing and is read-optimized: returns a DTO directly.
public sealed record GetOrderByIdQuery(Guid OrderId) : IRequest<OrderSummaryDto?>;

public sealed record OrderSummaryDto(Guid Id, Guid CustomerId, decimal Total, DateTime CreatedAt);

internal sealed class GetOrderByIdHandler(IOrderReadModel readModel)
    : IRequestHandler<GetOrderByIdQuery, OrderSummaryDto?>
{
    public Task<OrderSummaryDto?> Handle(GetOrderByIdQuery q, CancellationToken ct)
        => readModel.GetSummaryAsync(q.OrderId, ct); // Читаем из read-model / Read from read-model.
}

// ── PIPELINE BEHAVIOR: validation via DataAnnotations ─────────────
// Конвейерное поведение: валидация через DataAnnotations.
public sealed class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private readonly ILogger<ValidationBehavior<TRequest, TResponse>> _log;
    public ValidationBehavior(ILogger<ValidationBehavior<TRequest, TResponse>> log) => _log = log;

    public async Task<TResponse> Handle(TRequest req, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var ctx = new ValidationContext(req);
        var results = new List<ValidationResult>();
        if (!Validator.TryValidateObject(req, ctx, results, validateAllProperties: true))
        {
            var errors = string.Join("; ", results.Select(r => r.ErrorMessage));
            throw new ValidationException($"Validation failed for {typeof(TRequest).Name}: {errors}");
        }

        _log.LogDebug("Validated {Request}", typeof(TRequest).Name);
        return await next(ct); // Передаём дальше по цепочке / Continue the chain.
    }
}

// ── API endpoint: thin controller, business logic lives in handlers ─
// Тонкий контроллер: вся логика — в обработчиках.
public static class OrdersApi
{
    public static void MapOrders(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/orders");

        // POST /orders  → command
        group.MapPost("/", async (CreateOrderCommand cmd, IMediator mediator, CancellationToken ct) =>
        {
            var id = await mediator.Send(cmd, ct);
            return Results.Created($"/orders/{id}", new { id });
        });

        // GET /orders/{id} → query
        group.MapGet("/{id:guid}", async (Guid id, IMediator mediator, CancellationToken ct) =>
        {
            var dto = await mediator.Send(new GetOrderByIdQuery(id), ct);
            return dto is null ? Results.NotFound() : Results.Ok(dto);
        });
    }
}

// Domain + infrastructure stubs (for context only).
// Домен и инфраструктура (только для контекста).
public sealed record Order(Guid Id, Guid CustomerId, decimal Total, DateTime CreatedAt);
public interface IOrderRepository { Task AddAsync(Order order, CancellationToken ct); }
public interface IOrderReadModel { Task<OrderSummaryDto?> GetSummaryAsync(Guid id, CancellationToken ct); }
```

#### Best Practices

- Держите команды и запросы в отдельных папках/проектах; имя команды оканчивайте на `Command`, запроса — на `Query`. / Keep commands and queries in separate folders or projects; suffix commands with `Command` and queries with `Query`.
- Команда должна возвращать минимальный результат (id или void), запрос — DTO, а не доменную сущность. / A command should return the minimum (id or void); a query should return a DTO, never a domain entity.
- Выносите сквозную логику в pipeline behaviors, а не дублируйте её в каждом обработчике. / Move cross-cutting logic into pipeline behaviors instead of duplicating it in every handler.
- Обработчики должны быть короткими и stateless; тяжёлую работу делегируйте домену и репозиториям. / Keep handlers short and stateless; delegate heavy work to the domain and repositories.
- Регистрируйте обработчики и поведения через `AddMediatR` по сборке, чтобы не пропустить новые типы. / Register handlers and behaviors via `AddMediatR` by assembly so new types are picked up automatically.
- Используйте cancellation tokens во всех вызовах `Send` и `Handle`. / Pass cancellation tokens through every `Send` and `Handle` call.

#### Частые ошибки / Common Mistakes

- Размещение бизнес-логики в контроллерах вместо обработчиков → делайте контроллеры тонкими, вся логика — в `IRequestHandler`. / Putting business logic in controllers instead of handlers → keep controllers thin; all logic lives in `IRequestHandler`.
- Возврат доменных сущностей из запросов → возвращайте отдельные DTO/read-models. / Returning domain entities from queries → return dedicated DTOs or read-models.
- Использование CQRS для простого CRUD без боли → начинайте без CQRS, вводите только при реальной необходимости. / Applying CQRS to simple CRUD with no pain → start without CQRS, adopt it only when genuinely needed.
- Забывают регистрировать pipeline behavior в DI → поведение просто не вызывается. / Forgetting to register a pipeline behavior in DI → the behavior is silently never invoked.
- Один «божественный» обработчик на несколько запросов → один запрос — один обработчик (single responsibility). / One god-handler serving several requests → one request, one handler (single responsibility).
- Игнорирование cancellation tokens → запросы не отменяются при обрыве соединения. / Ignoring cancellation tokens → requests keep running after the client disconnects.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я могу объяснить разницу между Mediator (паттерн) и CQRS (архитектурный стиль). / I can explain the difference between the Mediator (pattern) and CQRS (architectural style).
- [ ] Я создал команду и запрос как отдельные `IRequest` с понятными именами. / I created a command and a query as separate `IRequest` types with clear names.
- [ ] Мои обработчики реализуют `IRequestHandler<TRequest, TResponse>` и зарегистрированы через DI. / My handlers implement `IRequestHandler<TRequest, TResponse>` and are registered via DI.
- [ ] Запросы возвращают DTO, а не доменные сущности; команды возвращают минимум. / Queries return DTOs, not domain entities; commands return the minimum.
- [ ] Я добавил хотя бы один `IPipelineBehavior` (валидация или логирование). / I added at least one `IPipelineBehavior` (validation or logging).
- [ ] Я обосновал, нужен ли CQRS в моём сценарии, а не применил его «по умолчанию». / I justified whether CQRS is needed for my scenario rather than applying it by default.
- [ ] Все вызовы `Send` и `Handle` прокидывают `CancellationToken`. / Every `Send` and `Handle` propagates the `CancellationToken`.

#### Ресурсы / Resources

- MediatR Wiki — https://github.com/jbogard/MediatR/wiki
- MediatR (NuGet) — https://www.nuget.org/packages/MediatR
- CQRS на Microsoft Learn — https://learn.microsoft.com/azure/architecture/patterns/cqrs
- Mediator pattern (Refactoring Guru) — https://refactoring.guru/design-patterns/mediator
- Jimmy Bogard — MediatR pipeline behaviors — https://lostechies.com/jimmybogard/

---

[⬆ К модулю M16](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
