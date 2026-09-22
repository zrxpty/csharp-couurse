---
[← К уроку M12-L02](lesson-M12-L02-code-first-dbcontext.md) | [⬆ К модулю M12](../README.md) | [Следующее ДЗ →](homework-M12-L03-migrations-seeding.md)
---

### Домашнее задание M12-L02: Code-first, сущности, DbContext, подключение / Homework M12-L02: Code-first, entities, DbContext, connection

**Урок / Lesson:** M12-L02
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) На практике освоить подход Code-first в EF Core 8: спроектировать сущности-POCO с инкапсуляцией инвариантов, реализовать DbContext с делегированием конфигурации в `IEntityTypeConfiguration<T>`, настроить подключение через `IConfiguration` и зарегистрировать контекст в DI как Scoped, корректно выполняя `SaveChangesAsync`. (EN) Gain hands-on experience with the Code-first approach in EF Core 8: design POCO entities with encapsulated invariants, implement a DbContext that delegates configuration to `IEntityTypeConfiguration<T>`, configure the connection through `IConfiguration`, register the context in DI as Scoped, and correctly call `SaveChangesAsync`.

#### Связь с уроком / Connection to the lesson

(RU) Урок вводит ключевые понятия Code-first: сущности как POCO, `DbSet<T>` как «окно» в таблицу, `DbContext` как оркестратор, `OnModelCreating` с Fluent API и `ApplyConfigurationsFromAssembly`, а также `DbContextOptions` и регистрацию через `AddDbContext`. ДЗ требует собрать всё это в один рабочий проект консольного каталога курсов и пройти по каждому лучу best practices и частых ошибок из урока: Scoped-регистрация, приватные сеттеры, `SaveChangesAsync`, запрет хардкода паролей, запрет Singleton, выбор между `EnsureCreated` и миграциями.

(EN) The lesson introduces the core Code-first concepts: POCO entities, `DbSet<T>` as a window into a table, `DbContext` as the orchestrator, `OnModelCreating` with Fluent API and `ApplyConfigurationsFromAssembly`, plus `DbContextOptions` and `AddDbContext` registration. This homework asks you to assemble all of that into one working course-catalog console project and to walk through every best practice and common mistake from the lesson: Scoped registration, private setters, `SaveChangesAsync`, no hardcoded passwords, no Singleton, the choice between `EnsureCreated` and migrations.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Представьте, что вы присоединились к команде образовательного стартапа «CourseShop», который строит каталог онлайн-курсов на C# 12 / .NET 8. Команда уже выбрала EF Core 8 как ORM и подход Code-first: модель данных должна жить в коде, а схема базы должна генерироваться из классов. Техлид дал вам первый спринт — заложить фундамент доменного слоя и инфраструктуры доступа к данным. От качества этого фундамента зависит, насколько легко команда later будет вводить миграции, репозитории, тесты и feature-флаги.

Вам нужно спроектировать две сущности — `Category` (категория курсов, например «Backend», «Frontend») и `Course` (конкретный курс с ценой, длительностью и ссылкой на категорию). Категория содержит навигационную коллекцию курсов — это связь «один-ко-многим». Курс обязан иметь цену больше нуля, непустой заголовок и обязательную ссылку на категорию. Эти инварианты должны быть защищены на уровне сущностей, а не только валидацией в сервисах: приватные сеттеры и конструкторы с обязательными параметрами — обязательны.

Далее вы реализуете `CourseShopDbContext` с двумя `DbSet`, принимающий `DbContextOptions<CourseShopDbContext>` через конструктор. Вся тонкая настройка схемы выносится в классы `IEntityTypeConfiguration<T>` и подключается через `ApplyConfigurationsFromAssembly`. Строка подключения хранится в `appsettings.json`, читается через `IConfiguration`, а DbContext регистрируется в DI как Scoped. В демо-режиме допускается `EnsureCreated`, но в комментарии должно быть явно указано, что в продакшене используются миграции. Финальный шаг — сервис `CatalogService`, который добавляет курс, проверяет наличие категории и корректно вызывает `SaveChangesAsync`. Это упражнение даст вам мышечную память для всего модуля M12.

#### Что нужно сделать (пошагово)

1. **Создайте решение и проект.** Откройте терминал в рабочей папке и выполните:
   ```bash
   dotnet new sln -n CourseShop
   dotnet new console -n CourseShop.App -o src/CourseShop.App --framework net8.0
   dotnet sln add src/CourseShop.App/CourseShop.App.csproj
   cd src/CourseShop.App
   dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 8.0.*
   dotnet add package Microsoft.EntityFrameworkCore.Design --version 8.0.*
   dotnet add package Microsoft.Extensions.Configuration.Json
   dotnet add package Microsoft.Extensions.DependencyInjection
   dotnet add package Microsoft.Extensions.Hosting
   ```
   Команда `dotnet add package` должна вывести `PackageReference ... added to CourseShop.App.csproj` и версию. Если версия ниже 8.0 — проверьте `--version`.

2. **Добавьте `appsettings.json`.** В корне проекта создайте файл `appsettings.json` со строкой подключения к LocalDB (без пароля, `Trusted_Connection=True`), установите ему `CopyToOutputDirectory = PreserveNewest` в `.csproj`. Не хардкодьте пароли в коде — для демо достаточно Windows-аутентификации.

3. **Спроектируйте сущности.** В файле `Domain/Category.cs` создайте класс `Category` с приватным сеттером `Id`, свойством `Name` с приватным сеттером и конструктором, принимающим `name`. Добавьте метод `Rename(string newName)` с защитой от пустого значения. В файле `Domain/Course.cs` создайте класс `Course` с `Title`, `Price`, `CategoryId` и навигацией `Category`. Конструктор `Course` должен принимать `title`, `price` и `categoryId` и проверять, что `price > 0`. Используйте nullable-аннотации (`null!` для обязательных ссылок) и приватные сеттеры — инварианты инкапсулированы.

