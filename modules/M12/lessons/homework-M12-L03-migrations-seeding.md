---
[← К уроку M12-L03](lesson-M12-L03-migrations-seeding.md) | [⬆ К модулю M12](../README.md) | [Предыдущее ДЗ ←](homework-M12-L02-code-first-dbcontext.md) | [Следующее ДЗ →](homework-M12-L04-relationships.md)
---

### Домашнее задание M12-L03: Миграции, dotnet ef, seeding / Homework M12-L03: Migrations, dotnet ef, seeding

**Урок / Lesson:** M12-L03
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться вести эволюцию схемы БД через миграции EF Core 8: создавать, применять и откатывать миграции командами `dotnet ef`, генерировать SQL-скрипты для CI/CD, и грамотно наполнять базу эталонными данными двумя способами — через `HasData` и через шаг инициализации в коде, избегая типовых ошибок (`EnsureCreated`, ручное удаление файлов, автоинкрементные ключи в `HasData`). (EN) Learn to drive database schema evolution through EF Core 8 migrations: create, apply and roll back migrations with `dotnet ef`, generate SQL scripts for CI/CD, and seed reference data two ways — via `HasData` and via an in-code initialisation step — while avoiding the typical mistakes (`EnsureCreated`, deleting files by hand, auto-increment keys in `HasData`).

#### Связь с уроком / Connection to the lesson
(RU) Урок M12-L03 показывает декларативную модель миграций: вы правите `DbContext`, а EF Core сравнивает снимок `ModelSnapshot.cs` с предыдущим состоянием и генерирует `Up()`/`Down()`. В этом задании вы пройдёте весь цикл — от `dotnet tool install` до `migrations script` и отката — на реалистичной модели каталога курсов с ролями, статусами и справочником категорий. Особое внимание уделено `HasData` для эталонных справочников и раздельному шагу динамического seeding, ровно как в уроке.
(EN) Lesson M12-L03 presents the declarative migration model: you edit the `DbContext`, EF Core diffs the `ModelSnapshot.cs` against the previous state and emits `Up()`/`Down()`. This homework walks you through the full cycle — from `dotnet tool install` to `migrations script` and a rollback — on a realistic course-catalogue model with roles, statuses and a category reference table. Special attention goes to `HasData` for reference data and a separate dynamic seeding step, exactly as in the lesson.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы продолжаете строить бэкенд учебной платформы «CourseShop». На прошлом уроке (M12-L02) вы спроектировали code-first модель `DbContext` для каталога курсов. Теперь схема должна «ожить» в реальной базе данных и при этом быть переносимой между машинами разработчиков, CI-сервером и продакшеном. Именно эту задачу решают миграции EF Core: вместо ручного написания `CREATE TABLE` вы описываете изменения в C#-модели, а `dotnet ef` генерирует упорядоченную историю DDL-операций, которую можно применить, откатить и превратить в SQL-скрипт.

Дополнительно нужно наполнить новую базу эталонными данными: три роли пользователей (`Admin`, `Author`, `Student`), пять статусов курсов (`Draft`, `InReview`, `Published`, `Archived`, `Deleted`) и четыре категории курсов. Эти справочники — часть «контракта» приложения: их значения зашиты в бизнес-логику (например, проверка `Status == "Published"`), поэтому они обязаны существовать сразу после наката миграций. Параллельно есть динамические данные — стартовый администратор и пара демо-курсов, — которые удобнее добавлять отдельным шагом инициализации, а не через `HasData`, потому что они могут меняться и их объём со временем растёт.

Задание моделирует реальный цикл работы команды: вы сделаете первоначальную миграцию, затем добавите колонку (эволюция схемы), сгенерируете SQL-скрипт, откатите изменение, и правильно организуете seeding обоих типов. Это именно те операции, которые каждый бэкенд-разработчик выполняет в реальном проекте, и именно те ошибки (ручное удаление миграций, `EnsureCreated` вместо `Migrate`, автоинкремент в `HasData`), которые урок выделяет как критические.

#### Что нужно сделать (пошагово)
1. **Подготовьте окружение.** Установите глобальный инструмент EF Core, если он ещё не установлен: `dotnet tool install --global dotnet-ef`. Проверьте версию: `dotnet ef --version` — она должна быть совместима с `Microsoft.EntityFrameworkCore.*` 8.x. Если обновление нужно: `dotnet tool update --global dotnet-ef`. Создайте проект консольного приложения (или используйте существующий из M12-L02): `dotnet new console -n CourseShop.MigrationsLab` и добавьте пакеты `Microsoft.EntityFrameworkCore.SqlServer` (или `Sqlite` для локальной работы без сервера) и `Microsoft.EntityFrameworkCore.Design` — именно `Design` нужен генератору миграций.

2. **Опишите модель.** В файле `Models/Course.cs` создайте сущности `Role`, `CourseStatus`, `CourseCategory`, `Course` и `User` с навигационными свойствами. В `Course` предусмотрите поля `Id` (Guid), `Title` (string, max 200), `Description` (string, nullable, max 2000), `Price` (decimal, precision 10/2), `StatusId` (int, FK), `CategoryId` (int, FK), `CreatedAt` (DateTime, default `NOW` на сервере), и добавьте позже поле `Slug` — оно понадобится во второй миграции. Настройте уникальный индекс по `User.Email`.

