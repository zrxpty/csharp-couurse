[⬆ К модулю M08](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M08-L06: Агрегаты: Sum/Min/Max/Average/Count / Aggregates: Sum/Min/Max/Average/Count

**Модуль / Module:** M08
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Агрегация — это свёртка всей последовательности в одно значение. Представьте себе гору монет: вы можете их пересчитать (`Count`), найти самую тяжёлую и самую лёгкую (`Min`/`Max`), сложить их номиналы (`Sum`) или вычислить средний вес (`Average`). LINQ даёт вам ровно такие инструменты, только работают они с любыми `IEnumerable<T>`.

**Базовые агрегаты.** `Count()` возвращает количество элементов. `LongCount()` — то же, но `long`, для огромных коллекций (более `int.MaxValue`). `Sum()` и `Average()` складывают или усредняют числовые проекции. `Min()` и `Max()` находят экстремумы. Все они имеют перегрузки: без аргументов (для последовательностей чисел) и с селектором-проекцией `selector => decimal` (например, `orders.Sum(o => o.Total)`).

**Пустые последовательности — главный сюрприз.** Поведение агрегатов на пустом вводе НЕодинаково:
- `Count`, `LongCount`, `Sum`, `Average` (а также `Min`/`Max` для nullable-типов) возвращают `0` или `null`. `Sum` пустой последовательности — это `0`, а не исключение.
- `Min`/`Max` для НЕ-nullable значимых типов (`int`, `double`) бросают `InvalidOperationException` на пустом входе. Это классический баг в продакшене.
- `Average` для НЕ-nullable чисел на пустой коллекции тоже бросает исключение; для nullable возвращает `null`.

Поэтому, если есть хоть малейший риск пустоты — используйте nullable-перегрузки или защищайтесь `Any()`.

**Nullable-семантика.** Агрегаты `Min`/`Max`/`Sum`/`Average` существуют в nullable-вариантах (`int?`, `double?`). Они автоматически пропускают `null`-элементы и возвращают `null`, если все элементы `null` или коллекция пуста. Это безопаснее, чем ловить исключения.

**Aggregate — универсальный агрегатор.** Когда встроенных функций не хватает (например, нужно накопить строку, собрать агрегат-объект или реализовать свою логику), приходит `Aggregate`. Он работает как `fold`/`reduce` в функциональных языках: идёт по элементам, накапливая результат в «аккумуляторе».

Три основные формы:
1. `Aggregate(func)` — первый элемент становится начальным аккумулятором; для 0 или 1 элемента бросает исключение.
2. `Aggregate(seed, func)` — задаёте начальное значение `seed`; безопасно для пустых последовательностей (вернёт `seed`).
3. `Aggregate(seed, func, resultSelector)` — финальная проекция результата.

Аналогия: `Sum` — это частный случай `Aggregate(0, (acc, x) => acc + x)`. Понимая `Aggregate`, вы понимаете весь механизм агрегации «под капотом».

**Производительность.** Все агрегаты — один проход по коллекции (O(n)). Для `Min`/`Max` на уже отсортированных данных можно взять `[0]` и `[^1]`, но LINQ-агрегаты универсальнее и читаемее. Не вызывайте несколько агрегатов подряд по одним данным, если можно обойти один раз вручную — но в 95% случаев читаемость важнее микросекунд.

**Когда НЕ использовать агрегаты.** Если вам нужна новая коллекция — берите `Select`/`Where`. Агрегат возвращает скаляр. Если нужно сгруппировать и просуммировать по группам — сначала `GroupBy`, затем агрегат по каждой группе.

#### Theory (EN)

Aggregation is the act of folding an entire sequence down to a single scalar value. Imagine a pile of coins: you can count them (`Count`), find the heaviest and lightest (`Min`/`Max`), add up their face values (`Sum`), or compute the average weight (`Average`). LINQ gives you exactly these tools, and they work on any `IEnumerable<T>`.

**The basic aggregates.** `Count()` returns the number of elements. `LongCount()` does the same but returns a `long`, which you need for truly huge sequences (more than `int.MaxValue` items). `Sum()` and `Average()` add or average numeric projections. `Min()` and `Max()` locate the extremes. Each comes in overloads: parameterless (for sequences of numbers) and with a selector projection like `selector => decimal` (e.g. `orders.Sum(o => o.Total)`).

**Empty sequences — the big surprise.** Aggregate behavior on empty input is NOT uniform:
- `Count`, `LongCount`, `Sum`, and `Average` (plus `Min`/`Max` for nullable types) return `0` or `null`. The sum of an empty sequence is `0`, never an exception.
- `Min`/`Max` on non-nullable value types (`int`, `double`) throw `InvalidOperationException` on empty input. This is a classic production bug.
- `Average` on non-nullable numbers also throws on an empty collection; the nullable overload returns `null`.

So whenever emptiness is even possible, prefer the nullable overloads or guard with `Any()`.

**Nullable semantics.** `Min`/`Max`/`Sum`/`Average` each have nullable variants (`int?`, `double?`). They automatically skip `null` elements and return `null` when every element is `null` or the sequence is empty. That is far safer than catching exceptions.

**Aggregate — the universal aggregator.** When the built-ins are not enough (say, you need to build up a string, accumulate a composite object, or implement custom logic), `Aggregate` steps in. It works like `fold`/`reduce` in functional languages: it walks the elements, accumulating a result in an "accumulator".

Three main forms:
1. `Aggregate(func)` — the first element becomes the initial accumulator; throws for zero or one element.
2. `Aggregate(seed, func)` — you supply a starting `seed`; safe for empty sequences (returns `seed`).
3. `Aggregate(seed, func, resultSelector)` — adds a final projection of the result.

The analogy: `Sum` is just `Aggregate(0, (acc, x) => acc + x)`. Once you understand `Aggregate`, you understand the entire aggregation mechanism under the hood.

**Performance.** All aggregates are a single pass (O(n)). For `Min`/`Max` on already-sorted data you could grab `[0]` and `[^1]`, but LINQ aggregates are more general and more readable. Avoid chaining several aggregates over the same data when one manual pass would do — but in 95% of cases readability beats microseconds.

**When NOT to use aggregates.** If you need a new collection, reach for `Select`/`Where`; an aggregate returns a scalar. If you need to group and then sum per group, first `GroupBy`, then aggregate each group.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Агрегаты LINQ / LINQ Aggregates
using System;
using System.Collections.Generic;
using System.Linq;

var orders = new List<Order>
{
    new(1, "Alice", 120.50m, 3),
    new(2, "Bob",   75.00m, 1),
    new(3, "Alice", 200.00m, 5),
    new(4, "Charlie", 0m, 0),     // бесплатный заказ / free order
    // new(5, "Dave", null, 2)    // представим, что Total может быть null
};

// --- Базовые агрегаты / Basic aggregates ---
int count = orders.Count;                         // 4 — общее число / total count
int aliceCount = orders.Count(o => o.Customer == "Alice"); // 2
long longCount = orders.LongCount();              // long — для огромных выборок

decimal totalSum = orders.Sum(o => o.Total);      // 395.50 — сумма / sum
decimal maxOrder = orders.Max(o => o.Total);      // 200.00 — максимум / max
decimal minOrder = orders.Min(o => o.Total);      // 0.00 — минимум / min
double avgItems  = orders.Average(o => o.Items);  // 2.25 — среднее / average

Console.WriteLine($"Count={count}, Sum={totalSum}, Max={maxOrder}, Avg={avgItems}");

// --- Nullable-безопасность / Nullable safety ---
// Минимум по пустой коллекции НЕ-nullable бросит исключение:
// orders.Where(o => o.Total > 1000).Min(o => o.Total); // InvalidOperationException!

// Безопасный вариант — через nullable-проекцию / Safe via nullable projection:
decimal? safeMin = orders
    .Where(o => o.Total > 1000)
    .Select(o => (decimal?)o.Total)
    .Min();                                       // null — без исключения / null, no throw

// Average по пустой / Average on empty:
double? safeAvg = orders
    .Where(o => o.Total > 1000)
    .Select(o => (double?)o.Items)
    .Average();                                   // null

Console.WriteLine($"Safe min (empty): {safeMin?.ToString() ?? "null"}");

// --- GroupBy + агрегаты / GroupBy + aggregates ---
var byCustomer = orders
    .GroupBy(o => o.Customer)
    .Select(g => new
    {
        Customer = g.Key,
        Orders   = g.Count(),
        Revenue  = g.Sum(o => o.Total),
        AvgCheck = g.Average(o => o.Total),
        MaxCheck = g.Max(o => o.Total),
    });

foreach (var s in byCustomer)
    Console.WriteLine($"{s.Customer}: {s.Orders} orders, revenue={s.Revenue}, avg={s.AvgCheck:F2}");

// --- Aggregate: пользовательская агрегация / Custom aggregation ---
// Конкатенация имён через запятую / Join names with commas
string names = orders
    .Select(o => o.Customer)
    .Distinct()
    .Aggregate(
        seed: "Customers: ",
        func: (acc, name) => acc + name + ", ",
        resultSelector: acc => acc.TrimEnd(',', ' '));
// "Customers: Alice, Bob, Charlie"
Console.WriteLine(names);

// Sum через Aggregate (для понимания «под капотом») / Sum via Aggregate (under the hood)
decimal manualSum = orders.Aggregate(0m, (acc, o) => acc + o.Total);
Console.WriteLine($"Manual sum = {manualSum}");

// Aggregate для накопления составного объекта / Accumulating a composite object
var stats = orders.Aggregate(
    seed: (Count: 0, Total: 0m, MaxItems: 0),
    func: (acc, o) => (acc.Count + 1, acc.Total + o.Total, Math.Max(acc.MaxItems, o.Items)));
Console.WriteLine($"Stats: count={stats.Count}, total={stats.Total}, maxItems={stats.MaxItems}");

// --- Пустая последовательность и Aggregate-seed / Empty sequence + seed ---
var empty = Array.Empty<Order>();
decimal emptySum = empty.Aggregate(0m, (acc, o) => acc + o.Total); // 0m — без исключений
Console.WriteLine($"Empty aggregate = {emptySum}");

record Order(int Id, string Customer, decimal Total, int Items);
```

#### Best Practices
- Для `Min`/`Max`/`Average` на данных, которые могут быть пустыми, используйте nullable-проекцию (`Select(x => (T?)x)`) или проверку `Any()` — это исключает `InvalidOperationException`.
- Предпочитайте перегрузку с селектором (`orders.Sum(o => o.Total)`) вместо `orders.Select(o => o.Total).Sum()` — один проход и читаемее.
- Для очень больших коллекций используйте `LongCount()`, чтобы избежать переполнения `int`.
- Используйте `Aggregate` с `seed` всегда, когда возможна пустая последовательность — без `seed` он бросит исключение.
- Для групповых сводок комбинируйте `GroupBy` и агрегаты в одной LINQ-цепочке — это декларативно и быстро.
- Remember that `Sum` of an empty sequence is `0`, not an error — design with this in mind rather than adding redundant guards.
- Prefer the overload with a selector (`orders.Sum(o => o.Total)`) over `orders.Select(o => o.Total).Sum()` — one pass, cleaner code.
- Use `LongCount()` for very large sequences to avoid `int` overflow.
- Always pass a `seed` to `Aggregate` when the sequence might be empty — without it, the call throws.
- Combine `GroupBy` with aggregates in a single LINQ chain for grouped summaries — declarative and efficient.

#### Частые ошибки / Common Mistakes
- `Min()`/`Max()` по пустой коллекции НЕ-nullable типов → `InvalidOperationException`. Избегайте: используйте `Select(x => (int?)x).Min()` или проверку `Any()`.
- Вызов `Average()` по пустой коллекции НЕ-nullable чисел → исключение. Избегайте: nullable-проекция или `if (collection.Any())`.
- `Aggregate(func)` без `seed` на пустой или одноэлементной коллекции → исключение. Избегайте: всегда передавайте `seed`.
- Путают `Count()` (метод LINQ, O(n)) со свойством `Count` (у `List<T>`/массивов, O(1)). Избегайте: для `ICollection<T>` свойство быстрее; LINQ `Count()` оптимизирован, но всё же проверяйте тип.
- Забывают, что `Sum` пустой последовательности равен `0`, а не `null` — пишут лишние проверки. Избегайте: помните семантику каждого агрегата.
- Используют несколько проходов (`Sum` + `Min` + `Max` + `Average` отдельно) там, где достаточно одного `Aggregate`. Избегайте: при критичности производительности сверните одним проходом.
- `Min()`/`Max()` on an empty non-nullable collection → `InvalidOperationException`. Avoid: use `Select(x => (int?)x).Min()` or guard with `Any()`.
- Calling `Average()` on an empty non-nullable numeric collection throws. Avoid: nullable projection or `if (collection.Any())`.
- `Aggregate(func)` without a `seed` on an empty or single-element sequence throws. Avoid: always pass a `seed`.
- Confusing `Count()` (LINQ method, O(n)) with the `Count` property (`List<T>`/arrays, O(1)). Avoid: for `ICollection<T>` prefer the property; LINQ `Count()` is optimized but check the type.
- Forgetting that `Sum` of an empty sequence is `0`, not `null` — adding redundant guards. Avoid: remember each aggregate's semantics.
- Running multiple passes (`Sum` + `Min` + `Max` + `Average` separately) where a single `Aggregate` suffices. Avoid: when performance matters, fold in one pass.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я знаю, какие агрегаты бросают исключение на пустой коллекции, а какие возвращают `0`/`null`.
- [ ] Я использую nullable-проекцию или `Any()` для `Min`/`Max`/`Average`, если возможна пустота.
- [ ] Я понимаю три формы `Aggregate` и всегда передаю `seed` для безопасности.
- [ ] Я выбираю перегрузку с селектором вместо `Select(...).Aggregate(...)`.
- [ ] Я знаю, что `Sum` пустой коллекции равен `0`, и не пишу лишних проверок.
- [ ] Я умею комбинировать `GroupBy` с агрегатами для сводок по группам.
- [ ] I know which aggregates throw on empty input and which return `0`/`null`.
- [ ] I use a nullable projection or `Any()` guard for `Min`/`Max`/`Average` when emptiness is possible.
- [ ] I understand the three `Aggregate` forms and always pass a `seed` for safety.
- [ ] I choose the overload with a selector over `Select(...).Aggregate(...)`.
- [ ] I know that `Sum` of an empty collection is `0` and skip redundant guards.
- [ ] I can combine `GroupBy` with aggregates for grouped summaries.

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.linq.enumerable.aggregate]

---

[⬆ К модулю M08](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
