---
[← К уроку M04-L07](lesson-M04-L07-record-basics.md) | [⬆ К модулю M04](../README.md) | [Следующее ДЗ →](homework-M04-L08-struct-vs-class.md)
---

### Домашнее задание M04-L07: record (кратко, для value-семантики) / Homework M04-L07: record (brief, value semantics)

**Урок / Lesson:** M04-L07
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться моделировать неизменяемые объекты-значения с помощью `record` и `record struct`, освоить value-равенство, `with`-выражения, `init`-свойства, позиционный синтаксис, автогенерируемый `ToString` и деконструкцию, а также осознанно выбирать между `record class`, `record struct`, обычным `class` и `struct`. (EN) Learn to model immutable value objects with `record` and `record struct`, master value equality, `with`-expressions, `init`-only properties, positional syntax, the compiler-generated `ToString` and deconstruction, and make a deliberate choice between `record class`, `record struct`, a plain `class`, and a plain `struct`.

#### Связь с уроком / Connection to the lesson
(RU — 2–3 предложения) Это задание напрямую опирается на урок M04-L07: вы повторите различие `record class` vs `record struct`, потренируете `with` и деконструкцию на тех же примерах (точка, деньги, температура), и закрепите выбор между record/class/struct. Все тонкости урока (init-only, вычисляемые свойства не входят в равенство, опасность изменяемых ссылочных полей, runtime-тип в `Equals`) отражены в задании и критериях приёмки.
(EN — same) This homework builds directly on lesson M04-L07: you will revisit the `record class` vs `record struct` distinction, practice `with` and deconstruction on the same examples (point, money, temperature), and reinforce the choice between record/class/struct. Every nuance from the lesson (init-only, computed properties excluded from equality, the danger of mutable reference fields, the runtime type in `Equals`) is reflected in the task and the acceptance criteria.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы пишете мини-библиотеку для учёта погодных наблюдений и валютных сумм. У вас есть два вида сущностей. Первый — **значения**: географическая точка (`Point`), денежная сумма в определённой валюте (`Money`), температура в градусах Цельсия (`Temperature`). Такие объекты — это «чистые данные»: их идентичность полностью определяется их содержимым, они не должны меняться после создания, и два экземпляра с одинаковыми полями должны быть равны независимо от того, где и когда они созданы. Второй вид — **сущности с поведением и идентичностью**: например, `WeatherStation` (станция наблюдения с номером, меняющимся состоянием датчиков) — это уже `class`, а не record, потому что её «равенство» — это равенство по номеру станции, а не по текущим показаниям.

В обычном `class` сравнение по умолчанию идёт по ссылке: два разных объекта с одинаковыми полями не равны. Для значения это неудобно — вы хотите, чтобы две точки `(1, 2)` были равны всегда. Можно переопределить `Equals`/`GetHashCode`/`==` вручную, но это рутина и частый источник багов (забыли `GetHashCode`, ошиблись в `==`, не учли `null`). `record` делает всю эту работу за вас: компилятор генерирует корректное value-равенство, `ToString` в формате `Type { Field = value, ... }`, защищённый конструктор копирования, метод `Clone()` для `with`, и `Deconstruct` для позиционной распаковки.

Цель задания — прочувствовать эту автоматику на практике, увидеть, где она помогает, а где может подвести (изменяемые поля в `HashSet`, наследование record с разными runtime-типами, дорогая копия `record struct` с большим числом полей). В результате у вас появится маленькая, но реалистичная библиотека, в которой выбор типа (record / record struct / class) обоснован требованиями, а не привычкой.

#### Что нужно сделать (пошагово)
1. **Создайте проект.** В терминале выполните:
   ```
   dotnet new console -n WeatherMoney -o WeatherMoney --framework net8.0
   cd WeatherMoney
   dotnet new sln -n WeatherMoney --force
   dotnet sln add WeatherMoney.csproj
   ```
   Убедитесь, что в `WeatherMoney.csproj` указано `<TargetFramework>net8.0</TargetFramework>` и `<LangVersion>latest</LangVersion>` (или 12). Откройте `Program.cs` — это будет точка входа.

2. **Опишите тип `Point` как позиционный `record class`.** Это ссылочный тип с value-равенством: две точки с одинаковыми координатами равны, даже если ссылки разные. Используйте минимальную позиционную форму:
   ```csharp
   public record class Point(int X, int Y);
   ```

3. **Опишите тип `Money` как позиционный `record struct`.** Это значимый тип: копируется при присваивании, живёт на стеке/inline, подходит для маленьких значений. Добавьте в тело короткий метод `Format()`, возвращающий строку вида `"19.95 EUR"`. Помните: вычисляемые свойства и методы не участвуют в равенстве — только позиционные `init`-поля.
   ```csharp
   public record struct Money(decimal Amount, string Currency)
   {
       public string Format() => $"{Amount} {Currency}";
   }
   ```

4. **Опишите `Temperature` как record с расширенным телом и валидацией.** Позиционный параметр `Celsius` должен проверяться в конструкторе: значение ниже `-273.15` (абсолютный ноль) выбрасывает `ArgumentOutOfRangeException`. Добавьте вычисляемое свойство `Fahrenheit => Celsius * 9.0 / 5.0 + 32.0;`. Подумайте: почему `Fahrenheit` не должен влиять на равенство двух температур?

