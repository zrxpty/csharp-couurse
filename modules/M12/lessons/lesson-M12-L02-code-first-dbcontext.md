[⬆ К модулю M12](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M12-L02: Code-first, сущности, DbContext, подключение / Code-first, entities, DbContext, connection

**Модуль / Module:** M12
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Entity Framework Core (EF Core) — это объектно-реляционный маппер (ORM) от Microsoft. Он берёт на себя «перевод» между миром объектов C# и миром таблиц реляционной базы данных. Подход **Code-first** означает, что мы сначала пишем классы сущностей на C#, а EF Core сам создаёт базу и таблицы под них. Это удобно: модель данных живёт в коде, версионирование идёт через миграции, а база всегда соответствует коду.

**Сущности (POCO).** Сущность — это обычный класс C# (Plain Old CLR Object). Никаких атрибутов или базовых классов не требуется: EF Core работает с чистыми классами по соглашениям. Например, свойство `Id` по умолчанию становится первичным ключом, а навигационное свойство `List<Order> Orders` превращается в связь «один-ко-многим». Хорошие сущности инкапсулируют инварианты: публичные свойства с приватными сеттерами, конструкторы с обязательными параметрами.

**DbSet&lt;T&gt;.** Это «окно» DbContext в таблицу конкретного типа. Каждое `DbSet<User> Users` отображается на таблицу `Users` (имя можно переопределить). Через DbSet мы добавляем, запрашиваем, изменяем и удаляем строки: `db.Users.Add(...)`, `db.Users.Where(...)`, `db.SaveChangesAsync()`.

**DbContext.** Главный класс-оркестратор. Он:
- хранит `DbSet`-свойства,
- управляет изменениями (Change Tracker),
- открывает соединение,
- формирует SQL и отправляет его в провайдер.

Аналогия: DbContext — это «менеджер склада», который знает, где лежит каждый товар (сущность), отслеживает перемещения (изменения) и выписывает накладные (SQL-запросы) поставщику (базе).

**OnModelCreating.** Метод, где мы тонко настраиваем схему, когда соглашений недостаточно: имена таблиц, длины строк, индексы, внешние ключи, уникальные ограничения. Используется Fluent API через `modelBuilder.Entity<T>()`. Правило: сложную конфигурацию лучше выносить в отдельные `IEntityTypeConfiguration<T>` классы и подключать через `modelBuilder.ApplyConfigurationsFromAssembly(...)` — так `OnModelCreating` остаётся читаемым.

**Строка подключения.** Текстовый параметр, описывающий, куда подключаться: сервер, база, учётные данные, таймауты, пул соединений. В .NET 8 её обычно хранят в `appsettings.json` и читают через `IConfiguration`. Никогда не храните пароли в исходниках — используйте User Secrets при разработке и переменные окружения/Key Vault в продакшене.

**DbContextOptions.** Конфигурация DbContext: какой провайдер (SQL Server, PostgreSQL, SQLite…), строка подключения, таймаут команд, чувствительность к регистру, логирование. Передаётся в конструктор через `DbContextOptions<TContext>`. На практике используют `AddDbContext<...>(opt => opt.UseSqlServer(cs))` в `Program.cs`, и DI сам соберёт `DbContextOptions`.

Классический жизненный цикл: регистрируем DbContext в DI как **Scoped** (один экземпляр на HTTP-запрос), инжектим в сервис, выполняем работу, вызываем `SaveChangesAsync`, DI dispose’нет контекст в конце запроса. Не делайте DbContext Singleton — он не потокобезопасен.

#### Theory (EN)

Entity Framework Core (EF Core) is Microsoft’s object-relational mapper (ORM). It translates between C# objects and relational database tables. The **Code-first** approach means we write entity classes first, and EF Core generates the database schema from them. The benefit is clear: the data model lives in code, version control tracks schema changes through migrations, and the database always reflects the codebase.

**Entities (POCO).** An entity is a plain C# class — a Plain Old CLR Object. No base class or attributes are required; EF Core works with clean classes via conventions. For example, an `Id` property becomes the primary key by default, and a `List<Order> Orders` navigation property produces a one-to-many relationship. Good entities encapsulate invariants: public getters with private setters, constructors that enforce required fields.

**DbSet&lt;T&gt;.** This is the DbContext “window” into a table of a given type. Each `DbSet<User> Users` maps to a `Users` table (the name can be overridden). Through a DbSet we add, query, modify, and delete rows: `db.Users.Add(...)`, `db.Users.Where(...)`, `db.SaveChangesAsync()`.

**DbContext.** The main orchestrator class. It:
- holds `DbSet` properties,
- tracks changes (Change Tracker),
- opens the connection,
- builds SQL and sends it to the provider.

Analogy: DbContext is a “warehouse manager” that knows where every item (entity) is stored, tracks movements (changes), and issues shipping orders (SQL queries) to the supplier (the database).

**OnModelCreating.** The method where we fine-tune the schema when conventions are not enough: table names, string lengths, indexes, foreign keys, unique constraints. We use the Fluent API via `modelBuilder.Entity<T>()`. Rule of thumb: move complex configuration into separate `IEntityTypeConfiguration<T>` classes and register them with `modelBuilder.ApplyConfigurationsFromAssembly(...)` so that `OnModelCreating` stays readable.

**Connection string.** A text parameter describing where to connect: server, database, credentials, timeouts, connection pooling. In .NET 8 it is usually stored in `appsettings.json` and read through `IConfiguration`. Never store passwords in source code — use User Secrets in development and environment variables or Azure Key Vault in production.

**DbContextOptions.** The DbContext configuration: which provider (SQL Server, PostgreSQL, SQLite…), the connection string, command timeout, case sensitivity, logging. It is passed to the constructor through `DbContextOptions<TContext>`. In practice you call `AddDbContext<...>(opt => opt.UseSqlServer(cs))` in `Program.cs` and DI assembles `DbContextOptions` for you.

Typical lifecycle: register DbContext in DI as **Scoped** (one instance per HTTP request), inject it into a service, do the work, call `SaveChangesAsync`, and let DI dispose the context at the end of the request. Never make DbContext a Singleton — it is not thread-safe.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8 — Code-first: сущности, DbContext, подключение
// Entities, DbContext, connection — bilingual comments (RU/EN)

using System.ComponentModel.DataAnnotations;
using Microsoft.EntityFrameworkCore;

namespace CourseShop.Domain;

// Сущность-категория / Category entity (POCO)
public class Category
{
    public int Id { get; private set; }              // PK по соглашению / PK by convention
    public string Name { get; private set; } = null!; // обязательное поле / required field

    // Навигационное свойство «один-ко-многим» / one-to-many navigation
    public ICollection<Product> Products { get; private set; } = new List<Product>();

    // Конструктор с инвариантом / constructor enforcing invariant
    public Category(string name) => Name = name;

    public void Rename(string newName)
    {
        if (string.IsNullOrWhiteSpace(newName))
            throw new ArgumentException("Имя не может быть пустым / Name cannot be empty");
        Name = newName;
    }
}

// Сущность-товар / Product entity
public class Product
{
    public int Id { get; private set; }
    public string Title { get; private set; } = null!;
    public decimal Price { get; private set; }

    // Внешний ключ / foreign key
    public int CategoryId { get; private set; }
    public Category Category { get; private set; } = null!; // навигация / navigation

    public Product(string title, decimal price, int categoryId)
    {
        Title = title;
        Price = price;
        CategoryId = categoryId;
    }
}

// Конфигурация сущности через Fluent API / Fluent API configuration
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> b)
    {
        b.ToTable("products");                         // имя таблицы / table name
        b.HasKey(x => x.Id);                           // первичный ключ / primary key
        b.Property(x => x.Title).HasMaxLength(200).IsRequired();
        b.Property(x => x.Price).HasPrecision(10, 2);  // деньги / money precision
        b.HasIndex(x => x.Title);                      // индекс / index
        b.HasOne(x => x.Category)
         .WithMany(c => c.Products)
         .HasForeignKey(x => x.CategoryId)
         .OnDelete(DeleteBehavior.Restrict);          // без каскада / no cascade
    }
}

