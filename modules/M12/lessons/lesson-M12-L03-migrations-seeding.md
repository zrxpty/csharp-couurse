[⬆ К модулю M12](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M12-L03: Миграции, dotnet ef, seeding / Migrations, dotnet ef, seeding

**Модуль / Module:** M12
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Миграции Entity Framework Core — это механизм, который переводит вашу C#-модель данных в реальные команды DDL (Data Definition Language) для базы данных и хранит «историю эволюции» схемы. Представьте себе книгу поправок к закону: каждое изменение (добавление статьи, удаление параграфа) — это отдельный лист с номером и датой, а база применяет их строго по порядку. Без миграций вам пришлось бы вручную писать `CREATE TABLE` и `ALTER TABLE`, а потом как-то синхронизировать это между командой разработчиков, тестовым стендом и продакшеном. Миграции решают эту задачу декларативно: вы правите классы `DbContext` и сущностей, а EF Core сравнивает текущую «снимок-модель» с предыдущей и генерирует разницу.

Основной рабочий инструмент — команда `dotnet ef migrations add <Name>`. Она запускает сборку проекта, строит модель из `DbContext`, сравнивает её с снимком в файле `ModelSnapshot.cs` и создаёт три артефакта в папке `Migrations`: класс миграции с методами `Up()` и `Down()`, файл метаданных и обновлённый снимок. В методе `Up()` описаны операции, которые надо применить к базе (создание таблиц, добавление колонок, индексов), а в `Down()` — обратные операции для отката. Это не строковые SQL-команды, а вызовы объектов `CreateTableOperation`, `AddColumnOperation` и т.д. — провайдер базы данных (SQL Server, PostgreSQL, SQLite) сам превратит их в нужный диалект. Классы миграций — это обычный C#-код, их можно редактировать, но осторожно: ручные правки должны быть симметричны в `Up` и `Down`.

Применение миграций к базе выполняется командой `dotnet ef database update`. EF Core читает таблицу `__EFMigrationsHistory`, сравнивает список уже применённых миграций с доступными в сборке и выполняет недостающие `Up()` по порядку. Если нужно откатиться — `dotnet ef database update <PreviousMigrationName>` выполнит `Down()` для всех более поздних. Это и есть downgrade: важно, что не все операции обратимы (например, удаление колонки с данными без резервной копии необратимо), поэтому продумывайте `Down()` заранее. Для генерации чистого SQL-скрипта без подключения к базе используется `dotnet ef migrations script` — он выдаёт `.sql` файл, который передаётся DBA или выполняется в CI/CD пайплайне. Можно указать диапазон `--from <Migration> --to <Migration>`, чтобы получить дельту между двумя точками.

Seeding — начальное наполнение базы справочниками, ролями, администраторами. В EF Core есть два подхода: «сырой» через `context.AddRange()` в `Program.cs` с проверкой наличия, и декларативный через `HasData()` в `OnModelCreating`. Второй подход интегрируется в миграции: данные попадают в `Up()` как `INSERT`, а первичные ключи должны быть фиксированы (постоянные значения, не автоинкрементные). `HasData` особенно полезен для эталонных справочников, которые входят в «контракт» приложения: роли, статусы, типы документов. Для динамических или больших данных предпочтительнее отдельный шаг инициализации в коде, потому что `HasData` генерирует полные `DELETE`/`INSERT` при каждом изменении значения. Помните: миграции — это код, поэтому они должны храниться в системе контроля версий, проходить ревью и тестироваться на копии прод-базы перед релизом.

#### Theory (EN)

Entity Framework Core migrations are the mechanism that translates your C# data model into real DDL (Data Definition Language) commands for the database and keeps a "history of evolution" of the schema. Think of a book of legal amendments: every change (a new article, a deleted paragraph) is a separate page with a number and a date, and the database applies them strictly in order. Without migrations you would have to write `CREATE TABLE` and `ALTER TABLE` by hand and somehow synchronise that across the dev team, the staging server, and production. Migrations solve this declaratively: you edit your `DbContext` and entity classes, EF Core compares the current "model snapshot" with the previous one, and produces the diff.

The main workhorse is the command `dotnet ef migrations add <Name>`. It builds the project, constructs the model from `DbContext`, compares it with the snapshot stored in `ModelSnapshot.cs`, and creates three artefacts in the `Migrations` folder: the migration class with `Up()` and `Down()` methods, a metadata file, and an updated snapshot. The `Up()` method describes the operations to apply to the database (creating tables, adding columns, indexes), while `Down()` describes the reverse operations for rollback. These are not raw SQL strings but calls to `CreateTableOperation`, `AddColumnOperation` and similar objects — the database provider (SQL Server, PostgreSQL, SQLite) turns them into the right dialect. Migration classes are regular C# code, so you can edit them, but carefully: manual edits must stay symmetric between `Up` and `Down`.

Applying migrations to the database is done with `dotnet ef database update`. EF Core reads the `__EFMigrationsHistory` table, compares the list of already applied migrations with the ones available in the assembly, and runs the missing `Up()` methods in order. To roll back, `dotnet ef database update <PreviousMigrationName>` executes `Down()` for all later migrations. That is the downgrade flow, and it is important to remember that not every operation is reversible (dropping a column with data and no backup is irreversible), so design `Down()` ahead of time. To generate a pure SQL script without connecting to a database, use `dotnet ef migrations script` — it produces a `.sql` file you can hand to a DBA or run inside a CI/CD pipeline. You can pass a range `--from <Migration> --to <Migration>` to get the delta between two points.

Seeding is the initial population of the database with dictionaries, roles, administrators. EF Core offers two approaches: a "raw" one via `context.AddRange()` in `Program.cs` with an existence check, and a declarative one via `HasData()` inside `OnModelCreating`. The second approach integrates with migrations: the data becomes part of `Up()` as `INSERT` statements, and primary keys must be fixed (constant values, not auto-incremented). `HasData` is especially useful for reference data that is part of the application "contract": roles, statuses, document types. For dynamic or large datasets a separate initialisation step in code is preferable, because `HasData` emits full `DELETE`/`INSERT` sets whenever a value changes. Remember that migrations are code: they belong in version control, they need code review, and they should be tested against a copy of the production database before release.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — DbContext с HasData seeding и пример миграции
// DbContext with HasData seeding and a sample migration

using Microsoft.EntityFrameworkCore;

namespace CourseShop.Modules.M12;

// Сущность справочника ролей / Role reference entity
public sealed class Role
{
    public int Id { get; set; }              // фиксированный PK для HasData / fixed PK for HasData
    public string Name { get; set; } = string.Empty;
    public ICollection<User> Users { get; set; } = new List<User>();
}

public sealed class User
{
    public Guid Id { get; set; }
    public string Email { get; set; } = string.Empty;
    public int RoleId { get; set; }
    public Role Role { get; set; } = null!;
}

public class AppDbContext : DbContext
{
    public DbSet<Role> Roles => Set<Role>();
    public DbSet<User> Users => Set<User>();

    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // HasData: эталонные данные попадут в миграцию как INSERT
        // HasData: reference data becomes INSERTs inside the migration
        modelBuilder.Entity<Role>().HasData(
            new Role { Id = 1, Name = "Admin" },
            new Role { Id = 2, Name = "User" },
            new Role { Id = 3, Name = "Guest" });

        modelBuilder.Entity<User>(b =>
        {
            b.HasIndex(u => u.Email).IsUnique();
            b.HasOne(u => u.Role)
             .WithMany(r => r.Users)
             .HasForeignKey(u => u.RoleId)
             .OnDelete(DeleteBehavior.Restrict);
        });
    }
}

