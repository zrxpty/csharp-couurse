---
[← К уроку M04-L09](lesson-M04-L09-namespaces-using.md) | [⬆ К модулю M04](../README.md) | [Следующее ДЗ →]()
---

### Домашнее задание M04-L09: Пространства имён, using, namespace / Homework M04-L09: Namespaces, using, namespace

**Урок / Lesson:** M04-L09
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться организовывать типы в пространства имён, применять file-scoped namespace, `global using` и алиасы, разрешать конфликты имён и следовать best practices из урока в реальном проекте на C# 12 / .NET 8. (EN) Learn to organize types into namespaces, apply file-scoped namespaces, `global using` and aliases, resolve name conflicts, and follow the lesson's best practices in a real C# 12 / .NET 8 project.

#### Связь с уроком / Connection to the lesson
(RU) Это задание напрямую закрепляет все ключевые темы урока M04-L09: классические и file-scoped пространства имён, директиву `using`, `global using`, неявные глобальные импорты (`ImplicitUsings`), алиасы и разрешение конфликтов одноимённых типов. Вы будете воспроизводить приёмы из демонстрационного кода урока (`ShopDemo`), но в самостоятельном проекте `InventoryTracker`, где конфликт возникает между двумя собственными типами `Point`. (EN) This assignment directly reinforces every key topic of lesson M04-L09: classic and file-scoped namespaces, the `using` directive, `global using`, implicit global imports (`ImplicitUsings`), aliases, and resolution of same-name type conflicts. You will reproduce the techniques from the lesson's demo code (`ShopDemo`) but in a standalone `InventoryTracker` project where the conflict arises between two of your own `Point` types.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы — backend-разработчик в небольшой компании, которая ведёт учёт складских остатков. Команда начала новый микросервис `InventoryTracker` на .NET 8, и архитектор попросил вас заложить правильную структуру пространств имён «с первого дня», чтобы по мере роста проекта типы было легко находить, а имена не конфликтовали. В проекте уже намечаются две независимые подсистемы, которым обе нужны координаты: геометрическое ядро (расчёт расстояний,immutable-точки) и подсистема рендеринга (рисование на экране, изменяемые точки с цветом). Естественно, обе команды назвали свой тип `Point` — это классический пример конфликта имён, описанный в уроке. Вам предстоит спроектировать проект так, чтобы оба типа `Point` мирно сосуществовали, а код оставался читаемым: без длинных полных имён на каждой строке и без «слепых» импортов на всякий случай. Параллельно вы должны вынести общие импорты в один файл `GlobalUsings.cs`, включить неявные глобальные `using` через `<ImplicitUsings>enable</ImplicitUsings>` и продемонстрировать разрешение конфликта как через локальный алиас, так и через полное имя. Это упражнение имитирует реальную ситуацию: новые разработчики часто копируют `using` из файла в файл, пока не получат ошибку неоднозначности, и не знают, что `global using` и алиасы решают проблему централизованно. Выполнив задание, вы получите muscle memory для всех шести пунктов чек-листа самопроверки урока.