5. **Опишите `WeatherStation` как обычный `class`.** Поля: `int StationId` (init-only), `string Location` (init-only), `mutable double LastReading`. Переопределите `Equals`/`GetHashCode` так, чтобы равенство шло **только по `StationId`** (идентичность сущности), а не по показаниям. Это контраст с record: здесь вы пишете равенство руками и осознанно исключаете изменяемое поле.

6. **В `Program.cs` продемонстрируйте всё:**
   - Создайте `p1 = new Point(1, 2)` и `p2 = new Point(1, 2)`. Выведите `p1 == p2` (ожидается `True`), `ReferenceEquals(p1, p2)` (ожидается `False`), `p1.ToString()` (ожидается `Point { X = 1, Y = 2 }`).
   - Создайте копию `moved = p1 with { X = 5 };` и убедитесь, что оригинал не изменился.
   - Деконструируйте `var (x, y) = p1;` и выведите `x`, `y`.
   - Для `Money`: создайте `price = new Money(19.95m, "EUR")`, сделайте `discounted = price with { Amount = price.Amount * 0.9m };`, выведите `discounted.Format()` (ожидается `17.955 EUR`).
   - Для `Temperature`: создайте `t = new Temperature(25.0)`, проверьте `t.Fahrenheit` (ожидается `77`), попробуйте `var hotter = t with { };` (копия), попытайтесь присвоить `t.Celsius = 30;` — это должна быть **ошибка компиляции** (init-only). Закомментируйте эту строку с пояснением.
   - Проверьте валидацию: `new Temperature(-300)` должно выбросить исключение — оберните в `try/catch` и выведите сообщение.
   - Для `HashSet<Point>`: добавьте `p1`, `p2`, `moved`, выведите `Count` (ожидается `2`, потому что `p1` и `p2` равны и схлопываются).
   - Для `WeatherStation`: создайте две станции с одинаковым `StationId`, но разными `LastReading`, и покажите, что они равны через ваш `Equals` (сущность = идентичность).

7. **Запустите и зафиксируйте вывод:**
   ```
   dotnet build
   dotnet run
   ```
   Скопируйте вывод в файл `output.txt` рядом с проектом.

8. **Краткий рефлексивный комментарий в конце `Program.cs`:** одной строкой объясните, почему `Point` — record class, `Money` — record struct, `Temperature` — record с телом, а `WeatherStation` — class. Это проверяет осознанность выбора типа.

#### Требования к решению
- Проект `WeatherMoney` собирается под .NET 8 без предупреждений уровня error и работает на C# 12 (`<LangVersion>` не ниже 12).
- Использованы ровно четыре типа с осознанным выбором: `record class Point`, `record struct Money`, `record Temperature` (с телом и валидацией), `class WeatherStation`.
- Все позиционные records остаются неизменяемыми: `init`-свойства по умолчанию, **никаких `set`** в позиционных records без явного оправдания в комментарии.
- `with`-выражения применены как минимум дважды (для `Point` и для `Money`), демонстрируется неизменность оригинала.
- Деконструкция `var (x, y) = p1;` присутствует и выводит корректные значения.
- Вычисляемое свойство `Fahrenheit` есть и **не влияет** на равенство: две температуры с одним `Celsius` равны независимо от способа получения `Fahrenheit`.
- Валидация в конструкторе `Temperature` выбрасывает `ArgumentOutOfRangeException` для значений ниже `-273.15`.
- `WeatherStation.Equals`/`GetHashCode` реализованы вручную и используют только `StationId`; изменяемое `LastReading` в равенстве не участвует.
- `HashSet<Point>` демонстрирует схлопывание равных записей (`Count == 2` для трёх добавленных, где два равны).
- Вывод программы соответствует ожидаемым значениям, зафиксирован в `output.txt`.
- Код снабжён краткими комментариями RU+EN в ключевых местах (не на каждой строке, но в местах, где проявляется семантика record).

