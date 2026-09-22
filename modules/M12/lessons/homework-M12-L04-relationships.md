---
[← К уроку M12-L04](lesson-M12-L04-relationships.md) | [⬆ К модулю M12](../README.md) | [Следующее ДЗ →](homework-M12-L05-linq-to-entities.md)
---

### Домашнее задание M12-L04: Отношения: 1:N, N:N, 1:1, навигационные свойства / Homework M12-L04: Relationships: 1:N, N:N, 1:1, navigation properties

**Урок / Lesson:** M12-L04
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться проектировать и явно конфигурировать отношения 1:N, N:N (в том числе с payload в join-таблице) и 1:1 в EF Core 8 на C# 12, управлять каскадным удалением, избегать циклов каскадов и грамотно загружать связанные данные через `Include`. (EN) Learn to design and explicitly configure 1:N, N:N (including a join table with payload), and 1:1 relationships in EF Core 8 on C# 12, control cascade delete, avoid cascade cycles, and load related data correctly with `Include`.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит три типа отношений через один домен «библиотека»: автор→книга (1:N), книга↔жанр (N:N), книга→профиль (1:1). В этом ДЗ вы расширяете тот же домен до цифровой библиотеки BookHive, добавляя необязательную связь к издателю, join-сущность с payload и две однотипные конфигурации `OnDelete`, чтобы на практике столкнуться с nullable FK, `SetNull` и потенциальным циклом каскадов. Вы используете ровно те же приёмы Fluent API (`HasMany/WithOne`, `HasOne/WithOne/HasForeignKey<>()`, составной `HasKey`), что и в коде урока.
(EN) The lesson introduces three relationship types through a single "library" domain: author→book (1:N), book↔genre (N:N), and book→profile (1:1). In this homework you extend that same domain into the BookHive digital library, adding an optional link to a publisher, a join entity with payload, and two `OnDelete` configurations so you can meet nullable FKs, `SetNull`, and a potential cascade cycle in practice. You use exactly the same Fluent API techniques (`HasMany/WithOne`, `HasOne/WithOne/HasForeignKey<>()`, composite `HasKey`) as in the lesson code.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы контрактный разработчик для стартапа BookHive — каталога электронной библиотеки. Команда уже спроектировала домен на уровне классов, но на ревью выяснилось, что модель «работает по конвенции», а это значит: поведение при удалении непредсказуемо, join-таблица жанров не хранит дату присвоения, а профиль книги то появляется, то пропадает из-за путаницы между principal и dependent. Продакт-менеджер сформулировал требования предметной области, а не таблиц: у книги обязательно есть автор, но издатель может быть неизвестен; жанры присваиваются администратором и важно знать, когда; профиль книги единственен и не может существовать без самой книги; при удалении автора должны исчезнуть его книги, при удалении издателя — книги остаются, но ссылка обнуляется.

Ваша задача — превратить «работающее по конвенции» приложение в явно сконфигурированную модель EF Core 8, в которой каждое отношение описано Fluent API, каждое поведение `OnDelete` задано осознанно, а загрузка связанных данных выполняется жадно через `Include` без скрытых N+1-запросов. Заодно вы обеспечите детерминированность миграций: если кто-то переименует навигационное свойство, схема БД не должна «поехать». Это типичный сценарий роста проекта, когда конвенции перестают справляться и команда переходит к явной спецификации модели.

#### Что нужно сделать (пошагово)
1. Создайте новый проект `BookHive` консольного типа на .NET 8 и установите пакеты `Microsoft.EntityFrameworkCore.Sqlite` (8.0.x) и `Microsoft.EntityFrameworkCore.Design` (8.0.x). Команды:
   - `dotnet new console -n BookHive -o BookHive -f net8.0`
   - `cd BookHive`
   - `dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 8.0.*`
   - `dotnet add package Microsoft.EntityFrameworkCore.Design --version 8.0.*`
2. Определите сущности: `Author` (Id, Name, Country, `ICollection<Book> Books`), `Publisher` (Id, Name, `ICollection<Book> Books`), `Book` (Id, Title, Year, AuthorId, Author, PublisherId?, Publisher, `ICollection<Genre> Genres` (skip-навигация), `ICollection<BookGenre> BookGenres`, BookProfile? Profile), `Genre` (Id, Name, `ICollection<Book> Books`, `ICollection<BookGenre> BookGenres`), `BookGenre` (BookId, Book, GenreId, Genre, AssignedAt, AssignedBy), `BookProfile` (Id, Description, PageCount, BookId, Book). Инициализируйте все навигационные коллекции в инициализаторах (`= new List<T>()`), а обязательные ссылочные навигации — `= null!`, как в уроке.
3. Реализуйте `BookHiveDbContext : DbContext` с `DbSet`-ами для каждой сущности. В `OnConfiguring` подключите SQLite к файлу `bookhive.db` (или используйте `OnModelCreating` + `DbContextOptions` — на ваш выбор). В `OnModelCreating` явно опишите все четыре отношения через Fluent API:
   - 1:N обязательный `Author → Book` через `HasMany(a => a.Books).WithOne(b => b.Author).HasForeignKey(b => b.AuthorId).OnDelete(DeleteBehavior.Cascade)`;
   - 1:N необязательный `Publisher → Book` через `HasMany(p => p.Books).WithOne(b => b.Publisher).HasForeignKey(b => b.PublisherId).OnDelete(DeleteBehavior.SetNull)` (FK nullable);
   - N:N с payload `Book ↔ Genre` через join-сущность `BookGenre`: составной `HasKey(bg => new { bg.BookId, bg.GenreId })` и два `HasOne(...).WithMany(...).HasForeignKey(...)`;
   - 1:1 `Book → BookProfile` через `HasOne(b => b.Profile).WithOne(p => p.Book).HasForeignKey<BookProfile>(p => p.BookId).OnDelete(DeleteBehavior.Cascade)`.
