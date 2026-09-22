[⬆ К модулю M12](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M12-L08: Raw SQL, FromSqlRaw, хранимые процедуры / Raw SQL, FromSqlRaw, stored procedures

**Модуль / Module:** M12
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Entity Framework Core — мощный LINQ-провайдер, но иногда LINQ не хватает. Существуют ситуации, когда сырой SQL предпочтительнее: сложные аналитические запросы с `CROSS APPLY`, оптимизированные хранимые процедуры, `GROUP BY CUBE`, полнотекстовый поиск, массовые операции `BULK INSERT`, или когда DBA передал вам готовый план запроса, выверенный до миллисекунд. EF Core даёт несколько механизмов для таких случаев, и важно понимать, когда и какой применять.

**`FromSqlRaw` и `FromSqlInterpolated`** — два способа выполнить SQL-запрос, результат которого EF Core материализует в сущности. Разница — в работе с параметрами и защите от SQL-инъекций. `FromSqlRaw` принимает строку и параметры отдельно: вы сами формируете плейсхолдеры `{0}`, `{1}`. `FromSqlInterpolated` принимает интерполированную строку `$"...{value}..."` — компилятор и EF Core автоматически превращают каждое внедрение в параметризованный аргумент. Это удобнее и безопаснее по умолчанию. Оба метода требуют, чтобы SQL возвращал **все столбцы сущности** в том же порядке и с теми же именами — иначе материализация упадёт. Псевдонимы столбцов должны совпадать со свойствами сущности.

Важное ограничение: после `FromSqlRaw` можно вызывать LINQ-операторы (`Where`, `OrderBy`, `Select`), но они добавляются к запросу только при использовании композиции — EF Core оборачивает ваш SQL в подзапрос. Это работает, если SQL начинается с `SELECT`. Для `JOIN` через `Include` нужно, чтобы базовый запрос возвращал сущности с навигационными свойствами. Отслеживание изменений (change tracking) по умолчанию включено, как и у обычных LINQ-запросов; `AsNoTracking` отключает его.

**`SqlQueryRaw` / `SqlQuery`** (появились в EF Core 7.0) — отдельный API для запросов, возвращающих произвольные типы, а не сущности. Идеально для проекций в DTO или скалярные значения: `context.Database.SqlQueryRaw<int>($"SELECT COUNT(*) FROM Products")`. Результат материализуется по именам столбцов в тип `T`. Для исполнения без результата — `ExecuteSqlRaw` / `ExecuteSqlInterpolated` (например, `UPDATE` или вызов хранимой процедуры).

**Хранимые процедуры** вызываются тем же API: `FromSqlRaw("EXEC dbo.GetActiveProducts @tenantId = {0}", tenantId)`. Если процедура возвращает результат, совпадающий со сущностью — используйте `FromSqlRaw`. Если возвращает произвольный набор — `SqlQueryRaw<T>`. Параметры передаются строго через плейсхолдеры; никогда не конкатенируйте строки — это прямой путь к инъекции. Сравните: `"... WHERE Name = '" + userInput + "'"` — катастрофа; `"... WHERE Name = {0}", userInput` — безопасно, EF Core обернёт значение в `DbParameter`.

**Защита от SQL-инъекций** строится на одном правиле: любые данные от пользователя проходят только как параметры, никогда — как часть SQL-строки. `FromSqlInterpolated` делает это автоматически. `FromSqlRaw` требует явной дисциплины. Имена таблиц и столбцов тоже нельзя параметризовать — если их нужно динамизировать, валидируйте по белому списку констант.

**Когда выбирать raw SQL:** (1) производительность критична и LINQ генерирует неоптимальный план; (2) используются специфичные конструкции СУБД (оконные функции, hints, `FOR XML`); (3) интеграция с существующими хранимыми процедурами; (4) массовые операции, где `SaveChanges` слишком медленный. Во всех остальных случаях предпочитайте LINQ — он типобезопасен, тестируется и портируется между СУБД.

Аналогия: LINQ — это ресторан с фиксированным меню, удобный и предсказуемый. Raw SQL — это кухня шеф-повара: больше свободы и скорости, но вы сами отвечаете за результат, отравление инъекцией и согласованность с моделью.

#### Theory (EN)

Entity Framework Core is a powerful LINQ provider, but sometimes LINQ is not enough. There are situations where raw SQL is preferable: complex analytical queries with `CROSS APPLY`, tuned stored procedures, `GROUP BY CUBE`, full-text search, bulk operations like `BULK INSERT`, or when a DBA hands you a hand-tuned execution plan measured in milliseconds. EF Core offers several mechanisms for these cases, and it is important to understand when and how to apply each one.

**`FromSqlRaw` and `FromSqlInterpolated`** are two ways to execute a SQL query whose result EF Core materializes into entities. The difference lies in parameter handling and SQL injection protection. `FromSqlRaw` takes a string and parameters separately — you build placeholders `{0}`, `{1}` yourself. `FromSqlInterpolated` takes an interpolated string `$"...{value}..."` — the compiler and EF Core automatically turn each interpolation into a parameterized argument. This is more convenient and safer by default. Both methods require that the SQL returns **all entity columns** in the same order and with the same names — otherwise materialization will throw. Column aliases must match the entity properties.

An important limitation: after `FromSqlRaw` you can call LINQ operators (`Where`, `OrderBy`, `Select`), but they are appended to the query only via composition — EF Core wraps your SQL in a subquery. This works when the SQL starts with `SELECT`. For `JOIN` via `Include`, the base query must return entities with navigation properties. Change tracking is on by default, just like with regular LINQ queries; `AsNoTracking` disables it.

**`SqlQueryRaw` / `SqlQuery`** (introduced in EF Core 7.0) is a separate API for queries returning arbitrary types rather than entities. It is ideal for projections into DTOs or scalar values: `context.Database.SqlQueryRaw<int>($"SELECT COUNT(*) FROM Products")`. The result is materialized by column names into type `T`. For execution without a result set — `ExecuteSqlRaw` / `ExecuteSqlInterpolated` (for example an `UPDATE` or a stored procedure call).

**Stored procedures** are invoked through the same API: `FromSqlRaw("EXEC dbo.GetActiveProducts @tenantId = {0}", tenantId)`. If the procedure returns a result matching an entity, use `FromSqlRaw`. If it returns an arbitrary shape, use `SqlQueryRaw<T>`. Parameters must always go through placeholders — never concatenate strings, that is a direct path to injection. Compare: `"... WHERE Name = '" + userInput + "'"` is a disaster; `"... WHERE Name = {0}", userInput` is safe, EF Core wraps the value in a `DbParameter`.

**SQL injection protection** rests on a single rule: any user-supplied data must travel as a parameter, never as part of the SQL string. `FromSqlInterpolated` does this automatically. `FromSqlRaw` demands explicit discipline. Table and column names cannot be parameterized either — if they must be dynamic, validate them against a white-list of constants.

**When to choose raw SQL:** (1) performance is critical and LINQ generates a suboptimal plan; (2) you need DBMS-specific constructs (window functions, hints, `FOR XML`); (3) integration with existing stored procedures; (4) bulk operations where `SaveChanges` is too slow. In every other case prefer LINQ — it is type-safe, testable, and portable across providers.

Analogy: LINQ is a restaurant with a fixed menu — convenient and predictable. Raw SQL is a chef's kitchen: more freedom and speed, but you alone are responsible for the result, injection poisoning, and consistency with the model.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8 / EF Core 8
// Демонстрация FromSqlRaw, FromSqlInterpolated, SqlQueryRaw и хранимых процедур
// Demo of FromSqlRaw, FromSqlInterpolated, SqlQueryRaw and stored procedures

using Microsoft.EntityFrameworkCore;

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public bool IsActive { get; set; }
    public int CategoryId { get; set; }
}

