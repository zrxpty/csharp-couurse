---
[← К уроку M12-L06](lesson-M12-L06-tracking-projections-include.md) | [⬆ К модулю M12](../README.md) | [Следующее ДЗ →](homework-M12-L07-transactions-savechanges.md)
---

### Домашнее задание M12-L06: AsNoTracking, проекции, Include/ThenInclude, N+1 / Homework M12-L06: AsNoTracking, projections, Include/ThenInclude, N+1

**Урок / Lesson:** M12-L06
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться писать эффективные запросы EF Core 8 на чтение: отключать change tracking через `AsNoTracking()` и `AsNoTrackingWithIdentityResolution()`, проецировать только нужные поля в DTO, жадно загружать графы через `Include`/`ThenInclude`, распознавать и устранять проблему N+1, а также применять `AsSplitQuery()` против декартова взрыва при нескольких коллекциях. (EN) Learn to write efficient read-only EF Core 8 queries: disable change tracking via `AsNoTracking()` and `AsNoTrackingWithIdentityResolution()`, project only the required fields into DTOs, eagerly load graphs with `Include`/`ThenInclude`, recognise and eliminate the N+1 problem, and apply `AsSplitQuery()` against cartesian explosion with multiple collections.

#### Связь с уроком / Connection to the lesson
(RU) Урок M12-L06 объясняет, что EF Core по умолчанию ведёт change tracking, и показывает, когда это вредно — в GET-эндпоинтах, отчётах и дашбордах. В ДЗ вы примените каждый приём из урока к реалистичной модели магазина: `AsNoTracking`, проекции в record-DTO, `Include`/`ThenInclude`, исправление N+1, `AsSplitQuery` и `AsNoTrackingWithIdentityResolution`. Особое внимание уделено SQL-логированию, чтобы вы увидели, сколько запросов реально уходит в базу, и убедились, что исправления работают, а не «кажется, что быстрее».
(EN) Lesson M12-L06 explains that EF Core performs change tracking by default and shows when that is harmful — in GET endpoints, reports and dashboards. In this homework you will apply every technique from the lesson to a realistic shop model: `AsNoTracking`, projections into record DTOs, `Include`/`ThenInclude`, fixing N+1, `AsSplitQuery`, and `AsNoTrackingWithIdentityResolution`. Special attention is given to SQL logging, so you can see how many queries actually hit the database and confirm that the fixes work — not just “feel faster”.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы backend-разработчик сервиса «ShopHub» — каталога пользователей, заказов, позиций заказов, товаров и отзывов. Команда заметила, что дашборд администратора и публичный API-эндпоинт списка пользователей начали «тормозить» по мере роста базы: при 5000 пользователей страница грузится несколько секунд, в логах PostgreSQL сотни одинаковых запросов, а контейнер API регулярно упирается в лимит памяти. Анализ показал три класса проблем, ровно тех, что разобраны в уроке M12-L06: (1) почти все GET-запросы идут через tracked-сущности, хотя ничего не редактируют, и ChangeTracker раздувается на каждый запрос; (2) несколько мест обращаются к навигационным свойствам внутри цикла `foreach`, порождая классический N+1; (3) есть «тяжёлый» отчёт, который через три `Include` коллекций загружает декартов взрыв из тысяч дублированных строк. Ваша задача — переписать слой чтения так, чтобы он遵循 уроку: использовать `AsNoTracking()` по умолчанию, проекции в DTO, `Include`/`ThenInclude` вместо доступа к навигациям в цикле, `AsSplitQuery()` против декартова взрыва и `AsNoTrackingWithIdentityResolution()` для сложного read-only графа. Вы должны не просто «сделать», но и доказать улучшение через SQL-логи: считать количество запросов до и после, показать, что tracked-сущности больше не оседают в ChangeTracker, и объяснить, почему каждое изменение работает. Это упражнение не про синтаксис — он тривиален — а про привычку измерять и выбирать правильный инструмент: проекция против `Include`, `AsNoTracking` против `AsNoTrackingWithIdentityResolution`, один JOIN против split queries. Урок подчёркивает: «не угадывайте производительность — измеряйте её». Поэтому в ДЗ обязательно включается логирование SQL и вывод `ChangeTracker.Entries().Count()` после каждого сценария.

#### Что нужно сделать (пошагово)
1. Создайте новый проект: `dotnet new console -n ShopHub.ReadQueries -o ShopHub.ReadQueries` в директории `modules/M12/homework/M12-L06/`. Целевой фреймворк — `net8.0`. Добавьте пакеты: `dotnet add package Microsoft.EntityFrameworkCore --version 8.*`, `dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 8.*` (используем SQLite InMemory, чтобы не поднимать Postgres, но запросы и SQL-логи будут реальными).
2. В файле `Program.cs` (top-level statements) определите сущности `User`, `Order`, `OrderItem`, `Product`, `Review` ровно по модели из урока: `User` имеет `Id`, `FullName`, `LastName`, `Email`, `IsActive`, коллекции `Orders` и `Reviews` (используйте collection expressions `[]` для инициализации); `Order` — `Id`, `UserId`, `User`, `Items`; `OrderItem` — `Id`, `OrderId`, `ProductId`, `Order`, `Product`; `Product` — `Id`, `Name`; `Review` — `Id`, `UserId`, `Text`. Используйте nullable reference annotations (`null!`) для обязательных навигаций.
3. Создайте `ShopDbContext : DbContext` с `DbSet<User>`, `DbSet<Order>`, `DbSet<OrderItem>`, `DbSet<Product>`, `DbSet<Review>`. В `OnConfiguring` включите SQLite InMemory (`UseSqlite("DataSource=:memory:")` с общим соединением через `KeepAlive`-connection, см. подсказки), а также включите логирование SQL: `optionsBuilder.LogTo(msg => Console.WriteLine(msg), new[] { DbLoggerCategory.Database.Command.Name }, LogLevel.Information).EnableSensitiveDataLogging();`. В `OnModelCreating` настройте ключи и связи (one-to-many User→Orders, Order→Items, OrderItem→Product, User→Reviews).
4. В методе `SeedAsync(DbContext db)` заполните базу: 50 пользователей, у каждого от 1 до 10 заказов, в каждом заказе от 2 до 5 позиций, 10 уникальных продуктов (Product будет переиспользоваться — важно для `AsNoTrackingWithIdentityResolution`), и у части пользователей по 1–3 отзыва. Это создаст граф, на котором N+1 и декартов взрыв становятся заметными.
5. Реализуйте СЦЕНАРИЙ A — «Плохой код» (baseline): метод `RunNPlusOneAsync(db)`, который грузит всех активных пользователей через `db.Users.AsNoTracking().ToListAsync()` (намеренно без `Include`), а затем в `foreach` печатает `u.Orders.Count`. В логе SQL вы должны увидеть 1 + N запросов. Выведите `db.ChangeTracker.Entries().Count()` — он будет равен числу пользователей (если убрать `AsNoTracking`) или нулю (если оставить). Зафиксируйте оба случая в комментарии.
6. Реализуйте СЦЕНАРИЙ B — исправление N+1 через `Include`: метод `RunFixedWithIncludeAsync(db)` — тот же цикл, но запрос с `.Include(u => u.Orders)`. В логе SQL должен быть один JOIN-запрос.
7. Реализуйте СЦЕНАРИЙ C — проекция в DTO: метод `RunProjectionAsync(db)`, который возвращает `List<UserBriefDto>` через `.Select(u => new UserBriefDto(u.Id, u.FullName, u.Email, u.Orders.Count))`. DTO — `public record UserBriefDto(int Id, string FullName, string Email, int OrderCount);`. В логе должен быть один запрос, выбирающий ровно 4 колонки, без JOIN к заказам (агрегат `Count` на стороне БД).
8. Реализуйте СЦЕНАРИЙ D — декартов взрыв и `AsSplitQuery()`: метод `RunCartesianAsync(db)` грузит пользователей с `.Include(u => u.Orders).ThenInclude(o => o.Items).ThenInclude(i => i.Product).Include(u => u.Reviews)` без split. Метод `RunSplitAsync(db)` — то же, но с `.AsSplitQuery()`. Сравните количество строк и round-trips в логе (печатайте число Materialized строк через счётчик).
9. Реализуйте СЦЕНАРИЙ E — `AsNoTrackingWithIdentityResolution()`: метод `RunIdentityResolutionAsync(db)` грузит граф `Orders → Items → Product` и демонстрирует, что один и тот же `Product` (по ключу) не дублируется в памяти: соберите все `Product` из графа в `HashSet<Product>` по reference identity и покажите, что дубликатов ссылок нет, но при этом `db.ChangeTracker.Entries().Count() == 0`.
10. В `Main` вызовите все сценарии по очереди, после каждого печатайте разделитель и краткий отчёт: сколько SQL-запросов ушло (считайте строки категории `Microsoft.EntityFrameworkCore.Database.Command`), сколько строк материализовано, сколько записей в ChangeTracker.
11. Запустите: `dotnet run --project ShopHub.ReadQueries`. Убедитесь, что в выводе видны: (a) N+1 → 1+N запросов, (b) после `Include` → 1 запрос, (c) проекция → 1 запрос с узким SELECT, (d) без split → один «толстый» результат, со split → несколько запросов, (e) identity resolution → 0 в трекере и 0 дублей продукта.
12. В файле `NOTES.md` рядом с проектом зафиксируйте таблицу: сценарий | кол-во SQL-запросов | кол-во строк | ChangeTracker entries | комментарий (почему так).

