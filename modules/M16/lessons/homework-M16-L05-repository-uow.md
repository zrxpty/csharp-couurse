---
[← К уроку M16-L05](lesson-M16-L05-repository-uow.md) | [⬆ К модулю M16](../README.md) | [Следующее ДЗ →](homework-M16-L06-factory-builder.md)
---

### Домашнее задание M16-L05: Repository + Unit of Work (и критика) / Homework M16-L05: Repository + Unit of Work (and critique)

**Урок / Lesson:** M16-L05
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) На реалистичном домене «Заказы» построить узкий доменный репозиторий для корня агрегата, тонкий Unit of Work поверх `DbContext` с явной границей транзакции, отдельную read-side ветку через `DbContext` напрямую (CQRS) и интеграционные тесты на SQLite in-memory; затем письменно обосновать, где репозиторий оправдан, а где — антипаттерн, дублирующий EF Core. (EN) On a realistic "Orders" domain, build a narrow domain repository for an aggregate root, a thin Unit of Work over `DbContext` with an explicit transaction boundary, a separate read side that uses `DbContext` directly (CQRS), and integration tests on SQLite in-memory; then justify in writing where a repository is justified and where it is an anti-pattern that merely duplicates EF Core.

#### Связь с уроком / Connection to the lesson

(RU) Урок доказывает, что `DbContext` уже является Unit of Work, а `DbSet<T>` — репозиторием, поэтому ручные обёртки часто дублируют EF и теряют LINQ. Это ДЗ закрепляет ту же мысль практически: вы пишете репозиторий только там, где он защищает инварианты агрегата и формализует доменный контракт, а read-side ведёте через `DbContext` напрямую с `AsNoTracking`. Критика паттерна из урока (generic `Repository<T>` с публичным `IQueryable`, `SaveChanges` в `Add`, mock-репозитории вместо реальной БД) отражена в критериях приёмки и в подсказках.

(EN) The lesson argues that `DbContext` is already a Unit of Work and `DbSet<T>` is already a repository, so hand-rolled wrappers often duplicate EF and lose LINQ power. This homework reinforces the same idea in practice: you write a repository only where it protects aggregate invariants and formalises a domain contract, while the read side uses `DbContext` directly with `AsNoTracking`. The lesson's critique (generic `Repository<T>` with a public `IQueryable`, `SaveChanges` inside `Add`, mock repositories instead of a real DB) is reflected in the acceptance criteria and the hints.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы пришли в команду, которая разрабатывает бэкенд интернет-магазина на C# 12 и .NET 8 с EF Core 8. В кодовой базе уже есть три проблемы, описанные в уроке M16-L05. Во-первых, разработчики оборачивают каждый `DbSet` в `GenericRepository<T>` с публичным методом `IQueryable<T> Query()`, и в сервисы утекает LINQ вместе с `Include`, `AsNoTracking` и пагинацией — абстракция стала тоньше бумаги. Во-вторых, в каждом методе репозитория вызывается `SaveChanges`, поэтому добавить заказ и списать склад можно в разных транзакциях, и при сбое остаётся «полу-заказ». В-третьих, тесты построены на mock-репозиториях: они проходят, но реальный SQL и навигационные свойства не проверяются вообще.

Вам поручают показать команде «правильный» путь: построить узкий доменный репозиторий только для корня агрегата `Order`, ввести тонкий `IUnitOfWork`, формализующий границу транзакции (`SaveChangesAsync` в одном месте), вынести read-side в отдельный query-класс через `DbContext` напрямую (CQRS), и покрыть всё интеграционными тестами на SQLite in-memory. Дополнительно вы должны написать короткую служебную записку, в которой честно объясните, почему в данном участке репозиторий оправдан, а `GenericRepository<T>` — нет, опираясь на аргументы урока. Цель — не «сделать по шаблону», а научиться различать абстракцию от дублирования и осознанно выбирать границу между доменом и инфраструктурой.

#### Что нужно сделать (пошагово)

1. Создайте решение и три проекта. Из пустой папки выполните `dotnet new sln -n Course.M16.L05.Homework`, затем `dotnet new classlib -n Course.M16.L05.Domain -o src/Course.M16.L05.Domain -f net8.0`, `dotnet new classlib -n Course.M16.L05.Infrastructure -o src/Course.M16.L05.Infrastructure -f net8.0` и `dotnet new xunit -n Course.M16.L05.Tests -o tests/Course.M16.L05.Tests -f net8.0`. Свяжите их `dotnet sln add **/*.csproj`, добавьте ссылки: Infrastructure → Domain, Tests → Infrastructure и Tests → Domain. Установите в Infrastructure и Tests пакет `Microsoft.EntityFrameworkCore.Sqlite` версии 8.0.x командой `dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 8.0.*`.

2. В проекте Domain опишите корень агрегата `Order` с инвариантами из урока: нельзя добавить позицию в отменённый заказ, нельзя отменить отгруженный, количество и цена неотрицательны. Используйте приватный конструктор, статическую фабрику `Order.Place`, приватное поле `_items` типа `List<OrderItem>` через collection expression `[]` и навигационное свойство `IReadOnlyCollection<OrderItem> Items`. Значение `OrderStatus` храните как enum.

3. Опишите узкий интерфейс `IOrderRepository` с методами `FindByIdAsync`, `FindByCustomerSinceAsync`, `AddAsync`, `Remove`. Никакого `IQueryable<Order>` наружу — только материализованные объекты и коллекции. Опишите `IUnitOfWork` с свойством `IOrderRepository Orders` и методом `SaveChangesAsync`, плюс `IUnitOfWorkFactory.Create()`.