#### Тонкости и подводные камни
- **record class vs record struct.** `record class` — ссылочный тип (живёт в куче, передаётся по ссылке, но равенство по полям). `record struct` — значимый тип (копируется при присваивании, живёт на стеке/inline). Не путайте: «value equality» ≠ «value type». `record class` имеет value equality, но остаётся ссылочным по размещению. Выбирайте `record struct` для маленьких (условно ≤ 16 байт) значений; для крупных DTO — `record class`, иначе копирование при каждом присваивании станет дорогим.
- **value equality учитывает runtime-тип.** Если у вас `record Person` и `record Animal` с одинаковыми полями, два экземпляра **не равны**, даже если данные совпадают. Более того, при наследовании record равенство требует совпадения runtime-типа: базовый и производный экземпляры с одинаковыми полями не равны. Поэтому в уроке советуют делать record `sealed`, если не нужна иерархия. В этом задании record не наследуются — но помните об этом.
- **`with` и изменяемые поля.** `with` работает благодаря защищённому конструктору копирования и `Clone()`. Если вы добавите `set` в позиционный record и положите его в `HashSet`/словарь, а потом измените поле — объект «потеряется» в хэш-таблице, потому что `GetHashCode` изменится. Поэтому неизменяемость — не стиль, а требование корректности для value-ключей.
- **init-only.** `init` разрешает присвоение только в конструкторе или инициализаторе. Позиционные параметры разворачиваются в `init`-свойства. Попытка `t.Celsius = 30;` — ошибка компиляции, что и нужно показать.
- **вычисляемые свойства не входят в равенство.** `Fahrenheit` — это `=>`, а не `init`/`get` поле, поэтому `Equals`/`GetHashCode` его игнорируют. Это правильно: температура определяется Цельсием, Фаренгейт — производное.
- **автогенерируемый `ToString`.** У record `ToString()` возвращает `Point { X = 1, Y = 2 }` — удобно для логирования и дебага. У обычного `class` вернёт имя типа, если не переопределить.
- **деконструкция.** `var (x, y) = p1;` работает, потому что позиционный record получает `Deconstruct`. У `class` этого нет, пока не напишете сами.
- **ссылочные поля в record.** Если в record лежит изменяемое ссылочное поле (например, `List<T>`), value equality всё равно сравнит ссылку, а не содержимое — это может удивить. Храните `ImmutableArray`/`ReadOnlyCollection`, если нужен неизменяемый набор.

#### Критерии приёмки
- [ ] Проект `WeatherMoney` создаётся командами выше и собирается под .NET 8 (`dotnet build` без ошибок).
- [ ] `Point` — позиционный `record class`, неизменяемый, без `set`.
- [ ] `Money` — позиционный `record struct` с методом `Format()`.
- [ ] `Temperature` — record с телом, валидацией в конструкторе и вычисляемым `Fahrenheit`.
- [ ] `WeatherStation` — обычный `class` с ручным `Equals`/`GetHashCode` по `StationId`.
- [ ] Демонстрируется `p1 == p2` → `True` и `ReferenceEquals(p1, p2)` → `False`.
- [ ] Применено `with` минимум дважды (для `Point` и `Money`), оригинал не изменён.
- [ ] Деконструкция `var (x, y) = p1;` присутствует и выводит корректные значения.
- [ ] Попытка `t.Celsius = 30;` закомментирована как ошибка компиляции (init-only).
- [ ] `new Temperature(-300)` выбрасывает `ArgumentOutOfRangeException`, перехвачено в `try/catch`.
- [ ] `HashSet<Point>` с тремя элементами (два равны) даёт `Count == 2`.
- [ ] `discounted.Format()` выводит `17.955 EUR`.
- [ ] Автогенерированный `ToString` выводит `Point { X = 1, Y = 2 }`.
- [ ] Две `WeatherStation` с одним `StationId` равны через `Equals` несмотря на разные `LastReading`.
- [ ] В `Program.cs` есть рефлексивный комментарий с обоснованием выбора типа для каждого из четырёх типов.
- [ ] Вывод зафиксирован в `output.txt`.

#### Подсказки (без прямого ответа)
- Вспомните, что позиционная форма `record Point(int X, int Y);` уже даёт вам `init`-свойства, `Equals`, `GetHashCode`, `==`, `!=`, `ToString`, `Deconstruct` и `with` — дописывать это вручную не нужно.
- Для валидации в `Temperature` посмотрите на пример из урока: позиционный параметр можно «перехватить» в теле record, переприсвоив свойство с проверкой.
- Для `WeatherStation.Equals` используйте `StationId.Equals(other.StationId)` и не забудьте `GetHashCode() => StationId.GetHashCode();` — иначе сломаете контракт Equals/GetHashCode.
- Чтобы показать, что `Fahrenheit` не входит в равенство, создайте две температуры с одним `Celsius` и сравните `==`.
- Для демонстрации init-only ошибки достаточно закомментировать строку с пояснением — компилятор не даст собрать, если раскомментировать.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — эталон решения WeatherMoney
// Эталонное решение: record class, record struct, record с телом, обычный class
using System;
using System.Collections.Generic;

// Позиционный record class — ссылочный тип с value-равенством / positional record class — reference type with value equality
// Идентичность = координаты, неизменяем, равенство по полям / identity = coordinates, immutable, field equality
public record class Point(int X, int Y);

// Позиционный record struct — значимый тип, копируется при присваивании / positional record struct — value type, copied on assignment
// Подходит для маленьких значений (≤ 16 байт) / suitable for small values (≤ 16 bytes)
public record struct Money(decimal Amount, string Currency)
{
    // Метод — не часть равенства, только позиционные init-поля / method is not part of equality, only positional init fields
    public string Format() => $"{Amount} {Currency}";
}

// Record с расширенным телом: валидация в конструкторе, вычисляемое свойство / record with body: validation + computed property
public record Temperature(double Celsius)
{
    // Перехватываем позиционный параметр с проверкой абсолютного нуля / intercept positional param, validate absolute zero
    public double Celsius { get; init; } = Celsius >= -273.15
        ? Celsius
        : throw new ArgumentOutOfRangeException(nameof(Celsius), "Below absolute zero");

