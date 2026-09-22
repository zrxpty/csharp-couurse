---
[← К уроку M08-L09](lesson-M08-L09-iqueryable-vs-ienumerable.md) | [⬆ К модулю M08](../README.md) | [Следующее ДЗ →]()
---

### Домашнее задание M08-L09: IEnumerable vs IQueryable, провайдеры / Homework M08-L09: IEnumerable vs IQueryable, providers

**Урок / Lesson:** M08-L09
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться осознанно выбирать между `IEnumerable<T>` и `IQueryable<T>`, понимать, где и как выполняется каждый оператор LINQ, фиксить «раннюю материализацию», переводить деревья выражений в SQL и диагностировать непереводимые выражения и N+1. (EN) Learn to choose deliberately between `IEnumerable<T>` and `IQueryable<T>`, understand where and how each LINQ operator executes, fix early materialization, read expression-tree-to-SQL translation, and diagnose non-translatable expressions and N+1.

#### Связь с уроком / Connection to the lesson

(RU) Урок вводит фундаментальное различие: `IEnumerable<T>` работает с `Func` (готовым кодом, выполняемым в памяти), а `IQueryable<T>` работает с `Expression` (данными, которые провайдер переводит в язык источника — SQL, JSON и т. п.). ДЗ требует построить мини-приложение с EF Core и SQLite, в котором одни и те же запросы написаны сначала «плохо» (с ранней материализацией и непереводимыми выражениями), а потом «хорошо» — и сравнить сгенерированный SQL и объём перебираемых данных.

(EN) The lesson introduces the fundamental split: `IEnumerable<T>` works with `Func` (compiled code executed in memory), while `IQueryable<T>` works with `Expression` (data that a provider translates into the source language — SQL, JSON, etc.). The homework asks you to build a small EF Core + SQLite app where the same queries are written first "badly" (with early materialization and non-translatable expressions) and then "well" — and to compare the generated SQL and the volume of data scanned.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Представьте, что вы работаете в команде, которая обслуживает библиотечный сервис «Книжная полка». В базе хранятся читатели (`Readers`), книги (`Books`) и записи о выдаче книг (`Loans`). Таблицы небольшие в тестах, но в проде `Loans` содержит миллионы строк, и любой запрос, который тащит всю таблицу в память приложения, мгновенно кладёт сервер. Команда замечает две категории проблем: (1) некоторые коллеги по привычке пишут `.AsEnumerable()` или `.ToList()` сразу после `DbSet`, после чего все фильтры, сортировки и проекции бесшумно уходят в LINQ to Objects; (2) другие коллеги вызывают внутри `Where` собственные методы вроде `r => r.IsReliable()`, которые EF Core не умеет переводить в SQL, и получают `InvalidOperationException` в рантайме. Ваша задача — построить лабораторную среду, в которой можно воспроизвести обе проблемы, измерить их стоимость (число строк, объём SQL, число round-trips) и переписать запросы так, чтобы они работали «по-правильному»: держали `IQueryable` до конца, переносили фильтры и проекции в SQL и материализовывали минимальный набор полей. Эта работа имитирует реальный code review, где вы должны объяснить коллеге не только *что* плохо, но и *почему* плохо с точки зрения механики `Func` vs `Expression` и работы провайдера `IQueryProvider`.

#### Что нужно сделать (пошагово)

1. **Создайте проект.** В папке `M08/L09/hw` выполните `dotnet new console -o Bookshelf -n Bookshelf --framework net8.0`. Перейдите в каталог: `cd Bookshelf`. Добавьте пакеты: `dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 8.*` и `dotnet add package Microsoft.EntityFrameworkCore.Design --version 8.*`. Убедитесь, что `dotnet build` проходит без ошибок.

2. **Опишите модель.** В `Program.cs` (top-level statements) объявите три `record` с primary-конструкторами: `Reader(int Id, string Name, int Age, string City)`, `Book(int Id, string Title, string Author, int Pages, string Genre)`, `Loan(int Id, int ReaderId, int BookId, DateTime TakenAt, DateTime? ReturnedAt)`. Используйте `file record`, если не хотите засорять глобальное пространство имён. Создайте `BookshelfContext : DbContext` с тремя `DbSet<T>` и настройкой `UseSqlite("Data Source=bookshelf.db")`.

3. **Засейте базу.** В `OnModelCreating` через `HasData` добавьте 6 читателей, 8 книг и 12 записей о выдаче (часть с `ReturnedAt == null` — невозвращенные). Запустите `dotnet ef migrations add Init` и `dotnet ef database update`, либо вызовите `db.Database.EnsureDeleted(); db.Database.EnsureCreated();` в `Main` для прототипа (это проще, но менее реалистично — в отчёте укажите разницу).

4. **Запрос A — «плохая» версия (IEnumerable через ранний ToList).** Напишите метод `QueryBadToList()`, который делает: `db.Loans.ToList().Where(l => l.ReturnedAt == null).OrderBy(l => l.TakenAt).ToList()`. Сначала выведите число строк, которое вернул `ToList()` на первой строке (через отдельный замер `db.Loans.Count()`), потом результат. Включите логирование SQL: `optionsBuilder.LogTo(Console.WriteLine, new[] { DbLoggerCategory.Database.Command.Name }).EnableSensitiveDataLogging();`. Зафиксируйте в отчёт `README.md`: какой SQL выполнился, сколько строк было материализовано на клиенте.

5. **Запрос A — «хорошая» версия.** Напишите `QueryGood()`: `db.Loans.Where(l => l.ReturnedAt == null).OrderBy(l => l.TakenAt).Select(l => new { l.Id, l.TakenAt, l.BookId, l.ReaderId }).ToList()`. Выведите SQL через `db.Loans.Where(...).Select(...).ToQueryString()`. Сравните: в «плохой» версии сервер вернул ВСЕ строки `Loans`, в «хорошей» — только невозвращенные и только нужные колонки.

6. **Запрос B — ловушка `AsEnumerable` посередине.** Напишите `QueryAsEnumerableTrap()`: `db.Readers.AsEnumerable().Where(r => r.Age >= 18).OrderBy(r => r.City).Select(r => r.Name).ToList()`. Покажите в логе, что первая команда SQL — это `SELECT "Id", "Name", "Age", "City" FROM "Readers"` (вся таблица), а фильтрация и сортировка происходят уже в памяти. Перепишите как `QueryAsEnumerableFixed()` и сравните SQL.

7. **Запрос C — непереводимое выражение.** Добавьте метод расширения `public static bool IsAdult(this Reader r) => r.Age >= 18;` и напишите `db.Readers.Where(r => r.IsAdult()).ToList()`. Должно упасть с `InvalidOperationException` о том, что клиентское выражение не может быть переведено. Перепишите как `db.Readers.Where(r => r.Age >= 18)` — теперь переводится. Зафиксируйте точный текст исключения и объясните в отчёте, *почему* провайдер не может перевести вызов метода: дерево выражений содержит узел `MethodCallExpression`, для которого нет SQL-эквивалента.

