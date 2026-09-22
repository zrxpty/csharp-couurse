---
[← К уроку M12-L07](lesson-M12-L07-transactions-savechanges.md) | [⬆ К модулю M12](../README.md) | [Следующее ДЗ →](homework-M12-L08-raw-sql.md)
---

### Домашнее задание M12-L07: Транзакции, SaveChanges, IDbContextTransaction / Homework M12-L07: Transactions, SaveChanges, IDbContextTransaction

**Урок / Lesson:** M12-L07
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться отличать ситуации, в которых неявной транзакции `SaveChanges` достаточно, от случаев, требующих явной `IDbContextTransaction`; научиться запускать, фиксировать и откатывать транзакции, выбирать уровень изоляции, комбинировать `ExecuteSqlRaw` и `SaveChanges` и писать устойчивые к сбоям блоки с `CreateExecutionStrategy`. (EN) Learn to tell apart scenarios where the implicit `SaveChanges` transaction is sufficient from those that demand an explicit `IDbContextTransaction`; learn to start, commit and roll back transactions, choose an isolation level, combine `ExecuteSqlRaw` and `SaveChanges`, and write fault-tolerant blocks with `CreateExecutionStrategy`.

#### Связь с уроком / Connection to the lesson
(RU) Урок M12-L07 показал, что `SaveChanges` уже атомарен, а явная транзакция нужна при нескольких `SaveChanges`, смешивании сырого SQL с Change Tracker и при выборе уровня изоляции. Это ДЗ закрепляет именно эти сценарии: вы построите мини-сервис заказов, где заказ, списание остатка и списание бонусов обязаны применяться вместе или не применляться вовсе.
(EN) Lesson M12-L07 showed that `SaveChanges` is already atomic, while an explicit transaction is required for multiple `SaveChanges` calls, mixing raw SQL with the Change Tracker, and choosing an isolation level. This homework reinforces exactly those scenarios: you will build a mini ordering service where the order, stock decrement and bonus charge must all apply together or not at all.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы разрабатываете бэкенд мини-сервиса интернет-магазина «Acme Shop» на C# 12 / .NET 8 / EF Core 8. Каталог содержит товары (`Product`) с остатком на складе `Stock`. Клиенты размещают заказы (`Order`), состоящие из строк (`OrderItem`). У каждого клиента есть счёт бонусных баллов `BonusAccount`, которые можно тратить на скидки. При оформлении заказа должно произойти сразу несколько связанных изменений в базе: создать заказ со строками, уменьшить остатки по каждому товару, списать бонусные баллы с аккаунта клиента и записать аудит-событие. Если хотя бы одна операция падает — например, остаток товара отрицательный или баллов недостаточно — ничего из этого не должно сохраниться. Это классический сценарий «всё или ничего», и для него неявной транзакции `SaveChanges` уже недостаточно: потребуется несколько `SaveChanges`, сырой SQL для аудита и явный уровень изоляции, чтобы избежать состояния «гонки» при параллельных заказах одного и того же товара. Кроме того, сервис работает с Azure SQL, где возможны временные сбои, поэтому успешное решение обязано использовать `CreateExecutionStrategy` для автоматических повторов. Цель — построить код, который ведёт себя предсказуемо под нагрузкой, корректно откатывается при любых ошибках и не оставляет «полу-заказов» в базе. Вы должны не просто «сделать, чтобы работало», а понимать, почему каждая строка с транзакционной семантикой написана именно так, и уметь обосновать выбор уровня изоляции.

#### Что нужно сделать (пошагово)
1. Создайте новый проект `dotnet new console -n AcmeShop.Transactions -o AcmeShop.Transactions` с .NET 8 и добавьте пакеты: `dotnet add package Microsoft.EntityFrameworkCore.SqlServer`, `dotnet add package Microsoft.EntityFrameworkCore.InMemory` (для тестов), `dotnet add package Microsoft.EntityFrameworkCore.Design`. Убедитесь через `dotnet --version`, что используется SDK 8.x.
2. Опишите модели в одном файле `Models.cs`: `Product` (Id, Name, Stock, Price), `Order` (Id, CustomerId, Total, List<OrderItem> Items), `OrderItem` (Id, OrderId, Product, Quantity, UnitPrice), `BonusAccount` (Id, CustomerId, Balance), `AuditEvent` (Id, OccurredAt, Message). Используйте коллекционные выражения и `required`-члены C# 12 там, где это уместно.
3. Создайте `ShopDbContext` с `DbSet<>` для каждой сущности и строкой подключения через `OnConfiguring` (читайте её из переменной окружения `SHOP_DB` через `Environment.GetEnvironmentVariable`, fallback на `"Server=.;Database=AcmeShop;Trusted_Connection=True;TrustServerCertificate=True"`).
4. Напишите метод `MigrateAndSeedAsync`, который применяет `db.Database.MigrateAsync()` и, если таблица `Products` пуста, добавляет три товара и один бонусный аккаунт с балансом 500.
5. Реализуйте `PlaceOrderAsync(int customerId, int productId, int qty, int bonusToSpend)` — главный метод: он оформляет заказ, уменьшает остаток товара, списывает бонусы и пишет аудит-событие. Используйте `await using var tx = await db.Database.BeginTransactionAsync(IsolationLevel.ReadCommitted)` и обязательно `tx.CommitAsync()` в конце; в `catch` вызывайте `tx.RollbackAsync()` и `throw`.
6. Внутри метода проверьте бизнес-правила: `product.Stock >= qty`, `account.Balance >= bonusToSpend`; при нарушении откатывайте транзакцию и возвращайте результат-объект `PlaceOrderResult { Success, Reason }`, НЕ выбрасывая исключение для контролируемых отказов.
7. Добавьте метод `ApplySurchargeAsync(int orderId, decimal factor)`, который использует `ExecuteSqlRawAsync` для `UPDATE Orders SET Total = Total * {0}` внутри одной транзакции вместе с `db.OrderItems.Add` строки-«надбавки» и `SaveChanges`. Это упражнение на микс сырого SQL и Change Tracker.
8. Добавьте обёртку `PlaceOrderRetrySafeAsync`, которая использует `db.Database.CreateExecutionStrategy().ExecuteAsync(...)` для автоматических повторов при transient-ошибках Azure SQL. Внутри стратегии снова запускайте `BeginTransactionAsync` и всю логику.
9. Напишите хост-программу `Program.cs` с top-level statements: она инициализирует DI через `Host.CreateApplicationBuilder`, регистрирует `ShopDbContext`, вызывает сидинг, демонстрирует успешный заказ, заказ с превышением остатка (отказ), заказ с нехваткой бонусов (отказ) и применяет надбавку. Вывод должен быть на русском с пояснениями, какие строки и почему зафиксированы или откатились.
10. Запустите `dotnet build`, затем `dotnet run` и убедитесь, что в БД после отказов не появилось «полу-заказов»: напишите проверочный запрос `db.Orders.CountAsync()` и `db.AuditEvents.CountAsync()` после каждого шага и напечатайте значения.
11. Добавьте unit-тесты через xUnit в проект `AcmeShop.Transactions.Tests`, использующий `UseInMemoryDatabase`. Замечание: InMemory-провайдер **не поддерживает реальные транзакции** (`BeginTransactionAsync` бросит или проигнорирует), поэтому транзакционные тесты должны идти против SQL Server LocalDB или через `SQLite` с `Microsoft.Data.Sqlite` и `UseSqlite("DataSource=:memory:")` — именно там `IDbContextTransaction` работает честно. Опишите это ограничение в комментарии к тестам.
12. Сформируйте `README.md` с описанием архитектуры, схемой БД (можно ASCII) и инструкцией запуска. Опишите, какой уровень изоляции выбран и почему именно `ReadCommitted`, а не `Serializable`.

