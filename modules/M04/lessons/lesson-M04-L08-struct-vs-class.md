[⬆ К модулю M04](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M04-L08: struct vs class / struct vs class

**Модуль / Module:** M04
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

В C# два главных способа описать собственный тип данных — `struct` и `class`. Их главное отличие не в синтаксисе, а в том, **где живёт значение** и **как оно копируется**. Это фундамент, на котором держится производительность и корректность многих программ.

`struct` — это **тип-значение (value type)**. Когда вы объявляете переменную типа `struct`, само значение хранится прямо в этой переменной. Присваивание одной переменной другой копирует всё значение целиком — поле за полем. Локальные `struct` обычно живут в **стеке (stack)** — быстрой области памяти, которая автоматически очищается при выходе из метода. Аналогия: `struct` — это блокнот с записями. Если вы хотите дать коллеге те же записи, вы переписываете их в его блокнот. Теперь у каждого свой экземпляр, изменения одного не касаются другого.

`class` — это **ссылочный тип (reference type)**. Переменная хранит не сами данные, а **ссылку** на объект, расположенный в **куче (heap)** — управляемой памяти, которую освобождает сборщик мусора. При присваивании копируется только ссылка, оба имени указывают на один объект. Аналогия: `class` — это документ в Google Docs. Вы даёте коллеге ссылку — оба видите один файл, правки видны обоим.

**Когда выбирать struct?** Майкрософтовская эвристика проста: `struct` уместен, когда тип **маленький** (как правило, ≤ 16 байт), **логически неделим** (координата, цвет, деньги, комплексное число) и желательно **неизменяемый (immutable)**. Большие изменяемые `struct` — источник боли: каждое присваивание копирует десятки байт, мутация через свойство молча не работает (`myPoint.X = 5` не скомпилируется, если `myPoint` — поле-значение, возвращаемое методом), а боксинг (boxing) при приведении к `object` незаметно аллоцирует в куче. `class` лучше для крупных, изменяемых, «идентичных» объектов — сущностей домена, коллекций, сервисов.

**record struct (C# 10).** Начиная с C# 10 можно объявить `record struct` — тип-значение с синтаксисом записей: автоматический `Equals`/`GetHashCode` по значениям полей, `with`-выражения, деконструкция, `ToString`. Сочетает скорость `struct` с удобством record. Объявляйте `readonly record struct` для безопасного неизменяемого DTO. Обычный `record` (без `struct`) — это ссылочный тип.

**ref struct.** Специальная категория — `ref struct` (например, `Span<T>`, `ReadOnlySpan<T>`). Такой тип гарантированно живёт только на стеке: компилятор запрещает класть его в кучу, в поле класса, в массив, боксировать, делать полем `async`-метода или захватывать в лямбду. Цена за гарантию — жёсткие ограничения; выигрыш — нулевые аллокации и безопасная работа с памятью. Используйте `ref struct` редко и осознанно, в основном при написании высокопроизводительных утилит работы с буферами.

Итог: `class` — по умолчанию для большинства типов; `struct` — для маленьких неизменяемых значений; `record struct` — когда нужны value-семантика и удобство record; `ref struct` — для стековых низкоуровневых абстракций.

#### Theory (EN)

C# offers two primary ways to define your own data types: `struct` and `class`. The difference is not really syntactic — it is about **where the value lives** and **how it is copied**. This distinction underpins both performance and correctness in many real programs.

A `struct` is a **value type**. When you declare a variable of a `struct` type, the value itself is stored inline in that variable. Assigning one variable to another copies the entire value, field by field. Local `struct` values typically live on the **stack** — a fast memory region that is reclaimed automatically when a method returns. Analogy: a `struct` is like a paper notebook. To share your notes with a colleague you copy them by hand into their notebook. Now each person owns a separate copy; editing one never touches the other.

A `class` is a **reference type**. The variable holds not the data but a **reference** to an object allocated on the **heap** — managed memory freed by the garbage collector. On assignment, only the reference is copied; both names point to the same object. Analogy: a `class` is like a shared Google Doc. You hand your colleague a link — both of you look at the same file, edits are visible to everyone.

**When to choose struct?** Microsoft's heuristic is straightforward: a `struct` is appropriate when the type is **small** (typically ≤ 16 bytes), **logically indivisible** (a coordinate, a color, money, a complex number), and ideally **immutable**. Large mutable `struct`s cause pain: every assignment copies dozens of bytes, mutation through a property silently does not work (`myPoint.X = 5` will not compile if `myPoint` is a value returned by a method), and boxing to `object` quietly allocates on the heap. `class` is better for large, mutable, "identity-bearing" objects — domain entities, collections, services.

**record struct (C# 10).** Since C# 10 you can declare a `record struct` — a value type with record syntax: automatic value-based `Equals`/`GetHashCode`, `with` expressions, deconstruction, and `ToString`. It combines the speed of a `struct` with the ergonomics of a record. Declare `readonly record struct` for safe immutable DTOs. A plain `record` (without `struct`) is a reference type.

**ref struct.** A special category is `ref struct` (for example `Span<T>`, `ReadOnlySpan<T>`). Such a type is guaranteed to live only on the stack: the compiler forbids boxing it, putting it on the heap, storing it in a class field or array, making it a field of an `async` method, or capturing it in a lambda. The price is hard restrictions; the payoff is zero allocations and safe low-level memory access. Use `ref struct` rarely and deliberately, mostly when writing high-performance buffer utilities.

Summary: `class` is the default for most types; `struct` for small immutable values; `record struct` when you want value semantics plus record ergonomics; `ref struct` for stack-only low-level abstractions.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — struct vs class vs record struct
using System;
using System.Numerics;

// 1) class — ссылочный тип, живёт в куче / reference type, lives on the heap
public class PlayerClass
{
    public string Name { get; set; }      // имя игрока / player name
    public int Score { get; set; }        // счёт / score
}

// 2) struct — тип-значение, копируется целиком / value type, copied wholesale
public struct PointStruct
{
    public int X { get; init; }           // координата X / X coordinate
    public int Y { get; init; }           // координата Y / Y coordinate

    public double DistanceTo(PointStruct other) =>
        Math.Sqrt(Math.Pow(other.X - X, 2) + Math.Pow(other.Y - Y, 2));
}

// 3) readonly record struct (C# 10) — неизменяемое значение + удобство record
//    immutable value + record ergonomics (Equals, with, deconstruct, ToString)
public readonly record struct Money(decimal Amount, string Currency)
{
    public Money Add(Money other) =>
        Currency == other.Currency
            ? this with { Amount = Amount + other.Amount }   // новый экземпляр / new instance
            : throw new InvalidOperationException("Валюты должны совпадать / currencies must match");
}

// 4) ref struct — только стек, без бокса и кучи / stack-only, no boxing, no heap
public ref struct SpanWalker
{
    private ReadOnlySpan<byte> _data;     // срез байтов / byte slice
    private int _pos;                     // текущая позиция / current position

    public SpanWalker(ReadOnlySpan<byte> data) { _data = data; _pos = 0; }

    public bool HasMore => _pos < _data.Length;

    public byte ReadByte() => _data[_pos++];   // читать и сдвинуться / read and advance
}

public static class Demo
{
    public static void Run()
    {
        // class: две переменные — один объект / two vars, one object
        var a = new PlayerClass { Name = "Alice", Score = 10 };
        var b = a;                          // копируется ссылка / copy the reference
        b.Score = 99;
        Console.WriteLine(a.Score);         // 99 — тот же объект / same object

        // struct: две переменные — два значения / two vars, two values
        var p1 = new PointStruct { X = 0, Y = 0 };
        var p2 = p1;                        // копируется значение / copy the value
        // p2.X = 5;                        // p2 — копия, p1 не меняется / p2 is a copy, p1 untouched
        Console.WriteLine(p1.X);            // 0

        // record struct: value-семантика + with-выражение / value semantics + with
        var m1 = new Money(10m, "RUB");
        var m2 = m1 with { Amount = 20m };  // новый экземпляр / new instance
        Console.WriteLine(m1 == m2);        // False — сравнение по значению / value equality
        Console.WriteLine(m1);              // Money { Amount = 10, Currency = RUB }

        // ref struct: работаем со стековым буфером / stack-only buffer
        byte[] buffer = { 1, 2, 3, 4 };
        var walker = new SpanWalker(buffer);
        while (walker.HasMore)
            Console.Write(walker.ReadByte() + " ");   // 1 2 3 4
    }
}
```

#### Best Practices
- RU: По умолчанию используйте `class`. Переходите на `struct` только для маленьких (≤ 16 байт), логически неделимых и желательно неизменяемых значений.
- RU: Предпочитайте `readonly struct` или `readonly record struct` — это снимает проблему скрытых копий при мутации и помогает компилятору оптимизировать.
- RU: Никогда не делайте большие изменяемые `struct` с полями-мутаторами — это ведёт к молчаливым багам при копировании и мутации через свойства.
- RU: Переопределяйте `Equals`/`GetHashCode` для `struct` осознанно: рефлексивное сравнение по умолчанию использует рефлексию и медленное.
- RU: `ref struct` применяйте только в низкоуровневых утилитах (буферы, парсеры), где критичны аллокации; не выставляйте его в публичный API общего назначения.
- EN: Default to `class`. Reach for `struct` only for small (≤ 16 bytes), logically indivisible, and ideally immutable values.
- EN: Prefer `readonly struct` or `readonly record struct` — this removes hidden-copy mutation pitfalls and lets the compiler optimize.
- EN: Never build large mutable `struct`s with mutating fields — they cause silent bugs through copying and property mutation.
- EN: Override `Equals`/`GetHashCode` for a `struct` deliberately: the default reflective comparison is slow.
- EN: Reserve `ref struct` for low-level utilities (buffers, parsers) where allocations are critical; do not expose it in general-purpose public APIs.

#### Частые ошибки / Common Mistakes
- RU: Изменение свойства у `struct`, возвращённого методом (`GetPoint().X = 5;`) → компилятор не даст присвоить; избегайте изменяемых `struct`, используйте `with` или возвращайте новое значение.
- RU: Боксинг `struct` в `object` или в `List<object>` → незаметная аллокация в куче; используйте `List<PointStruct>` вместо `ArrayList`/`List<object>`.
- RU: Большой `struct` с десятком полей копируется при каждом присваивании и передаче в метод → сделайте его `class` или уменьшите размер.
- RU: Рекурсивная структура `struct Node { Node Next; }` → компилятор выдаст ошибку цикла; для таких графов нужен `class`.
- RU: Захват `ref struct` в лямбду или возврат из `async`-метода → компилятор запретит; перепроектируйте без `ref struct`.
- EN: Mutating a property of a `struct` returned by a method (`GetPoint().X = 5;`) → the compiler rejects the assignment; avoid mutable `struct`, use `with` or return a new value.
- EN: Boxing a `struct` into `object` or a `List<object>` → silent heap allocation; use `List<PointStruct>` instead of `ArrayList`/`List<object>`.
- EN: A large `struct` with many fields is copied on every assignment and method call → make it a `class` or shrink it.
- EN: A recursive `struct Node { Node Next; }` → compiler cycle error; use `class` for such graphs.
- EN: Capturing a `ref struct` in a lambda or returning it from an `async` method → compiler forbids it; redesign without `ref struct`.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] RU: Я могу объяснить, чем тип-значение отличается от ссылочного типа, и где живёт каждый из них.
- [ ] RU: Я выбираю `struct` только для маленьких, неделимых, желательно неизменяемых значений.
- [ ] RU: Я понимаю, что `record struct` (C# 10) даёт value-семантику + удобство record, а `readonly record struct` — неизменяемость.
- [ ] RU: Я знаю ограничения `ref struct` (нельзя боксировать, класть в кучу/поле класса, захватывать в лямбду/`async`).
- [ ] RU: Я избегаю изменяемых больших `struct` и знаю про боксинг при приведении к `object`.
- [ ] EN: I can explain the difference between value and reference types and where each lives.
- [ ] EN: I choose `struct` only for small, indivisible, ideally immutable values.
- [ ] EN: I understand that `record struct` (C# 10) gives value semantics + record ergonomics, and `readonly record struct` immutability.
- [ ] EN: I know the limits of `ref struct` (no boxing, no heap/class field, no lambda/`async` capture).
- [ ] EN: I avoid large mutable `struct`s and am aware of boxing when casting to `object`.

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/struct](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/struct)
- [Struct types (C# reference) — https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/struct](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/struct)
- [Records (C# reference) — https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/record](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/record)
- [ref struct (C# reference) — https://learn.microsoft.com/dotnet/csharp/language-reference/language-specification/structs#148-ref-structs](https://learn.microsoft.com/dotnet/csharp/language-reference/language-specification/structs)

---

[⬆ К модулю M04](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
