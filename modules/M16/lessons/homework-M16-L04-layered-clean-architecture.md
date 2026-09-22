---
[← К уроку M16-L04](lesson-M16-L04-layered-clean-architecture.md) | [⬆ К модулю M16](../README.md) | [Следующее ДЗ →](homework-M16-L05-repository-uow.md)
---

### Домашнее задание M16-L04: Слоистая архитектура, clean architecture, ports & adapters / Homework M16-L04: Layered architecture, clean architecture, ports & adapters

**Урок / Lesson:** M16-L04
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться проектировать .NET 8 приложение по правилам Clean Architecture и Ports & Adapters: разделить решение на проекты Domain / Application / Infrastructure / Api, направить зависимости строго внутрь, объявить порты во внутренних проектах, реализовать адаптеры во внешних и собрать граф зависимостей в едином Composition Root. (EN) Learn to design a .NET 8 application following Clean Architecture and Ports & Adapters: split the solution into Domain / Application / Infrastructure / Api projects, point dependencies strictly inward, declare ports in inner projects, implement adapters in outer ones, and wire the dependency graph in a single Composition Root.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит три родственных подхода — N-layer, Clean Architecture и Ports & Adapters — и формулирует фундаментальное правило зависимостей: стрелки всегда идут внутрь. Домашнее задание превращает эту теорию в работающее решение из четырёх проектов, где сущность инкапсулирует инварианты, use case зависит только от портов, а EF Core и SMTP живут снаружи как адаптеры. Вы на практике почувствуете, почему домен не должен ссылаться на `Microsoft.EntityFrameworkCore` и почему контроллер обязан быть тонким.
(EN) The lesson introduces three related approaches — N-layer, Clean Architecture, and Ports & Adapters — and states the foundational dependency rule: arrows always point inward. This homework turns that theory into a working four-project solution where the entity encapsulates invariants, the use case depends only on ports, and EF Core and SMTP live outside as adapters. You will feel in practice why the domain must not reference `Microsoft.EntityFrameworkCore` and why the controller must stay thin.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Стартап «BookWorm» запускает сервис продажи электронных книг. Бизнес-требования просты на словах, но коварны в деталях: покупатель оформляет заказ на одну или несколько книг, заказ проходит через жизненный цикл `Pending → Paid → Cancelled` (или `Refunded` после оплаты), а на каждом переходе система должна уведомлять покупателя по email и списывать средства через внешний платёжный шлюз. Команда уже обожглась на предыдущем проекте: доменная сущность была увешана атрибутами EF Core, контроллер содержал половину бизнес-логики, а смена SMTP-провайдера обернулась недельным рефакторингом. Чтобы не повторять ошибок, архитектор предлагает сразу выстроить решение по Clean Architecture с явными портами и адаптерами. Вам поручено реализовать ядро сервиса — доменную модель, два use case (`PayOrder` и `CancelOrder`) и минимальный набор адаптеров — так, чтобы направление зависимостей было безупречным, а ядро можно было покрыть юнит-тестами без базы данных и HTTP. Дополнительная нагрузка: продемонстрировать «переворачиваемость» гексагона, добавив второй driving-адаптер (CLI) рядом с HTTP, и второй driven-адаптер (fake платежный шлюз для тестов) рядом со Stripe-подобной реализацией. Это упражнение не про CRUD — оно про дисциплину стрелок зависимостей и осязаемую заменяемость интеграций, которые и составляют суть урока M16-L04.

#### Что нужно сделать (пошагово)
1. Создайте решение и четыре проекта с правильным направлением Project References. Выполните команды:
   ```
   dotnet new sln -n BookWorm
   dotnet new classlib -n BookWorm.Domain        -o src/BookWorm.Domain       -f net8.0
   dotnet new classlib -n BookWorm.Application   -o src/BookWorm.Application  -f net8.0
   dotnet new classlib -n BookWorm.Infrastructure -o src/BookWorm.Infrastructure -f net8.0
   dotnet new webapi   -n BookWorm.Api           -o src/BookWorm.Api          -f net8.0 --use-minimal-apis false
   dotnet sln add (Get-ChildItem -r *.csproj)
   dotnet add src/BookWorm.Application     reference src/BookWorm.Domain
   dotnet add src/BookWorm.Infrastructure  reference src/BookWorm.Application
   dotnet add src/BookWorm.Api             reference src/BookWorm.Application
   dotnet add src/BookWorm.Infrastructure  package Microsoft.EntityFrameworkCore.InMemory --version 8.*
   ```
   Внимание: `Infrastructure` ссылается на `Application`, а не наоборот; `Api` ссылается на `Application` и не трогает `Infrastructure` напрямую на уровне кода — связывание происходит только в Composition Root через `AddReference` или через `dotnet add src/BookWorm.Api reference src/BookWorm.Infrastructure` исключительно ради доступа к классу `CompositionRoot` (или перенесите регистрацию в `Api`).

2. В проекте `BookWorm.Domain` создайте сущность `Order` и перечисление `OrderStatus`. Сущность должна быть «богатой»: приватные сеттеры, factory-метод `Order.Create(...)`, методы `Pay()`, `Cancel()`, `Refund()`, инкапсулирующие инварианты (нельзя оплатить дважды, нельзя отменить оплаченный — только вернуть, нельзя вернуть неоплаченный). Никаких атрибутов EF Core в домене.

