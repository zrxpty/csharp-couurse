[⬆ К модулю M05](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M05-L09: Composition vs inheritance (вступление к SOLID) / Composition vs inheritance (intro to SOLID)

**Модуль / Module:** M05
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

В объектно-ориентированном дизайне есть два классических способа reuse (повторного использования) поведения: **наследование** (inheritance, отношение *is-a* — «является») и **композиция** (composition, отношение *has-a* — «содержит»). Принцип **«Prefer composition over inheritance»** (предпочитай композицию наследованию) из книги «Gang of Four» (1994) — это фундамент, на котором позже выросли принципы SOLID, особенно буква **O** (Open/Closed) и **L** (Liskov Substitution).

**Наследование** создаёт жёсткую связь: дочерний класс зависит от внутренней реализации родителя. Это «белый ящик» — подкласс видит protected-члены и может полагаться на детали. Когда родитель меняется, все потомки могут сломаться. Иерархия фиксируется на этапе компиляции и не меняется во время выполнения. Кроме того, в C# класс может наследовать только один базовый класс, поэтому глубокие иерархии быстро приводят к «аду наследования» (inheritance hell): `ManagerEmployee`, `ManagerEmployeeWithBonus`, `ManagerEmployeeWithBonusAndStockOptions`…

**Композиция** — это когда объект *содержит* другие объекты и делегирует им часть работы. Это «чёрный ящик»: класс зависит только от публичного контракта своих компонентов. Связь задаётся через интерфейсы и инъекцию зависимостей, поэтому её можно менять в рантайме. Композиция даёт **полиморфизм через интерфейсы**, а не через иерархию классов.

**Аналогия:** представь автомобиль. Наследование говорит: «`SportsCar` *является* `Car`». Это правда, но если ты захочешь добавить возможность `Fly` (летать), ты попадёшь в тупик: «`FlyingSportsCar` extends `SportsCar`»? А если ещё и плавать? Композиция говорит: «`Car` *содержит* `Engine`, `Wheels`, `Stereo`». Чтобы добавить полёт, ты просто добавляешь компонент `WingAssembly` — без переписывания иерархии.

**Делегирование (delegation)** — техника, при которой внешний объект пересылает вызов внутреннему. Класс-обёртка реализует интерфейс и перенаправляет запросы полю-компоненту:

```csharp
public class LoggerDecorator : ILogger
{
    private readonly ILogger _inner;
    public LoggerDecorator(ILogger inner) => _inner = inner;
    public void Log(string msg) => _inner.Log($"[{DateTime.UtcNow}] {msg}");
}
```

Это и есть композиция + делегирование — основа паттернов Decorator, Adapter, Facade, Strategy.

**Когда наследование уместно.** Если между классами действительно есть отношение *is-a* и подкласс строго соблюдает контракт родителя (Liskov Substitution), наследование уместно. Примеры из BCL: `Queue<T>` и `Stack<T>` реализуют `ICollection<T>` (через интерфейс), но `KeyedCollection<TKey,TItem>` наследует от `Collection<T>`, потому что это истинное *is-a* с предсказуемым поведением.

**Когда композиция уместнее.** Когда нужно переиспользовать *поведение*, а не тип; когда хочется менять реализацию в рантайме; когда поведение нужно комбинировать из нескольких источников (множественное переиспользование без множественного наследования).

**Связь с SOLID:**
- **SRP** — композиция помогает разделить ответственности по компонентам.
- **OCP** — расширяй через новые компоненты, а не через новые подклассы.
- **LSP** — наследование часто нарушает LSP (квадрат vs прямоугольник); композиция избегает ловушки вовсе.
- **ISP** — компоненты реализуют маленькие интерфейсы.
- **DIP** — зависимости инъектируются через абстракции, что и есть композиция.

Итог: **наследование определяет, что объект *есть*, композиция — что он *делает***. Современный C# (интерфейсы с default-методами, records, primary constructors, `init`-свойства) делает композицию ещё удобнее. В следующих уроках мы разовьём каждую букву SOLID отдельно.

#### Theory (EN)

Object-oriented design offers two classic ways to reuse behaviour: **inheritance** (an *is-a* relationship — "to be") and **composition** (a *has-a* relationship — "to contain"). The **"Prefer composition over inheritance"** principle, stated by the *Gang of Four* in 1994, is the bedrock on which the SOLID principles were later built — especially the **O** (Open/Closed) and **L** (Liskov Substitution).

**Inheritance** creates a rigid link: a derived class depends on the internal implementation of its base class. It is a "white box": a subclass can see `protected` members and may rely on their details. When the base class changes, every descendant can break. The hierarchy is fixed at compile time and cannot change at runtime. Moreover, C# allows a class to inherit from a single base class, so deep hierarchies quickly produce "inheritance hell": `ManagerEmployee`, then `ManagerEmployeeWithBonus`, then `ManagerEmployeeWithBonusAndStockOptions`...

**Composition** means an object *contains* other objects and delegates work to them. It is a "black box": a class depends only on the public contract of its components. The relationship is expressed through interfaces and dependency injection, so it can be swapped at runtime. Composition delivers **polymorphism through interfaces**, not through class hierarchies.

**Analogy:** think of a car. Inheritance says "a `SportsCar` *is a* `Car`." True, but if you later want to add `Fly`, you hit a wall: `FlyingSportsCar extends SportsCar`? And what about swimming? Composition says "a `Car` *has an* `Engine`, `Wheels`, `Stereo`." To add flight, you simply add a `WingAssembly` component — without rewriting the hierarchy.

**Delegation** is a technique where an outer object forwards calls to an inner one. A wrapper class implements an interface and redirects requests to a component field:

```csharp
public class LoggerDecorator : ILogger
{
    private readonly ILogger _inner;
    public LoggerDecorator(ILogger inner) => _inner = inner;
    public void Log(string msg) => _inner.Log($"[{DateTime.UtcNow}] {msg}");
}
```

This is composition + delegation — the foundation of Decorator, Adapter, Facade, and Strategy patterns.

**When inheritance is appropriate.** When there is a genuine *is-a* relationship and the subclass strictly honours the base contract (Liskov Substitution), inheritance is fine. BCL examples: `Queue<T>` and `Stack<T>` implement `ICollection<T>` (through an interface), but `KeyedCollection<TKey,TItem>` inherits from `Collection<T>` because it is a true *is-a* with predictable behaviour.

**When composition wins.** When you want to reuse *behaviour* rather than type; when you want to swap implementations at runtime; when behaviour must be combined from multiple sources (multiple reuse without multiple inheritance).

**Connection to SOLID:**
- **SRP** — composition helps split responsibilities across components.
- **OCP** — extend through new components, not new subclasses.
- **LSP** — inheritance often violates LSP (square vs rectangle); composition sidesteps the trap entirely.
- **ISP** — components implement small, focused interfaces.
- **DIP** — dependencies injected through abstractions *are* composition.

**Trade-offs to weigh.** Composition adds a layer of indirection: a few more objects, slightly more code (forwarding methods), and sometimes less straightforward reading for newcomers. Inheritance, in contrast, is cheap to start with but expensive to maintain: a change in the base ripples to every descendant, and unit testing a deep hierarchy is painful. Composition favours testing (components are mockable through interfaces), favours flexibility (swap a `JsonSerializer` for an `XmlSerializer` by injecting a different `ISerializer`), and favours long-term evolution — which is exactly what SOLID is about.

Bottom line: **inheritance defines what an object *is*; composition defines what it *does*.** Modern C# (default interface methods, records, primary constructors, `init` properties) makes composition even more ergonomic. In the following lessons we will expand each SOLID letter individually.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Composition vs inheritance
// Композиция + делегирование через интерфейсы.
// Composition + delegation through interfaces.

using System;

namespace M05L09;

// --- Поведенческие "кусочки" через интерфейсы (ISP-friendly) ---
// Small behaviour contracts, composed at runtime.

public interface IEngine
{
    void Start();
    int PowerWatts { get; }
}

public interface IDrivetrain
{
    void Move(int distanceMeters);
}

public interface ILogger
{
    void Log(string message);
}

// --- Конкретные компоненты (composable, swappable) ---

public sealed class GasEngine(int powerWatts) : IEngine
{
    public int PowerWatts { get; } = powerWatts;
    public void Start() => Console.WriteLine("Двигатель ДВС заведён / Gas engine started");
}

public sealed class ElectricEngine(int powerWatts) : IEngine
{
    public int PowerWatts { get; } = powerWatts;
    public void Start() => Console.WriteLine("Электромотор включён / Electric motor started");
}

public sealed class WheelDrivetrain : IDrivetrain
{
    public void Move(int distanceMeters) =>
        Console.WriteLine($"Колёса едут {distanceMeters} м / Wheels rolling {distanceMeters} m");
}

public sealed class ConsoleLogger : ILogger
{
    public void Log(string message) => Console.WriteLine($"[log] {message}");
}

// --- Декоратор-логгер: композиция + делегирование ---
// Decorator: composition + delegation.

public sealed class TimestampedLogger(ILogger inner) : ILogger
{
    private readonly ILogger _inner = inner;
    public void Log(string message) =>
        _inner.Log($"[{DateTime.UtcNow:O}] {message}");
}

// --- Главный объект: Car СОДЕРЖИТ компоненты (has-a) ---
// Car HAS-A engine, drivetrain, logger — not IS-A.

public sealed class Car
{
    private readonly IEngine _engine;
    private readonly IDrivetrain _drivetrain;
    private readonly ILogger _logger;

    // Зависимости инъектируются — это и есть DIP + композиция.
    // Dependencies injected — DIP + composition in action.

    public Car(IEngine engine, IDrivetrain drivetrain, ILogger logger)
    {
        ArgumentNullException.ThrowIfNull(engine);
        ArgumentNullException.ThrowIfNull(drivetrain);
        ArgumentNullException.ThrowIfNull(logger);
        _engine = engine;
        _drivetrain = drivetrain;
        _logger = logger;
    }

    public void Drive(int distanceMeters)
    {
        _engine.Start();
        _logger.Log($"Мощность двигателя / Engine power: {_engine.PowerWatts} W");
        _drivetrain.Move(distanceMeters);
        _logger.Log($"Поездка завершена / Trip of {distanceMeters} m finished");
    }
}

// --- Демонстрация: разные "машины" собираются из компонентов ---
// Demo: different cars are assembled from components.

public static class Demo
{
    public static void Run()
    {
        ILogger logger = new TimestampedLogger(new ConsoleLogger());

        var gasCar = new Car(new GasEngine(120_000), new WheelDrivetrain(), logger);
        gasCar.Drive(500);

        Console.WriteLine();

        // Та же оболочка Car, другой двигатель — без новой иерархии классов.
        // Same Car shell, different engine — no new class hierarchy.
        var ev = new Car(new ElectricEngine(150_000), new WheelDrivetrain(), logger);
        ev.Drive(200);
    }
}
```

#### Best Practices

- Предпочитай композицию, когда переиспользуешь поведение, а не тип; оставь наследование для истинных отношений *is-a* с соблюдением LSP.
- Проектируй компоненты вокруг небольших интерфейсов (ISP) и инъектируй их через конструктор (DIP).
- Делегируй публичные методы полю-компоненту; не дублируй логику — используй декораторы и адаптеры.
- Запечатывай (`sealed`) классы по умолчанию — это поощряет композицию вместо наследования.
- Скрывай компоненты как `private readonly` поля; не выставляй внутренности наружу.
- Покрывай компоненты unit-тестами через mock-реализации интерфейсов.

- Prefer composition when you reuse behaviour rather than type; reserve inheritance for genuine *is-a* relationships that honour LSP.
- Design components around small interfaces (ISP) and inject them via the constructor (DIP).
- Delegate public methods to a component field; do not duplicate logic — use decorators and adapters.
- Seal (`sealed`) classes by default to steer the design toward composition over inheritance.
- Keep components as `private readonly` fields; do not leak internals.
- Unit-test components through mock implementations of their interfaces.

#### Частые ошибки / Common Mistakes

- Создание глубоких иерархий наследования (> 2–3 уровней) ради переиспользования кода → проектируй компоненты и инъектируй их (композиция).
- Использование базового класса как мешка с утилитами для потомков → вынеси утилиты в отдельный сервис/компонент.
- Нарушение LSP (Square : Rectangle, переопределение `Width` ломает `Height`) → используй композицию вместо наследования в таких случаях.
- Публичные mutable-поля компонентов, нарушающие инкапсуляцию → держи компоненты `private readonly` и раскрывай только поведение.
- Создание объектов через `new` внутри конструктора бизнес-класса → инъектируй через интерфейсы (DIP), чтобы можно было тестировать и менять реализацию.
- Смешивание "is-a" и "has-a" в одной иерархии → чётко разделяй типы (наследование) и поведения (композиция).

- Building deep inheritance hierarchies (> 2–3 levels) just to reuse code → design components and inject them (composition).
- Using a base class as a grab-bag of utilities for descendants → extract utilities into a separate service/component.
- Violating LSP (Square : Rectangle where overriding `Width` breaks `Height`) → use composition instead of inheritance here.
- Public mutable component fields that break encapsulation → keep components `private readonly` and expose only behaviour.
- Creating objects with `new` inside a business class's constructor → inject through interfaces (DIP) so you can test and swap implementations.
- Mixing "is-a" and "has-a" in the same hierarchy → clearly separate types (inheritance) from behaviours (composition).

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я выбираю наследование только для истинного *is-a* с соблюдением LSP.
- [ ] Поведение переиспользуется через композицию и интерфейсы, а не через базовые классы.
- [ ] Зависимости инъектируются через конструктор (DIP).
- [ ] Классы по умолчанию `sealed`; компоненты `private readonly`.
- [ ] Я могу заменить компонент (mock/stub) в unit-тестах без изменения тестируемого класса.
- [ ] Я знаю, какие буквы SOLID поддерживаются композицией (SRP, OCP, LSP, ISP, DIP).
- [ ] Иерархия не глубже 2–3 уровней; нет «наследовательного ада».

- [ ] I choose inheritance only for a genuine *is-a* relationship that honours LSP.
- [ ] Behaviour is reused through composition and interfaces, not base classes.
- [ ] Dependencies are injected through the constructor (DIP).
- [ ] Classes are `sealed` by default; components are `private readonly`.
- [ ] I can replace a component (mock/stub) in unit tests without touching the class under test.
- [ ] I know which SOLID letters composition supports (SRP, OCP, LSP, ISP, DIP).
- [ ] My hierarchy is no deeper than 2–3 levels; there is no inheritance hell.

#### Ресурсы / Resources

- Microsoft Learn — Microservices DDD/CQRS patterns (composition over inheritance in domain design): https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/
- Gang of Four, *Design Patterns* (1994) — original "Favor object composition over class inheritance" guidance.
- Microsoft Learn — C# interfaces and default interface methods: https://learn.microsoft.com/dotnet/csharp/fundamentals/types/interfaces
- Robert C. Martin, *Agile Principles, Patterns, and Practices in C#* — SOLID foundation.

---

[⬆ К модулю M05](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
