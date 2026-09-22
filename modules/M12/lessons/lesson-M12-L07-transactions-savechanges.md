[⬆ К модулю M12](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M12-L07: Транзакции, SaveChanges, IDbContextTransaction / Transactions, SaveChanges, IDbContextTransaction

**Модуль / Module:** M12
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Транзакция — это гарантия того, что несколько операций с базой данных либо все выполнятся успешно, либо ни одна из них не оставит следов. Классическая аналогия — банковский перевод: если списать деньги со счёта отправителя, но не зачислить на счёт получателя из-за сбоя, деньги просто исчезнут. Транзакция гарантирует принцип «всё или ничего» (ACID-свойство Atomicity).

В Entity Framework Core метод `SaveChanges` **уже сам по себе атомарный**. Когда вы меняете несколько сущностей и вызываете `SaveChanges`, EF Core оборачивает все эти команды `INSERT/UPDATE/DELETE` в одну неявную транзакцию. Если хотя бы одна операция упадёт — откатятся все. Это важный момент: для большинства CRUD-сценариев отдельная транзакция не нужна. Сравните с поваром, который готовит блюдо целиком и подаёт его разом — либо всё на тарелке, либо ничего.

Однако бывают случаи, когда неявной транзакции `SaveChanges` недостаточно:
1. Несколько последовательных вызовов `SaveChanges`, которые логически едины (например, создать заказ → уменьшить остатки товара → списать бонусные баллы).
2. Выполнение «сырых» SQL-команд через `Database.ExecuteSqlRaw` вперемешку с `SaveChanges`.
3. Необходимость задать конкретный **уровень изоляции** (ReadUncommitted, ReadCommitted, RepeatableRead, Serializable, Snapshot).
4. Использование распределённых транзакций или координация с внешним ресурсом.

Для этих сценариев применяют **явные транзакции** через `Database.BeginTransaction()` или асинхронный `BeginTransactionAsync()`. Метод возвращает объект `IDbContextTransaction`, который реализует `IDisposable`. Работа строится по схеме:

```csharp
using var tx = await db.Database.BeginTransactionAsync();
try {
    // ... изменения ...
    await db.SaveChangesAsync();
    await tx.CommitAsync();
} catch {
    await tx.RollbackAsync();
    throw;
}
```

Ключевое правило: **всегда вызывайте `Commit`** явно. Если вы забудете `Commit` и объект транзакции утилизируется (через `using`), EF Core автоматически сделает `Rollback`. Поэтому «незакоммиченная транзакция = откат» — это безопасное поведение, но оно часто становится источником тихих багов, когда программист забыл вызвать `Commit` и не понимает, почему данные не сохранились.

**Уровни изоляции (Isolation Levels)** определяют, как транзакция видит изменения, сделанные параллельными транзакциями:
- `ReadUncommitted` — можно читать «грязные» данные (не подтверждённые другими транзакциями). Самый быстрый, но небезопасный.
- `ReadCommitted` (по умолчанию в SQL Server) — нельзя читать неподтверждённые данные, но можно перечитывать и получать разные результаты (non-repeatable read, phantom read).
- `RepeatableRead` — повторное чтение той же строки даст тот же результат.
- `Serializable` — полная изоляция, транзакция ведёт себя так, будто других нет. Самый медленный.
- `Snapshot` — транзакция работает с «снимком» данных на момент старта (требует настройки в SQL Server).

Уровень изоляции задаётся при старте транзакции: `BeginTransactionAsync(IsolationLevel.RepeatableRead)`. Чем выше изоляция — тем больше блокировок и ниже параллелизм. Всегда выбирайте минимальный уровень, достаточный для корректности.

Метод `ExecuteInTransaction` (точнее, паттерн с использованием `TransactionScope` или хелпера `DbContextTransactionExtensions`) позволяет обернуть блок кода в транзакцию декларативно. В современном .NET предпочтительнее `Database.BeginTransaction` + `using`, а классический `TransactionScope` — для распределённых транзакций с поддержкой `MSDTC` или нового `TransactionScope` в .NET 9+ для асинхронных сценариев через `TransactionScopeAsyncFlowOption.Enabled`.

Помните: транзакции — это про **целостность данных**, а не про производительность. Длинная транзакция держит блокировки и мешает другим. Держите транзакции короткими, внутри них выполняйте только работу с БД, а всю «тяжёлую» бизнес-логику и вызовы внешних сервисов выносите за пределы.

#### Theory (EN)

A transaction is a guarantee that several database operations either all succeed together or leave no trace at all. The classic analogy is a bank transfer: if you debit the sender's account but fail to credit the receiver because of a crash, the money simply vanishes. A transaction enforces the "all or nothing" rule — the Atomicity part of the ACID properties.

In Entity Framework Core, the `SaveChanges` method is **already atomic by itself**. When you modify several entities and call `SaveChanges`, EF Core wraps all the `INSERT/UPDATE/DELETE` commands into a single implicit transaction. If even one operation fails, everything rolls back. This is a key point: for most CRUD scenarios you do not need an explicit transaction at all. Think of a chef who cooks a dish completely and serves it at once — either everything is on the plate or nothing.

However, there are cases where the implicit `SaveChanges` transaction is not enough:
1. Several sequential `SaveChanges` calls that are logically one unit (for example: create an order → decrement product stock → charge bonus points).
2. Running "raw" SQL commands via `Database.ExecuteSqlRaw` mixed with `SaveChanges`.
3. The need to set a specific **isolation level** (ReadUncommitted, ReadCommitted, RepeatableRead, Serializable, Snapshot).
4. Distributed transactions or coordination with an external resource.

For these scenarios you use **explicit transactions** through `Database.BeginTransaction()` or the asynchronous `BeginTransactionAsync()`. The method returns an `IDbContextTransaction` object that implements `IDisposable`. The pattern looks like this:

```csharp
using var tx = await db.Database.BeginTransactionAsync();
try {
    // ... changes ...
    await db.SaveChangesAsync();
    await tx.CommitAsync();
} catch {
    await tx.RollbackAsync();
    throw;
}
```

The crucial rule: **always call `Commit` explicitly**. If you forget `Commit` and the transaction object is disposed (via `using`), EF Core will automatically perform a `Rollback`. So "uncommitted transaction = rollback" is a safe behavior, but it is a frequent source of silent bugs when a developer forgets to call `Commit` and cannot understand why the data was not saved.

**Isolation levels** define how a transaction sees changes made by concurrent transactions:
- `ReadUncommitted` — you can read "dirty" data (not confirmed by other transactions). Fastest but unsafe.
- `ReadCommitted` (the default in SQL Server) — you cannot read uncommitted data, but you can re-read and get different results (non-repeatable read, phantom read).
- `RepeatableRead` — re-reading the same row gives the same result.
- `Serializable` — full isolation, the transaction behaves as if no others exist. Slowest.
- `Snapshot` — the transaction works with a "snapshot" of data taken at start (requires configuration in SQL Server).

You set the isolation level at the start of the transaction: `BeginTransactionAsync(IsolationLevel.RepeatableRead)`. The higher the isolation, the more locks and the lower the concurrency. Always pick the lowest level sufficient for correctness.

The `ExecuteInTransaction` approach (more precisely, the pattern using `TransactionScope` or the `DbContextTransactionExtensions` helper) lets you wrap a block of code in a transaction declaratively. In modern .NET, `Database.BeginTransaction` + `using` is preferred, while the classic `TransactionScope` is used for distributed transactions with `MSDTC` or the newer `TransactionScope` in .NET 9+ for asynchronous scenarios through `TransactionScopeAsyncFlowOption.Enabled`.

Remember: transactions are about **data integrity**, not performance. A long transaction holds locks and blocks others. Keep transactions short, do only database work inside them, and move all heavy business logic and external service calls outside the transaction boundary.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 / EF Core 8
// Демонстрация: неявная транзакция SaveChanges и явная IDbContextTransaction
// Demo: implicit SaveChanges transaction and explicit IDbContextTransaction

using System.Data;
using Microsoft.EntityFrameworkCore;

var builder = WebHost.CreateBuilder();
builder.Services.AddDbContext<ShopDbContext>(o =>
    o.UseSqlServer("Server=.;Database=Shop;Trusted_Connection=True;TrustServerCertificate=True"));
var app = builder.Build();

// Модель заказа / Order model
public class Order
{
    public int Id { get; set; }
    public decimal Total { get; set; }
    public List<OrderItem> Items { get; set; } = new();
}

public class OrderItem
{
    public int Id { get; set; }
    public int OrderId { get; set; }
    public string Product { get; set; } = "";
    public int Quantity { get; set; }
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public int Stock { get; set; } // остаток на складе / stock on hand
}

public class ShopDbContext : DbContext
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();
    public DbSet<Product> Products => Set<Product>();

    protected override void OnConfiguring(DbContextOptionsBuilder b)
        => b.UseSqlServer("Server=.;Database=Shop;Trusted_Connection=True;TrustServerCertificate=True");
}

