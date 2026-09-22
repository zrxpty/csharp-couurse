---
[← К уроку M04-L08](lesson-M04-L08-struct-vs-class.md) | [⬆ К модулю M04](../README.md) | [Предыдущее ДЗ ←](homework-M04-L07-record-basics.md) | [Следующее ДЗ →](homework-M04-L09-namespaces-using.md)
---

### Домашнее задание M04-L08: struct vs class / Homework M04-L08: struct vs class

**Урок / Lesson:** M04-L08
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться осознанно выбирать между `class`, `struct`, `record struct` и `ref struct`, понимать семантику значений и ссылок, видеть скрытые аллокации при боксе и копировании, и реализовывать маленькие неизменяемые значения с value-семантикой. (EN) Learn to make a conscious choice between `class`, `struct`, `record struct` and `ref struct`, understand value vs reference semantics, see hidden allocations from boxing and copying, and implement small immutable values with value semantics.

#### Связь с уроком / Connection to the lesson
(RU) Урок M04-L08 показывает фундаментальное различие типов-значений и ссылочных типов: где живёт значение, как копируется, что такое боксинг и почему большие изменяемые `struct` опасны. ДЗ закрепляет это на практике через создание четырёх типов, наблюдение аллокаций и сравнение поведения при присваивании. (EN) Lesson M04-L08 shows the fundamental difference between value and reference types: where the value lives, how it is copied, what boxing is, and why large mutable `struct`s are dangerous. The homework reinforces this through four types, allocation observations, and assignment behaviour comparisons.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы присоединяетесь к команде, которая пишет небольшой движок 2D-игры и финансовую библиотеку для внутреннего учёта. В кодовой базе уже смешали `class` и `struct` наугад: где-то точки и цвета объявлены как классы (и при каждом рендере кадра аллоцируются тысячи объектов в куче, что даёт лишнюю работу сборщику мусора и микрофризы), а где-то крупные объекты «Игрок» и «Счёт» сделаны `struct` и при каждом присваивании молча копируют десятки полей — это не только медленно, но и опасно, потому что мутации через свойства иногда «теряются». Лидер команды просит вас навести порядок: выбрать правильную форму типа для каждой сущности, объяснить выбор и доказать его измерениями.

Цель упражнения — не просто повторить синтаксис `struct`/`class`, а прочувствовать на собственном коде разницу в семантике. Вы создадите четыре типа: ссылочный класс `Player`, изменяемый `struct Point`, `readonly record struct Money` и `ref struct SpanWalker`. Для каждого вы напишете мини-демонстрацию: присваивание, сравнение, мутацию, и посмотрите, что реально происходит с памятью и равенством. Вы убедитесь, что value-семантика `struct` — это не косметика, а поведение, которое ломает привычные по классам паттерны (общий объект, общие правки), но зато даёт предсказуемость и скорость для маленьких значений.

Дополнительно вы измерите боксинг: поместите `struct` в нетипизированный контейнер `List<object>` и в типизированный `List<PointStruct>`, сравните количество аллокаций через `GC.GetAllocatedBytesForCurrentThread()`. Это даст вам интуицию, почему Microsoft Learn настойчиво рекомендует `struct` только для маленьких неделимых значений, а для всего остального — `class`. К концу упражнения вы должны уметь аргументированно ответить на вопрос «здесь нужен `struct` или `class`?» не «потому что так красивее», а «потому что значение ≤ 16 байт, неделимо, неизменяемо, и мы хотим избежать аллокаций».

#### Что нужно сделать (пошагово)

1. Создайте новый проект консольного приложения. Выполните в терминале:
   ```
   dotnet new console -n StructVsClass -o StructVsClass --framework net8.0
   cd StructVsClass
   ```
   Убедитесь, что в `StructVsClass.csproj` указано `<TargetFramework>net8.0</TargetFramework>` и `<LangVersion>latest</LangVersion>` (или `12`). При желании добавьте `<Nullable>enable</Nullable>` и `<ImplicitUsings>enable</ImplicitUsings>`.

2. В файле `Program.cs` (или в отдельных файлах `Player.cs`, `Point.cs`, `Money.cs`, `SpanWalker.cs`) опишите четыре типа:
   - `public class Player { public string Name { get; set; } public int Score { get; set; } }` — ссылочный тип для сущности «игрок».
   - `public struct Point { public int X { get; init; } public int Y { get; init; } public double DistanceTo(Point other) => Math.Sqrt(Math.Pow(other.X - X, 2) + Math.Pow(other.Y - Y, 2)); }` — тип-значение для координаты.
   - `public readonly record struct Money(decimal Amount, string Currency)` с методом `Add(Money other)`, использующим `with`-выражение и бросающим `InvalidOperationException` при несовпадении валюты.
   - `public ref struct SpanWalker` по образцу из урока, читающий байты из `ReadOnlySpan<byte>`.

3. Напишите метод `DemoClass()` и вызовите его из `Main`. Создайте `var a = new Player { Name = "Alice", Score = 10 };`, затем `var b = a;`, затем `b.Score = 99;`. Выведите `a.Score`. Ожидаемый вывод: `99` — докажите себе, что `a` и `b` смотрят на один объект в куче.

4. Напишите `DemoStruct()`. Создайте `var p1 = new Point { X = 0, Y = 0 }; var p2 = p1;`. Попробуйте раскомментировать `p2.X = 5;` (если `X` имеет `init`, компилятор не даст — объясните почему; если сделаете `set`, то `p1.X` останется `0`). Выведите `p1.X` — должно быть `0`, потому что `p2` — независимая копия. Затем покажите `p1.DistanceTo(new Point { X = 3, Y = 4 })` — ожидается `5`.