// ============================================================
// Пример сгенерированной миграции (упрощён, для понимания структуры)
// A generated migration (simplified, to show the structure)
// Файл: Migrations/20240115103000_Init.cs
// ============================================================
public partial class Init : Migration
{
    // Применение к базе / Apply to database
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "Roles",
            columns: table => new
            {
                Id = table.Column<int>(type: "int", nullable: false)
                    .Annotation("SqlServer:Identity", "1, 1"),
                Name = table.Column<string>(type: "nvarchar(64)", maxLength: 64, nullable: false)
            },
            constraints: table => table.PrimaryKey("PK_Roles", x => x.Id));

        migrationBuilder.CreateTable(
            name: "Users",
            columns: table => new
            {
                Id = table.Column<Guid>(type: "uniqueidentifier", nullable: false),
                Email = table.Column<string>(type: "nvarchar(256)", maxLength: 256, nullable: false),
                RoleId = table.Column<int>(type: "int", nullable: false)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_Users", x => x.Id);
                table.ForeignKey(
                    name: "FK_Users_Roles_RoleId",
                    column: x => x.RoleId,
                    principalTable: "Roles",
                    principalColumn: "Id",
                    onDelete: ReferentialAction.Restrict);
            });

        migrationBuilder.CreateIndex("IX_Users_Email", "Users", "Email", unique: true);

        // Seeding через HasData превращается в INSERT с фиксированными ключами
        // HasData seeding becomes INSERTs with fixed keys
        migrationBuilder.InsertData(
            table: "Roles",
            columns: new[] { "Id", "Name" },
            values: new object[,]
            {
                { 1, "Admin" },
                { 2, "User" },
                { 3, "Guest" }
            });
    }

    // Откат / Rollback (downgrade)
    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable("Users");
        migrationBuilder.DropTable("Roles");
    }
}

