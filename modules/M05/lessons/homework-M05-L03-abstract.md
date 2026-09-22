---
[← К уроку M05-L03](lesson-M05-L03-abstract.md) | [⬆ К модулю M05](../README.md) | [Следующее ДЗ →](homework-M05-L04-interfaces.md)
---

### Домашнее задание M05-L03: Абстрактные классы и методы / Homework M05-L03: Abstract classes and methods

**Урок / Lesson:** M05-L03
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться проектировать иерархию типов вокруг абстрактного базового класса, корректно переопределять абстрактные и виртуальные члены, использовать `protected` конструкторы, `sealed` наследников, промежуточные абстрактные классы и композицию с интерфейсами для безопасной и переиспользуемой архитектуры. (EN) Learn to design a type hierarchy around an abstract base class, correctly override abstract and virtual members, use `protected` constructors, `sealed` descendants, intermediate abstract classes and interface composition for a safe and reusable architecture.

#### Связь с уроком / Connection to the lesson
(RU) ДЗ напрямую закрепляет тему урока: вы построите абстрактный класс `Shape` с абстрактным методом `Area()` и виртуальным `Describe()`, как в примере, но с реальной геометрией. Вы пройдёте через каскад абстрактности, создав промежуточный абстрактный класс `Polygon`, и закроете контракт интерфейсом `I measurable`. Все частые ошибки урока — `new Shape()`, пропуск `override`, публичный конструктор, забытый `base(...)` — будут отражены в критериях приёмки. (EN) The homework directly reinforces the lesson topic: you will build an abstract `Shape` class with an abstract `Area()` and a virtual `Describe()`, as in the example, but with real geometry. You will traverse the abstractness cascade by introducing an intermediate abstract `Polygon` class, and you will express the contract with an `I measurable` interface. Every common mistake from the lesson — `new Shape()`, missing `override`, a public constructor, a forgotten `base(...)` — is captured in the acceptance criteria.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Представьте, что вы разрабатываете небольшой геометрический движок для учебной CAD-библиотеки. В нём должно быть единое представление «фигура», над которым можно выполнять обобщённые операции: вычислять площадь, печатать описание, группировать в коллекции и суммировать площади. При этом «просто фигура» как самостоятельный объект не имеет смысла — площадь у абстрактной концепции не существует, и инстанцировать её нельзя. Это ровно тот случай, когда абстрактный класс подходит лучше интерфейса: есть общее состояние (имя), есть общая механика (печать, форматирование), но есть и контракт без реализации по умолчанию (площадь зависит от конкретной формы).

Дополнительно в библиотеке присутствуют многоугольники — особое подсемейство фигур, у которых появляется новое обязательное свойство «количество вершин». Это естественным образом приводит к промежуточному абстрактному классу `Polygon`, который добавляет новый абстрактный член и продолжает каскад абстрактности вниз. Наконец, часть типов (например, круг и прямоугольник) хочется сделать «закрытыми» от дальнейшего наследования, чтобы помочь JIT-компилятору и явно зафиксировать намерение. Внешний потребитель хочет работать с фигурами через стабильный контракт, не зависящий от конкретной иерархии, — для этого вы добавите интерфейс `I measurable` с методом `Area()`. Так получится многослойная архитектура: контракт (интерфейс), общая механика (абстрактный `Shape`), специфика многоугольников (абстрактный `Polygon`) и конкретные классы (`Circle`, `Rectangle`, `Triangle`, `RegularPentagon`).

Задание построено так, чтобы каждое решение опиралось на конкретный концепт из урока. Вы не просто напишете код — вы обоснуете, почему здесь `abstract`, а не `virtual`, почему конструктор `protected`, а не `public`, и почему один класс помечен `sealed`, а другой оставлен открытым для дальнейшего наследования.

#### Что нужно сделать (пошагово)

1. Создайте новый проект консольного приложения .NET 8 с именем `GeometryEngine`. Выполните команды:
   ```
   dotnet new console -n GeometryEngine -o GeometryEngine --framework net8.0
   cd GeometryEngine
   dotnet build
   ```
   Убедитесь, что в `GeometryEngine.csproj` указано `<TargetFramework>net8.0</TargetFramework>` и `<LangVersion>latest</LangVersion>` (при необходимости добавьте последний вручную через `edit`).

2. В файле `Program.cs` организуйте код с использованием top-level statements. Вверху разместите `using System;` (или полагайтесь на неявные global usings — в .NET 8 они включены по умолчанию). Определите все типы в одном файле для удобства проверки, но логически сгруппируйте их комментариями-разделителями: «Контракт», «Абстрактный базовый класс», «Промежуточный абстрактный класс», «Конкретные наследники».

3. Объявите интерфейс `I measurable` с одним методом `double Area();`. Этот интерфейс — внешний контракт, который будет реализовывать абстрактный класс `Shape`.

