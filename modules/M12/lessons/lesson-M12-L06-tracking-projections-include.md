[⬆ К модулю M12](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M12-L06: AsNoTracking, проекции, Include/ThenInclude, N+1 / AsNoTracking, projections, Include/ThenInclude, N+1

**Модуль / Module:** M12
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

EF Core по умолчанию ведёт **change tracking** — отслеживание изменений. Когда вы загружаете сущность из базы, EF Core помещает её в внутренний «склад» (ChangeTracker), который запоминает текущие значения свойств. Это как кладовщик, который записывает, в каком состоянии товар покинул склад: при вызове `SaveChanges` он сравнивает текущее состояние с записанным и генерирует нужные `UPDATE`/`DELETE`. Это удобно для сценариев редактирования, но имеет цену: память и CPU тратятся на хранение снимков и сравнение.

Для **сценариев чтения** (отчёты, списки, дашборды, API GET-эндпоинты) отслеживание не нужно. Метод `AsNoTracking()` отключает его: сущности загружаются, но не попадают в ChangeTracker. Это экономит память и ускоряет запросы, особенно при загрузке больших списков. Существует также `AsNoTrackingWithIdentityResolution()` — он не отслеживает изменения, но обеспечивает, что один и тот же объект (по ключу) не будет дублирован в графе связей, что полезно при загрузке сложных графов только для чтения.

**Проекции** через `Select` — это мощный приём. Вместо загрузки всей сущности `User` со всеми колонками вы выбираете только нужные поля в DTO или анонимный тип. Проекции дают три выгоды: (1) меньше данных идёт из базы (только нужные колонки), (2) результат автоматически не отслеживается, даже если включает сущности — EF Core понимает, что вы читаете, а не редактируете, (3) SQL становится точнее. Всегда предпочитайте проекцию, когда вам не нужна полная сущность.

**Include / ThenInclude** — это eager loading (жадная загрузка) связанных данных. `Include(u => u.Orders)` загружает заказы пользователя одним запросом через `JOIN`. `ThenInclude(o => o.Items)` углубляется дальше — грузит позиции заказов. Без Include навигационные свойства остаются `null` (или пустыми коллекциями), и обращение к ним не подгрузит данные автоматически. Это ключевое отличие от lazy loading, который требует отдельной настройки и часто создаёт проблемы.

**Проблема N+1** — классическая ловушка. Представьте, что вы загрузили 100 пользователей без их заказов, а потом в цикле обращаетесь к `user.Orders`. EF Core выполнит 1 запрос для пользователей + 100 отдельных запросов для заказов каждого = 101 запрос. Это убивает производительность. Решения: (1) `Include` для eager loading — один запрос с `JOIN`, (2) проекция через `Select` — часто ещё эффективнее, (3) явная загрузка (`Load()`) — для редких случаев, (4) `Split queries` (`AsSplitQuery()`) — когда JOIN создаёт «декартов взрыв» (cartesian explosion) множества коллекций, разбивая один большой запрос на несколько простых.

**Split queries** решают проблему, когда один запрос с несколькими `Include` коллекций создаёт огромный декартов результат: если у пользователя 10 заказов и у каждого 20 позиций, то строки дублируются — 1 × 10 × 20 = 200 строк в одном результате. `AsSplitQuery()` разбивает это на несколько запросов (по одному на коллекцию), каждый из которых меньше и быстрее передаётся. Trade-off: больше сетевых round-trips, но меньше данных и памяти.

#### Theory (EN)

EF Core performs **change tracking** by default. When you load an entity, EF Core places it into an internal store (the ChangeTracker) that records the current property values — like a warehouse clerk noting the exact state of each item leaving the shelf. On `SaveChanges`, the clerk compares the current state to the snapshot and emits the appropriate `UPDATE`/`DELETE` statements. This is essential for edit workflows but costs memory and CPU to hold and diff snapshots.

For **read scenarios** (reports, lists, dashboards, API GET endpoints), tracking is unnecessary overhead. The `AsNoTracking()` method disables it: entities are materialized but never enter the ChangeTracker. This saves memory and speeds up queries, especially for large result sets. There is also `AsNoTrackingWithIdentityResolution()`, which skips change tracking but still ensures that the same entity (by key) is not duplicated within a relationship graph — useful when loading complex read-only graphs.

**Projections** via `Select` are a powerful technique. Instead of loading the entire `User` entity with every column, you project only the fields you need into a DTO or anonymous type. Projections give three wins: (1) less data travels from the database (only required columns), (2) the result is automatically not tracked — even if it includes entities, EF Core understands you are reading, not editing, (3) the generated SQL is tighter. Always prefer a projection when you do not need the full entity.

**Include / ThenInclude** provide eager loading of related data. `Include(u => u.Orders)` loads a user's orders in a single query using a `JOIN`. `ThenInclude(o => o.Items)` drills deeper — loading each order's line items. Without `Include`, navigation properties stay `null` (or empty collections) and accessing them does not silently fetch data. This is a deliberate contrast with lazy loading, which requires extra configuration and frequently causes performance problems.

The **N+1 problem** is the classic trap. Suppose you load 100 users without their orders, then in a loop you touch `user.Orders`. EF Core runs 1 query for users plus 100 separate queries for each user's orders = 101 queries. This destroys performance. The fixes are: (1) `Include` for eager loading — one `JOIN`ed query, (2) a `Select` projection — often the most efficient, (3) explicit loading (`Load()`) for rare cases, (4) **split queries** (`AsSplitQuery()`) — when a single JOIN causes a *cartesian explosion* across multiple collections, splitting one giant query into several smaller ones.

**Split queries** address the case where a single query with multiple collection `Include`s produces a bloated cartesian result: if a user has 10 orders and each order has 20 items, rows duplicate — 1 × 10 × 20 = 200 rows in a single result set. `AsSplitQuery()` breaks this into multiple queries (one per collection), each smaller and cheaper to transfer. The trade-off: more network round-trips, but less duplicated data and less memory.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8+, EF Core 8
// Демонстрация AsNoTracking, проекций, Include/ThenInclude и split queries.
// Demo of AsNoTracking, projections, Include/ThenInclude and split queries.

using Microsoft.EntityFrameworkCore;

// 1) AsNoTracking — чтение без отслеживания / read with no tracking
//    Используется для GET-эндпоинтов, отчётов, списков только для чтения.
//    Use for GET endpoints, reports, read-only lists.
List<User> users = await db.Users
    .AsNoTracking()           // отключаем change tracking / disable change tracking
    .Where(u => u.IsActive)
    .OrderBy(u => u.LastName)
    .ToListAsync();

// 2) Проекция (Select) — лучший выбор для чтения, когда не нужна вся сущность.
//    Projection — best for reads when the full entity is not needed.
//    Результат не отслеживается автоматически / result is not tracked automatically.
var userDtos = await db.Users
    .AsNoTracking()
    .Where(u => u.IsActive)
    .Select(u => new UserBriefDto(
        u.Id,
        u.FullName,
        u.Email,
        u.Orders.Count))         // агрегат на стороне БД / aggregate on the DB side
    .ToListAsync();

// 3) Include / ThenInclude — eager loading графа связанных данных.
//    Include / ThenInclude — eager load a related graph.
var usersWithOrders = await db.Users
    .AsNoTracking()
    .Include(u => u.Orders)              // заказы пользователя / user's orders
        .ThenInclude(o => o.Items)       // позиции каждого заказа / each order's items
        .ThenInclude(i => i.Product)     // продукт каждой позиции / each item's product
    .Where(u => u.IsActive)
    .ToListAsync();

// 4) N+1 — ТАК ДЕЛАТЬ НЕЛЬЗЯ / DO NOT DO THIS
//    1 запрос для пользователей + N запросов по заказам в цикле.
//    1 query for users + N queries for orders inside the loop.
foreach (User u in await db.Users.AsNoTracking().ToListAsync())
{
    Console.WriteLine(u.Orders.Count); // ленивая подгрузка / lazy fetch → N+1
}

// 5) Исправление N+1 через Include — один JOIN-запрос.
//    Fix N+1 with Include — a single JOINed query.
foreach (User u in await db.Users
             .AsNoTracking()
             .Include(u => u.Orders)
             .ToListAsync())
{
    Console.WriteLine(u.Orders.Count); // данные уже загружены / already loaded
}

// 6) Split queries — защита от декартова взрыва при нескольких коллекциях.
//    Split queries — guard against cartesian explosion with multiple collections.
var splitResult = await db.Users
    .AsNoTracking()
    .AsSplitQuery()                      // разбить на несколько запросов / split into several queries
    .Include(u => u.Orders)
        .ThenInclude(o => o.Items)
    .Include(u => u.Reviews)             // вторая коллекция → без split даёт декартов взрыв
                                         // second collection → without split causes cartesian explosion
    .ToListAsync();

// 7) AsNoTrackingWithIdentityResolution — сложный граф только для чтения.
//    AsNoTrackingWithIdentityResolution — complex read-only graph.
var graph = await db.Users
    .AsNoTrackingWithIdentityResolution()
    .Include(u => u.Orders)
        .ThenInclude(o => o.Items)
            .ThenInclude(i => i.Product)
    .ToListAsync();
// Одна и та же сущность Product не дублируется в памяти, но не отслеживается.
// The same Product entity is not duplicated in memory, but is not tracked.

// DTO-запись для примера выше / DTO record used above.
public record UserBriefDto(int Id, string FullName, string Email, int OrderCount);

// Сущности для контекста / entities for context.
public class User
{
    public int Id { get; set; }
    public string FullName { get; set; } = "";
    public string LastName { get; set; } = "";
    public string Email { get; set; } = "";
    public bool IsActive { get; set; }
    public List<Order> Orders { get; set; } = [];
    public List<Review> Reviews { get; set; } = [];
}

public class Order
{
    public int Id { get; set; }
    public int UserId { get; set; }
    public User User { get; set; } = null!;
    public List<OrderItem> Items { get; set; } = [];
}

public class OrderItem
{
    public int Id { get; set; }
    public int OrderId { get; set; }
    public int ProductId { get; set; }
    public Order Order { get; set; } = null!;
    public Product Product { get; set; } = null!;
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
}

public class Review
{
    public int Id { get; set; }
    public int UserId { get; set; }
    public string Text { get; set; } = "";
}
```

#### Best Practices

- Используйте `AsNoTracking()` для всех запросов только на чтение: это экономит память и CPU.
- Предпочитайте проекции (`Select` в DTO) загрузке полных сущностей, когда нужны лишь несколько полей.
- Применяйте `Include`/`ThenInclude` для eager loading вместо lazy loading в большинстве случаев.
- Рассматривайте `AsSplitQuery()`, когда один запрос включает несколько коллекций и создаёт декартов взрыв.
- Включайте `AsNoTrackingWithIdentityResolution()` для сложных графов только для чтения, чтобы избежать дубликатов в памяти.
- Измеряйте реальные запросы к БД (логирование SQL) — не угадывайте производительность.

- Use `AsNoTracking()` for every read-only query to save memory and CPU.
- Prefer projections (`Select` into a DTO) over full entities when only a few fields are needed.
- Use `Include`/`ThenInclude` for eager loading instead of lazy loading in most cases.
- Consider `AsSplitQuery()` when a single query spans multiple collections and causes cartesian explosion.
- Enable `AsNoTrackingWithIdentityResolution()` for complex read-only graphs to avoid in-memory duplicates.
- Measure actual DB queries (SQL logging) — do not guess performance.

#### Частые ошибки / Common Mistakes

- Забыли `AsNoTracking()` в GET-эндпоинте → ChangeTracker раздувается в памяти на каждом запросе → добавляйте `AsNoTracking()` (или делайте контекст `QueryTrackingBehavior.NoTracking` по умолчанию).
- Обращение к навигационному свойству в цикле без `Include` → N+1 запросов → добавьте `Include` или используйте проекцию `Select`.
- Загрузка всей сущности ради одного поля (`u.FullName`) → лишний трафик из БД → проецируйте только нужные поля в DTO.
- Один запрос с тремя коллекциями через `Include` → декартов взрыв строк → используйте `AsSplitQuery()`.
- Ожидание, что `AsNoTracking()` обновит сущность через `SaveChanges` → изменения не сохранятся → для редактирования используйте tracked-запрос или `Update()`.

- Forgot `AsNoTracking()` in a GET endpoint → ChangeTracker bloats memory on each request → add `AsNoTracking()` (or set the context default to `QueryTrackingBehavior.NoTracking`).
- Touching a navigation property in a loop without `Include` → N+1 queries → add `Include` or use a `Select` projection.
- Loading a whole entity to read a single field (`u.FullName`) → unnecessary DB traffic → project only the needed fields into a DTO.
- A single query with three collection `Include`s → cartesian explosion of rows → use `AsSplitQuery()`.
- Expecting `AsNoTracking()` entities to be saved via `SaveChanges` → changes are lost → for edits use a tracked query or `Update()`.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я добавил `AsNoTracking()` ко всем запросам, которые только читают данные.
- [ ] Я использую проекции (`Select` в DTO), когда не нужна вся сущность.
- [ ] Я применяю `Include`/`ThenInclude` вместо доступа к навигационным свойствам в цикле.
- [ ] Я знаю, как распознать N+1 по логу SQL и как его исправить.
- [ ] Я использую `AsSplitQuery()`, когда JOIN нескольких коллекций создаёт декартов взрыв.
- [ ] Я понимаю разницу между `AsNoTracking()` и `AsNoTrackingWithIdentityResolution()`.

- [ ] I added `AsNoTracking()` to all queries that only read data.
- [ ] I use projections (`Select` into DTOs) when the full entity is not needed.
- [ ] I use `Include`/`ThenInclude` instead of touching navigation properties in a loop.
- [ ] I can recognise N+1 in the SQL log and know how to fix it.
- [ ] I use `AsSplitQuery()` when a JOIN of multiple collections causes cartesian explosion.
- [ ] I understand the difference between `AsNoTracking()` and `AsNoTrackingWithIdentityResolution()`.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/ef/core/querying/tracking](https://learn.microsoft.com/ef/core/querying/tracking)
- [Microsoft Learn — Eager loading / https://learn.microsoft.com/ef/core/querying/related-data](https://learn.microsoft.com/ef/core/querying/related-data)
- [Microsoft Learn — Split queries / https://learn.microsoft.com/ef/core/querying/single-split-queries](https://learn.microsoft.com/ef/core/querying/single-split-queries)

---

[⬆ К модулю M12](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
