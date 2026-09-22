---
[← Предыдущий: M02-L05](lesson-M02-L05-nullable.md) | [⬆ К модулю M02](../README.md) | [Следующий: M02-L07 →](lesson-M02-L07-string-stringbuilder.md)
---

### Урок M02-L06: Преобразования типов: явные/неявные, Convert, Parse, TryParse / Type conversions: implicit/explicit, Convert, Parse, TryParse

**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

В C# типы данных делятся на «семьи», и между ними можно переходить — это и есть преобразование типов. Представьте валюту: рубли и копейки — это одно и то же «богатство», и переход между ними безопасен. Но рубли и доллары — это разные единицы, и перевод требует курса и внимания. Так же и с типами: где-то компилятор переводит сам (неявно), а где-то вы должны сказать ему «я знаю, что делаю» (явно).

**Неявные преобразования (implicit).** Компилятор делает их сам, потому что они никогда не теряют данные. Из `int` (32 бита) в `long` (64 бита), из `float` в `double`, из `char` в `int` — приёмник всегда «вмещает» источник целиком. Аналогия: перелить воду из маленького стакана в большой. Ничего не прольётся. Эти преобразования безопасны по определению: не может возникнуть ни потери значения, ни переполнения.

**Явные преобразования (explicit / cast).** Здесь вы ставите скобки с целевым типом: `long big = 1000L; int small = (int)big;`. Почему компилятор не делает это сам? Потому что возможна потеря: большой стакан в маленький — лишнее «прольётся» (старшие биты отбрасываются). Дробный в целый — дробная часть отбрасывается: `(int)3.9` даст `3`, а не `4`. Компилятор требует явного согласия: «да, я готов рискнуть». Это защищает от случайных ошибок.

**Convert.ToInt32.** Класс `Convert` — это «умный переводчик» для базовых типов. В отличие от обычного cast, `Convert.ToInt32(2.7)` округляет по правилам банка (round half to even): `2.5 → 2`, `3.5 → 4`. Для `null` он вернёт `0`, а не выбросит исключение. Но при переполнении `Convert` бросает `OverflowException` — он «знает» границы и не молчит. Используйте `Convert`, когда данные приходят из «чужого мира» (например, из БД как `object`), и вы хотите единое поведение.

**int.Parse.** Переводит строку в число: `int.Parse("42")`. Это «строгий учитель»: если строка кривая (`"abc"`, `""`, `null`) — бросает `FormatException` или `ArgumentNullException`; если число слишком велико — `OverflowException`. `Parse` хорош, когда вы уверены в формате или обернули вызов в `try/catch`. `Parse` — это метод конкретного типа (`int.Parse`, `double.Parse`), он не универсален.

**int.TryParse — безопасный ввод.** Это «дружелюбный учитель»: `bool ok = int.TryParse(input, out int value);`. Никаких исключений — только `true/false` и `out`-параметр. Если `false`, `value` получит `0`. Это идеальный выбор для ввода пользователя: вы не знаете, что он напечатает, и не хотите, чтобы приложение упало. Сравните: `Parse` на `"abc"` роняет программу, а `TryParse` просто скажет «не получилось» и даст шанс переспросить. В C# 7+ можно объявлять `out`-переменную прямо в вызове — лаконично и читаемо.

**Переполнение (overflow).** `int` вмещает значения от −2 147 483 648 до 2 147 483 647. Что будет при `int.MaxValue + 1`? По умолчанию — «молчаливый обход»: значение «оборачивается» и становится большим отрицательным числом. Это не ошибка, это особенности двоичной арифметики. Такой баг коварен: программа работает, но числа неверные — например, баланс счёта превращается в минус.

**Контекст checked.** Ключевое слово `checked` включает проверку переполнения: `checked { int x = int.MaxValue + 1; }` бросит `OverflowException`. Есть и `unchecked` — явно отключает. По умолчанию для констант компилятор сам включает `checked` (и не даст скомпилировать заведомо переполненное выражение), а для переменных во время выполнения — `unchecked`. Настройка `<CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>` в `.csproj` делает `checked` поведением по умолчанию для всего проекта. Правило: используйте `checked` там, где переполнение — это ошибка (финансы, количества), и `unchecked` там, где обход допустим (хеш-функции, побитовые операции).

