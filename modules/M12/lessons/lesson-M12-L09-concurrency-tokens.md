[⬆ К модулю M12](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M12-L09: Concurrency tokens, optimistic concurrency / Concurrency tokens, optimistic concurrency

**Модуль / Module:** M12
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Когда несколько пользователей одновременно редактируют одну и ту же строку в базе данных, возникает риск «потерянного обновления» (lost update). Представьте двух редакторов, которые правят один и тот же документ: первый сохраняет изменения, затем второй сохраняет свои — и перезаписывает работу первого, даже не подозревая об этом. Чтобы этого избежать, EF Core поддерживает **оптимистическую конкуренцию** (optimistic concurrency).

Идея оптимистичного подхода проста: мы не блокируем строку на чтение (как делает pessimistic concurrency), а предполагаем, что конфликты случаются редко. Вместо блокировки мы добавляем в таблицу специальное поле — **токен конкуренции** (concurrency token). При каждом `UPDATE` или `DELETE` EF Core автоматически добавляет в `WHERE` условие проверку: «значение токена должно совпадать с тем, что было при загрузке». Если за это время кто-то другой изменил строку — токен изменился, строка не находится, и `SaveChanges` выбрасывает `DbUpdateConcurrencyException`.

Самый распространённый токен в SQL Server — это атрибут `[Timestamp]` (или `byte[] RowVersion` с `IsRowVersion()`). Это автоматически обновляемое сервером поле: база сама инкрементирует его при каждом изменении строки, поэтому его невозможно «подделать» вручную. В других провайдерах (PostgreSQL, SQLite) аналог строится через обычное поле с пометкой `.IsConcurrencyToken()` — например, GUID или целое число, которое приложение увеличивает само.

Токены настраиваются двумя способами: через Data Annotations (`[Timestamp]`, `[ConcurrencyCheck]`) или через Fluent API в `OnModelCreating` — `Property(p => p.RowVersion).IsRowVersion()` или `.IsConcurrencyToken()`. Fluent API предпочтительнее: он чище и работает одинаково для всех провайдеров.

Когда летит `DbUpdateConcurrencyException`, у вас есть три стандартных стратегии разрешения конфликта. Первая — **Database wins**: перечитать актуальные значения из БД и показать пользователю ошибку, заставив его решить, что делать. Вторая — **Client wins**: перезаписать БД значениями из памяти (опасно, теряет чужие данные). Третья — **Custom merge**: объединить изменения вручную, например, взять свежее значение токена из БД, но оставить правки пользователя. В обработчике исключения доступны `entry.GetDatabaseValues()` — снимок текущего состояния в БД, `entry.OriginalValues` — то, что было при загрузке, и `entry.CurrentValues` — правки пользователя. Сравнивая их, можно реализовать любую политику.

Важно: оптимистичная конкуренция не бесплатна. Она требует дополнительного поля в каждой таблице и аккуратной обработки исключений на каждом `SaveChanges`, который может конфликтовать. Зато она масштабируется лучше пессимистичных блокировок: нет дедлоков, нет зависших транзакций, нет проблем с подключениями. Для веб-приложений, где запрос короткий и состояние между запросами не хранится, optimistic concurrency — де-факто стандарт.

Аналогия: optimistic concurrency — как номер редакции документа в системе контроля версий. Вы начали править редакцию №5, пока коллега уже сохранил редакцию №6. При попытке закоммитить вашу версию №5 система скажет: «отстань, основа устарела, перечитай и попробуй снова». Никаких замков на шкафу с документами — только честная проверка в момент сохранения.

#### Theory (EN)

When multiple users edit the same database row at the same time, you risk a **lost update**. Imagine two editors working on the same document: the first saves their changes, then the second saves theirs and silently overwrites the first one's work without even noticing. To prevent this, EF Core supports **optimistic concurrency**.

The optimistic approach is simple: instead of locking a row on read (as pessimistic concurrency does), we assume conflicts are rare. Rather than locking, we add a special column to the table — a **concurrency token**. On every `UPDATE` or `DELETE`, EF Core automatically appends a `WHERE` clause that checks «the token value must match what it was when the row was loaded». If someone else modified the row in between, the token has changed, the row is not found, and `SaveChanges` throws a `DbUpdateConcurrencyException`.

The most common token in SQL Server is the `[Timestamp]` attribute (or a `byte[] RowVersion` configured with `IsRowVersion()`). It is a server-maintained column: the database itself increments it on every row modification, so it cannot be faked by hand. On other providers (PostgreSQL, SQLite) the equivalent is a plain property marked with `.IsConcurrencyToken()` — for example a GUID or an integer that the application increments itself.

Tokens are configured in two ways: via Data Annotations (`[Timestamp]`, `[ConcurrencyCheck]`) or via the Fluent API in `OnModelCreating` — `Property(p => p.RowVersion).IsRowVersion()` or `.IsConcurrencyToken()`. Fluent API is preferred: it is cleaner and behaves consistently across providers.

When `DbUpdateConcurrencyException` is thrown, you have three standard resolution strategies. The first is **Database wins**: re-read the current values from the DB and show the user an error, forcing them to decide what to do. The second is **Client wins**: overwrite the DB with the in-memory values (dangerous — you lose someone else's data). The third is **Custom merge**: combine the changes manually, for example by taking the fresh token value from the DB but keeping the user's edits. In the exception handler you have access to `entry.GetDatabaseValues()` — a snapshot of the current DB state, `entry.OriginalValues` — what was there at load time, and `entry.CurrentValues` — the user's edits. By comparing them you can implement any policy.

Note that optimistic concurrency is not free. It requires an extra column on every table and careful exception handling on every `SaveChanges` that may conflict. But it scales far better than pessimistic locks: no deadlocks, no stuck transactions, no connection-bound issues. For web applications, where a request is short and no state is kept between requests, optimistic concurrency is the de facto standard.

Analogy: optimistic concurrency is like a document revision number in a version control system. You started editing revision 5, while a colleague already committed revision 6. When you try to commit your revision 5, the system says: «stop, your base is stale — re-read and try again». No padlocks on the document cabinet — just an honest check at the moment of saving.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8+ — Optimistic concurrency with a rowversion token
// Оптимистичная конкуренция через rowversion-токен

using Microsoft.EntityFrameworkCore;
using System.ComponentModel.DataAnnotations;

// Сущность с токеном конкуренции / Entity with a concurrency token
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }

    // [Timestamp] => byte[], SQL Server сам обновляет при каждом изменении строки
    // [Timestamp] => byte[], SQL Server auto-updates it on every row change
    [Timestamp]
    public byte[] RowVersion { get; set; } = Array.Empty<byte>();
}