// =========================================================
// 1. Неявная транзакция SaveChanges — всё или ничего
// 1. Implicit SaveChanges transaction — all or nothing
// =========================================================
public static async Task ImplicitExampleAsync(ShopDbContext db)
{
    var order = new Order { Total = 100m };
    order.Items.Add(new OrderItem { Product = "Pen", Quantity = 5 });

    db.Orders.Add(order);           // помечаем на вставку / marked for insert
    await db.SaveChangesAsync();    // одна транзакция под капотом / one implicit transaction

    // Если второй SaveChanges упадёт — первый уже зафиксирован и НЕ откатится!
    // If the second SaveChanges throws, the first is already committed and will NOT roll back!
}

// =========================================================
// 2. Явная транзакция: заказ + списание остатка — единое целое
// 2. Explicit transaction: order + stock decrement as one unit
// =========================================================
public static async Task<bool> PlaceOrderAsync(
    ShopDbContext db, int productId, int qty)
{
    // Запускаем явную транзакцию с уровнем изоляции ReadCommitted
    // Start explicit transaction with ReadCommitted isolation
    await using var tx = await db.Database
        .BeginTransactionAsync(IsolationLevel.ReadCommitted);

    try
    {
        var product = await db.Products
            .FirstAsync(p => p.Id == productId);

        if (product.Stock < qty)
        {
            // Недостаточно товара — откатываем и возвращаем false
            // Not enough stock — roll back and return false
            await tx.RollbackAsync();
            return false;
        }

        product.Stock -= qty;                       // уменьшаем остаток / decrement stock

        var order = new Order { Total = qty * 10m };
        order.Items.Add(new OrderItem
        {
            Product = product.Name,
            Quantity = qty
        });
        db.Orders.Add(order);

        await db.SaveChangesAsync();                // фиксируем изменения в транзакции
        await tx.CommitAsync();                     // подтверждаем / commit
        return true;
    }
    catch (Exception ex)
    {
        // Любая ошибка — полный откат / any error — full rollback
        await tx.RollbackAsync();
        Console.WriteLine($"Ошибка заказа / Order failed: {ex.Message}");
        throw;
    }
}