4. **Реализуйте конфигурации.** В `Infrastructure/Configurations/CategoryConfiguration.cs` и `CourseConfiguration.cs` реализуйте `IEntityTypeConfiguration<T>`. Для `Category`: имя таблицы `categories`, `Name` максимум 100 символов, `IsRequired`, уникальный индекс по `Name`. Для `Course`: имя таблицы `courses`, `Title` максимум 200, `Price` с точностью `(10,2)`, внешний ключ к `Category` через `CategoryId`, `OnDelete(DeleteBehavior.Restrict)` (без каскадного удаления), индекс по `Title`.

5. **Реализуйте DbContext.** В `Infrastructure/CourseShopDbContext.cs` создайте класс, наследующий `DbContext`, с конструктором `CourseShopDbContext(DbContextOptions<CourseShopDbContext> options) : base(options)`. Добавьте `DbSet<Category> Categories => Set<Category>();` и `DbSet<Course> Courses => Set<Course>();`. В `OnModelCreating` вызовите `modelBuilder.ApplyConfigurationsFromAssembly(typeof(CourseShopDbContext).Assembly);` и затем `base.OnModelCreating(modelBuilder);`.

6. **Настройте DI и подключение.** В `Program.cs` используйте top-level statements: постройте `Host`, зарегистрируйте `CourseShopDbContext` через `AddDbContext` с `UseSqlServer` и строкой из `IConfiguration`, зарегистрируйте `CatalogService`. Создайте scope, получите DbContext, вызовите `EnsureCreated()` с комментарием «demo only; use migrations in prod», и запустите сервис.

7. **Реализуйте `CatalogService`.** В `Services/CatalogService.cs` создайте класс, принимающий `CourseShopDbContext` через конструктор. Метод `AddCourseAsync(title, price, categoryName)` должен: найти категорию по имени через `FirstOrDefaultAsync`, бросить `InvalidOperationException`, если не найдена; создать `Course`, добавить в `Courses`, вызвать `await SaveChangesAsync()` и вернуть число затронутых строк. Метод `ListCoursesAsync` должен возвращать курсы с категорией через `Include(c => c.Category)`.

8. **Проверьте запуск.** Выполните `dotnet build`, затем `dotnet run`. Ожидаемый вывод: «Added 1 row(s)» и список курсов с категориями. Если выводится исключение о Singleton — проверьте регистрацию. Если категория не найдена — убедитесь, что в `EnsureCreated` после создания вы добавили категорию.

9. **Проверьте конфигурацию.** Откройте базу через SQL Server Object Explorer или `dotnet ef dbcontext info`. Убедитесь: таблицы `categories` и `courses`, индексы по `Name` и `Title`, внешний ключ с `Restrict`, тип `decimal(10,2)` для цены.

#### Требования к решению

- Проект компилируется без warning’ов `nullable` и работает под .NET 8 на C# 12 (top-level statements, nullable reference types, можно collection expressions для инициализации).
- Сущности `Category` и `Course` — POCO без атрибутов DataAnnotations (используем только Fluent API), с приватными сеттерами и конструкторами, защищающими инварианты (пустое имя, отрицательная цена).
- Навигационные свойства инициализированы как `null!` (ссылочные) и `new List<T>()` (коллекции), чтобы удовлетворить nullable-анализ.
- `CourseShopDbContext` принимает `DbContextOptions<CourseShopDbContext>`, не имеет дефолтного конструктора, не вызывает `UseSqlServer` внутри себя — провайдер задаётся только через DI.
- `OnModelCreating` делегирует конфигурацию в `IEntityTypeConfiguration<T>` через `ApplyConfigurationsFromAssembly` — внутри метода нет ручной настройки свойств.
- Строка подключения хранится в `appsettings.json`, читается через `IConfiguration.GetConnectionString("Default")`. Никаких паролей в исходниках.
- DbContext зарегистрирован как Scoped через `AddDbContext` — не Singleton, не Transient, не `new AppDbContext()` вручную.
- Используется `SaveChangesAsync` с `await`. В асинхронных путях синхронный `SaveChanges` запрещён.
- В демо допускается `EnsureCreated`, но в коде или комментарии явно отмечено: «в продакшене — миграции».
- Внешний ключ `Course.CategoryId` настроен с `OnDelete(DeleteBehavior.Restrict)`, чтобы предотвратить каскадное удаление курсов при удалении категории.
- `CatalogService` использует `Include` для загрузки навигации в `ListCoursesAsync`, чтобы избежать `null`/пустой коллекции (частая ошибка из урока).

#### Тонкости и подводные камни

- **Singleton DbContext — гонки данных.** Если случайно зарегистрировать контекст как `services.AddSingleton<AppDbContext>(...)`, два потока начнут делить один Change Tracker — появятся `InvalidOperationException` и утечки отслеживаемых сущностей. Только `AddDbContext` (по умолчанию Scoped).
- **Забытый `await`.** `SaveChangesAsync()` без `await` вернёт `Task`, который может не выполниться до завершения метода — изменения не сохранятся. Включите анализ `CS4014` или включите `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`.
- **Навигация без `Include`.** Запрос `db.Courses.ToListAsync()` оставит `Category` равным `null` — EF Core по умолчанию не подгружает связанные сущности (lazy loading выключен). Решение: `Include(c => c.Category)` или `Select`-проекция, либо явная загрузка `Entry(course).Reference(c => c.Category).LoadAsync()`.
- **`EnsureCreated` блокирует миграции.** Если вы когда-нибудь вызовете `EnsureCreated`, а потом попытаетесь `dotnet ef migrations add`, EF Core скажет, что таблицы уже есть и миграции несовместимы. Выберите один путь: демо с `EnsureCreated` или продакшен с миграциями (это тема следующего урока M12-L03).
- **Хардкод пароля.** Строка вида `Password=...;` в `Program.cs` — утечка секрета в git. Используйте User Secrets (`dotnet user-secrets init`, `dotnet user-secrets set "ConnectionStrings:Default" "..."`) или переменные окружения.
- **Два контекста, одна сущность.** Если вы загрузите `Course` в одном DbContext, а потом попытаетесь обновить его в другом — получите «instance already tracked». Решение: работайте в одном scope или используйте `AsNoTracking()` для чтения и `.Attach()`/`.Update()` с осторожностью.
- **`HasPrecision(10,2)` для денег.** Никогда не храните деньги в `float`/`double` — только `decimal` + `HasPrecision`. Иначе копейки «уплывут».
- **Уникальный индекс vs обычный.** `HasIndex(x => x.Name)` создаёт обычный индекс; для уникальности нужен `.IsUnique()`. Проверьте в DDL, что индекс `IX_categories_Name` имеет `UNIQUE`.
- **Nullable-предупреждения на навигации.** `public Category Category { get; private set; } = null!;` — `null!` успокаивает компилятор, но в рантайме EF Core заполнит свойство после загрузки. Альтернатива — `#nullable disable` на свойстве или конструктор, принимающий навигацию.
- **`Set<T>()` vs автосвойство.** `public DbSet<Category> Categories => Set<Category>();` — expression-bodied свойство, лениво создаёт DbSet. Равноценно `public DbSet<Category> Categories { get; set; }`, но `Set<T>()` явно показывает намерение и не позволяет внешнему коду перезаписать DbSet.

