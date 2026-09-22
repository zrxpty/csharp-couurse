[⬆ К модулю M08](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M08-L04: GroupBy, ToLookup / GroupBy, ToLookup

**Модуль / Module:** M08
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Группировка — одна из самых мощных операций LINQ. Она позволяет собрать последовательность элементов в «корзины» по общему признаку, чтобы затем обрабатывать каждую корзину отдельно. Представьте себе огромный склад, на который ежедневно прибывают посылки. Сначала их сваливают в одну кучу на конвейере, а потом сортировщик раскладывает их по полкам: посылки из Москвы — на одну полку, из Казани — на другую, из Новосибирска — на третью. Именно это делает `GroupBy`: берёт «конвейер» элементов и распределяет их по «полкам» по ключу.

Ключом группировки может быть что угодно: простое свойство (например, `City`), вычисляемое выражение (`Order.Total % 2 == 0 ? "чёт" : "нечёт"`), составной объект (анонимный тип `new { Year, Month }`). Главное требование — ключ должен корректно сравниваться, то есть для него должен работать `GetHashCode` и `Equals`. Если вы группируете по собственному классу, не переопределившему равенство, каждая запись уйдёт в отдельную группу — классическая ловушка.

Результат `GroupBy` — это `IEnumerable<IGrouping<TKey, TElement>>`. `IGrouping` — необычный интерфейс: он одновременно является коллекцией элементов группы (наследует `IEnumerable<TElement>`) и несёт в себе свойство `Key`. Получается «полка с этикеткой»: этикетка — это ключ, а содержимое — перечислимая коллекция элементов. Поэтому по группе можно итерироваться `foreach`, а ключ достать через `group.Key`. Группировка выполняется **отложенно**: до тех пор, пока вы не начнёте перечислять результат, ничего не происходит. Само распределение по «корзинам» запускается в момент первого обращения — например, при вызове `ToList()` или `foreach`.

Здесь важно помнить о **многократном перечислении**. Если вы вызовете `GroupBy` без материализации и обойдёте результат дважды, группировка выполнится дважды. Поэтому после `GroupBy` часто сразу ставят `ToList()` или `ToDictionary()`, чтобы закрепить результат. Особенно опасен `GroupBy` в шаблоне `if (query.Any()) { foreach (var g in query) ... }` — здесь группировка отработает два раза.

Когда отложенного выполнения становится слишком много, на сцену выходит `ToLookup`. Это `GroupBy`, который выполняется **немедленно**. Вызов `ToLookup` строит `ILookup<TKey, TElement>` — неизменяемую мульти-словарную структуру, где каждому ключу соответствует набор значений. В отличие от обычного `Dictionary<TKey, TValue>`, где ключ уникален и хранит одно значение, `ILookup` допускает множество значений на один ключ. Это как телефонная книга: одному имени (ключу) может соответствовать несколько номеров (значений). Доступ к значениям происходит через индексатор `lookup[key]` — и он **никогда не выбрасывает `KeyNotFoundException`**: если ключа нет, возвращается пустая последовательность. Это очень удобно и безопасно.

Когда что выбирать? Если вы строите конвейер LINQ и продолжаете его дальше (`Select`, `OrderBy`, `Sum` и т. д.), используйте `GroupBy` — отложенное выполнение позволяет оптимизировать весь запрос. Если же вам нужна готовая структура данных для многократных быстрых запросов по ключу, особенно когда ключей много и вы будете к ним возвращаться — берите `ToLookup`. Типичный сценарий `ToLookup`: загрузить список заказов, сгруппировать по `CustomerId`, а потом по ходу бизнес-логики часто отвечать на вопрос «какие заказы у этого клиента?».

Обе операции принимают компаратор `IEqualityComparer<TKey>` для нестандартного сравнения ключей (например, регистронезависимого сравнения строк). У `GroupBy` и `ToLookup` есть перегрузки с селектором элемента (`elementSelector`), позволяющие сохранить в группе не весь исходный объект, а только нужное поле — это экономит память и упрощает последующую обработку. Помните: группировка — это не окончательная агрегация; это лишь подготовка «корзин», содержимое которых вы затем агрегируете (`Count`, `Sum`, `Average`) или проецируете дальше.

#### Theory (EN)

Grouping is one of the most powerful LINQ operations. It lets you collect a sequence of elements into “buckets” that share a common trait, so you can process each bucket independently. Imagine a giant warehouse where parcels arrive all day long. First they pile up in a single heap on a conveyor belt, and then a sorter distributes them onto shelves: parcels from Moscow go to one shelf, from Kazan to another, from Novosibirsk to a third. That is exactly what `GroupBy` does: it takes the “conveyor” of elements and distributes them onto “shelves” keyed by a value.

The grouping key can be anything: a simple property (such as `City`), a computed expression (`Order.Total % 2 == 0 ? "even" : "odd"`), or a composite object (an anonymous type `new { Year, Month }`). The only requirement is that the key must be comparable, meaning `GetHashCode` and `Equals` must work correctly for it. If you group by a custom class that has not overridden equality, every item ends up in its own group — a classic trap.

The result of `GroupBy` is `IEnumerable<IGrouping<TKey, TElement>>`. `IGrouping` is an unusual interface: it is both a collection of the group’s elements (it inherits `IEnumerable<TElement>`) and carries a `Key` property. Think of it as a “labeled shelf”: the label is the key, and the contents are an enumerable collection of elements. That is why you can iterate a group with `foreach`, while also reading `group.Key`. Grouping is **deferred**: until you start enumerating the result, nothing actually happens. The distribution into “buckets” starts on first access — for example, when you call `ToList()` or start a `foreach`.

Here you must remember **multiple enumeration**. If you call `GroupBy` without materialization and iterate the result twice, grouping runs twice. That is why after `GroupBy` you often immediately put `ToList()` or `ToDictionary()` to pin the result. A particularly dangerous pattern is `GroupBy` inside `if (query.Any()) { foreach (var g in query) ... }` — grouping executes twice here.

When deferred execution becomes too costly, `ToLookup` enters the stage. It is `GroupBy` that executes **eagerly**. Calling `ToLookup` builds an `ILookup<TKey, TElement>` — an immutable, multi-value dictionary-like structure where each key maps to a collection of values. Unlike a regular `Dictionary<TKey, TValue>`, where a key is unique and stores a single value, `ILookup` allows many values per key. It is like a phone book: one name (key) can correspond to several phone numbers (values). You access the values through the indexer `lookup[key]`, and it **never throws `KeyNotFoundException`**: if the key is missing, you get back an empty sequence. That is very convenient and safe.

When should you choose which? If you are building a LINQ pipeline and continue it further (`Select`, `OrderBy`, `Sum`, and so on), use `GroupBy` — deferred execution lets the whole query be optimized together. If instead you need a ready-to-use data structure for many repeated fast lookups by key, especially when there are many keys and you will keep coming back to them — use `ToLookup`. A typical `ToLookup` scenario: load a list of orders, group them by `CustomerId`, and then during business logic repeatedly answer “what orders does this customer have?”.

Both operations accept an `IEqualityComparer<TKey>` for non-standard key comparison (case-insensitive string comparison, for example). `GroupBy` and `ToLookup` also have overloads with an element selector (`elementSelector`), which lets you store not the whole source object in the group but only the field you need — this saves memory and simplifies downstream processing. Remember: grouping is not a final aggregation; it is only preparation of “buckets”, whose contents you then aggregate (`Count`, `Sum`, `Average`) or project further.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8+ — GroupBy и ToLookup / GroupBy and ToLookup
using System;
using System.Collections.Generic;
using System.Linq;

namespace M08L04;

// Модель заказа / Order model
public sealed record Order(int OrderId, string CustomerId, string City, decimal Total);

public static class GroupingDemo
{
    public static void Run()
    {
        // Исходные данные — «конвейер» / Source data — the “conveyor”
        var orders = new List<Order>
        {
            new(1, "C1", "Moscow", 100m),
            new(2, "C2", "Kazan",  250m),
            new(3, "C1", "Moscow",  75m),
            new(4, "C3", "Kazan",  300m),
            new(5, "C2", "Moscow",  50m),
            new(6, "C1", "Novosibirsk", 400m),
        };

        // 1) GroupBy — отложенное выполнение / deferred execution.
        // Группируем по городу, элементом оставляем только Total.
        // Group by city, keep only Total as the element.
        var byCity = orders
            .GroupBy(o => o.City, o => o.Total)
            .Select(g => new { City = g.Key, Count = g.Count(), Sum = g.Sum() });

        Console.WriteLine("GroupBy по городу / GroupBy by city:");
        foreach (var row in byCity)
            Console.WriteLine($"  {row.City}: orders={row.Count}, sum={row.Sum}");

        // 2) Многократное перечисление опасно без материализации.
        // Multiple enumeration is dangerous without materialization.
        var grouped = orders.GroupBy(o => o.CustomerId); // ещё не выполнено / not executed yet
        var list = grouped.ToList(); // фиксируем результат / pin the result

        // 3) ToLookup — немедленное выполнение, мульти-значение по ключу.
        // Eager execution, multiple values per key.
        ILookup<string, Order> byCustomer = orders.ToLookup(o => o.CustomerId);

        // Индексатор НЕ выбрасывает KeyNotFoundException — возвращает пустую последовательность.
        // The indexer never throws — returns an empty sequence for missing keys.
        Console.WriteLine("\nToLookup по CustomerId / ToLookup by CustomerId:");
        foreach (var group in byCustomer)
        {
            Console.WriteLine($"  {group.Key}: {group.Count()} order(s)");
            foreach (var o in group)
                Console.WriteLine($"    - #{o.OrderId}, {o.City}, {o.Total}");
        }

        var unknown = byCustomer["NO_SUCH_CUSTOMER"]; // пусто, без исключения / empty, no exception
        Console.WriteLine($"\nНесуществующий ключ / Missing key: count={unknown.Count()}");

        // 4) Компаратор: регистронезависимая группировка по городу.
        // Comparator: case-insensitive grouping by city.
        var caseInsensitive = orders
            .GroupBy(o => o.City, StringComparer.OrdinalIgnoreCase)
            .Select(g => (City: g.Key, Count: g.Count()));

        Console.WriteLine("\nGroupBy с компаратором / GroupBy with comparer:");
        foreach (var (city, count) in caseInsensitive)
            Console.WriteLine($"  {city}: {count}");
    }
}

// Ключевая разница в одном предложении:
// Key difference in one sentence:
// GroupBy — отложенный конвейер IEnumerable<IGrouping<,>> для дальнейшей композиции;
// ToLookup — готовая структура ILookup<,> с быстрым доступом по ключу и мульти-значением.
```

#### Best Practices

- Сразу материализуйте результат `GroupBy` (`ToList`, `ToDictionary`), если планируете обходить его больше одного раза — иначе группировка выполнится повторно.
- Используйте `elementSelector`, чтобы хранить в группе только нужные поля: это сокращает потребление памяти и упрощает последующий код.
- Для нестандартного сравнения ключей (регистр строк, культура, кастомные классы) всегда передавайте `IEqualityComparer<TKey>` — не полагайтесь на `Equals` по умолчанию для своих типов.
- Выбирайте `ToLookup`, когда нужна многократная быстрая выборка по ключу в рамках бизнес-логики — это превращает «запрос» в готовый индекс.
- Не забывайте, что `IGrouping` — это и коллекция, и носитель `Key`: обращайтесь к `group.Key` явно, не пересчитывайте ключ заново внутри цикла.
- Prefer `ToLookup` over `Dictionary<TKey, List<T>>` built manually — it is immutable, handles missing keys gracefully, and communicates intent clearly.

#### Частые ошибки / Common Mistakes

- Группировка по собственному классу без переопределённых `Equals`/`GetHashCode` → каждая запись уходит в отдельную группу. Переопределите равенство или используйте компаратор / Group by a custom class without `Equals`/`GetHashCode` → each item lands in its own group. Override equality or use a comparer.
- Многократный обход `IEnumerable<IGrouping>` без `ToList` → группировка выполняется заново каждый раз. Материализуйте результат / Iterate `IEnumerable<IGrouping>` multiple times without `ToList` → grouping runs again each time. Materialize the result.
- Ожидание `KeyNotFoundException` от `lookup[key]` → его не будет, вернётся пустая последовательность. Проверяйте `Count()` или `Any()` / Expecting `KeyNotFoundException` from `lookup[key]` → it never happens, an empty sequence is returned. Check `Count()` or `Any()`.
- Использование `GroupBy` там, где нужен частый доступ по ключу → медленно на больших данных. Берите `ToLookup` / Using `GroupBy` where frequent key access is needed → slow on large data. Use `ToLookup`.
- Путаница между `elementSelector` и `resultSelector`: `elementSelector` задаёт, что хранить в группе, а `resultSelector` — во что превратить каждую группу в финальной проекции / Confusing `elementSelector` and `resultSelector`: `elementSelector` chooses what to store in a group, while `resultSelector` transforms each group into the final projection.
- Изменение исходной коллекции после `ToLookup` и ожидание, что «индекс» обновится → `ILookup` неизменяем и не отслеживает источник / Mutating the source collection after `ToLookup` and expecting the index to update → `ILookup` is immutable and does not track the source.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я понимаю, что `GroupBy` выполняется отложенно, а `ToLookup` — немедленно.
- [ ] Я знаю, что результат `GroupBy` — `IEnumerable<IGrouping<TKey, TElement>>`, и `IGrouping` несёт `Key` плюс элементы.
- [ ] Я материализую `GroupBy` (`ToList`), если обхожу результат более одного раза.
- [ ] Я помню, что `ILookup` допускает несколько значений на ключ, в отличие от `Dictionary`.
- [ ] Я знаю, что `lookup[key]` не выбрасывает исключение для отсутствующего ключа.
- [ ] Я передаю `IEqualityComparer<TKey>` при нестандартном сравнении ключей.
- [ ] I understand that `GroupBy` is deferred, while `ToLookup` is eager.
- [ ] I know the result of `GroupBy` is `IEnumerable<IGrouping<TKey, TElement>>`, and `IGrouping` carries `Key` plus elements.
- [ ] I materialize `GroupBy` with `ToList` when iterating the result more than once.
- [ ] I remember that `ILookup` supports multiple values per key, unlike `Dictionary`.
- [ ] I know that `lookup[key]` does not throw for a missing key.
- [ ] I pass `IEqualityComparer<TKey>` for non-standard key comparison.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.linq.enumerable.groupby](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.groupby)

---

[⬆ К модулю M08](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
