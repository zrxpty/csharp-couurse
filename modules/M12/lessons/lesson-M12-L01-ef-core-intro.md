[⬆ К модулю M12](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M12-L01: EF Core vs ADO.NET, провайдеры / EF Core vs ADO.NET, providers

**Модуль / Module:** M12
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

В мире доступа к данным в .NET есть два уровня: «ручной» (ADO.NET) и «автоматический» (ORM, в частности EF Core). Представьте дорогу: ADO.NET — это хождение пешком с полной картой в руках, где каждый шаг делаешь сам; EF Core — это такси с навигатором: ты говоришь пункт назначения (LINQ-запрос), а водитель сам строит маршрут и привозит.

**ADO.NET — низкоуровневый фундамент.** Это набор базовых типов (`DbConnection`, `DbCommand`, `DbDataReader`, `DbTransaction`) и поставщиков данных (`SqlConnection`, `SqliteCommand`). Ты сам открываешь соединение, пишешь SQL-строку, подставляешь параметры, выполняешь `ExecuteReader`, перебираешь строки и вручную мапишь поля колонок в свойства классов. Это даёт максимальный контроль, минимальный оверхед и полную независимость от фреймворка. Но цена — много «бойлерплейта»: ради одного объекта можно написать 20 строк кода, а любая опечатка в имени колонки вылезет только в рантайме.

**EF Core — объектно-реляционный маппер (ORM).** Он стоит над ADO.NET и использует тех же провайдеров, но берёт на себя рутину: переводит LINQ-выражения в SQL, строит и кэширует команды, материализует строки в объекты, отслеживает изменения и генерирует `INSERT/UPDATE/DELETE`. У тебя есть `DbContext` — «окно» в базу, `DbSet<T>` — коллекция сущностей, миграции — версионирование схемы. Ты мыслишь объектами и связями (`Order.Customer`, `Customer.Orders`), а не таблицами и внешними ключами, хотя при необходимости всегда можно опуститься до raw SQL через `FromSqlRaw`/`ExecuteSqlRaw`.

**Провайдеры — мост между EF Core и конкретной СУБД.** EF Core не разговаривает с базой напрямую; он делегирует работу провайдеру, который знает диалект SQL. Популярные провайдеры:
- **Microsoft SQL Server** (`Microsoft.EntityFrameworkCore.SqlServer`) — основной для Windows-стека, T-SQL.
- **SQLite** (`Microsoft.EntityFrameworkCore.Sqlite`) — локальная file-based БД, идеальна для тестов, мобильных и десктоп-приложений.
- **PostgreSQL** (`Npgsql.EntityFrameworkCore.PostgreSQL`) — мощная open-source СУБД, богатые типы (JSONB, массивы).
- **InMemory** (`Microsoft.EntityFrameworkCore.InMemory`) — НЕ настоящая БД, хранит данные в памяти процесса; используется только для быстрых unit-тестов логики, не для продакшена.

Смена провайдера обычно сводится к замене пакета и одного метода `UseSqlServer` → `UseSqlite` → `UseNpgsql`, что иллюстрирует абстракцию EF Core.

**Когда что выбирать.** ORM удобен для типовых CRUD-операций, сложных графов сущностей, быстрой разработки и миграций схемы. Raw SQL/ADO.NET уместен там, где нужна максимальная производительность, специфичные фичи СУБД, тяжёлые пакетные операции, или когда SQL уже написан и оптимизирован руками. Часто их комбинируют: EF Core для основной модели и сырые SQL-запросы для отчётов и узких мест. Главное правило: «выбирай инструмент под задачу, а не задачу под инструмент».

#### Theory (EN)

In .NET data access there are two levels: "manual" (ADO.NET) and "automatic" (ORM, specifically EF Core). Imagine a trip: ADO.NET is walking on foot with a full map, making every step yourself; EF Core is taking a taxi with a navigator — you name the destination (a LINQ query), and the driver builds the route and delivers you.

**ADO.NET — the low-level foundation.** It is a set of base types (`DbConnection`, `DbCommand`, `DbDataReader`, `DbTransaction`) and data providers (`SqlConnection`, `SqliteCommand`). You open the connection yourself, write an SQL string, add parameters, call `ExecuteReader`, iterate rows, and manually map column fields to class properties. This gives maximum control, minimal overhead, and full framework independence. The price is boilerplate: a single object can cost twenty lines of code, and any typo in a column name surfaces only at runtime.

**EF Core — an Object-Relational Mapper (ORM).** It sits on top of ADO.NET and reuses the same providers, but takes over the routine: it translates LINQ expressions into SQL, builds and caches commands, materializes rows into objects, tracks changes, and generates `INSERT/UPDATE/DELETE`. You have a `DbContext` — a "window" into the database, `DbSet<T>` — a collection of entities, and migrations — schema versioning. You think in objects and relationships (`Order.Customer`, `Customer.Orders`), not in tables and foreign keys, though you can always drop down to raw SQL via `FromSqlRaw`/`ExecuteSqlRaw` when needed.

**Providers — the bridge between EF Core and a concrete DBMS.** EF Core does not talk to the database directly; it delegates to a provider that knows the SQL dialect. Popular providers:
- **Microsoft SQL Server** (`Microsoft.EntityFrameworkCore.SqlServer`) — the primary choice for the Windows stack, T-SQL.
- **SQLite** (`Microsoft.EntityFrameworkCore.Sqlite`) — a local file-based DB, ideal for tests, mobile and desktop apps.
- **PostgreSQL** (`Npgsql.EntityFrameworkCore.PostgreSQL`) — a powerful open-source DBMS with rich types (JSONB, arrays).
- **InMemory** (`Microsoft.EntityFrameworkCore.InMemory`) — NOT a real database; it stores data in process memory, used only for fast unit tests of logic, never for production.

Switching providers usually comes down to swapping the package and one method `UseSqlServer` → `UseSqlite` → `UseNpgsql`, which illustrates the EF Core abstraction.

**When to choose what.** ORM is convenient for typical CRUD operations, complex entity graphs, rapid development, and schema migrations. Raw SQL/ADO.NET fits where you need maximum performance, DBMS-specific features, heavy bulk operations, or when SQL is already hand-written and optimized. They are often combined: EF Core for the main model and raw SQL for reports and hot spots. The golden rule: "pick the tool for the task, not the task for the tool".

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8+ — EF Core vs ADO.NET side by side
// Сравнение EF Core и ADO.NET на одном примере / Side-by-side comparison

using Microsoft.Data.SqlClient;          // ADO.NET provider for SQL Server / Провайдер ADO.NET
using Microsoft.EntityFrameworkCore;
using System.Data;

#region EF Core — высокая абстракция / High-level abstraction

// Модель сущности / Entity model
public class Product
{
    public int Id { get; set; }
    public required string Name { get; set; }   // C# 12 'required' / обязательное поле
    public decimal Price { get; set; }
    public bool IsActive { get; set; }
}

// Контекст базы данных / Database context
public class ShopDbContext(DbContextOptions<ShopDbContext> options) : DbContext(options)
{
    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.Entity<Product>().Property(p => p.Price).HasPrecision(18, 2); // точность decimal / precision
    }
}