8. **Запрос D — N+1.** Напишите цикл `foreach (var loan in db.Loans.Where(l => l.ReturnedAt == null)) { var book = db.Books.First(b => b.Id == loan.BookId); Console.WriteLine(...); }`. Включите логирование и подсчитайте число SQL-запросов: должно быть 1 + N. Перепишите через `Include` (если настроите навигационные свойства) или через `Join`/`Select` с DTO, чтобы стало 1 запрос. Зафиксируйте разницу в round-trips.

9. **Замеры.** Для каждого «плохого» и «хорошего» варианта запишите в отчёт: число SQL-запросов, объём SQL в символах, число материализованных строк на клиенте, число колонок в проекции. Сведите в таблицу.

10. **Эссе-объяснение.** В `README.md` напишите 200–300 слов о том, что такое «золотая диаграмма» урока (`Func` → код, который выполняется; `Expression` → данные, которые переводятся) и как она объясняет все четыре проблемы выше. Не копируйте текст урока дословно — переформулируйте своими словами и приведите свои примеры.

#### Требования к решению

- Проект собирается командой `dotnet build` без warning-ов уровня error и без ошибок на .NET 8 / C# 12.
- Используется top-level statements в `Program.cs`; модель — `record` с primary-конструкторами; коллекции при необходимости инициализируются collection expressions (`[a, b, c]`).
- Включено логирование SQL через `LogTo` и `EnableSensitiveDataLogging` только в Debug-конфигурации (используйте `#if DEBUG` или `optionsBuilder` проверку `IsConfigured`).
- Каждый «плохой» запрос сопровождается комментарием RU+EN с пометкой `// BAD: ...` и ссылкой на конкретную ошибку из списка «Частые ошибки» урока.
- Каждый «хороший» запрос сопровождается комментарием `// GOOD: ...` с указанием, какая часть ушла в SQL (`WHERE`, `ORDER BY`, `SELECT`-проекция).
- Для непереводимого выражения приведён точный текст исключения в отчёте.
- Для N+1 приведён подсчёт round-trips «до» и «после».
- В отчёте `README.md` есть таблица сравнения всех четырёх пар «плохо/хорошо».
- Код запускается детерминированно: база пересоздаётся при старте (`EnsureCreated` или `EnsureDeleted + EnsureCreated`), чтобы проверяющий получил тот же вывод.
- Запрещено использовать `IEnumerable<T>` как тип переменной для строящегося запроса к `DbSet` (это одна из частых ошибок урока) — используйте `IQueryable<T>` явно или `var`.

#### Тонкости и подводные камни

- **Компиляция лямбды зависит от типа параметра метода, а не от синтаксиса.** `r => r.Age >= 18` превращается в `Func<Reader,bool>` для `IEnumerable.Where`, и в `Expression<Func<Reader,bool>>` для `IQueryable.Where`. Это одна из главных причин, почему «всё работает одинаково» — пока не посмотришь на SQL. В ДЗ убедитесь, что вы можете предсказать, какой тип получится.
- **`AsEnumerable()` — это «граница перевода».** Всё, что после неё, идёт в LINQ to Objects. Частая ошибка — поставить её рано «чтобы было проще отлаживать». В ДЗ специально покажите, как один лишний `AsEnumerable()` меняет SQL с `WHERE Age >= 18` на `SELECT * FROM Readers` + фильтр в памяти.
- **`ToList()` в середине — то же самое, что ранний `AsEnumerable`, но ещё и материализует.** Запрос `db.Loans.ToList().Where(...)` не «оптимизируется» компилятором — он честно вытянет всю таблицу. Различайте `IQueryable<T>` (черновик, который можно достроить) и материализованный `List<T>` (готовый результат).
- **Не каждое C#-выражение переводится.** Вызов собственного метода, обращение к свойству, которого нет в маппинге, использование `DateTime.Now` в `Where` (иногда), сложные вычисления с `Aggregate` — типичные кандидаты на `InvalidOperationException`. Если логика действительно нужна — либо выразите её через примитивы (`r.Age >= 18` вместо `r.IsAdult()`), либо материализуйте *сознательно* и фильтруйте в памяти, написав комментарий, почему так.
- **N+1 маскируется под «всё работает».** Цикл `foreach` по `IQueryable` сам по себе не медленный — он медленный, потому что на каждой итерации дёргает ещё один запрос. Включите логирование и считайте строки `SELECT` в выводе. Лечится `Include` (для навигационных свойств) или явным `Join`/`Select` в DTO.
- **`var` может «съесть» тип.** Если написать `var q = db.Users.Where(...)` — `q` будет `IQueryable<User>`, всё хорошо. Но `IEnumerable<User> q = db.Users.Where(...)` бесшумно переключит тип, и следующий `.Where` уже пойдёт в память. Будьте внимательны к типу переменной, а не только к цепочке методов.
- **`ToQueryString()` — без выполнения.** `db.Users.Where(u => u.Age >= 18).ToQueryString()` показывает SQL, не обращаясь к базе. Это безопасный способ проверить перевод до того, как запрос пойдёт в прод. В ДЗ используйте его для всех «хороших» версий.
- **`EnableSensitiveDataLogging()` опасен в проде.** Он выводит значения параметров в лог. Используйте только в Debug. В ДЗ отметьте это комментарием.

#### Критерии приёмки

- [ ] Проект `Bookshelf` собирается и запускается на .NET 8 / C# 12 через `dotnet run`.
- [ ] Модель содержит три `record` с primary-конструкторами и `BookshelfContext : DbContext` с тремя `DbSet`.
- [ ] База засеяна через `HasData` или `EnsureCreated` с детерминированными данными (один и тот же вывод при повторном запуске).
- [ ] Логирование SQL включено только в Debug и выводит команды в консоль.
- [ ] Запрос A «плохо»: ранний `ToList()` тянет всю таблицу `Loans`; в отчёте зафиксировано число строк.
- [ ] Запрос A «хорошо»: `WHERE` и `ORDER BY` ушли в SQL; `ToQueryString()` подтверждает.
- [ ] Запрос B «плохо»: `AsEnumerable()` до фильтра → SQL `SELECT * FROM Readers`; в отчёте отмечено.
- [ ] Запрос B «хорошо»: фильтр в SQL; число возвращённых строк меньше.
- [ ] Запрос C: непереводимое выражение `r.IsAdult()` падает с `InvalidOperationException`; точный текст исключения в отчёте.
- [ ] Запрос C «хорошо»: переписано через примитивы `r.Age >= 18`; SQL содержит `WHERE`.
- [ ] Запрос D «плохо»: N+1 — в логе видно 1 + N запросов; подсчёт round-trips в отчёте.
- [ ] Запрос D «хорошо»: один запрос через `Join`/`Include`/`Select`-DTO.
- [ ] В отчёте `README.md` есть таблица сравнения всех четырёх пар «плохо/хорошо» (SQL, строки, round-trips, колонки).
- [ ] В отчёте есть эссе 200–300 слов о «золотой диаграмме» `Func` vs `Expression`.
- [ ] Каждый «плохой» фрагмент помечен комментарием `// BAD:` с указанием ошибки из урока.
- [ ] Код не использует `IEnumerable<T>` как тип для строящегося запроса к `DbSet`.

