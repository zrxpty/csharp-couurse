---
[← К уроку M12-L01](lesson-M12-L01-ef-core-intro.md) | [⬆ К модулю M12](../README.md) | [Следующее ДЗ →](homework-M12-L02-code-first-dbcontext.md)
---

### Домашнее задание M12-L01: EF Core vs ADO.NET, провайдеры / Homework M12-L01: EF Core vs ADO.NET, providers

**Урок / Lesson:** M12-L01
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) На одном и том же примере (складской каталог товаров) реализовать доступ к данным двумя способами — через EF Core (LINQ, `DbSet`, `AsNoTracking`) и через ADO.NET (`DbConnection`/`DbCommand`/`DbDataReader` с параметризованным SQL), научиться переключать провайдеры (SQL Server / SQLite / Npgsql / InMemory) одной строкой конфигурации и закрепить best practices: параметры против инъекций, `await using` для ресурсов, `AsNoTracking` для чтения, хранение строк подключения вне кода. (EN) On the same example (a warehouse product catalogue) implement data access two ways — through EF Core (LINQ, `DbSet`, `AsNoTracking`) and through ADO.NET (`DbConnection`/`DbCommand`/`DbDataReader` with parameterized SQL), learn to switch providers (SQL Server / SQLite / Npgsql / InMemory) with a single configuration line, and reinforce best practices: parameters vs injection, `await using` for resources, `AsNoTracking` for reads, keeping connection strings out of code.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит два уровня доступа к данным в .NET — низкоуровневый ADO.NET и ORM EF Core — и объясняет роль провайдеров как моста к конкретной СУБД. Это ДЗ заставляет прочувствовать разницу руками: вы напишете эквивалентный запрос в обеих парадигмах и увидите бойлерплейт ADO.NET и лаконичность EF Core, а также убедитесь, что смена провайдера сводится к замене одного пакета и одного метода `Use*`.
(EN) The lesson introduces two levels of .NET data access — the low-level ADO.NET and the EF Core ORM — and explains providers as the bridge to a concrete DBMS. This homework makes you feel the difference first-hand: you will write the equivalent query in both paradigms and see ADO.NET's boilerplate versus EF Core's brevity, and you will confirm that switching a provider comes down to swapping one package and one `Use*` method.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы приходите junior-разработчиком в небольшую компанию «СкладОК», где есть каталог товаров. Исторически приложение работало напрямую через ADO.NET: каждый запрос к базе писался вручную, параметры подставлялись строкой (иногда с риском инъекций), а маппинг строк в объекты делался в `while (reader.ReadAsync())`. Команда хочет постепенно перейти на EF Core, чтобы сократить бойлерплейт, но при этом сохранить возможность опускаться до raw SQL для тяжёлых отчётов. Ваш ментор даёт вам первое задание: взять простую сущность `Product` и реализовать для неё репозиторий двумя способами — на EF Core и на ADO.NET — через общий интерфейс, чтобы можно было сравнить их в действии. Дополнительно нужно сделать так, чтобы один и тот же код работал с разными СУБД: основная разработка идёт на SQLite (файловая БД, не требует сервера), на CI гоняется на InMemory (быстрые тесты), а на продакшене планируется SQL Server или PostgreSQL.

Цель задания — не просто «написать два репозитория», а на практике закрепить концепции урока: понять, что EF Core стоит над ADO.NET и использует тех же провайдеров; что провайдер — это мост между ORM и диалектом SQL; что LINQ-запрос транслируется в SQL автоматически, а в ADO.NET вы пишете SQL сами; что `AsNoTracking()` экономит память на чтении; что параметры защищают от инъекций; что `await using` гарантирует освобождение соединений и ридеров даже при исключениях; что `UseInMemoryDatabase` не подходит для продакшена, потому что это не настоящая БД без транзакций и внешних ключей. Вы должны научиться выбирать инструмент под задачу, а не задачу под инструмент.

#### Что нужно сделать (пошагово)
1. Создайте консольный проект .NET 8 с именем `WarehouseCatalog`:
   `dotnet new console -n WarehouseCatalog -o WarehouseCatalog -f net8.0`
   Перейдите в папку: `cd WarehouseCatalog`.
2. Установите пакеты провайдеров (все четыре, чтобы переключать в конфиге):
   `dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 8.*`
   `dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 8.*`
   `dotnet add package Microsoft.EntityFrameworkCore.InMemory --version 8.*`
   `dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL --version 8.*`
   Установите ADO.NET-провайдеры: `dotnet add package Microsoft.Data.SqlClient` и `dotnet add package Microsoft.Data.Sqlite`.
