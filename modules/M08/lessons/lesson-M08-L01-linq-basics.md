[⬆ К модулю M08](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M08-L01: Основы LINQ, method vs query syntax / LINQ basics, method vs query syntax

**Модуль / Module:** M08
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

**LINQ (Language Integrated Query)** — это встроенный в C# язык запросов к данным. Представьте, что вы работаете с базой данных: обычно вы пишете SQL-строки, которые компилятор не проверяет. LINQ превращает запрос в полноценную часть языка C#: компилятор проверяет типы, IntelliSense подсказывает имена полей, а опечатку вы ловите на этапе сборки, а не в рантайме.

В основе LINQ лежат два механизма. Первый — **методы расширения (extension methods)**. Это статические методы, которые «прилипают» к существующим типам и вызываются так, будто они были там всегда. Например, `Where`, `Select`, `OrderBy` определены в классе `Enumerable`, но вызываются на любом `IEnumerable<T>`. Второй механизм — **`IEnumerable<T>`** — контракт «перечислимой последовательности». Всё, что можно перебрать в `foreach` (массивы, списки, множества), реализует этот интерфейс, а значит, к нему применимы операции LINQ.

Синтаксис бывает двух видов. **Method syntax (синтаксис методов)** — это цепочка вызовов: `numbers.Where(n => n > 5).Select(n => n * 2).OrderBy(n => n)`. **Query syntax (синтаксис запросов)** — это SQL-подобная запись: `from n in numbers where n > 5 select n * 2`. Под капотом компилятор переводит query syntax в те же вызовы методов, поэтому оба варианта эквивалентны по производительности и результату. Выбор — вопрос читаемости: для простых цепочек удобнее методы, для сложных соединений (join, group) — запросы.

Ключевое свойство LINQ — **отложенность выполнения (deferred execution)**. Запрос, построенный через `Where` или `Select`, не выполняется сразу. Он лишь описывает, *что* нужно сделать. Реальное вычисление происходит в момент перебора: `foreach`, `ToList()`, `ToArray()`, `First()`, `Count()`. Это значит, что если исходная коллекция изменится между построением запроса и его перебором — результат будет отражать новое состояние. Плюс отложенности: можно строить запросы, не тратя ресурсы впустую. Минус: неожиданные многократные вычисления. Чтобы «заморозить» результат, материализуйте его через `ToList()` или `ToArray()`.

Аналогия: LINQ-запрос — это рецепт блюда. Пока вы не открыли холодильник и не начали готовить (перебрать результат), ничего не происходит. `ToList()` — это «приготовить и положить в контейнер»: результат зафиксирован.

Некоторые операторы выполняются **немедленно (eager)**: `Count`, `First`, `Any`, `Max`, `ToList`. Они вынуждены перебрать последовательность, чтобы вернуть скаляр или коллекцию. Запомните: всё, что возвращает не `IEnumerable<T>`, а конкретное значение или список, выполняется сразу.

#### Theory (EN)

**LINQ (Language Integrated Query)** is a query language built directly into C#. When you work with a database you normally write SQL strings that the compiler cannot check. LINQ turns the query into a first-class part of C#: the compiler verifies types, IntelliSense suggests field names, and a typo is caught at build time instead of at runtime.

Two mechanisms power LINQ. The first is **extension methods** — static methods that "attach" to existing types and are called as if they had always been there. For example, `Where`, `Select`, and `OrderBy` are defined in the `Enumerable` class, yet they can be called on any `IEnumerable<T>`. The second is **`IEnumerable<T>`** — the contract of an "enumerable sequence". Anything you can iterate with `foreach` (arrays, lists, sets) implements this interface, and therefore supports LINQ operations.

There are two syntax flavors. **Method syntax** is a chain of calls: `numbers.Where(n => n > 5).Select(n => n * 2).OrderBy(n => n)`. **Query syntax** is a SQL-like form: `from n in numbers where n > 5 select n * 2`. Under the hood the compiler translates query syntax into the same method calls, so both forms are equivalent in performance and result. The choice is about readability: for simple chains methods are cleaner; for complex joins or groupings the query form reads better.

A key property of LINQ is **deferred execution**. A query built with `Where` or `Select` does not run immediately — it only describes *what* to do. The actual evaluation happens when you iterate the result: `foreach`, `ToList()`, `ToArray()`, `First()`, `Count()`. This means if the source collection changes between building the query and enumerating it, the result reflects the new state. The upside: you can compose queries without wasting resources. The downside: unexpected repeated evaluations. To "freeze" a result, materialize it with `ToList()` or `ToArray()`.

Analogy: a LINQ query is a recipe. Until you open the fridge and start cooking (enumerate the result), nothing happens. `ToList()` is "cook and put it in a container": the result is fixed.

Some operators execute **eagerly**: `Count`, `First`, `Any`, `Max`, `ToList`. They must enumerate the sequence to return a scalar or a materialized list. Rule of thumb: anything that does not return `IEnumerable<T>` but returns a concrete value or list runs immediately.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8+ — рабочий пример LINQ: method vs query syntax / working LINQ example
using System;
using System.Collections.Generic;
using System.Linq;

var numbers = new List<int> { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

// Method syntax / Синтаксис методов
var methodResult = numbers
    .Where(n => n % 2 == 0)          // только чётные / only even
    .Select(n => n * n)              // возвести в квадрат / square each
    .OrderByDescending(n => n)       // по убыванию / descending
    .ToList();                       // материализовать / materialize (eager!)

// Query syntax / Синтаксис запросов — эквивалентно / equivalent
var queryResult = (from n in numbers
                   where n % 2 == 0
                   orderby n descending
                   select n * n).ToList();

// Отложенное выполнение / Deferred execution — запрос пока НЕ выполнен
IEnumerable<int> deferred = numbers.Where(n => n > 5);
numbers.Add(11);                     // меняем источник / mutate source
Console.WriteLine(deferred.Count()); // 6: {6,7,8,9,10,11} — учитывает 11!

// Мгновенные операторы / Eager operators
bool hasEven = numbers.Any(n => n % 2 == 0);   // true, выполнится сразу
int max = numbers.Max();                       // 11, выполнится сразу
int firstEven = numbers.First(n => n % 2 == 0); // 2, выполнится сразу

// Демонстрация результата / Show results
Console.WriteLine($"method: {string.Join(", ", methodResult)}"); // 100, 64, 36, 16, 4
Console.WriteLine($"query : {string.Join(", ", queryResult)}");  // 100, 64, 36, 16, 4
Console.WriteLine($"hasEven={hasEven}, max={max}, firstEven={firstEven}");
```

#### Best Practices

- Предпочитайте method syntax для простых цепочек — он компактнее и дружелюбнее к LINQ-операторам вроде `Distinct`, `Skip`, `Take` / Prefer method syntax for simple chains — it is more compact and friendlier to operators like `Distinct`, `Skip`, `Take`.
- Материализуйте результат через `ToList()`/`ToArray()`, если намерены перебирать его больше одного раза / Materialize the result with `ToList()`/`ToArray()` if you intend to enumerate it more than once.
- Используйте query syntax для сложных `join`/`group by` — он читается как SQL / Use query syntax for complex `join`/`group by` — it reads like SQL.
- Помните, что отложенный запрос перевычисляется при каждом переборе — фиксируйте состояние источника / Remember a deferred query re-evaluates on each enumeration — freeze the source state.
- Не смешивайте два синтаксиса в одной цепочке без необходимости / Do not mix the two syntaxes in one chain without a reason.

#### Частые ошибки / Common Mistakes

- Многократное перебирание отложенного запроса в цикле → материализуйте через `ToList()` один раз (RU)
- Ожидание, что `Where` выполнится сразу → помните про отложенность; используйте `ToList()`/`Count()` для форсирования (RU)
- Изменение исходной коллекции между построением и перебором запроса → зафиксируйте копию источника перед запросом (RU)
- Путаница `Select` и `Where` (`Select` преобразует, `Where` фильтрует) → проверяйте сигнатуру: `Where` всегда возвращает `bool` (RU)
- Зацикливание на query syntax для всего, даже для `Take(5)` → используйте method syntax для тривиальных операций (RU)
- Multiple enumeration of a deferred query in a loop → materialize once with `ToList()` (EN)
- Expecting `Where` to run immediately → remember deferred execution; use `ToList()`/`Count()` to force it (EN)
- Mutating the source collection between building and enumerating the query → snapshot the source before the query (EN)
- Confusing `Select` and `Where` (`Select` transforms, `Where` filters) → check the signature: `Where` always returns `bool` (EN)
- Forcing query syntax everywhere, even for `Take(5)` → use method syntax for trivial operations (EN)

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я могу объяснить, чем `IEnumerable<T>` отличается от конкретной коллекции (RU)
- [ ] Я понимаю разницу между method syntax и query syntax и могу выбрать подходящий (RU)
- [ ] Я знаю, что запрос через `Where`/`Select` НЕ выполняется сразу (RU)
- [ ] Я могу назвать минимум 3 eager-оператора (`Count`, `First`, `ToList`) (RU)
- [ ] Я могу переписать запрос из method syntax в query syntax и наоборот (RU)
- [ ] I can explain how `IEnumerable<T>` differs from a concrete collection (EN)
- [ ] I understand the difference between method syntax and query syntax and can pick the right one (EN)
- [ ] I know that a `Where`/`Select` query does NOT run immediately (EN)
- [ ] I can name at least 3 eager operators (`Count`, `First`, `ToList`) (EN)
- [ ] I can rewrite a query from method syntax to query syntax and back (EN)

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/linq/](https://learn.microsoft.com/dotnet/csharp/linq/)

---

[⬆ К модулю M08](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