3. В проекте `BookWorm.Application` объявите порты (интерфейсы) и use cases. Порты: `IOrderRepository` (driven), `IPaymentGateway` (driven), `INotificationService` (driven), `IUnitOfWork` (driven). Use cases: `PayOrderUseCase`, `CancelOrderUseCase`. Каждый use case принимает только порты через конструктор и возвращает record-результат (`PayOrderResult`, `CancelOrderResult`). Запрещено: `HttpContext`, `IActionResult`, типы из `Microsoft.EntityFrameworkCore`.

4. В проекте `BookWorm.Infrastructure` реализуйте адаптеры: `EfOrderRepository` поверх `InMemory` базы, `StripePaymentGateway` (заглушка с логированием), `SmtpNotificationService` (заглушка), `UnitOfWork` оборачивающий `SaveChangesAsync`. Настройте маппинг сущности через `IEntityTypeConfiguration<Order>` в Infrastructure, а не через атрибуты.

5. В проекте `BookWorm.Api` создайте тонкие контроллеры `OrdersController` с маршрутами `POST api/orders` (создание), `POST api/orders/{id}/pay`, `POST api/orders/{id}/cancel`. Контроллер только переводит HTTP в use case и формирует ответ. В `Program.cs` (top-level statements) соберите Composition Root: зарегистрируйте `DbContext`, адаптеры, порты и use cases через `AddScoped`.

6. Добавьте второй driving-адаптер — CLI. Создайте в `Api` (или отдельном проекте `BookWorm.Cli`) консольную программу, которая читает аргументы командной строки (`pay <guid>`, `cancel <guid>`) и вызывает те же use cases. Это докажет, что ядро не привязано к HTTP.

7. Покройте `PayOrderUseCase` и `CancelOrderUseCase` юнит-тестами (xUnit + NSubstitute), подменяя порты. Тесты не должны требовать `WebApplicationFactory` или реальной базы. Проверьте позитивные и негативные сценарии: оплата уже оплаченного заказа, отмена не-`Pending`, успешный путь с вызовом `IPaymentGateway` и `INotificationService`.

8. Запустите `dotnet build` и убедитесь, что `BookWorm.Domain` и `BookWorm.Application` компилируются без ссылок на `Microsoft.EntityFrameworkCore` и `Microsoft.AspNetCore.*`. Проверьте это явно: `dotnet list src/BookWorm.Domain package` и `dotnet list src/BookWorm.Application package` не должны показывать EF Core или ASP.NET Core.

#### Требования к решению
Решение должно представлять собой собираемое решение .NET 8 из четырёх проектов со строгим правилом зависимостей: `Domain` ← `Application` ← `Infrastructure`, `Domain` ← `Application` ← `Api`, и `Api`/`Cli` могут ссылаться на `Infrastructure` только ради Composition Root. Внутренние проекты (`Domain`, `Application`) обязаны быть свободными от любых ссылок на `Microsoft.EntityFrameworkCore*`, `Microsoft.AspNetCore*` и `Microsoft.Extensions.DependencyInjection.Abstractions` (последний — спорный, но в строгой трактовке Clean Architecture use case не должен знать про DI; в нашем задании допустимо использовать `Abstractions` только для атрибута `[FromKeyedServices]`, если без него нельзя — но предпочтительно обойтись без него). Сущность `Order` должна быть rich domain model: приватные сеттеры, фабрика, методы-переходы с проверками инвариантов, никаких публичных мутаторов «по желанию». Use cases обязаны зависеть исключительно от интерфейсов-портов, объявленных в `Application` или `Domain`. Контроллеры и CLI-точка входа обязаны быть тонкими: валидация входа, вызов use case, формирование ответа — и ничего более. Composition Root должен быть единственным и находиться во внешнем проекте (`Api`/`Cli`). Юнит-тесты use cases обязаны проходить без поднятия базы данных и HTTP-сервера, подменяя все четыре порта. Код должен использовать возможности C# 12: top-level statements в `Program.cs`, pattern matching в переходах состояний, collection expressions там, где уместно, `record` для команд и результатов, `sealed` для сущностей и use cases, file-scoped namespaces, raw string literals в комментариях-документации, если они помогают.

#### Тонкости и подводные камни
Самая частая ошибка — разместить интерфейс `IOrderRepository` в `Infrastructure` «потому что он про EF Core». Это ломает правило зависимостей: use case в `Application` вынужден сослаться на `Infrastructure`, и стрелка идёт наружу. Порт обязан жить в `Application` (или `Domain`), а `EfOrderRepository` в `Infrastructure` его реализует. Вторая ловушка — атрибуты EF Core на сущности: `[Table("Orders")]`, `[Column("total")]`, конструктор без параметров для прокси-классов. Всё это связывает домен с инфраструктурой. Правильный путь — приватный конструктор для EF Core (если нужен) и `IEntityTypeConfiguration<Order>` в Infrastructure, а в домене только бизнес-поведение. Третья ловушка — use case, возвращающий `IActionResult` или тянущий `CancellationToken` из `HttpContext`. Use case должен принимать `CancellationToken` как параметр, а не доставать его из контекста запроса; HTTP-трансляция — задача контроллера. Четвёртая — контроллер, который сам вызывает `dbContext.SaveChangesAsync()` или маппит DTO вручную в одну простыню кода: это смешивание слоёв, верный признак того, что контроллер перестал быть «тонким адаптером». Пятая — регистрация use case как `Transient`, а репозитория как `Scoped`: в InMemory это не страшно, но в реальном EF Core `DbContext` зарегистрирован как `Scoped`, и `Transient` use case, захватывающий `Scoped` зависимость, может привести к рассинхрону изменений. Держите все компоненты запроса в одном scope. Шестая — использование Clean Architecture для тривиального CRUD из двух таблиц: это overengineering, но наше задание намеренно богаче CRUD (платёжный шлюз, уведомления, жизненный цикл), поэтому архитектура оправдана. Седьмая — тестирование use case через реальную InMemory базу: даже InMemory — это инфраструктура; подменяйте `IOrderRepository` стабом. Восьмая — забытый `IUnitOfWork`, из-за чего `SaveChangesAsync` вызывается в репозитории, а не координируется use case'ом; в строгой трактовке репозиторий только манипулирует `DbSet`, а коммит — отдельный порт.