3. Создайте класс сущности `Product` со свойствами: `int Id`, `required string Name` (C# 12 `required`), `decimal Price`, `bool IsActive`, `DateTime CreatedAt`. Используйте `required` для обязательного строкового поля.
4. Создайте `ShopDbContext(DbContextOptions<ShopDbContext> options) : DbContext(options)` с `DbSet<Product> Products => Set<Product>();`. В `OnModelCreating` настройте: `Name` — обязательно, `HasMaxLength(200)`; `Price` — `HasPrecision(18, 2)`; `IsActive` — `HasDefaultValue(true)`. Добавьте начальные данные через `HasData` (3–4 товара).
5. Определите общий интерфейс `IProductRepository` с методами: `GetActiveAboveAsync(decimal minPrice, CancellationToken)`, `GetByIdAsync(int id, CancellationToken)`, `InsertAsync(Product, CancellationToken)`, `UpdatePriceAsync(int id, decimal newPrice, CancellationToken)`.
6. Реализуйте `EfCoreProductRepository(ShopDbContext db)`: для чтения используйте `AsNoTracking()`, для вставки — `Add` + `SaveChangesAsync`, для обновления цены — загрузите сущность (с трекингом), измените `Price`, вызовите `SaveChangesAsync`.
7. Реализуйте `AdoNetProductRepository` так, чтобы он принимал фабрику соединений `Func<DbConnection>` (абстракция ADO.NET, независимая от СУБД). Все SQL-строки оформляйте через raw string literal C# 12 (`""" ... """`). Все значения подставляйте только через именованные параметры (`@id`, `@minPrice`, `@newPrice`). Ридер и соединение оборачивайте в `await using`. Маппинг строк в `Product` делайте вручную через `reader.GetInt32(0)` и т.п.
8. Напишите метод-расширение `ChooseProvider(this DbContextOptionsBuilder b, string provider, string connStr)` через `switch`-выражение (pattern matching), поддерживающий `"SqlServer"`, `"Sqlite"`, `"Npgsql"`, `"InMemory"`, и выбрасывающий `ArgumentException` для неизвестного.
9. В `appsettings.json` добавьте секцию с `Provider` и `ConnectionStrings:Shop`. В `Program.cs` (top-level statements) прочитайте конфигурацию, зарегистрируйте `ShopDbContext` через `AddDbContext` с выбранным провайдером, зарегистрируйте оба репозитория в DI.
10. В `Program.cs` вызовите `EnsureCreatedAsync` (для демо), вставьте новый товар через EF-репозиторий, затем прочитайте список активных товаров с ценой выше порога через ОБА репозитория и выведите результаты рядом, чтобы убедиться, что они идентичны.
11. Для SQLite используйте строку подключения `"Data Source=shop.db"`. Запустите `dotnet run` и убедитесь, что файл `shop.db` создался, а вывод содержит товары из обоих репозиториев.
12. Переключите провайдер в `appsettings.json` на `"InMemory"` с именем `"shop-test"` и снова запустите — код должен работать без изменений. В комментариях отметьте, почему InMemory нельзя использовать в продакшене.

#### Требования к решению
Решение должно компилироваться под .NET 8 без предупреждений и запускаться как с SQLite, так и с InMemory заменой одной строки в `appsettings.json`. Используйте возможности C# 12: `required`-свойства, primary constructor для контекста и репозиториев, raw string literals для SQL, collection expressions для инициализации списков (`List<Product> list = [..];`), pattern matching (`switch` expression) в селекторе провайдера. Все асинхронные методы должны принимать `CancellationToken` и передавать его в нижележащие вызовы. Все SQL-запросы в ADO.NET-репозитории обязаны использовать параметры — конкатенация строк недопустима. Все `IDisposable`/`IAsyncDisposable` ресурсы (`DbConnection`, `DbCommand`, `DbDataReader`) оборачивайте в `await using`. В EF-репозитории запросы только на чтение помечайте `AsNoTracking()`. Строки подключения храните в `appsettings.json`, а не в коде. Код должен быть оформлен по best practices из урока: параметры против инъекций, `await using` для освобождения ресурсов, `AsNoTracking` для чтения, конфигурация вне кода.

#### Тонкости и подводные камни
- **SQL-инъекции через конкатенацию.** Самая частая и опасная ошибка в ADO.NET — писать `"... WHERE Name = '" + name + "'"`. Урок прямо предостерегает от этого. Решение — всегда параметры `@name`. В EF Core `FromSqlRaw` тоже требует параметров через `SqlParameter` или интерполяцию `FromSqlInterpolated`, иначе инъекция возможна даже в ORM.
- **InMemory как «псевдо-БД».** `UseInMemoryDatabase` не является настоящей СУБД: нет транзакций, нет внешних ключей, нет реляционной целостности, данные живут в памяти процесса. Урок подчёркивает: только для быстрых unit-тестов логики, никогда для продакшена. Для тестов, близких к продакшену, берите SQLite (in-memory режим `"DataSource=:memory:"`) или Testcontainers.
- **Забытый `AsNoTracking()`.** На тяжёлых read-only запросах без `AsNoTracking()` контекст трекит каждую сущность в `ChangeTracker`, что раздувает память и замедляет работу. Для списков и отчётов всегда ставьте `AsNoTracking()`.
- **Утечка соединений.** `SqlConnection` без `using`/`await using` не возвращается в пул, и при нагрузке пул исчерпывается. Аналогично — `DbDataReader`. Правило: `await using var conn = factory();` и `await using var reader = await cmd.ExecuteReaderAsync(ct);`.
- **Доступ к `DbSet` после `Dispose`.** Нельзяmaterialize lazily после освобождения контекста. Всегда вызывайте `ToListAsync(ct)` до того, как контекст выйдет из области видимости.
- **Точность `decimal`.** Без `HasPrecision(18, 2)` EF Core для SQL Server создаст `decimal(18,2)` по умолчанию, но для некоторых провайдеров (например Npgsql) поведение по умолчанию может отличаться, поэтому явная конфигурация — best practice.
- **`required` и EF Core.** Свойство `required string Name` работает с EF Core 8: при материализации EF Core установит значение. Но для value-objects с `init` будьте внимательны — конструкторы сущностей должны быть совместимы.
- **`EnsureCreated` vs Migrations.** Для демо `EnsureCreatedAsync` приемлем, но в реальном проекте используют миграции (`dotnet ef migrations add`). Это упоминается в уроке как «версионирование схемы».
- **Отмена операций.** Передавайте `CancellationToken` до провайдера: `ExecuteReaderAsync(ct)`, `ToListAsync(ct)`, `SaveChangesAsync(ct)`. Иначе запрос нельзя отменить.
- **Сырой SQL в ADO.NET и диалект.** Булевы значения: SQL Server — `IsActive = 1`, SQLite — `IsActive = 1`, PostgreSQL — `IsActive = TRUE`. Для кросс-провайдерного кода либо используйте EF Core (он сам транслирует), либо параметризуйте и тестируйте на целевой СУБД.

#### Критерии приёмки
- [ ] Проект `WarehouseCatalog` собирается под .NET 8 без ошибок и предупреждений.
- [ ] Сущность `Product` использует `required string Name` (C# 12).
- [ ] `ShopDbContext` использует primary constructor и настраивает `Price` через `HasPrecision(18, 2)`.
- [ ] `DbSet<Product> Products => Set<Product>();` определён через `Set<T>()`.
- [ ] В `OnModelCreating` есть `HasMaxLength`, `HasDefaultValue`, `HasData` (3–4 записи).
- [ ] Общий интерфейс `IProductRepository` реализован двумя классами.
- [ ] EF-репозиторий использует `AsNoTracking()` на всех read-запросах.
- [ ] ADO.NET-репозиторий принимает `Func<DbConnection>` (не конкретный `SqlConnection`).
- [ ] Все SQL в ADO.NET оформлены raw string literal `"""..."""`.
- [ ] Все параметры в ADO.NET передаются через `@name` — конкатенация строк отсутствует.
- [ ] `DbConnection`, `DbCommand`, `DbDataReader` обёрнуты в `await using`.
- [ ] Селектор провайдера — `switch` expression с четырьмя кейсами и `ArgumentException` по умолчанию.
- [ ] `appsettings.json` содержит `Provider` и `ConnectionStrings:Shop`.
- [ ] Приложение запускается с SQLite (`Data Source=shop.db`) и создаёт файл.
- [ ] При переключении `Provider` на `"InMemory"` код работает без изменений.
- [ ] Вывод показывает идентичные результаты из обоих репозиториев.
- [ ] В комментариях или README отмечено, почему InMemory не для продакшена.

#### Подсказки (без прямого ответа)
- Чтобы абстрагироваться от СУБД в ADO.NET, используйте базовый класс `System.Data.Common.DbConnection` и `DbCommand`/`DbParameter`, а не конкретные `SqlConnection`/`SqliteConnection`. Фабрика `Func<DbConnection>` скрывает конкретный тип.
- Для создания `DbParameter` без конкретного типа вызовите `cmd.CreateParameter()` и задайте `ParameterName` и `Value`. Это работает для любого провайдера.
- `AsNoTracking()` ставится в цепочке LINQ перед терминальным `ToListAsync`.
- В селекторе провайдера InMemory-кейс игнорирует строку подключения и берёт имя базы: `b.UseInMemoryDatabase(connectionString)`.
- Для создания файла `shop.db` достаточно `EnsureCreatedAsync` — схема создаётся по модели.
- Чтобы передать `CancellationToken` в top-level `Program.cs`, используйте `CancellationToken.None` или `await Task.Delay(..., ct)` для демо.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — Эталонное решение ДЗ M12-L01
// WarehouseCatalog — EF Core vs ADO.NET, провайдеры
// RU: демонстрирует оба подхода на одной сущности Product
// EN: demonstrates both approaches on a single Product entity

using Microsoft.Data.Sqlite;       // ADO.NET-провайдер SQLite / SQLite ADO.NET provider
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using System.Data;
using System.Data.Common;

// ===== Сущность / Entity =====
public class Product
{
    public int Id { get; set; }
    public required string Name { get; set; }      // C# 12 required / обязательное
    public decimal Price { get; set; }
    public bool IsActive { get; set; }
    public DateTime CreatedAt { get; set; }
}

// ===== Контекст / Context (primary constructor) =====
public class ShopDbContext(DbContextOptions<ShopDbContext> options) : DbContext(options)
{
    public DbSet<Product> Products => Set<Product>();   // Set<T>() — идиома EF Core 8

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.Entity<Product>(e =>
        {
            e.HasKey(p => p.Id);
            e.Property(p => p.Name).IsRequired().HasMaxLength(200);
            e.Property(p => p.Price).HasPrecision(18, 2);   // точность decimal / precision
            e.Property(p => p.IsActive).HasDefaultValue(true);
            e.Property(p => p.CreatedAt).HasDefaultValueSql("CURRENT_TIMESTAMP");
            e.HasData(
                new Product { Id = 1, Name = "Ноутбук Pro",  Price = 1200m, IsActive = true,  CreatedAt = new DateTime(2024,1,1) },
                new Product { Id = 2, Name = "Мышь USB",     Price = 25m,   IsActive = true,  CreatedAt = new DateTime(2024,1,2) },
                new Product { Id = 3, Name = "Клавиатура",   Price = 75m,   IsActive = false, CreatedAt = new DateTime(2024,1,3) }
            );
        });
    }
}

// ===== Общий интерфейс / Common interface =====
public interface IProductRepository
{
    Task<List<Product>> GetActiveAboveAsync(decimal minPrice, CancellationToken ct = default);
    Task<Product?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<int> InsertAsync(Product product, CancellationToken ct = default);
    Task<int> UpdatePriceAsync(int id, decimal newPrice, CancellationToken ct = default);
}

// ===== EF Core реализация / EF Core implementation =====
public class EfCoreProductRepository(ShopDbContext db) : IProductRepository
{
    public Task<List<Product>> GetActiveAboveAsync(decimal minPrice, CancellationToken ct = default) =>
        db.Products
          .AsNoTracking()                              // чтение без трекинга / read, no tracking
          .Where(p => p.IsActive && p.Price >= minPrice)
          .OrderBy(p => p.Price)
          .ToListAsync(ct);

    public Task<Product?> GetByIdAsync(int id, CancellationToken ct = default) =>
        db.Products.AsNoTracking().FirstOrDefaultAsync(p => p.Id == id, ct);

    public async Task<int> InsertAsync(Product product, CancellationToken ct = default)
    {
        db.Products.Add(product);                      // EF построит INSERT сам / EF builds INSERT
        await db.SaveChangesAsync(ct);
        return product.Id;                             // Id присваивается после SaveChanges
    }

    public async Task<int> UpdatePriceAsync(int id, decimal newPrice, CancellationToken ct = default)
    {
        var product = await db.Products.FirstAsync(p => p.Id == id, ct);  // с трекингом / tracked
        product.Price = newPrice;
        return await db.SaveChangesAsync(ct);          // генерирует UPDATE / generates UPDATE
    }
}

// ===== ADO.NET реализация / ADO.NET implementation =====
public class AdoNetProductRepository(Func<DbConnection> connFactory) : IProductRepository
{
    public async Task<List<Product>> GetActiveAboveAsync(decimal minPrice, CancellationToken ct = default)
    {
        const string sql = """
            SELECT Id, Name, Price, IsActive, CreatedAt
            FROM Products
            WHERE IsActive = 1 AND Price >= @minPrice
            ORDER BY Price
            """;                                       // raw string literal C# 12

        await using var conn = connFactory();          // await using — освобождение / release
        if (conn.State != ConnectionState.Open) await conn.OpenAsync(ct);
        await using var cmd = conn.CreateCommand();
        cmd.CommandText = sql;
        var p = cmd.CreateParameter();                 // кросс-провайдерный параметр
        p.ParameterName = "@minPrice";
        p.Value = minPrice;
        cmd.Parameters.Add(p);

        List<Product> list = [];                       // collection expression C# 12
        await using var reader = await cmd.ExecuteReaderAsync(ct);
        while (await reader.ReadAsync(ct))             // ручной маппинг / manual mapping
        {
            list.Add(new Product
            {
                Id        = reader.GetInt32(0),
                Name      = reader.GetString(1),
                Price     = reader.GetDecimal(2),
                IsActive  = reader.GetBoolean(3),
                CreatedAt = reader.GetDateTime(4)
            });
        }
        return list;
    }

    public async Task<Product?> GetByIdAsync(int id, CancellationToken ct = default)
    {
        const string sql = """
            SELECT Id, Name, Price, IsActive, CreatedAt
            FROM Products
            WHERE Id = @id
            """;
        await using var conn = connFactory();
        if (conn.State != ConnectionState.Open) await conn.OpenAsync(ct);
        await using var cmd = conn.CreateCommand();
        cmd.CommandText = sql;
        var p = cmd.CreateParameter(); p.ParameterName = "@id"; p.Value = id; cmd.Parameters.Add(p);

        await using var reader = await cmd.ExecuteReaderAsync(ct);
        if (!await reader.ReadAsync(ct)) return null;
        return new Product
        {
            Id = reader.GetInt32(0), Name = reader.GetString(1), Price = reader.GetDecimal(2),
            IsActive = reader.GetBoolean(3), CreatedAt = reader.GetDateTime(4)
        };
    }

    public async Task<int> InsertAsync(Product product, CancellationToken ct = default)
    {
        const string sql = """
            INSERT INTO Products (Name, Price, IsActive, CreatedAt)
            VALUES (@name, @price, @active, @created);
            SELECT last_insert_rowid();
            """;                                       // SQLite; для SQL Server — SCOPE_IDENTITY()
        await using var conn = connFactory();
        if (conn.State != ConnectionState.Open) await conn.OpenAsync(ct);
        await using var cmd = conn.CreateCommand();
        cmd.CommandText = sql;
        AddParam(cmd, "@name", product.Name);
        AddParam(cmd, "@price", product.Price);
        AddParam(cmd, "@active", product.IsActive);
        AddParam(cmd, "@created", product.CreatedAt);
        var rawId = await cmd.ExecuteScalarAsync(ct);
        return Convert.ToInt32(rawId);
    }

    public async Task<int> UpdatePriceAsync(int id, decimal newPrice, CancellationToken ct = default)
    {
        const string sql = """
            UPDATE Products SET Price = @newPrice WHERE Id = @id
            """;
        await using var conn = connFactory();
        if (conn.State != ConnectionState.Open) await conn.OpenAsync(ct);
        await using var cmd = conn.CreateCommand();
        cmd.CommandText = sql;
        AddParam(cmd, "@newPrice", newPrice);
        AddParam(cmd, "@id", id);
        return await cmd.ExecuteNonQueryAsync(ct);     // число строк / rows affected
    }

    private static void AddParam(DbCommand cmd, string name, object value)
    {
        var p = cmd.CreateParameter();
        p.ParameterName = name;
        p.Value = value;
        cmd.Parameters.Add(p);
    }
}

// ===== Селектор провайдера / Provider selector (switch expression) =====
public static class ProviderSelector
{
    public static DbContextOptionsBuilder ChooseProvider(
        this DbContextOptionsBuilder b, string provider, string connStr) => provider switch
    {
        "SqlServer" => b.UseSqlServer(connStr),
        "Sqlite"    => b.UseSqlite(connStr),
        "Npgsql"    => b.UseNpgsql(connStr),
        "InMemory"  => b.UseInMemoryDatabase(connStr),  // игнорирует строку, берёт имя / takes name
        _           => throw new ArgumentException($"Unknown provider: {provider}")
    };
}
```

**Разбор по строкам.** Сущность `Product` использует `required string Name` — это C# 12, который дружит с EF Core 8: при материализации EF Core корректно заполняет обязательное свойство. Контекст `ShopDbContext` оформлен через primary constructor `(DbContextOptions<ShopDbContext> options)` — снова C# 12, бойлерплейт сокращён. Свойство `DbSet<Product> Products => Set<Product>();` — это идиома EF Core 8 (вместо автосвойства с `set`), рекомендованная Microsoft: `Set<T>()` возвращает актуальный набор с трекингом. В `OnModelCreating` мы явно задаём `HasPrecision(18, 2)` для `Price` (урок упоминает точность decimal), `HasMaxLength(200)` и `HasDefaultValue(true)` для `IsActive`, а также `HasData` для начальных данных — это концепция «версионирования схемы» из урока.

Интерфейс `IProductRepository` намеренно общий: одна и та же семантика для обеих реализаций, чтобы можно было подменять их в DI и сравнивать. EF-репозиторий через `AsNoTracking()` на чтении — прямое применение best practice из урока («экономит память и ускоряет выполнение»); на вставке — `Add` + `SaveChangesAsync`, EF сам строит `INSERT`; на обновлении — загрузка с трекингом, изменение свойства, `SaveChangesAsync`, который генерирует `UPDATE` только изменившегося поля.

ADO.NET-репозиторий принимает `Func<DbConnection>` — это ключевой момент: мы программируем против базового `System.Data.Common.DbConnection`, а не конкретного `SqlConnection`, что даёт кросс-провайдерность (та же идея, что и у EF Core над ADO.NET). Все SQL — raw string literal `"""..."""` (C# 12). Все параметры — через `cmd.CreateParameter()`, что работает с любым провайдером; конкатенация строк отсутствует — это защита от инъекций, которую урок выделяет как критическую. `await using` на `conn`, `cmd`, `reader` гарантирует освобождение ресурсов даже при исключении — best practice из урока. Маппинг `reader.GetInt32(0)` — ручной, «бойлерплейт», о котором говорит урок. Collection expression `List<Product> list = []` — C# 12.

Селектор провайдера — `switch` expression (pattern matching C# 12): четыре кейса и default с `ArgumentException`. InMemory-кейс передаёт строку как имя базы (это демонстрирует, что InMemory игнорирует настоящую строку подключения). Смена провайдера — одна строка в конфиге, что иллюстрирует абстракцию EF Core. Таким образом, решение покрывает все ключевые концепции урока: два уровня доступа, роль провайдеров, LINQ→SQL, параметры против инъекций, `AsNoTracking`, `await using`, хранение строк подключения в конфиге, и ограничение InMemory.

#### Задания на углубление (бонус)
1. Добавьте метод `GetCountByActiveAsync(bool active)`, который через ADO.NET использует `ExecuteScalarAsync` для `SELECT COUNT(*) ...`, а через EF Core — `CountAsync(...)`. Сравните сгенерированный SQL через логирование (`LogTo(Console.WriteLine, LogLevel.Information)`).
2. Реализуйте «гибридный» репозиторий: основная логика на EF Core, но один тяжёлый отчёт через `FromSqlRaw` с параметром `SqlParameter`. Объясните, когда такой гибрид уместен.
3. Напишите unit-тесты (xUnit) для обоих репозиториев на InMemory-провайдере и отметьте в комментариях, какие сценарии InMemory не покроет (транзакции, FK-ограничения, RAW SQL-специфика).
4. Подключите PostgreSQL через Testcontainers в интеграционном тесте и убедитесь, что ADO.NET-репозиторий с `IsActive = 1` не работает (PostgreSQL требует `TRUE`). Обобщите селектор так, чтобы SQL адаптировался к диалекту.

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you join a small company called "WareHouseOK" as a junior developer. They have a product catalogue. Historically the application talked to the database directly through ADO.NET: every query was hand-written, parameters were sometimes glued with string concatenation (with injection risk), and mapping rows to objects lived inside a `while (reader.ReadAsync())` loop. The team wants to gradually migrate to EF Core to cut the boilerplate, but still keep the option to drop down to raw SQL for heavy reports. Your mentor gives you the first task: take a simple `Product` entity and implement a repository for it in two ways — with EF Core and with ADO.NET — behind a common interface, so the two can be compared in action. On top of that, the same code must run against different DBMSes: day-to-day development happens on SQLite (a file database, no server needed), CI runs on InMemory (fast tests), and production will use SQL Server or PostgreSQL.

The goal of the assignment is not merely "write two repositories" but to reinforce the lesson's concepts in practice: understand that EF Core sits on top of ADO.NET and reuses the same providers; that a provider is the bridge between the ORM and a SQL dialect; that a LINQ query is translated to SQL automatically while in ADO.NET you write SQL yourself; that `AsNoTracking()` saves memory on reads; that parameters defend against injection; that `await using` guarantees the release of connections and readers even under exceptions; and that `UseInMemoryDatabase` is not suitable for production because it is not a real database — it has no transactions and no foreign keys. You must learn to pick the tool for the task, not the task for the tool.

#### What to do step by step
1. Create a .NET 8 console project named `WarehouseCatalog`:
   `dotnet new console -n WarehouseCatalog -o WarehouseCatalog -f net8.0`
   Enter the folder: `cd WarehouseCatalog`.
2. Install the provider packages (all four, so you can switch in config):
   `dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 8.*`
   `dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 8.*`
   `dotnet add package Microsoft.EntityFrameworkCore.InMemory --version 8.*`
   `dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL --version 8.*`
   Install the ADO.NET providers: `dotnet add package Microsoft.Data.SqlClient` and `dotnet add package Microsoft.Data.Sqlite`.
3. Create the entity class `Product` with properties: `int Id`, `required string Name` (C# 12 `required`), `decimal Price`, `bool IsActive`, `DateTime CreatedAt`. Use `required` for the mandatory string field.
4. Create `ShopDbContext(DbContextOptions<ShopDbContext> options) : DbContext(options)` with `DbSet<Product> Products => Set<Product>();`. In `OnModelCreating` configure: `Name` — required, `HasMaxLength(200)`; `Price` — `HasPrecision(18, 2)`; `IsActive` — `HasDefaultValue(true)`. Add seed data through `HasData` (3–4 products).
5. Define a common interface `IProductRepository` with methods: `GetActiveAboveAsync(decimal minPrice, CancellationToken)`, `GetByIdAsync(int id, CancellationToken)`, `InsertAsync(Product, CancellationToken)`, `UpdatePriceAsync(int id, decimal newPrice, CancellationToken)`.
6. Implement `EfCoreProductRepository(ShopDbContext db)`: for reads use `AsNoTracking()`, for inserts — `Add` + `SaveChangesAsync`, for price updates — load the entity (tracked), change `Price`, call `SaveChangesAsync`.
7. Implement `AdoNetProductRepository` so it takes a connection factory `Func<DbConnection>` (an ADO.NET abstraction independent of the DBMS). Write every SQL string as a C# 12 raw string literal (`""" ... """`). Substitute values only through named parameters (`@id`, `@minPrice`, `@newPrice`). Wrap the reader and the connection in `await using`. Map rows to `Product` manually via `reader.GetInt32(0)` and so on.
8. Write an extension method `ChooseProvider(this DbContextOptionsBuilder b, string provider, string connStr)` as a `switch` expression (pattern matching) supporting `"SqlServer"`, `"Sqlite"`, `"Npgsql"`, `"InMemory"`, and throwing `ArgumentException` for the unknown case.
9. In `appsettings.json` add a section with `Provider` and `ConnectionStrings:Shop`. In `Program.cs` (top-level statements) read the configuration, register `ShopDbContext` through `AddDbContext` with the chosen provider, and register both repositories in DI.
10. In `Program.cs` call `EnsureCreatedAsync` (for the demo), insert a new product through the EF repository, then read the list of active products priced above a threshold through BOTH repositories and print the results side by side to confirm they are identical.
11. For SQLite use the connection string `"Data Source=shop.db"`. Run `dotnet run` and verify that the file `shop.db` is created and the output contains products from both repositories.
12. Switch the provider in `appsettings.json` to `"InMemory"` with the name `"shop-test"` and run again — the code must work unchanged. Note in comments why InMemory must not be used in production.

#### Requirements
The solution must compile under .NET 8 with no warnings and run with both SQLite and InMemory by changing a single line in `appsettings.json`. Use C# 12 features: `required` properties, primary constructors for the context and repositories, raw string literals for SQL, collection expressions for list initialization (`List<Product> list = [..];`), and pattern matching (`switch` expression) in the provider selector. All asynchronous methods must accept a `CancellationToken` and forward it to the underlying calls. All SQL queries in the ADO.NET repository must use parameters — string concatenation is forbidden. All `IDisposable`/`IAsyncDisposable` resources (`DbConnection`, `DbCommand`, `DbDataReader`) must be wrapped in `await using`. In the EF repository, all read-only queries must be marked `AsNoTracking()`. Keep connection strings in `appsettings.json`, not in code. The code must follow the lesson's best practices: parameters against injection, `await using` for resource release, `AsNoTracking` for reads, configuration outside code.

#### Pitfalls
- **SQL injection through concatenation.** The most common and dangerous ADO.NET mistake is writing `"... WHERE Name = '" + name + "'"`. The lesson warns against this directly. The fix is always to use parameters `@name`. In EF Core `FromSqlRaw` also requires parameters via `SqlParameter` or interpolation `FromSqlInterpolated`; otherwise injection is possible even in the ORM.
- **InMemory as a "pseudo-DB".** `UseInMemoryDatabase` is not a real DBMS: no transactions, no foreign keys, no relational integrity, data lives in process memory. The lesson stresses: only for fast unit tests of logic, never for production. For production-like tests use SQLite (the in-memory mode `"DataSource=:memory:"`) or Testcontainers.
- **Forgotten `AsNoTracking()`.** On heavy read-only queries without `AsNoTracking()` the context tracks every entity in the `ChangeTracker`, which bloats memory and slows things down. Always set `AsNoTracking()` for lists and reports.
- **Connection leaks.** A `SqlConnection` without `using`/`await using` is not returned to the pool, and under load the pool is exhausted. The same applies to `DbDataReader`. The rule: `await using var conn = factory();` and `await using var reader = await cmd.ExecuteReaderAsync(ct);`.
- **Accessing a `DbSet` after `Dispose`.** You cannot materialize lazily after the context is released. Always call `ToListAsync(ct)` before the context leaves scope.
- **`decimal` precision.** Without `HasPrecision(18, 2)` EF Core creates `decimal(18,2)` by default for SQL Server, but for some providers (Npgsql, for instance) the default behavior may differ, so explicit configuration is the best practice.
- **`required` and EF Core.** The `required string Name` property works with EF Core 8: on materialization EF Core sets the value. But be careful with value objects using `init` — entity constructors must be compatible.
- **`EnsureCreated` vs Migrations.** `EnsureCreatedAsync` is acceptable for a demo, but real projects use migrations (`dotnet ef migrations add`). The lesson mentions this as "schema versioning".
- **Cancellation.** Forward the `CancellationToken` down to the provider: `ExecuteReaderAsync(ct)`, `ToListAsync(ct)`, `SaveChangesAsync(ct)`. Otherwise the query cannot be cancelled.
- **Raw SQL dialect in ADO.NET.** Boolean values: SQL Server — `IsActive = 1`, SQLite — `IsActive = 1`, PostgreSQL — `IsActive = TRUE`. For cross-provider code either use EF Core (it translates itself) or parameterize and test against the target DBMS.

#### Acceptance criteria
- [ ] The `WarehouseCatalog` project builds under .NET 8 with no errors or warnings.
- [ ] The `Product` entity uses `required string Name` (C# 12).
- [ ] `ShopDbContext` uses a primary constructor and configures `Price` via `HasPrecision(18, 2)`.
- [ ] `DbSet<Product> Products => Set<Product>();` is defined via `Set<T>()`.
- [ ] `OnModelCreating` contains `HasMaxLength`, `HasDefaultValue`, `HasData` (3–4 records).
- [ ] The common interface `IProductRepository` is implemented by two classes.
- [ ] The EF repository uses `AsNoTracking()` on all read queries.
- [ ] The ADO.NET repository takes `Func<DbConnection>` (not a concrete `SqlConnection`).
- [ ] All SQL in ADO.NET is written as a raw string literal `"""..."""`.
- [ ] All parameters in ADO.NET are passed via `@name` — string concatenation is absent.
- [ ] `DbConnection`, `DbCommand`, `DbDataReader` are wrapped in `await using`.
- [ ] The provider selector is a `switch` expression with four cases and a default `ArgumentException`.
- [ ] `appsettings.json` contains `Provider` and `ConnectionStrings:Shop`.
- [ ] The app runs with SQLite (`Data Source=shop.db`) and creates the file.
- [ ] Switching `Provider` to `"InMemory"` works without code changes.
- [ ] The output shows identical results from both repositories.
- [ ] A comment or README note explains why InMemory is not for production.

#### Hints (no direct answer)
- To abstract away the DBMS in ADO.NET, use the base class `System.Data.Common.DbConnection` and `DbCommand`/`DbParameter`, not the concrete `SqlConnection`/`SqliteConnection`. A `Func<DbConnection>` factory hides the concrete type.
- To create a `DbParameter` without a concrete type, call `cmd.CreateParameter()` and set `ParameterName` and `Value`. This works for any provider.
- Place `AsNoTracking()` in the LINQ chain before the terminal `ToListAsync`.
- In the provider selector the InMemory case ignores the connection string and takes the database name: `b.UseInMemoryDatabase(connectionString)`.
- To create the `shop.db` file, `EnsureCreatedAsync` is enough — the schema is built from the model.
- To pass a `CancellationToken` in top-level `Program.cs`, use `CancellationToken.None`, or `await Task.Delay(..., ct)` for the demo.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — Reference solution for Homework M12-L01
// WarehouseCatalog — EF Core vs ADO.NET, providers
// EN: demonstrates both approaches on a single Product entity

using Microsoft.Data.Sqlite;       // SQLite ADO.NET provider
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using System.Data;
using System.Data.Common;

// ===== Entity =====
public class Product
{
    public int Id { get; set; }
    public required string Name { get; set; }      // C# 12 required
    public decimal Price { get; set; }
    public bool IsActive { get; set; }
    public DateTime CreatedAt { get; set; }
}

// ===== Context (primary constructor) =====
public class ShopDbContext(DbContextOptions<ShopDbContext> options) : DbContext(options)
{
    public DbSet<Product> Products => Set<Product>();   // Set<T>() — EF Core 8 idiom

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.Entity<Product>(e =>
        {
            e.HasKey(p => p.Id);
            e.Property(p => p.Name).IsRequired().HasMaxLength(200);
            e.Property(p => p.Price).HasPrecision(18, 2);   // decimal precision
            e.Property(p => p.IsActive).HasDefaultValue(true);
            e.Property(p => p.CreatedAt).HasDefaultValueSql("CURRENT_TIMESTAMP");
            e.HasData(
                new Product { Id = 1, Name = "Laptop Pro",  Price = 1200m, IsActive = true,  CreatedAt = new DateTime(2024,1,1) },
                new Product { Id = 2, Name = "USB Mouse",   Price = 25m,   IsActive = true,  CreatedAt = new DateTime(2024,1,2) },
                new Product { Id = 3, Name = "Keyboard",    Price = 75m,   IsActive = false, CreatedAt = new DateTime(2024,1,3) }
            );
        });
    }
}

// ===== Common interface =====
public interface IProductRepository
{
    Task<List<Product>> GetActiveAboveAsync(decimal minPrice, CancellationToken ct = default);
    Task<Product?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<int> InsertAsync(Product product, CancellationToken ct = default);
    Task<int> UpdatePriceAsync(int id, decimal newPrice, CancellationToken ct = default);
}

// ===== EF Core implementation =====
public class EfCoreProductRepository(ShopDbContext db) : IProductRepository
{
    public Task<List<Product>> GetActiveAboveAsync(decimal minPrice, CancellationToken ct = default) =>
        db.Products
          .AsNoTracking()                              // read, no tracking
          .Where(p => p.IsActive && p.Price >= minPrice)
          .OrderBy(p => p.Price)
          .ToListAsync(ct);

    public Task<Product?> GetByIdAsync(int id, CancellationToken ct = default) =>
        db.Products.AsNoTracking().FirstOrDefaultAsync(p => p.Id == id, ct);

    public async Task<int> InsertAsync(Product product, CancellationToken ct = default)
    {
        db.Products.Add(product);                      // EF builds INSERT itself
        await db.SaveChangesAsync(ct);
        return product.Id;                             // Id assigned after SaveChanges
    }

    public async Task<int> UpdatePriceAsync(int id, decimal newPrice, CancellationToken ct = default)
    {
        var product = await db.Products.FirstAsync(p => p.Id == id, ct);  // tracked
        product.Price = newPrice;
        return await db.SaveChangesAsync(ct);          // generates UPDATE
    }
}

// ===== ADO.NET implementation =====
public class AdoNetProductRepository(Func<DbConnection> connFactory) : IProductRepository
{
    public async Task<List<Product>> GetActiveAboveAsync(decimal minPrice, CancellationToken ct = default)
    {
        const string sql = """
            SELECT Id, Name, Price, IsActive, CreatedAt
            FROM Products
            WHERE IsActive = 1 AND Price >= @minPrice
            ORDER BY Price
            """;                                       // C# 12 raw string literal

        await using var conn = connFactory();          // await using — release
        if (conn.State != ConnectionState.Open) await conn.OpenAsync(ct);
        await using var cmd = conn.CreateCommand();
        cmd.CommandText = sql;
        var p = cmd.CreateParameter();                 // cross-provider parameter
        p.ParameterName = "@minPrice";
        p.Value = minPrice;
        cmd.Parameters.Add(p);

        List<Product> list = [];                       // C# 12 collection expression
        await using var reader = await cmd.ExecuteReaderAsync(ct);
        while (await reader.ReadAsync(ct))             // manual mapping
        {
            list.Add(new Product
            {
                Id        = reader.GetInt32(0),
                Name      = reader.GetString(1),
                Price     = reader.GetDecimal(2),
                IsActive  = reader.GetBoolean(3),
                CreatedAt = reader.GetDateTime(4)
            });
        }
        return list;
    }

    public async Task<Product?> GetByIdAsync(int id, CancellationToken ct = default)
    {
        const string sql = """
            SELECT Id, Name, Price, IsActive, CreatedAt
            FROM Products
            WHERE Id = @id
            """;
        await using var conn = connFactory();
        if (conn.State != ConnectionState.Open) await conn.OpenAsync(ct);
        await using var cmd = conn.CreateCommand();
        cmd.CommandText = sql;
        var p = cmd.CreateParameter(); p.ParameterName = "@id"; p.Value = id; cmd.Parameters.Add(p);

        await using var reader = await cmd.ExecuteReaderAsync(ct);
        if (!await reader.ReadAsync(ct)) return null;
        return new Product
        {
            Id = reader.GetInt32(0), Name = reader.GetString(1), Price = reader.GetDecimal(2),
            IsActive = reader.GetBoolean(3), CreatedAt = reader.GetDateTime(4)
        };
    }

    public async Task<int> InsertAsync(Product product, CancellationToken ct = default)
    {
        const string sql = """
            INSERT INTO Products (Name, Price, IsActive, CreatedAt)
            VALUES (@name, @price, @active, @created);
            SELECT last_insert_rowid();
            """;                                       // SQLite; SQL Server would use SCOPE_IDENTITY()
        await using var conn = connFactory();
        if (conn.State != ConnectionState.Open) await conn.OpenAsync(ct);
        await using var cmd = conn.CreateCommand();
        cmd.CommandText = sql;
        AddParam(cmd, "@name", product.Name);
        AddParam(cmd, "@price", product.Price);
        AddParam(cmd, "@active", product.IsActive);
        AddParam(cmd, "@created", product.CreatedAt);
        var rawId = await cmd.ExecuteScalarAsync(ct);
        return Convert.ToInt32(rawId);
    }

    public async Task<int> UpdatePriceAsync(int id, decimal newPrice, CancellationToken ct = default)
    {
        const string sql = """
            UPDATE Products SET Price = @newPrice WHERE Id = @id
            """;
        await using var conn = connFactory();
        if (conn.State != ConnectionState.Open) await conn.OpenAsync(ct);
        await using var cmd = conn.CreateCommand();
        cmd.CommandText = sql;
        AddParam(cmd, "@newPrice", newPrice);
        AddParam(cmd, "@id", id);
        return await cmd.ExecuteNonQueryAsync(ct);     // rows affected
    }

    private static void AddParam(DbCommand cmd, string name, object value)
    {
        var p = cmd.CreateParameter();
        p.ParameterName = name;
        p.Value = value;
        cmd.Parameters.Add(p);
    }
}

// ===== Provider selector (switch expression) =====
public static class ProviderSelector
{
    public static DbContextOptionsBuilder ChooseProvider(
        this DbContextOptionsBuilder b, string provider, string connStr) => provider switch
    {
        "SqlServer" => b.UseSqlServer(connStr),
        "Sqlite"    => b.UseSqlite(connStr),
        "Npgsql"    => b.UseNpgsql(connStr),
        "InMemory"  => b.UseInMemoryDatabase(connStr),  // ignores string, takes name
        _           => throw new ArgumentException($"Unknown provider: {provider}")
    };
}
```

**Line-by-line walk-through.** The `Product` entity uses `required string Name` — a C# 12 feature that plays nicely with EF Core 8: on materialization EF Core fills the mandatory property correctly. The `ShopDbContext` is written with a primary constructor `(DbContextOptions<ShopDbContext> options)` — again C# 12, boilerplate trimmed. The `DbSet<Product> Products => Set<Product>();` property is the EF Core 8 idiom (instead of an auto-property with `set`) recommended by Microsoft: `Set<T>()` returns the current tracked set. In `OnModelCreating` we explicitly set `HasPrecision(18, 2)` for `Price` (the lesson mentions decimal precision), `HasMaxLength(200)` and `HasDefaultValue(true)` for `IsActive`, plus `HasData` for seed data — this is the "schema versioning" concept from the lesson.

The `IProductRepository` interface is intentionally shared: the same semantics for both implementations, so you can swap them in DI and compare. The EF repository uses `AsNoTracking()` on reads — a direct application of the lesson's best practice ("saves memory and speeds up execution"); on insert it calls `Add` + `SaveChangesAsync`, and EF builds the `INSERT` itself; on update it loads the entity (tracked), changes the property, and calls `SaveChangesAsync`, which generates an `UPDATE` for the changed field only.

The ADO.NET repository takes `Func<DbConnection>` — a crucial point: we program against the base `System.Data.Common.DbConnection`, not a concrete `SqlConnection`, which gives cross-provider behavior (the same idea as EF Core over ADO.NET). All SQL strings are raw string literals `"""..."""` (C# 12). All parameters go through `cmd.CreateParameter()`, which works with any provider; string concatenation is absent — this is the injection defense the lesson calls critical. `await using` on `conn`, `cmd`, and `reader` guarantees resource release even on exceptions — a lesson best practice. The `reader.GetInt32(0)` mapping is manual, the "boilerplate" the lesson talks about. The collection expression `List<Product> list = []` is C# 12.

The provider selector is a `switch` expression (C# 12 pattern matching): four cases and a default that throws `ArgumentException`. The InMemory case passes the string as the database name (demonstrating that InMemory ignores a real connection string). Switching a provider is a single line in config, which illustrates the EF Core abstraction. Thus the solution covers every key concept of the lesson: the two access levels, the role of providers, LINQ→SQL, parameters against injection, `AsNoTracking`, `await using`, keeping connection strings in config, and the InMemory limitation.

#### Going deeper (bonus)
1. Add a `GetCountByActiveAsync(bool active)` method that uses `ExecuteScalarAsync` with `SELECT COUNT(*) ...` in ADO.NET and `CountAsync(...)` in EF Core. Compare the generated SQL via logging (`LogTo(Console.WriteLine, LogLevel.Information)`).
2. Implement a "hybrid" repository: the main logic on EF Core, but one heavy report through `FromSqlRaw` with an `SqlParameter` parameter. Explain when such a hybrid is appropriate.
3. Write xUnit unit tests for both repositories on the InMemory provider, and note in comments which scenarios InMemory will not cover (transactions, FK constraints, raw-SQL specifics).
4. Connect PostgreSQL via Testcontainers in an integration test and confirm that the ADO.NET repository with `IsActive = 1` does not work (PostgreSQL requires `TRUE`). Generalize the selector so the SQL adapts to the dialect.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект собирается под .NET 8 без ошибок и предупреждений.
- [ ] (RU) Реализованы оба репозитория за общим интерфейсом `IProductRepository`.
- [ ] (RU) ADO.NET-репозиторий использует только параметры, без конкатенации строк.
- [ ] (RU) Все ресурсы обёрнуты в `await using`.
- [ ] (RU) EF-репозиторий использует `AsNoTracking()` на чтении.
- [ ] (RU) Селектор провайдера — `switch` expression с 4 кейсами.
- [ ] (RU) Строки подключения в `appsettings.json`, не в коде.
- [ ] (RU) Код запускается с SQLite и InMemory сменой одной строки.
- [ ] (RU) Вывод сравнивает результаты обоих репозиториев.
- [ ] (RU) Отмечено, почему InMemory не для продакшена.
- [ ] (EN) The project builds under .NET 8 with no errors or warnings.
- [ ] (EN) Both repositories are implemented behind the common `IProductRepository` interface.
- [ ] (EN) The ADO.NET repository uses only parameters, with no string concatenation.
- [ ] (EN) All resources are wrapped in `await using`.
- [ ] (EN) The EF repository uses `AsNoTracking()` on reads.
- [ ] (EN) The provider selector is a `switch` expression with 4 cases.
- [ ] (EN) Connection strings live in `appsettings.json`, not in code.
- [ ] (EN) The code runs with SQLite and InMemory by changing one line.
- [ ] (EN) The output compares results from both repositories.
- [ ] (EN) A note explains why InMemory is not for production.

#### Ресурсы / Resources
- [Microsoft Learn — EF Core](https://learn.microsoft.com/ef/core/)
- [Microsoft Learn — ADO.NET overview](https://learn.microsoft.com/dotnet/framework/data/adonet/)
- [EF Core providers list](https://learn.microsoft.com/ef/core/providers/)
- [Npgsql EF Core provider](https://www.npgsql.org/efcore/)
- [EF Core — Performance (AsNoTracking)](https://learn.microsoft.com/ef/core/performance/)
- [Microsoft.Data.SqlClient docs](https://learn.microsoft.com/sql/connect/ado-net/microsoft-ado-net-sql-server)
