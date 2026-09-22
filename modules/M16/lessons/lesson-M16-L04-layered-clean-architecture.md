[⬆ К модулю M16](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M16-L04: Слоистая архитектура, clean architecture, ports & adapters / Layered architecture, clean architecture, ports & adapters

**Модуль / Module:** M16
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Архитектура приложения — это не про «красивые схемы», а про то, **куда направлены зависимости** и **где живёт бизнес-логика**. Рассмотрим три родственных подхода, которые эволюционируют один из другого: N-layer, Clean Architecture и Ports & Adapters (гексагональная).

**N-layer (многослойная архитектура).** Классика enterprise-разработки. Приложение делится на горизонтальные слои: Presentation (UI/API) → Business Logic → Data Access → Database. Каждый слой обращается только к слою, расположенному строго ниже. Аналогия — этажи здания: лифт едет сверху вниз, но не наоборот. Проблема: бизнес-логика часто «протекает» в контроллеры или в SQL-запросы, а Data Access слой знает слишком много о структуре БД. В итоге поменять базу данных — значит переписать половину приложения.

**Clean Architecture (Роберт Мартин, «Дядюшка Боб»).** Главная идея — **dependency rule**: зависимости всегда направлены внутрь, к центру. Слои (от внешних к внутренним):
- **Frameworks & Drivers** — ASP.NET Core, EF Core, БД, сторонние библиотеки.
- **Interface Adapters** — контроллеры, ViewModels, мапперы, репозитории-имплементации.
- **Use Cases (Application)** — конкретные сценарии приложения (CreateOrder, CancelOrder).
- **Entities (Enterprise)** — бизнес-правила самого верхнего уровня, независимы от фреймворка.

Ключевое правило: внутренний слой **не знает ничего** о внешнем. Entities не знают про EF Core. Use Cases не знают про HTTP. Внешние слои знают про внутренние. Это достигается через инверсию зависимостей (Dependency Inversion): внутренний слой объявляет интерфейс (порт), а внешний — предоставляет реализацию (адаптер). Аналогия: ядро приложения — это король в замке, к которому приходят гонцы (адаптеры), но сам король никуда не выходит.

**Ports & Adapters (гексагональная архитектура, Alistair Cockburn).** По сути, Clean Architecture, но с акцентом на изоляцию от «внешнего мира». Приложение помещается в центр (гексагон), а его границы — это **порты** (интерфейсы) и **адаптеры** (реализации). Порты бывают:
- **Driving (primary)** — как мир вызывает приложение: HTTP-контроллер, gRPC, CLI, тесты.
- **Driven (secondary)** — как приложение вызывает мир: репозиторий, SMTP-клиент, платёжный шлюз.

Гексагон можно «перевернуть» — заменить HTTP-адаптер на CLI, PostgreSQL-адаптер на MongoDB, а ядро не изменится ни на строку. Аналогия: USB-порт. Любое устройство, поддерживающее протокол USB, подключается и работает; компьютер не знает, флешка это или клавиатура.

**Dependency rule — фундамент всех трёх подходов.** В N-layer он слабый (только «сверху вниз»), в Clean и Hexagonal — строгий («всегда внутрь»). Именно направление стрелок зависимостей отличает хорошо спроектированную систему от спагетти-кода. В .NET это реализуется через: проекты-проекты (Project References) направлены внутрь; DI-контейнер собирает всё в Composition Root; интерфейсы лежат во внутренних проектах, реализации — во внешних.

**Когда что применять.** N-layer хорош для простых CRUD-приложений. Clean Architecture — для долгоживущих систем с меняющимися требованиями. Ports & Adapters — когда критична заменяемость интеграций (SaaS, мульти-тенант, разные БД в разных окружениях). Чрезмерная архитектура для простого CRUD — антипаттерн: не стрельбайте из пушки по воробьям.

#### Theory (EN)

Application architecture is not about «pretty diagrams»; it is about **where dependencies point** and **where business logic lives**. Let us examine three related approaches that evolve into one another: N-layer, Clean Architecture, and Ports & Adapters (hexagonal).

**N-layer (multi-tier architecture).** The enterprise classic. The application is split into horizontal layers: Presentation (UI/API) → Business Logic → Data Access → Database. Each layer talks only to the layer directly below it. The analogy is the floors of a building: the elevator goes top-down, never the other way. The problem is that business logic often «leaks» into controllers or into SQL queries, while the Data Access layer knows far too much about the database schema. As a result, swapping the database often means rewriting half the application.

**Clean Architecture (Robert C. Martin, «Uncle Bob»).** The core idea is the **dependency rule**: dependencies always point inward, toward the center. The layers, from outer to inner:
- **Frameworks & Drivers** — ASP.NET Core, EF Core, the database, third-party libraries.
- **Interface Adapters** — controllers, ViewModels, mappers, repository implementations.
- **Use Cases (Application)** — concrete application scenarios (CreateOrder, CancelOrder).
- **Entities (Enterprise)** — top-level business rules, independent of any framework.

The key rule: an inner layer **knows nothing** about an outer one. Entities do not know about EF Core. Use Cases do not know about HTTP. Outer layers know about inner ones. This is achieved through Dependency Inversion: an inner layer declares an interface (a port), and an outer layer provides the implementation (an adapter). The analogy: the application core is a king in a castle whom messengers (adapters) visit, but the king never leaves the castle himself.

**Ports & Adapters (hexagonal architecture, Alistair Cockburn).** Essentially Clean Architecture, but with an emphasis on isolating the application from «the outside world». The application lives in the center (the hexagon), and its boundaries are **ports** (interfaces) and **adapters** (implementations). Ports come in two flavors:
- **Driving (primary)** — how the world calls the application: an HTTP controller, gRPC, a CLI, tests.
- **Driven (secondary)** — how the application calls the world: a repository, an SMTP client, a payment gateway.

The hexagon can be «flipped» — replace the HTTP adapter with a CLI, or the PostgreSQL adapter with MongoDB, and the core does not change by a single line. The analogy is a USB port: any device that speaks the USB protocol plugs in and works; the computer neither knows nor cares whether it is a flash drive or a keyboard.

**The dependency rule is the foundation of all three approaches.** In N-layer it is weak (only «top-down»); in Clean and Hexagonal it is strict («always inward»). It is precisely the direction of the dependency arrows that separates a well-designed system from spaghetti code. In .NET this is realized through: project-to-project references pointing inward; the DI container wiring everything at the Composition Root; interfaces living in inner projects, implementations in outer ones.

**When to use which.** N-layer is fine for simple CRUD applications. Clean Architecture suits long-lived systems with changing requirements. Ports & Adapters shines when integration replaceability is critical (SaaS, multi-tenant, different databases per environment). Over-engineering a simple CRUD app is an anti-pattern: do not shoot sparrows with a cannon.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Clean Architecture + Ports & Adapters
// Структура проектов (зависимости направлены ВНУТРЬ):
//   Course.Domain        (Entities + Ports)        <- ничего не знает о внешнем мире
//   Course.Application   (Use Cases + Port interfaces)  <- ссылается на Domain
//   Course.Infrastructure(EF Core, SMTP — Adapters) <- ссылается на Application
//   Course.Api           (Controllers — Driving adapter) <- ссылается на Application
// Dependency rule: outer -> inner, never the reverse.

// === ВНУТРЕННИЙ СЛОЙ: Domain (Entities) ===
namespace Course.Domain.Entities;

// Сущность инкапсулирует бизнес-инварианты. Никаких атрибутов EF Core!
// The entity encapsulates business invariants. No EF Core attributes here!
public sealed class Order
{
    public Guid Id { get; private set; }
    public string CustomerEmail { get; private set; }
    public decimal TotalAmount { get; private set; }
    public OrderStatus Status { get; private set; }

    // Factory-метод гарантирует валидное состояние при создании.
    // A factory method guarantees a valid state on creation.
    public static Order Create(string customerEmail, decimal totalAmount)
    {
        if (string.IsNullOrWhiteSpace(customerEmail))
            throw new ArgumentException("Email обязателен / Email is required", nameof(customerEmail));
        if (totalAmount <= 0)
            throw new ArgumentOutOfRangeException(nameof(totalAmount), "Сумма > 0 / Amount must be positive");

        return new Order
        {
            Id = Guid.NewGuid(),
            CustomerEmail = customerEmail,
            TotalAmount = totalAmount,
            Status = OrderStatus.Pending
        };
    }

    public void Cancel()
    {
        // Бизнес-правило: отменять можно только Pending.
        // Business rule: only Pending orders may be cancelled.
        if (Status != OrderStatus.Pending)
            throw new InvalidOperationException("Нельзя отменить / Cannot cancel a non-pending order");
        Status = OrderStatus.Cancelled;
    }
}

public enum OrderStatus { Pending, Paid, Cancelled }

// === ВНУТРЕННИЙ СЛОЙ: Application — PORTS (интерфейсы) ===
// Порт — это контракт, объявленный ВНУТРИ. Реализация живёт снаружи.
// A port is a contract declared INSIDE. The implementation lives outside.
namespace Course.Application.Ports;

public interface IOrderRepository  // Driven port — как use case вызывает БД / how the use case calls the DB
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct);
    Task SaveAsync(Order order, CancellationToken ct);
}