#### Критерии приёмки

- [ ] Проект `CourseShop.App` собирается командой `dotnet build` без ошибок и nullable-warning’ов.
- [ ] Целевая платформа — `net8.0`, LangVersion 12 (top-level statements работают).
- [ ] Установлен `Microsoft.EntityFrameworkCore.SqlServer` версии 8.x.
- [ ] Сущность `Category` — POCO с приватными сеттерами, конструктором и методом `Rename` с валидацией.
- [ ] Сущность `Course` — POCO с инвариантом `price > 0` в конструкторе.
- [ ] Навигационные свойства помечены `null!`/инициализированы `new List<T>()`.
- [ ] `CategoryConfiguration` и `CourseConfiguration` реализуют `IEntityTypeConfiguration<T>`.
- [ ] `OnModelCreating` вызывает только `ApplyConfigurationsFromAssembly`, без ручной настройки.
- [ ] Внешний ключ `Course → Category` настроен с `OnDelete(DeleteBehavior.Restrict)`.
- [ ] `Price` имеет `HasPrecision(10, 2)`, `Title` — `HasMaxLength(200)`.
- [ ] Уникальный индекс по `Category.Name` и обычный по `Course.Title`.
- [ ] `CourseShopDbContext` принимает `DbContextOptions<CourseShopDbContext>`.
- [ ] DbContext зарегистрирован через `AddDbContext` (Scoped по умолчанию), не Singleton.
- [ ] Строка подключения в `appsettings.json`, читается через `GetConnectionString`, без пароля в коде.
- [ ] `CatalogService.AddCourseAsync` вызывает `await SaveChangesAsync()` и возвращает число строк.
- [ ] `ListCoursesAsync` использует `Include(c => c.Category)`.
- [ ] В коде есть комментарий про `EnsureCreated` → миграции в продакшене.
- [ ] `dotnet run` выводит добавленный курс со списком категорий.

#### Подсказки (без прямого ответа)

- Для построения хоста используйте `Host.CreateDefaultBuilder(args)` — он автоматически загрузит `appsettings.json` в `IConfiguration`.
- Чтобы получить строку, в `Program.cs` вызовите `builder.Configuration.GetConnectionString("Default")` до `Build()`.
- Для поиска категории используйте `_db.Categories.FirstOrDefaultAsync(c => c.Name == categoryName)`.
- `SaveChangesAsync` возвращает `int` — число затронутых строк; верните его из метода, чтобы тест мог проверить.
- Для уникального индекса в конфигурации: `.HasIndex(x => x.Name).IsUnique()`.
- Для проверки, что индекс создан, выполните `dotnet ef migrations script --idempotent` (если миграции включены) или посмотрите DDL в SQL Server Object Explorer.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — Эталонное ДЗ M12-L02
// Reference solution: Code-first, entities, DbContext, connection
// Двуязычные комментарии / Bilingual comments

using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using CourseShop.Domain;
using CourseShop.Infrastructure;
using CourseShop.Services;

// ── Program.cs (top-level statements) / Program entry ──
var builder = Host.CreateDefaultBuilder(args);

builder.ConfigureServices((ctx, services) =>
{
    // Строка подключения из IConfiguration / connection string from config
    var cs = ctx.Configuration.GetConnectionString("Default")
        ?? throw new InvalidOperationException("ConnectionStrings:Default not configured");

    // Scoped по умолчанию / Scoped by default (никогда не Singleton)
    services.AddDbContext<CourseShopDbContext>(opt =>
        opt.UseSqlServer(cs));

    services.AddScoped<CatalogService>();
});

var host = builder.Build();

// Демо: создаём схему / Demo: create schema
// В продакшене — миграции (см. урок M12-L03) / In prod — migrations
using (var scope = host.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<CourseShopDbContext>();
    await db.Database.EnsureCreatedAsync(); // demo only; use MigrateAsync() in prod

    // Сидим одну категорию, если пусто / seed a category if empty
    if (!await db.Categories.AnyAsync())
    {
        db.Categories.Add(new Category("Backend"));
        await db.SaveChangesAsync();
    }

    var svc = scope.ServiceProvider.GetRequiredService<CatalogService>();
    var affected = await svc.AddCourseAsync("C# 12 in Depth", 49.99m, "Backend");
    Console.WriteLine($"Added {affected} row(s)");

    await foreach (var c in svc.ListCoursesAsync())
        Console.WriteLine($"{c.Title} — {c.Price:C} — {c.Category?.Name}");
}

await host.RunAsync();

// ── Domain/Category.cs ──
namespace CourseShop.Domain;

public class Category
{
    public int Id { get; private set; }                  // PK по соглашению / PK by convention
    public string Name { get; private set; } = null!;    // обязательное / required

    // Навигация «один-ко-многим» / one-to-many navigation
    public ICollection<Course> Courses { get; private set; } = new List<Course>();

    public Category(string name)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Имя не может быть пустым / Name cannot be empty");
        Name = name;
    }

    public void Rename(string newName)
    {
        if (string.IsNullOrWhiteSpace(newName))
            throw new ArgumentException("Имя не может быть пустым / Name cannot be empty");
        Name = newName;
    }
}