5. Напишите `DemoRecordStruct()`. Создайте `var m1 = new Money(10m, "RUB"); var m2 = m1 with { Amount = 20m };`. Выведите `m1 == m2` — `False`, потому что сравнение по значению, а значения разные. Выведите `m1` — благодаря синтезированному `ToString` получите `Money { Amount = 10, Currency = RUB }`. Покажите `m1.Add(new Money(5m, "RUB"))` → `Money { Amount = 15, Currency = RUB }` и убедитесь, что `m1.Add(new Money(5m, "USD"))` бросает исключение.

6. Напишите `DemoRefStruct()`. Создайте `byte[] buffer = { 1, 2, 3, 4 }; var walker = new SpanWalker(buffer);` и в цикле `while (walker.HasMore) Console.Write(walker.ReadByte() + " ");` — ожидаемый вывод `1 2 3 4 `. Затем попробуйте (и закомментируйте с пояснением) нарушить ограничение: например, вернуть `SpanWalker` из метода или положить его в `List<SpanWalker>` — компилятор выдаст ошибку. Запишите текст ошибки в комментарий.

7. Напишите `DemoBoxing()`. Сравните две ветки:
   ```csharp
   long before1 = GC.GetAllocatedBytesForCurrentThread();
   var boxed = new List<object>();
   for (int i = 0; i < 100_000; i++) boxed.Add(new Point { X = i, Y = i });
   long after1 = GC.GetAllocatedBytesForCurrentThread();

   long before2 = GC.GetAllocatedBytesForCurrentThread();
   var typed = new List<Point>();
   for (int i = 0; i < 100_000; i++) typed.Add(new Point { X = i, Y = i });
   long after2 = GC.GetAllocatedBytesForCurrentThread();

   Console.WriteLine($"boxed: {after1 - before1} bytes");
   Console.WriteLine($"typed: {after2 - before2} bytes");
   ```
   Запустите `dotnet run -c Release`. Зафиксируйте числа в отчёте `REPORT.md` и объясните, почему типизированный список аллоцирует значительно меньше (нет бокса, `Point` хранится inline в массиве списка).

8. Запустите проект: `dotnet run`. Убедитесь, что вывод соответствует ожиданиям: `99`, `0`, `5`, `False`, `Money { Amount = 10, Currency = RUB }`, `Money { Amount = 15, Currency = RUB }`, `1 2 3 4 ` и две строки с байтами аллокаций.

9. Создайте файл `REPORT.md` рядом с проектом. В нём таблицей сравните четыре типа по столбцам: «где живёт значение», «семантика присваивания», «изменяемость по умолчанию», «боксится ли в object», «подходит для». Кратко (3–5 предложений) обоснуйте выбор типа для каждой сущности.

10. (Самопроверка) Раскомментируйте в уме рекурсивную попытку `struct Node { public Node Next; }` — компилятор должен выдать ошибку цикла (CS0523). Запишите в `REPORT.md`, почему для графов нужен `class`, и перепишите `Node` как `class Node { public Node? Next; }`.

#### Требования к решению

- Проект собирается командой `dotnet build` без предупреждений (уровень Warning 3) под .NET 8 / C# 12. Запрещено подавлять предупреждения через `#pragma` или `<NoWarn>`.
- Все четыре типа (`Player`, `Point`, `Money`, `SpanWalker`) реализованы ровно по спецификации урока: `Player` — `class` с авто-свойствами `get; set;`; `Point` — `struct` со свойствами `init` и методом `DistanceTo`; `Money` — `readonly record struct` с позиционным синтаксисом, методом `Add` и `with`; `SpanWalker` — `ref struct` с полем `ReadOnlySpan<byte>`.
- Демонстрационные методы вызываются из `Main`/`Top-level statements` и дают ровно тот вывод, что описан в шагах. Любое отклонение должно быть пояснено в `REPORT.md`.
- Измерение бокса использует `GC.GetAllocatedBytesForCurrentThread()` до и после цикла, запускается в `-c Release`, и числа зафиксированы в `REPORT.md` с указанием машины.
- Код должен честно показывать, что `var b = a;` для класса делит объект, а `var p2 = p1;` для `struct` — копирует значение. Запрещено «помогать» делу через `ref`/`in` параметры там, где упражнение требует показать копирование.
- Используется `nullable enable`: `Player.Name` — `string` (non-null), `Node.Next` — `Node?`. Ссылочные поля `struct` помечаются осознанно.
- Все тексты в `Console.WriteLine` двуязычные (`RU / EN`) или сопровождены комментарием на обоих языках, как в уроке.
- `REPORT.md` содержит таблицу сравнения типов, числа аллокаций и обоснование выбора — не менее 150 слов.

#### Тонкости и подводные камни

