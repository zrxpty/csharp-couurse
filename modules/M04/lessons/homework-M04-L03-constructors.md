---
[← К уроку M04-L03](lesson-M04-L03-constructors.md) | [⬆ К модулю M04](../README.md) | [Следующее ДЗ →](homework-M04-L04-encapsulation.md)
---

### Домашнее задание M04-L03: Конструкторы и инициализаторы / Homework M04-L03: Constructors and initializers

**Урок / Lesson:** M04-L03
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться проектировать классы с параметризованными конструкторами, перегрузками с делегированием `this(...)`, primary-конструкторами C# 12, инициализаторами объектов и коллекций, а также статическими конструкторами; гарантировать инварианты и избегать типичных ошибок (потеря конструктора по умолчанию, мутируемые параметры primary-конструктора, исключения в статическом конструкторе). (EN) Learn to design classes with parameterized constructors, overloads delegating via `this(...)`, C# 12 primary constructors, object and collection initializers, and static constructors; enforce invariants and avoid common mistakes (losing the default constructor, mutating primary-constructor parameters, exceptions in static constructors).

#### Связь с уроком / Connection to the lesson
(RU) Урок M04-L03 вводит пять механизмов: параметризованные конструкторы и перегрузки с `this(...)`, primary-конструкторы C# 12, инициализаторы объектов/коллекций и статические конструкторы. Это ДЗ закрепляет все пять на сквозном примере — мини-библиотеке геометрических фигур с конфигурацией, — чтобы вы прочувствовали, когда каждый механизм уместен и какие подводные камни он несёт.
(EN) Lesson M04-L03 introduces five mechanisms: parameterized constructors with `this(...)` delegation, C# 12 primary constructors, object/collection initializers, and static constructors. This homework reinforces all five through a single worked example — a small geometry-shapes library with configuration — so you feel when each fits and what pitfalls it carries.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы разрабатываете учебную библиотеку `ShapeLib` для вычисления площадей и периметров плоских фигур. Библиотека должна быть безопасной в использовании: нельзя создать «пустую» фигуру с нулевыми сторонами, нельзя получить отрицательный радиус, нельзя дважды инициализировать конфигурацию. Одновременно библиотека должна быть удобной для потребителя: дружелюбные конструкторы по умолчанию, инициализаторы для необязательных параметров, лёгкие DTO на primary-конструкторах. В реальных проектах именно так и устроены доменные модели: инварианты защищаются параметризованными конструкторами, а гибкость настройки даётся через инициализаторы и перегрузки. Статические данные (например, таблица констант или кэш) инициализируются один раз для типа — и здесь важно не попасть в ловушку `TypeInitializationException`. ДЗ моделирует все эти ситуации: вы спроектируете иерархию из нескольких классов, напишете код, который сознательно демонстрирует best practices и частые ошибки, а затем проверите себя через `dotnet test` и вывод в консоль. Цель — не «написать что-то работающее», а осознанно выбрать для каждой задачи правильный механизм из урока и объяснить выбор.