// ── Domain/Course.cs ──
public class Course
{
    public int Id { get; private set; }
    public string Title { get; private set; } = null!;
    public decimal Price { get; private set; }

    public int CategoryId { get; private set; }          // FK
    public Category Category { get; private set; } = null!; // навигация / navigation

    public Course(string title, decimal price, int categoryId)
    {
        if (string.IsNullOrWhiteSpace(title))
            throw new ArgumentException("Заголовок пуст / Title is empty");
        if (price <= 0)
            throw new ArgumentOutOfRangeException(nameof(price), "Цена должна быть > 0 / Price must be > 0");
        Title = title;
        Price = price;
        CategoryId = categoryId;
    }

    public void ChangePrice(decimal newPrice)
    {
        if (newPrice <= 0)
            throw new ArgumentOutOfRangeException(nameof(newPrice), "Цена должна быть > 0");
        Price = newPrice;
    }
}

// ── Infrastructure/CourseShopDbContext.cs ──
namespace CourseShop.Infrastructure;

public class CourseShopDbContext : DbContext
{
    public DbSet<Category> Categories => Set<Category>();
    public DbSet<Course> Courses => Set<Course>();

    public CourseShopDbContext(DbContextOptions<CourseShopDbContext> options)
        : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Делегируем в IEntityTypeConfiguration<T> / delegate to configurations
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(CourseShopDbContext).Assembly);
        base.OnModelCreating(modelBuilder);
    }
}

// ── Infrastructure/Configurations/CategoryConfiguration.cs ──
public class CategoryConfiguration : IEntityTypeConfiguration<Domain.Category>
{
    public void Configure(EntityTypeBuilder<Domain.Category> b)
    {
        b.ToTable("categories");
        b.HasKey(x => x.Id);
        b.Property(x => x.Name).HasMaxLength(100).IsRequired();
        b.HasIndex(x => x.Name).IsUnique();              // уникальный / unique
        b.HasMany(x => x.Courses)
         .WithOne(c => c.Category)
         .HasForeignKey(c => c.CategoryId)
         .OnDelete(DeleteBehavior.Restrict);            // без каскада / no cascade
    }
}

// ── Infrastructure/Configurations/CourseConfiguration.cs ──
public class CourseConfiguration : IEntityTypeConfiguration<Domain.Course>
{
    public void Configure(EntityTypeBuilder<Domain.Course> b)
    {
        b.ToTable("courses");
        b.HasKey(x => x.Id);
        b.Property(x => x.Title).HasMaxLength(200).IsRequired();
        b.Property(x => x.Price).HasPrecision(10, 2);    // деньги / money
        b.HasIndex(x => x.Title);                        // обычный индекс / non-unique index
    }
}

// ── Services/CatalogService.cs ──
namespace CourseShop.Services;

public class CatalogService
{
    private readonly CourseShopDbContext _db;
    public CatalogService(CourseShopDbContext db) => _db = db;

    public async Task<int> AddCourseAsync(string title, decimal price, string categoryName)
    {
        var category = await _db.Categories.FirstOrDefaultAsync(c => c.Name == categoryName)
            ?? throw new InvalidOperationException($"Категория '{categoryName}' не найдена / Category not found");

        _db.Courses.Add(new Course(title, price, category.Id));
        return await _db.SaveChangesAsync();             // число строк / rows affected
    }

