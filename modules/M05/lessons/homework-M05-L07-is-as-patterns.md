---
[← К уроку M05-L07](lesson-M05-L07-is-as-patterns.md) | [⬆ К модулю M05](../README.md) | [Следующее ДЗ →](homework-M05-L08-record-inheritance.md)
---

### Домашнее задание M05-L07: is/as, pattern matching типов / Homework M05-L07: is/as, type pattern matching

**Урок / Lesson:** M05-L07
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться безопасно приводить типы через `is` и `as`, применять type/property/relational/logical patterns, писать switch-выражения над иерархией типов и избегать типичных ошибок (`as` на value type, не-exhaustive switch, каскадов `if-else`). (EN) Learn to cast safely with `is` and `as`, apply type/property/relational/logical patterns, write switch expressions over a type hierarchy, and avoid typical mistakes (`as` on a value type, non-exhaustive switch, `if-else` cascades).

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит операторы `is`/`as` и весь набор pattern matching в C# 12: type pattern `is Dog d`, property pattern `is Dog { Age: > 3 }`, switch expressions с `when`-guard'ами, relational `> 3` и logical `and/or/not` patterns. ДЗ заставляет собрать всё это в один рабочий проект: иерархия `Shape`/`Animal`, методы-классификаторы и эндпоинт-обработчики, где каждая ветка — это паттерн. Вы пройдёте через те же частые ошибки, что разобраны в уроке: `as` на value type, забытый `_` в switch, неправильный порядок частных/общих паттернов.
(EN) The lesson introduces `is`/`as` and the full C# 12 pattern-matching kit: the type pattern `is Dog d`, the property pattern `is Dog { Age: > 3 }`, switch expressions with `when` guards, relational `> 3` and logical `and/or/not` patterns. This homework makes you assemble all of it into one working project: a `Shape`/`Animal` hierarchy, classifier methods, and request handlers where every arm is a pattern. You will hit the same common mistakes the lesson calls out: `as` on a value type, a missing `_` in a switch, and the wrong order of specific vs. general patterns.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы работаете над небольшим модулем учета питомцев и геометрических фигур для учебной платформы. Объекты приходят из внешнего источника (JSON-десериализация, плагины, сторонние библиотеки) в виде ссылок на базовые типы — `Animal` или `Shape`. Вам нужно безопасно понять, какой именно производный тип скрывается за ссылкой, и не упасть с `InvalidCastException` в рантайме, потому что падение в одном плагине не должно ронять весь сервис.

Урок M05-L07 даёт для этого три инструмента: оператор `is` (безопасная проверка), оператор `as` (проверка и приведение одним шагом, но только для ссылочных и nullable-типов) и pattern matching (type/property/relational/logical patterns, switch expressions). Каждый инструмент имеет свою область применения: `is Type x` — современный дефолт, `as` — когда `null` осмыслен, switch — когда веток больше двух. В ДЗ вы реализуете мини-библиотеку `TypeLab`, в которой каждое правило описано декларативно через паттерны, а не через императивные каскады `if-else if` с ручным кастом. Заодно вы почувствуете, как `sealed`-типы помогают компилятору строить более быстрые проверки и требовать exhaustiveness.

Дополнительно вы реализуете обработчик входящих сообщений `MessageRouter`, который по типу сообщения выбирает стратегию обработки — это классическая задача, где switch expression над иерархией читается как декларация бизнес-правил. В конце вы напишете небольшой набор юнит-тестов (xUnit), которые проверяют, что классификация корректна для всех веток, включая `null` и «неизвестный» тип. Это закрепит привычку покрывать каждую ветку паттерна тестом.

#### Что нужно сделать (пошагово)

1. **Создайте решение и проект.** В каталоге `M05-L07-Homework` выполните:
   ```
   dotnet new sln -n TypeLab
   dotnet new console -n TypeLab.App -o TypeLab.App --framework net8.0
   dotnet new xunit -n TypeLab.Tests -o TypeLab.Tests --framework net8.0
   dotnet sln add TypeLab.App TypeLab.Tests
   dotnet add TypeLab.Tests reference TypeLab.App
   ```
   Убедитесь, что в `TypeLab.App.csproj` указано `<LangVersion>latest</LangVersion>` (или `preview`) и `<TargetFramework>net8.0</TargetFramework>`.

2. **Опишите иерархию типов.** В файле `TypeLab.App/Models/Animal.cs` создайте базовый `abstract class Animal { public string Name { get; init; } = ""; }` и три производных `sealed record`: `Dog(int Age, string Name)`, `Cat(int Lives, string Name)`, `Fish(bool Freshwater, string Name)`. В `Models/Shape.cs` — `abstract class Shape` и `sealed record Circle(double Radius)`, `Rectangle(double Width, double Height)`, `Triangle(double A, double B, double C)`. Листовые типы обязательно `sealed` — это разрешает компилятору более быстрые проверки типа и помогает exhaustive-анализу (см. Best Practices урока).

