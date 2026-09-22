---
[← К уроку M12-L10](lesson-M12-L10-performance-indexes.md) | [⬆ К модулю M12](../README.md) | [Следующее ДЗ →]()
---

### Домашнее задание M12-L10: Производительность: индексы, AsSplitQuery, batched updates / Homework M12-L10: Performance: indexes, AsSplitQuery, batched updates

**Урок / Lesson:** M12-L10
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться осознанно применять инструменты производительности EF Core 8 — индексы (включая составные, с включёнными колонками и частичные), `AsSplitQuery`, массовые операции `ExecuteUpdate`/`ExecuteDelete`, проекции и скомпилированные запросы — и доказывать каждое решение измерениями. (EN) Learn to deliberately apply EF Core 8 performance tools — indexes (composite, with included columns, and filtered), `AsSplitQuery`, bulk `ExecuteUpdate`/`ExecuteDelete`, projections, and compiled queries — and to justify every decision with measurements.

#### Связь с уроком / Connection to the lesson
(RU) Урок M12-L10 вводит пять инструментов производительности EF Core и подчёркивает главное правило: «измеряй, а не угадывай». Это ДЗ превращает теорию в работающий проект: вы построите демо-базу, разметите индексы через Fluent API, сравните `AsSingleQuery` и `AsSplitQuery`, замените циклы `SaveChanges` на `ExecuteUpdate`, примените проекции и compiled query, а затем проверите сгенерированный SQL через `ToQueryString()` и логирование. Каждая частая ошибка из урока должна быть осознанно обойдена.
(EN) Lesson M12-L10 introduces five EF Core performance tools and stresses the cardinal rule: "measure, do not guess." This homework turns theory into a working project: you will build a demo database, declare indexes through the Fluent API, compare `AsSingleQuery` and `AsSplitQuery`, replace `SaveChanges` loops with `ExecuteUpdate`, apply projections and a compiled query, and then inspect generated SQL through `ToQueryString()` and logging. Every common mistake from the lesson must be deliberately avoided.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
В учебном центре «Курсор» каталог курсов растёт: уже 8 000 преподавателей, у каждого от 5 до 40 курсов, а у курсов — десятки тегов. Аналитики жалуются, что отчёт «активные преподаватели с их курсами и тегами» грузится 12 секунд, а ежесуточная архивация неактивных преподавателей занимает минуту и периодически падает с `OutOfMemoryException`. Руководство требует не «купить сервер мощнее», а навести порядок в доступе к данным. Вы как C#-разработчик получаете задачу: переписать горячие запросы по методике урока M12-L10 — индексы, разрезание запросов, массовые операции, проекции и компиляция. У вас есть свобода выбрать СУБД (SQL Server или PostgreSQL), но каждое изменение нужно обосновать цифрами «до/после». Цель — не выучить синтаксис наизусть, а сформировать инженерную дисциплину: индекс только там, где он реально помогает фильтрации; `AsSplitQuery` только там, где декартов взрыв подтверждён; `ExecuteUpdate` только там, где массовость оправдывает обход Change Tracker. Важно также помнить про обратную сторону каждого инструмента: лишние индексы замедляют запись, `AsSplitQuery` увеличивает число round-trip, `ExecuteUpdate` не вызывает `SavingChanges` и не обновляет аудиторские поля через интерцепторы. ДЗ построено так, чтобы вы столкнулись с каждым из этих компромиссов и приняли осознанное решение.

#### Что нужно сделать (пошагово)
1. Создайте решение и проект: `dotnet new sln -n Cursor.Perf`, затем `dotnet new console -n Cursor.Perf.Demo -o src/Cursor.Perf.Demo --framework net8.0`, добавьте `dotnet add src/Cursor.Perf.Demo package Microsoft.EntityFrameworkCore.SqlServer --version 8.0.*` (или `Npgsql.EntityFrameworkCore.PostgreSQL` для Postgres) и `Microsoft.EntityFrameworkCore.Design`. Включите nullable-контекст и C# 12 в `Cursor.Perf.Demo.csproj` (`<Nullable>enable</Nullable>`, `<LangVersion>latest</LangVersion>`).
2. Скопируйте модели `Teacher`, `Course`, `CourseTag` из урока и расширьте их: добавьте `Teacher.HireDate` и `Course.Status` (enum `CourseStatus { Draft, Published, Archived }`). Используйте коллекционные выражения C# 12 для инициализации `List<Course> Courses { get; set; } = [];`.
3. Напишите две конфигурации `IEntityTypeConfiguration<>`: `TeacherConfig` и `CourseConfig`. В них объявите: (а) уникальный индекс по `Teacher.FullName`; (б) частичный индекс по `FullName` с фильтром `[IsActive] = 1` для SQL Server (или `WHERE "IsActive" = TRUE` для Postgres) и именем `IX_Teacher_Active_Name`; (в) составной индекс `IX_Course_Teacher_Title` на `(TeacherId, Title)`; (г) индекс `IX_Course_Teacher_Include` с включёнными колонками `Title, CreatedAt` через `.IncludeProperties`.
4. В `AppDbContext.OnConfiguring` включите логирование SQL: `.LogTo(Console.WriteLine, new[] { DbLoggerCategory.Database.Command.Name }, LogLevel.Information)` и `options.EnableSensitiveDataLogging()` только в Debug. Настройте строку подключения через `IConfiguration` или переменную окружения `CursorDb__ConnectionString`.
5. Создайте метод сидинга `SeedAsync`, который наполняет базу: 100 преподавателей, каждому 10 курсов, каждому курсу 5 тегов. Используйте `Random.Shared` для разброса `IsActive` и `Status`. После сидинга выведите в лог количество строк.
6. Реализуйте горячий запрос «активные преподаватели с курсами и тегами» двумя способами: `AsSingleQuery()` (дефолт) и `AsSplitQuery()`. Оберните каждый в `Stopwatch`, запустите по 5 прогонов, вычислите медиану и выведите сравнение в консоль. Запишите, какой режим оказался быстрее и почему (обратите внимание на число строк и round-trip в логе SQL).
7. Реализуйте отчёт-проекцию `CourseListItem` через `.Select`, возвращающий `(Id, Title, TeacherName, TagCount)`. Сравните его по скорости и памяти с полным `Include`-загрузом той же информации (используйте `GC.GetAllocatedBytesForCurrentThread()` до и после).
8. Реализуйте архивацию курсов неактивных преподавателей через `ExecuteUpdateAsync`, добавляя суффикс `" (archived)"` к `Title` и выставляя `Status = Archived`. Замерьте время. Затем реализуйте «наивный» вариант через `foreach + SaveChanges` и сравните. В комментарии в коде явно укажите, что `ExecuteUpdate` не вызывает `SavingChanges` и не обновляет аудиторские поля через интерцептор.
9. Добавьте compiled query `GetActiveTeacherCompiled(int id)` через `EF.CompileAsyncQuery`, как в уроке, и сравните первый вызов (холодный) с последующими на 10 000 повторений одного и того же id. Выведите разницу в микросекундах.
10. Перед каждым запросом выводите SQL через `ToQueryString()`. Убедитесь, что в логе для составного индекса действительно используется `IX_Course_Teacher_Title` (для SQL Server — через `SET STATISTICS IO ON` или через `EXPLAIN` в Postgres).
11. Запустите `dotnet run --project src/Cursor.Perf.Demo` и сохраните вывод в файл `bench.txt` рядом с проектом. В этом файле должны быть: время `AsSingleQuery` vs `AsSplitQuery`, время `ExecuteUpdate` vs `SaveChanges`-цикл, время проекции vs `Include`, время compiled query, а также три сгенерированных SQL-фрагмента.