    // Include обязателен, иначе Category == null / Include is mandatory
    public IAsyncEnumerable<Course> ListCoursesAsync() =>
        _db.Courses
           .Include(c => c.Category)
           .AsAsyncEnumerable();
}
```

**Разбор по строкам (почему так).** Топ-левел `Program.cs` соответствует уроку: мы используем `Host.CreateDefaultBuilder`, который автоматически подгружает `appsettings.json` в `IConfiguration`, после чего `ConfigureServices` регистрирует `CourseShopDbContext` через `AddDbContext` с `UseSqlServer`. Это воспроизводит best practice «Scoped через DI», а строка подключения берётся из конфига — никакого хардкода пароля. Внутри scope мы вызываем `EnsureCreatedAsync` с комментарием «demo only; use MigrateAsync() in prod» — урок явно предупреждает, что `EnsureCreated` блокирует миграции, поэтому комментарий обязателен.

Сущность `Category` — классический POCO: `Id` с приватным сеттером (EF Core пишет через reflection), `Name` с приватным сеттером и конструктором, который выбрасывает `ArgumentException` на пустую строку. Это реализует best practice «инкапсулируйте инварианты». Навигационная коллекция `Courses` инициализирована `new List<Course>()` — это успокаивает nullable-анализатор и позволяет добавлять курсы через `category.Courses.Add(...)`, если понадобится. Метод `Rename` дублирует проверку — инвариант действует и после создания.

`Course` дополнительно защищает инвариант `price > 0` в конструкторе и в `ChangePrice`. Свойство `Category` помечено `null!` — урок упоминает эту идиому для обязательных навигаций. Внешний ключ `CategoryId` — отдельное свойство, что даёт EF Core явный FK-столбец (а не теневой).

`CourseShopDbContext` принимает `DbContextOptions<CourseShopDbContext>` — это позволяет DI собрать опции через `AddDbContext`. `DbSet`-свойства реализованы как `Set<T>()` через expression-bodied — соответствует best practice урока. В `OnModelCreating` только `ApplyConfigurationsFromAssembly` — конфигурация вынесена в отдельные классы, что сохраняет метод читаемым (ещё один best practice).

`CategoryConfiguration` задаёт имя таблицы `categories` (lowercase), уникальный индекс по `Name` и связь `HasMany...WithOne...HasForeignKey...OnDelete(Restrict)` — запрет каскадного удаления, как в примере урока. `CourseConfiguration` фиксирует `decimal(10,2)` для цены — критично для денег, и обычный индекс по `Title` для поиска.

`CatalogService` повторяет пример из урока: находит категорию, бросает исключение, если её нет, добавляет `Course` и возвращает результат `await SaveChangesAsync()`. `ListCoursesAsync` использует `Include(c => c.Category)` — это закрывает частую ошибку «навигация без Include даёт null». Возврат `IAsyncEnumerable` через `AsAsyncEnumerable()` эффективен для стриминга больших наборов. Вся логика асинхронна — синхронный `SaveChanges` отсутствует, что соответствует best practice урока.

#### Задания на углубление (бонус)

1. **Owned types.** Вынесите цену и валюту в `Money` value-object через `OwnsOne(c => c.Price)`. Сравните DDL — теперь `Price_Amount` и `Price_Currency` вместо одного столбца.
2. **Logging SQL.** Включите `opt.LogTo(Console.WriteLine, LogLevel.Information)` и проследите, какие SQL-запросы генерирует `Include`. Найдите место, где появляется `LEFT JOIN`.
3. **AsNoTracking.** Добавьте метод `ListCoursesReadOnlyAsync`, использующий `.AsNoTracking()`, и сравните производительность через `Stopwatch` на 10 000 строк.
4. **User Secrets.** Перенесите `ConnectionStrings:Default` в User Secrets (`dotnet user-secrets`), удалите из `appsettings.json` и убедитесь, что приложение всё ещё работает в dev-окружении.

---

## Statement in English / Постановка на английском

#### Context & motivation

Imagine you have joined an educational startup called “CourseShop” that is building a catalog of online courses on C# 12 / .NET 8. The team has already chosen EF Core 8 as its ORM and the Code-first approach: the data model must live in code, and the database schema must be generated from the classes. The tech lead has given you the first sprint — lay the foundation of the domain layer and the data-access infrastructure. The quality of this foundation determines how easily the team will later introduce migrations, repositories, tests, and feature flags.

You need to design two entities — `Category` (a course category, e.g. “Backend”, “Frontend”) and `Course` (a specific course with a price, duration, and a reference to the category). The category contains a navigational collection of courses — this is a one-to-many relationship. A course must have a price greater than zero, a non-empty title, and a mandatory reference to a category. These invariants must be protected at the entity level, not only by service-layer validation: private setters and constructors with required parameters are mandatory.

Next, you implement `CourseShopDbContext` with two `DbSet`s, accepting `DbContextOptions<CourseShopDbContext>` through its constructor. All fine-grained schema configuration is moved into `IEntityTypeConfiguration<T>` classes and registered via `ApplyConfigurationsFromAssembly`. The connection string is stored in `appsettings.json`, read through `IConfiguration`, and the DbContext is registered in DI as Scoped. In demo mode `EnsureCreated` is acceptable, but a comment must explicitly state that migrations are used in production. The final step is a `CatalogService` that adds a course, validates the category existence, and correctly calls `SaveChangesAsync`. This exercise gives you the muscle memory for the whole M12 module.

#### What to do step by step

1. **Create the solution and project.** Open a terminal in your working folder and run:
   ```bash
   dotnet new sln -n CourseShop
   dotnet new console -n CourseShop.App -o src/CourseShop.App --framework net8.0
   dotnet sln add src/CourseShop.App/CourseShop.App.csproj
   cd src/CourseShop.App
   dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 8.0.*
   dotnet add package Microsoft.EntityFrameworkCore.Design --version 8.0.*
   dotnet add package Microsoft.Extensions.Configuration.Json
   dotnet add package Microsoft.Extensions.DependencyInjection
   dotnet add package Microsoft.Extensions.Hosting
   ```
   The `dotnet add package` command should print `PackageReference ... added to CourseShop.App.csproj` along with the version. If the version is below 8.0, double-check the `--version` flag.

2. **Add `appsettings.json`.** At the project root create `appsettings.json` with a LocalDB connection string (no password, `Trusted_Connection=True`) and set `CopyToOutputDirectory = PreserveNewest` in the `.csproj`. Never hardcode passwords in code — Windows authentication is enough for the demo.

3. **Design the entities.** In `Domain/Category.cs` create a `Category` class with a private-setter `Id`, a `Name` property with a private setter, and a constructor that takes `name`. Add a `Rename(string newName)` method that guards against an empty value. In `Domain/Course.cs` create a `Course` class with `Title`, `Price`, `CategoryId`, and the `Category` navigation. The `Course` constructor must take `title`, `price`, and `categoryId`, and validate that `price > 0`. Use nullable annotations (`null!` for required references) and private setters — invariants are encapsulated.

4. **Implement the configurations.** In `Infrastructure/Configurations/CategoryConfiguration.cs` and `CourseConfiguration.cs` implement `IEntityTypeConfiguration<T>`. For `Category`: table name `categories`, `Name` max length 100, `IsRequired`, a unique index on `Name`. For `Course`: table name `courses`, `Title` max length 200, `Price` with precision `(10,2)`, a foreign key to `Category` via `CategoryId`, `OnDelete(DeleteBehavior.Restrict)` (no cascade delete), and an index on `Title`.

5. **Implement the DbContext.** In `Infrastructure/CourseShopDbContext.cs` create a class deriving from `DbContext` with the constructor `CourseShopDbContext(DbContextOptions<CourseShopDbContext> options) : base(options)`. Add `DbSet<Category> Categories => Set<Category>();` and `DbSet<Course> Courses => Set<Course>();`. In `OnModelCreating` call `modelBuilder.ApplyConfigurationsFromAssembly(typeof(CourseShopDbContext).Assembly);` followed by `base.OnModelCreating(modelBuilder);`.

6. **Configure DI and the connection.** In `Program.cs` use top-level statements: build a `Host`, register `CourseShopDbContext` via `AddDbContext` with `UseSqlServer` and the connection string from `IConfiguration`, register `CatalogService`. Create a scope, resolve the DbContext, call `EnsureCreated()` with the comment “demo only; use migrations in prod”, and run the service.

7. **Implement `CatalogService`.** In `Services/CatalogService.cs` create a class that takes `CourseShopDbContext` through its constructor. The method `AddCourseAsync(title, price, categoryName)` must: find the category by name using `FirstOrDefaultAsync`, throw `InvalidOperationException` if not found; create a `Course`, add it to `Courses`, call `await SaveChangesAsync()`, and return the number of affected rows. The method `ListCoursesAsync` must return courses with their category via `Include(c => c.Category)`.

8. **Verify the run.** Run `dotnet build`, then `dotnet run`. Expected output: “Added 1 row(s)” and a list of courses with categories. If a Singleton exception appears, check the registration. If the category is not found, make sure you add it after `EnsureCreated`.

9. **Verify the schema.** Open the database through SQL Server Object Explorer or `dotnet ef dbcontext info`. Confirm: tables `categories` and `courses`, indexes on `Name` and `Title`, a foreign key with `Restrict`, and a `decimal(10,2)` type for the price.

#### Requirements

- The project compiles without `nullable` warnings and runs on .NET 8 with C# 12 (top-level statements, nullable reference types; collection expressions may be used for initialization).
- The `Category` and `Course` entities are POCOs without DataAnnotations attributes (Fluent API only), with private setters and constructors that enforce invariants (empty name, negative price).
- Navigational properties are initialized as `null!` (references) and `new List<T>()` (collections) to satisfy nullable analysis.
- `CourseShopDbContext` accepts `DbContextOptions<CourseShopDbContext>`, has no default constructor, and does not call `UseSqlServer` internally — the provider is set only through DI.
- `OnModelCreating` delegates configuration to `IEntityTypeConfiguration<T>` via `ApplyConfigurationsFromAssembly` — no manual property setup inside the method.
- The connection string is stored in `appsettings.json` and read via `IConfiguration.GetConnectionString("Default")`. No passwords in source code.
- The DbContext is registered as Scoped through `AddDbContext` — not Singleton, not Transient, not `new AppDbContext()` manually.
- `SaveChangesAsync` is used with `await`. Synchronous `SaveChanges` is forbidden on async paths.
- In the demo `EnsureCreated` is allowed, but the code or a comment must explicitly say “migrations in production”.
- The foreign key `Course.CategoryId` is configured with `OnDelete(DeleteBehavior.Restrict)` to prevent cascade delete of courses when a category is deleted.
- `CatalogService` uses `Include` to load the navigation in `ListCoursesAsync` to avoid `null`/empty collection (a common mistake from the lesson).

#### Pitfalls

- **Singleton DbContext — data races.** If you accidentally register the context as `services.AddSingleton<AppDbContext>(...)`, two threads will share a single Change Tracker — `InvalidOperationException` and tracked-entity leaks appear. Use `AddDbContext` only (Scoped by default).
- **Forgotten `await`.** `SaveChangesAsync()` without `await` returns a `Task` that may not complete before the method ends — changes are not persisted. Enable `CS4014` analysis or `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`.
- **Navigation without `Include`.** `db.Courses.ToListAsync()` leaves `Category` as `null` — EF Core does not load related entities by default (lazy loading is off). Use `Include(c => c.Category)`, a `Select` projection, or explicit loading `Entry(course).Reference(c => c.Category).LoadAsync()`.
- **`EnsureCreated` blocks migrations.** If you ever call `EnsureCreated` and later run `dotnet ef migrations add`, EF Core will complain that tables already exist and migrations are incompatible. Pick one path: demo with `EnsureCreated` or production with migrations (the topic of the next lesson M12-L03).
- **Hardcoded password.** A string like `Password=...;` in `Program.cs` is a secret leak into git. Use User Secrets (`dotnet user-secrets init`, `dotnet user-secrets set "ConnectionStrings:Default" "..."`) or environment variables.
- **Two contexts, one entity.** If you load a `Course` in one DbContext and then try to update it in another, you get “instance already tracked”. Work in one scope or use `AsNoTracking()` for reads and `.Attach()`/`.Update()` carefully.
- **`HasPrecision(10,2)` for money.** Never store money in `float`/`double` — only `decimal` plus `HasPrecision`. Otherwise cents drift.
- **Unique vs non-unique index.** `HasIndex(x => x.Name)` creates a normal index; for uniqueness you need `.IsUnique()`. Verify in the DDL that `IX_categories_Name` has `UNIQUE`.
- **Nullable warnings on navigation.** `public Category Category { get; private set; } = null!;` — the `null!` silences the compiler, but at runtime EF Core fills the property after loading. Alternatives: `#nullable disable` on the property or a constructor that accepts the navigation.
- **`Set<T>()` vs auto-property.** `public DbSet<Category> Categories => Set<Category>();` is an expression-bodied property that lazily creates the DbSet. It is equivalent to `public DbSet<Category> Categories { get; set; }`, but `Set<T>()` makes the intent explicit and prevents external code from overwriting the DbSet.

