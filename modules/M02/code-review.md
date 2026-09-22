# Ревью кода модуля M02 / Code Review: M02
# Типы, переменные, операторы / Types, Variables, Operators

> Ревьюер / Reviewer: Агент 5 (Code Reviewer) — Senior/Lead C#
> Проверено уроков / Lessons reviewed: 8
> Целевая платформа / Target: C# 12 / .NET 8

## Сводка / Summary

- **Готово / Ready:** 5/8 уроков (M02-L02, L04, L05, L07, L08)
- **Нужно исправить / Needs fix (compile-blocking):**
  - **M02-L01** — `with`-выражение на обычном `readonly struct` не компилируется / `with` on a plain `readonly struct` does not compile.
  - **M02-L03** — `box is var value and > 10 and < 100` не компилируется для `object` / relational pattern on `object` does not compile.
  - **M02-L06** — `int guarded = checked { max + 1 };` — `checked { }` это statement, не выражение / `checked { }` is a statement, not an expression.
- **Editorial (текст, не код) / Text-only:** M02-L05 — китайский артефакт «标记те» в строке 19 / Chinese artifact in line 19.
- **Устаревает / Deprecated:** нет / none. Везде C# 12 / .NET 8, актуально.
- **Общая оценка / Overall:** Материал методически сильный, двуязычный, с хорошими аналогиями и best-practices. Однако в трёх уроках кодовые примеры не проходят компиляцию — это блокирующие ошибки, которые нужно исправить до публикации. После правок модуль готов к выпуску. / The material is methodologically strong, bilingual, with good analogies and best practices. However, three lessons contain code samples that do not compile — these are blocking issues that must be fixed before release. After fixes, the module is ready to ship.

---

## Детальный разбор по урокам / Per-lesson review

### M02-L01 — Значимые и ссылочные типы / Value vs Reference Types