#### Критерии приёмки
- [ ] Решение состоит из четырёх проектов: `Domain`, `Application`, `Infrastructure`, `Api` (+ опционально `Cli`).
- [ ] `dotnet build` проходит без ошибок и предупреждений, связанных с циклическими зависимостями.
- [ ] `Domain` и `Application` не ссылаются на `Microsoft.EntityFrameworkCore*` и `Microsoft.AspNetCore*` (проверяется `dotnet list package`).
- [ ] Все интерфейсы-порты (`IOrderRepository`, `IPaymentGateway`, `INotificationService`, `IUnitOfWork`) объявлены в `Application` или `Domain`.
- [ ] Сущность `Order` — rich domain model: приватные сеттеры, фабрика `Create`, методы `Pay`/`Cancel`/`Refund` с проверками инвариантов.
- [ ] На сущности `Order` нет атрибутов EF Core; маппинг вынесен в `IEntityTypeConfiguration<Order>` в `Infrastructure`.
- [ ] `PayOrderUseCase` и `CancelOrderUseCase` зависят только от портов и не знают про `HttpContext`, `IActionResult`, EF Core.
- [ ] Контроллеры тонкие: только валидация, вызов use case, формирование HTTP-ответа.
- [ ] Composition Root единственный и находится в `Api` (или `Cli`); `Program.cs` регистрирует адаптеры и порты.
- [ ] Юнит-тесты use cases проходят без БД и HTTP, подменяя все четыре порта через NSubstitute/Moq.
- [ ] Реализован второй driving-адаптер (CLI), вызывающий те же use cases, что и HTTP.
- [ ] Реализован второй driven-адаптер (fake payment gateway) для тестов, демонстрирующий заменяемость гексагона.
- [ ] Проверены негативные сценарии: двойная оплата, отмена не-Pending заказа, возврат неоплаченного.
- [ ] Код использует C# 12: top-level statements, pattern matching, `record`, `sealed`, file-scoped namespaces.
- [ ] `CancellationToken` пробрасывается во все порты и до EF Core, а не достаётся из `HttpContext`.

#### Подсказки (без прямого ответа)
- Вспомните аналогию урока про «короля в замке»: кто у вас король (ядро), а кто гонцы (адаптеры)? Если гонец знает дорогу к королю, а король не знает о гонце — стрелка направлена правильно.
- Переходы состояний удобно описать pattern matching'ом по `OrderStatus` — это и читаемо, и компилятор подскажет, если забыли ветку.
- Для `IEntityTypeConfiguration<Order>` в Infrastructure вспомните, что приватный конструктор для EF Core — допустимая уступка, но публичный API сущности остаётся бизнесовым.
- `IUnitOfWork` не обязан оборачивать транзакцию буквально — даже вызов `SaveChangesAsync` в одном месте уже повышает тестируемость.
- CLI-адаптер можно сделать в том же `Api` проекте, различая запуск по `args`, или вынести в отдельный проект со ссылкой на `Application` + `Infrastructure`.

