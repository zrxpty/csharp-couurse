---
[← Предыдущий: M02-L06](lesson-M02-L06-type-conversion.md) | [⬆ К модулю M02](../README.md) | [Следующий: M02-L08 →](lesson-M02-L08-enum-tuples.md)
---

### Урок M02-L07: string основы, интерполяция, StringBuilder / string basics, interpolation, StringBuilder

**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Строка в C# — это последовательность символов Unicode, представленная типом `string` (псевдоним `System.String`). Самое важное свойство строки — **неизменяемость (immutability)**: однажды созданная строка не может быть изменена. Любая операция, которая «меняет» строку (конкатенация, обрезка, замена), на самом деле создаёт **новый** объект строки в куче, а старый остаётся в памяти до сборки мусора.

Представьте строки как гравированные каменные таблички: чтобы изменить хоть одну букву, вы не можете стереть надпись — нужно взять новый камень и выбить текст заново. Это свойство делает строки потокобезопасными и позволяет им жить в одном пуле (интернирование), но цена — лишние выделения памяти при интенсивной сборке текста.

**Конкатенация vs интерполяция.** Сложить строки можно оператором `+`: `"Hello, " + name`. Это работает, но читается плохо, особенно когда переменных много. Начиная с C# 6 предпочтительнее **интерполяция** строк со знаком `$`: `$"Hello, {name}, you are {age} years old"`. Компилятор разворачивает её в вызов `string.Format` (а в C# 10+ — в более эффективный `DefaultInterpolatedStringHandler`), код становится самодокументируемым, а выравнивание и форматы задаются прямо в фигурных скобках: `$"{value,5:N2}"`.

**Verbatim-строки (`@"..."`).** Префикс `@` отключает escape-последовательности, поэтому обратные слэши трактуются буквально — идеально для путей Windows и регулярных выражений: `@"C:\temp\file.txt"`. Кроме того, verbatim-строка может занимать несколько строк, сохраняя переводы строк и отступы внутри литерала. Чтобы вставить сам символ кавычки внутри verbatim-строки, его удваивают: `@"He said ""hi""."`.

**Raw string literals (`"""..."""`), C# 11.** Когда нужно поместить в строку кавычки, фигурные скобки, JSON, XML или многострочный текст без экранирования, используют «сырые» строки. Они начинаются и заканчиваются тремя и более двойными кавычками: `"""` . Внутри можно свободно писать кавычки и `\`, а отступы вычисляются по закрывающему разделителю — лишние отступы убираются автоматически. В сочетании с `$` интерполяция работает, а фигурные скобки экранируются их количеством: `$$$"""{ { {} } }"""` интерполирует только тройные скобки.

**`string.Empty`.** Пустая строка `""` каждый раз в коде — это литерал; `string.Empty` — константа, которая читается яснее и в редких случаях экономит сравнения. Главное правило: для проверки на пустоту не используйте `if (s == "")` или `if (s.Length == 0)` без проверки на `null` — вызывайте `string.IsNullOrEmpty(s)` или `string.IsNullOrWhiteSpace(s)`.

**StringBuilder для циклов.** Неизменяемость бьёт по производительности, когда строки собираются в цикле. Выражение `s += part` создаёт новую строку длиной `len(s)+len(part)` и копирует оба содержимых — квадратичная сложность для N итераций. Класс `System.Text.StringBuilder` хранит внутренний изменяемый буфер (char-массив) и использует стратегию удвоения емкости, поэтому вызовы `Append`/`AppendLine` амортизируют до O(1), а итоговый `ToString()` строится один раз. Эмпирическое правило: **5+ итераций конкатенации → StringBuilder**. Для разовой сборки строки из нескольких полей интерполяция по-прежнему лучше.

#### Theory (EN)

A string in C# is a sequence of Unicode characters represented by the `string` type (an alias for `System.String`). The single most important property of a string is **immutability**: once created, a string object cannot be modified. Any operation that appears to "change" a string — concatenation, trimming, replacing — actually creates a brand-new string object on the heap, leaving the old one in memory until the garbage collector reclaims it.

Think of strings as engraved stone tablets: to change a single letter you cannot erase the inscription — you must take a fresh stone and chisel the whole text again. This makes strings thread-safe and lets identical literals be interned (shared) automatically, but the price is extra allocations when text is assembled heavily.

