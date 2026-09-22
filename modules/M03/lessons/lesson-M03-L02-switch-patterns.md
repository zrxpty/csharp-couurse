---
[← Предыдущий: M03-L01](lesson-M03-L01-if-ternary.md) | [⬆ К модулю M03](../README.md) | [Следующий: M03-L03 →](lesson-M03-L03-loops.md)
---

### Урок M03-L02: switch, switch expressions, pattern matching / switch, switch expressions, pattern matching

**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Оператор `switch` — это инструмент ветвления, который выбирает один из нескольких путей выполнения на основе значения выражения. Долгие годы в C# существовал только классический `switch` в стиле C: сравнение на равенство с константами, обязательные `break` (или `return`/`goto`) в каждой секции `case`, и поддержка только простых типов (числа, строки, перечисления). Это работало, но код получался многословным: каждая ветка требовала `case X:`, тела и `break;`, а порядок проверки был жёстко задан сверху вниз.

В C# 7 появились **паттерны (patterns)**, а в C# 8 — **switch expressions (выражения switch)**. Они перевернули подход: switch перестал быть «оператором, который ничего не возвращает», и стал выражением, которое вычисляет значение. Это ближе к математическому `match`, чем к классическому `switch`.

**Паттерны** — это способ проверить значение на соответствие форме. Основные виды:

- **Константный паттерн (constant pattern)** — `case 0:`, `case null:`, `case "admin"`. Проверка на равенство конкретному значению. По сути, это классический `case`, но в новой синтаксической обёртке.
- **Паттерн типа (type pattern)** — `case int i:`. Проверяет тип и одновременно объявляет переменную этого типа для использования в теле. Удобно для работы с `object` или интерфейсами.
- **Паттерн свойств (property pattern)** — `case { Length: > 0 }:`. Проверяет свойства объекта. Можно вкладывать проверки: `case { Name: "X", Age: >= 18 }:`. Компактнее, чем каскад `if`.
- **Discard `_`** — «всё остальное». Заменяет `default` в выражениях и означает «любое значение, которое мне не нужно».

**Фильтры `when`** позволяют добавить дополнительное условие к `case`, которое нельзя выразить одним паттерном: `case int i when i > 0:`. Это спасает, когда нужно учесть логику вне формы значения (например, состояние внешнего объекта или диапазон с дополнительным условием).

**Switch expressions** выглядят иначе: вместо `case X: return Y;` пишут `=> Y,` (стрелка и запятая). Каждая «рука» (arm) — это `pattern => expression`. Компилятор проверяет **полноту (exhaustiveness)**: если возможны значения, не покрытые паттернами, и нет `_`, будет ошибка или предупреждение. Это убирает целый класс багов «забыли ветку».

**C# 12 / .NET 8**: рекомендуется предпочитать **switch expressions** там, где нужно вычислить значение по входу. Они короче, декларативнее и заставляют думать о покрытии. Классический `switch` остаётся уместным, когда ветки содержат сложные побочные эффекты, несколько операторов или ранние `return`. Не смешивайте стили без необходимости.

Аналогия: классический `switch` — это сортировочная доска, где курьер вручную кладёт посылку в нужную ячейку и расписывается. Switch expression — это автоматический конвейер: вы описываете правила «если посылка такого вида — на такой выход», а система сама направляет. Меньше ручной работы, меньше ошибок, но правила должны быть полными.

Важно: с C# 7 порядок `case` имеет значение, потому что проверка идёт сверху вниз, и первый подошедший паттерн выигрывает. Поэтому более специфичные паттерны ставят выше, а `_` или `default` — последним.

#### Theory (EN)

The `switch` statement is a branching construct that selects one of several execution paths based on the value of an expression. For many years C# had only the classic C-style `switch`: equality comparison against constants, mandatory `break` (or `return`/`goto`) in each `case` section, and support limited to simple types (numbers, strings, enums). It worked, but the code was verbose: every branch needed `case X:`, a body, and `break;`, and the order of checks was strictly top-to-bottom.

C# 7 introduced **patterns**, and C# 8 added **switch expressions**. They changed the model: `switch` stopped being a statement that returns nothing and became an expression that computes a value. This is closer to a mathematical `match` than to the classic `switch`.

**Patterns** are a way to test a value against a shape. The main kinds:

- **Constant pattern** — `case 0:`, `case null:`, `case "admin"`. Equality check against a specific value. Essentially the classic `case`, in new syntactic clothing.
- **Type pattern** — `case int i:`. Checks the type and simultaneously binds a variable of that type for use in the body. Handy when working with `object` or interfaces.
- **Property pattern** — `case { Length: > 0 }:`. Inspects properties of an object. You can nest checks: `case { Name: "X", Age: >= 18 }:`. More compact than a cascade of `if`s.
- **Discard `_`** — “everything else”. Replaces `default` in expressions and means “any value I do not need”.