public static class EfCoreDemo
{
    // LINQ → SQL генерируется автоматически / LINQ to SQL is generated automatically
    public static async Task<List<Product>> GetActiveAboveAsync(ShopDbContext db, decimal minPrice)
    {
        return await db.Products
            .Where(p => p.IsActive && p.Price >= minPrice)
            .OrderBy(p => p.Price)
            .AsNoTracking()                 // только чтение — без трекинга / read-only, no tracking
            .ToListAsync();
    }

    public static async Task InsertAsync(ShopDbContext db, Product product)
    {
        db.Products.Add(product);           // EF сам построит INSERT / EF builds INSERT itself
        await db.SaveChangesAsync();
    }
}

#endregion

#region ADO.NET — ручное управление / Manual control

public static class AdoNetDemo
{
    // Тот же запрос, написанный руками / The same query, written by hand
    public static async Task<List<Product>> GetActiveAboveAsync(
        SqlConnection connection, decimal minPrice)
    {
        const string sql = """
            SELECT Id, Name, Price, IsActive
            FROM Products
            WHERE IsActive = 1 AND Price >= @minPrice
            ORDER BY Price
            """;                            // C# 12 raw string literal / буквальная строка

        await using var cmd = new SqlCommand(sql, connection);
        // Именованный параметр — защита от SQL-инъекций / Named parameter — SQL injection protection
        cmd.Parameters.Add("@minPrice", SqlDbType.Decimal).Value = minPrice;

        if (connection.State != ConnectionState.Open)
            await connection.OpenAsync();

        var list = new List<Product>();
        await using var reader = await cmd.ExecuteReaderAsync();
        while (await reader.ReadAsync())    // ручной маппинг строк → объекты / manual row→object mapping
        {
            list.Add(new Product
            {
                Id = reader.GetInt32(0),
                Name = reader.GetString(1),
                Price = reader.GetDecimal(2),
                IsActive = reader.GetBoolean(3)
            });
        }
        return list;
    }
}

#endregion

#region Регистрация провайдеров / Provider registration

// В Program.cs (.NET 8 minimal hosting) / In Program.cs
//
// SQL Server:
// builder.Services.AddDbContext<ShopDbContext>(o =>
//     o.UseSqlServer(builder.Configuration.GetConnectionString("Shop")));
//
// SQLite:
// builder.Services.AddDbContext<ShopDbContext>(o =>
//     o.UseSqlite(builder.Configuration.GetConnectionString("Shop")));
//
// PostgreSQL:
// builder.Services.AddDbContext<ShopDbContext>(o =>
//     o.UseNpgsql(builder.Configuration.GetConnectionString("Shop")));
//
// InMemory (только тесты / tests only):
// builder.Services.AddDbContext<ShopDbContext>(o => o.UseInMemoryDatabase("shop-test"));