#### Эталонное решение (разбор)
```csharp
// === BookWorm.Domain/Entities/Order.cs — ВНУТРЕННИЙ СЛОЙ, ничего не знает о внешнем мире ===
namespace BookWorm.Domain.Entities;

public enum OrderStatus { Pending, Paid, Cancelled, Refunded }

// sealed запрещает наследование — снижает риск случайного расширения инвариантов.
public sealed class Order
{
    public Guid Id { get; private set; }
    public string CustomerEmail { get; private set; } = "";
    public decimal TotalAmount { get; private set; }
    public OrderStatus Status { get; private set; }

    // EF Core нужен пустой конструктор — оставляем internal, чтобы не звали извне.
    // EF Core needs a parameterless ctor; keep it internal so callers don't use it.
    internal Order() { }

    // Фабрика гарантирует валидное состояние при создании — аналогия из урока.
    public static Order Create(string customerEmail, decimal totalAmount) =>
        new()
        {
            Id = Guid.NewGuid(),
            CustomerEmail = string.IsNullOrWhiteSpace(customerEmail)
                ? throw new ArgumentException("Email required", nameof(customerEmail))
                : customerEmail,
            TotalAmount = totalAmount <= 0
                ? throw new ArgumentOutOfRangeException(nameof(totalAmount), "Must be positive")
                : totalAmount,
            Status = OrderStatus.Pending
        };

    // Переходы состояний с проверкой инвариантов через pattern matching (C# 12).
    public void Pay() => Status = Status switch
    {
        OrderStatus.Pending => OrderStatus.Paid,
        _ => throw new InvalidOperationException($"Cannot pay order in {Status} state")
    };

    public void Cancel() => Status = Status switch
    {
        OrderStatus.Pending => OrderStatus.Cancelled,
        _ => throw new InvalidOperationException($"Cannot cancel order in {Status} state")
    };

    public void Refund() => Status = Status switch
    {
        OrderStatus.Paid => OrderStatus.Refunded,
        _ => throw new InvalidOperationException($"Cannot refund order in {Status} state")
    };
}

// === BookWorm.Application/Ports/Ports.cs — ПОРТЫ объявлены ВНУТРИ ===
namespace BookWorm.Application.Ports;

using BookWorm.Domain.Entities;

// Driven ports — как use case вызывает внешний мир.
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
}

public interface IPaymentGateway
{
    Task ChargeAsync(Guid orderId, decimal amount, CancellationToken ct);
}

public interface INotificationService
{
    Task NotifyAsync(string email, string message, CancellationToken ct);
}

public interface IUnitOfWork
{
    Task CommitAsync(CancellationToken ct);
}

// === BookWorm.Application/UseCases/PayOrderUseCase.cs ===
namespace BookWorm.Application.UseCases;

using BookWorm.Application.Ports;
using BookWorm.Domain.Entities;

public sealed record PayOrderCommand(Guid OrderId);
public sealed record PayOrderResult(bool Success, string? Error = null);

public sealed class PayOrderUseCase
{
    private readonly IOrderRepository _repo;
    private readonly IPaymentGateway _payments;
    private readonly INotificationService _notifications;
    private readonly IUnitOfWork _uow;

    public PayOrderUseCase(IOrderRepository repo, IPaymentGateway payments,
        INotificationService notifications, IUnitOfWork uow) =>
        (_repo, _payments, _notifications, _uow) = (repo, payments, notifications, uow);

    public async Task<PayOrderResult> HandleAsync(PayOrderCommand cmd, CancellationToken ct)
    {
        var order = await _repo.GetByIdAsync(cmd.OrderId, ct)
            ?? throw new InvalidOperationException("Order not found");

        // Сначала внешний эффект (платёж), потом изменение состояния, потом коммит.
        await _payments.ChargeAsync(order.Id, order.TotalAmount, ct);
        order.Pay();
        await _uow.CommitAsync(ct);
        await _notifications.NotifyAsync(order.CustomerEmail, "Order paid", ct);
        return new PayOrderResult(true);
    }
}

// === BookWorm.Infrastructure/Persistence/OrderConfiguration.cs — маппинг ВНЕ домена ===
namespace BookWorm.Infrastructure.Persistence;

using BookWorm.Domain.Entities;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> b)
    {
        b.ToTable("orders");
        b.HasKey(o => o.Id);
        b.Property(o => o.CustomerEmail).IsRequired().HasMaxLength(256);
        b.Property(o => o.TotalAmount).HasPrecision(18, 2);
        b.Property(o => o.Status).HasConversion<string>();
    }
}

// === BookWorm.Infrastructure/Persistence/EfOrderRepository.cs — DRIVEN ADAPTER ===
namespace BookWorm.Infrastructure.Persistence;

using BookWorm.Application.Ports;
using BookWorm.Domain.Entities;
using Microsoft.EntityFrameworkCore;

public sealed class EfOrderRepository(AppDbContext db) : IOrderRepository
{
    public Task<Order?> GetByIdAsync(Guid id, CancellationToken ct) =>
        db.Orders.AsNoTracking().FirstOrDefaultAsync(o => o.Id == id, ct);

    public Task AddAsync(Order order, CancellationToken ct)
    {
        db.Orders.Add(order);
        return Task.CompletedTask;
    }
}

// === BookWorm.Api/Program.cs — Composition Root (top-level statements) ===
using BookWorm.Application.Ports;
using BookWorm.Application.UseCases;
using BookWorm.Infrastructure.Persistence;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(o => o.UseInMemoryDatabase("bookworm"));
builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();
builder.Services.AddScoped<IPaymentGateway, StripePaymentGateway>();
builder.Services.AddScoped<INotificationService, SmtpNotificationService>();
builder.Services.AddScoped<IUnitOfWork, EfUnitOfWork>();
builder.Services.AddScoped<PayOrderUseCase>();
builder.Services.AddScoped<CancelOrderUseCase>();
builder.Services.AddControllers();

var app = builder.Build();
app.MapControllers();
app.Run();
```
Разбор по строкам. Сущность `Order` использует приватные сеттеры и `internal` конструктор без параметров — это уступка EF Core (ему нужен конструктор для материализации), но публичный API остаётся бизнесовым, как требует урок. Фабрика `Create` проверяет предусловия и задаёт начальный статус `Pending`, повторяя пример из урока. Переходы `Pay`/`Cancel`/`Refund` реализованы через `switch` выражение с pattern matching — компилятор гарантирует полноту, а каждая ветка явно выбрасывает исключение при нарушении инварианта, что соответствует принципу «богатой доменной модели». Порты `IOrderRepository`, `IPaymentGateway`, `INotificationService`, `IUnitOfWork` объявлены в `Application` — это и есть правило зависимостей из урока: контракт внутри, реализация снаружи. `PayOrderUseCase` зависит только от четырёх интерфейсов и не знает ни про `HttpContext`, ни про EF Core; он принимает `CancellationToken` как параметр, а не достаёт из контекста запроса — частая ошибка из урока предотвращена. Порядок операций в `HandleAsync` (платёж → изменение состояния → коммит → уведомление) сознательный: если платёж упадёт, домен не изменится. `OrderConfiguration` в Infrastructure через `IEntityTypeConfiguration<Order>` — прямая реализация рекомендации урока не вешать атрибуты на домен. `EfOrderRepository` использует `AsNoTracking` для чтения и не вызывает `SaveChangesAsync` — коммит делегирован `IUnitOfWork`, что повышает тестируемость и координирует несколько репозиториев в одной транзакции. Composition Root в `Program.cs` единственный: именно здесь порты связываются с адаптерами, и именно поэтому ядро не знает про DI-контейнер. CLI-адаптер (не показан в коде) вызывал бы те же use cases, доказывая «переворачиваемость» гексагона, а fake payment gateway в тестах доказывает заменяемость driven-стороны.

