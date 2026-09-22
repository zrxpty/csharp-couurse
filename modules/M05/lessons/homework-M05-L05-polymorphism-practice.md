---
[← К уроку M05-L05](lesson-M05-L05-polymorphism-practice.md) | [⬆ К модулю M05](../README.md) | [Следующее ДЗ →](homework-M05-L06-sealed-internal.md)
---

### Домашнее задание M05-L05: Полиморфизм на практике / Homework M05-L05: Polymorphism in practice

**Урок / Lesson:** M05-L05
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться применять полиморфизм через абстрактные классы и интерфейсы, понимать разницу между `virtual`/`override` и `new`, осознанно работать с динамической диспетчеризацией и избегать типичных ловушек скрытия методов и вызова виртуальных членов из конструктора. (EN) Learn to apply polymorphism through abstract classes and interfaces, understand the difference between `virtual`/`override` and `new`, work consciously with dynamic dispatch, and avoid typical pitfalls such as method hiding and calling virtual members from constructors.

#### Связь с уроком / Connection to the lesson
(RU) Урок M05-L05 показывает, что полиморфизм раскрывается не в точке определения класса, а в точке вызова метода по базовому типу. ДЗ закрепляет это, требуя спроектировать иерархию, в которой один и тот же вызов `Render()` и `Cost()` диспетчируется в разные реализации, и провести эксперимент, доказывающий разницу между `override` и `new`. Дополнительно вы поработаете с интерфейсом как с «can-do»-контрактом для неродственных типов.
(EN) Lesson M05-L05 shows that polymorphism is exercised not where a class is defined but where a method is called through a base type. This homework cements that idea by asking you to design a hierarchy in which the same `Render()` and `Cost()` call dispatches to different implementations, and to run an experiment that proves the difference between `override` and `new`. You will also use an interface as a "can-do" contract for unrelated types.

---

## Постановка на русском / Russian statement

#### Контекст и мотивации

Вы — backend-разработчик в небольшой команде, которая пишет мини-движок для генерации ASCII-отчётов и счетов. Продукт уже используется двумя клиентами, и теперь приходит маркетинг с просьбой: «Добавьте рамки, добавьте логотип, добавьте текстовые подписи, добавьте вычисление полной стоимости заказа из нескольких компонентов». Если вы начнёте писать ветвления `if (item is Rectangle)` или `switch (item.GetType().Name)`, код превратится в спагетти уже на третьей фиче. Полиморфизм — это инструмент, который позволяет добавлять новые типы компонентов, не меняя существующий клиентский код.

В этом задании вы спроектируете иерархию «элементов отчёта» (`ReportComponent`), каждый из которых умеет отдавать свой текстовое представление (`Render()`) и свою стоимость (`Cost()`). Некоторые элементы — геометрические фигуры (круг, прямоугольник), некоторые — текстовые подписи, некоторые — декоративные рамки вокруг других элементов. Параллельно вы реализуете интерфейс `IPriceable` и убедитесь, что через него можно единообразно посчитать сумму заказа, в который входят и фигуры, и рамки, и сторонний объект `Shipping` (внешний тип, не наследник `ReportComponent`).

Главный обучающий момент не в том, чтобы «написать много классов», а в том, чтобы увидеть, как **одна и та же строчка кода** `foreach (var c in components) total += c.Cost();` вызывает разные реализации в зависимости от фактического типа объекта, и как замена `override` на `new` незаметно ломает эту магию. Вы также столкнётесь с классической ловушкой вызова виртуального метода из конструктора базового класса и должны будете её обойти через фабричный метод.

#### Что нужно сделать (пошагово)

1. **Создайте проект.** В терминале выполните `dotnet new console -n M05L05.Homework -o M05L05.Homework --framework net8.0`. Перейдите в папку: `cd M05L05.Homework`. Откройте `Program.cs` и удалите шаблонный `Console.WriteLine("Hello, World!");`. Убедитесь, что в `.csproj` стоит `<TargetFramework>net8.0</TargetFramework>` и `<Nullable>enable</Nullable>`, `<ImplicitUsings>enable</ImplicitUsings>`.

2. **Определите интерфейс `IPriceable`.** В файле `IPriceable.cs` объявите интерфейс с одним методом `decimal Cost();` и свойством `string Currency { get; }`. Это «can-do»-контракт: всё, что имеет цену, должно уметь её вернуть.

3. **Определите абстрактный класс `ReportComponent`.** В файле `ReportComponent.cs` создайте абстрактный базовый класс, реализующий `IPriceable`. Он должен содержать:
   - защищённое поле `protected readonly string _name;`, инициализируемое в конструкторе;
   - абстрактный метод `public abstract string Render();`
   - виртуальный метод `public virtual decimal Cost() => 0m;` (базовая цена нулевая);
   - свойство `public string Currency => "RUB";`
   - переопределённый `public override string ToString() => $"{_name}: {Render()} [{Cost()} {Currency}]"`.

4. **Создайте производные классы.** В отдельных файлах добавьте: `CircleComponent` (параметр — радиус, `Render()` возвращает `○` и площадь, `Cost()` = площадь × 0.5m), `RectangleComponent` (ширина и высота, `Render()` возвращает `□` и площадь, `Cost()` = площадь × 0.4m), `TextComponent` (строка текста, `Cost()` = длина × 0.1m, `Render()` возвращает сам текст). Все три класса пометьте `sealed`.

5. **Добавьте декоратор `FramedComponent`.** Это класс, который оборачивает другой `ReportComponent` и добавляет рамку вокруг его `Render()`. Он должен наследоваться от `ReportComponent`, принимать внутренний компонент в конструкторе, переопределять `Render()` так, чтобы оборачивать внутренний текст в `+--- ... ---+`, и переопределять `Cost()` как `inner.Cost() + 10m` (стоимость рамки). **Это ключевой момент:** декоратор сам является компонентом, поэтому его можно вкладывать в массивы и списки вместе с обычными компонентами — полиморфизм в действии.