**`when` filters** add an extra condition to a `case` that a single pattern cannot express: `case int i when i > 0:`. This rescues you when the logic depends on something outside the value’s shape (for example, external state or a range with an extra condition).

**Switch expressions** look different: instead of `case X: return Y;` you write `=> Y,` (arrow and comma). Each arm is `pattern => expression`. The compiler checks **exhaustiveness**: if some values are possible but uncovered and there is no `_`, you get an error or warning. This eliminates a whole class of “forgot a branch” bugs.

**C# 12 / .NET 8**: prefer **switch expressions** when you need to compute a value from an input. They are shorter, more declarative, and force you to think about coverage. The classic `switch` remains appropriate when branches contain complex side effects, multiple statements, or early `return`s. Do not mix styles without reason.

Analogy: the classic `switch` is a sorting table where a courier manually places each parcel into the right slot and signs for it. A switch expression is an automatic conveyor: you describe rules “if the parcel has this shape — send it to this exit”, and the system routes it. Less manual work, fewer errors — but the rules must be complete.

Important: since C# 7 the order of `case` matters, because checks run top-to-bottom and the first matching pattern wins. Put more specific patterns higher and `_` or `default` last.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — top-level statements / pattern matching / switch expression
// Демонстрация: классический switch, switch expression, паттерны, when-фильтры, discard _
// Demonstration: classic switch, switch expression, patterns, when filters, discard _

using System;
using System.Collections.Generic;

// --- 1. Классический switch (оператор) / Classic switch statement ---
// Уместен, когда ветки содержат побочные эффекты или несколько операторов.
// Appropriate when branches have side effects or multiple statements.
string GradeToLabelClassic(char grade)
{
    switch (grade)              // сравнение с константами / equality with constants
    {
        case 'A':              // константный паттерн / constant pattern
            Console.WriteLine("Отлично / Excellent");
            return "Excellent";
        case 'B':
            return "Good";
        case 'C':
            return "Satisfactory";
        case 'D':
        case 'E':              // несколько меток на одну ветку / multiple labels, one section
            return "Pass";
        case 'F':
            return "Fail";
        default:               // всё остальное / everything else
            return "Unknown";
    }
}

// --- 2. Switch expression — вычисление значения / Switch expression computes a value ---
// Рекомендуемый стиль в C# 12 для отображения входа в значение.
// Recommended style in C# 12 for mapping input to a value.
string GradeToLabel(char grade) => grade switch
{
    'A' => "Excellent",
    'B' => "Good",
    'C' => "Satisfactory",
    'D' or 'E' => "Pass",      // or-паттерн объединяет константы / or-pattern combines constants
    'F' => "Fail",
    _   => "Unknown"           // discard — обязателен для полноты / discard — required for exhaustiveness
};

// --- 3. Паттерн типа + when-фильтр / Type pattern + when filter ---
decimal Price(object item) => item switch
{
    null                  => 0m,                // константный паттерн null / constant null pattern
    Book b when b.IsUsed  => b.BasePrice * 0.5m, // when-фильтр: доп. условие / when filter: extra condition
    Book b                => b.BasePrice,        // паттерн типа / type pattern
    ElectronProduct { WarrantyMonths: >= 12 } e  // паттерн свойств со вложенным сравнением
        => e.BasePrice * 1.1m,                  // property pattern with nested comparison
    ElectronProduct e     => e.BasePrice,        // менее специфичный — ниже / less specific — lower
    _                     => throw new ArgumentException("Неизвестный товар / Unknown item")
};

// --- 4. Паттерн свойств и кортежей / Property and tuple patterns ---
record Order(decimal Amount, string Country, bool IsVip);

decimal Tax(Order o) => o switch
{
    { Country: "DE", IsVip: true }  => 0m,       // Германия + VIP / Germany + VIP
    { Country: "DE" }               => o.Amount * 0.19m,
    { Country: "US" or "CA" }       => 0m,       // or-паттерн / or-pattern
    { IsVip: true }                 => 0m,       // любой VIP / any VIP
    _                               => o.Amount * 0.20m  // discard по умолчанию / discard default
};

// --- 5. Кортежный switch expression / Tuple switch expression ---
string Describe(int x, int y) => (x, y) switch
{
    (0, 0)   => "начало координат / origin",
    (0, _)   => "на оси Y / on Y axis",          // discard в кортеже / discard in tuple
    (_, 0)   => "на оси X / on X axis",
    (var a, var b) when a == b => "диагональ / diagonal", // when + var / when + var
    _        => "обычная точка / ordinary point"
};