#### Требования к решению
- Целевой рантайм: .NET 8, C# 12. Используйте top-level statements в `Program.cs`, collection expressions (`[]`) для инициализации коллекций, record для DTO, file-scoped namespaces, nullable reference types включены (`<Nullable>enable</Nullable>`).
- EF Core 8. Провайдер — SQLite InMemory (через `Microsoft.Data.Sqlite` connection с `DataSource=:memory:` и `Mode=Memory`, держащий соединение открытым на всё время работы). Это даёт реальные SQL-логи без внешней БД.
- Все запросы на чтение должны использовать `AsNoTracking()` или `AsNoTrackingWithIdentityResolution()`. Ни один GET-сценарий не должен оставлять записи в `ChangeTracker` — это проверяется выводом `db.ChangeTracker.Entries().Count()`.
- Обязательно включено логирование SQL команд (`LogTo` + категория `DbLoggerCategory.Database.Command`), чтобы можно было подсчитать запросы и увидеть `JOIN`/`LEFT JOIN`/`COUNT(*)`.
- Сценарий N+1 должен быть реализован намеренно («плохой» вариант) и затем исправлен — чтобы студент видел оба лога и разницу. Запрещено «сразу писать правильно» без демонстрации проблемы.
- Проекция (сценарий C) обязана использовать агрегат `u.Orders.Count` на стороне БД, а не клиентский `.Count()` после материализации. Проверьте по SQL, что `ORDER BY`/`JOIN` нет, а есть подзапрос `SELECT COUNT(*)`.
- `AsSplitQuery()` должен применяться только там, где реально несколько коллекций (`Orders.Items` + `Reviews`). Не применяйте split без необходимости — это лишние round-trips.
- Код должен компилироваться без warnings (`TreatWarningsAsErrors` допустимо). Используйте `await` корректно, `ConfigureAwait` не требуется (консольное приложение).
- В `NOTES.md` должна быть таблица сравнения всех пяти сценариев по числу запросов и записей в трекере.

#### Тонкости и подводные камни
- **AsNoTracking ≠ free**: метод убирает сущность из ChangeTracker, но не делает SQL быстрее автоматически — ускорение приходит от меньшего потребления памяти и отсутствия snapshot-диффа. Главное — что `SaveChanges` для такой сущности ничего не сохранит; не пытайтесь редактировать `AsNoTracking`-сущности и звать `SaveChanges`, как предупреждает урок. Если нужно обновить — используйте tracked-запрос или явный `Update()`.
- **AsNoTrackingWithIdentityResolution vs AsNoTracking**: обычный `AsNoTracking` при загрузке графа `Orders → Items → Product` может создать несколько разных экземпляров одного и того же `Product` (по ключу), потому что каждый `OrderItem` ссылается на свой Materialized Product. `AsNoTrackingWithIdentityResolution` гарантирует один экземпляр по ключу — но всё ещё не отслеживает изменения. Выбирайте его для сложных read-only графов с переиспользуемыми сущностями.
- **Проекция автоматически не трекается**: даже если вы `Select`-аете целиком сущность (`new User(...)` из полей), результат не попадает в ChangeTracker. Урок явно отмечает это преимущество. Но не злоупотребляйте проекцией «всех полей» — тогда это просто `AsNoTracking` с лишней работой.
- **N+1 маскируется**: если у вас `Include` на одну коллекцию, но потом в цикле вы трогаете вторую навигацию (например, `u.Reviews`) — вы получаете N+1 по второй коллекции, хоть первый `Include` и сработал. Всегда проверяйте все навигации, к которым обращаетесь внутри цикла.
- **Декартов взрыв считается как произведение**: 1 пользователь × 10 заказов × 20 позиций = 200 строк в одном результате, даже если вам нужны данные пользователя один раз. `AsSplitQuery` разбивает это на 3 запроса: пользователи + заказы + позиции (и продукты), каждый маленький. Trade-off из урока: больше round-trips, но меньше дублированных данных и памяти — оценивайте по своей сетевой топологии.
- **Lazy loading по умолчанию OFF в EF Core**: если вы не настроили прокси-lazy-loading, обращение к `u.Orders` без `Include` в контексте, уже после материализации, просто вернёт `null` (или пустую коллекцию, если инициализирована `[]`). То есть «N+1» в строгом смысле возможен только при включённом lazy loading. В ДЗ мы намеренно моделируем «доступ к навигации в цикле» как отдельный подзапрос через `db.Entry(u).Collection(x => x.Orders).LoadAsync()` или просто показываем, что без `Include` данные отсутствуют — уточните в коде, какой именно механизм вы демонстрируете. Главное — показать проблему «обращение к связанным данным в цикле порождает N отдельных обращений к БД».
- **SQL-логирование шумит**: включайте только категорию `Database.Command`, иначе лог забьётся infrastructure-сообщениями. `EnableSensitiveDataLogging` — только для учебного/локального использования; в проде он может залогировать значения параметров (PII).
- **SQLite InMemory и connection lifetime**: InMemory-база живёт, пока открыто соединение. Если вы создадите `DbContext` с `:memory:` каждый раз заново с новым соединением, база будет пустой. Держите один `SqliteConnection` открытым на всё время и передавайте его в `UseSqlite(conn)`.