6. **Реализуйте внешний тип `Shipping`.** В файле `Shipping.cs` создайте класс, **не** наследующий `ReportComponent`, но реализующий `IPriceable`. Пусть у него будет поле `decimal _distance` и `Cost()` возвращает `_distance * 2m`. `Currency` — `"RUB"`. Это покажет, что интерфейс позволяет работать с неродственными типами в одном цикле.

7. **Соберите всё в `Program.cs`.** Создайте `List<ReportComponent> components`, добавьте в него круг, прямоугольник, текст, и обёрнутый в рамку круг (`new FramedComponent(new CircleComponent(3))`). Выведите каждый через `Console.WriteLine(component)`. Затем создайте `List<IPriceable> order = new() { circle, rectangle, text, framed, new Shipping(50) };` и посчитайте `var total = order.Sum(x => x.Cost());`. Выведите `Итого: {total} RUB`.

8. **Проведите эксперимент `override` vs `new`.** В отдельном файле `DispatchExperiment.cs` создайте базовый класс `Base { public virtual string Id() => "Base"; }` и два производных: `DerivedOverride : Base { public override string Id() => "DerivedOverride"; }` и `DerivedNew : Base { public new string Id() => "DerivedNew"; }`. В `Run()` объявите `Base b1 = new DerivedOverride(); Base b2 = new DerivedNew();` и выведите `b1.Id()` и `b2.Id()`. **Зафиксируйте в комментарии, какой вывод вы получили и почему.** Это самая важная часть ДЗ — она доказывает разницу между динамической диспетчеризацией (`override`) и скрытием (`new`).

9. **Продемонстрируйте ловушку виртуального метода в конструкторе.** В файле `CtorTrap.cs` создайте базовый класс `LoggableBase` с конструктором, который вызывает `protected virtual void Init()` — и переопределите `Init()` в производном классе так, чтобы он печатал поле производного класса. Запустите, увидьте, что поле ещё в значении по умолчанию (`null` или `0`), и в комментарии объясните, почему. Затем перепишите через статический фабричный метод `Create()`, который сначала конструирует объект, а потом вызывает `Init()` — и убедитесь, что теперь всё работает корректно.

10. **Запустите и зафиксируйте вывод.** Выполните `dotnet run`. Скопируйте вывод в файл `output.txt` рядом с проектом (можно через `dotnet run > output.txt` в PowerShell, либо вручную). Убедитесь, что итоговая сумма считается корректно и что эксперимент `override`/`new` даёт именно тот результат, который вы предсказали в комментарии.

#### Требования к решению

- Проект собирается командой `dotnet build` без предупреждений (warning level как минимум `CS0108` должен быть осознан — если где-то используете `new`, добавьте `// intentional` комментарий).
- Используется C# 12 / .NET 8: разрешены file-scoped namespaces, top-level statements в `Program.cs`, `init`-сеттеры при желании, target-typed `new()`.
- Все листовые классы (`CircleComponent`, `RectangleComponent`, `TextComponent`, `FramedComponent`, `Shipping`) помечены `sealed`, кроме `FramedComponent`, который **может** быть не sealed, если вы планируете дальнейшее наследование декораторов (обоснуйте выбор в комментарии).
- Иерархия опирается на **абстрактный класс** для родственных «элементов отчёта» (общее состояние `_name`, общая «спина» поведения `ToString`) и на **интерфейс** `IPriceable` для объединения с неродственным `Shipping`.
- В коде **нет** ветвлений по типу через `is`/`switch`/`GetType().Name` — вся логика выбора реализации идёт через полиморфные вызовы. Если вам хочется написать `if (c is FramedComponent)` — остановитесь и подумайте, какой виртуальный метод нужно добавить, чтобы избежать этого.
- Эксперимент `override` vs `new` **обязательно** присутствует, его вывод зафиксирован и объяснён в комментарии не менее чем тремя предложениями.
- Ловушка виртуального метода в конструкторе показана и исправлена через фабричный метод; в комментарии объяснено, почему поля производного класса в этот момент имеют значения по умолчанию.
- Все денежные значения используют `decimal`, не `double` (цена — деньги, не измерение).
- Код компилируется и запускается; вывод программы соответствует описанию в разделе «Критерии приёмки».

#### Тонкости и подводные камни

- **`virtual`/`override` vs `new` — главная тема урока.** Если вы по ошибке напишете `public new string Render()` вместо `override` в `FramedComponent`, то при итерации по `List<ReportComponent>` вызов пойдёт в **базовую** реализацию `ReportComponent.Render()` (которая, в нашем случае, абстрактная — компилятор не даст так сделать, но если бы она была виртуальной с телом, вы бы получили молчаливый баг). Запомните правило: `override` подменяет слот в vtable, `new` создаёт **новый** слот и оставляет старый нетронутым. Поэтому `b2.Id()` возвращает `"Base"`, хотя фактический тип — `DerivedNew`.

- **Динамическая диспетчеризация работает только для `virtual`/`abstract`/`override`.** Обычный метод, вызванный по базовому типу, свяжется статически. Если вы добавите в `ReportComponent` невиртуальный `public string Tag() => "component";` и «перекроете» его в `CircleComponent` через `new`, то вызов `component.Tag()` по `List<ReportComponent>` всегда вернёт `"component"`. Это не баг полиморфизма — это его отсутствие.

- **Абстрактный класс нельзя инстанцировать.** `new ReportComponent("x")` не скомпилируется — и это благо: вы не сможете случайно создать «пустую» фигуру без реализации `Render()`. Переменная `ReportComponent c` при этом валидна и может ссылаться на любой производный объект.

- **Интерфейс не несёт состояния.** `IPriceable` не может иметь поле `_distance` — только сигнатуры. Поэтому `Shipping` хранит состояние сам, а интерфейс лишь требует, чтобы он умел вернуть `Cost()` и `Currency`. Класс может реализовать сколько угодно интерфейсов, но наследоваться только от одного класса — поэтому `Shipping` не наследуется от `ReportComponent`, а просто реализует `IPriceable`, и этого достаточно, чтобы попасть в `List<IPriceable>`.

- **Декоратор — это полиморфный паттерн.** `FramedComponent` наследуется от `ReportComponent` и **содержит** `ReportComponent`. Внешний код не знает, что внутри рамки — он просто вызывает `Render()` и `Cost()`. Именно так полиморфизм позволяет строить композиции: рамка вокруг рамки вокруг текста работает «из коробки».

