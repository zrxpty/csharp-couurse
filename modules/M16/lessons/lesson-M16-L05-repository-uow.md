[⬆ К модулю M16](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M16-L05: Repository + Unit of Work (и критика) / Repository + Unit of Work (and critique)

**Модуль / Module:** M16
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Паттерн **Repository** (Репозиторий) — это абстракция над слоем доступа к данным. Её задача — скрыть детали хранения и предоставить коллекционно-подобный интерфейс для работы с сущностями домена. Представьте себе библиотеку: вы не идёте напрямую в хранилище за книгой, а обращаетесь к библиотекарю. Библиотекарь знает, где лежит нужный экземпляр, как его найти по автору или жанру, и как оформить выдачу. Репозиторий — это и есть такой «библиотекарь» для ваших агрегатов в DDD. Он работает с корнями агрегатов (aggregate roots), а не с отдельными сущностями или value-объектами, что важно для поддержания инвариантов.

Классическая сигнатура репозитория выглядит так: `GetById`, `Find` (по критериям), `Add`, `Remove`, иногда `List` или спецификации (Specification). Важно, что репозиторий возвращает уже материализованные доменные объекты, а не `IQueryable`, иначе абстракция утекает в слой вызова. В DDD репозиторий для каждого корня агрегата — отдельный, с узким, предметно-ориентированным API: `OrderRepository.FindByCustomerSince(customerId, date)` вместо универсального `GetAll().Where(...)`.

**Generic Repository<T>** — попытка обобщить идею до одного класса на все сущности: `Repository<T>` с методами `Get`, `Add`, `Remove`, `Find`. Это удобно для CRUD-приложений и шаблонного кода, но часто превращается в «утечку абстракции»: разработчики добавляют `IQueryable<T>`, потом `Include`, потом пагинацию — и репозиторий становится тонкой обёрткой над EF, дублирующей его API 1:1. Именно поэтому многие авторы (включая Microsoft) предостерегают от `Repository<T>` поверх EF Core.

**Unit of Work (Единица работы, UoW)** координирует транзакцию: отслеживает изменения объектов, загруженных в рамках одной логической операции, и сохраняет их атомарно. Аналогия — кассир в супермаркете: вы кладёте товары на ленту (изменения), а кассир одной транзакцией списывает оплату. Если что-то не проходит — весь заказ откатывается. UoW гарантирует, что связанные изменения в нескольких репозиториях сохранятся вместе (или не сохранится ничего).

Важно: **`DbContext` в EF Core сам по себе уже является Unit of Work**, а `DbSet<T>` — репозиторием. `SaveChanges` фиксирует всю транзакцию, а change tracker отслеживает состояние. Поэтому классические ручные UoW+Repository поверх DbContext дублируют то, что уже есть в EF, добавляя сложность без выгоды.

**Критика.** Паттерн Repository над EF часто называют антипаттерном, потому что: (1) дублирует возможности EF (`DbSet` уже репозиторий, `DbContext` уже UoW); (2) теряет возможности LINQ и `IQueryable` — либо вы выставляете их наружу и убиваете абстракцию, либо пишете сотни методов-обёрток; (3) мешает использовать специфичные функции EF (lazy/eager loading, `AsNoTracking`, `FromSql`, `SplitQuery`); (4) усложняет тестирование — mock-репозиторий проверить не проще, чем `InMemory` или `TestContainers` с реальной базой.

**Когда уместен.** Repository оправдан, когда: вы реально хотите отвязаться от конкретной ORM (например, миграция с EF на Dapper); у вас DDD-домен с rich-моделями и строгими инвариантами агрегатов; вы работаете с несколькими источниками данных в одной UoW; вам нужен узкий доменный контракт для конкретных запросов. В типичном CRUD ASP.NET API лучше использовать `DbContext` напрямую через injection — это проще, быстрее и ближе к фреймворку. Компромисс — **CQRS**: команды через репозитории/aggregate roots для согласованности, запросы (read side) через EF/Dapper напрямую для скорости и гибкости. Именно этот подход рекомендует Microsoft в микросервисном руководстве по DDD/CQRS.

#### Theory (EN)

The **Repository** pattern is an abstraction over the data-access layer. Its job is to hide storage details and expose a collection-like API for working with domain entities. Think of a library: you do not walk into the stacks yourself to grab a book, you ask the librarian. The librarian knows where each copy lives, how to find it by author or genre, and how to record the loan. A repository is that librarian for your DDD aggregates. It operates on aggregate roots, not on individual entities or value objects, which is essential for preserving invariants inside each aggregate.

A classic repository signature looks like: `GetById`, `Find` (by criteria), `Add`, `Remove`, sometimes `List` or a Specification. The key rule is that the repository returns materialised domain objects, not an `IQueryable` — otherwise the abstraction leaks into the calling layer. In DDD each aggregate root gets its own repository with a narrow, domain-oriented API: `OrderRepository.FindByCustomerSince(customerId, date)` instead of a generic `GetAll().Where(...)`.

**Generic Repository<T>** is an attempt to generalise the idea into a single class for all entities: `Repository<T>` with `Get`, `Add`, `Remove`, `Find`. It is convenient for CRUD applications and boilerplate reduction, but often becomes an abstraction leak: developers add `IQueryable<T>`, then `Include`, then paging, and the repository ends up a thin wrapper over EF, duplicating its API 1:1. This is exactly why many authors, including Microsoft, caution against `Repository<T>` on top of EF Core.

**Unit of Work (UoW)** coordinates a transaction: it tracks changes to objects loaded during one logical operation and persists them atomically. The analogy is a supermarket cashier: you place items on the belt (the changes), and the cashier processes the whole order in one transaction. If the payment fails, the entire order is rolled back. UoW guarantees that related changes across several repositories commit together — or nothing commits at all.

Crucially, **`DbContext` in EF Core already is a Unit of Work**, and `DbSet<T>` is already a repository. `SaveChanges` commits the whole transaction, and the change tracker records state. So hand-rolled UoW + Repository on top of DbContext typically duplicate what EF already provides, adding complexity with no payoff.

**Critique.** The Repository pattern over EF is often called an anti-pattern because: (1) it duplicates EF capabilities (`DbSet` is a repository, `DbContext` is a UoW); (2) it loses LINQ and `IQueryable` power — either you expose them and kill the abstraction, or you write hundreds of wrapper methods; (3) it gets in the way of EF-specific features (`Include`, `AsNoTracking`, `FromSql`, `SplitQuery`); (4) it does not really improve testing — mocking a repository is not easier than using EF `InMemory` or `Testcontainers` with a real database.

**When it is appropriate.** A repository is justified when: you genuinely want to decouple from a specific ORM (for example, migrating from EF to Dapper); you have a DDD domain with rich models and strict aggregate invariants; you coordinate several data sources inside one UoW; or you need a narrow domain contract for specific queries. In a typical CRUD ASP.NET API it is usually better to inject `DbContext` directly — it is simpler, faster, and closer to the framework. A common compromise is **CQRS**: commands go through repositories / aggregate roots for consistency, while queries (the read side) hit EF or Dapper directly for speed and flexibility. This is exactly the approach Microsoft recommends in its DDD/CQRS microservices guidance.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 / EF Core 8
// Демонстрация: правильный доменный репозиторий, Unit of Work поверх DbContext,
// и сравнение с прямым использованием DbContext. / Domain repository, UoW over DbContext,
// and comparison with direct DbContext usage.

using Microsoft.EntityFrameworkCore;

namespace Course.M16.L05;

// --- Домен / Domain -----------------------------------------------------------

// Корень агрегата «Заказ» с инвариантом: нельзя добавить позицию в отменённый заказ.
// Aggregate root "Order"; invariant: no items may be added to a cancelled order.
public sealed class Order
{
    public Guid Id { get; private set; }          // Идентификатор / Id
    public Guid CustomerId { get; private set; }  // Клиент / Customer
    public OrderStatus Status { get; private set; }
    private readonly List<OrderItem> _items = [];
    public IReadOnlyCollection<OrderItem> Items => _items;
    public DateTimeOffset CreatedAt { get; private set; }

    // Фабрика вместо публичного конструктора — инкапсуляция инвариантов.
    // Factory instead of a public constructor — invariant encapsulation.
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

    public void Cancel()
    {
        if (Status == OrderStatus.Shipped)
            throw new InvalidOperationException("Отгруженный заказ нельзя отменить. / Shipped order cannot be cancelled.");
        Status = OrderStatus.Cancelled;
    }

    public void AddItem(Guid productId, int qty, decimal price)
    {
        if (Status == OrderStatus.Cancelled)
            throw new InvalidOperationException("Нельзя добавить позицию в отменённый заказ. / Cannot add item to a cancelled order.");
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

    internal OrderItem(Guid orderId, Guid productId, int qty, decimal price)
    {
        if (qty <= 0) throw new ArgumentOutOfRangeException(nameof(qty));
        if (price < 0) throw new ArgumentOutOfRangeException(nameof(price));
        OrderId = orderId; ProductId = productId; Quantity = qty; UnitPrice = price;
    }
}

public enum OrderStatus { Placed, Shipped, Cancelled }

// --- Узкий доменный репозиторий (НЕ generic Repository<T>) / Narrow domain repository (NOT generic Repository<T>)

public interface IOrderRepository
{
    // Возвращает материализованный агрегат или null / Returns materialised aggregate or null.
    Task<Order?> FindByIdAsync(Guid id, CancellationToken ct = default);

    // Предметно-ориентированный запрос / Domain-oriented query.
    Task<IReadOnlyList<Order>> FindByCustomerSinceAsync(Guid customerId, DateTimeOffset since, CancellationToken ct = default);

    Task AddAsync(Order order, CancellationToken ct = default);
    void Remove(Order order);
}

// EF-реализация. Обёртка тонкая и осмысленная: защищает инварианты агрегата.
// EF implementation. The wrapper is thin and meaningful: it protects aggregate invariants.
public sealed class OrderRepository(AppDbContext db) : IOrderRepository
{
    private readonly DbSet<Order> _orders = db.Orders;

    public Task<Order?> FindByIdAsync(Guid id, CancellationToken ct = default) =>
        _orders.Include(o => o.Items).FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task<IReadOnlyList<Order>> FindByCustomerSinceAsync(Guid customerId, DateTimeOffset since, CancellationToken ct = default)
    {
        var list = await _orders
            .Include(o => o.Items)
            .Where(o => o.CustomerId == customerId && o.CreatedAt >= since)
            .AsNoTracking()                       // Чтение без трекинга — оптимизация / Read without tracking
            .ToListAsync(ct);
        return list;
    }

    public Task AddAsync(Order order, CancellationToken ct = default) => _orders.AddAsync(order, ct).AsTask();
    public void Remove(Order order) => _orders.Remove(order);
}

// --- Unit of Work / Единица работы --------------------------------------------

// Обёртка над DbContext с явным контрактом Save-всё-или-ничего.
// Wraps DbContext with an explicit all-or-nothing Save contract.
public interface IUnitOfWork
{
    IOrderRepository Orders { get; }
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}

// DbContext сам уже UoW — мы лишь формализуем границу транзакции.
// DbContext itself is already a UoW — we merely formalise the transaction boundary.
public sealed class EfUnitOfWork : IUnitOfWork, IDisposable
{
    private readonly AppDbContext _db;
    public IOrderRepository Orders { get; }

    public EfUnitOfWork(AppDbContext db)
    {
        _db = db;
        Orders = new OrderRepository(_db);
    }

    public Task<int> SaveChangesAsync(CancellationToken ct = default) =>
        _db.SaveChangesAsync(ct);                 // атомарный коммит / atomic commit

    public void Dispose() => _db.Dispose();
}

// --- DbContext / Контекст -----------------------------------------------------

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
            // Приватное поле _items маппится через backing field EF Core 8.
            // Private field _items mapped via EF Core 8 backing field.
            e.Metadata.FindNavigation(nameof(Order.Items))!
             .SetPropertyAccessMode(PropertyAccessMode.Field);
        });
        b.Entity<OrderItem>().HasKey(i => i.Id);
    }
}