    // Вычисляемое свойство — НЕ входит в Equals/GetHashCode / computed — NOT part of Equals/GetHashCode
    public double Fahrenheit => Celsius * 9.0 / 5.0 + 32.0;
}

// Обычный class — сущность с идентичностью по StationId / plain class — entity with identity by StationId
// Изменяемое состояние (LastReading) НЕ должно влиять на равенство / mutable state must not affect equality
public class WeatherStation
{
    public int StationId { get; init; }
    public string Location { get; init; } = string.Empty;
    public double LastReading { get; set; } // изменяемое / mutable

    public override bool Equals(object? obj) =>
        obj is WeatherStation other && StationId == other.StationId;

    public override int GetHashCode() => StationId.GetHashCode();
}

class Program
{
    static void Main()
    {
        // --- Point: value equality у ссылочного record / value equality in a reference record ---
        var p1 = new Point(1, 2);
        var p2 = new Point(1, 2);
        Console.WriteLine(p1 == p2);                 // True — данные равны / data equal
        Console.WriteLine(ReferenceEquals(p1, p2));  // False — ссылки разные / refs differ
        Console.WriteLine(p1);                       // Point { X = 1, Y = 2 } — автоген ToString

        // with: копия с изменённым полем, оригинал не трогаем / copy with changed field, original untouched
        var moved = p1 with { X = 5 };
        Console.WriteLine(moved);                    // Point { X = 5, Y = 2 }
        Console.WriteLine(p1);                       // Point { X = 1, Y = 2 } — оригинал цел / original intact

        // Деконструкция позиционного record / deconstruction of a positional record
        var (x, y) = p1;
        Console.WriteLine($"Decomposed: x={x}, y={y}"); // Decomposed: x=1, y=2

        // --- Money: record struct копируется при присваивании / record struct copied on assignment ---
        var price = new Money(19.95m, "EUR");
        var discounted = price with { Amount = price.Amount * 0.9m };
        Console.WriteLine(discounted.Format());      // 17.955 EUR

        // --- Temperature: валидация + вычисляемое свойство / validation + computed property ---
        var t = new Temperature(25.0);
        Console.WriteLine(t.Fahrenheit);             // 77
        // t.Celsius = 30.0; // ОШИБКА КОМПИЛЯЦИИ: init-only / COMPILE ERROR: init-only

        var hotter = t with { };                     // копия с тем же значением / copy with same value
        Console.WriteLine(hotter == t);              // True — Fahrenheit не влияет на равенство / Fahrenheit excluded

        try
        {
            _ = new Temperature(-300);               // ниже абсолютного нуля / below absolute zero
        }
        catch (ArgumentOutOfRangeException ex)
        {
            Console.WriteLine($"Validation: {ex.ParamName}"); // Validation: Celsius
        }

        // --- HashSet<Point>: равные записи схлопываются / equal records collapse ---
        var set = new HashSet<Point> { p1, p2, moved };
        Console.WriteLine(set.Count);                // 2 — p1 и p2 равны, moved отличается

        // --- WeatherStation: равенство по идентичности (StationId), не по показаниям ---
        var s1 = new WeatherStation { StationId = 7, Location = "North", LastReading = 12.3 };
        var s2 = new WeatherStation { StationId = 7, Location = "North", LastReading = 99.9 };
        Console.WriteLine(s1.Equals(s2));            // True — одинаковый StationId, разные показания

        // Рефлексия: Point — значение-ссылка (record class), Money — маленькое значение (record struct),
        // Temperature — значение с инвариантом (record + тело), WeatherStation — сущность с идентичностью (class).
    }
}
```

**Разбор по строкам.** `public record class Point(int X, int Y);` — минимальная позиционная форма record class: компилятор генерирует `init`-свойства `X`/`Y`, value-равенство через `Equals(Point other)`, операторы `==`/`!=`, `GetHashCode`, `ToString` вида `Point { X = 1, Y = 2 }`, конструктор копирования и `Deconstruct`. Мы не пишем ничего этого вручную — это и есть ценность record. `p1 == p2` даёт `True`, потому что equality по полям, хотя `ReferenceEquals` даёт `False` (разные объекты в куче). `var moved = p1 with { X = 5 };` применяет автогенерированный конструктор копирования: создаётся новый экземпляр, у которого `X = 5`, а `Y` берётся из оригинала; оригинал `p1` не меняется — это неизменяемость через `init`.

`public record struct Money(...)` — значимый тип: при `var discounted = price with { ... }` создаётся копия значения (а не новой ссылки), что дёшево для маленьких значений. Метод `Format()` — это не поле, поэтому он не участвует в `Equals`; равенство учитывает только `Amount` и `Currency`. `Temperature` демонстрирует record с телом: позиционный параметр `Celsius` «перехватывается» свойством с тем же именем, где в инициализаторе стоит проверка `>= -273.15`. Вычисляемое свойство `Fahrenheit` (через `=>`) не входит в equality — поэтому `hotter == t` даёт `True`: сравнивается только `Celsius`. Попытка `t.Celsius = 30;` — ошибка компиляции, потому что свойство `init`. `WeatherStation` — обычный `class` с **ручным** `Equals`/`GetHashCode`, использующим только `StationId`: это контраст с record, где равенство пишет компилятор, и показывает, что для сущности с идентичностью равенство по идентификатору важнее, чем по текущему изменяемому состоянию. `HashSet<Point>` схлопывает `p1` и `p2` в один элемент (`Count == 2`), потому что value equality + `GetHashCode` согласованы — этого никогда не добиться с обычным `class` без переопределения. Применённые концепции урока: value semantics, `with`, `init`, позиционные records, автогенерируемый `ToString`, `Deconstruct`, выбор record class vs record struct vs class, опасность изменяемых полей в хэш-таблицах (избегается неизменяемостью), и учёт runtime-типа в `Equals` (здесь без наследования, но осознанно).

#### Задания на углубление (бонус)
1. **Сделайте `Temperature` `sealed`** и добавьте ещё один record `TemperatureRange(Temperature Min, Temperature Max)` с валидацией `Min <= Max`. Покажите, что два диапазона с одинаковыми границами равны.
2. **Добавьте неизменяемый набор показаний** в `WeatherStation` через `ImmutableArray<double> Readings` и сравните две станции с одинаковым `StationId`, но разными `Readings` — они всё равно равны по идентичности. Объясните, почему изменяемый `List<double>` здесь был бы ошибкой.
3. **Измерьте производительность** копирования: создайте `record struct BigStruct` с 20 полями и в цикле `10_000_000` раз выполните `with { }`. Сравните с `record class BigClass` с теми же полями. Сделайте вывод о границе применимости `record struct`.
4. **Наследование record:** создайте `record Animal(string Name)` и `record Dog(string Name, string Breed) : Animal(Name)`. Покажите, что `new Animal("Rex") == new Dog("Rex", "Lab")` даёт `False`, несмотря на одинаковые поля общего предка — равенство учитывает runtime-тип. Объясните, почему урок рекомендует `sealed`.

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you are writing a small library for weather observations and monetary amounts. You have two kinds of objects. The first kind is **values**: a geographic point (`Point`), a monetary amount in a specific currency (`Money`), a temperature in degrees Celsius (`Temperature`). Such objects are "pure data": their identity is fully determined by their contents, they must not change after creation, and two instances with the same fields must be equal no matter where or when they were created. The second kind is **entities with behavior and identity**: for example, a `WeatherStation` (an observation station with a number and mutable sensor state) is a `class`, not a record, because its "equality" is equality by station number, not by the current readings.

In an ordinary `class`, equality defaults to reference comparison: two distinct objects with identical fields are not equal. For a value this is inconvenient — you want two points `(1, 2)` to always be equal. You could override `Equals`/`GetHashCode`/`==` by hand, but that is tedious and a frequent source of bugs (forgot `GetHashCode`, got `==` wrong, did not handle `null`). A `record` does all of this for you: the compiler generates correct value equality, a `ToString` in the form `Type { Field = value, ... }`, a protected copy constructor, a `Clone()` method for `with`, and a `Deconstruct` for positional unpacking.

The goal of this assignment is to feel that machinery in practice, to see where it helps and where it can bite you (mutable fields in a `HashSet`, record inheritance with different runtime types, expensive copies of a `record struct` with many fields). The result is a small but realistic library in which the choice of type (record / record struct / class) is justified by requirements, not by habit.

#### What to do step by step
1. **Create the project.** In a terminal run:
   ```
   dotnet new console -n WeatherMoney -o WeatherMoney --framework net8.0
   cd WeatherMoney
   dotnet new sln -n WeatherMoney --force
   dotnet sln add WeatherMoney.csproj
   ```
   Make sure `WeatherMoney.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and `<LangVersion>latest</LangVersion>` (or 12). Open `Program.cs` — this will be the entry point.

