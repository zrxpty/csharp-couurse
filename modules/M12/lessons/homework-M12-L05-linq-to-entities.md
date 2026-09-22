---
[← К уроку M12-L05](lesson-M12-L05-linq-to-entities.md) | [⬆ К модулю M12](../README.md) | [Следующее ДЗ →](homework-M12-L06-tracking-projections-include.md)
---

### Домашнее задание M12-L05: LINQ to Entities, переводы в SQL / Homework M12-L05: LINQ to Entities, SQL translation

**Урок / Lesson:** M12-L05
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться писать переводимые в SQL запросы LINQ to Entities на C# 12 / .NET 8: строить цепочки `IQueryable<T>`, проектировать результаты в DTO, осознанно использовать `Include`/`ThenInclude` и `AsSplitQuery()`, включать и читать лог сгенерированного SQL, отличать переводимые выражения от непереводимых и избегать клиентского вычисления. (EN) Learn to write SQL-translatable LINQ to Entities queries on C# 12 / .NET 8: build `IQueryable<T>` chains, project results into DTOs, use `Include`/`ThenInclude` and `AsSplitQuery()` deliberately, enable and read generated SQL logs, distinguish translatable from untranslatable expressions, and avoid client-side evaluation.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит модель «LINQ как спецификация запроса», объясняет построение дерева выражения, материализацию, проекцию через `Select`, `Include`/`ThenInclude` и опасность декартова произведения, а также запрет клиентского вычисления с EF Core 3.0. Это задание заставляет применить каждую из этих идей на работающем коде и увидеть сгенерированный SQL своими глазами.
(EN) The lesson introduces the "LINQ as a query specification" model, expression-tree building, materialization, projection via `Select`, `Include`/`ThenInclude`, the Cartesian explosion danger, and the ban on client-side evaluation since EF Core 3.0. This homework makes you apply every one of those ideas in runnable code and inspect the generated SQL with your own eyes.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы присоединяетесь к команде, которая поддерживает небольшой интернет-магазин на .NET 8 и EF Core 8. В кодовой базе накопилось несколько «толстых» запросов: кто-то тянет `SELECT *` ради одного поля, кто-то вызывает собственный метод `IsVip` внутри `Where` и получает `InvalidOperationException`, кто-то каскадом из четырёх `Include` коллекций создаёт декартово произведение и жалуется на медленный каталог. Лид попросил вас навести порядок в слое доступа к данным и заодно доказать, что каждый запрос действительно переводится в SQL, а не досчитывается в памяти процесса.

Ваша задача — построить консольное приложение `ShopReports`, воспроизвести модель из урока (`Order`, `Customer`, `OrderItem`), наполнить базу тестовыми данными и реализовать набор отчётных запросов, каждый из которых должен быть переводимым, проектируемым в DTO и подтверждён логом SQL. Вы не пишете сырой SQL и не используете хранимки: всё через `IQueryable<T>`. Параллельно вы фиксируете в комментариях, какой SQL ожидается, и сравниваете ожидание с реальным логом — это и есть тренировка «видеть SQL».

Задание намеренно сконструировано так, чтобы каждый раздел урока всплыл в практике: вы столкнётесь с `Where`+`OrderBy`+`Select`, с `Include` и `ThenInclude`, с агрегацией `GroupBy`/`Sum`, с ловушкой клиентского вычисления и с разрастанием JOIN'ов. В конце вы пишете короткий разбор: какой запрос какой SQL породил и почему именно такой.

#### Что нужно сделать (пошагово)
1. Создайте решение и проект:
   ```
   dotnet new sln -n ShopReports
   dotnet new console -n ShopReports.App -o src/ShopReports.App --framework net8.0
   dotnet sln add src/ShopReports.App/ShopReports.App.csproj
   cd src/ShopReports.App
   dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 8.0.*
   dotnet add package Microsoft.EntityFrameworkCore.Design --version 8.0.*
   ```
   Используйте SQLite, как в уроке: файл `shop.db`, провайдер `UseSqlite`.

2. В файле `Models.cs` опишите классы `Order`, `Customer`, `OrderItem` ровно в той форме, что в уроке: навигационные свойства с инициализаторами (`= null!`, `= new()`), `Status` со значениями `"New"`, `"Paid"`, `"Cancelled"`.

3. В `ShopContext.cs` объявите `DbSet<Order> Orders` и `DbSet<Customer> Customers` через `Set<T>()`, переопределите `OnConfiguring` и включите логирование SQL: `.LogTo(Console.WriteLine, LogLevel.Information)`. Подключите `UseSqlite("Data Source=shop.db")`.

4. В `Program.cs` (top-level statements) при первом запуске создавайте и заполняйте базу: `db.Database.EnsureDeleted(); db.Database.EnsureCreated();` и добавьте 4–5 клиентов, 10–15 заказов с разными статусами и суммами, 2–3 позиции в каждом заказе. Сохраните через `SaveChangesAsync()`.

5. Реализуйте отчёт №1 «Оплаченные крупные заказы»: верните DTO `PaidOrderDto { int Id; decimal Total; string CustomerName; }` для заказов со `Status == "Paid"` и `Total > 1000`, отсортированных по убыванию `Total`. Используйте `Where` → `OrderByDescending` → `Select` → `ToListAsync`.

6. Реализуйте отчёт №2 «Детали одного заказа с клиентом»: `Include(o => o.Customer)` + `FirstOrDefaultAsync(o => o.Id == <id>)`. Зафиксируйте в комментарии ожидаемый SQL (`LEFT JOIN Customers`).