#### Критерии приёмки
- [ ] Проект `ShopHub.ReadQueries` создаётся одной командой `dotnet new console`, компилируется на .NET 8 / C# 12 без warnings.
- [ ] Сущности `User`, `Order`, `OrderItem`, `Product`, `Review` определены ровно по модели урока с collection expressions и nullable-навигациями.
- [ ] `ShopDbContext` использует SQLite InMemory с удерживаемым соединением и включает SQL-логирование категории `Database.Command`.
- [ ] Метод `SeedAsync` заполняет граф: ≥50 пользователей, заказы, позиции, переиспользуемые продукты, отзывы.
- [ ] Сценарий A (N+1) выводит в лог ≥1+N SQL-запросов и явно отмечен как «плохой».
- [ ] Сценарий B исправляет N+1 через `Include(u => u.Orders)` — ровно 1 JOIN-запрос.
- [ ] Сценарий C использует проекцию в `UserBriefDto` с агрегатом `u.Orders.Count` на стороне БД — SQL содержит `SELECT COUNT(*)`, без JOIN к заказам.
- [ ] Сценарий D демонстрирует декартов взрыв (без split) и его устранение через `AsSplitQuery()` — лог показывает несколько запросов вместо одного «толстого».
- [ ] Сценарий E использует `AsNoTrackingWithIdentityResolution()` и доказывает отсутствие дублей `Product` по ссылке при `ChangeTracker.Entries().Count() == 0`.
- [ ] После каждого сценария печатается `ChangeTracker.Entries().Count() == 0` (для read-only сценариев).
- [ ] В `NOTES.md` есть таблица сравнения всех 5 сценариев: кол-во запросов, кол-во строк, трекер-entries.
- [ ] Ни в одном read-only сценарии не вызывается `SaveChanges` (и нигде не ожидается, что `AsNoTracking`-сущность сохранится).
- [ ] Код использует top-level statements, file-scoped namespace, record-DTO, collection expressions `[]`.
- [ ] Запуск `dotnet run` завершается без исключений и печатает все 5 разделов отчёта.
- [ ] В комментариях кода указаны RU+EN пояснения к каждому `AsNoTracking`/`Include`/`AsSplitQuery`.

#### Подсказки (без прямого ответа)
- Для SQLite InMemory держите `SqliteConnection` открытым: `using var conn = new SqliteConnection("DataSource=:memory:"); conn.Open();` и передавайте `optionsBuilder.UseSqlite(conn)`. Не используйте `DataSource=:memory:` прямо в строке контекста — каждое новое соединение даст пустую базу.
- Чтобы подсчитать SQL-запросы программно, можно обернуть `LogTo` делегатом, который инкрементирует счётчик при строках, содержащих `Executed DbCommand`.
- Для демонстрации «N+1 без lazy loading» используйте явную загрузку внутри цикла: `await db.Entry(u).Collection(x => x.Orders).LoadAsync();` — это породит по одному запросу на пользователя и наглядно покажет N+1 в логе.
- Агрегат `u.Orders.Count` в проекции EF Core транслирует в подзапрос `SELECT COUNT(*) FROM Orders WHERE UserId = u.Id` — убедитесь, что в SQL нет клиентенного `JOIN`.
- Чтобы показать декартов взрыв количественно, материализуйте `ToList()` и распечатайте `result.Count` — это число строк в плоском результате, а не число пользователей.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 / EF Core 8 — ShopHub.ReadQueries / Program.cs
// Эталонное решение ДЗ M12-L06: AsNoTracking, проекции, Include/ThenInclude, N+1, split queries.
// Reference solution for HW M12-L06: AsNoTracking, projections, Include/ThenInclude, N+1, split queries.

using Microsoft.Data.Sqlite;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Diagnostics;
using System.Collections.Generic;

// Соединение держим открытым всё время работы — InMemory база живёт, пока открыто соединение.
// Keep the connection open for the whole run — the InMemory DB lives while the connection is open.
using var conn = new SqliteConnection("DataSource=:memory:");
conn.Open();

using var db = new ShopDbContext(conn);
await db.Database.EnsureDeletedAsync();
await db.Database.EnsureCreatedAsync();
await SeedAsync(db);

Log("=== СЦЕНАРИЙ A: N+1 (плохой код) / Scenario A: N+1 (bad code) ===");
await RunNPlusOneAsync(db);

Log("=== СЦЕНАРИЙ B: исправление через Include / Scenario B: fix via Include ===");
await RunFixedWithIncludeAsync(db);

Log("=== СЦЕНАРИЙ C: проекция в DTO / Scenario C: projection into DTO ===");
await RunProjectionAsync(db);

Log("=== СЦЕНАРИЙ D1: декартов взрыв (без split) / Scenario D1: cartesian explosion (no split) ===");
await RunCartesianAsync(db);

Log("=== СЦЕНАРИЙ D2: AsSplitQuery / Scenario D2: AsSplitQuery ===");
await RunSplitAsync(db);

Log("=== СЦЕНАРИЙ E: AsNoTrackingWithIdentityResolution / Scenario E: identity resolution ===");
await RunIdentityResolutionAsync(db);

return;

// --- A: намеренный N+1 через явную загрузку в цикле / deliberate N+1 via explicit load in a loop ---
static async Task RunNPlusOneAsync(ShopDbContext db)
{
    var users = await db.Users.AsNoTracking().Where(u => u.IsActive).ToListAsync();
    int queriesBefore = db.CommandCount;
    foreach (var u in users)
    {
        // Явная загрузка заказов внутри цикла → по одному запросу на пользователя → N+1.
        // Explicit loading of orders inside the loop → one query per user → N+1.
        await db.Entry(u).Collection(x => x.Orders).LoadAsync();
        Console.WriteLine($"  {u.FullName}: {u.Orders.Count} orders");
    }
    Log($"  SQL-запросов: {db.CommandCount - queriesBefore} (1 + {users.Count})");
    Log($"  ChangeTracker entries: {db.ChangeTracker.Entries().Count()}  (AsNoTracking → 0)");
}

// --- B: исправление N+1 через Include → один JOIN-запрос / fix N+1 via Include → a single JOIN ---
static async Task RunFixedWithIncludeAsync(ShopDbContext db)
{
    int queriesBefore = db.CommandCount;
    var users = await db.Users
        .AsNoTracking()                    // чтение без отслеживания / read with no tracking
        .Include(u => u.Orders)            // eager load orders одним JOIN / eager load orders in one JOIN
        .Where(u => u.IsActive)
        .ToListAsync();
    foreach (var u in users)
        Console.WriteLine($"  {u.FullName}: {u.Orders.Count} orders");
    Log($"  SQL-запросов: {db.CommandCount - queriesBefore}  (должно быть 1)");
    Log($"  ChangeTracker entries: {db.ChangeTracker.Entries().Count()}  (AsNoTracking → 0)");
}

// --- C: проекция в DTO с агрегатом на стороне БД / projection into a DTO with a DB-side aggregate ---
static async Task RunProjectionAsync(ShopDbContext db)
{
    int queriesBefore = db.CommandCount;
    List<UserBriefDto> dtos = await db.Users
        .AsNoTracking()
        .Where(u => u.IsActive)
        .OrderBy(u => u.LastName)
        .Select(u => new UserBriefDto(u.Id, u.FullName, u.Email, u.Orders.Count)) // COUNT(*) на стороне БД
        .ToListAsync();
    Log($"  SQL-запросов: {db.CommandCount - queriesBefore}  (1, узкий SELECT, без JOIN к заказам)");
    Log($"  DTO записей: {dtos.Count}; пример: {dtos[0]}");
    Log($"  ChangeTracker entries: {db.ChangeTracker.Entries().Count()}  (проекция → 0)");
}