4. Объявите абстрактный класс `Shape`, реализующий `I measurable`:
   - свойство `public string Name { get; }` — конкретное, общее для всех;
   - `protected Shape(string name)` конструктор, присваивающий `Name` через стрелочную форму;
   - абстрактный метод `public abstract double Area();` — без тела, реализация обязательна в наследниках;
   - виртуальный метод `public virtual string Describe() => $"{Name}: area = {Area():F2}";` — даёт реализацию по умолчанию, наследник может расширить;
   - конкретный метод `public void Print() => Console.WriteLine(Describe());` — общий код, не дублируется.

5. Объявите промежуточный абстрактный класс `Polygon : Shape`:
   - `protected Polygon(string name) : base(name) { }` — корректно вызывает базовый конструктор;
   - новый абстрактный член `public abstract int VertexCount { get; }` — каскад абстрактности продолжается, так как у многоугольника число вершин обязательно, но зависит от конкретного вида.

6. Создайте конкретные классы, каждый `sealed` (если не планируется дальнейшее наследование):
   - `Circle : Shape` — поле `Radius`, переопределённый `Area()` через `Math.PI * Radius * Radius`, переопределённый `Describe()` с вызовом `base.Describe()` и дополнением `, radius = {Radius}`;
   - `Rectangle : Shape` — поля `Width`, `Height`, `Area()` через `Width * Height`;
   - `Triangle : Polygon` — переопределяет и `VertexCount => 3`, и `Area()` (через формулу Герона по трём сторонам, передаваемым в конструктор);
   - `RegularPentagon : Polygon` — переопределяет `VertexCount => 5` и `Area()` через формулу площади правильного пятиугольника со стороной `a`: `(5 * a * a) / (4 * Math.Tan(Math.PI / 5))`.

7. В top-level части `Program.cs` создайте коллекцию фигур, используя коллекционное выражение C# 12:
   ```csharp
   List<Shape> shapes = [new Circle(2), new Rectangle(3, 4), new Triangle(3, 4, 5), new RegularPentagon(2)];
   ```
   Затем выполните полиморфный обход и суммирование площадей, используя pattern matching для подсчёта вершин у многоугольников:
   ```csharp
   double totalArea = 0;
   foreach (var s in shapes)
   {
       s.Print();
       totalArea += s.Area();
       if (s is Polygon p)
           Console.WriteLine($"  polygon vertices: {p.VertexCount}");
   }
   Console.WriteLine($"Total area: {totalArea:F2}");
   ```

8. Запустите проект командой `dotnet run` и зафиксируйте ожидаемый вывод. Для круга с радиусом 2 площадь ≈ 12.57; для прямоугольника 3×4 — 12.00; для треугольника со сторонами 3, 4, 5 — 6.00 (прямоугольный, формула Герона); для правильного пятиугольника со стороной 2 — ≈ 13.76. Итоговая сумма должна быть ≈ 44.33.

9. В отдельном комментарии в конце файла явно укажите, какие члены `abstract`, какие `virtual` и почему. Подтвердите, что попытка `new Shape("x")` вызывает ошибку компиляции CS0144 (Cannot create an instance of the abstract class or interface), — приведите закомментированную строку с этой попыткой рядом с пояснением.

10. (Проверка устойчивости.) Временно уберите `override` у `Area()` в `Circle` и выполните `dotnet build`: подтвердите, что компилятор выдаёт ошибку CS0534 («не переопределяет унаследованный абстрактный член»). Верните `override` обратно и убедитесь, что сборка проходит. Запишите наблюдение в комментарий у метода.

#### Требования к решению

Решение должно компилироваться без предупреждений уровня 4 и выше под .NET 8 с C# 12. Все абстрактные члены базового класса должны быть переопределены в каждом конкретном классе; ни одного «висящего» `abstract` метода без `override` в неабстрактном наследнике быть не должно. Конструкторы абстрактных классов обязаны быть `protected`, а не `public` — это документирует намерение и исключает ложное впечатление, будто класс можно инстанцировать. Каждый конкретный наследник, для которого не предусмотрено дальнейшее наследование, помечается `sealed`. Базовое состояние (свойство `Name`) инициализируется строго через `base(...)` — никаких прямых присваиваний в обход конструктора базового класса.

Код должен использовать современные возможности C# 12 там, где это уместно: top-level statements для точки входа, коллекционные выражения для инициализации списка фигур, стрелочные (expression-bodied) члены для коротких методов и свойств, форматирование с фиксированной точкой (`F2`) для вывода площадей. Имена типов — PascalCase, имена полей и локальных переменных — camelCase. Треугольник должен быть валидирован: если стороны не образуют треугольник (нарушено неравенство треугольника), конструктор бросает `ArgumentException` с понятным сообщением — это защищает инвариант базового класса. Все вычисления площадей должны быть аналитически точными, без эвристик и `0`-заглушек, как в упрощённом примере урока.

#### Тонкости и подводные камни

