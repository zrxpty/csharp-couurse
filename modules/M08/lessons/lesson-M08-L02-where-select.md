[⬆ К модулю M08](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M08-L02: Where, Select, проекции / Where, Select, projections

**Модуль / Module:** M08
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

LINQ — это единый язык запросов к любым источникам данных: коллекциям в памяти, базам данных, XML, веб-сервисам. В этом уроке мы разберём два самых фундаментальных оператора — `Where` и `Select`, — а также анонимные типы, `SelectMany` и технику построения цепочек запросов (chaining).

**`Where` — фильтрация.** Оператор `Where` принимает предикат (функцию, возвращающую `bool`) и оставляет в последовательности только те элементы, для которых предикат истинен. Аналогия из жизни: вы стоите перед стеллажом с книгами и отбираете только те, где год издания новее 2020. Остальные остаются на полке, в результат они не попадают. Важно понимать: `Where` не изменяет исходную коллекцию и не изменяет сам элемент — он лишь решает, пропускать ли элемент дальше по конвейеру.

**`Select` — проекция (преобразование).** Если `Where` уменьшает количество элементов, то `Select` меняет их форму. Он применяет функцию-преобразование к каждому элементу и возвращает новую последовательность результатов. Аналогия: на складе каждый товар лежит в большой коробке с полным описанием, но покупателю нужен только ярлык с ценой. `Select` «достаёт» нужное поле или собирает новую форму. Длина выходной последовательности совпадает с длиной входной (кроме случая с `SelectMany`).

**Анонимные типы.** Часто проекция должна собрать несколько полей в новый объект, но создавать отдельный класс ради этого — избыточно. Анонимные типы (`new { ... }`) позволяют на лету сформировать структуру с нужными свойствами. Компилятор сам генерирует класс с правильными свойствами и реализацией `Equals`/`GetHashCode` по значению полей. Начиная с C# 7 удобнее использовать кортежи (`(Name: x.Name, Price: x.Price)`), но анонимные типы незаменимы, когда результат передаётся в LINQ-выражения, требующие ссылочного типа, или сериализуется в JSON.

**`SelectMany` — сплющивание вложенных последовательностей.** Если проекция возвращает коллекцию (например, у каждого заказа есть список позиций), то обычный `Select` даст «последовательность последовательностей»: `IEnumerable<IEnumerable<Item>>`. `SelectMany` «разворачивает» её в плоскую `IEnumerable<Item>`. Аналогия: у вас несколько коробок с яблоками, вы высыпаете их все на один стол — получается одна куча яблок вместо набора коробок.

**Цепочки (chaining).** Все LINQ-операторы возвращают `IEnumerable<T>`, поэтому их можно соединять в цепочку: `source.Where(...).Select(...).OrderBy(...).Take(...)`. Данные проходят по конвейеру: каждый оператор берёт выход предыдущего и подаёт на вход следующему. Порядок имеет значение — обычно фильтрацию (`Where`) ставят раньше проекции (`Select`), чтобы не преобразовывать элементы, которые всё равно будут отброшены: это и быстрее, и часто безопаснее (поля, нужные для фильтра, ещё доступны).

**Отложенное исполнение.** `Where` и `Select` не выполняются в момент вызова — они лишь строят «рецепт» запроса. Реальная работа начинается при перечислении результата (`foreach`, `ToList()`, `ToArray()`, `Count()`). Это означает, что если исходная коллекция изменится между построением запроса и его выполнением, результат будет отражать актуальное состояние. Польза: можно строить запрос модульно; опасность: повторное перечисление повторно выполнит всю работу.

**Синтаксис методов vs синтаксис запросов.** В C# есть две записи LINQ: fluent-стиль (`xs.Where(x => x > 0).Select(x => x * 2)`) и query-стиль (`from x in xs where x > 0 select x * 2`). Они компилируются в одинаковый код; выбирайте тот, что читается яснее. Для `SelectMany` query-стиль часто лаконичнее: несколько `from`-клауз подряд разворачиваются именно в `SelectMany`.

#### Theory (EN)

LINQ (Language Integrated Query) is a single query language over any data source: in-memory collections, databases, XML, web services. In this lesson we focus on the two most fundamental operators — `Where` and `Select` — plus anonymous types, `SelectMany`, and the technique of chaining operators into a pipeline.

**`Where` — filtering.** `Where` takes a predicate (a function returning `bool`) and keeps only the elements for which the predicate is true. Real-life analogy: you stand in front of a bookshelf and pick only the books published after 2020; the rest stay on the shelf and never enter your result. Crucially, `Where` does not mutate the source collection and does not change the element itself — it only decides whether the element may continue down the conveyor.

**`Select` — projection (transformation).** If `Where` reduces the number of elements, `Select` changes their shape. It applies a transformation function to each element and returns a new sequence of results. Analogy: every product in a warehouse sits in a big box with a full description, but the customer only needs a price tag. `Select` extracts one field or assembles a brand-new shape. The output sequence has the same length as the input (except for `SelectMany`).

**Anonymous types.** Often a projection must bundle several fields into a new object, yet declaring a dedicated class for that is overkill. Anonymous types (`new { ... }`) let you shape a structure on the fly with named properties. The compiler generates a class with correct properties and value-based `Equals`/`GetHashCode`. Since C# 7, tuples (`(Name: x.Name, Price: x.Price)`) are often more convenient, but anonymous types are essential when the result feeds LINQ operators that expect a reference type or is serialized to JSON.

**`SelectMany` — flattening nested sequences.** When a projection returns a collection (for example, each order has a list of line items), a plain `Select` yields a "sequence of sequences": `IEnumerable<IEnumerable<Item>>`. `SelectMany` flattens it into a single `IEnumerable<Item>`. Analogy: you have several boxes of apples and pour them all onto one table — you get one pile of apples instead of a set of boxes.

**Chaining.** Every LINQ operator returns `IEnumerable<T>`, so they compose into a chain: `source.Where(...).Select(...).OrderBy(...).Take(...)`. Data flows along a conveyor: each operator takes the previous one's output and feeds the next one. Order matters — usually filtering (`Where`) goes before projection (`Select`) so that you do not transform elements that will be discarded anyway. This is both faster and often safer, because the fields the filter needs are still in scope.

**Deferred execution.** `Where` and `Select` do not run at the call site — they only build a query "recipe". The real work happens when the result is enumerated (`foreach`, `ToList()`, `ToArray()`, `Count()`). That means if the source changes between building the query and running it, the result reflects the current state. The upside: you can assemble a query in pieces; the danger: re-enumerating re-runs all the work.

**Method syntax vs query syntax.** C# has two LINQ notations: fluent style (`xs.Where(x => x > 0).Select(x => x * 2)`) and query style (`from x in xs where x > 0 select x * 2`). They compile to the same code; pick whichever reads more clearly. For `SelectMany`, query syntax is often terser: several consecutive `from` clauses translate directly into `SelectMany`.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8+ — рабочий пример Where, Select, анонимные типы, SelectMany, chaining
using System;
using System.Collections.Generic;
using System.Linq;

namespace M08L02;

public record Product(int Id, string Name, decimal Price, string Category, List<string> Tags);

public record Order(int OrderId, List<Product> Items);

public static class WhereSelectDemo
{
    public static void Run()
    {
        // Исходные данные / Source data
        var products = new List<Product>
        {
            new(1, "Ноутбук Pro / Laptop Pro", 1200m, "Electronics", new() { "new", "popular" }),
            new(2, "Мышка / Mouse", 25m, "Accessories", new() { "sale" }),
            new(3, "Клавиатура / Keyboard", 80m, "Accessories", new() { "popular" }),
            new(4, "Монитор 4K / Monitor 4K", 450m, "Electronics", new() { "new" }),
            new(5, "Коврик / Pad", 15m, "Accessories", new() { "sale", "popular" })
        };

        var orders = new List<Order>
        {
            new(101, new() { products[0], products[2] }),
            new(102, new() { products[1], products[3], products[4] })
        };

        // 1) Where — фильтр: только дорогие товары / only expensive products
        //    Фильтр ставим ПЕРВЫМ — не тратим работу на проекцию лишнего / filter first
        var expensive = products
            .Where(p => p.Price > 100m)
            .Select(p => p.Name);
        Console.WriteLine("Дорогие > 100 / Expensive > 100: " + string.Join(", ", expensive));

        // 2) Select с анонимным типом — новая форма на лету / new shape on the fly
        //    Анонимный тип удобен для «отчётной» формы без отдельного класса
        var priceList = products
            .Select(p => new { p.Name, p.Price, Tax = p.Price * 0.20m });
        foreach (var row in priceList)
            Console.WriteLine($"  {row.Name} = {row.Price:C} (tax {row.Tax:C})");

        // 3) SelectMany — сплющить теги всех товаров в один плоский список / flatten tags
        var allTags = products.SelectMany(p => p.Tags).Distinct().OrderBy(t => t);
        Console.WriteLine("Все теги / All tags: " + string.Join(", ", allTags));

        // 4) SelectMany по заказам → все позиции всех заказов / orders → all line items
        var allItems = orders.SelectMany(o => o.Items);
        Console.WriteLine($"Всего позиций во всех заказах / Total items: {allItems.Count()}");

        // 5) Цепочка: фильтр → проекция в анонимный тип → сортировка → взять топ-3
        //    Chain: filter → project → order → take top 3
        var topDeals = products
            .Where(p => p.Tags.Contains("sale"))   // только со скидкой / only on sale
            .Select(p => new { p.Name, p.Price })  // проекция / projection
            .OrderBy(x => x.Price)                 // дёшево → дорого / cheap → expensive
            .Take(3);                              // первые три / first three
        Console.WriteLine("Топ скидок / Top deals:");
        foreach (var d in topDeals)
            Console.WriteLine($"  {d.Name} — {d.Price:C}");

        // 6) Query-синтаксис — эквивалент цепочке выше / query syntax equivalent
        var topDealsQuery =
            (from p in products
             where p.Tags.Contains("sale")
             orderby p.Price
             select new { p.Name, p.Price }).Take(3);

        // 7) Отложенное исполнение — запрос не выполнен до перечисления
        //    Deferred execution — query not run until enumerated
        var query = products.Where(p => p.Price < 100m).Select(p => p.Name);
        products.Add(new Product(6, "Стикер / Sticker", 5m, "Accessories", new()));
        // При перечислении увидим и стикер — он добавлен ПОСЛЕ построения запроса
        Console.WriteLine("Дешёвые < 100 (со стикером) / Cheap < 100 (with sticker): "
            + string.Join(", ", query));

        // Чтобы «заморозить» результат — материализуем через ToList()
        // To freeze the result, materialize with ToList()
        var frozen = products.Where(p => p.Price < 100m).Select(p => p.Name).ToList();
        products.Add(new Product(7, "Ластик / Eraser", 3m, "Accessories", new()));
        Console.WriteLine($"Frozen count = {frozen.Count} (ластик не попал / eraser excluded)");
    }
}
```

#### Best Practices

- Ставьте `Where` до `Select`, чтобы не преобразовывать элементы, которые будут отброшены — это быстрее и безопаснее (поля фильтра ещё в области видимости).
- Материализуйте результат (`ToList()`/`ToArray()`), если планируете перечислять его несколько раз или если исходная коллекция может измениться — иначе каждое перечисление повторяет всю работу.
- Предпочитайте кортежи (`(Name, Price)`) анонимным типам, когда результат используется только локально — кортежи проще передавать между методами.
- Используйте `SelectMany` сознательно: если проекция возвращает `IEnumerable<T>`, а вам нужен плоский список, `SelectMany` — правильный выбор; обычный `Select` даст вложенность.
- Выбирайте query-синтаксис для запросов с несколькими `from` (они компилируются в `SelectMany`) и сложными `join`/`group` — он читается чище; для коротких цепочек fluent-стиль лаконичнее.
- Давайте переменным в лямбдах понятные имена (`p`, `order`, `item`), а не однобуквенные, если внутри больше одного уровня вложенности.

- Put `Where` before `Select` so you do not transform elements that will be discarded — it is faster and safer (the fields the filter needs are still in scope).
- Materialize the result (`ToList()`/`ToArray()`) when you plan to enumerate it multiple times or when the source may change — otherwise every enumeration re-runs all the work.
- Prefer tuples (`(Name, Price)`) over anonymous types when the result is only used locally — tuples are easier to pass between methods.
- Use `SelectMany` deliberately: when a projection returns `IEnumerable<T>` and you want a flat list, `SelectMany` is the right tool; plain `Select` would nest.
- Pick query syntax for queries with several `from` clauses (they compile into `SelectMany`) and complex `join`/`group` — it reads more clearly; for short chains, fluent style is terser.
- Give lambda variables meaningful names (`p`, `order`, `item`) rather than single letters when there is more than one level of nesting inside.

#### Частые ошибки / Common Mistakes

- Вызов `Where` после `Select`, который выбросил нужное для фильтра поле → фильтр не компилируется или приходится переписывать. Ставьте `Where` раньше `Select`.
- Ожидание, что `Where`/`Select` выполнятся немедленно → пустой `foreach` «не считается» и запрос не запускается; меняете исходник — меняется результат. Решение: `ToList()`, когда нужна фиксация.
- Путают `Select` и `SelectMany`: проекция возвращает коллекцию, а получают `IEnumerable<IEnumerable<T>>` вместо плоского списка → используйте `SelectMany`.
- Мутируют коллекцию во время `foreach` по живому LINQ-запросу → `InvalidOperationException`. Сначала материализуйте через `ToList()`, потом меняйте.
- Используют анонимный тип как возвращаемое значение метода → не компилируется (тип без имени нельзя объявить в сигнатуре). Возвращайте кортеж или именованный тип.
- Забывают, что повторное перечисление `IEnumerable` от LINQ-to-Objects повторно выполняет запрос — для дорогих источников это скрытая потеря производительности.

- Calling `Where` after a `Select` that dropped the field the filter needs → the filter will not compile or has to be rewritten. Put `Where` before `Select`.
- Assuming `Where`/`Select` run immediately → an empty `foreach` "does not count" and the query never starts; if you edit the source, the result changes. Fix: `ToList()` when you need a snapshot.
- Confusing `Select` and `SelectMany`: a projection returns a collection, yet you end up with `IEnumerable<IEnumerable<T>>` instead of a flat list → use `SelectMany`.
- Mutating a collection while `foreach`-ing over a live LINQ query → `InvalidOperationException`. Materialize with `ToList()` first, then mutate.
- Using an anonymous type as a method's return value → does not compile (an unnamed type cannot appear in a signature). Return a tuple or a named type instead.
- Forgetting that re-enumerating an `IEnumerable` from LINQ-to-Objects re-runs the whole query — a hidden performance trap for expensive sources.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я ставлю `Where` до `Select` и могу объяснить, почему это эффективнее.
- [ ] Я знаю разницу между отложенным исполнением и материализацией (`ToList()`).
- [ ] Я могу сплющить вложенную коллекцию через `SelectMany`.
- [ ] Я создаю анонимный тип через `new { ... }` и понимаю ограничения его использования.
- [ ] Я строю цепочку из 3+ операторов и понимаю порядок передачи данных.
- [ ] Я могу переписать fluent-запрос в query-синтаксис и наоборот.

- [ ] I place `Where` before `Select` and can explain why it is more efficient.
- [ ] I know the difference between deferred execution and materialization (`ToList()`).
- [ ] I can flatten a nested collection with `SelectMany`.
- [ ] I can build an anonymous type with `new { ... }` and know its usage limits.
- [ ] I can build a chain of 3+ operators and understand the data flow order.
- [ ] I can rewrite a fluent query into query syntax and back.

#### Ресурсы / Resources

- [Microsoft Learn — Standard query operators overview](https://learn.microsoft.com/dotnet/csharp/linq/standard-query-operators/)
- [Microsoft Learn — LINQ (C#)](https://learn.microsoft.com/dotnet/csharp/linq/)
- [Microsoft Learn — Anonymous types](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/anonymous-types)

---

[⬆ К модулю M08](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