// --- D1: декартов взрыв без split / cartesian explosion without split ---
static async Task RunCartesianAsync(ShopDbContext db)
{
    int queriesBefore = db.CommandCount;
    var rows = await db.Users
        .AsNoTracking()
        .Include(u => u.Orders).ThenInclude(o => o.Items).ThenInclude(i => i.Product)
        .Include(u => u.Reviews)            // вторая коллекция → декартов взрыв / second collection → cartesian explosion
        .Where(u => u.IsActive)
        .ToListAsync();
    int materializedRows = rows.SelectMany(u => u.Orders).SelectMany(o => o.Items).Count();
    Log($"  SQL-запросов: {db.CommandCount - queriesBefore}  (1 «толстый» JOIN)");
    Log($"  Позиций материализовано: {materializedRows}  (дублирование строк из-за JOIN)");
}

// --- D2: AsSplitQuery — несколько запросов вместо одного / several queries instead of one ---
static async Task RunSplitAsync(ShopDbContext db)
{
    int queriesBefore = db.CommandCount;
    var rows = await db.Users
        .AsNoTracking()
        .AsSplitQuery()                     // разбить на несколько запросов / split into several queries
        .Include(u => u.Orders).ThenInclude(o => o.Items).ThenInclude(i => i.Product)
        .Include(u => u.Reviews)
        .Where(u => u.IsActive)
        .ToListAsync();
    int materializedRows = rows.SelectMany(u => u.Orders).SelectMany(o => o.Items).Count();
    Log($"  SQL-запросов: {db.CommandCount - queriesBefore}  (несколько — по коллекции)");
    Log($"  Позиций материализовано: {materializedRows}  (без дублирования строк)");
}

// --- E: AsNoTrackingWithIdentityResolution — один Product по ключу, не в трекере / one Product by key, not tracked ---
static async Task RunIdentityResolutionAsync(ShopDbContext db)
{
    var graph = await db.Users
        .AsNoTrackingWithIdentityResolution()   // не трекать, но дедуплицировать по ключу / no tracking, but dedup by key
        .Include(u => u.Orders).ThenInclude(o => o.Items).ThenInclude(i => i.Product)
        .Where(u => u.IsActive)
        .ToListAsync();

    var products = new HashSet<Product>(ReferenceEqualityComparer.Instance);
    int productRefs = 0;
    foreach (var u in graph)
        foreach (var o in u.Orders)
            foreach (var i in o.Items)
            {
                productRefs++;
                products.Add(i.Product);
            }
    Log($"  Ссылок на Product в графе: {productRefs}; уникальных экземпляров: {products.Count}");
    Log($"  ChangeTracker entries: {db.ChangeTracker.Entries().Count()}  (должно быть 0)");
}

// --- Seed: 50 пользователей, заказы, позиции, переиспользуемые продукты, отзывы ---
static async Task SeedAsync(ShopDbContext db)
{
    var rnd = new Random(42);
    var products = new List<Product>();
    for (int i = 1; i <= 10; i++)
        products.Add(new Product { Id = i, Name = $"Product-{i}" });
    db.Products.AddRange(products);

    for (int u = 1; u <= 50; u++)
    {
        var user = new User
        {
            FullName = $"User {u}",
            LastName = $"Last{u}",
            Email = $"user{u}@shop.dev",
            IsActive = u % 10 != 0 // ~90% активны
        };
        int orderCount = rnd.Next(1, 11);
        for (int o = 1; o <= orderCount; o++)
        {
            var order = new Order();
            int itemCount = rnd.Next(2, 6);
            for (int it = 0; it < itemCount; it++)
                order.Items.Add(new OrderItem { Product = products[rnd.Next(products.Count)] });
            user.Orders.Add(order);
        }
        if (rnd.NextDouble() < 0.5)
            for (int r = 0; r < rnd.Next(1, 4); r++)
                user.Reviews.Add(new Review { Text = $"Review {r} for user {u}" });
        db.Users.Add(user);
    }
    await db.SaveChangesAsync();
}

static void Log(string msg) => Console.WriteLine(msg);

// --- DTO / DTO ---
public record UserBriefDto(int Id, string FullName, string Email, int OrderCount);

// --- Сущности / Entities ---
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

// --- DbContext со счётчиком команд / DbContext with a command counter ---
public class ShopDbContext : DbContext
{
    private readonly SqliteConnection _conn;
    public int CommandCount { get; private set; }

    public ShopDbContext(SqliteConnection conn) => _conn = conn;

