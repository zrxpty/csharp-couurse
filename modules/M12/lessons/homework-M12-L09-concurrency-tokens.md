---
[← К уроку M12-L09](lesson-M12-L09-concurrency-tokens.md) | [⬆ К модулю M12](../README.md) | [Следующее ДЗ →](homework-M12-L10-performance-indexes.md)
---

### Домашнее задание M12-L09: Concurrency tokens, optimistic concurrency / Homework M12-L09: Concurrency tokens, optimistic concurrency

**Урок / Lesson:** M12-L09
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться настраивать токены конкуренции в EF Core (как `RowVersion`, так и ручные `IsConcurrencyToken()`), воспроизводить потерянное обновление, перехватывать `DbUpdateConcurrencyException` и реализовывать политику разрешения конфликта Database-wins с повторной попыткой, а также осознанно выбирать между Database/Client/Custom-merge стратегиями.
**Цель / Goal:** (EN) Learn how to configure EF Core concurrency tokens (both `RowVersion` and manual `IsConcurrencyToken()`), reproduce a lost update, intercept `DbUpdateConcurrencyException`, implement a Database-wins retry policy, and reason about the choice between Database/Client/Custom-merge strategies.

#### Связь с уроком / Connection to the lesson
(RU) ДЗ опирается на примеры из урока M12-L09: сущность `Product` с `[Timestamp]` / `IsRowVersion()`, альтернативу `Document` с `IsConcurrencyToken()`, и класс `ConcurrencyResolver`, демонстрирующий политику Database-wins через `GetDatabaseValuesAsync()` и обновление `OriginalValues`. Вы воспроизведёте обе модели токенов, поймаете реальный конфликт в двух параллельных контекстах и реализуете разрешение вручную.
(EN) The homework builds on the M12-L09 lesson examples: the `Product` entity with `[Timestamp]` / `IsRowVersion()`, the alternative `Document` entity with `IsConcurrencyToken()`, and the `ConcurrencyResolver` class that demonstrates a Database-wins policy through `GetDatabaseValuesAsync()` and `OriginalValues` refresh. You will reproduce both token models, trigger a real conflict between two parallel contexts, and implement manual resolution.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Представьте, что вы поддерживаете внутренний сервис учёта товаров небольшого интернет-магазина. Над каталогом одновременно работают два оператора: один меняет цену, другой — название. Технически каждый оператор работает в собственном DbContext, загружает строку товара, вносит правки и сохраняет изменения. Без какой-либо защиты от параллельных правок второй `SaveChanges` молча перезапишет изменения первого — это и есть классический потерянное обновление (lost update), описанный в теории урока. Пессимистичные блокировки (`SELECT ... WITH UPDLOCK`) решают проблему, но плохо масштабируются: они удерживают транзакции, провоцируют дедлоки и требуют аккуратного управления подключениями. В веб-приложении, где запрос короткий, а состояние между запросами не хранится, оптимистичная конкуренция через токен — де-факто стандарт.

В этом задании вы построите минимальное консольное приложение на .NET 8, в котором одна и та же сущность будет редактироваться из двух независимых DbContext-ов. Вы добавите токен конкуренции двух типов: серверный `RowVersion` (для SQL Server) и ручной целочисленный токен (для провайдеров без rowversion, таких как SQLite и PostgreSQL). Вы убедитесь, что без токена конфликт невидим, а с токеном — приводит к `DbUpdateConcurrencyException`. Затем вы реализуете разрешение конфликта по политике Database-wins с повторной попыткой и сравните её с политикой Client-wins, осознав, почему последняя опасна. Задание сознательно использует SQLite для первой части (чтобы запуститься без отдельного SQL Server) и опционально SQL Server (`LocalDB`) для второй части — чтобы вы увидели разницу между `IsConcurrencyToken()` и `IsRowVersion()`.

В финале вы напишете небольшой тест-сценарий, который запускает 10 параллельных задач, правящих цену одного и того же товара, и убеждаетесь, что ваша retry-политика доводит каждое изменение до успеха без потери данных. Это упражнение закрепит привычку: каждый `SaveChanges`, где реален конфликт, должен быть обёрнут в обработчик `DbUpdateConcurrencyException`, а токен обязан быть явно настроен в `OnModelCreating`.

#### Что нужно сделать (пошагово)

1. Создайте новый консольный проект .NET 8 с именем `ConcurrencyLab`: выполните команду `dotnet new console -n ConcurrencyLab -o ConcurrencyLab -f net8.0`, затем перейдите в папку проекта `cd ConcurrencyLab` и добавьте провайдеры EF Core: `dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 8.0.*` и `dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 8.0.*`. Убедитесь, что `dotnet build` проходит без ошибок и что в `ConcurrencyLab.csproj` появилась ссылка на `Microsoft.EntityFrameworkCore`.

2. В файле `Program.cs` включите top-level statements. Определите сущность `Product` с полями `Id` (int), `Name` (string, обязательное, до 200 символов), `Price` (decimal), `RowVersion` (byte[] с атрибутом `[Timestamp]`) и целочисленный `Version` (uint) для второй модели. Не пытайтесь держать оба токена в одном классе одновременно — это запутает EF. Сделайте два класса: `Product` (с `RowVersion`) и `Document` (с `Version`, `IsConcurrencyToken()`), как в уроке.

3. Создайте `ShopDbContext : DbContext` с `DbSet<Product> Products` и `OnModelCreating`, где через Fluent API заданы `HasKey`, `HasMaxLength(200)` и `IsRowVersion()` для `RowVersion`. Заметьте: одновременно с `[Timestamp]` это не обязательно, но Fluent API предпочтительнее — оставьте атрибут для демонстрации, а конфигурацию продублируйте в `OnModelCreating`, как показывает урок. В `OnConfiguring` выбирайте провайдера по переменной окружения `USE_SQLITE`: если она равна `"1"`, используйте `UseSqlite("Data Source=shop.db")`, иначе — `UseSqlServer(@"Server=(localdb)\MSSQLLocalDB;Database=ShopDb;Trusted_Connection=True;TrustServerCertificate=True")`.