3. **Настройте `OnModelCreating`.** Опишите FK-связи (`HasOne`/`WithMany`/`HasForeignKey`), поведение удаления `Restrict` для справочников (чтобы нельзя было удалить роль, на которую ссылаются пользователи), и обязательно вызовите `HasData` для `Role`, `CourseStatus` и `CourseCategory` с **фиксированными** `Id` (1, 2, 3 для ролей; 1–5 для статусов; 1–4 для категорий). Это ключевая тонкость урока: `HasData` требует постоянных ключей, иначе идемпотентность `INSERT` ломается.

4. **Создайте первую миграцию.** Из папки решения выполните `dotnet ef migrations add InitialCreate --project src/CourseShop.MigrationsLab --startup-project src/CourseShop.MigrationsLab`. Откройте сгенерированный файл `Migrations/<timestamp>_InitialCreate.cs` и убедитесь, что `Up()` содержит `CreateTable`, `CreateIndex` и `InsertData` (для `HasData`-справочников), а `Down()` симметрично удаляет таблицы. Проверьте файл `ModelSnapshot.cs` — он должен отражать текущую модель.

5. **Примените миграцию.** `dotnet ef database update`. Убедитесь через любой SQL-клиент (или `dotnet ef migrations list`), что в базе появилась таблица `__EFMigrationsHistory` с записью `InitialCreate`, а справочники уже содержат seeded-строки.

6. **Эволюция схемы.** Добавьте в `Course` свойство `string Slug` и создайте вторую миграцию: `dotnet ef migrations add AddCourseSlug`. Откройте её, проверьте, что в `Up()` есть `AddColumn`, а в `Down()` — `DropColumn`. Примените: `dotnet ef database update`.

7. **Сгенерируйте SQL-скрипт.** `dotnet ef migrations script --output migrations.sql` — полный скрипт от пустой базы. Затем скрипт дельты: `dotnet ef migrations script InitialCreate AddCourseSlug --output delta.sql`. Откройте оба файла и убедитесь, что в первом есть `INSERT` для справочников, а во втором — только `ALTER TABLE ... ADD Slug`.

8. **Откат.** Откатитесь к первой миграции: `dotnet ef database update InitialCreate`. Убедитесь, что колонка `Slug` исчезла, а данные справочников остались. Вернитесь обратно: `dotnet ef database update`.

9. **Динамический seeding.** В `Program.cs` реализуйте метод-расширение `ApplyMigrationsAndSeedAsync`, который вызывает `db.Database.MigrateAsync()`, а затем, при условии `!await db.Users.AnyAsync()`, добавляет стартового администратора (`Email = "admin@courseshop.local"`, `RoleId = 1`) и два демо-курса, привязанных к статусу `Published` и категории «Backend». Сохраните изменения через `SaveChangesAsync()`. Запустите приложение дважды — второй запуск не должен дублировать администратора.

10. **Проверьте идемпотентность.** Удалите локальную базу (или файл `.db`), запустите приложение «с нуля» и убедитесь, что миграции создают схему, `HasData` наполняет справочники, а динамический шаг добавляет администратора ровно один раз.

#### Требования к решению
- Используется C# 12 / .NET 8: top-level statements в `Program.cs`, коллекционные выражения там, где они уместны, `init`-сеттеры или `required` для обязательных свойств, file-scoped namespaces.
- В проекте установлены `Microsoft.EntityFrameworkCore.SqlServer` (или `Sqlite`) и `Microsoft.EntityFrameworkCore.Design`. Версии 8.x.
- В `OnModelCreating` настроены: уникальный индекс `User.Email`, FK от `Course` к `CourseStatus` и `CourseCategory` с `DeleteBehavior.Restrict`, FK от `User` к `Role` с `Restrict`, `HasData` для трёх справочников с фиксированными целочисленными ключами.
- Созданы ровно две миграции: `InitialCreate` и `AddCourseSlug`. Обе применены. `Up()` и `Down()` симметричны.
- В `Program.cs` есть `ApplyMigrationsAndSeedAsync`, использующий `MigrateAsync()`, а НЕ `EnsureCreated()`. Динамический seeding защищён проверкой `AnyAsync()`.
- Команда `dotnet ef migrations script` выполнена, файл `migrations.sql` присутствует в репозитории и содержит `INSERT` для справочников.
- Откат `database update InitialCreate` выполнен и задокументирован (скриншот или описание, что колонка `Slug` исчезла).
- Миграции находятся в Git, история коммитов чистая.

