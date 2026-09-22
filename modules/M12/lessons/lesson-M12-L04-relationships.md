[⬆ К модулю M12](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M12-L04: Отношения: 1:N, N:N, 1:1, навигационные свойства / Relationships: 1:N, N:N, 1:1, navigation properties

**Модуль / Module:** M12
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Отношения между сущностями — это сердцевина любой предметной области. В EF Core отношения описываются через **навигационные свойства** (ссылки на связанные сущности в C#-классах) и **внешние ключи** (FK — скалярные поля, хранящие значение ключа связанной строки). EF Core умеет выводить большинство отношений по конвенции, но явная конфигурация через `HasOne`/`HasMany` делает модель понятной и устойчивой к рефакторингу.

Представьте библиотеку. Один автор пишет много книг — это **«один-ко-многим» (1:N)**. У книги есть ровно один автор (навигационное свойство `Author` и FK `AuthorId`), а у автора — коллекция книг (`ICollection<Book>`). EF Core создаёт FK-колонку `AuthorId` в таблице `Books` и при загрузке автора может «подтянуть» книги через жадную (`Include`) или ленивую загрузку.

Когда одна книга может принадлежать нескольким жанрам, а один жанр — многим книгам, возникает **«многие-ко-многим» (N:N)**. В .NET 8 EF Core поддерживает N:N без явного класса соединения: достаточно двух навигационных коллекций на обеих сторонах, и провайдер сам создаст join-таблицу `BookGenre` с двумя FK. Если же join-таблице нужны собственные данные (например, дата присвоения жанра), используйте «скачанный» join-тип — класс `BookGenre` с навигациями к обеим сторонам и конфигурацией через `HasMany(...).UsingEntity<>()`.

**«Один-к-одному» (1:1)** встречается, когда у книги есть ровно один детальный профиль (`BookProfile`). EF Core должен знать, какая сторона является «главной» (principal), а какая — «зависимой» (dependent). Зависимая сторона хранит FK и (опционально) уникальный индекс, который гарантирует, что два профиля не сошлются на одну книгу. Настройте это через `HasOne(...).WithOne(...).HasForeignKey<>()`.

**Каскадное удаление** определяет, что происходит с дочерними строками при удалении родителя. Поведение задаётся через `OnDelete(...)`. Значение `Cascade` удаляет детей вместе с родителем — подходит, когда дети не могут существовать без родителя (книги без автора). `Restrict`/`NoAction` запрещает удаление, пока есть дети. `SetNull` обнуляет FK (если колонка допускает NULL). Важно: каскад между двумя зависимыми сущностями с обязательным FK иногда создаёт «каскадные циклы», и EF Core выбрасывает `InvalidOperationException` при миграции — тогда переключите одну из связей на `Restrict` или сделайте FK необязательным.

Главное правило: проектируйте отношения в терминах предметной области, а не таблиц. Навигационные свойства — это API вашего домена; FK-колонки — деталь хранения. Явно конфигурируйте нетривиальные случаи, контролируйте `OnDelete` и держите моделируемый граф ацикличным.

#### Theory (EN)

Relationships between entities are the heart of any domain model. In EF Core, relationships are expressed through **navigation properties** (references to related entities in your C# classes) and **foreign keys** (FK — scalar fields that store the key value of the related row). EF Core can infer most relationships by convention, but explicit configuration with `HasOne`/`HasMany` keeps the model readable and resilient to refactoring.

Picture a library. One author writes many books — that is **one-to-many (1:N)**. A book has exactly one author (navigation `Author` plus FK `AuthorId`), while an author has a collection of books (`ICollection<Book>`). EF Core creates an `AuthorId` column in the `Books` table and, when loading an author, can pull the books through eager (`Include`) or lazy loading.

When a book can belong to several genres and a genre to many books, you get **many-to-many (N:N)**. In .NET 8, EF Core supports N:N without an explicit join class: two navigation collections on each side are enough, and the provider creates a `BookGenre` join table with two FKs. If the join table needs its own data (say, the date a genre was assigned), use a shared join entity — a `BookGenre` class with navigations to both sides, configured via `HasMany(...).UsingEntity<>()`.

**One-to-one (1:1)** shows up when a book has exactly one detail profile (`BookProfile`). EF Core must know which side is the **principal** and which is the **dependent**. The dependent side carries the FK and (optionally) a unique index that guarantees two profiles never reference the same book. Configure it with `HasOne(...).WithOne(...).HasForeignKey<>()`.

**Cascade delete** decides what happens to child rows when a parent is deleted, controlled by `OnDelete(...)`. `Cascade` deletes children with the parent — ideal when children cannot exist without a parent (books without an author). `Restrict`/`NoAction` forbids deletion while children exist. `SetNull` nulls out the FK (only when the column is nullable). Beware: cascades between two dependent entities with required FKs can create a **cascade cycle**, and EF Core throws `InvalidOperationException` during migration — switch one link to `Restrict` or make the FK optional.

The golden rule: model relationships in terms of the domain, not tables. Navigation properties are the API of your domain; FK columns are a storage detail. Explicitly configure non-trivial cases, control `OnDelete`, and keep the modeled graph acyclic.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 / EF Core 8
// Демонстрация отношений 1:N, N:N и 1:1 с явной конфигурацией.
// Demo of 1:N, N:N, and 1:1 relationships with explicit configuration.

using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

// --- Сущности / Entities ---

// 1:N: у одного автора много книг / one author has many books
public class Author
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    // Навигационная коллекция / navigation collection
    public ICollection<Book> Books { get; set; } = new List<Book>();
}

public class Book
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;

    // Внешний ключ / foreign key (scalar)
    public int AuthorId { get; set; }

    // Навигация к родителю / navigation to principal
    public Author Author { get; set; } = null!;

    // N:N: книга ↔ жанры / book ↔ genres (join table создастся автоматически)
    public ICollection<Genre> Genres { get; set; } = new List<Genre>();

    // 1:1: у книги ровно один профиль / book has exactly one profile
    public BookProfile? Profile { get; set; }
}

// Join-сущность с собственными данными / join entity with payload
public class BookGenre
{
    public int BookId { get; set; }
    public Book Book { get; set; } = null!;

    public int GenreId { get; set; }
    public Genre Genre { get; set; } = null!;

    public DateTime AssignedAt { get; set; } // payload / данные связи
}

public class Genre
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    public ICollection<Book> Books { get; set; } = new List<Book>();
}

