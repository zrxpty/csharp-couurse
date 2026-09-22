---
[← Предыдущий: M03-L02](lesson-M03-L02-switch-patterns.md) | [⬆ К модулю M03](../README.md) | [Следующий: M03-L04 →](lesson-M03-L04-break-continue.md)
---

### Урок M03-L03: Циклы for, while, do-while, foreach / Loops: for, while, do-while, foreach

**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Циклы позволяют выполнять блок кода многократно. В C# есть четыре основных вида циклов, и каждый решает свою задачу. Понимать разницу между ними — значит писать код, который легко читать и в котором трудно ошибиться.

**Цикл `for` — цикл со счётчиком.** Это классический выбор, когда заранее известно, сколько раз нужно повторить тело цикла. Заголовок цикла состоит из трёх частей: инициализации счётчика, условия продолжения и шага. Представьте конвейер, где рабочий должен обработать ровно 100 деталей: счётчик идёт от 0 до 99, шаг — +1, и на каждой итерации обрабатывается одна деталь. `for` удобен для прохода по индексам массива, генерации последовательностей и любых задач с предсказуемым числом повторений.

**Цикл `while` — цикл с предусловием.** Условие проверяется *до* выполнения тела. Если условие ложно с самого начала, тело не выполнится ни разу. Это идеальный выбор, когда количество итераций неизвестно заранее и зависит от состояния программы или внешних данных. Аналогия: «иди, пока не упрёшься в стену». Классические примеры — чтение потока до конца, ожидание изменения флага, обработка ввода пользователя. Главное правило: внутри тела должно происходить что-то, что в итоге сделает условие ложным, иначе цикл станет бесконечным.

**Цикл `do-while` — цикл с постусловием.** Условие проверяется *после* выполнения тела, поэтому тело гарантированно выполнится хотя бы один раз. Это полезно, когда первое действие обязательно, а повтор зависит от результата. Аналогия: «попробуй блюдо, а потом решай, заказывать ли ещё». Типичные сценарии — диалоги с пользователем (сначала показать меню, потом спросить «продолжить?»), валидация ввода (сначала прочитать значение, потом проверить), инициализация с откатом.

**Цикл `foreach` — проход по коллекции.** Он перебирает все элементы любого типа, реализующего `IEnumerable` или `IEnumerable<T>`: массивы, списки, словари, множества, результаты LINQ-запросов. Вам не нужен индекс и не нужно знать размер — компилятор сам извлекает элементы по одному. Это самый безопасный и читаемый способ обхода коллекции. Аналогия: вы передаёте корзину с яблоками, и помощник подаёт вам по одному яблоку, пока корзина не опустеет.

**Когда какой цикл выбирать.** `for` — когда есть счётчик и известен диапазон. `while` — когда повторение зависит от условия и может не случиться ни разу. `do-while` — когда тело нужно выполнить минимум один раз. `foreach` — когда нужно обойти всю коллекцию без изменения её структуры. В современной практике `foreach` и `for` покрывают большинство задач; `while` и `do-while` встречаются реже, но незаменимы в своих сценариях.

**Важное ограничение `foreach`: изменять коллекцию во время перебора нельзя.** Добавление, удаление элементов или изменение размера коллекции внутри тела `foreach` выбрасывает `InvalidOperationException` («Collection was modified»). Это защитный механизм: итератор отслеживает версию коллекции. Если нужно удалить элементы по условию, используйте обратный цикл `for`, метод `List<T>.RemoveAll` или сначала соберите элементы для удаления в отдельный список, а потом удалите их вторым проходом. Мутация *значений* элементов (изменение свойств объекта) разрешена — запрещена только структурная модификация самой коллекции.

#### Theory (EN)

Loops let you execute a block of code repeatedly. C# provides four main loop constructs, each suited to a particular kind of task. Knowing the difference is what lets you write code that is easy to read and hard to get wrong.

**The `for` loop — a counted loop.** This is the classic choice when you know in advance how many times the body must run. The header has three parts: an initializer for the counter, a continuation condition, and an increment step. Think of an assembly line where a worker must process exactly 100 parts: the counter goes from 0 to 99, the step is +1, and one part is processed per iteration. `for` is convenient for indexing arrays, generating sequences, and any task with a predictable number of repetitions.

**The `while` loop — a pre-condition loop.** The condition is checked *before* the body runs. If the condition is false from the start, the body never executes. This is ideal when the number of iterations is unknown in advance and depends on program state or external data. The analogy: "walk until you hit a wall." Classic examples are reading a stream to the end, waiting on a flag to change, or processing user input. The cardinal rule: something inside the body must eventually make the condition false, or the loop runs forever.

**The `do-while` loop — a post-condition loop.** The condition is checked *after* the body runs, so the body is guaranteed to execute at least once. This is useful when the first action is mandatory and repetition depends on the outcome. Analogy: "taste the dish, then decide whether to order more." Typical scenarios are user dialogs (show the menu first, then ask "continue?"), input validation (read a value, then check it), and initialization with rollback.