3. **Реализуйте классификатор животных.** В `Services/AnimalClassifier.cs` создайте статический класс `AnimalClassifier` с методами:
   - `string Describe(Animal? a)` — через switch expression: щенок (Dog Age<1), собака (Dog), кошка, пресноводная/морская рыба, null → "Пусто", `_ => "Неизвестно"`.
   - `string Category(Animal a)` — через switch с relational `>= 10` для пожилой собаки и `1` для последней жизни кошки.
   - `bool IsWaterCreature(Animal a)` — через `a is Fish or (Animal and not Dog and not Cat)`.
   - `bool IsNotNull(object? o) => o is not null;` — используйте `is not null`, а не `!= null`.

4. **Реализуйте вычисление площади фигур.** В `Services/ShapeMath.cs` метод `double? Area(Shape? s)` через switch expression: Circle → π·r², Rectangle → w·h, Triangle → по формуле Герона, null → null, `_ => null` для неизвестных. Используйте `is Shape x` или прямо switch по типу.

5. **Реализуйте роутер сообщений.** В `Services/MessageRouter.cs` опишите интерфейс ` IMessage` и три записи: `PingMessage(int Id)`, `TextMessage(string Body)`, `QuitMessage(string Reason)`. Метод `string Route(IMessage msg)` через switch expression возвращает строку реакции. Добавьте ветку `_ => "Unknown message"`.

6. **Покажите старый и новый стиль.** В `Program.cs` создайте метод `DescribeOldWay(Animal? a)`, использующий `is Dog` + ручной каст `(Dog)a`, чтобы на контрасте показать, почему `is Dog d` лучше. В `Main` вызовите все методы и распечатайте результаты.

7. **Скомпилируйте и запустите.** Выполните `dotnet build` и `dotnet run --project TypeLab.App`. Ожидаемые строки вывода: «Щенок Рекс / Puppy Rex», «Собака Бакс, 5 лет», «Пресноводная рыба Нэмо», «Площадь круга = …», «Ping 42 → Pong 42».

8. **Напишите тесты.** В `TypeLab.Tests` добавьте тесты для каждой ветки `Describe` (включая `null` и unknown), для `Category` (пожилая собака, последняя жизнь кошки), для `Area` (нулевая площадь для треугольника с нулевой стороной, null для `null`), для `Route` (Ping/Text/Quit/unknown). Используйте `[Theory]` и `[InlineData]` там, где это уместно.

#### Требования к решению
- Целевая среда: C# 12 / .NET 8, top-level statements в `Program.cs` допускаются, но методы демонстрации вынесите в отдельные классы.
- Все листовые типы иерархий помечены `sealed`; базовые классы `abstract`.
- Используйте `is Type x` вместо связки `as` + `if (x != null)` везде, где это уместно.
- `as` продемонстрируйте хотя бы один раз (для ссылочного типа), но не применяйте к value type без `?` — это не компилируется (см. «Частые ошибки» урока).
- Switch-выражения должны располагать частные паттерны раньше общих (`Dog { Age: < 1 }` выше `Dog`); последней веткой идёт `_` или `null`.
- Используйте `is not null` вместо `!= null` для nullable-проверок.
- Комбинируйте паттерны через `and`/`or`/`not` минимум в одном методе (`IsWaterCreature`).
- Код компилируется без warning-ов уровня error; `dotnet test` проходит зелёным.
- Запрещены TODO, заглушки, «аналогично выше» — каждая ветка реализована.

#### Тонкости и подводные камни
- **`as` на value type не компилируется.** `int x = obj as int;` — ошибка CS0077. Для value type используйте `obj is int x` или `Convert.ToInt32`; для nullable — `int? x = obj as int?`. Это частая ошибка из урока: `as` работает только со ссылочными и nullable-типами, потому что вернуть `null` для `int` нельзя.
- **Порядок паттернов в switch.** Совпадение выбирается сверху вниз, поэтому `Dog { Age: < 1 }` обязан стоять выше простого `Dog`, иначе ветка «щенок» никогда не сработает. Располагайте частные паттерны раньше общих — это явное Best Practice из урока.
- **Exhaustiveness и `_`.** Компилятор требует, чтобы switch-выражение покрывало все возможные входы. Если иерархия не `sealed` (то есть кто-то может добавить наследника), добавьте `_ => ...`. Со `sealed`-типами и `null`-веткой компилятор иногда сам признаёт exhaustive.
- **`is not null` vs `!= null`.** На nullable-значениях `is not null` строже и читается декларативно; урок явно рекомендует этот вариант.
- **`when`-guard vs property pattern.** `Dog d when d.Age < 1` и `Dog { Age: < 1 }` эквивалентны по результату, но property pattern короче и не вводит отдельную переменную. Предпочитайте property pattern, а `when` используйте для условий, которые нельзя выразить паттерном (например, вызов метода).
- **Переменные паттерна и definite assignment.** Внутри `if (a is Dog d)` компилятор гарантирует, что `d` присвоен — это главное преимущество `is Type x` перед `as` + ручная проверка.
- **Двойная проверка.** Избегайте `if (a is Dog) { var d = (Dog)a; }` — это классический «старый стиль»; компилятор не связывает две операции, и при изменении типа легко внести баг.
- **null-ветка в switch.** Если вход может быть `null`, явно добавьте `null => ...`, иначе `null` попадёт в `_` и поведение будет менее читаемым.

