[⬆ К модулю M05](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M05-L03: Абстрактные классы и методы / Abstract classes and methods

**Модуль / Module:** M05
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Абстрактный класс — это класс, помеченный модификатором `abstract`. Его ключевая особенность: **нельзя создать экземпляр** через `new`. Звучит как ограничение, но на деле это мощный инструмент проектирования: абстрактный класс задаёт «скелет» — общее состояние и поведение для группы родственных типов, а детали реализации оставляет наследникам.

Представьте чертёж автомобиля. Сам по себе «черновой автомобиль» не поедет — нельзя собрать «просто транспортное средство». Но на чертеже уже зафиксированы: наличие двигателя, колёс, руля и метод `Start()`. Конкретные модели — `Sedan`, `Truck`, `SportsCar` — берут этот каркас за основу и наполняют его жизнью. Абстрактный класс — именно такой чертёж: он объединяет то, что **обязательно для всех** наследников, но не существует сам по себе.

Абстрактный класс может содержать:
- **Обычные (конкретные) поля, свойства и методы** — с готовой реализацией, которую наследники получают «из коробки».
- **Абстрактные методы** — объявление без тела (`abstract void Start();`). Наследник **обязан** переопределить их через `override`. Никаких «может, а может и нет» — контракт строгий.
- **Конструкторы** — вызываются из производных классов через `: base(...)`.
- **Виртуальные методы** (`virtual`) — имеют реализацию, но наследник *может* её переопределить, если хочет.

Разница между `abstract` и `virtual`: `virtual` даёт реализацию по умолчанию, которую можно (но не обязательно) менять; `abstract` реализации не даёт вовсе и переопределение обязательно. Это два разных уровня свободы.

**Когда выбрать абстрактный класс, а не интерфейс?** Это классический вопрос, и ответ строится на трёх критериях:
1. **Общее состояние.** Если нужна общая логика и поля — абстрактный класс. Интерфейс в C# 8+ допускает реализацию по умолчанию, но не позволяет хранить состояние экземпляра.
2. **Семейство типов.** Если иерархия — это «виды одного рода» (животные, фигуры, документы), абстрактный класс семантически точнее.
3. **Одиночное наследование.** Класс может наследовать только один базовый класс (абстрактный или нет), но реализовывать много интерфейсов. Если типу нужно «быть» несколькими вещами сразу — интерфейсы.

Практическое правило: **начинайте с интерфейса**, если нужно лишь описать контракт. Переходите к абстрактному классу, когда появляется общее состояние или разделяемая реализация, которую не хочется копировать в каждый наследник.

**Конкретный наследник** — неабстрактный класс, который закрывает все абстрактные методы базового класса. Пока хоть один `abstract` метод не переопределён, производный класс тоже обязан быть `abstract`. Это естественная «каскадность»: абстрактность спускается вниз, пока не встретит полного переопределения. Только тогда `new` снова становится разрешённым.

Абстрактные классы прекрасно работают вместе с интерфейсами: базовый абстрактный класс может реализовывать интерфейс, предоставляя часть логики, а дочерние классы дописывают остаток. Получается чистая, переиспользуемая архитектура, где контракт (интерфейс) отделён от общей механики (абстрактный класс) и от специфики (конкретный класс).

#### Theory (EN)

An abstract class is a class marked with the `abstract` modifier. Its defining trait: **you cannot instantiate it** with `new`. That sounds like a restriction, but it is actually a powerful design tool. An abstract class defines a skeleton — shared state and behavior for a family of related types — while leaving the implementation details to its derived classes.

Think of a blueprint for a vehicle. The blueprint itself cannot be driven — there is no such thing as "just a vehicle" rolling off the line. Yet the blueprint already fixes certain things: it must have an engine, wheels, a steering wheel, and a `Start()` method. Concrete models — `Sedan`, `Truck`, `SportsCar` — take that skeleton as their foundation and bring it to life. An abstract class is exactly that blueprint: it captures what is **mandatory for all** descendants, but does not exist on its own.

An abstract class may contain:
- **Concrete fields, properties, and methods** — fully implemented, inherited "out of the box" by every descendant.
- **Abstract methods** — declarations without a body (`abstract void Start();`). A derived class **must** override them with `override`. There is no "maybe" — the contract is strict.
- **Constructors** — invoked from derived classes via `: base(...)`.
- **Virtual methods** (`virtual`) — they carry a default implementation that a descendant *may* override.

The difference between `abstract` and `virtual`: `virtual` provides a default implementation you can optionally change; `abstract` provides none and makes overriding mandatory. These are two distinct levels of freedom.

**When to choose an abstract class over an interface?** This is a classic question, and the answer rests on three criteria:
1. **Shared state.** When you need common logic and instance fields, use an abstract class. Interfaces in C# 8+ support default implementations, but they cannot hold per-instance state.
2. **A family of types.** When the hierarchy represents "kinds of the same thing" (animals, shapes, documents), an abstract class is the more semantically honest choice.
3. **Single inheritance.** A class can inherit from only one base class (abstract or not) but implement many interfaces. When a type needs to "be" several things at once, reach for interfaces.

A practical rule of thumb: **start with an interface** when you only need to describe a contract. Move to an abstract class when shared state or reusable implementation appears that you do not want to copy into every descendant.

A **concrete descendant** is a non-abstract class that overrides every abstract method of its base. As long as even one `abstract` method remains unoverridden, the derived class itself must be declared `abstract`. This is a natural cascade: abstractness flows downward until it meets a complete override. Only then does `new` become legal again.

Abstract classes compose cleanly with interfaces: a base abstract class can implement an interface, providing part of the logic, while subclasses fill in the rest. The result is a clean, reusable architecture where the contract (interface), the shared mechanics (abstract class), and the specifics (concrete class) each live in their own layer.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Абстрактные классы и методы / Abstract classes and methods
using System;

// Абстрактный базовый класс — экземпляр создать нельзя.
// Abstract base class — cannot be instantiated.
public abstract class Shape
{
    // Конкретное свойство — наследуется всеми фигурами.
    // Concrete property — inherited by every shape.
    public string Name { get; }

    // Конструктор вызывается из наследников через base(...).
    // Constructor is invoked from descendants via base(...).
    protected Shape(string name) => Name = name;

    // Абстрактный метод — без тела, наследник обязан переопределить.
    // Abstract method — no body, descendants must override it.
    public abstract double Area();

    // Виртуальный метод — есть реализация по умолчанию, можно переопределить.
    // Virtual method — has a default implementation, may be overridden.
    public virtual string Describe() => $"{Name}: area = {Area():F2}";

    // Общий код для всех фигур — переиспользуем, не дублируем.
    // Shared code for all shapes — reused, not duplicated.
    public void Print() => Console.WriteLine(Describe());
}

// Конкретный наследник — закрывает все абстрактные методы.
// Concrete descendant — closes every abstract method.
public sealed class Circle : Shape
{
    public double Radius { get; }

    public Circle(double radius) : base("Circle") => Radius = radius;

    // override обязателен для абстрактного метода базового класса.
    // override is mandatory for an abstract method of the base class.
    public override double Area() => Math.PI * Radius * Radius;

    // override необязателен для virtual — здесь расширяем описание.
    // override is optional for virtual — here we extend the description.
    public override string Describe() => $"{base.Describe()}, radius = {Radius}";
}

public sealed class Rectangle : Shape
{
    public double Width { get; }
    public double Height { get; }

    public Rectangle(double width, double height) : base("Rectangle")
    {
        Width = width;
        Height = height;
    }

    public override double Area() => Width * Height;
}

// Промежуточный абстрактный класс — добавляет новое абстрактное поведение.
// Intermediate abstract class — adds new abstract behavior.
public abstract class Polygon : Shape
{
    protected Polygon(string name) : base(name) { }

    // Новый абстрактный член — каскад абстрактности продолжается.
    // A new abstract member — the abstractness cascade continues.
    public abstract int VertexCount { get; }
}

// Конкретный класс в конце цепочки — закрывает и Area, и VertexCount.
// Concrete class at the end of the chain — closes both Area and VertexCount.
public sealed class Triangle : Polygon
{
    public override int VertexCount => 3;
    public override double Area() => 0; // упрощённо / simplified
}

class Program
{
    static void Main()
    {
        // Shape s = new Shape("x"); // ОШИБКА: нельзя создать экземпляр абстрактного класса.
        // ERROR: cannot create an instance of an abstract class.

        Shape[] shapes = { new Circle(2), new Rectangle(3, 4), new Triangle() };

        foreach (var s in shapes)
            s.Print(); // Полиморфный вызов / polymorphic call.
    }
}
```

#### Best Practices

- Делайте абстрактным только то, что действительно не имеет осмысленной реализации по умолчанию; остальное оставляйте `virtual` с разумной базовой логикой.
- Предпочитайте `protected` конструкторы в абстрактных классах — они не для внешнего вызова, а для наследников.
- Помечайте конкретные наследники `sealed`, если не планируете дальнейшее наследование — это помогает JIT и документирует намерение.
- Сосредотачивайте в абстрактном классе общее состояние и инварианты; специфичное поведение оставляйте производным классам.
- Закрывайте контракт интерфейсом, а общую механику — абстрактным классом: так слои не смешиваются.

- Mark abstract only what genuinely has no sensible default implementation; leave the rest `virtual` with a reasonable base behavior.
- Prefer `protected` constructors in abstract classes — they exist for descendants, not for external callers.
- Seal concrete descendants when no further derivation is intended — it aids the JIT and documents intent.
- Concentrate shared state and invariants in the abstract class; push specific behavior down to derived classes.
- Express the contract with an interface and the shared mechanics with an abstract class — keep the layers separate.

#### Частые ошибки / Common Mistakes

- Попытка `new Shape()` → экземпляр абстрактного класса создать нельзя; работайте через конкретных наследников.
- Забыли `override` у абстрактного метода → компилятор требует переопределения; либо добавьте `override`, либо пометьте производный класс `abstract`.
- Объявили `abstract` метод с телом → `abstract` методы тела не имеют; нужен `virtual` для реализации по умолчанию.
- Путаете `abstract` и `virtual` → `abstract` обязывает, `virtual` разрешает; выбирайте по строгости контракта.
- Используете публичный конструктор в абстрактном классе → он вводит в заблуждение; делайте его `protected`.
- Наследник не вызывает `base(...)` → общее состояние базового класса может остаться неинициализированным.

- Trying `new Shape()` → an abstract class cannot be instantiated; work through concrete descendants.
- Forgetting `override` on an abstract method → the compiler requires an override; either add `override` or mark the derived class `abstract`.
- Declaring an `abstract` method with a body → abstract methods have no body; use `virtual` for a default implementation.
- Confusing `abstract` and `virtual` → `abstract` mandates, `virtual` permits; choose by the strictness of the contract.
- Using a public constructor in an abstract class → it is misleading; make it `protected`.
- A descendant skips `base(...)` → shared base state may end up uninitialized.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я могу объяснить, почему экземпляр абстрактного класса создать нельзя.
- [ ] Я различаю `abstract` (обязательно переопределить) и `virtual` (можно переопределить).
- [ ] Я знаю, когда выбрать абстрактный класс, а когда — интерфейс.
- [ ] Мой конкретный наследник переопределяет все абстрактные методы базового класса.
- [ ] Я помечаю конструкторы абстрактных классов как `protected`.
- [ ] Я вызываю `base(...)` для корректной инициализации общего состояния.

- [ ] I can explain why an abstract class cannot be instantiated.
- [ ] I distinguish `abstract` (must override) from `virtual` (may override).
- [ ] I know when to choose an abstract class versus an interface.
- [ ] My concrete descendant overrides every abstract method of the base class.
- [ ] I mark abstract-class constructors as `protected`.
- [ ] I call `base(...)` to correctly initialize shared state.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/abstract](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/abstract)

---

[⬆ К модулю M05](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