public class ShopDbContext : DbContext
{
    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Эквивалент [Timestamp] через Fluent API (предпочтительный способ)
        // Fluent API equivalent of [Timestamp] (preferred)
        modelBuilder.Entity<Product>(b =>
        {
            b.HasKey(p => p.Id);
            b.Property(p => p.Name).IsRequired().HasMaxLength(200);
            b.Property(p => p.RowVersion).IsRowVersion(); // токен конкуренции / concurrency token
        });
    }
}

// Альтернатива без [Timestamp]: обычный токен, который инкрементируем сами
// Alternative without [Timestamp]: a plain token we increment ourselves
public class Document
{
    public int Id { get; set; }
    public string Body { get; set; } = string.Empty;

    // Токен конкуренции для провайдеров без rowversion (PostgreSQL, SQLite)
    // Concurrency token for providers without rowversion
    public uint Version { get; set; }
}

public class DocDbContext : DbContext
{
    public DbSet<Document> Documents => Set<Document>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Document>(b =>
        {
            b.HasKey(d => d.Id);
            b.Property(d => d.Version).IsConcurrencyToken(); // помечаем как токен / mark as token
        });
    }
}

// Обработчик конфликта с политикой Database-wins + повторная попытка
// Conflict handler with Database-wins policy + retry
public static class ConcurrencyResolver
{
    // Возвращает true, если обновление в итоге прошло успешно
    // Returns true if the update eventually succeeded
    public static async Task<bool> UpdatePriceAsync(
        ShopDbContext db, int productId, decimal newPrice, CancellationToken ct = default)
    {
        for (int attempt = 0; attempt < 3; attempt++)
        {
            var product = await db.Products.FindAsync([productId], ct);
            if (product is null)
                return false; // строку удалили / row was deleted

            product.Price = newPrice;

            try
            {
                await db.SaveChangesAsync(ct);
                return true; // успешно / success
            }
            catch (DbUpdateConcurrencyException ex)
            {
                // Конфликт: кто-то изменил строку раньше нас
                // Conflict: someone changed the row before us
                var entry = ex.Entries.Single();

                // Database-wins: перечитать актуальные значения из БД
                // Database-wins: re-read current values from the DB
                var dbValues = await entry.GetDatabaseValuesAsync(ct);
                if (dbValues is null)
                    return false; // строку удалили за время конфликта / deleted during conflict

                // Обновляем OriginalValues свежим токеном, чтобы следующая попытка прошла
                // Refresh OriginalValues with the fresh token so the next attempt succeeds
                entry.OriginalValues.SetValues(dbValues);
                // Сбрасываем CurrentValues цены, чтобы применить её к свежей версии
                // Reapply the price on top of the fresh version
                entry.CurrentValues.SetValues(new { Price = newPrice });
            }
        }
        return false; // слишком много конфликтов подряд / too many conflicts in a row
    }
}
```

#### Best Practices

- Используйте `[Timestamp]` / `IsRowVersion()` в SQL Server и `IsConcurrencyToken()` + инкрементируемое поле в PostgreSQL/SQLite — это самый дешёвый и надёжный токен.
- Предпочитайте Fluent API Data Annotations: конфигурация остаётся в одном месте и одинаково работает для всех провайдеров.
- По умолчанию выбирайте политику Database-wins: она безопаснее всего и не теряет чужие данные.
- Ловите `DbUpdateConcurrencyException` точечно, в тех `SaveChanges`, где конфликт реален; не оборачивайте им весь слой доступа к данным без нужды.
- Логируйте конфликты: их частота — отличный сигнал о том, что UI плохо информирует пользователя о состоянии данных.
- В UI показывайте пользователю разницу между его правками, оригиналом и актуальной версией из БД при конфликте.

- Use `[Timestamp]` / `IsRowVersion()` on SQL Server and `IsConcurrencyToken()` with an auto-incremented field on PostgreSQL/SQLite — the cheapest and most reliable token.
- Prefer Fluent API over Data Annotations: configuration stays in one place and behaves consistently across providers.
- Default to a Database-wins policy: it is the safest and never loses anyone's data.
- Catch `DbUpdateConcurrencyException` narrowly, only on `SaveChanges` where a conflict is realistic; do not wrap the whole data layer with it unnecessarily.
- Log conflicts: their frequency is a great signal that the UI poorly informs the user about data state.
- In the UI, show the user the diff between their edits, the original, and the live DB version on a conflict.

#### Частые ошибки / Common Mistakes

- Забыли пометить токен (`[Timestamp]` или `IsConcurrencyToken()`) → EF Core не добавит проверку в `WHERE`, и конфликты будут молча перезаписывать данные. Всегда явно настраивайте токен в `OnModelCreating`.
- Ловят `DbUpdateConcurrencyException`, но не обновляют `OriginalValues` перед повторной попыткой → следующий `SaveChanges` падает с тем же исключением. После конфликта всегда перечитывайте `GetDatabaseValues()` и ставьте их в `OriginalValues`.
- Используют `Guid.NewGuid()` как токен и забывают обновлять его при каждом изменении → токен никогда не меняется, проверка бесполезна. Для ручных токенов инкрементируйте значение в сеттере или в переопределённом `SaveChanges`.
- Думают, что `ConcurrencyCheck` на нескольких полях эквивалентен `Timestamp` → это работает, но проверяет все указанные поля, что дороже и хрупче. Используйте единый `RowVersion` вместо набора `ConcurrencyCheck`.
- При политике Client-wins перезаписывают всю строку значениями из памяти → теряются изменения коллег. Никогда не используйте Client-wins без осознанного решения бизнеса.

- Forgot to mark the token (`[Timestamp]` or `IsConcurrencyToken()`) → EF Core won't add the `WHERE` check, and conflicts will silently overwrite data. Always configure the token explicitly in `OnModelCreating`.
- Catch `DbUpdateConcurrencyException` but don't refresh `OriginalValues` before retrying → the next `SaveChanges` fails with the same exception. After a conflict, always re-read `GetDatabaseValues()` and set them into `OriginalValues`.
- Use `Guid.NewGuid()` as a token and forget to update it on every change → the token never changes, the check is useless. For manual tokens, increment the value in the setter or in an overridden `SaveChanges`.
- Assume `ConcurrencyCheck` on several fields is equivalent to `Timestamp` → it works, but it checks every listed field, which is costlier and more fragile. Use a single `RowVersion` instead of a set of `ConcurrencyCheck`.
- With a Client-wins policy, overwrite the whole row with in-memory values → colleagues' changes are lost. Never use Client-wins without a deliberate business decision.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] В каждой редактируемой сущности есть токен конкуренции (`RowVersion` или `IsConcurrencyToken()`).
- [ ] Токен настроен через Fluent API в `OnModelCreating`, а не только через атрибут.
- [ ] `SaveChanges`, где возможен конфликт, обёрнут в `try/catch (DbUpdateConcurrencyException)`.
- [ ] В обработчике исключения вызывается `GetDatabaseValuesAsync()` и обновляются `OriginalValues`.
- [ ] Выбрана и задокументирована политика разрешения конфликта (Database/Client/Custom).
- [ ] В UI пользователь видит причину конфликта и может принять решение.
- [ ] Конфликты логируются для последующего анализа частоты.

- [ ] Every editable entity has a concurrency token (`RowVersion` or `IsConcurrencyToken()`).
- [ ] The token is configured via Fluent API in `OnModelCreating`, not only via an attribute.
- [ ] `SaveChanges` where a conflict is possible is wrapped in `try/catch (DbUpdateConcurrencyException)`.
- [ ] The exception handler calls `GetDatabaseValuesAsync()` and refreshes `OriginalValues`.
- [ ] A conflict resolution policy (Database/Client/Custom) is chosen and documented.
- [ ] The UI shows the user the cause of the conflict and lets them decide.
- [ ] Conflicts are logged for later frequency analysis.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/ef/core/saving/concurrency]

---

[⬆ К модулю M12](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