#### Что нужно сделать (пошагово)
1. Создайте консольный проект .NET 8 командой `dotnet new console -n InventoryTracker -o InventoryTracker` и перейдите в его каталог (`cd InventoryTracker`). Откройте файл `InventoryTracker.csproj` и убедитесь, что там присутствуют `<ImplicitUsings>enable</ImplicitUsings>` и `<Nullable>enable</Nullable>` (шаблон .NET 8 включает их по умолчанию; если нет — добавьте вручную).
2. Внутри проекта создайте папки, повторяющие структуру пространств имён: `Models`, `Geometry`, `Rendering`, `Services`. Это не требование компилятора, а соглашение из best practices урока — имя пространства должно совпадать со структурой папок.
3. Создайте файл `GlobalUsings.cs` в корне проекта. В нём разместите проектные глобальные импорты: `global using InventoryTracker.Models;` (чтобы тип `Item` был доступен по короткому имени во всём проекте) и глобальный алиас `global using Stock = System.Collections.Generic.Dictionary<string, InventoryTracker.Models.Item>;`. Не добавляйте `global using System;`, `System.Collections.Generic` и `System.Linq` — они уже импортированы неявно через `ImplicitUsings`.
4. В `Models/Item.cs` объявите `namespace InventoryTracker.Models;` (file-scoped!) и тип `public sealed record Item(int Id, string Sku, int Quantity, decimal Cost)` со свойством `TotalValue`.
5. В `Geometry/Point.cs` объявите `namespace InventoryTracker.Geometry;` и `public readonly record struct Point(int X, int Y)` с методом `DistanceTo`.
6. В `Rendering/Point.cs` объявите `namespace InventoryTracker.Rendering;` и `public sealed class Point` с изменяемыми свойствами `X`, `Y`, `Color`. Теперь у вас два типа с именем `Point` — конфликт готов.
7. В `Services/StockService.cs` используйте локальные алиасы `using GeoPoint = InventoryTracker.Geometry.Point;` и `using RenderPoint = InventoryTracker.Rendering.Point;`, чтобы внутри сервиса обращаться к каждому типу однозначно. Реализуйте статические методы `Summarize`, `Classify` (с pattern matching) и `ToRender`.
8. В `Program.cs` (top-level statements) импортируйте `using InventoryTracker.Geometry;` и `using InventoryTracker.Services;` (а `Models` уже глобальный). Создайте экземпляр `Stock`, заполните его, выведите сводку, посчитайте расстояние от точки `(3,4)` до начала координат и преобразуйте геометрическую точку в точку рендеринга.
9. Соберите проект: `dotnet build`. Ожидаемый результат — сборка без ошибок и без предупреждений неоднозначности.
10. Запустите: `dotnet run`. Ожидаемый вывод (формат валюты зависит от локали системы): строки по каждому товару вида `A-1: 10 × 2,50 ₽ = 25,00 ₽`, строка `ИТОГО / TOTAL: 74,96 ₽`, строка `Distance to origin: 5.00` и строка `Render(3,4,red)`.
11. Временно раскомментируйте гипотетический `using InventoryTracker.Rendering;` в `Program.cs` и убедитесь, что компилятор выдаёт ошибку неоднозначности `Point` — это доказывает, что конфликт реален. Затем верните код обратно.

#### Требования к решению
- Целевая среда: C# 12 / .NET 8. Используйте top-level statements в `Program.cs`, file-scoped namespace во всех новых файлах, collection expressions (C# 12) для построения списков, pattern matching с `and` в методе `Classify` и raw string literal для баннера.
- Все типы должны находиться в пространствах имён, имена которых совпадают с путём к файлу от корня проекта: `InventoryTracker.Models`, `InventoryTracker.Geometry`, `InventoryTracker.Rendering`, `InventoryTracker.Services`. Глубина иерархии — не более 3 уровней (соглашение урока: 2–4 уровня).
- Каждый файл обязан начинаться с file-scoped объявления `namespace X;` с точкой с запятой. Классическая форма с фигурными скобками допускается только если внутри нужны вложенные пространства (здесь это не требуется).
- Проект должен собираться с включёнными `<ImplicitUsings>enable</ImplicitUsings>` и `<Nullable>enable</Nullable>`. В `GlobalUsings.cs` не должно быть дубликатов того, что уже дают неявные импорты.
- Конфликт имён `Point` должен быть разрешён двумя способами: локальным алиасом (в `StockService`) и полным именем (минимум одно использование `InventoryTracker.Rendering.Point` или `InventoryTracker.Geometry.Point` в `Program.cs`).
- Глобальный алиас `Stock` обязан использоваться как в `Program.cs`, так и в `StockService` — никаких «объявил и забыл».
- Код должен быть готов к запуску на любой ОС: не используйте `System.Drawing` или `System.Windows`, конфликт демонстрируется на собственных типах.

#### Тонкости и подводные камни
- Самая частая ошибка — забыть точку с запятой в file-scoped namespace: `namespace InventoryTracker.Models` без `;` даёт ошибку компиляции. Всегда завершайте объявление символом `;`.
- `global using` можно размещать только на верхнем уровне файла, до любого `namespace`. Попытка написать `global using` внутри namespace-блока или после file-scoped объявления приведёт к ошибке. Обычные (не глобальные) `using` и алиасы, напротив, можно ставить как до, так и после file-scoped `namespace` — они применяются в пределах файла.
- Алиас не создаёт новый тип, а лишь даёт второе имя существующему. `using GeoPoint = InventoryTracker.Geometry.Point;` не объявляет новый класс; `GeoPoint` и `InventoryTracker.Geometry.Point` — один и тот же тип. Это важно понимать, чтобы не пытаться через алиас «создать» совместимый тип.
- Не импортируйте пространства «на всякий случай». Лишние `using` не только засоряют файл, но и могут внезапно породить неоднозначность, когда в проект добавится новая зависимость с одноимённым типом. Импортируйте только то, что реально используется.
- Следите за глубиной иерархии. Имена вроде `Company.Division.Team.Project.Subsystem.Module` мешают чтению и удлиняют полные имена. Держите 2–4 уровня — в нашем проекте максимум `InventoryTracker.Rendering` (два сегмента после корня).
- Формат валюты (`:C`) и разделители зависят от локали ОС; в выводе могут быть `₽`, `$` или `€`, запятая или точка. Это не ошибка — не пытайтесь «жёстко» зашить символ.
- Если в `Program.cs` импортировать и `Geometry`, и `Rendering` одновременно, любое упоминание `Point` становится неоднозначным. Решение — убрать лишний `using`, использовать алиас или полное имя. Не пытайтесь «обойти» конфликт, переименовывая тип — это маскирует проблему.