    public DbSet<User> Users => Set<User>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Review> Reviews => Set<Review>();

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlite(_conn);
        // Логируем только команды БД и считаем их / log only DB commands and count them
        optionsBuilder.LogTo(msg =>
        {
            if (msg.Contains("Executed DbCommand"))
                CommandCount++;
        }, new[] { DbLoggerCategory.Database.Command.Name }, LogLevel.Information);
        optionsBuilder.EnableSensitiveDataLogging();
        optionsBuilder.ConfigureWarnings(w => w.Ignore(RelationalEventId.MultipleCollectionIncludeWarning));
    }

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.Entity<User>().HasMany(u => u.Orders).WithOne(o => o.User).HasForeignKey(o => o.UserId);
        b.Entity<Order>().HasMany(o => o.Items).WithOne(i => i.Order).HasForeignKey(i => i.OrderId);
        b.Entity<OrderItem>().HasOne(i => i.Product).WithMany().HasForeignKey(i => i.ProductId);
        b.Entity<User>().HasMany(u => u.Reviews).WithOne().HasForeignKey(r => r.UserId);
    }
}
```

**Разбор по строкам (почему так).** Сценарий A намеренно использует `Entry(u).Collection(x => x.Orders).LoadAsync()` внутри `foreach`, потому что в EF Core 8 lazy loading по умолчанию выключен, и простое чтение `u.Orders` без `Include` вернуло бы пустую коллекцию `[]`, а не породило N+1. Явная загрузка честно воспроизводит «обращение к связанным данным в цикле порождает N запросов» — это концептуально тот же N+1, что разобран в уроке, только механизм чуть иной. Счётчик `db.CommandCount` подсчитывает строки `Executed DbCommand` в логе, что соответствует реальному числу запросов, — урок требует «измерять, а не угадывать». Сценарий B добавляет `.Include(u => u.Orders)`: EF Core транслирует это в один `LEFT JOIN` (или `JOIN`), и лог показывает ровно одну команду — это прямое исправление N+1 из урока. Сценарий C — проекция в record `UserBriefDto` с агрегатом `u.Orders.Count`: EF Core понимает, что `Count` по навигационной коллекции — это серверный `SELECT COUNT(*) FROM Orders WHERE UserId = u.Id`, и генерирует один запрос без JOIN к заказам; это «лучшая практика» из урока — проецируйте только нужные поля. Сценарий D1 грузит две коллекции (`Orders.Items` через `ThenInclude` и `Reviews`) одним запросом — возникает декартов взрыв: число материализованных позиций огромно из-за дублирования строк (произведение `1 × заказы × позиции × отзывы`). D2 добавляет `.AsSplitQuery()`: EF Core разбивает запрос на несколько — по одному на каждую коллекцию — и дублирования строк нет, ценой дополнительных round-trips (это ровно trade-off из урока). Сценарий E использует `AsNoTrackingWithIdentityResolution()`: граф `Orders → Items → Product` содержит много ссылок на одни и те же 10 продуктов; обычный `AsNoTracking` создал бы отдельный экземпляр `Product` под каждую ссылку, а identity resolution гарантирует один экземпляр по ключу — при этом `ChangeTracker.Entries().Count()` остаётся `0`, потому что сущности не отслеживаются. Сбор продуктов в `HashSet<Product>` с `ReferenceEqualityComparer` доказывает отсутствие дублей по ссылке. Конфигурация контекста игнорирует warning `MultipleCollectionIncludeWarning`, чтобы D1 не пугал предупреждением (в проде — наоборот, не игнорируйте). Использование SQLite InMemory с удерживаемым `SqliteConnection` — стандартный приём для учебных/тестовых сценариев, дающий реальные SQL-логи без внешней БД. Все сущности используют collection expressions `[]` и nullable-навигации `null!` — idiomatic C# 12.

#### Задания на углубление (бонус)
1. Переведите контекст на `QueryTrackingBehavior.NoTracking` по умолчанию (в `OnConfiguring` через `UseQueryTrackingBehavior`) и покажите, что теперь `AsNoTracking()` избыточен, а tracked-запросы нужно явно помечать `AsTracking()`. Объясните, когда такой дефолт удобен, а когда опасен.
2. Добавьте сценарий F: сравните производительность проекции (D) и `Include` + клиентский маппинг в DTO на 5000 пользователей (увеличьте seed). Измерьте время через `Stopwatch` и пиковую память через `GC.GetTotalMemory`. Сделайте вывод, когда проекция выигрывает.
3. Реализуйте «частичный граф» через проекцию со вложенным DTO: `Select(u => new UserWithOrdersDto(u.FullName, u.Orders.Select(o => new OrderBriefDto(o.Id, o.Items.Count)).ToList()))` — покажите, что EF Core генерирует один запрос с подзапросами, без N+1 и без декартова взрыва.
4. Исследуйте `QuerySplittingMode` (`AsSingleQuery`/`AsSplitQuery`/`UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery)` глобально) и опишите в `NOTES.md`, когда глобальный split-дефолт вреден (например, для небольших запросов).

---

## Statement in English / Постановка на английском

#### Context & motivation
You are a backend engineer on “ShopHub”, a service that exposes users, orders, order items, products and reviews. The team has noticed that the admin dashboard and the public “list users” API endpoint have started to slow down as the database has grown: with 5000 users the page takes several seconds to load, the PostgreSQL logs show hundreds of identical queries, and the API container regularly hits its memory limit. The analysis revealed exactly the three classes of problems covered in lesson M12-L06: (1) almost every GET request goes through tracked entities even though nothing is being edited, so the ChangeTracker bloats on every request; (2) several places touch navigation properties inside a `foreach` loop, producing the classic N+1; (3) there is a “heavy” report that, through three collection `Include`s, loads a cartesian explosion of thousands of duplicated rows. Your task is to rewrite the read layer so that it follows the lesson: use `AsNoTracking()` by default, project into DTOs, use `Include`/`ThenInclude` instead of touching navigations in a loop, apply `AsSplitQuery()` against cartesian explosion, and use `AsNoTrackingWithIdentityResolution()` for complex read-only graphs. You must not just “make it work” but prove the improvement through SQL logs: count queries before and after, show that tracked entities no longer accumulate in the ChangeTracker, and explain why each change works. This exercise is not about syntax — that is trivial — it is about building the habit of measuring and choosing the right tool: projection versus `Include`, `AsNoTracking` versus `AsNoTrackingWithIdentityResolution`, a single JOIN versus split queries. The lesson insists: “do not guess performance — measure it.” That is why this homework makes SQL logging and `ChangeTracker.Entries().Count()` output after every scenario mandatory.

#### What to do (step by step)
1. Create a new project: `dotnet new console -n ShopHub.ReadQueries -o ShopHub.ReadQueries` inside `modules/M12/homework/M12-L06/`. Target framework `net8.0`. Add packages: `dotnet add package Microsoft.EntityFrameworkCore --version 8.*` and `dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 8.*` (we use SQLite InMemory so we do not have to run a Postgres instance, but the queries and SQL logs are real).
2. In `Program.cs` (top-level statements) define entities `User`, `Order`, `OrderItem`, `Product`, `Review` exactly following the lesson model: `User` has `Id`, `FullName`, `LastName`, `Email`, `IsActive`, collections `Orders` and `Reviews` (use collection expressions `[]` for initialisation); `Order` — `Id`, `UserId`, `User`, `Items`; `OrderItem` — `Id`, `OrderId`, `ProductId`, `Order`, `Product`; `Product` — `Id`, `Name`; `Review` — `Id`, `UserId`, `Text`. Use nullable reference annotations (`null!`) for required navigations.
3. Create `ShopDbContext : DbContext` with `DbSet<User>`, `DbSet<Order>`, `DbSet<OrderItem>`, `DbSet<Product>`, `DbSet<Review>`. In `OnConfiguring` enable SQLite InMemory (`UseSqlite("DataSource=:memory:")` with a shared connection kept alive; see hints) and also enable SQL logging: `optionsBuilder.LogTo(msg => Console.WriteLine(msg), new[] { DbLoggerCategory.Database.Command.Name }, LogLevel.Information).EnableSensitiveDataLogging();`. In `OnModelCreating` configure keys and relationships (one-to-many User→Orders, Order→Items, OrderItem→Product, User→Reviews).
4. In a `SeedAsync(DbContext db)` method populate the database: 50 users, each with 1–10 orders, each order with 2–5 items, 10 unique products (products will be reused — this matters for `AsNoTrackingWithIdentityResolution`), and 1–3 reviews for some users. This produces a graph on which N+1 and cartesian explosion become visible.
5. Implement SCENARIO A — “bad code” (baseline): a method `RunNPlusOneAsync(db)` that loads all active users via `db.Users.AsNoTracking().ToListAsync()` (deliberately without `Include`) and then, in a `foreach`, prints `u.Orders.Count`. In the SQL log you should see 1 + N queries. Print `db.ChangeTracker.Entries().Count()` — it will equal the number of users (if you remove `AsNoTracking`) or zero (if you keep it). Record both cases in a comment.
6. Implement SCENARIO B — fixing N+1 with `Include`: a method `RunFixedWithIncludeAsync(db)` — the same loop, but the query uses `.Include(u => u.Orders)`. The SQL log must show a single JOIN query.
7. Implement SCENARIO C — projection into a DTO: a method `RunProjectionAsync(db)` that returns `List<UserBriefDto>` via `.Select(u => new UserBriefDto(u.Id, u.FullName, u.Email, u.Orders.Count))`. The DTO is `public record UserBriefDto(int Id, string FullName, string Email, int OrderCount);`. The log must show a single query selecting exactly 4 columns, with no JOIN to orders (the `Count` aggregate is computed on the DB side).
8. Implement SCENARIO D — cartesian explosion and `AsSplitQuery()`: a method `RunCartesianAsync(db)` loads users with `.Include(u => u.Orders).ThenInclude(o => o.Items).ThenInclude(i => i.Product).Include(u => u.Reviews)` without split. A method `RunSplitAsync(db)` does the same but with `.AsSplitQuery()`. Compare the number of rows and round-trips in the log (print the count of materialised rows via a counter).
9. Implement SCENARIO E — `AsNoTrackingWithIdentityResolution()`: a method `RunIdentityResolutionAsync(db)` loads the graph `Orders → Items → Product` and demonstrates that the same `Product` (by key) is not duplicated in memory: collect every `Product` from the graph into a `HashSet<Product>` by reference identity and show that there are no duplicate references, while `db.ChangeTracker.Entries().Count() == 0`.
10. In `Main` call all scenarios in order; after each one print a separator and a short report: how many SQL queries ran (count lines of category `Microsoft.EntityFrameworkCore.Database.Command`), how many rows were materialised, how many entries are in the ChangeTracker.
11. Run: `dotnet run --project ShopHub.ReadQueries`. Make sure the output shows: (a) N+1 → 1+N queries, (b) after `Include` → 1 query, (c) projection → 1 query with a narrow SELECT, (d) without split → one “fat” result, with split → several queries, (e) identity resolution → 0 in the tracker and 0 duplicate products.
12. In a `NOTES.md` file next to the project, record a table: scenario | SQL query count | row count | ChangeTracker entries | comment (why).

#### Requirements
- Target runtime: .NET 8, C# 12. Use top-level statements in `Program.cs`, collection expressions (`[]`) for collection initialisation, a record for the DTO, file-scoped namespaces, nullable reference types enabled (`<Nullable>enable</Nullable>`).
- EF Core 8. Provider — SQLite InMemory (via a `Microsoft.Data.Sqlite` connection with `DataSource=:memory:` and `Mode=Memory`, kept open for the whole run). This gives real SQL logs without an external database.
- Every read query must use `AsNoTracking()` or `AsNoTrackingWithIdentityResolution()`. No GET scenario may leave entries in the `ChangeTracker` — this is verified by printing `db.ChangeTracker.Entries().Count()`.
- SQL command logging must be enabled (`LogTo` + category `DbLoggerCategory.Database.Command`) so you can count queries and see `JOIN`/`LEFT JOIN`/`COUNT(*)`.
- The N+1 scenario must be implemented deliberately (the “bad” variant) and then fixed — so the student sees both logs and the difference. It is forbidden to “just write it correctly” without demonstrating the problem.
- The projection (scenario C) must use the `u.Orders.Count` aggregate on the database side, not a client-side `.Count()` after materialisation. Verify in the SQL that there is no `JOIN` and there is a `SELECT COUNT(*)` subquery.
- `AsSplitQuery()` must be applied only where there are genuinely multiple collections (`Orders.Items` + `Reviews`). Do not apply split where it is not needed — that is wasted round-trips.
- The code must compile without warnings (`TreatWarningsAsErrors` is acceptable). Use `await` correctly; `ConfigureAwait` is not required (console application).
- `NOTES.md` must contain a comparison table for all five scenarios by query count and tracker entries.

#### Pitfalls
- **AsNoTracking is not free magic**: the method removes the entity from the ChangeTracker but does not automatically make SQL faster — the speedup comes from lower memory usage and the absence of snapshot diffing. The crucial point is that `SaveChanges` will not persist such an entity; do not try to edit an `AsNoTracking` entity and call `SaveChanges`, as the lesson warns. If you need to update, use a tracked query or an explicit `Update()`.
- **AsNoTrackingWithIdentityResolution vs AsNoTracking**: plain `AsNoTracking` while loading a graph `Orders → Items → Product` may create several different instances of the same `Product` (by key), because each `OrderItem` references its own materialised Product. `AsNoTrackingWithIdentityResolution` guarantees one instance per key — but still does not track changes. Choose it for complex read-only graphs with reused entities.
- **Projections are not tracked automatically**: even if you `Select` a whole entity (`new User(...)` from fields), the result does not enter the ChangeTracker. The lesson explicitly calls this out as an advantage. But do not abuse “project every field” projection — that is just `AsNoTracking` with extra work.
- **N+1 hides**: if you have an `Include` on one collection but then touch a second navigation in a loop (for example `u.Reviews`), you get N+1 on the second collection even though the first `Include` worked. Always check every navigation you touch inside a loop.
- **Cartesian explosion is a product**: 1 user × 10 orders × 20 items = 200 rows in a single result set, even if you only need the user’s data once. `AsSplitQuery` breaks this into 3 queries: users + orders + items (and products), each small. The trade-off from the lesson: more round-trips, but less duplicated data and less memory — evaluate it against your network topology.
- **Lazy loading is OFF by default in EF Core**: if you have not configured proxy lazy loading, touching `u.Orders` without an `Include`, after materialisation, simply returns `null` (or an empty collection if initialised with `[]`). So a strict “N+1” only happens with lazy loading enabled. In this homework we deliberately model “touching related data in a loop” as a separate query through `db.Entry(u).Collection(x => x.Orders).LoadAsync()` or simply show that without `Include` the data is absent — make clear in the code which mechanism you are demonstrating. The key point is to show the problem: “touching related data in a loop spawns N separate calls to the database”.
- **SQL logging is noisy**: enable only the `Database.Command` category, otherwise the log fills up with infrastructure messages. `EnableSensitiveDataLogging` is only for learning/local use; in production it may log parameter values (PII).
- **SQLite InMemory and connection lifetime**: the InMemory database lives as long as the connection is open. If you create a `DbContext` with `:memory:` each time with a new connection, the database will be empty. Keep one `SqliteConnection` open for the whole run and pass it into `UseSqlite(conn)`.

#### Acceptance criteria
- [ ] The project `ShopHub.ReadQueries` is created with a single `dotnet new console` command, compiles on .NET 8 / C# 12 without warnings.
- [ ] Entities `User`, `Order`, `OrderItem`, `Product`, `Review` are defined exactly as in the lesson model, with collection expressions and nullable navigations.
- [ ] `ShopDbContext` uses SQLite InMemory with a kept-open connection and enables SQL logging of category `Database.Command`.
- [ ] The `SeedAsync` method populates a graph: ≥50 users, orders, items, reused products, reviews.
- [ ] Scenario A (N+1) prints ≥1+N SQL queries in the log and is explicitly marked as “bad”.
- [ ] Scenario B fixes N+1 via `Include(u => u.Orders)` — exactly 1 JOIN query.
- [ ] Scenario C uses a projection into `UserBriefDto` with the `u.Orders.Count` aggregate on the DB side — the SQL contains `SELECT COUNT(*)` and no JOIN to orders.
- [ ] Scenario D demonstrates cartesian explosion (without split) and its elimination via `AsSplitQuery()` — the log shows several queries instead of one “fat” query.
- [ ] Scenario E uses `AsNoTrackingWithIdentityResolution()` and proves the absence of duplicate `Product` references while `ChangeTracker.Entries().Count() == 0`.
- [ ] After every scenario, `ChangeTracker.Entries().Count() == 0` is printed (for read-only scenarios).
- [ ] `NOTES.md` contains a comparison table for all 5 scenarios: query count, row count, tracker entries.
- [ ] `SaveChanges` is never called in a read-only scenario (and nowhere is it expected that an `AsNoTracking` entity will be saved).
- [ ] The code uses top-level statements, a file-scoped namespace, a record DTO, collection expressions `[]`.
- [ ] `dotnet run` finishes without exceptions and prints all 5 report sections.
- [ ] Code comments include RU+EN explanations for each `AsNoTracking`/`Include`/`AsSplitQuery`.

#### Hints (without giving the answer)
- For SQLite InMemory keep a `SqliteConnection` open: `using var conn = new SqliteConnection("DataSource=:memory:"); conn.Open();` and pass `optionsBuilder.UseSqlite(conn)`. Do not put `DataSource=:memory:` directly in the context string — every new connection will give an empty database.
- To count SQL queries programmatically, wrap the `LogTo` delegate so it increments a counter on lines that contain `Executed DbCommand`.
- To demonstrate “N+1 without lazy loading” use explicit loading inside the loop: `await db.Entry(u).Collection(x => x.Orders).LoadAsync();` — this spawns one query per user and clearly shows N+1 in the log.
- The `u.Orders.Count` aggregate in an EF Core projection is translated to a `SELECT COUNT(*) FROM Orders WHERE UserId = u.Id` subquery — make sure the SQL has no client-side `JOIN`.
- To show cartesian explosion quantitatively, materialise `ToList()` and print `result.Count` — that is the number of rows in the flat result, not the number of users.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 / EF Core 8 — ShopHub.ReadQueries / Program.cs
// Reference solution for HW M12-L06: AsNoTracking, projections, Include/ThenInclude, N+1, split queries.

using Microsoft.Data.Sqlite;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Diagnostics;
using System.Collections.Generic;

// Keep the connection open for the whole run — the InMemory DB lives while the connection is open.
using var conn = new SqliteConnection("DataSource=:memory:");
conn.Open();

using var db = new ShopDbContext(conn);
await db.Database.EnsureDeletedAsync();
await db.Database.EnsureCreatedAsync();
await SeedAsync(db);

Log("=== SCENARIO A: N+1 (bad code) ===");
await RunNPlusOneAsync(db);

Log("=== SCENARIO B: fix via Include ===");
await RunFixedWithIncludeAsync(db);

Log("=== SCENARIO C: projection into DTO ===");
await RunProjectionAsync(db);

Log("=== SCENARIO D1: cartesian explosion (no split) ===");
await RunCartesianAsync(db);

Log("=== SCENARIO D2: AsSplitQuery ===");
await RunSplitAsync(db);

Log("=== SCENARIO E: AsNoTrackingWithIdentityResolution ===");
await RunIdentityResolutionAsync(db);

return;

// --- A: deliberate N+1 via explicit load in a loop ---
static async Task RunNPlusOneAsync(ShopDbContext db)
{
    var users = await db.Users.AsNoTracking().Where(u => u.IsActive).ToListAsync();
    int queriesBefore = db.CommandCount;
    foreach (var u in users)
    {
        // Explicit loading of orders inside the loop → one query per user → N+1.
        await db.Entry(u).Collection(x => x.Orders).LoadAsync();
        Console.WriteLine($"  {u.FullName}: {u.Orders.Count} orders");
    }
    Log($"  SQL queries: {db.CommandCount - queriesBefore} (1 + {users.Count})");
    Log($"  ChangeTracker entries: {db.ChangeTracker.Entries().Count()}  (AsNoTracking → 0)");
}

// --- B: fix N+1 via Include → a single JOIN ---
static async Task RunFixedWithIncludeAsync(ShopDbContext db)
{
    int queriesBefore = db.CommandCount;
    var users = await db.Users
        .AsNoTracking()                    // read with no tracking
        .Include(u => u.Orders)            // eager load orders in one JOIN
        .Where(u => u.IsActive)
        .ToListAsync();
    foreach (var u in users)
        Console.WriteLine($"  {u.FullName}: {u.Orders.Count} orders");
    Log($"  SQL queries: {db.CommandCount - queriesBefore}  (should be 1)");
    Log($"  ChangeTracker entries: {db.ChangeTracker.Entries().Count()}  (AsNoTracking → 0)");
}

// --- C: projection into a DTO with a DB-side aggregate ---
static async Task RunProjectionAsync(ShopDbContext db)
{
    int queriesBefore = db.CommandCount;
    List<UserBriefDto> dtos = await db.Users
        .AsNoTracking()
        .Where(u => u.IsActive)
        .OrderBy(u => u.LastName)
        .Select(u => new UserBriefDto(u.Id, u.FullName, u.Email, u.Orders.Count)) // COUNT(*) on the DB side
        .ToListAsync();
    Log($"  SQL queries: {db.CommandCount - queriesBefore}  (1, narrow SELECT, no JOIN to orders)");
    Log($"  DTO records: {dtos.Count}; sample: {dtos[0]}");
    Log($"  ChangeTracker entries: {db.ChangeTracker.Entries().Count()}  (projection → 0)");
}

// --- D1: cartesian explosion without split ---
static async Task RunCartesianAsync(ShopDbContext db)
{
    int queriesBefore = db.CommandCount;
    var rows = await db.Users
        .AsNoTracking()
        .Include(u => u.Orders).ThenInclude(o => o.Items).ThenInclude(i => i.Product)
        .Include(u => u.Reviews)            // second collection → cartesian explosion
        .Where(u => u.IsActive)
        .ToListAsync();
    int materializedRows = rows.SelectMany(u => u.Orders).SelectMany(o => o.Items).Count();
    Log($"  SQL queries: {db.CommandCount - queriesBefore}  (1 \"fat\" JOIN)");
    Log($"  Items materialised: {materializedRows}  (row duplication due to JOIN)");
}

// --- D2: AsSplitQuery — several queries instead of one ---
static async Task RunSplitAsync(ShopDbContext db)
{
    int queriesBefore = db.CommandCount;
    var rows = await db.Users
        .AsNoTracking()
        .AsSplitQuery()                     // split into several queries
        .Include(u => u.Orders).ThenInclude(o => o.Items).ThenInclude(i => i.Product)
        .Include(u => u.Reviews)
        .Where(u => u.IsActive)
        .ToListAsync();
    int materializedRows = rows.SelectMany(u => u.Orders).SelectMany(o => o.Items).Count();
    Log($"  SQL queries: {db.CommandCount - queriesBefore}  (several — one per collection)");
    Log($"  Items materialised: {materializedRows}  (no row duplication)");
}

// --- E: AsNoTrackingWithIdentityResolution — one Product by key, not tracked ---
static async Task RunIdentityResolutionAsync(ShopDbContext db)
{
    var graph = await db.Users
        .AsNoTrackingWithIdentityResolution()   // no tracking, but dedup by key
        .Include(u => u.Orders).ThenInclude(o => o.Items).ThenInclude(i => i.Product)
        .Where(u => u.IsActive)
        .ToListAsync();

    var products = new HashSet<Product>(ReferenceEqualityComparer.Instance);
    int productRefs = 0;
    foreach (var u in graph)
        foreach (var o in u.Orders)
            foreach (var i in o.Items)
            {
                productRefs++;
                products.Add(i.Product);
            }
    Log($"  Product references in graph: {productRefs}; unique instances: {products.Count}");
    Log($"  ChangeTracker entries: {db.ChangeTracker.Entries().Count()}  (should be 0)");
}

// --- Seed: 50 users, orders, items, reused products, reviews ---
static async Task SeedAsync(ShopDbContext db)
{
    var rnd = new Random(42);
    var products = new List<Product>();
    for (int i = 1; i <= 10; i++)
        products.Add(new Product { Id = i, Name = $"Product-{i}" });
    db.Products.AddRange(products);

    for (int u = 1; u <= 50; u++)
    {
        var user = new User
        {
            FullName = $"User {u}",
            LastName = $"Last{u}",
            Email = $"user{u}@shop.dev",
            IsActive = u % 10 != 0 // ~90% active
        };
        int orderCount = rnd.Next(1, 11);
        for (int o = 1; o <= orderCount; o++)
        {
            var order = new Order();
            int itemCount = rnd.Next(2, 6);
            for (int it = 0; it < itemCount; it++)
                order.Items.Add(new OrderItem { Product = products[rnd.Next(products.Count)] });
            user.Orders.Add(order);
        }
        if (rnd.NextDouble() < 0.5)
            for (int r = 0; r < rnd.Next(1, 4); r++)
                user.Reviews.Add(new Review { Text = $"Review {r} for user {u}" });
        db.Users.Add(user);
    }
    await db.SaveChangesAsync();
}

static void Log(string msg) => Console.WriteLine(msg);

// --- DTO ---
public record UserBriefDto(int Id, string FullName, string Email, int OrderCount);

// --- Entities ---
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

// --- DbContext with a command counter ---
public class ShopDbContext : DbContext
{
    private readonly SqliteConnection _conn;
    public int CommandCount { get; private set; }

    public ShopDbContext(SqliteConnection conn) => _conn = conn;

    public DbSet<User> Users => Set<User>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Review> Reviews => Set<Review>();

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlite(_conn);
        // Log only DB commands and count them
        optionsBuilder.LogTo(msg =>
        {
            if (msg.Contains("Executed DbCommand"))
                CommandCount++;
        }, new[] { DbLoggerCategory.Database.Command.Name }, LogLevel.Information);
        optionsBuilder.EnableSensitiveDataLogging();
        optionsBuilder.ConfigureWarnings(w => w.Ignore(RelationalEventId.MultipleCollectionIncludeWarning));
    }

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.Entity<User>().HasMany(u => u.Orders).WithOne(o => o.User).HasForeignKey(o => o.UserId);
        b.Entity<Order>().HasMany(o => o.Items).WithOne(i => i.Order).HasForeignKey(i => i.OrderId);
        b.Entity<OrderItem>().HasOne(i => i.Product).WithMany().HasForeignKey(i => i.ProductId);
        b.Entity<User>().HasMany(u => u.Reviews).WithOne().HasForeignKey(r => r.UserId);
    }
}
```