2. **Define `Point` as a positional `record class`.** This is a reference type with value equality: two points with the same coordinates are equal even if the references differ. Use the minimal positional form:
   ```csharp
   public record class Point(int X, int Y);
   ```

3. **Define `Money` as a positional `record struct`.** This is a value type: copied on assignment, lives on the stack/inline, suitable for small values. Add a short `Format()` method in the body that returns a string like `"19.95 EUR"`. Remember: computed properties and methods are not part of equality — only positional `init` fields are.
   ```csharp
   public record struct Money(decimal Amount, string Currency)
   {
       public string Format() => $"{Amount} {Currency}";
   }
   ```

4. **Define `Temperature` as a record with an extended body and validation.** The positional parameter `Celsius` must be validated in the constructor: a value below `-273.15` (absolute zero) throws `ArgumentOutOfRangeException`. Add a computed property `Fahrenheit => Celsius * 9.0 / 5.0 + 32.0;`. Think: why must `Fahrenheit` not affect the equality of two temperatures?

5. **Define `WeatherStation` as a plain `class`.** Fields: `int StationId` (init-only), `string Location` (init-only), `mutable double LastReading`. Override `Equals`/`GetHashCode` so that equality is based **only on `StationId`** (entity identity), not on readings. This is the contrast with records: here you write equality by hand and deliberately exclude the mutable field.