- **Ловушка виртуального метода в конструкторе.** В C# конструктор базового класса выполняется **до** конструктора производного. Если базовый конструктор вызывает виртуальный `Init()`, среда диспетчирует вызов в **производную** реализацию (vtable уже построена), но поля производного класса ещё не инициализированы своими значениями — они в значениях по умолчанию (`0`, `null`, `false`). Отсюда неожиданные `NullReferenceException` или нули там, где вы ждёте данные. Решение — не вызывать виртуальные члены из конструктора; перенесите логику в фабричный метод `Create()`, который конструирует объект и только потом вызывает `Init()`.

- **`sealed` ускоряет и защищает.** Помечая листовые классы `sealed`, вы сообщаете компилятору и JIT, что дальнейших переопределений не будет — vtable для этих методов может быть «заморожена», а инварианты класса защищены от неожиданного подтипа. Это рекомендация из Best Practices урока.

- **`base.Method()` обязателен при инварианте.** Если базовый класс в `Cost()` логирует вызов или проверяет кэш, переопределение должно вызвать `base.Cost()`, иначе инвариант нарушится. В нашем `FramedComponent.Cost()` вызов `inner.Cost()` — это аналог `base`, потому что мы делегируем внутреннему объекту.

#### Критерии приёмки

- [ ] Проект `M05L05.Homework` создаётся командой `dotnet new console` и собирается через `dotnet build` без ошибок и без предупреждений.
- [ ] Определён интерфейс `IPriceable` с методом `Cost()` и свойством `Currency`.
- [ ] Определён абстрактный класс `ReportComponent`, реализующий `IPriceable`, с общим состоянием `_name` и абстрактным `Render()`.
- [ ] Созданы `sealed`-классы `CircleComponent`, `RectangleComponent`, `TextComponent`, каждый со своим `Render()` и `Cost()`.
- [ ] Создан декоратор `FramedComponent`, корректно оборачивающий любой `ReportComponent` и добавляющий рамку.
- [ ] Создан класс `Shipping`, реализующий `IPriceable`, но не наследующий `ReportComponent`.
- [ ] В `Program.cs` собран `List<ReportComponent>` с разными компонентами, включая вложенный `FramedComponent`.
- [ ] Собран `List<IPriceable>`, содержащий и компоненты, и `Shipping`; итоговая сумма вычисляется через `order.Sum(x => x.Cost())`.
- [ ] В коде отсутствуют ветвления по типу через `is`/`switch`/`GetType().Name` для выбора поведения.
- [ ] Реализован файл `DispatchExperiment.cs` с базовым классом и двумя производными (`override` и `new`).
- [ ] Вывод `b1.Id()` и `b2.Id()` зафиксирован и объяснён в комментарии не менее чем тремя предложениями.
- [ ] Реализован файл `CtorTrap.cs` с демонстрацией вызова виртуального метода из конструктора базового класса.
- [ ] Ловушка объяснена в комментарии и исправлена через статический фабричный метод `Create()`.
- [ ] Все денежные значения используют `decimal`.
- [ ] Вывод программы сохранён в `output.txt` и соответствует описанию.

#### Подсказки (без прямого ответа)

- Если вам кажется, что нужно написать `if (c is FramedComponent)`, вспомните: декоратор сам является `ReportComponent`, поэтому обычный полиморфный вызов `c.Render()` уже сделает правильную вещь. Ветвление по типу — это «анти-полиморфизм».
- В `FramedComponent.Render()` вам нужно вызвать `inner.Render()` и обернуть результат. Подумайте, что будет, если внутренний текст многострочный — рамку придется рисовать аккуратнее, но для упрощения можете считать текст однострочным.
- В эксперименте `override`/`new` ключевая подсказка: статический тип переменной — `Base`. Какой метод выберет компилятор, если в слоте `Id()` для типа `Base` лежит адрес `Base.Id()` (случай `new`) или адрес `DerivedOverride.Id()` (случай `override`)?
- В ловушке с конструктором вспомните порядок: сначала тело конструктора `LoggableBase`, потом тело конструктора производного класса. Виртуальный вызов в теле базового конструктора уже диспетчируется в производный — но поля производного ещё не успели получить свои значения из его конструктора.
- Для подсчёта суммы используйте `System.Linq` (`order.Sum(...)`). Убедитесь, что `ImplicitUsings` включает `System.Linq`, иначе добавьте `using System.Linq;`.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — Эталонное решение ДЗ M05-L05
// Reference solution for homework M05-L05
// Полиморфизм через абстрактный класс + интерфейс + декоратор
// Polymorphism via abstract class + interface + decorator

namespace M05L05.Homework;

// Интерфейс "can-do": всё, что имеет цену / "Can-do" interface: anything with a price
public interface IPriceable
{
    decimal Cost();          // Вернуть цену / Return the price
    string Currency { get; } // Валюта / Currency code
}

// Абстрактный базовый класс: общий "хребет" для элементов отчёта
// Abstract base: shared backbone for report components
public abstract class ReportComponent : IPriceable
{
    protected readonly string _name; // Общее состояние / Shared state

    protected ReportComponent(string name) => _name = name;

    // Абстрактный: у базы нет тела — производные обязаны реализовать
    // Abstract: no body in base — derived classes must implement
    public abstract string Render();

    // Виртуальный: базовая цена 0, можно переопределить
    // Virtual: base price 0, may be overridden
    public virtual decimal Cost() => 0m;

    public string Currency => "RUB"; // Реализация свойства интерфейса / Interface property impl

    // Полиморфный ToString: использует Render() и Cost() — диспетчеризация сработает
    // Polymorphic ToString: uses Render() and Cost() — dispatch will kick in
    public override string ToString() => $"{_name}: {Render()} [{Cost()} {Currency}]";
}

// Листовой класс — sealed: круг / Leaf class — sealed: circle
public sealed class CircleComponent : ReportComponent
{
    public double Radius { get; }
    public CircleComponent(double radius) : base("Круг/Circle") => Radius = radius;
    public override string Render() => $"○ площадь={Math.PI * Radius * Radius:F2}";
    public override decimal Cost() => (decimal)(Math.PI * Radius * Radius) * 0.5m;
}

