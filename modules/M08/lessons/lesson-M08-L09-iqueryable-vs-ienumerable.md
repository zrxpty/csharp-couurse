[⬆ К модулю M08](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M08-L09: IEnumerable vs IQueryable, провайдеры / IEnumerable vs IQueryable, providers

**Модуль / Module:** M08
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

LINQ — это единый язык запросов, но «единый» здесь означает лишь общий синтаксис. Под капотом у LINQ два совершенно разных мира: `IEnumerable<T>` и `IQueryable<T>`. Понимание разницы между ними — это граница между разработчиком, который «пишет запросы», и разработчиком, который понимает, где и как эти запросы выполняются.

`IEnumerable<T>` — это последовательность в памяти. Когда вы вызываете на нём методы LINQ (`Where`, `Select`, `OrderBy`), вы используете **LINQ to Objects** — набор методов расширения, которые работают с делегатами (`Func<T,bool>` и т. п.). Каждый вызов создаёт новый итератор, который при перечислении (`foreach`) обходит исходную коллекцию и применяет фильтры один за другим. Делегат — это скомпилированный код: готовая функция, которую можно вызвать, но нельзя «разобрать» на части. Это значит, что фильтрация всегда происходит на стороне клиента, в памяти процесса. Если коллекция содержит миллион элементов, весь миллион будет перебран.

`IQueryable<T>` устроен иначе. Он хранит не делегаты, а **дерево выражений (Expression Tree)** — `Expression<Func<T,bool>>`. Дерево выражений — это структура данных, которая описывает, *что именно* вы хотите сделать, но не делает этого сама. Это как чертёж: по нему можно построить дом, но сам чертёж — это просто линии на бумаге. Важно: лямбда-выражение `x => x.Age > 18` компилируется в `Func` или в `Expression<Func>` в зависимости от типа параметра метода — это один и тот же синтаксис, но разный результат компиляции.

За превращение дерева выражений во что-то полезное отвечает **провайдер** (`IQueryProvider`). Каждый провайдер знает, как «объяснить» дерево своему источнику данных. Провайдер Entity Framework Core переводит дерево в SQL и отправляет в базу через драйвер. Провайдер LINQ to SQL делал то же самое в своё время. Существуют и другие: для Elasticsearch, для Cosmos DB, для in-memory тестов. Один и тот же C#-код `dbContext.Users.Where(u => u.Age > 18)` превращается в `SELECT ... FROM Users WHERE Age > 18`, а для Elasticsearch — в JSON-запрос. Вы пишете на C#, провайдер переводит.

Перевод в SQL имеет ограничения: не каждый C#-выражение можно перевести. Вызов произвольного метода (`u.MyCustomCheck()`), использования свойств, которых нет в базе, или сложных вычислений обычно приводят к исключению во время выполнения — провайдер просто не знает, как это выразить в SQL. Поэтому `IQueryable` нужно держать как можно дольше, аmaterialизовать (`ToList`, `FirstOrDefault`, `AsEnumerable`) как можно позже и как можно ближе к концу запроса — чтобы фильтры, проекции и сортировки ушли в SQL, а не выполнялись в памяти.

**Trade-offs.** `IEnumerable` прост, работает с любой коллекцией, не требует провайдера, но тащит все данные в память. `IQueryable` эффективен для больших наборов данных в источнике, потому что фильтрует на стороне источника, но требует переводимого дерева выражений и знания, что именно провайдер поддерживает. Главное правило: для баз данных используйте `IQueryable` (через `DbSet<T>`), и не вызывайте `AsEnumerable()` или `ToList()` раньше, чем закончили строить фильтр. Запомните золотую диаграмму: `Func` → код, который *выполняется*; `Expression` → данные, которые *переводятся*.

#### Theory (EN)

LINQ looks like one query language, but under the hood there are two different worlds: `IEnumerable<T>` and `IQueryable<T>`. Knowing the difference is the line between a developer who "writes queries" and a developer who knows where and how those queries actually execute.

`IEnumerable<T>` is an in-memory sequence. When you call LINQ methods on it (`Where`, `Select`, `OrderBy`), you are using **LINQ to Objects**: a set of extension methods that take delegates (`Func<T,bool>` and friends). Each call wraps the source in another iterator, and when you enumerate it (with `foreach`) the iterator walks the collection and applies every filter one by one. A delegate is compiled code: a ready-to-run function that you can call but cannot take apart. That means filtering always happens on the client, in process memory. If the collection has a million items, all million will be scanned.

`IQueryable<T>` works differently. It stores not delegates but an **expression tree** — `Expression<Func<T,bool>>`. An expression tree is a data structure that describes *what* you want to do without actually doing it. Think of it as a blueprint: you can build a house from it, but the blueprint itself is just lines on paper. Crucially, the lambda `x => x.Age > 18` compiles into either a `Func` or an `Expression<Func>` depending on the parameter type of the method it is passed to — same syntax, different compiler output.

Turning the expression tree into something useful is the job of a **provider** (`IQueryProvider`). Each provider knows how to "explain" the tree to its data source. The Entity Framework Core provider translates the tree into SQL and sends it to the database through a driver. The old LINQ to SQL provider did the same thing. Others exist: for Elasticsearch, for Cosmos DB, for in-memory testing. The same C# code `dbContext.Users.Where(u => u.Age > 18)` becomes `SELECT ... FROM Users WHERE Age > 18` against a relational database, or a JSON body against Elasticsearch. You write C#; the provider translates.

Translation has limits: not every C# expression can be translated. Calling an arbitrary method (`u.MyCustomCheck()`), touching properties that do not exist in the store, or doing complex computations usually throws at runtime — the provider simply does not know how to express it in SQL. That is why you should keep `IQueryable` as long as possible, and materialize (`ToList`, `FirstOrDefault`, `AsEnumerable`) as late and as close to the end of the query as you can — so filters, projections, and ordering travel into SQL instead of running in memory.

**Trade-offs.** `IEnumerable` is simple, works with any collection, and needs no provider, but it pulls all data into memory. `IQueryable` is efficient for large remote data sets because it filters on the source side, but it requires a translatable expression tree and knowledge of what the provider supports. The core rule: for databases use `IQueryable` (via `DbSet<T>`), and do not call `AsEnumerable()` or `ToList()` before you have finished building the filter. Remember the golden diagram: `Func` → code that *executes*; `Expression` → data that *gets translated*.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — IEnumerable vs IQueryable: same syntax, very different behavior
using System;
using System.Collections.Generic;
using System.Linq;
using Microsoft.EntityFrameworkCore;

// === Часть 1: IEnumerable — LINQ to Objects, фильтрация в памяти ===
// === Part 1: IEnumerable — LINQ to Objects, filtering in memory ===

List<User> localUsers = new()
{
    new(1, "Анна", 17),
    new(2, "Борис", 25),
    new(3, "Вера", 30),
    new(4, "Глеб", 15),
};

// Версия IEnumerable: Where принимает Func<User, bool> — скомпилированный делегат.
// IEnumerable version: Where takes Func<User, bool> — a compiled delegate.
IEnumerable<User> adultsInMemory = localUsers.Where(u => u.Age >= 18);

// Делегат нельзя «разобрать» — это уже готовый код. Перечисление выполняет его.
// A delegate cannot be taken apart — it is ready-to-run code. Enumeration runs it.
foreach (var u in adultsInMemory)
{
    Console.WriteLine($"[In-memory] {u.Name}, {u.Age}");
}

// === Часть 2: IQueryable — дерево выражений, провайдер переводит в SQL ===
// === Part 2: IQueryable — expression tree, provider translates to SQL ===

using AppDbContext db = new();

// DbSet<User> реализует IQueryable<User>. Здесь Where принимает Expression<Func<User,bool>>.
// DbSet<User> implements IQueryable<User>. Here Where takes Expression<Func<User,bool>>.
IQueryable<User> adultsFromDb = db.Users.Where(u => u.Age >= 18);

// Ничего не отправляется в базу, пока мы не материализуем запрос.
// Nothing is sent to the database until we materialize the query.
Console.WriteLine("--- SQL будет выполнен только на строке ниже / SQL runs only on the next line ---");

// ToListAsync() запускает провайдер: дерево выражений переводится в SQL и выполняется в БД.
// ToListAsync() triggers the provider: the expression tree is translated to SQL and run in the DB.
List<User> adultUsers = await adultsFromDb.ToListAsync();

// EF Core умеет показать сгенерированный SQL — отличная отладочная привычка.
// EF Core can show the generated SQL — a great debugging habit.
Console.WriteLine(db.Users.Where(u => u.Age >= 18).ToQueryString());

// === Часть 3: ловушка ранней материализации (AsEnumerable) ===
// === Part 3: the early-materialization trap (AsEnumerable) ===

// ПЛОХО: AsEnumerable() переключает на LINQ to Objects прямо здесь.
// Все последующие операторы выполнятся в памяти, а не в БД.
// BAD: AsEnumerable() switches to LINQ to Objects right here.
// Every operator after this runs in memory, not in the DB.
var bad = db.Users
    .AsEnumerable()                  // ← дальше идёт IEnumerable<User> / from here it is IEnumerable<User>
    .Where(u => u.Age >= 18)         // фильтр в памяти, хотя мог бы быть в SQL / in-memory filter, could have been SQL
    .OrderBy(u => u.Name)
    .ToList();                       // тянет ВСЕ строки из таблицы / pulls ALL rows from the table

// ХОРОШО: AsEnumerable только после того, как фильтры и проекции построены.
// GOOD: AsEnumerable only after filters and projections are built.
var good = db.Users
    .Where(u => u.Age >= 18)         // → SQL WHERE Age >= 18
    .OrderBy(u => u.Name)            // → SQL ORDER BY
    .Select(u => new { u.Name, u.Age }) // → SQL SELECT (проекция уменьшает объём данных)
    .AsEnumerable()                  // материализуем минимальный набор / materialize a minimal set
    .ToList();

// === Модель данных и контекст ===
// === Data model and context ===

file record User(int Id, string Name, int Age);

file class AppDbContext : DbContext
{
    public DbSet<User> Users => Set<User>();

    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlite("Data Source=app.db");
}
```

#### Best Practices

- Держите запрос `IQueryable` как можно дольше; материализуйте (`ToList`, `FirstOrDefault`, `AsEnumerable`) только в конце, после всех фильтров и проекций.
- Проектируйте (`Select`) в DTO/анонимные типы до материализации, чтобы не тащить из источника лишние колонки.
- Проверяйте сгенерированный SQL (EF Core: `ToQueryString()`, логирование) — это лучший способ поймать неэффективные запросы.
- Не пишите сложную логику внутри выражений, которую провайдер не сможет перевести; выносите её в SQL-функции или материализуйте сознательно.
- Для коллекций в памяти используйте `IEnumerable<T>` — `IQueryable` там не даёт выгоды и только запутывает читателя.

- Keep the query `IQueryable` as long as possible; materialize (`ToList`, `FirstOrDefault`, `AsEnumerable`) only at the end, after all filters and projections.
- Project (`Select`) into DTOs/anonymous types before materialization so you do not pull unnecessary columns from the source.
- Inspect the generated SQL (EF Core: `ToQueryString()`, logging) — the best way to catch inefficient queries.
- Do not put logic inside expressions that the provider cannot translate; move it to SQL functions or materialize on purpose.
- For in-memory collections use `IEnumerable<T>` — `IQueryable` gives no benefit there and only confuses the reader.

#### Частые ошибки / Common Mistakes

- `AsEnumerable()` или `ToList()` в середине запроса → следующие операторы идут в память. Как избежать: материализуйте только в финальной строке цепочки.
- Смешивание `IEnumerable` и `IQueryable` через `IEnumerable<T> result = dbContext.Users.Where(...)`. → Все дальнейшие операторы бесшумно переключаются на LINQ to Objects. Как избежать: явно используйте `IQueryable<T>` для переменных, пока запрос не закончен.
- Вызов собственного метода внутри `Where` (`u => u.IsCool()`). → Провайдер бросает `InvalidOperationException` о непереводимом выражении. Как избежать: либо переводите логику в выражение из примитивов, либо материализуйте осознанно и фильтруйте в памяти.
- Ожидание, что `IQueryable` работает как `IEnumerable` с любым источником. → `IQueryable` бесполезен без провайдера. Как избежать: используйте `IEnumerable` для массивов и списков.
- Игнорирование N+1: цикл `foreach` по `IQueryable`, внутри которого идёт подзапрос. → N запросов к БД. Как избежать: используйте `Include`/`Join` и материализуйте до цикла.

- `AsEnumerable()` or `ToList()` in the middle of a query → later operators run in memory. How to avoid: materialize only on the final line of the chain.
- Mixing `IEnumerable` and `IQueryable` via `IEnumerable<T> result = dbContext.Users.Where(...)`. → All subsequent operators silently switch to LINQ to Objects. How to avoid: explicitly type in-flight variables as `IQueryable<T>`.
- Calling your own method inside `Where` (`u => u.IsCool()`). → The provider throws `InvalidOperationException` about a non-translatable expression. How to avoid: either express the logic with primitives, or materialize on purpose and filter in memory.
- Expecting `IQueryable` to work like `IEnumerable` against any source. → `IQueryable` is useless without a provider. How to avoid: use `IEnumerable` for arrays and lists.
- Ignoring N+1: a `foreach` over an `IQueryable` that issues a sub-query per iteration. → N database round-trips. How to avoid: use `Include`/`Join` and materialize before the loop.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я понимаю, что `IEnumerable<T>` использует `Func` (код), а `IQueryable<T>` использует `Expression` (данные).
- [ ] Я знаю, что `Where` на `IEnumerable` фильтрует в памяти, а на `IQueryable` — на стороне источника.
- [ ] Я могу объяснить роль `IQueryProvider` и привести два примера провайдеров (EF Core, LINQ to SQL).
- [ ] Я не ставлю `AsEnumerable()`/`ToList()` раньше, чем построил фильтры и проекции.
- [ ] Я проверяю сгенерированный SQL, когда есть сомнения в эффективности запроса.
- [ ] Я знаю, что не каждое C#-выражение переводится в SQL, и понимаю, как это диагностировать.

- [ ] I understand that `IEnumerable<T>` uses `Func` (code) while `IQueryable<T>` uses `Expression` (data).
- [ ] I know that `Where` on `IEnumerable` filters in memory, while on `IQueryable` it filters on the source side.
- [ ] I can explain the role of `IQueryProvider` and name two providers (EF Core, LINQ to SQL).
- [ ] I do not put `AsEnumerable()`/`ToList()` before filters and projections are built.
- [ ] I inspect the generated SQL when I doubt a query's efficiency.
- [ ] I know that not every C# expression translates to SQL, and I can diagnose it.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.linq.iqueryable-1](https://learn.microsoft.com/dotnet/api/system.linq.iqueryable-1)

---

[⬆ К модулю M08](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