6. **In `Program.cs`, demonstrate everything:**
   - Create `p1 = new Point(1, 2)` and `p2 = new Point(1, 2)`. Print `p1 == p2` (expected `True`), `ReferenceEquals(p1, p2)` (expected `False`), `p1.ToString()` (expected `Point { X = 1, Y = 2 }`).
   - Create a copy `moved = p1 with { X = 5 };` and confirm the original is unchanged.
   - Deconstruct `var (x, y) = p1;` and print `x`, `y`.
   - For `Money`: create `price = new Money(19.95m, "EUR")`, make `discounted = price with { Amount = price.Amount * 0.9m };`, print `discounted.Format()` (expected `17.955 EUR`).
   - For `Temperature`: create `t = new Temperature(25.0)`, check `t.Fahrenheit` (expected `77`), try `var hotter = t with { };` (a copy), then attempt `t.Celsius = 30;` — this must be a **compile error** (init-only). Comment that line out with an explanation.
   - Test validation: `new Temperature(-300)` should throw — wrap it in `try/catch` and print the message.
   - For `HashSet<Point>`: add `p1`, `p2`, `moved`, print `Count` (expected `2`, because `p1` and `p2` are equal and collapse).
   - For `WeatherStation`: create two stations with the same `StationId` but different `LastReading`, and show they are equal through your `Equals` (entity = identity).

7. **Run and capture the output:**
   ```
   dotnet build
   dotnet run
   ```
   Copy the output into a file `output.txt` next to the project.

8. **A short reflective comment at the end of `Program.cs`:** in one line explain why `Point` is a record class, `Money` is a record struct, `Temperature` is a record with a body, and `WeatherStation` is a class. This checks that the choice of type is deliberate.

#### Requirements
- The `WeatherMoney` project builds on .NET 8 with no error-level warnings and runs on C# 12 (`<LangVersion>` 12 or higher).
- Exactly four types are used with a deliberate choice: `record class Point`, `record struct Money`, `record Temperature` (with body and validation), `class WeatherStation`.
- All positional records remain immutable: `init` properties by default, **no `set`** in positional records without an explicit justification in a comment.
- `with`-expressions are used at least twice (for `Point` and for `Money`), and the immutability of the original is demonstrated.
- Deconstruction `var (x, y) = p1;` is present and prints correct values.
- The computed property `Fahrenheit` exists and **does not affect** equality: two temperatures with the same `Celsius` are equal regardless of how `Fahrenheit` is obtained.
- Validation in the `Temperature` constructor throws `ArgumentOutOfRangeException` for values below `-273.15`.
- `WeatherStation.Equals`/`GetHashCode` are implemented by hand and use only `StationId`; the mutable `LastReading` does not participate in equality.
- `HashSet<Point>` demonstrates collapsing of equal records (`Count == 2` for three added items where two are equal).
- The program output matches the expected values and is captured in `output.txt`.
- The code has brief RU+EN comments at key points (not on every line, but where record semantics show through).

#### Pitfalls
- **record class vs record struct.** A `record class` is a reference type (lives on the heap, passed by reference, but equality is by fields). A `record struct` is a value type (copied on assignment, lives on the stack/inline). Do not confuse "value equality" with "value type". A `record class` has value equality but is still reference-typed for storage. Pick `record struct` for small (roughly ≤ 16 bytes) values; for large DTOs use `record class`, otherwise copying on every assignment becomes expensive.
- **value equality honors the runtime type.** If you have `record Person` and `record Animal` with identical fields, two instances are **not equal**, even if the data matches. Moreover, with record inheritance equality requires the runtime types to match: a base instance and a derived instance with the same fields are not equal. This is why the lesson recommends making records `sealed` unless you need a hierarchy. In this task records do not inherit — but keep this in mind.
- **`with` and mutable fields.** `with` works thanks to a protected copy constructor and `Clone()`. If you add `set` to a positional record and put it in a `HashSet`/dictionary, then mutate a field — the object will be "lost" in the hash table because `GetHashCode` changed. Immutability here is not a style preference but a correctness requirement for value-keyed collections.
- **init-only.** `init` allows assignment only in a constructor or object initializer. Positional parameters expand to `init` properties. The attempt `t.Celsius = 30;` is a compile error, which is exactly what you need to show.
- **computed properties are not part of equality.** `Fahrenheit` is `=>`, not an `init`/`get` field, so `Equals`/`GetHashCode` ignore it. That is correct: a temperature is defined by Celsius; Fahrenheit is derived.
- **compiler-generated `ToString`.** For a record, `ToString()` returns `Point { X = 1, Y = 2 }` — handy for logging and debugging. For a plain `class` it returns the type name unless you override it.
- **deconstruction.** `var (x, y) = p1;` works because a positional record gets a `Deconstruct`. A `class` does not have one until you write it yourself.
- **reference fields in a record.** If a record holds a mutable reference field (say, `List<T>`), value equality still compares the reference, not the contents — which can surprise you. Store `ImmutableArray`/`ReadOnlyCollection` if you need an immutable collection.