**Сводка выбора.** Неявно — когда безопасно. Явный cast — когда возможна потеря и вы согласны. `Convert` — для единообразного перевода базовых типов. `Parse` — когда формат гарантирован. `TryParse` — для ненадёжного ввода. `checked` — когда переполнение недопустимо.

#### Theory (EN)

In C#, types belong to "families", and moving between them is called type conversion. Think of currency: cents and dollars are the same "wealth", and switching between them is safe. But dollars and euros are different units — converting requires an exchange rate and care. Types work the same way: sometimes the compiler converts automatically (implicitly), and sometimes you must tell it "I know what I am doing" (explicitly).

**Implicit conversions.** The compiler performs these on its own because they never lose data. From `int` (32 bits) to `long` (64 bits), from `float` to `double`, from `char` to `int` — the destination always fully contains the source. Analogy: pour water from a small glass into a large one. Nothing spills. These conversions are safe by definition: no value loss and no overflow can occur.

**Explicit conversions (cast).** Here you put the target type in parentheses: `long big = 1000L; int small = (int)big;`. Why won't the compiler do this itself? Because loss is possible: a large glass into a small one — the excess "spills" (high bits are discarded). A fractional into an integral — the fractional part is truncated: `(int)3.9` yields `3`, not `4`. The compiler demands explicit consent: "yes, I accept the risk." This guards against accidental mistakes.

**Convert.ToInt32.** The `Convert` class is a "smart translator" for base types. Unlike a plain cast, `Convert.ToInt32(2.7)` rounds using banker's rounding (round half to even): `2.5 → 2`, `3.5 → 4`. For `null` it returns `0` instead of throwing. But on overflow `Convert` throws `OverflowException` — it "knows" the bounds and does not stay silent. Use `Convert` when data arrives from a "foreign world" (for example, from a database as `object`) and you want consistent behavior.

**int.Parse.** Converts a string to a number: `int.Parse("42")`. It is a "strict teacher": if the string is malformed (`"abc"`, `""`, `null`), it throws `FormatException` or `ArgumentNullException`; if the number is too large — `OverflowException`. `Parse` is good when you trust the format or wrap the call in `try/catch`. `Parse` is a method of a specific type (`int.Parse`, `double.Parse`); it is not universal.

**int.TryParse — safe input.** This is the "friendly teacher": `bool ok = int.TryParse(input, out int value);`. No exceptions — only `true/false` and an `out` parameter. If `false`, `value` receives `0`. This is the ideal choice for user input: you never know what the user typed, and you do not want the application to crash. Compare: `Parse` on `"abc"` crashes the program, while `TryParse` simply says "did not work" and lets you ask again. Since C# 7 you can declare the `out` variable right in the call — concise and readable.

**Overflow.** An `int` holds values from −2,147,483,648 to 2,147,483,647. What happens at `int.MaxValue + 1`? By default — a "silent wraparound": the value "wraps" and becomes a large negative number. This is not an error; it is how binary arithmetic works. Such a bug is insidious: the program runs, but the numbers are wrong — for example, an account balance flips to negative.

**The checked context.** The `checked` keyword turns overflow checking on: `checked { int x = int.MaxValue + 1; }` throws `OverflowException`. There is also `unchecked`, which explicitly disables it. By default, the compiler applies `checked` to constant expressions (and refuses to compile a clearly overflowing expression), but `unchecked` to variables at runtime. The `<CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>` setting in `.csproj` makes `checked` the default for the whole project. Rule of thumb: use `checked` where overflow is an error (finance, quantities) and `unchecked` where wraparound is acceptable (hash functions, bitwise operations).

**Choosing at a glance.** Implicit — when safe. Explicit cast — when loss is possible and you accept it. `Convert` — for uniform translation of base types. `Parse` — when the format is guaranteed. `TryParse` — for unreliable input. `checked` — when overflow is unacceptable.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — top-level statements
// Демонстрация преобразований типов / Type conversion demo

using System;