public static class ProviderSetup
{
    // Универсальный селектор провайдера по строке конфига / Universal provider selector
    public static DbContextOptionsBuilder ChooseProvider(
        this DbContextOptionsBuilder b, string provider, string connectionString) => provider switch
    {
        "SqlServer" => b.UseSqlServer(connectionString),
        "Sqlite"    => b.UseSqlite(connectionString),
        "Npgsql"    => b.UseNpgsql(connectionString),
        "InMemory"  => b.UseInMemoryDatabase(connectionString),  // игнорирует строку, берёт имя / ignores conn string
        _           => throw new ArgumentException($"Unknown provider: {provider}")
    };
}

#endregion
```

#### Best Practices

- Выбирай EF Core для типового CRUD и доменной модели, а raw SQL/ADO.NET — для узких мест и отчётов; не бойся их комбинировать.
- Всегда используй параметры (`@minPrice`, `SqlParameter`) в ADO.NET и `FromSqlRaw` с параметрами в EF Core — это защита от SQL-инъекций.
- Закрывай соединения и ридеры через `await using` — это гарантирует освобождение ресурсов даже при исключении.
- Используй `AsNoTracking()` для запросов только на чтение — это экономит память и ускоряет выполнение.
- Храни строки подключения в `appsettings.json` или секретах среды, а не в исходном коде.
- Use EF Core for typical CRUD and the domain model, raw SQL/ADO.NET for hot spots and reports; do not be afraid to mix them.
- Always use parameters (`@minPrice`, `SqlParameter`) in ADO.NET and `FromSqlRaw` with parameters in EF Core — this is your SQL injection defense.
- Close connections and readers with `await using` — it guarantees resource release even on exceptions.
- Use `AsNoTracking()` for read-only queries — it saves memory and speeds up execution.
- Keep connection strings in `appsettings.json` or environment secrets, not in source code.

#### Частые ошибки / Common Mistakes

- Конкатенация строк в SQL (`"... WHERE Name = '" + name + "'"`) → используй параметры `@name` для защиты от инъекций (RU).
- Использование `UseInMemoryDatabase` в продакшене → это не настоящая БД, нет транзакций и FK; бери SQLite или Testcontainers (RU).
- Забыли `AsNoTracking()` на тяжёлых read-only запросах → контекст трекит тысячи сущностей и раздувает память (RU).
- Открыли `SqlConnection` без `using`/`await using` → утечка соединений и исчерпание пула (RU).
- Обращение к `DbSet` после `Dispose` контекста → используй `ToList`/`ToListAsync` до освобождения контекста (RU).
- String concatenation in SQL (`"... WHERE Name = '" + name + "'"`) → use `@name` parameters to prevent injection (EN).
- Using `UseInMemoryDatabase` in production → it is not a real DB, no transactions or FKs; use SQLite or Testcontainers (EN).
- Forgetting `AsNoTracking()` on heavy read-only queries → the context tracks thousands of entities and bloats memory (EN).
- Opening a `SqlConnection` without `using`/`await using` → connection leaks and pool exhaustion (EN).
- Accessing a `DbSet` after the context is disposed → call `ToList`/`ToListAsync` before releasing the context (EN).

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я могу объяснить разницу между ADO.NET и EF Core одним абзацем (RU)
- [ ] Я знаю, что провайдер — это мост между EF Core и конкретной СУБД (RU)
- [ ] Я могу перечислить 4 провайдера и их назначение (SQL Server/SQLite/PostgreSQL/InMemory) (RU)
- [ ] Я понимаю, когда выбрать ORM, а когда raw SQL (RU)
- [ ] Я использую параметры в SQL-запросах всегда (RU)
- [ ] Я знаю, что InMemory-провайдер не для продакшена (RU)
- [ ] I can explain the difference between ADO.NET and EF Core in one paragraph (EN)
- [ ] I know a provider is the bridge between EF Core and a concrete DBMS (EN)
- [ ] I can list the 4 providers and their purpose (SQL Server/SQLite/PostgreSQL/InMemory) (EN)
- [ ] I understand when to pick ORM vs raw SQL (EN)
- [ ] I always use parameters in SQL queries (EN)
- [ ] I know the InMemory provider is not for production (EN)

#### Ресурсы / Resources

- [Microsoft Learn — EF Core](https://learn.microsoft.com/ef/core/)
- [Microsoft Learn — ADO.NET overview](https://learn.microsoft.com/dotnet/framework/data/adonet/)
- [EF Core providers list](https://learn.microsoft.com/ef/core/providers/)
- [Npgsql EF Core provider](https://www.npgsql.org/efcore/)

---

[⬆ К модулю M12](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
