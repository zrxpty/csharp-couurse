---
[← К уроку M02-L06](lesson-M02-L06-type-conversion.md) | [⬆ К модулю M02](../README.md) | [Следующее ДЗ →](homework-M02-L07-string-stringbuilder.md)
---

### Домашнее задание M02-L06: Преобразования типов: явные/неявные, Convert, Parse, TryParse / Homework M02-L06: Type conversions: implicit/explicit, Convert, Parse, TryParse

**Урок / Lesson:** M02-L06
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться осознанно выбирать механизм преобразования типов в C# 12 / .NET 8: неявные и явные приведения, `Convert`, `Parse`, `TryParse`, контекст `checked`/`unchecked`, а также безопасное приведение ссылочных типов через pattern matching. Закрепить понимание того, когда преобразование теряет данные, когда бросает исключение, а когда молча возвращает значение по умолчанию, и научиться писать устойчивый к вводу пользователя код. (EN) Learn to consciously choose a type-conversion mechanism in C# 12 / .NET 8: implicit and explicit casts, `Convert`, `Parse`, `TryParse`, the `checked`/`unchecked` context, and safe reference casts via pattern matching. Reinforce the understanding of when a conversion loses data, when it throws, and when it silently returns a default, and learn to write input-resilient code.

#### Связь с уроком / Connection to the lesson
(RU) ДЗ напрямую опирается на все ключевые концепции урока M02-L06: неявные и явные преобразования, поведение `Convert.ToInt32` с банковским округлением и `null`, контракты `Parse` и `TryParse`, молчаливое переполнение и контекст `checked`, а также pattern matching `is Type t` для безопасного приведения. Вы будете повторять примеры урока, но в форме связной задачи — мини-приложения для обработки ввода пользователя.
(EN) The homework builds directly on every key concept of lesson M02-L06: implicit and explicit conversions, `Convert.ToInt32` behavior with banker's rounding and `null`, the contracts of `Parse` and `TryParse`, silent overflow and the `checked` context, and the `is Type t` pattern match for safe casts. You will reproduce lesson examples, but woven into a single coherent task — a small app that processes user input.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы пишете backend-утилиту для маленького кофейного киоска. Кассир вводит в консоль строки, которые приходят из разных источников: иногда это число порций, иногда — цена, иногда — флаг `yes`/`no`, а иногда — произвольный текст, который кто-то ввёл по ошибке. Ваша задача — превратить этот «поток строк» в надёжные значения типов `int`, `double`, `bool`, не уронив приложение на первом же нечисловом вводе и не «схлопотав» молчаливое переполнение при суммировании выручки за смену.

В уроке вы изучили несколько механизмов преобразования, и каждый из них имеет свой контракт. Неявное преобразование безопасно, потому что приёмник всегда вмещает источник целиком: из `int` в `long`, из `int` в `double`, из `char` в `int`. Явный cast `(int)double` уже опасен — он отбрасывает дробную часть, а не округляет, и старшие биты могут потеряться при сужении `long` в `int`. Класс `Convert` ведёт себя иначе: он округляет по правилам банка (`2.5 → 2`, `3.5 → 4`), переводит `null` в `0` и бросает `OverflowException` при выходе за границы. Метод `Parse` строгий и бросает исключения на любом кривом вводе, а `TryParse` — дружелюбный: возвращает `bool` и `out`-параметр, не роняя программу.

Контекст `checked`/`unchecked` управляет переполнением. По умолчанию для констант компилятор включает `checked` (и не даст скомпилировать заведомо переполненное выражение), а для переменных во время выполнения действует `unchecked` — значение «оборачивается» и становится большим отрицательным числом. В финансовой логике такой баг коварен: программа работает, но баланс уходит в минус. Поэтому в киоске вы будете явно использовать `checked` при суммировании выручки. Наконец, для приведения ссылочных типов `object → string` вы примените pattern matching `is string text`, который компактнее и безопаснее, чем классический `as` с последующей `null`-проверкой.

В результате у вас получится консольное приложение `CoffeeKiosk`, которое читает несколько строк, корректно преобразует их в нужные типы через правильно подобранные механизмы, аккумулирует выручку в `checked`-контексте и устойчиво сообщает пользователю об ошибках ввода, не падая. Это ровно тот набор навыков, который нужен в реальном backend-коде: данные всегда приходят «грязными», и ваша обязанность — превратить их в корректные значения.

