---
[← К уроку M12-L08](lesson-M12-L08-raw-sql.md) | [⬆ К модулю M12](../README.md) | [Следующее ДЗ →](homework-M12-L09-concurrency-tokens.md)
---

### Домашнее задание M12-L08: Raw SQL, FromSqlRaw, хранимые процедуры / Homework M12-L08: Raw SQL, FromSqlRaw, stored procedures

**Урок / Lesson:** M12-L08
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться осознанно выбирать между LINQ и raw SQL, безопасно параметризовать запросы через `FromSqlRaw` и `FromSqlInterpolated`, материализовать произвольные проекции через `SqlQueryRaw<T>`, вызывать хранимые процедуры через `EXEC ... @Param = {0}` и оформлять raw-доступ изолированным репозиторием с интеграционными тестами. (EN) Learn to choose deliberately between LINQ and raw SQL, parameterize queries safely with `FromSqlRaw` and `FromSqlInterpolated`, materialize arbitrary projections with `SqlQueryRaw<T>`, call stored procedures through `EXEC ... @Param = {0}`, and isolate raw access behind a repository covered by integration tests.

#### Связь с уроком / Connection to the lesson

(RU) Урок M12-L08 показывает четыре API EF Core для сырого SQL: `FromSqlRaw`, `FromSqlInterpolated`, `SqlQueryRaw<T>`/`SqlQuery<T>` и `ExecuteSqlRaw`/`ExecuteSqlInterpolated`. Домашнее задание заставляет применить каждый из них к одной реалистичной модели магазина, чтобы вы на практике почувствовали разницу между материализацией в сущность и проекцией в DTO, а также на себе испытали главную ловушку урока — требование, чтобы SQL возвращал все столбцы сущности в правильном порядке. Вы также отработаете безопасный вызов хранимых процедур и правило «никакой конкатенации строк».

(EN) Lesson M12-L08 introduces four EF Core APIs for raw SQL: `FromSqlRaw`, `FromSqlInterpolated`, `SqlQueryRaw<T>`/`SqlQuery<T>`, and `ExecuteSqlRaw`/`ExecuteSqlInterpolated`. This homework makes you apply each of them to one realistic shop model so that, in practice, you feel the difference between materializing into an entity and projecting into a DTO, and experience the lesson's main trap first-hand — the requirement that SQL return all entity columns in the correct order. You will also practice safe stored-procedure invocation and the "no string concatenation" rule.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы присоединились к команде интернет-магазина «Шестерёнка», где каталогом управляет EF Core 8 поверх SQL Server. Подавляющее большинство чтений отлично ложится на LINQ, но в каталоге есть четыре горячих места, где LINQ либо медленный, либо неспособен выразить нужную семантику. Во-первых, фильтр активных товаров по категории и минимальной цене вызывается на главной странице тысячи раз в минуту, и DBA прислал уже выверенный план с покрывающим индексом — LINQ генерирует `sp_executesql` с другим порядком предикатов, и план «плывёт». Во-вторых, поиск по подстроке в названии должен поддерживать `LIKE` с liderирующим wildcard, и команда хочет гарантировать параметризацию на уровне компилятора, чтобы ни один джун случайно не вклеил `userInput` в SQL-строку. В-третьих, дашборд категории показывает агрегированную сводку — среднюю цену, медиану через `PERCENTILE_CONT`, количество активных и долю распродажи — это оконные функции, которых LINQ не выражает напрямую. В-четвёртых, ночью крутится хранимая процедура `dbo.DeactivateExpired`, которая переводит в неактивное состояние товары с истёкшим сроком годности и логирует это в отдельную таблицу внутри одной транзакции; `SaveChanges` здесь не подходит, потому что нужны серверные `UPDATE ... OUTPUT` и journal-вставка в одном ROUND.

Руководство требует: никаких конкатенаций строк в SQL ни при каких условиях, все raw-вызовы изолированы в классе `ProductCatalogRepository`, каждый публичный метод снабжён XML-комментарием с причиной выбора raw SQL, а репозиторий покрыт интеграционными тестами на реальном SQL Server (допускается LocalDB или контейнер). Вы должны не просто «сделать, чтобы работало», а продемонстрировать понимание, какой API уместен в каждой из четырёх точек и почему. Это и есть центральный навык урока — осознанный выбор между LINQ и raw SQL, а не reflexive «напишу сырой SQL, потому что LINQ лень разбирать».

#### Что нужно сделать (пошагово)

1. Создайте решение и проект. Выполните `dotnet new sln -n GearShop` в пустой папке, затем `dotnet new console -n GearShop.Catalog -o src/GearShop.Catalog -f net8.0` и `dotnet sln add src/GearShop.Catalog/GearShop.Catalog.csproj`. Добавьте тестовый проект: `dotnet new xunit -n GearShop.Catalog.Tests -o tests/GearShop.Catalog.Tests -f net8.0` и `dotnet sln add tests/GearShop.Catalog.Tests/GearShop.Catalog.Tests.csproj`. В основном проекте выполните `dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 8.0.*`, в тестовом — `dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 8.0.*` и `dotnet add package xunit.runner.visualstudio`. Убедитесь, что `dotnet build` проходит без предупреждений.

2. Смоделируйте домен. В файле `Domain/Product.cs` опишите класс `Product` со свойствами `Id`, `Name`, `Price`, `IsActive`, `CategoryId`, `ExpiresAtUtc` (nullable `DateTime?`) и `DiscountPercent`. В `Domain/Category.cs` — класс `Category` с `Id` и `Name`. В `Domain/Dtos.cs` опишите `record ProductSummaryDto(int Id, string Name, decimal Price)`, `record CategoryDashboardDto(int CategoryId, string CategoryName, decimal AveragePrice, decimal MedianPrice, int ActiveCount, decimal DiscountShare)` и `record ScalarValue<T>(T Value)` — последний пригодится для безопасного чтения скаляров через `SqlQueryRaw<T>`, когда провайдер возвращает столбец с именем `Value`.

