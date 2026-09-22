[⬆ К модулю M12](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M12-L05: LINQ to Entities, переводы в SQL / LINQ to Entities, SQL translation

**Модуль / Module:** M12
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

LINQ to Entities — это способ писать запросы к базе данных на C#, а не на «голом» SQL. Вы используете знакомые методы `Where`, `Select`, `OrderBy`, `Include` на `DbSet<T>`, а Entity Framework Core переводит их в SQL и отправляет провайдеру (например, SQL Server или PostgreSQL). Представьте себе переводчика на переговорах: вы говорите по-русски (LINQ), а он мгновенно переводит для собеседника на английский (SQL). Вы не учите чужой язык — вы просто формулируете мысль знакомыми словами, а переводчик берёт остальное на себя.

Основная точка входа — это `DbContext`. Его свойства типа `DbSet<Order> Orders` представляют собой таблицы. Когда вы вызываете `context.Orders.Where(o => o.Total > 1000)`, EF Core не выполняет код немедленно. Вместо этого он строит **дерево выражения** (expression tree) — абстрактное описание того, что вы попросили. Из этого дерева генерируется строка SQL. Выполнение происходит только в момент материализации: `ToList()`, `FirstOrDefault()`, `Count()` или перечисление через `foreach`. До этого момента всё — просто план.

Не всё переводится в SQL. EF Core знает только те конструкции, для которых есть SQL-эквивалент у текущего провайдера. Предикаты вида `o => o.Status == "Paid"` переведутся легко, а вызов вашего метода `o => o.IsEligible(o)` — нет, потому что провайдер не умеет декомпилировать произвольный C#. Если в `Where` попадёт непереводимое выражение, EF Core выбросит `InvalidOperationException` ещё до обращения к базе.

Исторически в EF Core 2.x существовал механизм **client-side evaluation**: если выражение не переводилось, EF «дотягивал» данные и досчитывал на клиенте. Это было удобно, но опасно — один непереводимый вызов мог выкачать всю таблицу в память. Начиная с EF Core 3.0 этот режим **объявлен устаревшим и по умолчанию запрещён**: непереводимое выражение вызывает исключение, а не тихий pull всей таблицы. Это правильный выбор: лучше явная ошибка, чем скрытая деградация производительности.

Метод `Select` нужен не только для проекции формы результата, но и для оптимизации: `Select(o => new { o.Id, o.Total })` заставляет EF сгенерировать `SELECT Id, Total`, а не `SELECT *`. Это уменьшает объём данных по сети. Проекция — ваш главный инструмент для «тонких» запросов.

`Include` — это аналог `JOIN`, который вытягивает связанные сущности. `context.Orders.Include(o => o.Customer)` сгенерирует `LEFT JOIN Customers`. Несколько `Include` подряд дают несколько JOIN'ов и могут сильно разрастить результат (знаменитая «картезианская» проблема). Для деревьев связей используйте `ThenInclude`, а для независимых коллекций — разбивку на несколько запросов или `AsSplitQuery()`.

Главное правило: думайте о LINQ как о **спецификации запроса**, а не о C#-коде, выполняющемся строка за строкой. Каждое звено цепочки — это описание, а не действие. Проверяйте сгенерированный SQL через логирование (`LogTo` в `OnConfiguring` или `context.Database.Log`). Видеть SQL — половина успеха в LINQ to Entities.

#### Theory (EN)

LINQ to Entities is a way to write database queries in C# instead of raw SQL. You use familiar methods like `Where`, `Select`, `OrderBy`, and `Include` on a `DbSet<T>`, and Entity Framework Core translates them into SQL and sends the result to the provider — SQL Server, PostgreSQL, SQLite, and so on. Picture an interpreter at a negotiation: you speak Russian (LINQ), and the interpreter instantly translates to English (SQL). You never learn the other language — you just express the idea in your own words, and the interpreter handles the rest.

The main entry point is the `DbContext`. Its properties of type `DbSet<Order> Orders` represent tables. When you call `context.Orders.Where(o => o.Total > 1000)`, EF Core does not execute anything immediately. Instead, it builds an **expression tree** — an abstract description of what you asked for. From that tree EF generates a SQL string. Actual execution happens only at materialization: `ToList()`, `FirstOrDefault()`, `Count()`, or enumeration in a `foreach` loop. Until that moment, everything is just a plan.

Not everything translates to SQL. EF Core knows only the constructs that have a SQL equivalent for the current provider. A predicate like `o => o.Status == "Paid"` translates easily, but a call to your own method `o => o.IsEligible(o)` does not — the provider cannot decompile arbitrary C#. If an untranslatable expression lands inside `Where`, EF Core throws an `InvalidOperationException` before it ever touches the database.

Historically, EF Core 2.x supported **client-side evaluation**: if an expression could not be translated, EF pulled data and finished the work on the client. This was convenient but dangerous — a single untranslatable call could download an entire table into memory. Starting with EF Core 3.0, this mode is **deprecated and off by default**: an untranslatable expression throws an exception rather than silently pulling the whole table. That is the right call: an explicit error beats hidden performance degradation.

The `Select` method is not only for shaping results — it is an optimization tool. `Select(o => new { o.Id, o.Total })` makes EF emit `SELECT Id, Total` instead of `SELECT *`. That cuts the bytes traveling over the network. Projection is your primary weapon for slim queries.

`Include` is the equivalent of `JOIN` that pulls related entities. `context.Orders.Include(o => o.Customer)` produces a `LEFT JOIN Customers`. Several `Include` calls in a row add several JOINs and can blow up the result set — the famous Cartesian explosion. For relationship trees use `ThenInclude`; for independent collections consider splitting into multiple queries or calling `AsSplitQuery()`.

The golden rule: treat LINQ as a **query specification**, not as C# code that executes line by line. Each link in the chain is a description, not an action. Inspect the generated SQL via logging (`LogTo` in `OnConfiguring` or `context.Database.Log`). Seeing the SQL is half the battle in LINQ to Entities.

#### Пример кода / Code Example

```csharp
using Microsoft.EntityFrameworkCore;

// Модель / Model
public class Order
{
    public int Id { get; set; }
    public decimal Total { get; set; }
    public string Status { get; set; } = "New"; // "New", "Paid", "Cancelled"
    public int CustomerId { get; set; }
    public Customer Customer { get; set; } = null!;
    public List<OrderItem> Items { get; set; } = new();
}

public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public List<Order> Orders { get; set; } = new();
}

public class OrderItem
{
    public int Id { get; set; }
    public int OrderId { get; set; }
    public string Product { get; set; } = "";
    public decimal Price { get; set; }
}

public class ShopContext : DbContext
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();

    protected override void OnConfiguring(DbContextOptionsBuilder opts)
    {
        // Логирование SQL в консоль / Log generated SQL to console
        opts.UseSqlite("Data Source=shop.db")
            .LogTo(Console.WriteLine, LogLevel.Information);
    }
}

// --- LINQ to Entities: примеры / examples ---

using var db = new ShopContext();

// 1) Where + OrderBy + Select — проекция в DTO / projection to DTO
//    Переводится в: SELECT Id, Total FROM Orders WHERE Status = 'Paid' ORDER BY Total DESC
var paidDtos = await db.Orders
    .Where(o => o.Status == "Paid")            // переводится / translates
    .OrderByDescending(o => o.Total)           // переводится / translates
    .Select(o => new { o.Id, o.Total })        // проекция: SELECT Id, Total
    .ToListAsync();                             // материализация / materialization

// 2) Include — аналог JOIN для связанных сущностей / JOIN-like load of related data
//    Переводится в: SELECT o.*, c.* FROM Orders LEFT JOIN Customers ...
var orderWithCustomer = await db.Orders
    .Include(o => o.Customer)
    .FirstOrDefaultAsync(o => o.Id == 42);

// 3) Несколько Include + ThenInclude — дерево связей / relationship tree
var detailed = await db.Orders
    .Include(o => o.Customer)
    .Include(o => o.Items)                     // коллекция / collection
    .AsSplitQuery()                            // отдельные запросы вместо одного гигантского JOIN
    .Where(o => o.Total > 1000)
    .ToListAsync();

// 4) Aggregation, которая переводится в SQL / translatable aggregation
var topCustomer = await db.Orders
    .Where(o => o.Status == "Paid")
    .GroupBy(o => o.CustomerId)
    .Select(g => new { CustomerId = g.Key, Sum = g.Sum(o => o.Total) })
    .OrderByDescending(g => g.Sum)
    .FirstOrDefaultAsync();

// 5) НЕ переводится (бросит исключение, а не client-side eval) / NOT translatable
// bool IsVip(decimal total) => total > 5000;
// var bad = db.Orders.Where(o => IsVip(o.Total)).ToList(); // InvalidOperationException в EF Core 3+
```

#### Best Practices

- Проектируйте результат через `Select` в DTO, чтобы не тянуть `SELECT *` и не плодить «жирные» сущности в памяти.
- Use `Select` to project into DTOs so you avoid `SELECT *` and keep entities thin in memory.
- Включайте логирование SQL (`LogTo`) в разработке и тестах — видеть SQL значит понимать стоимость запроса.
- Enable SQL logging (`LogTo`) in dev and test environments — seeing the SQL means understanding its cost.
- Избегайте нескольких `Include` коллекций в одном запросе; используйте `AsSplitQuery()` для деревьев.
- Avoid multiple collection `Include`s in a single query; use `AsSplitQuery()` for trees.
- Храните предикаты в виде `Expression<Func<T, bool>>`, а не `Func<T, bool>`, иначе они вычислятся на клиенте.
- Store predicates as `Expression<Func<T, bool>>`, not `Func<T, bool>`, or they evaluate on the client.
- Явно материализуйте (`ToListAsync`, `FirstOrDefaultAsync`) — это место, где летит SQL.
- Materialize explicitly (`ToListAsync`, `FirstOrDefaultAsync`) — that is the moment SQL is sent.

#### Частые ошибки / Common Mistakes

- Использование `IEnumerable<T>` вместо `IQueryable<T>` после `.AsEnumerable()` → весь фильтр уходит в память. Не вызывайте `AsEnumerable()` до `Where`/`Select`.
- Calling `.AsEnumerable()` before `Where`/`Select` → the whole filter runs in memory. Keep the chain `IQueryable<T>` until the final materialization.
- Вызов собственного метода внутри `Where` (`o => MyFunc(o.Total)`) → `InvalidOperationException`. Выносите логику в переводимое выражение или в БД через вычисляемый столбец.
- Calling your own method inside `Where` (`o => MyFunc(o.Total)`) → `InvalidOperationException`. Move the logic into a translatable expression or a database computed column.
- `.Count()` перед `.ToList()` с двойным запросом → два SQL-запроса там, где хватит одного. Комбинируйте через проекцию.
- Calling `.Count()` and then `.ToList()` separately → two SQL round-trips where one suffices. Combine via a single projection.
- Каскад `Include` коллекций без `AsSplitQuery()` → декартово произведение и гигантский результат. Включайте split-запросы для деревьев.
- Cascading collection `Include`s without `AsSplitQuery()` → Cartesian explosion. Turn on split queries for trees.
- Ожидание client-side eval как в EF Core 2.x → в 3+ это запрещено. Не полагайтесь на «авось переведёт».
- Expecting EF Core 2.x-style client-side evaluation → in 3+ it is disabled. Never rely on "maybe it will translate".

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Цепочка запроса остаётся `IQueryable<T>` до момента материализации.
- [ ] The query chain stays `IQueryable<T>` until materialization.
- [ ] В `Where`/`Select` только переводимые выражения, без вызовов своих методов.
- [ ] Only translatable expressions inside `Where`/`Select`, with no calls to custom methods.
- [ ] Для фильтров используются `Expression<Func<T, bool>>`, а не `Func<T, bool>`.
- [ ] Filters use `Expression<Func<T, bool>>`, not `Func<T, bool>`.
- [ ] Результат спроецирован в DTO через `Select` (нет `SELECT *`).
- [ ] The result is projected into a DTO via `Select` (no `SELECT *`).
- [ ] Для деревьев связей включён `AsSplitQuery()` при нескольких `Include` коллекций.
- [ ] `AsSplitQuery()` is enabled when multiple collection `Include`s are used for a tree.
- [ ] SQL логирование включено в разработке, и я видел сгенерированный запрос.
- [ ] SQL logging is enabled in dev, and I have inspected the generated query.

#### Ресурсы / Resources

- [Microsoft Learn — Querying in EF Core — https://learn.microsoft.com/ef/core/querying/](https://learn.microsoft.com/ef/core/querying/)
- [Microsoft Learn — Client vs. Server Evaluation — https://learn.microsoft.com/ef/core/querying/client-eval](https://learn.microsoft.com/ef/core/querying/client-eval)
- [Microsoft Learn — Loading Related Data (Include) — https://learn.microsoft.com/ef/core/querying/related-data](https://learn.microsoft.com/ef/core/querying/related-data)
- [Microsoft Learn — Single vs. Split Queries — https://learn.microsoft.com/ef/core/querying/single-split-queries](https://learn.microsoft.com/ef/core/querying/single-split-queries)

---

[⬆ К модулю M12](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