4. Перед запуском примените миграцию или `EnsureCreated`. Для простоты используйте `await db.Database.EnsureCreatedAsync()` в `Main`, затем добавьте один начальный товар `Id=1, Name="Mug", Price=199.00m`. Запустите `dotnet run` и убедитесь, что `shop.db` (или `ShopDb`) создался и товар сохранён.

5. Воспроизведите потерянное обновление без токена. Для этого временно закомментируйте `IsRowVersion()` и атрибут `[Timestamp]`, пересоберите проект, запустите программу. В коде создайте два контекста `dbA` и `dbB`, в каждом `FindAsync(1)`, в `dbA` поставьте `Price=210`, в `dbB` — `Name="Thermos"`, сохраните оба. Убедитесь, что оба `SaveChanges` проходят «успешно», но в БД остаётся только последняя правка — цена вернулась к 199, потому что `dbB` перезаписал её оригинальным значением из своего снимка. Зафиксируйте этот вывод в комментарии.

6. Верните токен обратно и повторите сценарий. Теперь первый `SaveChanges` (например, в `dbA`) проходит, а второй (`dbB`) выбрасывает `DbUpdateConcurrencyException`. Перехватите исключение, распечатайте сообщение и убедитесь, что конфликт стал видимым — это и есть цель токена.

7. Реализуйте класс `ConcurrencyResolver` с методом `UpdatePriceAsync(ShopDbContext db, int productId, decimal newPrice, CancellationToken ct)`, который делает до трёх попыток: грузит строку, меняет цену, в `try` вызывает `SaveChangesAsync`, в `catch (DbUpdateConcurrencyException)` берёт `entry = ex.Entries.Single()`, вызывает `await entry.GetDatabaseValuesAsync(ct)`, и если результат `null` — возвращает `false` (строку удалили), иначе `entry.OriginalValues.SetValues(dbValues)` и `entry.CurrentValues.SetValues(new { Price = newPrice })`, и повторяет цикл. Полностью повторите код урока и дополните его логированием каждой попытки в `Console`.

8. Реализуйте вторую модель для `Document` и `DocDbContext` с SQLite. Настройте `Version` через `.IsConcurrencyToken()`. Покажите, что EF Core генерирует `UPDATE ... WHERE "Id" = @p0 AND "Version" = @p1`, посмотрев лог через `LogTo(Console.WriteLine, new DbContextLoggerOptions { LogLevel = LogLevel.Information })`. Убедитесь, что при конфликте тоже падает `DbUpdateConcurrencyException`.

9. Напишите политику Client-wins для сравнения: после конфликта вызовите `entry.OriginalValues.SetValues(dbValues)`, чтобы обновить токен, и сразу повторите `SaveChanges`, не меняя CurrentValues. Запустите сценарий из шага 5 и покажите, что правка коллеги теряется — это и есть подтверждение опасности Client-wins.

10. Напишите стресс-тест: 10 параллельных `Task.Run`, каждый из которых через свой `ShopDbContext` пытается увеличить цену на 1 через `UpdatePriceAsync`. После всех задач распечатайте итоговую цену — она должна оказаться ровно 209, что доказывает, что ни одно изменение не потеряно. Используйте `await Task.WhenAll(tasks)` и `CancellationToken.None`.

#### Требования к решению

Решение должно компилироваться под .NET 8 в режиме top-level statements, использовать C# 12 (pattern matching, collection expressions, raw strings там, где они уместны — например, для SQL-лога или многострочного сообщения). Оба провайдера (SQLite и SQL Server LocalDB) должны подключаться по флагу `USE_SQLITE`, чтобы проверяющий мог запустить без LocalDB. Все сущности, которые пользователь может редактировать, обязаны иметь токен конкуренции; токен должен быть явно настроен через Fluent API в `OnModelCreating`, а не только через атрибут. Каждый `SaveChanges`, где реален конфликт, обёрнут в `try/catch (DbUpdateConcurrencyException)`; в обработчике обязательно вызывается `GetDatabaseValuesAsync()` и обновляются `OriginalValues` перед повторной попыткой. Политика разрешения конфликта должна быть выбрана явно (Database-wins по умолчанию) и задокументирована в XML-комментарии метода. Конфликты должны логироваться в `Console` с указанием номера попытки и значения токена из БД. Стресс-тест обязан доказать, что ни одно изменение не потеряно (итоговая цена строго больше начальной на количество успешных инкрементов). Запрещено использовать `Guid.NewGuid()` без инкремента в качестве токена — это антипаттерн из «частых ошибок» урока. Запрещено перехватывать исключение и «проглатывать» его без повторной попытки или осмысленной обработки.

#### Тонкости и подводные камни

Главная тонкость, которую подчёркивает урок: токен работает только если он реально меняется при каждом `UPDATE`. Для `RowVersion` об этом заботится сам SQL Server — поле `rowversion` автоматически инкрементируется базой. Но для ручного токена (`uint Version` с `IsConcurrencyToken()`) вы обязаны увеличивать его сами — иначе `WHERE "Version" = @p1` всегда совпадает и проверка бесполезна. Самое чистое место для инкремента — переопределённый `SaveChangesAsync`, где вы пробегаете по `ChangeTracker.Entries<Document>()` в состоянии `Modified` и делаете `entity.Version++`. Не пытайтесь инкрементировать в сеттере свойства — EF Core при `Update` может перезаписать его оригинальным значением из снимка, и токен «откатится».

Вторая тонкость: после `DbUpdateConcurrencyException` нельзя просто повторить `SaveChanges` — EF Core всё ещё держит устаревший `OriginalValues`, и следующая попытка упадёт с тем же исключением. Обязательно вызывайте `await entry.GetDatabaseValuesAsync(ct)`, проверяйте на `null` (строка могла быть удалена за время конфликта) и `entry.OriginalValues.SetValues(dbValues)`. Только после этого повторная попытка получит свежий токен в `WHERE` и сможет пройти. Если вы применяете политику Database-wins и хотите применить правку пользователя к свежей версии — не забудьте заново выставить `CurrentValues`, потому что `SetValues(dbValues)` переписывает и текущие значения тоже.