#### Тонкости и подводные камни
- **Никогда не удаляйте файл миграции вручную.** Урок явно предупреждает: удаление `<timestamp>_Init.cs` из папки `Migrations` ломает синхронизацию с `ModelSnapshot.cs`. Всегда используйте `dotnet ef migrations remove` — он удалит и класс, и обновит снимок. Если миграция уже применена к базе, сначала откатите её (`database update <Previous>`), и только потом `migrations remove`.
- **`HasData` требует фиксированных ключей.** Не используйте автоинкрементные `Id` в `HasData` — EF не сможет гарантировать идемпотентность `INSERT` и при повторном применении сгенерирует `DELETE`/`INSERT` или вообще упадёт на конфликте. В уроке справочник `Role` имеет явно заданные `Id = 1, 2, 3`.
- **`EnsureCreated` ≠ `Migrate`.** `EnsureCreated` создаёт схему по модели без записи в `__EFMigrationsHistory` и блокирует будущие миграции. Для миграционного подхода используйте только `db.Database.MigrateAsync()`.
- **Не редактируйте применённую миграцию.** Любое изменение модели — это новая миграция. Правка старого `Up()` не приведёт к выполнению на базах, где она уже применена, и создаст расхождение между кодом и `__EFMigrationsHistory`.
- **`Down()` должен быть симметричен `Up()`.** Если в `Up` добавили колонку, в `Down` её удаляем. Если в `Up` создали таблицу и наполнили её `InsertData`, в `Down` сначала удаляем данные (или просто таблицу — данные уйдут вместе с ней). Для необратимых операций (drop column без бэкапа) документируйте необратимость в комментарии.
- **Изменение `HasData` требует новой миграции.** Стоит поменять имя роли в `HasData` — EF Core на следующем `migrations add` сгенерирует `UpdateData`. Если вы измените `HasData`, но забудете создать миграцию, база и модель разойдутся.
- **Конфликты `ModelSnapshot` в команде.** Если два разработчика одновременно добавили миграции, после мерджа пересоздайте миграцию поверх общего снимка, иначе `migrations list` покажет «двойную» историю.
- **`migrations script` без аргументов** генерирует скрипт от пустой базы до последней миграции — это удобно для развёртывания «с нуля». Для инкрементального обновления прод всегда указывайте `--from <последняя применённая> --to <целевая>`.
- **`decimal` и точность.** Укажите `.HasPrecision(10, 2)` для `Price`, иначе SQL Server может выбрать `float`-подобный тип и вы получите ошибки округления. Это частая скрытая проблема, которую миграция фиксирует.

#### Критерии приёмки
- [ ] `dotnet ef --version` возвращает версию 8.x, инструмент установлен глобально.
- [ ] В проекте есть пакеты `Microsoft.EntityFrameworkCore.SqlServer`/`Sqlite` и `Microsoft.EntityFrameworkCore.Design` версии 8.x.
- [ ] Модель содержит сущности `Role`, `CourseStatus`, `CourseCategory`, `Course`, `User` с корректными навигациями.
- [ ] `OnModelCreating` настраивает уникальный индекс по `User.Email`.
- [ ] FK-связи используют `DeleteBehavior.Restrict` для справочников.
- [ ] `HasData` вызван для `Role`, `CourseStatus`, `CourseCategory` с фиксированными целочисленными `Id`.
- [ ] Создана миграция `InitialCreate`, её `Up()` содержит `CreateTable`, `CreateIndex` и `InsertData`.
- [ ] `Down()` для `InitialCreate` симметричен: удаляет таблицы в обратном порядке.
- [ ] Создана миграция `AddCourseSlug`, корректно добавляющая и удаляющая колонку `Slug`.
- [ ] `dotnet ef database update` применяет обе миграции без ошибок.
- [ ] `dotnet ef migrations script --output migrations.sql` создаёт файл с `INSERT` для справочников.
- [ ] Команда `dotnet ef database update InitialCreate` успешно откатывает вторую миграцию.
- [ ] В `Program.cs` используется `MigrateAsync()`, а не `EnsureCreated()`.
- [ ] Динамический seeding (администратор + демо-курсы) выполняется ровно один раз благодаря `AnyAsync()`.
- [ ] Все миграции закоммичены в Git.

#### Подсказки (без прямого ответа)
- Если `dotnet ef migrations add` падает с «No project found», явно укажите `--project` и `--startup-project`. `Design`-пакет должен стоять именно в стартап-проекте.
- Для SQLite укажите `UseSqlite("Data Source=courseshop.db")` и помните, что файл `.db` создаётся рядом с бинарником при запуске из `bin/Debug/net8.0/`.
- `HasData` принимает анонимные объекты или экземпляры сущности — главное, чтобы PK был задан явно.
- Чтобы откат не падал на FK, в `Down()` EF Core уже сгенерирует удаление в правильном порядке; но если вы правите `Down` вручную — удаляйте зависимые таблицы раньше главных.
- Для проверки идемпотентности seeding достаточно удалить файл базы и запустить приложение ещё раз.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — Эталонное решение ДЗ M12-L03
// Reference solution for homework M12-L03
using Microsoft.EntityFrameworkCore;

namespace CourseShop.MigrationsLab;

// Справочник ролей: фиксированные Id для HasData / Role reference: fixed Ids for HasData
public sealed class Role
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public ICollection<User> Users { get; set; } = new List<User>();
}

public sealed class CourseStatus
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public ICollection<Course> Courses { get; set; } = new List<Course>();
}

public sealed class CourseCategory
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public ICollection<Course> Courses { get; set; } = new List<Course>();
}

public sealed class User
{
    public Guid Id { get; set; }
    public string Email { get; set; } = string.Empty;
    public int RoleId { get; set; }
    public Role Role { get; set; } = null!;
}

public sealed class Course
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string Title { get; set; } = string.Empty;
    public string? Description { get; set; }
    public decimal Price { get; set; }
    public int StatusId { get; set; }
    public CourseStatus Status { get; set; } = null!;
    public int CategoryId { get; set; }
    public CourseCategory Category { get; set; } = null!;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public string? Slug { get; set; } // добавлен во второй миграции / added in the 2nd migration
}