4. Создайте миграцию: `dotnet ef migrations add InitialRelations`. Откройте сгенерированный файл и проверьте, что в `Books` есть `AuthorId` (non-null) и `PublisherId` (nullable), в `BookGenres` — составной PK и две FK, а у `BookProfiles` колонка `BookId` помечена уникальным индексом. Примените: `dotnet ef database update`.
5. В `Program.cs` (top-level statements) постройте seed: одного автора с двумя книгами, одного издателя, три жанра, два `BookGenre` с разными `AssignedAt` и по одному `BookProfile` на каждую книгу. Сохраните через один `SaveChangesAsync` — EF должен проставить все FK по навигациям.
6. Напишите три запроса и выведите результаты в консоль:
   - жадная загрузка книги со всем графом: `db.Books.Include(b => b.Author).Include(b => b.Publisher).Include(b => b.Profile).Include(b => b.Genres).FirstAsync(...)`;
   - удаление издателя и проверка, что у книг `PublisherId` стал `null` (демонстрация `SetNull`);
   - удаление автора и проверка, что его книги исчезли вместе с профилями и связями жанров (демонстрация каскада по нескольким уровням).
7. Запустите `dotnet run` и зафиксируйте вывод. Ожидается: список книг с автором/издателем/профилем/жанрами; после шага 6 — книги без издателя; после шага 7 — пустой список книг удалённого автора.

#### Требования к решению
- Проект компилируется на .NET 8 / C# 12 без предупреждений уровня error. Top-level statements в `Program.cs`; включите `<Nullable>enable</Nullable>` и `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` (или хотя бы `nullable enable`).
- Все четыре отношения описаны явно в `OnModelCreating` через Fluent API. Конвенции не должны молча определять `OnDelete` — каждое поведение прописано вручную.
- Навигационные коллекции инициализированы (`new List<T>()`), ссылочные обязательные навигации помечены `null!`. Запрещено оставлять неинициализированные `ICollection<>`.
- Скалярные FK-свойства присутствуют явно (`AuthorId`, `PublisherId`, `BookId`), а не «только навигация» — это требование из раздела «Частые ошибки» урока.
- Миграция `InitialRelations` успешно создаётся и применяется к SQLite без ошибок цикла каскадов. Если EF сообщит о цикле — объясните в комментарии, какую связь вы переключили на `Restrict` и почему.
- Запросы используют `Include` (и `ThenInclude` при необходимости), а не ленивую загрузку. Ленивая загрузка должна быть выключена по умолчанию.
- Seed детерминирован: повторный запуск даёт ту же картину. Используйте `EnsureDeletedAsync`/`EnsureCreatedAsync` или пересоздавайте БД через миграции.
- Код сопровождается комментариями RU+EN в ключевых местах конфигурации отношений (по образцу урока).

#### Тонкости и подводные камни
- **Цикл каскадов.** В нашей модели оба «обязательных» каскада (`Author→Book` и `Book→BookProfile`) идут по одному пути, но не образуют цикл, потому что направление однонаправленное вниз. Однако если вы добавите обратную обязательную связь с каскадом (например, `Book → Author` как principal с обязательным FK), EF выбросит `InvalidOperationException` при миграции. Решение из урока: переключите одну из связей на `Restrict`/`NoAction` или сделайте FK nullable. Если в вашей реализации возникнет ошибка — это не баг, а ожидаемое поведение; consciously выберите, какую связь «ослабить».
- **`SetNull` требует nullable FK.** Колонка `PublisherId` обязана быть `int?`. Если оставить `int` (non-nullable), EF при миграции либо не даст применить `SetNull` (на уровне БД SQLite это превратится в `NO ACTION`), либо удаление упадёт с исключением. Это прямая иллюстрация ошибки «FK non-nullable but OnDelete = SetNull» из урока.
- **N:N с payload нельзя «упрощённо» через skip-навигацию.** Раз `BookGenre` хранит `AssignedAt` и `AssignedBy`, вы обязаны описать join-сущность явно: составной `HasKey`, два `HasOne/WithMany/HasForeignKey`. При этом у `Book` и `Genre` можно держать **обе** коллекции — и skip-навигацию `Genres`/`Books`, и коллекцию join-сущностей `BookGenres`. Урок показывает вариант без skip-навигации (`WithMany()` без параметра); вы можете выбрать любой, но в seed и запросах используйте именно ту, что настроили.
- **`WithOne`/`WithMany` указывайте всегда.** Пропуск обратной навигации заставляет EF «угадывать» направление и может развернуть отношение. В 1:1 обязательно `HasOne(b => b.Profile).WithOne(p => p.Book)` — обе стороны названы.
- **Principal vs dependent в 1:1.** `BookProfile` — dependent (он хранит FK `BookId`). `HasForeignKey<BookProfile>(p => p.BookId)` указывает EF, какая сторона зависимая. Если перепутать, EF может попытаться добавить FK в `Books`, что сломает семантику 1:1.
- **Инициализация коллекций.** `Genres = { new Genre {...} }` в инициализаторе работает только если коллекция уже создана. Урок инициализирует `new List<Book>()` — сделайте так же, иначе `NullReferenceException` при добавлении до `SaveChanges`.
- **Ленивая загрузка по умолчанию выключена** в EF Core, если вы не установите `Microsoft.EntityFrameworkCore.Proxies` и не включите `UseLazyLoadingProxies()`. Не включайте её — используйте `Include`, как требует best practice урока.

#### Критерии приёмки
- [ ] Проект `BookHive` на .NET 8 / C# 12 собирается без ошибок и предупреждений.
- [ ] Определены все шесть сущностей с корректными навигациями и скалярными FK.
- [ ] Навигационные коллекции инициализированы `new List<T>()`, обязательные ссылочные навигации — `null!`.
- [ ] 1:N `Author → Book` настроен `HasMany().WithOne().HasForeignKey().OnDelete(Cascade)`.
- [ ] 1:N необязательный `Publisher → Book` настроен с nullable FK и `OnDelete(SetNull)`.
- [ ] N:N с payload настроен через явный `BookGenre` с составным `HasKey` и двумя FK.
- [ ] 1:1 `Book → BookProfile` настроен `HasOne().WithOne().HasForeignKey<BookProfile>()`.
- [ ] Миграция `InitialRelations` создаётся и применяется без ошибок (включая отсутствие цикла каскадов).
- [ ] В сгенерированной миграции `BookProfiles.BookId` помечен уникальным индексом.
- [ ] Seed создаёт автора с двумя книгами, издателя, три жанра, два `BookGenre` и два `BookProfile`.
- [ ] Жадная загрузка через `Include` выводит полный граф книги без N+1.
- [ ] Удаление издателя обнуляет `PublisherId` у книг (демонстрация `SetNull`).
- [ ] Удаление автора каскадно удаляет книги, профили и связи жанров.
- [ ] Ленивая загрузка не включена; `Include` используется явно.
- [ ] Ключевые места конфигурации прокомментированы RU+EN.
- [ ] В `README` или комментариях объяснён выбор `OnDelete` для каждой связи.

