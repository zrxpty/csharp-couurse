[⬆ К модулю M08](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M08-L05: Join, GroupJoin, Zip / Join, GroupJoin, Zip

**Модуль / Module:** M08
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Операции соединения позволяют объединять данные из двух последовательностей по общему ключу — точно так же, как в SQL JOIN связывает таблицы. В LINQ для этого служат три оператора: `Join`, `GroupJoin` и `Zip`. Каждый решает свою задачу и имеет свою семантику.

**Join — внутреннее соединение (inner join).** Оператор `Join` находит для каждого элемента внешней последовательности все совпадающие по ключу элементы внутренней последовательности и формирует пары. Если совпадения нет — элемент внешней последовательности просто исчезает из результата. Это аналог `INNER JOIN` в SQL. Подумайте о двух списках: клиенты и заказы. `Join` вернёт только те пары «клиент–заказ», у которых `ClientId` совпадает. Клиенты без заказов и заказы без клиентов в результат не попадут. Сигнатура метода требует четыре параметра: внешнюю последовательность, внутреннюю последовательность, селектор ключа внешней и селектор ключа внутренней последовательности, а также функцию-проектор, которая формирует итоговый элемент из пары.

**GroupJoin — соединение с группировкой (left join / grouped join).** В отличие от `Join`, `GroupJoin` сохраняет **все** элементы внешней последовательности. Для каждого внешнего элемента он создаёт группу из всех совпадающих внутренних элементов; если совпадений нет, группа будет пустой. Это ведёт себя как `LEFT JOIN` с группировкой, но не как классический `LEFT JOIN` с дублированием строк: результат — иерархическая структура, «клиент и его список заказов». Это удобно для построения деревьев, отчётов «мастер–детали», группированных представлений. Чтобы превратить `GroupJoin` в плоский `LEFT JOIN`, используйте `SelectMany` с `DefaultIfEmpty`.

**Zip — попарное объединение по позиции.** `Zip` принципиально отличается от `Join` и `GroupJoin`: он не использует ключи вообще. Вместо этого он берёт элементы из двух последовательностей **по индексу** — первый с первым, второй со вторым, и так далее. Длина результата равна длине более короткой последовательности. Это идеальный инструмент, когда нужно соединить два параллельных массива одинаковой длины: например, список имён и список оценок студента, чтобы получить пары «имя–оценка».

**Ключи соединения.** В качестве ключа может выступать любой тип, для которого корректно определено равенство — обычно это примитивы (`int`, `string`) или анонимные типы. Составной ключ строится через анонимный тип: `new { o.CustomerId, o.Year }`. Важно, что анонимные типы в C# реализуют структурное равенство, поэтому два анонимных объекта равны, если совпадают все их свойства одного типа в одном порядке. Это позволяет соединять по нескольким полям сразу.

**Производительность join.** Реализация `Join` и `GroupJoin` в LINQ to Objects построена на хеш-таблице: внутренняя последовательность сначала полностью загружается в словарь по ключу (`O(n)` времени и памяти), затем внешняя последовательность сканируется с поиском в словаре (`O(m)`). Итоговая сложность — `O(n + m)`, что намного лучше наивного вложенного цикла с `O(n·m)`. Однако есть нюансы: если внутренняя последовательность велика, потребление памяти растёт; если ключи плохо хешируются (коллизии), деградирует скорость. Для LINQ to Entities (EF Core) соединения транслируются в SQL JOIN — там правила оптимизации другие и лежат на стороне СУБД. Старайтесь соединять на стороне базы, когда это возможно, и только потом фильтровать и проектировать в памяти. Избегайте соединений в цикле — каждый вызов `Join` строит новый словарь.

#### Theory (EN)

Join operations let you combine data from two sequences by a shared key — just like SQL JOIN links tables. LINQ provides three operators for this: `Join`, `GroupJoin`, and `Zip`. Each solves a different problem and has its own semantics.

**Join — inner join.** The `Join` operator finds, for every element of the outer sequence, all matching elements of the inner sequence by key and produces pairs. If there is no match, the outer element simply disappears from the result. This is the analog of SQL `INNER JOIN`. Picture two lists: customers and orders. `Join` returns only those customer–order pairs whose `CustomerId` matches. Customers without orders and orders without customers never appear. The method signature takes four parameters: the outer sequence, the inner sequence, an outer key selector, an inner key selector, and a result projector that builds the final element from the matched pair.

**GroupJoin — grouped join (left join / grouped join).** Unlike `Join`, `GroupJoin` preserves **all** outer elements. For each outer element it produces a group of all matching inner elements; when nothing matches, the group is empty. This behaves like a `LEFT JOIN` with grouping, not like a classic flat `LEFT JOIN` that duplicates rows: the result is a hierarchical structure — “customer and the list of their orders”. This is great for building trees, master–detail reports, and grouped views. To flatten `GroupJoin` into a classic `LEFT JOIN`, combine it with `SelectMany` and `DefaultIfEmpty`.

**Zip — positional pairing.** `Zip` is fundamentally different from `Join` and `GroupJoin`: it uses no keys at all. Instead, it pairs elements from two sequences **by index** — the first with the first, the second with the second, and so on. The result length equals the length of the shorter sequence. It is the perfect tool when you have two parallel arrays of equal length, for example a list of subject names and a list of a student’s grades, and you want pairs of “subject–grade”.

**Join keys.** A key can be any type with correct equality semantics — usually primitives (`int`, `string`) or anonymous types. A composite key is built with an anonymous type: `new { o.CustomerId, o.Year }`. Anonymous types in C# implement structural equality, so two anonymous objects are equal when all their properties, with matching types, match in the same order. That lets you join on several fields at once.

**Join performance.** The LINQ to Objects implementation of `Join` and `GroupJoin` is hash-based: the inner sequence is first loaded into a dictionary keyed by the join key (`O(n)` time and memory), then the outer sequence is scanned with dictionary lookups (`O(m)`). The overall complexity is `O(n + m)`, far better than a naive nested loop at `O(n·m)`. There are caveats: a large inner sequence grows memory usage, and poorly hashing keys (collisions) degrade speed. For LINQ to Entities (EF Core), joins are translated to SQL JOINs — optimization rules differ and live on the database side. Prefer joining on the database whenever possible, and only filter and project in memory afterward. Avoid joins inside loops — every `Join` call builds a new dictionary.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Join, GroupJoin, Zip
// Демонстрация трёх операторов соединения / Demo of three join operators

using System;
using System.Collections.Generic;
using System.Linq;

// Модели / Models
public record Customer(int Id, string Name);
public record Order(int OrderId, int CustomerId, decimal Amount, int Year);

var customers = new List<Customer>
{
    new(1, "Алиса"),            // Alice
    new(2, "Борис"),            // Boris — без заказов / no orders
    new(3, "Виктор"),           // Victor
};

var orders = new List<Order>
{
    new(101, 1, 500m, 2024),
    new(102, 1, 300m, 2023),
    new(103, 3, 700m, 2024),
};

// 1) Join — внутреннее соединение / inner join
// Вернёт только клиентов, у которых есть заказы / Only customers that have orders
var innerJoin = customers.Join(
    orders,
    c => c.Id,                  // внешний ключ / outer key
    o => o.CustomerId,          // внутренний ключ / inner key
    (c, o) => new { c.Name, o.OrderId, o.Amount });

Console.WriteLine("Join (inner):");
foreach (var row in innerJoin)
    Console.WriteLine($"  {row.Name} — заказ {row.OrderId}: {row.Amount}");
// Алиса — 101, Алиса — 102, Виктор — 103. Бориса нет / Boris is absent

// 2) GroupJoin — соединение с группировкой / grouped join
// Сохраняет ВСЕХ клиентов, включая тех, у кого нет заказов
// Keeps ALL customers, including those without orders
var grouped = customers.GroupJoin(
    orders,
    c => c.Id,
    o => o.CustomerId,
    (c, ordGroup) => new
    {
        Customer = c.Name,
        Orders = ordGroup.ToList(),
        Total = ordGroup.Sum(o => o.Amount)
    });

Console.WriteLine("GroupJoin (grouped):");
foreach (var g in grouped)
    Console.WriteLine($"  {g.Customer}: заказов={g.Orders.Count}, сумма={g.Total}");
// Борис: заказов=0, сумма=0 — остался в результате / Boris stays in result

// 3) GroupJoin + SelectMany + DefaultIfEmpty = плоский LEFT JOIN / flat LEFT JOIN
var leftJoin = customers
    .GroupJoin(orders, c => c.Id, o => o.CustomerId, (c, ordGroup) => new { c, ordGroup })
    .SelectMany(x => x.ordGroup.DefaultIfEmpty(),
                 (x, o) => new
                 {
                     x.c.Name,
                     OrderId = o?.OrderId,
                     Amount = o?.Amount
                 });

Console.WriteLine("Left join (flat):");
foreach (var row in leftJoin)
    Console.WriteLine($"  {row.Name} — заказ {row.OrderId?.ToString() ?? "—"}: {row.Amount?.ToString() ?? "—"}");

// 4) Составной ключ / composite key
var byCustomerAndYear = customers.Join(
    orders,
    c => new { Id = c.Id, Year = 2024 },          // внешний составной ключ
    o => new { Id = o.CustomerId, o.Year },        // внутренний составной ключ
    (c, o) => new { c.Name, o.OrderId, o.Year })
    .Where(x => x.Year == 2024);

Console.WriteLine("Composite key (2024 only):");
foreach (var row in byCustomerAndYear)
    Console.WriteLine($"  {row.Name} — заказ {row.OrderId} ({row.Year})");

// 5) Zip — попарное объединение по позиции / positional pairing
var subjects = new[] { "Математика", "Физика", "Программирование" }; // Math, Physics, Programming
var grades = new[] { 5, 4, 5 };

var reportCard = subjects.Zip(grades, (subject, grade) => new { subject, grade });

Console.WriteLine("Zip:");
foreach (var item in reportCard)
    Console.WriteLine($"  {item.subject}: {item.grade}");
// Если длины не совпадают — лишние элементы отбрасываются / Extra items are dropped
```

#### Best Practices

- Предпочитайте `Join` вложенным циклам с `Where`: реализация на хеш-таблице даёт `O(n+m)` вместо `O(n·m)`.
- Используйте `GroupJoin` для иерархий «мастер–детали», а не для плоских списков — так результат читается естественнее.
- Для плоского `LEFT JOIN` комбинируйте `GroupJoin` + `SelectMany` + `DefaultIfEmpty`.
- Соединяйте как можно раньше на стороне базы (EF Core), а в памяти только проектируйте.
- Делайте составные ключи через анонимные типы — они гарантируют структурное равенство.
- Используйте `Zip` только когда индексы двух последовательностей семантически совпадают; иначе это источник скрытых багов.

- Prefer `Join` over nested `Where` loops: the hash-based implementation gives `O(n+m)` instead of `O(n·m)`.
- Use `GroupJoin` for master–detail hierarchies, not for flat lists — the result reads more naturally.
- For a flat `LEFT JOIN`, combine `GroupJoin` + `SelectMany` + `DefaultIfEmpty`.
- Join as early as possible on the database side (EF Core); only project in memory afterward.
- Build composite keys with anonymous types — they guarantee structural equality.
- Use `Zip` only when the indices of two sequences are semantically aligned; otherwise it is a source of hidden bugs.

#### Частые ошибки / Common Mistakes

- Перепутан порядок аргументов `Join` (внешний/внутренний селекторы) → всегда следуйте сигнатуре `(outer, inner, outerKey, innerKey, result)`; проверяйте, какой селектор к какой последовательности относится.
- Ожидание, что `Join` вернёт элементы без совпадений (как `LEFT JOIN`) → для этого используйте `GroupJoin` или плоский `LEFT JOIN` через `DefaultIfEmpty`.
- Использование ссылочных типов в качестве ключа без переопределённого `Equals`/`GetHashCode` → ключи не находятся, соединение пустое; используйте `record` или анонимные типы.
- Соединение в цикле → каждый вызов строит новый словарь, `O(n²)` по памяти и времени; вынесите соединение за цикл или используйте `Lookup`.
- `Zip` на последовательностях разной длины без проверки → молча теряются данные; явно проверяйте `Count` или используйте `Zip` с третьим параметром-заполнителем в .NET 9+.
- Проекция с тяжёлыми вычислениями внутри `Join` → вычисления выполняются для каждой пары; выносите их позже через отдельный `Select`.

- Mixing up the argument order of `Join` (outer/inner selectors) → always follow `(outer, inner, outerKey, innerKey, result)`; double-check which selector belongs to which sequence.
- Expecting `Join` to return unmatched elements (like `LEFT JOIN`) → use `GroupJoin` or a flat `LEFT JOIN` via `DefaultIfEmpty` instead.
- Using reference types as keys without overridden `Equals`/`GetHashCode` → keys never match, the join is empty; use `record` or anonymous types.
- Joining inside a loop → every call builds a new dictionary, `O(n²)` in time and memory; hoist the join out of the loop or precompute a `Lookup`.
- `Zip` on sequences of different lengths without a check → data is silently dropped; explicitly verify `Count` or use the three-argument `Zip` filler in .NET 9+.
- Expensive computation inside the `Join` projector → it runs for every matched pair; move it to a later `Select`.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я понимаю разницу между `Join` (inner) и `GroupJoin` (grouped/left).
- [ ] Я могу превратить `GroupJoin` в плоский `LEFT JOIN` через `SelectMany` + `DefaultIfEmpty`.
- [ ] Я знаю, что `Zip` соединяет по индексу, а не по ключу.
- [ ] Я могу построить составной ключ через анонимный тип.
- [ ] Я знаю сложность `Join` (`O(n+m)`) и почему не стоит соединять в цикле.
- [ ] Я проверяю тип ключа на корректные `Equals`/`GetHashCode` (или использую `record`).
- [ ] Я переношу соединение на сторону базы, когда работаю с EF Core.

- [ ] I understand the difference between `Join` (inner) and `GroupJoin` (grouped/left).
- [ ] I can turn `GroupJoin` into a flat `LEFT JOIN` using `SelectMany` + `DefaultIfEmpty`.
- [ ] I know that `Zip` pairs by index, not by key.
- [ ] I can build a composite key with an anonymous type.
- [ ] I know the complexity of `Join` (`O(n+m)`) and why not to join inside a loop.
- [ ] I verify the key type has correct `Equals`/`GetHashCode` (or use `record`).
- [ ] I push the join to the database side when working with EF Core.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/linq/standard-query-operators/join-operations]

---

[⬆ К модулю M08](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