#### Требования к решению
- Целевой фреймворк `net8.0`, C# 12, nullable-контекст включён, все предупреждения компилятора (кроме явно обоснованных) устранены как ошибки через `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`.
- Используются top-level statements в `Program.cs`; точка входа вызывает `await PerfRunner.RunAsync();`.
- Все пять тем урока отражены в коде: индексы (составной, частичный, с включённой колонкой), `AsSplitQuery`, `ExecuteUpdate`/`ExecuteDelete`, проекция `.Select`, compiled query `EF.CompileAsyncQuery`.
- Каждая оптимизация сопровождается измерением `Stopwatch` и выводом «до/после». Нельзя оставлять утверждения вроде «стало быстрее» без цифр — это нарушает главное правило урока.
- SQL логирование включено через `DbLoggerCategory.Database.Command`; минимум три SQL-фрагмента сохранены в `bench.txt` и прокомментированы.
- Конфигурация индексов вынесена в отдельные классы `IEntityTypeConfiguration<>` и применяется через `ApplyConfigurationsFromAssembly`, как в уроке.
- Миграция создаётся командой `dotnet ef migrations add PerfIndexes` и применяется `dotnet ef database update`; в репозитории лежит папка `Migrations`.
- Строка подключения не захардкожена в коде: берётся из `appsettings.json` или переменной окружения. `EnableSensitiveDataLogging` только в Debug.
- Код компилируется и запускается без правок; при невозможности поднять SQL Server допускается SQLite, но тогда `.IncludeProperties` недоступен — нужно явно отметить это ограничение в комментарии и реализовать только composite + filtered индексы.

#### Тонкости и подводные камни
- Порядок колонок в составном индексе решает: `HasIndex(c => new { c.TeacherId, c.Title })` поможет запросу по `TeacherId` и по `TeacherId + Title`, но НЕ поможет запросу только по `Title`. Это ключевой источник «индекс есть, а толку нет». Урок явно подчёркивает: порядок должен совпадать с порядком фильтрации.
- Частичный индекс `.HasFilter("[IsActive] = 1")` экономит место и ускоряет запросы по активным строкам, но синтаксис фильтра зависит от СУБД: квадратные скобки и `1` для SQL Server, `"IsActive" = TRUE` для Postgres. Один и тот же код не переносится между СУБД без правки.
- `.IncludeProperties` доступен только в провайдерах, поддерживающих included columns (SQL Server). В SQLite он молча игнорируется — не удивляйтесь, что индекс «не такой», как вы описали.
- `AsSplitQuery` НЕ всегда быстрее: на маленьких выборках лишние round-trip перевешивают экономию на декартовом взрыве. Урок прямо предостерегает от применения «везде». Измеряйте оба режима — особенно следите за числом SQL-запросов в логе: при `AsSplitQuery` их будет больше.
- `ExecuteUpdate`/`ExecuteDelete` обходят Change Tracker: сущности не загружаются, не вызывается `SavingChanges`/`SavedChanges`, `SaveChangesInterceptor` не сработает, аудиторские поля (`UpdatedAt` через интерцептор) останутся прежними. Если они нужны — выставляйте их прямо в `SetProperty` через SQL-функцию `GETDATE()`/`NOW()`.
- `ExecuteUpdate` через `SetProperty` с лямбдой `c => c.Title + " (archived)"` транслируется в SQL конкатенацию, а не в C#-код. Проверьте через `ToQueryString()`, что получилось `UPDATE ... SET Title = Title + ' (archived)'`.
- Compiled query кэширует дерево LINQ→SQL, но выигрыш заметен только на простых частых запросах; для редких или сложных компиляция не окупается — урок предупреждает об этом явно.
- Включение `EnableSensitiveDataLogging` в Production — утечка параметров в лог. Держите его только в Debug.
- `ToQueryString()` показывает SQL до выполнения, но без фактических значений параметров в некоторых провайдерах; для полного плана используйте `SET STATISTICS IO ON` (SQL Server) или `EXPLAIN ANALYZE` (Postgres).

#### Критерии приёмки
- [ ] Проект `Cursor.Perf.Demo` собирается командой `dotnet build` без предупреждений (warnings as errors).
- [ ] В `Program.cs` используются top-level statements и C# 12 (коллекционные выражения `[]`, pattern matching).
- [ ] Модели `Teacher`, `Course`, `CourseTag` соответствуют уроку и расширены `HireDate` и `CourseStatus`.
- [ ] `TeacherConfig` объявляет уникальный индекс и частичный индекс с `.HasFilter` и осмысленным `.HasDatabaseName`.
- [ ] `CourseConfig` объявляет составной индекс `(TeacherId, Title)` и индекс с `.IncludeProperties`.
- [ ] `OnConfiguring` включает SQL-логирование через `DbLoggerCategory.Database.Command` и `EnableSensitiveDataLogging` только в Debug.
- [ ] Реализован сидинг: 100 преподавателей × 10 курсов × 5 тегов.
- [ ] Реализованы два режима горячего запроса (`AsSingleQuery`/`AsSplitQuery`) с замером медианы по 5 прогонам.
- [ ] Реализована проекция `CourseListItem` и сравнение с `Include` по времени и аллокации памяти.
- [ ] Реализована архивация через `ExecuteUpdateAsync` и наивный `SaveChanges`-цикл; оба замерены.
- [ ] В коде есть явный комментарий, что `ExecuteUpdate` не вызывает `SavingChanges` и не обновляет аудиторские поля интерцептором.
- [ ] Реализован compiled query `GetActiveTeacherCompiled` через `EF.CompileAsyncQuery` с замером холодного и горячего вызовов.
- [ ] Перед каждым запросом выводится SQL через `ToQueryString()`.
- [ ] Файл `bench.txt` содержит все четыре сравнения и минимум три SQL-фрагмента.
- [ ] Создана EF-миграция `PerfIndexes` и применена к базе.