// 1) Неявное преобразование: малый тип → большой, потерь нет / Implicit: small -> large, no loss
int smallInt = 42;
long bigLong = smallInt;          // компилятор делает сам / compiler does it automatically
double fromInt = smallInt;        // int -> double тоже безопасно / int -> double is safe too
Console.WriteLine($"Неявно / Implicit: {smallInt} -> long {bigLong}, double {fromInt}");

// 2) Явное преобразование (cast): возможна потеря / Explicit cast: loss possible
double pi = 3.99;
int truncated = (int)pi;          // дробная часть отбрасывается -> 3 / fractional part discarded -> 3
Console.WriteLine($"Cast double->int: {pi} -> {truncated}");

long huge = 5_000_000_000L;
int narrowed = (int)huge;         // старшие биты теряются / high bits lost
Console.WriteLine($"Cast long->int: {huge} -> {narrowed} (потеря / loss)");

// 3) Convert: «умный» перевод базовых типов / Convert: smart translation of base types
double half = 2.5;
int rounded = Convert.ToInt32(half); // banker's rounding -> 2 / банковское округление -> 2
Console.WriteLine($"Convert.ToInt32({half}) = {rounded} (round half to even)");

object? maybeNull = null;
int fromNull = Convert.ToInt32(maybeNull); // -> 0, без исключения / -> 0, no exception
Console.WriteLine($"Convert из null / from null: {fromNull}");

// 4) Parse: строго, бросает исключения / Parse: strict, throws on bad input
string goodNumber = "12345";
int parsed = int.Parse(goodNumber);
Console.WriteLine($"int.Parse(\"{goodNumber}\") = {parsed}");

try
{
    int bad = int.Parse("abc"); // FormatException
}
catch (FormatException ex)
{
    Console.WriteLine($"Parse ошибся / Parse failed: {ex.Message}");
}

// 5) TryParse: безопасный пользовательский ввод / TryParse: safe user input
//    out-переменная объявляется прямо в вызове (C# 7+) / out var declared inline
Console.Write("Введите возраст / Enter age: ");
string? input = Console.ReadLine();

if (int.TryParse(input, out int age))
{
    Console.WriteLine($"ОК, возраст = {age} / OK, age = {age}");
}
else
{
    Console.WriteLine($"Не похоже на число / Not a number: '{input}'");
}

// 6) Переполнение: молчаливый обход vs checked / Overflow: silent wrap vs checked
int max = int.MaxValue;
int wrapped = max + 1;            // по умолчанию unchecked -> отрицательное / default unchecked -> negative
Console.WriteLine($"Без checked / Unchecked: {max} + 1 = {wrapped}");

try
{
    // checked(...) — это выражение (возвращает значение). / checked(...) is an expression (returns a value).
    // checked { ... } был бы блоком-инструкцией и не вернул бы значение — нельзя присвоить.
    int guarded = checked(max + 1); // C# 12: checked-выражение / checked expression
    _ = guarded;
}
catch (OverflowException ex)
{
    Console.WriteLine($"checked бросил / checked threw: {ex.Message}");
}

// 7) Pattern matching для безопасного приведения ссылочных типов / Pattern matching for safe reference cast
object boxed = "hello";
if (boxed is string text)
{
    Console.WriteLine($"Pattern match: это строка длины {text.Length} / it's a string of length {text.Length}");
}