#### Задания на углубление (бонус)
1. Добавьте use case `RefundOrderUseCase` с собственным портом `IRefundService` и продемонстрируйте, что возврат возможен только из статуса `Paid`. Покройте тестами через подмену порта.
2. Реализуйте второй driven-адаптер `MongoOrderRepository` и переключите Composition Root на него одной строкой, не меняя ни одного use case. Это упражнение на «переворачиваемость» гексагона.
3. Добавьте доменное событие `OrderPaidEvent` (через `MediatR` или свой простой диспатчер во внутреннем проекте) и покажите, что уведомление можно убрать из use case в обработчик события, не нарушая правило зависимостей.
4. Напишите архитектурный тест (NetArchTest или вручную через рефлексию), который падает, если `Domain` или `Application` ссылается на `Microsoft.EntityFrameworkCore` или `Microsoft.AspNetCore.*`. Это превращает правило зависимостей из договорённости в исполняемое правило.

---

## Statement in English / Постановка на английском

#### Context & motivation
The startup «BookWorm» is launching an e-book store. The business rules sound simple but are treacherous in detail: a customer places an order for one or more books, the order flows through a lifecycle `Pending → Paid → Cancelled` (or `Refunded` after payment), and every transition must notify the buyer by email and charge money through an external payment gateway. The team has already been burned on a previous project: the domain entity was covered in EF Core attributes, the controller contained half the business logic, and swapping the SMTP provider turned into a week-long refactor. To avoid repeating those mistakes, the architect proposes building the solution with Clean Architecture and explicit ports and adapters from day one. You are tasked with implementing the core of the service — the domain model, two use cases (`PayOrder` and `CancelOrder`), and the minimal set of adapters — so that the dependency direction is impeccable and the core is unit-testable without a database or HTTP. The extra load: demonstrate the «flippability» of the hexagon by adding a second driving adapter (a CLI) alongside HTTP, and a second driven adapter (a fake payment gateway for tests) alongside a Stripe-like implementation. This exercise is not about CRUD — it is about the discipline of dependency arrows and the tangible replaceability of integrations that constitute the heart of lesson M16-L04.

#### What to do step by step
1. Create the solution and four projects with the correct direction of Project References. Run the commands:
   ```
   dotnet new sln -n BookWorm
   dotnet new classlib -n BookWorm.Domain        -o src/BookWorm.Domain       -f net8.0
   dotnet new classlib -n BookWorm.Application   -o src/BookWorm.Application  -f net8.0
   dotnet new classlib -n BookWorm.Infrastructure -o src/BookWorm.Infrastructure -f net8.0
   dotnet new webapi   -n BookWorm.Api           -o src/BookWorm.Api          -f net8.0 --use-minimal-apis false
   dotnet sln add (Get-ChildItem -r *.csproj)
   dotnet add src/BookWorm.Application     reference src/BookWorm.Domain
   dotnet add src/BookWorm.Infrastructure  reference src/BookWorm.Application
   dotnet add src/BookWorm.Api             reference src/BookWorm.Application
   dotnet add src/BookWorm.Infrastructure  package Microsoft.EntityFrameworkCore.InMemory --version 8.*
   ```
   Note that `Infrastructure` references `Application`, never the reverse; `Api` references `Application` and does not touch `Infrastructure` at the code level — the wiring happens only in the Composition Root.

2. In the `BookWorm.Domain` project, create the `Order` entity and the `OrderStatus` enum. The entity must be «rich»: private setters, a factory method `Order.Create(...)`, methods `Pay()`, `Cancel()`, `Refund()` that encapsulate invariants (no double payment, no cancellation of a paid order — only a refund, no refund of an unpaid order). No EF Core attributes in the domain.

3. In `BookWorm.Application`, declare the ports (interfaces) and the use cases. Ports: `IOrderRepository` (driven), `IPaymentGateway` (driven), `INotificationService` (driven), `IUnitOfWork` (driven). Use cases: `PayOrderUseCase`, `CancelOrderUseCase`. Each use case takes only ports through its constructor and returns a record result (`PayOrderResult`, `CancelOrderResult`). Forbidden: `HttpContext`, `IActionResult`, types from `Microsoft.EntityFrameworkCore`.

4. In `BookWorm.Infrastructure`, implement the adapters: `EfOrderRepository` on top of an `InMemory` database, `StripePaymentGateway` (a logging stub), `SmtpNotificationService` (a stub), and a `UnitOfWork` wrapping `SaveChangesAsync`. Configure entity mapping through `IEntityTypeConfiguration<Order>` in Infrastructure, not through attributes.

5. In `BookWorm.Api`, create thin controllers `OrdersController` with routes `POST api/orders` (creation), `POST api/orders/{id}/pay`, `POST api/orders/{id}/cancel`. The controller only translates HTTP into a use case and shapes the response. In `Program.cs` (top-level statements), assemble the Composition Root: register the `DbContext`, the adapters, the ports, and the use cases through `AddScoped`.

6. Add a second driving adapter — a CLI. Create a console program (in `Api` or a separate `BookWorm.Cli` project) that reads command-line arguments (`pay <guid>`, `cancel <guid>`) and calls the same use cases. This proves the core is not bound to HTTP.

7. Cover `PayOrderUseCase` and `CancelOrderUseCase` with unit tests (xUnit + NSubstitute) by substituting the ports. The tests must not require `WebApplicationFactory` or a real database. Check positive and negative scenarios: paying an already-paid order, cancelling a non-`Pending` order, and the happy path that calls `IPaymentGateway` and `INotificationService`.