3. Настройте контекст. В `Data/ShopDbContext.cs` объявите `DbSet<Product> Products` и `DbSet<Category> Categories`, переопределите `OnConfiguring` так, чтобы строка подключения бралась из `Configuration` через `IConfiguration`, а в `OnModelCreating` задайте схему `dbo`, индекс по `(CategoryId, IsActive, Price)` и обязательное `Name`. В `Data/DbSeeder.cs` реализуйте метод расширения `EnsureSeedAsync`, который через `ExecuteSqlRawAsync` выполняет `IF OBJECT_ID('dbo.DeactivateExpired', 'P') IS NULL CREATE PROCEDURE dbo.DeactivateExpired @Cutoff datetime AS BEGIN SET NOCOUNT ON; UPDATE Products SET IsActive = 0 OUTPUT inserted.Id, GETUTCDATE() INTO ProductDeactivationLog(ProductId, DeactivatedAtUtc) WHERE ExpiresAtUtc IS NOT NULL AND ExpiresAtUtc <= @Cutoff AND IsActive = 1; END` — это безопасно идемпотентно и не требует миграций в рамках ДЗ. Также создайте процедуру `dbo.GetActiveProductsByCategory`, принимающую `@CategoryId int`, `@MinPrice decimal(18,2)` и возвращающую полный набор столбцов `Product` — это важно для демонстрации материализации в сущность.

4. Реализуйте репозиторий. В `Data/ProductCatalogRepository.cs` создайте класс с шестью асинхронными методами: `GetActiveByCategoryRawAsync(int categoryId, decimal minPrice)` через `FromSqlRaw` с плейсхолдерами `{0}`, `{1}` и последующей композицией `OrderBy(p => p.Name)`; `SearchByNameInterpolatedAsync(string nameFragment)` через `FromSqlInterpolated` с `LIKE '%' + {nameFragment} + '%'`; `GetDashboardAsync(int categoryId)` через `SqlQueryRaw<CategoryDashboardDto>` с оконной функцией `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY Price) OVER (...)`, `AVG`, `COUNT` и `SUM(CASE WHEN DiscountPercent > 0 THEN 1 ELSE 0 END)`; `CountActiveAsync()` через `SqlQueryRaw<int>` со столбцом-алиасом `Value`; `CallStoredProcedureAsync(int categoryId, decimal minPrice)` через `FromSqlRaw("EXEC dbo.GetActiveProductsByCategory @CategoryId = {0}, @MinPrice = {1}", ...)`; `DeactivateExpiredAsync(DateTime cutoffUtc)` через `ExecuteSqlInterpolatedAsync($"EXEC dbo.DeactivateExpired @Cutoff = {cutoffUtc}")`. Каждый метод снабдите XML-комментарием с причиной raw SQL.

5. Напишите интеграционные тесты. В `tests/.../ProductCatalogRepositoryTests.cs` используйте `DbContextOptionsBuilder` с `UseInMemoryDatabase` только для smoke-тестов модели, а для raw-SQL тестов поднимайте реальный SQL Server LocalDB через строку `Server=(localdb)\MSSQLLocalDB;Database=GearShopTests;Trusted_Connection=True;TrustServerCertificate=True`, вызывайте `EnsureSeedAsync`, чистите таблицу `TRUNCATE TABLE Products`, вставляйте фикстуры и проверяйте, что `GetActiveByCategoryRawAsync(2, 100m)` возвращает ровно ожидаемые три строки, `SearchByNameInterpolatedAsync("bolt")` находит `M8 Bolt` и `M8 Bolted Flange`, `GetDashboardAsync(2)` даёт `AveragePrice` ≈ 150 и `MedianPrice` ≈ 140, `CountActiveAsync()` равен числу активных, `CallStoredProcedureAsync` совпадает по составу с `GetActiveByCategoryRawAsync`, а `DeactivateExpiredAsync` возвращает количество деактивированных и переводит строки в `IsActive = 0`. После теста — teardown через `EnsureDeletedAsync`.

6. Запустите и зафиксируйте вывод. Выполните `dotnet test --logger "console;verbosity=normal"` — все тесты зелёные. Затем `dotnet run --project src/GearShop.Catalog` — программа печатает по одной строке на каждый метод репозитория с числом возвращённых строк и одним примером значения, доказывая, что все шесть путей работают на реальной БД.

#### Требования к решению

Решение должно компилироваться под .NET 8 / C# 12 без предупреждений и использовать современные возможности языка: top-level statements в `Program.cs`, коллекционные выражения для инициализации фикстур (`new Product { ... }` можно группировать через `[ p1, p2, p3 ]` в массив), шаблоны списков там, где это уместно, и raw-string литералы `"""..."""` для многострочных SQL, чтобы не экранировать кавычки внутри `EXEC` и `PERCENTILE_CONT`. Все шесть методов репозитория обязаны быть асинхронными и принимать `CancellationToken` — даже если в ДЗ вы его не прокидываете в каждый вызов, сигнатура должна быть готова. Никакого `string.Concat` или интерполяции, которая попадает в `FromSqlRaw` как литерал: только `FromSqlInterpolated` для интерполированных строк и только плейсхолдеры `{0}` для `FromSqlRaw`. Имена таблиц и столбцов статичны — динамизация схемы через пользователя не требуется и будет считаться ошибкой проектирования.

Каждый публичный метод репозитория снабжается XML-комментарием `<summary>` минимум с тремя полями: причина raw SQL (производительность / конструкция СУБД / хранимая процедура / массовая операция), ожидаемый план или измеренная выгода, и напоминание, что обновление сущностей, возвращённых raw-запросом, требует контекста без `AsNoTracking`. Чтения, не ведущие к `SaveChanges`, должны использовать `AsNoTracking()` — это прямо рекомендация из урока. Хранимые процедуры вызываются строго через `EXEC dbo.Name @Param = {0}` — именованные параметры защищают от ошибок порядка. Тесты изолированы: каждый тест создаёт свою базу с уникальным суффиксом или использует `TRUNCATE` в `Setup`, чтобы параллельный запуск xUnit не давал гонок. Запрещено использовать `Database.SqlQuery<T>` для материализации в сущность `Product` — для сущностей только `FromSql*`, для DTO и скаляров только `SqlQueryRaw<T>`/`SqlQuery<T>`.

#### Тонкости и подводные камни