public record ProductSummaryDto(int Id, string Name, decimal Price);

public class ShopDbContext : DbContext
{
    public DbSet<Product> Products => Set<Product>();

    protected override void OnConfiguring(DbContextOptionsBuilder b)
        => b.UseSqlServer("Server=.;Database=Shop;Trusted_Connection=True;TrustServerCertificate=True");
}

public static class RawSqlExamples
{
    // 1) FromSqlRaw с явными параметрами — безопасно от инъекций
    //    FromSqlRaw with explicit parameters — safe from injection
    public static async Task<List<Product>> GetActiveByCategoryRawAsync(
        ShopDbContext db, int categoryId, decimal minPrice)
    {
        // Плейсхолдеры {0}, {1} превращаются в DbParameter — НЕ в конкатенацию строк
        // Placeholders {0}, {1} become DbParameter — NOT string concatenation
        return await db.Products
            .FromSqlRaw(
                "SELECT * FROM Products WHERE CategoryId = {0} AND Price >= {1} AND IsActive = 1",
                categoryId, minPrice)
            .AsNoTracking()
            .OrderBy(p => p.Name)
            .ToListAsync();
    }

    // 2) FromSqlInterpolated — компилятор сам строит параметры из интерполяции
    //    FromSqlInterpolated — compiler builds parameters from interpolation
    public static async Task<List<Product>> SearchByNameAsync(
        ShopDbContext db, string nameFragment)
    {
        // Каждое {…} значение становится параметром автоматически
        // Each {…} value becomes a parameter automatically
        return await db.Products
            .FromSqlInterpolated(
                $"SELECT * FROM Products WHERE Name LIKE {'%' + nameFragment + '%'} AND IsActive = 1")
            .AsNoTracking()
            .ToListAsync();
    }