8. Run `dotnet build` and make sure `BookWorm.Domain` and `BookWorm.Application` compile without references to `Microsoft.EntityFrameworkCore` and `Microsoft.AspNetCore.*`. Verify explicitly: `dotnet list src/BookWorm.Domain package` and `dotnet list src/BookWorm.Application package` must not show EF Core or ASP.NET Core.

#### Requirements
The solution must be a buildable .NET 8 solution of four projects with a strict dependency rule: `Domain` ← `Application` ← `Infrastructure`, `Domain` ← `Application` ← `Api`, and `Api`/`Cli` may reference `Infrastructure` only for the Composition Root. The inner projects (`Domain`, `Application`) must be free of any references to `Microsoft.EntityFrameworkCore*`, `Microsoft.AspNetCore*`, and, in a strict reading of Clean Architecture, `Microsoft.Extensions.DependencyInjection.Abstractions` (the latter is debatable — a use case should not know about DI; in this assignment it is acceptable to use `Abstractions` only for the `[FromKeyedServices]` attribute if there is no way around it, but it is preferable to do without it). The `Order` entity must be a rich domain model: private setters, a factory, transition methods with invariant checks, no public «at will» mutators. Use cases must depend exclusively on port interfaces declared in `Application` or `Domain`. Controllers and the CLI entry point must be thin: input validation, a use case call, response shaping — nothing more. The Composition Root must be unique and live in an outer project (`Api`/`Cli`). Unit tests of use cases must pass without spinning up a database or an HTTP server, substituting all four ports. The code must use C# 12 features: top-level statements in `Program.cs`, pattern matching in state transitions, collection expressions where appropriate, `record` for commands and results, `sealed` for entities and use cases, file-scoped namespaces, raw string literals in documentation comments where they help.

#### Pitfalls
The most common mistake is to place the `IOrderRepository` interface in `Infrastructure` «because it is about EF Core». This breaks the dependency rule: the use case in `Application` is forced to reference `Infrastructure`, and the arrow points outward. The port must live in `Application` (or `Domain`), and `EfOrderRepository` in `Infrastructure` implements it. The second trap is EF Core attributes on the entity: `[Table("Orders")]`, `[Column("total")]`, a parameterless constructor for proxy classes. All of this couples the domain to infrastructure. The right path is a private (or internal) constructor for EF Core if needed and an `IEntityTypeConfiguration<Order>` in Infrastructure, while the domain carries only business behavior. The third trap is a use case that returns `IActionResult` or pulls a `CancellationToken` out of `HttpContext`. The use case must take a `CancellationToken` as a parameter rather than fishing it out of the request context; HTTP translation is the controller's job. The fourth trap is a controller that calls `dbContext.SaveChangesAsync()` itself or maps DTOs in a giant wall of code: that is layer mixing, a clear sign the controller is no longer a «thin adapter». The fifth trap is registering the use case as `Transient` and the repository as `Scoped`: harmless with InMemory, but in real EF Core the `DbContext` is `Scoped`, and a `Transient` use case capturing a `Scoped` dependency can desynchronize changes. Keep all components of a request in one scope. The sixth trap is using Clean Architecture for a trivial two-table CRUD: that is overengineering, but this assignment is intentionally richer than CRUD (payment gateway, notifications, a lifecycle), so the architecture is justified. The seventh trap is testing a use case against a real InMemory database: even InMemory is infrastructure; substitute `IOrderRepository` with a stub. The eighth trap is a forgotten `IUnitOfWork`, so `SaveChangesAsync` is called inside the repository instead of being coordinated by the use case; in a strict reading the repository only manipulates a `DbSet`, and the commit is a separate port.

#### Acceptance criteria
- [ ] The solution consists of four projects: `Domain`, `Application`, `Infrastructure`, `Api` (+ optionally `Cli`).
- [ ] `dotnet build` passes without errors or warnings related to circular dependencies.
- [ ] `Domain` and `Application` do not reference `Microsoft.EntityFrameworkCore*` or `Microsoft.AspNetCore*` (verified via `dotnet list package`).
- [ ] All port interfaces (`IOrderRepository`, `IPaymentGateway`, `INotificationService`, `IUnitOfWork`) are declared in `Application` or `Domain`.
- [ ] The `Order` entity is a rich domain model: private setters, a `Create` factory, `Pay`/`Cancel`/`Refund` methods with invariant checks.
- [ ] The `Order` entity has no EF Core attributes; mapping is moved to `IEntityTypeConfiguration<Order>` in `Infrastructure`.
- [ ] `PayOrderUseCase` and `CancelOrderUseCase` depend only on ports and know nothing about `HttpContext`, `IActionResult`, or EF Core.
- [ ] Controllers are thin: only validation, a use case call, and shaping the HTTP response.
- [ ] The Composition Root is unique and lives in `Api` (or `Cli`); `Program.cs` registers the adapters and ports.
- [ ] Unit tests of use cases pass without a database or HTTP, substituting all four ports via NSubstitute/Moq.
- [ ] A second driving adapter (CLI) is implemented, calling the same use cases as HTTP.
- [ ] A second driven adapter (fake payment gateway) is implemented for tests, demonstrating the replaceability of the hexagon.
- [ ] Negative scenarios are covered: double payment, cancellation of a non-`Pending` order, refund of an unpaid order.
- [ ] The code uses C# 12: top-level statements, pattern matching, `record`, `sealed`, file-scoped namespaces.
- [ ] `CancellationToken` is threaded through all ports down to EF Core, not pulled from `HttpContext`.