#### Подсказки (без прямого ответа)

- Чтобы увидеть, что лямбда компилируется в `Expression`, наведите в отладчике на параметр `Where` — тип будет `Expression<Func<...>>`, а не `Func<...>`.
- Для подсчёта round-trips проще всего считать строки лога, начинающиеся с `Executed DbCommand` или `SELECT`.
- `ToQueryString()` доступен начиная с EF Core 5.0; в 8.0 работает на любой `IQueryable` от провайдера БД.
- Если `Include` не работает — значит, вы не объявили навигационные свойства. Для этого ДЗ достаточно `Join` в `Select`-DTO.
- Чтобы «плохой» запрос A не падал по `OutOfMemory` на больших данных, в тестах данных мало — но в отчёте опишите, что было бы на миллионе строк.
- Помните: `IEnumerable` для коллекций в памяти — это нормально и правильно. Проблема только тогда, когда `IEnumerable` появляется *после* `DbSet`.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — Эталонное решение ДЗ M08-L09
// Reference solution for homework M08-L09
// Top-level statements, EF Core 8, SQLite.

using System.Collections.Generic;
using System.Linq;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;

// === Модель данных (file record не засоряет глобальное пространство) ===
// === Data model (file record keeps the global namespace clean) ===
file record Reader(int Id, string Name, int Age, string City);
file record Book(int Id, string Title, string Author, int Pages, string Genre);
file record Loan(int Id, int ReaderId, int BookId, DateTime TakenAt, DateTime? ReturnedAt);

// === Контекст EF Core ===
// === EF Core context ===
file class BookshelfContext : DbContext
{
    public DbSet<Reader> Readers => Set<Reader>();
    public DbSet<Book> Books => Set<Book>();
    public DbSet<Loan> Loans => Set<Loan>();

    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        options.UseSqlite("Data Source=bookshelf.db");

        // Логирование SQL только в Debug — в проде EnableSensitiveDataLogging опасен.
        // SQL logging only in Debug — EnableSensitiveDataLogging is dangerous in prod.
#if DEBUG
        options
            .LogTo(msg => Console.WriteLine("  [SQL] " + msg),
                   new[] { DbLoggerCategory.Database.Command.Name })
            .EnableSensitiveDataLogging()
            .EnableDetailedErrors();
#endif
    }

    protected override void OnModelCreating(ModelBuilder mb)
    {
        // Детерминированный сид — одинаковый вывод при каждом запуске.
        // Deterministic seed — same output on every run.
        mb.Entity<Reader>().HasData(
            new Reader(1, "Анна", 17, "Москва"),
            new Reader(2, "Борис", 25, "Москва"),
            new Reader(3, "Вера", 30, "Тверь"),
            new Reader(4, "Глеб", 15, "Тверь"),
            new Reader(5, "Дина", 40, "Казань"),
            new Reader(6, "Егор", 22, "Казань"));

        mb.Entity<Book>().HasData(
            new Book(1, "CLR via C#", "Рихтер", 800, "tech"),
            new Book(2, "C# in Depth", "Скит", 500, "tech"),
            new Book(3, "Грокаем алгоритмы", "Бхаргава", 256, "tech"),
            new Book(4, "Чистый код", "Мартин", 464, "tech"),
            new Book(5, "1984", "Оруэлл", 320, "fiction"),
            new Book(6, "Собачье сердце", "Булгаков", 350, "fiction"),
            new Book(7, "Мастер и Маргарита", "Булгаков", 480, "fiction"),
            new Book(8, "SICP", "Абельсон", 657, "tech"));

        var t = new DateTime(2024, 1, 15);
        mb.Entity<Loan>().HasData(
            new Loan(1, 1, 1, t.AddDays(-30), t.AddDays(-10)),
            new Loan(2, 1, 5, t.AddDays(-20), null),          // невозвращена
            new Loan(3, 2, 2, t.AddDays(-15), null),          // невозвращена
            new Loan(4, 2, 6, t.AddDays(-5), null),           // невозвращена
            new Loan(5, 3, 3, t.AddDays(-25), t.AddDays(-1)),
            new Loan(6, 3, 7, t.AddDays(-3), null),           // невозвращена
            new Loan(7, 4, 4, t.AddDays(-2), null),           // невозвращена
            new Loan(8, 5, 8, t.AddDays(-40), t.AddDays(-12)),
            new Loan(9, 5, 1, t.AddDays(-7), null),           // невозвращена
            new Loan(10, 6, 5, t.AddDays(-1), null),          // невозвращена
            new Loan(11, 6, 6, t.AddDays(-60), t.AddDays(-50)),
            new Loan(12, 6, 7, t.AddDays(-4), null));         // невозвращена
    }
}

// Метод расширения, который НЕ переводится в SQL (демонстрация ошибки C).
// Extension method that does NOT translate to SQL (mistake C demo).
file static class ReaderQueryExtensions
{
    public static bool IsAdult(this Reader r) => r.Age >= 18;
}

// === Запрос A — «плохо»: ранний ToList тянет ВСЮ таблицу ===
// === Query A — "bad": early ToList pulls the WHOLE table ===
static void QueryBadToList(BookshelfContext db)
{
    Console.WriteLine(">> A BAD: db.Loans.ToList().Where(...)");
    // BAD: ToList() в начале → материализуем ВСЕ строки Loans, потом фильтруем в памяти.
    // BAD: ToList() at the start → materialize ALL Loans rows, then filter in memory.
    var allRowsCount = db.Loans.Count(); // для отчёта / for the report
    var result = db.Loans
        .ToList()                                  // ← всё, дальше IEnumerable
        .Where(l => l.ReturnedAt == null)
        .OrderBy(l => l.TakenAt)
        .ToList();
    Console.WriteLine($"  materialized {allRowsCount} rows from DB, kept {result.Count}");
}

// === Запрос A — «хорошо»: фильтр и сортировка в SQL, минимальная проекция ===
// === Query A — "good": filter and ordering in SQL, minimal projection ===
static void QueryGood(BookshelfContext db)
{
    Console.WriteLine(">> A GOOD: db.Loans.Where(...).OrderBy(...).Select(...)");
    IQueryable<Loan> baseQ = db.Loans;             // явно IQueryable / explicitly IQueryable
    var built = baseQ
        .Where(l => l.ReturnedAt == null)          // → SQL WHERE
        .OrderBy(l => l.TakenAt)                   // → SQL ORDER BY
        .Select(l => new { l.Id, l.TakenAt, l.BookId, l.ReaderId }); // → SQL SELECT (projection)

    Console.WriteLine("  SQL:");
    Console.WriteLine("  " + built.ToQueryString());

    var result = built.ToList();
    Console.WriteLine($"  returned {result.Count} rows from DB");
}