public sealed class RectangleComponent : ReportComponent
{
    public double Width { get; }
    public double Height { get; }
    public RectangleComponent(double w, double h) : base("Прямоугольник/Rectangle")
    {
        Width = w; Height = h;
    }
    public override string Render() => $"□ площадь={Width * Height:F2}";
    public override decimal Cost() => (decimal)(Width * Height) * 0.4m;
}

public sealed class TextComponent : ReportComponent
{
    private readonly string _text;
    public TextComponent(string text) : base("Текст/Text") => _text = text;
    public override string Render() => _text;
    public override decimal Cost() => _text.Length * 0.1m;
}

// Декоратор: сам является ReportComponent и содержит ReportComponent
// Decorator: is itself a ReportComponent and holds a ReportComponent
public sealed class FramedComponent : ReportComponent
{
    private readonly ReportComponent _inner;
    public FramedComponent(ReportComponent inner) : base("Рамка/Frame") => _inner = inner;

    // Переопределение делегирует внутреннему и оборачивает
    // Override delegates to inner and wraps it
    public override string Render() => $"+--- {_inner.Render()} ---+";
    public override decimal Cost() => _inner.Cost() + 10m; // цена рамки / frame price
}

// Неродственный тип, реализующий только интерфейс / Unrelated type, interface only
public sealed class Shipping : IPriceable
{
    private readonly decimal _distance;
    public Shipping(decimal distance) => _distance = distance;
    public decimal Cost() => _distance * 2m;
    public string Currency => "RUB";
}

// Эксперимент: override vs new / Experiment: override vs new
public class Base
{
    public virtual string Id() => "Base";
}
public class DerivedOverride : Base
{
    public override string Id() => "DerivedOverride"; // подменяет слот vtable / replaces vtable slot
}
public class DerivedNew : Base
{
    public new string Id() => "DerivedNew"; // новый слот, старый нетронут / new slot, old untouched
}

public static class DispatchExperiment
{
    public static void Run()
    {
        Base b1 = new DerivedOverride();
        Base b2 = new DerivedNew();
        Console.WriteLine($"b1.Id() = {b1.Id()}"); // DerivedOverride — dynamic dispatch
        Console.WriteLine($"b2.Id() = {b2.Id()}"); // Base — статическая диспетчеризация / static dispatch
        // Почему? override подменяет адрес в слоте Id() типа Base, поэтому вызов по Base попадает в DerivedOverride.
        // new создаёт новый метод с тем же именем, но слот Base.Id() остаётся指向 Base — вызов по Base возвращает "Base".
    }
}

// Ловушка виртуального метода в конструкторе / Virtual-in-ctor trap
public abstract class LoggableBase
{
    protected LoggableBase()
    {
        Console.WriteLine("LoggableBase ctor: вызываем Init() / calling Init()");
        Init(); // ОПАСНО: производный ещё не инициализирован / DANGER: derived not ready
    }
    protected abstract void Init();
}

public sealed class DerivedLoggable : LoggableBase
{
    private readonly string _tag;
    // Поля производного класса на момент вызова Init() из базового ctor ещё null/0.
    public DerivedLoggable(string tag) => _tag = tag;
    protected override void Init()
    {
        // Если бы вызвали из базового ctor — _tag был бы null.
        Console.WriteLine($"Init(): tag = {_tag ?? "(null)"}");
    }
    // Фабричный метод, исправляющий ловушку / Factory method that fixes the trap
    public static DerivedLoggable Create(string tag)
    {
        var obj = new DerivedLoggable(tag); // сначала полностью конструируем
        obj.Init();                          // потом инициализируем логику
        return obj;
    }
}