public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Role> Roles => Set<Role>();
    public DbSet<CourseStatus> Statuses => Set<CourseStatus>();
    public DbSet<CourseCategory> Categories => Set<CourseCategory>();
    public DbSet<Course> Courses => Set<Course>();
    public DbSet<User> Users => Set<User>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // HasData: эталонные справочники с фиксированными ключами / Reference data with fixed keys
        modelBuilder.Entity<Role>().HasData(
            new Role { Id = 1, Name = "Admin" },
            new Role { Id = 2, Name = "Author" },
            new Role { Id = 3, Name = "Student" });

        modelBuilder.Entity<CourseStatus>().HasData(
            new CourseStatus { Id = 1, Name = "Draft" },
            new CourseStatus { Id = 2, Name = "InReview" },
            new CourseStatus { Id = 3, Name = "Published" },
            new CourseStatus { Id = 4, Name = "Archived" },
            new CourseStatus { Id = 5, Name = "Deleted" });

        modelBuilder.Entity<CourseCategory>().HasData(
            new CourseCategory { Id = 1, Name = "Backend" },
            new CourseCategory { Id = 2, Name = "Frontend" },
            new CourseCategory { Id = 3, Name = "DevOps" },
            new CourseCategory { Id = 4, Name = "Data" });

        modelBuilder.Entity<User>(b =>
        {
            b.HasIndex(u => u.Email).IsUnique();
            b.HasOne(u => u.Role)
             .WithMany(r => r.Users)
             .HasForeignKey(u => u.RoleId)
             .OnDelete(DeleteBehavior.Restrict);
        });

        modelBuilder.Entity<Course>(b =>
        {
            b.Property(c => c.Price).HasPrecision(10, 2); // важная тонкость урока / lesson pitfall
            b.HasOne(c => c.Status)
             .WithMany(s => s.Courses)
             .HasForeignKey(c => c.StatusId)
             .OnDelete(DeleteBehavior.Restrict);
            b.HasOne(c => c.Category)
             .WithMany(cat => cat.Courses)
             .HasForeignKey(c => c.CategoryId)
             .OnDelete(DeleteBehavior.Restrict);
            b.HasIndex(c => c.Slug).IsUnique(); // уникальный slug после 2-й миграции
        });
    }
}

