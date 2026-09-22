[⬆ К модулю M08](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M08-L07: First/SingleOrDefault/ElementAt и исключения / First/SingleOrDefault/ElementAt and exceptions

**Модуль / Module:** M08
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Представьте длинный коридор с комнатами, в каждой из которых лежит предмет. Вам нужно взять **первый** предмет, **единственный** предмет или предмет из комнаты с **конкретным номером**. LINQ даёт для этого три семейства операторов: `First`, `Single` и `ElementAt`. Каждое из них имеет «бросающую» версию и версию с `Default` — и именно выбор между ними определяет, получите ли вы исключение или `default(T)`.

**First / FirstOrDefault** возвращают самый первый элемент последовательности, удовлетворяющий условию (или просто первый, если условие не задано). Аналогия: вы открываете первую дверь и забираете то, что лежит внутри. Если дверь оказалась пустой (пустая последовательность) или ни одна комната не подошла, `First` бросает `InvalidOperationException`, а `FirstOrDefault` молча возвращает `default(T)` — `null` для ссылочных типов, `0` для `int`, `false` для `bool` и т.д. Используйте `First`, когда **пустой результат — это ошибка программы** (должен быть хотя бы один элемент), и `FirstOrDefault`, когда пустота — нормальный сценарий, который вы обработаете отдельно.

**Single / SingleOrDefault** строже. Они требуют, чтобы последовательность содержала **ровно один** элемент (или ровно один, удовлетворяющий условию). Аналогия: вы идёте по коридору и проверяете, что подходящая комната — единственная; если нашли вторую такую же, бросаете ключи на пол и уходите. `Single` бросает `InvalidOperationException` в двух случаях: когда элементов ноль и когда их больше одного. `SingleOrDefault` отличает эти ситуации — при нуле элементов вернёт `default`, но при двух и более всё равно бросит исключение. Это критическая деталь: суффикс `OrDefault` спасает только от «пусто», но **не от «слишком много»**. `Single` идеален для поиска по уникальному ключу (`users.Single(u => u.Id == 42)`), где наличие дубликата означает серьёзную ошибку данных.

**ElementAt / ElementAt.ElementAtOrDefault** достают элемент по индексу. Аналогия: вы идёте к комнате с конкретным номером и забираете предмет. `ElementAt` бросает `ArgumentOutOfRangeException`, если индекс отрицательный или выходит за границы; `ElementAtOrDefault` вместо этого вернёт `default`. Начиная с .NET 6 у `ElementAt` есть перегрузка, принимающая `Index` и `Range`, что позволяет писать `items.ElementAt(^1)` для последнего элемента — удобно и читаемо.

**Когда что выбирать?**
- `First` — «я уверен, что последовательность не пуста, иначе это баг».
- `FirstOrDefault` — «может быть пусто, я проверю результат на `null`/`default`».
- `Single` — «должен быть ровно один элемент, дубликаты недопустимы».
- `SingleOrDefault` — «должен быть один или ноль, но не больше».
- `ElementAt` — «нужен элемент по известной позиции».

Производительность: `First` и `ElementAt` на `IList<T>` работают за O(1) (LINQ видит индексатор и не перебирает), а на `IEnumerable` общего вида — O(n). `Single` всегда перебирает до второго совпадения, чтобы убедиться в уникальности, поэтому он чуть дороже `First`, но даёт строгую гарантию. Не используйте `Single` там, где достаточно `First`, — это лишняя работа и риск ложного исключения.

Главное правило: выбирайте оператор так, чтобы **семантика исключения совпадала с бизнес-логикой**. Если «нет элемента» — ошибка, пусть бросается; если норма — берите `OrDefault` и обрабатывайте `default` осознанно, не путая его с валидным значением.

#### Theory (EN)

Imagine a long hallway of rooms, each holding one item. You need to grab the **first** item, the **single** item, or the item from a room with a **specific number**. LINQ gives you three families for this: `First`, `Single`, and `ElementAt`. Each comes in a “throwing” flavor and an `OrDefault` flavor — and picking the right one decides whether you get an exception or `default(T)`.

**First / FirstOrDefault** return the first element that matches a predicate (or simply the first element when no predicate is given). Analogy: you open the first door and take what is inside. If the door is empty (empty sequence) or no room qualified, `First` throws `InvalidOperationException`, while `FirstOrDefault` silently returns `default(T)` — `null` for reference types, `0` for `int`, `false` for `bool`, and so on. Use `First` when **an empty result is a programming error** (there must be at least one element), and `FirstOrDefault` when emptiness is a legitimate scenario you will handle separately.

**Single / SingleOrDefault** are stricter. They demand exactly one element in the sequence (or exactly one matching the predicate). Analogy: you walk the hallway and verify the matching room is unique; if you spot a second match you drop the keys and leave. `Single` throws `InvalidOperationException` in two cases: zero elements or more than one. `SingleOrDefault` distinguishes these — with zero elements it returns `default`, but with two or more it still throws. This is a critical detail: the `OrDefault` suffix saves you only from “empty”, **not from “too many”**. `Single` is ideal for lookups by a unique key (`users.Single(u => u.Id == 42)`), where a duplicate signals a serious data error.

**ElementAt / ElementAtOrDefault** fetch an element by index. Analogy: you walk to the room with the given number and take the item. `ElementAt` throws `ArgumentOutOfRangeException` when the index is negative or out of range; `ElementAtOrDefault` returns `default` instead. Since .NET 6, `ElementAt` has an overload accepting `Index` and `Range`, so you can write `items.ElementAt(^1)` for the last element — convenient and readable.

**When to pick which?**
- `First` — “I am sure the sequence is non-empty, otherwise it is a bug.”
- `FirstOrDefault` — “it might be empty, I will check the result for `null`/`default`.”
- `Single` — “there must be exactly one element, duplicates are forbidden.”
- `SingleOrDefault` — “there must be one or zero, but never more.”
- `ElementAt` — “I need the element at a known position.”

Performance: `First` and `ElementAt` run in O(1) on `IList<T>` (LINQ detects the indexer and skips enumeration), but in O(n) on a general `IEnumerable`. `Single` always enumerates up to the second match to prove uniqueness, so it is slightly more expensive than `First` but gives a strict guarantee. Do not use `Single` where `First` would do — it is extra work and a risk of a false exception.

The guiding rule: choose the operator so the **semantics of the exception match the business logic**. If “no element” is an error, let it throw; if it is normal, take the `OrDefault` variant and handle `default` deliberately, never confusing it with a valid value.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — First / Single / ElementAt и их OrDefault-варианты
// First / Single / ElementAt and their OrDefault variants

using System;
using System.Collections.Generic;
using System.Linq;

var users = new List<User>
{
    new(1, "Alice"),   // индекс 0 / index 0
    new(2, "Bob"),     // индекс 1 / index 1
    new(3, "Charlie"), // индекс 2 / index 2
};

// 1) First — бросает, если пусто / throws when empty
User first = users.First();
Console.WriteLine($"Первый / First: {first.Name}");

// FirstOrDefault — возвращает default (null), если пусто
// returns default (null) when empty
User? maybeFirst = users.FirstOrDefault(u => u.Name.StartsWith("Z"));
Console.WriteLine($"Может null / Maybe null: {maybeFirst?.Name ?? "—"}");

// 2) Single — ровно один элемент, иначе исключение
// exactly one element, otherwise exception
User byId = users.Single(u => u.Id == 2);
Console.WriteLine($"По Id=2 / By Id=2: {byId.Name}");

// SingleOrDefault — ноль или один, но НЕ больше
// zero or one, but NOT more
User? maybeAdmin = users.SingleOrDefault(u => u.Id == 999);
Console.WriteLine($"Нет такого / No such: {maybeAdmin?.Name ?? "—"}");

// 3) ElementAt — по индексу, бросает ArgumentOutOfRangeException
// by index, throws ArgumentOutOfRangeException when out of range
User second = users.ElementAt(1);
Console.WriteLine($"Индекс 1 / Index 1: {second.Name}");

// ElementAtOrDefault — default вместо исключения
// default instead of exception
User? maybeTenth = users.ElementAtOrDefault(10);
Console.WriteLine($"Индекс 10 / Index 10: {maybeTenth?.Name ?? "—"}");

// .NET 6+: ElementAt принимает Index — последний элемент
// .NET 6+: ElementAt accepts Index — last element
User last = users.ElementAt(^1);
Console.WriteLine($"Последний / Last: {last.Name}");

// 4) Демонстрация исключений / Demonstrating exceptions
try
{
    // Пусто → First бросает InvalidOperationException
    // Empty → First throws InvalidOperationException
    _ = Array.Empty<User>().First();
}
catch (InvalidOperationException ex)
{
    Console.WriteLine($"First на пустом / First on empty: {ex.Message}");
}

try
{
    // Два совпадения → Single бросает, даже OrDefault не спасёт
    // Two matches → Single throws, even OrDefault won't help
    _ = users.SingleOrDefault(u => u.Id > 0);
}
catch (InvalidOperationException ex)
{
    Console.WriteLine($"Single >1 совпадения / Single >1 match: {ex.Message}");
}

try
{
    // Индекс за границей → ArgumentOutOfRangeException
    // Index out of range → ArgumentOutOfRangeException
    _ = users.ElementAt(99);
}
catch (ArgumentOutOfRangeException ex)
{
    Console.WriteLine($"ElementAt вне границ / ElementAt out of range: {ex.Message}");
}

// Запись для примера / Record for the example
internal sealed record User(int Id, string Name);
```

#### Best Practices

- Выбирайте `First`, когда пустая последовательность — это баг; `FirstOrDefault`, когда пустота допустима. / Choose `First` when an empty sequence is a bug; `FirstOrDefault` when emptiness is acceptable.
- Используйте `Single` для поиска по уникальному ключу, чтобы ловить дубликаты данных на раннем этапе. / Use `Single` for unique-key lookups to catch data duplicates early.
- Не путайте `SingleOrDefault` с «безопасным всегда» — он бросает при >1 элементе. / Do not mistake `SingleOrDefault` for “always safe” — it throws on >1 element.
- Проверяйте результат `FirstOrDefault` на `default` явно, особенно для значимых типов, где `0` может быть валидным. / Check the `FirstOrDefault` result for `default` explicitly, especially for value types where `0` may be valid.
- Предпочитайте `ElementAt(^1)` вместо `Last()`, если нужна позиция через `Index` — это читаемее. / Prefer `ElementAt(^1)` over `Last()` when you want positional access via `Index` — it is more readable.
- На горячих путях передавайте `IList<T>`/массивы, чтобы `First` и `ElementAt` работали за O(1). / On hot paths pass `IList<T>`/arrays so `First` and `ElementAt` run in O(1).

#### Частые ошибки / Common Mistakes

- Использование `Single` там, где достаточно `First` → получаете ложное исключение при дубликатах, которые не критичны. / Using `Single` where `First` suffices → you get a false exception on non-critical duplicates.
- Ожидание, что `SingleOrDefault` вернёт `null` при дубликатах → на самом деле бросает исключение. / Expecting `SingleOrDefault` to return `null` on duplicates → it actually throws.
- Проверка `FirstOrDefault(u => u.Age == 0)` через `== default` для ссылочных типов → `null` спутан с валидным значением. / Checking `FirstOrDefault` with `== default` and confusing `null` with a valid value for value types.
- Игнорирование того, что `First` на `IEnumerable` общего вида — это O(n), а не мгновенная операция. / Ignoring that `First` on a general `IEnumerable` is O(n), not an instant operation.
- Цепочка `Where(...).First()` вместо `First(predicate)` → лишний проход и аллокация итератора. / Chaining `Where(...).First()` instead of `First(predicate)` → extra pass and iterator allocation.
- Вызов `ElementAt(users.Count)` → всегда исключение, индексация с нуля. / Calling `ElementAt(users.Count)` → always an exception, indexing is zero-based.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я могу объяснить разницу между `First` и `FirstOrDefault`. / I can explain the difference between `First` and `FirstOrDefault`.
- [ ] Я понимаю, что `SingleOrDefault` бросает при более чем одном элементе. / I understand that `SingleOrDefault` throws on more than one element.
- [ ] Я знаю, какое исключение бросает `ElementAt` при неверном индексе. / I know which exception `ElementAt` throws on a bad index.
- [ ] Я выбираю оператор по семантике ошибки, а не по привычке. / I choose the operator by error semantics, not by habit.
- [ ] Я проверяю результат `OrDefault`-вариантов на `default` осознанно. / I check the result of `OrDefault` variants for `default` deliberately.
- [ ] Я использую `Single` только там, где уникальность — реальное требование. / I use `Single` only where uniqueness is a real requirement.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.linq.enumerable.first](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.first)
- [Microsoft Learn — Enumerable.Single](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.single)
- [Microsoft Learn — Enumerable.ElementAt](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.elementat)

---

[⬆ К модулю M08](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