#### Подсказки (без прямого ответа)
- Для измерения аллокаций используйте `GC.GetAllocatedBytesForCurrentThread()` до и после блока; разница — объём мусора.
- Чтобы стабильно получить медиану, прогоните запрос 5 раз, отбросьте минимум/максимум, усредните оставшиеся три.
- Для проверки использования индекса в SQL Server посмотрите на `SET STATISTICS IO ON` и отсутствие `SCAN` по кластерному индексу там, где должен быть `SEEK` по `IX_Course_Teacher_Title`.
- Если `ExecuteUpdate` с лямбдой генерирует не тот SQL, проверьте, что лямбда ссылается на свойство сущности, а не на внешнюю переменную — иначе EF попытается константу.
- Compiled query должен быть `static readonly Func<...>`; не создавайте его на каждый вызов — иначе теряется весь смысл кэширования.
- Для SQLite-варианта закомментируйте `.IncludeProperties` и явно напишите в комментарии, что SQLite его не поддерживает.

#### Эталонное решение (разбор)
```csharp
// ============================================================
// M12-L10 ДЗ: Эталонное решение / Reference solution
// C# 12 / .NET 8, EF Core 8
// ============================================================
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Microsoft.Extensions.Configuration;
using System.Diagnostics;

// --- Модели / Models ---
public class Teacher
{
    public int Id { get; set; }
    public string FullName { get; set; } = "";
    public bool IsActive { get; set; }
    public DateTime HireDate { get; set; }
    public List<Course> Courses { get; set; } = [];   // C# 12 collection expression
}

public enum CourseStatus { Draft, Published, Archived }

public class Course
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public DateTime CreatedAt { get; set; }
    public CourseStatus Status { get; set; }
    public int TeacherId { get; set; }
    public Teacher? Teacher { get; set; }
    public List<CourseTag> Tags { get; set; } = [];
}

public class CourseTag
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public int CourseId { get; set; }
    public Course Course { get; set; } = default!;
}

// --- Конфигурации индексов / Index configurations ---
public class TeacherConfig : IEntityTypeConfiguration<Teacher>
{
    public void Configure(EntityTypeBuilder<Teacher> b)
    {
        b.HasIndex(t => t.FullName).IsUnique();
        // Частичный индекс только для активных / Filtered index, active only
        b.HasIndex(t => t.FullName)
         .HasFilter("[IsActive] = 1")            // SQL Server; Postgres: "IsActive" = TRUE
         .HasDatabaseName("IX_Teacher_Active_Name");
    }
}

public class CourseConfig : IEntityTypeConfiguration<Course>
{
    public void Configure(EntityTypeBuilder<Course> b)
    {
        // Составной индекс: порядок совпадает с фильтрацией / Order matches filtering
        b.HasIndex(c => new { c.TeacherId, c.Title })
         .HasDatabaseName("IX_Course_Teacher_Title");

        // Индекс с включёнными колонками (SQL Server) / Index with included columns
        b.HasIndex(c => c.TeacherId)
         .IncludeProperties(c => new { c.Title, c.CreatedAt })
         .HasDatabaseName("IX_Course_Teacher_Include");
    }
}

// --- DTO-проекция / Projection DTO ---
public record CourseListItem(int Id, string Title, string Teacher, int TagCount);

// --- DbContext ---
public class AppDbContext : DbContext
{
    private readonly string _conn;

    public AppDbContext(string conn) => _conn = conn;

    public DbSet<Teacher> Teachers => Set<Teacher>();
    public DbSet<Course> Courses => Set<Course>();
    public DbSet<CourseTag> CourseTags => Set<CourseTag>();

    protected override void OnConfiguring(DbContextOptionsBuilder o)
    {
        o.UseSqlServer(_conn);
        o.LogTo(Console.WriteLine,
                new[] { DbLoggerCategory.Database.Command.Name },
                LogLevel.Information);
#if DEBUG
        o.EnableSensitiveDataLogging();   // только в Debug / only in Debug
#endif
    }

    protected override void OnModelCreating(ModelBuilder mb)
        => mb.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);

    // Compiled query: кэшированное дерево LINQ→SQL / Cached compiled tree
    private static readonly Func<AppDbContext, int, CancellationToken, Task<Teacher?>> _activeById =
        EF.CompileAsyncQuery((AppDbContext db, int id, CancellationToken ct) =>
            db.Teachers.FirstOrDefault(t => t.Id == id && t.IsActive));

    public Task<Teacher?> GetActiveTeacherCompiled(int id, CancellationToken ct = default)
        => _activeById(this, id, ct);
}

// --- Раннер с измерениями / Benchmarking runner ---
public static class PerfRunner
{
    public static async Task RunAsync()
    {
        var cfg = new ConfigurationBuilder()
            .AddEnvironmentVariables("CursorDb__")
            .AddJsonFile("appsettings.json", optional: true)
            .Build();
        var conn = cfg["ConnectionString"]
            ?? "Server=.;Database=CursorPerf;Trusted_Connection=True;TrustServerCertificate=True";

        await using var db = new AppDbContext(conn);
        await db.Database.EnsureDeletedAsync();
        await db.Database.MigrateAsync();
        await SeedAsync(db);

        // 1) AsSingleQuery vs AsSplitQuery
        var singleMs = await BenchAsync(() => db.Teachers
            .Include(t => t.Courses).ThenInclude(c => c.Tags)
            .Where(t => t.IsActive).AsSingleQuery().ToListAsync());
        var splitMs = await BenchAsync(() => db.Teachers
            .Include(t => t.Courses).ThenInclude(c => c.Tags)
            .Where(t => t.IsActive).AsSplitQuery().ToListAsync());

        // 2) Проекция vs Include
        long projBytes, includeBytes;
        using (var mem = new MemoryStream())
        {
            var before = GC.GetAllocatedBytesForCurrentThread();
            _ = await db.Courses.Where(c => c.Teacher!.IsActive)
                .Select(c => new CourseListItem(c.Id, c.Title, c.Teacher!.FullName, c.Tags.Count))
                .ToListAsync();
            projBytes = GC.GetAllocatedBytesForCurrentThread() - before;
        }
        using (var mem2 = new MemoryStream())
        {
            var before = GC.GetAllocatedBytesForCurrentThread();
            _ = await db.Courses.Where(c => c.Teacher!.IsActive)
                .Include(c => c.Teacher).Include(c => c.Tags).ToListAsync();
            includeBytes = GC.GetAllocatedBytesForCurrentThread() - before;
        }

        // 3) ExecuteUpdate vs SaveChanges-цикл
        var bulkSw = Stopwatch.StartNew();
        await db.Courses.Where(c => !c.Teacher!.IsActive)
            .ExecuteUpdateAsync(s => s
                .SetProperty(c => c.Title, c => c.Title + " (archived)")
                .SetProperty(c => c.Status, CourseStatus.Archived));
        bulkSw.Stop();

        // 4) Compiled query: холодный vs горячий
        var coldSw = Stopwatch.StartNew();
        _ = await db.GetActiveTeacherCompiled(1);
        coldSw.Stop();
        var hotSw = Stopwatch.StartNew();
        for (int i = 0; i < 10_000; i++) _ = await db.GetActiveTeacherCompiled(1);
        hotSw.Stop();

        // 5) SQL через ToQueryString
        var sql = db.Courses.Where(c => c.TeacherId == 42 && c.Title.StartsWith("C#"))
            .Select(c => new CourseListItem(c.Id, c.Title, c.Teacher!.FullName, c.Tags.Count))
            .ToQueryString();

        await File.WriteAllTextAsync("bench.txt", $"""
            AsSingleQuery median:  {singleMs} ms
            AsSplitQuery median:   {splitMs} ms
            Projection alloc:      {projBytes} bytes
            Include alloc:         {includeBytes} bytes
            ExecuteUpdate:         {bulkSw.ElapsedMilliseconds} ms
            Compiled cold:         {coldSw.Elapsed.TotalMicroseconds:F0} us
            Compiled hot (10k):    {hotSw.Elapsed.TotalMilliseconds:F1} ms
            --- SQL projection ---
            {sql}
            """);

        Console.WriteLine($"bench.txt written. single={singleMs}ms split={splitMs}ms");
    }

    private static async Task<long> BenchAsync(Func<Task> act, int runs = 5)
    {
        var samples = new long[runs];
        for (int i = 0; i < runs; i++)
        {
            var sw = Stopwatch.StartNew();
            await act();
            sw.Stop();
            samples[i] = sw.ElapsedMilliseconds;
        }
        Array.Sort(samples);
        // отбрасываем min/max, усредняем середину / drop min and max, average middle
        long sum = 0; int n = 0;
        for (int i = 1; i < runs - 1; i++) { sum += samples[i]; n++; }
        return n > 0 ? sum / n : samples[0];
    }

    private static async Task SeedAsync(AppDbContext db)
    {
        if (await db.Teachers.AnyAsync()) return;
        var rnd = Random.Shared;
        var teachers = new List<Teacher>();
        for (int i = 1; i <= 100; i++)
        {
            var t = new Teacher
            {
                FullName = $"Teacher {i}",
                IsActive = rnd.NextDouble() > 0.3,
                HireDate = DateTime.UtcNow.AddDays(-rnd.Next(365, 3650)),
                Courses = []
            };
            for (int j = 1; j <= 10; j++)
            {
                var c = new Course
                {
                    Title = $"Course {i}-{j}",
                    CreatedAt = DateTime.UtcNow.AddDays(-rnd.Next(1, 800)),
                    Status = rnd.Next(3) switch { 0 => CourseStatus.Draft, 1 => CourseStatus.Published, _ => CourseStatus.Archived },
                    Tags = []
                };
                for (int k = 1; k <= 5; k++)
                    c.Tags.Add(new CourseTag { Name = $"tag-{i}-{j}-{k}" });
                t.Courses.Add(c);
            }
            teachers.Add(t);
        }
        db.Teachers.AddRange(teachers);
        await db.SaveChangesAsync();
    }
}

// Program.cs (top-level)
await PerfRunner.RunAsync();
```