// === Запрос B — ловушка AsEnumerable посередине ===
// === Query B — the AsEnumerable-in-the-middle trap ===
static void QueryAsEnumerableTrap(BookshelfContext db)
{
    Console.WriteLine(">> B BAD: db.Readers.AsEnumerable().Where(...)");
    // BAD: AsEnumerable() переключает на LINQ to Objects до фильтра.
    // BAD: AsEnumerable() switches to LINQ to Objects before the filter.
    var bad = db.Readers
        .AsEnumerable()
        .Where(r => r.Age >= 18)
        .OrderBy(r => r.City)
        .Select(r => r.Name)
        .ToList();
    Console.WriteLine($"  pulled all {db.Readers.Count()} readers, kept {bad.Count}");
}

static void QueryAsEnumerableFixed(BookshelfContext db)
{
    Console.WriteLine(">> B GOOD: db.Readers.Where(...).OrderBy(...).Select(...)");
    var good = db.Readers
        .Where(r => r.Age >= 18)         // → SQL WHERE
        .OrderBy(r => r.City)            // → SQL ORDER BY
        .Select(r => r.Name)             // → SQL SELECT (только Name)
        .ToList();
    Console.WriteLine($"  SQL: {db.Readers.Where(r => r.Age >= 18).OrderBy(r => r.City).Select(r => r.Name).ToQueryString()}");
    Console.WriteLine($"  returned {good.Count} rows");
}

// === Запрос C — непереводимое выражение ===
// === Query C — non-translatable expression ===
static void QueryNonTranslatable(BookshelfContext db)
{
    Console.WriteLine(">> C BAD: db.Readers.Where(r => r.IsAdult())");
    try
    {
        // Провайдер не умеет переводить вызов метода IsAdult → InvalidOperationException.
        // The provider cannot translate the IsAdult method call → InvalidOperationException.
        var bad = db.Readers.Where(r => r.IsAdult()).ToList();
        Console.WriteLine($"  unexpected success: {bad.Count}");
    }
    catch (InvalidOperationException ex)
    {
        Console.WriteLine("  Exception (expected): " + ex.Message.Split('\n')[0]);
    }
}

static void QueryTranslatable(BookshelfContext db)
{
    Console.WriteLine(">> C GOOD: db.Readers.Where(r => r.Age >= 18)");
    var good = db.Readers.Where(r => r.Age >= 18);
    Console.WriteLine("  SQL: " + good.ToQueryString());
    Console.WriteLine($"  returned {good.ToList().Count} rows");
}

// === Запрос D — N+1 ===
// === Query D — N+1 ===
static void QueryNPlusOne(BookshelfContext db)
{
    Console.WriteLine(">> D BAD: foreach over IQueryable + inner First()");
    // BAD: foreach по IQueryable + подзапрос First на каждой итерации → 1 + N запросов.
    // BAD: foreach over IQueryable + inner First per iteration → 1 + N queries.
    foreach (var loan in db.Loans.Where(l => l.ReturnedAt == null))
    {
        var book = db.Books.First(b => b.Id == loan.BookId);
        Console.WriteLine($"  loan #{loan.Id} → '{book.Title}'");
    }
}

static void QuerySingleRoundTrip(BookshelfContext db)
{
    Console.WriteLine(">> D GOOD: single query via Join + DTO projection");
    // GOOD: один запрос через Join + проекция в DTO.
    // GOOD: single query via Join + DTO projection.
    var good = from l in db.Loans
               where l.ReturnedAt == null
               join b in db.Books on l.BookId equals b.Id
               orderby l.TakenAt
               select new { l.Id, BookTitle = b.Title, l.TakenAt };

    Console.WriteLine("  SQL: " + good.ToQueryString());
    foreach (var x in good.ToList())
        Console.WriteLine($"  loan #{x.Id} → '{x.BookTitle}' @ {x.TakenAt:d}");
}

// === Точка входа (top-level) ===
// === Entry point (top-level) ===
using (var db = new BookshelfContext())
{
    db.Database.EnsureDeleted();   // детерминированный старт / deterministic start
    db.Database.EnsureCreated();

    QueryBadToList(db);
    QueryGood(db);
    QueryAsEnumerableTrap(db);
    QueryAsEnumerableFixed(db);
    QueryNonTranslatable(db);
    QueryTranslatable(db);
    QueryNPlusOne(db);
    QuerySingleRoundTrip(db);
}
```

**Разбор по строкам.** Строки с `file record Reader/Book/Loan` используют C# 12: `file` ограничивает видимость типом файла, а primary-конструктор заменяет явные свойства — ровно то, что демонстрирует урок для современного компактного кода. В `OnConfiguring` `UseSqlite` — это и есть подключение провайдера EF Core; именно он реализует `IQueryProvider`, который переводит `Expression` в SQL. Блок `#if DEBUG ... #endif` вокруг `LogTo` — лучшая практика из урока: проверять сгенерированный SQL полезно, но `EnableSensitiveDataLogging` в проде опасен, поэтому ограничиваем Debug-конфигурацией. Сид через `HasData` даёт детерминированный набор данных — это важно для воспроизводимости замеров и для проверяющего.

`QueryBadToList` — эталон «плохой» версии запроса A: `ToList()` материализует ВСЮ таблицу `Loans`, после чего переменная имеет тип `List<Loan>` (то есть `IEnumerable<Loan>`), и все последующие операторы уходят в LINQ to Objects. Это иллюстрирует пункт «Частые ошибки» урока про ранний `ToList`. `QueryGood`, напротив, объявляет переменную как `IQueryable<Loan>` (не `IEnumerable`!), последовательно навешивает `Where`, `OrderBy`, `Select` — и только в конце вызывает `ToList()`. Строка `built.ToQueryString()` демонстрирует приём из «Best Practices»: посмотреть SQL до выполнения. Проекция `Select` в анонимный тип уменьшает число колонок — это второй пункт Best Practices.

`QueryAsEnumerableTrap` показывает, что `AsEnumerable()` — это граница перевода: после неё компилятор видит `IEnumerable<Reader>`, и `Where` принимает `Func`, а не `Expression`. `QueryAsEnumerableFixed` переносит фильтр и сортировку до `AsEnumerable`/`ToList`, поэтому `WHERE` и `ORDER BY` уходят в SQL. `QueryNonTranslatable` намеренно использует метод расширения `IsAdult` внутри `Where` — дерево выражений содержит `MethodCallExpression`, у которого нет SQL-эквивалента, поэтому EF Core бросает `InvalidOperationException` (это типичная ошибка из урока). `QueryTranslatable` заменяет вызов метода на примитивное сравнение `r.Age >= 18`, которое переводится.