Главная ловушка `FromSqlRaw` и `FromSqlInterpolated` — обязательное возвращение **всех** столбцов сущности в том же порядке и с теми же именами. Если ваш `SELECT` забудет `IsActive`, EF Core бросит исключение материализации ещё до того, как вы увидите данные. Поэтому для хранимой процедуры `GetActiveProductsByCategory` убедитесь, что `SELECT` внутри перечисляет ровно `Id, Name, Price, IsActive, CategoryId, ExpiresAtUtc, DiscountPercent`. Псевдонимы столбцов тоже должны совпадать со свойствами; `SELECT p.Id AS ProductId` сломает материализацию в `Product.Id`. Для DTO всё проще: `SqlQueryRaw<CategoryDashboardDto>` сопоставляет по именам столбцов, поэтому алиасы вида `AVG(Price) AS AveragePrice` обязательны — иначе получите нули или ошибку.

Вторая ловушка — композиция LINQ поверх raw SQL. Урок прямо предупреждает: после `FromSqlRaw` можно вызывать `Where`, `OrderBy`, но EF Core оборачивает ваш SQL в подзапрос, и это работает только если базовый SQL начинается с `SELECT`. Если ваш raw SQL содержит `GROUP BY` или `INTO`, композиция падает — переносите агрегацию в `SqlQueryRaw<T>` с DTO. Третья ловушка — `AsNoTracking`. По умолчанию raw-чтения трекаются, как обычные LINQ; если вы потом вызовете `SaveChanges` с пересечением по `Id`, EF попытается обновить строки, которые вы не собирались менять. Четвёртая — `ExecuteSqlInterpolated` возвращает число затронутых строк, а не сущности; не пытайтесь материализовать результат, его просто нет. Пятая — скаляр через `SqlQueryRaw<int>` возвращает последовательность, а не одно значение: нужно `ToListAsync` и взять `[0]`, либо использовать `.FirstAsync()` — и обязательно алиас `AS Value`, потому что EF Core 8 ожидает имя `Value` для скалярных проекций при отражении по умолчанию.

Шестая и самая опасная — SQL-инъекция. Любая конкатенация `"... WHERE Name = '" + input + "'"` это прямой путь к утечке данных; даже если сегодня `input` приходит из доверенного источника, завтра его подключат к форме. Урок требует: данные — только как параметры, имена таблиц/столбцов — только через белый список констант. В ДЗ нет динамических имён, поэтому соблазна меньше, но в коде ревью ищите любую интерполяцию, попадающую в `FromSqlRaw` без фигурных скобок-плейсхолдеров — это автоматический red flag.

#### Критерии приёмки

- [ ] Решение `dotnet build` без предупреждений под .NET 8 / C# 12.
- [ ] `Program.cs` использует top-level statements.
- [ ] Многострочный SQL оформлен raw-string литералами `"""..."""`.
- [ ] Все шесть методов `ProductCatalogRepository` асинхронны с `CancellationToken`.
- [ ] `GetActiveByCategoryRawAsync` использует `FromSqlRaw` с `{0}`, `{1}` и `AsNoTracking`.
- [ ] `SearchByNameInterpolatedAsync` использует `FromSqlInterpolated`, `LIKE` с wildcard.
- [ ] `GetDashboardAsync` использует `SqlQueryRaw<CategoryDashboardDto>` с `PERCENTILE_CONT`.
- [ ] `CountActiveAsync` использует `SqlQueryRaw<int>` с алиасом `Value`.
- [ ] `CallStoredProcedureAsync` вызывает `EXEC dbo.GetActiveProductsByCategory @CategoryId = {0}, @MinPrice = {1}`.
- [ ] `DeactivateExpiredAsync` вызывает `ExecuteSqlInterpolatedAsync` и возвращает число строк.
- [ ] Каждый метод имеет XML-комментарий с причиной raw SQL.
- [ ] Чтения без обновлений используют `AsNoTracking()`.
- [ ] Интеграционные тесты проходят на реальном SQL Server (LocalDB/контейнер).
- [ ] Тесты проверяют состав и значения, а не только отсутствие исключений.
- [ ] В коде нет ни одной конкатенации строк в SQL.

#### Подсказки (без прямого ответа)

- Если материализация падает с «нельзя материализовать», проверьте, что SQL возвращает все столбцы сущности и в том же порядке, что и свойства.
- Для `SqlQueryRaw<int>` не ждите одно число — это последовательность; возьмите первый элемент.
- `PERCENTILE_CONT` требует `WITHIN GROUP (ORDER BY ...)`, и результат — `numeric`; приведите к `decimal` в DTO, а не в SQL.
- Raw-string литералы `"""..."""` позволяют писать `EXEC dbo.DeactivateExpired @Cutoff = {0}` без экранирования кавычек.
- Помните: `FromSqlInterpolated` превращает каждое `{value}` в `DbParameter` автоматически — это и есть ваша защита от инъекции.
- Если `Include` после raw SQL падает, убедитесь, что базовый запрос возвращает полную сущность, а не проекцию.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 / EF Core 8 — GearShop.Catalog
// Эталонное решение ДЗ M12-L08
// Reference solution for homework M12-L08

using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;

namespace GearShop.Catalog.Data;

public sealed class ProductCatalogRepository(ShopDbContext db)
{
    // Причина raw SQL: DBA прислал выверенный план под покрывающий индекс;
    // LINQ генерирует другой порядок предикатов, и план "плывёт".
    // Reason: a DBA tuned the plan for a covering index; LINQ reorders
    // predicates and the plan degrades.
    public async Task<List<Product>> GetActiveByCategoryRawAsync(
        int categoryId, decimal minPrice, CancellationToken ct = default)
    {
        // Плейсхолдеры {0},{1} → DbParameter, не конкатенация.
        // Placeholders become DbParameter, not string concatenation.
        return await db.Products
            .FromSqlRaw("""
                SELECT Id, Name, Price, IsActive, CategoryId, ExpiresAtUtc, DiscountPercent
                FROM dbo.Products
                WHERE CategoryId = {0} AND Price >= {1} AND IsActive = 1
                """, categoryId, minPrice)
            .AsNoTracking()           // чтение без обновлений / read-only
            .OrderBy(p => p.Name)     // композиция поверх подзапроса / composition over subquery
            .ToListAsync(ct);
    }