**Line-by-line walk-through (why it is this way).** Scenario A deliberately uses `Entry(u).Collection(x => x.Orders).LoadAsync()` inside a `foreach`, because in EF Core 8 lazy loading is off by default and a plain read of `u.Orders` without an `Include` would return the empty `[]` collection rather than spawning N+1. Explicit loading honestly reproduces “touching related data in a loop spawns N queries” — this is conceptually the same N+1 discussed in the lesson, only the mechanism is slightly different. The `db.CommandCount` counter counts `Executed DbCommand` lines in the log, which corresponds to the real number of queries — the lesson demands “measure, do not guess”. Scenario B adds `.Include(u => u.Orders)`: EF Core translates this into a single `LEFT JOIN` (or `JOIN`), and the log shows exactly one command — a direct fix for N+1 from the lesson. Scenario C is a projection into the `UserBriefDto` record with the `u.Orders.Count` aggregate: EF Core understands that `Count` over a navigation collection is a server-side `SELECT COUNT(*) FROM Orders WHERE UserId = u.Id` and generates a single query without a JOIN to orders; this is the “best practice” from the lesson — project only the fields you need. Scenario D1 loads two collections (`Orders.Items` via `ThenInclude` and `Reviews`) in one query — a cartesian explosion arises: the number of materialised items is huge because rows are duplicated (the product `1 × orders × items × reviews`). D2 adds `.AsSplitQuery()`: EF Core breaks the query into several — one per collection — and there is no row duplication, at the cost of extra round-trips (exactly the trade-off from the lesson). Scenario E uses `AsNoTrackingWithIdentityResolution()`: the graph `Orders → Items → Product` contains many references to the same 10 products; plain `AsNoTracking` would create a separate `Product` instance for every reference, while identity resolution guarantees one instance per key — at the same time `ChangeTracker.Entries().Count()` stays `0`, because the entities are not tracked. Collecting products into a `HashSet<Product>` with `ReferenceEqualityComparer` proves there are no duplicate references. The context configuration ignores the `MultipleCollectionIncludeWarning` so D1 does not scare you with a warning (in production — do the opposite, do not ignore it). Using SQLite InMemory with a kept-alive `SqliteConnection` is a standard technique for learning/test scenarios, giving real SQL logs without an external database. All entities use collection expressions `[]` and nullable navigations `null!` — idiomatic C# 12.