Главная тонкость — строгое различие `abstract` и `virtual`. `Area()` объявлен `abstract`, потому что у абстрактной «фигуры» нет осмысленной площади по умолчанию: круг, прямоугольник и треугольник считают её принципиально по-разному, и оставить «заглушку» значило бы ввести ошибку в контракт. `Describe()`, напротив, `virtual`: базовая реализация `"{Name}: area = {Area():F2}"` уже осмысленна для любой фигуры, но конкретный наследник (например, `Circle`) может её расширить, вызвав `base.Describe()` и добавив свои данные. Путаница этих двух модификаторов — самая частая ошибка: если поставить `virtual` там, где нужен обязательный контракт, наследник «забудет» переопределить и появится молчаливый баг; если поставить `abstract` там, где есть разумная реализация, придётся дублировать код в каждом наследнике.

Вторая тонкость — каскад абстрактности через `Polygon`. Этот класс добавляет новый абстрактный член `VertexCount`, поэтому сам обязан быть `abstract`, хотя он и переопределяет... нет, он не переопределяет `Area()` — а значит, `Area()` остаётся не закрытым. В уроке чётко сказано: пока хоть один `abstract` метод не переопределён, производный класс тоже обязан быть `abstract`. Поэтому `Polygon` остаётся абстрактным, и только конкретные `Triangle` и `RegularPentagon` закрывают и `Area()`, и `VertexCount`, после чего становятся инстанцируемыми и помечаются `sealed`. Пропуск `abstract` у `Polygon` — типичная ошибка компиляции CS0534.

Третья тонкость — конструкторы. `protected Shape(string name)` нельзя вызвать снаружи, но можно из `Circle`, `Rectangle` и `Polygon` через `: base(name)`. Если сделать конструктор `public`, внешнему коду покажется, что `Shape` можно создать, — это противоречит самой идее абстрактного класса. Четвёртая тонкость — забытый `base(...)`: если наследник не вызовет базовый конструктор, `Name` останется `null`, и `Describe()` напечатает `": area = ..."`. Пятая тонкость — избыточное наследование: помечайте `sealed` всё, что не предназначено для дальнейшего наследования; это и сигнал читателю, и подсказка JIT о девиртуализации. Шестая — композиция с интерфейсом: `I measurable` позволяет внешнему коду зависеть только от контракта площади, не зная о внутренней иерархии, что упрощает тестирование и подстановки.

#### Критерии приёмки

- [ ] Проект `GeometryEngine` создан под .NET 8 с C# 12 и собирается без ошибок и предупреждений уровня 4+.
- [ ] Интерфейс `I measurable` объявлен и содержит `double Area();`.
- [ ] Абстрактный класс `Shape` реализует `I measurable` и помечен `abstract`.
- [ ] `Shape` содержит свойство `Name`, `protected` конструктор, абстрактный `Area()`, виртуальный `Describe()` и конкретный `Print()`.
- [ ] Попытка `new Shape("x")` закомментирована с пояснением ошибки CS0144.
- [ ] Промежуточный абстрактный класс `Polygon : Shape` добавляет абстрактное свойство `VertexCount`.
- [ ] `Polygon` помечен `abstract` (обоснованно: не закрывает `Area()`).
- [ ] `Circle` переопределяет `Area()` и `Describe()` с вызовом `base.Describe()`.
- [ ] `Rectangle` переопределяет `Area()` через `Width * Height`.
- [ ] `Triangle` переопределяет `VertexCount` и `Area()` (формула Герона) и валидирует стороны.
- [ ] `RegularPentagon` переопределяет `VertexCount` и `Area()` по формуле правильного пятиугольника.
- [ ] Все конкретные наследники помечены `sealed`.
- [ ] Все конструкторы абстрактных классов — `protected`.
- [ ] В `Program.cs` используется коллекционное выражение `[ ... ]` для списка фигур.
- [ ] Полиморфный обход через `foreach` вызывает `Print()`, суммирует `Area()` и через `is Polygon` выводит `VertexCount`.
- [ ] Вывод `dotnet run` содержит площади ≈ 12.57, 12.00, 6.00, 13.76 и сумму ≈ 44.33.
- [ ] Временно убранный `override` у `Area()` в `Circle` воспроизводит ошибку CS0534 (зафиксировано в комментарии).

#### Подсказки (без прямого ответа)

- Вспомните чертёж автомобиля из урока: «просто транспортное средство» не существует. Какая фигура здесь играет ту же роль?
- Формула Герона для треугольника со сторонами `a, b, c`: полупериметр `s = (a+b+c)/2`, площадь `sqrt(s(s-a)(s-b)(s-c))`. Не забудьте `Math.Sqrt`.
- Неравенство треугольника: каждая сторона строго меньше суммы двух других.
- Площадь правильного n-угольника со стороной `a`: `n * a² / (4 * tan(π/n))`. Для пятиугольника `n = 5`.
- Если компилятор ругается CS0534 — вы забыли `override` где-то в цепочке. Проверьте каждый конкретный класс.
- `base.Describe()` возвращает строку базовой реализации; её можно дополнить через интерполяцию, не переписывая с нуля.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — Эталонное решение ДЗ M05-L03
// Reference solution for homework M05-L03
using System;

// ── Контракт / Contract ───────────────────────────────────────────
// Внешний контракт площади, независимый от иерархии.
// External area contract, independent of the hierarchy.
public interface I measurable
{
    double Area();
}

