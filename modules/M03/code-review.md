# Ревью кода модуля M03 / Code Review: M03
# Управление потоком, методы / Control flow, methods

> Ревьюер / Reviewer: Агент 5 (Code Reviewer)
> Проверено уроков / Lessons reviewed: 8
> Целевая платформа / Target: C# 12 / .NET 8

## Сводка / Summary
- **Готово / Ready:** 6/8 уроков (L01, L02, L04, L05, L06, L08) — компилируются и работают на C# 12 / .NET 8.
- **Нужно исправить / Needs fix:**
  - **M03-L03** — отсутствует `using System.Runtime.InteropServices;` (для `CollectionsMarshal.AsSpan`) и `using System.Linq;` (для `.Select`). Код не компилируется.
  - **M03-L07** — отсутствует `using System.Linq;` (для `.Sum()`, `.Select()`, `.SelectMany()`, `.ToArray()`). Код не компилируется.
  - **M03-L02** — опечатка в имени типа `ElectronProduct` (должно быть `ElectronicProduct`); пример `GradeToLabelClassic` несогласован (только ветка `'A'` печатает в консоль).
  - **M03-L04** — содержательный пробел: ключевое слово `break` ни разу не встречается в коде, хотя это заголовок урока.
- **Устаревает / Deprecated:** нет устаревшего API. Мелкие замечания по актуальности: `when`-фильтры в L02 можно заменить на property-patterns (C# 9+, более идиоматично для C# 12).
- **Общая оценка / Overall:** Материал методически сильный, двуязычный, аккуратно структурированный; подавляющее большинство примеров современные и компилируемые. Главная проблема — два урока (L03, L07) не собираются из-за пропущенных `using`. После исправления usings и пары мелочей модуль готов.

---

## Детальный разбор по урокам / Per-lesson review

### M03-L01 — if/else, тернарный оператор
#### ✅ Что ок / What's good
- Современный, чистый код: guard clauses, pattern matching `is { ... }`, switch expression, record `User` с `init`/nullable `Email`. / Modern, clean code: guard clauses, `is { ... }` pattern matching, switch expression, `User` record.
- Хороший параллельный пример `Classify` (switch) и `ClassifyWithIf` (if/else) для сравнения. / Nice side-by-side `Classify` vs `ClassifyWithIf`.
- Корректное использование short-circuit `&&` с nullable `Email` (строка 74). / Correct short-circuit `&&` with nullable `Email` (line 74).

#### ⚠️ Что улучшить / What to improve
- **Строки 96–108 (`ClassifyWithIf`):** `else if` после `return` — избыточный `else` (else-after-return smell). → Уберите `else`, оставьте просто `if (...) return ...;`. Это особенно важно в уроке про best practices. / **Lines 96–108:** `else if` after `return` is redundant (else-after-return). Drop `else`, use plain `if`.
- **Строка 112 (`CanVote`):** параметр `User u` без `?`, но метод использует `is { ... }` (null-безопасная проверка). Для консистентности с `Describe(User? user)` сделайте `User? u`. / **Line 112:** `User u` non-nullable but uses null-safe `is { ... }`. Make it `User?` for consistency with `Describe`.
- **Строка 74:** `user.Email is not null && user.Email.Contains('@')` обращается к `user.Email` дважды. → Современнее: `user.Email is { } email && email.Contains('@')` или `user.Email is string e && e.Contains('@')`. / **Line 74:** double access to `user.Email`. Use `is { } email` binding.

#### 🚫 Code smells
- Else-after-return в `ClassifyWithIf` (см. выше). / Else-after-return in `ClassifyWithIf`.
- Незначительная NRT-несогласованность `CanVote(User u)`. / Minor NRT inconsistency in `CanVote`.

#### 🏷️ Статус
- **Готово / Ready** (с мелкими правками). / Ready with minor tweaks.

---

### M03-L02 — switch, switch expressions, pattern matching
#### ✅ Что ок / What's good
- Покрыты все ключевые виды паттернов: константный, типа, свойств, кортежный, `or`, `_`. / All major pattern kinds covered.
- Правильный порядок «от специфичного к общему» в `Price` и `Tax`. / Correct specific-to-general arm ordering.
- Хороший пример `Describe(int x, int y)` с кортежным switch и `when`. / Good tuple-switch with `when`.

#### ⚠️ Что улучшить / What to improve
- **Строки 104, 106, 134:** имя типа `ElectronProduct` — опечатка (пропущена `ic`). → Переименуйте в `ElectronicProduct`. Компилируется, но режет глаз и учит студентов неправильному слову. / **Lines 104/106/134:** typo `ElectronProduct` → `ElectronicProduct`.
- **Строки 64–83 (`GradeToLabelClassic`):** только ветка `'A'` вызывает `Console.WriteLine("Отлично...")`, остальные молча возвращают строку. → Либо уберите побочный эффект из `'A'`, либо добавьте его во все ветки для согласованности. / **Lines 64–83:** only `'A'` prints; inconsistent side effect.
- **Строка 102:** `Book b when b.IsUsed` — `when`-фильтр уместен, но в C# 9+ идиоматичнее property-pattern `Book { IsUsed: true } b`. → Покажите оба варианта или замените. / **Line 102:** prefer `Book { IsUsed: true } b` property pattern over `when`.
- **Строка 99 (`Price(object item)`):** параметр `object` — code smell в реальном коде. → Достаточно для демо паттернов типа, но добавьте примечание, что в продакшене используют полиморфизм/интерфейсы/визитор. / **Line 99:** `object` param is a smell in production; note this in a comment.

#### 🚫 Code smells
- `object` как тип параметра (приемлемо для демо, но стоит оговорить). / `object` param (acceptable for demo).
- Несогласованный побочный эффект в классическом switch. / Inconsistent side effect in classic switch.

#### 🏷️ Статус
- **Needs fix** (опечатка + согласованность). / Needs fix (typo + consistency).

---

### M03-L03 — Циклы for/while/do-while/foreach
#### ✅ Что ок / What's good
- Чёткая демонстрация всех четырёх циклов. / Clear demo of all four loops.
- Отличная секция про безопасное удаление (`RemoveAll`, обратный `for`) с правильным предупреждением о `InvalidOperationException`. / Great safe-removal section.
- Использование pattern matching `guess is < 1 or > 10` (строка 71). / Pattern matching `is < 1 or > 10`.

#### ⚠️ Что улучшить / What to improve
- **🚫 БЛОКИРУЮЩЕЕ / Blocking — строки 101, 105:** пример `foreach (ref var it in CollectionsMarshal.AsSpan(items))` требует `using System.Runtime.InteropServices;`, а `.Select(...)` в строке 105 требует `using System.Linq;`. В файле (строки 43–44) импортированы только `System` и `System.Collections.Generic`. → **Код не компилируется.** Добавьте оба `using`. / **Blocking — lines 101/105:** missing `using System.Runtime.InteropServices;` and `using System.Linq;`. **Does not compile.**
- **Строки 99–105 (`CollectionsMarshal.AsSpan`):** техника продвинутая для урока M03 (interop + `ref foreach` + value-tuple mutation). → Либо вынесите в опциональный блок «для продвинутых», либо замените на более простой пример мутации (например, класс с settable-свойством в `foreach`). / **Lines 99–105:** advanced for M03; consider moving to an optional block or simplifying.
- **Строки 55–61 (`while` + `Console.ReadLine`):** интерактивный ввод затрудняет автотестирование примера. → Добавьте вариант с фиксированным вводом (например, `IEnumerable<string>` источник) для воспроизводимости. / **Lines 55–61:** interactive input hampers testing; provide a deterministic variant.

#### 🚫 Code smells
- Пропущенные `using` (фактическая ошибка компиляции). / Missing `using`s (compile error).
- Слишком «магический» продвинутый пример без оговорки. / Over-advanced example without framing.

#### 🏷️ Статус
- **Needs fix** (компиляция). / Needs fix (compilation).

---

### M03-L04 — break/continue/return
#### ✅ Что ок / What's good
- Ясные демонстрации `continue` (строки 64–76) и `return` как раннего выхода (строки 50–60). / Clear `continue` and early-`return` demos.
- Хороший контраст `return` vs `throw` в `Divide` (строки 106–114) с обработкой `DivideByZeroException`. / Good `return`-vs-`throw` contrast.
- Корректное замечание, что `switch` expression не требует `break` (строка 101). / Correct note that switch expression needs no `break`.

#### ⚠️ Что улучшить / What to improve
- **Содержательный пробел:** заголовок урока — «break/continue/return», но ключевое слово `break` **ни разу** не встречается в коде. `FindFirstLarge` использует `return` (строка 56), `ContainsPair` — `return true` (строка 88). → Добавьте хотя бы один пример с реальным `break` (например, классический `switch` с `break` или линейный поиск с досрочным выходом из цикла). / **Content gap:** `break` never appears despite the title. Add a real `break` example.
- **Строки 78–93 (`ContainsPair`):** комментарий говорит «break in a nested loop + label via goto», но код использует `return true` (что лучше, но противоречит комментарию про `goto`). → Синхронизируйте комментарий и код либо покажите `goto`-вариант как контраст. / **Lines 78–93:** comment mentions `goto`, code uses `return`; align them.
- **Строка 120:** `ContainsPair(new[] { new[] { 1, 2 }, new[] { 101 } })` — вызов выглядит как поиск пары, но метод ищет одно значение в матрице. → Имя `ContainsPair` вводит в заблуждение; лучше `ContainsValue` или `Contains`. / **Line 120:** `ContainsPair` name is misleading; rename to `ContainsValue`.

#### 🚫 Code smells
- Вводящее в заблуждение имя `ContainsPair`. / Misleading `ContainsPair` name.
- Рассогласование комментария и кода в примере с `goto`. / Comment/code mismatch re `goto`.

#### 🏷️ Статус
- **Needs fix** (добавить пример `break`). / Needs fix (add a `break` example).

---

### M03-L05 — Объявление методов, сигнатура, возвращаемые значения
#### ✅ Что ок / What's good
- Чистая иллюстрация `static` vs instance (`OrderService` static, `Order.PrintSummary` instance). / Clean static-vs-instance illustration.
- Перегрузка `CalculatePoints` с разной сигнатурой (строки 103–118). / Overloading by signature.
- Идиоматичный `customer?.IsLoyal == true` для nullable bool (строка 117). / Idiomatic `customer?.IsLoyal == true`.
- `init`-свойства у `Customer`/`Order` (C# 9+). / `init` properties.

#### ⚠️ Что улучшить / What to improve
- **Строки 33–40 (теория, `CalculateDiscount`):** параметр `Customer customer` без `?`, но проверяется `if (customer is null)`. → В теоретическом примере сделайте `Customer?` для NRT-консистентности (в коде урока `Customer?` уже правильный). / **Theory lines 33–40:** `Customer customer` non-nullable but null-checked; use `Customer?`.
- **Строка 132:** `System.Console.WriteLine(...)` с полным именем при отсутствии `using System;`. → Либо добавьте `using System;`, либо везде используйте полное имя. Сейчас стилистически несогласованно. / **Line 132:** fully-qualified `System.Console` without `using System;`; be consistent.
- **Строки 128–135 (`PrintSummary`):** экземплярный `void`-метод, который вычисляет и печатает — мягкое нарушение CQS (запрос + побочный эффект). → Урок сам учит CQS; стоит либо вынести вычисление в отдельный query-метод, либо добавить комментарий, что для демо допускается. / **Lines 128–135:** `void` method computes + prints (mild CQS breach); note or split.

#### 🚫 Code smells
- Лёгкое нарушение CQS в `PrintSummary`. / Mild CQS breach in `PrintSummary`.

#### 🏷️ Статус
- **Готово / Ready** (мелкие стилистические правки). / Ready (minor style tweaks).

---

### M03-L06 — Параметры: ref, out, in, params, значения по умолчанию
#### ✅ Что ок / What's good
- Полное покрытие `ref`/`out`/`in`/`params`/defaults/named с минимальными примерами. / Full coverage of all parameter kinds.
- C# 12 primary constructor в `readonly struct BigPoint(double x, double y)` (строки 78–82). / C# 12 primary constructor.
- Collection expression `Sum([10, 20, 30])` (строка 104) — современно. / Collection expression.
- Деконструкция кортежа в `Swap` (строка 117). / Tuple deconstruction in `Swap`.

#### ⚠️ Что улучшить / What to improve
- **Строки 54, 60:** `void IncrementByValue(int n) => n += 100;` и `IncrementByRef(ref int n) => n += 100;` — expression-bodied `void` с оператором присваивания. Это допустимо, но неидиоматично и может сбить студента (похоже на возврат значения). → Перепишите блочным телом `{ n += 100; }` для ясности. / **Lines 54/60:** expression-bodied `void` with `+=` is unidiomatic; use block body.
- **Строки 54–62:** учебный момент про «по значению — изменение не видно» хороший, но `IncrementByValue` мутирует копию параметра `n` (а не внешний `x`) — студенты могут спутать «копию аргумента» с «копией параметра». → Добавьте комментарий, что `n` — локальная копия параметра. / Clarify that `n` is a local copy of the parameter.
- **Строки 66–71 (`TryParseInt`):** дублирует встроенный `int.TryParse`. → Это нормально для демо `out`, но стоит явно сказать «в реальном коде используйте `int.TryParse`». / **Lines 66–71:** duplicates built-in `int.TryParse`; note this.
- Локальные функции (`IncrementByValue`, `IncrementByRef`, `Swap`, `Analyze`) не захватывают переменные и могут быть `static`. Урок L07 учит этому — стоит показать здесь для преемственности. / Local functions could be `static`; aligns with L07 teaching.

#### 🚫 Code smells
- Expression-bodied `void` с побочным эффектом. / Expression-bodied `void` with side effect.
- Дублирование `int.TryParse`. / Duplicating `int.TryParse`.

#### 🏷️ Статус
- **Готово / Ready** (компилируется; мелкие стилистические правки). / Ready (compiles; minor style).

---

### M03-L07 — Перегрузка методов, область видимости
#### ✅ Что ок / What's good
- Чёткая демонстрация перегрузок по типам/числу параметров и `params` (строки 71–74). / Clear overloads demo.
- Отличный пример `static` локальной функции `Transliterate` (строки 114–124) и замечание про локальные функции, объявленные после `return` (строки 132–139). / Great `static` local function demo + post-return declaration note.
- Демонстрация неоднозначности `Add(1, 2L)` (строки 52–53). / Ambiguity demo.

#### ⚠️ Что улучшить / What to improve
- **🚫 БЛОКИРУЮЩЕЕ / Blocking — строки 74, 127–129:** `values.Sum()`, `lower.Select(...)`, `.SelectMany(...)`, `.ToArray()` требуют `using System.Linq;`. В файле (строка 40) импортирован только `System.Globalization`. → **Код не компилируется.** Добавьте `using System.Linq;`. / **Blocking:** missing `using System.Linq;`. **Does not compile.**
- **Строки 131–139 (`TrimDashes`):** `s.Replace("--", "-", StringComparison.Ordinal)` делает один проход и не схлопывает три и более дефиса подряд (`"---"` → `"--"`). → Для надёжности используйте цикл `while (s.Contains("--"))` или `Regex.Replace(s, "-{2,}", "-")`. На текущем входе работает, но хрупко. / **Lines 131–139:** single-pass `Replace` doesn't collapse 3+ dashes; use a loop or regex.
- **Строки 81, 87:** поле `_count = 0;` в конструкторе избыточно — `int` по умолчанию `0`. → Можно убрать для краткости. / **Lines 81/87:** redundant `_count = 0;`.
- **Строка 76 (закомментировано):** `// public long Add(int a, long b) ...` — хороший комментарий про неоднозначность, но стоит добавить, что её можно разрешить перегрузкой `(int, long)` ИЛИ `(long, int)`, если убрать `(double, double)`. / Expand the ambiguity comment.

#### 🚫 Code smells
- Пропущенный `using System.Linq;` (ошибка компиляции). / Missing `using System.Linq;` (compile error).
- Хрупкая замена `--` без цикла. / Fragile dash-collapse.

#### 🏷️ Статус
- **Needs fix** (компиляция). / Needs fix (compilation).

---

### M03-L08 — Рекурсия и tail-оптимизация
#### ✅ Что ок / What's good
- Сбалансированный набор: наивная, хвостовая и итеративная формы факториала; наивный и итеративный Фибоначчи; рекурсивный и итеративный обход дерева. / Balanced set: naive/tail/iterative factorial; naive/iterative Fibonacci; recursive/iterative tree traversal.
- Корректное и важное предупреждение: C# не гарантирует TCO (строки 19, 55). / Correct, important warning: C# does not guarantee TCO.
- Tuple deconstruction `(prev, curr) = (curr, prev + curr)` (строка 82). / Tuple deconstruction.
- Хороший комментарий про `StackOverflowException` как неперехватываемое (строки 132–135). / Good note on uncatchable `StackOverflowException`.

#### ⚠️ Что улучшить / What to improve
- **Строка 138:** `sealed record Node(int Value, Node? Left = null, Node? Right = null);` объявлен после использования (строки 90, 94, 105). Компилируется, но для читателя лучше объявить тип до использования. → Перенесите `Node` выше примеров. / **Line 138:** `Node` declared after use; move it up for readability.
- **Строки 94–101 (`InOrderRecursive`):** локальная функция, принимающая `List<int> acc` и возвращающая его — мутация переданного списка. → Стоит прокомментировать, что это_side-effectful_ подход; альтернатива — возвращать новый список через `Concat`. / **Lines 94–101:** mutates passed `acc` list; note this or use immutable concat.
- **Строка 50 (`FactorialRecursive`):** `n <= 0 ? 1` возвращает 1 для отрицательных `n` — математически спорно (факториал отрицательного не определён). → Для демо допустимо, но в Best Practices стоит явно сказать «валидируйте вход для отрицательных». / **Line 50:** `n <= 0 ? 1` returns 1 for negatives; note input validation.
- **Строки 105–122 (`InOrderIterative`):** `var stack = new Stack<Node>();` — non-nullable `Node` в дженерике, но туда кладутся значения, которые могут быть `null`? Нет, `Push(current)` только когда `current is not null` (строка 113). Тут `Stack<Node>` корректен. → Но тип `Node` объявлен как `Node? Left/Right`, а сам `Node` non-nullable — всё согласовано. ОК. / OK.

#### 🚫 Code smells
- Мутация переданного `acc` в `InOrderRecursive` (мягкий). / Mutation of passed `acc` (mild).
- Объявление `Node` после использования (стиль). / `Node` declared after use (style).

#### 🏷️ Статус
- **Готово / Ready** (мелкие правки). / Ready (minor tweaks).

---

## Общие рекомендации по модулю / Module-wide recommendations
- **Единообразие `using`-директив / `using` consistency:** в нескольких уроках (L03, L07) пропущены `using System.Linq;` / `System.Runtime.InteropServices;`. Заведите общий шаблон заголовка для всех уроков модуля и проверяйте компиляцию `dotnet build` перед публикацией. / Establish a shared header template and run `dotnet build` before publishing.
- **NRT-консистентность / NRT consistency:** там, где методы используют `is null` / `is { ... }`, параметр должен быть `T?`. Проверьте `CanVote(User u)` (L01) и теоретический `CalculateDiscount` (L05). / Where methods null-check, make params `T?`.
- **Else-after-return:** избегайте `else` после `return` в учебных примерах по best practices (L01 `ClassifyWithIf`). / Avoid `else` after `return` in best-practices lessons.
- **Согласованность комментариев и кода:** убедитесь, что заголовки секций и текст совпадают с фактическим кодом (L04: секция про `break`/`goto` без `break`/`goto`; L02: `Console.WriteLine` только в `'A'`). / Align comments/section headers with code.
- **Именование:** PascalCase для типов и методов выдержан; исправить опечатку `ElectronProduct` → `ElectronicProduct` (L02). Избегайте вводящих в заблуждение имён вроде `ContainsPair` для поиска одного значения (L04). / Fix `ElectronProduct` typo; rename misleading `ContainsPair`.
- **Актуальность C# 12:** преимущественно используется хорошо (primary constructors, collection expressions, target-typed `new`, `init`, switch expressions). Местами можно осовременить: `when`-фильтры → property patterns (L02); expression-bodied `void` с побочным эффектом → блочное тело (L06). / Mostly good C# 12 usage; modernize `when` filters and `void` expression bodies where possible.
- **Тестируемость примеров:** интерактивные примеры с `Console.ReadLine` (L03) лучше дублировать детерминированным вариантом для автотестов. / Provide deterministic variants alongside interactive examples.
- **CQS:** в уроках, обучающих CQS (L05), избегайте `void`-методов, вычисляющих значения; это противоречит собственному правилу. / In CQS lessons, avoid computing `void` methods.