#### Going deeper (bonus)
1. Switch the context to `QueryTrackingBehavior.NoTracking` by default (in `OnConfiguring` via `UseQueryTrackingBehavior`) and show that now `AsNoTracking()` is redundant, while tracked queries must be explicitly marked with `AsTracking()`. Explain when such a default is convenient and when it is dangerous.
2. Add scenario F: compare the performance of the projection (C) versus `Include` + client-side mapping into a DTO on 5000 users (increase the seed). Measure time with `Stopwatch` and peak memory with `GC.GetTotalMemory`. Conclude when the projection wins.
3. Implement a “partial graph” via a projection with a nested DTO: `Select(u => new UserWithOrdersDto(u.FullName, u.Orders.Select(o => new OrderBriefDto(o.Id, o.Items.Count)).ToList()))` — show that EF Core generates a single query with subqueries, without N+1 and without cartesian explosion.
4. Investigate `QuerySplittingMode` (`AsSingleQuery`/`AsSplitQuery`/`UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery)` globally) and describe in `NOTES.md` when a global split default is harmful (for example, for small queries).

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `ShopHub.ReadQueries` в `modules/M12/homework/M12-L06/` компилируется на .NET 8 / C# 12.
- [ ] (RU) Реализованы все 5 сценариев (A–E) с SQL-логированием и подсчётом `ChangeTracker.Entries()`.
- [ ] (RU) Сценарий N+1 показан как «плохой» и исправлен через `Include`.
- [ ] (RU) Проекция использует агрегат `u.Orders.Count` на стороне БД.
- [ ] (RU) `AsSplitQuery()` применён только там, где несколько коллекций.
- [ ] (RU) `AsNoTrackingWithIdentityResolution()` доказывает отсутствие дублей `Product`.
- [ ] (RU) Файл `NOTES.md` содержит таблицу сравнения сценариев.
- [ ] (EN) Project `ShopHub.ReadQueries` under `modules/M12/homework/M12-L06/` compiles on .NET 8 / C# 12.
- [ ] (EN) All 5 scenarios (A–E) are implemented with SQL logging and `ChangeTracker.Entries()` counting.
- [ ] (EN) The N+1 scenario is shown as “bad” and fixed with `Include`.
- [ ] (EN) The projection uses the `u.Orders.Count` aggregate on the DB side.
- [ ] (EN) `AsSplitQuery()` is applied only where there are multiple collections.
- [ ] (EN) `AsNoTrackingWithIdentityResolution()` proves there are no duplicate `Product` instances.
- [ ] (EN) `NOTES.md` contains the scenario comparison table.