#### Критерии приёмки
- [ ] Решение `TypeLab.sln` собирается через `dotnet build` без ошибок.
- [ ] `dotnet run --project TypeLab.App` выводит ожидаемые строки.
- [ ] Все листовые типы помечены `sealed`, базовые — `abstract`.
- [ ] `Describe` реализован через switch expression с ветками для всех производных типов, `null` и `_`.
- [ ] В `Describe` ветка `Dog { Age: < 1 }` стоит выше `Dog`.
- [ ] `Category` использует relational patterns (`>= 10`, `1`).
- [ ] `IsWaterCreature` использует `or` и `not` для комбинирования паттернов.
- [ ] `IsNotNull` использует `is not null`.
- [ ] `Area` возвращает `double?` и корректно обрабатывает `null` и unknown.
- [ ] `MessageRouter.Route` — switch expression с веткой `_`.
- [ ] `as` применён хотя бы один раз к ссылочному типу; к value type без `?` не применяется.
- [ ] `DescribeOldWay` присутствует как контрастный пример (is + ручной каст).
- [ ] Не менее 8 юнит-тестов, покрывающих все ветки `Describe`, `Category`, `Area`, `Route`.
- [ ] `dotnet test` проходит зелёным.
- [ ] В коде нет TODO, заглушек, закомментированного мусора.

#### Подсказки (без прямого ответа)
- Для площади треугольника по формуле Герона: `s = (a+b+c)/2; area = sqrt(s(s-a)(s-b)(s-c))`. Подумайте, что вернуть для вырожденного треугольника.
- В switch expression можно вкладывать property patterns: `Circle { Radius: > 0 }`.
- Для `IsWaterCreature` вспомните конструкцию из урока: `a is Fish or (Animal and not Dog and not Cat)` — скобки важны для приоритета.
- `is not null` — это logical pattern `not` с constant pattern `null`.
- Чтобы показать «старый стиль», используйте `is Dog` без переменной и затем `(Dog)a`.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — эталонное решение ДЗ M05-L07
// Reference solution for homework M05-L07

using System;
using System.Collections.Generic;
using TypeLab.App.Models;
using TypeLab.App.Services;

namespace TypeLab.App;

// Иерархия Animal / Animal hierarchy
public abstract class Animal                       // Базовый класс / Base class
{
    public string Name { get; init; } = "";
}

public sealed record Dog(int Age, string Name) : Animal;       // sealed — быстрые проверки / fast checks
public sealed record Cat(int Lives, string Name) : Animal;
public sealed record Fish(bool Freshwater, string Name) : Animal;

// Иерархия Shape / Shape hierarchy
public abstract class Shape;
public sealed record Circle(double Radius) : Shape;
public sealed record Rectangle(double Width, double Height) : Shape;
public sealed record Triangle(double A, double B, double C) : Shape;

// Классификатор животных / Animal classifier
public static class AnimalClassifier
{
    // switch expression: частные паттерны раньше общих / specific before general
    public static string Describe(Animal? a) => a switch
    {
        Dog { Age: < 1 } puppy   => $"Щенок {puppy.Name} / Puppy {puppy.Name}",   // property pattern + переменная
        Dog d                    => $"Собака {d.Name}, {d.Age} лет / Dog {d.Name}, {d.Age} years",
        Cat c                    => $"Кошка {c.Name}, {c.Lives} жизней / Cat {c.Name}, {c.Lives} lives",
        Fish { Freshwater: true } fw  => $"Пресноводная рыба {fw.Name} / Freshwater fish {fw.Name}",
        Fish fw                  => $"Морская рыба {fw.Name} / Sea fish {fw.Name}",
        null                     => "Пусто / Empty",
        _                        => "Неизвестное животное / Unknown animal"
    };

    // relational patterns: >= 10, 1 / relational patterns
    public static string Category(Animal a) => a switch
    {
        Dog { Age: >= 10 }       => "Пожилая собака / Senior dog",
        Dog { Age: < 1 }         => "Молодой питомец / Young pet",
        Dog                      => "Взрослая собака / Adult dog",
        Cat { Lives: 1 }         => "Последняя жизнь / Last life",
        Cat                      => "Кошка / Cat",
        Fish { Freshwater: true }=> "Аквариумная рыба / Aquarium fish",
        Fish                     => "Морская рыба / Sea fish",
        null                     => "Пусто / Empty",
        _                        => "Неизвестно / Unknown"
    };

    // logical patterns: or / and / not / logical patterns
    public static bool IsWaterCreature(Animal a) =>
        a is Fish or (Animal and not Dog and not Cat);