#### Acceptance criteria

- [ ] The `CourseShop.App` project builds with `dotnet build` without errors and without nullable warnings.
- [ ] The target framework is `net8.0`, LangVersion 12 (top-level statements work).
- [ ] `Microsoft.EntityFrameworkCore.SqlServer` version 8.x is installed.
- [ ] The `Category` entity is a POCO with private setters, a constructor, and a validating `Rename` method.
- [ ] The `Course` entity is a POCO with the `price > 0` invariant in the constructor.
- [ ] Navigational properties are marked `null!` / initialized with `new List<T>()`.
- [ ] `CategoryConfiguration` and `CourseConfiguration` implement `IEntityTypeConfiguration<T>`.
- [ ] `OnModelCreating` calls only `ApplyConfigurationsFromAssembly`, with no manual setup.
- [ ] The `Course → Category` foreign key is configured with `OnDelete(DeleteBehavior.Restrict)`.
- [ ] `Price` has `HasPrecision(10, 2)`, `Title` has `HasMaxLength(200)`.
- [ ] A unique index on `Category.Name` and a normal index on `Course.Title`.
- [ ] `CourseShopDbContext` accepts `DbContextOptions<CourseShopDbContext>`.
- [ ] The DbContext is registered via `AddDbContext` (Scoped by default), not Singleton.
- [ ] The connection string is in `appsettings.json`, read via `GetConnectionString`, no password in code.
- [ ] `CatalogService.AddCourseAsync` calls `await SaveChangesAsync()` and returns the number of rows.
- [ ] `ListCoursesAsync` uses `Include(c => c.Category)`.
- [ ] The code includes a comment about `EnsureCreated` → migrations in production.
- [ ] `dotnet run` prints the added course with the list of categories.