// Program.cs — top-level statements, применение миграций + динамический seeding
// Apply migrations + dynamic seeding at startup
public static class MigrationSeedExtensions
{
    public static async Task ApplyMigrationsAndSeedAsync(this IServiceProvider services)
    {
        using var scope = services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

        // MigrateAsync — НЕ EnsureCreated / MigrateAsync, NOT EnsureCreated
        await db.Database.MigrateAsync();

        // Динамический seeding: проверка AnyAsync, чтобы не дублировать / Dynamic seeding with existence check
        if (!await db.Users.AnyAsync())
        {
            db.Users.Add(new User
            {
                Id = Guid.NewGuid(),
                Email = "admin@courseshop.local",
                RoleId = 1 // Admin
            });

            db.Courses.AddRange(
                new Course { Title = "C# 12 Basics", StatusId = 3, CategoryId = 1, Price = 49.99m, Slug = "csharp-12-basics" },
                new Course { Title = "ASP.NET Core 8", StatusId = 3, CategoryId = 1, Price = 59.99m, Slug = "aspnet-core-8" });

            await db.SaveChangesAsync();
        }
    }
}
```

Разбор по строкам. Класс `AppDbContext` использует первичный конструктор C# 12 (`DbContextOptions<AppDbContext> options)`) — это устраняет шаблонный конструктор и соответствует стилю урока. `DbSet` объявлен через выражение-свойство `=> Set<T>()`, что гарантирует возврат актуального `Set` даже если поле не создано. В `OnModelCreating` три вызова `HasData` определяют эталонные справочники с **фиксированными целочисленными ключами** — это ключевое требование урока, без которого идемпотентность `INSERT` нарушается: EF Core не сможет сопоставить строки при повторном применении. Поведение удаления для всех FK установлено в `Restrict`, потому что удаление роли или статуса, на которые ссылаются сущности, привело бы к «висячим» ссылкам. Для `Price` вызван `HasPrecision(10, 2)` — это та тонкая настройка, которую урок упоминает как часто упускаемую: без неё SQL Server может выбрать тип с плавающей точкой и дать ошибки округления. Уникальный индекс на `Email` повторяет пример из урока и препятствует дублированию пользователей. Поле `Slug` помечено как добавляемое во второй миграции (`AddCourseSlug`): первая миграция `InitialCreate` создаёт схему без него, вторая добавляет колонку, а `Down()` второй миграции её удаляет — симметрия, которой требует урок. В `MigrationSeedExtensions.ApplyMigrationsAndSeedAsync` вызывается `MigrateAsync()`, а не `EnsureCreated()`: это прямое следствие предупреждения урока, что `EnsureCreated` не пишет историю `__EFMigrationsHistory` и блокирует будущие миграции. Динамический seeding защищён `!await db.Users.AnyAsync()` — это гарантирует однократное добавление администратора и демо-курсов при повторных запусках, и это рекомендуемый уроком способ для данных, которые не являются частью «контракта» приложения. `SaveChangesAsync` фиксирует изменения. Коллекционные выражения в `AddRange` и использование `required`/`init`-стиля соответствуют C# 12. Команды `dotnet ef migrations add InitialCreate`, затем `dotnet ef migrations add AddCourseSlug`, `dotnet ef database update`, `dotnet ef migrations script --output migrations.sql` и `dotnet ef database update InitialCreate` для отката полностью покрывают цикл, описанный в уроке. Таким образом решение применяет все ключевые концепции: декларативную модель, симметрию `Up`/`Down`, два способа seeding и набор best-practices по работе с `dotnet ef` CLI.

#### Задания на углубление (бонус)
1. **Кастомная миграция с данными.** Добавьте третью миграцию `SeedDemoAuthors`, в теле `Up()` вручную (через `migrationBuilder.Sql(...)`) вставьте двух авторов и свяжите их с существующими демо-курсами через `UpdateData`. В `Down()` удалите только этих авторов по `Email`. Убедитесь, что откат работает идемпотентно.
2. **Идемпотентный SQL-скрипт для CI/CD.** Сгенерируйте скрипт с флагом `--idempotent`: `dotnet ef migrations script --idempotent --output idempotent.sql`. Объясните, чем он отличается от обычного и почему подходит для повторного применения на уже частично обновлённой базе.
3. **Откат с потерей данных.** Создайте миграцию `AddCourseAuditColumn` с колонкой `AuditNote string?`, примените её, заполните несколько строк, затем откатитесь и проанализируйте, что данные в `AuditNote` потеряны безвозвратно. Документируйте в `README`, почему `Down()` для drop column необратим, и какие меры (бэкап, soft-delete) применяют в проде.
4. **Командная работа.** Смоделируйте конфликт: в двух ветках создайте разные миграции от общего `InitialCreate`, слейте ветки, удалите «лишнюю» миграцию через `dotnet ef migrations remove` после отката, и пересоздайте общую. Опишите шаги в `TEAMWORK.md`.

---

## Statement in English / Постановка на английском

#### Context and motivation
You keep building the back end of the "CourseShop" learning platform. In the previous lesson (M12-L02) you designed a code-first `DbContext` model for a course catalogue. Now the schema must come alive inside a real database and, at the same time, stay portable between developer machines, the CI server and production. This is exactly the problem EF Core migrations solve: instead of writing `CREATE TABLE` by hand, you describe changes in the C# model and `dotnet ef` generates an ordered history of DDL operations that can be applied, rolled back and turned into a SQL script.

Additionally, you must seed the fresh database with reference data: three user roles (`Admin`, `Author`, `Student`), five course statuses (`Draft`, `InReview`, `Published`, `Archived`, `Deleted`) and four course categories. These dictionaries are part of the application "contract" — their values are hard-coded in business logic (for example, a check `Status == "Published"`), so they must exist immediately after the migrations are applied. In parallel there is dynamic data — the bootstrap administrator and a couple of demo courses — that is more convenient to add through a separate initialisation step rather than through `HasData`, because it can change over time and its volume grows.

This homework simulates the real team cycle: you will produce the initial migration, then evolve the schema with a new column, generate a SQL script, roll the change back, and organise seeding of both kinds correctly. These are precisely the operations every back-end developer performs in a real project, and precisely the mistakes (deleting migrations by hand, using `EnsureCreated` instead of `Migrate`, auto-increment keys in `HasData`) that the lesson flags as critical.

#### What to do step by step
1. **Prepare the environment.** Install the global EF Core tool if it is not installed yet: `dotnet tool install --global dotnet-ef`. Verify the version with `dotnet ef --version` — it must be compatible with `Microsoft.EntityFrameworkCore.*` 8.x. If an update is needed, run `dotnet tool update --global dotnet-ef`. Create a console project (or reuse the one from M12-L02): `dotnet new console -n CourseShop.MigrationsLab`, and add the packages `Microsoft.EntityFrameworkCore.SqlServer` (or `Sqlite` for local work without a server) and `Microsoft.EntityFrameworkCore.Design` — the `Design` package is what the migration generator needs.

2. **Describe the model.** In `Models/Course.cs` create the entities `Role`, `CourseStatus`, `CourseCategory`, `Course` and `User` with navigation properties. In `Course` provide the fields `Id` (Guid), `Title` (string, max 200), `Description` (string, nullable, max 2000), `Price` (decimal, precision 10/2), `StatusId` (int, FK), `CategoryId` (int, FK), `CreatedAt` (DateTime, server default `NOW`), and add the field `Slug` later — it will be needed for the second migration. Configure a unique index on `User.Email`.

3. **Configure `OnModelCreating`.** Describe the FK relationships (`HasOne`/`WithMany`/`HasForeignKey`), use `DeleteBehavior.Restrict` for reference tables (so that a role referenced by users cannot be deleted), and make sure to call `HasData` for `Role`, `CourseStatus` and `CourseCategory` with **fixed** `Id` values (1, 2, 3 for roles; 1–5 for statuses; 1–4 for categories). This is the key subtlety of the lesson: `HasData` requires constant keys, otherwise `INSERT` idempotency breaks.

4. **Create the first migration.** From the solution folder run `dotnet ef migrations add InitialCreate --project src/CourseShop.MigrationsLab --startup-project src/CourseShop.MigrationsLab`. Open the generated `Migrations/<timestamp>_InitialCreate.cs` and verify that `Up()` contains `CreateTable`, `CreateIndex` and `InsertData` (for the `HasData` dictionaries), and that `Down()` removes the tables symmetrically. Inspect `ModelSnapshot.cs` — it must reflect the current model.

5. **Apply the migration.** Run `dotnet ef database update`. Through any SQL client (or `dotnet ef migrations list`) verify that the database now has the `__EFMigrationsHistory` table with an `InitialCreate` row, and that the reference tables are already populated with the seeded rows.

6. **Schema evolution.** Add a `string Slug` property to `Course` and create the second migration: `dotnet ef migrations add AddCourseSlug`. Open it, verify that `Up()` has `AddColumn` and `Down()` has `DropColumn`. Apply it: `dotnet ef database update`.

7. **Generate the SQL script.** Run `dotnet ef migrations script --output migrations.sql` for the full script from an empty database. Then produce the delta: `dotnet ef migrations script InitialCreate AddCourseSlug --output delta.sql`. Open both files and confirm that the first contains `INSERT` statements for the dictionaries, while the second contains only `ALTER TABLE ... ADD Slug`.

8. **Rollback.** Roll back to the first migration: `dotnet ef database update InitialCreate`. Confirm that the `Slug` column is gone but the reference data is still there. Move forward again: `dotnet ef database update`.

9. **Dynamic seeding.** In `Program.cs` implement the extension method `ApplyMigrationsAndSeedAsync` that calls `db.Database.MigrateAsync()`, and then, under the condition `!await db.Users.AnyAsync()`, adds the bootstrap administrator (`Email = "admin@courseshop.local"`, `RoleId = 1`) and two demo courses tied to the `Published` status and the "Backend" category. Persist with `SaveChangesAsync()`. Run the application twice — the second run must not duplicate the administrator.

10. **Verify idempotency.** Delete the local database (or the `.db` file), run the application from scratch and verify that the migrations create the schema, `HasData` populates the dictionaries, and the dynamic step adds the administrator exactly once.

#### Requirements
- C# 12 / .NET 8 is used: top-level statements in `Program.cs`, collection expressions where appropriate, `init` setters or `required` for mandatory properties, file-scoped namespaces.
- The project references `Microsoft.EntityFrameworkCore.SqlServer` (or `Sqlite`) and `Microsoft.EntityFrameworkCore.Design`, version 8.x.
- `OnModelCreating` configures: a unique index on `User.Email`, FKs from `Course` to `CourseStatus` and `CourseCategory` with `DeleteBehavior.Restrict`, an FK from `User` to `Role` with `Restrict`, and `HasData` for the three reference tables with fixed integer keys.
- Exactly two migrations exist: `InitialCreate` and `AddCourseSlug`. Both are applied. `Up()` and `Down()` are symmetric.
- `Program.cs` contains `ApplyMigrationsAndSeedAsync` that uses `MigrateAsync()`, NOT `EnsureCreated()`. Dynamic seeding is guarded by an `AnyAsync()` check.
- The command `dotnet ef migrations script` has been executed, the file `migrations.sql` is present in the repository and contains `INSERT` statements for the dictionaries.
- The rollback `database update InitialCreate` has been performed and documented (screenshot or note that the `Slug` column disappeared).
- Migrations are committed to Git with a clean history.

#### Pitfalls
- **Never delete a migration file by hand.** The lesson warns explicitly: deleting `<timestamp>_Init.cs` from the `Migrations` folder breaks synchronisation with `ModelSnapshot.cs`. Always use `dotnet ef migrations remove`, which removes the class and updates the snapshot. If the migration has already been applied to the database, roll it back first (`database update <Previous>`) and only then run `migrations remove`.
- **`HasData` requires fixed keys.** Do not use auto-increment `Id` values in `HasData` — EF Core cannot guarantee `INSERT` idempotency and on reapplication will emit `DELETE`/`INSERT` or even fail on a conflict. In the lesson the `Role` reference table has explicitly set `Id = 1, 2, 3`.
- **`EnsureCreated` is not `Migrate`.** `EnsureCreated` creates the schema from the model without writing to `__EFMigrationsHistory` and blocks future migrations. For the migration workflow use only `db.Database.MigrateAsync()`.
- **Do not edit an applied migration.** Any model change is a new migration. Patching an old `Up()` has no effect on databases where it has already been applied and creates a drift between code and `__EFMigrationsHistory`.
- **`Down()` must be symmetric to `Up()`.** If `Up` adds a column, `Down` drops it. If `Up` creates a table and fills it with `InsertData`, `Down` either deletes the data first or simply drops the table (the data goes with it). For irreversible operations (drop column without a backup) document the irreversibility in a comment.
- **Changing `HasData` requires a new migration.** As soon as you change a role name in `HasData`, the next `migrations add` will produce an `UpdateData` call. If you change `HasData` but forget to add a migration, the database and the model drift apart.
- **`ModelSnapshot` conflicts in a team.** If two developers add migrations at the same time, after the merge recreate the migration on top of the shared snapshot, otherwise `migrations list` shows a "double" history.
- **`migrations script` with no arguments** generates a script from the empty database to the latest migration — handy for "from-scratch" deployments. For incremental production updates always pass `--from <last applied> --to <target>`.
- **`decimal` precision.** Specify `.HasPrecision(10, 2)` for `Price`, otherwise SQL Server may pick a floating type and you will get rounding errors. This is a common hidden issue that the migration fixes.

#### Acceptance criteria
- [ ] `dotnet ef --version` returns an 8.x version, the tool is installed globally.
- [ ] The project references `Microsoft.EntityFrameworkCore.SqlServer`/`Sqlite` and `Microsoft.EntityFrameworkCore.Design` version 8.x.
- [ ] The model contains the entities `Role`, `CourseStatus`, `CourseCategory`, `Course`, `User` with correct navigations.
- [ ] `OnModelCreating` configures a unique index on `User.Email`.
- [ ] FK relationships use `DeleteBehavior.Restrict` for reference tables.
- [ ] `HasData` is called for `Role`, `CourseStatus`, `CourseCategory` with fixed integer `Id` values.
- [ ] The migration `InitialCreate` exists; its `Up()` contains `CreateTable`, `CreateIndex` and `InsertData`.
- [ ] The `Down()` of `InitialCreate` is symmetric: it drops tables in reverse order.
- [ ] The migration `AddCourseSlug` exists and correctly adds and drops the `Slug` column.
- [ ] `dotnet ef database update` applies both migrations without errors.
- [ ] `dotnet ef migrations script --output migrations.sql` produces a file containing `INSERT` for the dictionaries.
- [ ] `dotnet ef database update InitialCreate` successfully rolls back the second migration.
- [ ] `Program.cs` uses `MigrateAsync()`, not `EnsureCreated()`.
- [ ] Dynamic seeding (administrator + demo courses) runs exactly once thanks to `AnyAsync()`.
- [ ] All migrations are committed to Git.

#### Hints
- If `dotnet ef migrations add` fails with "No project found", pass `--project` and `--startup-project` explicitly. The `Design` package must live in the startup project.
- For SQLite use `UseSqlite("Data Source=courseshop.db")` and remember that the `.db` file is created next to the binary when running from `bin/Debug/net8.0/`.
- `HasData` accepts anonymous objects or entity instances — the key requirement is that the PK is set explicitly.
- To keep the rollback from failing on FKs, EF Core already generates deletion in the correct order; if you edit `Down` by hand, drop dependent tables before principal tables.
- To check seeding idempotency it is enough to delete the database file and run the application again.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — Reference solution for homework M12-L03
using Microsoft.EntityFrameworkCore;

namespace CourseShop.MigrationsLab;

// Role reference: fixed Ids for HasData
public sealed class Role
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public ICollection<User> Users { get; set; } = new List<User>();
}

public sealed class CourseStatus
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public ICollection<Course> Courses { get; set; } = new List<Course>();
}

public sealed class CourseCategory
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public ICollection<Course> Courses { get; set; } = new List<Course>();
}

public sealed class User
{
    public Guid Id { get; set; }
    public string Email { get; set; } = string.Empty;
    public int RoleId { get; set; }
    public Role Role { get; set; } = null!;
}

public sealed class Course
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string Title { get; set; } = string.Empty;
    public string? Description { get; set; }
    public decimal Price { get; set; }
    public int StatusId { get; set; }
    public CourseStatus Status { get; set; } = null!;
    public int CategoryId { get; set; }
    public CourseCategory Category { get; set; } = null!;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public string? Slug { get; set; } // added in the 2nd migration
}

public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Role> Roles => Set<Role>();
    public DbSet<CourseStatus> Statuses => Set<CourseStatus>();
    public DbSet<CourseCategory> Categories => Set<CourseCategory>();
    public DbSet<Course> Courses => Set<Course>();
    public DbSet<User> Users => Set<User>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // HasData: reference data with fixed keys
        modelBuilder.Entity<Role>().HasData(
            new Role { Id = 1, Name = "Admin" },
            new Role { Id = 2, Name = "Author" },
            new Role { Id = 3, Name = "Student" });

        modelBuilder.Entity<CourseStatus>().HasData(
            new CourseStatus { Id = 1, Name = "Draft" },
            new CourseStatus { Id = 2, Name = "InReview" },
            new CourseStatus { Id = 3, Name = "Published" },
            new CourseStatus { Id = 4, Name = "Archived" },
            new CourseStatus { Id = 5, Name = "Deleted" });

        modelBuilder.Entity<CourseCategory>().HasData(
            new CourseCategory { Id = 1, Name = "Backend" },
            new CourseCategory { Id = 2, Name = "Frontend" },
            new CourseCategory { Id = 3, Name = "DevOps" },
            new CourseCategory { Id = 4, Name = "Data" });

        modelBuilder.Entity<User>(b =>
        {
            b.HasIndex(u => u.Email).IsUnique();
            b.HasOne(u => u.Role)
             .WithMany(r => r.Users)
             .HasForeignKey(u => u.RoleId)
             .OnDelete(DeleteBehavior.Restrict);
        });

        modelBuilder.Entity<Course>(b =>
        {
            b.Property(c => c.Price).HasPrecision(10, 2); // lesson pitfall
            b.HasOne(c => c.Status)
             .WithMany(s => s.Courses)
             .HasForeignKey(c => c.StatusId)
             .OnDelete(DeleteBehavior.Restrict);
            b.HasOne(c => c.Category)
             .WithMany(cat => cat.Courses)
             .HasForeignKey(c => c.CategoryId)
             .OnDelete(DeleteBehavior.Restrict);
            b.HasIndex(c => c.Slug).IsUnique();
        });
    }
}

// Program.cs — top-level statements: apply migrations + dynamic seeding
public static class MigrationSeedExtensions
{
    public static async Task ApplyMigrationsAndSeedAsync(this IServiceProvider services)
    {
        using var scope = services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

        // MigrateAsync, NOT EnsureCreated
        await db.Database.MigrateAsync();

        // Dynamic seeding with existence check
        if (!await db.Users.AnyAsync())
        {
            db.Users.Add(new User
            {
                Id = Guid.NewGuid(),
                Email = "admin@courseshop.local",
                RoleId = 1 // Admin
            });

            db.Courses.AddRange(
                new Course { Title = "C# 12 Basics", StatusId = 3, CategoryId = 1, Price = 49.99m, Slug = "csharp-12-basics" },
                new Course { Title = "ASP.NET Core 8", StatusId = 3, CategoryId = 1, Price = 59.99m, Slug = "aspnet-core-8" });

            await db.SaveChangesAsync();
        }
    }
}
```