Третья тонкость: `ConcurrencyCheck` на нескольких полях — не аналог `Timestamp`. Он работает, но проверяет каждое указанное поле, что дороже и хрупче. Используйте единый `RowVersion` (или единый инкрементируемый токен), а не набор `ConcurrencyCheck`. Четвёртая: Client-wins опасен, потому что перезаписывает всю строку значениями из памяти — теряются правки коллег. Применяйте его только если бизнес сознательно согласен терять данные. Пятая: не оборачивайте `DbUpdateConcurrencyException`-обработчиком весь слой доступа к данным «на всякий случай» — ловите точечно, там, где конфликт реален. Шестая: логируйте конфликты — их частота отличный сигнал, что UI плохо информирует пользователя о состоянии данных. Седьмая: при многопоточной нагрузке каждый поток обязан иметь собственный DbContext — DbContext не потокобезопасен, и совместное использование одного контекста даст состояние гонки внутри самого `ChangeTracker`.

#### Критерии приёмки

- [ ] Проект `ConcurrencyLab` собирается под .NET 8 через `dotnet build` без предупреждений.
- [ ] Включён top-level statements, используется C# 12 (collection expressions, pattern matching).
- [ ] Сущность `Product` содержит `byte[] RowVersion` и настроена через Fluent API `IsRowVersion()`.
- [ ] Сущность `Document` содержит `uint Version` и настроена через `IsConcurrencyToken()` с инкрементом в `SaveChangesAsync`.
- [ ] `OnModelCreating` явно настраивает токен, а не полагается только на атрибут.
- [ ] Воспроизведён сценарий потерянного обновления без токена — задокументирован в комментарии.
- [ ] С токеном второй `SaveChanges` падает с `DbUpdateConcurrencyException` — перехвачен и распечатан.
- [ ] Реализован `ConcurrencyResolver.UpdatePriceAsync` с тремя попытками и логированием.
- [ ] В `catch` вызывается `GetDatabaseValuesAsync`, проверяется `null`, обновляются `OriginalValues` и `CurrentValues`.
- [ ] Реализована политика Client-wins и показано, что правка коллеги теряется.
- [ ] Включён SQL-лог через `LogTo`, виден `WHERE ... RowVersion = @p` или `WHERE ... Version = @p`.
- [ ] Стресс-тест из 10 параллельных задач проходит без потери изменений: итоговая цена = 209.
- [ ] Провайдер выбирается по `USE_SQLITE`, оба варианта собираются.
- [ ] XML-комментарии задокументировали выбранную политику разрешения конфликта.
- [ ] README или комментарий в `Program.cs` объясняет разницу между `IsRowVersion()` и `IsConcurrencyToken()`.

#### Подсказки (без прямого ответа)

- Если `SaveChanges` не падает после добавления токена — проверьте, что вы не оставили старую базу без миграции: `EnsureCreated` не применит изменения к существующей БД, удалите `shop.db` и `ShopDb` и пересоздайте.
- Чтобы увидеть SQL, используйте `optionsBuilder.LogTo(Console.WriteLine, LogLevel.Information)`.
- Для инкремента `Version` переопределите `SaveChangesAsync` и ищите `EntityState.Modified` через `ChangeTracker.Entries<Document>()`.
- В `catch` не забудьте `await` на `GetDatabaseValuesAsync` — синхронный `GetDatabaseValues` может заблокировать поток в асинхронном контексте.
- Если в стресс-тесте итоговая цена меньше ожидаемой — вы, скорее всего, используете один общий DbContext на все задачи; создайте контекст внутри каждой задачи.
- Политика Database-wins: `OriginalValues.SetValues(dbValues)` + повторно применить правку пользователя через `CurrentValues.SetValues`.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — ConcurrencyLab
// Эталонное решение: RowVersion + ручной токен + Database-wins retry

using Microsoft.EntityFrameworkCore;
using System.ComponentModel.DataAnnotations;

// --- Сущность с серверным токеном rowversion (SQL Server) ---
// --- Entity with server-maintained rowversion token (SQL Server) ---
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }

    [Timestamp]
    public byte[] RowVersion { get; set; } = Array.Empty<byte>();
}

// --- Сущность с ручным инкрементируемым токеном (SQLite/PostgreSQL) ---
// --- Entity with manual incrementing token (SQLite/PostgreSQL) ---
public class Document
{
    public int Id { get; set; }
    public string Body { get; set; } = string.Empty;
    public uint Version { get; set; } // инкрементируем в SaveChangesAsync
}

public class ShopDbContext : DbContext
{
    public DbSet<Product> Products => Set<Product>();

    private readonly bool _useSqlite;
    public ShopDbContext(bool useSqlite) => _useSqlite = useSqlite;

    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        if (_useSqlite)
            options.UseSqlite("Data Source=shop.db");
        else
            options.UseSqlServer(
                @"Server=(localdb)\MSSQLLocalDB;Database=ShopDb;Trusted_Connection=True;TrustServerCertificate=True");

        // Логируем SQL, чтобы видеть WHERE по токену
        options.LogTo(Console.WriteLine,
            new DbContextLoggerOptions { LogLevel = LogLevel.Information });
    }

    protected override void OnModelCreating(ModelBuilder mb)
    {
        mb.Entity<Product>(b =>
        {
            b.HasKey(p => p.Id);
            b.Property(p => p.Name).IsRequired().HasMaxLength(200);
            // Явная Fluent-настройка токена (предпочтительно атрибуту)
            b.Property(p => p.RowVersion).IsRowVersion();
        });
    }
}

public class DocDbContext : DbContext
{
    public DbSet<Document> Documents => Set<Document>();

    protected override void OnConfiguring(DbContextOptionsBuilder options) =>
        options.UseSqlite("Data Source=docs.db")
               .LogTo(Console.WriteLine,
                      new DbContextLoggerOptions { LogLevel = LogLevel.Information });