**The `foreach` loop — iterating over a collection.** It walks every element of any type implementing `IEnumerable` or `IEnumerable<T>`: arrays, lists, dictionaries, sets, LINQ query results. You do not need an index or the size — the compiler pulls elements out one at a time. This is the safest and most readable way to traverse a collection. Analogy: you hand over a basket of apples and a helper passes you one apple at a time until the basket is empty.

**When to choose which.** `for` — when you have a counter and a known range. `while` — when repetition depends on a condition and may not happen at all. `do-while` — when the body must run at least once. `foreach` — when you want to walk the whole collection without changing its structure. In modern practice `foreach` and `for` cover most tasks; `while` and `do-while` are less common but indispensable in their niches.

**Key restriction of `foreach`: you cannot modify the collection during iteration.** Adding, removing elements, or resizing the collection inside a `foreach` body throws `InvalidOperationException` ("Collection was modified"). This is a safety mechanism — the iterator tracks the collection's version. To remove elements by condition, use a reverse `for` loop, the `List<T>.RemoveAll` method, or first collect the elements to remove into a separate list and delete them on a second pass. Mutating element *values* (changing an object's properties) is allowed — only structural modification of the collection itself is forbidden.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — top-level statements
// Демонстрация всех четырёх циклов / Demonstration of all four loops

using System;
using System.Collections.Generic;

// --- for: счётчик, известный диапазон / for: counter, known range ---
int sumFor = 0;
for (int i = 1; i <= 10; i++)          // 1..10 включительно / 1..10 inclusive
{
    sumFor += i;                        // накапливаем сумму / accumulate sum
}
Console.WriteLine($"for: sum 1..10 = {sumFor}");   // 55

// --- while: предусловие, может не выполниться / while: pre-condition, may not run ---
string? line = Console.ReadLine();
int wordCount = 0;
while (line is not null && line.Length > 0)        // пока есть ввод / while there is input
{
    wordCount += line.Split(' ', StringSplitOptions.RemoveEmptyEntries).Length;
    line = Console.ReadLine();                    // продвижение к выходу / advance toward exit
}
Console.WriteLine($"while: words read = {wordCount}");

// --- do-while: постусловие, минимум один раз / do-while: post-condition, at least once ---
int guess;
do
{
    Console.Write("Введите число от 1 до 10: ");  // Enter a number 1..10
    guess = int.TryParse(Console.ReadLine(), out var v) ? v : -1;
}
while (guess is < 1 or > 10);                      // pattern matching C# 12 / pattern matching
Console.WriteLine($"do-while: accepted = {guess}");

// --- foreach: обход IEnumerable / foreach: walk IEnumerable ---
var fruits = new List<string> { "apple", "banana", "cherry" };
foreach (var fruit in fruits)                      // без индекса / no index needed
{
    Console.WriteLine($"foreach: {fruit}");
}

// --- Безопасное удаление по условию / Safe conditional removal ---
var numbers = new List<int> { 1, 2, 3, 4, 5, 6, 7, 8 };
// ❌ Ошибка: нельзя менять коллекцию в foreach / Wrong: cannot modify collection in foreach
// foreach (var n in numbers) if (n % 2 == 0) numbers.Remove(n);  // InvalidOperationException

// ✅ Вариант 1: RemoveAll / Option 1: RemoveAll
var copy1 = new List<int> { 1, 2, 3, 4, 5, 6, 7, 8 };
copy1.RemoveAll(n => n % 2 == 0);                  // удалить все чётные / remove all even
Console.WriteLine($"RemoveAll: [{string.Join(", ", copy1)}]");   // 1, 3, 5, 7

// ✅ Вариант 2: обратный for / Option 2: reverse for
var copy2 = new List<int> { 1, 2, 3, 4, 5, 6, 7, 8 };
for (int i = copy2.Count - 1; i >= 0; i--)         // с конца / from the end
{
    if (copy2[i] % 2 == 0) copy2.RemoveAt(i);      // индексы не сбиваются / indices stay valid
}
Console.WriteLine($"reverse for: [{string.Join(", ", copy2)}]"); // 1, 3, 5, 7