    // Причина: гарантировать параметризацию на уровне компилятора, чтобы
    // никакой джун не вклеил userInput в SQL-строку.
    // Reason: enforce compile-time parameterization so no junior developer
    // ever pastes userInput into the SQL string.
    public async Task<List<Product>> SearchByNameInterpolatedAsync(
        string nameFragment, CancellationToken ct = default)
    {
        // FromSqlInterpolated: каждое {…} → DbParameter автоматически.
        // Each {…} becomes a DbParameter automatically.
        var pattern = $"%{nameFragment}%";
        return await db.Products
            .FromSqlInterpolated($"""
                SELECT Id, Name, Price, IsActive, CategoryId, ExpiresAtUtc, DiscountPercent
                FROM dbo.Products
                WHERE Name LIKE {pattern} AND IsActive = 1
                """)
            .AsNoTracking()
            .ToListAsync(ct);
    }

    // Причина: оконные функции PERCENTILE_CONT, которые LINQ не выражает.
    // Reason: PERCENTILE_CONT window function not expressible in LINQ.
    public async Task<List<CategoryDashboardDto>> GetDashboardAsync(
        int categoryId, CancellationToken ct = default)
    {
        // SqlQueryRaw<T>: материализация в DTO по именам столбцов.
        // SqlQueryRaw<T>: materialize into DTO by column names.
        return await db.Database
            .SqlQueryRaw<CategoryDashboardDto>($"""
                SELECT
                    {categoryId} AS CategoryId,
                    c.Name AS CategoryName,
                    AVG(p.Price) AS AveragePrice,
                    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY p.Price)
                        OVER (PARTITION BY p.CategoryId) AS MedianPrice,
                    COUNT(*) AS ActiveCount,
                    CAST(SUM(CASE WHEN p.DiscountPercent > 0 THEN 1 ELSE 0 END)
                         AS decimal(18,4)) / COUNT(*) AS DiscountShare
                FROM dbo.Products p
                JOIN dbo.Categories c ON c.Id = p.CategoryId
                WHERE p.CategoryId = {categoryId} AND p.IsActive = 1
                GROUP BY c.Name, p.CategoryId
                """)
            .ToListAsync(ct);
    }

    // Причина: скалярный COUNT, удобнее как SqlQueryRaw<int>, чем FromSqlRaw
    // с сущностью, ради одного числа.
    // Reason: scalar COUNT is cleaner as SqlQueryRaw<int> than FromSqlRaw
    // with an entity just for one number.
    public async Task<int> CountActiveAsync(CancellationToken ct = default)
    {
        var rows = await db.Database
            .SqlQueryRaw<int>("SELECT COUNT(*) AS Value FROM dbo.Products WHERE IsActive = 1")
            .ToListAsync(ct);
        return rows.Count == 0 ? 0 : rows[0];
    }

    // Причина: интеграция с существующей хранимой процедурой DBA.
    // Reason: integration with a DBA-owned stored procedure.
    public async Task<List<Product>> CallStoredProcedureAsync(
        int categoryId, decimal minPrice, CancellationToken ct = default)
    {
        // Именованные параметры защищают от ошибок порядка аргументов.
        // Named parameters protect against argument-order mistakes.
        return await db.Products
            .FromSqlRaw(
                "EXEC dbo.GetActiveProductsByCategory @CategoryId = {0}, @MinPrice = {1}",
                categoryId, minPrice)
            .AsNoTracking()
            .ToListAsync(ct);
    }