`QueryNPlusOne` — классический N+1: `foreach` по `IQueryable` и `db.Books.First` внутри итерации порождают 1 + N запросов. Лечение в `QuerySingleRoundTrip` через `join` (query syntax) и проекцию в DTO: один SQL-запрос с `JOIN` и `WHERE`. Здесь применены концепции урока о «держать IQueryable до конца» и «избегать N+1 через `Include`/`Join`». Точка входа пересоздаёт базу через `EnsureDeleted + EnsureCreated`, что даёт детерминированный старт и упрощает проверку.

#### Задания на углубление (бонус)

1. **Своя обёртка-логгер.** Напишите класс, который оборачивает `IQueryable<T>` и при каждом материализаторе (`ToList`, `FirstOrDefault`) пишет в лог время, число строк и объём SQL. Подсказка: `IQueryProvider` можно перехватить через `DbCommandSourceInterceptor` или через декоратор над `IQueryable`.
2. **Expression-визитор.** С помощью `ExpressionVisitor` пройдитесь по дереву выражения `db.Readers.Where(r => r.Age >= 18 && r.City == "Москва")` и распечатайте типы узлов (`NodeType`). Это поможет увидеть, что `Expression` — это действительно данные, а не код.
3. **In-memory провайдер.** Подключите `Microsoft.EntityFrameworkCore.InMemory` и сравните, как ведутся запросы с ним vs с SQLite: `ToQueryString` для InMemory вернёт пустую строку — объясните почему.
4. **AsSplitQuery.** Если в `QuerySingleRoundTrip` вы добавите `Include` для нескольких коллекций, возникнет «декартово произведение». Примените `AsSplitQuery()` и сравните число SQL-запросов и объём данных.

---

## Statement in English / Постановка на английском

#### Context & motivation

Imagine you work on a team maintaining a library service called "Bookshelf". The database stores readers (`Readers`), books (`Books`), and loan records (`Loans`). In tests the tables are tiny, but in production `Loans` holds millions of rows, and any query that drags the whole table into application memory instantly kills the server. The team keeps hitting two families of problems: (1) some colleagues, out of habit, write `.AsEnumerable()` or `.ToList()` right after a `DbSet`, after which every filter, ordering, and projection silently falls back to LINQ to Objects; (2) other colleagues call their own helper methods inside `Where`, such as `r => r.IsReliable()`, which EF Core cannot translate to SQL, and they get a runtime `InvalidOperationException`. Your job is to build a lab environment in which you can reproduce both problems, measure their cost (number of rows, SQL size, number of round-trips), and then rewrite the queries the "right" way: keep `IQueryable` to the very end, push filters and projections into SQL, and materialize a minimal set of columns. This work simulates a real code review where you must explain to a teammate not only *what* is wrong, but *why* it is wrong in terms of the `Func` vs `Expression` mechanics and the role of `IQueryProvider`.

#### What to do step by step

1. **Create the project.** Inside `M08/L09/hw` run `dotnet new console -o Bookshelf -n Bookshelf --framework net8.0`. Move into the folder with `cd Bookshelf`. Add the packages: `dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 8.*` and `dotnet add package Microsoft.EntityFrameworkCore.Design --version 8.*`. Make sure `dotnet build` succeeds with no errors.

2. **Describe the model.** In `Program.cs` (top-level statements) declare three `record` types with primary constructors: `Reader(int Id, string Name, int Age, string City)`, `Book(int Id, string Title, string Author, int Pages, string Genre)`, `Loan(int Id, int ReaderId, int BookId, DateTime TakenAt, DateTime? ReturnedAt)`. Use `file record` if you want to avoid polluting the global namespace. Create `BookshelfContext : DbContext` with three `DbSet<T>` properties and `UseSqlite("Data Source=bookshelf.db")`.

3. **Seed the database.** In `OnModelCreating` use `HasData` to insert 6 readers, 8 books, and 12 loan records (some with `ReturnedAt == null`, meaning not returned). Run `dotnet ef migrations add Init` and `dotnet ef database update`, or call `db.Database.EnsureDeleted(); db.Database.EnsureCreated();` in `Main` for a prototype (simpler, but less realistic — note the difference in your report).

4. **Query A — "bad" version (IEnumerable via early ToList).** Write a method `QueryBadToList()` that does: `db.Loans.ToList().Where(l => l.ReturnedAt == null).OrderBy(l => l.TakenAt).ToList()`. First print the number of rows the first `ToList()` returned (via a separate `db.Loans.Count()`), then the result. Turn on SQL logging: `optionsBuilder.LogTo(Console.WriteLine, new[] { DbLoggerCategory.Database.Command.Name }).EnableSensitiveDataLogging();`. Capture in `README.md`: which SQL ran, how many rows were materialized on the client.

5. **Query A — "good" version.** Write `QueryGood()`: `db.Loans.Where(l => l.ReturnedAt == null).OrderBy(l => l.TakenAt).Select(l => new { l.Id, l.TakenAt, l.BookId, l.ReaderId }).ToList()`. Print the SQL via `db.Loans.Where(...).Select(...).ToQueryString()`. Compare: in the "bad" version the server returned ALL rows of `Loans`; in the "good" version only the unreturned ones and only the needed columns.

6. **Query B — the `AsEnumerable` trap in the middle.** Write `QueryAsEnumerableTrap()`: `db.Readers.AsEnumerable().Where(r => r.Age >= 18).OrderBy(r => r.City).Select(r => r.Name).ToList()`. Show in the log that the first SQL command is `SELECT "Id", "Name", "Age", "City" FROM "Readers"` (the whole table), while filtering and ordering happen in memory afterward. Rewrite it as `QueryAsEnumerableFixed()` and compare the SQL.

7. **Query C — a non-translatable expression.** Add an extension method `public static bool IsAdult(this Reader r) => r.Age >= 18;` and write `db.Readers.Where(r => r.IsAdult()).ToList()`. It should throw an `InvalidOperationException` saying a client expression cannot be translated. Rewrite as `db.Readers.Where(r => r.Age >= 18)` — now it translates. Capture the exact exception message in your report and explain *why* the provider cannot translate a method call: the expression tree contains a `MethodCallExpression` node for which there is no SQL equivalent.

8. **Query D — N+1.** Write a loop `foreach (var loan in db.Loans.Where(l => l.ReturnedAt == null)) { var book = db.Books.First(b => b.Id == loan.BookId); Console.WriteLine(...); }`. Turn on logging and count the SQL statements: it should be 1 + N. Rewrite with `Include` (if you wire up navigation properties) or with a `Join`/`Select` into a DTO so that it becomes a single query. Capture the difference in round-trips.