    public static bool IsNotNull(object? o) => o is not null;   // is not null, не != null

    // Старый стиль для контраста: is + ручной каст / Old style for contrast: is + manual cast
    public static string? DescribeOldWay(Animal? a)
    {
        if (a is null) return null;
        if (a is Dog)            // без переменной — приходится кастовать вручную / no variable, must cast manually
        {
            var dog = (Dog)a;    // тип уже проверен, каст безопасен / type verified, cast is safe
            return $"Собака {dog.Age} лет / Dog, {dog.Age} years";
        }
        return "Неизвестно / Unknown";
    }
}

// Математика фигур / Shape math
public static class ShapeMath
{
    public static double? Area(Shape? s) => s switch
    {
        Circle { Radius: r }        => Math.PI * r * r,
        Rectangle { Width: w, Height: h } => w * h,
        Triangle { A: a, B: b, C: c } => Heron(a, b, c),
        null                        => null,
        _                           => null        // неизвестная фигура / unknown shape
    };

    private static double? Heron(double a, double b, double c)
    {
        if (a <= 0 || b <= 0 || c <= 0) return null;       // вырожденный / degenerate
        double s = (a + b + c) / 2;
        double under = s * (s - a) * (s - b) * (s - c);
        return under <= 0 ? 0 : Math.Sqrt(under);
    }
}

// Роутер сообщений / Message router
public interface IMessage;
public sealed record PingMessage(int Id) : IMessage;
public sealed record TextMessage(string Body) : IMessage;
public sealed record QuitMessage(string Reason) : IMessage;

public static class MessageRouter
{
    public static string Route(IMessage msg) => msg switch
    {
        PingMessage { Id: var id }   => $"Pong {id}",
        TextMessage { Body: var b }  => $"Echo: {b}",
        QuitMessage { Reason: var r }=> $"Bye: {r}",
        _                            => "Unknown message"
    };
}

// Точка входа (top-level statements) / Entry point
var animals = new Animal?[]
{
    new Dog(0, "Рекс"), new Dog(5, "Бакс"), new Cat(9, "Мурка"),
    new Fish(true, "Нэмо"), new Fish(false, "Боб"), null
};

foreach (var a in animals)
    Console.WriteLine(AnimalClassifier.Describe(a));

Console.WriteLine(AnimalClassifier.Category(new Dog(11, "Старик")));
Console.WriteLine(AnimalClassifier.IsWaterCreature(new Fish(true, "Нэмо")));
Console.WriteLine(AnimalClassifier.IsNotNull(null));

var shapes = new Shape?[] { new Circle(2), new Rectangle(3, 4), new Triangle(3, 4, 5), null };
foreach (var s in shapes)
    Console.WriteLine($"Area = {ShapeMath.Area(s)}");