public interface INotificationService  // Driven port — отправка уведомлений / sending notifications
{
    Task NotifyAsync(string email, string message, CancellationToken ct);
}

// === ВНУТРЕННИЙ СЛОЙ: Application — USE CASE ===
namespace Course.Application.UseCases;

public sealed record CancelOrderCommand(Guid OrderId);

public sealed class CancelOrderUseCase
{
    private readonly IOrderRepository _repository;
    private readonly INotificationService _notifications;

    // Use case зависит только от ПОРТОВ, не от инфраструктуры.
    // The use case depends only on PORTS, never on infrastructure.
    public CancelOrderUseCase(IOrderRepository repository, INotificationService notifications)
    {
        _repository = repository;
        _notifications = notifications;
    }

    public async Task HandleAsync(CancelOrderCommand command, CancellationToken ct)
    {
        var order = await _repository.GetByIdAsync(command.OrderId, ct)
            ?? throw new InvalidOperationException("Заказ не найден / Order not found");

        order.Cancel();  // бизнес-правило в сущности / business rule inside the entity

        await _repository.SaveAsync(order, ct);
        await _notifications.NotifyAsync(order.CustomerEmail, "Заказ отменён / Order cancelled", ct);
    }
}

// === ВНЕШНИЙ СЛОЙ: Infrastructure — ADAPTER (реализация порта через EF Core) ===
namespace Course.Infrastructure.Adapters;

