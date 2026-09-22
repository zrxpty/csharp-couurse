[⬆ К модулю M05](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M05-L05: Полиморфизм на практике / Polymorphism in practice

**Модуль / Module:** M05
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Полиморфизм — это способность одного и того же вызова метода вести себя по-разному в зависимости от фактического типа объекта. Если инкапсуляция прячет данные, а наследование переиспользует код, то полиморфизм позволяет писать код, который работает с целым *семейством* типов, не зная заранее, какой именно попадётся. Представьте кнопку «Позвать к столу» в доме: собака прибежит виляя хвостом, кошка проигнорирует, попугай повторит фразу. Один сигнал — разная реакция. Это и есть полиморфизм в чистом виде.

В C# полиморфизм реализуется двумя основными путями: через **абстрактные классы** и через **интерфейсы**.

**Абстрактный класс** описывает «является» (is-a). Это базовый тип с частью реализации и частью «контракта». Методы, помеченные `abstract`, не имеют тела в базовом классе — производный класс обязан их переопределить через `override`. Методы с `virtual` уже имеют реализацию, но разрешают переопределение. Абстрактный класс нельзя создать напрямую (`new Animal()` не скомпилируется), но переменная такого типа может ссылаться на любой производный объект. Классическим примером служит иерархия фигур: `Shape` объявляет абстрактный метод `Area()`, а `Circle` и `Rectangle` реализуют его по своим формулам. Код `double total = shapes.Sum(s => s.Area())` работает единообразно, хотя под капотом вызываются разные формулы.

**Интерфейс** описывает «умеет» (can-do). Это чистый контракт: только сигнатуры членов (методы, свойства, события, индексаторы), без состояния и без реализации (до C# 8, где появились дефолтные методы — но их мы пока оставим за скобками). Класс может реализовать сколько угодно интерфейсов, но наследоваться только от одного базового класса. Поэтому интерфейсы предпочтительнее, когда нужно описать способность, общую для unrelated типов: `IComparable`, `IEnumerable`, `IDisposable`, `IDrawable`.

**Dispatch (диспетчеризация вызова).** Когда компилятор видит вызов `s.Area()`, где `s` имеет статический тип `Shape`, а фактический — `Circle`, он не может на этапе компиляции знать, какую именно реализацию вызывать. Решение принимается во время выполнения — это называется **динамической диспетчеризацией** (dynamic dispatch). Для невиртуальных методов компилятор жёстко связывает вызов с реализацией по статическому типу — это **статическая диспетчеризация**.

**Virtual table (vtable) — обзор.** Чтобы динамическая диспетчеризация была быстрой, среда CLR для каждого типа с виртуальными/абстрактными методами строит скрытую таблицу — *virtual method table*, или *vtable*. Это массив указателей на реальные реализации методов. Каждый объект такого типа содержит скрытый указатель на свою vtable (вместе с заголовком объекта). При вызове виртуального метода среда: (1) берёт указатель на объект, (2) по нему находит vtable, (3) по слоту метода (индексу, который компилятор присвоил ещё на этапе компиляции) берёт адрес нужной реализации, (4) передаёт управление туда. Стоимость такого вызова — пара косвенных обращений к памяти, что на современных процессорах почти неощутимо. Зато переопределение метода в производном классе просто подменяет указатель в соответствующем слоте — и вызов автоматически попадает в новую реализацию.

Практическое правило выбора простое: если у типов есть общее состояние и общая «спина» реализации — берите абстрактный класс; если общая только способность (методы), и типы не связаны родством — берите интерфейс. И помните: полиморфизм раскрывается не там, где вы определяете класс, а там, где вы вызываете метод по базовому типу и позволяете среде выбрать реализацию.

#### Theory (EN)

Polymorphism is the ability of a single method call to behave differently depending on the actual type of the object behind a variable. If encapsulation hides data and inheritance reuses code, polymorphism lets you write code against an entire *family* of types without knowing in advance which one you will get. Picture a "Dinner's ready!" button in a smart home: the dog comes running and wagging its tail, the cat ignores it, the parrot repeats the phrase. One signal, different reactions — that is polymorphism in a nutshell.

C# gives you two primary paths to polymorphism: through **abstract classes** and through **interfaces**.

An **abstract class** models an "is-a" relationship. It is a base type that combines some implementation with some pure contract. Methods marked `abstract` have no body in the base class — derived classes must override them with `override`. Methods marked `virtual` already have a body but may be overridden. You cannot instantiate an abstract class directly (`new Animal()` will not compile), but a variable of that type can hold any derived object. The canonical example is a shape hierarchy: `Shape` declares an abstract `Area()` method, while `Circle` and `Rectangle` implement it with their own formulas. Code like `double total = shapes.Sum(s => s.Area())` works uniformly even though completely different formulas run under the hood.

An **interface** models a "can-do" relationship. It is a pure contract: only member signatures (methods, properties, events, indexers), no instance state, and traditionally no implementation (C# 8 added default interface methods, but set that aside for now). A class can implement any number of interfaces but inherit from only one base class, which makes interfaces preferable when you want to describe a capability shared by otherwise unrelated types: `IComparable`, `IEnumerable`, `IDisposable`, `IDrawable`.

**Dispatch.** When the compiler sees a call `s.Area()` where the static type of `s` is `Shape` but the actual object is a `Circle`, it cannot know at compile time which implementation to call. The decision is made at run time — this is called **dynamic dispatch**. For non-virtual methods the compiler binds the call to the implementation based on the static type — that is **static dispatch**.

**Virtual table (vtable) — overview.** To make dynamic dispatch fast, the CLR builds, for every type that has virtual or abstract methods, a hidden table called the *virtual method table*, or *vtable*. It is an array of pointers to the real method implementations. Every object of such a type carries a hidden pointer to its vtable (alongside the object header). When a virtual method is called, the runtime: (1) takes the object pointer, (2) follows it to the vtable, (3) uses the method's slot (an index assigned at compile time) to fetch the address of the right implementation, (4) jumps there. The cost is a couple of memory indirections, which is nearly invisible on modern CPUs. Overriding a method in a derived class simply replaces the pointer in that slot, so the next call automatically lands in the new implementation.

The practical rule is simple: if your types share state and a common "backbone" of behavior, choose an abstract class; if they share only a capability (methods) and are not related by lineage, choose an interface. And remember: polymorphism is not exercised where you define a class — it is exercised where you call a method through a base type and let the runtime pick the implementation.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Полиморфизм через абстрактный класс и интерфейс
// Polymorphism via abstract class and interface

namespace M05L05;

// Интерфейс: контракт "умеет" / Interface: a "can-do" contract
public interface IDrawable
{
    // Реализация не задаётся — только сигнатура / No body, signature only
    void Draw(); // Нарисовать себя / Draw itself
}

// Абстрактный базовый класс: "является фигурой" / Abstract base: "is a shape"
public abstract class Shape : IDrawable
{
    // Общее состояние для всех фигур / Shared state for all shapes
    public string Name { get; }

    protected Shape(string name) => Name = name;

    // Абстрактный метод: у базового класса нет формулы / Abstract: no formula in base
    public abstract double Area(); // Площадь / Area

    // Виртуальный метод: есть реализация, но можно переопределить
    // Virtual: has a body, may be overridden
    public virtual string Describe() => $"{Name}: площадь/Area = {Area():F2}";

    // Реализация интерфейса по умолчанию для всех фигур
    // Default interface implementation shared by all shapes
    public virtual void Draw() => Console.WriteLine($"[%] {Name}");
}

public sealed class Circle : Shape
{
    public double Radius { get; }

    public Circle(double radius) : base("Круг/Circle") => Radius = radius;

    // override — переопределение абстрактного/виртуального метода
    public override double Area() => Math.PI * Radius * Radius;
}

public sealed class Rectangle : Shape
{
    public double Width { get; }
    public double Height { get; }

    public Rectangle(double w, double h) : base("Прямоугольник/Rectangle")
    {
        Width = w;
        Height = h;
    }

    public override double Area() => Width * Height;

    // Переопределяем виртуальный метод — своя подпись / Override the virtual method
    public override string Describe() => $"[Прямоугольник] {Width}x{Height} => {Area():F2}";
}

// Тип, реализующий интерфейс, но не связанный с Shape родством
// Type that implements the interface but is NOT related to Shape
public sealed class TextLabel : IDrawable
{
    private readonly string _text;
    public TextLabel(string text) => _text = text;

    public void Draw() => Console.WriteLine($"[Текст] {_text}"); // Своё поведение / Own behavior
}

public static class Demo
{
    public static void Run()
    {
        // Массив базового типа хранит разные производные объекты
        // Base-typed array holds different derived objects
        Shape[] shapes =
        {
            new Circle(5),
            new Rectangle(3, 4),
            new Circle(1)
        };

        // Одинаковый вызов — разная реализация (динамическая диспетчеризация)
        // Same call — different implementation (dynamic dispatch)
        double total = 0;
        foreach (Shape s in shapes)
        {
            Console.WriteLine(s.Describe()); // polymorphism in action
            total += s.Area();
        }
        Console.WriteLine($"Сумма площадей / Total area = {total:F2}");

        // Полиморфизм через интерфейс: фигуры и текст в одном списке
        // Polymorphism via interface: shapes and text in one list
        List<IDrawable> scene = new() { new Circle(2), new TextLabel("Hello"), new Rectangle(1, 1) };
        foreach (IDrawable d in scene) d.Draw(); // каждый рисует по-своему / each draws its own way
    }
}
```

#### Best Practices

- Предпочитайте интерфейсы, когда описываете способность, общую для неродственных типов; выбирайте абстрактный класс, когда есть общее состояние и общая «спина» реализации.
- Помечайте листовые классы `sealed`, если не планируете наследование — это ускоряет вызовы (отключает vtable-слоты для дальнейшего переопределения) и защищает инварианты.
- Не переопределяйте поведение, изменяя только тип: полиморфизм должен опираться на переопределённые методы, а не на ветвления `if (x is Circle)` — иначе это уже не полиморфизм, а нарушенная инкапсуляция.
- Всегда вызывайте `base.Method()` при переопределении, если базовая реализация задаёт инвариант (например, логирование или валидацию состояния).

- Prefer interfaces when you describe a capability shared by unrelated types; choose an abstract class when there is shared state and a common implementation backbone.
- Mark leaf classes `sealed` when you do not plan further inheritance — it speeds up calls (disables vtable slots for future overrides) and protects invariants.
- Do not fake polymorphism with `if (x is Circle)` branches; route behavior through overridden methods — otherwise you have broken encapsulation, not polymorphism.
- Always call `base.Method()` when overriding, if the base implementation enforces an invariant such as logging or state validation.

#### Частые ошибки / Common Mistakes

- Скрытие метода через `new` вместо `override` → вызов по базовому типу пойдёт в базовую реализацию. Избегайте: используйте `override` для виртуальных методов и оставляйте `new` только для осознанного скрытия.
- Вызов виртуальных методов из конструктора базового класса → производный класс ещё не инициализирован, поля ещё в состоянии по умолчанию. Избегайте: не вызывайте виртуальные члены в конструкторах, перенесите логику в фабричный метод.
- Создание «богатых» абстрактных базовых классов с десятками абстрактных методов → производным классам тяжело их реализовать корректно. Избегайте: дробите ответственность через интерфейсы.
- Путаница статического и динамического типа → ожидание, что метод без `virtual`/`override` диспетчируется динамически. Избегайте: помните, что только `virtual`/`abstract`/`override` участвуют в динамической диспетчеризации.

- Hiding a method with `new` instead of `override` → a call through the base type lands in the base implementation. Avoid it: use `override` for virtual methods and reserve `new` for deliberate hiding.
- Calling virtual methods from the base constructor → the derived class is not yet initialized, fields still hold defaults. Avoid it: never call virtual members from constructors; move the logic into a factory method.
- Bloated abstract base classes with dozens of abstract members → derived classes struggle to implement them correctly. Avoid it: split responsibilities across interfaces.
- Confusing static and dynamic type → expecting a non-`virtual`/non-`override` method to dispatch dynamically. Avoid it: only `virtual`/`abstract`/`override` participate in dynamic dispatch.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я могу объяснить разницу между абстрактным классом и интерфейсом и выбрать правильный инструмент.
- [ ] Я понимаю, что `override` включает динамическую диспетчеризацию, а `new` — нет.
- [ ] Я знаю, как vtable связывает вызов метода с реализацией во время выполнения.
- [ ] Я не вызываю виртуальные методы из конструкторов.
- [ ] Мой код вызывает методы по базовому типу/интерфейсу, а не ветвится через `is`/`switch` по типам.

- [ ] I can explain the difference between an abstract class and an interface and pick the right tool.
- [ ] I understand that `override` enables dynamic dispatch while `new` does not.
- [ ] I know how the vtable links a method call to an implementation at run time.
- [ ] I do not call virtual methods from constructors.
- [ ] My code calls methods through the base type/interface rather than branching on `is`/`switch` over types.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/fundamentals/object-oriented/polymorphism](https://learn.microsoft.com/dotnet/csharp/fundamentals/object-oriented/polymorphism)

---

[⬆ К модулю M05](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