- **Value vs reference semantics — главное.** Для `class` присваивание копирует ссылку: оба имени смотрят на один объект, мутация через одно видна через другое. Для `struct` присваивание копирует значение целиком: два независимых значения, мутация одного не касается другого. Это ломает интуицию, перенесённую с классов, особенно когда вы передаёте `struct` в метод и пытаетесь изменить его внутри — изменение потеряется после возврата.
- **Боксинг при приведении struct к object.** Любое приведение `struct` к `object` (или к интерфейсу, или помещение в `List<object>`, `ArrayList`, не-обобщённую коллекцию) аллоцирует копию значения в куче. Это незаметная аллокация: тип вроде «маленький и безопасный» вдруг ведёт себя как класс по стоимости. Поэтому `List<Point>` лучше `List<object>`.
- **Mutable struct опасности.** Изменение свойства у `struct`, возвращённого методом (`GetPoint().X = 5;`), не компилируется: вы мутировали бы временную копию, которая тут же выбрасывается. Ещё хуже — `List<Point>[0].X = 5;` тоже не работает: индексатор возвращает копию. Решение — `readonly struct` + `with`, или возврат нового значения, или `ref`-возврат.
- **readonly struct.** Помечая `struct` как `readonly`, вы запрещаете мутацию полей и обещаете компилятору, что при вызове метода через `in`-параметр защитная копия не нужна — это ускоряет код и снимает класс багов. `readonly record struct` добавляет ещё value-равенство и `with`.
- **Когда выбирать struct.** Эвристика Microsoft: ≤ 16 байт, логически неделим, желательно неизменяем. Если хотя бы одно условие нарушено — берите `class`. Большие `struct` копируются при каждом присваивании и передаче аргумента; рекурсивные `struct Node { Node Next; }` вообще не компилируются (CS0523).
- **Размер и аллокация: stack vs heap.** Локальные `struct` обычно живут на стеке метода и не аллоцируются в куче — но это деталь реализации (JIT может положить значение в поле класса, тогда оно переедет в кучу). Главное для разработчика — отсутствие самостоятельной аллокации объекта при `new Point()` для `struct` (в отличие от `new Player()` для класса).
- **struct без параметров-конструктора.** До C# 10 у `struct` не было неявного конструктора без параметров; поле инициализировалось `default`. С C# 10 можно объявить явный беспараметрный конструктор, и `new Point()` вызовет его, но `default(Point)` по-прежнему даёт «нулевой» экземпляр. Не путайте.
- **record struct vs record.** `record struct` — тип-значение с синтезированными `Equals`/`GetHashCode` по значениям, `with`, деконструкцией, `ToString`. Обычный `record` (без `struct`) — ссылочный тип. `with` для `record struct` создаёт новый экземпляр-копию с изменённым полем; для `class record` — тоже новый, но на куче.
- **ref struct ограничения.** `ref struct` нельзя боксировать, класть в поле класса, в массив, захватывать в лямбду, делать полем `async`-метода, возвращать из `async`. Это цена за гарантию «только стек». Используйте осознанно.

#### Критерии приёмки

- [ ] Проект `StructVsClass` собирается под .NET 8 / C# 12 без предупреждений уровня 3.
- [ ] Реализованы `Player` (class), `Point` (struct), `Money` (readonly record struct), `SpanWalker` (ref struct) по спецификации.
- [ ] `DemoClass` показывает, что `b.Score = 99` меняет `a.Score` → вывод `99`.
- [ ] `DemoStruct` показывает независимость копий → вывод `0` для `p1.X`.
- [ ] `DistanceTo(new Point{3,4})` возвращает `5`.
- [ ] `DemoRecordStruct` показывает `m1 == m2` → `False` и `with`-копию с другим `Amount`.
- [ ] `Money.Add` складывает одинаковые валюты и бросает `InvalidOperationException` для разных.
- [ ] `DemoRefStruct` печатает `1 2 3 4 ` и содержит закомментированную попытку нарушения с текстом ошибки компилятора.
- [ ] `DemoBoxing` запускается в Release и печатает две строки с байтами аллокаций; типизированный список аллоцирует меньше.
- [ ] В `REPORT.md` есть таблица сравнения четырёх типов и обоснование выбора — ≥ 150 слов.
- [ ] В `REPORT.md` зафиксированы числа аллокаций и описание машины.
- [ ] Объяснено, почему `struct Node { Node Next; }` не компилируется (CS0523), и `Node` переписан как `class`.
- [ ] Код двуязычный: комментарии и выводы на RU + EN, как в уроке.
- [ ] Включён `nullable enable`, `Player.Name` — non-null, `Node.Next` — `Node?`.
- [ ] Запрещено подавление предупреждений `#pragma`/`<NoWarn>`.
- [ ] Команда `dotnet run` воспроизводит весь ожидаемый вывод.

#### Подсказки (без прямого ответа)

- Подумайте, что именно копируется при `var b = a;`: для класса — «адрес объекта», для `struct` — «содержимое». Если бы вы могли заглянуть в память, что вы бы увидели по адресам `a` и `b`?
- Для измерения бокса используйте `GC.GetAllocatedBytesForCurrentThread()` — это точнее, чем `GC.GetTotalMemory`, потому что не зависит от других потоков и сборок.
- Если компилятор ругается на `GetPoint().X = 5;`, спросите себя: чему именно вы присваиваете? Копии, которая живёт до конца выражения?
- `with` для `record struct` синтезирует конструктор копии и присваивание поля. Что будет, если поля помечены `init`, а не `set`?
- Для `ref struct` попробуйте `static SpanWalker Make() => new SpanWalker(default);` — компилятор скажет, что `ref struct` нельзя вернуть из обычного метода? (Подсказка: вернуть-то можно, нельзя из `async` и нельзя боксировать. Проверьте, какое именно ограничение нарушится в вашем эксперименте.)

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — struct vs class vs record struct vs ref struct
// Двуязычные комментарии RU / EN

using System;
using System.Collections.Generic;

// 1) class — ссылочный тип, живёт в куче / reference type, lives on the heap
public class Player
{
    public string Name { get; set; } = "";   // имя игрока (non-null) / player name
    public int Score { get; set; }           // счёт / score
}

// 2) struct — тип-значение, копируется целиком / value type, copied wholesale
public struct Point
{
    public int X { get; init; }              // координата X / X coordinate
    public int Y { get; init; }              // координата Y / Y coordinate

    // дистанция до другой точки / distance to another point
    public double DistanceTo(Point other) =>
        Math.Sqrt(Math.Pow(other.X - X, 2) + Math.Pow(other.Y - Y, 2));
}

// 3) readonly record struct (C# 10) — неизменяемое значение + удобство record
//    immutable value + record ergonomics (Equals, with, deconstruct, ToString)
public readonly record struct Money(decimal Amount, string Currency)
{
    // сложение с проверкой валюты / addition with currency check
    public Money Add(Money other) =>
        Currency == other.Currency
            ? this with { Amount = Amount + other.Amount }   // новый экземпляр / new instance
            : throw new InvalidOperationException(
                "Валюты должны совпадать / currencies must match");
}