#### Ресурсы / Resources
- [Microsoft Learn — Tracking / https://learn.microsoft.com/ef/core/querying/tracking](https://learn.microsoft.com/ef/core/querying/tracking)
- [Microsoft Learn — No-tracking / https://learn.microsoft.com/ef/core/querying/tracking#no-tracking-queries](https://learn.microsoft.com/ef/core/querying/tracking#no-tracking-queries)
- [Microsoft Learn — Related data / Include / https://learn.microsoft.com/ef/core/querying/related-data](https://learn.microsoft.com/ef/core/querying/related-data)
- [Microsoft Learn — Split queries / https://learn.microsoft.com/ef/core/querying/single-split-queries](https://learn.microsoft.com/ef/core/querying/single-split-queries)
- [Microsoft Learn — Logging / https://learn.microsoft.com/ef/core/logging-events-diagnostics/](https://learn.microsoft.com/ef/core/logging-events-diagnostics/)
- [Microsoft Learn — Performance / https://learn.microsoft.com/ef/core/performance/](https://learn.microsoft.com/ef/core/performance/)

---
[← К уроку M12-L06](lesson-M12-L06-tracking-projections-include.md) | [⬆ К модулю M12](../README.md) | [Следующее ДЗ →](homework-M12-L07-transactions-savechanges.md)