// =========================================================
// 3. ExecuteSqlRaw + SaveChanges в одной транзакции
// 3. Raw SQL + SaveChanges inside one transaction
// =========================================================
public static async Task MixedAsync(ShopDbContext db, int orderId)
{
    using var tx = await db.Database.BeginTransactionAsync();
    try
    {
        // «Сырой» SQL-запрос внутри транзакции
        // Raw SQL inside the transaction
        await db.Database.ExecuteSqlRawAsync(
            "UPDATE Orders SET Total = Total * 1.1 WHERE Id = {0}", orderId);

        // Изменения через Change Tracker в той же транзакции
        // Change Tracker changes in the same transaction
        var item = new OrderItem { OrderId = orderId, Product = "Surcharge", Quantity = 1 };
        db.OrderItems.Add(item);

        await db.SaveChangesAsync();
        await tx.CommitAsync();
    }
    catch
    {
        await tx.RollbackAsync();
        throw;
    }
}

// =========================================================
// 4. Паттерн ExecuteInTransaction (retry-safe) для Azure SQL
// 4. ExecuteInTransaction retry-safe pattern for Azure SQL
// =========================================================
public static async Task RetrySafeAsync(ShopDbContext db)
{
    var strategy = db.Database.CreateExecutionStrategy();
    await strategy.ExecuteAsync(async () =>
    {
        using var tx = await db.Database.BeginTransactionAsync();
        try
        {
            db.Orders.Add(new Order { Total = 42m });
            await db.SaveChangesAsync();
            await tx.CommitAsync();
        }
        catch
        {
            await tx.RollbackAsync();
            throw;
        }
    });
}
```

#### Best Practices
- Доверяйте неявной транзакции `SaveChanges` для простых CRUD; оборачивайте в явную транзакцию только когда логика требует нескольких `SaveChanges` или смешанных SQL-команд.
- Всегда используйте `using` для `IDbContextTransaction`, чтобы гарантировать освобождение ресурсов даже при исключении.
- Держите транзакции максимально короткими: внутри — только работа с БД, без HTTP-вызовов и тяжёлых вычислений.
- Задавайте уровень изоляции осознанно: `ReadCommitted` — разумный дефолт, `Serializable` — только при строгой необходимости.
- Используйте `CreateExecutionStrategy().ExecuteAsync(...)` для облачных БД (Azure SQL) — это добавит автоматические повторы при временных сбоях.

#### Best Practices (EN)
- Trust the implicit `SaveChanges` transaction for simple CRUD; wrap in an explicit transaction only when the logic requires multiple `SaveChanges` calls or mixed SQL commands.
- Always use `using` for `IDbContextTransaction` to guarantee resource release even on exception.
- Keep transactions as short as possible: inside — only database work, no HTTP calls and no heavy computation.
- Choose the isolation level deliberately: `ReadCommitted` is a sensible default, `Serializable` only when strictly required.
- Use `CreateExecutionStrategy().ExecuteAsync(...)` for cloud databases (Azure SQL) — it adds automatic retries on transient failures.

#### Частые ошибки / Common Mistakes
- **Несколько `SaveChanges` без общей транзакции** → первый успел сохраниться, второй упал — данные рассогласованы. Оборачивайте связанные вызовы в один `BeginTransaction`.
- **Забыли вызвать `Commit`** → транзакция тихо откатывается при `Dispose`, данные не сохраняются, ошибка не очевидна. Всегда явно вызывайте `Commit` после `SaveChanges`.
- **Держат транзакцию открытой во время долгой бизнес-логики или HTTP-вызова** → блокировки держатся минутами, деградация параллелизма. Выносите не-БД работу за пределы транзакции.
- **Используют `Serializable` «для надёжности» везде** → сильные блокировки и падение пропускной способности. Выбирайте минимальный достаточный уровень изоляции.
- **Микс `TransactionScope` без `TransactionScopeAsyncFlowOption.Enabled` в async-коде** → транзакция не «перетекает» между потоками, баги с `InvalidOperationException`. Включайте async-flow или используйте `IDbContextTransaction`.

#### Common Mistakes (EN)
- **Multiple `SaveChanges` without a shared transaction** → the first one persisted, the second threw — data is now inconsistent. Wrap related calls in one `BeginTransaction`.
- **Forgot to call `Commit`** → the transaction silently rolls back on `Dispose`, data is not saved, the error is not obvious. Always call `Commit` explicitly after `SaveChanges`.
- **Keeping the transaction open during long business logic or an HTTP call** → locks are held for minutes, concurrency degrades. Move non-DB work outside the transaction.
- **Using `Serializable` "for safety" everywhere** → heavy locks and throughput collapse. Pick the minimum sufficient isolation level.
- **Mixing `TransactionScope` without `TransactionScopeAsyncFlowOption.Enabled` in async code** → the transaction does not flow across threads, `InvalidOperationException` bugs appear. Enable async-flow or switch to `IDbContextTransaction`.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я знаю, что `SaveChanges` по умолчанию атомарный и оборачивает все изменения в одну транзакцию.
- [ ] Я умею запускать явную транзакцию через `Database.BeginTransactionAsync` с заданным уровнем изоляции.
- [ ] Я всегда вызываю `Commit` явно и помещаю транзакцию в `using`.
- [ ] Я обрабатываю исключения и вызываю `Rollback` при сбое.
- [ ] Я держу транзакции короткими и не делаю внутри них HTTP-вызовы.
- [ ] Я понимаю разницу между уровнями изоляции `ReadCommitted`, `RepeatableRead`, `Serializable`, `Snapshot`.
- [ ] Я знаю, как совместить `ExecuteSqlRaw` и `SaveChanges` в одной транзакции.
- [ ] I know that `SaveChanges` is atomic by default and wraps all changes in one transaction.
- [ ] I can start an explicit transaction via `Database.BeginTransactionAsync` with a chosen isolation level.
- [ ] I always call `Commit` explicitly and wrap the transaction in `using`.
- [ ] I handle exceptions and call `Rollback` on failure.
- [ ] I keep transactions short and never make HTTP calls inside them.
- [ ] I understand the difference between `ReadCommitted`, `RepeatableRead`, `Serializable`, `Snapshot` isolation levels.
- [ ] I know how to combine `ExecuteSqlRaw` and `SaveChanges` within a single transaction.

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/ef/core/saving/transactions](https://learn.microsoft.com/ef/core/saving/transactions)

---

[⬆ К модулю M12](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
