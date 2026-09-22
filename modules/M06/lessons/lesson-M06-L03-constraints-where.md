[⬆ К модулю M06](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M06-L03: Ограничения where (class/struct/new/interface) / Constraints where (class/struct/new/interface)

**Модуль / Module:** M06
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Обобщённые типы (generics) в C# по умолчанию принимают **любой** тип в качестве аргумента `T`. Это даёт максимальную гибкость, но часто мешает: компилятор ничего не знает о `T`, поэтому нельзя вызывать методы, обращаться к свойствам, сравнивать значения или создавать экземпляры. Ограничения `where` сужают множество допустимых типов и одновременно **расширяют** возможности компилятора внутри метода или класса.

Представьте `T` как посылку, которую доставляет курьерская служба. Без ограничений вы получаете sealed-коробку: открыть нельзя, взвесить нельзя,甚至连 тип содержимого неизвестен. Ограничение `where` — это таможенная декларация, которая говорит: «внутри только reference-тип» или «там обязательно есть публичный конструктор без параметров». Декларация сужает список грузоотправителей, зато открывает доступ к нужным операциям.

Основные виды ограничений:

1. **`where T : class`** — ссылочный тип (class, interface, delegate, array, string). Гарантирует, что `T` никогда не будет значимым типом, поэтому переменные `T` можно сравнивать с `null` и использовать операторы `as`/`is`. Применимо для репозиториев, кэшей и сервисов, хранящих объекты.

2. **`where T : struct`** — значимый тип, не-nullable. Включает примитивы (`int`, `double`), перечисления `enum` и пользовательские `struct`. Исключает `Nullable<T>`. Идеально для расчётов, векторов, координат и `Span`-подобных структур, где важна производительность без аллокаций.

3. **`where T : new()`** — наличие публичного конструктора без параметров. Позволяет написать `new T()`. Должно стоять **последним** в списке ограничений. Удобно для фабрик и активаторов, но не работает с типами вроде `string` (у которого нет дефолтного конструктора) и абстрактными классами.

4. **`where T : <интерфейс>`** — тип обязан реализовать интерфейс (например `IComparable<T>`, `IEnumerable`). Тогда внутри метода можно вызывать методы этого интерфейса. Можно перечислить несколько интерфейсов через запятую.

5. **`where T : <базовый класс>`** — тип обязан быть наследником указанного класса (или самим им). Дает доступ ко всем protected/public членам базового класса. Чтобы запретить дальнейшее наследование от `T`, используют `where T : BaseClass`.

6. **`where T : notnull`** — аргумент типа не может быть nullable reference type (работает в контексте nullable annotations). Полезно в библиотеках с включённым `<Nullable>enable</Nullable>`.

7. **`where T : unmanaged`** — значимый тип, все поля которого тоже не-managed (без ссылок). Используется в `Span<T>`, `MemoryMarshal`, interop и P/Invoke.

8. **`where T : enum`** / **`where T : delegate`** — специальные ограничения для перечислений и делегатов (появились в C# 7.3).

**Комбинации ограничений.** Можно задать несколько ограничений для одного `T`: `where T : class, IComparable<T>, new()`. Порядок важен: `class`/`struct` идут первыми, затем базовый класс или интерфейсы, а `new()` — всегда последним. Для разных параметров пишут несколько секций `where`: `where T1 : class where T2 : struct, new()`.

Особый случай — **`where T : U`**, означающий, что `T` должен быть convertible к `U` (T — тот же тип, наследник или реализация). Это ограничение «conversion type» помогает в методах вроде `Cast<T>`.

В C# 8+ и 10+ добавились уточнения для nullable-контекста: `where T : class?` допускает nullable ссылочный тип, `where T : default` снимает ограничения при наследовании обобщённого класса. Начиная с C# 11 конструктор без параметров можно объявить через `static abstract` в интерфейсах, что частично заменяет `new()`.

Главное правило: **ограничения должны быть минимально достаточными**. Каждое лишнее `where` сужает переиспользование. Если методу нужно только сравнивать элементы — достаточно `IComparable<T>`, а не `class` + `new()`.

#### Theory (EN)

Generics in C# by default accept **any** type as the type argument `T`. That maximizes flexibility but is often limiting: because the compiler knows nothing about `T`, you cannot call methods, access properties, compare values, or instantiate it. The `where` constraint narrows the set of allowed types and at the same time **broadens** what the compiler lets you do inside the method or class.

Think of `T` as a package delivered by a courier. Without constraints you get a sealed box: you cannot open it, cannot weigh it, even the type of contents is unknown. A `where` constraint is a customs declaration that says "only reference types inside" or "a public parameterless constructor is guaranteed". The declaration shrinks the list of senders but unlocks the operations you need.

The main kinds of constraints:

1. **`where T : class`** — a reference type (class, interface, delegate, array, string). Guarantees that `T` is never a value type, so variables of type `T` can be compared to `null` and used with `as`/`is`. Great for repositories, caches and services that store objects.

2. **`where T : struct`** — a non-nullable value type. Includes primitives (`int`, `double`), enums and user-defined `struct`. Excludes `Nullable<T>`. Ideal for math, vectors, coordinates and `Span`-like structures where allocation-free performance matters.

3. **`where T : new()`** — a public parameterless constructor is present, so `new T()` compiles. Must appear **last** in the constraint list. Handy for factories and activators, but does not work with types like `string` (no default ctor) or abstract classes.

4. **`where T : <interface>`** — the type must implement the interface (e.g. `IComparable<T>`, `IEnumerable`). Inside the method you can call members of that interface. Several interfaces can be listed comma-separated.

5. **`where T : <base class>`** — the type must derive from the given class (or be it). Gives access to all protected/public members of the base class. To forbid deriving further from `T`, use `where T : BaseClass`.

6. **`where T : notnull`** — the type argument cannot be a nullable reference type (works under nullable annotation context). Useful in libraries with `<Nullable>enable</Nullable>`.

7. **`where T : unmanaged`** — a value type whose fields are also unmanaged (no references). Used in `Span<T>`, `MemoryMarshal`, interop and P/Invoke.

8. **`where T : enum`** / **`where T : delegate`** — special constraints for enums and delegates, added in C# 7.3.

**Combinations.** Several constraints can apply to the same `T`: `where T : class, IComparable<T>, new()`. Order matters: `class`/`struct` come first, then base class or interfaces, and `new()` always last. For different parameters use several `where` clauses: `where T1 : class where T2 : struct, new()`.

A special case is **`where T : U`**, meaning `T` must be convertible to `U` (same type, derived, or implementing). This "conversion type" constraint helps in methods like `Cast<T>`.

C# 8+ and 10+ added nullable-context refinements: `where T : class?` allows a nullable reference type, `where T : default` relaxes constraints when inheriting a generic class. Starting with C# 11 a parameterless constructor can be declared via `static abstract` members in interfaces, partially replacing `new()`.

The guiding rule: **constraints should be minimally sufficient**. Every extra `where` reduces reuse. If a method only needs to compare items, `IComparable<T>` is enough — do not pile on `class` + `new()`.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Ограничения where / where constraints
using System;
using System.Collections.Generic;

namespace M06L03;

// Репозиторий только для ссылочных типов с идентификатором
// Repository restricted to reference types with an Id
public class Repository<TEntity> where TEntity : class, IEntity, new()
{
    private readonly Dictionary<int, TEntity> _store = new();

    public void Add(int id)
    {
        // new() разрешает создание экземпляра / new() allows instantiation
        var entity = new TEntity { Id = id };
        _store[id] = entity;
    }

    public TEntity? Find(int id) => _store.TryGetValue(id, out var e) ? e : null;
}

public interface IEntity
{
    int Id { get; init; }   // Доступен благодаря ограничению IEntity / Available because of IEntity constraint
}

public sealed class Product : IEntity
{
    public int Id { get; init; }
    public string Name { get; init; } = "";
}

// Значимый тип-вектор: struct + unmanaged для高性能 математики
// Value-type vector: struct + unmanaged for high-perf math
public readonly struct Vector2<T> : IEquatable<Vector2<T>>
    where T : struct, INumber<T>      // INumber<T> из System.Numerics / from System.Numerics
{
    public T X { get; }
    public T Y { get; }

    public Vector2(T x, T y) { X = x; Y = y; }

    public Vector2<T> Add(Vector2<T> other) => new(X + other.X, Y + other.Y);

    public bool Equals(Vector2<T> other) => X == other.X && Y == other.Y;
    public override bool Equals(object? obj) => obj is Vector2<T> v && Equals(v);
    public override int GetHashCode() => HashCode.Combine(X, Y);
}

// Универсальный минимальный интерфейс System.Numerics.INumber<T> доступен в .NET 7+.
// System.Numerics.INumber<T> is available in .NET 7+.

// Сравниватель с ограничением по интерфейсу IComparable<T>
// Comparer constrained by the IComparable<T> interface
public static class Sorter
{
    public static T Max<T>(T a, T b) where T : IComparable<T>
        => a.CompareTo(b) >= 0 ? a : b;
}

// notnull — запрещаем nullable ссылочные типы в nullable-контексте
// notnull — forbid nullable reference types under nullable context
#nullable enable
public class Cache<TKey, TValue> where TKey : notnull
{
    private readonly Dictionary<TKey, TValue> _items = new();
    public void Set(TKey key, TValue value) => _items[key] = value;
    public TValue? Get(TKey key) => _items.TryGetValue(key, out var v) ? v : default;
}

// Комбинация нескольких where для разных параметров
// Multiple where clauses for different parameters
public class Converter<TSource, TTarget>
    where TSource : class
    where TTarget : class, new()
{
    public TTarget Convert(TSource source, Func<TSource, TTarget> map)
    {
        ArgumentNullException.ThrowIfNull(source);
        ArgumentNullException.ThrowIfNull(map);
        return map(source);
    }
}

// Демонстрация / Demo
public static class Demo
{
    public static void Run()
    {
        var repo = new Repository<Product>();
        repo.Add(1);
        Console.WriteLine(repo.Find(1)?.Id);          // 1

        var v1 = new Vector2<double>(1.0, 2.0);
        var v2 = new Vector2<double>(3.0, 4.0);
        var v3 = v1.Add(v2);
        Console.WriteLine(v3);                        // Vector2`1

        Console.WriteLine(Sorter.Max(3, 7));           // 7
        Console.WriteLine(Sorter.Max("apple", "pear")); // pear

        var cache = new Cache<string, int>();
        cache.Set("answer", 42);
        Console.WriteLine(cache.Get("answer"));       // 42
    }
}
```

#### Best Practices

- Ставьте минимально достаточные ограничения: если нужен только `IComparable<T>`, не добавляйте `class` и `new()`.
- Используйте `struct`/`unmanaged` для расчётных структур, чтобы избежать boxing и аллокаций.
- Предпочитайте интерфейсные ограничения базовому классу — это сохраняет гибкость наследования.
- Для nullable-контекста применяйте `notnull` в публичных API библиотек.
- Документируйте в XML-комментариях, **почему** наложено ограничение.
- Используйте `static abstract` интерфейсы (C# 11+) вместо `new()`, когда нужна типобезопасная фабрика.

- Apply the minimal sufficient constraints: if only `IComparable<T>` is needed, do not add `class` and `new()`.
- Use `struct`/`unmanaged` for computational structures to avoid boxing and allocations.
- Prefer interface constraints over base-class constraints — they keep inheritance flexible.
- In a nullable context, apply `notnull` in the public API of libraries.
- Document in XML comments **why** a constraint exists.
- Prefer `static abstract` interface members (C# 11+) over `new()` when a type-safe factory is needed.

#### Частые ошибки / Common Mistakes

- **`where T : new()` указан не последним** → Ставьте `new()` всегда в конце списка ограничений.
- **Попытка `where T : struct` с `string` или `Nullable<int>`** → `string` — ссылочный тип, `Nullable<T>` запрещён; используйте `class` или снимите `struct`.
- **Лишние ограничения `class + new()` «на всякий случай»** → Уберите всё лишнее; оставьте только то, что реально вызывается в коде.
- **`where T : class` и сравнение `== null` для значимых типов** → Это невозможно по определению; пересмотрите выбор ограничения.
- **Несовместимые комбинации `struct` + `class`** → Они взаимоисключающи; выберите одно.
- **Забыли `IEquatable<T>`/`IComparable<T>` и используете `object.Equals`** → Добавьте ограничение, чтобы избежать boxing и повысить производительность.

- **`where T : new()` not placed last** → Always put `new()` at the end of the constraint list.
- **Trying `where T : struct` with `string` or `Nullable<int>`** → `string` is a reference type and `Nullable<T>` is forbidden; use `class` or drop `struct`.
- **Adding `class + new()` "just in case"** → Remove the extras; keep only what the code actually calls.
- **`where T : class` and comparing `== null` for value types** → Impossible by definition; rethink the constraint.
- **Incompatible combinations `struct` + `class`** → They are mutually exclusive; pick one.
- **Forgetting `IEquatable<T>`/`IComparable<T>` and relying on `object.Equals`** → Add the constraint to avoid boxing and improve performance.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я выбираю минимально достаточное ограничение, а не максимальное.
- [ ] `new()` стоит последним в списке ограничений.
- [ ] `class` и `struct` не используются одновременно для одного `T`.
- [ ] Для ссылочных типов я учитываю nullable-контекст (`class?`, `notnull`).
- [ ] Интерфейсные ограничения выбраны вместо базового класса там, где важна гибкость.
- [ ] Код компилируется в C# 12 / .NET 8 без warning-ов уровня nullable.
- [ ] XML-комментарии объясняют причину каждого ограничения.
- [ ] `unmanaged` используется осознанно (interop/Span/performance).

- [ ] I pick the minimal sufficient constraint rather than the maximal one.
- [ ] `new()` is placed last in the constraint list.
- [ ] `class` and `struct` are not used together for the same `T`.
- [ ] For reference types I account for the nullable context (`class?`, `notnull`).
- [ ] Interface constraints are chosen over a base class where flexibility matters.
- [ ] The code compiles on C# 12 / .NET 8 with no nullable-level warnings.
- [ ] XML comments explain the reason for each constraint.
- [ ] `unmanaged` is used deliberately (interop/Span/performance).

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/where-generic-type-constraint](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/where-generic-type-constraint)
- [Generics (C# Programming Guide) — https://learn.microsoft.com/dotnet/csharp/fundamentals/types/generics](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/generics)
- [System.Numerics.INumber<T> — https://learn.microsoft.com/dotnet/api/system.numerics.inumber-1](https://learn.microsoft.com/dotnet/api/system.numerics.inumber-1)

---

[⬆ К модулю M06](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