4. В проекте Infrastructure создайте `AppDbContext` с `DbSet<Order>` и `DbSet<OrderItem>`, настройте маппинг приватного поля `_items` через `PropertyAccessMode.Field` и конверсию `OrderStatus` в строку. Реализуйте `OrderRepository` и `EfUnitOfWork` так, чтобы `SaveChangesAsync` вызывал `_db.SaveChangesAsync`, а `FindByIdAsync` делал `Include(o => o.Items)`. В read-side сделайте класс `OrderSummaryQuery`, использующий `AppDbContext` напрямую с `AsNoTracking` и проекцией в record `Summary`.

5. Напишите два обработчика команд: `AddItemToOrderHandler` и `CancelOrderHandler`, каждый создаёт UoW через фабрику, читает агрегат, вызывает доменный метод, коммитит одной транзакцией. Покройте их интеграционными тестами на SQLite in-memory: создайте `SqliteConnection` с `Mode=Memory` и `Cache=Shared`, инициализируйте схему `db.Database.EnsureCreated()`, проверьте, что добавление позиции в отменённый заказ бросает исключение и не сохраняет ничего, а успешный путь сохраняет позицию.

6. Напишите служебную записку (markdown-файл `NOTES.md` в корне решения) на 200–300 слов: обоснуйте, почему здесь репозиторий оправдан (DDD-инварианты, явный доменный контракт, защита агрегата), а `GenericRepository<T>` — нет (дублирование `DbSet`, утечка `IQueryable`, потеря LINQ и `AsNoTracking`). Запустите `dotnet test` — все тесты должны быть зелёными. Запустите `dotnet build -warnaserror` — без предупреждений.

#### Требования к решению

Решение должно быть разделено на слои: Domain не ссылается на EF Core и на Infrastructure; Infrastructure ссылается на Domain и на `Microsoft.EntityFrameworkCore`; Tests ссылается на оба. Репозиторий создаётся только для корня агрегата `Order`, для `OrderItem` репозитория быть не должно — позиции изменяются только через методы `Order`. Репозиторий возвращает `Order`, `IReadOnlyList<Order>` или `null`, но никогда `IQueryable<Order>`.

`IUnitOfWork` должен быть тонким: только `Orders` и `SaveChangesAsync`, без перекладывания доменной логики. Коммит происходит исключительно в `SaveChangesAsync` UoW; методы `AddAsync` и `Remove` репозитория не вызывают `SaveChanges`. Каждая команда (`AddItemToOrderHandler`, `CancelOrderHandler`) работает в одной UoW и одной транзакции — если инвариант нарушен, ничего не сохраняется.

Read-side (`OrderSummaryQuery`) идёт через `DbContext` напрямую, с `AsNoTracking` и проекцией в DTO, без репозитория и без `Include` — это демонстрирует CQRS-компромисс из урока. Тесты — интеграционные, на SQLite in-memory, без mock-репозиториев: они должны проверять реальное сохранение, реальное чтение и реальное соблюдение инвариантов. Код использует возможности C# 12: top-level statements в тестовом хосте, collection expressions, primary constructors, record-типы для DTO, raw string literals где уместно (например, в SQL-комментариях).

#### Тонкости и подводные камни

Главная тонкость — приватное поле `_items` и навигационное свойство `Items`. EF Core 8 маппит поле через `SetPropertyAccessMode(PropertyAccessMode.Field)`, но если вы забудете `Include(o => o.Items)` в `FindByIdAsync`, то `Items` будет пустым, и инвариант «нельзя добавить в отменённый» сработает, но вы не увидите существующих позиций при отладке. Для read-side с `AsNoTracking` обязательно делайте проекцию через `Select` до `ToListAsync`, иначе `AsNoTracking` не спасёт от загрузки целых сущностей и лишних столбцов.

Вторая тонкость — транзакционность. Если вы случайно вызовете `SaveChanges` внутри `OrderRepository.AddAsync` («по привычке»), то тест «добавить позицию в отменённый заказ не должно сохраняться» пройдёт ложно: исключение бросится после коммита. Поэтому `AddAsync` только добавляет в change tracker, а коммит — в `EfUnitOfWork.SaveChangesAsync`. Третья тонкость — время жизни `DbContext`: `EfUnitOfWork` должен быть `IDisposable` и использоваться через `await using`, иначе `SqliteConnection` с `Cache=Shared` может остаться висеть. Четвёртая — `OrderStatus` конвертируется в строку через `HasConversion<string>()`; если этого не сделать, enum хранится как int и читается хуже в отладке. Пятая — в SQLite нет `DateTimeOffset` нативно, EF хранит как строку; это нормально для тестов, но помните об этом при сравнении дат в `FindByCustomerSinceAsync`. Шестая — не делайте `GenericRepository<T>` «на всякий случай»: это ровно та ошибка, которую критикует урок, и она ломает инварианты агрегата, потому что `T` может быть любой сущностью. Седьмая — в тестах используйте общий `SqliteConnection` (keep-alive), иначе `Mode=Memory` закроется и схема пропадёт.

#### Критерии приёмки

- [ ] Решение разбито на три проекта: Domain, Infrastructure, Tests, с правильными ссылками.
- [ ] Domain не ссылается на `Microsoft.EntityFrameworkCore` и на Infrastructure.
- [ ] `Order` — корень агрегата с приватным конструктором, статической фабрикой `Place` и инвариантами.
- [ ] `OrderItem` создается только через `Order` (internal-конструктор), у него нет своего репозитория.
- [ ] `IOrderRepository` не выставляет `IQueryable<Order>` — только материализованные объекты и коллекции.
- [ ] `IUnitOfWork` тонкий: только `Orders` и `SaveChangesAsync`, без доменной логики.
- [ ] `SaveChangesAsync` вызывается только в `EfUnitOfWork`, не в методах репозитория.
- [ ] `FindByIdAsync` делает `Include(o => o.Items)`, иначе агрегат загружается неполным.
- [ ] `OrderSummaryQuery` использует `DbContext` напрямую с `AsNoTracking` и `Select`-проекцией.
- [ ] `AddItemToOrderHandler` и `CancelOrderHandler` работают в одной UoW и одной транзакции.
- [ ] Интеграционные тесты на SQLite in-memory проходят без mock-репозиториев.
- [ ] Тест «добавить позицию в отменённый заказ» бросает исключение и ничего не сохраняет.
- [ ] Тест «отменить отгруженный заказ» бросает исключение.
- [ ] `dotnet build -warnaserror` проходит без предупреждений.
- [ ] `NOTES.md` содержит обоснование «почему репозиторий оправдан, а `GenericRepository<T>` — нет» со ссылками на аргументы урока.
- [ ] В коде нет `GenericRepository<T>` с публичным `IQueryable` — это антипаттерн из урока.