using Course.Application.Ports;
using Course.Domain.Entities;
using Microsoft.EntityFrameworkCore;

public sealed class EfOrderRepository : IOrderRepository  // Driven adapter
{
    private readonly AppDbContext _db;

    public EfOrderRepository(AppDbContext db) => _db = db;

    public Task<Order?> GetByIdAsync(Guid id, CancellationToken ct) =>
        _db.Orders.AsNoTracking().FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task SaveAsync(Order order, CancellationToken ct)
    {
        _db.Orders.Update(order);
        await _db.SaveChangesAsync(ct);
    }
}

// === ВНЕШНИЙ СЛОЙ: API — DRIVING ADAPTER (ASP.NET Core) ===
namespace Course.Api.Controllers;

using Course.Application.UseCases;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/orders")]
public sealed class OrdersController : ControllerBase
{
    private readonly CancelOrderUseCase _cancelOrder;

    public OrdersController(CancelOrderUseCase cancelOrder) => _cancelOrder = cancelOrder;

    [HttpPost("{id:guid}/cancel")]
    public async Task<IActionResult> Cancel(Guid id, CancellationToken ct)
    {
        // Контроллер — тонкий адаптер: только переводит HTTP в use case.
        // The controller is a thin adapter: it only translates HTTP into a use case.
        await _cancelOrder.HandleAsync(new CancelOrderCommand(id), ct);
        return NoContent();
    }
}