// DbContext — оркестратор / DbContext orchestrator
public class AppDbContext : DbContext
{
    public DbSet<Category> Categories => Set<Category>();
    public DbSet<Product> Products => Set<Product>();

    // Принимаем опции извне (DI) / options injected from DI
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Применяем все IEntityTypeConfiguration из сборки / apply all configurations
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
        base.OnModelCreating(modelBuilder);
    }
}

// appsettings.json (фрагмент) / snippet
/*
{
  "ConnectionStrings": {
    "Default": "Server=(localdb)\\MSSQLLocalDB;Database=CourseShop;Trusted_Connection=True;TrustServerCertificate=True;"
  }
}
*/

// Регистрация в Program.cs / registration in Program.cs
/*
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

var app = builder.Build();

using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    db.Database.EnsureCreated(); // для демо; в проде — миграции / demo only; use migrations in prod
}

app.Run();
*/

// Пример использования / usage sample
public class CatalogService
{
    private readonly AppDbContext _db;
    public CatalogService(AppDbContext db) => _db = db;

    public async Task<int> AddProductAsync(string title, decimal price, string categoryName)
    {
        var category = await _db.Categories
            .FirstOrDefaultAsync(c => c.Name == categoryName)
            ?? throw new InvalidOperationException("Категория не найдена / Category not found");

        _db.Products.Add(new Product(title, price, category.Id));
        return await _db.SaveChangesAsync(); // возвращает число строк / returns rows affected
    }
}
```

#### Best Practices

- Регистрируйте DbContext как **Scoped** и инжектируйте через DI — один контекст на запрос. / Register DbContext as **Scoped** and inject via DI — one context per request.
- Инкапсулируйте инварианты в сущностях: приватные сеттеры и конструкторы с обязательными параметрами. / Encapsulate invariants in entities: private setters and constructors with required parameters.
- Выносите конфигурацию в `IEntityTypeConfiguration<T>` и подключайте через `ApplyConfigurationsFromAssembly`. / Move configuration into `IEntityTypeConfiguration<T>` and register via `ApplyConfigurationsFromAssembly`.
- Храните строку подключения в `appsettings.json`, секреты — в User Secrets / Key Vault. / Keep the connection string in `appsettings.json`, secrets in User Secrets / Key Vault.
- Используйте `SaveChangesAsync`, а не синхронный `SaveChanges`, в асинхронных путях. / Use `SaveChangesAsync`, not the synchronous `SaveChanges`, on async paths.
- В продакшене используйте миграции, а не `EnsureCreated`. / In production use migrations, not `EnsureCreated`.
- Не делайте DbContext Singleton и не переиспользуйте один экземпляр между потоками. / Never make DbContext a Singleton and never share one instance across threads.

#### Частые ошибки / Common Mistakes

- DbContext зарегистрирован как Singleton → гонки данных и утечки отслеживаемых сущностей. Как избежать: только **Scoped** через `AddDbContext`. / DbContext registered as Singleton → data races and tracked-entity leaks. How to avoid: use **Scoped** only via `AddDbContext`.
- Забыли `await SaveChangesAsync()` → изменения не сохранились. Как избежать: всегда await в конце логики записи. / Forgot `await SaveChangesAsync()` → changes not persisted. How to avoid: always await at the end of write logic.
- Навигационное свойство без `Include` даёт `null`/пустую коллекцию при запросе. Как избежать: используйте `Include`/`Select` или явную загрузку. / Navigation property without `Include` returns `null`/empty collection. How to avoid: use `Include`/`Select` or explicit loading.
- Хардкод строки подключения с паролем в исходниках → утечка секрета. Как избежать: User Secrets + переменные окружения. / Hardcoded connection string with password in source → secret leak. How to avoid: User Secrets + environment variables.
- `EnsureCreated` блокирует миграции. Как избежать: выберите один подход — миграции для продакшена. / `EnsureCreated` blocks migrations. How to avoid: pick one approach — migrations for production.
- Два экземпляра DbContext пытаются отслеживать одну и ту же сущность → исключение. Как избежать: работайте в одном контексте или отсоединяйте сущности (`AsNoTracking`). / Two DbContext instances tracking the same entity → exception. How to avoid: work in one context or detach (`AsNoTracking`).

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Сущности — POCO, без обязательных атрибутов, с инкапсуляцией инвариантов.
- [ ] Каждый DbSet соответствует таблице и доступен только для чтения (`Set<T>()` или автосвойство).
- [ ] Конструктор DbContext принимает `DbContextOptions<T>`.
- [ ] `OnModelCreating` делегирует конфигурацию в `IEntityTypeConfiguration<T>`.
- [ ] Строка подключения берётся из `IConfiguration`, пароли — не в коде.
- [ ] DbContext зарегистрирован как Scoped через `AddDbContext`.
- [ ] Используется `SaveChangesAsync`, а не `SaveChanges`.
- [ ] В продакшене включены миграции, `EnsureCreated` — только для демо.
- [ ] Entities are POCO, no required attributes, with encapsulated invariants.
- [ ] Each DbSet maps to a table and is read-only (`Set<T>()` or auto-property).
- [ ] DbContext constructor accepts `DbContextOptions<T>`.
- [ ] `OnModelCreating` delegates configuration to `IEntityTypeConfiguration<T>`.
- [ ] Connection string comes from `IConfiguration`, passwords not in code.
- [ ] DbContext registered as Scoped via `AddDbContext`.
- [ ] `SaveChangesAsync` is used instead of `SaveChanges`.
- [ ] Migrations are enabled in production, `EnsureCreated` only for demos.

#### Ресурсы / Resources

- Microsoft Learn — DbContext creation and configuration — https://learn.microsoft.com/ef/core/dbcontext-creation/
- EF Core — Code-first conventions — https://learn.microsoft.com/ef/core/modeling/
- EF Core — Fluent API configuration — https://learn.microsoft.com/ef/core/modeling/entity-properties
- Connection strings reference — https://learn.microsoft.com/dotnet/framework/data/adonet/connection-strings

---

[⬆ К модулю M12](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