// 4) ref struct — только стек, без бокса и кучи / stack-only, no boxing, no heap
public ref struct SpanWalker
{
    private readonly ReadOnlySpan<byte> _data;  // срез байтов / byte slice
    private int _pos;                            // текущая позиция / current position

    public SpanWalker(ReadOnlySpan<byte> data) { _data = data; _pos = 0; }
    public bool HasMore => _pos < _data.Length;
    public byte ReadByte() => _data[_pos++];     // читать и сдвинуться / read and advance
}

// 5) рекурсивный узел: class, не struct / recursive node: class, not struct
public class Node
{
    public int Value { get; init; }
    public Node? Next { get; set; }             // nullable, чтобы разорвать цикл / nullable to break cycle
}

public static class Program
{
    public static void Main()
    {
        DemoClass();
        DemoStruct();
        DemoRecordStruct();
        DemoRefStruct();
        DemoBoxing();
    }

    static void DemoClass()
    {
        var a = new Player { Name = "Alice", Score = 10 };
        var b = a;                          // копируется ссылка / copy the reference
        b.Score = 99;
        Console.WriteLine($"class: a.Score = {a.Score}  (ожидается 99 / expected 99)");
    }

    static void DemoStruct()
    {
        var p1 = new Point { X = 0, Y = 0 };
        var p2 = p1;                        // копируется значение / copy the value
        // p2.X = 5;                        // init-свойство: нельзя; даже с set p1.X остался бы 0
        Console.WriteLine($"struct: p1.X = {p1.X}  (ожидается 0 / expected 0)");
        var d = p1.DistanceTo(new Point { X = 3, Y = 4 });
        Console.WriteLine($"struct: distance = {d}  (ожидается 5 / expected 5)");
    }

    static void DemoRecordStruct()
    {
        var m1 = new Money(10m, "RUB");
        var m2 = m1 with { Amount = 20m };  // новый экземпляр / new instance
        Console.WriteLine($"record struct: m1 == m2 = {m1 == m2}  (ожидается False / expected False)");
        Console.WriteLine($"record struct: m1 = {m1}");
        var sum = m1.Add(new Money(5m, "RUB"));
        Console.WriteLine($"record struct: m1.Add(5 RUB) = {sum}");
        try { _ = m1.Add(new Money(5m, "USD")); }
        catch (InvalidOperationException ex) { Console.WriteLine($"record struct: {ex.Message}"); }
    }

    static void DemoRefStruct()
    {
        byte[] buffer = { 1, 2, 3, 4 };
        var walker = new SpanWalker(buffer);
        Console.Write("ref struct: ");
        while (walker.HasMore) Console.Write(walker.ReadByte() + " ");
        Console.WriteLine("(ожидается 1 2 3 4 / expected 1 2 3 4)");
        // Попытка нарушения / attempt to violate:
        //   static SpanWalker Make() => new SpanWalker(default);  // OK — обычный метод
        //   async Task<SpanWalker> MakeAsync() => new SpanWalker(default); // CS4007 / нельзя в async
        //   var list = new List<SpanWalker>();                    // CS8779 — ref struct нельзя в дженерике
    }