**Concatenation vs interpolation.** You can join strings with the `+` operator: `"Hello, " + name`. It works, but reads poorly when there are many variables. Since C# 6, **interpolation** with the `$` prefix is preferred: `$"Hello, {name}, you are {age} years old"`. The compiler lowers it to `string.Format` (and in C# 10+ to a more efficient `DefaultInterpolatedStringHandler`), the code becomes self-documenting, and alignment/format specifiers live right in the braces: `$"{value,5:N2}"`.

**Verbatim strings (`@"..."`).** The `@` prefix disables escape sequences, so backslashes are treated literally — perfect for Windows paths and regular expressions: `@"C:\temp\file.txt"`. A verbatim string can also span multiple lines, preserving line breaks and indentation inside the literal. To embed a quote inside a verbatim string, double it: `@"He said ""hi""."`.

**Raw string literals (`"""..."""`), C# 11.** When you need to embed quotes, braces, JSON, XML or multi-line text without escaping, use raw strings. They are delimited by three or more double quotes: `"""`. Inside, you can freely write quotes and backslashes; leading indentation is computed from the closing delimiter and trimmed automatically. Combined with `$`, interpolation still works, and braces are escaped by repeating them: `$$$"""{ { {} } }"""` interpolates only triple braces.

**`string.Empty`.** The literal `""` is fine, but `string.Empty` is a constant that reads clearer and occasionally saves comparisons. The real rule for emptiness checks: never write `if (s == "")` or `if (s.Length == 0)` without first guarding against `null` — use `string.IsNullOrEmpty(s)` or `string.IsNullOrWhiteSpace(s)`.

**StringBuilder for loops.** Immutability hurts performance when strings are assembled in a loop. The expression `s += part` allocates a new string of length `len(s)+len(part)` and copies both contents — quadratic over N iterations. `System.Text.StringBuilder` keeps a mutable internal char buffer with a doubling-capacity strategy, so `Append`/`AppendLine` calls amortize to O(1), and the final `ToString()` is built once. Rule of thumb: **5+ concatenation iterations → StringBuilder**. For a one-off assembly of a few fields, interpolation remains preferable.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — top-level statements / top-level statements
using System.Text;

string userName = "Anna";
int age = 28;
decimal balance = 1234.56m;

// 1) Конкатенация против интерполяции / Concatenation vs interpolation
string concatenated = "User: " + userName + ", age: " + age;       // работает, но шумно / works but noisy
string interpolated = $"User: {userName}, age: {age}";              // читаемо, предпочтительно / readable, preferred
Console.WriteLine(interpolated);

// Форматирование прямо в скобках / Formatting right in the braces
Console.WriteLine($"Balance: {balance,10:C}");                      // валютный формат, ширина 10 / currency, width 10

// 2) Verbatim-строки — пути без экранирования / Verbatim — paths without escaping
string path = @"C:\temp\report.txt";
string quoted = @"She said ""hello"".";                              // удвоенная кавычка / doubled quote
Console.WriteLine(path);
Console.WriteLine(quoted);

// 3) Raw string literals (C# 11) — JSON/HTML без экранирования / Raw — JSON/HTML without escaping
string json = $$"""
{
  "user": "{{userName}}",
  "age": {{age}},
  "tags": ["vip", "new"]
}
""";
Console.WriteLine(json);

// 4) string.Empty и проверки на пустоту / string.Empty and emptiness checks
string maybe = "";
if (string.IsNullOrEmpty(maybe))
{
    maybe = string.Empty;  // явно присваиваем пустую / explicitly assign empty
}

// 5) StringBuilder для циклов — O(n) вместо O(n^2) / StringBuilder for loops
var lines = new[] { "alpha", "beta", "gamma", "delta", "epsilon", "zeta" };
var sb = new StringBuilder();
foreach (var line in lines)
{
    sb.Append("- ").AppendLine(line);   // изменяемый буфер, без новых строк / mutable buffer, no new strings
}

string result = sb.ToString();          // один финальный аллок / single final allocation
Console.WriteLine(result);

