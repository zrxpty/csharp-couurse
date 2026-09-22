[⬆ К модулю M04](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M04-L02: Поля, свойства (auto-properties, init-only, required) / Fields, properties (auto-properties, init-only, required)

**Модуль / Module:** M04
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

В C# данные класса хранятся в **полях** (fields), а доступ к ним чаще всего открывают через **свойства** (properties). Поле — это просто переменная внутри типа; свойство — это «умный прокси» над данными, у которого есть `get` и `set`-аксессоры. Аналогия: поле — это открытый сейф в коридоре, любой может положить или забрать что угодно. Свойство — это сейф за стойкой администратора: вы передаёте значение, но администратор (аксессор `set`) проверит его, прежде чем положить внутрь.

Исторически разработчики писали громоздкий код: приватное поле `private string _name;` и публичное свойство с ручными `get { return _name; }` и `set { _name = value; }`. Это «backing field» (опорное поле). Такой шаблон даёт полный контроль: можно добавить валидацию в `set`, логирование, ленивые вычисления в `get`. Но для тривиальных случаев это лишний шум.

С C# 3.0 появились **автосвойства** (auto-properties): `public string Name { get; set; }`. Компилятор сам создаёт скрытое backing-поле. Код становится чистым, а контроль над инкапсуляцией сохраняется. С C# 6 автосвойства можно инициализировать прямо в объявлении: `public List<int> Items { get; } = new();`, а также делать их только для чтения без `private set`.

Обычный `set` позволяет менять свойство в любой момент жизни объекта. Но часто объект должен быть **неизменяемым** (immutable) после создания — это упрощает многопоточность и рассуждения о состоянии. C# 9 ввёл **init-only**-аксессор: `public string Name { get; init; }`. Такое свойство можно задать только во время инициализации объекта (в объектном инициализаторе `new Person { Name = "X" }` или в конструкторе), но потом изменить нельзя. Это идеальный баланс между удобством синтаксиса инициализатора и неизменяемостью.

Иногда нужно гарантировать, что свойство **обязательно** задано при создании — иначе объект будет в невалидном состоянии. Раньше это достигалось только конструктором с параметрами. C# 11 добавил модификатор `required`: `public required string Name { get; init; }`. Компилятор выдаст ошибку, если в инициализаторе объекта не указать все `required`-свойства. Это работает вместе с атрибутом `[SetsRequiredMembers]` для конструкторов, которые присваивают все обязательные поля.

Свойства могут быть **expression-bodied** (в виде выражения): `public string FullName => $"{First} {Last}";` — компактный `get` без тела. То же работает для `set`: `set => field = value;`, но чаще применяется к вычисляемым свойствам только для чтения.

**Валидация в set.** Внутри `set` можно проверять `value` и бросать `ArgumentException`. Например: `set { if (value < 0) throw new...; _age = value; }`. Начиная с C# 13/.NET 9 можно использовать ключевое слово `field` (semi-auto properties) прямо в теле свойства без явного backing-field, но в C# 12/.NET 8 его ещё нет — поэтому в этом уроке используем классический backing-field, где нужна валидация, или автосвойства, где не нужна.

Главное правило инкапсуляции: **публичные поля — это плохая практика**. Они не дают ни валидации, ни возможности позже изменить реализацию, ни бинарной совместимости. Свойства же можно эволюционировать: начать с автосвойства, а позже заменить на backing-field с проверкой, не ломая вызывающий код. Поэтому по умолчанию выбирайте автосвойства, для неизменяемых данных — `init`, для обязательных — `required`, а валидацию добавляйте через backing-field.

#### Theory (EN)

In C#, a type's data lives in **fields**, and access to that data is usually exposed through **properties**. A field is simply a variable declared inside a type; a property is a "smart proxy" over data that exposes `get` and `set` accessors. Analogy: a field is an open safe in a hallway — anyone can put in or take out anything. A property is a safe behind a reception desk: you hand a value to the receptionist (the `set` accessor), who validates it before placing it inside.

Historically developers wrote verbose code: a private field `private string _name;` plus a public property with manual `get { return _name; }` and `set { _name = value; }`. That hidden variable is the **backing field**. This pattern gives full control: you can add validation in `set`, logging, lazy computation in `get`. But for trivial cases it is needless noise.

C# 3.0 introduced **auto-properties**: `public string Name { get; set; }`. The compiler synthesizes a hidden backing field automatically. The code stays clean while encapsulation remains intact. Since C# 6, auto-properties can be initialized inline: `public List<int> Items { get; } = new();`, and read-only auto-properties no longer require a `private set`.

A regular `set` lets a property be changed at any point in the object's lifetime. But often an object should be **immutable** after construction — that simplifies concurrency and reasoning about state. C# 9 introduced the **init-only** accessor: `public string Name { get; init; }`. Such a property can be assigned only during object initialization (in an object initializer `new Person { Name = "X" }` or in a constructor) and never afterwards. This is the sweet spot between the convenience of initializer syntax and immutability.

Sometimes you must guarantee that a property is **required** at construction — otherwise the object is in an invalid state. Previously only a constructor with parameters could enforce this. C# 11 added the `required` modifier: `public required string Name { get; init; }`. The compiler emits an error if an object initializer does not set every `required` property. This pairs with the `[SetsRequiredMembers]` attribute for constructors that assign all required fields.

Properties can be **expression-bodied**: `public string FullName => $"{First} {Last}";` — a compact `get` without a body block. The same syntax works for `set`: `set => field = value;`, though it is most common on computed read-only properties.

**Validation in set.** Inside `set` you can inspect `value` and throw `ArgumentException`. For example: `set { if (value < 0) throw new...; _age = value; }`. Starting with C# 13/.NET 9 you can use the `field` keyword (semi-auto properties) directly in a property body without an explicit backing field; in C# 12/.NET 8 it is not yet available, so this lesson uses a classic backing field where validation is needed and auto-properties where it is not.

The core encapsulation rule: **public fields are a bad practice**. They offer no validation, no way to change implementation later, and no binary compatibility. Properties, by contrast, can evolve: start with an auto-property, later replace it with a backing field plus checks, without breaking callers. So choose auto-properties by default, `init` for immutable data, `required` for mandatory data, and add validation through a backing field.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — поля, свойства, init-only, required, валидация / fields, properties, init-only, required, validation
using System;
using System.Collections.Generic;

// 1) Публичное поле — НЕ рекомендуется (нет валидации, нет инкапсуляции) / Public field — NOT recommended
public class BadExample
{
    public int Age; // Любой может записать -5, и никто не возразит / Anyone can store -5 unchecked
}

// 2) Автосвойство — чистый и безопасный стандартный вариант / Auto-property — clean default
public class PersonAuto
{
    public string Name { get; set; }            // Читаемое и записываемое / read-write
    public DateTime CreatedAt { get; } = DateTime.UtcNow; // Только для чтения, инициализировано / read-only, initialized
    public List<string> Tags { get; } = new();  // Коллекция только для роста, без пересоздания / grow-only collection
}

// 3) init-only (C# 9) — неизменяемый после создания / immutable after construction
public class Point
{
    public int X { get; init; }
    public int Y { get; init; }
}

// 4) required (C# 11) — обязателен при создании / mandatory at construction
public class Account
{
    public required string Email { get; init; }       // Должен быть задан в инициализаторе / must be set in initializer
    public required Guid Id { get; init; }
    public string? DisplayName { get; init; }          // Необязателен / optional
}

// 5) Backing field + валидация в set + expression-bodied get / backing field with validation
public class Product
{
    private decimal _price; // backing field — приватное хранилище / private storage

    public decimal Price
    {
        get => _price;                          // expression-bodied get
        set
        {
            if (value < 0)
                throw new ArgumentOutOfRangeException(nameof(value), "Цена не может быть отрицательной / Price cannot be negative");
            _price = value;                     // валидное значение сохраняем / store valid value
        }
    }

    // Вычисляемое свойство только для чтения / computed read-only property
    public string Formatted => $"{_price:C}";   // expression-bodied, без set
}

// 6) Конструктор с [SetsRequiredMembers] — задаёт required-свойства / constructor that sets required members
public class Order
{
    [System.Diagnostics.CodeAnalysis.SetsRequiredMembers]
    public Order(Guid id, string email)
    {
        Id = id;
        Email = email;
    }

    public required Guid Id { get; init; }
    public required string Email { get; init; }
    public IReadOnlyList<string> Items { get; init; } = Array.Empty<string>();
}

class Demo
{
    static void Main()
    {
        // Автосвойства / auto-properties
        var p = new PersonAuto { Name = "Alice" };
        p.Name = "Bob";                      // OK — set доступен / set is available
        // p.CreatedAt = DateTime.Now;       // Ошибка компиляции — get-only / compile error: get-only

        // init-only — задать можно только при создании / can only set during init
        var pt = new Point { X = 1, Y = 2 };
        // pt.X = 5;                         // Ошибка: init нельзя менять после создания / error: init is immutable after init

        // required — пропустить Email нельзя / cannot skip Email
        var acc = new Account { Email = "a@b.c", Id = Guid.NewGuid() };

        // Валидация в set / validation in set
        var prod = new Product();
        prod.Price = 10m;                    // OK
        // prod.Price = -1m;                 // Бросит ArgumentOutOfRangeException / throws

        // Конструктор с SetsRequiredMembers снимает требование инициализатора / constructor satisfies required
        var order = new Order(Guid.NewGuid(), "x@y.z");
        Console.WriteLine($"{p.Name} {pt.X},{pt.Y} {acc.Email} {prod.Formatted} {order.Email}");
    }
}
```

#### Best Practices

- Делайте поля приватными; публичный API открывайте через свойства, чтобы сохранить контроль и совместимость / Keep fields private; expose public API through properties to keep control and compatibility.
- По умолчанию используйте автосвойства `get; set;` — меньше шума, компилятор сам создаёт backing-field / Use auto-properties `get; set;` by default — less noise, compiler synthesizes the backing field.
- Для неизменяемых данных предпочитайте `init`, а не `private set` — это ясно выражает намерение и разрешает инициализаторы / Prefer `init` over `private set` for immutable data — it clearly expresses intent and allows initializers.
- Используйте `required` для обязательных свойств, чтобы ошибки всплывали на этапе компиляции, а не рантайма / Use `required` for mandatory properties so errors surface at compile time, not runtime.
- Валидируйте значения в `set` через backing-field и бросайте `ArgumentOutOfRangeException`/`ArgumentException` с понятным сообщением / Validate values in `set` via a backing field and throw `ArgumentOutOfRangeException`/`ArgumentException` with a clear message.
- Вычисляемые свойства делайте expression-bodied и без `set` / Make computed properties expression-bodied and setter-less.
- Коллекции-свойства открывайте как `IReadOnlyList<T>` или инициализируйте `get;`-автосвойство, чтобы запретить пересоздание коллекции / Expose collection properties as `IReadOnlyList<T>` or use a `get;`-only auto-property to prevent reassignment.

#### Частые ошибки / Common Mistakes

- Публичное поле вместо свойства → позже нельзя добавить валидацию без ломающих изменений. Используйте свойство с самого начала / Public field instead of property → cannot add validation later without breaking changes. Use a property from the start.
- `private set` вместо `init` для неизменяемых данных → инициализаторы объекта недоступны, семантика «изменяемое внутри класса» вводит в заблуждение. Используйте `init` / `private set` instead of `init` for immutable data → object initializers are unavailable and "mutable inside the class" is misleading. Use `init`.
- Забыли `required` на обязательном свойстве → объект создаётся в невалидном состоянии. Пометьте `required` или требуйте через конструктор / Forgot `required` on a mandatory property → object is created invalid. Mark it `required` or enforce via constructor.
- Валидация только в конструкторе, а `set` открытый и без проверок → можно сделать объект невалидным позже. Перенесите проверки в `set` / Validation only in constructor while `set` is public and unchecked → object can become invalid later. Move checks into `set`.
- Возврат публичного `List<T>` через `get; set;` → вызывающий код может пересоздать или очистить коллекцию. Возвращайте `IReadOnlyList<T>` или `get;`-автосвойство / Returning a public `List<T>` via `get; set;` → caller can replace or clear it. Return `IReadOnlyList<T>` or a `get;`-only auto-property.
- Изменение `init`-свойства через рефлексию или хак с `init` в наследнике → ломает неизменяемость. Не делайте этого, проектируйте API честно / Mutating an `init` property via reflection or an `init` override in a subtype → breaks immutability. Do not do this; design the API honestly.
- Использование `field`-ключевого слова в C# 12/.NET 8 → не компилируется; это C# 13+. До перехода используйте явный backing-field / Using the `field` keyword in C# 12/.NET 8 → does not compile; it is C# 13+. Until then use an explicit backing field.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Все поля приватны; публичный доступ — только через свойства / All fields are private; public access is only through properties.
- [ ] Обычные изменяемые данные используют автосвойство `get; set;` / Ordinary mutable data uses an auto-property `get; set;`.
- [ ] Неизменяемые после создания данные используют `init`, а не `private set` / Data immutable after creation uses `init`, not `private set`.
- [ ] Обязательные свойства помечены `required` / Mandatory properties are marked `required`.
- [ ] В `set` с валидацией есть backing-field и бросается исключение с понятным сообщением / A `set` with validation has a backing field and throws an exception with a clear message.
- [ ] Вычисляемые свойства — expression-bodied и без `set` / Computed properties are expression-bodied and have no `set`.
- [ ] Коллекции не отдаются как `List<T> set;` — используется `IReadOnlyList<T>` или `get;`-автосвойство / Collections are not exposed as `List<T> set;` — `IReadOnlyList<T>` or a `get;`-only auto-property is used.
- [ ] Код компилируется на C# 12 / .NET 8 без использования `field`-ключевого слова / The code compiles on C# 12 / .NET 8 without using the `field` keyword.

#### Ресурсы / Resources

- [Microsoft Learn — Properties — https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/properties](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/properties)
- [Microsoft Learn — init (C# 9) — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/init](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/init)
- [Microsoft Learn — required (C# 11) — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/required](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/required)
- [Microsoft Learn — Auto-implemented properties — https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/auto-implemented-properties](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/auto-implemented-properties)

---

[⬆ К модулю M04](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