    static void DemoBoxing()
    {
        const int n = 100_000;

        long before1 = GC.GetAllocatedBytesForCurrentThread();
        var boxed = new List<object>(n);
        for (int i = 0; i < n; i++) boxed.Add(new Point { X = i, Y = i });
        long after1 = GC.GetAllocatedBytesForCurrentThread();

        long before2 = GC.GetAllocatedBytesForCurrentThread();
        var typed = new List<Point>(n);
        for (int i = 0; i < n; i++) typed.Add(new Point { X = i, Y = i });
        long after2 = GC.GetAllocatedBytesForCurrentThread();

        Console.WriteLine($"boxing: List<object> allocated {after1 - before1} bytes");
        Console.WriteLine($"typed:  List<Point>   allocated {after2 - before2} bytes");
    }
}
```

Разбор по строкам. `Player` объявлен как `class` — это сущность с идентичностью, её разумно мутировать и делить между владельцами (`b = a` копирует ссылку, `b.Score = 99` виден через `a`). `Point` — `struct` со свойствами `init`: координата логически неделима, занимает 8 байт, неизменяема после создания — идеальный кандидат на тип-значение; присваивание `p2 = p1` делает независимую копию, поэтому `p1.X` остаётся `0`. Метод `DistanceTo` принимает `Point` по значению — для маленькой структуры это дешевле, чем ссылка. `Money` — `readonly record struct`: позиционный синтаксис даёт `Amount`/`Currency` как init-свойства, компилятор синтезирует `Equals`/`GetHashCode` по значениям, `ToString` в виде `Money { Amount = 10, Currency = RUB }`, деконструкцию и поддержку `with`. Метод `Add` использует `with` — создаёт новый экземпляр, оставляя исходный нетронутым (неизменяемость). Проверка валюты — domain-логика; `InvalidOperationException` сигнализирует о нарушении инварианта. `SpanWalker` — `ref struct`: хранит `ReadOnlySpan<byte>`, который сам является `ref struct`, поэтому вся структура обязана быть стековой; это даёт нулевые аллокации при чтении буфера, но запрещает бокс, поле класса и `async`-захват. `Node` переписан как `class` с `Node? Next`, потому что рекурсивный `struct Node { Node Next; }` не компилируется (CS0523) — тип-значение не может содержать себя по значению (бесконечный размер). `DemoBoxing` измеряет разницу: `List<object>` боксит каждый `Point` (аллоцирует копию в куче + объект-обёртку), `List<Point>` хранит значения inline в массиве — отсюда многократная разница в байтах. Применённые концепции урока: value vs reference semantics (демо `a`/`b` и `p1`/`p2`), боксинг при `object` (`DemoBoxing`), опасность mutable struct (init + readonly), `readonly record struct` и `with`, `ref struct` ограничения, эвристика выбора (≤ 16 байт, неделимость, неизменяемость), CS0523 для рекурсивных struct.

#### Задания на углубление (бонус)

1. Добавьте `readonly struct Point` (вместо `struct`) и измерьте, ускоряет ли это вызов `DistanceTo` через `in`-параметр на 10 млн итераций в Release. Сравните с обычным `struct`.
2. Реализуйте `readonly record struct Color(byte R, byte G, byte B)` с методами `WithR(byte)`, `WithG(byte)`, `WithB(byte)` через `with` и операторами `+(Color, Color)` (с насыщением 0–255). Покажите value-равенство двух одинаковых цветов.
3. Сравните `record struct Money` и `record class MoneyRef` (ссылочный record) по `Equals`/`GetHashCode` на 1 млн объектов: какой быстрее и почему? Запишите в `REPORT.md`.
4. Напишите `ref struct LineReader(ReadOnlySpan<char> text)`, который построчно читает текст через `Span<char>` и `IndexOf('\n')`. Объясните, почему такой тип нельзя вернуть из `async`-метода, и перепишите асинхронную версию на `class` с `Memory<char>`.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are joining a team that writes a small 2D game engine and an internal accounting library. The codebase has been mixing `class` and `struct` at random: in some places points and colors are declared as classes (so every rendered frame allocates thousands of heap objects, giving the garbage collector extra work and causing micro-stutters), while in other places large "Player" and "Account" objects are made `struct` and silently copy dozens of fields on every assignment — which is not only slow but also dangerous, because mutations through properties sometimes "get lost". The team lead asks you to clean this up: pick the right shape for each entity, explain the choice, and back it with measurements.

The goal of the exercise is not to rehash the syntax of `struct`/`class`, but to feel the difference in semantics on your own code. You will create four types: a reference class `Player`, a mutable `struct Point`, a `readonly record struct Money`, and a `ref struct SpanWalker`. For each you will write a mini-demo: assignment, comparison, mutation, and observe what actually happens with memory and equality. You will see that the value semantics of a `struct` is not a cosmetic detail but a behaviour that breaks the patterns you are used to with classes (shared object, shared edits), but in return gives predictability and speed for small values.

Additionally you will measure boxing: place a `struct` both in an untyped `List<object>` and in a typed `List<Point>`, and compare allocation counts using `GC.GetAllocatedBytesForCurrentThread()`. This will give you the intuition for why Microsoft Learn insists that `struct` is only appropriate for small indivisible values, while everything else should be a `class`. By the end of the exercise you should be able to answer the question "should this be a `struct` or a `class`?" not with "because it looks nicer" but with "because the value is ≤ 16 bytes, indivisible, immutable, and we want to avoid allocations". This is exactly the reasoning the lesson trains: value vs reference semantics, boxing, mutable-struct dangers, `readonly struct`, and the heuristic for choosing `struct`.

#### What to do step by step

1. Create a new console application. In the terminal run:
   ```
   dotnet new console -n StructVsClass -o StructVsClass --framework net8.0
   cd StructVsClass
   ```
   Verify that `StructVsClass.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and `<LangVersion>latest</LangVersion>` (or `12`). Optionally add `<Nullable>enable</Nullable>` and `<ImplicitUsings>enable</ImplicitUsings>`.

2. In `Program.cs` (or in separate files `Player.cs`, `Point.cs`, `Money.cs`, `SpanWalker.cs`) describe four types:
   - `public class Player { public string Name { get; set; } public int Score { get; set; } }` — a reference type for the "player" entity.
   - `public struct Point { public int X { get; init; } public int Y { get; init; } public double DistanceTo(Point other) => Math.Sqrt(Math.Pow(other.X - X, 2) + Math.Pow(other.Y - Y, 2)); }` — a value type for a coordinate.
   - `public readonly record struct Money(decimal Amount, string Currency)` with an `Add(Money other)` method using a `with` expression and throwing `InvalidOperationException` when the currencies differ.
   - `public ref struct SpanWalker` modelled on the lesson, reading bytes from a `ReadOnlySpan<byte>`.

3. Write a `DemoClass()` method and call it from `Main`. Create `var a = new Player { Name = "Alice", Score = 10 };`, then `var b = a;`, then `b.Score = 99;`. Print `a.Score`. Expected output: `99` — proving to yourself that `a` and `b` look at one heap object.

4. Write `DemoStruct()`. Create `var p1 = new Point { X = 0, Y = 0 }; var p2 = p1;`. Try uncommenting `p2.X = 5;` (with `init` the compiler refuses — explain why; with `set`, `p1.X` would still be `0`). Print `p1.X` — it must be `0`, because `p2` is an independent copy. Then show `p1.DistanceTo(new Point { X = 3, Y = 4 })` — expect `5`.

5. Write `DemoRecordStruct()`. Create `var m1 = new Money(10m, "RUB"); var m2 = m1 with { Amount = 20m };`. Print `m1 == m2` — `False`, because equality is value-based and the values differ. Print `m1` — thanks to the synthesized `ToString` you get `Money { Amount = 10, Currency = RUB }`. Show `m1.Add(new Money(5m, "RUB"))` → `Money { Amount = 15, Currency = RUB }` and confirm `m1.Add(new Money(5m, "USD"))` throws.

6. Write `DemoRefStruct()`. Create `byte[] buffer = { 1, 2, 3, 4 }; var walker = new SpanWalker(buffer);` and in a loop `while (walker.HasMore) Console.Write(walker.ReadByte() + " ");` — expected output `1 2 3 4 `. Then try (and comment out with an explanation) to break the constraint: for instance, return a `SpanWalker` from a method or put one in `List<SpanWalker>` — the compiler will error. Record the error text in a comment.