#### Требования к решению
- Целевая среда: C# 12, .NET 8, EF Core 8. Используйте top-level statements, коллекционные выражения (`new()` / `[]`), `required`, pattern matching, raw string literals для SQL.
- Все публичные методы работы с БД — `async` и возвращают `Task<T>`; никаких синхронных `.Result` / `.Wait()`. Используйте `await using` для `IDbContextTransaction` и `CancellationToken` в сигнатурах.
- Каждое место, где открывается транзакция, обязано содержать `try / catch` с `RollbackAsync` и `throw` (или возвращать `PlaceOrderResult` без исключения для контролируемых бизнес-отказов — на ваше усмотрение, но не «глотать» исключения).
- Запрещено вызывать несколько `SaveChanges` без общей транзакции там, где они логически едины. Если в коде два `SaveChanges` подряд без `BeginTransaction` — это баг по критериям приёмки.
- Все вызовы `ExecuteSqlRaw`/`ExecuteSqlRawAsync` должны использовать **параметризацию** через `{0}`/`{1}` или интерполяцию `ExecuteSqlInterpolatedAsync($"UPDATE ... WHERE Id = {orderId}")`. Конкатенация строк в SQL — автоматический незачёт.
- Уровень изоляции указывается явно хотя бы в одном методе (`ReadCommitted` или выше) с комментарием-обоснованием.
- Должен быть представлен retry-safe вариант через `CreateExecutionStrategy().ExecuteAsync`.
- Сервис должен логировать (через `ILogger<T>`) ключевые точки: начало транзакции, успешный commit, rollback с причиной.
- Тесты: хотя бы один зелёный тест на успешный заказ и один — на откат при нехватке остатка, оба против SQLite in-memory.

#### Тонкости и подводные камни
- **`SaveChanges` уже атомарен.** Не оборачивайте одиночный `SaveChanges` в `BeginTransaction` «для надёжности» — это лишний round-trip и блокировка. Явная транзакция нужна только когда `SaveChanges` несколько либо есть сырой SQL.
- **Забытый `Commit` = тихий откат.** Если вы положили `IDbContextTransaction` в `using`, но забыли `CommitAsync`, при выходе из блока транзакция откатится молча. Данные не сохранятся, исключения не будет — классический «тихий баг». Всегда явно вызывайте `Commit`.
- **`await using`, а не `using`.** Для асинхронного `Dispose` используйте `await using var tx = ...`, иначе `Dispose` блокирует поток.
- **Несколько `SaveChanges` без общей транзакции.** Если первый `SaveChanges` успел зафиксироваться, а второй упал — данные рассогласованы. Это главная причина оборачивать связанную логику в одну `BeginTransaction`.
- **Уровни изоляции.** `ReadCommitted` — разумный дефолт для заказов; `Serializable` «для надёжности» везде даст блокировки и просадит пропускную способность. `Snapshot` требует настройки в SQL Server (`ALLOW_SNAPSHOT_ISOLATION ON`). Выбирайте минимальный достаточный уровень.
- **InMemory-провайдер не поддерживает транзакции.** Проверяйте транзакционную логику только против реального движка: SQL Server или SQLite. Иначе тест «зелёный», а баг скрыт.
- **`TransactionScope` в async.** Если всё же используете `TransactionScope`, обязательно `TransactionScopeAsyncFlowOption.Enabled`, иначе транзакция не перетечёт между потоками и будет `InvalidOperationException`. В современном коде предпочтительнее `IDbContextTransaction`.
- **Не держите транзакцию открытой во время HTTP-вызовов.** Внутри транзакции — только работа с БД; внешние вызовы выносите наружу, иначе блокировки держатся минутами.
- **`ExecuteSqlRaw` параметризация.** Строка вида `"... WHERE Id = " + orderId` — это SQL-injection risk и нарушение best practices. Только `{0}` или interpolated `$"..."` через `ExecuteSqlInterpolated`.
- **Повторы и идемпотентность.** `CreateExecutionStrategy` может выполнить блок дважды; убедитесь, что логика идемпотентна или что повтор после частичного коммита безопасен (часто — нельзя коммитить внутри стратегии без повторного `BeginTransaction`).