    // Причина: серверная массовая операция с OUTPUT INTO journal-таблицу
    // в одной серверной транзакции — SaveChanges так не умеет.
    // Reason: server-side bulk operation with OUTPUT INTO a journal table
    // inside one server transaction — SaveChanges cannot do this.
    public async Task<int> DeactivateExpiredAsync(DateTime cutoffUtc, CancellationToken ct = default)
        => await db.Database.ExecuteSqlInterpolatedAsync(
            $"EXEC dbo.DeactivateExpired @Cutoff = {cutoffUtc}", ct);
}
```

Разбор по строкам. Конструктор `ProductCatalogRepository(ShopDbContext db)` использует primary constructor C# 12 — поле `db` захватывается автоматически, без явного поля. Метод `GetActiveByCategoryRawAsync` — канонический пример из урока: `FromSqlRaw` с плейсхолдерами `{0}`, `{1}`, затем `AsNoTracking` (чтение без обновлений, экономия памяти и CPU) и `OrderBy` как композиция — EF Core обернёт наш `SELECT` в подзапрос и добавит `ORDER BY` снаружи. Raw-string литерал `"""..."""` позволяет многострочный SQL без `\r\n` и экранирования кавычек — это идиома C# 11+, которую урок косвенно предполагает через «C# 12+». `SearchByNameInterpolatedAsync` deliberately выбирает `FromSqlInterpolated`, чтобы параметризация была видна компилятору: интерполированная строка передаётся как `FormattableString`, и EF Core превращает каждое `{pattern}` в `DbParameter` — это та самая «автоматическая защита от инъекции», о которой говорит урок. `GetDashboardAsync` применяет `SqlQueryRaw<CategoryDashboardDto>` — API EF Core 7+, идеальный для проекций: имена столбцов `AveragePrice`, `MedianPrice` совпадают со свойствами рекорда, поэтому материализация проходит без шума. `PERCENTILE_CONT` — оконная функция, недоступная в LINQ, что прямо соответствует пункту «when to choose raw SQL» из урока.

`CountActiveAsync` иллюстрирует тонкость скаляров: `SqlQueryRaw<int>` возвращает последовательность, а не одно значение; мы материализуем в `List<int>` и берём `[0]`, при этом алиас `AS Value` обязателен, потому что EF Core 8 при отражении в `int` ищет столбец `Value`. `CallStoredProcedureAsync` вызывает хранимую процедуру через `EXEC dbo.GetActiveProductsByCategory @CategoryId = {0}, @MinPrice = {1}` — именованные параметры защищают от ошибок порядка, а плейсхолдеры гарантируют параметризацию; результат материализуется в сущность `Product`, поэтому процедура внутри обязана `SELECT` ровно те же столбцы в том же порядке. `DeactivateExpiredAsync` использует `ExecuteSqlInterpolatedAsync` — для DML/процедур без набора строк; возвращается число затронутых строк, что мы и отдаём наверх. Вся六точечная цепочка демонстрирует, что выбор API не произволен: `FromSql*` — для сущностей, `SqlQueryRaw<T>` — для DTO и скаляров, `ExecuteSql*` — для DML, и каждый выбор обоснован причиной из урока.

#### Задания на углубление (бонус)

1. Добавьте метод `GetTopExpensivePerCategoryAsync`, возвращающий три самых дорогих активных товара в каждой категории через `ROW_NUMBER() OVER (PARTITION BY CategoryId ORDER BY Price DESC)` и проекцию в новый DTO `TopProductDto`. Сравните план с LINQ-вариантом через `GroupBy` и `.Take(3)`.
2. Реализуйте динамический фильтр по произвольному набору категорий: пользователь передаёт `int[] categoryIds`, и вы формируете `WHERE CategoryId IN (...)` безопасно — через `SqlParameter` с табличным типом или через `json`/`OPENJSON`. Покажите, что конкатенация `string.Join` запрещена, и обоснуйте альтернативу.
3. Добавьте кеш на стороне репозитория через `IMemoryCache` с ключом, зависящим от параметров, и инвалидируйте его в `DeactivateExpiredAsync`. Покажите, что raw-чтения тоже можно кешировать, но только read-only сущности.
4. Покройте репозиторий тестами на SQL-инъекцию: передайте `nameFragment = "'; DROP TABLE Products;--"` и убедитесь, что таблица на месте, а поиск возвращает пустой список.

---

## Statement in English / Постановка на английском

#### Context & motivation

You have joined the team of the "Gear" online shop, where the catalog is managed by EF Core 8 on top of SQL Server. The overwhelming majority of reads map cleanly to LINQ, but the catalog has four hot spots where LINQ is either too slow or unable to express the required semantics. First, the filter for active products by category and minimum price is invoked on the home page thousands of times per minute, and the DBA has sent an already-tuned plan that relies on a covering index; LINQ generates an `sp_executesql` call with a different predicate order, and the plan "drifts". Second, the substring search in the product name must support a `LIKE` with a leading wildcard, and the team wants to guarantee parameterization at the compiler level so that no junior developer ever accidentally pastes `userInput` into the SQL string. Third, the category dashboard shows an aggregate summary — average price, median via `PERCENTILE_CONT`, the active count and the discount share — these are window functions that LINQ cannot express directly. Fourth, at night a stored procedure `dbo.DeactivateExpired` runs that flips expired products to inactive and logs the change into a separate table inside a single server-side transaction; `SaveChanges` is not a fit here because the procedure needs `UPDATE ... OUTPUT` and a journal insert in one round trip.

Management requires: no string concatenation in SQL under any circumstance, all raw calls isolated inside a `ProductCatalogRepository` class, every public method annotated with an XML comment stating the reason raw SQL was chosen, and the repository covered by integration tests on a real SQL Server instance (LocalDB or a container is acceptable). You must not simply "make it work" but demonstrate understanding of which API fits each of the four spots and why. That is the central skill of the lesson — a deliberate choice between LINQ and raw SQL, not a reflexive "I will write raw SQL because LINQ is too much bother to figure out".

#### What to do step by step

1. Create the solution and projects. Run `dotnet new sln -n GearShop` in an empty folder, then `dotnet new console -n GearShop.Catalog -o src/GearShop.Catalog -f net8.0` and `dotnet sln add src/GearShop.Catalog/GearShop.Catalog.csproj`. Add a test project: `dotnet new xunit -n GearShop.Catalog.Tests -o tests/GearShop.Catalog.Tests -f net8.0` and `dotnet sln add tests/GearShop.Catalog.Tests/GearShop.Catalog.Tests.csproj`. In the main project run `dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 8.0.*`; in the test project run `dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 8.0.*` and `dotnet add package xunit.runner.visualstudio`. Confirm that `dotnet build` succeeds with no warnings.

2. Model the domain. In `Domain/Product.cs` declare a `Product` class with properties `Id`, `Name`, `Price`, `IsActive`, `CategoryId`, `ExpiresAtUtc` (nullable `DateTime?`) and `DiscountPercent`. In `Domain/Category.cs` declare a `Category` class with `Id` and `Name`. In `Domain/Dtos.cs` declare `record ProductSummaryDto(int Id, string Name, decimal Price)`, `record CategoryDashboardDto(int CategoryId, string CategoryName, decimal AveragePrice, decimal MedianPrice, int ActiveCount, decimal DiscountShare)` and `record ScalarValue<T>(T Value)` — the last one is handy for safely reading scalars through `SqlQueryRaw<T>` when the provider returns a column named `Value`.

3. Configure the context. In `Data/ShopDbContext.cs` declare `DbSet<Product> Products` and `DbSet<Category> Categories`, override `OnConfiguring` so that the connection string is taken from `IConfiguration`, and in `OnModelCreating` set the schema to `dbo`, add an index on `(CategoryId, IsActive, Price)` and make `Name` required. In `Data/DbSeeder.cs` implement an `EnsureSeedAsync` extension method that, via `ExecuteSqlRawAsync`, runs `IF OBJECT_ID('dbo.DeactivateExpired', 'P') IS NULL CREATE PROCEDURE dbo.DeactivateExpired @Cutoff datetime AS BEGIN SET NOCOUNT ON; UPDATE Products SET IsActive = 0 OUTPUT inserted.Id, GETUTCDATE() INTO ProductDeactivationLog(ProductId, DeactivatedAtUtc) WHERE ExpiresAtUtc IS NOT NULL AND ExpiresAtUtc <= @Cutoff AND IsActive = 1; END` — this is safely idempotent and does not require migrations within the homework. Also create a procedure `dbo.GetActiveProductsByCategory` that accepts `@CategoryId int` and `@MinPrice decimal(18,2)` and returns the full set of `Product` columns — this matters for demonstrating entity materialization.

4. Implement the repository. In `Data/ProductCatalogRepository.cs` create a class with six asynchronous methods: `GetActiveByCategoryRawAsync(int categoryId, decimal minPrice)` via `FromSqlRaw` with placeholders `{0}`, `{1}` and subsequent `OrderBy(p => p.Name)` composition; `SearchByNameInterpolatedAsync(string nameFragment)` via `FromSqlInterpolated` with `LIKE '%' + {nameFragment} + '%'`; `GetDashboardAsync(int categoryId)` via `SqlQueryRaw<CategoryDashboardDto>` with a window function `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY Price) OVER (...)`, `AVG`, `COUNT` and `SUM(CASE WHEN DiscountPercent > 0 THEN 1 ELSE 0 END)`; `CountActiveAsync()` via `SqlQueryRaw<int>` with a column alias `Value`; `CallStoredProcedureAsync(int categoryId, decimal minPrice)` via `FromSqlRaw("EXEC dbo.GetActiveProductsByCategory @CategoryId = {0}, @MinPrice = {1}", ...)`; `DeactivateExpiredAsync(DateTime cutoffUtc)` via `ExecuteSqlInterpolatedAsync($"EXEC dbo.DeactivateExpired @Cutoff = {cutoffUtc}")`. Each method must carry an XML comment with the reason for raw SQL.

5. Write integration tests. In `tests/.../ProductCatalogRepositoryTests.cs` use `DbContextOptionsBuilder` with `UseInMemoryDatabase` only for model smoke tests, and for raw-SQL tests spin up a real SQL Server LocalDB with the connection string `Server=(localdb)\MSSQLLocalDB;Database=GearShopTests;Trusted_Connection=True;TrustServerCertificate=True`, call `EnsureSeedAsync`, clean up with `TRUNCATE TABLE Products`, insert fixtures and verify that `GetActiveByCategoryRawAsync(2, 100m)` returns exactly three expected rows, `SearchByNameInterpolatedAsync("bolt")` finds `M8 Bolt` and `M8 Bolted Flange`, `GetDashboardAsync(2)` yields `AveragePrice` ≈ 150 and `MedianPrice` ≈ 140, `CountActiveAsync()` equals the number of active products, `CallStoredProcedureAsync` matches `GetActiveByCategoryRawAsync` in row composition, and `DeactivateExpiredAsync` returns the number of deactivated rows and flips them to `IsActive = 0`. After each test run teardown via `EnsureDeletedAsync`.

6. Run and capture the output. Execute `dotnet test --logger "console;verbosity=normal"` — all tests must be green. Then `dotnet run --project src/GearShop.Catalog` — the program prints one line per repository method with the row count and one sample value, proving that all six paths work against a real database.

#### Requirements

The solution must compile under .NET 8 / C# 12 with no warnings and use modern language features: top-level statements in `Program.cs`, collection expressions for fixture initialization (you may group `new Product { ... }` instances into an array via `[ p1, p2, p3 ]`), list patterns where appropriate, and raw-string literals `"""..."""` for multi-line SQL so that quotes inside `EXEC` and `PERCENTILE_CONT` do not need escaping. All six repository methods must be asynchronous and accept a `CancellationToken` — even if the homework does not thread it into every call, the signature must be ready. No `string.Concat` and no interpolation that lands in `FromSqlRaw` as a literal: only `FromSqlInterpolated` for interpolated strings and only `{0}` placeholders for `FromSqlRaw`. Table and column names are static — dynamic schema driven by user input is not required and will be treated as a design error.

Every public repository method carries an XML `<summary>` comment with at least three fields: the reason for raw SQL (performance / DBMS construct / stored procedure / bulk operation), the expected plan or measured gain, and a reminder that updating entities returned by a raw query requires a context without `AsNoTracking`. Reads that do not lead to `SaveChanges` must use `AsNoTracking()` — this is a direct recommendation from the lesson. Stored procedures are invoked strictly through `EXEC dbo.Name @Param = {0}` — named parameters protect against argument-order mistakes. Tests are isolated: each test creates its own database with a unique suffix or uses `TRUNCATE` in `Setup` so that parallel xUnit execution does not race. Using `Database.SqlQuery<T>` to materialize the `Product` entity is forbidden — entities go only through `FromSql*`, DTOs and scalars only through `SqlQueryRaw<T>`/`SqlQuery<T>`.

#### Pitfalls

The main trap of `FromSqlRaw` and `FromSqlInterpolated` is the mandatory return of **all** entity columns in the same order and with the same names. If your `SELECT` forgets `IsActive`, EF Core throws a materialization exception before you ever see data. That is why, for the `GetActiveProductsByCategory` procedure, make sure the inner `SELECT` lists exactly `Id, Name, Price, IsActive, CategoryId, ExpiresAtUtc, DiscountPercent`. Column aliases must also match properties; `SELECT p.Id AS ProductId` breaks materialization into `Product.Id`. For DTOs it is simpler: `SqlQueryRaw<CategoryDashboardDto>` maps by column names, so aliases like `AVG(Price) AS AveragePrice` are mandatory — otherwise you get zeros or an error.

The second trap is LINQ composition over raw SQL. The lesson warns explicitly: after `FromSqlRaw` you may call `Where`, `OrderBy`, but EF Core wraps your SQL in a subquery, and this only works when the base SQL starts with `SELECT`. If your raw SQL contains `GROUP BY` or `INTO`, composition breaks — move the aggregation into `SqlQueryRaw<T>` with a DTO. The third trap is `AsNoTracking`. By default raw reads are tracked like regular LINQ; if you later call `SaveChanges` with overlapping `Id`s, EF will try to update rows you never intended to change. The fourth trap is that `ExecuteSqlInterpolated` returns the number of affected rows, not entities; do not try to materialize a result set, there is none. The fifth trap is that a scalar through `SqlQueryRaw<int>` returns a sequence, not a single value: you need `ToListAsync` and then `[0]`, or `.FirstAsync()` — and the alias `AS Value` is mandatory because EF Core 8 expects the name `Value` for scalar projections during default reflection.

The sixth and most dangerous trap is SQL injection. Any concatenation of the form `"... WHERE Name = '" + input + "'"` is a direct path to data leakage; even if today `input` comes from a trusted source, tomorrow it will be wired to a form. The lesson requires: data travels only as parameters, table and column names only through a white-list of constants. The homework has no dynamic names, so the temptation is smaller, but in code review look for any interpolation that lands in `FromSqlRaw` without curly-brace placeholders — that is an automatic red flag.

#### Acceptance criteria

- [ ] `dotnet build` succeeds with no warnings under .NET 8 / C# 12.
- [ ] `Program.cs` uses top-level statements.
- [ ] Multi-line SQL uses raw-string literals `"""..."""`.
- [ ] All six `ProductCatalogRepository` methods are asynchronous with `CancellationToken`.
- [ ] `GetActiveByCategoryRawAsync` uses `FromSqlRaw` with `{0}`, `{1}` and `AsNoTracking`.
- [ ] `SearchByNameInterpolatedAsync` uses `FromSqlInterpolated`, `LIKE` with a wildcard.
- [ ] `GetDashboardAsync` uses `SqlQueryRaw<CategoryDashboardDto>` with `PERCENTILE_CONT`.
- [ ] `CountActiveAsync` uses `SqlQueryRaw<int>` with a `Value` alias.
- [ ] `CallStoredProcedureAsync` invokes `EXEC dbo.GetActiveProductsByCategory @CategoryId = {0}, @MinPrice = {1}`.
- [ ] `DeactivateExpiredAsync` invokes `ExecuteSqlInterpolatedAsync` and returns the row count.
- [ ] Each method has an XML comment with the reason for raw SQL.
- [ ] Reads without updates use `AsNoTracking()`.
- [ ] Integration tests pass on a real SQL Server (LocalDB or container).
- [ ] Tests verify row composition and values, not only the absence of exceptions.
- [ ] There is not a single string concatenation in SQL anywhere in the code.

#### Hints (no direct answer)

- If materialization throws "cannot materialize", check that SQL returns all entity columns and in the same order as the properties.
- For `SqlQueryRaw<int>` do not expect a single number — it is a sequence; take the first element.
- `PERCENTILE_CONT` requires `WITHIN GROUP (ORDER BY ...)` and returns `numeric`; cast to `decimal` in the DTO, not in SQL.
- Raw-string literals `"""..."""` let you write `EXEC dbo.DeactivateExpired @Cutoff = {0}` without escaping quotes.
- Remember: `FromSqlInterpolated` turns every `{value}` into a `DbParameter` automatically — that is your injection defense.
- If `Include` after raw SQL throws, make sure the base query returns the full entity, not a projection.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 / EF Core 8 — GearShop.Catalog
// Reference solution for homework M12-L08

using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;

namespace GearShop.Catalog.Data;

public sealed class ProductCatalogRepository(ShopDbContext db)
{
    // Reason: a DBA tuned the plan for a covering index; LINQ reorders
    // predicates and the plan degrades.
    public async Task<List<Product>> GetActiveByCategoryRawAsync(
        int categoryId, decimal minPrice, CancellationToken ct = default)
    {
        // Placeholders become DbParameter, not string concatenation.
        return await db.Products
            .FromSqlRaw("""
                SELECT Id, Name, Price, IsActive, CategoryId, ExpiresAtUtc, DiscountPercent
                FROM dbo.Products
                WHERE CategoryId = {0} AND Price >= {1} AND IsActive = 1
                """, categoryId, minPrice)
            .AsNoTracking()           // read-only
            .OrderBy(p => p.Name)     // composition over subquery
            .ToListAsync(ct);
    }

    // Reason: enforce compile-time parameterization so no junior developer
    // ever pastes userInput into the SQL string.
    public async Task<List<Product>> SearchByNameInterpolatedAsync(
        string nameFragment, CancellationToken ct = default)
    {
        // FromSqlInterpolated: each {…} becomes a DbParameter automatically.
        var pattern = $"%{nameFragment}%";
        return await db.Products
            .FromSqlInterpolated($"""
                SELECT Id, Name, Price, IsActive, CategoryId, ExpiresAtUtc, DiscountPercent
                FROM dbo.Products
                WHERE Name LIKE {pattern} AND IsActive = 1
                """)
            .AsNoTracking()
            .ToListAsync(ct);
    }

    // Reason: PERCENTILE_CONT window function not expressible in LINQ.
    public async Task<List<CategoryDashboardDto>> GetDashboardAsync(
        int categoryId, CancellationToken ct = default)
    {
        // SqlQueryRaw<T>: materialize into DTO by column names.
        return await db.Database
            .SqlQueryRaw<CategoryDashboardDto>($"""
                SELECT
                    {categoryId} AS CategoryId,
                    c.Name AS CategoryName,
                    AVG(p.Price) AS AveragePrice,
                    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY p.Price)
                        OVER (PARTITION BY p.CategoryId) AS MedianPrice,
                    COUNT(*) AS ActiveCount,
                    CAST(SUM(CASE WHEN p.DiscountPercent > 0 THEN 1 ELSE 0 END)
                         AS decimal(18,4)) / COUNT(*) AS DiscountShare
                FROM dbo.Products p
                JOIN dbo.Categories c ON c.Id = p.CategoryId
                WHERE p.CategoryId = {categoryId} AND p.IsActive = 1
                GROUP BY c.Name, p.CategoryId
                """)
            .ToListAsync(ct);
    }

    // Reason: scalar COUNT is cleaner as SqlQueryRaw<int> than FromSqlRaw
    // with an entity just for one number.
    public async Task<int> CountActiveAsync(CancellationToken ct = default)
    {
        var rows = await db.Database
            .SqlQueryRaw<int>("SELECT COUNT(*) AS Value FROM dbo.Products WHERE IsActive = 1")
            .ToListAsync(ct);
        return rows.Count == 0 ? 0 : rows[0];
    }

    // Reason: integration with a DBA-owned stored procedure.
    public async Task<List<Product>> CallStoredProcedureAsync(
        int categoryId, decimal minPrice, CancellationToken ct = default)
    {
        // Named parameters protect against argument-order mistakes.
        return await db.Products
            .FromSqlRaw(
                "EXEC dbo.GetActiveProductsByCategory @CategoryId = {0}, @MinPrice = {1}",
                categoryId, minPrice)
            .AsNoTracking()
            .ToListAsync(ct);
    }

    // Reason: server-side bulk operation with OUTPUT INTO a journal table
    // inside one server transaction — SaveChanges cannot do this.
    public async Task<int> DeactivateExpiredAsync(DateTime cutoffUtc, CancellationToken ct = default)
        => await db.Database.ExecuteSqlInterpolatedAsync(
            $"EXEC dbo.DeactivateExpired @Cutoff = {cutoffUtc}", ct);
}
```