9. **Measurements.** For each "bad" and "good" variant record in the report: number of SQL statements, SQL size in characters, number of rows materialized on the client, number of columns in the projection. Put everything into a table.

10. **Explanatory essay.** In `README.md` write 200–300 words about the "golden diagram" from the lesson (`Func` → code that executes; `Expression` → data that gets translated) and how it explains all four problems above. Do not copy the lesson text verbatim — rephrase in your own words and bring your own examples.

#### Requirements

- The project builds with `dotnet build` with no error-level warnings and no errors on .NET 8 / C# 12.
- Top-level statements are used in `Program.cs`; the model uses `record` with primary constructors; where collections are needed, use collection expressions (`[a, b, c]`).
- SQL logging is on, but only in the Debug configuration, via `#if DEBUG` or an `IsConfigured` check inside `optionsBuilder`.
- Every "bad" query has a RU+EN comment with a `// BAD: ...` tag and a reference to a specific mistake from the lesson's "Common Mistakes" list.
- Every "good" query has a `// GOOD: ...` comment naming which part went to SQL (`WHERE`, `ORDER BY`, `SELECT` projection).
- For the non-translatable expression, the exact exception text is captured in the report.
- For N+1, the round-trip count "before" and "after" is captured.
- The `README.md` report contains a comparison table for all four "bad/good" pairs.
- The code runs deterministically: the database is recreated at startup (`EnsureCreated` or `EnsureDeleted + EnsureCreated`) so the checker gets the same output.
- It is forbidden to use `IEnumerable<T>` as the type of a variable that is building a query against a `DbSet` (this is one of the lesson's common mistakes) — use `IQueryable<T>` explicitly or `var`.

#### Pitfalls

- **Lambda compilation depends on the parameter type of the method, not on the syntax.** `r => r.Age >= 18` becomes `Func<Reader,bool>` for `IEnumerable.Where` and `Expression<Func<Reader,bool>>` for `IQueryable.Where`. This is one of the main reasons everything "looks the same" — until you look at the SQL. In this homework, make sure you can predict which type you will get.
- **`AsEnumerable()` is the "translation border".** Everything after it goes to LINQ to Objects. A common mistake is to put it early "for easier debugging". In the homework, deliberately show how a single extra `AsEnumerable()` turns SQL with `WHERE Age >= 18` into `SELECT * FROM Readers` plus an in-memory filter.
- **`ToList()` in the middle is the same as an early `AsEnumerable`, but it also materializes.** The query `db.Loans.ToList().Where(...)` is not "optimized" by the compiler — it honestly pulls the entire table. Distinguish `IQueryable<T>` (a draft you can extend) from a materialized `List<T>` (a finished result).
- **Not every C# expression translates.** Calling your own method, touching a property that is not mapped, using `DateTime.Now` in `Where` (sometimes), or complex `Aggregate` computations are typical candidates for `InvalidOperationException`. If the logic is genuinely needed — either express it with primitives (`r.Age >= 18` instead of `r.IsAdult()`), or materialize *on purpose* and filter in memory, with a comment explaining why.
- **N+1 disguises itself as "everything works".** A `foreach` over an `IQueryable` is not slow by itself — it is slow because on every iteration it triggers another query. Turn on logging and count the `SELECT` lines in the output. The fix is `Include` (for navigation properties) or an explicit `Join`/`Select` into a DTO.
- **`var` can "swallow" the type.** If you write `var q = db.Users.Where(...)` — `q` is `IQueryable<User>`, fine. But `IEnumerable<User> q = db.Users.Where(...)` silently switches the type, and the next `.Where` runs in memory. Pay attention to the variable type, not only to the method chain.
- **`ToQueryString()` runs without execution.** `db.Users.Where(u => u.Age >= 18).ToQueryString()` shows the SQL without touching the database. It is a safe way to check the translation before the query goes to production. In the homework, use it for all "good" variants.
- **`EnableSensitiveDataLogging()` is dangerous in production.** It logs parameter values. Use it only in Debug. In the homework, mark this with a comment.

#### Acceptance criteria

- [ ] The `Bookshelf` project builds and runs on .NET 8 / C# 12 via `dotnet run`.
- [ ] The model contains three `record` types with primary constructors and a `BookshelfContext : DbContext` with three `DbSet`s.
- [ ] The database is seeded via `HasData` or `EnsureCreated` with deterministic data (same output on every run).
- [ ] SQL logging is on, but only in Debug, and prints commands to the console.
- [ ] Query A "bad": early `ToList()` pulls the entire `Loans` table; the row count is captured in the report.
- [ ] Query A "good": `WHERE` and `ORDER BY` went to SQL; `ToQueryString()` confirms it.
- [ ] Query B "bad": `AsEnumerable()` before the filter → SQL `SELECT * FROM Readers`; noted in the report.
- [ ] Query B "good": filter in SQL; fewer rows returned.
- [ ] Query C: the non-translatable expression `r.IsAdult()` throws `InvalidOperationException`; the exact text is in the report.
- [ ] Query C "good": rewritten with primitives `r.Age >= 18`; SQL contains `WHERE`.
- [ ] Query D "bad": N+1 — the log shows 1 + N queries; round-trip count in the report.
- [ ] Query D "good": a single query via `Join`/`Include`/`Select`-DTO.
- [ ] The `README.md` report has a comparison table for all four "bad/good" pairs (SQL, rows, round-trips, columns).
- [ ] The report has a 200–300 word essay on the `Func` vs `Expression` "golden diagram".
- [ ] Every "bad" fragment is tagged with a `// BAD:` comment pointing at the lesson mistake.
- [ ] The code never uses `IEnumerable<T>` as the type for a query being built against a `DbSet`.

#### Hints (no direct answer)

- To see that the lambda compiles to `Expression`, hover over the `Where` parameter in the debugger — the type will be `Expression<Func<...>>`, not `Func<...>`.
- The easiest way to count round-trips is to count log lines starting with `Executed DbCommand` or `SELECT`.
- `ToQueryString()` is available from EF Core 5.0; in 8.0 it works on any `IQueryable` coming from a database provider.
- If `Include` does not work — you have not declared navigation properties. For this homework a `Join` inside a `Select`-DTO is enough.
- To keep the "bad" Query A from `OutOfMemory` on large data, the test data is small — but in the report describe what would happen on a million rows.
- Remember: `IEnumerable` for in-memory collections is normal and correct. The problem only starts when `IEnumerable` appears *after* a `DbSet`.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — Reference solution for homework M08-L09
using System.Collections.Generic;
using System.Linq;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;

// === Data model (file record keeps the global namespace clean) ===
file record Reader(int Id, string Name, int Age, string City);
file record Book(int Id, string Title, string Author, int Pages, string Genre);
file record Loan(int Id, int ReaderId, int BookId, DateTime TakenAt, DateTime? ReturnedAt);

// === EF Core context ===
file class BookshelfContext : DbContext
{
    public DbSet<Reader> Readers => Set<Reader>();
    public DbSet<Book> Books => Set<Book>();
    public DbSet<Loan> Loans => Set<Loan>();

    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        options.UseSqlite("Data Source=bookshelf.db");

        // SQL logging only in Debug — EnableSensitiveDataLogging is dangerous in prod.
#if DEBUG
        options
            .LogTo(msg => Console.WriteLine("  [SQL] " + msg),
                   new[] { DbLoggerCategory.Database.Command.Name })
            .EnableSensitiveDataLogging()
            .EnableDetailedErrors();
#endif
    }

    protected override void OnModelCreating(ModelBuilder mb)
    {
        // Deterministic seed — same output on every run.
        mb.Entity<Reader>().HasData(
            new Reader(1, "Anna", 17, "Moscow"),
            new Reader(2, "Boris", 25, "Moscow"),
            new Reader(3, "Vera", 30, "Tver"),
            new Reader(4, "Gleb", 15, "Tver"),
            new Reader(5, "Dina", 40, "Kazan"),
            new Reader(6, "Egor", 22, "Kazan"));

        mb.Entity<Book>().HasData(
            new Book(1, "CLR via C#", "Richter", 800, "tech"),
            new Book(2, "C# in Depth", "Skeet", 500, "tech"),
            new Book(3, "Grokking Algorithms", "Bhargava", 256, "tech"),
            new Book(4, "Clean Code", "Martin", 464, "tech"),
            new Book(5, "1984", "Orwell", 320, "fiction"),
            new Book(6, "Heart of a Dog", "Bulgakov", 350, "fiction"),
            new Book(7, "Master and Margarita", "Bulgakov", 480, "fiction"),
            new Book(8, "SICP", "Abelson", 657, "tech"));

        var t = new DateTime(2024, 1, 15);
        mb.Entity<Loan>().HasData(
            new Loan(1, 1, 1, t.AddDays(-30), t.AddDays(-10)),
            new Loan(2, 1, 5, t.AddDays(-20), null),          // not returned
            new Loan(3, 2, 2, t.AddDays(-15), null),          // not returned
            new Loan(4, 2, 6, t.AddDays(-5), null),           // not returned
            new Loan(5, 3, 3, t.AddDays(-25), t.AddDays(-1)),
            new Loan(6, 3, 7, t.AddDays(-3), null),           // not returned
            new Loan(7, 4, 4, t.AddDays(-2), null),           // not returned
            new Loan(8, 5, 8, t.AddDays(-40), t.AddDays(-12)),
            new Loan(9, 5, 1, t.AddDays(-7), null),           // not returned
            new Loan(10, 6, 5, t.AddDays(-1), null),          // not returned
            new Loan(11, 6, 6, t.AddDays(-60), t.AddDays(-50)),
            new Loan(12, 6, 7, t.AddDays(-4), null));         // not returned
    }
}