#### Acceptance criteria
- [ ] The `WeatherMoney` project is created with the commands above and builds on .NET 8 (`dotnet build` with no errors).
- [ ] `Point` is a positional `record class`, immutable, with no `set`.
- [ ] `Money` is a positional `record struct` with a `Format()` method.
- [ ] `Temperature` is a record with a body, validation in the constructor, and a computed `Fahrenheit`.
- [ ] `WeatherStation` is a plain `class` with hand-written `Equals`/`GetHashCode` keyed on `StationId`.
- [ ] `p1 == p2` → `True` and `ReferenceEquals(p1, p2)` → `False` are demonstrated.
- [ ] `with` is used at least twice (for `Point` and for `Money`), and the original is unchanged.
- [ ] Deconstruction `var (x, y) = p1;` is present and prints correct values.
- [ ] The attempt `t.Celsius = 30;` is commented out as a compile error (init-only).
- [ ] `new Temperature(-300)` throws `ArgumentOutOfRangeException`, caught in `try/catch`.
- [ ] A `HashSet<Point>` with three items (two equal) yields `Count == 2`.
- [ ] `discounted.Format()` prints `17.955 EUR`.
- [ ] The auto-generated `ToString` prints `Point { X = 1, Y = 2 }`.
- [ ] Two `WeatherStation` instances with the same `StationId` are equal via `Equals` despite different `LastReading`.
- [ ] `Program.cs` contains a reflective comment justifying the choice of type for each of the four types.
- [ ] The output is captured in `output.txt`.

#### Hints (no direct answer)
- Recall that the positional form `record Point(int X, int Y);` already gives you `init` properties, `Equals`, `GetHashCode`, `==`, `!=`, `ToString`, `Deconstruct`, and `with` — you do not need to write any of that by hand.
- For validation in `Temperature`, look at the lesson example: a positional parameter can be "intercepted" in the record body by re-assigning the property with a check.
- For `WeatherStation.Equals`, use `StationId.Equals(other.StationId)` and do not forget `GetHashCode() => StationId.GetHashCode();` — otherwise you break the Equals/GetHashCode contract.
- To show that `Fahrenheit` is not part of equality, create two temperatures with the same `Celsius` and compare them with `==`.
- For the init-only error, it is enough to comment the line out with an explanation — the compiler will refuse to build if you uncomment it.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — reference solution for WeatherMoney
// Reference solution: record class, record struct, record with body, plain class
using System;
using System.Collections.Generic;

// Positional record class — reference type with value equality
// Identity = coordinates, immutable, field-based equality
public record class Point(int X, int Y);

// Positional record struct — value type, copied on assignment
// Suitable for small values (≤ 16 bytes)
public record struct Money(decimal Amount, string Currency)
{
    // A method is not part of equality — only positional init fields are
    public string Format() => $"{Amount} {Currency}";
}

// Record with an extended body: validation in the constructor + a computed property
public record Temperature(double Celsius)
{
    // Intercept the positional parameter and validate against absolute zero
    public double Celsius { get; init; } = Celsius >= -273.15
        ? Celsius
        : throw new ArgumentOutOfRangeException(nameof(Celsius), "Below absolute zero");

    // Computed property — NOT part of Equals/GetHashCode
    public double Fahrenheit => Celsius * 9.0 / 5.0 + 32.0;
}

// Plain class — an entity with identity by StationId
// Mutable state (LastReading) must NOT affect equality
public class WeatherStation
{
    public int StationId { get; init; }
    public string Location { get; init; } = string.Empty;
    public double LastReading { get; set; } // mutable

    public override bool Equals(object? obj) =>
        obj is WeatherStation other && StationId == other.StationId;

    public override int GetHashCode() => StationId.GetHashCode();
}