// 6) Pattern matching + интерполяция в switch / Pattern matching + interpolation in switch
string classification = age switch
{
    < 18 => $"Child: {userName}",
    >= 18 and < 65 => $"Adult: {userName}",
    _ => $"Senior: {userName}"
};
Console.WriteLine(classification);
```

#### Best Practices
- Используйте `$`-интерполяцию вместо `+` для сборки строки из нескольких значений — это самодокументируемый код. / Use `$`-interpolation instead of `+` to assemble a string from several values — it is self-documenting code.
- В циклах с 5+ итерациями сборки текста применяйте `StringBuilder` с `Append`/`AppendLine`. / Use `StringBuilder` with `Append`/`AppendLine` in loops with 5+ iterations of text assembly.
- Для путей Windows и regex используйте verbatim `@"..."`, для многострочного JSON/HTML — raw `"""..."""`. / Use verbatim `@"..."` for Windows paths and regex, raw `"""..."""` for multi-line JSON/HTML.
- Проверяйте пустоту через `string.IsNullOrEmpty` или `string.IsNullOrWhiteSpace`, а не через `== ""`. / Check emptiness with `string.IsNullOrEmpty` or `string.IsNullOrWhiteSpace`, not `== ""`.
- Не храните секреты в литералах строк — выносите их в конфигурацию; строки интернируются и живут в памяти. / Do not store secrets in string literals — move them to configuration; strings are interned and persist in memory.

#### Частые ошибки / Common Mistakes
- `s += part` в цикле на тысячи итераций → квадратичная аллокация и тормоза; используйте `StringBuilder`. / `s += part` in a loop of thousands of iterations → quadratic allocation and slowdowns; use `StringBuilder`.
- `if (s == "")` при `s == null` выбрасывает или даёт ложное поведение → используйте `string.IsNullOrEmpty(s)`. / `if (s == "")` when `s == null` throws or misbehaves → use `string.IsNullOrEmpty(s)`.
- Экранирование обратных слэшей в путях `"C:\\temp\\file.txt"` вместо `@"C:\temp\file.txt"` → шум и баги. / Escaping backslashes in paths `"C:\\temp\\file.txt"` instead of `@"C:\temp\file.txt"` → noise and bugs.
- Путаница кавычек в JSON-литерале вручную → переходите на raw string `"""..."""` с правильным числом `$`. / Manual quote juggling in a JSON literal → switch to raw string `"""..."""` with the correct count of `$`.
- Вызов `.ToString()` на `StringBuilder` после каждого `Append` «на всякий случай» → убивает преимущество, вызывайте один раз. / Calling `.ToString()` on `StringBuilder` after every `Append` "just in case" → kills the benefit, call once.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я объясняю, почему строки неизменяемы, и называю последствия (аллокации, потокобезопасность).
- [ ] Я выбираю `$`-интерполяцию для сборки строки из значений и `StringBuilder` для циклов 5+.
- [ ] Я корректно использую `@"..."` для путей и `"""..."""` для многострочного JSON/HTML.
- [ ] Я проверяю пустоту через `IsNullOrEmpty` / `IsNullOrWhiteSpace`, а не через `== ""`.
- [ ] Я могу показать разницу O(n) vs O(n^2) между `StringBuilder` и конкатенацией в цикле.
- [ ] I can explain why strings are immutable and name the consequences (allocations, thread safety).
- [ ] I choose `$`-interpolation for assembling from values and `StringBuilder` for 5+ iteration loops.
- [ ] I correctly use `@"..."` for paths and `"""..."""` for multi-line JSON/HTML.
- [ ] I check emptiness with `IsNullOrEmpty` / `IsNullOrWhiteSpace`, not `== ""`.
- [ ] I can demonstrate the O(n) vs O(n^2) difference between `StringBuilder` and in-loop concatenation.

#### Ресурсы / Resources
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/tour-of-csharp/program-building-blocks#strings — Строки (C#): основы / Strings (C#): basics
- Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.text.stringbuilder — Класс StringBuilder: изменяемые строки / StringBuilder class: mutable strings

---
[← Предыдущий: M02-L06](lesson-M02-L06-type-conversion.md) | [⬆ К модулю M02](../README.md) | [Следующий: M02-L08 →](lesson-M02-L08-enum-tuples.md)
---