#### Что нужно сделать (пошагово)
1. Создайте новый консольный проект .NET 8 с именем `CoffeeKiosk`. Выполните в пустом каталоге команду `dotnet new console -n CoffeeKiosk -f net8.0`, затем перейдите в папку проекта `cd CoffeeKiosk` и откройте `Program.cs`. Убедитесь, что в `.csproj` присутствует `<TargetFramework>net8.0</TargetFramework>` и при необходимости добавьте `<CheckForOverflowUnderflow>false</CheckForOverflowUnderflow>` (значение по умолчанию), чтобы переполнение в обычных выражениях было «молчаливым» — именно так вы увидите разницу с `checked`.
2. В `Program.cs` напишите код на top-level statements (без `class Program` и `static void Main`). Структура должна включать несколько логических блоков, помеченных комментариями: блок неявных преобразований, блок явных cast, блок `Convert`, блок `Parse` с `try/catch`, блок `TryParse` для пользовательского ввода, блок переполнения с `checked` и блок pattern matching для ссылочных типов.
3. В блоке неявных преобразований продемонстрируйте переходы, которые компилятор делает сам: `int → long`, `int → double`, `char → int`. Выведите каждый результат через интерполяцию строк, чтобы видно было исходное и итоговое значение. Прокомментируйте, почему именно эти переходы безопасны (приёмник полностью вмещает источник).
4. В блоке явных cast покажите сужение `double → int` (дробная часть отбрасывается) и `long → int` (старшие биты теряются). Возьмите `double pi = 3.99;` и `long huge = 5_000_000_000L;`. Выведите результаты и явно укажите в комментарии, что произошла потеря данных. Сравните `(int)2.9` (получится `2`) с интуитивным ожиданием округления, чтобы зафиксировать частую ошибку из урока.
5. В блоке `Convert` вызовите `Convert.ToInt32(2.5)` и `Convert.ToInt32(3.5)` — убедитесь, что банковское округление даёт `2` и `4` соответственно. Затем передайте в `Convert.ToInt32` значение `null` (через переменную `object? maybeNull = null;`) и покажите, что результат — `0`, а не исключение. Это контрастирует с поведением `Parse`, которое бросает `ArgumentNullException` на `null`.
6. В блоке `Parse` оберните вызов `int.Parse("abc")` в `try/catch (FormatException)` и распечатайте сообщение исключения. Также покажите `int.Parse("42")`, который проходит успешно. Зафиксируйте в комментарии, что `Parse` — метод конкретного типа и его стоит использовать только тогда, когда формат гарантирован.
7. В блоке `TryParse` прочитайте с консоли возраст пользователя через `Console.ReadLine()`, передайте результат в `int.TryParse(input, out int age)`. Если вернулось `true` — напечатайте «ОК, возраст = ...», если `false` — напечатайте понятное сообщение с указанием, что именно было введено. Используйте inline-объявление `out`-переменной (C# 7+), как в уроке.
8. В блоке суммирования выручки организуйте цикл из трёх итераций: на каждой просите ввести цену чашки. Каждую цену преобразуйте через `TryParse` в `double`, а накопление суммы делайте в `checked`-выражении: `sum = checked(sum + price);`. Подготовьте ввод, при котором сумма переполнит `int` (например, если ввести три значения рядом с `int.MaxValue`), и покажите, что `checked` бросает `OverflowException`, а без `checked` значение «уезжает» в минус.
9. В блоке pattern matching возьмите `object boxed = "latte";` и примените `if (boxed is string text) { ... }`. Выведите длину строки. Затем попробуйте `object boxedNum = 42;` и покажите, что `is string` даёт `false`, а `is int` — `true`, демонстрируя безопасное приведение по иерархии типов.
10. Запустите проект командой `dotnet run` и убедитесь, что все блоки выводят ожидаемые значения. Проверьте граничные вводы: пустую строку, `null` (через Ctrl+D/ввод пустой строки в зависимости от ОС), очень большое число, нечисловой текст. Приложение не должно падать ни в одном случае — для пользовательского ввода всегда должен быть путь `TryParse → false → понятное сообщение`.

#### Требования к решению
Решение должно быть оформлено как один файл `Program.cs` с top-level statements, целевой фреймворк `net8.0`, без устаревшей конструкции `class Program { static void Main() }`. Каждый логический блок отделяется пустой строкой и заголовочным комментарием вида `// === Блок N: ... ===`. Все преобразования должны использовать именно тот механизм, который требует урок: не подменяйте `TryParse` на `Parse` ради «короткого кода» и не прячьте `checked` за `unchecked`. Для пользовательского ввода обязательно применяется `TryParse` — это best practice из урока, убирающий целый класс исключений «по контролируемому потоку».

Код должен компилироваться без предупреждений (`dotnet build` с `-warnaserror` приветствуется). Имена переменных — осмысленные (`price`, `age`, `sum`, `maybeNull`), а не однобуквенные. Вывод должен быть двуязычным или хотя бы понятным: каждое сообщение содержит как число/значение, так и краткое пояснение. Используйте интерполяцию строк `$"{...}"` и, где уместно, `CultureInfo.InvariantCulture` для предсказуемого разбора чисел.

Контекст `checked` должен применяться только там, где переполнение — это бизнес-ошибка (сумма выручки). Для остальных операций оставьте поведение по умолчанию (`unchecked`). В комментариях явно отмечайте, какой контракт у каждого механизма: например, «`Convert.ToInt32(null)` → 0, без исключения» или «`int.Parse(null)` бросает `ArgumentNullException`». Это фиксирует понимание разницы контрактов — ключевую мысль урока.

Обработка `false` от `TryParse` должна быть содержательной: не просто «ошибка», а указание, какой ввод был некорректен. В блоке `Parse` обязательно наличие `try/catch` с конкретным типом исключения (`FormatException`), а не универсального `catch (Exception)`. Pattern matching `is Type t` используется вместо `as` + `null`-проверки, как требует best practice урока. Весь код должен работать на .NET 8 и использовать возможности C# 12 (inline `out`-переменные доступны с C# 7, но также приветствуются pattern matching, collection expressions и raw strings, если они уместны).

#### Тонкости и подводные камни
Главная тонкость, на которой спотыкаются новички, — разница между cast `(int)double` и `Convert.ToInt32(double)`. Cast всегда отбрасывает дробную часть (truncation toward zero): `(int)3.9 → 3`, `(int)-3.9 → -3`. `Convert.ToInt32` округляет по правилам банка (round half to even): `2.5 → 2`, `3.5 → 4`, `-2.5 → -2`. Если вы хотите «обычное» округление, используйте `Math.Round` с `MidpointRounding.AwayFromZero`. Запомните: cast — это «обрезание», а не «округление», и путать их — частая ошибка из урока.

Вторая тонкость — поведение с `null`. `Convert.ToInt32(null)` возвращает `0` и не бросает исключение, потому что `Convert` спроектирован для «чужого мира» (БД, `object`). А `int.Parse(null)` бросает `ArgumentNullException`, потому что `Parse` ожидает валидную строку. Это разные контракты: выбирая инструмент, вы выбираете и поведение при ошибке. Не используйте `Parse` для данных, которые могут прийти `null`, и не ждите от `Convert` исключения на `null`.

Третья тонкость — переполнение. По умолчанию (если в `.csproj` не стоит `<CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>`) арифметика на переменных идёт в `unchecked`-режиме: `int.MaxValue + 1` молча становится отрицательным. Для констант компилятор включает `checked` и не даст скомпилировать `int x = int.MaxValue + 1;` — это «статический» checked. Чтобы включить проверку во время выполнения для переменных, используйте `checked(...)` как выражение или `checked { ... }` как блок. В финансовой логике всегда `checked`; в хеш-функциях и битовых масках — `unchecked` (и помечайте явно, чтобы читатель видел намерение).

Четвёртая тонкость — `TryParse` возвращает `0` в `out`-параметре при `false`. Это значит, что вы не можете отличить «ввели 0» от «ввели мусор» по значению `out` — смотрите только на возвращаемый `bool`. Также `TryParse` чувствителен к культуре: `int.TryParse("1,234")` в `ru-RU` даст одно поведение, в `InvariantCulture` — другое. Для ввода, который приходит от пользователя в одной локали, явно указывайте `CultureInfo.InvariantCulture` или нужную культуру, чтобы поведение было предсказуемым.

Пятая тонкость — pattern matching `is Type t` компактен, но помните, что он проверяет и тип, и привязывает переменную только в ветке `true`. Не путайте с `as`, который возвращает `null` при неудаче и требует отдельной проверки. Для типов значений `as` вообще неприменим (только для nullable-обёрток). Наконец, помните, что `char → int` — это неявное преобразование, дающее код символа; это часто используется, но легко спутать с приведением строки `"5"` к числу `5` (для строки нужен `int.Parse`, а не cast).

#### Критерии приёмки
- [ ] Проект `CoffeeKiosk` создан командой `dotnet new console -f net8.0` и собирается без ошибок через `dotnet build`.
- [ ] В `Program.cs` используются top-level statements без явного `class Program`/`Main`.
- [ ] Присутствует блок неявных преобразований: `int → long`, `int → double`, `char → int` с выводом и комментарием о безопасности.
- [ ] Присутствует блок явных cast: `(int)3.99` и `(int)5_000_000_000L` с указанием потери данных.
- [ ] Блок `Convert` демонстрирует банковское округление (`2.5 → 2`, `3.5 → 4`) и `Convert.ToInt32(null) → 0`.
- [ ] Блок `Parse` обёрнут в `try/catch (FormatException)` и обрабатывает `int.Parse("abc")`.
- [ ] Блок `TryParse` читает возраст с консоли и корректно обрабатывает как `true`, так и `false`.
- [ ] Суммирование выручки выполняется в `checked`-выражении и бросает `OverflowException` при переполнении.
- [ ] Pattern matching `is string text` / `is int` применён для безопасного приведения `object`.
- [ ] Приложение не падает ни на одном вводе: пустая строка, `null`, нечисловой текст, очень большое число.
- [ ] В комментариях зафиксированы контракты каждого механизма (что бросает, что возвращает по умолчанию).
- [ ] Используется `CultureInfo.InvariantCulture` для разбора чисел (или явно обосновано использование другой культуры).
- [ ] Сообщения об ошибках ввода содержат указание, какой именно ввод был некорректен.
- [ ] Нет предупреждений компилятора (или они обоснованы в комментариях).
- [ ] Код запускается командой `dotnet run` и выводит все ожидаемые значения.

#### Подсказки (без прямого ответа)
- Вспомните аналогию урока: «маленький стакан в большой — ничего не прольётся» (неявно), «большой в маленький — лишнее прольётся» (явно). Подумайте, какой тип — «стакан» в каждом вашем преобразовании.
- Для `Convert.ToInt32(2.5)` не пытайтесь угадать результат по интуиции — вспомните правило round half to even и проверьте на нескольких значениях (`2.5`, `3.5`, `4.5`).
- Чтобы вызвать `checked`-переполнение на `int`, достаточно сложить два значения рядом с `int.MaxValue`. Не пытайтесь «переполнить» `double` — у него другая арифметика.
- Для `TryParse` с культурой используйте перегрузку с тремя параметрами: `int.TryParse(input, NumberStyles.Integer, CultureInfo.InvariantCulture, out int n)`. Но для простого случая подойдёт и двухпараметровая, если вы уверены в локали.
- Для pattern matching помните: `boxed is string text` объявляет `text` только в области `if`, и переменная недоступна в `else`. Если нужно значение в обеих ветках — объявляйте отдельно.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — top-level statements
// CoffeeKiosk: демонстрация преобразований типов / Type conversion demo
using System;
using System.Globalization;

// === Блок 1: Неявные преобразования / Implicit conversions ===
// Компилятор делает сам, потерь нет / Compiler does it, no loss
int smallInt = 42;
long bigLong = smallInt;          // int -> long: приёмник шире / wider destination
double fromInt = smallInt;        // int -> double: безопасно / safe
char ch = 'A';
int codePoint = ch;               // char -> int: код символа / char code
Console.WriteLine($"Неявно / Implicit: int {smallInt} -> long {bigLong}, double {fromInt:F1}, char '{ch}' -> int {codePoint}");

// === Блок 2: Явные cast (возможна потеря) / Explicit casts (loss possible) ===
double pi = 3.99;
int truncated = (int)pi;          // дробная часть отбрасывается -> 3 / truncated -> 3
long huge = 5_000_000_000L;
int narrowed = (int)huge;         // старшие биты теряются / high bits lost
Console.WriteLine($"Cast: (int){pi} = {truncated} (отбрасывание, не округление / truncation, not rounding)");
Console.WriteLine($"Cast: (int){huge} = {narrowed} (потеря старших бит / high bits lost)");

// === Блок 3: Convert — умный перевод / Convert — smart translation ===
double half = 2.5, otherHalf = 3.5;
int r1 = Convert.ToInt32(half);       // banker's rounding -> 2
int r2 = Convert.ToInt32(otherHalf);  // banker's rounding -> 4
object? maybeNull = null;
int fromNull = Convert.ToInt32(maybeNull); // -> 0, без исключения / -> 0, no throw
Console.WriteLine($"Convert.ToInt32({half}) = {r1}, Convert.ToInt32({otherHalf}) = {r2} (round half to even)");
Console.WriteLine($"Convert.ToInt32(null) = {fromNull} (не бросает / no throw)");

// === Блок 4: Parse — строго, бросает / Parse — strict, throws ===
string goodNumber = "42";
int parsed = int.Parse(goodNumber);
Console.WriteLine($"int.Parse(\"{goodNumber}\") = {parsed}");
try
{
    int bad = int.Parse("abc"); // FormatException
    _ = bad;
}
catch (FormatException ex)
{
    Console.WriteLine($"Parse ошибся / Parse failed: {ex.Message}");
}

// === Блок 5: TryParse — безопасный ввод / TryParse — safe input ===
Console.Write("Введите возраст / Enter age: ");
string? input = Console.ReadLine();
if (int.TryParse(input, NumberStyles.Integer, CultureInfo.InvariantCulture, out int age))
{
    Console.WriteLine($"ОК, возраст = {age} / OK, age = {age}");
}
else
{
    Console.WriteLine($"Не похоже на число / Not a number: '{input}'");
}

// === Блок 6: Сумма выручки в checked / Revenue sum under checked ===
long sum = 0;
for (int i = 0; i < 3; i++)
{
    Console.Write($"Цена {i + 1}/3 / Price {i + 1}/3: ");
    string? line = Console.ReadLine();
    if (int.TryParse(line, NumberStyles.Integer, CultureInfo.InvariantCulture, out int price) && price >= 0)
    {
        sum = checked(sum + price); // бросает OverflowException при переполнении / throws on overflow
    }
    else
    {
        Console.WriteLine($"Пропуск некорректной цены / Skipping invalid price: '{line}'");
    }
}
Console.WriteLine($"Сумма / Sum: {sum}");

// === Блок 7: Pattern matching для ссылочных типов / Pattern matching for refs ===
object boxed = "latte";
if (boxed is string text)
{
    Console.WriteLine($"Pattern match: это строка длины {text.Length} / it's a string of length {text.Length}");
}
object boxedNum = 42;
if (boxedNum is int n)
{
    Console.WriteLine($"Pattern match: это int = {n} / it's an int = {n}");
}
else
{
    Console.WriteLine("Не int / Not an int");
}
```

Разбор по строкам. Блок 1 показывает неявные преобразования, которые компилятор выполняет сам, потому что приёмник полностью вмещает источник: `int` (32 бита) → `long` (64 бита), `int` → `double` (всякая `int`-величина точно представима в `double`), `char` → `int` (код символа помещается в `int` без потерь). Это безопасные переходы, и идея урока — «маленький стакан в большой» — здесь буквальна. В комментарии явно отмечено, почему именно эти преобразования неявные.

Блок 2 — явные cast. `(int)3.99` даёт `3`, потому что cast отбрасывает дробную часть (truncation), а не округляет. `(int)5_000_000_000L` даёт неожиданное значение, потому что старшие биты `long` не помещаются в `int` и теряются. В комментарии подчёркнуто ключевое отличие cast от `Convert`: cast — «обрезание», а не «округление». Это частая ошибка из урока, и решение её фиксирует.

Блок 3 — `Convert`. `Convert.ToInt32(2.5)` даёт `2`, а `Convert.ToInt32(3.5)` даёт `4` — это банковское округление (round half to even), которое отличается от «школьного» округления половины вверх. `Convert.ToInt32(null)` возвращает `0` и не бросает исключение — это контракт `Convert`, спроектированный для «чужого мира» (БД, `object`). Контраст с `Parse` (который бросает `ArgumentNullException` на `null`) зафиксирован в комментарии.

Блок 4 — `Parse`. `int.Parse("42")` проходит успешно, а `int.Parse("abc")` бросает `FormatException`, который мы ловим конкретным `catch (FormatException ex)`. Универсальный `catch (Exception)` здесь был бы антипаттерном: мы хотим обработать именно ошибку формата, а не маскировать другие возможные исключения. В комментарии отмечено, что `Parse` стоит использовать только при гарантированном формате.

Блок 5 — `TryParse` для пользовательского ввода. Используется перегрузка с `NumberStyles.Integer` и `CultureInfo.InvariantCulture`, чтобы разбор был предсказуем на любой локали (например, не зависел от того, что в `ru-RU` разделитель — запятая). Ветвь `false` печатает, какой именно ввод был некорректен, — это best practice: пользователь должен понять, что он ввёл не так. Inline-объявление `out int age` — это C# 7+, лаконично и читаемо.

Блок 6 — суммирование выручки в `checked`. `sum = checked(sum + price);` — это checked-выражение, которое бросает `OverflowException`, если сумма переполняется. В реальном киоске переполнение `long` маловероятно, но если заменить `long` на `int` и ввести три больших значения — исключение сработает. В комментарии указано, что `checked` здесь — намерение: переполнение выручки это бизнес-ошибка, и молчаливый обход недопустим. Для хеш-функций тут был бы `unchecked`.

Блок 7 — pattern matching. `boxed is string text` проверяет тип и привязывает переменную `text` только в ветке `true`. Это компактнее и безопаснее, чем `var s = boxed as string; if (s != null) ...`. Для `boxedNum = 42` срабатывает `is int n`, демонстрируя безопасное приведение по иерархии типов. В комментариях отмечено, что `as` для типов значений неприменим, и pattern matching — предпочтительный способ.

Применённые концепции урока: неявные и явные преобразования, `Convert` с банковским округлением и `null`-контрактом, `Parse` с `FormatException`, `TryParse` с инвариантной культурой, `checked` для финансов, pattern matching `is Type t`. Каждое решение обосновано выбором контракта, а не привычкой — это главная мысль урока.

#### Задания на углубление (бонус)
1. Замените `int` в блоке суммирования на `decimal` и сравните поведение при переполнении. `decimal` всегда работает в `checked`-режиме по умолчанию — подумайте, почему это разумно для финансов. Выведите, при каких значениях возникает `OverflowException`.
2. Добавьте чтение цены в формате `"1,234.56"` (с разделителем тысяч и десятичной точкой) через `decimal.TryParse` с `NumberStyles.Currency` и `CultureInfo.InvariantCulture`. Обработайте случай, когда пользователь вводит `"1.234,56"` (европейский формат) — определите культуру по наличию запятой.
3. Реализуйте функцию `T? ParseOrDefault<T>(string? input) where T : struct, IParsable<T>`, которая использует `T.TryParse` (доступен с .NET 7 через интерфейс `IParsable<T>`) и возвращает `null` при неудаче. Продемонстрируйте её для `int`, `double`, `decimal`.
4. Сравните производительность `int.Parse` + `try/catch` и `int.TryParse` на 100 000 итераций с mixesным вводом (90 % корректных, 10 % мусора). Измерьте через `Stopwatch` и объясните, почему `TryParse` быстрее на «мусорном» вводе (исключения дороги).

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you are writing a small backend utility for a tiny coffee kiosk. The cashier types strings into the console that come from different sources: sometimes it is a number of portions, sometimes a price, sometimes a `yes`/`no` flag, and sometimes arbitrary text someone typed by mistake. Your job is to turn this "stream of strings" into reliable `int`, `double`, and `bool` values, without crashing the app on the first non-numeric input and without silently overflowing when you sum the day's revenue.

In the lesson you studied several conversion mechanisms, and each has its own contract. Implicit conversion is safe because the destination always fully contains the source: `int → long`, `int → double`, `char → int`. An explicit cast `(int)double` is already dangerous — it truncates the fractional part rather than rounding, and high bits can be lost when narrowing `long` to `int`. The `Convert` class behaves differently: it rounds using banker's rounding (`2.5 → 2`, `3.5 → 4`), turns `null` into `0`, and throws `OverflowException` when a value is out of range. The `Parse` method is strict and throws on any malformed input, while `TryParse` is friendly: it returns a `bool` and an `out` parameter, never crashing the program.

The `checked`/`unchecked` context governs overflow. By default, the compiler applies `checked` to constant expressions (and refuses to compile a clearly overflowing expression), but `unchecked` to variables at runtime — the value "wraps around" and becomes a large negative number. In financial logic this bug is insidious: the program keeps running, but the balance flips negative. That is why in the kiosk you will explicitly use `checked` when summing revenue. Finally, for casting reference types `object → string` you will apply the `is string text` pattern match, which is more compact and safer than the classic `as` followed by a `null` check.

The result is a console application `CoffeeKiosk` that reads several lines, converts them to the right types through properly chosen mechanisms, accumulates revenue in a `checked` context, and gracefully reports input errors to the user without crashing. This is exactly the skill set you need in real backend code: data always arrives "dirty", and your job is to turn it into correct values.

#### What to do step by step
1. Create a new .NET 8 console project named `CoffeeKiosk`. Run `dotnet new console -n CoffeeKiosk -f net8.0` in an empty directory, then `cd CoffeeKiosk` and open `Program.cs`. Make sure the `.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and, if needed, add `<CheckForOverflowUnderflow>false</CheckForOverflowUnderflow>` (the default) so that overflow in ordinary expressions is "silent" — this is how you will see the contrast with `checked`.
2. In `Program.cs` write the code using top-level statements (no `class Program` and no `static void Main`). The structure should include several logical blocks separated by comments: an implicit conversions block, an explicit cast block, a `Convert` block, a `Parse` block with `try/catch`, a `TryParse` block for user input, an overflow block with `checked`, and a pattern matching block for reference types.
3. In the implicit conversions block, demonstrate transitions the compiler does on its own: `int → long`, `int → double`, `char → int`. Print each result with string interpolation so both the source and the destination value are visible. Comment on why exactly these transitions are safe (the destination fully contains the source).
4. In the explicit cast block, show the narrowing `double → int` (fractional part truncated) and `long → int` (high bits lost). Use `double pi = 3.99;` and `long huge = 5_000_000_000L;`. Print the results and explicitly note in the comment that data was lost. Compare `(int)2.9` (yields `2`) with the intuitive expectation of rounding, to fix the common mistake from the lesson.
5. In the `Convert` block, call `Convert.ToInt32(2.5)` and `Convert.ToInt32(3.5)` — confirm that banker's rounding yields `2` and `4` respectively. Then pass `null` to `Convert.ToInt32` (via a variable `object? maybeNull = null;`) and show that the result is `0`, not an exception. This contrasts with `Parse`, which throws `ArgumentNullException` on `null`.
6. In the `Parse` block, wrap `int.Parse("abc")` in `try/catch (FormatException)` and print the exception message. Also show `int.Parse("42")` succeeding. Note in the comment that `Parse` is a method of a specific type and should be used only when the format is guaranteed.
7. In the `TryParse` block, read the user's age from the console via `Console.ReadLine()`, pass the result to `int.TryParse(input, out int age)`. If it returns `true`, print "OK, age = ..."; if `false`, print a clear message showing what was actually entered. Use the inline `out` variable declaration (C# 7+), as in the lesson.
8. In the revenue summation block, run a loop of three iterations: on each, ask for a cup price. Parse each price with `TryParse` into a `double`, and accumulate the sum in a `checked` expression: `sum = checked(sum + price);`. Prepare input that overflows an `int` (for example, three values near `int.MaxValue`) and show that `checked` throws `OverflowException`, whereas without `checked` the value slides into the negatives.
9. In the pattern matching block, take `object boxed = "latte";` and apply `if (boxed is string text) { ... }`. Print the string length. Then try `object boxedNum = 42;` and show that `is string` yields `false` while `is int` yields `true`, demonstrating a safe cast along the type hierarchy.
10. Run the project with `dotnet run` and verify that every block prints the expected values. Test boundary inputs: an empty string, `null` (via Ctrl+D or an empty line depending on the OS), a very large number, non-numeric text. The app must not crash in any case — for user input there must always be a `TryParse → false → clear message` path.

#### Requirements
The solution must be a single `Program.cs` file with top-level statements, targeting `net8.0`, without the legacy `class Program { static void Main() }` construct. Each logical block is separated by a blank line and a header comment like `// === Block N: ... ===`. All conversions must use the exact mechanism the lesson demands: do not replace `TryParse` with `Parse` for brevity and do not hide `checked` behind `unchecked`. For user input, `TryParse` is mandatory — this is the lesson's best practice that removes a whole class of "control-flow via exceptions".

The code must compile without warnings (`dotnet build` with `-warnaserror` is welcome). Variable names should be meaningful (`price`, `age`, `sum`, `maybeNull`), not single letters. Output should be bilingual or at least clear: each message includes both the value and a short explanation. Use string interpolation `$"{...}"` and, where appropriate, `CultureInfo.InvariantCulture` for predictable number parsing.

The `checked` context must be applied only where overflow is a business error (revenue sum). For the rest, leave the default behavior (`unchecked`). In comments, explicitly mark each mechanism's contract: for example, "`Convert.ToInt32(null)` → 0, no exception" or "`int.Parse(null)` throws `ArgumentNullException`". This fixes the understanding of contract differences — the lesson's key point.

Handling `false` from `TryParse` must be meaningful: not just "error", but an indication of which input was invalid. In the `Parse` block, a `try/catch` with a specific exception type (`FormatException`) is required, not a universal `catch (Exception)`. The `is Type t` pattern match is used instead of `as` + `null` check, as the lesson's best practice demands. The whole code must run on .NET 8 and use C# 12 features (inline `out` variables are available since C# 7, but pattern matching, collection expressions, and raw strings are also welcome where appropriate).

#### Pitfalls
The main pitfall that trips up newcomers is the difference between a cast `(int)double` and `Convert.ToInt32(double)`. A cast always truncates the fractional part (truncation toward zero): `(int)3.9 → 3`, `(int)-3.9 → -3`. `Convert.ToInt32` rounds using banker's rounding (round half to even): `2.5 → 2`, `3.5 → 4`, `-2.5 → -2`. If you want "ordinary" rounding, use `Math.Round` with `MidpointRounding.AwayFromZero`. Remember: a cast is "cutting off", not "rounding", and confusing them is a common mistake from the lesson.

The second pitfall is behavior with `null`. `Convert.ToInt32(null)` returns `0` and does not throw, because `Convert` is designed for a "foreign world" (databases, `object`). And `int.Parse(null)` throws `ArgumentNullException`, because `Parse` expects a valid string. These are different contracts: by choosing the tool you choose the error behavior. Do not use `Parse` for data that may arrive as `null`, and do not expect `Convert` to throw on `null`.

The third pitfall is overflow. By default (unless `<CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>` is set in `.csproj`), arithmetic on variables runs in `unchecked` mode: `int.MaxValue + 1` silently becomes negative. For constants, the compiler applies `checked` and will not let you compile `int x = int.MaxValue + 1;` — this is "static" checked. To enable runtime checking for variables, use `checked(...)` as an expression or `checked { ... }` as a block. In financial logic, always `checked`; in hash functions and bitmasks, `unchecked` (and mark it explicitly so the reader sees the intent).

The fourth pitfall is that `TryParse` returns `0` in the `out` parameter on `false`. This means you cannot distinguish "the user entered 0" from "the user entered garbage" by the `out` value — look only at the returned `bool`. Also, `TryParse` is culture-sensitive: `int.TryParse("1,234")` behaves differently in `ru-RU` than in `InvariantCulture`. For input that arrives from a user in a single locale, explicitly specify `CultureInfo.InvariantCulture` or the required culture so behavior is predictable.

The fifth pitfall is that the `is Type t` pattern match is compact, but remember that it checks the type and binds the variable only in the `true` branch. Do not confuse it with `as`, which returns `null` on failure and requires a separate check. For value types, `as` is not applicable at all (only via nullable wrappers). Finally, remember that `char → int` is an implicit conversion that yields the character code; it is often used but easily confused with converting the string `"5"` to the number `5` (for a string you need `int.Parse`, not a cast).

#### Acceptance criteria
- [ ] The `CoffeeKiosk` project is created with `dotnet new console -f net8.0` and builds cleanly via `dotnet build`.
- [ ] `Program.cs` uses top-level statements without an explicit `class Program`/`Main`.
- [ ] An implicit conversions block is present: `int → long`, `int → double`, `char → int` with output and a safety comment.
- [ ] An explicit cast block is present: `(int)3.99` and `(int)5_000_000_000L` with data-loss indication.
- [ ] The `Convert` block demonstrates banker's rounding (`2.5 → 2`, `3.5 → 4`) and `Convert.ToInt32(null) → 0`.
- [ ] The `Parse` block is wrapped in `try/catch (FormatException)` and handles `int.Parse("abc")`.
- [ ] The `TryParse` block reads age from the console and correctly handles both `true` and `false`.
- [ ] Revenue summation runs in a `checked` expression and throws `OverflowException` on overflow.
- [ ] The `is string text` / `is int` pattern match is applied for safe `object` casts.
- [ ] The app does not crash on any input: empty string, `null`, non-numeric text, a very large number.
- [ ] Comments record each mechanism's contract (what it throws, what it returns by default).
- [ ] `CultureInfo.InvariantCulture` is used for number parsing (or another culture is explicitly justified).
- [ ] Input error messages indicate which exact input was invalid.
- [ ] There are no compiler warnings (or they are justified in comments).
- [ ] The code runs with `dotnet run` and prints all expected values.

#### Hints (no direct answer)
- Recall the lesson's analogy: "a small glass into a large one — nothing spills" (implicit), "a large one into a small one — the excess spills" (explicit). Think about which type is the "glass" in each of your conversions.
- For `Convert.ToInt32(2.5)`, do not guess the result by intuition — recall the round half to even rule and check several values (`2.5`, `3.5`, `4.5`).
- To trigger `checked` overflow on an `int`, it is enough to add two values near `int.MaxValue`. Do not try to "overflow" a `double` — its arithmetic is different.
- For `TryParse` with a culture, use the three-parameter overload: `int.TryParse(input, NumberStyles.Integer, CultureInfo.InvariantCulture, out int n)`. But for a simple case the two-parameter one is fine if you are sure of the locale.
- For pattern matching, remember: `boxed is string text` declares `text` only inside the `if`, and the variable is unavailable in `else`. If you need the value in both branches, declare it separately.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — top-level statements
// CoffeeKiosk: type conversion demo
using System;
using System.Globalization;

// === Block 1: Implicit conversions ===
// The compiler does these on its own, no loss
int smallInt = 42;
long bigLong = smallInt;          // int -> long: wider destination
double fromInt = smallInt;        // int -> double: safe
char ch = 'A';
int codePoint = ch;               // char -> int: character code
Console.WriteLine($"Implicit: int {smallInt} -> long {bigLong}, double {fromInt:F1}, char '{ch}' -> int {codePoint}");

// === Block 2: Explicit casts (loss possible) ===
double pi = 3.99;
int truncated = (int)pi;          // fractional part truncated -> 3
long huge = 5_000_000_000L;
int narrowed = (int)huge;         // high bits lost
Console.WriteLine($"Cast: (int){pi} = {truncated} (truncation, not rounding)");
Console.WriteLine($"Cast: (int){huge} = {narrowed} (high bits lost)");

// === Block 3: Convert — smart translation ===
double half = 2.5, otherHalf = 3.5;
int r1 = Convert.ToInt32(half);       // banker's rounding -> 2
int r2 = Convert.ToInt32(otherHalf);  // banker's rounding -> 4
object? maybeNull = null;
int fromNull = Convert.ToInt32(maybeNull); // -> 0, no throw
Console.WriteLine($"Convert.ToInt32({half}) = {r1}, Convert.ToInt32({otherHalf}) = {r2} (round half to even)");
Console.WriteLine($"Convert.ToInt32(null) = {fromNull} (no throw)");

// === Block 4: Parse — strict, throws ===
string goodNumber = "42";
int parsed = int.Parse(goodNumber);
Console.WriteLine($"int.Parse(\"{goodNumber}\") = {parsed}");
try
{
    int bad = int.Parse("abc"); // FormatException
    _ = bad;
}
catch (FormatException ex)
{
    Console.WriteLine($"Parse failed: {ex.Message}");
}

// === Block 5: TryParse — safe input ===
Console.Write("Enter age: ");
string? input = Console.ReadLine();
if (int.TryParse(input, NumberStyles.Integer, CultureInfo.InvariantCulture, out int age))
{
    Console.WriteLine($"OK, age = {age}");
}
else
{
    Console.WriteLine($"Not a number: '{input}'");
}

// === Block 6: Revenue sum under checked ===
long sum = 0;
for (int i = 0; i < 3; i++)
{
    Console.Write($"Price {i + 1}/3: ");
    string? line = Console.ReadLine();
    if (int.TryParse(line, NumberStyles.Integer, CultureInfo.InvariantCulture, out int price) && price >= 0)
    {
        sum = checked(sum + price); // throws OverflowException on overflow
    }
    else
    {
        Console.WriteLine($"Skipping invalid price: '{line}'");
    }
}
Console.WriteLine($"Sum: {sum}");

// === Block 7: Pattern matching for reference types ===
object boxed = "latte";
if (boxed is string text)
{
    Console.WriteLine($"Pattern match: it's a string of length {text.Length}");
}
object boxedNum = 42;
if (boxedNum is int n)
{
    Console.WriteLine($"Pattern match: it's an int = {n}");
}
else
{
    Console.WriteLine("Not an int");
}
```

Line-by-line walk-through. Block 1 shows implicit conversions that the compiler performs on its own because the destination fully contains the source: `int` (32 bits) → `long` (64 bits), `int` → `double` (every `int` value is exactly representable as a `double`), `char` → `int` (the character code fits in an `int` without loss). These are safe transitions, and the lesson's "small glass into a large one" idea is literal here. The comment explicitly notes why exactly these conversions are implicit.

Block 2 is explicit casts. `(int)3.99` yields `3` because a cast truncates the fractional part (truncation), it does not round. `(int)5_000_000_000L` yields an unexpected value because the high bits of the `long` do not fit in an `int` and are lost. The comment stresses the key difference between a cast and `Convert`: a cast is "cutting off", not "rounding". This is a common mistake from the lesson, and the solution fixes it.

Block 3 is `Convert`. `Convert.ToInt32(2.5)` yields `2` and `Convert.ToInt32(3.5)` yields `4` — this is banker's rounding (round half to even), which differs from "school" rounding of halves upward. `Convert.ToInt32(null)` returns `0` and does not throw — this is the `Convert` contract, designed for a "foreign world" (databases, `object`). The contrast with `Parse` (which throws `ArgumentNullException` on `null`) is recorded in the comment.

Block 4 is `Parse`. `int.Parse("42")` succeeds, while `int.Parse("abc")` throws `FormatException`, which we catch with a specific `catch (FormatException ex)`. A universal `catch (Exception)` would be an anti-pattern here: we want to handle exactly the format error, not mask other possible exceptions. The comment notes that `Parse` should be used only when the format is guaranteed.

Block 5 is `TryParse` for user input. The overload with `NumberStyles.Integer` and `CultureInfo.InvariantCulture` is used so parsing is predictable in any locale (for example, not dependent on the fact that in `ru-RU` the separator is a comma). The `false` branch prints which exact input was invalid — this is a best practice: the user must understand what they entered wrong. The inline `out int age` declaration is C# 7+, concise and readable.

Block 6 is revenue summation under `checked`. `sum = checked(sum + price);` is a checked expression that throws `OverflowException` if the sum overflows. In a real kiosk, overflowing a `long` is unlikely, but if you replace `long` with `int` and enter three large values, the exception fires. The comment notes that `checked` here is the intent: revenue overflow is a business error, and silent wraparound is unacceptable. For hash functions this would be `unchecked`.

Block 7 is pattern matching. `boxed is string text` checks the type and binds the variable `text` only in the `true` branch. This is more compact and safer than `var s = boxed as string; if (s != null) ...`. For `boxedNum = 42`, `is int n` fires, demonstrating a safe cast along the type hierarchy. The comments note that `as` is not applicable for value types, and pattern matching is the preferred way.

Lesson concepts applied: implicit and explicit conversions, `Convert` with banker's rounding and the `null` contract, `Parse` with `FormatException`, `TryParse` with an invariant culture, `checked` for finance, the `is Type t` pattern match. Every decision is justified by choosing a contract, not by habit — this is the lesson's main point.

#### Going deeper (bonus)
1. Replace `int` in the summation block with `decimal` and compare overflow behavior. `decimal` always runs in `checked` mode by default — think about why this is sensible for finance. Print the values at which `OverflowException` occurs.
2. Add reading a price in the format `"1,234.56"` (with a thousands separator and a decimal point) via `decimal.TryParse` with `NumberStyles.Currency` and `CultureInfo.InvariantCulture`. Handle the case where the user enters `"1.234,56"` (European format) — detect the culture by the presence of a comma.
3. Implement a function `T? ParseOrDefault<T>(string? input) where T : struct, IParsable<T>` that uses `T.TryParse` (available since .NET 7 through the `IParsable<T>` interface) and returns `null` on failure. Demonstrate it for `int`, `double`, `decimal`.
4. Compare the performance of `int.Parse` + `try/catch` and `int.TryParse` on 100,000 iterations with mixed input (90 % valid, 10 % garbage). Measure with `Stopwatch` and explain why `TryParse` is faster on "garbage" input (exceptions are expensive).

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `CoffeeKiosk` создан и собирается без ошибок.
- [ ] (RU) Использованы top-level statements, без `class Program`/`Main`.
- [ ] (RU) Реализованы все 7 блоков: неявные, явные, `Convert`, `Parse`, `TryParse`, `checked`, pattern matching.
- [ ] (RU) Для пользовательского ввода везде применяется `TryParse` с обработкой `false`.
- [ ] (RU) `checked` используется в блоке суммирования выручки.
- [ ] (RU) Приложение не падает ни на одном вводе.
- [ ] (RU) В комментариях зафиксированы контракты каждого механизма.
- [ ] (RU) Использована `CultureInfo.InvariantCulture` для разбора чисел.
- [ ] (RU) Сообщения об ошибках указывают некорректный ввод.
- [ ] (EN) The `CoffeeKiosk` project is created and builds without errors.
- [ ] (EN) Top-level statements are used, with no `class Program`/`Main`.
- [ ] (EN) All 7 blocks are implemented: implicit, explicit, `Convert`, `Parse`, `TryParse`, `checked`, pattern matching.
- [ ] (EN) `TryParse` with `false` handling is used everywhere for user input.
- [ ] (EN) `checked` is used in the revenue summation block.
- [ ] (EN) The app does not crash on any input.
- [ ] (EN) Comments record each mechanism's contract.
- [ ] (EN) `CultureInfo.InvariantCulture` is used for number parsing.
- [ ] (EN) Error messages indicate the invalid input.

#### Ресурсы / Resources
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/programming-guide/types/casting-and-type-conversions — Приведение и преобразование типов (C#) / Casting and Type Conversions (C#)
- Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.int32.tryparse — Int32.TryParse Method / Метод Int32.TryParse
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/statements/checked-and-unchecked — Операторы checked и unchecked / checked and unchecked statements
- Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.convert.toint32 — Convert.ToInt32 Method / Метод Convert.ToInt32
- Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.iparsable-1 — IParsable<T> interface / Интерфейс IParsable<T>
- Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.midpointrounding — MidpointRounding enum / Перечисление MidpointRounding

---
[← К уроку M02-L06](lesson-M02-L06-type-conversion.md) | [⬆ К модулю M02](../README.md) | [Следующее ДЗ →](homework-M02-L07-string-stringbuilder.md)
---
