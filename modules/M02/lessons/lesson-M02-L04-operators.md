---
[← Предыдущий: M02-L03](lesson-M02-L03-var-const.md) | [⬆ К модулю M02](../README.md) | [Следующий: M02-L05 →](lesson-M02-L05-nullable.md)
---

### Урок M02-L04: Операторы: арифметика, сравнение, логика, побитовые / Operators: arithmetic, comparison, logical, bitwise

**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Оператор — это символ или ключевое слово, которое говорит компилятору, какую операцию выполнить над операндами. В C# операторы разбиты на группы; мы пройдём по каждой и свяжем их единой аналогией — **кухней ресторана**.

**Арифметические операторы** — это «базовые поварские движения». `+`, `-`, `*`, `/` работают как в математике, но с важным нюансом: деление целых чисел всегда целочисленное (`7 / 2 == 3`, потому что дробная часть отбрасывается, а не округляется). Остаток от деления даёт `%` (`7 % 2 == 1`) — удобно для проверки чётности и циклических вычислений. Инкремент `++` и декремент `--` бывают префиксные (`++x` — сначала увеличь, потом используй) и постфиксные (`x++` — сначала используй, потом увеличь). Разница критична в выражениях: `int a = 5; int b = a++;` даёт `b == 5`, а `int b = ++a;` даёт `b == 6`.

**Операторы сравнения** (`==`, `!=`, `<`, `>`, `<=`, `>=`) — это «контролёр качества», который сравнивает две тарелки и выдаёт вердикт `true`/`false`. Для строк `==` сравнивает значения, а не ссылки, что часто удивляет программистов с C++ фоном.

**Логические операторы** — «логика повара»: `&&` (И — все условия должны выполняться), `||` (ИЛИ — хотя бы одно), `!` (НЕ — инверсия). Важное свойство: `&&` и `||` **закорочены** (short-circuit). Если левый операнд `&&` уже `false`, правый не вычисляется — это защищает от `NullReferenceException`: `if (obj != null && obj.IsValid)` безопасно.

**Побитовые операторы** работают с битами числа напрямую, как с рядом лампочек. `&` (И), `|` (ИЛИ), `^` (исключающее ИЛИ — XOR), `~` (побитовое НЕ, инверсия всех битов), `<<` и `>>` (сдвиг влево/вправо). Сдвиг влево на 1 — это умножение на 2, вправо — деление на 2. Побитовые операции незаменимы для флагов и масок: `Permissions.Read | Permissions.Write` включает оба права, `flags & Permissions.Read` проверяет наличие.

**Тернарный оператор** `условие ? знач1 : знач2` — компактный выбор: «если тарелка пуста — подать суп, иначе — десерт». Удобен, но не злоупотребляйте вложенностью.

**Null-coalescing** `??` возвращает левый операнд, если он не `null`, иначе правый. `??=` присваивает только когда левая часть `null`. Это «страхующий су-шеф»: `name ?? "Гость"` никогда не вернёт null.

**Приоритет операторов** определяет порядок вычислений. Умножение важнее сложения, `!` важнее `&&`, которое важнее `||`. Когда сомневаетесь — ставьте скобки: они бесплатны и повышают читаемость.

**checked/unchecked** управляют переполнением. По умолчанию арифметика целых чисел в C# «молча» переполняется (`int.MaxValue + 1` даёт отрицательное число). Блок `checked { ... }` включает проверку и бросает `OverflowException`. Используйте его там, где переполнение — ошибка, а не ожидаемое поведение.

#### Theory (EN)

An operator is a symbol or keyword that tells the compiler which operation to perform on operands. In C#, operators are grouped into families; we will walk through each one using a single analogy — **a restaurant kitchen**.

**Arithmetic operators** are the “basic knife skills.” `+`, `-`, `*`, `/` behave as in math, but with a crucial catch: integer division is always integral (`7 / 2 == 3`, because the fractional part is truncated, not rounded). The remainder operator `%` gives the leftover (`7 % 2 == 1`) — handy for parity checks and cyclic computations. Increment `++` and decrement `--` come in prefix (`++x` — increment first, then use) and postfix (`x++` — use first, then increment) forms. The difference matters inside expressions: `int a = 5; int b = a++;` yields `b == 5`, while `int b = ++a;` yields `b == 6`.