// Extension method that does NOT translate to SQL (mistake C demo).
file static class ReaderQueryExtensions
{
    public static bool IsAdult(this Reader r) => r.Age >= 18;
}

// === Query A — "bad": early ToList pulls the WHOLE table ===
static void QueryBadToList(BookshelfContext db)
{
    Console.WriteLine(">> A BAD: db.Loans.ToList().Where(...)");
    // BAD: ToList() at the start → materialize ALL Loans rows, then filter in memory.
    var allRowsCount = db.Loans.Count(); // for the report
    var result = db.Loans
        .ToList()                                  // ← from here it is IEnumerable
        .Where(l => l.ReturnedAt == null)
        .OrderBy(l => l.TakenAt)
        .ToList();
    Console.WriteLine($"  materialized {allRowsCount} rows from DB, kept {result.Count}");
}

// === Query A — "good": filter and ordering in SQL, minimal projection ===
static void QueryGood(BookshelfContext db)
{
    Console.WriteLine(">> A GOOD: db.Loans.Where(...).OrderBy(...).Select(...)");
    IQueryable<Loan> baseQ = db.Loans;             // explicitly IQueryable
    var built = baseQ
        .Where(l => l.ReturnedAt == null)          // → SQL WHERE
        .OrderBy(l => l.TakenAt)                   // → SQL ORDER BY
        .Select(l => new { l.Id, l.TakenAt, l.BookId, l.ReaderId }); // → SQL SELECT (projection)

    Console.WriteLine("  SQL:");
    Console.WriteLine("  " + built.ToQueryString());

    var result = built.ToList();
    Console.WriteLine($"  returned {result.Count} rows from DB");
}

// === Query B — the AsEnumerable-in-the-middle trap ===
static void QueryAsEnumerableTrap(BookshelfContext db)
{
    Console.WriteLine(">> B BAD: db.Readers.AsEnumerable().Where(...)");
    // BAD: AsEnumerable() switches to LINQ to Objects before the filter.
    var bad = db.Readers
        .AsEnumerable()
        .Where(r => r.Age >= 18)
        .OrderBy(r => r.City)
        .Select(r => r.Name)
        .ToList();
    Console.WriteLine($"  pulled all {db.Readers.Count()} readers, kept {bad.Count}");
}

static void QueryAsEnumerableFixed(BookshelfContext db)
{
    Console.WriteLine(">> B GOOD: db.Readers.Where(...).OrderBy(...).Select(...)");
    var good = db.Readers
        .Where(r => r.Age >= 18)         // → SQL WHERE
        .OrderBy(r => r.City)            // → SQL ORDER BY
        .Select(r => r.Name)             // → SQL SELECT (only Name)
        .ToList();
    Console.WriteLine($"  SQL: {db.Readers.Where(r => r.Age >= 18).OrderBy(r => r.City).Select(r => r.Name).ToQueryString()}");
    Console.WriteLine($"  returned {good.Count} rows");
}

// === Query C — non-translatable expression ===
static void QueryNonTranslatable(BookshelfContext db)
{
    Console.WriteLine(">> C BAD: db.Readers.Where(r => r.IsAdult())");
    try
    {
        // The provider cannot translate the IsAdult method call → InvalidOperationException.
        var bad = db.Readers.Where(r => r.IsAdult()).ToList();
        Console.WriteLine($"  unexpected success: {bad.Count}");
    }
    catch (InvalidOperationException ex)
    {
        Console.WriteLine("  Exception (expected): " + ex.Message.Split('\n')[0]);
    }
}

static void QueryTranslatable(BookshelfContext db)
{
    Console.WriteLine(">> C GOOD: db.Readers.Where(r => r.Age >= 18)");
    var good = db.Readers.Where(r => r.Age >= 18);
    Console.WriteLine("  SQL: " + good.ToQueryString());
    Console.WriteLine($"  returned {good.ToList().Count} rows");
}