#### Что нужно сделать (пошагово)
1. Создайте решение и проект: `dotnet new sln -n ShapeLib`, затем `dotnet new classlib -n ShapeLib -f net8.0` и `dotnet sln add ShapeLib/ShapeLib.csproj`. Добавьте тестовый проект: `dotnet new xunit -n ShapeLib.Tests -f net8.0`, `dotnet sln add ShapeLib.Tests/ShapeLib.Tests.csproj`, `dotnet add ShapeLib.Tests/ShapeLib.Tests.csproj reference ShapeLib/ShapeLib.csproj`. Убедитесь, что `<LangVersion>12</LangVersion>` и `<Nullable>enable</Nullable>` в обоих `.csproj`.
2. В `ShapeLib/` создайте класс `Circle` с параметризованным конструктором `Circle(double radius)`, проверяющим `radius > 0` через `ArgumentException`. Добавьте перегрузку `Circle()` через `this(1.0)` — «единичный круг». Добавьте свойство `Area => Math.PI * _radius * _radius`.
3. Создайте класс `Rectangle` с primary-конструктором C# 12: `public class Rectangle(double width, double height)`. Захватите параметры в свойства только для чтения `Width => width` и `Height => height`. Добавьте вторичный конструктор `Rectangle(double side) : this(side, side)` для квадрата. Добавьте `Area => width * height`.
4. Создайте класс `ShapeConfig` со свойствами, задаваемыми через инициализатор: `public string Label { get; set; } = "shape"; public ConsoleColor Color { get; set; } = ConsoleColor.White; public List<string> Tags { get; set; } = new();`. Продемонстрируйте инициализатор объекта и коллекции: `new ShapeConfig { Label = "demo", Color = ConsoleColor.Green, Tags = { "geo", "test" } }`.
5. Создайте статический класс `ShapeConstants` со статическим конструктором, который один раз заполняет `IReadOnlyDictionary<string, double>` значениями `["pi"] = Math.PI`, `["e"] = Math.E`. В статическом конструкторе выведите в `Console` сообщение «Static ctor: loading constants». Покажите, что при повторном обращении к `ShapeConstants.Lookup["pi"]` сообщение не выводится повторно.
6. Напишите метод расширения (или метод в `ShapeExtensions`) `Describe(this object shape)`, который через pattern matching возвращает строку с типом и площадью: `circle => $"Circle r={...}, area={...:F3}"`.
7. В тестовом проекте напишите тесты: создание `Circle(-1)` выбрасывает `ArgumentException`; `Circle()` имеет радиус 1; `Rectangle(3,4).Area == 12`; `ShapeConstants.Lookup["pi"]` близко к `Math.PI` (используйте `Assert.Equal` с точностью); обращение к `ShapeConstants` дважды не дублирует вывод (проверьте через флаг или просто доверившись гарантии «один раз»).
8. Добавьте демонстрационный `Program.cs` (top-level statements), который создаёт по одной фигуре каждого вида, инициализирует `ShapeConfig`, обращается к `ShapeConstants.Lookup`, и печатает результат `Describe`.
9. Запустите `dotnet build`, `dotnet test`, `dotnet run --project ShapeLib.Demo` (создайте отдельный консольный проект, если нужно). Убедитесь, что всё компилируется без предупреждений и тесты зелёные.
10. В комментариях в коде явно отметьте, какой механизм урока применён в каждом классе: «параметризованный + this(...)», «primary-конструктор C# 12», «инициализаторы», «статический конструктор».

#### Требования к решению
Решение должно компилироваться под .NET 8 с C# 12, без предупреждений компилятора (`TreatWarningsAsErrors` приветствуется, но не обязателен). Все инварианты должны быть защищены: отрицательные радиус/стороны выбрасывают `ArgumentException` с осмысленным сообщением и `paramName`. Параметры primary-конструктора должны быть захвачены в свойства только для чтения (`get => param` или `{ get; } = param`), чтобы сохранить неизменность. Инициализаторы должны использоваться только для свойств с публичным `set`; инвариантные свойства должны иметь `private set` или `init`. Статический конструктор должен быть безусловным и не выбрасывать исключений; рискованные операции (если есть) оборачивайте в `try/catch` с запасным значением. Код должен быть оформлен в неймспейс `ShapeLib`, классы — `public`, тесты — в `ShapeLib.Tests`. Запрещено использовать `null` для представительных значений; включите `<Nullable>enable</Nullable>`. Покройте каждый публичный класс хотя бы одним тестом. Имена файлов: `Circle.cs`, `Rectangle.cs`, `ShapeConfig.cs`, `ShapeConstants.cs`, `ShapeExtensions.cs`.