// --- Вспомогательные типы / Helper types ---
record Book(decimal BasePrice, bool IsUsed);
record ElectronProduct(decimal BasePrice, int WarrantyMonths);

// --- Демонстрационный запуск / Demo run ---
Console.WriteLine(GradeToLabel('B'));                 // Good
Console.WriteLine(Price(new Book(100m, true)));       // 50
Console.WriteLine(Price(new ElectronProduct(200m, 24))); // 220
Console.WriteLine(Tax(new Order(1000m, "DE", true))); // 0
Console.WriteLine(Describe(5, 5));                    // диагональ / diagonal
```

#### Best Practices

- Предпочитай switch expression, когда результат — это отображение входа в значение; используй классический switch для веток с побочными эффектами и ранними return. (RU)
- Prefer switch expressions when the result is a mapping from input to a value; use the classic switch for branches with side effects and early returns. (EN)
- Располагай более специфичные паттерны выше, а `_`/`default` — последним: проверка идёт сверху вниз, и первый совпавший выигрывает. (RU)
- Place more specific patterns higher and `_`/`default` last: matching runs top-to-bottom and the first match wins. (EN)
- Включай `_` или `default` осознанно: он закрывает полноту, но может скрыть забытую ветку — предпочитай явный перечень для закрытого множества (enum). (RU)
- Add `_` or `default` deliberately: it satisfies exhaustiveness but can hide a forgotten branch — prefer an explicit list for closed sets (enums). (EN)
- Не повторяй логику в нескольких `case`: используй `or`-паттерн или вынеси общую часть в метод. (RU)
- Do not repeat logic across several `case`s: use an `or` pattern or extract the shared part into a method. (EN)

#### Частые ошибки / Common Mistakes

- Забыли `break`/`return` в классическом switch → fall-through между ветками. → В C# 8+ компилятор запрещает неявный fall-through; используй switch expression, где `=> expr,` заменяет `break`. (RU)
- Forgot `break`/`return` in a classic switch → fall-through between branches. → In C# 8+ the compiler forbids implicit fall-through; use a switch expression where `=> expr,` replaces `break`. (EN)
- Поставили общий паттерн выше специфичного → общая ветка «съест» все случаи. → Сортируй от частного к общему, `_` всегда последним. (RU)
- Put a general pattern above a specific one → the general branch swallows all cases. → Sort from specific to general, with `_` always last. (EN)
- Использовали `_` вместо перечисления всех значений enum → при добавлении нового значения enum баг тихо попадёт в default. → Перечисляйте все варианты явно, чтобы компилятор сообщил о новом enum-члене. (RU)
- Used `_` instead of listing all enum values → adding a new enum member silently routes the bug into default. → List all cases explicitly so the compiler warns about a new enum member. (EN)
- Полагались на порядок в switch expression, не проверив полноту → runtime-исключение на непокрытом значении. → Включайте `_` или `throw` для действительно невозможных входов, и тестируйте граничные случаи. (RU)
- Relied on arm order in a switch expression without checking exhaustiveness → runtime exception on an uncovered value. → Add `_` or `throw` for genuinely impossible inputs, and test boundary cases. (EN)

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я выбрал switch expression для отображения значения и классический switch для побочных эффектов. (RU)
- [ ] Каждый паттерн расположен от специфичного к общему, `_`/`default` — последним. (RU)
- [ ] Полнота (exhaustiveness) обеспечена: либо `_`/`default`, либо полный перечень enum/типов. (RU)
- [ ] `when`-фильтры применены только там, где паттерн не выражает условие. (RU)
- [ ] Код компилируется на C# 12 / .NET 8 без предупреждений о непокрытых ветках. (RU)
- [ ] I chose a switch expression for value mapping and the classic switch for side effects. (EN)
- [ ] Every pattern is ordered from specific to general, with `_`/`default` last. (EN)
- [ ] Exhaustiveness is guaranteed: either `_`/`default` or a full list of enums/types. (EN)
- [ ] `when` filters are used only where a pattern cannot express the condition. (EN)
- [ ] The code compiles on C# 12 / .NET 8 with no warnings about uncovered arms. (EN)

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/statements/selection-statements — Selection statements (if, switch) / Операторы выбора (if, switch)](https://learn.microsoft.com/dotnet/csharp/language-reference/statements/selection-statements)
- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/fundamentals/functional/pattern-matching — Pattern matching / Сопоставление шаблонов](https://learn.microsoft.com/dotnet/csharp/fundamentals/functional/pattern-matching)

---
[← Предыдущий: M03-L01](lesson-M03-L01-if-ternary.md) | [⬆ К модулю M03](../README.md) | [Следующий: M03-L03 →](lesson-M03-L03-loops.md)