#### Hints (no direct answer)
- Recall the lesson's «king in a castle» analogy: who is the king (the core) and who are the messengers (the adapters)? If a messenger knows the way to the king but the king knows nothing of the messenger, the arrow points correctly.
- State transitions are convenient to describe with pattern matching on `OrderStatus` — readable, and the compiler will warn if a branch is missing.
- For `IEntityTypeConfiguration<Order>` in Infrastructure, recall that a private constructor for EF Core is an acceptable concession, but the entity's public API stays business-focused.
- `IUnitOfWork` does not have to literally wrap a transaction — even a single `SaveChangesAsync` call in one place already improves testability.
- The CLI adapter can live in the same `Api` project, distinguished by `args`, or in a separate project referencing `Application` + `Infrastructure`.

#### Reference solution walk-through
```csharp
// === BookWorm.Domain/Entities/Order.cs — INNER LAYER, knows nothing of the outside world ===
namespace BookWorm.Domain.Entities;

public enum OrderStatus { Pending, Paid, Cancelled, Refunded }

// sealed prevents inheritance — reduces the risk of accidentally widening invariants.
public sealed class Order
{
    public Guid Id { get; private set; }
    public string CustomerEmail { get; private set; } = "";
    public decimal TotalAmount { get; private set; }
    public OrderStatus Status { get; private set; }

    // EF Core needs a parameterless ctor; keep it internal so callers don't use it.
    internal Order() { }

    // The factory guarantees a valid state on creation — analogy from the lesson.
    public static Order Create(string customerEmail, decimal totalAmount) =>
        new()
        {
            Id = Guid.NewGuid(),
            CustomerEmail = string.IsNullOrWhiteSpace(customerEmail)
                ? throw new ArgumentException("Email required", nameof(customerEmail))
                : customerEmail,
            TotalAmount = totalAmount <= 0
                ? throw new ArgumentOutOfRangeException(nameof(totalAmount), "Must be positive")
                : totalAmount,
            Status = OrderStatus.Pending
        };

    // State transitions with invariant checks via pattern matching (C# 12).
    public void Pay() => Status = Status switch
    {
        OrderStatus.Pending => OrderStatus.Paid,
        _ => throw new InvalidOperationException($"Cannot pay order in {Status} state")
    };

    public void Cancel() => Status = Status switch
    {
        OrderStatus.Pending => OrderStatus.Cancelled,
        _ => throw new InvalidOperationException($"Cannot cancel order in {Status} state")
    };

    public void Refund() => Status = Status switch
    {
        OrderStatus.Paid => OrderStatus.Refunded,
        _ => throw new InvalidOperationException($"Cannot refund order in {Status} state")
    };
}

// === BookWorm.Application/Ports/Ports.cs — PORTS declared INSIDE ===
namespace BookWorm.Application.Ports;

using BookWorm.Domain.Entities;

// Driven ports — how the use case calls the outside world.
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
}

public interface IPaymentGateway
{
    Task ChargeAsync(Guid orderId, decimal amount, CancellationToken ct);
}

public interface INotificationService
{
    Task NotifyAsync(string email, string message, CancellationToken ct);
}

public interface IUnitOfWork
{
    Task CommitAsync(CancellationToken ct);
}

// === BookWorm.Application/UseCases/PayOrderUseCase.cs ===
namespace BookWorm.Application.UseCases;

using BookWorm.Application.Ports;
using BookWorm.Domain.Entities;

public sealed record PayOrderCommand(Guid OrderId);
public sealed record PayOrderResult(bool Success, string? Error = null);

public sealed class PayOrderUseCase
{
    private readonly IOrderRepository _repo;
    private readonly IPaymentGateway _payments;
    private readonly INotificationService _notifications;
    private readonly IUnitOfWork _uow;

    public PayOrderUseCase(IOrderRepository repo, IPaymentGateway payments,
        INotificationService notifications, IUnitOfWork uow) =>
        (_repo, _payments, _notifications, _uow) = (repo, payments, notifications, uow);

    public async Task<PayOrderResult> HandleAsync(PayOrderCommand cmd, CancellationToken ct)
    {
        var order = await _repo.GetByIdAsync(cmd.OrderId, ct)
            ?? throw new InvalidOperationException("Order not found");

        // External effect first, then state change, then commit.
        await _payments.ChargeAsync(order.Id, order.TotalAmount, ct);
        order.Pay();
        await _uow.CommitAsync(ct);
        await _notifications.NotifyAsync(order.CustomerEmail, "Order paid", ct);
        return new PayOrderResult(true);
    }
}

// === BookWorm.Infrastructure/Persistence/OrderConfiguration.cs — mapping OUTSIDE the domain ===
namespace BookWorm.Infrastructure.Persistence;

using BookWorm.Domain.Entities;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> b)
    {
        b.ToTable("orders");
        b.HasKey(o => o.Id);
        b.Property(o => o.CustomerEmail).IsRequired().HasMaxLength(256);
        b.Property(o => o.TotalAmount).HasPrecision(18, 2);
        b.Property(o => o.Status).HasConversion<string>();
    }
}

// === BookWorm.Infrastructure/Persistence/EfOrderRepository.cs — DRIVEN ADAPTER ===
namespace BookWorm.Infrastructure.Persistence;

using BookWorm.Application.Ports;
using BookWorm.Domain.Entities;
using Microsoft.EntityFrameworkCore;

public sealed class EfOrderRepository(AppDbContext db) : IOrderRepository
{
    public Task<Order?> GetByIdAsync(Guid id, CancellationToken ct) =>
        db.Orders.AsNoTracking().FirstOrDefaultAsync(o => o.Id == id, ct);

    public Task AddAsync(Order order, CancellationToken ct)
    {
        db.Orders.Add(order);
        return Task.CompletedTask;
    }
}

// === BookWorm.Api/Program.cs — Composition Root (top-level statements) ===
using BookWorm.Application.Ports;
using BookWorm.Application.UseCases;
using BookWorm.Infrastructure.Persistence;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(o => o.UseInMemoryDatabase("bookworm"));
builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();
builder.Services.AddScoped<IPaymentGateway, StripePaymentGateway>();
builder.Services.AddScoped<INotificationService, SmtpNotificationService>();
builder.Services.AddScoped<IUnitOfWork, EfUnitOfWork>();
builder.Services.AddScoped<PayOrderUseCase>();
builder.Services.AddScoped<CancelOrderUseCase>();
builder.Services.AddControllers();

var app = builder.CreateBuilder(args);
app.MapControllers();
app.Run();
```
Line-by-line walk-through. The `Order` entity uses private setters and an `internal` parameterless constructor — a concession to EF Core (it needs a constructor for materialization), but the public API stays business-focused, as the lesson requires. The `Create` factory validates preconditions and sets the initial `Pending` status, mirroring the lesson's example. The `Pay`/`Cancel`/`Refund` transitions are implemented with `switch` expressions and pattern matching — the compiler guarantees exhaustiveness, and each arm explicitly throws when an invariant is violated, which matches the «rich domain model» principle. The ports `IOrderRepository`, `IPaymentGateway`, `INotificationService`, `IUnitOfWork` are declared in `Application` — this is exactly the dependency rule from the lesson: the contract inside, the implementation outside. `PayOrderUseCase` depends only on four interfaces and knows nothing about `HttpContext` or EF Core; it takes a `CancellationToken` as a parameter rather than fishing it out of the request context — a frequent lesson mistake, prevented here. The order of operations in `HandleAsync` (payment → state change → commit → notification) is deliberate: if the payment fails, the domain does not change. `OrderConfiguration` in Infrastructure via `IEntityTypeConfiguration<Order>` is a direct implementation of the lesson's recommendation not to decorate the domain with attributes. `EfOrderRepository` uses `AsNoTracking` for reads and does not call `SaveChangesAsync` — the commit is delegated to `IUnitOfWork`, which improves testability and coordinates several repositories in one transaction. The Composition Root in `Program.cs` is unique: this is where ports are bound to adapters, and that is exactly why the core does not know about the DI container. The CLI adapter (not shown in the code) would call the same use cases, proving the «flippability» of the hexagon, and a fake payment gateway in tests proves the replaceability of the driven side.