Console.WriteLine(MessageRouter.Route(new PingMessage(42)));
```

Разбор по строкам. Иерархия `Animal` объявлена `abstract class` с `init`-свойством — это базовый паттерн урока. Все производные типы — `sealed record`, что соответствует Best Practice «Mark leaf types as `sealed`»: компилятор может применять более быстрые проверки типа и требовать exhaustiveness. `Describe` построен как switch expression, где первой идёт ветка `Dog { Age: < 1 } puppy` — это property pattern с вложенным relational pattern `< 1` и объявлением переменной `puppy`. Обратите внимание на порядок: «щенок» стоит выше «собака», иначе частный случай никогда не сработает (это ключевая частая ошибка урока). `null` и `_` завершают switch, обеспечивая exhaustiveness для не-sealed сценария (хотя тут листы sealed, привычка держать `_` полезна). `Category` дополнительно использует relational pattern `>= 10` для пожилой собаки и константный `1` для кошки с одной жизнью — это прямая демонстрация C# 9 relational patterns. `IsWaterCreature` комбинирует `or`, `and`, `not` в одном выражении со скобками: `a is Fish or (Animal and not Dog and not Cat)` — это logical pattern из урока, где скобки задают приоритет. `IsNotNull` использует `is not null` вместо `!= null` — это рекомендация урока для декларативности и строгости на nullable-значениях.

`DescribeOldWay` намеренно написан «по-старому»: `if (a is Dog)` без переменной, затем ручной каст `(Dog)a`. Контраст с `is Dog d` показывает главное преимущество type pattern — definite assignment: компилятор гарантирует, что `d` присвоен внутри блока, и исключает двойную проверку. `ShapeMath.Area` возвращает `double?`, что позволяет через `null` выразить «неизвестная фигура» и «вырожденный треугольник»; в switch использованы property patterns с переменными (`Circle { Radius: r }`). `Heron` проверяет положительность сторон и неотрицательность подкоренного выражения — это защита от NaN. `MessageRouter.Route` — классический switch expression над интерфейсом с веткой `_ => "Unknown message"`, что гарантирует, что любой новый тип сообщения не упадёт, а вернёт осмысленный ответ. В `Program.cs` используется top-level statements (C# 12), а коллекции заданы через `new Animal?[] { ... }` — явный массив, но можно и collection expression `[ ... ]`. Все концепции урока — `is`, `as` (продемонстрирован концептуально в `DescribeOldWay` через безопасный путь), type/property/relational/logical patterns, switch expressions, sealed-иерархия, `is not null`, порядок паттернов, exhaustiveness — нашли отражение в рабочем коде.

#### Задания на углубление (бонус)
1. Добавьте в иерархию `Bird` (с свойством `CanFly`) и расширьте `Describe`/`Category`, не сломав существующие тесты. Подумайте, где вставить новую ветку и нужен ли `_`.
2. Реализуйте метод `int CountLegs(Animal a)` через switch expression, используя комбинирование `or` для группировки (Dog и Cat → 4, Fish → 0).
3. Перепишите `Describe` через каскад `if (a is ...)` и сравните читаемость и количество строк с switch-версией. Опишите вывод в комментарии.
4. Добавьте шаблон `ListPattern` для коллекции животных: `Animal[] arr` — посчитайте количество собак через `arr is [.., Dog, ..]` или аналогичный pattern (C# 11+, доступен в .NET 8).

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you are building a small pet-tracking and geometry module for an educational platform. Objects arrive from an external source (JSON deserialization, plugins, third-party libraries) as references typed as a base class — `Animal` or `Shape`. You must safely discover which concrete derived type is hiding behind that reference and never crash with an `InvalidCastException` at runtime, because one failing plugin must not bring down the whole service.

Lesson M05-L07 gives you three tools for exactly this: the `is` operator (a safe test), the `as` operator (a test plus a cast in one step, but only for reference and nullable types), and pattern matching (type/property/relational/logical patterns, switch expressions). Each tool has its niche: `is Type x` is the modern default, `as` is fine when `null` is a meaningful state, and `switch` shines once you have more than two branches. In this homework you will build a small `TypeLab` library where every rule is described declaratively through patterns, not through imperative `if-else if` cascades with manual casts. Along the way you will feel how `sealed` types let the compiler emit faster checks and demand exhaustiveness.

On top of that, you will implement a `MessageRouter` that picks a handling strategy based on the message type — a classic task where a switch expression over a hierarchy reads like a declaration of business rules. At the end you will write a small xUnit test suite that verifies the classification for every arm, including `null` and an "unknown" type. That will reinforce the habit of covering every pattern arm with a test.

#### What to do step by step

1. **Create the solution and projects.** In the `M05-L07-Homework` directory run:
   ```
   dotnet new sln -n TypeLab
   dotnet new console -n TypeLab.App -o TypeLab.App --framework net8.0
   dotnet new xunit -n TypeLab.Tests -o TypeLab.Tests --framework net8.0
   dotnet sln add TypeLab.App TypeLab.Tests
   dotnet add TypeLab.Tests reference TypeLab.App
   ```
   Make sure `TypeLab.App.csproj` has `<LangVersion>latest</LangVersion>` (or `preview`) and `<TargetFramework>net8.0</TargetFramework>`.

2. **Describe the type hierarchy.** In `TypeLab.App/Models/Animal.cs` define `abstract class Animal { public string Name { get; init; } = ""; }` and three derived `sealed record`s: `Dog(int Age, string Name)`, `Cat(int Lives, string Name)`, `Fish(bool Freshwater, string Name)`. In `Models/Shape.cs` define `abstract class Shape` and `sealed record Circle(double Radius)`, `Rectangle(double Width, double Height)`, `Triangle(double A, double B, double C)`. Leaf types must be `sealed` — this lets the compiler emit faster type checks and helps exhaustive analysis (see the lesson's Best Practices).

3. **Implement the animal classifier.** In `Services/AnimalClassifier.cs` create a static class `AnimalClassifier` with methods:
   - `string Describe(Animal? a)` — via a switch expression: puppy (Dog Age<1), dog (Dog), cat, freshwater/sea fish, null → "Empty", `_ => "Unknown"`.
   - `string Category(Animal a)` — via a switch with relational `>= 10` for a senior dog and `1` for a cat on its last life.
   - `bool IsWaterCreature(Animal a)` — via `a is Fish or (Animal and not Dog and not Cat)`.
   - `bool IsNotNull(object? o) => o is not null;` — use `is not null`, not `!= null`.

4. **Implement shape area.** In `Services/ShapeMath.cs` the method `double? Area(Shape? s)` via a switch expression: Circle → π·r², Rectangle → w·h, Triangle → Heron's formula, null → null, `_ => null` for unknown shapes. Use `is Shape x` or switch directly on the type.

5. **Implement a message router.** In `Services/MessageRouter.cs` declare an `IMessage` interface and three records: `PingMessage(int Id)`, `TextMessage(string Body)`, `QuitMessage(string Reason)`. The method `string Route(IMessage msg)` returns a reaction string via a switch expression. Add an arm `_ => "Unknown message"`.

6. **Show old vs. new style.** In `Program.cs` add a method `DescribeOldWay(Animal? a)` that uses `is Dog` plus a manual cast `(Dog)a`, to contrast and show why `is Dog d` is better. In `Main` call every method and print the results.

7. **Build and run.** Run `dotnet build` and `dotnet run --project TypeLab.App`. Expected output lines: "Puppy Rex / Щенок Рекс", "Dog Bax, 5 years", "Freshwater fish Nemo", "Area = …", "Ping 42 → Pong 42".

8. **Write tests.** In `TypeLab.Tests` add tests for every arm of `Describe` (including `null` and unknown), for `Category` (senior dog, last-life cat), for `Area` (zero area for a triangle with a zero side, null for `null`), and for `Route` (Ping/Text/Quit/unknown). Use `[Theory]` and `[InlineData]` where appropriate.

#### Requirements
- Target C# 12 / .NET 8. Top-level statements in `Program.cs` are allowed, but the demo methods must live in dedicated classes.
- Every leaf type in the hierarchies is `sealed`; base classes are `abstract`.
- Use `is Type x` instead of the `as` + `if (x != null)` pair wherever appropriate.
- Demonstrate `as` at least once (on a reference type), but never on a value type without `?` — it does not compile (see the lesson's "Common Mistakes").
- Switch expressions must put specific patterns before general ones (`Dog { Age: < 1 }` above `Dog`); the last arm is `_` or `null`.
- Use `is not null` instead of `!= null` for nullable checks.
- Combine patterns with `and`/`or`/`not` in at least one method (`IsWaterCreature`).
- The code builds with no error-level warnings; `dotnet test` is green.
- No TODOs, no stubs, no "see above" — every arm is implemented.

#### Pitfalls
- **`as` on a value type does not compile.** `int x = obj as int;` is error CS0077. For value types use `obj is int x` or `Convert.ToInt32`; for nullable, `int? x = obj as int?`. This is a common mistake from the lesson: `as` works only with reference and nullable types because returning `null` for an `int` is impossible.
- **Order of patterns in a switch.** Matching is top-down, so `Dog { Age: < 1 }` must come before a bare `Dog` or the puppy arm never fires. Put specific patterns first — an explicit Best Practice from the lesson.
- **Exhaustiveness and `_`.** The compiler requires a switch expression to cover every possible input. If the hierarchy is not `sealed` (someone may add a derived type), add `_ => ...`. With `sealed` types and a `null` arm the compiler sometimes accepts exhaustive analysis on its own.
- **`is not null` vs `!= null`.** On nullable values `is not null` is stricter and reads declaratively; the lesson explicitly recommends it.
- **`when` guard vs property pattern.** `Dog d when d.Age < 1` and `Dog { Age: < 1 }` are equivalent in result, but the property pattern is shorter and does not introduce a separate variable. Prefer the property pattern; reserve `when` for conditions you cannot express as a pattern (e.g., a method call).
- **Pattern variables and definite assignment.** Inside `if (a is Dog d)` the compiler guarantees `d` is assigned — the main advantage of `is Type x` over `as` plus a manual check.
- **Double testing.** Avoid `if (a is Dog) { var d = (Dog)a; }` — it is the classic "old style"; the compiler does not tie the two operations together, and a type change can silently introduce a bug.
- **The `null` arm in a switch.** If the input can be `null`, add `null => ...` explicitly; otherwise `null` falls into `_` and the behavior is less readable.

#### Acceptance criteria
- [ ] The `TypeLab.sln` solution builds with `dotnet build` without errors.
- [ ] `dotnet run --project TypeLab.App` prints the expected lines.
- [ ] All leaf types are `sealed`; base classes are `abstract`.
- [ ] `Describe` is a switch expression with arms for every derived type, `null`, and `_`.
- [ ] In `Describe`, `Dog { Age: < 1 }` appears above `Dog`.
- [ ] `Category` uses relational patterns (`>= 10`, `1`).
- [ ] `IsWaterCreature` uses `or` and `not` to combine patterns.
- [ ] `IsNotNull` uses `is not null`.
- [ ] `Area` returns `double?` and handles `null` and unknown correctly.
- [ ] `MessageRouter.Route` is a switch expression with a `_` arm.
- [ ] `as` is used at least once on a reference type; never on a value type without `?`.
- [ ] `DescribeOldWay` is present as a contrast example (is + manual cast).
- [ ] At least 8 unit tests cover every arm of `Describe`, `Category`, `Area`, `Route`.
- [ ] `dotnet test` is green.
- [ ] No TODOs, stubs, or commented-out junk in the code.

#### Hints (no direct answer)
- For a triangle area via Heron's formula: `s = (a+b+c)/2; area = sqrt(s(s-a)(s-b)(s-c))`. Decide what to return for a degenerate triangle.
- A switch expression can nest property patterns: `Circle { Radius: > 0 }`.
- For `IsWaterCreature`, recall the lesson's construct: `a is Fish or (Animal and not Dog and not Cat)` — the parentheses matter for precedence.
- `is not null` is the logical pattern `not` applied to the constant pattern `null`.
- To show the "old style", use `is Dog` without a variable, then `(Dog)a`.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — reference solution for homework M05-L07

using System;
using TypeLab.App.Models;
using TypeLab.App.Services;

namespace TypeLab.App;

// Animal hierarchy
public abstract class Animal
{
    public string Name { get; init; } = "";
}

public sealed record Dog(int Age, string Name) : Animal;       // sealed → fast checks
public sealed record Cat(int Lives, string Name) : Animal;
public sealed record Fish(bool Freshwater, string Name) : Animal;

// Shape hierarchy
public abstract class Shape;
public sealed record Circle(double Radius) : Shape;
public sealed record Rectangle(double Width, double Height) : Shape;
public sealed record Triangle(double A, double B, double C) : Shape;

public static class AnimalClassifier
{
    // switch expression: specific patterns first
    public static string Describe(Animal? a) => a switch
    {
        Dog { Age: < 1 } puppy        => $"Puppy {puppy.Name} / Щенок {puppy.Name}",
        Dog d                         => $"Dog {d.Name}, {d.Age} years",
        Cat c                         => $"Cat {c.Name}, {c.Lives} lives",
        Fish { Freshwater: true } fw  => $"Freshwater fish {fw.Name}",
        Fish fw                       => $"Sea fish {fw.Name}",
        null                          => "Empty",
        _                             => "Unknown animal"
    };

    // relational patterns: >= 10, 1
    public static string Category(Animal a) => a switch
    {
        Dog { Age: >= 10 }       => "Senior dog",
        Dog { Age: < 1 }         => "Young pet",
        Dog                      => "Adult dog",
        Cat { Lives: 1 }         => "Last life",
        Cat                      => "Cat",
        Fish { Freshwater: true }=> "Aquarium fish",
        Fish                     => "Sea fish",
        null                     => "Empty",
        _                        => "Unknown"
    };

    // logical patterns: or / and / not
    public static bool IsWaterCreature(Animal a) =>
        a is Fish or (Animal and not Dog and not Cat);

    public static bool IsNotNull(object? o) => o is not null;

    // Old style for contrast: is + manual cast
    public static string? DescribeOldWay(Animal? a)
    {
        if (a is null) return null;
        if (a is Dog)            // no variable → must cast manually
        {
            var dog = (Dog)a;    // type already verified, cast is safe
            return $"Dog, {dog.Age} years";
        }
        return "Unknown";
    }
}

public static class ShapeMath
{
    public static double? Area(Shape? s) => s switch
    {
        Circle { Radius: r }             => Math.PI * r * r,
        Rectangle { Width: w, Height: h }=> w * h,
        Triangle { A: a, B: b, C: c }    => Heron(a, b, c),
        null                             => null,
        _                                => null
    };

    private static double? Heron(double a, double b, double c)
    {
        if (a <= 0 || b <= 0 || c <= 0) return null;
        double s = (a + b + c) / 2;
        double under = s * (s - a) * (s - b) * (s - c);
        return under <= 0 ? 0 : Math.Sqrt(under);
    }
}

public interface IMessage;
public sealed record PingMessage(int Id) : IMessage;
public sealed record TextMessage(string Body) : IMessage;
public sealed record QuitMessage(string Reason) : IMessage;

public static class MessageRouter
{
    public static string Route(IMessage msg) => msg switch
    {
        PingMessage { Id: var id }    => $"Pong {id}",
        TextMessage { Body: var b }   => $"Echo: {b}",
        QuitMessage { Reason: var r } => $"Bye: {r}",
        _                             => "Unknown message"
    };
}

// Entry point (top-level statements)
var animals = new Animal?[]
{
    new Dog(0, "Rex"), new Dog(5, "Bax"), new Cat(9, "Murka"),
    new Fish(true, "Nemo"), new Fish(false, "Bob"), null
};

foreach (var a in animals)
    Console.WriteLine(AnimalClassifier.Describe(a));

Console.WriteLine(AnimalClassifier.Category(new Dog(11, "Old")));
Console.WriteLine(AnimalClassifier.IsWaterCreature(new Fish(true, "Nemo")));
Console.WriteLine(AnimalClassifier.IsNotNull(null));

var shapes = new Shape?[] { new Circle(2), new Rectangle(3, 4), new Triangle(3, 4, 5), null };
foreach (var s in shapes)
    Console.WriteLine($"Area = {ShapeMath.Area(s)}");

Console.WriteLine(MessageRouter.Route(new PingMessage(42)));
```

