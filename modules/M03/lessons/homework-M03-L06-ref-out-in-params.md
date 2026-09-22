---
[← К уроку M03-L06](lesson-M03-L06-ref-out-in-params.md) | [⬆ К модулю M03](../README.md) | [Следующее ДЗ →](homework-M03-L07-overloading-scope.md)
---

### Домашнее задание M03-L06: Параметры: ref, out, in, params, значения по умолчанию / Homework M03-L06: Parameters: ref, out, in, params, defaults

**Урок / Lesson:** M03-L06
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться осознанно выбирать и корректно применять модификаторы параметров `ref`, `out`, `in`, `params`, а также значения по умолчанию и именованные аргументы в реалистичном мини-проекте на C# 12 / .NET 8, избегая типичных ошибок из урока. (EN) Learn to deliberately choose and correctly apply the parameter modifiers `ref`, `out`, `in`, `params`, plus default values and named arguments, inside a realistic C# 12 / .NET 8 mini-project, while avoiding the common mistakes highlighted in the lesson.

#### Связь с уроком / Connection to the lesson
(RU) ДЗ напрямую закрепляет все шесть концепций урока: передачу по значению как фон для контраста, двусторонний канал `ref` на примере нормализации угла и обмена значений, обязательное присваивание `out` на парсинге и анализе текста, `in` для большой `readonly struct`, `params` для логгера, а также дефолты и именованные аргументы в API отчёта. Особое внимание уделено частым ошибкам урока: модификаторы в обеих сторонах, инициализация до вызова, запрет мутации `in`, `params` только последним, риск бинарной совместимости дефолтов.
(EN) The homework reinforces all six lesson concepts directly: pass-by-value as the contrasting baseline, the two-way `ref` channel via angle normalization and swap, mandatory `out` assignment via parsing and text analysis, `in` for a large `readonly struct`, `params` for a logger, and defaults plus named arguments in a report API. Special attention is paid to the lesson's common mistakes: modifiers on both sides, initialization before the call, the `in` mutation ban, `params` only as the last parameter, and the binary-compatibility risk of defaults.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы присоединились к команде, которая пишет внутреннюю утилитарную библиотеку `MiniUtils` для консольного приложения, обрабатывающего текстовые отчёты и трёхмерные координаты. Архитектор проекта требует, чтобы каждая функция использовала именно тот вид параметров, который лучше всего выражает намерение: не «все через `ref`», а осознанный выбор между `ref`, `out`, `in`, `params` и значениями по умолчанию. Это не академическое требование — от выбора зависит и производительность (копии больших структур), и читаемость вызовов (понятно ли, что метод мутирует аргумент), и устойчивость API к будущим изменениям (бинарная совместимость дефолтов).

Конкретно библиотека должна уметь: вычислять расстояние между тяжёлыми точками-структурами без копирования; разбирать шестнадцатеричные строки с сигналом успеха; анализировать текст и возвращать одновременно число слов, символов и строк; нормализовать угол в диапазон `[0, 360)` и обменивать значения местами; логировать переменное число сообщений; формировать параметры отчёта с разумными значениями по умолчанию. Каждый из этих сценариев идеально ложится ровно на один модификатор из урока, и ваша задача — реализовать их так, чтобы код компилировался без предупреждений, был идиоматичен для C# 12 / .NET 8 и не содержал частых ошибок, перечисленных в уроке.

Важно понимать контраст с передачей по значению: обычный `int` копируется, и переназначение параметра наружу не видно; для большой структуры копия на каждом вызове стоит дорого — именно здесь `in` экономит без риска мутации. Для ссылочных типов копируется ссылка, но переназначение параметра на новый объект тоже невидимо снаружи. Эти различия — фундамент, на котором строится выбор модификатора.