// Точка входа / Entry point
public static class Program
{
    public static void Main()
    {
        var circle = new CircleComponent(3);
        var rect = new RectangleComponent(4, 5);
        var text = new TextComponent("Hello polymorphism");
        var framed = new FramedComponent(circle);

        List<ReportComponent> components = new() { circle, rect, text, framed };
        foreach (var c in components) Console.WriteLine(c);

        List<IPriceable> order = new() { circle, rect, text, framed, new Shipping(50) };
        var total = order.Sum(x => x.Cost());
        Console.WriteLine($"Итого / Total: {total} RUB");

        Console.WriteLine("--- Эксперимент override vs new ---");
        DispatchExperiment.Run();

        Console.WriteLine("--- Ловушка виртуального метода в ctor ---");
        // Опасный путь: new DerivedLoggable("demo") вызовет Init() из базового ctor с _tag == null.
        // Безопасный путь: фабричный метод.
        var safe = DerivedLoggable.Create("demo");
    }
}
```

**Разбор по строкам.** `IPriceable` — это «can-do»-контракт из урока: только сигнатуры, без состояния, поэтому `Shipping` может его реализовать, не влезая в иерархию `ReportComponent`. `ReportComponent` — абстрактный класс с общим состоянием `_name` и «спиной» поведения `ToString()`: он моделирует «is-a»-отношение для родственных элементов отчёта. `Render()` помечен `abstract`, потому что у базы нет осмысленной формулы — производные классы обязаны его реализовать. `Cost()` помечен `virtual` с телом по умолчанию `0m`, что позволяет отдельным классам не переопределять его, если они бесплатны (например, если бы добавили `SeparatorComponent`). Каждый листовой класс помечен `sealed` согласно Best Practices — это ускоряет диспетчеризацию и защищает инварианты. `FramedComponent` — это декоратор: он наследуется от `ReportComponent` и **содержит** `ReportComponent`, поэтому один и тот же вызов `c.Render()` по `List<ReportComponent>` корректно обходит и обычные компоненты, и обёрнутые. `Shipping` доказывает силу интерфейса: неродственный тип попадает в общий `List<IPriceable>`, и `order.Sum(x => x.Cost())` работает единообразно — это и есть полиморфизм, раскрывающийся в точке вызова. Эксперимент `DispatchExperiment` — ядро урока: `b1.Id()` возвращает `"DerivedOverride"`, потому что `override` подменил указатель в слоте vtable типа `Base`; `b2.Id()` возвращает `"Base"`, потому что `new` создал новый слот, а старый остался指向ать на `Base.Id()`. Это наглядно демонстрирует, что только `virtual`/`abstract`/`override` участвуют в динамической диспетчеризации. Ловушка `LoggableBase` показывает: виртуальный вызов `Init()` из базового конструктора диспетчируется в производный класс, но поля производного класса (`_tag`) ещё не инициализированы — будет `null`. Фабричный метод `Create()` решает проблему, разделяя «конструирование» и «инициализацию логики». Все цены используют `decimal`, потому что деньги требуют точного десятичного представления, а `double` накапливает ошибку.

#### Задания на углубление (бонус)

1. **Вложенные рамки.** Сделайте `FramedComponent` обёрнутым в другой `FramedComponent` и убедитесь, что `Render()` корректно рисует двойную рамку. Подумайте, почему это работает «бесплатно» — какой принцип полиморфизма здесь вступает в силу?
2. **Кэширование через `base`.** Добавьте в `ReportComponent` виртуальный метод `Cost()` с кэшем (словарь «аргумент → цена»). В `FramedComponent.Cost()` вызовите `base.Cost()` для проверки кэша. Объясните, когда `base` безопасен, а когда нет.
3. **Паттерн-полиморфизм через `switch` выражение.** Добавьте метод `string DescribeShape(ReportComponent c) => c switch { CircleComponent => ..., RectangleComponent => ..., _ => "other" };` и обсудите в комментарии, чем это отличается от классического полиморфизма и когда уместен каждый подход.
4. **Default interface methods.** Изучите самостоятельно (вне требований урока) default interface methods из C# 8+: добавьте в `IPriceable` метод `decimal CostWithTax() => Cost() * 1.2m;` с телом по умолчанию и объясните, почему это не нарушает идею «интерфейс без реализации».

---

## Statement in English / Постановка на английском

#### Context & motivation

You are a backend developer in a small team building a tiny engine for generating ASCII reports and invoices. The product is already used by two customers, and now marketing comes back with a request: "Add borders, add a logo, add text captions, add a way to compute the total price of an order composed of many different components." If you start writing `if (item is Rectangle)` branches or a `switch (item.GetType().Name)` cascade, your code will turn into spaghetti by the third feature. Polymorphism is precisely the tool that lets you add new component types without touching existing client code.

In this assignment you will design a hierarchy of "report components" (`ReportComponent`), each of which knows how to render itself (`Render()`) and how to report its price (`Cost()`). Some components are geometric shapes (a circle, a rectangle), some are text captions, and some are decorative frames wrapping other components. In parallel, you will implement an `IPriceable` interface and see that through it you can uniformly sum the total of an order that mixes shapes, frames, and a foreign `Shipping` type (an external class that does **not** inherit from `ReportComponent`).

The teaching moment is not "write a lot of classes" — it is to observe how **the very same line** `foreach (var c in components) total += c.Cost();` calls different implementations depending on the actual runtime type of each object, and how replacing `override` with `new` silently breaks that magic. You will also meet the classic trap of calling a virtual method from a base constructor and must work around it using a factory method.

#### What to do step by step

1. **Create the project.** From a terminal run `dotnet new console -n M05L05.Homework -o M05L05.Homework --framework net8.0`. Move into the folder: `cd M05L05.Homework`. Open `Program.cs` and remove the default `Console.WriteLine("Hello, World!");`. Make sure your `.csproj` has `<TargetFramework>net8.0</TargetFramework>` along with `<Nullable>enable</Nullable>` and `<ImplicitUsings>enable</ImplicitUsings>`.

2. **Define the `IPriceable` interface.** In `IPriceable.cs` declare an interface with one method `decimal Cost();` and one property `string Currency { get; }`. This is a "can-do" contract: anything that has a price must be able to return it.

3. **Define the abstract class `ReportComponent`.** In `ReportComponent.cs` create an abstract base class that implements `IPriceable`. It must contain:
   - a protected field `protected readonly string _name;` initialised in the constructor;
   - an abstract method `public abstract string Render();`
   - a virtual method `public virtual decimal Cost() => 0m;` (the base price is zero);
   - the property `public string Currency => "RUB";`
   - an overridden `public override string ToString() => $"{_name}: {Render()} [{Cost()} {Currency}]"`.

4. **Create the derived classes.** In separate files add: `CircleComponent` (parameter: radius; `Render()` returns `○` plus the area; `Cost()` = area × 0.5m), `RectangleComponent` (width and height; `Render()` returns `□` plus the area; `Cost()` = area × 0.4m), and `TextComponent` (a text string; `Cost()` = length × 0.1m; `Render()` returns the text itself). Mark all three classes `sealed`.

5. **Add the `FramedComponent` decorator.** This class wraps another `ReportComponent` and draws a frame around its `Render()` output. It must inherit from `ReportComponent`, accept the inner component in the constructor, override `Render()` so that it wraps the inner text in `+--- ... ---+`, and override `Cost()` as `inner.Cost() + 10m` (the price of the frame). **This is the crux:** the decorator is itself a component, so it can sit in arrays and lists alongside plain components — polymorphism in action.

6. **Implement the foreign type `Shipping`.** In `Shipping.cs` create a class that does **not** inherit from `ReportComponent` but implements `IPriceable`. Give it a `decimal _distance` field and a `Cost()` that returns `_distance * 2m`. Set `Currency` to `"RUB"`. This shows that interfaces let you work with unrelated types inside the same loop.

7. **Wire everything up in `Program.cs`.** Build a `List<ReportComponent> components` containing a circle, a rectangle, a text, and a framed circle (`new FramedComponent(new CircleComponent(3))`). Print each one with `Console.WriteLine(component)`. Then build a `List<IPriceable> order = new() { circle, rectangle, text, framed, new Shipping(50) };` and compute `var total = order.Sum(x => x.Cost());`. Print `Total: {total} RUB`.

8. **Run the `override` vs `new` experiment.** In a separate file `DispatchExperiment.cs` create a base class `Base { public virtual string Id() => "Base"; }` and two derived classes: `DerivedOverride : Base { public override string Id() => "DerivedOverride"; }` and `DerivedNew : Base { public new string Id() => "DerivedNew"; }`. In `Run()` declare `Base b1 = new DerivedOverride(); Base b2 = new DerivedNew();` and print `b1.Id()` and `b2.Id()`. **Record in a comment what output you got and why.** This is the most important part of the homework — it proves the difference between dynamic dispatch (`override`) and method hiding (`new`).

9. **Demonstrate the virtual-method-in-constructor trap.** In `CtorTrap.cs` create a base class `LoggableBase` whose constructor calls `protected virtual void Init()` — and override `Init()` in a derived class so that it prints a derived-class field. Run it, observe that the field is still at its default value (`null` or `0`), and explain in a comment why this happens. Then rewrite the code using a static factory method `Create()` that first constructs the object and only then calls `Init()` — and verify that everything now works correctly.

10. **Run and capture the output.** Execute `dotnet run`. Copy the output into a file `output.txt` next to the project (you can use `dotnet run > output.txt` in PowerShell, or save it manually). Make sure the total is computed correctly and that the `override`/`new` experiment produces exactly the result you predicted in the comment.

#### Requirements

- The project builds with `dotnet build` without warnings (at minimum, `CS0108` must be intentional — if you use `new` anywhere, add a `// intentional` comment).
- The code targets C# 12 / .NET 8: file-scoped namespaces are allowed, top-level statements may live in `Program.cs`, `init`-only setters are fine if you wish, and target-typed `new()` is welcome.
- All leaf classes (`CircleComponent`, `RectangleComponent`, `TextComponent`, `FramedComponent`, `Shipping`) are marked `sealed`, except `FramedComponent` which **may** stay unsealed if you plan further decorator inheritance (justify the choice in a comment).
- The hierarchy leans on an **abstract class** for the related "report components" (shared state `_name`, shared behaviour backbone `ToString`) and on the **interface** `IPriceable` to bring in the unrelated `Shipping`.
- There are **no** type-based branches via `is`/`switch`/`GetType().Name` in the code — every behavioural choice goes through polymorphic calls. If you are tempted to write `if (c is FramedComponent)` — stop and think about which virtual method you should add to avoid it.
- The `override` vs `new` experiment is **mandatory**: its output is captured and explained in a comment of at least three sentences.
- The virtual-method-in-constructor trap is demonstrated and fixed via a factory method; the comment explains why the derived fields are at their defaults at that moment.
- All monetary values use `decimal`, not `double` (price is money, not a measurement).
- The code compiles and runs; the program output matches the description in "Acceptance criteria".