#### Тонкости и подводные камни
- Потеря конструктора по умолчанию: как только вы добавили `Circle(double)`, «бесплатный» `Circle()` исчез. Если он нужен для сериализации или DI — объявите его явно с `: this(...)`. В `Rectangle` primary-конструктор тоже «съедает» default; вторичный `Rectangle(double side)` не заменяет безпараметровый — если он нужен, добавьте `Rectangle() : this(1,1)`.
- Делегирование `this(...)`: вызов должен стоять в заголовке, не в теле. Тело текущего конструктора выполняется *после* вызванного конструктора. Не дублируйте проверки в каждой перегрузке — выполняйте их в «главном» конструкторе.
- Primary-конструктор и изменяемость: параметры primary-конструктора неявно захватываются и могут мутировать, если ссылаться на них в методах. Чтобы гарантировать неизменность, захватывайте их в `readonly`-поля или get-only свойства: `public double Width => width;` создаёт свойство, но значение хранится в синтезированном поле; для полной иммутабельности лучше `public double Width { get; } = width;` (поле вычисляется один раз).
- Инициализаторы выполняются *после* конструктора: любые значения, заданные в конструкторе, могут быть перетёрты инициализатором, если свойство имеет `set`. Не полагайтесь на порядок свойств в инициализаторе и не делайте в них побочных эффектов. Для коллекций синтаксис `Tags = { "a", "b" }` вызывает `Add` на существующей коллекции, а не создаёт новую.
- Статический конструктор: выполняется один раз и лениво, до первого обращения к статическому члену или созданию экземпляра. Исключение в нём делает тип непригодным на всё время жизни процесса (`TypeInitializationException`). Не делайте I/O и сети в статическом конструкторе — предпочтите `Lazy<T>`. Не создавайте циклов между статическими конструкторами двух типов.
- `init`-свойства — компромисс: позволяют инициализатор, но запрещают изменение после создания, что сохраняет инварианты лучше, чем `set`.
- Проверка через `throw`-выражение в тернарнике (`x > 0 ? x : throw new ArgumentException(...)`) — идиоматичный C# 12 способ валидации прямо в присваивании.

#### Критерии приёмки
- [ ] Решение собирается под .NET 8 / C# 12 без ошибок и предупреждений.
- [ ] `Circle(double radius)` выбрасывает `ArgumentException` при `radius <= 0` с `paramName`.
- [ ] `Circle()` делегирует в `Circle(1.0)` через `this(...)`.
- [ ] `Rectangle` использует primary-конструктор C# 12 и захватывает параметры в get-only свойства.
- [ ] `Rectangle(double side)` создаёт квадрат через `this(side, side)`.
- [ ] `ShapeConfig` инициализируется через object/collection initializer.
- [ ] `ShapeConstants` содержит статический конструктор, заполняющий `IReadOnlyDictionary<string, double>`.
- [ ] Сообщение «Static ctor: loading constants» выводится ровно один раз за запуск.
- [ ] `Describe` использует pattern matching по типу фигуры.
- [ ] Все классы — `public`, в неймспейсе `ShapeLib`.
- [ ] Включён `<Nullable>enable</Nullable>`, нет `null`-представительных значений.
- [ ] Тесты `dotnet test` — зелёные; есть тест на выброс исключения и на площадь.
- [ ] В комментариях указано, какой механизм урока применён в каждом классе.
- [ ] `Program.cs` (top-level) демонстрирует все четыре механизма.
- [ ] Код воспроизводит best practices и сознательно обходит common mistakes из урока.

#### Подсказки (без прямого ответа)
- Вспомните, что `Math.Sqrt` не нужен для площади круга — достаточно `Math.PI * r * r`.
- Для проверки «статический конструктор один раз» достаточно вывести сообщение в нём и дважды обратиться к `ShapeConstants.Lookup`; тест можно построить на перехвате `Console.Out` через `StringWriter`, но проще убедиться глазами в выводе `dotnet run`.
- Для pattern matching в `Describe` используйте `switch`-выражение: `shape switch { Circle c => ..., Rectangle r => ..., _ => ... }`.
- Чтобы primary-конструктор был иммутабельным, сравните `public double Width => width;` и `public double Width { get; } = width;` — подумайте, какой формально «замораживает» значение.
- `init` вместо `set` для `Label`/`Color` в `ShapeConfig` сделает конфигурацию неизменяемой после создания — это соответствует best practice «private set или init для инвариантов».

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — Эталонное решение ДЗ M04-L03
// Reference solution for homework M04-L03
namespace ShapeLib;

using System;
using System.Collections.Generic;

// 1) Параметризованный конструктор + перегрузки с this(...) / Parameterized ctor + overloads with this(...)
public class Circle
{
    private readonly double _radius; // Поле только для чтения хранит инвариант / Readonly field holds the invariant