7. Реализуйте отчёт №3 «Дерево связей»: для заказов с `Total > 500` загрузите `Customer` и коллекцию `Items` через два `Include`, вызовите `AsSplitQuery()` и материализуйте. Запишите в комментарии, почему split-запрос здесь уместен.

8. Реализуйте отчёт №4 «Топ клиентов по выручке»: `GroupBy(o => o.CustomerId)` → `Select(g => new { CustomerId = g.Key, Sum = g.Sum(o => o.Total), Count = g.Count() })` → `OrderByDescending` → `Take(3)` → `ToListAsync`.

9. Реализуйте «анти-пример» №5 в отдельном методе `BadQuery_Untranslatable()`: попытайтесь вызвать собственный метод `bool IsVip(decimal total) => total > 5000;` внутри `Where`. Оберните вызов в `try/catch (InvalidOperationException)` и распечатайте сообщение — продемонстрируйте, что EF Core 3+ бросает исключение, а не уходит в клиентское вычисление.

10. Реализуйте «анти-пример» №6 `BadQuery_CartesianRisk()`: четыре `Include` коллекций без `AsSplitQuery()`. Не запускайте на больших данных — просто покажите структуру и объясните в комментарии риск декартова произведения. Затем покажите исправленную версию со `AsSplitQuery()`.

11. В каждом отчёте над вызовом `ToListAsync`/`FirstOrDefaultAsync` напишите комментарий с ожидаемым SQL. После запуска сравните реальный лог из консоли с ожиданием и добавьте примечание: совпало или нет.

12. Включите предикат-параметр как `Expression<Func<Order, bool>>`: создайте метод `Task<List<PaidOrderDto>> PaidOrdersAsync(ShopContext db, Expression<Func<Order, bool>> predicate)` и передавайте в него `o => o.Status == "Paid" && o.Total > 1000`. Объясните, почему `Expression`, а не `Func`.

13. Запустите приложение (`dotnet run`), соберите вывод SQL из консоли в файл `sql-log.txt` и приложите к сдаче. Убедитесь, что каждый отчёт породил ровно один SQL-запрос (кроме split-запроса — там несколько).

14. В `README.md` проекта напишите таблицу: «Отчёт → ожидаемый SQL → реальный SQL → совпадение (да/нет) → заметка».

#### Требования к решению
- Целевая платформа — .NET 8, язык C# 12: top-level statements в `Program.cs`, коллекционные выражения (`new()` для навигационных свойств, `List<T> items = [..]` при сидировании), pattern matching где уместно, raw string literals `"""..."""` для длинных SQL-комментариев в коде.
- Все запросы остаются `IQueryable<T>` до момента материализации. Запрещено вызывать `AsEnumerable()`, `ToList()` или `foreach` до конца цепочки.
- Все предикаты передаются как `Expression<Func<T, bool>>`, а не `Func<T, bool>`.
- Результаты отчётов спроецированы в DTO/анонимные типы через `Select`; `SELECT *` (возврат полной сущности `Order`) допустим только в отчёте №2, где нужен `Include`.
- Используется `await` и асинхронные расширения EF (`ToListAsync`, `FirstOrDefaultAsync`, `SaveChangesAsync`).
- Логирование SQL включено через `LogTo` в `OnConfiguring`.
- Анти-примеры №5 и №6 явно помечены и защищены (`try/catch` для №5; комментарий-предупреждение для №6).
- Код компилируется без предупреждений (`dotnet build` чист), `dotnet run` выполняется без необработанных исключений (№5 ловится).

#### Тонкости и подводные камни
- **Материализация — это граница.** До `ToListAsync`/`FirstOrDefaultAsync`/`Count`/`foreach` SQL не летит. Если вы случайно поставите `.ToList()` посреди цепочки, всё последующее выполняется в памяти (LINQ to Objects), а не в базе. Следите за типом: после `AsEnumerable()` тип становится `IEnumerable<T>`, и провайдер больше не участвует.
- **Вызов своего метода в `Where` — капкан.** `o => IsVip(o.Total)` не переводится: провайдер не умеет декомпилировать C#. В EF Core 3+ это `InvalidOperationException`, а не тихий pull таблицы. Выносите логику в переводимое выражение (`o => o.Total > 5000`) или в вычисляемый столбец БД.
- **`Func` vs `Expression`.** `Func<Order, bool>` — это скомпилированный делегат, EF не может его разобрать обратно в дерево. `Expression<Func<Order, bool>>` — это дерево выражения, которое EF переводит в SQL. Запомните: для репозиториев и спецификаций — только `Expression`.
- **`Include` коллекций — декартово произведение.** Несколько `Include` коллекций в одном запросе умножают строки: одна позиция `Order` × N `Items` × M `Tags` даёт N×M строк. Используйте `AsSplitQuery()`, чтобы EF выполнил отдельный запрос на каждую коллекцию и собрал результат в памяти.
- **Проекция уменьшает трафик.** `Select(o => new { o.Id, o.Total })` → `SELECT Id, Total`. Без проекции и с `Include` вы получаете `SELECT o.*, c.*`, что тяжело по сети и по памяти.
- **Двойной запрос `Count`+`ToList`.** Если вы вызываете `.Count()` и отдельно `.ToList()`, летит два SQL. Комбинируйте через одну проекцию с агрегатом или возвращайте `Count` и `Items` одним запросом.
- **Логирование — ваш инструмент.** Без `LogTo` вы не увидите, что EF реально сгенерировал. Включайте в dev и тестах, отключайте в prod (или переведите на `Debug` уровень).
- **`FirstOrDefaultAsync` vs `FindAsync`.** `Find` использует tracker и может взять сущность из памяти, а `FirstOrDefaultAsync` всегда идёт в БД. Для отчётов — `FirstOrDefaultAsync`.