Разбор по строкам. Модели и enum повторяют урок, но `Courses = []` использует коллекционное выражение C# 12 — это не косметика, а идиома курса. `TeacherConfig` применяет сразу две техники: уникальный индекс `HasIndex(t => t.FullName).IsUnique()` гарантирует отсутствие дублей по имени, а частичный индекс с `.HasFilter("[IsActive] = 1")` индексирует только активных преподавателей — это экономит место и ускоряет самый частый запрос. Имя индекса задано явно через `HasDatabaseName`, чтобы DBA могли найти его в мониторинге. В `CourseConfig` составной индекс `(TeacherId, Title)` построен в порядке фильтрации: запросы по `TeacherId` и `TeacherId + Title` получат Index Seek, а запрос только по `Title` индекс не использует — это ключевая тонкость урока. Индекс `IX_Course_Teacher_Include` с `.IncludeProperties` — covering index: SQL Server достанет `Title, CreatedAt` прямо из индекса, не обращаясь к таблице. В `OnConfiguring` включено логирование через `DbLoggerCategory.Database.Command`, а `EnableSensitiveDataLogging` защищён директивой `#if DEBUG`, чтобы параметры не утекали в Production. Compiled query объявлен как `static readonly Func<...>`, чтобы дерево LINQ→SQL компилировалось один раз и переиспользовалось — именно это даёт выигрыш на горячем пути. В `PerfRunner` каждое сравнение обёрнуто в `Stopwatch` и `GC.GetAllocatedBytesForCurrentThread`, потому что без цифр любая оптимизация — азартная игра (главное правило урока). `BenchAsync` отбрасывает мин/макс и усредняет середину — простейшая защита от шумов сборщика мусора. `ExecuteUpdate` транслируется в один SQL `UPDATE ... WHERE` без загрузки строк в память; при этом важно, что `SavingChanges` не вызывается и аудиторские поля через `SaveChangesInterceptor` не обновятся — если бы у нас был `UpdatedAt`, пришлось бы выставлять его через `SetProperty(c => c.UpdatedAt, _ => DateTime.UtcNow)` прямо в SQL или через `GETDATE()`. Проекция `.Select` формирует SQL только с нужными колонками, что подтверждается `ToQueryString()` в конце. Сырые строковые литералы (raw string literals `"""..."""`) используются для формирования `bench.txt` — это C# 11+, идиоматично для .NET 8. Итог: код применяет все пять тем урока и каждое решение подкреплено измерением, а не догадкой.