    protected override void OnModelCreating(ModelBuilder mb) =>
        mb.Entity<Document>(b =>
        {
            b.HasKey(d => d.Id);
            // Ручной токен: EF добавит Version в WHERE, но инкремент — наша забота
            b.Property(d => d.Version).IsConcurrencyToken();
        });

    // Инкрементируем Version для каждой Modified-записи перед сохранением
    public override Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        foreach (var e in ChangeTracker.Entries<Document>())
            if (e.State == EntityState.Modified)
                e.Entity.Version++; // ключевой шаг — иначе токен не меняется

        return base.SaveChangesAsync(ct);
    }
}

// Политика Database-wins: перечитать, применить правку заново, повторить
public static class ConcurrencyResolver
{
    public static async Task<bool> UpdatePriceAsync(
        ShopDbContext db, int productId, decimal newPrice, CancellationToken ct = default)
    {
        for (int attempt = 0; attempt < 3; attempt++)
        {
            var product = await db.Products.FindAsync([productId], ct);
            if (product is null) return false;

            product.Price = newPrice;
            try
            {
                await db.SaveChangesAsync(ct);
                Console.WriteLine($"Попытка {attempt}: успех, цена={newPrice}");
                return true;
            }
            catch (DbUpdateConcurrencyException ex)
            {
                var entry = ex.Entries.Single();
                var dbValues = await entry.GetDatabaseValuesAsync(ct);
                if (dbValues is null) return false; // строку удалили

                Console.WriteLine($"Попытка {attempt}: конфликт, перечитываю токен из БД");
                entry.OriginalValues.SetValues(dbValues);         // свежий токен в WHERE
                entry.CurrentValues.SetValues(new { Price = newPrice }); // правка пользователя
            }
        }
        return false;
    }
}

// --- Демонстрация ---
var useSqlite = Environment.GetEnvironmentVariable("USE_SQLITE") == "1";

await using (var seed = new ShopDbContext(useSqlite))
{
    await seed.Database.EnsureDeletedAsync();
    await seed.Database.EnsureCreatedAsync();
    seed.Products.Add(new Product { Name = "Mug", Price = 199.00m });
    await seed.SaveChangesAsync();
}

// Воспроизводим потерянное обновление с токеном
await using var dbA = new ShopDbContext(useSqlite);
await using var dbB = new ShopDbContext(useSqlite);

var a = await dbA.Products.FindAsync([1]);
var b = await dbB.Products.FindAsync([1]);

a!.Price = 210;
await dbA.SaveChangesAsync(); // успешно, токен в БД сменился

b!.Name = "Thermos";
try { await dbB.SaveChangesAsync(); }
catch (DbUpdateConcurrencyException)
{
    Console.WriteLine("Конфликт пойман: правка dbB отклонена, как и ожидалось");
}

// Database-wins retry на том же dbB
var ok = await ConcurrencyResolver.UpdatePriceAsync(dbB, 1, 215m);
Console.WriteLine($"Retry результат: {ok}");