    // Главный конструктор с проверкой / Main constructor with validation
    public Circle(double radius)
    {
        // Идиома C# 12: throw-выражение в тернарнике / C# 12 idiom: throw-expression in ternary
        _radius = radius > 0
            ? radius
            : throw new ArgumentException("Radius must be positive / Радиус должен быть положительным", nameof(radius));
    }

    // Делегирование в главный конструктор / Delegate to the main constructor
    // Конструктор по умолчанию объявлен явно, т.к. «бесплатный» исчез / Default ctor declared explicitly since the free one vanished
    public Circle() : this(1.0) { }

    public double Radius => _radius;
    public double Area => Math.PI * _radius * _radius;
}

// 2) Primary constructor (C# 12) / Primary constructor (C# 12)
public class Rectangle(double width, double height)
{
    // Проверка инварианта прямо в теле primary-конструктора / Invariant check in primary-ctor body
    public double Width { get; } = width > 0 ? width : throw new ArgumentException("Width must be positive", nameof(width));
    public double Height { get; } = height > 0 ? height : throw new ArgumentException("Height must be positive", nameof(height));

    // Вторичный конструктор делегирует в primary / Secondary ctor delegates to primary
    public Rectangle(double side) : this(side, side) { }

    public double Area => Width * Height;
}

// 3) Инициализаторы объектов и коллекций / Object and collection initializers
public class ShapeConfig
{
    public string Label { get; init; } = "shape";           // init — неизменяем после создания / init — immutable after creation
    public ConsoleColor Color { get; init; } = ConsoleColor.White;
    public List<string> Tags { get; init; } = new();

    // Дружелюбный конструктор по умолчанию остаётся доступным / Friendly default ctor stays available
    public ShapeConfig() { }
}

// 4) Статический конструктор / Static constructor
public static class ShapeConstants
{
    public static IReadOnlyDictionary<string, double> Lookup { get; }

    // Выполняется один раз для типа, лениво / Runs once for the type, lazily
    static ShapeConstants()
    {
        Console.WriteLine("Static ctor: loading constants / Статический ктр: загрузка констант");
        // Безусловно и без исключений / Unconditional and exception-free
        Lookup = new Dictionary<string, double>
        {
            ["pi"] = Math.PI,
            ["e"] = Math.E,
        };
    }
}