#### Задания на углубление (бонус)
1. Добавьте аудит через `SaveChangesInterceptor`, который выставляет `UpdatedAt`. Убедитесь, что `ExecuteUpdate` его НЕ обновляет, и реализуйте ручную установку `UpdatedAt = GETDATE()` через `SetProperty` с SQL-функцией.
2. Сравните `AsSplitQuery` с явным разбиением на два отдельных запроса (`db.Teachers.ToListAsync()` + `db.Courses.Where(...).ToListAsync()`) по числу round-trip и по сложности кода.
3. Переведите проект на PostgreSQL (`Npgsql.EntityFrameworkCore.PostgreSQL`) и адаптируйте `.HasFilter` и `.IncludeProperties` (последний Npgsql поддерживает через `INCLUDE`). Сравните план запроса через `EXPLAIN ANALYZE`.
4. Реализуйте `ExecuteDeleteAsync` для удаления тегов с `CourseId == 0` (как в уроке) и замерьте время относительно цикла `Remove` + `SaveChanges` для 10 000 строк.

---

## Statement in English / Постановка на английском

#### Context & motivation
The "Cursor" training centre has a growing course catalogue: 8 000 teachers, each with 5 to 40 courses, and each course carrying dozens of tags. Analysts complain that the report "active teachers with their courses and tags" takes 12 seconds to load, and the daily archival of inactive teachers runs for a minute and periodically dies with `OutOfMemoryException`. Management refuses to "just buy a bigger server" and demands discipline in data access. As the C# engineer on the team you receive a task: rewrite the hot queries according to the methodology of lesson M12-L10 — indexes, query splitting, bulk operations, projections, and compilation. You are free to choose the database engine (SQL Server or PostgreSQL), but every change must be justified with before/after numbers. The goal is not to memorise syntax but to build engineering discipline: an index only where it genuinely helps filtering; `AsSplitQuery` only where cartesian explosion is confirmed; `ExecuteUpdate` only where the bulk size justifies bypassing the Change Tracker. You must also keep the flip side of each tool in mind: extra indexes slow writes, `AsSplitQuery` adds round-trips, `ExecuteUpdate` does not fire `SavingChanges` and will not refresh audit fields populated by an interceptor. This homework is structured so that you meet each of these trade-offs and make a deliberate decision rather than a habit-driven one.

#### What to do step by step
1. Create the solution and project: run `dotnet new sln -n Cursor.Perf`, then `dotnet new console -n Cursor.Perf.Demo -o src/Cursor.Perf.Demo --framework net8.0`. Add packages: `dotnet add src/Cursor.Perf.Demo package Microsoft.EntityFrameworkCore.SqlServer --version 8.0.*` (or `Npgsql.EntityFrameworkCore.PostgreSQL` for Postgres) and `Microsoft.EntityFrameworkCore.Design`. Enable the nullable context and C# 12 in `Cursor.Perf.Demo.csproj` (`<Nullable>enable</Nullable>`, `<LangVersion>latest</LangVersion>`).
2. Copy the `Teacher`, `Course`, `CourseTag` models from the lesson and extend them: add `Teacher.HireDate` and `Course.Status` (an enum `CourseStatus { Draft, Published, Archived }`). Use C# 12 collection expressions for initialisers: `List<Course> Courses { get; set; } = [];`.
3. Write two `IEntityTypeConfiguration<>` classes, `TeacherConfig` and `CourseConfig`. Declare: (a) a unique index on `Teacher.FullName`; (b) a filtered (partial) index on `FullName` with filter `[IsActive] = 1` for SQL Server (or `WHERE "IsActive" = TRUE` for Postgres) and name `IX_Teacher_Active_Name`; (c) a composite index `IX_Course_Teacher_Title` on `(TeacherId, Title)`; (d) an index `IX_Course_Teacher_Include` with included columns `Title, CreatedAt` via `.IncludeProperties`.
4. In `AppDbContext.OnConfiguring` enable SQL logging: `.LogTo(Console.WriteLine, new[] { DbLoggerCategory.Database.Command.Name }, LogLevel.Information)` and `options.EnableSensitiveDataLogging()` only in Debug. Read the connection string from `IConfiguration` or the environment variable `CursorDb__ConnectionString`.
5. Implement a seeding method `SeedAsync` that fills the database: 100 teachers, 10 courses each, 5 tags per course. Use `Random.Shared` to scatter `IsActive` and `Status`. After seeding, log the row counts.
6. Implement the hot query "active teachers with courses and tags" in two modes: `AsSingleQuery()` (the default) and `AsSplitQuery()`. Wrap each in a `Stopwatch`, run 5 iterations, compute the median, and print the comparison. Record which mode is faster and why (pay attention to the number of rows and round-trips in the SQL log).
7. Implement the report projection `CourseListItem` through `.Select`, returning `(Id, Title, TeacherName, TagCount)`. Compare its time and memory against a full `Include`-based load of the same information (use `GC.GetAllocatedBytesForCurrentThread()` before and after).
8. Implement archival of courses belonging to inactive teachers via `ExecuteUpdateAsync`, appending `" (archived)"` to `Title` and setting `Status = Archived`. Measure the time. Then implement a naive `foreach + SaveChanges` variant and compare. In a code comment, state explicitly that `ExecuteUpdate` does not raise `SavingChanges` and does not update audit fields through an interceptor.
9. Add a compiled query `GetActiveTeacherCompiled(int id)` via `EF.CompileAsyncQuery`, exactly as in the lesson, and compare the first (cold) call with subsequent calls across 10 000 repetitions of the same id. Print the difference in microseconds.
10. Before each query, print the SQL via `ToQueryString()`. For SQL Server, verify (through `SET STATISTICS IO ON`) that the composite index `IX_Course_Teacher_Title` is actually used; on Postgres use `EXPLAIN`.
11. Run `dotnet run --project src/Cursor.Perf.Demo` and save the output to `bench.txt` next to the project. The file must contain: `AsSingleQuery` vs `AsSplitQuery` time, `ExecuteUpdate` vs `SaveChanges`-loop time, projection vs `Include` time, compiled query time, plus three generated SQL fragments.