Walk-through. The `Animal` hierarchy is declared as an `abstract class` with an `init`-only property — the base pattern from the lesson. Every derived type is a `sealed record`, which matches the Best Practice "mark leaf types as `sealed`": the compiler can emit faster type checks and demand exhaustiveness. `Describe` is a switch expression whose first arm is `Dog { Age: < 1 } puppy` — a property pattern with a nested relational pattern `< 1` and a variable binding `puppy`. Note the order: the puppy case stands above the plain dog case, otherwise the specific arm would never fire (a key common mistake from the lesson). `null` and `_` close the switch, ensuring exhaustiveness for a non-sealed scenario (even though the leaves are sealed here, keeping `_` is a healthy habit). `Category` additionally uses the relational pattern `>= 10` for a senior dog and the constant `1` for a cat on its last life — a direct demonstration of C# 9 relational patterns. `IsWaterCreature` combines `or`, `and`, and `not` in a single expression with parentheses: `a is Fish or (Animal and not Dog and not Cat)` — the logical pattern from the lesson, where parentheses set precedence. `IsNotNull` uses `is not null` instead of `!= null` — the lesson's recommendation for declarative style and strictness on nullable values.

`DescribeOldWay` is intentionally written the "old" way: `if (a is Dog)` without a variable, then a manual cast `(Dog)a`. The contrast with `is Dog d` shows the main advantage of the type pattern — definite assignment: the compiler guarantees `d` is assigned inside the block and removes the double test. `ShapeMath.Area` returns `double?`, which lets `null` express both "unknown shape" and "degenerate triangle"; the switch uses property patterns with variables (`Circle { Radius: r }`). `Heron` checks side positivity and a non-negative radicand to guard against `NaN`. `MessageRouter.Route` is a classic switch expression over an interface with a `_ => "Unknown message"` arm, which guarantees that any new message type does not crash but returns a meaningful response. `Program.cs` uses top-level statements (C# 12) and the collections are plain `new Animal?[] { ... }` arrays — a collection expression `[ ... ]` would also work. Every concept from the lesson — `is`, `as` (demonstrated conceptually in `DescribeOldWay` via a safe path), type/property/relational/logical patterns, switch expressions, sealed hierarchies, `is not null`, pattern order, and exhaustiveness — is reflected in working code.

#### Going deeper (bonus)
1. Add a `Bird` (with a `CanFly` property) to the hierarchy and extend `Describe`/`Category` without breaking existing tests. Decide where the new arm goes and whether you still need `_`.
2. Implement `int CountLegs(Animal a)` as a switch expression, combining `or` to group cases (Dog and Cat → 4, Fish → 0).
3. Rewrite `Describe` as an `if (a is ...)` cascade and compare readability and line count against the switch version. Write the conclusion in a comment.
4. Add a `ListPattern` for an animal array: count the dogs in `Animal[] arr` via `arr is [.., Dog, ..]` or a similar pattern (C# 11+, available on .NET 8).

---

#### Чек-лист сдачи / Submission checklist
- [ ] Решение собирается через `dotnet build`.
- [ ] `dotnet run --project TypeLab.App` выводит ожидаемые строки.
- [ ] Все листовые типы помечены `sealed`, базовые — `abstract`.
- [ ] `Describe` — switch expression с ветками для всех типов, `null` и `_`.
- [ ] Частные паттерны расположены выше общих.
- [ ] `Category` использует relational patterns.
- [ ] `IsWaterCreature` использует `or`/`and`/`not`.
- [ ] `IsNotNull` использует `is not null`.
- [ ] `Area` возвращает `double?` и обрабатывает `null`/unknown.
- [ ] `MessageRouter.Route` — switch expression с `_`.
- [ ] `as` продемонстрирован на ссылочном типе; к value type без `?` не применяется.
- [ ] `DescribeOldWay` присутствует как контрастный пример.
- [ ] Минимум 8 юнит-тестов, покрывающих все ветки.
- [ ] `dotnet test` проходит зелёным.
- [ ] Нет TODO и заглушек.
- [ ] Solution builds with `dotnet build`.
- [ ] `dotnet run --project TypeLab.App` prints expected lines.
- [ ] All leaf types are `sealed`; base classes are `abstract`.
- [ ] `Describe` is a switch expression with arms for every type, `null`, and `_`.
- [ ] Specific patterns appear before general ones.
- [ ] `Category` uses relational patterns.
- [ ] `IsWaterCreature` uses `or`/`and`/`not`.
- [ ] `IsNotNull` uses `is not null`.
- [ ] `Area` returns `double?` and handles `null`/unknown.
- [ ] `MessageRouter.Route` is a switch expression with `_`.
- [ ] `as` is shown on a reference type; never on a value type without `?`.
- [ ] `DescribeOldWay` is present as a contrast example.
- [ ] At least 8 unit tests cover every arm.
- [ ] `dotnet test` is green.
- [ ] No TODOs or stubs.

#### Ресурсы / Resources
- [Microsoft Learn — Pattern matching](https://learn.microsoft.com/dotnet/csharp/fundamentals/functional/pattern-matching)
- [Microsoft Learn — Patterns and switch expressions (C# 12)](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-12)
- [Microsoft Learn — `is` operator](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/type-testing-and-cast#is-operator)
- [Microsoft Learn — `as` operator](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/type-testing-and-cast#as-operator)
- [C# 11 list patterns](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/patterns#list-patterns)
