[⬆ К модулю M06](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M06-L06: IEnumerable<T>/IEnumerator<T>, итерация / IEnumerable<T>/IEnumerator<T>, iteration

**Модуль / Module:** M06
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

`IEnumerable<T>` — это самый фундаментальный интерфейс коллекций в .NET. Если коротко, он отвечает на один-единственный вопрос: «Можно ли по тебе пройтись перечислителем?». Любая коллекция, поддерживающая `foreach`, реализует именно его. Без `IEnumerable<T>` не существовало бы ни LINQ, ни `foreach`, ни большинства современных абстракций поверх данных.

Под капотом `IEnumerable<T>` имеет всего один метод — `GetEnumerator()`, который возвращает `IEnumerator<T>`. Аналогия: представьте библиотеку. `IEnumerable<T>` — это библиотекарь, который выдаёт вам карточку-указатель. `IEnumerator<T>` — это сама карточка, с помощью которой вы переходите от одной книги к следующей, читаете текущую и в конце возвращаете карточку обратно. Сам библиотекарь книги не листает — он лишь даёт инструмент.

`IEnumerator<T>` предоставляет три ключевых члена:
- `bool MoveNext()` — сдвинуть указатель на следующий элемент; возвращает `false`, когда элементы закончились;
- `T Current` — текущий элемент (доступен только после первого успешного `MoveNext()`);
- `void Reset()` — вернуть указатель в начало (почти не используется и часто бросает `NotSupportedException`).

Как работает `foreach` под капотом? Конструкция `foreach (var x in collection)` разворачивается компилятором примерно в такой код: вызывается `GetEnumerator()`, затем в цикле вызывается `MoveNext()`, и если он вернул `true`, читается `Current`. В конце, поскольку `IEnumerator` реализует `IDisposable`, вызывается `Dispose()`. Это важно: даже если вы написали короткий `foreach`, компилятор заботится о корректном освобождении ресурсов перечислителя.

Вторая важнейшая концепция — **отложенность (deferred execution)**. Методы LINQ вроде `Where`, `Select`, `OrderBy` не выполняются немедленно. Они лишь строят «рецепт» вычисления: обёртывают один `IEnumerable<T>` другим, не трогая исходные данные. Реальная работа начинается только в момент перечисления — то есть при первом `foreach`, `ToList()`, `ToArray()` или `Count()`. Это значит, что один и тот же запрос LINQ можно выполнять многократно, и каждый раз он заново пройдёт по источнику. Побочный эффект: если источник меняется между запусками, результат тоже меняется; если источник тяжёлый (например, чтение из БД), каждое перечисление повторяет работу.

Жадные операторы (`ToList`, `ToArray`, `ToDictionary`, `Count`, `First`, `Any`) форсируют немедленное вычисление и «замораживают» результат в памяти. Между отложенными и жадными операторами нужно чётко различать: первые описывают конвейер, вторые его запускают.

Третья концепция — **множественное перечисление**. Каждый вызов `GetEnumerator()` создаёт новый проход. Поэтому если вы дважды пишете `foreach` по одному и тому же `IEnumerable<T>` или несколько раз вызываете LINQ-методы, источник перечитывается заново. Для дешёвых списков в памяти это приемлемо, для дорогих источников — нет: лучше один раз вызвать `ToList()` и работать с материализованным списком.

Наконец, `yield return` — синтаксис для написания собственных отложенных последовательностей. Метод с `yield return` компилируется в скрытый класс-конечный автомат, реализующий `IEnumerable<T>`/`IEnumerator<T>`. Значения вычисляются по одному, по мере запроса, что позволяет работать с потенциально бесконечными последовательностями (например, `Enumerable.Range` или генератор чисел Фибоначчи) без переполнения памяти.

#### Theory (EN)

`IEnumerable<T>` is the most foundational collection interface in .NET. In short, it answers a single question: "Can I get an enumerator to walk through you?". Any collection that supports `foreach` implements exactly this interface. Without `IEnumerable<T>` there would be no LINQ, no `foreach`, and most modern data abstractions would simply not exist.

Under the hood `IEnumerable<T>` exposes exactly one method — `GetEnumerator()` — which returns an `IEnumerator<T>`. Analogy: imagine a library. `IEnumerable<T>` is the librarian who hands you a reading pointer card. `IEnumerator<T>` is the card itself, which you use to move from one book to the next, read the current one, and eventually return the card. The librarian does not flip through the books — they merely provide the tool.

`IEnumerator<T>` exposes three core members:
- `bool MoveNext()` — advance the pointer to the next element; returns `false` when there is nothing left;
- `T Current` — the current element (valid only after a successful `MoveNext()`);
- `void Reset()` — rewind the pointer to the beginning (rarely used and often throws `NotSupportedException`).

How does `foreach` work under the hood? The construct `foreach (var x in collection)` is expanded by the compiler into roughly the following: it calls `GetEnumerator()`, then loops calling `MoveNext()`; if that returns `true`, it reads `Current`. At the end, because `IEnumerator` implements `IDisposable`, it calls `Dispose()`. This matters: even a tiny `foreach` you wrote gets proper resource cleanup from the compiler.

The second key concept is **deferred execution**. LINQ methods like `Where`, `Select`, `OrderBy` do not run immediately. They only build a "recipe" for computation: they wrap one `IEnumerable<T>` with another without touching the source data. Real work starts only when the sequence is enumerated — that is, on the first `foreach`, `ToList()`, `ToArray()`, or `Count()`. This means the same LINQ query can be executed many times, and each run re-walks the source. Side effect: if the source changes between runs, the result changes too; if the source is expensive (say, a database query), every enumeration repeats the cost.

Eager operators (`ToList`, `ToArray`, `ToDictionary`, `Count`, `First`, `Any`) force immediate evaluation and freeze the result in memory. You must clearly distinguish deferred from eager operators: the former describe a pipeline, the latter trigger it.

The third concept is **multiple enumeration**. Each call to `GetEnumerator()` starts a new pass. So if you write `foreach` twice over the same `IEnumerable<T>` or call several LINQ methods, the source is re-read each time. For cheap in-memory lists this is acceptable; for expensive sources it is not — better to call `ToList()` once and work with the materialized list.

Finally, `yield return` is the syntax for writing your own deferred sequences. A method with `yield return` is compiled into a hidden state-machine class implementing `IEnumerable<T>`/`IEnumerator<T>`. Values are produced one at a time, on demand, which lets you work with potentially infinite sequences (such as `Enumerable.Range` or a Fibonacci generator) without exhausting memory.

#### Пример кода / Code Example

```csharp
using System;
using System.Collections;
using System.Collections.Generic;
using System.Linq;

// Demonstrates IEnumerable<T> / IEnumerator<T> and deferred execution.
// Демонстрирует IEnumerable<T> / IEnumerator<T> и отложенное выполнение.

namespace M06L06;

public static class IterationDemo
{
    // A custom lazy sequence built with yield return.
    // Собственная отложенная последовательность через yield return.
    public static IEnumerable<int> Fibonacci()
    {
        // RU: Бесконечный генератор; значения считаются только по запросу.
        // EN: Infinite generator; values are computed only on demand.
        int prev = 0, curr = 1;
        while (true)
        {
            yield return curr;
            (prev, curr) = (curr, prev + curr);
        }
    }

    public static void Run()
    {
        int[] numbers = { 1, 2, 3, 4, 5 };

        // RU: foreach разворачивается в GetEnumerator + MoveNext + Current + Dispose.
        // EN: foreach expands into GetEnumerator + MoveNext + Current + Dispose.
        foreach (int n in numbers)
        {
            Console.WriteLine(n);
        }

        // RU: Ручная итерация — то же самое, что делает foreach.
        // EN: Manual iteration — exactly what foreach does internally.
        using IEnumerator<int> e = numbers.AsEnumerable().GetEnumerator();
        while (e.MoveNext())
        {
            Console.WriteLine(e.Current);
        }

        // RU: LINQ строит конвейер, но НЕ выполняет его сейчас.
        // EN: LINQ builds a pipeline but does NOT execute it now.
        IEnumerable<int> query = numbers
            .Where(x => x % 2 == 1)
            .Select(x => x * 10);

        // RU: Материализация запускает вычисление ровно один раз.
        // EN: Materialization triggers evaluation exactly once.
        List<int> materialized = query.ToList();
        Console.WriteLine(string.Join(", ", materialized)); // 10, 30, 50

        // RU: Берём первые 5 чисел Фибоначчи из бесконечного генератора.
        // EN: Take the first 5 Fibonacci numbers from an infinite generator.
        List<int> fibs = Fibonacci().Take(5).ToList();
        Console.WriteLine(string.Join(", ", fibs)); // 1, 1, 2, 3, 5
    }
}
```

#### Best Practices
- Материализуйте дорогие источники (`IQueryable`, потоки, БД) через `ToList()`/`ToArray()` один раз и переиспользуйте результат, чтобы избежать повторных вычислений.
- Предпочитайте `IEnumerable<T>` как тип параметра, когда достаточно перечисления; это расширяет совместимость с LINQ и любыми коллекциями.
- Реализуйте `IDisposable` в собственных перечислителях, если они держат неуправляемые ресурсы — `foreach` вызовет `Dispose()`.
- Используйте `yield return` для ленивых последовательностей вместо ручной реализации `IEnumerator<T>`, когда это уместно.
- Prefer `IEnumerable<T>` as a parameter type when enumeration is all you need; it maximizes LINQ and collection compatibility.
- Materialize expensive sources (`IQueryable`, streams, DB) via `ToList()`/`ToArray()` once and reuse the result to avoid repeated work.
- Implement `IDisposable` in custom enumerators that hold unmanaged resources — `foreach` will call `Dispose()`.
- Use `yield return` for lazy sequences instead of hand-writing `IEnumerator<T>` where practical.

#### Частые ошибки / Common Mistakes
- Многократное перечисление одного `IEnumerable<T>` → материализуйте один раз через `ToList()`, если источник дорогой (RU).
- Изменение коллекции во время `foreach` → используйте `ToList()` перед модификацией или отдельную коллекцию для добавлений (RU).
- Ожидание немедленного выполнения LINQ → помните, что `Where`/`Select`/`OrderBy` отложены; вызывайте жадный оператор, когда нужен результат (RU).
- Возврат `List<T>` как `IEnumerable<T>` и последующее приведение к `List<T>` для изменения → возвращайте копию или документируйте контракт (RU).
- Зависимость от порядка `IEnumerable<T>` без гарантии → используйте `IReadOnlyList<T>`/массив, если порядок критичен (RU).
- Enumerating the same `IEnumerable<T>` multiple times → materialize once with `ToList()` when the source is expensive (EN).
- Modifying a collection during `foreach` → call `ToList()` before mutation or use a separate collection for additions (EN).
- Expecting LINQ to execute immediately → remember `Where`/`Select`/`OrderBy` are deferred; call an eager operator when you need the result (EN).
- Returning a `List<T>` typed as `IEnumerable<T>` and later casting back to `List<T>` to mutate → return a copy or document the contract (EN).
- Relying on ordering of `IEnumerable<T>` with no guarantee → use `IReadOnlyList<T>` or an array when order matters (EN).

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я могу объяснить, чем `IEnumerable<T>` отличается от `IEnumerator<T>` (RU).
- [ ] Я понимаю, что `foreach` вызывает `GetEnumerator`, `MoveNext`, `Current`, `Dispose` (RU).
- [ ] Я различаю отложенные и жадные LINQ-операторы (RU).
- [ ] Я знаю, почему множественное перечисление может быть дорогим (RU).
- [ ] Я умею писать ленивые последовательности через `yield return` (RU).
- [ ] I can explain the difference between `IEnumerable<T>` and `IEnumerator<T>` (EN).
- [ ] I understand that `foreach` calls `GetEnumerator`, `MoveNext`, `Current`, `Dispose` (EN).
- [ ] I distinguish deferred from eager LINQ operators (EN).
- [ ] I know why multiple enumeration can be expensive (EN).
- [ ] I can write lazy sequences using `yield return` (EN).

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.collections.generic.ienumerable-1]

---

[⬆ К модулю M06](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