// 1:1: зависимая сущность / dependent side
public class BookProfile
{
    public int Id { get; set; }
    public string Description { get; set; } = string.Empty;

    // FK = первичный ключ книги → гарантирует 1:1 / FK points to Book PK → guarantees 1:1
    public int BookId { get; set; }
    public Book Book { get; set; } = null!;
}

// --- DbContext с конфигурацией / DbContext with configuration ---

public class LibraryDbContext : DbContext
{
    public DbSet<Author> Authors => Set<Author>();
    public DbSet<Book> Books => Set<Book>();
    public DbSet<Genre> Genres => Set<Genre>();
    public DbSet<BookProfile> Profiles => Set<BookProfile>();
    public DbSet<BookGenre> BookGenres => Set<BookGenre>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // 1:N: Author -> Book, каскадное удаление / cascade delete
        // Одна сторона — HasMany, другая — HasOne с HasForeignKey.
        modelBuilder.Entity<Author>()
            .HasMany(a => a.Books)
            .WithOne(b => b.Author)
            .HasForeignKey(b => b.AuthorId)
            .OnDelete(DeleteBehavior.Cascade);

        // N:N через join-сущность с payload / many-to-many via join entity with payload
        modelBuilder.Entity<BookGenre>()
            .HasKey(bg => new { bg.BookId, bg.GenreId }); // составной PK / composite PK

        modelBuilder.Entity<BookGenre>()
            .HasOne(bg => bg.Book)
            .WithMany() // у Book нет явной коллекции BookGenre
            .HasForeignKey(bg => bg.BookId)
            .OnDelete(DeleteBehavior.Cascade);

        modelBuilder.Entity<BookGenre>()
            .HasOne(bg => bg.Genre)
            .WithMany()
            .HasForeignKey(bg => bg.GenreId)
            .OnDelete(DeleteBehavior.Cascade);