// --- Использование в приложении / Usage in application ------------------------

// Команда, использующая репозиторий + UoW для согласованности.
// Command using repository + UoW for consistency.
public sealed class AddItemToOrderHandler(IUnitOfWorkFactory uowFactory)
{
    public async Task HandleAsync(Guid orderId, Guid productId, int qty, decimal price, CancellationToken ct = default)
    {
        // Одна UoW = одна транзакция. / One UoW = one transaction.
        await using var uow = uowFactory.Create();
        var order = await uow.Orders.FindByIdAsync(orderId, ct)
            ?? throw new InvalidOperationException("Заказ не найден. / Order not found.");

        order.AddItem(productId, qty, price);     // инвариант проверяется в домене / invariant checked in domain
        await uow.SaveChangesAsync(ct);            // коммит одной транзакцией / commit in one transaction
    }
}

public interface IUnitOfWorkFactory
{
    IUnitOfWork Create();
}

// --- Прямой DbContext для read-side (CQRS query) / Direct DbContext for the read side (CQRS query) ----

// На стороне чтения репозиторий избыточен — используем DbContext напрямую для гибкости LINQ.
// On the read side a repository is overkill — use DbContext directly for LINQ flexibility.
public sealed class OrderSummaryQuery(AppDbContext db)
{
    public record Summary(Guid OrderId, Guid CustomerId, int Items, decimal Total, string Status);