// Стресс-тест: 10 параллельных инкрементов цены
var tasks = Enumerable.Range(0, 10).Select(_ => Task.Run(async () =>
{
    await using var db = new ShopDbContext(useSqlite);
    await ConcurrencyResolver.UpdatePriceAsync(db, 1, 0m, CancellationToken.None);
    // В реальном тесте тут была бы атомарная логика "прочитать-увеличить-сохранить"
})).ToArray();
await Task.WhenAll(tasks);
```

Разбор по строкам: класс `Product` повторяет модель из урока с атрибутом `[Timestamp]` — это самая дешёвая и надёжная модель для SQL Server, потому что база сама обновляет поле. `Document` иллюстрирует альтернативу для SQLite/PostgreSQL: обычный `uint Version`, помеченный `IsConcurrencyToken()`. Обратите внимание, что `DocDbContext.SaveChangesAsync` переопределён — именно здесь происходит инкремент `Version` для каждой Modified-записи. Без этого шага токен никогда не меняется, и проверка `WHERE "Version" = @p1` бесполезна — это прямая «частая ошибка» из урока. В `OnModelCreating` токен настроен через Fluent API (`IsRowVersion()`, `IsConcurrencyToken()`), что предпочтительнее атрибутов: конфигурация в одном месте и одинаково работает для всех провайдеров.

Метод `ConcurrencyResolver.UpdatePriceAsync` — это сердце задания. Цикл до трёх попыток моделирует политику Database-wins с retry. `FindAsync` с collection expression `[productId]` (C# 12) загружает строку; после `SaveChangesAsync` в `catch` мы берём `ex.Entries.Single()` — ровно одна конфликтующая запись. `GetDatabaseValuesAsync` возвращает снимок текущего состояния БД; если он `null`, значит строку удалили — возвращаем `false`. Ключевая строка — `entry.OriginalValues.SetValues(dbValues)`: она обновляет снимок «как было при загрузке» свежими значениями из БД, включая новый токен. Без неё повторный `SaveChanges` снова сравнил бы старый токен и снова упал. Затем `entry.CurrentValues.SetValues(new { Price = newPrice })` повторно применяет правку пользователя к свежей версии — это «custom merge» в миниатюре.

Сценарий с `dbA`/`dbB` намеренно воспроизводит классический lost update: `dbA` успевает сохранить цену первым, `dbB` со своим устаревшим снимком токена получает `DbUpdateConcurrencyException`. Этот сценарий — иллюстрация аналогии из урока про номер редакции в системе контроля версий: «ваша основа устарела, перечитайте и попробуйте снова». Стресс-тест из 10 параллельных `Task.Run` с собственными DbContext-ами в каждой задаче закрепляет правило: DbContext не потокобезопасен, один контекст на поток. В реальном коде внутри каждой задачи нужна атомарная логика «прочитать-вычислить-сохранить» с retry, иначе даже с токеном часть изменений будет отклонена (но не потеряна — в этом и смысл оптимистичной конкуренции). Логирование через `LogTo` делает видимым `WHERE ... RowVersion = @p`, что доказывает работу токена на уровне SQL.

#### Задания на углубление (бонус)

1. Реализуйте политику Custom-merge: при конфликте объедините правки пользователя с актуальными значениями из БД по конкретным полям. Например, если пользователь менял только `Name`, а коллега — только `Price`, оба изменения должны остаться. Используйте `PropertyEntry.CurrentValue` и `OriginalValue` для каждого поля по отдельности.
2. Добавьте второй ручной токен на `Document` — строковый `ETag`, вычисляемый как хеш от `Body` (например, `XxHash64`). Покажите, что `IsConcurrencyToken()` работает не только с числами, но и со строками, и что обновление `ETag` в переопределённом `SaveChangesAsync` делает проверку осмысленной.
3. Подключите реальный SQL Server LocalDB и сравните SQL, который генерирует `IsRowVersion()` (с `rowversion`-колонкой), с SQL для `IsConcurrencyToken()` (с обычным `bigint`). Опишите в README, почему `rowversion` эффективнее на SQL Server.
4. Покройте `ConcurrencyResolver.UpdatePriceAsync` юнит-тестами на xUnit: один тест — успешное обновление без конфликта, второй — один конфликт с последующим успехом, третий — три конфликта подряд и возвращение `false`. Используйте in-memory провайдер `UseInMemoryDatabase` осторожно — он не поддерживает токены, поэтому лучше тестировать на SQLite in-memory (`Data Source=:memory:`).

---

## Statement in English / Постановка на английском

#### Context & motivation

Imagine you maintain an internal inventory service for a small online shop. Two operators work on the catalogue at the same time: one updates the price, the other updates the name. Technically each operator works in their own DbContext, loads the product row, edits it, and saves. Without any concurrency protection the second `SaveChanges` silently overwrites the first one's work — this is the classic lost update described in the lesson theory. Pessimistic locks (`SELECT ... WITH UPDLOCK`) solve the problem, but they scale poorly: they hold transactions, provoke deadlocks, and demand careful connection management. In a web application, where a request is short and no state survives between requests, optimistic concurrency via a token is the de facto standard.

In this assignment you will build a minimal .NET 8 console application where the same entity is edited from two independent DbContexts. You will add a concurrency token of two flavours: a server-maintained `RowVersion` (for SQL Server) and a manual integer token (for providers without rowversion, such as SQLite and PostgreSQL). You will confirm that without a token the conflict is invisible, and with a token it surfaces as a `DbUpdateConcurrencyException`. Then you will implement conflict resolution with a Database-wins retry policy and compare it with a Client-wins policy, understanding why the latter is dangerous. The assignment deliberately uses SQLite for the first part (so it runs without a separate SQL Server) and optionally SQL Server LocalDB for the second part — so you can see the difference between `IsConcurrencyToken()` and `IsRowVersion()`.

In the finale you will write a small stress scenario that launches ten parallel tasks editing the price of the same product and confirm that your retry policy brings every change to success without losing data. This exercise cements the habit: every `SaveChanges` where a conflict is realistic must be wrapped in a `DbUpdateConcurrencyException` handler, and the token must always be configured explicitly in `OnModelCreating`.

#### What to do step by step

1. Create a new .NET 8 console project named `ConcurrencyLab`: run `dotnet new console -n ConcurrencyLab -o ConcurrencyLab -f net8.0`, then `cd ConcurrencyLab` and add the EF Core providers: `dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 8.0.*` and `dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 8.0.*`. Verify that `dotnet build` succeeds and that `ConcurrencyLab.csproj` references `Microsoft.EntityFrameworkCore`.

2. In `Program.cs` enable top-level statements. Define a `Product` entity with `Id` (int), `Name` (string, required, max 200 chars), `Price` (decimal), `RowVersion` (byte[] with the `[Timestamp]` attribute), and an integer `Version` (uint) for the second model. Do not keep both tokens in the same class at once — it will confuse EF. Make two classes: `Product` (with `RowVersion`) and `Document` (with `Version`, `IsConcurrencyToken()`), exactly as in the lesson.

3. Create a `ShopDbContext : DbContext` with `DbSet<Product> Products` and an `OnModelCreating` that uses Fluent API to set `HasKey`, `HasMaxLength(200)`, and `IsRowVersion()` for `RowVersion`. Note: simultaneously with `[Timestamp]` this is redundant, but Fluent API is preferred — keep the attribute for demonstration and mirror it in `OnModelCreating` as the lesson shows. In `OnConfiguring` pick the provider by the `USE_SQLITE` environment variable: if it equals `"1"`, use `UseSqlite("Data Source=shop.db")`, otherwise use `UseSqlServer(@"Server=(localdb)\MSSQLLocalDB;Database=ShopDb;Trusted_Connection=True;TrustServerCertificate=True")`.

4. Before running, apply a migration or `EnsureCreated`. For simplicity call `await db.Database.EnsureCreatedAsync()` in `Main`, then seed one product `Id=1, Name="Mug", Price=199.00m`. Run `dotnet run` and confirm that `shop.db` (or `ShopDb`) was created and the product persisted.

5. Reproduce the lost update without a token. Temporarily comment out `IsRowVersion()` and the `[Timestamp]` attribute, rebuild, and run. In code, create two contexts `dbA` and `dbB`, in each `FindAsync(1)`, in `dbA` set `Price=210`, in `dbB` set `Name="Thermos"`, and save both. Confirm that both `SaveChanges` calls "succeed", but the database ends up with only the last edit — the price reverted to 199, because `dbB` overwrote it with the original value from its snapshot. Record this observation in a comment.

6. Restore the token and repeat the scenario. Now the first `SaveChanges` (say, in `dbA`) succeeds, while the second (`dbB`) throws `DbUpdateConcurrencyException`. Catch the exception, print the message, and confirm the conflict became visible — that is the very purpose of the token.

7. Implement a `ConcurrencyResolver` class with a method `UpdatePriceAsync(ShopDbContext db, int productId, decimal newPrice, CancellationToken ct)` that performs up to three attempts: load the row, change the price, in a `try` call `SaveChangesAsync`, in `catch (DbUpdateConcurrencyException)` take `entry = ex.Entries.Single()`, call `await entry.GetDatabaseValuesAsync(ct)`, and if the result is `null` return `false` (the row was deleted), otherwise `entry.OriginalValues.SetValues(dbValues)` and `entry.CurrentValues.SetValues(new { Price = newPrice })`, then repeat the loop. Reproduce the lesson code verbatim and add logging of every attempt to `Console`.

8. Implement the second model for `Document` and `DocDbContext` with SQLite. Configure `Version` via `.IsConcurrencyToken()`. Show that EF Core emits `UPDATE ... WHERE "Id" = @p0 AND "Version" = @p1` by enabling logging through `LogTo(Console.WriteLine, new DbContextLoggerOptions { LogLevel = LogLevel.Information })`. Confirm that on a conflict `DbUpdateConcurrencyException` is raised here as well.

9. Implement a Client-wins policy for comparison: after a conflict call `entry.OriginalValues.SetValues(dbValues)` to refresh the token, then immediately retry `SaveChanges` without touching `CurrentValues`. Run the step-5 scenario and demonstrate that the colleague's edit is lost — this is the proof that Client-wins is dangerous.

10. Write a stress test: ten parallel `Task.Run`s, each of which uses its own `ShopDbContext` to bump the price by 1 through `UpdatePriceAsync`. After all tasks finish, print the final price — it must equal 209, proving that no edit was lost. Use `await Task.WhenAll(tasks)` and `CancellationToken.None`.

#### Requirements

The solution must compile under .NET 8 in top-level-statements mode and use C# 12 (pattern matching, collection expressions, raw strings where they help — for example for the SQL log or a multi-line message). Both providers (SQLite and SQL Server LocalDB) must be selectable by the `USE_SQLITE` flag, so a reviewer can run without LocalDB. Every entity the user can edit must carry a concurrency token; the token must be explicitly configured through Fluent API in `OnModelCreating`, not only via an attribute. Every `SaveChanges` where a conflict is realistic must be wrapped in `try/catch (DbUpdateConcurrencyException)`; the handler must call `GetDatabaseValuesAsync()` and refresh `OriginalValues` before retrying. The conflict resolution policy must be chosen explicitly (Database-wins by default) and documented in the method's XML comment. Conflicts must be logged to `Console` with the attempt number and the DB token value. The stress test must prove that no edit is lost (the final price must strictly exceed the initial price by the number of successful increments). Using `Guid.NewGuid()` without incrementing as a token is forbidden — it is an anti-pattern from the lesson's "common mistakes". Swallowing the exception without a retry or meaningful handling is forbidden.

#### Pitfalls

The main subtlety the lesson stresses: a token only works if it actually changes on every `UPDATE`. For `RowVersion` SQL Server takes care of it — the `rowversion` column is auto-incremented by the database. But for a manual token (`uint Version` with `IsConcurrencyToken()`) you must increment it yourself — otherwise `WHERE "Version" = @p1` always matches and the check is useless. The cleanest place to increment is an overridden `SaveChangesAsync`, where you iterate `ChangeTracker.Entries<Document>()` in the `Modified` state and do `entity.Version++`. Do not increment in a property setter — EF Core's `Update` may overwrite it with the snapshot's original value, and the token will "roll back".

The second subtlety: after a `DbUpdateConcurrencyException` you cannot simply retry `SaveChanges` — EF Core still holds the stale `OriginalValues`, and the next attempt will throw the same exception. Always call `await entry.GetDatabaseValuesAsync(ct)`, check for `null` (the row may have been deleted during the conflict), and `entry.OriginalValues.SetValues(dbValues)`. Only then will the retry carry the fresh token in `WHERE` and be able to succeed. If you follow a Database-wins policy and want to reapply the user's edit on top of the fresh version, do not forget to set `CurrentValues` again, because `SetValues(dbValues)` overwrites current values too.

The third subtlety: `ConcurrencyCheck` on several fields is not equivalent to `Timestamp`. It works, but it checks every listed field, which is costlier and more fragile. Use a single `RowVersion` (or a single incrementing token), not a set of `ConcurrencyCheck`. Fourth: Client-wins is dangerous, because it overwrites the whole row with in-memory values — colleagues' edits are lost. Use it only if the business consciously accepts data loss. Fifth: do not wrap the entire data layer in a `DbUpdateConcurrencyException` handler "just in case" — catch narrowly, only where a conflict is realistic. Sixth: log conflicts — their frequency is a great signal that the UI poorly informs the user about data state. Seventh: under multi-threaded load each thread must own its DbContext — DbContext is not thread-safe, and sharing one context will give a race inside `ChangeTracker` itself.

#### Acceptance criteria

- [ ] The `ConcurrencyLab` project builds under .NET 8 via `dotnet build` without warnings.
- [ ] Top-level statements are enabled; C# 12 features (collection expressions, pattern matching) are used.
- [ ] The `Product` entity has a `byte[] RowVersion` configured via the Fluent API `IsRowVersion()`.
- [ ] The `Document` entity has a `uint Version` configured via `IsConcurrencyToken()` with an increment in `SaveChangesAsync`.
- [ ] `OnModelCreating` configures the token explicitly, not relying solely on the attribute.
- [ ] The lost-update scenario without a token is reproduced and documented in a comment.
- [ ] With the token, the second `SaveChanges` throws `DbUpdateConcurrencyException`, which is caught and printed.
- [ ] `ConcurrencyResolver.UpdatePriceAsync` is implemented with three attempts and logging.
- [ ] The `catch` block calls `GetDatabaseValuesAsync`, checks for `null`, refreshes `OriginalValues` and `CurrentValues`.
- [ ] A Client-wins policy is implemented and shown to lose the colleague's edit.
- [ ] The SQL log via `LogTo` shows `WHERE ... RowVersion = @p` or `WHERE ... Version = @p`.
- [ ] The ten-task stress test runs without losing edits: the final price equals 209.
- [ ] The provider is selected by `USE_SQLITE`; both variants build.
- [ ] XML comments document the chosen conflict resolution policy.
- [ ] A README or a comment in `Program.cs` explains the difference between `IsRowVersion()` and `IsConcurrencyToken()`.

#### Hints

- If `SaveChanges` does not throw after adding the token, check that you did not keep an old database without migration: `EnsureCreated` does not apply changes to an existing DB; delete `shop.db` and `ShopDb` and recreate.
- To see the SQL, use `optionsBuilder.LogTo(Console.WriteLine, LogLevel.Information)`.
- To increment `Version`, override `SaveChangesAsync` and look for `EntityState.Modified` via `ChangeTracker.Entries<Document>()`.
- In `catch`, do not forget to `await` `GetDatabaseValuesAsync` — the synchronous `GetDatabaseValues` may block the thread in an async context.
- If the stress-test price is lower than expected, you are probably sharing one DbContext across all tasks; create a context inside each task.
- Database-wins policy: `OriginalValues.SetValues(dbValues)` + reapply the user's edit via `CurrentValues.SetValues`.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — ConcurrencyLab
// Reference solution: RowVersion + manual token + Database-wins retry

using Microsoft.EntityFrameworkCore;
using System.ComponentModel.DataAnnotations;

// --- Entity with a server-maintained rowversion token (SQL Server) ---
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }

    [Timestamp]
    public byte[] RowVersion { get; set; } = Array.Empty<byte>();
}

// --- Entity with a manual incrementing token (SQLite/PostgreSQL) ---
public class Document
{
    public int Id { get; set; }
    public string Body { get; set; } = string.Empty;
    public uint Version { get; set; } // incremented in SaveChangesAsync
}

public class ShopDbContext : DbContext
{
    public DbSet<Product> Products => Set<Product>();

    private readonly bool _useSqlite;
    public ShopDbContext(bool useSqlite) => _useSqlite = useSqlite;

    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        if (_useSqlite)
            options.UseSqlite("Data Source=shop.db");
        else
            options.UseSqlServer(
                @"Server=(localdb)\MSSQLLocalDB;Database=ShopDb;Trusted_Connection=True;TrustServerCertificate=True");

        // Log SQL so the token-based WHERE is visible
        options.LogTo(Console.WriteLine,
            new DbContextLoggerOptions { LogLevel = LogLevel.Information });
    }

    protected override void OnModelCreating(ModelBuilder mb)
    {
        mb.Entity<Product>(b =>
        {
            b.HasKey(p => p.Id);
            b.Property(p => p.Name).IsRequired().HasMaxLength(200);
            // Explicit Fluent configuration of the token (preferred over the attribute)
            b.Property(p => p.RowVersion).IsRowVersion();
        });
    }
}

public class DocDbContext : DbContext
{
    public DbSet<Document> Documents => Set<Document>();

    protected override void OnConfiguring(DbContextOptionsBuilder options) =>
        options.UseSqlite("Data Source=docs.db")
               .LogTo(Console.WriteLine,
                      new DbContextLoggerOptions { LogLevel = LogLevel.Information });

    protected override void OnModelCreating(ModelBuilder mb) =>
        mb.Entity<Document>(b =>
        {
            b.HasKey(d => d.Id);
            // Manual token: EF adds Version to WHERE, but the increment is our job
            b.Property(d => d.Version).IsConcurrencyToken();
        });

    // Increment Version for every Modified entry before saving
    public override Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        foreach (var e in ChangeTracker.Entries<Document>())
            if (e.State == EntityState.Modified)
                e.Entity.Version++; // crucial — otherwise the token never changes

        return base.SaveChangesAsync(ct);
    }
}

// Database-wins policy: re-read, reapply the edit, retry
public static class ConcurrencyResolver
{
    public static async Task<bool> UpdatePriceAsync(
        ShopDbContext db, int productId, decimal newPrice, CancellationToken ct = default)
    {
        for (int attempt = 0; attempt < 3; attempt++)
        {
            var product = await db.Products.FindAsync([productId], ct);
            if (product is null) return false;

            product.Price = newPrice;
            try
            {
                await db.SaveChangesAsync(ct);
                Console.WriteLine($"Attempt {attempt}: success, price={newPrice}");
                return true;
            }
            catch (DbUpdateConcurrencyException ex)
            {
                var entry = ex.Entries.Single();
                var dbValues = await entry.GetDatabaseValuesAsync(ct);
                if (dbValues is null) return false; // row was deleted

                Console.WriteLine($"Attempt {attempt}: conflict, refreshing token from DB");
                entry.OriginalValues.SetValues(dbValues);              // fresh token in WHERE
                entry.CurrentValues.SetValues(new { Price = newPrice }); // user's edit
            }
        }
        return false;
    }
}

// --- Demo ---
var useSqlite = Environment.GetEnvironmentVariable("USE_SQLITE") == "1";

await using (var seed = new ShopDbContext(useSqlite))
{
    await seed.Database.EnsureDeletedAsync();
    await seed.Database.EnsureCreatedAsync();
    seed.Products.Add(new Product { Name = "Mug", Price = 199.00m });
    await seed.SaveChangesAsync();
}

// Reproduce the lost update with a token
await using var dbA = new ShopDbContext(useSqlite);
await using var dbB = new ShopDbContext(useSqlite);

var a = await dbA.Products.FindAsync([1]);
var b = await dbB.Products.FindAsync([1]);

a!.Price = 210;
await dbA.SaveChangesAsync(); // succeeds, the DB token changes

b!.Name = "Thermos";
try { await dbB.SaveChangesAsync(); }
catch (DbUpdateConcurrencyException)
{
    Console.WriteLine("Conflict caught: dbB's edit rejected, as expected");
}

// Database-wins retry on the same dbB
var ok = await ConcurrencyResolver.UpdatePriceAsync(dbB, 1, 215m);
Console.WriteLine($"Retry result: {ok}");

// Stress test: 10 parallel price increments
var tasks = Enumerable.Range(0, 10).Select(_ => Task.Run(async () =>
{
    await using var db = new ShopDbContext(useSqlite);
    await ConcurrencyResolver.UpdatePriceAsync(db, 1, 0m, CancellationToken.None);
    // A real test would have atomic "read-compute-save" logic here
})).ToArray();
await Task.WhenAll(tasks);
```