class Program
{
    static void Main()
    {
        // --- Point: value equality in a reference record ---
        var p1 = new Point(1, 2);
        var p2 = new Point(1, 2);
        Console.WriteLine(p1 == p2);                 // True — data equal
        Console.WriteLine(ReferenceEquals(p1, p2));  // False — refs differ
        Console.WriteLine(p1);                       // Point { X = 1, Y = 2 } — auto-generated ToString

        // with: a copy with a changed field, the original is untouched
        var moved = p1 with { X = 5 };
        Console.WriteLine(moved);                    // Point { X = 5, Y = 2 }
        Console.WriteLine(p1);                       // Point { X = 1, Y = 2 } — original intact

        // Deconstruction of a positional record
        var (x, y) = p1;
        Console.WriteLine($"Decomposed: x={x}, y={y}"); // Decomposed: x=1, y=2

        // --- Money: a record struct is copied on assignment ---
        var price = new Money(19.95m, "EUR");
        var discounted = price with { Amount = price.Amount * 0.9m };
        Console.WriteLine(discounted.Format());      // 17.955 EUR

        // --- Temperature: validation + a computed property ---
        var t = new Temperature(25.0);
        Console.WriteLine(t.Fahrenheit);             // 77
        // t.Celsius = 30.0; // COMPILE ERROR: init-only

        var hotter = t with { };                     // a copy with the same value
        Console.WriteLine(hotter == t);              // True — Fahrenheit excluded from equality

        try
        {
            _ = new Temperature(-300);               // below absolute zero
        }
        catch (ArgumentOutOfRangeException ex)
        {
            Console.WriteLine($"Validation: {ex.ParamName}"); // Validation: Celsius
        }

        // --- HashSet<Point>: equal records collapse ---
        var set = new HashSet<Point> { p1, p2, moved };
        Console.WriteLine(set.Count);                // 2 — p1 and p2 are equal, moved differs

        // --- WeatherStation: equality by identity (StationId), not by readings ---
        var s1 = new WeatherStation { StationId = 7, Location = "North", LastReading = 12.3 };
        var s2 = new WeatherStation { StationId = 7, Location = "North", LastReading = 99.9 };
        Console.WriteLine(s1.Equals(s2));            // True — same StationId, different readings

        // Reflection: Point is a value-reference (record class), Money is a small value (record struct),
        // Temperature is a value with an invariant (record + body), WeatherStation is an entity with identity (class).
    }
}
```

**Line-by-line walk-through.** `public record class Point(int X, int Y);` is the minimal positional form of a record class: the compiler generates `init` properties `X`/`Y`, value equality via `Equals(Point other)`, the `==`/`!=` operators, `GetHashCode`, a `ToString` of the form `Point { X = 1, Y = 2 }`, a copy constructor, and `Deconstruct`. You do not write any of this by hand — that is the point of records. `p1 == p2` is `True` because equality is field-based, even though `ReferenceEquals` is `False` (distinct objects on the heap). `var moved = p1 with { X = 5 };` uses the auto-generated copy constructor: a new instance is created with `X = 5` and `Y` taken from the original; the original `p1` is unchanged — immutability through `init`.

`public record struct Money(...)` is a value type: in `var discounted = price with { ... }` a copy of the value is made (not a new reference), which is cheap for small values. The `Format()` method is not a field, so it does not participate in `Equals`; equality considers only `Amount` and `Currency`. `Temperature` shows a record with a body: the positional parameter `Celsius` is "intercepted" by a property of the same name whose initializer checks `>= -273.15`. The computed property `Fahrenheit` (via `=>`) is not part of equality — so `hotter == t` is `True`: only `Celsius` is compared. The attempt `t.Celsius = 30;` is a compile error because the property is `init`. `WeatherStation` is a plain `class` with **hand-written** `Equals`/`GetHashCode` using only `StationId`: this contrasts with records, where the compiler writes equality, and shows that for an entity with identity, equality by identifier matters more than by the current mutable state. `HashSet<Point>` collapses `p1` and `p2` into a single element (`Count == 2`) because value equality and `GetHashCode` agree — something you can never get from a plain `class` without overriding. Lesson concepts applied: value semantics, `with`, `init`, positional records, the auto-generated `ToString`, `Deconstruct`, the choice between record class vs record struct vs class, the danger of mutable fields in hash tables (avoided through immutability), and honoring the runtime type in `Equals` (here without inheritance, but deliberately).

#### Going deeper (bonus)
1. **Make `Temperature` `sealed`** and add another record `TemperatureRange(Temperature Min, Temperature Max)` that validates `Min <= Max`. Show that two ranges with the same bounds are equal.
2. **Add an immutable set of readings** to `WeatherStation` via `ImmutableArray<double> Readings` and compare two stations with the same `StationId` but different `Readings` — they are still equal by identity. Explain why a mutable `List<double>` would be a mistake here.
3. **Measure copy performance**: create a `record struct BigStruct` with 20 fields and in a loop of `10_000_000` iterations do `with { }`. Compare it to a `record class BigClass` with the same fields. Draw a conclusion about the boundary of applicability for `record struct`.
4. **Record inheritance:** create `record Animal(string Name)` and `record Dog(string Name, string Breed) : Animal(Name)`. Show that `new Animal("Rex") == new Dog("Rex", "Lab")` is `False` despite identical fields of the common ancestor — equality honors the runtime type. Explain why the lesson recommends `sealed`.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `WeatherMoney` собирается под .NET 8 / C# 12.
- [ ] (RU) Четыре типа реализованы с осознанным выбором (record class / record struct / record + тело / class).
- [ ] (RU) `with`, деконструкция, init-only, валидация, `HashSet` схлопывание — продемонстрированы.
- [ ] (RU) Вывод зафиксирован в `output.txt`.
- [ ] (RU) Рефлексивный комментарий с обоснованием выбора типа.
- [ ] (EN) The `WeatherMoney` project builds on .NET 8 / C# 12.
- [ ] (EN) Four types implemented with a deliberate choice (record class / record struct / record + body / class).
- [ ] (EN) `with`, deconstruction, init-only, validation, `HashSet` collapsing — all demonstrated.
- [ ] (EN) Output captured in `output.txt`.
- [ ] (EN) A reflective comment justifying the choice of type.

#### Ресурсы / Resources
- [Microsoft Learn — Records (C# reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/record)
- [Microsoft Learn — Records (fundamentals)](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/records)
- [Microsoft Learn — init (C# reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/init)
- [Microsoft Learn — with expression](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/with-expression)
- [Microsoft Learn — Deconstruct](https://learn.microsoft.com/dotnet/csharp/fundamentals/functional/deconstruct)

---

[⬆ К модулю M04](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