#### ✅ Что ок / What's good
- Отличная аналогия «ценная бумага vs адрес сейфа» (строка 13/33), RU+EN同步. / Excellent "bearer bond vs safe address" analogy, mirrored RU+EN.
- `readonly struct Money` с primary constructor (C# 12, строки 54-64) — современный, корректный синтаксис. / Modern, correct primary-constructor syntax.
- Корректно показан boxing/unboxing (строки 88-92) и альтернатива через дженерики (строки 94-97). / Boxing/unboxing and the generic alternative shown correctly.
- Collection expression `List<int> numbers = [1, 2, 3, 4, 5];` (строка 95) — C# 12, актуально. / C# 12 collection expression, up to date.
- Pattern matching `switch` (строки 106-114) с property pattern `BankAccount { Owner: var o }` — корректно. / Pattern matching is correct.

#### ⚠️ Что улучшить / What to improve
- **[BLOCKING] Строки 61, 80: `with`-выражение на обычном `readonly struct`.**
  ```csharp
  this with { Amount = Amount + other.Amount }   // строка 61
  b = b with { Amount = 200m };                    // строка 80
  ```
  `with` поддерживается только для `record`, `record struct` (C# 10+), `record class` и анонимных типов. Для обычного `readonly struct` компилятор выдаст ошибку (get-only свойства нельзя установить в инициализаторе). / `with` is only supported on `record`/`record struct`/anonymous types — a plain `readonly struct` will not compile.
  → **Рекомендация:** объявить как `public readonly record struct Money(decimal amount, string currency)`. Тогда `with`, автогенерируемые `Equals`/`GetHashCode` и `==` заработают — и это ещё лучше соответствует Best Practices урока (строка 127 прямо советует `record struct`). / Declare it as `readonly record struct` — then `with`, auto-`Equals`/`GetHashCode`, and `==` all work, matching the lesson's own advice in line 127.

#### 🚫 Code smells
- Строка 108: `int i when i > 0` — guard-паттерн `when` избыточен для чистого relational-паттерна. / The `when` guard is redundant.
  → Заменить на `int and > 0` (C# 9+ relational pattern). Cleaner: `> 0 => ...` после типизированного `int i`. / Replace with a relational pattern `int and > 0`.
- Строка 21 (RU): «две переменные с одинаковым литералом безопасно „шарятся“ ссылку» — грамматика (падеж) + сленг. / Grammar + slang in RU text.
  → «безопасно разделяют ссылку». / "safely share a reference".

#### 🏷️ Статус
- **Needs fix** (блокирующая ошибка компиляции с `with`). / Blocking compile error with `with`.

---

### M02-L02 — Примитивы / Primitives

#### ✅ Что ок / What's good
- Все литералы и суффиксы корректны: `8_100_000_000L`, `4_000_000_000u`, `10_000_000_000ul`, `36.6f`, `19.99m` (строки 51-54, 61, 66). / All literals and suffixes are correct.
- `0.1 + 0.2` demo (строка 62) с предупреждением — отличная иллюстрация IEEE 754. / Great IEEE 754 illustration.
- Raw string literal для JSON (строки 85-91) — C# 11, актуально. / C# 11 raw string literal, up to date.
- Pattern matching `switch` по runtime-типам (строки 97-108) — корректный, компилируемый код. / Pattern matching compiles and is correct.
- Финансовая арифметика в `decimal` (строки 66-69) с `:C` форматом — best practice. / Money in `decimal` with currency format — best practice.

#### ⚠️ Что улучшить / What to improve
- Строка 52: `byte age = 255;` — технически верно, но `age = 255` для «возраста» семантически странно. / Semantically odd value for "age".
  → Использовать реалистичное `byte age = 35;` или переименовать переменную в `maxByte` для демо диапазона. / Use a realistic value or rename the variable.
- Строка 63: вывод `0.1+0.2={binarySum}` выведет `0.30000000000000004` — стоит явно показать ожидание vs реальность. / Show expected vs actual explicitly.

#### 🚫 Code smells
- Существенных code smells нет. / No significant code smells.

#### 🏷️ Статус
- **Готово / Ready.**

---

### M02-L03 — var, const, readonly

#### ✅ Что ок / What's good
- Чёткое разделение явной/неявной типизации, `const` vs `readonly` (строки 34-57). / Clear distinction between explicit/implicit typing and `const` vs `readonly`.
- Класс `Configuration` (строки 180-186) с `const` + `readonly` — корректная иллюстрация. / Correct illustration.
- camelCase-именование показано явно (строки 174-178). / camelCase naming shown explicitly.
- `var files = Directory.GetFiles(...)` (строка 145) — уместное применение `var`. / Appropriate use of `var`.

#### ⚠️ Что улучшить / What to improve
- **[BLOCKING] Строка 169: `if (box is var value and > 10 and < 100)`.**
  `box` имеет тип `object`. Relational-паттерны `> 10` / `< 100` требуют, чтобы тип входного значения поддерживал сравнение; `object` его не поддерживает → ошибка компиляции (CS8515 / "relational pattern cannot be used with type 'object'"). / `box` is `object`; relational patterns require a comparable input type, so this does not compile.
  → **Рекомендация:** типизированный паттерн `if (box is int value and > 10 and < 100)`. Тогда `value` — `int`, и relational-паттерны валидны. / Use a typed pattern `box is int value and > 10 and < 100`.

#### 🚫 Code smells
- Строка 169 (после правки): паттерн `is var value` сам по себе всегда `true` и добавляет мало смысла — лучше сразу `is int value`. / `is var value` always succeeds; prefer a typed pattern.
- Строка 145: `Directory.GetFiles` возвращает `string[]`, не `List` — комментарий «LINQ и сложные дженерики» в теории (строка 30) чуть неточен для этого примера. / Minor theory/example mismatch.

#### 🏷️ Статус
- **Needs fix** (блокирующая ошибка компиляции relational-паттерна на `object`). / Blocking compile error with relational pattern on `object`.

---

### M02-L04 — Операторы / Operators

#### ✅ Что ок / What's good
- Целочисленное деление vs дробное (строки 53-55) показано наглядно. / Integer vs fractional division shown clearly.
- Short-circuit `&&` с защитой от null (строка 64) — корректно и поучительно. / Short-circuit null guard is correct and instructive.
- `[Flags] enum Permissions` с побитовыми операциями `|`, `&`, `^=` (строки 73-78) — best practice. / `[Flags]` enum with bitwise ops — best practice.
- `??=` с target-typed `new()` (строки 90-91) — C# 9/12, актуально. / `??=` with target-typed `new()` — modern.
- `checked(max + 1)` выражение (строка 100) — корректный синтаксис checked-выражения. / Correct checked expression syntax.

#### ⚠️ Что улучшить / What to improve
- Строка 76: `bool canWrite = (perms & Permissions.Write) != 0;` — сравнение enum с `0` работает, но идиоматичнее `(perms & Permissions.Write) == Permissions.Write` или `perms.HasFlag(Permissions.Write)`. / Idiomatic flag check prefers `HasFlag` or `== flag`.
- Строка 82: `~x = -2` — стоит добавить однострочное пояснение two's complement, иначе новичок запутается. / Add a one-line two's-complement note.
- Строка 87: вложенный тернарный `score >= 90 ? "A" : score >= 60 ? "C" : "F"` — допустимо, но в Best Practices тут же сказано «избегать вложенности». Лёгкое противоречие текст/код. / Mild text/code contradiction about nested ternaries.

#### 🚫 Code smells
- Строка 73: объявление `[Flags] enum` внутри top-level statements работает, но в реальном коде enums обычно в отдельных файлах. Для урока — допустимо. / Declaring an enum inside top-level statements is fine for a lesson but atypical in real code.

#### 🏷️ Статус
- **Готово / Ready.**

---

### M02-L05 — null и nullable-типы / null and nullable types

#### ✅ Что ок / What's good
- `#nullable enable` (строка 44) — правильное включение NRT в примере. / NRT correctly enabled.
- `int?` / `Nullable<int>` с `HasValue`/`Value` (строки 52-64) — корректно. / Nullable value types handled correctly.
- `?.`-цепочка `p?.Address?.City ?? "нет города"` (строки 85-87) — эталонная демонстрация. / Textbook null-conditional chain.
- `required string Name { get; init; }` (строка 156) — C# 11, актуально. / C# 11 `required` + `init`.
- `suffix is { } s ? ... : string.Empty` (строка 145) — правильная альтернатива `!`-оператору. / Good alternative to the `!` operator.
- Pattern matching с nullable в `switch` (строки 92-99) — корректный, компилируемый код. / Pattern matching compiles and is correct.

#### ⚠️ Что улучшить / What to improve
- **[Editorial, не код] Строка 19 (RU-теория):** «для ссылочных типов включайте NRT во всём проекте и явно**标记те** nullable-поля через `?`» — вставлен китайский фрагмент «标记» (вместо «помечайте»). / Chinese fragment "标记" accidentally inserted instead of "помечайте".
  → Заменить на «явно помечайте nullable-поля через `?`». / Replace with "explicitly mark nullable fields with `?`".
- Строка 129: `readings.Where(r => r.HasValue).Select(r => r!.Value)` — `!` здесь избыточен (после `Where(r => r.HasValue)` компилятор ещё не знает, что `r` не null, так что `!` технически нужен, но лучше `Select(r => r.Value)` срабатывает без `!`? — на самом деле компилятор не отслеживает `.Where`, поэтому `!` обоснован). Стоит прокомментировать, почему `!` здесь легитимен. / The `!` is legitimate but should be commented.
- Строка 70: `string city = "Saint-Petersburg";` (non-null) → `city ?? "неизвестно"` никогда не сработает; для демо `??` лучше показать с `string? city = null;`. / Demo `??` with a non-null variable never triggers the fallback.

#### 🚫 Code smells
- Существенных code smells нет. / No significant code smells.

#### 🏷️ Статус
- **Готово / Ready** (с editorial правкой текста в строке 19). / Ready (with a text fix in line 19).

---

### M02-L06 — Преобразования типов / Type conversions

#### ✅ Что ок / What's good
- Неявное vs явное приведение (строки 54-66) — корректно. / Implicit vs explicit casts correct.
- `Convert.ToInt32(2.5)` → banker's rounding (строка 70) с пояснением — отлично. / Banker's rounding explained well.
- `Convert.ToInt32(null)` → 0 vs `int.Parse(null)` → throw (строки 73-75) — важное различие контрактов. / Important contract distinction.
- `int.TryParse(input, out int age)` (строка 96) — best practice для пользовательского ввода. / Best practice for user input.
- Pattern matching `boxed is string text` (строки 121-125) — корректная альтернатива `as`. / Correct `is`-pattern cast.

#### ⚠️ Что улучшить / What to improve
- **[BLOCKING] Строка 112: `int guarded = checked { max + 1 };`.**
  `checked { ... }` — это **statement** (блок), а не выражение, и не возвращает значение. Присвоить его `int guarded` нельзя — ошибка компиляции. / `checked { }` is a statement, not an expression, so this assignment does not compile.
  → **Рекомендация:** использовать checked-выражение `int guarded = checked(max + 1);` (как в L04 строка 100) ИЛИ блок:
  ```csharp
  try
  {
      checked { int guarded = max + 1; _ = guarded; }
  }
  catch (OverflowException ex) { ... }
  ```
  / Use `checked(max + 1)` expression, or a `checked { }` statement block.

#### 🚫 Code smells
- Строка 132: `int.TryParse(Console.ReadLine(), out int n) && n >= 0` — `Console.ReadLine()` может вернуть `null` (EOF), `TryParse` на `null` вернёт `false` без исключения, так что код безопасен. ОК, но стоит отметить в комментарии. / Safe, but worth a comment.

#### 🏷️ Статус
- **Needs fix** (блокирующая ошибка: `checked { }` как выражение). / Blocking: `checked { }` used as an expression.

---

### M02-L07 — string, интерполяция, StringBuilder

#### ✅ Что ок / What's good
- Демонстрация неизменяемости и конкатенации vs интерполяции (строки 48-50). / Immutability and `+` vs `$` shown well.
- Форматирование в скобках `{balance,10:C}` (строка 53) — наглядно. / Inline formatting demonstrated.
- Verbatim `@"..."` с удвоенной кавычкой (строки 56-57) — корректно. / Verbatim strings correct.
- Raw string literal с `$$"""..."""` и `{{ }}` интерполяцией (строки 62-68) — C# 11, правильное число `$`. / C# 11 raw string with correct `$` count.
- `StringBuilder` в цикле с `Append().AppendLine()` (строки 80-87) — best practice. / StringBuilder loop — best practice.
- `string.IsNullOrEmpty` (строка 73) — правильная проверка пустоты. / Correct emptiness check.

#### ⚠️ Что улучшить / What to improve
- Строки 72-76: блок с `string.IsNullOrEmpty(maybe)` и затем `maybe = string.Empty;` — `maybe` уже `""`, присваивание `string.Empty` семантически избыточно для читателя. Лучше показать `null`-случай. / The `maybe = string.Empty` assignment is redundant; show the `null` case instead.
- Строка 86: комментарий «один финальный аллок» — `sb.ToString()` делает одну аллокацию, верно, но `AppendLine` уже создаёт строки для аргументов? Нет, `AppendLine(string)` не аллоцирует. Комментарий корректен. / Comment is accurate.

#### 🚫 Code smells
- Существенных нет. / None significant.

#### 🏷️ Статус
- **Готово / Ready.**

---

### M02-L08 — enum и кортежи / enum and tuples

#### ✅ Что ок / What's good
- `enum PowerMode` и `[Flags] enum AccessRights` со степенями двойки и `None = 0` (строки 35-50) — best practice. / `[Flags]` with powers of two and `None = 0` — best practice.
- `0b0001` бинарные литералы (строки 46-49) — читаемо. / Binary literals are readable.
- Кортеж с именованными полями и деконструкция `var (code, msg) = GetResult();` (строки 76-77) — корректно. / Named-field tuple with deconstruction is correct.
- `record Point(int X, int Y)` с `with` (строки 88-94) — `with` для `record` работает (в отличие от L01). / `with` on a `record` works correctly.
- `pointA == pointB` для `ValueTuple` по значению (строка 86) — корректно. / Value-tuple `==` compares by value.

#### ⚠️ Что улучшить / What to improve
- Строка 68: `bool canExec = (mine & AccessRights.Exec) == AccessRights.Exec;` — корректно, но в Common Mistakes (строка 108) сказано «использовать `HasFlag` или `(x & flag) == flag`» — пример соответствует, ОК. / Example matches the guidance.
- Строка 88: `record Point(int X, int Y);` объявлен внутри top-level statements между исполняемым кодом — работает, но для чистоты лучше перенести типы в начало файла или в отдельный файл. / For cleanliness, move type declarations out of the top-level flow.

#### 🚫 Code smells
- Существенных нет. / None significant.

#### 🏷️ Статус
- **Готово / Ready.**

---

## Общие рекомендации по модулю / Module-wide recommendations

- **Компилируемость в первую очередь / Compile-first:** три из восьми примеров не собираются (L01 `with` на `struct`, L03 relational-паттерн на `object`, L06 `checked {}` как выражение). Рекомендуется прогнать все примеры через `dotnet run` в реальном проекте `.NET 8` перед публикацией — это поймает подобные ошибки автоматически. / Run every sample through a real `dotnet run` on .NET 8 before publishing; this catches compile errors automatically.
- **Единый стиль `record struct` для value-типов:** в L01 советуют `record struct` (строка 127), но сам пример использует обычный `readonly struct` + `with`, что противоречит совету и не компилируется. Привести пример к `readonly record struct` — это снимет и ошибку, и противоречие. / Align the L01 sample with its own advice by using `readonly record struct`.
- **Top-level statements и объявления типов:** несколько уроков объявляют `class`/`struct`/`enum`/`record` после исполняемых top-level statements (L01, L03, L05, L08). Это работает, но для единообразия стоит выносить типы в начало блока или в отдельные файлы. / Consider placing type declarations before top-level statements or in separate files for consistency.
- **Двуязычная консистентность / Bilingual consistency:** в RU-тексте L05 (строка 19) затесался китайский фрагмент «标记те» — нужна корректура. В L01 (строка 21) — грамматическая/стилевая правка «шарятся ссылку». Рекомендуется пройти по всем RU-текстам на предмет падежей и сленга. / Proofread RU texts for grammar/slang and stray non-Russian fragments.
- **Pattern matching — унификация:** L01 использует `when`-guard (`int i when i > 0`), L02/L05 используют чистые relational-паттерны. Стоит унифицировать к современному стилю `is int and > 0` / relational без `when`. / Unify pattern-matching style toward relational patterns without `when` guards.
- **`checked` единообразно:** L04 использует `checked(max + 1)` (выражение — верно), L06 — `checked { max + 1 }` (блок-как-выражение — ошибка). Унифицировать к `checked(expr)`. / Unify to `checked(expr)` expression form.
- **Best Practices ↔ Code consistency:** убедиться, что каждый «Best Practice» в уроке выполняется в собственном примере кода (L01 `record struct`, L04 вложенный тернарний vs «избегать вложенности»). / Ensure each lesson's code follows its own stated best practices.
- **Именование:** переменные вроде `d`, `dd`, `b` в L02 (строки 102-103) — короткие, но в switch-демо допустимы; в продакшен-коде стоит избегать. Модуль в целом соблюдает camelCase/PascalCase — хорошо. / Mostly good camelCase/PascalCase discipline; avoid very short names in production-grade samples.

> Итог / Verdict: после исправления трёх блокирующих ошибок компиляции (L01, L03, L06) и editorial-правки в L05 модуль M02 готов к публикации. / After fixing the three blocking compile errors (L01, L03, L06) and the editorial fix in L05, module M02 is ready for release.
