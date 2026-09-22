[⬆ К модулю M06](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M06-L04: Dictionary<K,V>, HashSet<T>, равенство и GetHashCode / Dictionary<K,V>, HashSet<T>, equality and GetHashCode

**Модуль / Module:** M06
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

`Dictionary<TKey, TValue>` и `HashSet<T>` — это хеш-таблицы. Они дают доступ к элементам в среднем за **O(1)**, потому что вычисляют положение элемента прямо из его содержимого, а не перебирают всё подряд. Представьте себе огромную библиотеку, где номер полки вычисляется из первой буквы фамилии автора: чтобы найти книгу Достоевского, вам не нужно обходить все полки — вы сразу идёте к секции «Д». `Dictionary` хранит пары «ключ → значение», а `HashSet` — просто уникальные значения без привязки к данным.

Внутри работает связка из двух методов: `GetHashCode()` и `Equals()`. Когда вы кладёте ключ в таблицу, среда вызывает `GetHashCode()` и по хеш-коду вычисляет «корзину» (bucket). Внутри корзины может оказаться несколько элементов — например, при коллизии, когда разные ключи дали одинаковый хеш. Тогда для точного сравнения вызывается `Equals()`. Поэтому работает **контракт**: если `a.Equals(b)` истинно, то `a.GetHashCode()` обязан вернуть то же самое число, что и `b.GetHashCode()`. Обратное неверно — одинаковые хеши могут быть у разных объектов (коллизия), это нормально. Но нарушение прямого правила ломает таблицу: вы положите элемент, а потом не сможете его найти.

По умолчанию для ключей используется `EqualityComparer<TKey>.Default`, который для примитивов и строк сравнивает значения, а для классов — ссылки. Если вы хотите, скажем, сравнивать строки без учёта регистра или использовать свой тип в качестве ключа по значению полей — нужен **кастомный `IEqualityComparer<T>`**. Он задаёт сразу две вещи: как считать хеш и как сравнивать на равенство. В `Dictionary` и `HashSet` comparer передаётся через конструктор.

Особый случай — собственный тип как ключ. Чтобы он корректно работал «из коробки» (без comparer), нужно переопределить `Equals` и `GetHashCode` в самом типе, а ещё лучше — реализовать `IEquatable<T>` для типобезопасного сравнения без боксинга. Хеш-код должен быть **стабильным**: один и тот же объект обязан возвращать один и тот же хеш, пока жив. И **неизменным относительно полей, участвующих в равенстве**: если положить объект в `Dictionary`, а потом изменить поле, влияющее на хеш, — ключ «потеряется» в корзине, и достать его будет невозможно. Поэтому ключи делают либо неизменяемыми (record, readonly struct), либо осторожно помечают поля `readonly`.

Мутирующий ключ — источник самых коварных багов в хеш-таблицах: утечки памяти в `HashSet`, «пропавшие» значения в `Dictionary`, бесконечные циклы при переборе. Простое правило: **ключ в таблице — константа**. Если нужно менять данные — выносите их в `TValue`, а ключ оставьте стабильным.

#### Theory (EN)

`Dictionary<TKey, TValue>` and `HashSet<T>` are hash tables. They give you **O(1)** average access, because they compute where an element lives from its content rather than scanning everything. Imagine a huge library where the shelf number is derived from the first letter of the author's surname: to find Dostoevsky you do not walk every aisle — you go straight to the «D» section. `Dictionary` stores key → value pairs, while `HashSet` stores just unique values with no payload attached.

Internally two methods drive everything: `GetHashCode()` and `Equals()`. When you insert a key, the runtime calls `GetHashCode()` and uses the hash to pick a «bucket». A bucket may hold several entries — for example when different keys collide and produce the same hash. Then `Equals()` is called for precise comparison. This gives rise to the **contract**: if `a.Equals(b)` is true, then `a.GetHashCode()` must return the same number as `b.GetHashCode()`. The reverse does not hold — different objects may share a hash (a collision), which is fine. But breaking the direct rule breaks the table: you insert an element and then cannot find it.

By default, keys use `EqualityComparer<TKey>.Default`, which compares values for primitives and strings and references for classes. If you want, say, case-insensitive string comparison or your own type as a key by field values — you need a **custom `IEqualityComparer<T>`**. It defines two things at once: how to compute the hash and how to compare for equality. In `Dictionary` and `HashSet` the comparer is passed through the constructor.

A special case is your own type used as a key. To make it work out of the box (without a comparer) you override `Equals` and `GetHashCode` in the type itself, and ideally implement `IEquatable<T>` for type-safe, boxing-free comparison. The hash code must be **stable**: the same object must keep returning the same hash while it lives. And **immutable with respect to the fields used in equality**: if you put an object into a `Dictionary` and then mutate a field that affects the hash, the key gets «lost» in its bucket and you can never retrieve it. So keys are made either immutable (record, readonly struct) or carefully marked `readonly`.

A mutating key is the source of the nastiest bugs in hash tables: memory leaks in `HashSet`, «vanished» values in `Dictionary`, infinite loops during enumeration. The simple rule is: **a key in a table is a constant**. If you need to mutate data, move it into `TValue` and keep the key stable.

#### Пример кода / Code Example

```csharp
using System;
using System.Collections.Generic;

// Кастомный тип-ключ: неизменяемый, реализует IEquatable<T> / Custom key type: immutable, implements IEquatable<T>
public readonly struct Point : IEquatable<Point>
{
    public int X { get; }
    public int Y { get; }

    public Point(int x, int y) { X = x; Y = y; }

    // Типобезопасное сравнение без боксинга / Type-safe comparison without boxing
    public bool Equals(Point other) => X == other.X && Y == other.Y;

    public override bool Equals(object? obj) => obj is Point p && Equals(p);

    // Контракт: равные объекты дают равный хэш / Contract: equal objects produce equal hash
    public override int GetHashCode() => HashCode.Combine(X, Y);

    public static bool operator ==(Point a, Point b) => a.Equals(b);
    public static bool operator !=(Point a, Point b) => !a.Equals(b);
}

// Кастомный компарер строк без учёта регистра / Case-insensitive string comparer
public sealed class CaseInsensitiveComparer : IEqualityComparer<string>
{
    public bool Equals(string? x, string? y)
        => string.Equals(x, y, StringComparison.OrdinalIgnoreCase);

    public int GetHashCode(string obj)
        => StringComparer.OrdinalIgnoreCase.GetHashCode(obj);
}

class Program
{
    static void Main()
    {
        // Dictionary O(1): ключ → значение / Dictionary O(1): key → value
        var cache = new Dictionary<Point, string>();
        cache[new Point(1, 2)] = "A";   // RU: добавление  EN: add
        cache[new Point(3, 4)] = "B";

        if (cache.TryGetValue(new Point(1, 2), out var label))
            Console.WriteLine($"Найдено / Found: {label}"); // → A

        // HashSet: только уникальные значения / HashSet: unique values only
        var seen = new HashSet<string>(new CaseInsensitiveComparer());
        seen.Add("Apple");
        seen.Add("APPLE");   // RU: дубликат игнорируется  EN: duplicate ignored
        Console.WriteLine($"Уникальных / Unique: {seen.Count}"); // → 1

        // Готовый компарер из BCL / Built-in comparer from BCL
        var ci = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase)
        {
            ["Moscow"] = 1,
            ["moscow"] = 2   // RU: тот же ключ  EN: same key
        };
        Console.WriteLine($"mOsCoW => {ci["mOsCoW"]}"); // → 2 (перезаписано / overwritten)
    }
}
```

#### Best Practices

- Делайте ключи неизменяемыми: используйте `record` или `readonly struct`, чтобы хеш не менялся после добавления в таблицу.
- Реализуйте `IEquatable<T>` для своих типов-ключей — это убирает боксинг и ускоряет сравнение значимых типов.
- Считайте хеш через `HashCode.Combine(...)` — он устойчив к коллизиям лучше, чем ручной `X * 31 + Y`.
- Передавайте `IEqualityComparer<T>` (например, `StringComparer.OrdinalIgnoreCase`) в конструктор вместо написания «ручных» проверок по всему коду.
- Задавайте начальную ёмкость (`new Dictionary<K,V>(capacity)`), когда размер известен заранее — это избегает лишних перехеширований.

#### Best Practices

- Make keys immutable: use `record` or `readonly struct` so the hash never changes after insertion.
- Implement `IEquatable<T>` for custom key types — it removes boxing and speeds up equality checks for value types.
- Compute hashes with `HashCode.Combine(...)` — it resists collisions better than a hand-rolled `X * 31 + Y`.
- Pass an `IEqualityComparer<T>` (e.g. `StringComparer.OrdinalIgnoreCase`) through the constructor instead of ad-hoc checks scattered across the code.
- Pre-size the collection (`new Dictionary<K,V>(capacity)`) when the count is known up front — it avoids repeated rehashing.

#### Частые ошибки / Common Mistakes

- [Ключ-класс с изменяемым полем после вставки в `Dictionary`] → [Объявляйте ключи неизменяемыми (`record`, `readonly struct`) или помечайте поля `readonly`.] (RU)
- [Переопределён `Equals`, но не `GetHashCode` (или наоборот)] → [Всегда переопределяйте оба вместе: нарушение контракта ломает поиск в хеш-таблице.] (RU)
- [Хеш зависит от изменяемого поля] → [Включайте в `GetHashCode` только поля, которые участвуют в `Equals` и не меняются.] (RU)
- [Сравнение строк с учётом регистра там, где нужен регистронезависимый ключ] → [Передавайте `StringComparer.OrdinalIgnoreCase` в конструктор `Dictionary`/`HashSet`.] (RU)
- [Ожидание O(1) при плохом comparer, который возвращает константный хеш] → [Распределяйте хеш равномерно; используйте `HashCode.Combine`, а не `return 0`.] (RU)

- [Mutable class key inserted into a `Dictionary` and then mutated] → [Declare keys immutable (`record`, `readonly struct`) or mark fields `readonly`.] (EN)
- [Overriding `Equals` but not `GetHashCode` (or vice versa)] → [Always override both together: breaking the contract breaks hash-table lookup.] (EN)
- [Hash depends on a mutable field] → [Include in `GetHashCode` only fields that take part in `Equals` and never change.] (EN)
- [Case-sensitive string comparison where a case-insensitive key is needed] → [Pass `StringComparer.OrdinalIgnoreCase` to the `Dictionary`/`HashSet` constructor.] (EN)
- [Expecting O(1) with a poor comparer that returns a constant hash] → [Distribute the hash evenly; use `HashCode.Combine`, never `return 0`.] (EN)

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Для своего типа-ключа переопределены и `Equals`, и `GetHashCode` согласованно. (RU)
- [ ] Ключи неизменяемы — после добавления в таблицу поля, влияющие на хеш, не меняются. (RU)
- [ ] Хеш вычисляется через `HashCode.Combine` только из полей, входящих в `Equals`. (RU)
- [ ] Реализован `IEquatable<T>` для значимых типов-ключей (без боксинга). (RU)
- [ ] Для регистронезависимых строковых ключей используется `StringComparer.OrdinalIgnoreCase`. (RU)
- [ ] Понимаю, что O(1) — среднее время, а худший случай (коллизии) даёт O(n). (RU)

- [ ] For a custom key type, both `Equals` and `GetHashCode` are overridden consistently. (EN)
- [ ] Keys are immutable — fields affecting the hash do not change after insertion. (EN)
- [ ] The hash is computed with `HashCode.Combine` from the same fields used in `Equals`. (EN)
- [ ] `IEquatable<T>` is implemented for value-type keys (no boxing). (EN)
- [ ] Case-insensitive string keys use `StringComparer.OrdinalIgnoreCase`. (EN)
- [ ] I understand that O(1) is average time; the worst case (collisions) is O(n). (EN)

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.collections.generic.dictionary-2]

---

[⬆ К модулю M06](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