#### Критерии приёмки
- [ ] Проект собирается `dotnet build` без warning-ов уровня error и без ошибок на .NET 8.
- [ ] Использованы top-level statements, коллекционные выражения, `required`, raw string literals там, где это уместно.
- [ ] Все БД-методы `async`, используют `await using` для `IDbContextTransaction` и принимают `CancellationToken`.
- [ ] Везде, где есть несколько `SaveChanges` или микс с `ExecuteSqlRaw`, открыта общая явная транзакция.
- [ ] `CommitAsync` вызывается явно ровно один раз на ветку успеха.
- [ ] В `catch` вызывается `RollbackAsync` и исключение пробрасывается (или возвращается `PlaceOrderResult` без «глотания»).
- [ ] Бизнес-проверки остатков и бонусов выполнены до `SaveChanges` и приводят к откату без исключения.
- [ ] `ExecuteSqlRaw`/`ExecuteSqlInterpolated` параметризован; нет конкатенации строк в SQL.
- [ ] Уровень изоляции указан явно хотя бы в одном методе с комментарием-обоснованием.
- [ ] Реализован retry-safe вариант через `CreateExecutionStrategy().ExecuteAsync`.
- [ ] Логирование через `ILogger` ключевых точек (начало, commit, rollback с причиной).
- [ ] После отказа (`Stock < qty`) в БД нет нового заказа, остаток не уменьшен, аудит не записан — есть assertion в тесте.
- [ ] Unit-тесты против SQLite in-memory: успешный заказ зелёный, откат при нехватке остатка зелёный.
- [ ] `README.md` описывает схему БД, выбор уровня изоляции и инструкцию запуска.
- [ ] В комментариях к тестам отмечено, почему InMemory-провайдер неприменим для транзакций.

#### Подсказки (без прямого ответа)
- Подумайте, какой `IsolationLevel` защитит от «двойного списания» остатка при параллельных заказах, и почему `ReadCommitted` всё же достаточно при использовании `UPDATE Products SET Stock = Stock - ... WHERE Id = ... AND Stock >= ...` (атомарный декремент).
- В `CreateExecutionStrategy` повтор выполнит весь делегат заново — значит, `BeginTransactionAsync` должен быть **внутри** делегата, а не снаружи.
- Для SQLite in-memory используйте `Microsoft.Data.Sqlite` + общий `DbConnection` между экземплярами контекста, иначе база «умрёт» при закрытии соединения.
- Не забудьте, что `ExecuteSqlInterpolated` сам формирует параметр из `{orderId}` — это не конкатенация.
- В `catch` после `RollbackAsync` обязательно `throw`, иначе вы «проглотите» реальную причину и клиент не узнает о сбое.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 / EF Core 8
// AcmeShop.Transactions — заказы, остатки, бонусы, аудит — всё в одной транзакции
// AcmeShop.Transactions — orders, stock, bonuses, audit — all in one transaction

using System.Data;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;

var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddDbContext<ShopDbContext>(o =>
    o.UseSqlServer(Environment.GetEnvironmentVariable("SHOP_DB")
        ?? "Server=.;Database=AcmeShop;Trusted_Connection=True;TrustServerCertificate=True"));
builder.Services.AddSingleton<OrderService>();
var app = builder.Build();

await using (var scope = app.Services.CreateAsyncScope())
{
    var svc = scope.ServiceProvider.GetRequiredService<OrderService>();
    await svc.MigrateAndSeedAsync(app.Services.GetRequiredService<IHostEnvironment>().ApplicationStopping);

    var ok1 = await svc.PlaceOrderAsync(customerId: 1, productId: 1, qty: 2, bonusToSpend: 50,
        app.Services.GetRequiredService<IHostEnvironment>().ApplicationStopping);
    Console.WriteLine($"Заказ 1 / Order 1: {ok1.Success} ({ok1.Reason})");

    var ok2 = await svc.PlaceOrderAsync(customerId: 1, productId: 1, qty: 999, bonusToSpend: 0,
        app.Services.GetRequiredService<IHostEnvironment>().ApplicationStopping);
    Console.WriteLine($"Заказ 2 / Order 2: {ok2.Success} ({ok2.Reason})");

    await svc.ApplySurchargeAsync(orderId: 1, factor: 1.1m,
        app.Services.GetRequiredService<IHostEnvironment>().ApplicationStopping);
}

// ---------- Модели / Models ----------
public class Product
{
    public int Id { get; set; }
    public required string Name { get; set; }
    public int Stock { get; set; }
    public decimal Price { get; set; }
}

public class Order
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
    public decimal Total { get; set; }
    public List<OrderItem> Items { get; set; } = [];
}

public class OrderItem
{
    public int Id { get; set; }
    public int OrderId { get; set; }
    public required string Product { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}

public class BonusAccount
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
    public decimal Balance { get; set; }
}

public class AuditEvent
{
    public int Id { get; set; }
    public DateTime OccurredAt { get; set; } = DateTime.UtcNow;
    public required string Message { get; set; }
}

public class PlaceOrderResult(bool success, string reason)
{
    public bool Success { get; } = success;
    public string Reason { get; } = reason;
}

// ---------- DbContext ----------
public class ShopDbContext : DbContext
{
    public ShopDbContext(DbContextOptions<ShopDbContext> o) : base(o) { }
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();
    public DbSet<BonusAccount> BonusAccounts => Set<BonusAccount>();
    public DbSet<AuditEvent> AuditEvents => Set<AuditEvent>();
}

// ---------- Сервис заказов / Order service ----------
public class OrderService(ShopDbContext db, ILogger<OrderService> log)
{
    public async Task MigrateAndSeedAsync(CancellationToken ct)
    {
        await db.Database.MigrateAsync(ct);
        if (!await db.Products.AnyAsync(ct))
        {
            db.Products.AddRange(
                new Product { Name = "Pen",   Stock = 100, Price = 10m },
                new Product { Name = "Notebook", Stock = 50, Price = 25m },
                new Product { Name = "Mug",  Stock = 20, Price = 15m });
            db.BonusAccounts.Add(new BonusAccount { CustomerId = 1, Balance = 500m });
            await db.SaveChangesAsync(ct);
        }
    }