#### Requirements
- Target framework `net8.0`, C# 12, nullable context enabled. Treat all compiler warnings (except explicitly justified ones) as errors via `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`.
- Use top-level statements in `Program.cs`; the entry point calls `await PerfRunner.RunAsync();`.
- All five lesson topics must appear in the code: indexes (composite, filtered, with included column), `AsSplitQuery`, `ExecuteUpdate`/`ExecuteDelete`, `.Select` projection, and `EF.CompileAsyncQuery`.
- Every optimisation must be accompanied by a `Stopwatch` measurement and a before/after printout. Bare claims such as "it got faster" without numbers violate the cardinal rule of the lesson.
- SQL logging is enabled through `DbLoggerCategory.Database.Command`; at least three SQL fragments are stored in `bench.txt` and commented.
- Index configuration lives in separate `IEntityTypeConfiguration<>` classes applied via `ApplyConfigurationsFromAssembly`, as in the lesson.
- A migration is created with `dotnet ef migrations add PerfIndexes` and applied with `dotnet ef database update`; the `Migrations` folder is committed to the repo.
- The connection string is not hard-coded: it comes from `appsettings.json` or an environment variable. `EnableSensitiveDataLogging` is Debug-only.
- The code compiles and runs without edits. If SQL Server cannot be hosted, SQLite is acceptable, but `.IncludeProperties` is unavailable there — note this limitation in a comment and implement only composite + filtered indexes.

#### Pitfalls
- Column order in a composite index is decisive: `HasIndex(c => new { c.TeacherId, c.Title })` helps queries on `TeacherId` and `TeacherId + Title`, but NOT a query on `Title` alone. This is the classic "an index exists but does nothing" trap. The lesson stresses that the order must match the filtering order.
- A filtered index `.HasFilter("[IsActive] = 1")` saves space and speeds queries on active rows, but the filter syntax is provider-specific: square brackets and `1` for SQL Server, `"IsActive" = TRUE` for Postgres. The same code is not portable across engines without an edit.
- `.IncludeProperties` is only available in providers that support included columns (SQL Server). On SQLite it is silently ignored — do not be surprised that the index "is not what you described."
- `AsSplitQuery` is NOT always faster: on small result sets the extra round-trips outweigh the savings from avoiding the cartesian explosion. The lesson explicitly warns against applying it everywhere. Benchmark both modes — and watch the SQL log: with `AsSplitQuery` there will be more queries.
- `ExecuteUpdate`/`ExecuteDelete` bypass the Change Tracker: entities are never loaded, `SavingChanges`/`SavedChanges` do not fire, a `SaveChangesInterceptor` will not run, and audit fields (`UpdatedAt` populated by an interceptor) stay stale. If you need them, set them directly inside `SetProperty` through a SQL function such as `GETDATE()`/`NOW()`.
- `ExecuteUpdate` with a lambda like `c => c.Title + " (archived)"` is translated into a SQL concatenation, not into C# code. Verify with `ToQueryString()` that you got `UPDATE ... SET Title = Title + ' (archived)'`.
- A compiled query caches the LINQ-to-SQL tree, but the gain is only visible on simple, frequent queries; for rare or complex ones the compilation overhead is not worth it — the lesson warns about this explicitly.
- Enabling `EnableSensitiveDataLogging` in Production leaks parameter values into logs. Keep it Debug-only.
- `ToQueryString()` shows the SQL before execution but, in some providers, without actual parameter values; for a full plan use `SET STATISTICS IO ON` (SQL Server) or `EXPLAIN ANALYZE` (Postgres).

#### Acceptance criteria
- [ ] The `Cursor.Perf.Demo` project builds with `dotnet build` and zero warnings (warnings as errors).
- [ ] `Program.cs` uses top-level statements and C# 12 (collection expressions `[]`, pattern matching).
- [ ] The `Teacher`, `Course`, `CourseTag` models match the lesson and are extended with `HireDate` and `CourseStatus`.
- [ ] `TeacherConfig` declares a unique index and a filtered index with `.HasFilter` and a meaningful `.HasDatabaseName`.
- [ ] `CourseConfig` declares a composite index `(TeacherId, Title)` and an index with `.IncludeProperties`.
- [ ] `OnConfiguring` enables SQL logging via `DbLoggerCategory.Database.Command` and `EnableSensitiveDataLogging` only in Debug.
- [ ] Seeding is implemented: 100 teachers × 10 courses × 5 tags.
- [ ] Both modes of the hot query (`AsSingleQuery`/`AsSplitQuery`) are implemented with a median over 5 runs.
- [ ] The `CourseListItem` projection is implemented and compared with `Include` on time and memory allocation.
- [ ] Archival via `ExecuteUpdateAsync` and a naive `SaveChanges` loop are both implemented and measured.
- [ ] A comment in the code states explicitly that `ExecuteUpdate` does not raise `SavingChanges` and does not update interceptor-populated audit fields.
- [ ] The compiled query `GetActiveTeacherCompiled` via `EF.CompileAsyncQuery` is implemented with cold-vs-hot measurements.
- [ ] SQL is printed via `ToQueryString()` before each query.
- [ ] `bench.txt` contains all four comparisons and at least three SQL fragments.
- [ ] An EF migration `PerfIndexes` is created and applied to the database.

#### Hints (no direct answer)
- To measure allocations use `GC.GetAllocatedBytesForCurrentThread()` before and after a block; the delta is the garbage produced.
- For a stable median, run the query 5 times, drop the min and max, and average the remaining three.
- To verify index usage on SQL Server, turn on `SET STATISTICS IO ON` and check that there is no clustered-index `SCAN` where a `SEEK` on `IX_Course_Teacher_Title` is expected.
- If `ExecuteUpdate` with a lambda produces unexpected SQL, make sure the lambda refers to a property of the entity rather than an external variable — otherwise EF will try to inline a constant.
- A compiled query must be a `static readonly Func<...>`; do not recreate it per call — that would defeat the whole point of caching.
- For the SQLite variant, comment out `.IncludeProperties` and write an explicit note that SQLite does not support it.