Walk-through, line by line. The `AppDbContext` class uses the C# 12 primary constructor syntax (`DbContextOptions<AppDbContext> options)`), removing the boilerplate constructor and matching the lesson's style. Each `DbSet` is declared with an expression-bodied property `=> Set<T>()`, which always returns the live `Set` even when no backing field is allocated. In `OnModelCreating` the three `HasData` calls define the reference tables with **fixed integer keys** — this is the lesson's core requirement, without which `INSERT` idempotency breaks: EF Core cannot match rows on reapplication. The delete behaviour for every FK is `Restrict`, because deleting a role or a status referenced by entities would leave dangling references. `Price` is configured with `HasPrecision(10, 2)` — a subtle setting the lesson mentions as frequently forgotten: without it SQL Server may choose a floating type and produce rounding errors. The unique index on `Email` mirrors the lesson's example and prevents duplicate users. The `Slug` field is marked as something introduced by the second migration (`AddCourseSlug`): the first migration `InitialCreate` builds the schema without it, the second adds the column, and the `Down()` of the second migration drops it — the symmetry the lesson demands. In `MigrationSeedExtensions.ApplyMigrationsAndSeedAsync` we call `MigrateAsync()`, not `EnsureCreated()`: this directly follows the lesson's warning that `EnsureCreated` does not write the `__EFMigrationsHistory` chain and blocks future migrations. The dynamic seeding is guarded by `!await db.Users.AnyAsync()`, which guarantees the bootstrap administrator and the demo courses are added only once across repeated runs — this is the lesson-recommended approach for data that is not part of the application "contract". `SaveChangesAsync` persists the changes. The collection expressions in `AddRange` and the `required`/`init`-style property declarations fit the C# 12 idiom. The CLI commands `dotnet ef migrations add InitialCreate`, then `dotnet ef migrations add AddCourseSlug`, `dotnet ef database update`, `dotnet ef migrations script --output migrations.sql` and `dotnet ef database update InitialCreate` for the rollback together cover the whole cycle described in the lesson. The solution therefore exercises every key concept: the declarative model, `Up`/`Down` symmetry, both seeding strategies, and the set of best practices for the `dotnet ef` CLI.