#### Подсказки (без прямого ответа)

- Подумайте, кто владеет `OrderItem`: если у `OrderItem` появляется публичный репозиторий, инвариант «позиции только через заказ» нарушен. Сделайте конструктор `internal`.
- Для тестов заведите базовый класс `SqliteTestBase`, который в `[AssemblyInitialize]` (или в конструкторе базового класса `IAsyncLifetime`) открывает один `SqliteConnection` и держит его открытым на время теста.
- В `OnModelCreating` не забудьте `SetPropertyAccessMode(PropertyAccessMode.Field)` для навигации `Items`, иначе EF не сможет писать в приватное `_items`.
- Чтобы доказать, что коммит не произошёл при нарушении инварианта, после ожидаемого исключения перечитайте заказ из нового `DbContext` и убедитесь, что позиций столько же, сколько до.
- В `OrderSummaryQuery` делайте `Select` до `ToListAsync`: проекция на стороне сервера уменьшает объём данных и автоматически отключает трекинг.
- В `NOTES.md` опирайтесь на четыре пункта критики из урока: дублирование EF, потеря LINQ, мешанина EF-фич, ложное упрощение тестирования.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 / EF Core 8 — Эталон ДЗ M16-L05.
// Reference solution: narrow domain repository, thin UoW, CQRS read-side, SQLite tests.

using Microsoft.EntityFrameworkCore;
using Microsoft.Data.Sqlite;

namespace Course.M16.L05.Hw;

// --- Domain / Домен -----------------------------------------------------------

public enum OrderStatus { Placed, Shipped, Cancelled }

// Корень агрегата. Приватный конструктор + фабрика защищают инварианты.
// Aggregate root. Private constructor + factory protect invariants.
public sealed class Order
{
    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    private readonly List<OrderItem> _items = [];          // C# 12 collection expression
    public IReadOnlyCollection<OrderItem> Items => _items;
    public DateTimeOffset CreatedAt { get; private set; }

    private Order() { }                                    // для EF / for EF

    public static Order Place(Guid customerId, IEnumerable<(Guid productId, int qty, decimal price)> lines)
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            Status = OrderStatus.Placed,
            CreatedAt = DateTimeOffset.UtcNow
        };
        foreach (var (productId, qty, price) in lines)
            order._items.Add(new OrderItem(order.Id, productId, qty, price));
        return order;
    }

    public void Ship()
    {
        if (Status != OrderStatus.Placed)
            throw new InvalidOperationException("Можно отгрузить только размещённый заказ. / Only placed order can be shipped.");
        Status = OrderStatus.Shipped;
    }

    public void Cancel()
    {
        if (Status == OrderStatus.Shipped)
            throw new InvalidOperationException("Отгруженный заказ нельзя отменить. / Shipped order cannot be cancelled.");
        Status = OrderStatus.Cancelled;
    }

    public void AddItem(Guid productId, int qty, decimal price)
    {
        if (Status == OrderStatus.Cancelled)
            throw new InvalidOperationException("Нельзя добавить позицию в отменённый заказ. / Cannot add to cancelled order.");
        _items.Add(new OrderItem(Id, productId, qty, price));
    }
}

public sealed class OrderItem
{
    public Guid Id { get; private set; } = Guid.NewGuid();
    public Guid OrderId { get; private set; }
    public Guid ProductId { get; private set; }
    public int Quantity { get; private set; }
    public decimal UnitPrice { get; private set; }

    internal OrderItem(Guid orderId, Guid productId, int qty, decimal price)  // internal — инкапсуляция
    {
        if (qty <= 0) throw new ArgumentOutOfRangeException(nameof(qty));
        if (price < 0) throw new ArgumentOutOfRangeException(nameof(price));
        OrderId = orderId; ProductId = productId; Quantity = qty; UnitPrice = price;
    }
}

// --- Узкий доменный репозиторий / Narrow domain repository -------------------

public interface IOrderRepository
{
    Task<Order?> FindByIdAsync(Guid id, CancellationToken ct = default);
    Task<IReadOnlyList<Order>> FindByCustomerSinceAsync(Guid customerId, DateTimeOffset since, CancellationToken ct = default);
    Task AddAsync(Order order, CancellationToken ct = default);
    void Remove(Order order);
}

public interface IUnitOfWork : IDisposable
{
    IOrderRepository Orders { get; }
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}

public interface IUnitOfWorkFactory
{
    IUnitOfWork Create();
}

// --- Infrastructure / Инфраструктура -----------------------------------------

public sealed class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.Entity<Order>(e =>
        {
            e.HasKey(o => o.Id);
            e.Property(o => o.Status).HasConversion<string>();
            e.HasMany(o => o.Items).WithOne().HasForeignKey(i => i.OrderId);
            e.Metadata.FindNavigation(nameof(Order.Items))!
                       .SetPropertyAccessMode(PropertyAccessMode.Field);   // приватное _items
        });
        b.Entity<OrderItem>().HasKey(i => i.Id);
    }
}

