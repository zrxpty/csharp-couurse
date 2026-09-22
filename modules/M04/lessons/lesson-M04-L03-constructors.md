[⬆ К модулю M04](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M04-L03: Конструкторы и инициализаторы / Constructors and initializers

**Модуль / Module:** M04
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Конструктор — это специальный метод, который вызывается при создании объекта и подготавливает его к работе. Представьте, что класс — это чертёж дома, а конструктор — бригада строителей, которая по этому чертежу возводит конкретный дом и сразу подключает электричество и воду. Без конструктора объект остался бы «голым» — поля хранили бы значения по умолчанию, а инварианты класса не были бы гарантированы.

**Конструктор по умолчанию (default constructor)** не принимает параметров. Если вы не объявили ни одного конструктора, компилятор C# генерирует публичный конструктор без параметров автоматически — но только для классов (для структур это всегда так). Как только вы добавляете свой конструктор с параметрами, «бесплатный» конструктор по умолчанию исчезает, и его нужно объявлять явно, если он нужен.

**Параметризованный конструктор (parameterized constructor)** принимает аргументы и позволяет задать начальное состояние объекта сразу при создании. Это предпочтительный способ обеспечить инварианты: например, объект `Account` можно создать только с указанием владельца и начального баланса, и тогда невозможно получить «пустой» счёт.

**Перегрузка конструкторов (constructor overloading)** позволяет определить несколько конструкторов с разными сигнатурами. Чтобы не дублировать логику, один конструктор может вызвать другой через `this(...)` — это называется *конструктор-делегирование*. Вызов `this(...)` должен стоять в заголовке конструктора перед телом и выполняется раньше тела текущего конструктора.

**Primary constructors (C# 12)** — новый синтаксис, появившийся в .NET 8. Параметры класса записываются прямо в заголовке объявления типа: `public class Point(double x, double y)`. Эти параметры доступны во всех членах класса и автоматически сохраняются в неявные поля, если на них есть ссылка. Primary-конструктор упрощает код для DTO, record-подобных классов и небольших сервисов: исчезает Boilerplate с явными полями и конструкторами. Параметры primary-конструктора можно «захватить» в свойство (`public double X => x;`) или передать в базовый конструктор.

**Инициализаторы объектов (object initializers)** позволяют задавать свойства сразу после `new`, в фигурных скобках: `new Person { Name = "Иван", Age = 30 }`. Это работает только для доступных set-свойств и идёт *после* выполнения конструктора. Инициализаторы удобны, когда у класса много необязательных параметров, и помогают создавать читаемые конфигурации. Для коллекций есть аналогичный синтаксис — `new List<int> { 1, 2, 3 }`.

**Статические конструкторы (static constructors)** выполняются один раз для типа, до первого обращения к любому статическому члену или созданием экземпляра. Они не имеют модификаторов доступа и параметров, используются для инициализации статических полей — например, загрузки конфигурации, подготовки кэша. Гарантия выполнения отложенная, но обязательно один раз в домене приложения. Если статический конструктор выбрасывает исключение, тип становится непригодным (`TypeInitializationException` при последующих обращениях).

Связка этих механизмов даёт гибкость: primary-конструкторы убирают рутину, перегрузки с `this(...)` переиспользуют логику, инициализаторы дают «свободный» способ настройки, а статические конструкторы готовят данные уровня типа.

#### Theory (EN)

A constructor is a special method invoked when an object is created, preparing it for use. Think of a class as a blueprint for a house, and the constructor as the construction crew that turns that blueprint into an actual house and immediately wires up electricity and water. Without a constructor, the object stays “bare”: its fields hold default values, and class invariants are not guaranteed.

**Default constructor** takes no parameters. If you declare no constructors at all, the C# compiler generates a public parameterless constructor automatically — for classes this is true only when no custom constructor exists; for structs a parameterless constructor is always available. As soon as you add any constructor with parameters, the “free” default constructor disappears, and you must declare it explicitly if you still need it.

**Parameterized constructor** accepts arguments and lets you set initial state at creation time. This is the preferred way to enforce invariants: for example, an `Account` can only be created with an owner and an opening balance, making an “empty” account impossible.

**Constructor overloading** allows multiple constructors with different signatures. To avoid duplicating logic, one constructor may call another through `this(...)` — known as *constructor delegation*. The `this(...)` call must appear in the constructor header before the body and runs before the body of the current constructor.

**Primary constructors (C# 12)**, introduced with .NET 8, let you declare parameters directly in the type header: `public class Point(double x, double y)`. These parameters are available across all members of the class and are automatically captured into hidden fields when referenced. Primary constructors remove boilerplate for DTOs, record-like classes, and small services. You can expose a primary-constructor parameter as a property (`public double X => x;`) or forward it to a base constructor.

**Object initializers** let you set properties right after `new`, inside braces: `new Person { Name = "Ivan", Age = 30 }`. This works only for accessible set-properties and runs *after* the constructor body. Initializers are convenient when a class has many optional parameters and produce readable configuration code. Collections have an analogous syntax — `new List<int> { 1, 2, 3 }`.

**Static constructors** run once per type, before the first access to any static member or the first instance creation. They have no access modifiers and no parameters, and are used to initialize static fields — for example, loading configuration or preparing a cache. Execution is lazy but guaranteed exactly once per application domain. If a static constructor throws, the type becomes unusable (`TypeInitializationException` on subsequent access).

Together these mechanisms give flexibility: primary constructors remove routine code, overloads with `this(...)` reuse logic, initializers provide a free-form configuration path, and static constructors prepare type-level data.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8 — Конструкторы и инициализаторы / Constructors and initializers
using System;
using System.Collections.Generic;

// 1) Параметризованный конструктор + перегрузки с this(...) / Parameterized ctor + overloads with this(...)
public class Account
{
    public string Owner { get; }
    public decimal Balance { get; private set; }

    // Главный конструктор / Main constructor
    public Account(string owner, decimal balance)
    {
        Owner = string.IsNullOrWhiteSpace(owner)
            ? throw new ArgumentException("Owner required / Владелец обязателен", nameof(owner))
            : owner;
        Balance = balance >= 0 ? balance : throw new ArgumentException("Balance < 0", nameof(balance));
    }

    // Делегирование в главный конструктор / Delegate to the main constructor
    public Account(string owner) : this(owner, 0m) { }

    // Конструктор по умолчанию (объявлен явно) / Default ctor declared explicitly
    public Account() : this("Unknown", 0m) { }

    public void Deposit(decimal amount) => Balance += amount;
}

// 2) Primary constructor (C# 12) / Primary constructor (C# 12)
public class Point(double x, double y)
{
    public double X { get; } = x;          // Захват параметра в свойство / Capture param into property
    public double Y { get; } = y;
    public double DistanceToOrigin => Math.Sqrt(x * x + y * y); // Параметр доступен напрямую / Param accessible directly

    // Вторичный конструктор, делегирует в primary / Secondary ctor delegates to primary
    public Point() : this(0, 0) { }
}

// 3) Инициализаторы объектов / Object initializers
public class Person
{
    public string Name { get; set; } = string.Empty;
    public int Age { get; set; }
    public List<string> Hobbies { get; set; } = new();
}

// 4) Статический конструктор / Static constructor
public static class Config
{
    public static IReadOnlyDictionary<string, string> Settings { get; }

    // Выполняется один раз для типа / Runs once for the type
    static Config()
    {
        Console.WriteLine("Static ctor: loading config / Статический ктр: загрузка конфигурации");
        Settings = new Dictionary<string, string>
        {
            ["env"] = "production",
            ["timeout"] = "30"
        };
    }
}

internal static class Demo
{
    public static void Run()
    {
        var a1 = new Account("Иван", 100m);   // Параметризованный / Parameterized
        var a2 = new Account("Ольга");         // Через this(...) / Through this(...)
        var a3 = new Account();                // По умолчанию / Default
        a1.Deposit(50m);
        Console.WriteLine($"{a1.Owner}: {a1.Balance}, {a2.Owner}: {a2.Balance}, {a3.Owner}: {a3.Balance}");

        var p = new Point(3, 4);
        Console.WriteLine($"Point({p.X}, {p.Y}) distance = {p.DistanceToOrigin}");

        // Инициализатор объекта / Object initializer
        var person = new Person { Name = "Игорь", Age = 28, Hobbies = { "chess", "music" } };
        Console.WriteLine($"{person.Name}, {person.Age}, hobbies: {string.Join(", ", person.Hobbies)}");

        // Статический конструктор вызовется при первом обращении / Static ctor fires on first access
        Console.WriteLine($"env = {Config.Settings["env"]}");
    }
}
```

#### Best Practices

- Делайте инварианты явными через параметризованные конструкторы; оставляйте конструктор по умолчанию только когда он даёт осмысленное состояние.
- Используйте `this(...)` для переиспользования логики инициализации и избегайте дублирования между перегрузками.
- Применяйте primary-конструкторы (C# 12) для небольших типов, DTO и сервисов — это убирает boilerplate и повышает читаемость.
- Делайте свойства set-only для инициализаторов там, где это безопасно; для инвариантов используйте `private set` или `init`.
- Держите статические конструкторы простыми и безусловными: любое исключение делает тип непригодным до конца работы приложения.
- Избегайте тяжёлой работы (I/O, сеть) в статических конструкторах — предпочтите ленивую инициализацию (`Lazy<T>`).

- Make invariants explicit through parameterized constructors; keep a default constructor only when it yields a meaningful state.
- Use `this(...)` to reuse initialization logic and avoid duplication across overloads.
- Adopt primary constructors (C# 12) for small types, DTOs, and services — they remove boilerplate and improve readability.
- Expose set-properties for initializers only where safe; use `private set` or `init` to protect invariants.
- Keep static constructors simple and exception-free: any thrown exception makes the type unusable for the rest of the application.
- Avoid heavy work (I/O, network) in static constructors — prefer lazy initialization (`Lazy<T>`).

#### Частые ошибки / Common Mistakes

- Объявили параметризованный конструктор и потеряли конструктор по умолчанию → явно объявите `public MyClass() : this(...)`, если он нужен для сериализации/DI.
- Вызов `this(...)` поставили в теле конструктора, а не в заголовке → переносите вызов в заголовок: `public Foo() : this(0) { }`.
- Параметры primary-конструктора изменяются в методах, и вы ожидаете неизменность → захватывайте их в `get`-свойства (`public int X => x;`) или в поля `readonly`.
- Полагаетесь на порядок полей при инициализаторе объекта → инициализатор срабатывает после конструктора; порядок свойств в нём не гарантирован и не должен иметь побочных эффектов.
- Статический конструктор выбрасывает исключение → тип «ломается» на всё время жизни процесса; оберните рискованные действия в try/catch с запасным значением.
- Создаёте циклические зависимости между статическими конструкторами двух типов → получите `TypeInitializationException`; разрывайте цикл через `Lazy<T>` или инъекцию.

- Declared a parameterized constructor and lost the default one → explicitly declare `public MyClass() : this(...)` if needed for serialization/DI.
- Put the `this(...)` call in the body instead of the header → move it to the header: `public Foo() : this(0) { }`.
- Primary-constructor parameters are mutated in methods but you expect immutability → capture them in get-only properties (`public int X => x;`) or in `readonly` fields.
- Relying on field order in an object initializer → the initializer runs after the constructor; property order is not guaranteed and should not have side effects.
- Static constructor throws → the type is “broken” for the entire process lifetime; wrap risky operations in try/catch with a fallback value.
- Creating circular dependencies between static constructors of two types → you get `TypeInitializationException`; break the cycle with `Lazy<T>` or injection.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я знаю разницу между конструктором по умолчанию и параметризованным и понимаю, когда компилятор генерирует «бесплатный» конструктор.
- [ ] Я могу делегировать вызов через `this(...)` и знаю, что он выполняется до тела текущего конструктора.
- [ ] Я могу записать primary-конструктор (C# 12) и захватить его параметры в свойства или поля.
- [ ] Я умею применять инициализаторы объектов и коллекций и понимаю, что они выполняются после конструктора.
- [ ] Я знаю назначение и ограничения статического конструктора и его поведение при исключении.

- [ ] I can distinguish a default constructor from a parameterized one and know when the compiler generates a “free” constructor.
- [ ] I can delegate through `this(...)` and know it runs before the body of the current constructor.
- [ ] I can write a primary constructor (C# 12) and capture its parameters into properties or fields.
- [ ] I can use object and collection initializers and understand that they run after the constructor.
- [ ] I know the purpose and limits of static constructors and their behavior on exception.

#### Ресурсы / Resources

- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/constructors
- Primary constructors (C# 12) — https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-12#primary-constructors
- Object and collection initializers — https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/object-and-collection-initializers
- Static constructors — https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/static-constructors

---

[⬆ К модулю M04](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
