[⬆ К модулю M06](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M06-L02: Обобщённые классы и методы / Generic classes and methods

**Модуль / Module:** M06
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Обобщения (generics) — это механизм C#, который позволяет написать код один раз и применять его к разным типам данных без потери типобезопасности и без накладных расходов на упаковку (boxing). Представьте себе складской стеллаж: сами полки одинаковые, но на них можно класть книги, инструменты или посуду. Обобщённый класс — это и есть «стеллаж», а параметр типа `<T>` — это метка, говорящая, что именно на полке хранится.

**Объявление обобщённого класса.** Синтаксис прост: после имени класса в угловых скобках указывается один или несколько параметров-типов. По соглашению параметр называется `T`, но это не обязательно.

```csharp
public class Repo<T>
{
    private readonly List<T> _items = new();
    public void Add(T item) => _items.Add(item);
    public T Get(int index) => _items[index];
}
```

Здесь `T` — «заглушка» для любого типа. Когда мы пишем `Repo<User>` или `Repo<Order>`, компилятор создаёт строго типизированную версию: `Add` принимает только `User`, а `Get` возвращает `User`. Попытка подсунуть строку в `Repo<User>` вызовет ошибку компиляции, а не исключение во время выполнения.

**Обобщённые методы.** Обобщения можно применять не ко всему классу, а к отдельному методу. Это удобно, когда тип-параметр нужен только внутри одной операции. Классический пример — фабричный метод или утилита сравнения.

```csharp
public static T First<T>(IList<T> list) => list[0];
```

При вызове `First(users)` компилятор сам определит, что `T` — это `User`.

**Вывод типов (type inference).** В большинстве случаев параметр-тип можно не указывать явно — компилятор выведет его из аргументов. Это делает код чище. Однако при неоднозначных ситуациях (например, пустой список) тип приходится задавать вручную: `First<string>(new List<string>())`.

**Несколько параметров-типов.** Обобщения поддерживают произвольное число параметров. Самый распространённый пример — словарь с ключом и значением:

```csharp
public class Map<TKey, TValue>
{
    private readonly Dictionary<TKey, TValue> _data = new();
    public void Put(TKey key, TValue value) => _data[key] = value;
    public TValue Get(TKey key) => _data[key];
}
```

По соглашению параметры-типы с особыми ролями получают говорящие имена: `TKey`, `TValue`, `TResult`, `TInput`.

**Ограничения (constraints).** Параметр-тип можно ограничить, чтобы получить доступ к конкретным возможностям. Например, `where T : class` требует ссылочный тип, `where T : new()` требует конструктор без параметров, `where T : IComparable<T>` гарантирует наличие метода `CompareTo`. Без ограничений компилятор знает о `T` только то, что это `object`.

**Почему обобщения важны.** В отличие от `ArrayList` из раннего .NET, где всё хранилось как `object` и требовало упаковки/распаковки, обобщённые коллекции хранят значения напрямую. Для типов-значений (`int`, `struct`) это убирает накладные расходы на boxing и ошибки приведения типов во время выполнения. Для ссылочных типов выигрыш меньше, но типобезопасность остаётся.

**Аналогия.** Обобщённый класс — как конструкторская документация, по которой можно собрать станок под конкретный материал: один чертёж, но станок режет дерево, металл или пластик. Меняется только заготовка, а механизм один.

Запомните главное правило: параметр-тип `T` следует использовать тогда, когда логика класса или метода одинакова для разных типов, а сам тип не важен для поведения — важна лишь типобезопасность.

#### Theory (EN)

Generics are a C# feature that lets you write code once and reuse it across many data types without sacrificing type safety and without the boxing overhead. Imagine a warehouse shelf: the shelf itself is identical, but you can store books, tools, or dishes on it. A generic class is that shelf, and the type parameter `<T>` is the label that says what is stored there.

**Declaring a generic class.** The syntax is simple: after the class name, one or more type parameters are listed in angle brackets. By convention the parameter is named `T`, but this is not mandatory.

```csharp
public class Repo<T>
{
    private readonly List<T> _items = new();
    public void Add(T item) => _items.Add(item);
    public T Get(int index) => _items[index];
}
```

Here `T` is a placeholder for any type. When you write `Repo<User>` or `Repo<Order>`, the compiler produces a strongly typed version: `Add` accepts only `User`, and `Get` returns `User`. Trying to pass a string to a `Repo<User>` produces a compile-time error, not a runtime exception.

**Generic methods.** Generics can be applied to a single method rather than the whole class. This is useful when the type parameter is needed only inside one operation. A typical example is a factory method or a comparison helper.

```csharp
public static T First<T>(IList<T> list) => list[0];
```

When you call `First(users)`, the compiler infers that `T` is `User`.

**Type inference.** In most cases you do not need to specify the type parameter explicitly — the compiler derives it from the arguments. This keeps the code clean. However, in ambiguous situations (an empty list, for instance) the type must be supplied manually: `First<string>(new List<string>())`.

**Multiple type parameters.** Generics support an arbitrary number of parameters. The most common example is a dictionary with a key and a value:

```csharp
public class Map<TKey, TValue>
{
    private readonly Dictionary<TKey, TValue> _data = new();
    public void Put(TKey key, TValue value) => _data[key] = value;
    public TValue Get(TKey key) => _data[key];
}
```

By convention, type parameters with a specific role get descriptive names: `TKey`, `TValue`, `TResult`, `TInput`.

**Constraints.** A type parameter can be constrained to gain access to specific capabilities. For example, `where T : class` requires a reference type, `where T : new()` requires a parameterless constructor, `where T : IComparable<T>` guarantees a `CompareTo` method. Without constraints the compiler knows only that `T` is an `object`.

**Why generics matter.** Unlike the early .NET `ArrayList`, where everything was stored as `object` and required boxing/unboxing, generic collections store values directly. For value types (`int`, `struct`) this removes boxing overhead and runtime cast errors. For reference types the gain is smaller, but type safety remains.

**Analogy.** A generic class is like engineering documentation from which you can build a machine for a specific material: one blueprint, but the machine cuts wood, metal, or plastic. Only the workpiece changes; the mechanism is the same.

Remember the key rule: use a type parameter `T` when the logic of a class or method is identical across types and the type itself does not affect behavior — only type safety matters.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Обобщённые классы, методы, вывод типов, несколько параметров-типов
// C# 12 / .NET 8 — Generic classes, methods, type inference, multiple type parameters

using System;
using System.Collections.Generic;

// Обобщённый класс-репозиторий / Generic repository class
// T — параметр типа, заменяется конкретным типом при использовании
// T — type parameter, replaced by a concrete type on use
public class Repo<T>
{
    private readonly List<T> _items = new();

    public void Add(T item)        // принимает только T / accepts only T
    {
        _items.Add(item);
    }

    public T Get(int index)        // возвращает T / returns T
    {
        return _items[index];
    }

    public int Count => _items.Count;
}

// Класс с несколькими параметрами-типов / Class with multiple type parameters
// TKey — тип ключа, TValue — тип значения / TKey — key type, TValue — value type
public class Map<TKey, TValue> where TKey : notnull
{
    private readonly Dictionary<TKey, TValue> _data = new();

    public void Put(TKey key, TValue value) => _data[key] = value;

    public TValue? GetOrDefault(TKey key) =>
        _data.TryGetValue(key, out var value) ? value : default;
}

// Обобщённый метод в обычном (необобщённом) классе
// Generic method inside a non-generic class
public static class Picker
{
    // Вывод типов: компилятор сам определит T по аргументу list
    // Type inference: the compiler derives T from the list argument
    public static T First<T>(IList<T> list)
    {
        if (list.Count == 0)
            throw new InvalidOperationException("Список пуст / List is empty");
        return list[0];
    }

    // Несколько параметров-типов в методе / Multiple type parameters in a method
    public static KeyValuePair<TKey, TValue> Pair<TKey, TValue>(TKey key, TValue value)
        => new(key, value);
}

// Демонстрация / Demo
public class Demo
{
    public static void Run()
    {
        var users = new Repo<string>();           // Repo<string> — T заменён на string
        users.Add("Alice");                       // OK / OK
        users.Add("Bob");
        Console.WriteLine(users.Get(0));          // Alice / Alice
        Console.WriteLine($"Count: {users.Count}"); // Count: 2

        var ages = new Repo<int>();               // Repo<int> — T заменён на int
        ages.Add(30);
        ages.Add(41);
        Console.WriteLine(ages.Get(1));           // 41 / 41

        // Несколько параметров-типов / Multiple type parameters
        var settings = new Map<string, int>();
        settings.Put("timeout", 5000);
        Console.WriteLine(settings.GetOrDefault("timeout"));   // 5000
        Console.WriteLine(settings.GetOrDefault("missing"));  // 0 (default для int / default for int)

        // Вывод типов в обобщённом методе / Type inference in a generic method
        var first = Picker.First(users._itemsRepresentation()); // вывод T = string / infers T = string

        // Явное указание типа при неоднозначности / Explicit type when ambiguous
        var empty = Picker.First<string>(new List<string>());

        // Метод с двумя параметрами-типов / Method with two type parameters
        var kv = Picker.Pair("id", 42);           // Pair<string,int> выведен / inferred
        Console.WriteLine($"{kv.Key}={kv.Value}"); // id=42
    }
}

// Вспомогательное представление для демо (в реальном коде используйте публичный геттер)
// Helper representation for the demo (use a public getter in real code)
public static class RepoExtensions
{
    public static IList<T> _itemsRepresentation<T>(this Repo<T> repo)
    {
        // Возвращает внутренний список для демонстрации вывода типов
        // Returns the internal list to demonstrate type inference
        // В реальном проекте лучше暴露 expose публичный метод, а не внутреннее состояние
        return new List<T>();
    }
}
```

#### Best Practices

- Называйте параметры-типов осмысленно для роли: `T` для общего случая, `TKey`/`TValue` для словарей, `TResult` для результатов.
- Применяйте ограничения (`where`) точечно: они дают доступ к возможностям типа и документируют контракт, но чрезмерные ограничения снижают переиспользуемость.
- Предпочитайте обобщённые коллекции (`List<T>`, `Dictionary<TKey,TValue>`) неуниверсальным (`ArrayList`, `Hashtable`), чтобы избежать boxing и ошибок приведения.
- Не используйте обобщения там, где работает обычный полиморфизм через интерфейс или базовый класс — это упрощает код.
- Держите обобщённый класс сфокусированным на одной ответственности; не добавляйте параметры-типы «на всякий случай».

- Name type parameters meaningfully by role: `T` for the general case, `TKey`/`TValue` for dictionaries, `TResult` for results.
- Apply constraints (`where`) sparingly: they unlock type capabilities and document the contract, but excessive constraints reduce reuse.
- Prefer generic collections (`List<T>`, `Dictionary<TKey,TValue>`) over non-generic ones (`ArrayList`, `Hashtable`) to avoid boxing and cast errors.
- Do not use generics where ordinary polymorphism via an interface or base class is enough — it keeps the code simpler.
- Keep a generic class focused on a single responsibility; do not add type parameters "just in case".

#### Частые ошибки / Common Mistakes

- Использование `object` вместо `T` «для универсальности» → теряется типобезопасность и появляется boxing. Используйте параметр-тип.
- Путаница между статическими полями в обобщённом классе → каждое закрытие `Repo<int>` и `Repo<string>` получает отдельную копию статического поля; помните об этом при кэшировании.
- Пустой список и вывод типов → `Picker.First(new List<int>())` выводит `int`, но `Picker.First(new List<int>())` может быть неоднозначно с `IList`; явно укажите тип при сомнениях.
- Слишком жёсткие ограничения вроде `where T : class, new(), IComparable<T>, IDisposable` → почти ничего не подойдёт. Ослабьте до необходимого.
- Изменение `T` через `var` без понимания → `var x = repo.Get(0)` выводит `T`, но при `default(T)` для ссылочного типа получится `null`. Проверяйте `null`.

- Using `object` instead of `T` "for flexibility" → you lose type safety and get boxing. Use a type parameter.
- Static field confusion in a generic class → each closed type `Repo<int>` and `Repo<string>` gets its own copy of a static field; remember this when caching.
- Empty list and type inference → `Picker.First(new List<int>())` infers `int`, but `new List<int>()` against an `IList` parameter may be ambiguous; specify the type explicitly when unsure.
- Overly strict constraints like `where T : class, new(), IComparable<T>, IDisposable` → almost nothing will fit. Relax to what is actually needed.
- Mutating `T` through `var` without understanding → `var x = repo.Get(0)` infers `T`, but `default(T)` for a reference type yields `null`. Always check for `null`.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Класс объявлен как `class Repo<T>` с параметром-типом в угловых скобках.
- [ ] Метод `Add` принимает `T`, а `Get` возвращает `T` — типобезопасность соблюдена.
- [ ] Есть обобщённый метод с выводом типов (без явного `<T>` в вызове).
- [ ] Реализован класс с двумя параметрами-типами (`TKey`, `TValue`).
- [ ] Применено хотя бы одно ограничение (`where T : ...`) с обоснованием.
- [ ] Код компилируется под C# 12 / .NET 8 без предупреждений.
- [ ] Использованы осмысленные имена параметров-типов (`TKey`, `TValue`, `TResult`).

- [ ] The class is declared as `class Repo<T>` with a type parameter in angle brackets.
- [ ] `Add` accepts `T` and `Get` returns `T` — type safety is preserved.
- [ ] There is a generic method that uses type inference (no explicit `<T>` at the call site).
- [ ] A class with two type parameters (`TKey`, `TValue`) is implemented.
- [ ] At least one constraint (`where T : ...`) is applied with justification.
- [ ] The code compiles under C# 12 / .NET 8 with no warnings.
- [ ] Meaningful type-parameter names are used (`TKey`, `TValue`, `TResult`).

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/fundamentals/types/generics](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/generics)

---

[⬆ К модулю M06](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