Line-by-line walk-through: the `Product` class mirrors the lesson model with the `[Timestamp]` attribute — the cheapest and most reliable token for SQL Server, because the database itself updates the column. `Document` illustrates the alternative for SQLite/PostgreSQL: a plain `uint Version` marked with `IsConcurrencyToken()`. Note that `DocDbContext.SaveChangesAsync` is overridden — this is where `Version` is incremented for every Modified entry. Without this step the token never changes, and the `WHERE "Version" = @p1` check is useless — a direct "common mistake" from the lesson. In `OnModelCreating` the token is configured through Fluent API (`IsRowVersion()`, `IsConcurrencyToken()`), which is preferred over attributes: the configuration lives in one place and behaves consistently across providers.

The `ConcurrencyResolver.UpdatePriceAsync` method is the heart of the assignment. The loop of up to three attempts models a Database-wins retry policy. `FindAsync` with the collection expression `[productId]` (C# 12) loads the row; after `SaveChangesAsync` the `catch` block takes `ex.Entries.Single()` — exactly one conflicting entry. `GetDatabaseValuesAsync` returns a snapshot of the current DB state; if it is `null`, the row was deleted — return `false`. The key line is `entry.OriginalValues.SetValues(dbValues)`: it refreshes the "as loaded" snapshot with the fresh DB values, including the new token. Without it the retry would again compare the stale token and throw again. Then `entry.CurrentValues.SetValues(new { Price = newPrice })` reapplies the user's edit on top of the fresh version — a miniature "custom merge".

The `dbA`/`dbB` scenario deliberately reproduces the classic lost update: `dbA` saves the price first, `dbB` with its stale token snapshot gets `DbUpdateConcurrencyException`. This scenario illustrates the lesson analogy about a document revision number in a version control system: "your base is stale — re-read and try again". The ten-task stress test with its own DbContext per task cements the rule: DbContext is not thread-safe, one context per thread. In real code each task needs atomic "read-compute-save" logic with retry, otherwise even with a token some edits will be rejected (but never lost — that is the point of optimistic concurrency). Logging via `LogTo` makes the `WHERE ... RowVersion = @p` visible, proving the token works at the SQL level.

#### Going deeper (bonus)

1. Implement a Custom-merge policy: on a conflict, merge the user's edits with the live DB values field by field. For example, if the user changed only `Name` and a colleague changed only `Price`, both edits should survive. Use `PropertyEntry.CurrentValue` and `OriginalValue` per field.
2. Add a second manual token on `Document` — a string `ETag` computed as a hash of `Body` (for instance `XxHash64`). Show that `IsConcurrencyToken()` works not only with numbers but also with strings, and that refreshing the `ETag` in an overridden `SaveChangesAsync` makes the check meaningful.
3. Connect a real SQL Server LocalDB and compare the SQL produced by `IsRowVersion()` (with a `rowversion` column) against the SQL for `IsConcurrencyToken()` (with a plain `bigint`). Describe in the README why `rowversion` is more efficient on SQL Server.
4. Cover `ConcurrencyResolver.UpdatePriceAsync` with xUnit unit tests: one test — a successful update without a conflict, the second — one conflict followed by success, the third — three conflicts in a row returning `false`. Be careful with the in-memory provider `UseInMemoryDatabase` — it does not support tokens, so prefer SQLite in-memory (`Data Source=:memory:`).

---

#### Чек-лист сдачи / Submission checklist

- [ ] Проект `ConcurrencyLab` собирается под .NET 8 без предупреждений.
- [ ] Токены настроены через Fluent API в `OnModelCreating` для `Product` и `Document`.
- [ ] Воспроизведён lost-update без токена и с токеном — оба сценария задокументированы.
- [ ] Реализован `ConcurrencyResolver.UpdatePriceAsync` с retry и логированием.
- [ ] В `catch` обновляются `OriginalValues` и `CurrentValues` через `GetDatabaseValuesAsync`.
- [ ] Реализована политика Client-wins, показана потеря данных.
- [ ] SQL-лог показывает `WHERE` по токену.
- [ ] Стресс-тест из 10 задач не теряет изменения (цена = 209).
- [ ] Провайдер переключается через `USE_SQLITE`.
- [ ] XML-комментарии и README объясняют политику и разницу токенов.
- [ ] Project `ConcurrencyLab` builds under .NET 8 with no warnings.
- [ ] Tokens are configured via Fluent API in `OnModelCreating` for both `Product` and `Document`.
- [ ] The lost-update scenario is reproduced with and without a token and documented.
- [ ] `ConcurrencyResolver.UpdatePriceAsync` with retry and logging is implemented.
- [ ] The `catch` block refreshes `OriginalValues` and `CurrentValues` via `GetDatabaseValuesAsync`.
- [ ] A Client-wins policy is implemented and data loss is demonstrated.
- [ ] The SQL log shows the token-based `WHERE`.
- [ ] The 10-task stress test loses no edits (price = 209).
- [ ] The provider is switchable through `USE_SQLITE`.
- [ ] XML comments and the README explain the policy and the difference between tokens.

#### Ресурсы / Resources

- [Microsoft Learn — Optimistic concurrency in EF Core](https://learn.microsoft.com/ef/core/saving/concurrency)
- [Microsoft Learn — Tokens and row versions](https://learn.microsoft.com/ef/core/modeling/concurrency)
- [EF Core reference — `DbUpdateConcurrencyException`](https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.dbupdateconcurrencyexception)
- [SQLite row version alternatives](https://www.sqlite.org/lang_createtable.html)
- [Microsoft Learn — `LogTo` SQL logging](https://learn.microsoft.com/ef/core/logging-events-diagnostics/)