#### Hints (no direct answer)

- To build the host, use `Host.CreateDefaultBuilder(args)` — it automatically loads `appsettings.json` into `IConfiguration`.
- To get the connection string, in `Program.cs` call `builder.Configuration.GetConnectionString("Default")` before `Build()`.
- To find the category, use `_db.Categories.FirstOrDefaultAsync(c => c.Name == categoryName)`.
- `SaveChangesAsync` returns an `int` — the number of affected rows; return it so tests can assert.
- For a unique index in the configuration: `.HasIndex(x => x.Name).IsUnique()`.
- To verify the index, run `dotnet ef migrations script --idempotent` (if migrations are enabled) or inspect the DDL in SQL Server Object Explorer.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — Reference homework M12-L02
// Reference solution: Code-first, entities, DbContext, connection
// Bilingual comments / Bilingual comments

using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using CourseShop.Domain;
using CourseShop.Infrastructure;
using CourseShop.Services;

// ── Program.cs (top-level statements) ──
var builder = Host.CreateDefaultBuilder(args);

builder.ConfigureServices((ctx, services) =>
{
    // Connection string from IConfiguration
    var cs = ctx.Configuration.GetConnectionString("Default")
        ?? throw new InvalidOperationException("ConnectionStrings:Default not configured");

    // Scoped by default (never Singleton)
    services.AddDbContext<CourseShopDbContext>(opt =>
        opt.UseSqlServer(cs));

    services.AddScoped<CatalogService>();
});

var host = builder.Build();

// Demo: create schema / In prod — migrations (see lesson M12-L03)
using (var scope = host.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<CourseShopDbContext>();
    await db.Database.EnsureCreatedAsync(); // demo only; use MigrateAsync() in prod

    // Seed a category if the table is empty
    if (!await db.Categories.AnyAsync())
    {
        db.Categories.Add(new Category("Backend"));
        await db.SaveChangesAsync();
    }

    var svc = scope.ServiceProvider.GetRequiredService<CatalogService>();
    var affected = await svc.AddCourseAsync("C# 12 in Depth", 49.99m, "Backend");
    Console.WriteLine($"Added {affected} row(s)");

    await foreach (var c in svc.ListCoursesAsync())
        Console.WriteLine($"{c.Title} — {c.Price:C} — {c.Category?.Name}");
}

await host.RunAsync();

// ── Domain/Category.cs ──
namespace CourseShop.Domain;

public class Category
{
    public int Id { get; private set; }                  // PK by convention
    public string Name { get; private set; } = null!;    // required

    // One-to-many navigation
    public ICollection<Course> Courses { get; private set; } = new List<Course>();

    public Category(string name)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Name cannot be empty");
        Name = name;
    }

    public void Rename(string newName)
    {
        if (string.IsNullOrWhiteSpace(newName))
            throw new ArgumentException("Name cannot be empty");
        Name = newName;
    }
}

// ── Domain/Course.cs ──
public class Course
{
    public int Id { get; private set; }
    public string Title { get; private set; } = null!;
    public decimal Price { get; private set; }

    public int CategoryId { get; private set; }          // FK
    public Category Category { get; private set; } = null!; // navigation

    public Course(string title, decimal price, int categoryId)
    {
        if (string.IsNullOrWhiteSpace(title))
            throw new ArgumentException("Title is empty");
        if (price <= 0)
            throw new ArgumentOutOfRangeException(nameof(price), "Price must be > 0");
        Title = title;
        Price = price;
        CategoryId = categoryId;
    }

    public void ChangePrice(decimal newPrice)
    {
        if (newPrice <= 0)
            throw new ArgumentOutOfRangeException(nameof(newPrice), "Price must be > 0");
        Price = newPrice;
    }
}

// ── Infrastructure/CourseShopDbContext.cs ──
namespace CourseShop.Infrastructure;

public class CourseShopDbContext : DbContext
{
    public DbSet<Category> Categories => Set<Category>();
    public DbSet<Course> Courses => Set<Course>();

    public CourseShopDbContext(DbContextOptions<CourseShopDbContext> options)
        : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Delegate to IEntityTypeConfiguration<T>
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(CourseShopDbContext).Assembly);
        base.OnModelCreating(modelBuilder);
    }
}

// ── Infrastructure/Configurations/CategoryConfiguration.cs ──
public class CategoryConfiguration : IEntityTypeConfiguration<Domain.Category>
{
    public void Configure(EntityTypeBuilder<Domain.Category> b)
    {
        b.ToTable("categories");
        b.HasKey(x => x.Id);
        b.Property(x => x.Name).HasMaxLength(100).IsRequired();
        b.HasIndex(x => x.Name).IsUnique();              // unique
        b.HasMany(x => x.Courses)
         .WithOne(c => c.Category)
         .HasForeignKey(c => c.CategoryId)
         .OnDelete(DeleteBehavior.Restrict);            // no cascade
    }
}

// ── Infrastructure/Configurations/CourseConfiguration.cs ──
public class CourseConfiguration : IEntityTypeConfiguration<Domain.Course>
{
    public void Configure(EntityTypeBuilder<Domain.Course> b)
    {
        b.ToTable("courses");
        b.HasKey(x => x.Id);
        b.Property(x => x.Title).HasMaxLength(200).IsRequired();
        b.Property(x => x.Price).HasPrecision(10, 2);    // money
        b.HasIndex(x => x.Title);                        // non-unique index
    }
}

// ── Services/CatalogService.cs ──
namespace CourseShop.Services;

public class CatalogService
{
    private readonly CourseShopDbContext _db;
    public CatalogService(CourseShopDbContext db) => _db = db;