#### Подсказки (без прямого ответа)
- Если EF ругается на цикл каскадов при миграции, посмотрите, какие две обязательные связи образуют замкнутый путь, и спросите себя: какая из них семантически допускает `Restrict` или nullable FK?
- Чтобы проверить уникальность 1:1, попробуйте в тесте создать два `BookProfile` с одним `BookId` — EF/БД должны отклонить.
- Для жадной загрузки жанров через join-сущность используйте `Include(b => b.BookGenres).ThenInclude(bg => bg.Genre)`; для skip-навигации — просто `Include(b => b.Genres)`.
- `EnsureDeleted`/`EnsureCreated` удобнее для демо, но миграции — требование задания; не подменяйте одно другим.
- Чтобы увидеть SQL, добавьте `optionsBuilder.LogTo(Console.WriteLine)` — это поможет убедиться, что `Include` порождает один JOIN, а не десятки запросов.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 / EF Core 8 — BookHive: 1:N, N:N (payload), 1:1
// Полный рабочий пример конфигурации отношений по сценарию ДЗ.

using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

// --- Сущности / Entities ---

public class Author            // Principal в 1:N / principal in 1:N
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Country { get; set; } = string.Empty;
    public ICollection<Book> Books { get; set; } = new List<Book>();
}

public class Publisher         // Principal в необязательном 1:N / principal in optional 1:N
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public ICollection<Book> Books { get; set; } = new List<Book>();
}

public class Book
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public int Year { get; set; }

    // Обязательный FK к Author / required FK to Author
    public int AuthorId { get; set; }
    public Author Author { get; set; } = null!;

    // Необязательный FK к Publisher (nullable → SetNull допустим)
    // Optional FK to Publisher (nullable → SetNull is valid)
    public int? PublisherId { get; set; }
    public Publisher? Publisher { get; set; }

    // N:N skip-навигация / skip navigation
    public ICollection<Genre> Genres { get; set; } = new List<Genre>();

    // N:N через join-сущность с payload / N:N via join entity with payload
    public ICollection<BookGenre> BookGenres { get; set; } = new List<BookGenre>();

    // 1:1 principal-сторона / 1:1 principal side
    public BookProfile? Profile { get; set; }
}

public class Genre
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public ICollection<Book> Books { get; set; } = new List<Book>();
    public ICollection<BookGenre> BookGenres { get; set; } = new List<BookGenre>();
}

// Join-сущность с payload / join entity with payload
public class BookGenre
{
    public int BookId { get; set; }
    public Book Book { get; set; } = null!;

    public int GenreId { get; set; }
    public Genre Genre { get; set; } = null!;

    public DateTime AssignedAt { get; set; }   // payload
    public string AssignedBy { get; set; } = string.Empty; // payload
}

// 1:1 dependent / зависимая сторона
public class BookProfile
{
    public int Id { get; set; }
    public string Description { get; set; } = string.Empty;
    public int PageCount { get; set; }

    // FK = ссылка на Book; уникальный индекс гарантирует 1:1
    // FK to Book; unique index guarantees 1:1
    public int BookId { get; set; }
    public Book Book { get; set; } = null!;
}

// --- DbContext ---

public class BookHiveDbContext : DbContext
{
    public DbSet<Author> Authors => Set<Author>();
    public DbSet<Publisher> Publishers => Set<Publisher>();
    public DbSet<Book> Books => Set<Book>();
    public DbSet<Genre> Genres => Set<Genre>();
    public DbSet<BookGenre> BookGenres => Set<BookGenre>();
    public DbSet<BookProfile> Profiles => Set<BookProfile>();

    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options
            .UseSqlite("Data Source=bookhive.db")
            .LogTo(Console.WriteLine, LogLevel.Information); // SQL в консоль / SQL to console