#### Going deeper (bonus)
1. Add a `RefundOrderUseCase` with its own `IRefundService` port and demonstrate that a refund is only possible from the `Paid` state. Cover it with tests via a substituted port.
2. Implement a second driven adapter, `MongoOrderRepository`, and switch the Composition Root to it with a single line, without touching a single use case. This is an exercise in the «flippability» of the hexagon.
3. Add a domain event `OrderPaidEvent` (via `MediatR` or a simple in-process dispatcher in the inner project) and show that the notification can be moved out of the use case into an event handler without breaking the dependency rule.
4. Write an architecture test (NetArchTest or a hand-rolled reflection check) that fails if `Domain` or `Application` references `Microsoft.EntityFrameworkCore` or `Microsoft.AspNetCore.*`. This turns the dependency rule from a gentlemen's agreement into an executable rule.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Решение из 4+ проектов собирается через `dotnet build`.
- [ ] `Domain` и `Application` не ссылаются на EF Core / ASP.NET Core.
- [ ] Все порты во внутренних проектах, все адаптеры во внешних.
- [ ] Сущность `Order` — rich domain model без атрибутов EF Core.
- [ ] Use cases зависят только от портов, не знают `HttpContext`/`IActionResult`.
- [ ] Контроллеры тонкие; Composition Root единственный.
- [ ] Юнит-тесты use cases без БД и HTTP, через подмену портов.
- [ ] Второй driving-адаптер (CLI) и второй driven-адаптер (fake payment).
- [ ] Негативные сценарии покрыты тестами.
- [ ] Использованы возможности C# 12 (top-level, pattern matching, records, sealed).
- [ ] Solution consists of 4+ projects and builds with `dotnet build`.
- [ ] `Domain` and `Application` do not reference EF Core / ASP.NET Core.
- [ ] All ports are in inner projects, all adapters in outer projects.
- [ ] The `Order` entity is a rich domain model without EF Core attributes.
- [ ] Use cases depend only on ports and know nothing of `HttpContext`/`IActionResult`.
- [ ] Controllers are thin; the Composition Root is unique.
- [ ] Use case unit tests run without DB or HTTP, with ports substituted.
- [ ] A second driving adapter (CLI) and a second driven adapter (fake payment) exist.
- [ ] Negative scenarios are covered by tests.
- [ ] C# 12 features are used (top-level, pattern matching, records, sealed).

#### Ресурсы / Resources
- [Microsoft Learn — Microservices DDD/CQRS patterns](https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/)
- [Clean Architecture — Robert C. Martin (The Clean Architecture blog post)](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Hexagonal Architecture — Alistair Cockburn](https://alistair.cockburn.us/hexagonal-architecture/)
- [Microsoft Learn — Dependency injection in .NET](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection)
- [EF Core — IEntityTypeConfiguration](https://learn.microsoft.com/ef/core/modeling/bulk-configuration)