**Comparison operators** (`==`, `!=`, `<`, `>`, `<=`, `>=`) are the “quality inspector” that compares two plates and returns a `true`/`false` verdict. For strings, `==` compares values, not references — a frequent surprise for programmers with a C++ background.

**Logical operators** are the “chef’s reasoning”: `&&` (AND — all conditions must hold), `||` (OR — at least one), `!` (NOT — inversion). The key property: `&&` and `||` are **short-circuit**. If the left operand of `&&` is already `false`, the right is never evaluated — this guards against `NullReferenceException`: `if (obj != null && obj.IsValid)` is safe.

**Bitwise operators** work on the bits of a number directly, like a row of light switches. `&` (AND), `|` (OR), `^` (exclusive OR — XOR), `~` (bitwise NOT, inverts every bit), `<<` and `>>` (shift left/right). A left shift by 1 is multiplication by 2; a right shift is integer division by 2. Bitwise ops are indispensable for flags and masks: `Permissions.Read | Permissions.Write` enables both rights, `flags & Permissions.Read` tests presence.

**The ternary operator** `condition ? val1 : val2` is a compact choice: “if the plate is empty — serve soup, else — dessert.” Convenient, but avoid deep nesting.

**Null-coalescing** `??` returns the left operand if it is not `null`, otherwise the right. `??=` assigns only when the left side is `null`. This is the “backup sous-chef”: `name ?? "Guest"` never returns null.

**Operator precedence** decides evaluation order. Multiplication binds tighter than addition, `!` binds tighter than `&&`, which binds tighter than `||`. When in doubt — add parentheses: they are free and improve readability.

**checked/unchecked** control overflow. By default, integer arithmetic in C# silently overflows (`int.MaxValue + 1` yields a negative number). A `checked { ... }` block turns on checking and throws `OverflowException`. Use it where overflow is a bug, not expected behavior.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — top-level statements / top-level инструкции
using System;

// 1. Арифметика и инкремент / Arithmetic and increment
int a = 7, b = 2;
Console.WriteLine($"7 / 2 = {a / b}");          // 3 — целочисленное деление / integer division
Console.WriteLine($"7 % 2 = {a % b}");          // 1 — остаток / remainder
Console.WriteLine($"7.0 / 2 = {7.0 / b}");      // 3.5 — дробный результат / fractional result

int counter = 5;
int post = counter++;   // post = 5, counter = 6 (сначала используем / use first)
int pre  = ++counter;   // pre  = 7, counter = 7 (сначала увеличиваем / increment first)
Console.WriteLine($"post={post}, pre={pre}, counter={counter}");

// 2. Сравнения и логика / Comparison and logic
string? name = null;
bool hasName = name != null && name.Length > 0;   // short-circuit: безопасно / safe
Console.WriteLine($"hasName={hasName}");

int age = 20;
bool canDrive = age >= 18 && age <= 75;           // И / AND
bool discount = age < 18 || age >= 65;            // ИЛИ / OR
Console.WriteLine($"canDrive={canDrive}, discount={discount}");

// 3. Побитовые операции и флаги / Bitwise ops and flags
[Flags] enum Permissions { None = 0, Read = 1, Write = 2, Execute = 4 }

Permissions perms = Permissions.Read | Permissions.Write;  // включаем оба / set both
bool canWrite = (perms & Permissions.Write) != 0;          // проверяем флаг / test flag
perms ^= Permissions.Write;                                // XOR — переключаем / toggle
Console.WriteLine($"perms={perms}, canWrite={canWrite}");

int x = 1;
Console.WriteLine($"x << 3 = {x << 3}");   // 8 — сдвиг = умножение на 2^n / shift = multiply by 2^n
Console.WriteLine($"~x = {~x}");           // -2 — инверсия битов / bitwise NOT

// 4. Тернарный и null-coalescing / Ternary and null-coalescing
string displayName = name ?? "Гость / Guest";   // если null — запасное значение / fallback
int score = 75;
string grade = score >= 90 ? "A" : score >= 60 ? "C" : "F";   // вложенность допустима, но умеренно
Console.WriteLine($"displayName={displayName}, grade={grade}");

List<string>? items = null;
items ??= new();                                 // присваиваем только если null / assign only if null
items.Add("first");
Console.WriteLine($"items count={items.Count}");