    public async Task<int> AddCourseAsync(string title, decimal price, string categoryName)
    {
        var category = await _db.Categories.FirstOrDefaultAsync(c => c.Name == categoryName)
            ?? throw new InvalidOperationException($"Category '{categoryName}' not found");

        _db.Courses.Add(new Course(title, price, category.Id));
        return await _db.SaveChangesAsync();             // rows affected
    }

    // Include is mandatory, otherwise Category == null
    public IAsyncEnumerable<Course> ListCoursesAsync() =>
        _db.Courses
           .Include(c => c.Category)
           .AsAsyncEnumerable();
}
```

**Line-by-line walk-through (why this way).** The top-level `Program.cs` mirrors the lesson: we use `Host.CreateDefaultBuilder`, which automatically loads `appsettings.json` into `IConfiguration`, and then `ConfigureServices` registers `CourseShopDbContext` through `AddDbContext` with `UseSqlServer`. This reproduces the “Scoped via DI” best practice, and the connection string comes from configuration — no hardcoded password. Inside the scope we call `EnsureCreatedAsync` with the comment “demo only; use MigrateAsync() in prod” — the lesson explicitly warns that `EnsureCreated` blocks migrations, so the comment is mandatory.

The `Category` entity is a classic POCO: `Id` with a private setter (EF Core writes through reflection), `Name` with a private setter and a constructor that throws `ArgumentException` on an empty string. This implements the “encapsulate invariants” best practice. The `Courses` navigation collection is initialized with `new List<Course>()` — this soothes the nullable analyzer and allows adding courses via `category.Courses.Add(...)` if needed. The `Rename` method duplicates the check — the invariant stays in force after construction.

`Course` additionally guards the `price > 0` invariant in both the constructor and `ChangePrice`. The `Category` property is marked `null!` — the lesson mentions this idiom for required navigations. The `CategoryId` is a separate property, giving EF Core an explicit FK column (rather than a shadow one).

`CourseShopDbContext` accepts `DbContextOptions<CourseShopDbContext>` — this lets DI assemble the options through `AddDbContext`. The `DbSet` properties use `Set<T>()` via expression bodies — matching the lesson best practice. In `OnModelCreating` we only call `ApplyConfigurationsFromAssembly` — configuration is moved into separate classes, which keeps the method readable (another best practice).

`CategoryConfiguration` sets the table name to `categories` (lowercase), a unique index on `Name`, and the relationship `HasMany...WithOne...HasForeignKey...OnDelete(Restrict)` — disabling cascade delete, as in the lesson example. `CourseConfiguration` fixes `decimal(10,2)` for the price — critical for money — and a non-unique index on `Title` for search.

`CatalogService` reproduces the lesson example: it finds the category, throws if missing, adds a `Course`, and returns the result of `await SaveChangesAsync()`. `ListCoursesAsync` uses `Include(c => c.Category)` — this closes the common mistake “navigation without Include gives null”. Returning `IAsyncEnumerable` via `AsAsyncEnumerable()` is efficient for streaming large result sets. All logic is asynchronous — no synchronous `SaveChanges`, matching the lesson best practice.

#### Going deeper (bonus)

1. **Owned types.** Move price and currency into a `Money` value object via `OwnsOne(c => c.Price)`. Compare the DDL — you now have `Price_Amount` and `Price_Currency` instead of a single column.
2. **Logging SQL.** Enable `opt.LogTo(Console.WriteLine, LogLevel.Information)` and trace the SQL generated by `Include`. Find where the `LEFT JOIN` appears.
3. **AsNoTracking.** Add a `ListCoursesReadOnlyAsync` method using `.AsNoTracking()` and benchmark it with `Stopwatch` on 10 000 rows.
4. **User Secrets.** Move `ConnectionStrings:Default` into User Secrets (`dotnet user-secrets`), remove it from `appsettings.json`, and confirm the app still runs in the dev environment.

---

#### Чек-лист сдачи / Submission checklist

- [ ] (RU) Проект `CourseShop.App` собирается без ошибок и nullable-warning’ов.
- [ ] (RU) Сущности POCO с приватными сеттерами и инвариантами в конструкторе.
- [ ] (RU) Конфигурации в `IEntityTypeConfiguration<T>`, `OnModelCreating` делегирует.
- [ ] (RU) DbContext принимает `DbContextOptions<T>`, зарегистрирован через `AddDbContext` (Scoped).
- [ ] (RU) Строка подключения в `appsettings.json`, без пароля в коде.
- [ ] (RU) `SaveChangesAsync` с `await`, `Include` в `ListCoursesAsync`.
- [ ] (RU) Комментарий про `EnsureCreated` → миграции в продакшене.
- [ ] (RU) `dotnet run` выводит добавленный курс с категорией.
- [ ] (EN) The `CourseShop.App` project builds without errors or nullable warnings.
- [ ] (EN) POCO entities with private setters and constructor-enforced invariants.
- [ ] (EN) Configurations in `IEntityTypeConfiguration<T>`, `OnModelCreating` delegates.
- [ ] (EN) DbContext accepts `DbContextOptions<T>`, registered via `AddDbContext` (Scoped).
- [ ] (EN) Connection string in `appsettings.json`, no password in code.
- [ ] (EN) `SaveChangesAsync` with `await`, `Include` in `ListCoursesAsync`.
- [ ] (EN) Comment about `EnsureCreated` → migrations in production.
- [ ] (EN) `dotnet run` prints the added course with its category.

#### Ресурсы / Resources

- Microsoft Learn — DbContext creation and configuration — https://learn.microsoft.com/ef/core/dbcontext-creation/
- EF Core — Code-first conventions — https://learn.microsoft.com/ef/core/modeling/
- EF Core — Fluent API configuration — https://learn.microsoft.com/ef/core/modeling/entity-properties
- EF Core — Relationships — https://learn.microsoft.com/ef/core/modeling/relationships
- EF Core — Indexes — https://learn.microsoft.com/ef/core/modeling/indexes
- Connection strings reference — https://learn.microsoft.com/dotnet/framework/data/adonet/connection-strings
- User Secrets in .NET — https://learn.microsoft.com/aspnet/core/security/app-secrets
- `IAsyncEnumerable` and EF Core — https://learn.microsoft.com/ef/core/querying/async

---