#### Критерии приёмки
- [ ] Проект `ShopReports.App` создаётся командами выше, собирается `dotnet build` без предупреждений и ошибок.
- [ ] Используется .NET 8, C# 12, EF Core 8 (пакет `Microsoft.EntityFrameworkCore.Sqlite` 8.0.*).
- [ ] Модель совпадает с уроком: `Order`, `Customer`, `OrderItem` с навигационными свойствами-инициализаторами.
- [ ] `ShopContext.OnConfiguring` содержит `UseSqlite` и `LogTo(Console.WriteLine, LogLevel.Information)`.
- [ ] Сидирование базы выполняется при первом запуске (`EnsureCreated` + 4–5 клиентов, 10–15 заказов, позиции).
- [ ] Отчёт №1 реализован через `Where`→`OrderByDescending`→`Select`→`ToListAsync` и возвращает DTO.
- [ ] Отчёт №2 использует `Include(o => o.Customer)` и `FirstOrDefaultAsync`.
- [ ] Отчёт №3 использует два `Include` + `AsSplitQuery()`; в комментарии объяснён выбор split-запроса.
- [ ] Отчёт №4 использует `GroupBy` + `Sum` + `Count` + `Take(3)`; агрегат переводится в SQL.
- [ ] Анти-пример №5 вызывает непереводимый метод в `Where` и ловит `InvalidOperationException`.
- [ ] Анти-пример №6 показывает риск декартова произведения и исправленную версию со `AsSplitQuery()`.
- [ ] Предикат передаётся как `Expression<Func<Order, bool>>` в отдельном методе-репозитории.
- [ ] Над каждым `ToListAsync`/`FirstOrDefaultAsync` есть комментарий с ожидаемым SQL.
- [ ] Файл `sql-log.txt` приложен; в `README.md` есть таблица «отчёт → ожидаемый → реальный SQL → совпадение».
- [ ] Запросы остаются `IQueryable<T>` до материализации; нет `AsEnumerable()` до конца цепочки.
- [ ] Все методы асинхронные (`async`/`await`, `*Async` расширения).

#### Подсказки (без прямого ответа)
- Включите лог SQL до первого запроса — иначе не увидите, что улетает в базу.
- Если EF бросает `InvalidOperationException` с текстом про перевод — это не баг EF, это ваше непереводимое выражение. Читайте сообщение: оно обычно указывает на конкретный вызов.
- Для split-запроса достаточно одного вызова `AsSplitQuery()` в конце цепочки, до материализации.
- `Expression<Func<Order, bool>>` отличается от `Func<Order, bool>` одним словом в сигнатуре, но кардинально по сути — проверьте, что передаёте именно дерево.
- Чтобы проверить «один запрос или несколько», смотрите на количество строк `SELECT` в логе: один `ToListAsync` без `Include` коллекций = один SELECT; со split-запросом = несколько SELECT.
- DTO можно сделать `record`-ами — это идиоматично для C# 12 и удобно для проекций.