    // Главный метод: заказ + списание остатка + списание бонусов + аудит — единая транзакция
    // Main method: order + stock decrement + bonus charge + audit — single transaction
    public async Task<PlaceOrderResult> PlaceOrderAsync(
        int customerId, int productId, int qty, int bonusToSpend, CancellationToken ct)
    {
        // Retry-safe обёртка для Azure SQL (transient-сбои)
        // Retry-safe wrapper for Azure SQL (transient failures)
        var strategy = db.Database.CreateExecutionStrategy();
        return await strategy.ExecuteAsync(async () =>
        {
            // ReadCommitted — разумный дефолт: не читаем «грязные» данные, но не блокируем всё подряд.
            // ReadCommitted — sensible default: no dirty reads, but no global locks.
            await using var tx = await db.Database
                .BeginTransactionAsync(IsolationLevel.ReadCommitted, ct);
            try
            {
                // Атомарный декремент остатка с проверкой: UPDATE ... WHERE Stock >= qty
                // Atomic stock decrement with guard: UPDATE ... WHERE Stock >= qty
                var updated = await db.Products
                    .Where(p => p.Id == productId && p.Stock >= qty)
                    .ExecuteUpdateAsync(s => s.SetProperty(p => p.Stock, p => p.Stock - qty), ct);

                if (updated == 0)
                    return new PlaceOrderResult(false, "Недостаточно остатка / Insufficient stock");

                var product = await db.Products.FirstAsync(p => p.Id == productId, ct);

                // Списываем бонусы атомарно тем же приёмом
                // Charge bonuses atomically using the same pattern
                if (bonusToSpend > 0)
                {
                    var charged = await db.BonusAccounts
                        .Where(a => a.CustomerId == customerId && a.Balance >= bonusToSpend)
                        .ExecuteUpdateAsync(s => s.SetProperty(a => a.Balance, a => a.Balance - bonusToSpend), ct);
                    if (charged == 0)
                        return new PlaceOrderResult(false, "Недостаточно бонусов / Insufficient bonuses");
                }

                var order = new Order
                {
                    CustomerId = customerId,
                    Total = qty * product.Price
                };
                order.Items.Add(new OrderItem
                {
                    Product = product.Name,
                    Quantity = qty,
                    UnitPrice = product.Price
                });
                db.Orders.Add(order);

                db.AuditEvents.Add(new AuditEvent
                {
                    Message = $"Заказ для клиента / Order for customer {customerId}: {product.Name} x{qty}"
                });

                await db.SaveChangesAsync(ct);
                await tx.CommitAsync(ct); // явный коммит / explicit commit
                log.LogInformation("Заказ сохранён / Order committed: OrderId={OrderId}", order.Id);
                return new PlaceOrderResult(true, "OK");
            }
            catch (Exception ex)
            {
                await tx.RollbackAsync(ct);
                log.LogWarning(ex, "Откат заказа / Order rolled back");
                throw; // не глотаем / do not swallow
            }
        });
    }