    protected override void OnModelCreating(ModelBuilder mb)
    {
        // 1:N обязательный: Author -> Book (Cascade)
        // required 1:N: cascade deletes books with author
        mb.Entity<Author>()
            .HasMany(a => a.Books)
            .WithOne(b => b.Author)
            .HasForeignKey(b => b.AuthorId)
            .OnDelete(DeleteBehavior.Cascade);

        // 1:N необязательный: Publisher -> Book (SetNull)
        // optional 1:N: nulls out PublisherId when publisher deleted
        mb.Entity<Publisher>()
            .HasMany(p => p.Books)
            .WithOne(b => b.Publisher)
            .HasForeignKey(b => b.PublisherId)
            .OnDelete(DeleteBehavior.SetNull);

        // N:N с payload: составной PK + два FK
        // N:N with payload: composite PK + two FKs
        mb.Entity<BookGenre>()
            .HasKey(bg => new { bg.BookId, bg.GenreId });

        mb.Entity<BookGenre>()
            .HasOne(bg => bg.Book)
            .WithMany(b => b.BookGenres)
            .HasForeignKey(bg => bg.BookId)
            .OnDelete(DeleteBehavior.Cascade);

        mb.Entity<BookGenre>()
            .HasOne(bg => bg.Genre)
            .WithMany(g => g.BookGenres)
            .HasForeignKey(bg => bg.GenreId)
            .OnDelete(DeleteBehavior.Cascade);

        // 1:1: Book (principal) -> BookProfile (dependent)
        // FK BookId в dependent; уникальный индекс обеспечит единственность
        mb.Entity<Book>()
            .HasOne(b => b.Profile)
            .WithOne(p => p.Book)
            .HasForeignKey<BookProfile>(p => p.BookId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}

// --- Program (top-level statements) ---

using var db = new BookHiveDbContext();
await db.Database.EnsureDeletedAsync();     // чистый старт / clean start
await db.Database.MigrateAsync();           // применяем миграции / apply migrations

// Seed / наполнение
var author = new Author { Name = "Лев Толстой", Country = "Россия" };
var publisher = new Publisher { Name = "Альпина" };
var classic = new Genre { Name = "Классика" };
var epic = new Genre { Name = "Эпос" };
var novel = new Genre { Name = "Роман" };

var book1 = new Book
{
    Title = "Война и мир", Year = 1869, Author = author, Publisher = publisher,
    Profile = new BookProfile { Description = "Эпопея", PageCount = 1225 }
};
var book2 = new Book
{
    Title = "Анна Каренина", Year = 1877, Author = author, Publisher = publisher,
    Profile = new BookProfile { Description = "Роман", PageCount = 864 }
};

// N:N через join-сущность с payload / N:N via join entity with payload
book1.BookGenres.Add(new BookGenre { Genre = classic, AssignedAt = DateTime.UtcNow, AssignedBy = "admin" });
book1.BookGenres.Add(new BookGenre { Genre = epic,     AssignedAt = DateTime.UtcNow, AssignedBy = "admin" });
book2.BookGenres.Add(new BookGenre { Genre = novel,    AssignedAt = DateTime.UtcNow, AssignedBy = "admin" });

db.Books.AddRange(book1, book2);
await db.SaveChangesAsync();

// 1) Жадная загрузка полного графа / eager load full graph
var loaded = await db.Books
    .Include(b => b.Author)
    .Include(b => b.Publisher)
    .Include(b => b.Profile)
    .Include(b => b.BookGenres).ThenInclude(bg => bg.Genre)
    .ToListAsync();

foreach (var b in loaded)
    Console.WriteLine($"{b.Title} ({b.Year}) — {b.Author.Name} — издатель: {b.Publisher?.Name ?? "—"} — профиль: {b.Profile?.Description} — жанров: {b.BookGenres.Count}");

// 2) Удаление издателя → SetNull / delete publisher → SetNull
var pub = await db.Publishers.FirstAsync();
db.Publishers.Remove(pub);
await db.SaveChangesAsync();

var afterPub = await db.Books.AsNoTracking().ToListAsync();
Console.WriteLine($"Книг без издателя: {afterPub.Count(b => b.PublisherId is null)}");

// 3) Удаление автора → каскад по уровням / delete author → multi-level cascade
var aut = await db.Authors.FirstAsync();
db.Authors.Remove(aut);
await db.SaveChangesAsync();

var remaining = await db.Books.AsNoTracking().CountAsync();
Console.WriteLine($"Книг осталось: {remaining}");
```

**Разбор по строкам.** Класс `Author` — principal сторона обязательного 1:N: у него коллекция `Books`, инициализированная `new List<Book>()` (best practice урока: инициализируйте коллекции). `Book.AuthorId` — скалярный FK (требование «не только навигация»), а `Book.Author` помечен `null!`, потому что после загрузки из БД ссылка обязательно будет проставлена, и компилятор не должен тревожить нас nullable-предупреждениями. `PublisherId` сделан `int?` — это ключевой момент: именно nullable-тип делает законным `OnDelete(DeleteBehavior.SetNull)`. Если бы колонка была non-nullable, SQLite тихо превратил бы поведение в `NO ACTION`, и удаление издателя упало бы с исключением — это и есть ошибка «FK non-nullable but OnDelete = SetNull» из урока.

Join-сущность `BookGenre` хранит payload (`AssignedAt`, `AssignedBy`), поэтому N:N нельзя реализовать «упрощённо» только skip-навигацией: нужен явный класс. Составной `HasKey(bg => new { bg.BookId, bg.GenreId })` формирует первичный ключ join-таблицы, а два `HasOne/WithMany/HasForeignKey` описывают две связи 1:N, сходящиеся в join-сущности. Заметьте, что `WithMany(b => b.BookGenres)` указывает обратную навигацию явно — это защищает от «угадывания» направления (частая ошибка «забыт `WithOne`/`WithMany`»). Одновременно у `Book` осталась skip-навигация `Genres`; в нашем разборе мы грузим жанры через `BookGenres` + `ThenInclude`, чтобы показать работу с payload, но при желании можно было бы настроить `UsingEntity` и обращаться к `Genres` напрямую.

Связь 1:1 `Book → BookProfile` описана строго по шаблону урока: `HasOne(b => b.Profile).WithOne(p => p.Book).HasForeignKey<BookProfile>(p => p.BookId)`. Параметр-тип `<BookProfile>` у `HasForeignKey` указывает EF, какая сторона dependent (несёт FK), а `BookId` становится и FK, и (благодаря уникальному индексу, который EF создаёт автоматически) гарантом единственности профиля на книгу. `OnDelete(Cascade)` означает: удаляем книгу — профиль исчезает. Все каскады в модели (`Author→Book`, `Book→BookProfile`, `Book→BookGenre`, `Genre→BookGenre`) однонаправленные вниз, поэтому цикла не возникает — именно такой ацикличный граф и рекомендует урок.

В `OnConfiguring` включён `LogTo` для вывода SQL — это наглядно доказывает, что `Include` + `ThenInclude` порождает один запрос с JOIN, а не N+1. Top-level statements в `Program.cs` соответствуют C# 12 / .NET 8; `EnsureDeletedAsync` + `MigrateAsync` дают чистый старт и применяют миграции. Удаление издателя (шаг 2) демонстрирует `SetNull` — после него `PublisherId` у книг становится `null`, а книги остаются. Удаление автора (шаг 3) запускает каскад на нескольких уровнях: исчезают книги → профили → записи `BookGenre`. `AsNoTracking` в проверочных запросах — best practice для read-only: меньше накладных расходов и чище SQL.

#### Задания на углубление (бонус)
1. Добавьте второе 1:1 отношение `Book → BookCover`, но на этот раз используйте **общий первичный ключ** (`BookCover.Id` одновременно и PK, и FK к `Book`), настроив `HasOne().WithOne().HasForeignKey<BookCover>(c => c.Id)`. Сравните две конфигурации 1:1 (отдельный FK + уникальный индекс vs. shared PK) с точки зрения миграций и семантики.
2. Добавьте **N:N без payload** — `Book ↔ Tag` через skip-навигации на обеих сторонах без явного join-класса. Убедитесь, что EF сам создаст join-таблицу `BookTag`. Сравните SQL, который порождает skip-навигация, с SQL для `BookGenre` с payload.
3. Намеренно спровоцируйте **цикл каскадов**: добавьте обратную обязательную связь `Book → Author` (например, `PrimaryAuthorId`) с `Cascade`. Зафиксируйте `InvalidOperationException` при `migrations add`, затем исправьте модель, переключив одну из связей на `Restrict` или сделав FK nullable. Опишите в комментарии, почему выбрали именно этот вариант.
4. Перепишите запрос жадной загрузки как **проекцию (DTO)** через `Select`, чтобы вернуть только нужные поля (`Title`, `Author.Name`, `Publisher?.Name`, список жанров). Сравните количество и форму SQL-запросов с версией на `Include`.

---

## Statement in English / Постановка на английском

#### Context & motivation
You are a contract developer for the BookHive startup, a catalogue for a digital library. The team already designed the domain at the class level, but a review revealed that the model "works by convention": delete behavior is unpredictable, the genre join table does not store the assignment date, and the book profile appears and disappears because of confusion between principal and dependent. The product manager phrased the requirements in domain terms rather than table terms: a book must have an author, but a publisher may be unknown; genres are assigned by an administrator and it is important to know when; a book profile is unique and cannot exist without the book itself; deleting an author must remove their books, while deleting a publisher must keep the books but null out the reference.

Your task is to turn a "works by convention" application into an explicitly configured EF Core 8 model, in which every relationship is described with the Fluent API, every `OnDelete` behavior is chosen deliberately, and related data is loaded eagerly through `Include` without hidden N+1 queries. Along the way you must make migrations deterministic: if someone renames a navigation property, the database schema must not drift. This is a typical growth scenario, when conventions stop coping and the team moves to an explicit model specification.

#### What to do step by step
1. Create a new .NET 8 console project `BookHive` and install the packages `Microsoft.EntityFrameworkCore.Sqlite` (8.0.x) and `Microsoft.EntityFrameworkCore.Design` (8.0.x). Commands:
   - `dotnet new console -n BookHive -o BookHive -f net8.0`
   - `cd BookHive`
   - `dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 8.0.*`
   - `dotnet add package Microsoft.EntityFrameworkCore.Design --version 8.0.*`
2. Define the entities: `Author` (Id, Name, Country, `ICollection<Book> Books`), `Publisher` (Id, Name, `ICollection<Book> Books`), `Book` (Id, Title, Year, AuthorId, Author, PublisherId?, Publisher, `ICollection<Genre> Genres` (skip navigation), `ICollection<BookGenre> BookGenres`, BookProfile? Profile), `Genre` (Id, Name, `ICollection<Book> Books`, `ICollection<BookGenre> BookGenres`), `BookGenre` (BookId, Book, GenreId, Genre, AssignedAt, AssignedBy), `BookProfile` (Id, Description, PageCount, BookId, Book). Initialize every navigation collection in an initializer (`= new List<T>()`), and mark required reference navigations with `= null!`, exactly as in the lesson.
3. Implement `BookHiveDbContext : DbContext` with a `DbSet` for every entity. In `OnConfiguring` connect SQLite to a file `bookhive.db` (or use `OnModelCreating` + `DbContextOptions` — your choice). In `OnModelCreating` describe all four relationships explicitly with the Fluent API:
   - required 1:N `Author → Book` via `HasMany(a => a.Books).WithOne(b => b.Author).HasForeignKey(b => b.AuthorId).OnDelete(DeleteBehavior.Cascade)`;
   - optional 1:N `Publisher → Book` via `HasMany(p => p.Books).WithOne(b => b.Publisher).HasForeignKey(b => b.PublisherId).OnDelete(DeleteBehavior.SetNull)` (nullable FK);
   - N:N with payload `Book ↔ Genre` via the join entity `BookGenre`: composite `HasKey(bg => new { bg.BookId, bg.GenreId })` and two `HasOne(...).WithMany(...).HasForeignKey(...)`;
   - 1:1 `Book → BookProfile` via `HasOne(b => b.Profile).WithOne(p => p.Book).HasForeignKey<BookProfile>(p => p.BookId).OnDelete(DeleteBehavior.Cascade)`.
4. Create a migration: `dotnet ef migrations add InitialRelations`. Open the generated file and verify that `Books` has `AuthorId` (non-null) and `PublisherId` (nullable), that `BookGenres` has a composite PK and two FKs, and that `BookProfiles.BookId` is backed by a unique index. Apply it: `dotnet ef database update`.
5. In `Program.cs` (top-level statements) build a seed: one author with two books, one publisher, three genres, two `BookGenre` rows with different `AssignedAt`, and one `BookProfile` per book. Save with a single `SaveChangesAsync` — EF must fill in all FKs from the navigations.
6. Write three queries and print the results to the console:
   - eager-load a book with the full graph: `db.Books.Include(b => b.Author).Include(b => b.Publisher).Include(b => b.Profile).Include(b => b.Genres).FirstAsync(...)`;
   - delete the publisher and verify that the books' `PublisherId` became `null` (a `SetNull` demo);
   - delete the author and verify that the books disappeared along with their profiles and genre links (a multi-level cascade demo).
7. Run `dotnet run` and record the output. Expected: a list of books with author/publisher/profile/genres; after step 6 — books with no publisher; after step 7 — an empty list for the deleted author.

#### Requirements
- The project compiles on .NET 8 / C# 12 without error-level warnings. Top-level statements in `Program.cs`; enable `<Nullable>enable</Nullable>` and `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` (or at least `nullable enable`).
- All four relationships are described explicitly in `OnModelCreating` through the Fluent API. Conventions must not silently decide `OnDelete` — every behavior is written by hand.
- Navigation collections are initialized (`new List<T>()`), required reference navigations are marked `null!`. Leaving an uninitialized `ICollection<>` is forbidden.
- Scalar FK properties are present explicitly (`AuthorId`, `PublisherId`, `BookId`), not "navigation only" — this is a requirement from the "Common Mistakes" section of the lesson.
- The migration `InitialRelations` is created and applied to SQLite without cascade-cycle errors. If EF reports a cycle — explain in a comment which link you switched to `Restrict` and why.
- Queries use `Include` (and `ThenInclude` where needed), not lazy loading. Lazy loading must be off by default.
- The seed is deterministic: a re-run yields the same picture. Use `EnsureDeletedAsync`/`EnsureCreatedAsync` or rebuild the DB through migrations.
- The code is annotated with RU+EN comments at the key configuration spots (following the lesson style).

#### Pitfalls
- **Cascade cycles.** In our model both "required" cascades (`Author→Book` and `Book→BookProfile`) walk the same path, but they do not form a cycle because the direction is strictly downward. However, if you add a reverse required link with a cascade (say, `Book → Author` as principal with a required FK), EF will throw `InvalidOperationException` during migration. The lesson's remedy: switch one of the links to `Restrict`/`NoAction` or make the FK nullable. If your implementation hits the error — it is not a bug, it is expected; consciously choose which link to "loosen".
- **`SetNull` requires a nullable FK.** The `PublisherId` column must be `int?`. If you leave it as `int` (non-nullable), EF either refuses to apply `SetNull` (SQLite will silently turn it into `NO ACTION`) or the delete throws. This is a direct illustration of the lesson's "FK non-nullable but OnDelete = SetNull" mistake.
- **N:N with payload cannot be simplified to a skip navigation.** Because `BookGenre` stores `AssignedAt` and `AssignedBy`, you must describe the join entity explicitly: composite `HasKey`, two `HasOne/WithMany/HasForeignKey`. You may keep **both** collections on `Book` and `Genre` — the skip navigation `Genres`/`Books` and the join-entity collection `BookGenres`. The lesson shows a variant without a skip navigation (`WithMany()` with no parameter); pick either, but use the one you configured in the seed and in the queries.
- **Always specify `WithOne`/`WithMany`.** Omitting the back navigation makes EF "guess" the direction and may flip the relationship. In 1:1 always write `HasOne(b => b.Profile).WithOne(p => p.Book)` — both sides are named.
- **Principal vs dependent in 1:1.** `BookProfile` is the dependent (it carries the FK `BookId`). `HasForeignKey<BookProfile>(p => p.BookId)` tells EF which side is dependent. If you swap them, EF may try to add the FK to `Books`, which breaks the 1:1 semantics.
- **Collection initialization.** `Genres = { new Genre {...} }` in an initializer only works if the collection is already created. The lesson initializes `new List<Book>()` — do the same, otherwise you get a `NullReferenceException` when adding before `SaveChanges`.
- **Lazy loading is off by default** in EF Core unless you install `Microsoft.EntityFrameworkCore.Proxies` and call `UseLazyLoadingProxies()`. Do not enable it — use `Include`, as the lesson's best practice demands.

#### Acceptance criteria
- [ ] The `BookHive` project on .NET 8 / C# 12 builds without errors or warnings.
- [ ] All six entities are defined with correct navigations and scalar FKs.
- [ ] Navigation collections are initialized with `new List<T>()`, required reference navigations with `null!`.
- [ ] 1:N `Author → Book` is configured with `HasMany().WithOne().HasForeignKey().OnDelete(Cascade)`.
- [ ] Optional 1:N `Publisher → Book` is configured with a nullable FK and `OnDelete(SetNull)`.
- [ ] N:N with payload is configured through an explicit `BookGenre` with a composite `HasKey` and two FKs.
- [ ] 1:1 `Book → BookProfile` is configured with `HasOne().WithOne().HasForeignKey<BookProfile>()`.
- [ ] The migration `InitialRelations` is created and applied without errors (including the absence of a cascade cycle).
- [ ] In the generated migration `BookProfiles.BookId` is backed by a unique index.
- [ ] The seed creates an author with two books, a publisher, three genres, two `BookGenre` rows, and two `BookProfile` rows.
- [ ] Eager loading through `Include` prints the full book graph without N+1.
- [ ] Deleting the publisher nulls out `PublisherId` on the books (a `SetNull` demo).
- [ ] Deleting the author cascades the deletion of books, profiles, and genre links.
- [ ] Lazy loading is not enabled; `Include` is used explicitly.
- [ ] Key configuration spots are commented RU+EN.
- [ ] A `README` or comments explain the `OnDelete` choice for every link.

#### Hints (no direct answer)
- If EF complains about a cascade cycle during migration, look at which two required links form a closed path and ask yourself: which of them semantically tolerates `Restrict` or a nullable FK?
- To verify 1:1 uniqueness, try creating two `BookProfile` rows with the same `BookId` in a test — EF/the DB must reject it.
- To eagerly load genres through the join entity use `Include(b => b.BookGenres).ThenInclude(bg => bg.Genre)`; for a skip navigation use plain `Include(b => b.Genres)`.
- `EnsureDeleted`/`EnsureCreated` are handy for demos, but migrations are a hard requirement; do not substitute one for the other.
- To see the SQL, add `optionsBuilder.LogTo(Console.WriteLine)` — it will help you confirm that `Include` produces a single JOIN rather than dozens of queries.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 / EF Core 8 — BookHive: 1:N, N:N (payload), 1:1
// Full working example of relationship configuration per the assignment.

using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

// --- Entities ---

public class Author            // principal in 1:N
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Country { get; set; } = string.Empty;
    public ICollection<Book> Books { get; set; } = new List<Book>();
}

public class Publisher         // principal in optional 1:N
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public ICollection<Book> Books { get; set; } = new List<Book>();
}

public class Book
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public int Year { get; set; }

    // required FK to Author
    public int AuthorId { get; set; }
    public Author Author { get; set; } = null!;

    // optional FK to Publisher (nullable → SetNull is valid)
    public int? PublisherId { get; set; }
    public Publisher? Publisher { get; set; }

    // N:N skip navigation
    public ICollection<Genre> Genres { get; set; } = new List<Genre>();

    // N:N via join entity with payload
    public ICollection<BookGenre> BookGenres { get; set; } = new List<BookGenre>();

    // 1:1 principal side
    public BookProfile? Profile { get; set; }
}

public class Genre
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public ICollection<Book> Books { get; set; } = new List<Book>();
    public ICollection<BookGenre> BookGenres { get; set; } = new List<BookGenre>();
}

// join entity with payload
public class BookGenre
{
    public int BookId { get; set; }
    public Book Book { get; set; } = null!;

    public int GenreId { get; set; }
    public Genre Genre { get; set; } = null!;

    public DateTime AssignedAt { get; set; }   // payload
    public string AssignedBy { get; set; } = string.Empty; // payload
}

// 1:1 dependent side
public class BookProfile
{
    public int Id { get; set; }
    public string Description { get; set; } = string.Empty;
    public int PageCount { get; set; }

    // FK to Book; unique index guarantees 1:1
    public int BookId { get; set; }
    public Book Book { get; set; } = null!;
}

// --- DbContext ---

public class BookHiveDbContext : DbContext
{
    public DbSet<Author> Authors => Set<Author>();
    public DbSet<Publisher> Publishers => Set<Publisher>();
    public DbSet<Book> Books => Set<Book>();
    public DbSet<Genre> Genres => Set<Genre>();
    public DbSet<BookGenre> BookGenres => Set<BookGenre>();
    public DbSet<BookProfile> Profiles => Set<BookProfile>();

    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options
            .UseSqlite("Data Source=bookhive.db")
            .LogTo(Console.WriteLine, LogLevel.Information); // SQL to console

    protected override void OnModelCreating(ModelBuilder mb)
    {
        // required 1:N: Author -> Book (Cascade)
        mb.Entity<Author>()
            .HasMany(a => a.Books)
            .WithOne(b => b.Author)
            .HasForeignKey(b => b.AuthorId)
            .OnDelete(DeleteBehavior.Cascade);

        // optional 1:N: Publisher -> Book (SetNull)
        mb.Entity<Publisher>()
            .HasMany(p => p.Books)
            .WithOne(b => b.Publisher)
            .HasForeignKey(b => b.PublisherId)
            .OnDelete(DeleteBehavior.SetNull);

        // N:N with payload: composite PK + two FKs
        mb.Entity<BookGenre>()
            .HasKey(bg => new { bg.BookId, bg.GenreId });

        mb.Entity<BookGenre>()
            .HasOne(bg => bg.Book)
            .WithMany(b => b.BookGenres)
            .HasForeignKey(bg => bg.BookId)
            .OnDelete(DeleteBehavior.Cascade);

        mb.Entity<BookGenre>()
            .HasOne(bg => bg.Genre)
            .WithMany(g => g.BookGenres)
            .HasForeignKey(bg => bg.GenreId)
            .OnDelete(DeleteBehavior.Cascade);

        // 1:1: Book (principal) -> BookProfile (dependent)
        mb.Entity<Book>()
            .HasOne(b => b.Profile)
            .WithOne(p => p.Book)
            .HasForeignKey<BookProfile>(p => p.BookId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}

// --- Program (top-level statements) ---

using var db = new BookHiveDbContext();
await db.Database.EnsureDeletedAsync();     // clean start
await db.Database.MigrateAsync();           // apply migrations

// Seed
var author = new Author { Name = "Leo Tolstoy", Country = "Russia" };
var publisher = new Publisher { Name = "Alpina" };
var classic = new Genre { Name = "Classic" };
var epic = new Genre { Name = "Epic" };
var novel = new Genre { Name = "Novel" };

var book1 = new Book
{
    Title = "War and Peace", Year = 1869, Author = author, Publisher = publisher,
    Profile = new BookProfile { Description = "Epic novel", PageCount = 1225 }
};
var book2 = new Book
{
    Title = "Anna Karenina", Year = 1877, Author = author, Publisher = publisher,
    Profile = new BookProfile { Description = "Novel", PageCount = 864 }
};

// N:N via join entity with payload
book1.BookGenres.Add(new BookGenre { Genre = classic, AssignedAt = DateTime.UtcNow, AssignedBy = "admin" });
book1.BookGenres.Add(new BookGenre { Genre = epic,     AssignedAt = DateTime.UtcNow, AssignedBy = "admin" });
book2.BookGenres.Add(new BookGenre { Genre = novel,    AssignedAt = DateTime.UtcNow, AssignedBy = "admin" });

db.Books.AddRange(book1, book2);
await db.SaveChangesAsync();

// 1) Eager-load the full graph
var loaded = await db.Books
    .Include(b => b.Author)
    .Include(b => b.Publisher)
    .Include(b => b.Profile)
    .Include(b => b.BookGenres).ThenInclude(bg => bg.Genre)
    .ToListAsync();

foreach (var b in loaded)
    Console.WriteLine($"{b.Title} ({b.Year}) — {b.Author.Name} — publisher: {b.Publisher?.Name ?? "—"} — profile: {b.Profile?.Description} — genres: {b.BookGenres.Count}");

// 2) Delete publisher → SetNull
var pub = await db.Publishers.FirstAsync();
db.Publishers.Remove(pub);
await db.SaveChangesAsync();

var afterPub = await db.Books.AsNoTracking().ToListAsync();
Console.WriteLine($"Books without publisher: {afterPub.Count(b => b.PublisherId is null)}");

// 3) Delete author → multi-level cascade
var aut = await db.Authors.FirstAsync();
db.Authors.Remove(aut);
await db.SaveChangesAsync();

var remaining = await db.Books.AsNoTracking().CountAsync();
Console.WriteLine($"Books remaining: {remaining}");
```

**Line-by-line walk-through.** The `Author` class is the principal side of a required 1:N: it owns a `Books` collection initialized with `new List<Book>()` (a lesson best practice: initialize collections). `Book.AuthorId` is a scalar FK (the "not navigation only" requirement), and `Book.Author` is marked `null!` because after loading from the DB the reference will always be set and the compiler should not bother us with nullable warnings. `PublisherId` is `int?` — the pivotal detail: the nullable type is what makes `OnDelete(DeleteBehavior.SetNull)` legal. If the column were non-nullable, SQLite would silently turn the behavior into `NO ACTION` and the publisher delete would throw — exactly the lesson's "FK non-nullable but OnDelete = SetNull" mistake.

The join entity `BookGenre` carries a payload (`AssignedAt`, `AssignedBy`), so the N:N cannot be implemented "simply" with a skip navigation alone: an explicit class is required. The composite `HasKey(bg => new { bg.BookId, bg.GenreId })` forms the primary key of the join table, and the two `HasOne/WithMany/HasForeignKey` calls describe the two 1:N links converging on the join entity. Note that `WithMany(b => b.BookGenres)` names the back navigation explicitly — this guards against EF "guessing" the direction (the common "forgot `WithOne`/`WithMany`" mistake). At the same time `Book` keeps the skip navigation `Genres`; in this walk-through we load genres through `BookGenres` + `ThenInclude` to show payload access, but you could configure `UsingEntity` and reach into `Genres` directly.

The 1:1 link `Book → BookProfile` follows the lesson template exactly: `HasOne(b => b.Profile).WithOne(p => p.Book).HasForeignKey<BookProfile>(p => p.BookId)`. The type parameter `<BookProfile>` on `HasForeignKey` tells EF which side is the dependent (the one carrying the FK), and `BookId` becomes both the FK and (thanks to the unique index EF creates automatically) the guarantor of one profile per book. `OnDelete(Cascade)` means: delete the book — the profile goes too. All cascades in the model (`Author→Book`, `Book→BookProfile`, `Book→BookGenre`, `Genre→BookGenre`) point strictly downward, so no cycle arises — exactly the acyclic graph the lesson recommends.

In `OnConfiguring` the `LogTo` call surfaces SQL — it visibly proves that `Include` + `ThenInclude` yields a single query with JOINs, not N+1. Top-level statements in `Program.cs` match C# 12 / .NET 8; `EnsureDeletedAsync` + `MigrateAsync` give a clean start and apply migrations. Deleting the publisher (step 2) demonstrates `SetNull` — afterward `PublisherId` is `null` and the books remain. Deleting the author (step 3) triggers a multi-level cascade: books vanish → profiles vanish → `BookGenre` rows vanish. `AsNoTracking` on the verification queries is a best practice for read-only paths: less overhead and cleaner SQL.

#### Going deeper (bonus)
1. Add a second 1:1 link `Book → BookCover`, but this time use a **shared primary key** (`BookCover.Id` is both the PK and the FK to `Book`), configuring `HasOne().WithOne().HasForeignKey<BookCover>(c => c.Id)`. Compare the two 1:1 configurations (separate FK + unique index vs. shared PK) from the standpoint of migrations and semantics.
2. Add an **N:N without payload** — `Book ↔ Tag` through skip navigations on both sides, with no explicit join class. Confirm that EF creates the `BookTag` join table by itself. Compare the SQL produced by the skip navigation with the SQL for the payload-bearing `BookGenre`.
3. Deliberately provoke a **cascade cycle**: add a reverse required link `Book → Author` (say, `PrimaryAuthorId`) with `Cascade`. Capture the `InvalidOperationException` from `migrations add`, then fix the model by switching one link to `Restrict` or making the FK nullable. Explain in a comment why you picked that option.
4. Rewrite the eager-load query as a **projection (DTO)** through `Select`, returning only the fields you need (`Title`, `Author.Name`, `Publisher?.Name`, the list of genres). Compare the number and shape of the SQL queries with the `Include` version.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `BookHive` собирается на .NET 8 / C# 12 без ошибок.
- [ ] Все шесть сущностей определены с навигациями и скалярными FK.
- [ ] Коллекции инициализированы, обязательные навигации — `null!`.
- [ ] Все четыре отношения настроены явно через Fluent API.
- [ ] `OnDelete` задан вручную для каждой связи; циклов каскадов нет.
- [ ] Миграция `InitialRelations` создаётся и применяется.
- [ ] `BookProfiles.BookId` имеет уникальный индекс в миграции.
- [ ] Seed детерминирован и сохраняется одним `SaveChangesAsync`.
- [ ] Жадная загрузка через `Include`/`ThenInclude` выводит полный граф.
- [ ] Удаление издателя обнуляет `PublisherId`; удаление автора каскадно чистит граф.
- [ ] Ленивая загрузка выключена; SQL логируется через `LogTo`.
- [ ] Ключевые места прокомментированы RU+EN.
- [ ] Бонусные задания выполнены (опционально) и описаны в `README`.

- [ ] The `BookHive` project builds on .NET 8 / C# 12 without errors.
- [ ] All six entities are defined with navigations and scalar FKs.
- [ ] Collections are initialized, required navigations are `null!`.
- [ ] All four relationships are configured explicitly with the Fluent API.
- [ ] `OnDelete` is set by hand for every link; no cascade cycles.
- [ ] The migration `InitialRelations` is created and applied.
- [ ] `BookProfiles.BookId` has a unique index in the migration.
- [ ] The seed is deterministic and saved with a single `SaveChangesAsync`.
- [ ] Eager loading through `Include`/`ThenInclude` prints the full graph.
- [ ] Deleting the publisher nulls `PublisherId`; deleting the author cascades the graph clean.
- [ ] Lazy loading is off; SQL is logged via `LogTo`.
- [ ] Key spots are commented RU+EN.
- [ ] Bonus tasks are done (optional) and described in the `README`.

#### Ресурсы / Resources
- [Microsoft Learn — Relationships — https://learn.microsoft.com/ef/core/modeling/relationships](https://learn.microsoft.com/ef/core/modeling/relationships)
- [Microsoft Learn — Cascade delete — https://learn.microsoft.com/ef/core/saving/cascade-delete](https://learn.microsoft.com/ef/core/saving/cascade-delete)
- [Microsoft Learn — Many-to-many — https://learn.microsoft.com/ef/core/modeling/relationships/many-to-many](https://learn.microsoft.com/ef/core/modeling/relationships/many-to-many)
- [Microsoft Learn — One-to-one — https://learn.microsoft.com/ef/core/modeling/relationships/one-to-one](https://learn.microsoft.com/ef/core/modeling/relationships/one-to-one)
- [Microsoft Learn — Eager loading — https://learn.microsoft.com/ef/core/querying/related-data/eager](https://learn.microsoft.com/ef/core/querying/related-data/eager)