// ── Абстрактный базовый класс / Abstract base class ───────────────
// Нельзя инстанцировать; задаёт общее состояние и поведение.
// Cannot be instantiated; defines shared state and behavior.
public abstract class Shape : I measurable
{
    // Конкретное свойство — наследуется всеми фигурами.
    // Concrete property — inherited by every shape.
    public string Name { get; }

    // protected конструктор: только для наследников.
    // protected constructor: for descendants only.
    protected Shape(string name) => Name = name;

    // abstract: тело отсутствует, override обязателен.
    // abstract: no body, override is mandatory.
    public abstract double Area();

    // virtual: есть реализация по умолчанию, можно расширить.
    // virtual: has a default implementation, may be extended.
    public virtual string Describe() => $"{Name}: area = {Area():F2}";

    // Общий код — не дублируется в наследниках.
    // Shared code — not duplicated across descendants.
    public void Print() => Console.WriteLine(Describe());
}

// ── Промежуточный абстрактный класс / Intermediate abstract class ─
// Добавляет новый абстрактный член VertexCount → сам остаётся abstract.
// Adds a new abstract member VertexCount → stays abstract itself.
public abstract class Polygon : Shape
{
    protected Polygon(string name) : base(name) { }

    public abstract int VertexCount { get; }
}

// ── Конкретные наследники / Concrete descendants ──────────────────
public sealed class Circle : Shape
{
    public double Radius { get; }

    public Circle(double radius) : base("Circle") => Radius = radius;

    public override double Area() => Math.PI * Radius * Radius;

    // Расширяем виртуальное описание через base.Describe().
    // Extend the virtual description via base.Describe().
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

public sealed class Triangle : Polygon
{
    private double A { get; }
    private double B { get; }
    private double C { get; }

    public Triangle(double a, double b, double c) : base("Triangle")
    {
        // Инвариант: стороны образуют треугольник.
        // Invariant: sides form a valid triangle.
        if (a + b <= c || a + c <= b || b + c <= a)
            throw new ArgumentException("Sides do not form a triangle.");
        A = a; B = b; C = c;
    }

    public override int VertexCount => 3;

    // Формула Герона / Heron's formula.
    public override double Area()
    {
        double s = (A + B + C) / 2.0;
        return Math.Sqrt(s * (s - A) * (s - B) * (s - C));
    }
}

public sealed class RegularPentagon : Polygon
{
    public double Side { get; }

    public RegularPentagon(double side) : base("RegularPentagon") => Side = side;

    public override int VertexCount => 5;

