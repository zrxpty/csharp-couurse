[⬆ К модулю M12](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M12-L10: Производительность: индексы, AsSplitQuery, batched updates / Performance: indexes, AsSplitQuery, batched updates

**Модуль / Module:** M12
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Производительность EF Core — это не магия, а дисциплина. Представьте базу данных как огромную библиотеку, а индекс — как каталог карточек. Без каталога библиотекарь обходит каждый шкаф (Seq Scan), с каталогом он идёт прямо к нужной полке (Index Seek). Первое правило: индексируйте колонки, по которым вы фильтруете, сортируете и соединяете.

`HasIndex` описывает индекс в Fluent API. Составной индекс (composite) эффективен, когда порядок колонок совпадает с порядком фильтрации: `HasIndex(c => new { c.TeacherId, c.Title })` поможет, если вы ищете по преподавателю и, опционально, по названию. EF Core 7+ поддерживает индексы с включёнными колонками (`.IncludeProperties`) и частичные индексы (`.HasFilter`) — последние экономят место, индексируя только активные строки. Не создавайте лишних индексов: каждый замедляет запись, ведь движку нужно поддерживать его при каждом INSERT/UPDATE/DELETE.

Вторая тема — `AsSplitQuery`. По умолчанию EF Core шлёт один большой запрос с JOIN'ами и материализует «декартову» корзину (cartesian explosion): для N преподавателей с M курсами вы получаете N×M строк, и контекст дедуплицирует их в памяти. Для связей N:N с большими коллекциями это критично — память и CPU уходят на разбор дублей. `AsSplitQuery()` разбивает запрос на несколько: один для корня, отдельные для каждой коллекции. Минус — больше сетевых заходов (round-trips); для мелких выборок дефолтный `AsSingleQuery` дешевле. Всегда измеряйте оба варианта.

Третья тема — bulk-операции `ExecuteUpdate` и `ExecuteDelete` (EF Core 7+). Классический `SaveChanges` для 10 000 строк тянет их в память, генерирует UPDATE на каждую и устраивает round-trip на каждую. `ExecuteUpdate` сразу транслирует SQL `UPDATE ... WHERE`, ничего не загружая. Идеально для массовых изменений статуса, архивации, удаления старых записей. Помните: эти операции обходят Change Tracker, поэтому не вызывают события `SavingChanges`/`SavedChanges` и не обновляют вычисляемые на стороне C# колонки — например, аудиторские поля через `SaveChangesInterceptor` придётся выставлять в SQL.

Четвёртая — проекции. Загружать целую сущность ради двух полей расточительно: EF тянет все колонки, тратит память и время на материализацию. `.Select(x => new Dto { ... })` формирует SQL только с нужными столбцами. Это и быстрее, и безопаснее — не даёт случайно модифицировать то, что вы не планировали.

Пятая — compiled queries. На горячих путях, где запрос повторяется тысячи раз, компиляция LINQ→SQL каждый раз съедает миллисекунды. `EF.CompileAsyncQuery` кэширует скомпилированное дерево. Выигрыш заметен на простых, частых запросах; для редких или сложных компиляция не окупается. Профилируйте сначала, оптимизируйте потом.

Аналогии: индексы — оглавление книги, `AsSplitQuery` — доставка посылок отдельными машинами вместо одного гигантского фургона, `ExecuteUpdate` — приказ армии «всем сменить форму», а не сбор по одному солдату, проекции — заказ конкретных блюд вместо шведского стола, compiled queries — заранее заготовленные шаблоны писем.

Главное правило: измеряйте. Без цифр оптимизация — азартная игра. Используйте `ToQueryString()`, логирование `DbLoggerCategory.Database.Command`, MiniProfiler или EF Core logging. Ускорение «в теории» часто оказывается медленнее на практике из-за распределения данных и плана запроса.

#### Theory (EN)

EF Core performance is discipline, not magic. Picture your database as a vast library and an index as the card catalogue. Without a catalogue, the librarian walks every shelf (a sequential scan); with one, she goes straight to the right shelf (an index seek). The first rule is therefore: index the columns you filter, sort, and join on.

`HasIndex` declares an index through the Fluent API. A composite index is effective when its column order matches the order of your filtering: `HasIndex(c => new { c.TeacherId, c.Title })` helps if you query by teacher and, optionally, by title. EF Core 7+ supports indexes with included columns (`.IncludeProperties`) and filtered indexes (`.HasFilter`) — the latter save space by indexing only active rows. Do not over-index: every index slows writes, because the engine must maintain it on every INSERT, UPDATE, and DELETE.

The second topic is `AsSplitQuery`. By default EF Core issues one large query with JOINs and materializes a "cartesian" result: for N teachers with M courses you get N×M rows, and the context de-duplicates them in memory. For many-to-many relations with sizeable collections this is ruinous — memory and CPU vanish into parsing duplicates. `AsSplitQuery()` breaks the query into several: one for the root, a separate query per collection. The trade-off is more network round-trips; for small result sets the default `AsSingleQuery` is cheaper. Always benchmark both before committing.

The third topic is the `ExecuteUpdate` and `ExecuteDelete` bulk operations (EF Core 7+). Classic `SaveChanges` for ten thousand rows loads them into memory, generates an UPDATE per row, and makes a round-trip per row. `ExecuteUpdate` instead translates directly to SQL `UPDATE ... WHERE` without loading anything. It is ideal for mass status changes, archival, and deleting old records. Remember that these operations bypass the Change Tracker, so they do not trigger `SavingChanges`/`SavedChanges` events and do not update columns computed on the C# side — for example, audit fields populated by a `SaveChangesInterceptor` must be set inside the SQL itself.

The fourth topic is projections. Loading an entire entity to read two fields is wasteful: EF fetches every column and spends time and memory materializing it. `.Select(x => new Dto { ... })` produces SQL with only the needed columns. This is both faster and safer — you cannot accidentally modify fields you never intended to touch.

The fifth topic is compiled queries. On hot paths where the same query runs thousands of times, the LINQ-to-SQL compilation eats milliseconds each time. `EF.CompileAsyncQuery` caches the compiled tree. The gain is visible on simple, frequent queries; for rare or complex ones the compilation overhead is not worth it. Profile first, optimize second.

Analogies: indexes are a book's table of contents; `AsSplitQuery` is shipping parcels in separate vans instead of one giant truck; `ExecuteUpdate` is an order to an army "everyone change uniforms" rather than drafting soldiers one by one; projections are à la carte ordering instead of a buffet; compiled queries are pre-stamped letter templates.

The cardinal rule of optimization is: measure. Without numbers, optimization is gambling. Use `ToQueryString()`, enable `DbLoggerCategory.Database.Command` logging, attach MiniProfiler or EF Core logging. A theoretical speed-up often turns out slower in practice because of data distribution and the query plan. Never optimize by guess — only by numbers.

#### Пример кода / Code Example

```csharp
// ============================================================
// M12-L10 Производительность EF Core / EF Core Performance
// C# 12 / .NET 8, EF Core 8
// ============================================================
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

// --- Модели / Models ---
public class Teacher
{
    public int Id { get; set; }
    public string FullName { get; set; } = "";
    public bool IsActive { get; set; }
    public List<Course> Courses { get; set; } = new();
}

public class Course
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public DateTime CreatedAt { get; set; }
    public int TeacherId { get; set; }
    public Teacher? Teacher { get; set; }
    public List<CourseTag> Tags { get; set; } = new();
}

public class CourseTag
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public int CourseId { get; set; }
    public Course Course { get; set; } = default!;
}

// --- Конфигурация индексов / Index configuration ---
public class TeacherConfig : IEntityTypeConfiguration<Teacher>
{
    public void Configure(EntityTypeBuilder<Teacher> b)
    {
        // Уникальный индекс по имени / Unique index on name
        b.HasIndex(t => t.FullName).IsUnique();

        // Частичный индекс только для активных / Filtered index, active only
        // SQL Server синтаксис фильтра / SQL Server filter syntax
        b.HasIndex(t => t.FullName)
         .HasFilter("[IsActive] = 1")
         .HasDatabaseName("IX_Teacher_Active_Name");
    }
}

public class CourseConfig : IEntityTypeConfiguration<Course>
{
    public void Configure(EntityTypeBuilder<Course> b)
    {
        // Составной индекс: сначала по преподавателю, потом по названию
        // Composite index: teacher first, then title (match filter order)
        b.HasIndex(c => new { c.TeacherId, c.Title })
         .HasDatabaseName("IX_Course_Teacher_Title");

        // Индекс с включённой колонкой (EF Core 7+) / Index with included column
        b.HasIndex(c => c.TeacherId)
         .IncludeProperties(c => new { c.Title, c.CreatedAt })
         .HasDatabaseName("IX_Course_Teacher_Include");
    }
}

// --- DbContext ---
public class AppDbContext : DbContext
{
    public DbSet<Teacher> Teachers => Set<Teacher>();
    public DbSet<Course> Courses => Set<Course>();
    public DbSet<CourseTag> CourseTags => Set<CourseTag>();

    protected override void OnConfiguring(DbContextOptionsBuilder o)
        => o.UseSqlServer("Server=.;Database=Demo;Trusted_Connection=True;TrustServerCertificate=True")
             .LogTo(Console.WriteLine,
                    new[] { DbLoggerCategory.Database.Command.Name },
                    LogLevel.Information);

    protected override void OnModelCreating(ModelBuilder mb)
        => mb.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);

    // --- Compiled query: кэшированное дерево LINQ→SQL / Cached tree ---
    // Выигрыш на горячих простых запросах / Wins on hot simple queries
    private static readonly Func<AppDbContext, int, CancellationToken, Task<Teacher?>> _activeTeacherById =
        EF.CompileAsyncQuery((AppDbContext db, int id, CancellationToken ct) =>
            db.Teachers.FirstOrDefault(t => t.Id == id && t.IsActive));

    public Task<Teacher?> GetActiveTeacherCompiled(int id, CancellationToken ct = default)
        => _activeTeacherById(this, ct, id);
}

// --- DTO-проекция / Projection DTO ---
public record CourseListItem(int Id, string Title, string Teacher, int TagCount);

// --- Демонстрация / Demo ---
public static class PerformanceDemo
{
    public static async Task RunAsync(AppDbContext db, CancellationToken ct = default)
    {
        // 1) AsSplitQuery: каждая коллекция — отдельный SQL, без декартова взрыва
        //    Each collection gets its own SQL, no cartesian explosion
        var teachers = await db.Teachers
            .Include(t => t.Courses)
                .ThenInclude(c => c.Tags)
            .Where(t => t.IsActive)
            .AsSplitQuery()          // или .AsSingleQuery() — измеряйте оба / benchmark both
            .ToListAsync(ct);

        // 2) Проекция: только нужные колонки / Projection: only needed columns
        var dto = await db.Courses
            .Where(c => c.TeacherId == 42)
            .Select(c => new CourseListItem(
                c.Id,
                c.Title,
                c.Teacher!.FullName,
                c.Tags.Count))
            .ToListAsync(ct);

        // 3) Bulk ExecuteUpdate: один SQL UPDATE, без загрузки в память
        //    Single SQL UPDATE, no in-memory load, bypasses Change Tracker
        await db.Courses
            .Where(c => !c.Teacher!.IsActive)
            .ExecuteUpdateAsync(s => s
                .SetProperty(c => c.Title, c => c.Title + " (archived)"), ct);

        // 4) Bulk ExecuteDelete: один SQL DELETE
        await db.CourseTags
            .Where(t => t.CourseId == 0) // пример условия / sample predicate
            .ExecuteDeleteAsync(ct);

        // 5) Compiled query для горячего пути / compiled query for hot path
        var t = await db.GetActiveTeacherCompiled(7, ct);

        // 6) Просмотр сгенерированного SQL до выполнения / Inspect SQL before run
        var sql = db.Teachers.Where(x => x.IsActive).ToQueryString();
        Console.WriteLine(sql);
    }
}
```

#### Best Practices
- Индексируйте колонки из WHERE, JOIN, ORDER BY — но не каждую колонку подряд; каждый индекс замедляет запись.
- Используйте `AsSplitQuery` для N:N с большими коллекциями и измеряйте оба режима (`AsSingleQuery`/`AsSplitQuery`) на реальных данных.
- Для массовых изменений статуса применяйте `ExecuteUpdate`/`ExecuteDelete` вместо цикла `SaveChanges`.
- Проектируйте чтение через `.Select`, чтобы не тянуть лишние колонки и не рисковать случайной модификацией.
- Кэшируйте горячие простые запросы через `EF.CompileAsyncQuery`; для редких сложных это не окупается.
- Включайте логирование SQL (`DbLoggerCategory.Database.Command`) и измеряйте время до и после каждой оптимизации.
- Index columns used in WHERE, JOIN, and ORDER BY — but not every column; each index slows writes.
- Use `AsSplitQuery` for many-to-many with large collections and benchmark both modes (`AsSingleQuery`/`AsSplitQuery`) on real data.
- For bulk status changes prefer `ExecuteUpdate`/`ExecuteDelete` over a `SaveChanges` loop.
- Project reads through `.Select` so you do not fetch extra columns or risk accidental mutation.
- Cache simple, hot queries with `EF.CompileAsyncQuery`; for rare complex ones it does not pay off.
- Enable SQL logging (`DbLoggerCategory.Database.Command`) and measure before and after every optimization.

#### Частые ошибки / Common Mistakes
- Индекс на каждую колонку → запись замедляется. → Индексируйте только поля, реально используемые в фильтрах и соединениях.
- `AsSplitQuery` везде → лишние round-trip на мелких выборках. → Измеряйте оба режима и выбирайте по данным.
- `SaveChanges` в цикле для 10 000 строк → OOM и медленно. → Используйте `ExecuteUpdate`/`ExecuteDelete`.
- Загрузка целой сущности ради одного поля → лишний трафик и материализация. → Применяйте `.Select`-проекцию.
- Ожидание, что `ExecuteUpdate` вызовет `SavingChanges`-хуки. → Помните: Change Tracker обходится; аудиторские поля выставляйте в SQL.
- Оптимизация «на глаз». → Действуйте только по цифрам профилировщика и `ToQueryString()`.
- Indexing every column → writes get slow. → Index only fields actually used in filters and joins.
- `AsSplitQuery` everywhere → extra round-trips on small sets. → Benchmark both modes and choose by data.
- `SaveChanges` in a loop for 10k rows → OOM and slow. → Use `ExecuteUpdate`/`ExecuteDelete`.
- Loading a full entity to read one field → wasted traffic and materialization. → Apply a `.Select` projection.
- Expecting `ExecuteUpdate` to fire `SavingChanges` hooks. → Remember: the Change Tracker is bypassed; set audit fields in SQL.
- Optimizing by eye. → Act only on profiler numbers and `ToQueryString()` output.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я добавил `HasIndex` для колонок из WHERE/JOIN/ORDER BY и проверил составной порядок.
- [ ] Я выбрал `AsSplitQuery` или `AsSingleQuery` по результатам измерений, а не по привычке.
- [ ] Я заменил циклы `SaveChanges` на `ExecuteUpdate`/`ExecuteDelete` для массовых операций.
- [ ] Я использую проекции `.Select` для чтения узкого набора полей.
- [ ] Я кэшировал горячие запросы через `EF.CompileAsyncQuery`.
- [ ] Я включил логирование SQL и проверил сгенерированные запросы через `ToQueryString()`.
- [ ] I added `HasIndex` for WHERE/JOIN/ORDER BY columns and verified composite order.
- [ ] I chose `AsSplitQuery` or `AsSingleQuery` based on benchmarks, not habit.
- [ ] I replaced `SaveChanges` loops with `ExecuteUpdate`/`ExecuteDelete` for bulk ops.
- [ ] I use `.Select` projections for narrow field reads.
- [ ] I cached hot queries with `EF.CompileAsyncQuery`.
- [ ] I enabled SQL logging and inspected generated queries via `ToQueryString()`.

#### Ресурсы / Resources
- [Microsoft Learn — EF Core Performance — https://learn.microsoft.com/ef/core/performance/](https://learn.microsoft.com/ef/core/performance/)
- [Single vs. Split Queries — https://learn.microsoft.com/ef/core/querying/single-split-queries](https://learn.microsoft.com/ef/core/querying/single-split-queries)
- [ExecuteUpdate/ExecuteDelete (bulk) — https://learn.microsoft.com/ef/core/saving/execute-insert-update-delete](https://learn.microsoft.com/ef/core/saving/execute-insert-update-delete)
- [Indexes (HasIndex, Include, HasFilter) — https://learn.microsoft.com/ef/core/modeling/indexes](https://learn.microsoft.com/ef/core/modeling/indexes)
- [Compiled queries & advanced performance — https://learn.microsoft.com/ef/core/performance/advanced-performance-topics#compiled-queries](https://learn.microsoft.com/ef/core/performance/advanced-performance-topics#compiled-queries)

---

[⬆ К модулю M12](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