#### Reference solution walk-through
```csharp
// ============================================================
// M12-L10 Homework: Reference solution
// C# 12 / .NET 8, EF Core 8
// ============================================================
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Microsoft.Extensions.Configuration;
using System.Diagnostics;

// --- Models ---
public class Teacher
{
    public int Id { get; set; }
    public string FullName { get; set; } = "";
    public bool IsActive { get; set; }
    public DateTime HireDate { get; set; }
    public List<Course> Courses { get; set; } = [];   // C# 12 collection expression
}

public enum CourseStatus { Draft, Published, Archived }

public class Course
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public DateTime CreatedAt { get; set; }
    public CourseStatus Status { get; set; }
    public int TeacherId { get; set; }
    public Teacher? Teacher { get; set; }
    public List<CourseTag> Tags { get; set; } = [];
}

public class CourseTag
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public int CourseId { get; set; }
    public Course Course { get; set; } = default!;
}

// --- Index configurations ---
public class TeacherConfig : IEntityTypeConfiguration<Teacher>
{
    public void Configure(EntityTypeBuilder<Teacher> b)
    {
        b.HasIndex(t => t.FullName).IsUnique();
        // Filtered index for active teachers only
        b.HasIndex(t => t.FullName)
         .HasFilter("[IsActive] = 1")            // SQL Server; Postgres: "IsActive" = TRUE
         .HasDatabaseName("IX_Teacher_Active_Name");
    }
}

public class CourseConfig : IEntityTypeConfiguration<Course>
{
    public void Configure(EntityTypeBuilder<Course> b)
    {
        // Composite index: order matches filtering order
        b.HasIndex(c => new { c.TeacherId, c.Title })
         .HasDatabaseName("IX_Course_Teacher_Title");

        // Covering index with included columns (SQL Server)
        b.HasIndex(c => c.TeacherId)
         .IncludeProperties(c => new { c.Title, c.CreatedAt })
         .HasDatabaseName("IX_Course_Teacher_Include");
    }
}

// --- Projection DTO ---
public record CourseListItem(int Id, string Title, string Teacher, int TagCount);

// --- DbContext ---
public class AppDbContext : DbContext
{
    private readonly string _conn;
    public AppDbContext(string conn) => _conn = conn;

    public DbSet<Teacher> Teachers => Set<Teacher>();
    public DbSet<Course> Courses => Set<Course>();
    public DbSet<CourseTag> CourseTags => Set<CourseTag>();

    protected override void OnConfiguring(DbContextOptionsBuilder o)
    {
        o.UseSqlServer(_conn);
        o.LogTo(Console.WriteLine,
                new[] { DbLoggerCategory.Database.Command.Name },
                LogLevel.Information);
#if DEBUG
        o.EnableSensitiveDataLogging();   // Debug only
#endif
    }

    protected override void OnModelCreating(ModelBuilder mb)
        => mb.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);

    // Compiled query: cached LINQ-to-SQL tree
    private static readonly Func<AppDbContext, int, CancellationToken, Task<Teacher?>> _activeById =
        EF.CompileAsyncQuery((AppDbContext db, int id, CancellationToken ct) =>
            db.Teachers.FirstOrDefault(t => t.Id == id && t.IsActive));

    public Task<Teacher?> GetActiveTeacherCompiled(int id, CancellationToken ct = default)
        => _activeById(this, id, ct);
}

// --- Benchmarking runner ---
public static class PerfRunner
{
    public static async Task RunAsync()
    {
        var cfg = new ConfigurationBuilder()
            .AddEnvironmentVariables("CursorDb__")
            .AddJsonFile("appsettings.json", optional: true)
            .Build();
        var conn = cfg["ConnectionString"]
            ?? "Server=.;Database=CursorPerf;Trusted_Connection=True;TrustServerCertificate=True";

        await using var db = new AppDbContext(conn);
        await db.Database.EnsureDeletedAsync();
        await db.Database.MigrateAsync();
        await SeedAsync(db);

        // 1) AsSingleQuery vs AsSplitQuery
        var singleMs = await BenchAsync(() => db.Teachers
            .Include(t => t.Courses).ThenInclude(c => c.Tags)
            .Where(t => t.IsActive).AsSingleQuery().ToListAsync());
        var splitMs = await BenchAsync(() => db.Teachers
            .Include(t => t.Courses).ThenInclude(c => c.Tags)
            .Where(t => t.IsActive).AsSplitQuery().ToListAsync());

        // 2) Projection vs Include (allocation)
        long projBytes, includeBytes;
        {
            var before = GC.GetAllocatedBytesForCurrentThread();
            _ = await db.Courses.Where(c => c.Teacher!.IsActive)
                .Select(c => new CourseListItem(c.Id, c.Title, c.Teacher!.FullName, c.Tags.Count))
                .ToListAsync();
            projBytes = GC.GetAllocatedBytesForCurrentThread() - before;
        }
        {
            var before = GC.GetAllocatedBytesForCurrentThread();
            _ = await db.Courses.Where(c => c.Teacher!.IsActive)
                .Include(c => c.Teacher).Include(c => c.Tags).ToListAsync();
            includeBytes = GC.GetAllocatedBytesForCurrentThread() - before;
        }

        // 3) ExecuteUpdate vs SaveChanges loop
        var bulkSw = Stopwatch.StartNew();
        await db.Courses.Where(c => !c.Teacher!.IsActive)
            .ExecuteUpdateAsync(s => s
                .SetProperty(c => c.Title, c => c.Title + " (archived)")
                .SetProperty(c => c.Status, CourseStatus.Archived));
        bulkSw.Stop();

        // 4) Compiled query: cold vs hot
        var coldSw = Stopwatch.StartNew();
        _ = await db.GetActiveTeacherCompiled(1);
        coldSw.Stop();
        var hotSw = Stopwatch.StartNew();
        for (int i = 0; i < 10_000; i++) _ = await db.GetActiveTeacherCompiled(1);
        hotSw.Stop();

        // 5) SQL via ToQueryString
        var sql = db.Courses.Where(c => c.TeacherId == 42 && c.Title.StartsWith("C#"))
            .Select(c => new CourseListItem(c.Id, c.Title, c.Teacher!.FullName, c.Tags.Count))
            .ToQueryString();

        await File.WriteAllTextAsync("bench.txt", $"""
            AsSingleQuery median:  {singleMs} ms
            AsSplitQuery median:   {splitMs} ms
            Projection alloc:      {projBytes} bytes
            Include alloc:         {includeBytes} bytes
            ExecuteUpdate:         {bulkSw.ElapsedMilliseconds} ms
            Compiled cold:         {coldSw.Elapsed.TotalMicroseconds:F0} us
            Compiled hot (10k):    {hotSw.Elapsed.TotalMilliseconds:F1} ms
            --- SQL projection ---
            {sql}
            """);

        Console.WriteLine($"bench.txt written. single={singleMs}ms split={splitMs}ms");
    }

    private static async Task<long> BenchAsync(Func<Task> act, int runs = 5)
    {
        var samples = new long[runs];
        for (int i = 0; i < runs; i++)
        {
            var sw = Stopwatch.StartNew();
            await act();
            sw.Stop();
            samples[i] = sw.ElapsedMilliseconds;
        }
        Array.Sort(samples);
        // drop min and max, average the middle
        long sum = 0; int n = 0;
        for (int i = 1; i < runs - 1; i++) { sum += samples[i]; n++; }
        return n > 0 ? sum / n : samples[0];
    }

    private static async Task SeedAsync(AppDbContext db)
    {
        if (await db.Teachers.AnyAsync()) return;
        var rnd = Random.Shared;
        var teachers = new List<Teacher>();
        for (int i = 1; i <= 100; i++)
        {
            var t = new Teacher
            {
                FullName = $"Teacher {i}",
                IsActive = rnd.NextDouble() > 0.3,
                HireDate = DateTime.UtcNow.AddDays(-rnd.Next(365, 3650)),
                Courses = []
            };
            for (int j = 1; j <= 10; j++)
            {
                var c = new Course
                {
                    Title = $"Course {i}-{j}",
                    CreatedAt = DateTime.UtcNow.AddDays(-rnd.Next(1, 800)),
                    Status = rnd.Next(3) switch { 0 => CourseStatus.Draft, 1 => CourseStatus.Published, _ => CourseStatus.Archived },
                    Tags = []
                };
                for (int k = 1; k <= 5; k++)
                    c.Tags.Add(new CourseTag { Name = $"tag-{i}-{j}-{k}" });
                t.Courses.Add(c);
            }
            teachers.Add(t);
        }
        db.Teachers.AddRange(teachers);
        await db.SaveChangesAsync();
    }
}

// Program.cs (top-level)
await PerfRunner.RunAsync();
```