Line-by-line walk-through. The `ProductCatalogRepository(ShopDbContext db)` constructor uses the C# 12 primary constructor — the `db` field is captured automatically, with no explicit field declaration. `GetActiveByCategoryRawAsync` is the canonical example from the lesson: `FromSqlRaw` with placeholders `{0}`, `{1}`, then `AsNoTracking` (a read with no update, saving memory and CPU) and `OrderBy` as composition — EF Core wraps our `SELECT` in a subquery and appends `ORDER BY` on the outside. The raw-string literal `"""..."""` allows multi-line SQL without `\r\n` concatenation and without escaping quotes — this is a C# 11+ idiom that the lesson implicitly assumes through the "C# 12+" caption. `SearchByNameInterpolatedAsync` deliberately chooses `FromSqlInterpolated` so that parameterization is visible to the compiler: the interpolated string is passed as a `FormattableString`, and EF Core turns each `{pattern}` into a `DbParameter` — this is exactly the "automatic injection defense" the lesson describes. `GetDashboardAsync` applies `SqlQueryRaw<CategoryDashboardDto>` — the EF Core 7+ API ideal for projections: the column names `AveragePrice`, `MedianPrice` match the record properties, so materialization succeeds silently. `PERCENTILE_CONT` is a window function unavailable in LINQ, which maps directly to the "when to choose raw SQL" section of the lesson.