// === Query D — N+1 ===
static void QueryNPlusOne(BookshelfContext db)
{
    Console.WriteLine(">> D BAD: foreach over IQueryable + inner First()");
    // BAD: foreach over IQueryable + inner First per iteration → 1 + N queries.
    foreach (var loan in db.Loans.Where(l => l.ReturnedAt == null))
    {
        var book = db.Books.First(b => b.Id == loan.BookId);
        Console.WriteLine($"  loan #{loan.Id} → '{book.Title}'");
    }
}

static void QuerySingleRoundTrip(BookshelfContext db)
{
    Console.WriteLine(">> D GOOD: single query via Join + DTO projection");
    // GOOD: single query via Join + DTO projection.
    var good = from l in db.Loans
               where l.ReturnedAt == null
               join b in db.Books on l.BookId equals b.Id
               orderby l.TakenAt
               select new { l.Id, BookTitle = b.Title, l.TakenAt };

    Console.WriteLine("  SQL: " + good.ToQueryString());
    foreach (var x in good.ToList())
        Console.WriteLine($"  loan #{x.Id} → '{x.BookTitle}' @ {x.TakenAt:d}");
}

// === Entry point (top-level) ===
using (var db = new BookshelfContext())
{
    db.Database.EnsureDeleted();   // deterministic start
    db.Database.EnsureCreated();

    QueryBadToList(db);
    QueryGood(db);
    QueryAsEnumerableTrap(db);
    QueryAsEnumerableFixed(db);
    QueryNonTranslatable(db);
    QueryTranslatable(db);
    QueryNPlusOne(db);
    QuerySingleRoundTrip(db);
}
```

**Line-by-line walk-through.** The `file record Reader/Book/Loan` lines use C# 12: `file` limits visibility to the file, and the primary constructor replaces explicit properties — exactly what the lesson demonstrates for modern, compact code. In `OnConfiguring`, `UseSqlite` is how you plug in the EF Core provider; that provider implements `IQueryProvider`, which is the thing that translates the `Expression` into SQL. The `#if DEBUG ... #endif` block around `LogTo` reflects a lesson best practice: inspecting the generated SQL is useful, but `EnableSensitiveDataLogging` is dangerous in production, so we confine it to the Debug configuration. The seed via `HasData` gives a deterministic dataset — important for reproducible measurements and for the checker.

`QueryBadToList` is the reference "bad" Query A: `ToList()` materializes the ENTIRE `Loans` table, after which the variable has type `List<Loan>` (that is, `IEnumerable<Loan>`), and all subsequent operators go to LINQ to Objects. This illustrates the "Common Mistakes" point about an early `ToList`. `QueryGood`, in contrast, declares the variable as `IQueryable<Loan>` (not `IEnumerable`!), chains `Where`, `OrderBy`, and `Select`, and only calls `ToList()` at the very end. The `built.ToQueryString()` line demonstrates a technique from the "Best Practices": look at the SQL before you execute. The `Select` projection into an anonymous type reduces the number of columns — the second Best Practice.

`QueryAsEnumerableTrap` shows that `AsEnumerable()` is the translation border: after it, the compiler sees `IEnumerable<Reader>`, and `Where` takes a `Func`, not an `Expression`. `QueryAsEnumerableFixed` moves the filter and ordering before `AsEnumerable`/`ToList`, so `WHERE` and `ORDER BY` travel into SQL. `QueryNonTranslatable` deliberately uses the `IsAdult` extension method inside `Where` — the expression tree contains a `MethodCallExpression` with no SQL equivalent, so EF Core throws `InvalidOperationException` (a classic mistake from the lesson). `QueryTranslatable` replaces the method call with a primitive comparison `r.Age >= 18`, which translates.

`QueryNPlusOne` is the classic N+1: a `foreach` over an `IQueryable` plus `db.Books.First` inside each iteration produces 1 + N queries. The fix in `QuerySingleRoundTrip` is a `join` (query syntax) with a DTO projection: one SQL query with `JOIN` and `WHERE`. Here the lesson concepts "keep IQueryable to the end" and "avoid N+1 with `Include`/`Join`" are applied. The entry point recreates the database with `EnsureDeleted + EnsureCreated`, giving a deterministic start and simplifying the check.

#### Going deeper (bonus)

1. **Custom logging wrapper.** Write a class that wraps `IQueryable<T>` and on every materializer (`ToList`, `FirstOrDefault`) logs the time, the row count, and the SQL size. Hint: `IQueryProvider` can be intercepted via `DbCommandSourceInterceptor` or via a decorator over `IQueryable`.
2. **Expression visitor.** With an `ExpressionVisitor`, walk the expression tree of `db.Readers.Where(r => r.Age >= 18 && r.City == "Moscow")` and print the `NodeType` of each node. This will help you see that `Expression` is genuinely data, not code.
3. **In-memory provider.** Plug in `Microsoft.EntityFrameworkCore.InMemory` and compare how queries behave against it versus SQLite: `ToQueryString` for InMemory returns an empty string — explain why.
4. **AsSplitQuery.** If you add `Include` for several collections to `QuerySingleRoundTrip`, a "cartesian product" appears. Apply `AsSplitQuery()` and compare the number of SQL statements and the data volume.

---

#### Чек-лист сдачи / Submission checklist

- [ ] (RU) Проект `Bookshelf` собран и запущен через `dotnet run` на .NET 8 / C# 12.
- [ ] (RU) В `README.md` есть таблица сравнения четырёх пар «плохо/хорошо».
- [ ] (RU) В отчёте приведён точный текст исключения для непереводимого выражения.
- [ ] (RU) В отчёте зафиксировано число round-trips для N+1 «до» и «после».
- [ ] (RU) Эссе о «золотой диаграмме» `Func` vs `Expression` на 200–300 слов.
- [ ] (RU) Логирование SQL включено только в Debug.
- [ ] (EN) The `Bookshelf` project builds and runs via `dotnet run` on .NET 8 / C# 12.
- [ ] (EN) `README.md` contains the comparison table for the four "bad/good" pairs.
- [ ] (EN) The exact exception text for the non-translatable expression is in the report.
- [ ] (EN) The N+1 round-trip count "before" and "after" is captured in the report.
- [ ] (EN) The 200–300 word essay on the `Func` vs `Expression` "golden diagram" is present.
- [ ] (EN) SQL logging is enabled only in Debug.

#### Ресурсы / Resources

- [Microsoft Learn — IQueryable&lt;T&gt;](https://learn.microsoft.com/dotnet/api/system.linq.iqueryable-1)
- [Microsoft Learn — IEnumerable&lt;T&gt;](https://learn.microsoft.com/dotnet/api/system.collections.generic.ienumerable-1)
- [Microsoft Learn — EF Core logging](https://learn.microsoft.com/ef/core/logging-events-diagnostics/)
- [Microsoft Learn — ToQueryString](https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.entityframeworkqueryableextensions.toquerystring)
- [Microsoft Learn — Expression Trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/)

---

[⬆ К модулю M08](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