#### Going deeper (bonus)
1. **Custom data migration.** Add a third migration, `SeedDemoAuthors`, and inside `Up()` insert two authors manually via `migrationBuilder.Sql(...)` and link them to the existing demo courses with `UpdateData`. In `Down()` remove only those authors by `Email`. Verify the rollback is idempotent.
2. **Idempotent SQL script for CI/CD.** Generate the script with the `--idempotent` flag: `dotnet ef migrations script --idempotent --output idempotent.sql`. Explain how it differs from a plain script and why it fits reapplication on a partially-updated database.
3. **Destructive rollback.** Create a migration `AddCourseAuditColumn` with a `AuditNote string?` column, apply it, fill several rows, then roll back and analyse that the `AuditNote` data is irrecoverable. Document in `README` why `Down()` for a drop column is irreversible and which safeguards (backup, soft delete) production uses.
4. **Team workflow.** Simulate a conflict: in two branches create different migrations from a shared `InitialCreate`, merge the branches, remove the "extra" migration with `dotnet ef migrations remove` after rolling back, then recreate a common one. Describe the steps in `TEAMWORK.md`.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Установлен `dotnet-ef` 8.x, `dotnet ef --version` работает.
- [ ] Проект содержит `DbContext`, сущности и `OnModelCreating` с `HasData` и `Restrict`.
- [ ] Созданы миграции `InitialCreate` и `AddCourseSlug`, обе применены.
- [ ] `Up()` и `Down()` симметричны, проверены глазами.
- [ ] `migrations.sql` сгенерирован и закоммичен.
- [ ] Откат `database update InitialCreate` выполнен и задокументирован.
- [ ] `Program.cs` использует `MigrateAsync()`, а не `EnsureCreated()`.
- [ ] Динамический seeding идемпотентен (проверен повторным запуском).
- [ ] Все миграции в Git.