`CountActiveAsync` illustrates the scalar subtlety: `SqlQueryRaw<int>` returns a sequence, not a single value; we materialize into `List<int>` and take `[0]`, and the alias `AS Value` is mandatory because EF Core 8, when reflecting into `int`, looks for a column named `Value`. `CallStoredProcedureAsync` invokes the stored procedure through `EXEC dbo.GetActiveProductsByCategory @CategoryId = {0}, @MinPrice = {1}` — named parameters protect against argument-order mistakes and placeholders guarantee parameterization; the result materializes into the `Product` entity, so the procedure internally must `SELECT` exactly the same columns in the same order. `DeactivateExpiredAsync` uses `ExecuteSqlInterpolatedAsync` — for DML/procedures without a row set; the affected row count is returned, which we propagate upward. The whole six-point chain shows that the API choice is not arbitrary: `FromSql*` for entities, `SqlQueryRaw<T>` for DTOs and scalars, `ExecuteSql*` for DML, and every choice is justified by a reason drawn from the lesson.

#### Going deeper (bonus)

1. Add a `GetTopExpensivePerCategoryAsync` method that returns the three most expensive active products per category via `ROW_NUMBER() OVER (PARTITION BY CategoryId ORDER BY Price DESC)` and a projection into a new `TopProductDto`. Compare the plan with the LINQ variant using `GroupBy` and `.Take(3)`.
2. Implement a dynamic filter over an arbitrary set of categories: the user passes `int[] categoryIds`, and you build `WHERE CategoryId IN (...)` safely — through a `SqlParameter` with a table-valued type or through `json`/`OPENJSON`. Show that `string.Join` concatenation is forbidden and justify the alternative.
3. Add a repository-side cache via `IMemoryCache` with a key dependent on the parameters, and invalidate it in `DeactivateExpiredAsync`. Show that raw reads can be cached too, but only read-only entities.
4. Cover the repository with an injection test: pass `nameFragment = "'; DROP TABLE Products;--"` and confirm the table is intact and the search returns an empty list.