    // Микс ExecuteSqlInterpolated + Change Tracker в одной транзакции
    // Mix of ExecuteSqlInterpolated + Change Tracker in one transaction
    public async Task ApplySurchargeAsync(int orderId, decimal factor, CancellationToken ct)
    {
        await using var tx = await db.Database.BeginTransactionAsync(ct);
        try
        {
            // Параметризованный «сырой» SQL — НЕ конкатенация
            // Parameterized raw SQL — NOT concatenation
            await db.Database.ExecuteSqlInterpolatedAsync(
                $"UPDATE Orders SET Total = Total * {factor} WHERE Id = {orderId}", ct);

            db.OrderItems.Add(new OrderItem
            {
                OrderId = orderId,
                Product = "Surcharge",
                Quantity = 1,
                UnitPrice = 0m
            });

            await db.SaveChangesAsync(ct);
            await tx.CommitAsync(ct);
        }
        catch
        {
            await tx.RollbackAsync(ct);
            throw;
        }
    }
}
```

Разбор по строкам. Метод `PlaceOrderAsync` открывает явную транзакцию с `IsolationLevel.ReadCommitted` — это уровень по умолчанию для SQL Server, и он достаточен здесь, потому что мы не читаем остаток «обычным» `SELECT`, а сразу делаем атомарный `ExecuteUpdateAsync` с предикатом `Stock >= qty`. Этот приём снимает классическую гонку: два параллельных заказа не «прочитают» один и тот же остаток и не продадут товар дважды — один из `ExecuteUpdate` просто обновит 0 строк и вернёт контролируемый отказ без исключения. `CreateExecutionStrategy().ExecuteAsync` оборачивает всю логику, включая `BeginTransactionAsync`: это обязательно, потому что при повторе после transient-сбоя делегат стартует заново и должен открыть новую транзакцию. Внутри `try` мы используем `await using var tx` — это корректный асинхронный `Dispose`, который в отличие от синхронного `using` не блокирует поток. На ветке успеха — ровно один `CommitAsync`: если забыть его, при выходе из `using` транзакция откатится молча (классический «тихий баг» из урока). В `catch` мы делаем `RollbackAsync` и обязательно `throw`, чтобы не «глотать» реальную причину сбоя и дать вызывающей стороне шанс отреагировать. Для контролируемых бизнес-отказов (нехватка остатка или бонусов) мы не выбрасываем исключение, а возвращаем `PlaceOrderResult(false, reason)` — это отделяет ожидаемые отказы от инфраструктурных сбоев. Метод `ApplySurchargeAsync` демонстрирует вторую ключевую тему урока: микс сырого SQL (`ExecuteSqlInterpolatedAsync`) и Change Tracker (`db.OrderItems.Add` + `SaveChanges`) в одной транзакции. Параметризация через interpolated handler формирует настоящий SQL-параметр из `{factor}` и `{orderId}` — это безопасно и не является конкатенацией. `SaveChanges` внутри уже существующей явной транзакции «вливается» в неё, а не создаёт свою неявную. Вся конструкция охвачена `try/catch/rollback/throw` по той же схеме. Коллекционные выражения (`Items = []`) и `required`-члены — это возможности C# 12, которые делают модель лаконичной и защищают от забытой инициализации.

#### Задания на углубление (бонус)
1. Реализуйте параллельный стресс-тест: 100 одновременных `PlaceOrderAsync` на один товар со остатком 10 и `qty=1`; убедитесь, что ровно 10 заказов успешны, а 90 — отклонены. Объясните, почему `ReadCommitted` + атомарный декремент достаточно, а `Serializable` здесь избыточен.
2. Переведите `PlaceOrderAsync` на `TransactionScope` с `TransactionScopeAsyncFlowOption.Enabled` и сравните читаемость и поведение с `IDbContextTransaction`. Опишите, в чём плюсы и минусы каждого подхода.
3. Добавьте уровень изоляции `Snapshot` и явно настройте `ALTER DATABASE AcmeShop SET ALLOW_SNAPSHOT_ISOLATION ON`. Измерьте разницу в блокировках через `sys.dm_tran_locks` при параллельных заказах.
4. Реализуйте сагу-компенсацию: если аудит-событие упало после коммита заказа (имитируйте через отдельный `SaveChanges` вне транзакции), компенсируйте предыдущую операцию и запишите компенсирующее событие. Объясните, почему это уже не транзакция ACID, а паттерн Saga.

---

## Statement in English / Постановка на английском

#### Context & motivation
You are building the back end of a mini online-shop "Acme Shop" on C# 12 / .NET 8 / EF Core 8. The catalogue holds products (`Product`) with an on-hand `Stock` value. Customers place orders (`Order`) that consist of line items (`OrderItem`). Every customer has a `BonusAccount` holding bonus points that can be spent on discounts. Placing an order must perform several tightly coupled changes in the database at once: create the order with its line items, decrement the stock for every product, charge bonus points from the customer's account, and append an audit event. If any single operation fails — for example the stock would go negative or the bonus balance is insufficient — none of the changes may persist. This is a textbook "all or nothing" scenario, and the implicit `SaveChanges` transaction is no longer enough: you need several `SaveChanges` calls, raw SQL for the audit, and an explicit isolation level to avoid a race when two customers order the last item simultaneously. On top of that, the service runs against Azure SQL, where transient failures are expected, so a correct solution must wrap the critical block in `CreateExecutionStrategy` to get automatic retries. The goal is to build code that behaves predictably under load, rolls back cleanly on every error, and never leaves half-orders in the database. You should not merely "make it work"; you should understand why every line with transactional semantics is written the way it is, and be able to justify the chosen isolation level out loud.

#### What to do step by step
1. Create a new project `dotnet new console -n AcmeShop.Transactions -o AcmeShop.Transactions` on .NET 8 and add the packages: `dotnet add package Microsoft.EntityFrameworkCore.SqlServer`, `dotnet add package Microsoft.EntityFrameworkCore.InMemory` (for throw-away tests), `dotnet add package Microsoft.EntityFrameworkCore.Design`. Verify with `dotnet --version` that the SDK is 8.x.
2. Describe the models in a single `Models.cs`: `Product` (Id, Name, Stock, Price), `Order` (Id, CustomerId, Total, List<OrderItem> Items), `OrderItem` (Id, OrderId, Product, Quantity, UnitPrice), `BonusAccount` (Id, CustomerId, Balance), `AuditEvent` (Id, OccurredAt, Message). Use collection expressions and `required` members from C# 12 where they fit.
3. Create `ShopDbContext` with a `DbSet<>` for each entity and a connection string in `OnConfiguring` read from the `SHOP_DB` environment variable via `Environment.GetEnvironmentVariable`, falling back to `"Server=.;Database=AcmeShop;Trusted_Connection=True;TrustServerCertificate=True"`.
4. Write a `MigrateAndSeedAsync` method that runs `db.Database.MigrateAsync()` and, if the `Products` table is empty, seeds three products plus one bonus account with a balance of 500.
5. Implement `PlaceOrderAsync(int customerId, int productId, int qty, int bonusToSpend)` — the main method: it creates the order, decrements product stock, charges bonuses, and writes the audit event. Use `await using var tx = await db.Database.BeginTransactionAsync(IsolationLevel.ReadCommitted)` and an explicit `tx.CommitAsync()` at the end; in the `catch` block call `tx.RollbackAsync()` and `throw`.
6. Inside the method, enforce business rules: `product.Stock >= qty`, `account.Balance >= bonusToSpend`; on violation roll back the transaction and return a `PlaceOrderResult { Success, Reason }` object instead of throwing an exception for these controlled refusals.
7. Add a method `ApplySurchargeAsync(int orderId, decimal factor)` that uses `ExecuteSqlRawAsync` to `UPDATE Orders SET Total = Total * {0}` inside a single transaction together with `db.OrderItems.Add` of a "surcharge" line and `SaveChanges`. This is the exercise on mixing raw SQL and the Change Tracker.
8. Add a wrapper `PlaceOrderRetrySafeAsync` that uses `db.Database.CreateExecutionStrategy().ExecuteAsync(...)` to get automatic retries on Azure SQL transient errors. Inside the strategy, start `BeginTransactionAsync` again and run the whole body.
9. Write a host `Program.cs` with top-level statements: initialise DI via `Host.CreateApplicationBuilder`, register `ShopDbContext`, run seeding, demonstrate a successful order, an over-stock refusal, an insufficient-bonus refusal, and the surcharge. Output must be in English with explanations of which rows were committed and which were rolled back.
10. Run `dotnet build`, then `dotnet run`, and verify that the database contains no half-orders after refusals: write a check `db.Orders.CountAsync()` and `db.AuditEvents.CountAsync()` after each step and print the values.
11. Add xUnit tests in an `AcmeShop.Transactions.Tests` project using `UseInMemoryDatabase`. Note that the InMemory provider **does not support real transactions** (`BeginTransactionAsync` will throw or be ignored), so transactional tests must run against SQL Server LocalDB or against SQLite via `Microsoft.Data.Sqlite` and `UseSqlite("DataSource=:memory:")` — only there does `IDbContextTransaction` behave honestly. Document this limitation in a test comment.
12. Produce a `README.md` describing the architecture, the database schema (ASCII is fine), and the run instructions. Explain which isolation level you chose and why it is `ReadCommitted` rather than `Serializable`.

#### Requirements
- Target environment: C# 12, .NET 8, EF Core 8. Use top-level statements, collection expressions (`new()` / `[]`), `required`, pattern matching, raw string literals for SQL where appropriate.
- Every public database method is `async` and returns `Task<T>`; no synchronous `.Result` / `.Wait()`. Use `await using` for `IDbContextTransaction` and accept a `CancellationToken` in signatures.
- Every place that opens a transaction must contain a `try / catch` with `RollbackAsync` and `throw` (or return a `PlaceOrderResult` for controlled business refusals, at your discretion — but never swallow exceptions).
- Calling several `SaveChanges` without a shared transaction where they are logically one unit is forbidden. Two consecutive `SaveChanges` without `BeginTransaction` is a bug by the acceptance criteria.
- All `ExecuteSqlRaw` / `ExecuteSqlRawAsync` calls must use parameterisation via `{0}` / `{1}` or interpolation through `ExecuteSqlInterpolatedAsync($"UPDATE ... WHERE Id = {orderId}")`. String concatenation in SQL is an automatic fail.
- The isolation level is stated explicitly in at least one method (`ReadCommitted` or higher) with a justifying comment.
- A retry-safe variant through `CreateExecutionStrategy().ExecuteAsync` must be present.
- The service must log (via `ILogger<T>`) the key points: transaction start, successful commit, rollback with reason.
- Tests: at least one green test for a successful order and one for a rollback on insufficient stock, both against SQLite in-memory.

#### Pitfalls
- **`SaveChanges` is already atomic.** Do not wrap a single `SaveChanges` in `BeginTransaction` "for safety" — it is an extra round-trip and extra locking. Explicit transactions are only needed when there are several `SaveChanges` calls or raw SQL.
- **A forgotten `Commit` is a silent rollback.** If you put `IDbContextTransaction` in a `using` but forget `CommitAsync`, the transaction rolls back silently on scope exit. No data is saved, no exception is thrown — a classic "silent bug". Always call `Commit` explicitly.
- **`await using`, not `using`.** For asynchronous `Dispose` use `await using var tx = ...`, otherwise `Dispose` blocks a thread.
- **Several `SaveChanges` without a shared transaction.** If the first `SaveChanges` committed and the second threw, the data is now inconsistent. This is the main reason to wrap related logic in one `BeginTransaction`.
- **Isolation levels.** `ReadCommitted` is a sensible default for orders; `Serializable` "for safety" everywhere brings heavy locks and collapses throughput. `Snapshot` requires SQL Server configuration (`ALLOW_SNAPSHOT_ISOLATION ON`). Pick the lowest sufficient level.
- **The InMemory provider does not support transactions.** Verify transactional logic only against a real engine: SQL Server or SQLite. Otherwise the test is green while the bug is hidden.
- **`TransactionScope` in async code.** If you do use `TransactionScope`, you must set `TransactionScopeAsyncFlowOption.Enabled`, otherwise the transaction will not flow across threads and you will get `InvalidOperationException`. In modern code `IDbContextTransaction` is preferred.
- **Do not keep the transaction open during HTTP calls.** Inside the transaction — only database work; external calls must go outside, otherwise locks are held for minutes.
- **`ExecuteSqlRaw` parameterisation.** A string like `"... WHERE Id = " + orderId` is a SQL-injection risk and a violation of best practices. Only `{0}` or interpolated `$"..."` through `ExecuteSqlInterpolated`.
- **Retries and idempotency.** `CreateExecutionStrategy` may run the block twice; make sure the logic is idempotent or that a retry after a partial commit is safe (typically you must not commit inside the strategy without a fresh `BeginTransaction`).

#### Acceptance criteria
- [ ] The project builds with `dotnet build` with no error-level warnings and no errors on .NET 8.
- [ ] Top-level statements, collection expressions, `required`, and raw string literals are used where appropriate.
- [ ] All database methods are `async`, use `await using` for `IDbContextTransaction`, and accept a `CancellationToken`.
- [ ] Everywhere there are several `SaveChanges` or a mix with `ExecuteSqlRaw`, a shared explicit transaction is open.
- [ ] `CommitAsync` is called explicitly exactly once on the success branch.
- [ ] In `catch`, `RollbackAsync` is called and the exception is rethrown (or a `PlaceOrderResult` is returned without swallowing).
- [ ] Business checks for stock and bonuses run before `SaveChanges` and lead to a rollback without an exception.
- [ ] `ExecuteSqlRaw` / `ExecuteSqlInterpolated` is parameterised; no string concatenation in SQL.
- [ ] The isolation level is stated explicitly in at least one method with a justifying comment.
- [ ] A retry-safe variant through `CreateExecutionStrategy().ExecuteAsync` is implemented.
- [ ] Logging through `ILogger` of the key points (start, commit, rollback with reason).
- [ ] After a refusal (`Stock < qty`) the database has no new order, the stock is not decremented, and the audit is not written — asserted in a test.
- [ ] Unit tests against SQLite in-memory: a successful order is green, a rollback on insufficient stock is green.
- [ ] `README.md` describes the database schema, the chosen isolation level, and the run instructions.
- [ ] A test comment explains why the InMemory provider is unsuitable for transactions.

#### Hints (without the direct answer)
- Think about which `IsolationLevel` would protect against a "double sale" of stock under concurrent orders, and why `ReadCommitted` is still enough when you use `UPDATE Products SET Stock = Stock - ... WHERE Id = ... AND Stock >= ...` (atomic decrement).
- In `CreateExecutionStrategy` the retry re-runs the entire delegate, so `BeginTransactionAsync` must be **inside** the delegate, not outside.
- For SQLite in-memory use `Microsoft.Data.Sqlite` plus a shared `DbConnection` between context instances, otherwise the database dies when the connection closes.
- Remember that `ExecuteSqlInterpolated` itself turns `{orderId}` into a real parameter — it is not concatenation.
- In `catch` after `RollbackAsync` always `throw`, otherwise you swallow the real cause and the caller never learns about the failure.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 / EF Core 8
// AcmeShop.Transactions — orders, stock, bonuses, audit — all in one transaction

using System.Data;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;

var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddDbContext<ShopDbContext>(o =>
    o.UseSqlServer(Environment.GetEnvironmentVariable("SHOP_DB")
        ?? "Server=.;Database=AcmeShop;Trusted_Connection=True;TrustServerCertificate=True"));
builder.Services.AddSingleton<OrderService>();
var app = builder.Build();

await using (var scope = app.Services.CreateAsyncScope())
{
    var svc = scope.ServiceProvider.GetRequiredService<OrderService>();
    await svc.MigrateAndSeedAsync(app.Services.GetRequiredService<IHostEnvironment>().ApplicationStopping);

    var ok1 = await svc.PlaceOrderAsync(customerId: 1, productId: 1, qty: 2, bonusToSpend: 50,
        app.Services.GetRequiredService<IHostEnvironment>().ApplicationStopping);
    Console.WriteLine($"Order 1: {ok1.Success} ({ok1.Reason})");

    var ok2 = await svc.PlaceOrderAsync(customerId: 1, productId: 1, qty: 999, bonusToSpend: 0,
        app.Services.GetRequiredService<IHostEnvironment>().ApplicationStopping);
    Console.WriteLine($"Order 2: {ok2.Success} ({ok2.Reason})");

    await svc.ApplySurchargeAsync(orderId: 1, factor: 1.1m,
        app.Services.GetRequiredService<IHostEnvironment>().ApplicationStopping);
}

// ---------- Models ----------
public class Product
{
    public int Id { get; set; }
    public required string Name { get; set; }
    public int Stock { get; set; }
    public decimal Price { get; set; }
}

public class Order
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
    public decimal Total { get; set; }
    public List<OrderItem> Items { get; set; } = [];
}

public class OrderItem
{
    public int Id { get; set; }
    public int OrderId { get; set; }
    public required string Product { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}

public class BonusAccount
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
    public decimal Balance { get; set; }
}

public class AuditEvent
{
    public int Id { get; set; }
    public DateTime OccurredAt { get; set; } = DateTime.UtcNow;
    public required string Message { get; set; }
}

public class PlaceOrderResult(bool success, string reason)
{
    public bool Success { get; } = success;
    public string Reason { get; } = reason;
}

// ---------- DbContext ----------
public class ShopDbContext : DbContext
{
    public ShopDbContext(DbContextOptions<ShopDbContext> o) : base(o) { }
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();
    public DbSet<BonusAccount> BonusAccounts => Set<BonusAccount>();
    public DbSet<AuditEvent> AuditEvents => Set<AuditEvent>();
}

// ---------- Order service ----------
public class OrderService(ShopDbContext db, ILogger<OrderService> log)
{
    public async Task MigrateAndSeedAsync(CancellationToken ct)
    {
        await db.Database.MigrateAsync(ct);
        if (!await db.Products.AnyAsync(ct))
        {
            db.Products.AddRange(
                new Product { Name = "Pen",   Stock = 100, Price = 10m },
                new Product { Name = "Notebook", Stock = 50, Price = 25m },
                new Product { Name = "Mug",  Stock = 20, Price = 15m });
            db.BonusAccounts.Add(new BonusAccount { CustomerId = 1, Balance = 500m });
            await db.SaveChangesAsync(ct);
        }
    }

    // Order + stock decrement + bonus charge + audit — single transaction
    public async Task<PlaceOrderResult> PlaceOrderAsync(
        int customerId, int productId, int qty, int bonusToSpend, CancellationToken ct)
    {
        // Retry-safe wrapper for Azure SQL (transient failures)
        var strategy = db.Database.CreateExecutionStrategy();
        return await strategy.ExecuteAsync(async () =>
        {
            // ReadCommitted: no dirty reads, no global locks — sensible default.
            await using var tx = await db.Database
                .BeginTransactionAsync(IsolationLevel.ReadCommitted, ct);
            try
            {
                // Atomic stock decrement with guard: UPDATE ... WHERE Stock >= qty
                var updated = await db.Products
                    .Where(p => p.Id == productId && p.Stock >= qty)
                    .ExecuteUpdateAsync(s => s.SetProperty(p => p.Stock, p => p.Stock - qty), ct);

                if (updated == 0)
                    return new PlaceOrderResult(false, "Insufficient stock");

                var product = await db.Products.FirstAsync(p => p.Id == productId, ct);

                // Charge bonuses atomically using the same pattern
                if (bonusToSpend > 0)
                {
                    var charged = await db.BonusAccounts
                        .Where(a => a.CustomerId == customerId && a.Balance >= bonusToSpend)
                        .ExecuteUpdateAsync(s => s.SetProperty(a => a.Balance, a => a.Balance - bonusToSpend), ct);
                    if (charged == 0)
                        return new PlaceOrderResult(false, "Insufficient bonuses");
                }

                var order = new Order
                {
                    CustomerId = customerId,
                    Total = qty * product.Price
                };
                order.Items.Add(new OrderItem
                {
                    Product = product.Name,
                    Quantity = qty,
                    UnitPrice = product.Price
                });
                db.Orders.Add(order);

                db.AuditEvents.Add(new AuditEvent
                {
                    Message = $"Order for customer {customerId}: {product.Name} x{qty}"
                });

                await db.SaveChangesAsync(ct);
                await tx.CommitAsync(ct); // explicit commit
                log.LogInformation("Order committed: OrderId={OrderId}", order.Id);
                return new PlaceOrderResult(true, "OK");
            }
            catch (Exception ex)
            {
                await tx.RollbackAsync(ct);
                log.LogWarning(ex, "Order rolled back");
                throw; // do not swallow
            }
        });
    }

    // Mix of ExecuteSqlInterpolated + Change Tracker in one transaction
    public async Task ApplySurchargeAsync(int orderId, decimal factor, CancellationToken ct)
    {
        await using var tx = await db.Database.BeginTransactionAsync(ct);
        try
        {
            // Parameterized raw SQL — NOT concatenation
            await db.Database.ExecuteSqlInterpolatedAsync(
                $"UPDATE Orders SET Total = Total * {factor} WHERE Id = {orderId}", ct);

            db.OrderItems.Add(new OrderItem
            {
                OrderId = orderId,
                Product = "Surcharge",
                Quantity = 1,
                UnitPrice = 0m
            });

            await db.SaveChangesAsync(ct);
            await tx.CommitAsync(ct);
        }
        catch
        {
            await tx.RollbackAsync(ct);
            throw;
        }
    }
}
```