// 5) Pattern matching для Describe / Pattern matching for Describe
public static class ShapeExtensions
{
    public static string Describe(this object shape) => shape switch
    {
        Circle c => $"Circle r={c.Radius:F3}, area={c.Area:F3}",
        Rectangle r => $"Rectangle {r.Width:F3}x{r.Height:F3}, area={r.Area:F3}",
        _ => $"Unknown shape: {shape.GetType().Name}",
    };
}
```

Разбор по строкам. `Circle` демонстрирует первый механизм урока — параметризованный конструктор с инвариантом и делегированием. Поле `_radius` объявлено `readonly`, что физически гарантирует неизменность после конструктора. Валидация `radius > 0` выполнена через throw-выражение прямо в присваивании — это идиома C# 12, рекомендованная в best practices урока. Конструктор `Circle() : this(1.0)` решает проблему «исчезнувшего конструктора по умолчанию» — когда вы добавили параметризованный конструктор, компилятор перестал генерировать бесплатный, поэтому безпараметровый нужно объявить явно с делегированием. `Rectangle` показывает primary-конструктор C# 12: параметры `(double width, double height)` в заголовке. Чтобы избежать мутаций (common mistake урока: «параметры primary-конструктора изменяются в методах»), параметры захвачены в get-only свойства с инициализаторами `{ get; } = width` — значение «замораживается» один раз, и ссылаться на изменяемый параметр в методах нельзя. Вторичный `Rectangle(double side) : this(side, side)` иллюстрирует делегирование из вторичного в primary-конструктор. `ShapeConfig` раскрывает инициализаторы: `init`-свойства позволяют инициализатор, но запрещают изменение после создания — это компромисс между гибкостью и инвариантами, рекомендованный в best practices урока («private set или init для инвариантов»). Конструктор по умолчанию оставлен, чтобы инициализатор был применим. `ShapeConstants` — статический конструктор: выполняется один раз, лениво, до первого обращения к `Lookup`. Сообщение в `Console.WriteLine` позволяет увидеть, что он не повторяется. Код внутри безусловный и без `try/catch` — потому что нет рискованных операций; если бы была загрузка с диска, нужен был бы `try/catch` с запасным значением (common mistake урока: «статический конструктор выбрасывает исключение → тип ломается»). `ShapeExtensions.Describe` использует switch-выражение с pattern matching по типу — современный C# 12 способ диспетчеризации. Все классы — `public` и в неймспейсе `ShapeLib`, включён nullable-контекст по умолчанию для .NET 8. Решение сознательно применяет все пять механизмов урока и обходит перечисленные common mistakes.

#### Задания на углубление (бонус)
1. Добавьте `record Triangle(double a, double b, double c)` с primary-конструктором и валидацией неравенства треугольника (`a + b > c` и т. д.). Сравните поведение `with`-выражения: какие поля замораживаются, какие нет.
2. Реализуйте ленивую инициализацию констант через `Lazy<IReadOnlyDictionary<string, double>>` вместо статического конструктора и сравните момент первого обращения.
3. Добавьте `ShapeFactory` с приватным конструктором и статическим методом `Create(string kind)`, возвращающим `object`; через pattern matching в `Describe` покажите диспетчеризацию. Обсудите, почему приватный конструктор уместен для фабрики.
4. Покройте тестом поведение «статический конструктор один раз», перехватив `Console.Out` через `StringWriter` и проверив, что сообщение «Static ctor» встречается ровно один раз при двух обращениях.

---

## Statement in English / Постановка на английском

#### Context & motivation
You are building an educational library `ShapeLib` for computing areas and perimeters of flat shapes. The library must be safe to use: you cannot create an "empty" shape with zero sides, you cannot obtain a negative radius, you cannot initialize configuration twice. At the same time the library must be convenient for consumers: friendly default constructors, initializers for optional parameters, lightweight DTOs on primary constructors. In real projects this is exactly how domain models are built: invariants are guarded by parameterized constructors, while configuration flexibility comes from initializers and overloads. Static data (a constants table or a cache) is initialized once per type — and here it is critical not to fall into the `TypeInitializationException` trap. This homework models all those situations: you will design a small hierarchy of classes, write code that deliberately demonstrates best practices and common mistakes, and then verify yourself with `dotnet test` and console output. The goal is not "to write something that works" but to consciously choose, for each task, the right mechanism from the lesson and to explain the choice.

#### What to do step by step
1. Create a solution and project: `dotnet new sln -n ShapeLib`, then `dotnet new classlib -n ShapeLib -f net8.0`, and `dotnet sln add ShapeLib/ShapeLib.csproj`. Add a test project: `dotnet new xunit -n ShapeLib.Tests -f net8.0`, `dotnet sln add ShapeLib.Tests/ShapeLib.Tests.csproj`, `dotnet add ShapeLib.Tests/ShapeLib.Tests.csproj reference ShapeLib/ShapeLib.csproj`. Make sure `<LangVersion>12</LangVersion>` and `<Nullable>enable</Nullable>` are set in both `.csproj` files.
2. In `ShapeLib/` create a class `Circle` with a parameterized constructor `Circle(double radius)` that validates `radius > 0` and throws `ArgumentException`. Add an overload `Circle()` delegating via `this(1.0)` — the "unit circle". Add a property `Area => Math.PI * _radius * _radius`.
3. Create a class `Rectangle` with a C# 12 primary constructor: `public class Rectangle(double width, double height)`. Capture the parameters into read-only properties `Width => width` and `Height => height`. Add a secondary constructor `Rectangle(double side) : this(side, side)` for a square. Add `Area => width * height`.
4. Create a class `ShapeConfig` with properties set via an initializer: `public string Label { get; set; } = "shape"; public ConsoleColor Color { get; set; } = ConsoleColor.White; public List<string> Tags { get; set; } = new();`. Demonstrate the object and collection initializer: `new ShapeConfig { Label = "demo", Color = ConsoleColor.Green, Tags = { "geo", "test" } }`.
5. Create a static class `ShapeConstants` with a static constructor that fills an `IReadOnlyDictionary<string, double>` once with `["pi"] = Math.PI`, `["e"] = Math.E`. In the static constructor print "Static ctor: loading constants" to `Console`. Show that on a second access to `ShapeConstants.Lookup["pi"]` the message is not printed again.
6. Write an extension method (or a method in `ShapeExtensions`) `Describe(this object shape)` that uses pattern matching to return a string with the type and area: `circle => $"Circle r={...}, area={...:F3}"`.
7. In the test project write tests: creating `Circle(-1)` throws `ArgumentException`; `Circle()` has radius 1; `Rectangle(3,4).Area == 12`; `ShapeConstants.Lookup["pi"]` is close to `Math.PI` (use `Assert.Equal` with precision); accessing `ShapeConstants` twice does not duplicate the output (verify via a flag, or simply trust the "once" guarantee).
8. Add a demo `Program.cs` (top-level statements) that creates one shape of each kind, initializes a `ShapeConfig`, accesses `ShapeConstants.Lookup`, and prints the result of `Describe`.
9. Run `dotnet build`, `dotnet test`, `dotnet run --project ShapeLib.Demo` (create a separate console project if needed). Make sure everything compiles without warnings and tests are green.
10. In code comments explicitly mark which lesson mechanism is applied in each class: "parameterized + this(...)", "primary constructor C# 12", "initializers", "static constructor".

#### Requirements
The solution must compile under .NET 8 with C# 12, without compiler warnings (`TreatWarningsAsErrors` is welcome but not required). All invariants must be protected: negative radius/sides throw `ArgumentException` with a meaningful message and `paramName`. Primary-constructor parameters must be captured into read-only properties (`get => param` or `{ get; } = param`) to preserve immutability. Initializers must be used only for properties with a public `set`; invariant properties must have `private set` or `init`. The static constructor must be unconditional and must not throw; risky operations (if any) should be wrapped in `try/catch` with a fallback value. Code must live in namespace `ShapeLib`, classes must be `public`, tests in `ShapeLib.Tests`. Using `null` as a sentinel value is forbidden; enable `<Nullable>enable</Nullable>`. Cover every public class with at least one test. File names: `Circle.cs`, `Rectangle.cs`, `ShapeConfig.cs`, `ShapeConstants.cs`, `ShapeExtensions.cs`.

#### Pitfalls
- Losing the default constructor: as soon as you add `Circle(double)`, the "free" `Circle()` disappears. If you need it for serialization or DI — declare it explicitly with `: this(...)`. In `Rectangle` the primary constructor also "eats" the default; the secondary `Rectangle(double side)` does not replace a parameterless one — if needed, add `Rectangle() : this(1,1)`.
- Delegation via `this(...)`: the call must be in the header, not in the body. The body of the current constructor runs *after* the delegated constructor. Do not duplicate checks across overloads — perform them in the "main" constructor.
- Primary constructor and mutability: primary-constructor parameters are implicitly captured and may mutate if referenced in methods. To guarantee immutability, capture them into `readonly` fields or get-only properties: `public double Width => width;` creates a property but the value is stored in a synthesized field; for full immutability prefer `public double Width { get; } = width;` (the field is computed once).
- Initializers run *after* the constructor: any values set in the constructor may be overwritten by the initializer if the property has a `set`. Do not rely on property order in the initializer and do not have side effects in it. For collections the syntax `Tags = { "a", "b" }` calls `Add` on the existing collection rather than creating a new one.
- Static constructor: runs once and lazily, before the first access to a static member or instance creation. An exception in it makes the type unusable for the whole process lifetime (`TypeInitializationException`). Do not do I/O or networking in a static constructor — prefer `Lazy<T>`. Do not create cycles between static constructors of two types.
- `init`-properties are a compromise: they allow an initializer but forbid changes after creation, which protects invariants better than `set`.
- Validation via a `throw`-expression in a ternary (`x > 0 ? x : throw new ArgumentException(...)`) is the idiomatic C# 12 way to validate directly in an assignment.

#### Acceptance criteria
- [ ] The solution builds under .NET 8 / C# 12 without errors or warnings.
- [ ] `Circle(double radius)` throws `ArgumentException` when `radius <= 0`, with `paramName`.
- [ ] `Circle()` delegates to `Circle(1.0)` via `this(...)`.
- [ ] `Rectangle` uses a C# 12 primary constructor and captures parameters into get-only properties.
- [ ] `Rectangle(double side)` creates a square via `this(side, side)`.
- [ ] `ShapeConfig` is initialized via an object/collection initializer.
- [ ] `ShapeConstants` contains a static constructor populating an `IReadOnlyDictionary<string, double>`.
- [ ] The message "Static ctor: loading constants" is printed exactly once per run.
- [ ] `Describe` uses pattern matching on the shape type.
- [ ] All classes are `public`, in namespace `ShapeLib`.
- [ ] `<Nullable>enable</Nullable>` is on; no `null` sentinel values.
- [ ] `dotnet test` is green; there is a test for the thrown exception and for the area.
- [ ] Comments state which lesson mechanism is applied in each class.
- [ ] `Program.cs` (top-level) demonstrates all four mechanisms.
- [ ] The code reproduces best practices and deliberately avoids the lesson's common mistakes.

#### Hints (no direct answer)
- Recall that `Math.Sqrt` is not needed for the area of a circle — `Math.PI * r * r` is enough.
- To verify "static constructor once" it is enough to print a message inside it and access `ShapeConstants.Lookup` twice; a test can intercept `Console.Out` via `StringWriter`, but it is easier to see it in `dotnet run` output.
- For pattern matching in `Describe` use a switch expression: `shape switch { Circle c => ..., Rectangle r => ..., _ => ... }`.
- To make the primary constructor immutable, compare `public double Width => width;` and `public double Width { get; } = width;` — think about which one formally "freezes" the value.
- Using `init` instead of `set` for `Label`/`Color` in `ShapeConfig` makes the configuration immutable after creation — this matches the best practice "private set or init for invariants".

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — Reference solution for homework M04-L03
namespace ShapeLib;

using System;
using System.Collections.Generic;

// 1) Parameterized constructor + overloads with this(...)
public class Circle
{
    private readonly double _radius; // Readonly field holds the invariant

    // Main constructor with validation
    public Circle(double radius)
    {
        // C# 12 idiom: throw-expression in ternary
        _radius = radius > 0
            ? radius
            : throw new ArgumentException("Radius must be positive", nameof(radius));
    }

    // Delegate to the main constructor
    // Default ctor declared explicitly since the free one vanished
    public Circle() : this(1.0) { }

    public double Radius => _radius;
    public double Area => Math.PI * _radius * _radius;
}

// 2) Primary constructor (C# 12)
public class Rectangle(double width, double height)
{
    // Invariant check inside the primary-ctor body
    public double Width { get; } = width > 0 ? width : throw new ArgumentException("Width must be positive", nameof(width));
    public double Height { get; } = height > 0 ? height : throw new ArgumentException("Height must be positive", nameof(height));

    // Secondary ctor delegates to the primary one
    public Rectangle(double side) : this(side, side) { }

    public double Area => Width * Height;
}

// 3) Object and collection initializers
public class ShapeConfig
{
    public string Label { get; init; } = "shape";           // init — immutable after creation
    public ConsoleColor Color { get; init; } = ConsoleColor.White;
    public List<string> Tags { get; init; } = new();

    // Friendly default ctor stays available
    public ShapeConfig() { }
}

// 4) Static constructor
public static class ShapeConstants
{
    public static IReadOnlyDictionary<string, double> Lookup { get; }

    // Runs once for the type, lazily
    static ShapeConstants()
    {
        Console.WriteLine("Static ctor: loading constants");
        // Unconditional and exception-free
        Lookup = new Dictionary<string, double>
        {
            ["pi"] = Math.PI,
            ["e"] = Math.E,
        };
    }
}

// 5) Pattern matching for Describe
public static class ShapeExtensions
{
    public static string Describe(this object shape) => shape switch
    {
        Circle c => $"Circle r={c.Radius:F3}, area={c.Area:F3}",
        Rectangle r => $"Rectangle {r.Width:F3}x{r.Height:F3}, area={r.Area:F3}",
        _ => $"Unknown shape: {shape.GetType().Name}",
    };
}
```