// ============================================================
// Program.cs: применение миграций при старте (типовой паттерн)
// Program.cs: apply migrations on startup (common pattern)
// ============================================================
public static class MigrationExtensions
{
    public static async Task ApplyMigrationsAsync(this WebApplication app)
    {
        using var scope = app.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

        // Применяем все неприменённые миграции / Apply pending migrations
        await db.Database.MigrateAsync();

        // Для динамического seeding (большие данные) — отдельный шаг, не HasData
        // For dynamic seeding (large data) — a separate step, not HasData
        if (!await db.Users.AnyAsync())
        {
            db.Users.Add(new User
            {
                Id = Guid.NewGuid(),
                Email = "root@example.com",
                RoleId = 1 // Admin
            });
            await db.SaveChangesAsync();
        }
    }
}
```

Команды CLI для работы с миграциями (выполнять из папки проекта):

```bash
# Установка инструментов EF Core (один раз) / Install EF Core tools (once)
dotnet tool install --global dotnet-ef

# Создать миграцию / Create a migration
dotnet ef migrations add Init --project src/CourseShop.Modules.M12 --startup-project src/CourseShop.Api

# Применить к базе / Apply to database
dotnet ef database update --startup-project src/CourseShop.Api

# Откатить на одну миграцию назад / Roll back one migration
dotnet ef database update PreviousMigrationName --startup-project src/CourseShop.Api

# Сгенерировать SQL-скрипт (без подключения к БД) / Generate SQL script (no DB connection)
dotnet ef migrations script --output migrations.sql --startup-project src/CourseShop.Api

# Скрипт между двумя миграциями / Script between two migrations
dotnet ef migrations script 20240101_Init 20240201_AddAudit --output delta.sql