#### Что нужно сделать (пошагово)
1. Создайте проект: `dotnet new console -n MiniUtils -o MiniUtils` (C# 12 / .NET 8 по умолчанию). Перейдите в папку: `cd MiniUtils`. Откройте `Program.cs` — здесь будут top-level statements.
2. Определите `readonly struct BigPoint(double x, double y, double z)` с тремя `init`-свойствами `X`, `Y`, `Z` и вычисляемым свойством `NormSquared`. Эта структура «тяжёлая» (24+ байт) — кандидат на `in`.
3. В статическом классе `Geometry` реализуйте `double Distance(in BigPoint a, in BigPoint b)`, вычисляющий корень из суммы квадратов разностей координат. Убедитесь, что попытка `a.X = 0;` вызывает ошибку компиляции (CS8331/CS0204) — закомментируйте её с пояснением.
4. В статическом классе `TextOps` реализуйте `bool TryParseHex(string s, out int value)`: метод обязан присвоить `value` на любом пути возврата. Строки вида `"0xFF"` и `"ff"` должны разбираться в `255`; пустая или `null`-строка возвращает `false`. Используйте `int.TryParse` с `NumberStyles.HexNumber`.
5. В том же `TextOps` добавьте `void Analyze(string text, out int words, out int chars, out int lines)`. Метод обязан присвоить все три `out`-параметра. Разделение слов — через `Split` с `StringSplitOptions.RemoveEmptyEntries | TrimEntries`; строки — по `'\n'`. Обработайте `null`/пустоту корректно (ноль везде).
6. В статическом классе `NumOps` реализуйте `void NormalizeAngle(ref double angle)`, приводящий угол в `[0, 360)` через `% 360.0` и добавление `360.0` при отрицательном результате. Добавьте `void Swap(ref int a, ref int b)` через кортежную деконструкцию `(a, b) = (b, a)`.
7. В статическом классе `Logger` реализуйте `void Log(string level, params string[] messages)` и `int Sum(params int[] numbers)`. `params` обязан быть последним параметром. Продемонстрируйте вызов с коллекцией-выражением `Sum([10, 20, 30])`.
8. В статическом классе `Report` реализуйте `void Print(int page = 1, string title = "Untitled", bool landscape = false)`. Значения по умолчанию — константы времени компиляции.
9. В `Program.cs` после классов напишите блок демонстрации через top-level statements: создайте точки, вызовите `Distance(in p1, in p2)`, разберите `"0xFF"`, проанализируйте многострочный текст, нормализуйте угол `740.0`, обменяйте `1` и `2`, залогируйте три сообщения, просуммируйте коллекцию, вызовите `Print()` всеми четырьмя способами из урока (все дефолты, позиционный, именованный, смешанный любой порядок).
10. Запустите `dotnet run` и сверьте вывод с ожидаемым: `Distance: 3.000`, `Hex: 255`, `Words=5, Chars=..., Lines=2`, `Angle: 20`, `Swap: a=2, b=1`, `Sum: 15`, `Sum(collection): 60`, и четыре строки отчёта.
11. Временно нарушьте правила, чтобы увидеть ошибки компилятора: уберите `ref` в месте вызова `Swap`; передайте неинициализированную переменную в `ref`; попытайтесь изменить `in`-параметр; поставьте `params` не последним. Зафиксируйте номера ошибок в комментарии, затем верните код обратно.

#### Требования к решению
- Целевой фреймворк `net8.0`, язык C# 12, top-level statements в `Program.cs`; код компилируется без ошибок и предупреждений уровня `nullable enable`-совместимости.
- Каждый модификатор применён ровно там, где он выражает намерение: `in` — для `BigPoint`, `out` — для парсинга и анализа, `ref` — для нормализации и обмена, `params` — для логгера и суммы, дефолты — для отчёта.
- `ref`/`out`/`in` модификаторы присутствуют и в сигнатуре метода, и в месте вызова — это ключевое требование урока.
- Переменные, передаваемые в `ref` и `in`, инициализированы до вызова; в `out` — объявляются inline через `out int x` (C# 7+) или инициализируются вне.
- Все `out`-параметры гарантированно присвоены на каждом пути возврата (включая ранние `return false`).
- `params`-параметр ровно один и стоит последним в списке параметров метода.
- Значения по умолчанию являются константами времени компиляции (`1`, `"Untitled"`, `false`).
- Использованы возможности C# 12: коллекции-выражения `[10, 20, 30]`, кортежная деконструкция в `Swap`, primary-конструктор `readonly struct BigPoint(double x, double y, double z)`.
- Демонстрационный вывод соответствует ожидаемому; формат чисел — `F3` для расстояния, целое для угла и суммы.
- В комментарии в коде отмечены номера ошибок компилятора для трёх намеренных нарушений (пункт 11) — это доказывает, что вы видите, как компилятор защищает контракт.

#### Тонкости и подводные камни
- **Модификатор в обе стороны.** `ref`/`out`/`in` должны стоять и в объявлении метода, и при вызове. Забыли `ref` при вызове `Swap(ref a, ref b)` — компилятор сообщит об ошибке, потому что сигнатура требует ссылки. Это не украшение, а часть контракта: вызывающая сторона явно подтверждает, что знает о мутации.
- **Инициализация до вызова для `ref` и `in`.** `ref` и `in` требуют, чтобы переменная уже имела значение. Передача неинициализированной переменной в `ref` — ошибка. Если значения ещё нет — это сигнал использовать `out`: метод сам обязан его присвоить.
- **`out` обязан присвоить.** Компилятор проверяет, что все `out`-параметры присвоены на каждом пути возврата, включая ранние `return false`. Это защищает вызывающую сторону от чтения неопределённого значения. Шаблон: первой строкой пишите `value = 0;` (или `default`), затем при успехе перезаписываете.
- **`in` нельзя изменять.** Попытка `a.X = 0;` внутри `Distance(in BigPoint a, ...)` — ошибка компиляции. Если действительно нужно мутировать — меняйте модификатор на `ref`. `in` оптимизирован для чтения больших структур без копий; для маленьких (до ~16 байт) он не даёт выгоды и лишь добавляет шум.
- **`params` только последним и только один.** `void Log(params string[] m, int x)` — ошибка. Несколько `params` — тоже ошибка. Можно передать массив напрямую или через коллекцию-выражение `[...]`; компилятор сам упакует перечисленные аргументы в массив.
- **Бинарная совместимость дефолтов.** Значение по умолчанию «вшивается» в место вызова клиента. Если вы меняете `title = "Untitled"` на `title = "Draft"` в новой версии библиотеки, перекомпилированные клиенты увидят `"Draft"`, а старые бинарники продолжат передавать `"Untitled"`. В публичных API библиотек предпочитайте перегрузки, а не дефолты; дефолты безопасны внутри одного решения.
- **Именованные аргументы для пропуска средних.** `Print(landscape: true)` пропускает `page` и `title`, беря их дефолты — это идиоматичный способ вызывать API с дефолтами. Любой порядок допустим, но смешивание позиционных и именованных требует, чтобы позиционные шли до именованных.
- **Передача по значению как фон.** Обычный `int n` копируется: `n += 100` внутри метода не меняет оригинал. Для ссылочного типа копируется ссылка: мутация полей объекта видна снаружи, но переназначение параметра на новый объект — нет. Этот контраст — причина, по которой вообще нужны `ref`/`out`/`in`.

#### Критерии приёмки
- [ ] Проект `MiniUtils` создан на `net8.0`/C# 12, `dotnet build` без ошибок и предупреждений.
- [ ] `readonly struct BigPoint` с primary-конструктором и свойствами `X`, `Y`, `Z`, `NormSquared`.
- [ ] `Geometry.Distance(in BigPoint, in BigPoint)` вычисляет корректное расстояние, `in`-модификаторы в сигнатуре и при вызове.
- [ ] Попытка мутации `in`-параметра закомментирована с указанием ошибки компиляции.
- [ ] `TextOps.TryParseHex(string, out int)` возвращает `true`/`255` для `"0xFF"` и `"ff"`, `false` для пустой/null.
- [ ] Все `out`-параметры присвоены на каждом пути возврата (включая ранние `return false`).
- [ ] `TextOps.Analyze(string, out int, out int, out int)` корректно считает слова, символы, строки; обрабатывает `null`/пустоту.
- [ ] `NumOps.NormalizeAngle(ref double)` приводит `740.0` к `20.0`, `-45.0` к `315.0`; `ref` в обе стороны.
- [ ] `NumOps.Swap(ref int, ref int)` через кортежную деконструкцию обменивает значения.
- [ ] `Logger.Log(string, params string[])` — `params` последним; вызов с тремя аргументами работает.
- [ ] `Logger.Sum(params int[])` вызван и с перечислением, и с коллекцией-выражением `[10, 20, 30]` (результат `60`).
- [ ] `Report.Print(int=1, string="Untitled", bool=false)` вызван четырьмя способами: все дефолты, позиционный, именованный, смешанный любой порядок.
- [ ] Значения по умолчанию — константы времени компиляции.
- [ ] Вывод `dotnet run` совпадает с ожидаемым по всем пунктам демонстрации.
- [ ] В комментариях зафиксированы номера ошибок компилятора для трёх намеренных нарушений (нет `ref` при вызове; неинициализированная переменная в `ref`; `params` не последним).

#### Подсказки (без прямого ответа)
- Подумайте, какой модификатор выражает «метод должен вернуть результат и сообщить об успехе» — это классический сценарий из урока с `int.TryParse`.
- Для `in` вспомните аналогию урока: «документ для чтения без права редактирования». Где компилятор физически не даст вам написать?
- Чтобы гарантировать присваивание `out`, поставьте дефолт первой строкой тела метода — тогда любой ранний `return` уже безопасен.
- Для `params` и коллекций вспомните синтаксис коллекций-выражений C# 12: `[1, 2, 3]` неявно конвертируется в `int[]`.
- Дефолты «вшиваются» в место вызова — спросите себя, что увидит старый клиент, если вы поменяете дефолт в новой версии библиотеки.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — MiniUtils: домашнее задание M03-L06
// Запуск / Run: dotnet run
using System;
using System.Globalization;

// Большая readonly-структура — кандидат на in / large readonly struct, candidate for `in`.
readonly struct BigPoint(double x, double y, double z)
{
    public double X { get; } = x;     // init-only, неизменяем после создания / immutable after construction
    public double Y { get; } = y;
    public double Z { get; } = z;
    public double NormSquared => X * X + Y * Y + Z * Z;
}

static class Geometry
{
    // in — ref readonly: читаем тяжёлые структуры без копий / read heavy structs without copies.
    public static double Distance(in BigPoint a, in BigPoint b)
    {
        // a.X = 0; // CS8331: нельзя изменять in-параметр / cannot modify an in-parameter
        double dx = a.X - b.X;
        double dy = a.Y - b.Y;
        double dz = a.Z - b.Z;
        return Math.Sqrt(dx * dx + dy * dy + dz * dz);
    }
}

static class TextOps
{
    // out — метод обязан присвоить value до любого возврата / must assign `value` before any return.
    public static bool TryParseHex(string s, out int value)
    {
        value = 0;                                              // обязаны присвоить / must assign
        if (string.IsNullOrWhiteSpace(s)) return false;
        if (s.StartsWith("0x", StringComparison.OrdinalIgnoreCase))
            s = s[2..];                                         // срез строки / range slice, C# 8+
        return int.TryParse(s, NumberStyles.HexNumber, CultureInfo.InvariantCulture, out value);
    }

    // Несколько out — множественный возврат без кортежа / multiple out, no tuple.
    public static void Analyze(string text, out int words, out int chars, out int lines)
    {
        words = string.IsNullOrEmpty(text) ? 0
            : text.Split(' ', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries).Length;
        chars = text?.Length ?? 0;
        lines = string.IsNullOrEmpty(text) ? 0 : text.Split('\n').Length;
    }
}

static class NumOps
{
    // ref — двусторонний канал: читаем и меняем существующее / two-way: read and mutate existing.
    public static void NormalizeAngle(ref double angle)
    {
        angle %= 360.0;                // остаток от деления / remainder
        if (angle < 0) angle += 360.0; // отрицательный угол — в плюс / wrap negative into positive
    }

    // Кортежная деконструкция C# 12 / C# 12 tuple deconstruction swap.
    public static void Swap(ref int a, ref int b) => (a, b) = (b, a);
}

static class Logger
{
    // params — переменное число аргументов, только последним / variable args, must be last.
    public static void Log(string level, params string[] messages)
        => Console.WriteLine($"[{level}] {string.Join(" | ", messages)}");

    public static int Sum(params int[] numbers)
    {
        int total = 0;
        foreach (var n in numbers) total += n;
        return total;
    }
}

static class Report
{
    // Значения по умолчанию — константы времени компиляции / defaults are compile-time constants.
    public static void Print(int page = 1, string title = "Untitled", bool landscape = false)
        => Console.WriteLine($"Report '{title}', page {page}, landscape={landscape}");
}

// --- Демонстрация через top-level statements / demo via top-level statements ---
var p1 = new BigPoint(0, 0, 0);
var p2 = new BigPoint(1, 2, 2);
Console.WriteLine($"Distance: {Geometry.Distance(in p1, in p2):F3}");   // 3.000

if (TextOps.TryParseHex("0xFF", out int hex))
    Console.WriteLine($"Hex: {hex}");                                    // 255

TextOps.Analyze("C# params are powerful\nSecond line", out int w, out int c, out int l);
Console.WriteLine($"Words={w}, Chars={c}, Lines={l}");                   // Words=5, Lines=2

double angle = 740.0;
NumOps.NormalizeAngle(ref angle);
Console.WriteLine($"Angle: {angle}");                                    // 20

int a = 1, b = 2;
NumOps.Swap(ref a, ref b);
Console.WriteLine($"Swap: a={a}, b={b}");                                // a=2, b=1

Logger.Log("INFO", "boot", "ready", "ok");                               // params из 3 сообщений
Console.WriteLine($"Sum: {Logger.Sum(1, 2, 3, 4, 5)}");                  // 15
Console.WriteLine($"Sum(collection): {Logger.Sum([10, 20, 30])}");       // 60 — коллекция-выражение

Report.Print();                                                          // все дефолты
Report.Print(5);                                                         // позиционный
Report.Print(title: "Sales Q4");                                         // именованный
Report.Print(landscape: true, title: "Wide", page: 2);                   // смешанный, любой порядок
```

Разбор по строкам. `readonly struct BigPoint` объявлен через primary-конструктор C# 12: координаты сразу становятся `init`-свойствами, структура неизменяема — это обязательное условие для `in`, иначе компилятор сделал бы защитную копию и выигрыш от ссылки исчез. `Geometry.Distance(in BigPoint, in BigPoint)` передаёт точки по ссылке только для чтения: копии 24+ байт на каждый вызов avoided; закомментированная попытка `a.X = 0` демонстрирует, что компилятор физически блокирует мутацию (CS8331) — это и есть контракт `in` из урока. `TryParseHex(string, out int value)` первой строкой присваивает `value = 0`, поэтому любой ранний `return false` безопасен: компилятор доволен, вызывающая сторона никогда не читает неопределённое значение. Срез `s[2..]` удаляет префикс `0x`; `int.TryParse` с `NumberStyles.HexNumber` делает тяжёлую работу. `Analyze` с тремя `out` — пример множественного возврата без кортежа, как в уроке: метод обязан присвоить все три до возврата, что и делает через тернарные выражения с защитой от `null`. `NormalizeAngle(ref double)` — двусторонний канал: метод и читает, и меняет `angle`, изменение видно вызывающей стороне; `ref` стоит и в сигнатуре, и при вызове. `Swap` через `(a, b) = (b, a)` использует кортежную деконструкцию — идиома C# 12 из урока, короче временной переменной. `Logger.Log(string, params string[])` показывает, что `params` обязан быть последним: `level` идёт первым как обязательный, дальше — переменное число сообщений, упаковываемых в массив. `Sum([10, 20, 30])` — коллекция-выражение C# 12, неявно конвертируемое в `int[]`. `Report.Print` с тремя дефолтами и четыре стиля вызова (все дефолты, позиционный, именованный, смешанный любой порядок) закрепляют комбинацию дефолтов и именованных аргументов из урока. Наконец, намеренные нарушения в пункте 11 (убрать `ref` при вызове → CS1620/CS1503; неинициализированная переменная в `ref` → CS0165; `params` не последним → CS0231) доказывают, что компилятор стоит на страже контракта каждого модификатора.

#### Задания на углубление (бонус)
1. Перепишите `Analyze`, чтобы он возвращал `(int Words, int Chars, int Lines)`-кортеж вместо трёх `out`. Сравните читаемость и обсудите, в каком случае кортеж предпочтительнее `out` согласно best practices урока.
2. Добавьте перегрузку `Print(string title)` без дефолтов и обсудите, почему в публичном API библиотеки перегрузки безопаснее изменения дефолта с точки зрения бинарной совместимости. Покажите сценарий, где старый бинарник «видит» устаревший дефолт.
3. Реализуйте метод `void Translate(ref BigPoint p, double dx, double dy, double dz)` и объясните, почему `ref` здесь уместнее, чем `in`, несмотря на то, что `BigPoint` — большая структура. Когда `in` всё же предпочтительнее?
4. Измерьте (через `BenchmarkDotNet` или ручной `Stopwatch`) разницу во времени между `Distance(in BigPoint, in BigPoint)` и `Distance(BigPoint, BigPoint)` по значению на 1 000 000 вызовов. Объясните, при каком размере структуры `in` начинает давать выигрыш.

---

## Statement in English / Постановка на английском

#### Context & motivation
You have joined a team building an internal utility library `MiniUtils` for a console application that processes text reports and three-dimensional coordinates. The project architect insists that every function uses exactly the kind of parameter that best expresses intent: not “everything via `ref`”, but a deliberate choice between `ref`, `out`, `in`, `params`, and default values. This is not an academic demand — the choice drives both performance (copies of large structs), readability of call sites (is it obvious that a method mutates its argument), and API resilience to future changes (binary compatibility of defaults).

Concretely, the library must be able to compute the distance between heavy point structs without copying them; parse hexadecimal strings with a success signal; analyze text and return word, character, and line counts simultaneously; normalize an angle into `[0, 360)` and swap values; log a variable number of messages; and produce report parameters with sensible defaults. Each of these scenarios maps cleanly onto exactly one modifier from the lesson, and your job is to implement them so the code compiles without warnings, is idiomatic for C# 12 / .NET 8, and avoids the common mistakes enumerated in the lesson.

It is crucial to keep the contrast with pass-by-value in mind: a plain `int` is copied, so reassigning the parameter inside the method is invisible outside; for a large struct a copy on every call is expensive — this is exactly where `in` saves cost without mutation risk. For reference types the reference is copied, but reassigning the parameter to a new object is again invisible to the caller. These distinctions are the foundation on which the choice of modifier rests.

#### What to do step by step
1. Create the project: `dotnet new console -n MiniUtils -o MiniUtils` (C# 12 / .NET 8 by default). Move into the folder: `cd MiniUtils`. Open `Program.cs` — top-level statements will live here.
2. Define a `readonly struct BigPoint(double x, double y, double z)` with three init-only properties `X`, `Y`, `Z` and a computed `NormSquared`. This struct is “heavy” (24+ bytes) — a candidate for `in`.
3. In a static class `Geometry`, implement `double Distance(in BigPoint a, in BigPoint b)` computing the square root of the sum of squared coordinate deltas. Verify that attempting `a.X = 0;` produces a compile error (CS8331/CS0204) — comment it out with an explanation.
4. In a static class `TextOps`, implement `bool TryParseHex(string s, out int value)`: the method must assign `value` on every return path. Strings like `"0xFF"` and `"ff"` must parse to `255`; an empty or null string returns `false`. Use `int.TryParse` with `NumberStyles.HexNumber`.
5. In the same `TextOps`, add `void Analyze(string text, out int words, out int chars, out int lines)`. The method must assign all three `out` parameters. Split words via `Split` with `StringSplitOptions.RemoveEmptyEntries | TrimEntries`; split lines on `'\n'`. Handle `null`/empty gracefully (zero everywhere).
6. In a static class `NumOps`, implement `void NormalizeAngle(ref double angle)` mapping the angle into `[0, 360)` via `% 360.0` and adding `360.0` when the result is negative. Add `void Swap(ref int a, ref int b)` using tuple deconstruction `(a, b) = (b, a)`.
7. In a static class `Logger`, implement `void Log(string level, params string[] messages)` and `int Sum(params int[] numbers)`. The `params` parameter must be the last one. Demonstrate a call with a collection expression `Sum([10, 20, 30])`.
8. In a static class `Report`, implement `void Print(int page = 1, string title = "Untitled", bool landscape = false)`. Defaults must be compile-time constants.
9. In `Program.cs`, after the classes, write a demo block using top-level statements: create the points, call `Distance(in p1, in p2)`, parse `"0xFF"`, analyze a multi-line text, normalize the angle `740.0`, swap `1` and `2`, log three messages, sum a collection, and call `Print()` in all four styles from the lesson (all defaults, positional, named, mixed any-order).
10. Run `dotnet run` and compare the output with the expected: `Distance: 3.000`, `Hex: 255`, `Words=5, Chars=..., Lines=2`, `Angle: 20`, `Swap: a=2, b=1`, `Sum: 15`, `Sum(collection): 60`, and four report lines.
11. Temporarily break the rules to see the compiler errors: drop `ref` at the `Swap` call site; pass an uninitialized variable to `ref`; try to mutate an `in` parameter; place `params` somewhere other than last. Record the error codes in a comment, then restore the code.

#### Requirements
- Target framework `net8.0`, language C# 12, top-level statements in `Program.cs`; the code compiles without errors or `nullable enable`-level warnings.
- Each modifier is applied exactly where it expresses intent: `in` for `BigPoint`, `out` for parsing and analysis, `ref` for normalization and swap, `params` for the logger and sum, defaults for the report.
- The `ref`/`out`/`in` modifiers appear both in the method signature and at the call site — a key lesson requirement.
- Variables passed to `ref` and `in` are initialized before the call; for `out` they are declared inline via `out int x` (C# 7+) or assigned outside.
- All `out` parameters are guaranteed assigned on every return path (including early `return false`).
- The `params` parameter is exactly one and is the last in the method's parameter list.
- Default values are compile-time constants (`1`, `"Untitled"`, `false`).
- C# 12 features are used: collection expressions `[10, 20, 30]`, tuple deconstruction in `Swap`, the primary constructor `readonly struct BigPoint(double x, double y, double z)`.
- The demo output matches expectations; number formatting is `F3` for distance, integer for the angle and sum.
- A comment records the compiler error codes for the three intentional violations (step 11) — proof that you can see the compiler guarding the contract.

#### Pitfalls
- **Modifier on both sides.** `ref`/`out`/`in` must appear in both the method declaration and the call. Forget `ref` at the `Swap(ref a, ref b)` call and the compiler reports an error because the signature demands a reference. This is not decoration but part of the contract: the caller explicitly acknowledges the mutation.
- **Initialization before the call for `ref` and `in`.** `ref` and `in` require the variable to already hold a value. Passing an uninitialized variable to `ref` is an error. If there is no value yet, that is a signal to use `out`: the method is obliged to assign it.
- **`out` must assign.** The compiler verifies that every `out` parameter is assigned on every return path, including early `return false`. This protects the caller from reading an undefined value. The pattern: write `value = 0;` (or `default`) as the first line, then overwrite on success.
- **`in` cannot be modified.** Attempting `a.X = 0;` inside `Distance(in BigPoint a, ...)` is a compile error. If mutation is genuinely needed, switch the modifier to `ref`. `in` is optimized for reading large structs without copies; for small ones (up to ~16 bytes) it adds noise without benefit.
- **`params` only last and only one.** `void Log(params string[] m, int x)` is an error. Multiple `params` are also an error. You can pass an array directly or via a collection expression `[...]`; the compiler packs listed arguments into the array itself.
- **Binary compatibility of defaults.** A default value is “baked into” the caller's call site. If you change `title = "Untitled"` to `title = "Draft"` in a new library version, recompiled clients see `"Draft"`, but old binaries keep passing `"Untitled"`. In public library APIs prefer overloads over defaults; defaults are safe inside a single solution.
- **Named arguments to skip the middle.** `Print(landscape: true)` skips `page` and `title`, taking their defaults — the idiomatic way to call default-bearing APIs. Any order is allowed, but mixing positional and named requires positional arguments to come first.
- **Pass-by-value as the baseline.** A plain `int n` is copied: `n += 100` inside the method does not change the original. For a reference type the reference is copied: field mutations are visible outside, but reassigning the parameter to a new object is not. This contrast is the reason `ref`/`out`/`in` exist at all.

#### Acceptance criteria
- [ ] Project `MiniUtils` created on `net8.0`/C# 12; `dotnet build` is error- and warning-free.
- [ ] `readonly struct BigPoint` with a primary constructor and properties `X`, `Y`, `Z`, `NormSquared`.
- [ ] `Geometry.Distance(in BigPoint, in BigPoint)` computes the correct distance; `in` modifiers present in signature and at the call site.
- [ ] The attempted mutation of the `in` parameter is commented out with the compile-error code noted.
- [ ] `TextOps.TryParseHex(string, out int)` returns `true`/`255` for `"0xFF"` and `"ff"`, `false` for empty/null.
- [ ] All `out` parameters are assigned on every return path (including early `return false`).
- [ ] `TextOps.Analyze(string, out int, out int, out int)` correctly counts words, characters, lines; handles `null`/empty.
- [ ] `NumOps.NormalizeAngle(ref double)` maps `740.0` to `20.0` and `-45.0` to `315.0`; `ref` on both sides.
- [ ] `NumOps.Swap(ref int, ref int)` swaps values via tuple deconstruction.
- [ ] `Logger.Log(string, params string[])` has `params` last; a three-argument call works.
- [ ] `Logger.Sum(params int[])` is called both with a list and with a collection expression `[10, 20, 30]` (result `60`).
- [ ] `Report.Print(int=1, string="Untitled", bool=false)` is called in four ways: all defaults, positional, named, mixed any-order.
- [ ] Default values are compile-time constants.
- [ ] The `dotnet run` output matches expectations across all demo points.
- [ ] Compiler error codes for the three intentional violations are recorded in comments (no `ref` at the call; uninitialized variable to `ref`; `params` not last).

#### Hints (no direct answer)
- Think about which modifier expresses “the method must return a result and signal success” — the classic `int.TryParse` scenario from the lesson.
- For `in`, recall the lesson's analogy: “a document for reading with no edit rights.” Where would the compiler physically stop you from writing?
- To guarantee `out` assignment, set the default on the first line of the body — then any early `return` is already safe.
- For `params` and collections, recall C# 12 collection-expression syntax: `[1, 2, 3]` implicitly converts to `int[]`.
- Defaults are baked into the call site — ask yourself what an old client would see if you changed a default in a new library version.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — MiniUtils: homework M03-L06
// Run: dotnet run
using System;
using System.Globalization;

// Large readonly struct — candidate for `in`.
readonly struct BigPoint(double x, double y, double z)
{
    public double X { get; } = x;     // init-only, immutable after construction
    public double Y { get; } = y;
    public double Z { get; } = z;
    public double NormSquared => X * X + Y * Y + Z * Z;
}

static class Geometry
{
    // in — ref readonly: read heavy structs without copies.
    public static double Distance(in BigPoint a, in BigPoint b)
    {
        // a.X = 0; // CS8331: cannot modify an in-parameter
        double dx = a.X - b.X;
        double dy = a.Y - b.Y;
        double dz = a.Z - b.Z;
        return Math.Sqrt(dx * dx + dy * dy + dz * dz);
    }
}

static class TextOps
{
    // out — the method must assign `value` before any return.
    public static bool TryParseHex(string s, out int value)
    {
        value = 0;                                              // must assign
        if (string.IsNullOrWhiteSpace(s)) return false;
        if (s.StartsWith("0x", StringComparison.OrdinalIgnoreCase))
            s = s[2..];                                         // range slice, C# 8+
        return int.TryParse(s, NumberStyles.HexNumber, CultureInfo.InvariantCulture, out value);
    }

    // Multiple out — multiple return values without a tuple.
    public static void Analyze(string text, out int words, out int chars, out int lines)
    {
        words = string.IsNullOrEmpty(text) ? 0
            : text.Split(' ', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries).Length;
        chars = text?.Length ?? 0;
        lines = string.IsNullOrEmpty(text) ? 0 : text.Split('\n').Length;
    }
}

static class NumOps
{
    // ref — two-way channel: read and mutate an existing value.
    public static void NormalizeAngle(ref double angle)
    {
        angle %= 360.0;                // remainder
        if (angle < 0) angle += 360.0; // wrap negative into positive
    }

    // C# 12 tuple deconstruction swap.
    public static void Swap(ref int a, ref int b) => (a, b) = (b, a);
}

static class Logger
{
    // params — variable number of arguments, must be last.
    public static void Log(string level, params string[] messages)
        => Console.WriteLine($"[{level}] {string.Join(" | ", messages)}");

    public static int Sum(params int[] numbers)
    {
        int total = 0;
        foreach (var n in numbers) total += n;
        return total;
    }
}

static class Report
{
    // Defaults are compile-time constants.
    public static void Print(int page = 1, string title = "Untitled", bool landscape = false)
        => Console.WriteLine($"Report '{title}', page {page}, landscape={landscape}");
}

// --- Demo via top-level statements ---
var p1 = new BigPoint(0, 0, 0);
var p2 = new BigPoint(1, 2, 2);
Console.WriteLine($"Distance: {Geometry.Distance(in p1, in p2):F3}");   // 3.000

if (TextOps.TryParseHex("0xFF", out int hex))
    Console.WriteLine($"Hex: {hex}");                                    // 255

TextOps.Analyze("C# params are powerful\nSecond line", out int w, out int c, out int l);
Console.WriteLine($"Words={w}, Chars={c}, Lines={l}");                   // Words=5, Lines=2

double angle = 740.0;
NumOps.NormalizeAngle(ref angle);
Console.WriteLine($"Angle: {angle}");                                    // 20

int a = 1, b = 2;
NumOps.Swap(ref a, ref b);
Console.WriteLine($"Swap: a={a}, b={b}");                                // a=2, b=1

Logger.Log("INFO", "boot", "ready", "ok");                               // params of 3 messages
Console.WriteLine($"Sum: {Logger.Sum(1, 2, 3, 4, 5)}");                  // 15
Console.WriteLine($"Sum(collection): {Logger.Sum([10, 20, 30])}");       // 60 — collection expression

Report.Print();                                                          // all defaults
Report.Print(5);                                                         // positional
Report.Print(title: "Sales Q4");                                         // named
Report.Print(landscape: true, title: "Wide", page: 2);                   // mixed, any order
```

Line-by-line walk-through. The `readonly struct BigPoint` is declared via a C# 12 primary constructor: coordinates become init-only properties immediately, the struct is immutable — a mandatory precondition for `in`, otherwise the compiler would insert a defensive copy and the reference benefit would vanish. `Geometry.Distance(in BigPoint, in BigPoint)` passes the points by read-only reference: copies of 24+ bytes per call are avoided; the commented-out attempt `a.X = 0` shows the compiler physically blocking mutation (CS8331) — this is precisely the `in` contract from the lesson. `TryParseHex(string, out int value)` assigns `value = 0` on the first line, so any early `return false` is safe: the compiler is satisfied, and the caller never reads an undefined value. The `s[2..]` slice strips the `0x` prefix; `int.TryParse` with `NumberStyles.HexNumber` does the heavy lifting. `Analyze` with three `out` parameters is the lesson's example of multiple return values without a tuple: the method must assign all three before returning, which it does via ternary expressions guarded against `null`. `NormalizeAngle(ref double)` is a two-way channel: the method both reads and mutates `angle`, and the change is visible to the caller; `ref` appears in both the signature and the call. `Swap` via `(a, b) = (b, a)` uses tuple deconstruction — a C# 12 idiom from the lesson, shorter than a temporary variable. `Logger.Log(string, params string[])` shows that `params` must be last: `level` comes first as required, followed by a variable number of messages packed into an array. `Sum([10, 20, 30])` is a C# 12 collection expression implicitly convertible to `int[]`. `Report.Print` with three defaults and four call styles (all defaults, positional, named, mixed any-order) cements the combination of defaults and named arguments from the lesson. Finally, the intentional violations in step 11 (dropping `ref` at the call → CS1620/CS1503; an uninitialized variable to `ref` → CS0165; `params` not last → CS0231) demonstrate that the compiler stands guard over each modifier's contract.

#### Going deeper (bonus)
1. Rewrite `Analyze` to return a `(int Words, int Chars, int Lines)` tuple instead of three `out` parameters. Compare readability and discuss, per the lesson's best practices, when a tuple is preferable to `out`.
2. Add an overload `Print(string title)` without defaults and discuss why, in a public library API, overloads are safer than changing a default from a binary-compatibility standpoint. Show a scenario where an old binary “sees” a stale default.
3. Implement `void Translate(ref BigPoint p, double dx, double dy, double dz)` and explain why `ref` is more appropriate than `in` here, even though `BigPoint` is a large struct. When would `in` still be preferable?
4. Measure (via `BenchmarkDotNet` or a manual `Stopwatch`) the time difference between `Distance(in BigPoint, in BigPoint)` and pass-by-value `Distance(BigPoint, BigPoint)` over 1,000,000 calls. Explain at what struct size `in` starts to pay off.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `MiniUtils` собирается на `net8.0`/C# 12 без ошибок и предупреждений.
- [ ] (RU) Все пять модификаторов (`ref`, `out`, `in`, `params`, дефолты) применены в назначенных сценариях; модификаторы в сигнатуре и при вызове.
- [ ] (RU) Вывод `dotnet run` совпадает с ожидаемым по всем демонстрационным точкам.
- [ ] (RU) В комментариях зафиксированы номера ошибок компилятора для трёх намеренных нарушений.
- [ ] (RU) Файл `Program.cs` отправлен в репозиторий; README кратко описывает, какой модификатор где применён и почему.
- [ ] (EN) Project `MiniUtils` builds on `net8.0`/C# 12 with no errors or warnings.
- [ ] (EN) All five modifiers (`ref`, `out`, `in`, `params`, defaults) are applied in the assigned scenarios; modifiers present in both signature and call.
- [ ] (EN) The `dotnet run` output matches expectations across all demo points.
- [ ] (EN) Compiler error codes for the three intentional violations are recorded in comments.
- [ ] (EN) `Program.cs` is committed; a README briefly notes which modifier is used where and why.

#### Ресурсы / Resources
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/ref — Ключевое слово ref / ref keyword
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/method-parameters — Параметры методов (ref, out, in, params, значения по умолчанию) / Method parameters
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/named-and-optional-arguments — Именованные и необязательные аргументы / Named and optional arguments
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/in — Модификатор in (передача по ссылке только для чтения) / in modifier (ref readonly)
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/out — Модификатор out / out modifier
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/params — Модификатор params / params modifier
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-12 — Что нового в C# 12 (коллекции-выражения, primary-конструкторы) / What's new in C# 12 (collection expressions, primary constructors)

---
[← К уроку M03-L06](lesson-M03-L06-ref-out-in-params.md) | [⬆ К модулю M03](../README.md) | [Следующее ДЗ →](homework-M03-L07-overloading-scope.md)
---