    public async Task<IReadOnlyList<Summary>> RunAsync(Guid customerId, CancellationToken ct = default) =>
        await db.Orders
            .AsNoTracking()
            .Where(o => o.CustomerId == customerId)
            .Select(o => new Summary(
                o.Id, o.CustomerId, o.Items.Count, o.Items.Sum(i => i.Quantity * i.UnitPrice), o.Status.ToString()))
            .ToListAsync(ct);
}

// --- ПРИМЕР, КОГО ИЗБЕГАТЬ / EXAMPLE TO AVOID ---------------------------------
//
// public class GenericRepository<T> where T : class
// {
//     private readonly DbSet<T> _set;
//     public GenericRepository(DbContext db) => _set = db.Set<T>();
//     public Task<T?> GetAsync(Guid id) => _set.FindAsync(id).AsTask();
//     public IQueryable<T> Query() => _set;           // утечка IQueryable наружу
//     public void Add(T e) => _set.Add(e);
// }
//
// Проблемы: (1) дублирует DbSet 1:1; (2) IQueryable утекает в вызывающий код и убивает абстракцию;
// (3) нет доменного смысла; (4) ломает инварианты агрегатов, потому что T может быть любой сущностью.
// Issues: (1) duplicates DbSet 1:1; (2) IQueryable leaks out and kills the abstraction;
// (3) no domain meaning; (4) breaks aggregate invariants because T can be any entity.
```

#### Best Practices
- Делайте репозиторий для **корня агрегата**, а не для каждой сущности; возвращайте материализованные доменные объекты, а не `IQueryable`.
- Не оборачивайте EF в generic `Repository<T>` «на всякий случай» — это дублирование, а не абстракция.
- Если используете Unit of Work поверх `DbContext`, держите его тонким: `SaveChangesAsync` + доступ к репозиториям, без перекладывания логики.
- Разделяйте запись и чтение (CQRS): команды через репозитории/агрегаты, запросы — через `DbContext`/Dapper напрямую с `AsNoTracking`.
- Тестируйте репозитории на реальной БД (SQLite in-memory, Testcontainers) — mock-репозиторий доказывает лишь то, что вы написали мок.
- Keep one repository per **aggregate root**, not per entity; return materialised domain objects, not `IQueryable`.
- Do not wrap EF in a generic `Repository<T>` “just in case” — that is duplication, not abstraction.
- If you build a Unit of Work over `DbContext`, keep it thin: `SaveChangesAsync` plus access to repositories, with no relocated logic.
- Separate reads and writes (CQRS): commands via repositories/aggregates, queries directly through `DbContext`/Dapper with `AsNoTracking`.
- Test repositories against a real database (SQLite in-memory, Testcontainers) — a mock repository only proves that you wrote a mock.

#### Частые ошибки / Common Mistakes
- `Repository<T>` с публичным `IQueryable<T>` → потеря абстракции и утекание LINQ в сервисы; уберите `IQueryable`, верните коллекции или спецификации.
- Репозиторий для каждой сущности (OrderItem, Address) вместо корня агрегата → нарушение инвариантов; создавайте репозиторий только для корней агрегатов.
- `SaveChanges` в каждом методе репозитория (`Add` сразу коммитит) → ломает транзакционность; коммитьте только в Unit of Work.
- Ручной `IUnitOfWork` с собственным change tracker поверх EF → дублирование `DbContext`; используйте `DbContext` как UoW.
- Mock-репозитории вместо интеграционных тестов → ложное чувство безопасности; тестируйте на реальной БД (SQLite/Testcontainers).
- `Repository<T>` with a public `IQueryable<T>` → lost abstraction and LINQ leaking into services; drop `IQueryable`, return collections or specifications.
- Repository per entity (OrderItem, Address) instead of aggregate root → broken invariants; create repositories only for aggregate roots.
- `SaveChanges` inside every repository method (`Add` commits immediately) → breaks transactional integrity; commit only in the Unit of Work.
- Hand-rolled `IUnitOfWork` with its own change tracker over EF → duplicating `DbContext`; use `DbContext` as the UoW.
- Mock repositories instead of integration tests → false safety; test against a real DB (SQLite/Testcontainers).

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Репозиторий создан только для корня агрегата, а не для каждой сущности.
- [ ] Репозиторий возвращает доменные объекты/коллекции, а не `IQueryable<T>`.
- [ ] `SaveChangesAsync` вызывается только в Unit of Work, не в `Add`/`Remove`.
- [ ] Я обосновал, зачем мне репозиторий (DDD-инварианты, отвязка от ORM, несколько источников данных), а не «по шаблону».
- [ ] Read-side (запросы) идёт через `DbContext`/Dapper напрямую, без репозитория.
- [ ] Тесты доступа к данным используют реальную БД (SQLite/Testcontainers), а не моки репозиториев.
- [ ] Repository is created only for the aggregate root, not for every entity.
- [ ] Repository returns domain objects/collections, not `IQueryable<T>`.
- [ ] `SaveChangesAsync` is called only in the Unit of Work, not in `Add`/`Remove`.
- [ ] I can justify why I need a repository (DDD invariants, ORM decoupling, multiple data sources) — not “by template”.
- [ ] Read-side (queries) goes through `DbContext`/Dapper directly, without a repository.
- [ ] Data-access tests use a real DB (SQLite/Testcontainers), not repository mocks.

#### Ресурсы / Resources
- [Microsoft Learn — микросервисы, DDD и CQRS паттерны — https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/](https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/)
- [Microsoft Learn — Repository pattern (инфраструктура в DDD) — https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design](https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design)
- [EF Core — DbContext lifetime, config, init — https://learn.microsoft.com/ef/core/dbcontext-configuration/](https://learn.microsoft.com/ef/core/dbcontext-configuration/)
- [Martin Fowler — Catalog of Patterns of Enterprise Architecture — Repository / Unit of Work — https://martinfowler.com/eaaCatalog/](https://martinfowler.com/eaaCatalog/)

---

[⬆ К модулю M16](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