// === Composition Root: связываем порты с адаптерами через DI ===
// builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();
// builder.Services.AddScoped<INotificationService, SmtpNotificationService>();
// builder.Services.AddScoped<CancelOrderUseCase>();
```

#### Best Practices
- Размещай интерфейсы (порты) во внутренних проектах, а реализации (адаптеры) — во внешних; это обеспечивает соблюдение правила зависимостей.
- Делай сущности богатыми (rich domain model): инкапсулируй инварианты и бизнес-правила внутри них, а не размазывай по сервисам.
- Используй Project References так, чтобы стрелки шли только внутрь; проверяй это на сборке — внутренний проект не должен ссылаться на `Microsoft.EntityFrameworkCore` или `Microsoft.AspNetCore.*`.
- Выноси сборку графа зависимостей в единую Composition Root (обычно `Program.cs` API-проекта), чтобы ядро не знало про DI-контейнер.
- Покрывай use cases и сущности юнит-тестами без БД и HTTP — если для теста нужен EF Core или `WebApplicationFactory`, архитектура нарушена.

- Place interfaces (ports) in inner projects and implementations (adapters) in outer ones; this enforces the dependency rule.
- Make entities rich: encapsulate invariants and business rules inside them rather than spreading them across services.
- Use Project References so the arrows only point inward; verify at build time — an inner project must not reference `Microsoft.EntityFrameworkCore` or `Microsoft.AspNetCore.*`.
- Centralize dependency graph wiring in a single Composition Root (usually the API project's `Program.cs`) so the core stays unaware of the DI container.
- Unit-test use cases and entities without a database or HTTP — if a test needs EF Core or `WebApplicationFactory`, the architecture is broken.

#### Частые ошибки / Common Mistakes
- Доменная сущность с атрибутами EF Core (`[Table]`, `[Column]`) → это связывает домен с инфраструктурой; держи домен чистым, маппи через EF Core `IEntityTypeConfiguration` в Infrastructure. (RU)
- Use case, который дёргает `HttpContext` или возвращает `IActionResult` → смешивает слои; возвращай DTO/record, а HTTP-трансляцию оставь контроллеру. (RU)
- Интерфейс репозитория, объявленный в Infrastructure, а в Application на него ссылаются → нарушено правило зависимостей; перенеси порт в Application/Domain. (RU)
- Использование Clean Architecture для простого CRUD из двух таблиц → избыточно; для CRUD хватит N-layer или вообще Vertical Slices. (RU)
- Тестирование use case через реальную БД → медленно и хрупко; подменяй порты (Moq/NSubstitute) и тестируй логику изолированно. (RU)

- A domain entity decorated with EF Core attributes (`[Table]`, `[Column]`) → this couples the domain to infrastructure; keep the domain pure and map via EF Core `IEntityTypeConfiguration` in Infrastructure. (EN)
- A use case that reaches into `HttpContext` or returns `IActionResult` → layers are blurred; return a DTO/record and leave HTTP translation to the controller. (EN)
- A repository interface declared in Infrastructure that Application then references → the dependency rule is broken; move the port into Application/Domain. (EN)
- Using Clean Architecture for a trivial two-table CRUD app → overkill; for CRUD, N-layer or even Vertical Slices is enough. (EN)
- Testing a use case against a real database → slow and brittle; substitute the ports (Moq/NSubstitute) and test the logic in isolation. (EN)

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Проекты Domain и Application не ссылаются на `Microsoft.EntityFrameworkCore` или `Microsoft.AspNetCore.*`.
- [ ] Все интерфейсы (порты), которые нужны use case, объявлены во внутреннем проекте, а не в Infrastructure.
- [ ] Сущности содержат бизнес-правила и инварианты (factory-методы, проверки в мутаторах), а не просто набор `{ get; set; }`.
- [ ] Контроллеры тонкие: только валидация входа, вызов use case и формирование HTTP-ответа.
- [ ] Use case можно покрыть юнит-тестами без поднятия БД и HTTP-сервера.
- [ ] Composition Root единственный, и он находится во внешнем (API) проекте.

- [ ] The Domain and Application projects do not reference `Microsoft.EntityFrameworkCore` or `Microsoft.AspNetCore.*`.
- [ ] Every interface (port) the use cases need is declared in an inner project, not in Infrastructure.
- [ ] Entities carry business rules and invariants (factory methods, validation in mutators) rather than being bags of `{ get; set; }`.
- [ ] Controllers are thin: only input validation, a use case call, and shaping the HTTP response.
- [ ] Use cases can be unit-tested without spinning up a database or an HTTP server.
- [ ] There is a single Composition Root, located in the outer (API) project.

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/](https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/)

---

[⬆ К модулю M16](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