# Удалить последнюю миграцию (если ещё не применена к БД) / Remove last migration (not applied yet)
dotnet ef migrations remove
```

#### Best Practices

- Храните миграции в системе контроля версий и не удаляйте применённые на проде миграции — это ломает историю `__EFMigrationsHistory`.
- Называйте миграции содержательно (`AddUserAvatarColumn`, а не `Fix1`), чтобы ревью и аудит были читаемыми.
- Дробите большие изменения: одна миграция = одна логическая правка схемы, так проще откатывать и тестировать.
- Используйте `HasData` только для эталонных справочников с фиксированными ключами; для динамических данных — отдельный шаг инициализации.
- Генерируйте SQL-скрипт (`migrations script`) и проверяйте его в PR-ревью перед применением на проде, особенно для `DROP` и `ALTER`.
- Делайте `Down()` симметричным `Up()` и тестируйте откат на копии базы.

- Keep migrations in version control and never delete migrations that have been applied in production — it breaks the `__EFMigrationsHistory` chain.
- Give migrations meaningful names (`AddUserAvatarColumn`, not `Fix1`) so review and audit stay readable.
- Split large changes: one migration = one logical schema change, so rollback and testing are easier.
- Use `HasData` only for reference data with fixed keys; for dynamic data use a separate initialisation step.
- Generate a SQL script (`migrations script`) and review it in the PR before applying on production, especially for `DROP` and `ALTER`.
- Keep `Down()` symmetric to `Up()` and test the rollback against a copy of the database.

#### Частые ошибки / Common Mistakes

- Удаление файла миграции вручную из папки `Migrations` → всегда используйте `dotnet ef migrations remove`, чтобы синхронно обновить `ModelSnapshot`.
- Два разработчика одновременно создают миграцию с конфликтующим снимком → сливайте ветки и пересоздавайте миграцию после мерджа, проверяйте `ModelSnapshot` в ревью.
- Изменение seeded-данных в `HasData` без создания новой миграции → всегда выполняйте `migrations add` после правки `HasData`, иначе база и модель разойдутся.
- Автоинкрементный ключ в `HasData` → используйте фиксированные `Id`, иначе EF не сможет гарантировать идемпотентность `INSERT`.
- Применение миграций через `EnsureCreated` вместо `Migrate` → `EnsureCreated` не пишет историю миграций и блокирует дальнейшие миграции; используйте `db.Database.MigrateAsync()`.
- Редактирование уже применённой миграции → создавайте новую миграцию для любых изменений; старую править нельзя.
- Забытый `Down()` для опасных операций (drop column) → продумывайте откат заранее или документируйте необратимость.

- Deleting a migration file by hand from the `Migrations` folder → always use `dotnet ef migrations remove` so `ModelSnapshot` stays in sync.
- Two developers creating migrations at the same time with conflicting snapshots → merge branches, recreate the migration after merge, and review `ModelSnapshot` in code review.
- Changing `HasData` seed data without creating a new migration → always run `migrations add` after editing `HasData`, otherwise the database and the model drift apart.
- Auto-increment key in `HasData` → use fixed `Id` values, otherwise EF cannot guarantee `INSERT` idempotency.
- Applying migrations with `EnsureCreated` instead of `Migrate` → `EnsureCreated` does not write migration history and blocks future migrations; use `db.Database.MigrateAsync()`.
- Editing an already-applied migration → create a new migration for any change; never patch an old one.
- A missing `Down()` for dangerous operations (drop column) → design the rollback ahead of time or document that it is irreversible.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я установил `dotnet-ef` и могу запустить `dotnet ef --version`.
- [ ] Я создал миграцию через `dotnet ef migrations add <Name>`, а не правлю схему руками.
- [ ] Я проверил, что `Up()` и `Down()` симметричны и корректны.
- [ ] Я применил миграции через `dotnet ef database update` или `MigrateAsync()`.
- [ ] Я знаю команду отката `database update <PreviousMigration>` и проверил её работу.
- [ ] Я использую `HasData` только для эталонных справочников с фиксированными ключами.
- [ ] Я умею генерировать SQL-скрипт через `migrations script` и проверять его перед продом.
- [ ] Я не использую `EnsureCreated` там, где нужны миграции.
- [ ] Миграции хранятся в Git и проходят код-ревью.

- [ ] I installed `dotnet-ef` and can run `dotnet ef --version`.
- [ ] I created a migration with `dotnet ef migrations add <Name>` rather than editing the schema by hand.
- [ ] I verified that `Up()` and `Down()` are symmetric and correct.
- [ ] I applied migrations via `dotnet ef database update` or `MigrateAsync()`.
- [ ] I know the rollback command `database update <PreviousMigration>` and tested it.
- [ ] I use `HasData` only for reference data with fixed keys.
- [ ] I can generate a SQL script via `migrations script` and review it before production.
- [ ] I do not use `EnsureCreated` where migrations are required.
- [ ] Migrations are committed to Git and go through code review.

#### Ресурсы / Resources

- [Microsoft Learn — Migrations overview](https://learn.microsoft.com/ef/core/managing-schemas/migrations/)
- [Microsoft Learn — Data seeding](https://learn.microsoft.com/ef/core/modeling/data-seeding)
- [Microsoft Learn — dotnet ef CLI reference](https://learn.microsoft.com/ef/core/cli/dotnet)
- [Microsoft Learn — Managing migrations in team environments](https://learn.microsoft.com/ef/core/managing-schemas/migrations/teams)

---

[⬆ К модулю M12](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