// --- Мутация значения объекта разрешена / Mutating object values is allowed ---
var items = new List<(string Name, int Qty)> { ("A", 1), ("B", 2) };
foreach (ref var it in CollectionsMarshal.AsSpan(items)) // ref-доступ к элементу / ref access
{
    it.Qty += 10;                                  // меняем значение, не структуру / value, not structure
}
Console.WriteLine($"mutated: {string.Join(", ", items.Select(x => $"{x.Name}={x.Qty}"))}");
```

#### Best Practices
- Предпочитайте `foreach` для обхода коллекций: он читается проще и исключает ошибки с индексами. / Prefer `foreach` for collection traversal: it reads more clearly and rules out index errors.
- Используйте `for` только когда реально нужен индекс или шаг, отличный от +1. / Use `for` only when you genuinely need an index or a non-+1 step.
- В `while` всегда проверяйте, что тело продвигает условие к `false`, иначе получите бесконечный цикл. / In `while`, always confirm the body moves the condition toward `false`, or you get an infinite loop.
- Применяйте `do-while` там, где тело обязано выполниться хотя бы раз (меню, валидация). / Apply `do-while` where the body must run at least once (menu, validation).
- Не модифицируйте структуру коллекции внутри `foreach`; используйте `RemoveAll`, обратный `for` или два прохода. / Do not structurally modify a collection inside `foreach`; use `RemoveAll`, a reverse `for`, or two passes.
- Избегайте «магических» чисел в условиях `for` — выносите границы в именованные переменные. / Avoid magic numbers in `for` conditions — name the bounds.
- Для линейного поиска по коллекции предпочитайте LINQ (`First`, `Any`) вместо ручного `foreach` с флагом. / For linear search over a collection, prefer LINQ (`First`, `Any`) over a hand-rolled `foreach` with a flag.

#### Частые ошибки / Common Mistakes
- Условие `for` со знаком `<=` вместо `<` даёт ошибку «на единицу» (off-by-one). → Чётко определяйте, включительна ли верхняя граница, и проверяйте граничные значения тестами. / A `for` condition using `<=` instead of `<` causes an off-by-one error. → Decide explicitly whether the upper bound is inclusive and cover the boundary with tests.
- Бесконечный `while`: забыли изменить переменную условия внутри тела. → Гарантируйте продвижение условия и добавьте предохранительный счётчик итераций. / Infinite `while`: forgot to change the condition variable inside the body. → Guarantee progress and add a safety iteration counter.
- Мутация коллекции в `foreach` → `InvalidOperationException`. → Используйте `RemoveAll`, обратный `for` или соберите удаляемые элементы в отдельный список. / Modifying a collection inside `foreach` → `InvalidOperationException`. → Use `RemoveAll`, a reverse `for`, or collect items to remove into a separate list.
- Изменение переменной цикла `for` внутри тела «сбивает» логику шага. → Считайте счётчик неизменяемым; управляйте потоком через `break`/`continue`. / Changing the `for` counter inside the body throws off the step logic. → Treat the counter as immutable; steer flow with `break`/`continue`.
- Использование `do-while` там, где тело не должно выполняться при ложном условии с первого раза. → Замените на `while`, если ноль итераций допустим. / Using `do-while` where the body should not run when the condition is false up front. → Replace with `while` when zero iterations are valid.
- Обход словаря `Dictionary<TKey,TValue>` через `foreach (var k in dict)` и попытка обратиться к значению через `dict[k]` вместо `foreach (var kvp in dict)`. → Перебирайте `KeyValuePair` сразу со значением — быстрее и чище. / Walking a `Dictionary<TKey,TValue>` with `foreach (var k in dict)` and looking up `dict[k]` instead of `foreach (var kvp in dict)`. → Iterate `KeyValuePair` directly — faster and cleaner.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я выбрал тип цикла, соответствующий задаче (счётчик / условие / коллекция). / I chose the loop type that matches the task (counter / condition / collection).
- [ ] В каждом `while` и `do-while` тело гарантированно приближает условие к `false`. / In every `while` and `do-while` the body guarantees progress toward `false`.
- [ ] Границы `for` корректны, off-by-one исключён тестами на краях. / `for` bounds are correct; off-by-one is excluded by edge tests.
- [ ] Я не модифицирую структуру коллекции внутри `foreach`. / I do not structurally modify a collection inside `foreach`.
- [ ] Для удаления по условию использован `RemoveAll` или обратный `for`. / For conditional removal I use `RemoveAll` or a reverse `for`.
- [ ] Там, где нужен индекс, я использую `for`, а не `foreach` с отдельным счётчиком. / Where an index is needed I use `for`, not `foreach` with a side counter.
- [ ] Цикл не содержит «магических» чисел — границы именованы. / The loop has no magic numbers — bounds are named.
- [ ] Для досрочного выхода или пропуска используются `break`/`continue`, а не вложенные флаги. / Early exit or skip uses `break`/`continue`, not nested flags.

#### Ресурсы / Resources
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/statements/iteration-statements — Операторы итерации (for, while, do-while, foreach) / Iteration statements (for, while, do-while, foreach)
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/foreach-in — Ключевое слово foreach, in / The foreach, in keyword

---
[← Предыдущий: M03-L02](lesson-M03-L02-switch-patterns.md) | [⬆ К модулю M03](../README.md) | [Следующий: M03-L04 →](lesson-M03-L04-break-continue.md)
---
