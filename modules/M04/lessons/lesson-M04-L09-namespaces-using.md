[⬆ К модулю M04](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M04-L09: Пространства имён, using, namespace / Namespaces, using, namespace

**Модуль / Module:** M04
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

**Пространство имён (namespace)** — это логический контейнер для типов. Представьте себе файловую систему: файл `report.pdf` может лежать в папке `Finance/2024`, а другой файл с тем же именем — в `Marketing/2024`. Папки не меняют содержимое файлов, но дают уникальный адрес. В C# роль «папок» играют пространства имён, а «файлы» — это классы, структуры, интерфейсы, перечисления и делегаты. Благодаря этому в программе могут мирно сосуществовать два класса с одинаковым именем, если они находятся в разных пространствах.

Объявление классического пространства имён выглядит так: `namespace MyApp.Models { ... }`, и всё, что стоит внутри фигурных скобок, принадлежит этому пространству. Внутри можно вкладывать другие пространства через точку: `namespace MyApp.Models.Orders` — это то же самое, что вложенное объявление. Имя пространства традиционно повторяет структуру папок проекта (`MyApp/Models/Orders`), но компилятор этого не требует — это соглашение, а не правило.

Чтобы не писать полное имя каждый раз (`System.Console.WriteLine`), существует директива **`using`**. Она «импортирует» пространство в текущий файл, и тогда достаточно короткого имени (`Console.WriteLine`). Это похоже на то, как в телефонной книге вы один раз сохраняете контакт «Мама», а потом набираете без ввода полного номера.

Начиная с **C# 10** появилось два удобных нововведения. Первое — **file-scoped namespace** (пространство имён на уровне файла). Запись `namespace MyApp.Models;` с точкой с запятой в конце означает, что весь код файла относится к этому пространству, без лишнего уровня отступов. Это особенно приятно в небольших файлах с одним классом. Второе — **`global using`**. Если написать `global using System.Linq;` в любом файле проекта, это `using` автоматически применяется ко всем файлам. В .NET 6+ большинство шаблонов проектов используют «неявные глобальные using» (`<ImplicitUsings>enable</ImplicitUsings>`), которые добавляют базовые пространства (`System`, `System.Collections.Generic`, `System.Linq` и др.) без явных директив.

Иногда два пространства содержат типы с одинаковым именем. Например, и в `System`, и в `System.Drawing`, и в `System.Windows.Shapes` есть класс `Point`. Если импортировать их одновременно, компилятор выдаст ошибку неоднозначности. Решений несколько: указать полное имя явно, либо убрать лишний `using`, либо применить **алиас** (alias). Синтаксис алиаса: `using Point = System.Drawing.Point;` — после этого слово `Point` в этом файле будет означать именно `System.Drawing.Point`. Алиасы также удобны для сокращения длинных имён: `using Lines = System.Collections.Generic.List<string>;`. Заметьте: алиас создаёт не новый тип, а лишь альтернативное имя для существующего.

Глобальные алиасы тоже возможны: `global using Lines = System.Collections.Generic.List<string>;` — и это короткое имя доступно во всём проекте. Главное правило хорошего тона: располагайте `global using` в одном специальном файле (часто `GlobalUsings.cs`), чтобы не размазывать их по проекту.

#### Theory (EN)

A **namespace** is a logical container for types. Think of a filesystem: a file named `report.pdf` can live in a folder `Finance/2024`, while another file with the exact same name lives in `Marketing/2024`. The folders do not change the file contents; they give each file a unique address. In C#, namespaces play the role of those folders, while classes, structs, interfaces, enums, and delegates play the role of files. Because of this, two classes with the same simple name can peacefully coexist in one program as long as they live in different namespaces.

A classic namespace declaration looks like `namespace MyApp.Models { ... }`, and everything inside the braces belongs to that namespace. You can nest namespaces with a dot: `namespace MyApp.Models.Orders` is equivalent to a nested declaration. By convention the namespace mirrors the project folder structure (`MyApp/Models/Orders`), but the compiler does not enforce this — it is an agreement, not a rule.

To avoid writing the fully qualified name every time (`System.Console.WriteLine`), C# offers the **`using`** directive. It imports a namespace into the current file so the short name is enough (`Console.WriteLine`). It is like saving a contact called “Mom” in your phone once, so you never have to dial the full number again.

Starting with **C# 10**, two convenient features arrived. The first is the **file-scoped namespace**: `namespace MyApp.Models;` (note the trailing semicolon) means the whole file belongs to that namespace, with no extra indentation level — a relief for small single-class files. The second is **`global using`**. Writing `global using System.Linq;` in any file makes that `using` apply to every file in the project. In .NET 6+ most project templates enable “implicit global usings” (`<ImplicitUsings>enable</ImplicitUsings>`), which silently add common namespaces such as `System`, `System.Collections.Generic`, `System.Linq`, and others without explicit directives.

Sometimes two namespaces expose types with the same name. For instance, a class `Point` exists in `System`, in `System.Drawing`, and in `System.Windows.Shapes`. Importing them together produces an ambiguity error. The fixes are: use the fully qualified name explicitly, remove the offending `using`, or introduce an **alias**. The alias syntax is `using Point = System.Drawing.Point;` — after that, the word `Point` in this file refers specifically to `System.Drawing.Point`. Aliases are also handy to shorten long names: `using Lines = System.Collections.Generic.List<string>;`. Note that an alias does not create a new type; it merely gives an existing type an alternative name.

Global aliases work too: `global using Lines = System.Collections.Generic.List<string>;` makes that short name available across the whole project. A good practice is to keep all `global using` declarations in one dedicated file (often `GlobalUsings.cs`) rather than scattering them across the project, so the list of project-wide imports stays easy to review.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8
// Демонстрация: namespace, using, file-scoped namespace, global using, alias, разрешение конфликтов
// Demo: namespace, using, file-scoped namespace, global using, alias, conflict resolution

// Файл GlobalUsings.cs (глобальные импорты для всего проекта)
// File GlobalUsings.cs (global imports for the whole project)
global using System;
global using System.Collections.Generic;
// Алиас, доступный во всём проекте / alias available project-wide
global using Lines = System.Collections.Generic.List<string>;

namespace ShopDemo; // file-scoped namespace (C# 10+) — весь файл относится к ShopDemo
                    // file-scoped namespace — the whole file belongs to ShopDemo

// Короткое имя, разрешённое через global using / short name resolved via global using
// using System; здесь писать уже не нужно — он глобальный / no need to write "using System" — it is global

public record Product(int Id, string Name, decimal Price);

public class Cart
{
    private readonly Lines _items = new(); // Lines — это глобальный алиас для List<string>
                                           // Lines is a global alias for List<string>

    public void Add(string sku) => _items.Add(sku);

    public int Count => _items.Count;
}

// --- Разрешение конфликта имён / Resolving a name conflict ---
// Допустим, нам нужен System.Drawing.Point, а не System.Windows.Point.
// Suppose we need System.Drawing.Point, not System.Windows.Point.

namespace ShopDemo.Geometry
{
    // Локальный алиас только для этого файла / file-local alias
    using Point = System.Drawing.Point;       // теперь "Point" = System.Drawing.Point
    using WinPoint = System.Windows.Point;    // второе имя для второго типа / second name for the second type

    public sealed class Canvas
    {
        public Point Origin { get; } = new(0, 0);          // System.Drawing.Point
        public WinPoint Corner { get; } = new(0, 0);       // System.Windows.Point
        // Полное имя тоже работает / fully qualified name also works
        public System.Drawing.Point Center { get; } = new(50, 50);
    }
}

// --- Использование / Usage ---
public static class Program
{
    public static void Run()
    {
        var cart = new Cart();
        cart.Add("SKU-001");
        cart.Add("SKU-002");
        Console.WriteLine($"Count = {cart.Count}");        // Count = 2

        var p = new Product(1, "Mug", 9.99m);
        Console.WriteLine($"{p.Name}: {p.Price:C}");       // Mug: 9,99 ₽ (локаль зависит от системы)

        var canvas = new ShopDemo.Geometry.Canvas();
        Console.WriteLine(canvas.Origin);                  // Point [ X=0, Y=0 ]
    }
}
```

#### Best Practices

- Держите имя пространства имён в синхронизации со структурой папок проекта — это снижает путаницу при поиске типов.
- В новых файлах (C# 10+) предпочитайте file-scoped namespace (`namespace X;`) классическому — меньше отступов и чище код.
- Вынесите все `global using` в один файл `GlobalUsings.cs`, чтобы список проектных импортов был в одном месте и легко обозревался.
- Используйте алиасы осознанно: только для разрешения конфликтов имён или для значительного сокращения громоздких дублированных имён (`Dictionary<string, List<Order>>`).
- Не импортируйте «на всякий случай» десятки пространств — лишние `using` усложняют чтение и могут породить неоднозначности.

- Keep namespace names in sync with the project folder structure — it reduces confusion when locating types.
- In new files (C# 10+), prefer the file-scoped namespace (`namespace X;`) over the classic form — fewer indentation levels, cleaner code.
- Put all `global using` declarations in a single `GlobalUsings.cs` file so the list of project-wide imports lives in one reviewable place.
- Use aliases deliberately: only to resolve name conflicts or to substantially shorten verbose repeated names (`Dictionary<string, List<Order>>`).
- Do not import “just in case” — unused `using` clutter the file and may introduce ambiguity.

#### Частые ошибки / Common Mistakes

- [Импорт двух пространств с одинаковым именем типа (например, `System.Drawing.Point` и `System.Windows.Point`)] → [Используйте алиас или пишите полное имя там, где нужен конкретный тип.]
- [Попытка объявить `global using` внутри пространства имён или после него] → [Размещайте `global using` только на верхнем уровне файла, до объявления `namespace`.]
- [Забытая точка с запятой в file-scoped namespace: `namespace MyApp` без `;`] → [Компилятор выдаст ошибку; всегда завершайте file-scoped объявление символом `;`.]
- [Слишком «глубокие» имена вроде `Company.Division.Team.Project.Subsystem.Module`] → [Держите иерархию в 2–4 уровня; излишняя вложенность мешает чтению и импорту.]
- [Дублирование `using` в каждом файле, хотя они нужны во всём проекте] → [Перенесите общие импорты в `global using`, включите `<ImplicitUsings>enable</ImplicitUsings>` в `.csproj`.]

- [Importing two namespaces that expose a type with the same name (e.g. `System.Drawing.Point` and `System.Windows.Point`)] → [Use an alias or write the fully qualified name where the specific type is needed.]
- [Trying to declare a `global using` inside or after a namespace block] → [Place `global using` only at the top level of a file, before any `namespace` declaration.]
- [Missing the semicolon in a file-scoped namespace: `namespace MyApp` without `;`] → [The compiler will error; always end a file-scoped declaration with `;`.]
- [Overly deep names like `Company.Division.Team.Project.Subsystem.Module`] → [Keep the hierarchy to 2–4 levels; excessive nesting hurts readability and imports.]
- [Duplicating the same `using` in every file when they are project-wide] → [Move shared imports to `global using`, enable `<ImplicitUsings>enable</ImplicitUsings>` in the `.csproj`.]

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я могу объяснить разницу между полным и коротким именем типа.
- [ ] Я использую file-scoped namespace (`namespace X;`) в новых файлах.
- [ ] Я вынес общие импорты в `global using` / `GlobalUsings.cs`.
- [ ] Я умею разрешать конфликт имён через алиас (`using X = ...`).
- [ ] Я знаю, что алиас не создаёт новый тип, а лишь даёт второе имя.
- [ ] Имя моего пространства имён соответствует структуре папок проекта.

- [ ] I can explain the difference between a fully qualified and a short type name.
- [ ] I use a file-scoped namespace (`namespace X;`) in new files.
- [ ] I moved shared imports to `global using` / `GlobalUsings.cs`.
- [ ] I can resolve a name conflict with an alias (`using X = ...`).
- [ ] I know that an alias does not create a new type, it only adds a second name.
- [ ] My namespace name matches the project folder structure.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/fundamentals/types/namespaces](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/namespaces)
- [C# 10 features (file-scoped namespace, global using) — https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-10](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-10)
- [using directive (C# Reference) — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/using-directive](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/using-directive)

---

[⬆ К модулю M04](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