public sealed class OrderRepository(AppDbContext db) : IOrderRepository
{
    private readonly DbSet<Order> _orders = db.Orders;

    public Task<Order?> FindByIdAsync(Guid id, CancellationToken ct = default) =>
        _orders.Include(o => o.Items).FirstOrDefaultAsync(o => o.Id == id, ct);   // полный агрегат

    public async Task<IReadOnlyList<Order>> FindByCustomerSinceAsync(Guid customerId, DateTimeOffset since, CancellationToken ct = default)
    {
        var list = await _orders
            .Include(o => o.Items)
            .Where(o => o.CustomerId == customerId && o.CreatedAt >= since)
            .AsNoTracking()
            .ToListAsync(ct);
        return list;
    }

    public Task AddAsync(Order order, CancellationToken ct = default) =>
        _orders.AddAsync(order, ct).AsTask();   // НЕТ SaveChanges — коммит в UoW

    public void Remove(Order order) => _orders.Remove(order);
}

public sealed class EfUnitOfWork(AppDbContext db) : IUnitOfWork
{
    public IOrderRepository Orders { get; } = new OrderRepository(db);
    public Task<int> SaveChangesAsync(CancellationToken ct = default) => db.SaveChangesAsync(ct);
    public void Dispose() => db.Dispose();
}

public sealed class EfUnitOfWorkFactory(AppDbContext db) : IUnitOfWorkFactory
{
    public IUnitOfWork Create() => new EfUnitOfWork(db);
}

// --- Команды / Commands -----------------------------------------------------

public sealed class AddItemToOrderHandler(IUnitOfWorkFactory factory)
{
    public async Task HandleAsync(Guid orderId, Guid productId, int qty, decimal price, CancellationToken ct = default)
    {
        await using var uow = factory.Create();           // одна UoW = одна транзакция
        var order = await uow.Orders.FindByIdAsync(orderId, ct)
            ?? throw new InvalidOperationException("Заказ не найден. / Order not found.");
        order.AddItem(productId, qty, price);             // инвариант в домене
        await uow.SaveChangesAsync(ct);                   // коммит один раз
    }
}

public sealed class CancelOrderHandler(IUnitOfWorkFactory factory)
{
    public async Task HandleAsync(Guid orderId, CancellationToken ct = default)
    {
        await using var uow = factory.Create();
        var order = await uow.Orders.FindByIdAsync(orderId, ct)
            ?? throw new InvalidOperationException("Заказ не найден. / Order not found.");
        order.Cancel();
        await uow.SaveChangesAsync(ct);
    }
}

// --- Read-side (CQRS) / Сторона чтения --------------------------------------

public sealed class OrderSummaryQuery(AppDbContext db)
{
    public record Summary(Guid OrderId, Guid CustomerId, int Items, decimal Total, string Status);

    public async Task<IReadOnlyList<Summary>> RunAsync(Guid customerId, CancellationToken ct = default) =>
        await db.Orders
            .AsNoTracking()
            .Where(o => o.CustomerId == customerId)
            .Select(o => new Summary(
                o.Id, o.CustomerId, o.Items.Count,
                o.Items.Sum(i => i.Quantity * i.UnitPrice),
                o.Status.ToString()))
            .ToListAsync(ct);
}

// --- Тестовый хост (SQLite in-memory) / Test host ---------------------------

public sealed class SqliteTestBase : IAsyncLifetime
{
    private SqliteConnection? _connection;
    protected AppDbContext Db = null!;

    public async Task InitializeAsync()
    {
        _connection = new SqliteConnection("DataSource=:memory:");
        await _connection.OpenAsync();                    // keep-alive, иначе схема пропадёт
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlite(_connection)
            .Options;
        Db = new AppDbContext(options);
        await Db.Database.EnsureCreatedAsync();
    }

