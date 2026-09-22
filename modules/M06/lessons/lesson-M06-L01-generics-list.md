[⬆ К модулю M06](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M06-L01: Зачем дженерики, List<T> / Why generics, List<T>

**Модуль / Module:** M06
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

До появления дженериков в C# 2.0 (2005 год) программисты хранили коллекции значений в структурах на основе `object` — например, в `System.Collections.ArrayList`. Поскольку `object` — базовый тип для всех ссылочных типов, но **значимые типы** (`int`, `double`, `struct`) не являются ссылочными, любое значение типа `int`, помещённое в `ArrayList`, приходилось «упаковывать» (boxing) в объект-обёртку в куче. Извлечение значения обратно требовало распаковки (unboxing) с явным приведением. Это порождало три проблемы:

1. **Производительность.** Boxing выделяет память в куче и копирует значение; unboxing копирует обратно и проверяет тип. Для коллекции из миллиона чисел это миллионы лишних аллокаций, которые потом убирает сборщик мусора.
2. **Типобезопасность.** Компилятор не знает, что лежит внутри `ArrayList`. Можно случайно добавить строку в «список чисел» — ошибка всплывёт только в рантайме как `InvalidCastException`.
3. **Читаемость.** Код пестрит приведениями `(int)items[0]`, которые ничего не говорят о намерении.

**Дженерики** решают все три проблемы разом. Конструкция `List<T>` — это параметризованный тип: буква `T` (от *type*) — placeholder, который заполняется конкретным типом при создании, например `new List<int>()`. Компилятор подставляет `int` всюду, где встречается `T`, и генерирует специализированную структуру хранения. Для значимых типов `List<int>` хранит значения прямо во внутреннем массиве `int[]` — **boxing не происходит**, память компактна, доступ по индексу — это обычное чтение из массива.

Аналогия: `ArrayList` — это большая картонная коробка, куда кладут вещи, каждую обмотав плёнкой и приклеив бирку «объект». Чтобы достать нужную вещь, надо снять плёнку и проверить, та ли это вещь. `List<int>` — это ящик с ячейками ровно под отвёртки: ничего заворачивать не надо, и кладёшь, и берёшь напрямую.

Ключевые операции `List<T>`:
- `Add(item)` — добавить в конец; при переполнении внутренний массив пересоздаётся с удвоенной ёмкостью (`Capacity`), поэтому добавление в среднем O(1).
- `AddRange(collection)` — добавить сразу несколько.
- `Remove(item)` — найти первое совпадение и удалить; оставшиеся элементы сдвигаются, O(n).
- `RemoveAt(index)` — удалить по индексу, O(n).
- `Count` — текущее количество элементов (свойство, не метод).
- `Capacity` — размер внутреннего массива (≥ `Count`).
- Индексатор `list[i]` — доступ за O(1).
- `Contains`, `IndexOf`, `Sort`, `ToArray`, `Clear`.

Важно различать `Count` (сколько реально положили) и `Capacity` (сколько ещё поместится без реаллокации). Лишняя `Capacity` — это занятая, но неиспользуемая память. Если итоговый размер известен заранее, передайте его в конструктор `new List<int>(capacity)` или вызовите `TrimExcess()` после заполнения.

В современном API принято принимать коллекции через **`IReadOnlyList<T>`** в параметрах методов и публичных свойствах, а возвращать `List<T>` только тогда, когда вызывающая сторона действительно будет менять список. `IReadOnlyList<T>` даёт индексатор и `Count`, но не даёт `Add`/`Remove` — это «договор о ненарушении». Внутри реализации по-прежнему живёт `List<T>`. Такой приём уменьшает связность кода и защищает данные от случайной модификации.

Итог: дженерики — это типобезопасность без накладных расходов, а `List<T>` и `IReadOnlyList<T>` — базовые инструменты для работы с изменяемыми и предназначенными только для чтения последовательностями.

#### Theory (EN)

Before generics arrived in C# 2.0 (2005), developers stored value collections inside object-based containers such as `System.Collections.ArrayList`. Because `object` is the base of every reference type but **value types** (`int`, `double`, any `struct`) are not references, every `int` placed into an `ArrayList` had to be **boxed** into a heap-allocated wrapper object. Pulling the value back out required **unboxing** with an explicit cast. This created three problems:

1. **Performance.** Boxing allocates heap memory and copies the value; unboxing copies it back and performs a type check. For a collection of a million integers, that is a million unnecessary allocations for the garbage collector to chase.
2. **Type safety.** The compiler does not know what lives inside an `ArrayList`. You can accidentally drop a string into a "list of numbers"; the mistake surfaces only at runtime as an `InvalidCastException`.
3. **Readability.** Code fills up with casts like `(int)items[0]` that say nothing about intent.

**Generics** solve all three at once. `List<T>` is a parameterized type: `T` (for *type*) is a placeholder filled in when the instance is created, e.g. `new List<int>()`. The compiler substitutes `int` everywhere `T` appears and produces a specialized storage layout. For value types, `List<int>` keeps values directly in an internal `int[]` — **no boxing occurs**, memory is compact, and indexed access is a plain array read.

Analogy: `ArrayList` is a big cardboard box where each item is wrapped in film and tagged "object". To retrieve one, you peel the film and check whether it is the right thing. `List<int>` is a tray with slots shaped exactly for screwdrivers — nothing to wrap, put in or take out directly.

Core `List<T>` operations:
- `Add(item)` — append; when the internal array overflows it is reallocated with doubled `Capacity`, so amortized `Add` is O(1).
- `AddRange(collection)` — append many at once.
- `Remove(item)` — find the first match and remove it; remaining elements shift, O(n).
- `RemoveAt(index)` — remove by index, O(n).
- `Count` — the current number of elements (a property, not a method).
- `Capacity` — the size of the internal array (≥ `Count`).
- Indexer `list[i]` — O(1) access.
- `Contains`, `IndexOf`, `Sort`, `ToArray`, `Clear`.

Distinguish `Count` (how many you actually put in) from `Capacity` (how many fit before reallocation). Extra `Capacity` is allocated but unused memory. If the final size is known up front, pass it to the constructor `new List<int>(capacity)` or call `TrimExcess()` after filling.

Modern API design accepts collections through **`IReadOnlyList<T>`** in method parameters and public properties, and returns `List<T>` only when callers genuinely need to mutate. `IReadOnlyList<T>` exposes the indexer and `Count` but not `Add`/`Remove` — a "do not touch" contract. Underneath, the implementation may still be a `List<T>`. This pattern reduces coupling and shields data from accidental modification.

In short: generics give type safety with no runtime overhead, and `List<T>` together with `IReadOnlyList<T>` are the everyday tools for mutable and read-only sequences.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Generics & List<T>
// Демонстрация: boxing против List<T>, базовые операции, IReadOnlyList<T>
// Demo: boxing vs List<T>, basic operations, IReadOnlyList<T>

using System;
using System.Collections;
using System.Collections.Generic;

namespace M06L01;

internal static class Program
{
    private static void Main()
    {
        // --- Старый подход: ArrayList на базе object ---
        // --- Old approach: ArrayList backed by object ---
        ArrayList boxed = new();
        boxed.Add(1);          // boxing: int → object  /  boxing: int → object
        boxed.Add(2);
        boxed.Add("oops");     // компилятор не возражает / compiler does not complain

        // int sum = (int)boxed[0] + (int)boxed[1];  // OK
        // foreach (int n in boxed) ...              // InvalidCastException в рантайме на "oops"
        //                                            // InvalidCastException at runtime on "oops"

        // --- Современный подход: List<T> ---
        // --- Modern approach: List<T> ---
        List<int> numbers = new(capacity: 4) { 10, 20, 30 };  // ёмкость задана заранее / capacity set up front
        numbers.Add(40);            // добавить в конец, O(1) в среднем / append, amortized O(1)
        numbers.AddRange([50, 60]); // коллекционное выражение C# 12 / C# 12 collection expression

        Console.WriteLine($"Count = {numbers.Count}, Capacity = {numbers.Capacity}");
        // Count = 6, Capacity = 8 (удвоение при переполнении / doubled on overflow)

        // Доступ по индексу без приведения, без boxing / indexed access, no cast, no boxing
        int first = numbers[0];
        Console.WriteLine($"numbers[0] = {first}");

        // Удаление / Removal
        bool removed = numbers.Remove(30);   // найти значение и удалить / find value and remove
        Console.WriteLine($"Remove(30) → {removed}, Count = {numbers.Count}");
        numbers.RemoveAt(0);                 // удалить по индексу / remove by index
        Console.WriteLine($"After RemoveAt(0) Count = {numbers.Count}");

        // Поиск / Lookup
        Console.WriteLine($"Contains(50) = {numbers.Contains(50)}");
        Console.WriteLine($"IndexOf(50) = {numbers.IndexOf(50)}");

        // --- Передача по «только для чтения» интерфейсу ---
        // --- Hand off via the read-only interface ---
        IReadOnlyList<int> readOnly = numbers;   // неявное приведение / implicit conversion
        PrintAll(readOnly);                       // метод видит только Count и индексатор
                                                   // method sees only Count and the indexer

        // readOnly.Add(99);  // не компилируется: Add недоступен / won't compile: Add is not exposed

        numbers.TrimExcess();   // освободить лишнюю Capacity / release spare Capacity
    }

    // Метод принимает «обещание не менять» — контракт на чтение.
    // Method accepts a "promise not to mutate" — a read contract.
    private static void PrintAll(IReadOnlyList<int> items)
    {
        for (int i = 0; i < items.Count; i++)
        {
            Console.WriteLine($"  [{i}] = {items[i]}");
        }
    }
}
```

#### Best Practices

- Выбирайте `List<T>`, а не `ArrayList` или `object[]` — это даёт типобезопасность и убирает boxing для значимых типов.
- Принимайте коллекции в параметрах как `IEnumerable<T>` или `IReadOnlyList<T>`, если не планируете их менять; возвращайте `List<T>` только когда вызывающая сторона будет мутировать список.
- Задавайте `capacity` в конструкторе `List<T>`, когда размер известен заранее, чтобы избежать лишних реаллокаций и разрастания `Capacity`.
- Используйте `IReadOnlyList<T>` (или `ImmutableList<T>` из `System.Collections.Immutable`) для публичных свойств, чтобы защитить внутреннее состояние от внешней модификации.
- Предпочитайте `RemoveAt(index)` вместо `Remove(item)`, когда известен индекс: поиск значения лишний и дорогой для больших списков.

- Choose `List<T>` over `ArrayList` or `object[]` — it gives type safety and removes boxing for value types.
- Accept collections in parameters as `IEnumerable<T>` or `IReadOnlyList<T>` when you will not mutate them; return `List<T>` only when callers need to modify it.
- Pass `capacity` to the `List<T>` constructor when the size is known up front, to avoid repeated reallocations and `Capacity` growth.
- Use `IReadOnlyList<T>` (or `ImmutableList<T>` from `System.Collections.Immutable`) for public properties to shield internal state from external changes.
- Prefer `RemoveAt(index)` over `Remove(item)` when the index is known: value lookup is extra work and expensive for large lists.

#### Частые ошибки / Common Mistakes

- Использование `ArrayList` для значимых типов → всегда выбирайте `List<T>`, чтобы избежать boxing и потери типобезопасности.
- Путаница `Count` и `Capacity` → `Count` — реальное число элементов, `Capacity` — размер внутреннего массива; не вызывайте `Add`, ориентируясь на `Capacity`.
- Мутирование списка во время `foreach` по нему → получите `InvalidOperationException`; соберите изменения и примените через `RemoveAll` или после цикла.
- Публичное свойство типа `List<T>` без защиты → возвращайте `IReadOnlyList<T>` или копию, чтобы наружу не утекла изменяемая ссылка.
- `Remove(item)` для ссылочных типов, не переопределивших `Equals` → удаляется по ссылочному равенству; проверьте семантику `Equals`/`IEquatable<T>`.
- Игнорирование `IndexOf` с `-1` → всегда проверяйте, что индекс найден, перед `RemoveAt`.

- Using `ArrayList` for value types → always pick `List<T>` to avoid boxing and loss of type safety.
- Confusing `Count` with `Capacity` → `Count` is the real element count, `Capacity` is the internal array size; do not drive `Add` off `Capacity`.
- Mutating a list while `foreach`-ing over it → you get `InvalidOperationException`; collect changes and apply via `RemoveAll` or after the loop.
- Exposing a public `List<T>` property unprotected → return `IReadOnlyList<T>` or a copy so a mutable reference does not leak.
- Calling `Remove(item)` on reference types without overriding `Equals` → removal uses reference equality; verify your `Equals`/`IEquatable<T>` semantics.
- Ignoring a `-1` from `IndexOf` → always check the index was found before calling `RemoveAt`.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я могу объяснить, почему `ArrayList` вызывает boxing для `int`, а `List<int>` — нет.
- [ ] Я знаю разницу между `Count` и `Capacity` и когда задавать `capacity` в конструкторе.
- [ ] Я могу назвать сложность `Add`, `Remove`, `RemoveAt`, доступа по индексу.
- [ ] Я использую `IReadOnlyList<T>` для параметров и свойств, которые не должны меняться.
- [ ] Я не мутирую список внутри `foreach` и знаю про `RemoveAll`.
- [ ] Мой код компилируется под C# 12 / .NET 8 без предупреждений о `null` и `boxing`.

- [ ] I can explain why `ArrayList` boxes an `int` while `List<int>` does not.
- [ ] I know the difference between `Count` and `Capacity` and when to pass `capacity` to the constructor.
- [ ] I can state the complexity of `Add`, `Remove`, `RemoveAt`, and indexed access.
- [ ] I use `IReadOnlyList<T>` for parameters and properties that must not change.
- [ ] I do not mutate a list inside `foreach` and I know about `RemoveAll`.
- [ ] My code compiles under C# 12 / .NET 8 with no `null` or `boxing` warnings.

#### Ресурсы / Resources

- [Microsoft Learn — Generics — https://learn.microsoft.com/dotnet/csharp/fundamentals/types/generics]
- [Microsoft Learn — List<T> — https://learn.microsoft.com/dotnet/api/system.collections.generic.list-1]
- [Microsoft Learn — IReadOnlyList<T> — https://learn.microsoft.com/dotnet/api/system.collections.generic.ireadonlylist-1]
- [Microsoft Learn — Boxing and Unboxing — https://learn.microsoft.com/dotnet/csharp/programming-guide/types/boxing-and-unboxing]

---

[⬆ К модулю M06](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