// 8) Попробуйте сами: чтение нескольких чисел через TryParse в цикле / Try it: read several numbers safely
int sum = 0;
for (int i = 0; i < 3; i++)
{
    Console.Write($"Число {i + 1}/3 / Number {i + 1}/3: ");
    if (int.TryParse(Console.ReadLine(), out int n) && n >= 0)
    {
        sum += n;
    }
}
Console.WriteLine($"Сумма корректных вводов / Sum of valid inputs: {sum}");
```

#### Best Practices

- Предпочитайте `TryParse` для любого ввода, который приходит от пользователя или из внешнего источника — это убирает целый класс исключений «по контролируемому потоку».
- Prefer `TryParse` for any input that originates from a user or an external source — it removes a whole class of "control-flow via exceptions".
- Используйте `checked` (или `<CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>`) в доменах, где переполнение — это бизнес-ошибка: финансы, количества, координаты.
- Use `checked` (or `<CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>`) in domains where overflow is a business error: finance, quantities, coordinates.
- Не полагайтесь на «молчаливый обход» как на норму: если обход допустим (хеши, битовые маски), помечайте это явно `unchecked`, чтобы читатель видел намерение.
- Do not rely on "silent wraparound" as the norm: if wraparound is acceptable (hashes, bitmasks), mark it explicitly with `unchecked` so the reader sees the intent.
- Для приведения по иерархии типов используйте pattern matching `is Type t` вместо классического `as` + `null`-проверки — это компактнее и безопаснее.
- For casts along a type hierarchy prefer pattern matching `is Type t` over the classic `as` + `null`-check — it is more compact and safer.
- Валидируйте и нормализуйте ввод до приведения: уберите пробелы, разделители тысяч, выберите `CultureInfo` — `TryParse(input, out var n, CultureInfo.InvariantCulture)` предсказуем на любой локали.
- Validate and normalize input before conversion: trim spaces, remove thousands separators, pick a `CultureInfo` — `TryParse(input, out var n, CultureInfo.InvariantCulture)` is predictable across locales.

#### Частые ошибки / Common Mistakes

- Использовать `int.Parse(userInput)` без `try/catch` → программу «роняет» любой нечисловой ввод; замените на `TryParse` и сообщайте пользователю понятную ошибку. (RU)
- Ожидать, что `(int)2.9` даст `3` → cast всегда отбрасывает дробную часть и даёт `2`; для округления используйте `Convert.ToInt32` или `Math.Round`. (RU)
- Забывать про переполнение при сложении `int` в циклах и суммах → итог «уезжает» в отрицательные числа; включайте `checked` для финансовых расчётов. (RU)
- Путать `Convert.ToInt32(null)` (вернёт `0`) и `int.Parse(null)` (бросит `ArgumentNullException`) → выбирайте инструмент по контракту, а не по привычке. (RU)
- Using `int.Parse(userInput)` without `try/catch` → any non-numeric input crashes the app; switch to `TryParse` and show the user a clear message. (EN)
- Expecting `(int)2.9` to yield `3` → cast always truncates the fraction and yields `2`; use `Convert.ToInt32` or `Math.Round` for rounding. (EN)
- Forgetting overflow when adding `int` in loops and sums → the total "slides" into negative values; enable `checked` for financial math. (EN)
- Confusing `Convert.ToInt32(null)` (returns `0`) with `int.Parse(null)` (throws `ArgumentNullException`) → pick the tool by its contract, not by habit. (EN)

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я объясняю разницу между неявным и явным преобразованием и знаю, почему компилятор требует cast. (RU)
- [ ] Я выбираю `TryParse` для пользовательского ввода и могу показать корректную обработку `false`. (RU)
- [ ] Я понимаю, что `Convert.ToInt32` округляет, а cast `double -> int` — отбрасывает. (RU)
- [ ] Я знаю, когда применять `checked` и как включить его на уровне проекта. (RU)
- [ ] Я не путаю `Parse` (бросает) и `TryParse` (не бросает) и их контракты. (RU)
- [ ] I can explain the difference between implicit and explicit conversion and why the compiler requires a cast. (EN)
- [ ] I choose `TryParse` for user input and can demonstrate correct handling of `false`. (EN)
- [ ] I understand that `Convert.ToInt32` rounds while a `double -> int` cast truncates. (EN)
- [ ] I know when to apply `checked` and how to enable it at the project level. (EN)
- [ ] I do not confuse `Parse` (throws) and `TryParse` (does not throw) and their contracts. (EN)

#### Ресурсы / Resources

- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/programming-guide/types/casting-and-type-conversions — Приведение и преобразование типов (C#) / Casting and Type Conversions (C#)
- Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.int32.tryparse — Int32.TryParse Method / Метод Int32.TryParse
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/statements/checked-and-unchecked — Операторы checked и unchecked / checked and unchecked statements
- Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.convert.toint32 — Convert.ToInt32 Method / Метод Convert.ToInt32

---
[← Предыдущий: M02-L05](lesson-M02-L05-nullable.md) | [⬆ К модулю M02](../README.md) | [Следующий: M02-L07 →](lesson-M02-L07-string-stringbuilder.md)
---