Line-by-line walk-through. The `PlaceOrderAsync` method opens an explicit transaction with `IsolationLevel.ReadCommitted` — the default level for SQL Server and a sufficient one here, because we do not read the stock with a plain `SELECT`; instead we issue an atomic `ExecuteUpdateAsync` with the predicate `Stock >= qty`. This pattern removes the classic race: two concurrent orders will not both read the same stock and oversell the item — one of the `ExecuteUpdate` calls simply updates zero rows and returns a controlled refusal without an exception. `CreateExecutionStrategy().ExecuteAsync` wraps the whole body, including `BeginTransactionAsync`: this is mandatory, because on a retry after a transient failure the delegate starts over and must open a fresh transaction. Inside `try` we use `await using var tx` — the correct asynchronous `Dispose` that, unlike a synchronous `using`, does not block a thread. On the success branch there is exactly one `CommitAsync`: forgetting it would silently roll the transaction back on `using` exit (the classic "silent bug" from the lesson). In `catch` we call `RollbackAsync` and always `throw`, so we never swallow the real cause and the caller gets a chance to react. For controlled business refusals (insufficient stock or bonuses) we do not throw; we return `PlaceOrderResult(false, reason)`, separating expected refusals from infrastructure failures. The `ApplySurchargeAsync` method demonstrates the second key topic of the lesson: mixing raw SQL (`ExecuteSqlInterpolatedAsync`) and the Change Tracker (`db.OrderItems.Add` + `SaveChanges`) inside a single transaction. The parameterisation through the interpolated handler turns `{factor}` and `{orderId}` into real SQL parameters — it is safe and is not concatenation. `SaveChanges` inside an already-open explicit transaction "joins" it instead of creating its own implicit one. The whole construct is wrapped in the same `try/catch/rollback/throw` shape. Collection expressions (`Items = []`) and `required` members are C# 12 features that keep the model concise and protect against forgotten initialisation. The `OrderService(ShopDbContext db, ILogger<OrderService> log)` primary constructor is a C# 12 nicety that removes boilerplate field assignments.