        // 1:1: Book -> BookProfile
        // Principal = Book, Dependent = BookProfile; FK = BookId.
        modelBuilder.Entity<Book>()
            .HasOne(b => b.Profile)
            .WithOne(p => p.Book)
            .HasForeignKey<BookProfile>(p => p.BookId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}

// --- Использование / Usage ---

public static class RelationshipsDemo
{
    public static async Task RunAsync(LibraryDbContext db)
    {
        // Создаём автора с книгами, жанрами и профилем за один SaveChanges.
        // EF сам проставит FK по навигациям / EF fills FKs from navigations.
        var author = new Author { Name = "Толстой / Tolstoy" };
        var book = new Book
        {
            Title = "Война и мир / War and Peace",
            Author = author,
            Genres = { new Genre { Name = "Classic" } },
            Profile = new BookProfile { Description = "Эпопея / Epic novel" }
        };

        db.Books.Add(book);
        await db.SaveChangesAsync();

        // Жадная загрузка связанных данных / eager load related data
        var loaded = await db.Books
            .Include(b => b.Author)
            .Include(b => b.Profile)
            .Include(b => b.Genres)
            .FirstAsync(b => b.Id == book.Id);

        Console.WriteLine($"{loaded.Title} — {loaded.Author.Name} — {loaded.Profile?.Description}");
    }
}
```

#### Best Practices

- Конфигурируйте отношения явно через Fluent API в `OnModelCreating` — это стабильнее конвенций и читается как спецификация модели.
- Давайте навигационным свойствам осмысленные имена в терминах домена (`Author`, `Books`, `Profile`), а не в терминах таблиц.
- Явно указывайте `OnDelete`, чтобы поведение при удалении было предсказуемым и не зависело от правила «обязательный FK → каскад».
- Инициализируйте навигационные коллекции (`= new List<T>()`) — это упрощает юнит-тесты и добавление детей до сохранения.
- Для N:N с данными в join-таблице используйте явный join-класс и `HasKey` для составного первичного ключа.

- Configure relationships explicitly with the Fluent API in `OnModelCreating` — it is more stable than conventions and reads like a model spec.
- Name navigation properties in domain terms (`Author`, `Books`, `Profile`), not in table terms.
- Set `OnDelete` explicitly so delete behavior is predictable and does not silently depend on the “required FK ⇒ cascade” rule.
- Initialize navigation collections (`= new List<T>()`) to simplify unit tests and child insertion before save.
- For N:N with data in the join table, use an explicit join class and `HasKey` for a composite primary key.

#### Частые ошибки / Common Mistakes

- [Нет FK-свойства, только навигация] → [Добавьте скалярное `AuthorId`, чтобы миграции и обновления были детерминированными.] (RU)
- [Два обязательных каскада на одном пути] → [EF выбросит ошибку цикла; переключите одну связь на `Restrict` или сделайте FK nullable.] (RU)
- [N:N без join-класса, но нужен payload] → [Введите явный `BookGenre` с `UsingEntity<>` или составным `HasKey`.] (RU)
- [Ленивая загрузка включена «на всякий случай»] → [Выключайте по умолчанию; используйте `Include`/`AsNoTracking`, чтобы избежать N+1 и скрытых SQL-запросов.] (RU)
- [Забыт `WithOne`/`WithMany`] → [EF может развернуть отношение не в ту сторону; всегда указывайте обратную навигацию явно.] (RU)
- [FK зависит от NULL, а `OnDelete = SetNull`] → [Колонка non-nullable приведёт к исключению; сделайте FK nullable или выберите `Restrict`.] (RU)

- [Missing FK property, navigation only] → [Add a scalar `AuthorId` so migrations and updates are deterministic.] (EN)
- [Two required cascades on the same path] → [EF throws a cycle error; switch one link to `Restrict` or make the FK nullable.] (EN)
- [N:N without join class, but you need payload] → [Introduce an explicit `BookGenre` with `UsingEntity<>` or a composite `HasKey`.] (EN)
- [Lazy loading enabled “just in case”] → [Disable by default; use `Include`/`AsNoTracking` to avoid N+1 and hidden SQL.] (EN)
- [Missing `WithOne`/`WithMany`] → [EF may flip the relationship direction; always specify the back navigation explicitly.] (EN)
- [FK is non-nullable but `OnDelete = SetNull`] → [A non-nullable column throws; make the FK nullable or pick `Restrict`.] (EN)

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Для каждого отношения указаны навигация и FK. (RU)
- [ ] 1:N настроено через `HasMany().WithOne().HasForeignKey()`. (RU)
- [ ] N:N либо без join-класса, либо с явным `BookGenre` и составным ключом. (RU)
- [ ] 1:1 настроено через `HasOne().WithOne().HasForeignKey<Dependent>()`. (RU)
- [ ] `OnDelete` задан явно и не создаёт цикл каскадов. (RU)
- [ ] Жадная загрузка используется вместо скрытой ленивой. (RU)
- [ ] Коллекции навигаций инициализированы в конструкторе/инициализаторе. (RU)

- [ ] Every relationship has both a navigation and an FK. (EN)
- [ ] 1:N is configured with `HasMany().WithOne().HasForeignKey()`. (EN)
- [ ] N:N is either skip-navigation or an explicit `BookGenre` with a composite key. (EN)
- [ ] 1:1 is configured with `HasOne().WithOne().HasForeignKey<Dependent>()`. (EN)
- [ ] `OnDelete` is set explicitly and does not form a cascade cycle. (EN)
- [ ] Eager loading is used instead of hidden lazy loading. (EN)
- [ ] Navigation collections are initialized in the constructor/initializer. (EN)

#### Ресурсы / Resources

- [Microsoft Learn — Relationships — https://learn.microsoft.com/ef/core/modeling/relationships](https://learn.microsoft.com/ef/core/modeling/relationships)

---

[⬆ К модулю M12](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