#### Эталонное решение (разбор)
```csharp
// Program.cs — top-level statements, C# 12 / .NET 8
// Полный работающий пример по мотивам урока M12-L05.
using Microsoft.EntityFrameworkCore;
using System.Linq.Expressions;
using static System.Console;

await using var db = new ShopContext();

// Сидирование / Seeding
await db.Database.EnsureDeletedAsync();
await db.Database.EnsureCreatedAsync();

var customers = new List<Customer>
{
    new() { Name = "Анна" },
    new() { Name = "Борис" },
    new() { Name = "Виктор" },
    new() { Name = "Галина" },
};
db.Customers.AddRange(customers);
await db.SaveChangesAsync();

var rnd = new Random(42);
var statuses = new[] { "New", "Paid", "Cancelled" };
for (int i = 1; i <= 15; i++)
{
    var order = new Order
    {
        Total = rnd.Next(100, 9000),
        Status = statuses[rnd.Next(statuses.Length)],
        CustomerId = customers[rnd.Next(customers.Count)].Id,
        Items = new List<OrderItem>
        {
            new() { Product = $"Товар-{i}-A", Price = rnd.Next(50, 500) },
            new() { Product = $"Товар-{i}-B", Price = rnd.Next(50, 500) },
        },
    };
    db.Orders.Add(order);
}
await db.SaveChangesAsync();

// --- Отчёт №1: Оплаченные крупные заказы ---
// Ожидаемый SQL:
//   SELECT o."Id", o."Total", c."Name"
//   FROM "Orders" AS o
//   LEFT JOIN "Customers" AS c ON o."CustomerId" = c."Id"
//   WHERE o."Status" = 'Paid' AND o."Total" > 1000.0
//   ORDER BY o."Total" DESC
var paidBig = await db.Orders
    .Where(o => o.Status == "Paid" && o.Total > 1000)
    .OrderByDescending(o => o.Total)
    .Select(o => new PaidOrderDto(o.Id, o.Total, o.Customer.Name))
    .ToListAsync();
WriteLine($"Отчёт 1: {paidBig.Count} заказов");

// --- Отчёт №2: Детали одного заказа с клиентом ---
// Ожидаемый SQL: LEFT JOIN Customers
var withCustomer = await db.Orders
    .Include(o => o.Customer)
    .FirstOrDefaultAsync(o => o.Id == 1);
WriteLine($"Отчёт 2: заказ #{withCustomer?.Id}, клиент {withCustomer?.Customer?.Name}");

// --- Отчёт №3: Дерево связей со split-запросом ---
// Несколько Include коллекций → Cartesian risk → AsSplitQuery()
var tree = await db.Orders
    .Where(o => o.Total > 500)
    .Include(o => o.Customer)
    .Include(o => o.Items)
    .AsSplitQuery()
    .ToListAsync();
WriteLine($"Отчёт 3: {tree.Count} заказов с позициями (split-query)");

// --- Отчёт №4: Топ-3 клиентов по выручке ---
// GroupBy + Sum + Count переводятся в SQL
var top = await db.Orders
    .Where(o => o.Status == "Paid")
    .GroupBy(o => o.CustomerId)
    .Select(g => new
    {
        CustomerId = g.Key,
        Sum = g.Sum(o => o.Total),
        Count = g.Count(),
    })
    .OrderByDescending(g => g.Sum)
    .Take(3)
    .ToListAsync();
WriteLine($"Отчёт 4: топ-3 клиентов:");
foreach (var row in top)
    WriteLine($"  customer {row.CustomerId}: sum={row.Sum}, count={row.Count}");

// --- Отчёт №1 через параметр-предикат (Expression!) ---
Expression<Func<Order, bool>> paidAndBig = o => o.Status == "Paid" && o.Total > 1000;
var viaPredicate = await PaidOrdersAsync(db, paidAndBig);
WriteLine($"Отчёт 1 (через предикат): {viaPredicate.Count} заказов");

// --- Анти-пример №5: непереводимый метод в Where ---
try
{
    var bad = db.Orders
        .Where(o => IsVip(o.Total))   // НЕ переводится / NOT translatable
        .ToList();
}
catch (InvalidOperationException ex)
{
    WriteLine($"Анти-пример 5 поймал исключение: {ex.Message.Split('.')[0]}");
}

// --- Анти-пример №6: риск декартова произведения ---
// Плохо: каскад Include коллекций без AsSplitQuery → Cartesian explosion
// var bad = db.Orders.Include(o => o.Items).Include(o => o.ExtraTags).Include(...).ToList();
// Хорошо: AsSplitQuery()
var safe = await db.Orders
    .Include(o => o.Items)
    .AsSplitQuery()
    .ToListAsync();
WriteLine($"Анти-пример 6: безопасный вариант, {safe.Count} строк");

// Вспомогательные / Helpers
static bool IsVip(decimal total) => total > 5000;

static async Task<List<PaidOrderDto>> PaidOrdersAsync(
    ShopContext db, Expression<Func<Order, bool>> predicate)
{
    // Expression<Func<...>> — дерево, переводится в SQL.
    // Если бы был Func<Order, bool> — EF вычислил бы на клиенте.
    return await db.Orders
        .Where(predicate)
        .OrderByDescending(o => o.Total)
        .Select(o => new PaidOrderDto(o.Id, o.Total, o.Customer.Name))
        .ToListAsync();
}

// Records / DTO
public record PaidOrderDto(int Id, decimal Total, string CustomerName);

// Models
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
        opts.UseSqlite("Data Source=shop.db")
            .LogTo(Console.WriteLine, LogLevel.Information);
    }
}
```

Разбор по строкам. Сидирование через `EnsureDeleted` + `EnsureCreated` — простейший способ получить чистую базу для учебного запуска; в проде вы бы использовали миграции. `customers` инициализируется коллекционным выражением `new() {...}` — это C# 12 target-typed `new`. Заказы создаются в цикле со случайными `Status` и `Total`, чтобы в отчётах были и «Paid», и «New», и порог `> 1000` реально фильтровал.

Отчёт №1 — канонический пример урока: `Where` → `OrderByDescending` → `Select` в DTO → `ToListAsync`. `Select(o => new PaidOrderDto(o.Id, o.Total, o.Customer.Name))` неявно подтягивает `Customer` через JOIN, но не плодит декартово произведение, потому что `Customer` — ссылка, а не коллекция: одна строка заказа → одна строка клиента. Здесь применяется концепция проекции из урока: мы тянем только три поля, а не `SELECT *`.

Отчёт №2 — `Include(o => o.Customer)`: явная загрузка связанной сущности, переводится в `LEFT JOIN`. Мы возвращаем полную сущность `Order` — это допустимо, потому что нам нужны и `Customer`, и сам заказ; но в реальном коде лучше тоже проектировать в DTO.

Отчёт №3 — два `Include` (один ссылка, один коллекция) + `AsSplitQuery()`. Здесь коллекция `Items` — источник риска декартова произведения: без split EF выполнит один большой JOIN и размножит строки. `AsSplitQuery()` заставляет EF выполнить отдельный SELECT на `Items` и собрать дерево в памяти. Это прямое применение best practice из урока.

Отчёт №4 — `GroupBy(o => o.CustomerId)` + `g.Sum(...)` + `g.Count()`. В EF Core 8 групповые агрегаты переводятся в SQL (`GROUP BY` + `SUM` + `COUNT`), без клиентского вычисления. `Take(3)` → `LIMIT 3`. Здесь работает правило «агрегат, который переводится, — это нормально».

Метод `PaidOrdersAsync` принимает `Expression<Func<Order, bool>>`: это дерево выражения, которое EF разбирает и переводит в `WHERE`. Если бы параметр был `Func<Order, bool>`, EF принял бы его как скомпилированный делегат и либо бросил исключение, либо (в старых версиях) вытянул бы всю таблицу и фильтровал в памяти. Это ключевое правило урока: предикаты — только `Expression`.

Анти-пример №5 с `IsVip` демонстрирует запрет клиентского вычисления с EF Core 3.0: вызов собственного метода в `Where` не переводится, EF бросает `InvalidOperationException`, а не «дотягивает» данные. `try/catch` перехватывает и печатает начало сообщения — обычно там написано «The LINQ expression could not be translated».