#### Критерии приёмки
- [ ] Проект `InventoryTracker` создан на .NET 8 и собирается без ошибок и предупреждений через `dotnet build`.
- [ ] В `.csproj` включены `<ImplicitUsings>enable</ImplicitUsings>` и `<Nullable>enable</Nullable>`.
- [ ] Все четыре файла типов используют file-scoped `namespace X;` с точкой с запятой.
- [ ] Имена пространств совпадают со структурой папок (`Models/`, `Geometry/`, `Rendering/`, `Services/`).
- [ ] Файл `GlobalUsings.cs` существует и содержит `global using InventoryTracker.Models;` и глобальный алиас `Stock`.
- [ ] В `GlobalUsings.cs` нет дубликатов неявных импортов (`System`, `System.Collections.Generic`, `System.Linq`).
- [ ] Присутствуют два разных типа `Point` — в `Geometry` (record struct) и в `Rendering` (class).
- [ ] В `StockService` конфликт разрешён локальными алиасами `GeoPoint` и `RenderPoint`.
- [ ] В `Program.cs` хотя бы один раз использовано полное имя `Point` (например, `InventoryTracker.Geometry.Point`).
- [ ] Метод `Classify` использует pattern matching с `and`.
- [ ] В `Program.cs` применена collection expression (C# 12) и raw string literal.
- [ ] Глобальный алиас `Stock` используется и в `Program.cs`, и в `StockService`.
- [ ] `dotnet run` выводит сводку по товарам, итоговую сумму, расстояние `5.00` и строку `Render(3,4,red)`.
- [ ] Временное добавление `using InventoryTracker.Rendering;` в `Program.cs` действительно вызывает ошибку неоднозначности (проверено).
- [ ] Код запускается на любой ОС без зависимостей от `System.Drawing` / `System.Windows`.

#### Подсказки (без прямого ответа)
- Вспомните, что `global using` с пространством и `global using` с алиасом — это две разные формы одной директивы; обе допустимы в `GlobalUsings.cs`.
- Для target-typed `new` с record struct используйте `new(0, 0)`, а для класса с инициализатором свойств — `new() { X = ..., Color = ... }`.
- Чтобы доказать конфликт, не нужно менять рабочую версию файла — достаточно временно добавить лишний `using` и скомпилировать.
- Collection expression со spread `[.. source]` работает с любым `IEnumerable<T>` и создаёт `List<T>`.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — InventoryTracker
// Эталонное решение / Reference solution

// Файл: GlobalUsings.cs
// Единое место для проектных глобальных импортов / Single place for project-wide imports
// System, System.Collections.Generic, System.Linq и др. уже добавлены неявно через <ImplicitUsings>
// System, System.Collections.Generic, System.Linq, etc. are already added implicitly via <ImplicitUsings>
global using InventoryTracker.Models;
global using Stock = System.Collections.Generic.Dictionary<string, InventoryTracker.Models.Item>;

// Файл: Models/Item.cs
namespace InventoryTracker.Models; // file-scoped namespace — весь файл относится к Models

public sealed record Item(int Id, string Sku, int Quantity, decimal Cost)
{
    public decimal TotalValue => Quantity * Cost;
}

// Файл: Geometry/Point.cs
namespace InventoryTracker.Geometry;

public readonly record struct Point(int X, int Y)
{
    public double DistanceTo(Point other)
    {
        var dx = X - other.X;
        var dy = Y - other.Y;
        return Math.Sqrt(dx * dx + dy * dy); // Math доступен из неявного using System
    }
}

// Файл: Rendering/Point.cs — второй тип с именем Point! / second type named Point!
namespace InventoryTracker.Rendering;

public sealed class Point
{
    public int X { get; set; }
    public int Y { get; set; }
    public string Color { get; set; } = "black";
    public override string ToString() => $"Render({X},{Y},{Color})";
}

// Файл: Services/StockService.cs
using GeoPoint = InventoryTracker.Geometry.Point;   // локальный алиас / local alias
using RenderPoint = InventoryTracker.Rendering.Point;

namespace InventoryTracker.Services;

public static class StockService
{
    public static string Summarize(Stock stock)
    {
        // Коллекционное выражение (C# 12) со spread / Collection expression (C# 12) with spread
        List<string> lines =
        [
            .. stock.Values.Select(i => $"{i.Sku}: {i.Quantity} × {i.Cost:C} = {i.TotalValue:C}"),
        ];
        var total = stock.Values.Sum(i => i.TotalValue);
        lines.Add($"ИТОГО / TOTAL: {total:C}");
        return string.Join(Environment.NewLine, lines);
    }

    // Pattern matching со свойствами и оператором and / Property pattern matching with and
    public static string Classify(Item i) => i switch
    {
        { Quantity: 0 } => "нет на складе / out of stock",
        { Quantity: > 0 and <= 5 } => "мало / low",
        { Quantity: > 5 and <= 50 } => "норма / normal",
        _ => "много / plenty"
    };

    public static RenderPoint ToRender(GeoPoint geo, string color) =>
        new() { X = geo.X, Y = geo.Y, Color = color };
}

// Файл: Program.cs — top-level statements
using InventoryTracker.Geometry;
using InventoryTracker.Services;
// using InventoryTracker.Rendering; — НЕ импортируем: это вызвало бы конфликт Point
// NOT imported: it would cause a Point ambiguity

var banner = """
    === Склад / Warehouse ===
    """; // raw string literal (C# 11+)
Console.WriteLine(banner);

var stock = new Stock
{
    ["A-1"] = new Item(1, "A-1", 10, 2.5m),
    ["A-2"] = new Item(2, "A-2", 4, 9.99m),
    ["A-3"] = new Item(3, "A-3", 100, 0.10m),
};

Console.WriteLine(StockService.Summarize(stock));
foreach (var i in stock.Values)
    Console.WriteLine($"  {i.Sku}: {StockService.Classify(i)}");

var geo = new Point(3, 4); // Point = Geometry.Point, т.к. Rendering не импортирован
Console.WriteLine($"Distance to origin: {geo.DistanceTo(new(0, 0)):F2}"); // 5.00

// Полное имя разрешает конкретный Point без using / Fully qualified name picks the right Point
var render = StockService.ToRender(geo, "red");
Console.WriteLine(render); // Render(3,4,red)
```

Разбор по строкам. `GlobalUsings.cs` концентрирует проектные импорты в одном обозреваемом месте — это best practice урока. Мы намеренно не дублируем `System` и `System.Linq`, потому что `<ImplicitUsings>enable</ImplicitUsings>` уже добавил их: повторное объявление было бы нарушением правила «не импортируй на всякий случай». Глобальный алиас `Stock` даёт короткое имя громоздкому `Dictionary<string, Item>` сразу для всего проекта — ровно тот приём, что в уроке показан как `global using Lines = ...`. Каждый файл типа открывается file-scoped `namespace X;` с точкой с запятой — это устраняет лишний отступ и соответствует рекомендации урока для новых файлов. Имена пространств (`InventoryTracker.Models`, `InventoryTracker.Geometry` и т. д.) совпадают с путями файлов — соглашение, снижающее путаницу при поиске типов. Два типа `Point` в разных пространствах воспроизводят классический конфликт из урока (там `System.Drawing.Point` против `System.Windows.Point`), но на собственных типах, чтобы код跑ал на любой ОС. В `StockService` конфликт решён локальными алиасами `GeoPoint` и `RenderPoint`: важно, что алиасы не создают новые типы, а лишь дают вторые имена — это ключевое замечание урока. В `Program.cs` мы импортируем только `Geometry` и `Services`, поэтому короткое имя `Point` однозначно разрешается в `Geometry.Point`; к `Rendering.Point` обращаемся косвенно через `StockService.ToRender`, который внутри использует алиас. Collection expression `[.. source]` и pattern matching с `and` в `Classify` демонстрируют современные возможности C# 12, а raw string literal упрощает многострочный баннер без экранирования. Если временно вернуть `using InventoryTracker.Rendering;`, компилятор немедленно сообщит об неоднозначности `Point` — практическое подтверждение, что конфликт реален и что описанные в уроке инструменты (`global using`, алиас, полное имя) работают именно так, как заявлено.

#### Задания на углубление (бонус)
1. Добавьте третье пространство `InventoryTracker.Mapping` со своим типом `Point` (например, геокоординаты широты/долготы) и разрешите тройной конфликт одновременно через алиасы и полные имена в одном файле.
2. Переведите проект на `global using` для `Geometry` и `Services` вместо локальных `using` в `Program.cs`. Объясните в комментарии, почему в этом случае локальный `using` всё же безопаснее для `Rendering`.
3. Создайте вложенное пространство `InventoryTracker.Services.Reporting` (классическая форма с фигурными скобками) и разместите там тип `Report`. Покажите, что file-scoped и классическая формы могут сосуществовать в одном проекте.
4. Измерьте, как `global using` влияет на читаемость: перепишите `StockService` без алиасов, используя только полные имена, и сравните длину строк. Опишите вывод в комментарии.

---

## Statement in English / Постановка на английском

#### Context & motivation
You are a backend developer at a small company that tracks warehouse inventory. The team has just started a new .NET 8 microservice called `InventoryTracker`, and the architect asked you to establish a correct namespace structure "from day one", so that as the project grows types stay easy to locate and names never collide. Two independent subsystems are already planned, and both need coordinates: a geometry core (distance calculations, immutable points) and a rendering subsystem (on-screen drawing, mutable points carrying a color). Naturally, both teams called their type `Point` — the textbook name conflict described in the lesson. Your job is to design the project so that both `Point` types coexist peacefully while the code stays readable: no long fully qualified names on every line, and no "blind" imports added just in case. In parallel you must move shared imports into a single `GlobalUsings.cs` file, enable implicit global `using` through `<ImplicitUsings>enable</ImplicitUsings>`, and demonstrate conflict resolution both with a local alias and with a fully qualified name. This exercise simulates a real situation: junior developers often copy `using` directives from file to file until they hit an ambiguity error, and they do not know that `global using` and aliases solve the problem centrally. Completing the task gives you muscle memory for every one of the six items in the lesson's self-check checklist.

#### What to do step by step
1. Create a .NET 8 console project with `dotnet new console -n InventoryTracker -o InventoryTracker` and enter its directory (`cd InventoryTracker`). Open `InventoryTracker.csproj` and confirm that `<ImplicitUsings>enable</ImplicitUsings>` and `<Nullable>enable</Nullable>` are present (the .NET 8 template enables them by default; add them manually if they are missing).
2. Inside the project, create folders that mirror the namespace structure: `Models`, `Geometry`, `Rendering`, `Services`. The compiler does not require this, but it is the lesson's best practice — namespace names should match the folder layout.
3. Create a `GlobalUsings.cs` file at the project root. Put the project-wide global imports there: `global using InventoryTracker.Models;` (so the `Item` type is available by its short name across the whole project) and a global alias `global using Stock = System.Collections.Generic.Dictionary<string, InventoryTracker.Models.Item>;`. Do not add `global using System;`, `System.Collections.Generic`, or `System.Linq` — they are already imported implicitly through `ImplicitUsings`.
4. In `Models/Item.cs` declare `namespace InventoryTracker.Models;` (file-scoped!) and the type `public sealed record Item(int Id, string Sku, int Quantity, decimal Cost)` with a `TotalValue` property.
5. In `Geometry/Point.cs` declare `namespace InventoryTracker.Geometry;` and `public readonly record struct Point(int X, int Y)` with a `DistanceTo` method.
6. In `Rendering/Point.cs` declare `namespace InventoryTracker.Rendering;` and `public sealed class Point` with mutable `X`, `Y`, `Color` properties. You now have two types named `Point` — the conflict is ready.
7. In `Services/StockService.cs` use local aliases `using GeoPoint = InventoryTracker.Geometry.Point;` and `using RenderPoint = InventoryTracker.Rendering.Point;` so each type can be referenced unambiguously inside the service. Implement static methods `Summarize`, `Classify` (with pattern matching), and `ToRender`.
8. In `Program.cs` (top-level statements) import `using InventoryTracker.Geometry;` and `using InventoryTracker.Services;` (`Models` is already global). Create a `Stock` instance, fill it, print the summary, compute the distance from point `(3,4)` to the origin, and convert the geometry point to a render point.
9. Build the project: `dotnet build`. The expected result is a clean build with no errors and no ambiguity warnings.
10. Run it: `dotnet run`. Expected output (currency format depends on the system locale): one line per item like `A-1: 10 × 2.50 $ = 25.00 $`, a line `ИТОГО / TOTAL: 74.96 $`, a line `Distance to origin: 5.00`, and a line `Render(3,4,red)`.
11. Temporarily uncomment a hypothetical `using InventoryTracker.Rendering;` in `Program.cs` and confirm the compiler raises a `Point` ambiguity error — proof that the conflict is real. Then revert the code.

#### Requirements
- Target environment: C# 12 / .NET 8. Use top-level statements in `Program.cs`, file-scoped namespaces in all new files, collection expressions (C# 12) to build lists, pattern matching with `and` in `Classify`, and a raw string literal for the banner.
- Every type must live in a namespace whose name matches the file path from the project root: `InventoryTracker.Models`, `InventoryTracker.Geometry`, `InventoryTracker.Rendering`, `InventoryTracker.Services`. Hierarchy depth must not exceed 3 levels (the lesson's convention is 2–4 levels).
- Every file must start with a file-scoped `namespace X;` declaration terminated with a semicolon. The classic brace form is allowed only when nested namespaces are needed inside a file (not the case here).
- The project must build with `<ImplicitUsings>enable</ImplicitUsings>` and `<Nullable>enable</Nullable>` enabled. `GlobalUsings.cs` must not duplicate anything the implicit imports already provide.
- The `Point` name conflict must be resolved in two ways: with a local alias (in `StockService`) and with a fully qualified name (at least one use of `InventoryTracker.Rendering.Point` or `InventoryTracker.Geometry.Point` in `Program.cs`).
- The global alias `Stock` must be used both in `Program.cs` and in `StockService` — no "declared and forgotten" globals.
- The code must run on any OS: do not use `System.Drawing` or `System.Windows`; the conflict is demonstrated on your own types.

#### Pitfalls
- The most common mistake is forgetting the semicolon in a file-scoped namespace: `namespace InventoryTracker.Models` without `;` is a compile error. Always terminate the declaration with `;`.
- `global using` may only appear at the top level of a file, before any `namespace`. Putting `global using` inside a namespace block or after a file-scoped declaration is an error. Regular (non-global) `using` directives and aliases, by contrast, may be placed either before or after a file-scoped `namespace` — they apply within the file.
- An alias does not create a new type; it only gives an existing type a second name. `using GeoPoint = InventoryTracker.Geometry.Point;` does not declare a new class — `GeoPoint` and `InventoryTracker.Geometry.Point` are the same type. Understanding this prevents you from trying to "create" a compatible type through an alias.
- Do not import namespaces "just in case". Unused `using` directives not only clutter the file but can suddenly trigger ambiguity when a new dependency exposing a same-named type is added. Import only what you actually use.
- Watch the hierarchy depth. Names like `Company.Division.Team.Project.Subsystem.Module` hurt readability and bloat fully qualified names. Keep 2–4 levels — our project tops out at `InventoryTracker.Rendering` (two segments after the root).
- The currency format (`:C`) and separators depend on the OS locale; the output may show `₽`, `$`, or `€`, with a comma or a dot. This is not an error — do not hard-code the symbol.
- If you import both `Geometry` and `Rendering` in `Program.cs` at once, any mention of `Point` becomes ambiguous. The fix is to drop the extra `using`, use an alias, or use the fully qualified name. Do not "work around" the conflict by renaming the type — that hides the problem.

#### Acceptance criteria
- [ ] The `InventoryTracker` project targets .NET 8 and builds cleanly with no errors or warnings via `dotnet build`.
- [ ] `<ImplicitUsings>enable</ImplicitUsings>` and `<Nullable>enable</Nullable>` are present in `.csproj`.
- [ ] All four type files use a file-scoped `namespace X;` terminated with a semicolon.
- [ ] Namespace names match the folder structure (`Models/`, `Geometry/`, `Rendering/`, `Services/`).
- [ ] `GlobalUsings.cs` exists and contains `global using InventoryTracker.Models;` and the global alias `Stock`.
- [ ] `GlobalUsings.cs` does not duplicate implicit imports (`System`, `System.Collections.Generic`, `System.Linq`).
- [ ] Two distinct `Point` types exist — in `Geometry` (record struct) and in `Rendering` (class).
- [ ] `StockService` resolves the conflict with local aliases `GeoPoint` and `RenderPoint`.
- [ ] `Program.cs` uses a fully qualified `Point` name at least once (e.g. `InventoryTracker.Geometry.Point`).
- [ ] The `Classify` method uses pattern matching with `and`.
- [ ] `Program.cs` uses a collection expression (C# 12) and a raw string literal.
- [ ] The global alias `Stock` is used in both `Program.cs` and `StockService`.
- [ ] `dotnet run` prints the per-item summary, the total, the distance `5.00`, and the line `Render(3,4,red)`.
- [ ] Temporarily adding `using InventoryTracker.Rendering;` to `Program.cs` does trigger an ambiguity error (verified).
- [ ] The code runs on any OS with no `System.Drawing` / `System.Windows` dependency.

#### Hints (no direct answer)
- Recall that `global using` with a namespace and `global using` with an alias are two forms of the same directive; both are valid in `GlobalUsings.cs`.
- For target-typed `new` with a record struct use `new(0, 0)`, and for a class with property initializers use `new() { X = ..., Color = ... }`.
- To prove the conflict you do not need to change the working file — just temporarily add the extra `using` and compile.
- The collection expression with spread `[.. source]` works with any `IEnumerable<T>` and produces a `List<T>`.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — InventoryTracker
// Reference solution

// File: GlobalUsings.cs
// Single place for project-wide imports
// System, System.Collections.Generic, System.Linq, etc. are already added implicitly via <ImplicitUsings>
global using InventoryTracker.Models;
global using Stock = System.Collections.Generic.Dictionary<string, InventoryTracker.Models.Item>;

// File: Models/Item.cs
namespace InventoryTracker.Models; // file-scoped namespace — the whole file belongs to Models

public sealed record Item(int Id, string Sku, int Quantity, decimal Cost)
{
    public decimal TotalValue => Quantity * Cost;
}

// File: Geometry/Point.cs
namespace InventoryTracker.Geometry;

public readonly record struct Point(int X, int Y)
{
    public double DistanceTo(Point other)
    {
        var dx = X - other.X;
        var dy = Y - other.Y;
        return Math.Sqrt(dx * dx + dy * dy); // Math is available via implicit using System
    }
}

// File: Rendering/Point.cs — a second type named Point!
namespace InventoryTracker.Rendering;

public sealed class Point
{
    public int X { get; set; }
    public int Y { get; set; }
    public string Color { get; set; } = "black";
    public override string ToString() => $"Render({X},{Y},{Color})";
}

// File: Services/StockService.cs
using GeoPoint = InventoryTracker.Geometry.Point;   // local alias
using RenderPoint = InventoryTracker.Rendering.Point;

namespace InventoryTracker.Services;

public static class StockService
{
    public static string Summarize(Stock stock)
    {
        // Collection expression (C# 12) with spread
        List<string> lines =
        [
            .. stock.Values.Select(i => $"{i.Sku}: {i.Quantity} × {i.Cost:C} = {i.TotalValue:C}"),
        ];
        var total = stock.Values.Sum(i => i.TotalValue);
        lines.Add($"TOTAL / ИТОГО: {total:C}");
        return string.Join(Environment.NewLine, lines);
    }

    // Property pattern matching with and
    public static string Classify(Item i) => i switch
    {
        { Quantity: 0 } => "out of stock",
        { Quantity: > 0 and <= 5 } => "low",
        { Quantity: > 5 and <= 50 } => "normal",
        _ => "plenty"
    };

    public static RenderPoint ToRender(GeoPoint geo, string color) =>
        new() { X = geo.X, Y = geo.Y, Color = color };
}

// File: Program.cs — top-level statements
using InventoryTracker.Geometry;
using InventoryTracker.Services;
// using InventoryTracker.Rendering; — NOT imported: it would cause a Point ambiguity

var banner = """
    === Warehouse / Склад ===
    """; // raw string literal (C# 11+)
Console.WriteLine(banner);

var stock = new Stock
{
    ["A-1"] = new Item(1, "A-1", 10, 2.5m),
    ["A-2"] = new Item(2, "A-2", 4, 9.99m),
    ["A-3"] = new Item(3, "A-3", 100, 0.10m),
};

Console.WriteLine(StockService.Summarize(stock));
foreach (var i in stock.Values)
    Console.WriteLine($"  {i.Sku}: {StockService.Classify(i)}");

var geo = new Point(3, 4); // Point = Geometry.Point, since Rendering is not imported
Console.WriteLine($"Distance to origin: {geo.DistanceTo(new(0, 0)):F2}"); // 5.00

// Fully qualified name resolves the right Point without a using directive
var render = StockService.ToRender(geo, "red");
Console.WriteLine(render); // Render(3,4,red)
```

Line-by-line walk-through. `GlobalUsings.cs` concentrates the project-wide imports in one reviewable place — the lesson's best practice. We deliberately do not duplicate `System` and `System.Linq`, because `<ImplicitUsings>enable</ImplicitUsings>` already added them; re-declaring them would violate the "do not import just in case" rule. The global alias `Stock` gives a short name to the verbose `Dictionary<string, Item>` for the entire project — exactly the technique the lesson shows as `global using Lines = ...`. Each type file opens with a file-scoped `namespace X;` terminated by a semicolon, which removes an extra indentation level and matches the lesson's recommendation for new files. The namespace names (`InventoryTracker.Models`, `InventoryTracker.Geometry`, etc.) match the file paths — a convention that reduces confusion when locating types. The two `Point` types in different namespaces reproduce the classic conflict from the lesson (there `System.Drawing.Point` vs `System.Windows.Point`), but on your own types so the code runs on any OS. In `StockService` the conflict is resolved with local aliases `GeoPoint` and `RenderPoint`: crucially, aliases do not create new types, they only add second names — a key point of the lesson. In `Program.cs` we import only `Geometry` and `Services`, so the short name `Point` resolves unambiguously to `Geometry.Point`; we reach `Rendering.Point` indirectly through `StockService.ToRender`, which uses the alias internally. The collection expression `[.. source]` and the pattern matching with `and` in `Classify` showcase modern C# 12 features, while the raw string literal simplifies a multiline banner without escaping. If you temporarily restore `using InventoryTracker.Rendering;`, the compiler immediately reports a `Point` ambiguity — practical proof that the conflict is real and that the tools described in the lesson (`global using`, aliases, fully qualified names) behave exactly as advertised.

#### Going deeper (bonus)
1. Add a third namespace `InventoryTracker.Mapping` with its own `Point` type (say, latitude/longitude geo-coordinates) and resolve a three-way conflict at once using aliases and fully qualified names in a single file.
2. Switch the project to `global using` for `Geometry` and `Services` instead of local `using` directives in `Program.cs`. Explain in a comment why a local `using` is still safer for `Rendering`.
3. Create a nested namespace `InventoryTracker.Services.Reporting` (classic brace form) and place a `Report` type there. Show that file-scoped and classic forms can coexist in the same project.
4. Measure how `global using` affects readability: rewrite `StockService` without aliases, using only fully qualified names, and compare line length. Describe the conclusion in a comment.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект собирается командой `dotnet build` без ошибок и предупреждений.
- [ ] Команда `dotnet run` выдаёт ожидаемый вывод (сводка, итог, `5.00`, `Render(3,4,red)`).
- [ ] Все файлы используют file-scoped namespace с точкой с запятой.
- [ ] `GlobalUsings.cs` содержит глобальный импорт и алиас, без дублей неявных импортов.
- [ ] Конфликт `Point` разрешён и алиасом, и полным именем.
- [ ] В коде есть collection expression, pattern matching и raw string literal.
- [ ] The project builds with `dotnet build` with no errors or warnings.
- [ ] `dotnet run` produces the expected output (summary, total, `5.00`, `Render(3,4,red)`).
- [ ] All files use a file-scoped namespace with a semicolon.
- [ ] `GlobalUsings.cs` contains a global import and an alias, without duplicating implicit imports.
- [ ] The `Point` conflict is resolved both with an alias and with a fully qualified name.
- [ ] The code contains a collection expression, pattern matching, and a raw string literal.

#### Ресурсы / Resources
- [Microsoft Learn — Namespaces](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/namespaces)
- [C# 10 features (file-scoped namespace, global using)](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-10)
- [using directive (C# Reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/using-directive)
- [What's new in C# 12 (collection expressions)](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-12)
- [Raw string literals (C# 11 feature)](https://learn.microsoft.com/dotnet/csharp/language-reference/tokens/raw-string)