#### Pitfalls

- **`virtual`/`override` vs `new` — the lesson's headline.** If you accidentally write `public new string Render()` instead of `override` in `FramedComponent`, then iterating over a `List<ReportComponent>` will call the **base** `ReportComponent.Render()` implementation (which in our case is abstract, so the compiler will stop you — but if it were virtual with a body, you would get a silent bug). Remember the rule: `override` replaces the slot in the vtable; `new` creates a **new** slot and leaves the old one untouched. That is why `b2.Id()` returns `"Base"` even though the actual type is `DerivedNew`.

- **Dynamic dispatch only applies to `virtual`/`abstract`/`override`.** A plain non-virtual method called through a base type binds statically. If you add a non-virtual `public string Tag() => "component";` to `ReportComponent` and "shadow" it in `CircleComponent` via `new`, then calling `component.Tag()` through a `List<ReportComponent>` will always return `"component"`. This is not a polymorphism bug — it is the absence of polymorphism.

- **An abstract class cannot be instantiated.** `new ReportComponent("x")` will not compile — and that is a good thing: you cannot accidentally create an "empty" shape with no `Render()` implementation. A variable `ReportComponent c` is still valid and may refer to any derived object.

- **An interface carries no state.** `IPriceable` cannot have a `_distance` field — only signatures. So `Shipping` stores its state itself, while the interface merely demands that it return `Cost()` and `Currency`. A class may implement any number of interfaces but inherit from only one class — that is why `Shipping` does not inherit from `ReportComponent`; implementing `IPriceable` is enough to land in a `List<IPriceable>`.

- **The decorator is a polymorphic pattern.** `FramedComponent` inherits from `ReportComponent` and **contains** a `ReportComponent`. Outside code does not know what is inside the frame — it just calls `Render()` and `Cost()`. That is how polymorphism enables composition: a frame around a frame around a text works out of the box.

- **The virtual-method-in-constructor trap.** In C# the base constructor runs **before** the derived constructor body. If the base constructor calls a virtual `Init()`, the runtime dispatches the call to the **derived** implementation (the vtable is already built), but the derived fields have not yet been assigned their constructor values — they still hold defaults (`0`, `null`, `false`). This leads to surprising `NullReferenceException`s or zeros where you expected data. The fix is to never call virtual members from a constructor; move the logic into a factory method `Create()` that constructs the object first and only then calls `Init()`.

- **`sealed` speeds things up and protects.** Marking leaf classes `sealed` tells the compiler and JIT that no further overrides will exist — the vtable for those methods can be "frozen", and class invariants are protected from unexpected subtypes. This is a Best Practice straight from the lesson.

- **`base.Method()` is mandatory when the base enforces an invariant.** If the base `Cost()` logs the call or checks a cache, the override must call `base.Cost()`, otherwise the invariant is broken. In our `FramedComponent.Cost()` the call to `inner.Cost()` plays the same role as `base` because we delegate to the wrapped object.

#### Acceptance criteria

