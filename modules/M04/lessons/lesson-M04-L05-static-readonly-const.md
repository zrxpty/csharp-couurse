[⬆ К модулю M04](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M04-L05: static vs instance, const vs readonly / static vs instance, const vs readonly

**Модуль / Module:** M04
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

В C# понятия `static`, `instance`, `const` и `readonly` описывают, как данные и поведение принадлежат типу или его экземплярам, и как именно изменяются значения полей.

**Instance (экземплярные) члены** принадлежат конкретному объекту. Класс `Counter` может иметь поле `_count`, и каждый новый объект `Counter` получает собственную копию этого поля. Чтобы обратиться к instance-члену, нужно сначала создать объект через `new`.

**static члены** принадлежат самому типу, а не экземплярам. Поле `_totalInstancesCreated` существует в единственном экземпляре и разделяется всеми объектами типа `Counter`. Обращение происходит через имя типа: `Counter.TotalInstances`, без `new`. Статические методы не могут напрямую обращаться к instance-полям (ведь у них нет конкретного экземпляра), но instance-методы свободно обращаются к static.

**Статические классы** (`static class MathHelper`) — это классы, которые нельзя инстанцировать и от которых нельзя наследоваться. Все их члены обязаны быть статическими. Это удобный контейнер для вспомогательных функций (математика, утилиты) и глобальных констант. Компилятор гарантирует, что никто не создаст объект `MathHelper` случайно и не унаследуется от него.

**const** — константа времени компиляции (compile-time constant). Значение вычисляется компилятором и «вшивается» непосредственно в вызывающий код на этапе сборки. `public const double Pi = 3.141592653589793` заменяется в местах использования на литерал. Отсюда следствие: если вы измените `Pi` в библиотеке и перекомпилируете только её, уже собранный вызывающий код продолжит использовать старое значение. `const` неявно статичен, не может быть instance-полем и должен инициализироваться литеральным выражением, вычислимым во время компиляции (никаких `DateTime.Now`).

**readonly** — поле, доступное только для чтения на этапе выполнения (runtime). Инициализируется один раз — либо при объявлении, либо в конструкторе — и может отличаться между экземплярами. Значение вычисляется в рантайме, поэтому можно писать `public static readonly DateTime BuildDate = DateTime.UtcNow`. При изменении `readonly`-поля в библиотеке и перекомпиляции только её вызывающий код подхватит новое значение без перекомпиляции.

**Когда что использовать.** `const` — для истинных констант, известных во время компиляции и никогда не меняющихся (`Pi`, `MaxRetryCount`, `DaysPerWeek`). `readonly` — для значений, вычисляемых один раз при запуске или в конструкторе, но одинаковых для всех последующих обращений (`ConnectionString`, `ApplicationVersion`). `static` — для разделяемого состояния и утилитарных функций без состояния. Instance — для данных, уникальных для каждого объекта.

**Константы vs конфигурация.** Не путайте константы с конфигурацией. Если значение может измениться между релизами без перекомпиляции (URL API, лимиты, таймауты, ключи), это не `const` и часто не `readonly` — это конфигурация: `appsettings.json`, переменные окружения, Azure App Configuration, IOptions-паттерн. `const` уместен только тогда, когда изменение значения потребует перекомпиляции всего приложения и это приемлемо. В противном случае читайте значение из конфигурации в рантайме и инжектируйте через DI.

Аналогия: `const` — это номер паспорта, выбитый на металле, его нельзя поменять без перевыпуска; `readonly` — это запись в карточке, которую заполняют один раз при выдаче; конфигурация — это записка на холодильнике, которую можно заменить в любой момент.

#### Theory (EN)

In C#, the keywords `static`, `instance`, `const`, and `readonly` describe how data and behavior belong to a type or its instances, and how field values are allowed to change.

**Instance members** belong to a specific object. A `Counter` class might have a field `_count`, and every new `Counter` object gets its own private copy of that field. To reach an instance member you must first create an object with `new`.

**Static members** belong to the type itself, not to any instance. The field `_totalInstancesCreated` exists exactly once and is shared by all `Counter` objects. You access it through the type name — `Counter.TotalInstances` — without `new`. Static methods cannot directly touch instance fields (they have no instance to work with), but instance methods freely use static members.

**Static classes** (`static class MathHelper`) are classes that cannot be instantiated and cannot be inherited from. Every member must itself be static. They make a convenient container for helper functions (math, utilities) and global constants. The compiler guarantees no one will accidentally create a `MathHelper` object or subclass it.

**const** is a compile-time constant. Its value is computed by the compiler and baked directly into the calling code at build time. `public const double Pi = 3.141592653589793` is replaced at every call site with the literal. This has a consequence: if you change `Pi` in a library and recompile only the library, the already-compiled callers keep using the old value. A `const` is implicitly static, cannot be an instance field, and must be initialized with a literal expression that can be evaluated at compile time — no `DateTime.Now` allowed.

**readonly** is a field whose value is fixed at runtime. It is assigned exactly once — either at the declaration or inside a constructor — and may differ between instances. The value is computed at runtime, so you can write `public static readonly DateTime BuildDate = DateTime.UtcNow`. When you change a `readonly` field in a library and recompile just that library, callers pick up the new value without being recompiled themselves.

**When to use what.** Use `const` for true constants known at compile time that never change (`Pi`, `MaxRetryCount`, `DaysPerWeek`). Use `readonly` for values computed once at startup or in a constructor but identical for all subsequent uses (`ConnectionString`, `ApplicationVersion`). Use `static` for shared state and stateless utility functions. Use instance for data that is unique to each object.

**Constants vs configuration.** Do not confuse constants with configuration. If a value can change between releases without a recompile — API URLs, rate limits, timeouts, keys — it is not `const` and usually not `readonly` either; it is configuration: `appsettings.json`, environment variables, Azure App Configuration, the IOptions pattern. `const` is appropriate only when a change in value would require recompiling the entire application and that is acceptable. Otherwise read the value from configuration at runtime and inject it through DI.

Analogy: `const` is a passport number stamped into metal — you cannot change it without reissuing the passport; `readonly` is an entry on a card filled in once when the card is issued; configuration is a sticky note on the fridge that you can replace at any moment.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — рабочий пример / working example
using System;

// Статический класс: нельзя создать экземпляр, нельзя унаследовать.
// Static class: cannot be instantiated, cannot be inherited.
public static class MathHelper
{
    // const — константа времени компиляции, «вшивается» в вызывающий код.
    // const — compile-time constant, baked into the caller.
    public const double Pi = 3.141592653589793;
    public const int MaxRetryCount = 3;
    public const string DefaultLanguage = "en-US";

    // static readonly — вычисляется один раз при инициализации типа (runtime).
    // static readonly — computed once at type initialization (runtime).
    public static readonly DateTime BuildDate = DateTime.UtcNow;
    public static readonly int ProcessorCount = Environment.ProcessorCount;

    // Разделяемое состояние / shared state
    private static int _callCount;

    public static int CallCount => _callCount;

    public static double CircleArea(double radius)
    {
        _callCount++;
        return Pi * radius * radius; // const используется как литерал / const used as literal
    }
}

// Экземплярный класс с mixed членами / instance class with mixed members.
public sealed class Counter
{
    // readonly-поле: задаётся один раз в конструкторе, может отличаться по экземплярам.
    // readonly field: set once in constructor, may differ per instance.
    public readonly string Name;

    // instance-поле: своя копия у каждого объекта / instance field: own copy per object.
    private int _count;

    // static-поле: одно на все экземпляры / static field: one for all instances.
    private static int _totalInstancesCreated;

    public Counter(string name)
    {
        Name = name;            // OK: присваивание readonly в конструкторе / OK: readonly assigned in ctor
        _count = 0;
        _totalInstancesCreated++;
    }

    public int Value => _count;
    public void Increment() => _count++;
    public static int TotalInstances => _totalInstancesCreated;
}

// Демонстрация: значение, которое лучше не делать const, а брать из конфигурации.
// Demo: a value better read from config than baked as const.
public sealed class ApiClient
{
    private readonly HttpClient _http;
    private readonly Uri _endpoint; // readonly — задан в конструкторе из конфигурации

    public ApiClient(HttpClient http, string endpoint)
    {
        _http = http;
        _endpoint = new Uri(endpoint); // runtime-значение из appsettings.json / runtime value from config
    }
}

class Program
{
    static void Main()
    {
        // const и static readonly доступны через тип без new.
        // const and static readonly accessed via type, no new.
        Console.WriteLine(MathHelper.Pi);                 // 3.141592653589793
        Console.WriteLine(MathHelper.MaxRetryCount);      // 3
        Console.WriteLine(MathHelper.BuildDate);          // текущий момент сборки / build moment

        // Instance-члены требуют объект / instance members require an object.
        var orders = new Counter("orders");
        orders.Increment();
        orders.Increment();
        Console.WriteLine($"{orders.Name}: {orders.Value}"); // orders: 2

        var users = new Counter("users");
        Console.WriteLine(Counter.TotalInstances);          // 2 — static, общий счётчик / shared counter

        // Нельзя создать экземпляр статического класса:
        // var mh = new MathHelper(); // CS0723: Cannot create instance of static class
    }
}
```

#### Best Practices

- Используйте `const` только для значений, которые действительно никогда не меняются и известны на этапе компиляции (`Pi`, `DaysPerWeek`); всё, что может меняться между релизами, делайте `static readonly` или конфигурацией.
- Делайте класс `static` только если он действительно не имеет состояния или несёт только статические утилиты; избегайте статических «бог-классов» с глобальным изменяемым состоянием.
- Предпочитайте `readonly` изменяемым `private` полям для полей, которые задаются один раз — это защищает от случайной перезаписи и помогает компилятору проверять инварианты.
- Значения, зависящие от окружения (URL, таймауты, лимиты), выносите в `appsettings.json` / IOptions, а не в `const`.
- В многопоточной среде защищайте изменяемые `static` поля через `lock` или `Interlocked`, либо используйте потокобезопасные типы.

- Use `const` only for values that truly never change and are known at compile time (`Pi`, `DaysPerWeek`); anything that may change between releases should be `static readonly` or configuration.
- Make a class `static` only when it genuinely has no state and carries pure static utilities; avoid static "god classes" with global mutable state.
- Prefer `readonly` over plain mutable `private` fields for fields set exactly once — it guards against accidental reassignment and lets the compiler enforce invariants.
- Move environment-dependent values (URLs, timeouts, limits) to `appsettings.json` / IOptions rather than `const`.
- In multi-threaded code, protect mutable `static` fields with `lock` or `Interlocked`, or use thread-safe types.

#### Частые ошибки / Common Mistakes

- Объявление `const` для значений, которые меняются между релизами (например, `const string ApiUrl = "...";`) → используйте конфигурацию или `static readonly`, иначе после обновления библиотеки старый код будет держать старое значение.
- Использование `const double Pi` в нескольких сборках и изменение точности → перекомпилируйте все зависимые сборки, либо перейдите на `static readonly`.
- Попытка присвоить `readonly`-полю значение вне конструктора (`field = newValue;` в методе) → присваивайте только в конструкторе или при объявлении; CS0191.
- Объявление instance-поля как `const` (`public const int X = 5;` внутри экземплярного контекста ожидания) → `const` неявно статичен; используйте `readonly`, если значение должно быть привязано к экземпляру.
- Глобальное изменяемое `static` состояние без синхронизации в многопоточной среде → используйте `lock`/`Interlocked` или неизменяемые структуры данных.
- Обращение к instance-члену из `static` метода без объекта → передайте экземпляр параметром или сделайте метод instance.
- Создание экземпляра статического класса (`new MathHelper()`) → CS0723; статический класс можно использовать только через его статические члены.

- Declaring `const` for values that change between releases (e.g. `const string ApiUrl = "...";`) → use configuration or `static readonly`, otherwise after a library update old callers keep the stale value.
- Using `const double Pi` across multiple assemblies and changing precision → recompile all dependent assemblies, or switch to `static readonly`.
- Trying to assign a `readonly` field outside the constructor (`field = newValue;` inside a method) → assign only in the constructor or at declaration; CS0191.
- Declaring an instance field as `const` (`public const int X = 5;` expecting per-instance value) → `const` is implicitly static; use `readonly` if the value must be per-instance.
- Global mutable `static` state without synchronization in multi-threaded code → use `lock`/`Interlocked` or immutable data structures.
- Reaching an instance member from a `static` method without an object → pass the instance as a parameter or make the method instance.
- Instantiating a static class (`new MathHelper()`) → CS0723; a static class can only be used through its static members.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я понимаю разницу: `static` принадлежит типу, instance принадлежит объекту.
- [ ] Я могу объяснить, когда `new` не требуется (доступ к `static` членам).
- [ ] Я знаю, что статический класс нельзя инстанцировать и от него нельзя наследоваться.
- [ ] Я отличаю `const` (compile-time, вшивается в вызывающий код) от `readonly` (runtime, задаётся один раз).
- [ ] Я знаю, что `const` неявно статичен и не может быть instance-полем.
- [ ] Я понимаю риск «устаревшего» значения `const` при частичной перекомпиляции.
- [ ] Я выбираю конфигурацию (`appsettings.json`/IOptions) для значений, зависящих от окружения.
- [ ] Я защищаю изменяемые `static` поля в многопоточной среде.

- [ ] I understand the difference: `static` belongs to the type, instance belongs to the object.
- [ ] I can explain when `new` is not needed (accessing `static` members).
- [ ] I know a static class cannot be instantiated and cannot be inherited from.
- [ ] I distinguish `const` (compile-time, baked into callers) from `readonly` (runtime, set once).
- [ ] I know that `const` is implicitly static and cannot be an instance field.
- [ ] I understand the stale-value risk of `const` under partial recompilation.
- [ ] I choose configuration (`appsettings.json`/IOptions) for environment-dependent values.
- [ ] I protect mutable `static` fields in multi-threaded scenarios.

#### Ресурсы / Resources

- [Microsoft Learn — static (C# Reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/static)
- [Microsoft Learn — const (C# Reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/const)
- [Microsoft Learn — readonly (C# Reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/readonly)
- [Microsoft Learn — Static Classes and Static Class Members](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/static-classes-and-static-class-members)
- [Microsoft Learn — Configuration in .NET](https://learn.microsoft.com/dotnet/core/extensions/configuration)

---

[⬆ К модулю M04](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