- [ ] `dotnet-ef` 8.x is installed, `dotnet ef --version` works.
- [ ] The project contains `DbContext`, entities and `OnModelCreating` with `HasData` and `Restrict`.
- [ ] Migrations `InitialCreate` and `AddCourseSlug` are created and applied.
- [ ] `Up()` and `Down()` are symmetric and reviewed.
- [ ] `migrations.sql` is generated and committed.
- [ ] The rollback `database update InitialCreate` is performed and documented.
- [ ] `Program.cs` uses `MigrateAsync()`, not `EnsureCreated()`.
- [ ] Dynamic seeding is idempotent (verified by a repeated run).
- [ ] All migrations are in Git.

#### Ресурсы / Resources
- [Microsoft Learn — Migrations overview](https://learn.microsoft.com/ef/core/managing-schemas/migrations/)
- [Microsoft Learn — Data seeding](https://learn.microsoft.com/ef/core/modeling/data-seeding)
- [Microsoft Learn — dotnet ef CLI reference](https://learn.microsoft.com/ef/core/cli/dotnet)
- [Microsoft Learn — Managing migrations in team environments](https://learn.microsoft.com/ef/core/managing-schemas/migrations/teams)
- [Microsoft Learn — Applying migrations](https://learn.microsoft.com/ef/core/managing-schemas/migrations/applying)
- [Microsoft Learn — Migration scripts](https://learn.microsoft.com/ef/core/managing-schemas/migrations/managing?tabs=dotnet)

---
[← К уроку M12-L03](lesson-M12-L03-migrations-seeding.md) | [⬆ К модулю M12](../README.md) | [Предыдущее ДЗ ←](homework-M12-L02-code-first-dbcontext.md) | [Следующее ДЗ →](homework-M12-L04-relationships.md)