7. Write `DemoBoxing()`. Compare two branches:
   ```csharp
   long before1 = GC.GetAllocatedBytesForCurrentThread();
   var boxed = new List<object>();
   for (int i = 0; i < 100_000; i++) boxed.Add(new Point { X = i, Y = i });
   long after1 = GC.GetAllocatedBytesForCurrentThread();

   long before2 = GC.GetAllocatedBytesForCurrentThread();
   var typed = new List<Point>();
   for (int i = 0; i < 100_000; i++) typed.Add(new Point { X = i, Y = i });
   long after2 = GC.GetAllocatedBytesForCurrentThread();

   Console.WriteLine($"boxed: {after1 - before1} bytes");
   Console.WriteLine($"typed: {after2 - before2} bytes");
   ```
   Run `dotnet run -c Release`. Record the numbers in `REPORT.md` and explain why the typed list allocates far less (no boxing, `Point` stored inline in the list's array).

8. Run the project: `dotnet run`. Confirm the output matches expectations: `99`, `0`, `5`, `False`, `Money { Amount = 10, Currency = RUB }`, `Money { Amount = 15, Currency = RUB }`, `1 2 3 4 ` and two allocation-byte lines.

9. Create `REPORT.md` next to the project. In it, tabulate the four types by columns: "where the value lives", "assignment semantics", "default mutability", "boxes to object?", "good for". Briefly (3–5 sentences) justify the type choice for each entity.

10. (Self-check) Mentally uncomment a recursive attempt `struct Node { public Node Next; }` — the compiler must emit a cycle error (CS0523). Record in `REPORT.md` why graphs require `class`, and rewrite `Node` as `class Node { public Node? Next; }`.

#### Requirements

- The project builds with `dotnet build` without warnings (level 3) on .NET 8 / C# 12. Suppressing warnings via `#pragma` or `<NoWarn>` is forbidden.
- All four types (`Player`, `Point`, `Money`, `SpanWalker`) are implemented exactly per the lesson spec: `Player` — `class` with `get; set;` auto-properties; `Point` — `struct` with `init` properties and a `DistanceTo` method; `Money` — `readonly record struct` with positional syntax, an `Add` method and `with`; `SpanWalker` — `ref struct` with a `ReadOnlySpan<byte>` field.
- Demo methods are called from `Main`/top-level statements and produce exactly the output described in the steps. Any deviation must be explained in `REPORT.md`.
- The boxing measurement uses `GC.GetAllocatedBytesForCurrentThread()` before and after the loop, runs in `-c Release`, and the numbers are recorded in `REPORT.md` with the machine described.
- The code must honestly show that `var b = a;` for a class shares the object, while `var p2 = p1;` for a `struct` copies the value. "Helping" with `ref`/`in` parameters where the exercise asks to show copying is forbidden.
- `nullable enable` is on: `Player.Name` is `string` (non-null), `Node.Next` is `Node?`. Reference fields of a `struct` are deliberate.
- All `Console.WriteLine` text is bilingual (`RU / EN`) or accompanied by comments in both languages, matching the lesson.
- `REPORT.md` contains the type-comparison table, allocation numbers, and a justification of choices — at least 150 words.

#### Pitfalls

- **Value vs reference semantics — the crux.** For a `class`, assignment copies the reference: both names look at one object, a mutation through one is visible through the other. For a `struct`, assignment copies the value wholesale: two independent values, a mutation of one never touches the other. This breaks intuition carried over from classes, especially when you pass a `struct` into a method and try to mutate it inside — the change is lost after return.
- **Boxing when casting a struct to object.** Any cast of a `struct` to `object` (or to an interface, or storage in `List<object>`, `ArrayList`, a non-generic collection) allocates a copy of the value on the heap. It is a hidden allocation: a type that seems "small and safe" suddenly behaves like a class in cost. That is why `List<Point>` beats `List<object>`.
- **Mutable struct dangers.** Mutating a property of a `struct` returned by a method (`GetPoint().X = 5;`) does not compile: you would be mutating a temporary copy that is immediately discarded. Worse, `List<Point>[0].X = 5;` does not work either: the indexer returns a copy. The fix is `readonly struct` + `with`, or returning a new value, or a `ref` return.
- **readonly struct.** Marking a `struct` `readonly` forbids field mutation and promises the compiler that no defensive copy is needed when calling a method through an `in` parameter — this speeds up code and removes a whole class of bugs. `readonly record struct` adds value equality and `with`.
- **When to choose struct.** Microsoft's heuristic: ≤ 16 bytes, logically indivisible, ideally immutable. If any condition is violated, take `class`. Large `struct`s are copied on every assignment and argument pass; recursive `struct Node { Node Next; }` does not even compile (CS0523).
- **Size and allocation: stack vs heap.** Local `struct` values usually live on the method's stack and are not heap-allocated — but that is an implementation detail (the JIT may place a value into a class field, moving it to the heap). The key point for the developer is that `new Point()` for a `struct` does not allocate a standalone object (unlike `new Player()` for a class).
- **struct without a parameterless constructor.** Before C# 10 a `struct` had no implicit parameterless constructor; the fields were initialized to `default`. Since C# 10 you can declare an explicit parameterless constructor, and `new Point()` calls it, but `default(Point)` still gives a "zero" instance. Do not confuse the two.
- **record struct vs record.** A `record struct` is a value type with synthesized `Equals`/`GetHashCode` by field values, plus `with`, deconstruction, and `ToString`. A plain `record` (without `struct`) is a reference type. `with` on a `record struct` creates a new copy instance with one field changed; on a `class record` it also creates a new one, but on the heap.
- **ref struct limits.** A `ref struct` cannot be boxed, stored in a class field, put in an array, captured in a lambda, made a field of an `async` method, or returned from `async`. That is the price of a "stack-only" guarantee. Use it deliberately.

#### Acceptance criteria

- [ ] The `StructVsClass` project builds on .NET 8 / C# 12 with no level-3 warnings.
- [ ] `Player` (class), `Point` (struct), `Money` (readonly record struct), `SpanWalker` (ref struct) are implemented per spec.
- [ ] `DemoClass` shows that `b.Score = 99` changes `a.Score` → prints `99`.
- [ ] `DemoStruct` shows copy independence → prints `0` for `p1.X`.
- [ ] `DistanceTo(new Point{3,4})` returns `5`.
- [ ] `DemoRecordStruct` shows `m1 == m2` → `False` and a `with`-copy with a different `Amount`.
- [ ] `Money.Add` adds same currencies and throws `InvalidOperationException` for differing ones.
- [ ] `DemoRefStruct` prints `1 2 3 4 ` and contains a commented-out violation attempt with the compiler error text.
- [ ] `DemoBoxing` runs in Release and prints two allocation-byte lines; the typed list allocates less.
- [ ] `REPORT.md` contains the four-type comparison table and a justification of choices — ≥ 150 words.
- [ ] `REPORT.md` records the allocation numbers and describes the machine.
- [ ] It is explained why `struct Node { Node Next; }` fails to compile (CS0523), and `Node` is rewritten as `class`.
- [ ] The code is bilingual: comments and output in RU + EN, as in the lesson.
- [ ] `nullable enable` is on; `Player.Name` is non-null, `Node.Next` is `Node?`.
- [ ] No `#pragma`/`<NoWarn>` warning suppression.
- [ ] `dotnet run` reproduces the entire expected output.

#### Hints (no direct answer)

- Think about what exactly is copied by `var b = a;`: for a class — "the object's address"; for a `struct` — "the contents". If you could look into memory, what would you see at the addresses of `a` and `b`?
- For boxing measurement use `GC.GetAllocatedBytesForCurrentThread()` — it is more precise than `GC.GetTotalMemory` because it is per-thread and ignores other threads' allocations and collections.
- If the compiler complains about `GetPoint().X = 5;`, ask yourself: what exactly are you assigning to? A copy that only lives until the end of the statement?
- `with` on a `record struct` synthesizes a copy constructor and a field assignment. What happens if the fields are `init` rather than `set`?
- For a `ref struct` try `static SpanWalker Make() => new SpanWalker(default);` — does the compiler say a `ref struct` cannot be returned from a normal method? (Hint: returning is fine; it cannot be boxed or returned from `async`. Find which constraint your experiment actually breaks.)

#### Reference solution walk-through (EN)

```csharp
// C# 12 / .NET 8 — struct vs class vs record struct vs ref struct
// Bilingual comments RU / EN

using System;
using System.Collections.Generic;

// 1) class — reference type, lives on the heap
public class Player
{
    public string Name { get; set; } = "";   // player name (non-null)
    public int Score { get; set; }           // score
}

// 2) struct — value type, copied wholesale
public struct Point
{
    public int X { get; init; }              // X coordinate
    public int Y { get; init; }              // Y coordinate

    public double DistanceTo(Point other) =>
        Math.Sqrt(Math.Pow(other.X - X, 2) + Math.Pow(other.Y - Y, 2));
}

// 3) readonly record struct (C# 10) — immutable value + record ergonomics
public readonly record struct Money(decimal Amount, string Currency)
{
    public Money Add(Money other) =>
        Currency == other.Currency
            ? this with { Amount = Amount + other.Amount }   // new instance
            : throw new InvalidOperationException(
                "Валюты должны совпадать / currencies must match");
}

// 4) ref struct — stack-only, no boxing, no heap
public ref struct SpanWalker
{
    private readonly ReadOnlySpan<byte> _data;  // byte slice
    private int _pos;                            // current position

    public SpanWalker(ReadOnlySpan<byte> data) { _data = data; _pos = 0; }
    public bool HasMore => _pos < _data.Length;
    public byte ReadByte() => _data[_pos++];     // read and advance
}

// 5) recursive node: class, not struct
public class Node
{
    public int Value { get; init; }
    public Node? Next { get; set; }             // nullable to break the cycle
}

public static class Program
{
    public static void Main()
    {
        DemoClass();
        DemoStruct();
        DemoRecordStruct();
        DemoRefStruct();
        DemoBoxing();
    }

    static void DemoClass()
    {
        var a = new Player { Name = "Alice", Score = 10 };
        var b = a;                          // copy the reference
        b.Score = 99;
        Console.WriteLine($"class: a.Score = {a.Score}  (expected 99)");
    }

    static void DemoStruct()
    {
        var p1 = new Point { X = 0, Y = 0 };
        var p2 = p1;                        // copy the value
        // p2.X = 5;                        // init-only: illegal; even with set p1.X would stay 0
        Console.WriteLine($"struct: p1.X = {p1.X}  (expected 0)");
        var d = p1.DistanceTo(new Point { X = 3, Y = 4 });
        Console.WriteLine($"struct: distance = {d}  (expected 5)");
    }

    static void DemoRecordStruct()
    {
        var m1 = new Money(10m, "RUB");
        var m2 = m1 with { Amount = 20m };  // new instance
        Console.WriteLine($"record struct: m1 == m2 = {m1 == m2}  (expected False)");
        Console.WriteLine($"record struct: m1 = {m1}");
        var sum = m1.Add(new Money(5m, "RUB"));
        Console.WriteLine($"record struct: m1.Add(5 RUB) = {sum}");
        try { _ = m1.Add(new Money(5m, "USD")); }
        catch (InvalidOperationException ex) { Console.WriteLine($"record struct: {ex.Message}"); }
    }

    static void DemoRefStruct()
    {
        byte[] buffer = { 1, 2, 3, 4 };
        var walker = new SpanWalker(buffer);
        Console.Write("ref struct: ");
        while (walker.HasMore) Console.Write(walker.ReadByte() + " ");
        Console.WriteLine("(expected 1 2 3 4)");
        // Violation attempts:
        //   static SpanWalker Make() => new SpanWalker(default);  // OK — normal method
        //   async Task<SpanWalker> MakeAsync() => new SpanWalker(default); // CS4007 — not in async
        //   var list = new List<SpanWalker>();                    // CS8779 — ref struct not allowed as generic argument
    }

    static void DemoBoxing()
    {
        const int n = 100_000;

        long before1 = GC.GetAllocatedBytesForCurrentThread();
        var boxed = new List<object>(n);
        for (int i = 0; i < n; i++) boxed.Add(new Point { X = i, Y = i });
        long after1 = GC.GetAllocatedBytesForCurrentThread();

        long before2 = GC.GetAllocatedBytesForCurrentThread();
        var typed = new List<Point>(n);
        for (int i = 0; i < n; i++) typed.Add(new Point { X = i, Y = i });
        long after2 = GC.GetAllocatedBytesForCurrentThread();

        Console.WriteLine($"boxing: List<object> allocated {after1 - before1} bytes");
        Console.WriteLine($"typed:  List<Point>   allocated {after2 - before2} bytes");
    }
}
```

Line-by-line walk-through. `Player` is declared as a `class` — it is an identity-bearing entity, reasonable to mutate and share between owners (`b = a` copies the reference, `b.Score = 99` is visible through `a`). `Point` is a `struct` with `init` properties: a coordinate is logically indivisible, occupies 8 bytes, and is immutable after construction — an ideal value-type candidate; assigning `p2 = p1` makes an independent copy, so `p1.X` stays `0`. The `DistanceTo` method takes `Point` by value — for a small struct this is cheaper than a reference. `Money` is a `readonly record struct`: positional syntax gives `Amount`/`Currency` as init-only properties, and the compiler synthesizes value-based `Equals`/`GetHashCode`, a `ToString` of the form `Money { Amount = 10, Currency = RUB }`, deconstruction, and `with` support. The `Add` method uses `with` to produce a new instance, leaving the original untouched (immutability). The currency check is domain logic; `InvalidOperationException` signals a broken invariant. `SpanWalker` is a `ref struct`: it holds a `ReadOnlySpan<byte>`, which is itself a `ref struct`, so the whole struct must be stack-only — this gives zero allocations while reading a buffer, but forbids boxing, class fields, and `async` capture. `Node` is rewritten as a `class` with `Node? Next`, because a recursive `struct Node { Node Next; }` does not compile (CS0523) — a value type cannot contain itself by value (infinite size). `DemoBoxing` measures the gap: `List<object>` boxes every `Point` (allocating a heap copy plus a wrapper object), while `List<Point>` stores the values inline in its array — hence the multi-fold byte difference. Lesson concepts applied: value vs reference semantics (the `a`/`b` and `p1`/`p2` demos), boxing through `object` (`DemoBoxing`), the mutable-struct hazard (init + readonly), `readonly record struct` and `with`, `ref struct` constraints, the selection heuristic (≤ 16 bytes, indivisible, immutable), and CS0523 for recursive structs.

#### Going deeper (bonus)

1. Replace `struct Point` with `readonly struct Point` and measure whether passing it by `in` to `DistanceTo` speeds up 10M iterations in Release. Compare with a plain `struct`.
2. Implement `readonly record struct Color(byte R, byte G, byte B)` with `WithR`/`WithG`/`WithB` methods via `with` and `+(Color, Color)` operators (with 0–255 saturation). Show value equality of two identical colors.
3. Compare `record struct Money` against `record class MoneyRef` (a reference record) on `Equals`/`GetHashCode` over 1M objects: which is faster and why? Record the result in `REPORT.md`.
4. Write a `ref struct LineReader(ReadOnlySpan<char> text)` that reads lines via `Span<char>` and `IndexOf('\n')`. Explain why this type cannot be returned from an `async` method, and rewrite an async version as a `class` over `Memory<char>`.

---

#### Чек-лист сдачи / Submission checklist

RU:
- [ ] Проект `StructVsClass` собирается под .NET 8 / C# 12 без предупреждений уровня 3.
- [ ] Реализованы `Player`, `Point`, `Money`, `SpanWalker` по спецификации урока.
- [ ] Все пять демо-методов (`DemoClass` … `DemoBoxing`) вызваны из `Main` и дают ожидаемый вывод.
- [ ] Измерения бокса запущены в Release, числа зафиксированы в `REPORT.md`.
- [ ] `REPORT.md` содержит таблицу сравнения четырёх типов и обоснование выбора (≥ 150 слов).
- [ ] Объяснён CS0523 для рекурсивного `struct Node`, `Node` переписан как `class`.
- [ ] Код двуязычный (RU + EN комментарии/выводы), `nullable enable` включён.
- [ ] Нет подавления предупреждений через `#pragma`/`<NoWarn>`.
- [ ] Команда `dotnet run` воспроизводит весь вывод.

EN:
- [ ] The `StructVsClass` project builds on .NET 8 / C# 12 with no level-3 warnings.
- [ ] `Player`, `Point`, `Money`, `SpanWalker` are implemented per the lesson spec.
- [ ] All five demo methods (`DemoClass` … `DemoBoxing`) are called from `Main` and produce the expected output.
- [ ] Boxing measurements run in Release; numbers are recorded in `REPORT.md`.
- [ ] `REPORT.md` contains the four-type comparison table and a justification of choices (≥ 150 words).
- [ ] CS0523 for a recursive `struct Node` is explained; `Node` is rewritten as `class`.
- [ ] The code is bilingual (RU + EN comments/output); `nullable enable` is on.
- [ ] No warning suppression via `#pragma`/`<NoWarn>`.
- [ ] `dotnet run` reproduces the entire output.

#### Ресурсы / Resources
- [Microsoft Learn — Struct types (C# reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/struct)
- [Microsoft Learn — Records (C# reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/record)
- [Microsoft Learn — Choosing between class and struct](https://learn.microsoft.com/dotnet/standard/design-guidelines/choosing-between-class-and-struct)
- [Microsoft Learn — ref structs and span safety](https://learn.microsoft.com/dotnet/csharp/language-reference/language-specification/structs)
- [Microsoft Learn — `GC.GetAllocatedBytesForCurrentThread`](https://learn.microsoft.com/dotnet/api/system.gc.getallocatedbytesforcurrentthread)