Анти-пример №6 показывает, как каскад `Include` коллекций без `AsSplitQuery()` создаёт декартово произведение, и тут же даёт исправление. В эталоне мы ограничились одним `Include(o => o.Items)`, но закомментированный фрагмент показывает, куда добавились бы ещё коллекции.

Наконец, `ShopContext.OnConfiguring` включает `LogTo` — без него вы не увидите сгенерированный SQL, а «видеть SQL — половина успеха в LINQ to Entities».

#### Задания на углубление (бонус)
1. Перепишите отчёт №4 без `GroupBy`, через `Join` и подзапрос `Select`, и сравните сгенерированный SQL. Какой вариант чище?
2. Добавьте фильтр по дате (`Order.PlacedAt`) и объясните, почему `DateTime.AddDays` переводится в SQL, а ваш метод `IsRecent(DateTime)` — нет.
3. Реализуйте `AsNoTracking()` для всех отчётов и измерьте (через `Stopwatch`), меняется ли объём SQL и время выполнения. Объясните, почему для read-only отчётов трекинг не нужен.
4. Переведите один из отчётов на raw SQL через `FromSqlInterpolated` и сравните читаемость, безопасность и контроль над SQL. Когда raw-запрос оправдан?

---

## Statement in English / Постановка на английском

#### Context & motivation
You join a team that maintains a small online shop on .NET 8 and EF Core 8. The codebase has accumulated several "fat" queries: one developer pulls `SELECT *` to read a single field, another calls a custom `IsVip` method inside `Where` and gets an `InvalidOperationException`, yet another stacks four collection `Include`s and complains that the catalogue is slow because of a Cartesian explosion. The lead asks you to clean up the data-access layer and, at the same time, to prove that every query actually translates to SQL instead of being finished off in process memory.

Your task is to build a console application `ShopReports`, reproduce the model from the lesson (`Order`, `Customer`, `OrderItem`), seed the database with test data, and implement a set of reporting queries, each of which must be translatable, projected into a DTO, and confirmed by a SQL log. You do not write raw SQL and you do not use stored procedures: everything goes through `IQueryable<T>`. Along the way you record, in comments, which SQL you expect, and then compare that expectation with the real log — this is the "see the SQL" training the lesson insists on.

The homework is deliberately constructed so that every section of the lesson surfaces in practice: you meet `Where`+`OrderBy`+`Select`, `Include` and `ThenInclude`, `GroupBy`/`Sum` aggregation, the client-side evaluation trap, and JOIN explosion. At the end you write a short walk-through: which query produced which SQL and why.

#### What to do step by step
1. Create the solution and project:
   ```
   dotnet new sln -n ShopReports
   dotnet new console -n ShopReports.App -o src/ShopReports.App --framework net8.0
   dotnet sln add src/ShopReports.App/ShopReports.App.csproj
   cd src/ShopReports.App
   dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 8.0.*
   dotnet add package Microsoft.EntityFrameworkCore.Design --version 8.0.*
   ```
   Use SQLite as in the lesson: file `shop.db`, provider `UseSqlite`.

2. In `Models.cs` describe `Order`, `Customer`, `OrderItem` exactly in the form from the lesson: navigation properties with initializers (`= null!`, `= new()`), `Status` with values `"New"`, `"Paid"`, `"Cancelled"`.

3. In `ShopContext.cs` declare `DbSet<Order> Orders` and `DbSet<Customer> Customers` via `Set<T>()`, override `OnConfiguring`, and turn on SQL logging: `.LogTo(Console.WriteLine, LogLevel.Information)`. Wire up `UseSqlite("Data Source=shop.db")`.

4. In `Program.cs` (top-level statements) on first run create and fill the database: `db.Database.EnsureDeleted(); db.Database.EnsureCreated();` and add 4–5 customers, 10–15 orders with different statuses and totals, 2–3 items per order. Persist through `SaveChangesAsync()`.

5. Implement report #1 "Paid large orders": return a DTO `PaidOrderDto { int Id; decimal Total; string CustomerName; }` for orders with `Status == "Paid"` and `Total > 1000`, sorted by `Total` descending. Use `Where` → `OrderByDescending` → `Select` → `ToListAsync`.

6. Implement report #2 "Single order with customer": `Include(o => o.Customer)` + `FirstOrDefaultAsync(o => o.Id == <id>)`. Record in a comment the expected SQL (`LEFT JOIN Customers`).

7. Implement report #3 "Relationship tree": for orders with `Total > 500` load `Customer` and the `Items` collection via two `Include`s, call `AsSplitQuery()`, and materialize. Note in a comment why a split query fits here.

8. Implement report #4 "Top customers by revenue": `GroupBy(o => o.CustomerId)` → `Select(g => new { CustomerId = g.Key, Sum = g.Sum(o => o.Total), Count = g.Count() })` → `OrderByDescending` → `Take(3)` → `ToListAsync`.

9. Implement anti-example #5 in a separate method `BadQuery_Untranslatable()`: try to call your own method `bool IsVip(decimal total) => total > 5000;` inside `Where`. Wrap the call in `try/catch (InvalidOperationException)` and print the message — demonstrate that EF Core 3+ throws rather than slipping into client-side evaluation.

10. Implement anti-example #6 `BadQuery_CartesianRisk()`: four collection `Include`s without `AsSplitQuery()`. Do not run it on large data — just show the structure and explain in a comment the Cartesian-explosion risk. Then show the fixed version with `AsSplitQuery()`.

11. Above every `ToListAsync`/`FirstOrDefaultAsync` write a comment with the expected SQL. After running, compare the real console log with the expectation and add a note: matched or not.

12. Introduce a predicate parameter as `Expression<Func<Order, bool>>`: create a method `Task<List<PaidOrderDto>> PaidOrdersAsync(ShopContext db, Expression<Func<Order, bool>> predicate)` and pass `o => o.Status == "Paid" && o.Total > 1000` into it. Explain why `Expression`, not `Func`.