---

#### Чек-лист сдачи / Submission checklist

- [ ] Решение собирается под .NET 8 / C# 12 без предупреждений.
- [ ] Все шесть методов репозитория реализованы и асинхронны.
- [ ] Использованы `FromSqlRaw`, `FromSqlInterpolated`, `SqlQueryRaw<T>`, `ExecuteSqlInterpolatedAsync`.
- [ ] Хранимые процедуры вызываются через `EXEC dbo.Name @Param = {0}`.
- [ ] Каждый метод снабжён XML-комментарием с причиной raw SQL.
- [ ] Чтения без обновлений используют `AsNoTracking()`.
- [ ] Интеграционные тесты проходят на реальном SQL Server.
- [ ] В коде нет конкатенации строк в SQL.
- [ ] Solution builds under .NET 8 / C# 12 with no warnings.
- [ ] All six repository methods are implemented and asynchronous.
- [ ] `FromSqlRaw`, `FromSqlInterpolated`, `SqlQueryRaw<T>`, `ExecuteSqlInterpolatedAsync` are all used.
- [ ] Stored procedures are invoked through `EXEC dbo.Name @Param = {0}`.
- [ ] Each method has an XML comment with the reason for raw SQL.
- [ ] Reads without updates use `AsNoTracking()`.
- [ ] Integration tests pass on a real SQL Server.
- [ ] There is no string concatenation in SQL anywhere in the code.

#### Ресурсы / Resources

- [Microsoft Learn — Raw SQL queries in EF Core](https://learn.microsoft.com/ef/core/querying/raw-sql)
- [Microsoft Learn — `FromSqlRaw` and `FromSqlInterpolated`](https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.relationalqueryableextensions.fromsqlraw)
- [Microsoft Learn — `SqlQueryRaw<T>` (EF Core 7+)](https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.databaserelationalextensions.sqlqueryraw)
- [Microsoft Learn — `ExecuteSqlInterpolatedAsync`](https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.relationaldatabasefacadeextensions.executesqlinterpolatedasync)
- [Microsoft SQL Docs — `PERCENTILE_CONT`](https://learn.microsoft.com/sql/t-sql/functions/percentile-cont-transact-sql)
- [OWASP — SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