#### Going deeper (bonus)
1. Implement a concurrent stress test: 100 simultaneous `PlaceOrderAsync` calls on a single product with `Stock = 10` and `qty = 1`; verify that exactly 10 orders succeed and 90 are rejected. Explain why `ReadCommitted` plus the atomic decrement is enough and `Serializable` would be overkill here.
2. Port `PlaceOrderAsync` to `TransactionScope` with `TransactionScopeAsyncFlowOption.Enabled` and compare readability and behaviour with `IDbContextTransaction`. Describe the pros and cons of each approach.
3. Switch the isolation level to `Snapshot` and explicitly enable `ALTER DATABASE AcmeShop SET ALLOW_SNAPSHOT_ISOLATION ON`. Measure the difference in locks through `sys.dm_tran_locks` under concurrent orders.
4. Implement a compensating saga: if the audit event fails after the order has been committed (simulate it with a separate `SaveChanges` outside the transaction), compensate the previous operation and write a compensating event. Explain why this is no longer an ACID transaction but the Saga pattern.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `AcmeShop.Transactions` собирается на .NET 8 без ошибок.
- [ ] Реализованы модели, `ShopDbContext`, сидинг.
- [ ] `PlaceOrderAsync` использует явную транзакцию с `ReadCommitted` и `CommitAsync`.
- [ ] Бизнес-отказы возвращают `PlaceOrderResult`, инфраструктурные — `throw`.
- [ ] `ApplySurchargeAsync` комбинирует `ExecuteSqlInterpolated` и `SaveChanges` в одной транзакции.
- [ ] Реализован retry-safe вариант через `CreateExecutionStrategy`.
- [ ] Unit-тесты против SQLite in-memory зелёные (успех + откат).
- [ ] В `README.md` описаны схема БД, выбор уровня изоляции и запуск.
- [ ] Project `AcmeShop.Transactions` builds on .NET 8 without errors.
- [ ] Models, `ShopDbContext`, and seeding are implemented.
- [ ] `PlaceOrderAsync` uses an explicit transaction with `ReadCommitted` and `CommitAsync`.
- [ ] Business refusals return `PlaceOrderResult`; infrastructure failures `throw`.
- [ ] `ApplySurchargeAsync` combines `ExecuteSqlInterpolated` and `SaveChanges` in one transaction.
- [ ] A retry-safe variant through `CreateExecutionStrategy` is implemented.
- [ ] Unit tests against SQLite in-memory are green (success + rollback).
- [ ] `README.md` covers the DB schema, the isolation-level choice, and how to run.

#### Ресурсы / Resources
- [Microsoft Learn — Transactions in EF Core](https://learn.microsoft.com/ef/core/saving/transactions)
- [Microsoft Learn — Connection resiliency (CreateExecutionStrategy)](https://learn.microsoft.com/ef/core/miscellaneous/connection-resiliency)
- [Microsoft Learn — Isolation levels in ADO.NET](https://learn.microsoft.com/dotnet/api/system.data.isolationlevel)
- [Microsoft Learn — ExecuteSqlInterpolated / raw SQL](https://learn.microsoft.com/ef/core/querying/raw-sql)
- [Microsoft Learn — TransactionScopeAsyncFlowOption](https://learn.microsoft.com/dotnet/api/system.transactions.transactionscopeasyncflowoption)