Line-by-line walk-through. `Circle` demonstrates the first lesson mechanism — a parameterized constructor with an invariant and delegation. The `_radius` field is declared `readonly`, which physically guarantees immutability after the constructor. Validation `radius > 0` is done through a throw-expression directly in the assignment — this is the C# 12 idiom recommended in the lesson's best practices. The constructor `Circle() : this(1.0)` solves the "vanished default constructor" problem: once you add a parameterized constructor the compiler stops generating the free one, so a parameterless constructor must be declared explicitly with delegation. `Rectangle` shows a C# 12 primary constructor: parameters `(double width, double height)` in the header. To avoid mutations (the lesson's common mistake: "primary-constructor parameters are mutated in methods"), the parameters are captured into get-only properties with initializers `{ get; } = width` — the value is "frozen" once, and you cannot reference a mutable parameter in methods. The secondary `Rectangle(double side) : this(side, side)` illustrates delegation from a secondary to a primary constructor. `ShapeConfig` reveals initializers: `init`-properties allow an initializer but forbid changes after creation — this is the compromise between flexibility and invariants recommended in the lesson's best practices ("private set or init for invariants"). The default constructor is kept so the initializer is applicable. `ShapeConstants` is a static constructor: it runs once, lazily, before the first access to `Lookup`. The `Console.WriteLine` message lets you see that it does not repeat. The code inside is unconditional and has no `try/catch` — because there are no risky operations; if there were disk loading, you would need a `try/catch` with a fallback value (the lesson's common mistake: "static constructor throws → the type is broken"). `ShapeExtensions.Describe` uses a switch expression with pattern matching on type — the modern C# 12 way to dispatch. All classes are `public` and in namespace `ShapeLib`; the nullable context is on by default for .NET 8. The solution deliberately applies all five lesson mechanisms and avoids the listed common mistakes.