    // Площадь правильного n-угольника: n*a²/(4*tan(π/n)), n=5.
    // Area of a regular n-gon: n*a²/(4*tan(π/n)), n=5.
    public override double Area() => 5 * Side * Side / (4 * Math.Tan(Math.PI / 5));
}

// ── Точка входа (top-level statements) / Entry point ──────────────
// Shape s = new Shape("x"); // CS0144: нельзя создать экземпляр abstract класса.

List<Shape> shapes = [
    new Circle(2),
    new Rectangle(3, 4),
    new Triangle(3, 4, 5),
    new RegularPentagon(2)
];

double totalArea = 0;
foreach (var s in shapes)
{
    s.Print();
    totalArea += s.Area();
    if (s is Polygon p)
        Console.WriteLine($"  polygon vertices: {p.VertexCount}");
}
Console.WriteLine($"Total area: {totalArea:F2}");
```

Разбор по строкам. Интерфейс `I measurable` — это внешний контракт: внешний потребитель может зависеть только от него, не зная о `Shape` и его иерархии; это иллюстрирует best practice «контракт интерфейсом, механика абстрактным классом». `Shape` помечен `abstract` и реализует `I measurable`, но не предоставляет тела `Area()` — потому что «просто фигура» не имеет площади, и попытка дать реализацию по умолчанию была бы ошибкой проектирования (та самая путаница `abstract` vs `virtual` из урока). `Name` — конкретное свойство, общее для всех; его инициализирует `protected` конструктор, который нельзя вызвать снаружи, что согласуется с принципом «protected конструкторы в абстрактных классах». `Describe()` — `virtual`, а не `abstract`: базовая строка `"{Name}: area = {Area():F2}"` уже осмысленна, и `Circle` расширяет её через `base.Describe()`, не переписывая с нуля — это и есть отличие `virtual` (можно переопределить) от `abstract` (обязательно). `Print()` — конкретный метод, который не дублируется в наследниках; он вызывает `Describe()` полиморфно, демонстрируя, что общий код можно сосредоточить в базовом классе.

`Polygon` — ключевой момент каскада абстрактности. Он наследует `Shape`, но не переопределяет `Area()`, а добавляет новый абстрактный член `VertexCount`. Согласно уроку, пока хотя бы один `abstract` метод остаётся не переопределённым, производный класс обязан быть `abstract`. Поэтому `Polygon` помечен `abstract`, и только `Triangle` и `RegularPentagon`, закрыв и `Area()`, и `VertexCount`, становятся конкретными и `sealed`. Это прямая иллюстрация «каскадности абстрактности» и принципа «помечай `sealed`, если дальнейшее наследование не планируется». `Triangle` дополнительно защищает инвариант через проверку неравенства треугольника в конструкторе — это пример того, как абстрактный класс может требовать от наследников корректной инициализации общего состояния через `base(...)`. В точке входа используется коллекционное выражение C# 12 `[ ... ]`, pattern matching `is Polygon p` для извлечения числа вершин, и полиморфный обход `foreach` с вызовом `Print()` и суммированием `Area()` — всё это закрепляет идею, что работа ведётся с базовым типом `Shape`, а конкретное поведение определяется во время выполнения. Закомментированная строка `new Shape("x")` с пояснением CS0144 фиксирует самую частую ошибку — попытку инстанцировать абстрактный класс.

#### Задания на углубление (бонус)

1. Добавьте абстрактное свойство `Perimeter` в `Shape` и переопределите его во всех конкретных классах. Подумайте: должно ли оно быть `abstract` или `virtual` с реализацией по умолчанию через сумму сторон? Обоснуйте выбор.
2. Реализуйте метод `bool Contains(Point p)` в `Polygon` как `virtual` с обобщённым алгоритмом луча (ray casting), переопределяйте его в `Triangle` и `RegularPentagon` только при необходимости оптимизации. Как это демонстрирует преимущество `virtual` перед `abstract`, когда есть разумный общий алгоритм?
3. Добавьте класс `Square : Rectangle` и сделайте `Rectangle` не `sealed`. Проследите, как изменится девиртуализация и какие новые ошибки проектирования могут возникнуть (например, нарушение инварианта «ширина == высота»).
4. Сериализуйте список фигур в JSON через `System.Text.Json` с пользовательским конвертером, использующим pattern matching по типу. Какие абстрактные члены помогают полиморфной сериализации, а какие мешают?

---

## Statement in English / Постановка на английском

#### Context & motivation

Imagine you are building a small geometry engine for an educational CAD library. It must offer a single notion of "shape" on which generic operations can be performed: compute area, print a description, group into collections, and sum areas. Yet a "bare shape" as a standalone object has no meaning — area does not exist for an abstract concept, and instantiating it is impossible. This is precisely the situation where an abstract class is a better fit than an interface: there is shared state (the name), shared mechanics (printing, formatting), but also a contract without a default implementation (area depends on the concrete form).

Additionally, the library contains polygons — a special subfamily of shapes that introduces a new mandatory property, "vertex count". This naturally leads to an intermediate abstract class `Polygon` that adds a new abstract member and continues the cascade of abstractness downward. Finally, some types (such as circle and rectangle) should be "closed" to further derivation, both to help the JIT compiler and to make intent explicit. An external consumer wants to work with shapes through a stable contract that does not depend on the concrete hierarchy — for this you will add an `I measurable` interface with an `Area()` method. The result is a layered architecture: the contract (interface), the shared mechanics (abstract `Shape`), the polygon specificity (abstract `Polygon`), and the concrete classes (`Circle`, `Rectangle`, `Triangle`, `RegularPentagon`).

The assignment is structured so that every decision leans on a concrete concept from the lesson. You will not merely write code — you will justify why a member is `abstract` rather than `virtual`, why the constructor is `protected` rather than `public`, and why one class is `sealed` while another is left open for further derivation.

#### What to do step by step

1. Create a new .NET 8 console application named `GeometryEngine`. Run the commands:
   ```
   dotnet new console -n GeometryEngine -o GeometryEngine --framework net8.0
   cd GeometryEngine
   dotnet build
   ```
   Ensure `GeometryEngine.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and `<LangVersion>latest</LangVersion>` (add the latter manually via `edit` if needed).

2. Organize the code in `Program.cs` using top-level statements. Place `using System;` at the top (or rely on implicit global usings, which are enabled by default in .NET 8). Define all types in a single file for easy review, but group them logically with comment separators: "Contract", "Abstract base class", "Intermediate abstract class", "Concrete descendants".

3. Declare an interface `I measurable` with a single method `double Area();`. This interface is the external contract that the abstract `Shape` class will implement.

4. Declare the abstract class `Shape` implementing `I measurable`:
   - a concrete property `public string Name { get; }` shared by all shapes;
   - a `protected Shape(string name)` constructor that assigns `Name` using an expression-bodied form;
   - an abstract method `public abstract double Area();` — no body, implementation is mandatory in descendants;
   - a virtual method `public virtual string Describe() => $"{Name}: area = {Area():F2}";` — provides a default implementation that a descendant may extend;
   - a concrete method `public void Print() => Console.WriteLine(Describe());` — shared code, not duplicated.

5. Declare an intermediate abstract class `Polygon : Shape`:
   - `protected Polygon(string name) : base(name) { }` — correctly invokes the base constructor;
   - a new abstract member `public abstract int VertexCount { get; }` — the abstractness cascade continues, since every polygon must declare its vertex count, which depends on the concrete kind.

6. Create the concrete classes, each `sealed` (where no further derivation is intended):
   - `Circle : Shape` — a `Radius` field, an overridden `Area()` via `Math.PI * Radius * Radius`, an overridden `Describe()` that calls `base.Describe()` and appends `, radius = {Radius}`;
   - `Rectangle : Shape` — `Width` and `Height` fields, `Area()` via `Width * Height`;
   - `Triangle : Polygon` — overrides both `VertexCount => 3` and `Area()` (Heron's formula from three sides passed to the constructor);
   - `RegularPentagon : Polygon` — overrides `VertexCount => 5` and `Area()` via the regular-pentagon area formula with side `a`: `(5 * a * a) / (4 * Math.Tan(Math.PI / 5))`.

7. In the top-level section of `Program.cs`, build a collection of shapes using a C# 12 collection expression:
   ```csharp
   List<Shape> shapes = [new Circle(2), new Rectangle(3, 4), new Triangle(3, 4, 5), new RegularPentagon(2)];
   ```
   Then perform a polymorphic traversal and area summation, using pattern matching to count vertices of polygons:
   ```csharp
   double totalArea = 0;
   foreach (var s in shapes)
   {
       s.Print();
       totalArea += s.Area();
       if (s is Polygon p)
           Console.WriteLine($"  polygon vertices: {p.VertexCount}");
   }
   Console.WriteLine($"Total area: {totalArea:F2}");
   ```

8. Run the project with `dotnet run` and record the expected output. For a circle of radius 2 the area is ≈ 12.57; for a 3×4 rectangle — 12.00; for a triangle with sides 3, 4, 5 — 6.00 (a right triangle, via Heron's formula); for a regular pentagon with side 2 — ≈ 13.76. The total sum should be ≈ 44.33.

9. In a comment at the end of the file, explicitly state which members are `abstract`, which are `virtual`, and why. Confirm that the attempt `new Shape("x")` triggers compiler error CS0144 (Cannot create an instance of the abstract class or interface) — include the commented-out line with that attempt next to the explanation.

10. (Resilience check.) Temporarily remove `override` from `Area()` in `Circle` and run `dotnet build`: confirm that the compiler reports error CS0534 ("does not implement inherited abstract member"). Restore `override` and verify the build passes. Record the observation in a comment near the method.

#### Requirements

The solution must compile without level-4-or-higher warnings under .NET 8 with C# 12. Every abstract member of the base class must be overridden in each concrete class; no dangling `abstract` member without an `override` may remain in a non-abstract descendant. Constructors of abstract classes must be `protected`, not `public` — this documents intent and removes the false impression that the class can be instantiated. Every concrete descendant that is not intended for further derivation is marked `sealed`. The shared state (the `Name` property) is initialized strictly through `base(...)` — no direct assignments bypassing the base constructor.

The code should use modern C# 12 features where appropriate: top-level statements for the entry point, collection expressions for the shape list, expression-bodied members for short methods and properties, fixed-point formatting (`F2`) for area output. Type names use PascalCase; field and local variable names use camelCase. The triangle must be validated: if the sides do not form a triangle (the triangle inequality is violated), the constructor throws `ArgumentException` with a clear message — this protects the invariant of the base class. All area computations must be analytically exact, without heuristics or `0` placeholders as in the simplified lesson example.

#### Pitfalls

The key pitfall is the strict distinction between `abstract` and `virtual`. `Area()` is declared `abstract` because an abstract "shape" has no meaningful default area: circle, rectangle, and triangle compute it in fundamentally different ways, and leaving a placeholder would introduce a bug into the contract. `Describe()`, by contrast, is `virtual`: the base implementation `"{Name}: area = {Area():F2}"` is already meaningful for any shape, yet a concrete descendant (such as `Circle`) may extend it by calling `base.Describe()` and adding its own data. Confusing these two modifiers is the most common mistake: using `virtual` where a mandatory contract is needed lets a descendant "forget" to override and silently introduces a bug; using `abstract` where a reasonable implementation exists forces code duplication in every descendant.

The second pitfall is the abstractness cascade through `Polygon`. This class adds a new abstract member, `VertexCount`, so it must itself be `abstract`, even though it does not override `Area()`. The lesson is explicit: as long as even one `abstract` method remains unoverridden, the derived class must be declared `abstract`. Therefore `Polygon` stays abstract, and only the concrete `Triangle` and `RegularPentagon` close both `Area()` and `VertexCount`, becoming instantiable and `sealed`. Forgetting `abstract` on `Polygon` is a typical CS0534 compilation error.

The third pitfall concerns constructors. `protected Shape(string name)` cannot be called from outside, but can be called from `Circle`, `Rectangle`, and `Polygon` via `: base(name)`. Making the constructor `public` would suggest to external code that `Shape` is creatable — which contradicts the very idea of an abstract class. The fourth pitfall is a forgotten `base(...)`: if a descendant does not invoke the base constructor, `Name` remains `null`, and `Describe()` prints `": area = ..."`. The fifth pitfall is excessive derivation: mark `sealed` everything not intended for further inheritance; it is both a signal to the reader and a devirtualization hint to the JIT. The sixth is composition with an interface: `I measurable` lets external code depend only on the area contract, unaware of the internal hierarchy, which simplifies testing and substitution.

#### Acceptance criteria

- [ ] The `GeometryEngine` project is created under .NET 8 with C# 12 and builds without errors or level-4+ warnings.
- [ ] The `I measurable` interface is declared and contains `double Area();`.
- [ ] The abstract class `Shape` implements `I measurable` and is marked `abstract`.
- [ ] `Shape` contains the `Name` property, a `protected` constructor, an abstract `Area()`, a virtual `Describe()`, and a concrete `Print()`.
- [ ] The attempt `new Shape("x")` is commented out with an explanation of error CS0144.
- [ ] The intermediate abstract class `Polygon : Shape` adds the abstract `VertexCount` property.
- [ ] `Polygon` is marked `abstract` (justified: it does not close `Area()`).
- [ ] `Circle` overrides `Area()` and `Describe()` with a `base.Describe()` call.
- [ ] `Rectangle` overrides `Area()` via `Width * Height`.
- [ ] `Triangle` overrides `VertexCount` and `Area()` (Heron's formula) and validates the sides.
- [ ] `RegularPentagon` overrides `VertexCount` and `Area()` via the regular-pentagon formula.
- [ ] All concrete descendants are marked `sealed`.
- [ ] All constructors of abstract classes are `protected`.
- [ ] `Program.cs` uses a collection expression `[ ... ]` for the shape list.
- [ ] The polymorphic `foreach` traversal calls `Print()`, sums `Area()`, and prints `VertexCount` via `is Polygon`.
- [ ] The `dotnet run` output contains areas ≈ 12.57, 12.00, 6.00, 13.76 and a total ≈ 44.33.
- [ ] Temporarily removing `override` from `Area()` in `Circle` reproduces error CS0534 (recorded in a comment).

#### Hints

- Recall the vehicle blueprint from the lesson: there is no "just a vehicle". Which shape plays the same role here?
- Heron's formula for a triangle with sides `a, b, c`: semiperimeter `s = (a+b+c)/2`, area `sqrt(s(s-a)(s-b)(s-c))`. Do not forget `Math.Sqrt`.
- Triangle inequality: each side must be strictly less than the sum of the other two.
- Area of a regular n-gon with side `a`: `n * a² / (4 * tan(π/n))`. For a pentagon `n = 5`.
- If the compiler reports CS0534, you forgot an `override` somewhere in the chain. Check every concrete class.
- `base.Describe()` returns the base implementation's string; you can extend it via interpolation rather than rewriting from scratch.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — Reference solution for homework M05-L03
using System;

// ── Contract ──────────────────────────────────────────────────────
// External area contract, independent of the hierarchy.
public interface I measurable
{
    double Area();
}

// ── Abstract base class ───────────────────────────────────────────
// Cannot be instantiated; defines shared state and behavior.
public abstract class Shape : I measurable
{
    // Concrete property — inherited by every shape.
    public string Name { get; }

    // protected constructor: for descendants only.
    protected Shape(string name) => Name = name;

    // abstract: no body, override is mandatory.
    public abstract double Area();

    // virtual: has a default implementation, may be extended.
    public virtual string Describe() => $"{Name}: area = {Area():F2}";

    // Shared code — not duplicated across descendants.
    public void Print() => Console.WriteLine(Describe());
}

// ── Intermediate abstract class ───────────────────────────────────
// Adds a new abstract member VertexCount → stays abstract itself.
public abstract class Polygon : Shape
{
    protected Polygon(string name) : base(name) { }

    public abstract int VertexCount { get; }
}

// ── Concrete descendants ──────────────────────────────────────────
public sealed class Circle : Shape
{
    public double Radius { get; }

    public Circle(double radius) : base("Circle") => Radius = radius;

    public override double Area() => Math.PI * Radius * Radius;

    // Extend the virtual description via base.Describe().
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

public sealed class Triangle : Polygon
{
    private double A { get; }
    private double B { get; }
    private double C { get; }

    public Triangle(double a, double b, double c) : base("Triangle")
    {
        // Invariant: sides form a valid triangle.
        if (a + b <= c || a + c <= b || b + c <= a)
            throw new ArgumentException("Sides do not form a triangle.");
        A = a; B = b; C = c;
    }

    public override int VertexCount => 3;

    // Heron's formula.
    public override double Area()
    {
        double s = (A + B + C) / 2.0;
        return Math.Sqrt(s * (s - A) * (s - B) * (s - C));
    }
}

public sealed class RegularPentagon : Polygon
{
    public double Side { get; }

    public RegularPentagon(double side) : base("RegularPentagon") => Side = side;

    public override int VertexCount => 5;

    // Area of a regular n-gon: n*a²/(4*tan(π/n)), n=5.
    public override double Area() => 5 * Side * Side / (4 * Math.Tan(Math.PI / 5));
}

// ── Entry point (top-level statements) ────────────────────────────
// Shape s = new Shape("x"); // CS0144: cannot create an instance of an abstract class.

List<Shape> shapes = [
    new Circle(2),
    new Rectangle(3, 4),
    new Triangle(3, 4, 5),
    new RegularPentagon(2)
];

double totalArea = 0;
foreach (var s in shapes)
{
    s.Print();
    totalArea += s.Area();
    if (s is Polygon p)
        Console.WriteLine($"  polygon vertices: {p.VertexCount}");
}
Console.WriteLine($"Total area: {totalArea:F2}");
```

The interface `I measurable` is the external contract: an external consumer may depend on it alone, unaware of `Shape` and its hierarchy; this illustrates the best practice "contract with an interface, mechanics with an abstract class". `Shape` is marked `abstract` and implements `I measurable`, but provides no body for `Area()` — because a "bare shape" has no area, and a default implementation would be a design error (exactly the `abstract` vs `virtual` confusion from the lesson). `Name` is a concrete property shared by all; it is initialized by a `protected` constructor that cannot be invoked from outside, consistent with the principle "use `protected` constructors in abstract classes". `Describe()` is `virtual`, not `abstract`: the base string `"{Name}: area = {Area():F2}"` is already meaningful, and `Circle` extends it via `base.Describe()` rather than rewriting from scratch — this is precisely the difference between `virtual` (may override) and `abstract` (must override). `Print()` is a concrete method that is not duplicated in descendants; it calls `Describe()` polymorphically, demonstrating that shared code can be concentrated in the base class.

`Polygon` is the key point of the abstractness cascade. It inherits from `Shape` but does not override `Area()`; instead it adds a new abstract member, `VertexCount`. According to the lesson, as long as at least one `abstract` method remains unoverridden, the derived class must itself be declared `abstract`. Therefore `Polygon` is marked `abstract`, and only `Triangle` and `RegularPentagon`, having closed both `Area()` and `VertexCount`, become concrete and `sealed`. This is a direct illustration of the "abstractness cascade" and the principle "seal when further derivation is not planned". `Triangle` additionally protects an invariant through a triangle-inequality check in its constructor — an example of how an abstract class can require descendants to initialize shared state correctly via `base(...)`. The entry point uses a C# 12 collection expression `[ ... ]`, the `is Polygon p` pattern for extracting vertex counts, and a polymorphic `foreach` traversal that calls `Print()` and sums `Area()` — all of which reinforce the idea that work is performed against the base type `Shape`, while concrete behavior is determined at runtime. The commented-out line `new Shape("x")` with its CS0144 note captures the most common mistake: attempting to instantiate an abstract class.

#### Going deeper (bonus)

1. Add an abstract `Perimeter` property to `Shape` and override it in every concrete class. Consider: should it be `abstract`, or `virtual` with a default implementation based on a sum of sides? Justify the choice.
2. Implement `bool Contains(Point p)` in `Polygon` as a `virtual` method with a general ray-casting algorithm, overriding it in `Triangle` and `RegularPentagon` only when optimization is warranted. How does this demonstrate the advantage of `virtual` over `abstract` when a reasonable general algorithm exists?
3. Add a `Square : Rectangle` class and make `Rectangle` non-sealed. Trace how devirtualization changes and what new design hazards emerge (for instance, violating the "width == height" invariant).
4. Serialize the shape list to JSON via `System.Text.Json` with a custom converter that pattern-matches on type. Which abstract members help polymorphic serialization, and which get in the way?

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект собирается под .NET 8, C# 12, без предупреждений уровня 4+.
- [ ] (RU) Реализованы `I measurable`, `Shape`, `Polygon` и четыре конкретных класса.
- [ ] (RU) Все `abstract` члены переопределены; `sealed` и `protected` применены по назначению.
- [ ] (RU) Вывод `dotnet run` соответствует ожидаемым площадям и сумме.
- [ ] (RU) Закомментирована попытка `new Shape()` с пояснением CS0144.
- [ ] (EN) The project builds under .NET 8, C# 12, with no level-4+ warnings.
- [ ] (EN) `I measurable`, `Shape`, `Polygon` and the four concrete classes are implemented.
- [ ] (EN) All `abstract` members are overridden; `sealed` and `protected` are applied as intended.
- [ ] (EN) The `dotnet run` output matches the expected areas and total.
- [ ] (EN) The `new Shape()` attempt is commented out with a CS0144 explanation.

#### Ресурсы / Resources
- [Microsoft Learn — abstract (C# reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/abstract)
- [Microsoft Learn — Abstract and sealed classes and class members](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/abstract-and-sealed-classes-and-class-members)
- [Microsoft Learn — Interfaces (C# Programming Guide)](https://learn.microsoft.com/dotnet/csharp/programming-guide/interfaces/)
- [Microsoft Learn — Versioning with the override and new keywords](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/knowing-when-to-use-override-and-new-keywords)
- [.NET 8 release notes](https://learn.microsoft.com/dotnet/core/whats-new/dotnet-8)

---
[← К уроку M05-L03](lesson-M05-L03-abstract.md) | [⬆ К модулю M05](../README.md) | [Следующее ДЗ →](homework-M05-L04-interfaces.md)