- [ ] The project `M05L05.Homework` is created with `dotnet new console` and builds via `dotnet build` without errors and without warnings.
- [ ] The `IPriceable` interface is defined with a `Cost()` method and a `Currency` property.
- [ ] The abstract class `ReportComponent` implements `IPriceable`, has shared state `_name`, and declares an abstract `Render()`.
- [ ] The `sealed` classes `CircleComponent`, `RectangleComponent`, and `TextComponent` are present, each with its own `Render()` and `Cost()`.
- [ ] The `FramedComponent` decorator correctly wraps any `ReportComponent` and draws a frame.
- [ ] The `Shipping` class implements `IPriceable` but does not inherit from `ReportComponent`.
- [ ] `Program.cs` builds a `List<ReportComponent>` with different components, including a nested `FramedComponent`.
- [ ] A `List<IPriceable>` is built containing both components and `Shipping`; the total is computed via `order.Sum(x => x.Cost())`.
- [ ] No type-based branching via `is`/`switch`/`GetType().Name` is used to choose behaviour.
- [ ] `DispatchExperiment.cs` exists with a base class and two derived classes (`override` and `new`).
- [ ] The output of `b1.Id()` and `b2.Id()` is captured and explained in a comment of at least three sentences.
- [ ] `CtorTrap.cs` demonstrates a virtual method call from a base-class constructor.
- [ ] The trap is explained in a comment and fixed using a static factory method `Create()`.
- [ ] All monetary values use `decimal`.
- [ ] The program output is saved to `output.txt` and matches the description.

#### Hints (no direct answer)

- If you feel you must write `if (c is FramedComponent)`, remember: the decorator is itself a `ReportComponent`, so a plain polymorphic call to `c.Render()` already does the right thing. Branching on type is "anti-polymorphism".
- In `FramedComponent.Render()` you need to call `inner.Render()` and wrap the result. Think about what happens if the inner text is multi-line — the frame must be drawn carefully — but for simplicity you may assume single-line text.
- In the `override`/`new` experiment the key hint is: the static type of the variable is `Base`. Which method does the compiler pick if the `Id()` slot for type `Base` holds the address of `Base.Id()` (the `new` case) versus the address of `DerivedOverride.Id()` (the `override` case)?
- For the constructor trap, recall the order: the body of `LoggableBase`'s constructor runs first, then the body of the derived constructor. A virtual call inside the base constructor already dispatches to the derived class — but the derived fields have not yet received their constructor-assigned values.
- For the sum, use `System.Linq` (`order.Sum(...)`). Make sure `ImplicitUsings` includes `System.Linq`, or add `using System.Linq;` explicitly.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — Reference solution for homework M05-L05
// Polymorphism via abstract class + interface + decorator

namespace M05L05.Homework;

// "Can-do" interface: anything that has a price
public interface IPriceable
{
    decimal Cost();          // Return the price
    string Currency { get; } // Currency code
}

// Abstract base: shared backbone for report components
public abstract class ReportComponent : IPriceable
{
    protected readonly string _name; // Shared state

    protected ReportComponent(string name) => _name = name;

    // Abstract: no body in base — derived classes must implement
    public abstract string Render();

    // Virtual: base price 0, may be overridden
    public virtual decimal Cost() => 0m;

    public string Currency => "RUB"; // Interface property impl

    // Polymorphic ToString: uses Render() and Cost() — dispatch will kick in
    public override string ToString() => $"{_name}: {Render()} [{Cost()} {Currency}]";
}

// Leaf class — sealed: circle
public sealed class CircleComponent : ReportComponent
{
    public double Radius { get; }
    public CircleComponent(double radius) : base("Circle") => Radius = radius;
    public override string Render() => $"○ area={Math.PI * Radius * Radius:F2}";
    public override decimal Cost() => (decimal)(Math.PI * Radius * Radius) * 0.5m;
}

public sealed class RectangleComponent : ReportComponent
{
    public double Width { get; }
    public double Height { get; }
    public RectangleComponent(double w, double h) : base("Rectangle")
    {
        Width = w; Height = h;
    }
    public override string Render() => $"□ area={Width * Height:F2}";
    public override decimal Cost() => (decimal)(Width * Height) * 0.4m;
}

public sealed class TextComponent : ReportComponent
{
    private readonly string _text;
    public TextComponent(string text) : base("Text") => _text = text;
    public override string Render() => _text;
    public override decimal Cost() => _text.Length * 0.1m;
}

// Decorator: is itself a ReportComponent and holds a ReportComponent
public sealed class FramedComponent : ReportComponent
{
    private readonly ReportComponent _inner;
    public FramedComponent(ReportComponent inner) : base("Frame") => _inner = inner;

    // Override delegates to inner and wraps it
    public override string Render() => $"+--- {_inner.Render()} ---+";
    public override decimal Cost() => _inner.Cost() + 10m; // frame price
}

// Unrelated type, implements only the interface
public sealed class Shipping : IPriceable
{
    private readonly decimal _distance;
    public Shipping(decimal distance) => _distance = distance;
    public decimal Cost() => _distance * 2m;
    public string Currency => "RUB";
}

// Experiment: override vs new
public class Base
{
    public virtual string Id() => "Base";
}
public class DerivedOverride : Base
{
    public override string Id() => "DerivedOverride"; // replaces the vtable slot
}
public class DerivedNew : Base
{
    public new string Id() => "DerivedNew"; // new slot, old one untouched
}

public static class DispatchExperiment
{
    public static void Run()
    {
        Base b1 = new DerivedOverride();
        Base b2 = new DerivedNew();
        Console.WriteLine($"b1.Id() = {b1.Id()}"); // DerivedOverride — dynamic dispatch
        Console.WriteLine($"b2.Id() = {b2.Id()}"); // Base — static dispatch
        // Why? override replaces the address in the Base.Id() slot, so a call through Base lands in DerivedOverride.
        // new creates a brand-new method with the same name, but the Base.Id() slot still points at Base — a call through Base returns "Base".
    }
}

// Virtual-in-ctor trap
public abstract class LoggableBase
{
    protected LoggableBase()
    {
        Console.WriteLine("LoggableBase ctor: calling Init()");
        Init(); // DANGER: derived class is not ready yet
    }
    protected abstract void Init();
}

public sealed class DerivedLoggable : LoggableBase
{
    private readonly string _tag;
    // At the moment Init() is called from the base ctor, _tag is still null.
    public DerivedLoggable(string tag) => _tag = tag;
    protected override void Init()
    {
        Console.WriteLine($"Init(): tag = {_tag ?? "(null)"}");
    }
    // Factory method that fixes the trap
    public static DerivedLoggable Create(string tag)
    {
        var obj = new DerivedLoggable(tag); // fully construct first
        obj.Init();                          // then run the logic
        return obj;
    }
}