#### Going deeper (bonus)
1. Add a `record Triangle(double a, double b, double c)` with a primary constructor and validation of the triangle inequality (`a + b > c` and so on). Compare the behavior of the `with`-expression: which fields are frozen and which are not.
2. Implement lazy initialization of constants via `Lazy<IReadOnlyDictionary<string, double>>` instead of a static constructor and compare the moment of first access.
3. Add a `ShapeFactory` with a private constructor and a static method `Create(string kind)` returning `object`; through pattern matching in `Describe` show the dispatch. Discuss why a private constructor is appropriate for a factory.
4. Cover with a test the "static constructor once" behavior by intercepting `Console.Out` via `StringWriter` and checking that the "Static ctor" message appears exactly once across two accesses.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Решение собирается под .NET 8 / C# 12 без предупреждений.
- [ ] (RU) Все пять механизмов урока применены и отмечены в комментариях.
- [ ] (RU) Тесты зелёные, есть тесты на выброс исключения и на площадь.
- [ ] (RU) Вывод `dotnet run` показывает сообщение статического конструктора ровно один раз.
- [ ] (RU) Файлы названы согласно требованиям: `Circle.cs`, `Rectangle.cs`, `ShapeConfig.cs`, `ShapeConstants.cs`, `ShapeExtensions.cs`.
- [ ] (EN) The solution builds under .NET 8 / C# 12 without warnings.
- [ ] (EN) All five lesson mechanisms are applied and noted in comments.
- [ ] (EN) Tests are green; there are tests for the thrown exception and for the area.
- [ ] (EN) The `dotnet run` output shows the static-constructor message exactly once.
- [ ] (EN) Files are named per requirements: `Circle.cs`, `Rectangle.cs`, `ShapeConfig.cs`, `ShapeConstants.cs`, `ShapeExtensions.cs`.

#### Ресурсы / Resources
- Microsoft Learn — Constructors: https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/constructors
- Primary constructors (C# 12): https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-12#primary-constructors
- Object and collection initializers: https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/object-and-collection-initializers
- Static constructors: https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/static-constructors
- `init` accessors: https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/init
- Pattern matching: https://learn.microsoft.com/dotnet/csharp/fundamentals/functional/pattern-matching