    public async Task DisposeAsync()
    {
        await Db.DisposeAsync();
        if (_connection is not null) await _connection.DisposeAsync();
    }
}
```

Разбор по строкам. `Order` — корень агрегата: приватный конструктор и фабрика `Place` гарантируют, что заказ нельзя создать «пустым» или в невалидном статусе; методы `Ship`, `Cancel`, `AddItem` проверяют инварианты в домене, а не в сервисе. `OrderItem` имеет `internal`-конструктор, поэтому позиции создаются только через `Order`, и отдельного репозитория для `OrderItem` нет — это прямо best practice из урока. `IOrderRepository` возвращает `Order` и `IReadOnlyList<Order>`, но не `IQueryable` — абстракция не утекает. `OrderRepository.FindByIdAsync` делает `Include(o => o.Items)`, иначе агрегат загрузился бы без позиций и инвариант проверки статуса работал бы на неполных данных; `AsNoTracking` в `FindByCustomerSinceAsync` оптимизирует чтение. `EfUnitOfWork` тонкий: только `Orders` и `SaveChangesAsync`, вызов `db.SaveChangesAsync` — единственный коммит. `AddAsync` намеренно не вызывает `SaveChanges`, поэтому нарушенный инвариант откатывает всю транзакцию — это и есть UoW-семантика. Команды `AddItemToOrderHandler` и `CancelOrderHandler` работают в `await using var uow`, что гарантирует dispose и закрытие `DbContext`. `OrderSummaryQuery` демонстрирует CQRS: read-side идёт через `DbContext` напрямую, с `AsNoTracking` и `Select`-проекцией, без репозитория — LINQ остаётся доступным, и это не противоречит абстракции, потому что здесь нет инвариантов. `SqliteTestBase` держит `SqliteConnection` открытым на время теста (`DataSource=:memory:` с keep-alive), иначе in-memory база закроется и схема пропадёт; `EnsureCreatedAsync` строит схему по модели. Вся эта структура отвечает критике урока: нет `GenericRepository<T>`, нет `IQueryable` наружу, нет `SaveChanges` в `Add`, нет mock-репозиториев.

#### Задания на углубление (бонус)

1. Добавьте спецификацию `OrderSpecification` (паттерн Specification) для составных запросов «по статусу и диапазону дат», не возвращая `IQueryable`. Покажите, как EF-реализация транслирует спецификацию в `Where`.
2. Замените `EfUnitOfWork` на вариант с явной `IDbContextTransaction` и `IsolationLevel.Serializable`, докажите тестом, что две параллельные команды `Cancel` не приводят к гонке.
3. Подключите Testcontainers for .NET с реальным PostgreSQL и повторите тесты; сравните поведение `DateTimeOffset` и конверсии enum в строку.
4. Добавьте доменное событие `OrderCancelledDomainEvent` и диспетчер внутри `SaveChangesAsync` (before/after commit); обсудите, почему диспатч именно после `SaveChanges` безопаснее.

---

## Statement in English / Постановка на английском

#### Context & motivation

You have joined a team building the backend of an online shop on C# 12 and .NET 8 with EF Core 8. The code base already suffers from the three problems described in lesson M16-L05. First, developers wrap every `DbSet` in a `GenericRepository<T>` that exposes a public `IQueryable<T> Query()`, so LINQ together with `Include`, `AsNoTracking` and paging leaks into services — the abstraction has become thinner than paper. Second, each repository method calls `SaveChanges`, so adding an order and decrementing stock can happen in two separate transactions, leaving a "half-order" behind on failure. Third, tests are built on mock repositories: they pass, but no real SQL and no real navigation properties are ever exercised.

Your task is to show the team the "right" path: build a narrow domain repository only for the `Order` aggregate root, introduce a thin `IUnitOfWork` that formalises the transaction boundary (a single `SaveChangesAsync`), move the read side to a separate query class that uses `DbContext` directly (CQRS), and cover everything with integration tests on SQLite in-memory. You must also write a short design memo that honestly explains why a repository is justified in this part of the code, while `GenericRepository<T>` is not, grounded in the arguments of the lesson. The goal is not to "follow a template", but to learn to tell abstraction from duplication and to deliberately choose the boundary between the domain and infrastructure layers.

#### What to do step by step

1. Create a solution and three projects. From an empty folder run `dotnet new sln -n Course.M16.L05.Homework`, then `dotnet new classlib -n Course.M16.L05.Domain -o src/Course.M16.L05.Domain -f net8.0`, `dotnet new classlib -n Course.M16.L05.Infrastructure -o src/Course.M16.L05.Infrastructure -f net8.0`, and `dotnet new xunit -n Course.M16.L05.Tests -o tests/Course.M16.L05.Tests -f net8.0`. Link them with `dotnet sln add **/*.csproj`, and add references: Infrastructure → Domain, Tests → Infrastructure and Tests → Domain. Install the `Microsoft.EntityFrameworkCore.Sqlite` package version 8.0.x in Infrastructure and Tests via `dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 8.0.*`.

2. In the Domain project, describe the `Order` aggregate root with the invariants from the lesson: an item cannot be added to a cancelled order, a shipped order cannot be cancelled, quantity and price must be non-negative. Use a private constructor, a static `Order.Place` factory, a private `_items` field of type `List<OrderItem>` initialised with the C# 12 collection expression `[]`, and a navigation property `IReadOnlyCollection<OrderItem> Items`. Keep `OrderStatus` as an enum.

3. Describe the narrow `IOrderRepository` interface with methods `FindByIdAsync`, `FindByCustomerSinceAsync`, `AddAsync`, `Remove`. No `IQueryable<Order>` leaks out — only materialised objects and collections. Describe `IUnitOfWork` with an `IOrderRepository Orders` property and a `SaveChangesAsync` method, plus an `IUnitOfWorkFactory.Create()` method.

4. In the Infrastructure project, create `AppDbContext` with `DbSet<Order>` and `DbSet<OrderItem>`, configure the mapping of the private `_items` field through `PropertyAccessMode.Field`, and convert `OrderStatus` to a string. Implement `OrderRepository` and `EfUnitOfWork` so that `SaveChangesAsync` calls `_db.SaveChangesAsync`, and `FindByIdAsync` performs `Include(o => o.Items)`. For the read side, build a class `OrderSummaryQuery` that uses `AppDbContext` directly with `AsNoTracking` and a projection into a `Summary` record.

5. Write two command handlers, `AddItemToOrderHandler` and `CancelOrderHandler`; each one creates a UoW through the factory, loads the aggregate, calls a domain method, and commits in a single transaction. Cover them with integration tests on SQLite in-memory: open a `SqliteConnection` with `Mode=Memory` and `Cache=Shared`, initialise the schema with `db.Database.EnsureCreated()`, and verify that adding an item to a cancelled order throws and persists nothing, while the happy path persists the item.

6. Write a short memo (a markdown file `NOTES.md` at the solution root, 200–300 words): justify why a repository is justified here (DDD invariants, an explicit domain contract, aggregate protection), while `GenericRepository<T>` is not (duplicates `DbSet`, leaks `IQueryable`, loses LINQ and `AsNoTracking`). Run `dotnet test` — all tests must be green. Run `dotnet build -warnaserror` — no warnings.

#### Requirements

The solution must be layered: Domain does not reference EF Core or Infrastructure; Infrastructure references Domain and `Microsoft.EntityFrameworkCore`; Tests references both. The repository is created only for the `Order` aggregate root — there must be no `OrderItem` repository, because items change only through `Order` methods. The repository returns `Order`, `IReadOnlyList<Order>` or `null`, but never `IQueryable<Order>`.

`IUnitOfWork` must be thin: only `Orders` and `SaveChangesAsync`, with no relocated domain logic. Commit happens exclusively in the UoW's `SaveChangesAsync`; the repository's `AddAsync` and `Remove` methods do not call `SaveChanges`. Each command (`AddItemToOrderHandler`, `CancelOrderHandler`) runs inside a single UoW and a single transaction — if an invariant is violated, nothing is persisted.

The read side (`OrderSummaryQuery`) goes through `DbContext` directly, with `AsNoTracking` and a projection into a DTO, without a repository and without `Include` — this demonstrates the CQRS compromise from the lesson. Tests are integration tests on SQLite in-memory, with no mock repositories: they must verify real persistence, real reads, and real invariant enforcement. The code uses C# 12 features: top-level statements in the test host, collection expressions, primary constructors, record types for DTOs, and raw string literals where appropriate (for example, in SQL comments).

#### Pitfalls

The main pitfall is the private `_items` field and the `Items` navigation property. EF Core 8 maps the field through `SetPropertyAccessMode(PropertyAccessMode.Field)`, but if you forget `Include(o => o.Items)` in `FindByIdAsync`, then `Items` will be empty and the "cannot add to cancelled" invariant still fires, yet you will not see the existing items during debugging. For the read side with `AsNoTracking`, always project through `Select` before `ToListAsync`, otherwise `AsNoTracking` alone will not save you from loading whole entities and unnecessary columns.

The second pitfall is transactional integrity. If you accidentally call `SaveChanges` inside `OrderRepository.AddAsync` ("out of habit"), then the test "adding an item to a cancelled order must not persist" will pass falsely: the exception will be thrown after the commit. Therefore `AddAsync` only adds to the change tracker, and the commit lives in `EfUnitOfWork.SaveChangesAsync`. The third pitfall is `DbContext` lifetime: `EfUnitOfWork` must be `IDisposable` and used through `await using`, otherwise a `SqliteConnection` with `Cache=Shared` may linger. The fourth: `OrderStatus` is converted to a string through `HasConversion<string>()`; without it, the enum is stored as an int and is harder to read in debugging. The fifth: SQLite has no native `DateTimeOffset`, EF stores it as a string; that is fine for tests, but keep it in mind when comparing dates in `FindByCustomerSinceAsync`. The sixth: do not write a `GenericRepository<T>` "just in case" — that is exactly the mistake the lesson criticises, and it breaks aggregate invariants because `T` can be any entity. The seventh: in tests, keep a single `SqliteConnection` alive, otherwise `Mode=Memory` closes and the schema disappears.

#### Acceptance criteria

- [ ] The solution is split into three projects: Domain, Infrastructure, Tests, with correct references.
- [ ] Domain does not reference `Microsoft.EntityFrameworkCore` or Infrastructure.
- [ ] `Order` is an aggregate root with a private constructor, a static `Place` factory, and invariants.
- [ ] `OrderItem` is created only through `Order` (an internal constructor); it has no repository of its own.
- [ ] `IOrderRepository` does not expose `IQueryable<Order>` — only materialised objects and collections.
- [ ] `IUnitOfWork` is thin: only `Orders` and `SaveChangesAsync`, with no domain logic.
- [ ] `SaveChangesAsync` is called only in `EfUnitOfWork`, not in repository methods.
- [ ] `FindByIdAsync` performs `Include(o => o.Items)`, otherwise the aggregate is loaded incomplete.
- [ ] `OrderSummaryQuery` uses `DbContext` directly with `AsNoTracking` and a `Select` projection.
- [ ] `AddItemToOrderHandler` and `CancelOrderHandler` run in a single UoW and a single transaction.
- [ ] Integration tests on SQLite in-memory pass without mock repositories.
- [ ] The test "add an item to a cancelled order" throws and persists nothing.
- [ ] The test "cancel a shipped order" throws.
- [ ] `dotnet build -warnaserror` passes without warnings.
- [ ] `NOTES.md` contains a justification of "why a repository is justified, and `GenericRepository<T>` is not" with references to the lesson's arguments.
- [ ] There is no `GenericRepository<T>` with a public `IQueryable` in the code — it is the anti-pattern from the lesson.

#### Hints (no direct answer)

- Think about who owns `OrderItem`: if `OrderItem` gets a public repository, the "items only through the order" invariant is broken. Make the constructor `internal`.
- For tests, create a `SqliteTestBase` base class that opens a single `SqliteConnection` in its initialiser (or via `IAsyncLifetime`) and keeps it open for the duration of the test.
- In `OnModelCreating`, do not forget `SetPropertyAccessMode(PropertyAccessMode.Field)` for the `Items` navigation, otherwise EF will not be able to write to the private `_items`.
- To prove the commit did not happen when an invariant was violated, after the expected exception reload the order from a fresh `DbContext` and assert the item count is unchanged.
- In `OrderSummaryQuery`, do the `Select` before `ToListAsync`: server-side projection reduces the data volume and automatically disables tracking.
- In `NOTES.md`, lean on the four critique points from the lesson: EF duplication, LINQ loss, EF-feature interference, false testing simplification.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 / EF Core 8 — Reference for homework M16-L05.
// Narrow domain repository, thin UoW, CQRS read-side, SQLite tests.

using Microsoft.EntityFrameworkCore;
using Microsoft.Data.Sqlite;

namespace Course.M16.L05.Hw;

// --- Domain ------------------------------------------------------------------

public enum OrderStatus { Placed, Shipped, Cancelled }

// Aggregate root. Private constructor + factory protect invariants.
public sealed class Order
{
    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    private readonly List<OrderItem> _items = [];          // C# 12 collection expression
    public IReadOnlyCollection<OrderItem> Items => _items;
    public DateTimeOffset CreatedAt { get; private set; }

    private Order() { }                                    // for EF

    public static Order Place(Guid customerId, IEnumerable<(Guid productId, int qty, decimal price)> lines)
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            Status = OrderStatus.Placed,
            CreatedAt = DateTimeOffset.UtcNow
        };
        foreach (var (productId, qty, price) in lines)
            order._items.Add(new OrderItem(order.Id, productId, qty, price));
        return order;
    }

    public void Ship()
    {
        if (Status != OrderStatus.Placed)
            throw new InvalidOperationException("Only a placed order can be shipped.");
        Status = OrderStatus.Shipped;
    }

    public void Cancel()
    {
        if (Status == OrderStatus.Shipped)
            throw new InvalidOperationException("A shipped order cannot be cancelled.");
        Status = OrderStatus.Cancelled;
    }

    public void AddItem(Guid productId, int qty, decimal price)
    {
        if (Status == OrderStatus.Cancelled)
            throw new InvalidOperationException("Cannot add an item to a cancelled order.");
        _items.Add(new OrderItem(Id, productId, qty, price));
    }
}

public sealed class OrderItem
{
    public Guid Id { get; private set; } = Guid.NewGuid();
    public Guid OrderId { get; private set; }
    public Guid ProductId { get; private set; }
    public int Quantity { get; private set; }
    public decimal UnitPrice { get; private set; }

    internal OrderItem(Guid orderId, Guid productId, int qty, decimal price)  // internal — encapsulation
    {
        if (qty <= 0) throw new ArgumentOutOfRangeException(nameof(qty));
        if (price < 0) throw new ArgumentOutOfRangeException(nameof(price));
        OrderId = orderId; ProductId = productId; Quantity = qty; UnitPrice = price;
    }
}

// --- Narrow domain repository ------------------------------------------------

public interface IOrderRepository
{
    Task<Order?> FindByIdAsync(Guid id, CancellationToken ct = default);
    Task<IReadOnlyList<Order>> FindByCustomerSinceAsync(Guid customerId, DateTimeOffset since, CancellationToken ct = default);
    Task AddAsync(Order order, CancellationToken ct = default);
    void Remove(Order order);
}

public interface IUnitOfWork : IDisposable
{
    IOrderRepository Orders { get; }
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}

public interface IUnitOfWorkFactory
{
    IUnitOfWork Create();
}

// --- Infrastructure ----------------------------------------------------------

public sealed class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.Entity<Order>(e =>
        {
            e.HasKey(o => o.Id);
            e.Property(o => o.Status).HasConversion<string>();
            e.HasMany(o => o.Items).WithOne().HasForeignKey(i => i.OrderId);
            e.Metadata.FindNavigation(nameof(Order.Items))!
                       .SetPropertyAccessMode(PropertyAccessMode.Field);   // private _items
        });
        b.Entity<OrderItem>().HasKey(i => i.Id);
    }
}

public sealed class OrderRepository(AppDbContext db) : IOrderRepository
{
    private readonly DbSet<Order> _orders = db.Orders;

    public Task<Order?> FindByIdAsync(Guid id, CancellationToken ct = default) =>
        _orders.Include(o => o.Items).FirstOrDefaultAsync(o => o.Id == id, ct);   // full aggregate

    public async Task<IReadOnlyList<Order>> FindByCustomerSinceAsync(Guid customerId, DateTimeOffset since, CancellationToken ct = default)
    {
        var list = await _orders
            .Include(o => o.Items)
            .Where(o => o.CustomerId == customerId && o.CreatedAt >= since)
            .AsNoTracking()
            .ToListAsync(ct);
        return list;
    }

    public Task AddAsync(Order order, CancellationToken ct = default) =>
        _orders.AddAsync(order, ct).AsTask();   // NO SaveChanges — commit in UoW

    public void Remove(Order order) => _orders.Remove(order);
}

public sealed class EfUnitOfWork(AppDbContext db) : IUnitOfWork
{
    public IOrderRepository Orders { get; } = new OrderRepository(db);
    public Task<int> SaveChangesAsync(CancellationToken ct = default) => db.SaveChangesAsync(ct);
    public void Dispose() => db.Dispose();
}

public sealed class EfUnitOfWorkFactory(AppDbContext db) : IUnitOfWorkFactory
{
    public IUnitOfWork Create() => new EfUnitOfWork(db);
}

// --- Commands ----------------------------------------------------------------

public sealed class AddItemToOrderHandler(IUnitOfWorkFactory factory)
{
    public async Task HandleAsync(Guid orderId, Guid productId, int qty, decimal price, CancellationToken ct = default)
    {
        await using var uow = factory.Create();           // one UoW = one transaction
        var order = await uow.Orders.FindByIdAsync(orderId, ct)
            ?? throw new InvalidOperationException("Order not found.");
        order.AddItem(productId, qty, price);             // invariant in domain
        await uow.SaveChangesAsync(ct);                   // commit once
    }
}

public sealed class CancelOrderHandler(IUnitOfWorkFactory factory)
{
    public async Task HandleAsync(Guid orderId, CancellationToken ct = default)
    {
        await using var uow = factory.Create();
        var order = await uow.Orders.FindByIdAsync(orderId, ct)
            ?? throw new InvalidOperationException("Order not found.");
        order.Cancel();
        await uow.SaveChangesAsync(ct);
    }
}

// --- Read side (CQRS) --------------------------------------------------------

public sealed class OrderSummaryQuery(AppDbContext db)
{
    public record Summary(Guid OrderId, Guid CustomerId, int Items, decimal Total, string Status);

    public async Task<IReadOnlyList<Summary>> RunAsync(Guid customerId, CancellationToken ct = default) =>
        await db.Orders
            .AsNoTracking()
            .Where(o => o.CustomerId == customerId)
            .Select(o => new Summary(
                o.Id, o.CustomerId, o.Items.Count,
                o.Items.Sum(i => i.Quantity * i.UnitPrice),
                o.Status.ToString()))
            .ToListAsync(ct);
}

// --- Test host (SQLite in-memory) -------------------------------------------

public sealed class SqliteTestBase : IAsyncLifetime
{
    private SqliteConnection? _connection;
    protected AppDbContext Db = null!;

    public async Task InitializeAsync()
    {
        _connection = new SqliteConnection("DataSource=:memory:");
        await _connection.OpenAsync();                    // keep-alive, or the schema is lost
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlite(_connection)
            .Options;
        Db = new AppDbContext(options);
        await Db.Database.EnsureCreatedAsync();
    }

    public async Task DisposeAsync()
    {
        await Db.DisposeAsync();
        if (_connection is not null) await _connection.DisposeAsync();
    }
}
```

Walk-through, line by line. `Order` is the aggregate root: a private constructor and the `Place` factory guarantee that an order cannot be created "empty" or in an invalid status; the `Ship`, `Cancel`, and `AddItem` methods enforce invariants in the domain, not in a service. `OrderItem` has an `internal` constructor, so items are only ever created through `Order`, and there is no separate `OrderItem` repository — this is exactly the best practice from the lesson. `IOrderRepository` returns `Order` and `IReadOnlyList<Order>`, but never `IQueryable`, so the abstraction does not leak. `OrderRepository.FindByIdAsync` performs `Include(o => o.Items)`, otherwise the aggregate would load without items and the status invariant would run on incomplete data; `AsNoTracking` in `FindByCustomerSinceAsync` optimises the read. `EfUnitOfWork` is thin: only `Orders` and `SaveChangesAsync`, and the call to `db.SaveChangesAsync` is the only commit. `AddAsync` deliberately does not call `SaveChanges`, so a violated invariant rolls back the whole transaction — that is precisely UoW semantics. The commands `AddItemToOrderHandler` and `CancelOrderHandler` run inside `await using var uow`, which guarantees disposal and closes the `DbContext`. `OrderSummaryQuery` demonstrates CQRS: the read side goes through `DbContext` directly, with `AsNoTracking` and a `Select` projection, without a repository — LINQ stays available, and this does not contradict abstraction because there are no invariants here. `SqliteTestBase` keeps a `SqliteConnection` open for the duration of the test (`DataSource=:memory:` with keep-alive), otherwise the in-memory database closes and the schema disappears; `EnsureCreatedAsync` builds the schema from the model. The whole structure answers the lesson's critique: there is no `GenericRepository<T>`, no `IQueryable` leaking out, no `SaveChanges` in `Add`, and no mock repositories.

#### Going deeper (bonus)

1. Add an `OrderSpecification` (the Specification pattern) for compound queries "by status and date range" without returning `IQueryable`. Show how the EF implementation translates the specification into a `Where` clause.
2. Replace `EfUnitOfWork` with a variant that uses an explicit `IDbContextTransaction` and `IsolationLevel.Serializable`, and prove with a test that two concurrent `Cancel` commands do not cause a race.
3. Bring in Testcontainers for .NET with a real PostgreSQL and rerun the tests; compare the behaviour of `DateTimeOffset` and the enum-to-string conversion.
4. Add a domain event `OrderCancelledDomainEvent` and a dispatcher inside `SaveChangesAsync` (before/after commit); discuss why dispatching after `SaveChanges` is safer.

---

#### Чек-лист сдачи / Submission checklist

- [ ] (RU) Решение собирается через `dotnet build -warnaserror` без предупреждений.
- [ ] (RU) `dotnet test` проходит все интеграционные тесты на SQLite in-memory.
- [ ] (RU) Репозиторий создан только для `Order`, для `OrderItem` его нет.
- [ ] (RU) `SaveChangesAsync` вызывается только в `EfUnitOfWork`, не в методах репозитория.
- [ ] (RU) Read-side (`OrderSummaryQuery`) использует `DbContext` напрямую с `AsNoTracking`.
- [ ] (RU) `NOTES.md` содержит обоснование выбора репозитория и критику `GenericRepository<T>`.
- [ ] (RU) В коде нет `GenericRepository<T>` с публичным `IQueryable`.
- [ ] (EN) Solution builds with `dotnet build -warnaserror` and no warnings.
- [ ] (EN) `dotnet test` passes all integration tests on SQLite in-memory.
- [ ] (EN) The repository exists only for `Order`; there is none for `OrderItem`.
- [ ] (EN) `SaveChangesAsync` is called only in `EfUnitOfWork`, never in repository methods.
- [ ] (EN) The read side (`OrderSummaryQuery`) uses `DbContext` directly with `AsNoTracking`.
- [ ] (EN) `NOTES.md` justifies the repository choice and critiques `GenericRepository<T>`.
- [ ] (EN) No `GenericRepository<T>` with a public `IQueryable` exists in the code.

#### Ресурсы / Resources

- [Microsoft Learn — DDD and CQRS microservice patterns — https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/](https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/)
- [Microsoft Learn — Repository pattern (infrastructure in DDD) — https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design](https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design)
- [EF Core — DbContext lifetime, configuration, init — https://learn.microsoft.com/ef/core/dbcontext-configuration/](https://learn.microsoft.com/ef/core/dbcontext-configuration/)
- [EF Core — SQLite testing with in-memory — https://learn.microsoft.com/ef/core/miscellaneous/testing/sqlite](https://learn.microsoft.com/ef/core/miscellaneous/testing/sqlite)
- [Martin Fowler — Catalog of Patterns of Enterprise Architecture — Repository / Unit of Work — https://martinfowler.com/eaaCatalog/](https://martinfowler.com/eaaCatalog/)
- [Testcontainers for .NET — https://dotnet.testcontainers.org/](https://dotnet.testcontainers.org/)

---
