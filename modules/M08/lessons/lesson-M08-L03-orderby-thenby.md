[⬆ К модулю M08](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M08-L03: OrderBy/ThenBy, Reverse / OrderBy/ThenBy, Reverse

**Модуль / Module:** M08
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Сортировка — одна из самых частых операций над данными. В LINQ для неё служат методы `OrderBy`, `OrderByDescending`, `ThenBy`, `ThenByDescending` и `Reverse`. Все они работают как операторы запроса и возвращают новую последовательность `IOrderedEnumerable<T>`, не изменяя исходную коллекцию.

`OrderBy` принимает ключ-селектор — функцию, которая для каждого элемента возвращает значение, по которому будет идти сортировка. По умолчанию порядок возрастающий (ascending). Например, `.OrderBy(x => x.Age)` расставит элементы от младшего к старшему. `OrderByDescending` делает то же самое, но по убыванию.

Аналогия: представьте стопку экзаменационных работ. `OrderBy` раскладывает их по одной — сначала все с оценкой 2, потом 3, 4, 5. Если внутри каждой оценки нужно упорядочить работы по фамилии, добавляется `ThenBy`. Получится основной критерий (оценка) и вторичный (фамилия). `ThenBy` можно вызывать несколько раз, создавая сколько угодно уровней: оценка → фамилия → имя.

Важно понимать: `ThenBy` применяется ТОЛЬКО к результату `OrderBy`/`OrderByDescending`, то есть к `IOrderedEnumerable<T>`. Вызов `.ThenBy()` сразу на `IEnumerable<T>` не скомпилируется. Это логично: вторичный критерий не имеет смысла без первичного.

`Reverse` просто переворачивает порядок элементов — последний становится первым, и наоборот. Это не сортировка по значению, а инверсия текущей последовательности. `Reverse` работает за O(n) и не требует сравнения элементов.

Стабильность сортировки — ключевое свойство LINQ. `OrderBy` стабилен: если два элемента имеют равные ключи, они сохраняют свой исходный относительный порядок. Благодаря этому можно сначала отсортировать по вторичному ключу, потом по первичному — и вторичный порядок «выживет» для равных первых ключей. Именно так работает цепочка `OrderBy().ThenBy()`: внутренне это два стабильных прохода сортировки, и компилятор запоминает цепочку критериев.

`IComparer<T>` позволяет задать кастомную логику сравнения. Например, нужно сравнивать строки без учёта регистра или по длине. Вместо собственного компаратора можно использовать `StringComparer.OrdinalIgnoreCase` — готовый `IComparer<string>`. `OrderBy` и `ThenBy` имеют перегрузки, принимающие `IComparer<TKey>` (обратите внимание: компаратор для типа ключа, а не элемента).

Производительность: `OrderBy` использует отсроченное выполнение (deferred execution) — реальная сортировка происходит только при перечислении результата. Внутри применяется быстрая сортировка со сложностью O(n log n). Каждый вызов `ThenBy` добавляет ещё один стабильный проход, поэтому при большом числе критериев иногда стоит рассмотреть составной ключ (например, анонимный объект или кортеж) в одном `OrderBy`.

На практике комбинируйте: основной критерий через `OrderBy`, уточняющие через `ThenBy`, убывание через `OrderByDescending`, инверсию через `Reverse`, кастомные правила через `IComparer<T>`.

#### Theory (EN)

Sorting is one of the most common operations on data. In LINQ it is handled by `OrderBy`, `OrderByDescending`, `ThenBy`, `ThenByDescending`, and `Reverse`. They all act as query operators and return a new `IOrderedEnumerable<T>` sequence, leaving the original collection untouched.

`OrderBy` takes a key selector — a function that returns, for each element, the value used as the sort key. By default the order is ascending. For example, `.OrderBy(x => x.Age)` arranges elements from youngest to oldest. `OrderByDescending` does the same but in descending order.

Analogy: imagine a stack of exam papers. `OrderBy` lays them out by grade — first all the papers graded 2, then 3, 4, 5. If you also want them ordered by last name within each grade, you add `ThenBy`. The result is a primary criterion (grade) and a secondary one (surname). `ThenBy` can be chained multiple times to create as many levels as you need: grade → surname → first name.

Important: `ThenBy` applies ONLY to the result of `OrderBy` or `OrderByDescending` — that is, to an `IOrderedEnumerable<T>`. Calling `.ThenBy()` directly on an `IEnumerable<T>` will not compile. This makes sense: a secondary criterion is meaningless without a primary one.

`Reverse` simply flips the order of elements — the last becomes first and vice versa. It is not a value-based sort; it is an inversion of the current sequence. `Reverse` runs in O(n) and requires no comparison between elements.

Sort stability is a key LINQ property. `OrderBy` is stable: when two elements have equal keys, they keep their original relative order. Because of this, you can first sort by a secondary key and then by a primary one, and the secondary order survives for equal primary keys. That is exactly how the `OrderBy().ThenBy()` chain works internally: two stable sorting passes, with the compiler remembering the chain of criteria.

`IComparer<T>` lets you supply custom comparison logic. For example, you may need case-insensitive string comparison or comparison by length. Instead of writing your own comparer you can use `StringComparer.OrdinalIgnoreCase` — a ready-made `IComparer<string>`. `OrderBy` and `ThenBy` have overloads that accept `IComparer<TKey>` (note: a comparer for the key type, not the element type).

Performance: `OrderBy` uses deferred execution — the actual sort happens only when the result is enumerated. Internally it uses quicksort with O(n log n) complexity. Each `ThenBy` call adds another stable pass, so when you have many criteria you may consider a composite key (for example, an anonymous object or tuple) in a single `OrderBy`.

In practice: combine a primary criterion with `OrderBy`, refine with `ThenBy`, use `OrderByDescending` for descending order, `Reverse` for inversion, and `IComparer<T>` for custom rules.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8+ — OrderBy, ThenBy, OrderByDescending, Reverse, IComparer<T>
// Двуязычные комментарии RU+EN / Bilingual comments

using System;
using System.Collections.Generic;
using System.Linq;

// Модель студента / Student model
public record Student(string Name, int Grade, int Age);

public static class SortingDemo
{
    public static void Run()
    {
        var students = new List<Student>
        {
            new("Иванов",   5, 20), // Ivanov
            new("Петров",   4, 22), // Petrov
            new("Сидоров",  5, 19), // Sidorov
            new("Алексеев", 4, 22), // Alekseev
            new("Борисов",  3, 21), // Borisov
        };

        // 1. OrderBy — сортировка по оценке по возрастанию
        //    OrderBy — sort by grade ascending
        var byGrade = students.OrderBy(s => s.Grade);
        Console.WriteLine("OrderBy(Grade):");
        foreach (var s in byGrade)
            Console.WriteLine($"  {s.Name}: grade={s.Grade}, age={s.Age}");

        // 2. OrderBy + ThenBy — основной критерий оценка, вторичный возраст
        //    Primary by grade, secondary by age
        var byGradeThenAge = students
            .OrderBy(s => s.Grade)
            .ThenBy(s => s.Age);
        Console.WriteLine("\nOrderBy(Grade).ThenBy(Age):");
        foreach (var s in byGradeThenAge)
            Console.WriteLine($"  {s.Name}: grade={s.Grade}, age={s.Age}");

        // 3. OrderByDescending — по оценке по убыванию
        //    Descending by grade
        var byGradeDesc = students.OrderByDescending(s => s.Grade);
        Console.WriteLine("\nOrderByDescending(Grade):");
        foreach (var s in byGradeDesc)
            Console.WriteLine($"  {s.Name}: grade={s.Grade}");

        // 4. Стабильность: равные ключи сохраняют исходный порядок.
        //    Stability: equal keys keep their original order.
        //    Иванов(5) идёт раньше Сидорова(5) — как в исходном списке.
        //    Ivanov(5) appears before Sidorov(5) — same as in the source list.

        // 5. Reverse — инверсия порядка, а не сортировка
        //    Reverse — invert order, not a sort
        var reversed = students.AsEnumerable().Reverse();
        Console.WriteLine("\nReverse:");
        foreach (var s in reversed)
            Console.WriteLine($"  {s.Name}");

        // 6. IComparer<T> через готовый StringComparer — без учёта регистра
        //    Built-in StringComparer — case-insensitive
        var byNameCI = students.OrderBy(s => s.Name, StringComparer.OrdinalIgnoreCase);
        Console.WriteLine("\nOrderBy(Name, OrdinalIgnoreCase):");
        foreach (var s in byNameCI)
            Console.WriteLine($"  {s.Name}");

        // 7. Свой компаратор: сначала по длине имени, затем по алфавиту
        //    Custom comparer: by name length, then alphabetically
        var byLengthThenName = students.OrderBy(s => s.Name, new ByLengthThenNameComparer());
        Console.WriteLine("\nOrderBy with custom IComparer<string>:");
        foreach (var s in byLengthThenName)
            Console.WriteLine($"  {s.Name} (len={s.Name.Length})");

        // 8. Цепочка из трёх критериев: оценка → возраст → имя
        //    Three-level chain: grade → age → name
        var multi = students
            .OrderBy(s => s.Grade)
            .ThenBy(s => s.Age)
            .ThenBy(s => s.Name, StringComparer.OrdinalIgnoreCase);
        Console.WriteLine("\nOrderBy(Grade).ThenBy(Age).ThenBy(Name):");
        foreach (var s in multi)
            Console.WriteLine($"  {s.Name}: grade={s.Grade}, age={s.Age}");
    }
}

// Пользовательский компаратор строк / Custom string comparer
public sealed class ByLengthThenNameComparer : IComparer<string>
{
    public int Compare(string? x, string? y)
    {
        // Сначала по длине / First by length
        int lenX = x?.Length ?? 0;
        int lenY = y?.Length ?? 0;
        int lenCmp = lenX.CompareTo(lenY);
        if (lenCmp != 0) return lenCmp;

        // Потом по алфавиту без учёта регистра / Then alphabetically, case-insensitive
        return string.Compare(x, y, StringComparison.OrdinalIgnoreCase);
    }
}

// Запуск / Entry point
SortingDemo.Run();
```

#### Best Practices

- Используйте `ThenBy` для вторичных критериев вместо нескольких `OrderBy`: каждый новый `OrderBy` отменяет предыдущую сортировку, а `ThenBy` её уточняет.
- Предпочитайте готовые компараторы (`StringComparer.OrdinalIgnoreCase`, `Comparer<T>.Default`) ручной реализации — меньше ошибок и краевых случаев.
- Помните, что сортировка отложенная: результат материализуется только при перечислении; если нужны стабильные данные — вызовите `.ToList()` или `.ToArray()`.
- Для большого числа критериев рассмотрите составной ключ в одном `OrderBy` (кортеж/анонимный объект), чтобы избежать лишних проходов.

- Use `ThenBy` for secondary criteria instead of multiple `OrderBy` calls: each new `OrderBy` discards the previous sort, while `ThenBy` refines it.
- Prefer built-in comparers (`StringComparer.OrdinalIgnoreCase`, `Comparer<T>.Default`) over hand-rolled ones — fewer errors and edge cases.
- Remember sorting is deferred: the result materializes only on enumeration; for stable data call `.ToList()` or `.ToArray()`.
- With many criteria, consider a composite key in a single `OrderBy` (tuple/anonymous object) to avoid extra passes.

#### Частые ошибки / Common Mistakes

- Вызов `ThenBy` напрямую на `IEnumerable<T>` без `OrderBy` → не компилируется; всегда начинайте с `OrderBy`/`OrderByDescending`.
- Цепочка из нескольких `OrderBy` в ожидании составной сортировки → учитывается только последний `OrderBy`; используйте `ThenBy` для вторичных ключей.
- Уверенность, что `Reverse` сортирует по значению → `Reverse` только инвертирует текущий порядок без сравнения элементов.
- Передача компаратора для типа элемента вместо типа ключа → перегрузка ожидает `IComparer<TKey>`, а не `IComparer<TElement>`.
- Изменение исходной коллекции после `OrderBy` и ожидание, что отсортированный результат обновится → результат вычисляется при перечислении; материализуйте его, если нужен снимок.

- Calling `ThenBy` directly on `IEnumerable<T>` without `OrderBy` → won't compile; always start with `OrderBy`/`OrderByDescending`.
- Chaining several `OrderBy` expecting a combined sort → only the last `OrderBy` is applied; use `ThenBy` for secondary keys.
- Assuming `Reverse` sorts by value → `Reverse` only inverts the current order without comparing elements.
- Passing a comparer for the element type instead of the key type → the overload expects `IComparer<TKey>`, not `IComparer<TElement>`.
- Mutating the source collection after `OrderBy` and expecting the sorted result to update → the result is evaluated on enumeration; materialize it if you need a snapshot.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я понимаю разницу между `OrderBy` (первичный ключ) и `ThenBy` (вторичный).
- [ ] Я знаю, что LINQ-сортировка стабильна: равные ключи сохраняют исходный порядок.
- [ ] Я могу применить `IComparer<T>` (или `StringComparer`) для кастомной логики сравнения.
- [ ] Я понимаю, что `Reverse` — это инверсия порядка, а не сортировка по значению.
- [ ] Я помню, что сортировка отложена и материализуется при перечислении.
- [ ] Я отличаю `OrderBy` (возрастание) от `OrderByDescending` (убывание).

- [ ] I understand the difference between `OrderBy` (primary key) and `ThenBy` (secondary).
- [ ] I know LINQ sorting is stable: equal keys keep their original order.
- [ ] I can apply `IComparer<T>` (or `StringComparer`) for custom comparison logic.
- [ ] I understand `Reverse` is an order inversion, not a value-based sort.
- [ ] I remember sorting is deferred and materializes on enumeration.
- [ ] I distinguish `OrderBy` (ascending) from `OrderByDescending` (descending).

#### Ресурсы / Resources

- [Microsoft Learn — Enumerable.OrderBy](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.orderby)
- [Microsoft Learn — Enumerable.ThenBy](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.thenby)
- [Microsoft Learn — Enumerable.Reverse](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.reverse)
- [Microsoft Learn — IComparer<T>](https://learn.microsoft.com/dotnet/api/system.collections.generic.icomparer-1)

---

[⬆ К модулю M08](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