// Entry point
public static class Program
{
    public static void Main()
    {
        var circle = new CircleComponent(3);
        var rect = new RectangleComponent(4, 5);
        var text = new TextComponent("Hello polymorphism");
        var framed = new FramedComponent(circle);

        List<ReportComponent> components = new() { circle, rect, text, framed };
        foreach (var c in components) Console.WriteLine(c);

        List<IPriceable> order = new() { circle, rect, text, framed, new Shipping(50) };
        var total = order.Sum(x => x.Cost());
        Console.WriteLine($"Total: {total} RUB");

        Console.WriteLine("--- Experiment override vs new ---");
        DispatchExperiment.Run();

        Console.WriteLine("--- Virtual-in-ctor trap ---");
        // Dangerous path: new DerivedLoggable("demo") would call Init() from the base ctor with _tag == null.
        // Safe path: the factory method.
        var safe = DerivedLoggable.Create("demo");
    }
}
```

**Line-by-line walk-through.** `IPriceable` is the "can-do" contract from the lesson: only signatures, no state, so `Shipping` can implement it without joining the `ReportComponent` hierarchy. `ReportComponent` is an abstract class with shared state `_name` and a behaviour backbone `ToString()`: it models the "is-a" relationship for the related report components. `Render()` is marked `abstract` because the base has no meaningful formula — derived classes are forced to implement it. `Cost()` is marked `virtual` with a default body of `0m`, which lets individual classes skip overriding it if they are free (for example, a hypothetical `SeparatorComponent`). Every leaf class is marked `sealed` per the Best Practices — this speeds up dispatch and protects invariants. `FramedComponent` is a decorator: it inherits from `ReportComponent` and **holds** a `ReportComponent`, so the very same `c.Render()` call over a `List<ReportComponent>` correctly handles both plain and wrapped components. `Shipping` proves the power of interfaces: an unrelated type lands in a shared `List<IPriceable>`, and `order.Sum(x => x.Cost())` works uniformly — this is polymorphism exercised at the call site. The `DispatchExperiment` is the heart of the lesson: `b1.Id()` returns `"DerivedOverride"` because `override` replaced the pointer in the `Base` vtable slot; `b2.Id()` returns `"Base"` because `new` created a brand-new slot while the old one still points at `Base.Id()`. This vividly shows that only `virtual`/`abstract`/`override` participate in dynamic dispatch. The `LoggableBase` trap shows that a virtual `Init()` call from the base constructor dispatches to the derived class, but the derived fields (`_tag`) are not yet initialised — you get `null`. The factory method `Create()` fixes the problem by separating "construction" from "logic initialisation". All prices use `decimal` because money demands exact decimal representation, while `double` accumulates rounding error.

#### Going deeper (bonus)

1. **Nested frames.** Wrap a `FramedComponent` inside another `FramedComponent` and verify that `Render()` draws a double frame correctly. Think about why this works "for free" — which polymorphism principle is at play here?
2. **Caching via `base`.** Add a virtual `Cost()` to `ReportComponent` that caches results in a dictionary keyed by some argument. In `FramedComponent.Cost()` call `base.Cost()` to consult the cache. Explain when `base` is safe and when it is not.
3. **Pattern-based polymorphism with `switch`.** Add a method `string DescribeShape(ReportComponent c) => c switch { CircleComponent => ..., RectangleComponent => ..., _ => "other" };` and discuss in a comment how this differs from classic polymorphism and when each approach is appropriate.
4. **Default interface methods.** Study on your own (beyond the lesson scope) the default interface methods introduced in C# 8+: add `decimal CostWithTax() => Cost() * 1.2m;` with a body to `IPriceable` and explain why this does not violate the "interface without implementation" idea.

---

#### Чек-лист сдачи / Submission checklist

- [ ] (RU) Проект `M05L05.Homework` собирается без ошибок и предупреждений.
- [ ] (RU) Интерфейс `IPriceable` и абстрактный класс `ReportComponent` реализованы.
- [ ] (RU) Созданы `sealed`-классы фигур и текста, а также декоратор `FramedComponent`.
- [ ] (RU) Класс `Shipping` реализует интерфейс, не наследуясь от `ReportComponent`.
- [ ] (RU) В `Program.cs` собраны `List<ReportComponent>` и `List<IPriceable>`, итог посчитан через `Sum`.
- [ ] (RU) Эксперимент `override` vs `new` присутствует, вывод объяснён в комментарии.
- [ ] (RU) Ловушка виртуального метода в конструкторе показана и исправлена через фабричный метод.
- [ ] (RU) В коде нет ветвлений по типу через `is`/`switch`/`GetType().Name`.
- [ ] (RU) Вывод сохранён в `output.txt`.
- [ ] (EN) The project `M05L05.Homework` builds without errors or warnings.
- [ ] (EN) The `IPriceable` interface and the `ReportComponent` abstract class are implemented.
- [ ] (EN) The `sealed` shape and text classes, plus the `FramedComponent` decorator, are present.
- [ ] (EN) The `Shipping` class implements the interface without inheriting from `ReportComponent`.
- [ ] (EN) `Program.cs` builds both `List<ReportComponent>` and `List<IPriceable>`; the total is computed via `Sum`.
- [ ] (EN) The `override` vs `new` experiment is present and its output is explained in a comment.
- [ ] (EN) The virtual-method-in-constructor trap is shown and fixed with a factory method.
- [ ] (EN) No type-based branching via `is`/`switch`/`GetType().Name` exists in the code.
- [ ] (EN) The output is saved to `output.txt`.

#### Ресурсы / Resources

- [Microsoft Learn — Polymorphism](https://learn.microsoft.com/dotnet/csharp/fundamentals/object-oriented/polymorphism)
- [Microsoft Learn — Abstract and sealed classes](https://learn.microsoft.com/dotnet/csharp/fundamentals/object-oriented/abstract-and-sealed)
- [Microsoft Learn — Interfaces](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/interfaces)
- [Microsoft Learn — Knowing when to use override and new keywords](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/knowing-when-to-use-override-and-new-keywords)
