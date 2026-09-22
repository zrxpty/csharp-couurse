[⬆ К модулю M16](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M16-L06: Factory, Abstract Factory, Builder / Factory, Abstract Factory, Builder

**Модуль / Module:** M16
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Порождающие паттерны отвечают на один вопрос: «как создавать объекты так, чтобы код не развалился, когда типов станет много?». В этом уроке разберём три классических паттерна — Factory Method, Abstract Factory и Builder — и посмотрим, как C# 12 с первичными конструкторами делает их лаконичнее.

**Factory Method** — это «полиморфное создание». Вместо того чтобы вызывать `new ConcreteType()` напрямую, вы объявляете метод (часто виртуальный или абстрактный), который возвращает интерфейс или базовый класс. Каждый подкласс решает, какой конкретный объект создать. Аналогия: логистическая компания объявляет метод `CreateTransport()`, который возвращает `ITransport`. Подкласс `RoadLogistics` возвращает `Truck`, а `SeaLogistics` — `Ship`. Клиентский код работает с `ITransport` и не знает, грузовик это или корабль. Главный выигрыш — слабая связность: добавление нового транспорта не трогает существующий код.

**Когда применять Factory (не Builder):** когда несколько родственных продуктов создаются по одному и тому же контракту, но с разной реализацией; когда вы хотите изолировать клиента от конкретных типов; когда тип объекта выбирается по конфигурации или среде (dev/prod, Windows/Linux). Если же у вас один объект с десятком необязательных параметров — это территория Builder, а не Factory.

**Abstract Factory** — это «семейство продуктов». Представьте, что вы делаете UI-библиотеку, и вам нужно сразу несколько согласованных виджетов: кнопка, чекбокс, текстовое поле. Для Windows это один набор, для macOS — другой, для Linux — третий. Abstract Factory определяет интерфейс с методами `CreateButton()`, `CreateCheckbox()`, `CreateTextField()`, а конкретные фабрики (`WindowsFactory`, `MacFactory`) производят согласованные семейства. Кlient получает фабрику, вызывает её методы и получает гарантированно совместимые объекты. Главное отличие от Factory Method: Abstract Factory создаёт семейство, а не один продукт, и обычно через композицию, а не наследование.

**Builder** решает другую боль — конструкторы с десятью параметрами, половина из которых `null`. Builder собирает объект пошагово: вы задаёте нужные части, а в конце вызываете `Build()`, который возвращает готовый неизменяемый объект. Аналогия: заказ бургера — вы по очереди выбираете булку, котлету, соус, добавки, касса собирает финальный заказ.

**Fluent Builder** — вариант Builder, где каждый метод возвращает `this`, позволяя строить цепочку: `new BurgerBuilder().WithBun("sezam").WithPatty("beef").WithSauce("bbq").Build()`. Читается как предложение, типобезопасен, и хорошо сочетается с immutable-объектами через `with`-выражения и рекорды.

**Как C# 12 primary ctors упрощают:** первичные конструкторы (`class Foo(Config cfg)`) убирают шаблон: не нужно объявлять приватное поле и конструктор, который просто присваивает параметр полю. Для Factory и Builder это означает, что класс фабрики или билдера становится короче на 3–5 строк. Параметры первичного конструктора доступны во всём теле класса, включая методы — ровно то, что нужно фабрикам и билдерам, хранящим конфигурацию или накопленное состояние. Сочетая primary ctors с `record` и `init`-свойствами, вы получаете неизменяемые результаты сборки без лишнего церемониала.

**Итог:** Factory Method — полиморфное создание одного продукта, Abstract Factory — согласованные семейства, Builder — пошаговая сборка сложного объекта. В C# 12 все три пишутся компактнее благодаря первичным конструкторам и рекордам.

#### Theory (EN)

Creational patterns answer one question: "how do we create objects without the code collapsing when the number of types grows?" This lesson covers three classic patterns — Factory Method, Abstract Factory, and Builder — and shows how C# 12 with primary constructors makes them more concise.

**Factory Method** is "polymorphic creation". Instead of calling `new ConcreteType()` directly, you declare a method (often virtual or abstract) that returns an interface or base class. Each subclass decides which concrete object to create. Analogy: a logistics company declares a `CreateTransport()` method returning `ITransport`. The `RoadLogistics` subclass returns a `Truck`, while `SeaLogistics` returns a `Ship`. Client code works with `ITransport` and does not know whether it is a truck or a ship. The main win is loose coupling: adding a new transport does not touch existing code.

**When to use Factory (not Builder):** when several related products are created under the same contract but with different implementations; when you want to isolate the client from concrete types; when the object type is chosen by configuration or environment (dev/prod, Windows/Linux). If instead you have one object with a dozen optional parameters, that is Builder territory, not Factory.

**Abstract Factory** is about "families of products". Imagine building a UI library that needs several consistent widgets at once: a button, a checkbox, a text field. For Windows that is one set, for macOS another, for Linux a third. Abstract Factory defines an interface with methods `CreateButton()`, `CreateCheckbox()`, `CreateTextField()`, and concrete factories (`WindowsFactory`, `MacFactory`) produce consistent families. The client receives a factory, calls its methods, and gets guaranteed-compatible objects. The key difference from Factory Method: Abstract Factory creates a family rather than a single product, and usually through composition rather than inheritance.

**Builder** addresses a different pain — constructors with ten parameters, half of them `null`. Builder assembles an object step by step: you set the parts you need, then call `Build()`, which returns a ready, immutable object. Analogy: ordering a burger — you pick the bun, the patty, the sauce, the extras in sequence, and the register assembles the final order.

**Fluent Builder** is a variant where each method returns `this`, enabling a chain: `new BurgerBuilder().WithBun("sesame").WithPatty("beef").WithSauce("bbq").Build()`. It reads like a sentence, is type-safe, and pairs well with immutable objects through `with` expressions and records.

**How C# 12 primary ctors simplify:** primary constructors (`class Foo(Config cfg)`) remove boilerplate: you no longer declare a private field and a constructor that just assigns the parameter to the field. For Factory and Builder this means the factory or builder class is 3–5 lines shorter. Primary constructor parameters are in scope across the whole class body, including methods — exactly what factories and builders need when they hold configuration or accumulated state. Combining primary ctors with `record` and `init` properties gives immutable build results without extra ceremony.

**Summary:** Factory Method is polymorphic creation of one product, Abstract Factory is consistent families, Builder is step-by-step assembly of a complex object. In C# 12 all three are written more compactly thanks to primary constructors and records.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Factory Method, Abstract Factory, Fluent Builder
// Иллюстрация всех трёх паттернов в одном файле.

using System;

// ────────────────────────────────────────────────────────────
// 1) FACTORY METHOD — полиморфное создание одного продукта
//    Polymorphic creation of a single product
// ────────────────────────────────────────────────────────────

public interface INotification
{
    void Send(string message);        // Отправить сообщение / Send a message
}

public sealed class EmailNotification(string recipient) : INotification
{
    // Primary ctor: параметр recipient доступен во всём классе
    // Primary ctor: the recipient parameter is in scope for the whole class
    public void Send(string message) =>
        Console.WriteLine($"[Email → {recipient}] {message}");
}

public sealed class SmsNotification(string phone) : INotification
{
    public void Send(string message) =>
        Console.WriteLine($"[SMS → {phone}] {message}");
}

// Базовый класс-создатель с фабричным методом
// Creator base class with the factory method
public abstract class NotificationSender(string defaultTarget)
{
    // Primary ctor хранит defaultTarget без явного поля
    // Primary ctor keeps defaultTarget without an explicit field
    public abstract INotification Create();   // Factory Method

    public void Notify(string message)
    {
        // Клиент не знает конкретный тип — работает с INotification
        // The client does not know the concrete type — works with INotification
        var notification = Create();
        notification.Send(message);
    }

    protected string DefaultTarget => defaultTarget;
}

public sealed class EmailSender : NotificationSender
{
    public EmailSender(string defaultTarget) : base(defaultTarget) { }
    public override INotification Create() =>
        new EmailNotification(DefaultTarget);
}

public sealed class SmsSender : NotificationSender
{
    public SmsSender(string defaultTarget) : base(defaultTarget) { }
    public override INotification Create() =>
        new SmsNotification(DefaultTarget);
}

// ────────────────────────────────────────────────────────────
// 2) ABSTRACT FACTORY — согласованные семейства продуктов
//    Consistent families of products
// ────────────────────────────────────────────────────────────

public interface IButton    { void Render(); }
public interface ICheckbox  { void Render(); }

public sealed class WindowsButton   : IButton   { public void Render() => Console.WriteLine("Windows Button"); }
public sealed class WindowsCheckbox : ICheckbox { public void Render() => Console.WriteLine("Windows Checkbox"); }
public sealed class MacButton       : IButton   { public void Render() => Console.WriteLine("Mac Button"); }
public sealed class MacCheckbox     : ICheckbox { public void Render() => Console.WriteLine("Mac Checkbox"); }

// Абстрактная фабрика — контракт на всё семейство
// Abstract factory — the contract for the whole family
public interface IUiFactory
{
    IButton   CreateButton();
    ICheckbox CreateCheckbox();
}

public sealed class WindowsFactory : IUiFactory
{
    public IButton   CreateButton()   => new WindowsButton();
    public ICheckbox CreateCheckbox() => new WindowsCheckbox();
}

public sealed class MacFactory : IUiFactory
{
    public IButton   CreateButton()   => new MacButton();
    public ICheckbox CreateCheckbox() => new MacCheckbox();
}

// Клиент зависит только от IUiFactory — получает согласованное семейство
// The client depends only on IUiFactory — gets a consistent family
public sealed class UiApplication(IUiFactory factory)
{
    // Primary ctor: factory доступен напрямую в RenderAll
    // Primary ctor: factory is available directly in RenderAll
    public void RenderAll()
    {
        factory.CreateButton().Render();
        factory.CreateCheckbox().Render();
    }
}

// ────────────────────────────────────────────────────────────
// 3) FLUENT BUILDER — пошаговая сборка сложного объекта
//    Step-by-step assembly of a complex object
// ────────────────────────────────────────────────────────────

// Неизменяемый результат сборки / Immutable build result
public sealed record ReportConfig
{
    public required string Title      { get; init; }
    public string?         Subtitle   { get; init; }
    public string?         Author     { get; init; }
    public bool            IncludeToc { get; init; }
    public string          Format     { get; init; } = "PDF";
}

public sealed class ReportBuilder
{
    private string  _title      = string.Empty;
    private string? _subtitle;
    private string? _author;
    private bool    _includeToc;
    private string  _format     = "PDF";

    // Каждый метод возвращает this — fluent-цепочка
    // Each method returns this — fluent chain
    public ReportBuilder WithTitle(string title)        { _title = title; return this; }
    public ReportBuilder WithSubtitle(string? subtitle) { _subtitle = subtitle; return this; }
    public ReportBuilder WithAuthor(string? author)     { _author = author; return this; }
    public ReportBuilder WithToc(bool include = true)   { _includeToc = include; return this; }
    public ReportBuilder WithFormat(string format)      { _format = format; return this; }

    public ReportConfig Build()
    {
        if (string.IsNullOrWhiteSpace(_title))
            throw new InvalidOperationException(
                "Title is required / Заголовок обязателен");

        return new ReportConfig
        {
            Title      = _title,
            Subtitle   = _subtitle,
            Author     = _author,
            IncludeToc = _includeToc,
            Format     = _format
        };
    }
}

// ────────────────────────────────────────────────────────────
// Демонстрация / Demo
// ────────────────────────────────────────────────────────────

public static class Demo
{
    public static void Run()
    {
        // Factory Method
        NotificationSender sender = new EmailSender("admin@acme.io");
        sender.Notify("Build passed / Сборка прошла");

        // Abstract Factory
        var app = new UiApplication(new MacFactory());
        app.RenderAll();

        // Fluent Builder
        ReportConfig report = new ReportBuilder()
            .WithTitle("Q3 Report")
            .WithSubtitle("Sales dept / Отдел продаж")
            .WithAuthor("Alice")
            .WithToc()
            .WithFormat("PDF")
            .Build();

        Console.WriteLine($"Report: {report.Title} ({report.Format})");
    }
}
```

#### Best Practices

- Выбирайте Factory/Abstract Factory только когда типов действительно несколько и они образуют семейство или полиморфный контракт; не заводите фабрику ради одного класса.
- Возвращайте из фабрик интерфейсы, а не конкретные типы — это сохраняет слабую связность и упрощает тестирование через подмену (mock/stub).
- Делайте результаты Builder неизменяемыми (`record` + `init`) — собранный объект не должен меняться после `Build()`.
- Используйте primary constructors C# 12 для фабрик и билдеров, чтобы убрать boilerplate-поля и явные конструкторы-присваивания.
- Валидируйте обязательные поля в `Build()`, а не в каждом сеттере — это концентрирует проверки в одном месте.

- Use Factory/Abstract Factory only when there are genuinely several types forming a family or a polymorphic contract; do not introduce a factory for a single class.
- Return interfaces from factories, not concrete types — this keeps coupling low and makes testing easier via mocks/stubs.
- Make Builder results immutable (`record` + `init`) — an assembled object should not change after `Build()`.
- Use C# 12 primary constructors for factories and builders to remove boilerplate fields and explicit assignment constructors.
- Validate required fields in `Build()`, not in every setter — this concentrates checks in one place.

#### Частые ошибки / Common Mistakes

- Фабрика ради фабрики (один продукт, один тип) → оставьте прямой `new` или статический фабричный метод, пока полиморфизм не понадобится реально.
- Abstract Factory для одного продукта вместо Factory Method → выбирайте по числу продуктов: один — Factory Method, семейство — Abstract Factory.
- Mutable Builder, у которого сеттеры меняют уже собранный объект → разделяйте билдер (изменяемый) и результат (immutable `record`).
- Бог-конструктор с 10 параметрами вместо Builder → если параметров больше 3–4 и часть необязательна, переходите на Builder.
- Сильная связность: клиент зависит от `ConcreteFactory` или `ConcreteProduct` → зависите от интерфейса, внедряйте фабрику через DI.
- Primary ctor скрывает важное состояние, которое нужно валидировать → добавляйте проверку в фабричный метод или `Build()`, не полагайтесь на «автоматически правильно».

- Factory-for-the-sake-of-factory (one product, one type) → keep a direct `new` or a static factory method until polymorphism is genuinely needed.
- Abstract Factory for a single product instead of Factory Method → choose by product count: one — Factory Method, a family — Abstract Factory.
- Mutable Builder whose setters mutate the already-built object → separate the builder (mutable) from the result (immutable `record`).
- God-constructor with 10 parameters instead of Builder → if there are more than 3–4 parameters and some are optional, switch to Builder.
- Tight coupling: the client depends on `ConcreteFactory` or `ConcreteProduct` → depend on the interface, inject the factory via DI.
- Primary ctor hides important state that needs validation → add a check in the factory method or `Build()`, do not rely on "automatically correct".

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Фабричный метод возвращает интерфейс/базовый класс, а не конкретный тип.
- [ ] Подклассы-создатели решают, какой конкретный продукт вернуть, не трогая клиентский код.
- [ ] Abstract Factory используется только для семейств, а не для одиночных продуктов.
- [ ] Builder собирает объект пошагово, результат неизменяемый (record/init).
- [ ] В `Build()` проверяются обязательные поля, выбрасывается понятное исключение.
- [ ] Fluent-методы возвращают `this`, образуя читаемую цепочку.
- [ ] Primary constructors C# 12 применены там, где они убирают boilerplate.
- [ ] Фабрики и билдеры зарегистрированы в DI, клиент не знает конкретных типов.

- [ ] The factory method returns an interface/base class, not a concrete type.
- [ ] Creator subclasses decide which concrete product to return without touching client code.
- [ ] Abstract Factory is used only for families, not for single products.
- [ ] Builder assembles the object step by step; the result is immutable (record/init).
- [ ] `Build()` validates required fields and throws a clear exception.
- [ ] Fluent methods return `this`, forming a readable chain.
- [ ] C# 12 primary constructors are applied where they remove boilerplate.
- [ ] Factories and builders are registered in DI; the client does not know concrete types.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/](https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/)

---

[⬆ К модулю M16](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