// 5. checked / unchecked — контроль переполнения / overflow control
int max = int.MaxValue;
Console.WriteLine($"unchecked overflow: {max + 1}");   // молча переполняется / silent overflow
try
{
    int overflow = checked(max + 1);   // бросает OverflowException / throws
}
catch (OverflowException)
{
    Console.WriteLine("checked: переполнение обнаружено / overflow detected");
}

// 6. Приоритет: скобки делают намерение явным / Precedence: parentheses make intent explicit
bool result = !(a > b && b > 0) || (a + b == 9);
Console.WriteLine($"result={result}");
```

#### Best Practices

- Используйте скобки для ясности порядка вычислений, даже когда приоритет и так сработает верно — это документирует намерение для следующего разработчика.
- Предпочитайте `&&`/`||` одиночным `&`/`|` в логических условиях: short-circuit экономит вычисления и защищает от null.
- Применяйте `[Flags]` enum + побитовые операции для набора переключаемых опций вместо нескольких `bool` полей.
- Включайте `checked` (или `unchecked` явно для горячих путей) там, где переполнение семантически важно; для финансовой арифметики используйте `decimal`.
- Use parentheses to clarify evaluation order, even when precedence would already work — it documents intent for the next developer.
- Prefer `&&`/`||` over single `&`/`|` in logical conditions: short-circuit saves work and guards against null.
- Use `[Flags]` enums with bitwise ops for a set of toggleable options instead of several `bool` fields.
- Turn on `checked` (or `unchecked` explicitly for hot paths) where overflow is semantically meaningful; use `decimal` for financial math.

#### Частые ошибки / Common Mistakes

- Целочисленное деление `7 / 2 == 3` вместо `3.5` → приводите один операнд к `double` (`7.0 / 2`) или умножайте на `1.0`.
- Путаница `a++` vs `++a` в выражениях → не используйте инкремент внутри сложных выражений; выносите на отдельную строку.
- `&` вместо `&&` без short-circuit вызывает `NullReferenceException` → в логических условиях всегда `&&`/`||`.
- Ожидание, что `int.MaxValue + 1` бросит исключение → по умолчанию оно молча переполняется; явно используйте `checked`.
- Сравнение строк через `ReferenceEquals` вместо `==` → `==` сравнивает значения для `string`.
- Integer division `7 / 2 == 3` instead of `3.5` → cast one operand to `double` (`7.0 / 2`) or multiply by `1.0`.
- Confusing `a++` vs `++a` inside expressions → don’t embed increment in complex expressions; put it on its own line.
- Using `&` instead of `&&` without short-circuit triggers `NullReferenceException` → always use `&&`/`||` in logical conditions.
- Expecting `int.MaxValue + 1` to throw → by default it silently overflows; use `checked` explicitly.
- Comparing strings with `ReferenceEquals` instead of `==` → `==` compares values for `string`.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я понимаю, что `7 / 2` даёт `3`, а `7.0 / 2` — `3.5`.
- [ ] Я различаю префиксный и постфиксный инкремент и не смешиваю их в выражениях.
- [ ] Я знаю, что `&&` и `||` short-circuit и защищают от null.
- [ ] Я могу использовать `&`, `|`, `^`, `~`, `<<`, `>>` для флагов и масок.
- [ ] Я применяю `?:`, `??` и `??=` для компактной обработки null и выбора.
- [ ] Я понимаю приоритет операторов и ставлю скобки для ясности.
- [ ] Я знаю, когда нужен `checked`, чтобы поймать переполнение.
- [ ] I understand that `7 / 2` yields `3` while `7.0 / 2` yields `3.5`.
- [ ] I distinguish prefix and postfix increment and don’t mix them inside expressions.
- [ ] I know that `&&` and `||` short-circuit and guard against null.
- [ ] I can use `&`, `|`, `^`, `~`, `<<`, `>>` for flags and masks.
- [ ] I apply `?:`, `??`, and `??=` for compact null handling and selection.
- [ ] I understand operator precedence and add parentheses for clarity.
- [ ] I know when `checked` is needed to catch overflow.

#### Ресурсы / Resources

- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/operators/ — Операторы C# (справочник) / C# operators (reference)
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/operators/arithmetic-operators — Арифметические операторы C# / Arithmetic operators in C#

---
[← Предыдущий: M02-L03](lesson-M02-L03-var-const.md) | [⬆ К модулю M02](../README.md) | [Следующий: M02-L05 →](lesson-M02-L05-nullable.md)
---