    // 3) SqlQueryRaw<T> — произвольная проекция в DTO (EF Core 7+)
    //    SqlQueryRaw<T> — arbitrary projection into DTO (EF Core 7+)
    public static async Task<List<ProductSummaryDto>> GetSummaryAsync(ShopDbContext db)
    {
        // Имена столбцов в SELECT должны совпадать со свойствами DTO
        // Column names in SELECT must match DTO properties
        return await db.Database
            .SqlQueryRaw<ProductSummaryDto>(
                "SELECT Id, Name, Price FROM Products WHERE IsActive = 1 ORDER BY Price DESC")
            .ToListAsync();
    }

    // 4) Скалярный запрос через SqlQueryRaw<T>
    //    Scalar query via SqlQueryRaw<T>
    public static async Task<int> CountActiveAsync(ShopDbContext db)
    {
        var rows = await db.Database
            .SqlQueryRaw<int>($"SELECT COUNT(*) AS Value FROM Products WHERE IsActive = 1")
            .ToListAsync();
        return rows.Count == 0 ? 0 : rows[0];
    }

    // 5) Хранимая процедура с выходным набором, совпадающим со сущностью
    //    Stored procedure with a result set matching the entity
    public static async Task<List<Product>> CallStoredProcedureAsync(
        ShopDbContext db, int tenantId)
    {
        return await db.Products
            .FromSqlRaw("EXEC dbo.GetActiveProducts @TenantId = {0}", tenantId)
            .AsNoTracking()
            .ToListAsync();
    }

    // 6) Хранимая процедура без возврата набора — ExecuteSqlInterpolated
    //    Stored procedure without result set — ExecuteSqlInterpolated
    public static async Task<int> DeactivateExpiredAsync(ShopDbContext db, DateTime cutoff)
    {
        // Возвращает число затронутых строк / Returns affected row count
        return await db.Database
            .ExecuteSqlInterpolatedAsync($"EXEC dbo.DeactivateExpired @Cutoff = {cutoff}");
    }
}
```

#### Best Practices
- Предпочитай `FromSqlInterpolated` — он автоматически параметризует значения и снижает риск инъекции.
- Используй `AsNoTracking()` для raw-чтений, если не планируешь обновлять сущности — экономит память и CPU.
- Оборачивай raw-вызовы в репозиторий и покрывай интеграционными тестами на реальной СУБД.
- Документируй причину raw SQL в комментарии: план, производитель, измеренная выгода.

- Prefer `FromSqlInterpolated` — it auto-parameterizes values and reduces injection risk.
- Apply `AsNoTracking()` for raw reads you do not intend to update — it saves memory and CPU.
- Wrap raw calls in a repository and cover them with integration tests on a real DBMS.
- Document the reason for raw SQL in a comment: provider, plan, measured gain.

#### Частые ошибки / Common Mistakes
- [Конкатенация строк в SQL `"... WHERE Name = '" + input + "'"`] → [Используй `FromSqlInterpolated` или плейсхолдеры `{0}` в `FromSqlRaw`].
- [SQL не возвращает все столбцы сущности] → [Перечисли все столбцы или используй `SqlQueryRaw<T>` с DTO].
- [LINQ после `FromSqlRaw` с `GROUP BY`] → [Композиция падает; заверни логику в хранимую процедуру или DTO-запрос].
- [Вызов `Include` после raw SQL без `SELECT *`] → [Возвращай полный набор сущности и не злоупотребляй композицией].

- [String concatenation in SQL `"... WHERE Name = '" + input + "'"`] → [Use `FromSqlInterpolated` or `{0}` placeholders in `FromSqlRaw`].
- [SQL does not return all entity columns] → [List all columns or use `SqlQueryRaw<T>` with a DTO].
- [LINQ after `FromSqlRaw` containing `GROUP BY`] → [Composition breaks; move logic into a stored procedure or a DTO query].
- [Calling `Include` after raw SQL without `SELECT *`] → [Return the full entity set and avoid overusing composition].

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Все пользовательские данные передаются как параметры, а не конкатенацией.
- [ ] Для raw-чтений без обновлений включён `AsNoTracking()`.
- [ ] Имена столбцов в SQL совпадают со свойствами сущности/DTO.
- [ ] Raw-вызовы изолированы в репозитории и покрыты тестами.
- [ ] В комментарии указана причина выбора raw SQL вместо LINQ.
- [ ] Хранимые процедуры вызываются через `EXEC … @Param = {0}`.
- [ ] Скалярные/произвольные результаты читаются через `SqlQueryRaw<T>`.

- [ ] All user data is passed as parameters, not via concatenation.
- [ ] `AsNoTracking()` is enabled for raw reads that are not updated.
- [ ] SQL column names match entity/DTO properties.
- [ ] Raw calls are isolated in a repository and covered by tests.
- [ ] The comment states why raw SQL was chosen over LINQ.
- [ ] Stored procedures are invoked via `EXEC … @Param = {0}`.
- [ ] Scalar/arbitrary results are read via `SqlQueryRaw<T>`.

#### Ресурсы / Resources
- [Microsoft Learn — Raw SQL queries](https://learn.microsoft.com/ef/core/querying/raw-sql)

---

[⬆ К модулю M12](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