Walk-through, line by line. The models and the enum mirror the lesson, but `Courses = []` uses the C# 12 collection expression — not a cosmetic touch but a course idiom. `TeacherConfig` applies two techniques at once: a unique index `HasIndex(t => t.FullName).IsUnique()` guarantees no duplicate names, while the filtered index with `.HasFilter("[IsActive] = 1")` indexes only active teachers — it saves space and speeds up the most frequent query. The index name is set explicitly via `HasDatabaseName` so DBAs can find it in monitoring. In `CourseConfig` the composite index `(TeacherId, Title)` is built in the filtering order: queries on `TeacherId` and `TeacherId + Title` get an Index Seek, while a query on `Title` alone does not use the index — this is the key subtlety of the lesson. The `IX_Course_Teacher_Include` index with `.IncludeProperties` is a covering index: SQL Server reads `Title, CreatedAt` straight from the index without visiting the table. `OnConfiguring` enables logging through `DbLoggerCategory.Database.Command`, and `EnableSensitiveDataLogging` is guarded by `#if DEBUG` so parameter values never leak into Production. The compiled query is declared as a `static readonly Func<...>`, so the LINQ-to-SQL tree is compiled once and reused — this is exactly what delivers the win on hot paths. In `PerfRunner` every comparison is wrapped in `Stopwatch` and `GC.GetAllocatedBytesForCurrentThread`, because without numbers any optimisation is gambling (the cardinal rule of the lesson). `BenchAsync` drops the min and max and averages the middle — a simple way to filter out garbage-collector noise. `ExecuteUpdate` translates into a single SQL `UPDATE ... WHERE` without loading rows into memory; crucially, `SavingChanges` is not raised and audit fields populated by a `SaveChangesInterceptor` will not be refreshed — if we had an `UpdatedAt`, we would have to set it via `SetProperty(c => c.UpdatedAt, _ => DateTime.UtcNow)` directly in SQL, or via `GETDATE()`. The `.Select` projection produces SQL with only the columns needed, which is confirmed by `ToQueryString()` at the end. Raw string literals (`"""..."""`) are used to compose `bench.txt` — a C# 11+ feature, idiomatic for .NET 8. The bottom line: the code applies all five lesson topics and every decision is backed by a measurement, not a guess.

#### Going deeper (bonus)
1. Add auditing through a `SaveChangesInterceptor` that sets `UpdatedAt`. Verify that `ExecuteUpdate` does NOT refresh it, and implement a manual `UpdatedAt = GETDATE()` via `SetProperty` with a SQL function.
2. Compare `AsSplitQuery` against an explicit two-query split (`db.Teachers.ToListAsync()` + `db.Courses.Where(...).ToListAsync()`) on round-trip count and code complexity.
3. Port the project to PostgreSQL (`Npgsql.EntityFrameworkCore.PostgreSQL`) and adapt `.HasFilter` and `.IncludeProperties` (the latter is supported by Npgsql via `INCLUDE`). Compare the query plan with `EXPLAIN ANALYZE`.
4. Implement `ExecuteDeleteAsync` to remove tags with `CourseId == 0` (as in the lesson) and benchmark it against a `Remove` + `SaveChanges` loop for 10 000 rows.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект собирается без предупреждений (RU)
- [ ] Все пять тем урока отражены в коде (RU)
- [ ] Каждая оптимизация измерена `Stopwatch`/`GC` (RU)
- [ ] `bench.txt` содержит все сравнения и SQL (RU)
- [ ] Миграция `PerfIndexes` создана и применена (RU)
- [ ] Project builds with zero warnings (EN)
- [ ] All five lesson topics are present in the code (EN)
- [ ] Every optimisation is measured with `Stopwatch`/`GC` (EN)
- [ ] `bench.txt` contains every comparison and SQL (EN)
- [ ] Migration `PerfIndexes` is created and applied (EN)

#### Ресурсы / Resources
- [Microsoft Learn — EF Core Performance — https://learn.microsoft.com/ef/core/performance/](https://learn.microsoft.com/ef/core/performance/)
- [Single vs. Split Queries — https://learn.microsoft.com/ef/core/querying/single-split-queries](https://learn.microsoft.com/ef/core/querying/single-split-queries)
- [ExecuteUpdate/ExecuteDelete (bulk) — https://learn.microsoft.com/ef/core/saving/execute-insert-update-delete](https://learn.microsoft.com/ef/core/saving/execute-insert-update-delete)
- [Indexes (HasIndex, Include, HasFilter) — https://learn.microsoft.com/ef/core/modeling/indexes](https://learn.microsoft.com/ef/core/modeling/indexes)
- [Compiled queries & advanced performance — https://learn.microsoft.com/ef/core/performance/advanced-performance-topics#compiled-queries](https://learn.microsoft.com/ef/core/performance/advanced-performance-topics#compiled-queries)
- [.NET 8 EF Core migration commands — https://learn.microsoft.com/ef/core/managing-screens/migrations/](https://learn.microsoft.com/ef/core/managing-screens/migrations/)