13. Run the application (`dotnet run`), capture the SQL output from the console into a file `sql-log.txt`, and submit it. Make sure each report produced exactly one SQL query (except the split query — there it is several).

14. In the project `README.md` write a table: "Report → expected SQL → actual SQL → match (yes/no) → note".

#### Requirements
- Target platform is .NET 8, language C# 12: top-level statements in `Program.cs`, collection expressions (`new()` for navigation properties, `List<T> items = [..]` when seeding), pattern matching where appropriate, raw string literals `"""..."""` for long SQL comments in code.
- All queries stay `IQueryable<T>` until materialization. It is forbidden to call `AsEnumerable()`, `ToList()`, or `foreach` before the end of the chain.
- All predicates are passed as `Expression<Func<T, bool>>`, not `Func<T, bool>`.
- Report results are projected into DTOs/anonymous types via `Select`; `SELECT *` (returning a full `Order` entity) is allowed only in report #2, where `Include` is needed.
- `await` and EF async extensions (`ToListAsync`, `FirstOrDefaultAsync`, `SaveChangesAsync`) are used.
- SQL logging is enabled through `LogTo` in `OnConfiguring`.
- Anti-examples #5 and #6 are explicitly marked and guarded (`try/catch` for #5; warning comment for #6).
- The code compiles without warnings (clean `dotnet build`), and `dotnet run` executes without unhandled exceptions (#5 is caught).

#### Pitfalls
- **Materialization is the boundary.** Until `ToListAsync`/`FirstOrDefaultAsync`/`Count`/`foreach` no SQL is sent. If you accidentally put `.ToList()` in the middle of the chain, everything after it runs in memory (LINQ to Objects), not in the database. Watch the type: after `AsEnumerable()` the type becomes `IEnumerable<T>` and the provider is out of the game.
- **Calling your own method in `Where` is a trap.** `o => IsVip(o.Total)` does not translate: the provider cannot decompile C#. In EF Core 3+ this is an `InvalidOperationException`, not a silent table pull. Move the logic into a translatable expression (`o => o.Total > 5000`) or into a database computed column.
- **`Func` vs `Expression`.** `Func<Order, bool>` is a compiled delegate that EF cannot unscramble back into a tree. `Expression<Func<Order, bool>>` is an expression tree that EF translates into SQL. Remember: for repositories and specifications — only `Expression`.
- **Collection `Include` is a Cartesian product.** Several collection `Include`s in one query multiply rows: one `Order` row × N `Items` × M `Tags` gives N×M rows. Use `AsSplitQuery()` so EF runs a separate query per collection and stitches the result in memory.
- **Projection shrinks traffic.** `Select(o => new { o.Id, o.Total })` → `SELECT Id, Total`. Without projection and with `Include` you get `SELECT o.*, c.*`, which is heavy over the wire and in memory.
- **Double round-trip with `Count`+`ToList`.** If you call `.Count()` and separately `.ToList()`, two SQL queries fly. Combine via a single projection with an aggregate, or return `Count` and `Items` in one query.
- **Logging is your instrument.** Without `LogTo` you will not see what EF actually generated. Turn it on in dev and tests, turn it off in prod (or move it to `Debug` level).
- **`FirstOrDefaultAsync` vs `FindAsync`.** `Find` uses the tracker and may pull an entity from memory, while `FirstOrDefaultAsync` always goes to the database. For reports — `FirstOrDefaultAsync`.

#### Acceptance criteria
- [ ] The `ShopReports.App` project is created with the commands above, builds with `dotnet build` without warnings or errors.
- [ ] .NET 8, C# 12, EF Core 8 are used (package `Microsoft.EntityFrameworkCore.Sqlite` 8.0.*).
- [ ] The model matches the lesson: `Order`, `Customer`, `OrderItem` with navigation-property initializers.
- [ ] `ShopContext.OnConfiguring` contains `UseSqlite` and `LogTo(Console.WriteLine, LogLevel.Information)`.
- [ ] Database seeding runs on first start (`EnsureCreated` + 4–5 customers, 10–15 orders, items).
- [ ] Report #1 is implemented via `Where`→`OrderByDescending`→`Select`→`ToListAsync` and returns a DTO.
- [ ] Report #2 uses `Include(o => o.Customer)` and `FirstOrDefaultAsync`.
- [ ] Report #3 uses two `Include`s + `AsSplitQuery()`; a comment explains the split-query choice.
- [ ] Report #4 uses `GroupBy` + `Sum` + `Count` + `Take(3)`; the aggregate translates to SQL.
- [ ] Anti-example #5 calls an untranslatable method in `Where` and catches `InvalidOperationException`.
- [ ] Anti-example #6 shows the Cartesian-explosion risk and the fixed version with `AsSplitQuery()`.
- [ ] The predicate is passed as `Expression<Func<Order, bool>>` in a separate repository method.
- [ ] Above every `ToListAsync`/`FirstOrDefaultAsync` there is a comment with the expected SQL.
- [ ] The file `sql-log.txt` is attached; the `README.md` has the "report → expected → actual SQL → match" table.
- [ ] Queries stay `IQueryable<T>` until materialization; no `AsEnumerable()` before the end of the chain.
- [ ] All methods are asynchronous (`async`/`await`, `*Async` extensions).

#### Hints (no direct answer)
- Turn on SQL logging before the first query — otherwise you will not see what flies to the database.
- If EF throws `InvalidOperationException` about translation, it is not an EF bug — it is your untranslatable expression. Read the message: it usually points at the specific call.
- For a split query one `AsSplitQuery()` call at the end of the chain, before materialization, is enough.
- `Expression<Func<Order, bool>>` differs from `Func<Order, bool>` by one word in the signature but radically in essence — check that you pass a tree.
- To check "one query or several", count the `SELECT` lines in the log: one `ToListAsync` without collection `Include`s = one SELECT; with a split query = several SELECTs.
- DTOs can be `record`s — that is idiomatic for C# 12 and convenient for projections.

#### Reference solution walk-through
```csharp
// Program.cs — top-level statements, C# 12 / .NET 8
// Full runnable example based on lesson M12-L05.
using Microsoft.EntityFrameworkCore;
using System.Linq.Expressions;
using static System.Console;

await using var db = new ShopContext();

// Seeding
await db.Database.EnsureDeletedAsync();
await db.Database.EnsureCreatedAsync();

var customers = new List<Customer>
{
    new() { Name = "Anna" },
    new() { Name = "Boris" },
    new() { Name = "Victor" },
    new() { Name = "Galina" },
};
db.Customers.AddRange(customers);
await db.SaveChangesAsync();

var rnd = new Random(42);
var statuses = new[] { "New", "Paid", "Cancelled" };
for (int i = 1; i <= 15; i++)
{
    var order = new Order
    {
        Total = rnd.Next(100, 9000),
        Status = statuses[rnd.Next(statuses.Length)],
        CustomerId = customers[rnd.Next(customers.Count)].Id,
        Items = new List<OrderItem>
        {
            new() { Product = $"Item-{i}-A", Price = rnd.Next(50, 500) },
            new() { Product = $"Item-{i}-B", Price = rnd.Next(50, 500) },
        },
    };
    db.Orders.Add(order);
}
await db.SaveChangesAsync();

// --- Report #1: paid large orders ---
// Expected SQL:
//   SELECT o."Id", o."Total", c."Name"
//   FROM "Orders" AS o
//   LEFT JOIN "Customers" AS c ON o."CustomerId" = c."Id"
//   WHERE o."Status" = 'Paid' AND o."Total" > 1000.0
//   ORDER BY o."Total" DESC
var paidBig = await db.Orders
    .Where(o => o.Status == "Paid" && o.Total > 1000)
    .OrderByDescending(o => o.Total)
    .Select(o => new PaidOrderDto(o.Id, o.Total, o.Customer.Name))
    .ToListAsync();
WriteLine($"Report 1: {paidBig.Count} orders");

// --- Report #2: single order with customer ---
// Expected SQL: LEFT JOIN Customers
var withCustomer = await db.Orders
    .Include(o => o.Customer)
    .FirstOrDefaultAsync(o => o.Id == 1);
WriteLine($"Report 2: order #{withCustomer?.Id}, customer {withCustomer?.Customer?.Name}");

// --- Report #3: relationship tree with split query ---
// Multiple collection Includes → Cartesian risk → AsSplitQuery()
var tree = await db.Orders
    .Where(o => o.Total > 500)
    .Include(o => o.Customer)
    .Include(o => o.Items)
    .AsSplitQuery()
    .ToListAsync();
WriteLine($"Report 3: {tree.Count} orders with items (split-query)");

// --- Report #4: top-3 customers by revenue ---
// GroupBy + Sum + Count translate to SQL
var top = await db.Orders
    .Where(o => o.Status == "Paid")
    .GroupBy(o => o.CustomerId)
    .Select(g => new
    {
        CustomerId = g.Key,
        Sum = g.Sum(o => o.Total),
        Count = g.Count(),
    })
    .OrderByDescending(g => g.Sum)
    .Take(3)
    .ToListAsync();
WriteLine($"Report 4: top-3 customers:");
foreach (var row in top)
    WriteLine($"  customer {row.CustomerId}: sum={row.Sum}, count={row.Count}");

// --- Report #1 via a predicate parameter (Expression!) ---
Expression<Func<Order, bool>> paidAndBig = o => o.Status == "Paid" && o.Total > 1000;
var viaPredicate = await PaidOrdersAsync(db, paidAndBig);
WriteLine($"Report 1 (via predicate): {viaPredicate.Count} orders");

// --- Anti-example #5: untranslatable method in Where ---
try
{
    var bad = db.Orders
        .Where(o => IsVip(o.Total))   // NOT translatable
        .ToList();
}
catch (InvalidOperationException ex)
{
    WriteLine($"Anti-example 5 caught: {ex.Message.Split('.')[0]}");
}

// --- Anti-example #6: Cartesian-explosion risk ---
// Bad: cascade of collection Includes without AsSplitQuery → Cartesian explosion
// var bad = db.Orders.Include(o => o.Items).Include(o => o.ExtraTags).Include(...).ToList();
// Good: AsSplitQuery()
var safe = await db.Orders
    .Include(o => o.Items)
    .AsSplitQuery()
    .ToListAsync();
WriteLine($"Anti-example 6: safe variant, {safe.Count} rows");

// Helpers
static bool IsVip(decimal total) => total > 5000;

static async Task<List<PaidOrderDto>> PaidOrdersAsync(
    ShopContext db, Expression<Func<Order, bool>> predicate)
{
    // Expression<Func<...>> is a tree that translates to SQL.
    // With Func<Order, bool> EF would evaluate on the client.
    return await db.Orders
        .Where(predicate)
        .OrderByDescending(o => o.Total)
        .Select(o => new PaidOrderDto(o.Id, o.Total, o.Customer.Name))
        .ToListAsync();
}

// Records / DTO
public record PaidOrderDto(int Id, decimal Total, string CustomerName);

// Models
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
        opts.UseSqlite("Data Source=shop.db")
            .LogTo(Console.WriteLine, LogLevel.Information);
    }
}
```

Walk-through, line by line. Seeding via `EnsureDeleted` + `EnsureCreated` is the simplest way to get a clean database for a learning run; in production you would use migrations. `customers` is initialized with a C# 12 target-typed `new()` collection expression. Orders are created in a loop with random `Status` and `Total` so that reports see a mix of "Paid", "New", and the `> 1000` threshold actually filters something.

Report #1 is the canonical lesson example: `Where` → `OrderByDescending` → `Select` into a DTO → `ToListAsync`. `Select(o => new PaidOrderDto(o.Id, o.Total, o.Customer.Name))` implicitly pulls `Customer` through a JOIN but does not create a Cartesian product, because `Customer` is a reference, not a collection: one order row → one customer row. This applies the projection concept from the lesson: we fetch only three columns instead of `SELECT *`.

Report #2 is `Include(o => o.Customer)`: explicit loading of a related entity, translated into a `LEFT JOIN`. We return the full `Order` entity — acceptable because both `Customer` and the order itself are needed, though in real code a DTO projection is still better.

Report #3 combines two `Include`s (one reference, one collection) with `AsSplitQuery()`. The `Items` collection is the source of Cartesian risk: without split EF would run one giant JOIN and duplicate rows. `AsSplitQuery()` makes EF run a separate SELECT for `Items` and assemble the tree in memory. This is a direct application of the lesson's best practice.

Report #4 is `GroupBy(o => o.CustomerId)` + `g.Sum(...)` + `g.Count()`. In EF Core 8 grouped aggregates translate to SQL (`GROUP BY` + `SUM` + `COUNT`) without client evaluation. `Take(3)` becomes `LIMIT 3`. This is the "translatable aggregate is fine" rule in action.

The `PaidOrdersAsync` method accepts `Expression<Func<Order, bool>>`: this is an expression tree that EF parses and translates into `WHERE`. If the parameter were `Func<Order, bool>`, EF would treat it as a compiled delegate and either throw or (in older versions) pull the whole table and filter in memory. This is the key rule of the lesson: predicates are `Expression` only.

Anti-example #5 with `IsVip` demonstrates the ban on client-side evaluation since EF Core 3.0: a custom method call in `Where` does not translate, EF throws `InvalidOperationException` instead of "pulling" data. The `try/catch` intercepts and prints the start of the message — it usually reads "The LINQ expression could not be translated".

Anti-example #6 shows how a cascade of collection `Include`s without `AsSplitQuery()` creates a Cartesian product, and immediately gives the fix. In the reference we kept a single `Include(o => o.Items)`, but the commented fragment shows where more collections would go.

Finally, `ShopContext.OnConfiguring` enables `LogTo` — without it you would not see the generated SQL, and "seeing the SQL is half the battle in LINQ to Entities".

#### Going deeper (bonus)
1. Rewrite report #4 without `GroupBy`, using `Join` and a subquery in `Select`, and compare the generated SQL. Which version is cleaner?
2. Add a date filter (`Order.PlacedAt`) and explain why `DateTime.AddDays` translates to SQL while your own `IsRecent(DateTime)` method does not.
3. Apply `AsNoTracking()` to all reports and measure (with `Stopwatch`) whether the SQL volume and execution time change. Explain why tracking is unnecessary for read-only reports.
4. Convert one of the reports to raw SQL via `FromSqlInterpolated` and compare readability, safety, and control over SQL. When is a raw query justified?

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `ShopReports.App` собирается `dotnet build` без предупреждений.
- [ ] Использованы .NET 8, C# 12, EF Core 8, SQLite.
- [ ] Модель и `ShopContext` совпадают с уроком, включён `LogTo`.
- [ ] База сидируется при первом запуске.
- [ ] Реализованы отчёты №1–№4 и анти-примеры №5–№6.
- [ ] Предикат передаётся как `Expression<Func<Order, bool>>`.
- [ ] Над каждой материализацией есть комментарий с ожидаемым SQL.
- [ ] Приложен `sql-log.txt`, в `README.md` — таблица сравнения.
- [ ] Цепочки остаются `IQueryable<T>` до материализации.
- [ ] The `ShopReports.App` project builds with `dotnet build` without warnings.
- [ ] .NET 8, C# 12, EF Core 8, SQLite are used.
- [ ] The model and `ShopContext` match the lesson; `LogTo` is enabled.
- [ ] The database is seeded on first run.
- [ ] Reports #1–#4 and anti-examples #5–#6 are implemented.
- [ ] The predicate is passed as `Expression<Func<Order, bool>>`.
- [ ] Every materialization has a comment with the expected SQL.
- [ ] `sql-log.txt` is attached; `README.md` contains the comparison table.
- [ ] Chains stay `IQueryable<T>` until materialization.

#### Ресурсы / Resources
- [Microsoft Learn — Querying in EF Core — https://learn.microsoft.com/ef/core/querying/](https://learn.microsoft.com/ef/core/querying/)
- [Microsoft Learn — Client vs. Server Evaluation — https://learn.microsoft.com/ef/core/querying/client-eval](https://learn.microsoft.com/ef/core/querying/client-eval)
- [Microsoft Learn — Loading Related Data (Include) — https://learn.microsoft.com/ef/core/querying/related-data](https://learn.microsoft.com/ef/core/querying/related-data)
- [Microsoft Learn — Single vs. Split Queries — https://learn.microsoft.com/ef/core/querying/single-split-queries](https://learn.microsoft.com/ef/core/querying/single-split-queries)
- [Microsoft Learn — Logging and Interception — https://learn.microsoft.com/ef/core/logging-events-diagnostics/](https://learn.microsoft.com/ef/core/logging-events-diagnostics/)
