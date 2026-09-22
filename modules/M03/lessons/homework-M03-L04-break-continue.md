---
[← К уроку M03-L04](lesson-M03-L04-break-continue.md) | [⬆ К модулю M03](../README.md) | [Следующее ДЗ →](homework-M03-L05-methods.md)
---

### Домашнее задание M03-L04: break/continue/return / Homework M03-L04: break/continue/return

**Урок / Lesson:** M03-L04
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться осознанно выбирать между `break`, `continue`, `return` и `throw`, грамотно выходить из циклов и вложенных конструкций, применять ранний возврат для снижения вложенности и понимать, где современный C# 12 (pattern matching, `switch` expression, top-level statements) заменяет классические jump-конструкции. (EN) Learn to consciously choose between `break`, `continue`, `return`, and `throw`, exit loops and nested constructs correctly, apply early return to reduce nesting, and understand where modern C# 12 (pattern matching, `switch` expression, top-level statements) replaces classic jump statements.

#### Связь с уроком / Connection to the lesson
(RU) ДЗ напрямую закрепляет три ключевых оператора урока — `break`, `continue`, `return` — а также их отличие от `throw`. Вы будете применять ранний `return` вместо глубокой вложенности, практиковать выход из вложенных циклов через вынос метода, использовать `continue` для фильтрации и `break` для поиска, а также работать с `switch` expression, который не требует `break`. (EN) The homework directly reinforces the three key statements from the lesson — `break`, `continue`, `return` — and their difference from `throw`. You will apply early `return` instead of deep nesting, practice exiting nested loops by extracting a method, use `continue` for filtering and `break` for search, and work with the `switch` expression that needs no `break`.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Представьте, что вы пишете небольшой модуль анализа логов для сервера онлайн-игры. Логи поступают потоково и сохраняются в памяти как массивы и списки строк. Вам нужно уметь быстро находить первое сообщение об ошибке, подсчитывать суммарную «длительность» сессий, фильтровать «шумовые» записи и корректно выходить из многоуровневого перебора, как только найдено нужное совпадение. Все эти задачи идеально ложатся на операторы перехода `break`, `continue` и `return`, изученные в уроке M03-L04.

Главное, что должно дойти до мышечной памяти: `break` завершает ближайший цикл или ветку `switch`, `continue` пропускает текущую итерацию, а `return` выходит из метода целиком и, в отличие от `throw`, не несёт семантики ошибки. Это различие критично: если «элемент не найден» — это ожидаемый результат поиска, мы возвращаем `-1`, `null` или `false`, а не бросаем исключение. Исключения медленные и ломают читаемость потока выполнения. Урок также подчёркивает, что для выхода из вложенных циклов лучше вынести внутренний цикл в отдельный метод и использовать `return`, а не увлекаться `goto`.

В этом задании вы построите консольное приложение на C# 12 / .NET 8, использующее top-level statements, современные коллекционные выражения и pattern matching. Каждая функция должна быть небольшой, с ранним возвратом, без «спагетти» из множества переходов. Вы получите набор тестовых данных и точные ожидаемые выводы, чтобы сразу видеть, корректно ли работают ваши операторы перехода.

#### Что нужно сделать (пошагово)

1. **Создайте проект.** В терминале выполните `dotnet new console -n LogAnalyzer -o LogAnalyzer --framework net8.0`. Перейдите в каталог: `cd LogAnalyzer`. Убедитесь, что `LogAnalyzer.csproj` содержит `<TargetFramework>net8.0</TargetFramework>` и при желании добавьте `<Nullable>enable</Nullable>` и `<LangVersion>latest</LangVersion>`.
2. **Структура файлов.** Откройте `Program.cs`. Так как мы используем top-level statements, файл `Program.cs` будет точкой входа. В том же файле объявите вспомогательные локальные функции или статические методы ниже основной последовательности операторов.
3. **Подготовьте данные.** Создайте массив строк-логов с помощью коллекционного выражения: `string[] logs = ["INFO Start", "WARN slow query", "ERROR disk full", "INFO retry", "ERROR timeout", "INFO done"];`. Также подготовьте «матрицу сессий» — массив массивов целых чисел: `int[][] sessions = [[10, 20, 30], [5, 15], [0, 25, 40], [40]];`.
4. **Функция FindFirstError.** Реализуйте метод `string? FindFirstError(string[] logs)`, который перебирает массив и возвращает первую строку, начинающуюся с `"ERROR"`. Используйте ранний `return` внутри цикла. Если ошибки нет — верните `null`. Выведите результат: `First error: ERROR disk full`.
5. **Функция SumValidSessions.** Реализуйте `int SumValidSessions(int[][] sessions)`, которая суммирует все ненулевые длительности, пропуская нули через `continue`. Ожидаемый результат для данных выше: `10+20+30+5+15+25+40+40 = 185`. Вывод: `Sum of valid sessions: 185`.
6. **Функция ContainsSessionLongerThan.** Реализуйте `bool ContainsSessionLongerThan(int[][] sessions, int threshold)`, которая возвращает `true`, как только найдёт хотя бы одно значение больше `threshold`. Должен произойти ранний выход из обоих уровней цикла через `return`. Проверьте: `ContainsSessionLongerThan(sessions, 35)` → `True`; `ContainsSessionLongerThan(sessions, 100)` → `False`.
7. **Функция ClassifyMessage.** Реализуйте `string ClassifyMessage(string log)` через `switch` expression (без `break`). Правила: строки, начинающиеся с `"ERROR"` → `"Critical"`; `"WARN"` → `"Warning"`; `"INFO"` → `"Info"`; иначе → `"Unknown"`. Используйте pattern matching с `when` или метод `StartsWith`.
8. **Функция DivideSafe.** Реализуйте `int DivideSafe(int a, int b)`, которая бросает `DivideByZeroException` при `b == 0` (это исключительная ситуация) и возвращает `a / b` иначе. Покажите вызов в `try/catch` и вывод `Caught: Divisor is zero.`.
9. **Демонстрация во `switch` statement.** Добавьте классический `switch` statement по `DayOfWeek`, где каждая непустая ветка завершается `break`. Продемонстрируйте, что отсутствие `break` в непустой ветке — ошибка компиляции (попробуйте временно убрать `break` и убедитесь, что `dotnet build` падает с CS0163).
10. **Сборка и запуск.** Выполните `dotnet build`, затем `dotnet run`. Зафиксируйте точный вывод в отчёте `report.txt`. Он должен соответствовать эталонным строкам, приведённым выше.
11. **Рефлексия по throw vs return.** В комментарии в начале `Program.cs` (2–3 строки) объясните, почему `FindFirstError` возвращает `null`, а `DivideSafe` бросает исключение, опираясь на правило из урока: ожидаемый результат — `return`, исключительная ситуация — `throw`.

#### Требования к решению

- Целевая платформа — .NET 8, язык C# 12. Используйте top-level statements в `Program.cs`; явный `class Program` и `static void Main` писать не нужно.
- Все функции должны быть небольшими (до ~15 строк тела) и использовать ранний `return` там, где это уменьшает вложенность. Избегайте глубоко вложенных `if-else` лестниц.
- В `FindFirstError` обязательно используйте ранний `return` внутри цикла; не подсчитывайте результат отдельным флагом, если можно выйти сразу.
- В `SumValidSessions` обязательно используйте `continue` для пропуска нулей — это намеренная демонстрация оператора.
- В `ContainsSessionLongerThan` продемонстрируйте выход из вложенных циклов через `return` из метода (а не `goto` и не двойной `break` с флагом).
- `ClassifyMessage` должен быть реализован именно через `switch` expression, а не через серию `if-else` или классический `switch` statement.
- `DivideSafe` должен кидать именно `DivideByZeroException` (не `ArgumentException` и не возвращать специальное значение), потому что деление на ноль — исключительная ситуация.
- Включите `Nullable` enable и корректно возвращайте `null` из `FindFirstError`; компилятор не должен выдавать предупреждений уровня error.
- Код должен собираться без предупреждений (`dotnet build` с `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` по желанию), а вывод `dotnet run` — точно совпадать с эталоном.
- Все комментарии в коде — двуязычные (RU+EN), как в примерах урока.

#### Тонкости и подводные камни

- **`break` выходит только из самого внутреннего цикла.** Если вы напишете `break` внутри внутреннего `foreach`, внешний цикл продолжит выполняться. Это самая частая ошибка при работе с матрицами. Чтобы выйти из обоих уровней сразу, вынесите логику в метод и используйте `return`, как требует `ContainsSessionLongerThan`. Не пытайтесь компенсировать это флагом `found` и проверкой во внешнем цикле — это менее читаемо.
- **`continue` не работает в `switch`.** Если вы попытаетесь использовать `continue` внутри ветки `switch` для перехода к следующей ветке, компилятор сообщит об ошибке или поведение будет не тем. `continue` предназначен только для циклов. В современном C# ветки `switch` и так не «проваливаются».
- **`switch` expression не требует `break`.** Это частый источник путаницы: студенты, привыкшие к классическому `switch`, пытаются добавить `break` в `switch` expression — это синтаксическая ошибка. Запомните: `switch` statement → `break` обязателен в непустой ветке; `switch` expression → `break` не нужен и даже запрещён.
- **Забытый `break` в `switch` statement.** В C# нет неявного fall-through, но непустая ветка без `break`/`return`/`goto`/`throw` вызывает ошибку компиляции CS0163. В пустой ветке (где несколько `case` идут подряд без кода) fall-through разрешён, но как только появляется оператор — нужен явный выход.
- **`return` в `void`-методе.** Если метод объявлен `void`, можно написать только `return;` без значения. `return expression;` в `void`-методе — ошибка компиляции. В top-level statements `return;` тоже допустим и завершает точку входа.
- **Не используйте `throw` как «особый» результат.** Если `FindFirstError` не нашёл ошибку — это нормальный, ожидаемый результат поиска, возвращайте `null`. Бросать `InvalidOperationException("not found")` — антипаттерн: это медленно, скрывает поток выполнения и заставляет вызывающего писать `try/catch` для обыденной логики.
- **`throw` для действительно исключительных ситуаций.** Деление на ноль, отсутствие обязательного файла, нарушение инварианта — это `throw`. Урок ясно разделяет: ожидаемое — `return`, исключительное — `throw`.
- **`goto` для выхода из вложенных циклов.** Урок допускает `goto` с меткой для выхода из многоуровневых циклов, но рекомендует применять редко и с комментариями. В этом ДЗ вы должны предпочесть `return` из метода — это чище и идиоматичнее.
- **Pattern matching заменяет `switch` + `break`.** Где результат — значение, используйте `switch` expression (`ClassifyMessage`). Где нужна побочная логика или несколько операторов — классический `switch` statement с `break`.

#### Критерии приёмки

- [ ] Проект `LogAnalyzer` создан через `dotnet new console` с `--framework net8.0`, собирается без ошибок.
- [ ] `Program.cs` использует top-level statements, без явного `static void Main`.
- [ ] Коллекционные выражения (`[...]`) применены для инициализации `logs` и `sessions`.
- [ ] `FindFirstError` возвращает `null`, если ошибки нет, и использует ранний `return` внутри цикла.
- [ ] `SumValidSessions` использует `continue` для пропуска нулей и возвращает `185` для тестовых данных.
- [ ] `ContainsSessionLongerThan` выходит из вложенных циклов через `return` (не `goto`, не флаг+`break`).
- [ ] `ClassifyMessage` реализован через `switch` expression без `break`.
- [ ] `DivideSafe` бросает `DivideByZeroException` при `b == 0` и оборачивается в `try/catch` в `Main`.
- [ ] Классический `switch` statement по `DayOfWeek` присутствует, каждая ветка завершается `break`.
- [ ] Вывод `dotnet run` точно соответствует эталонным строкам (`First error: ERROR disk full`, `Sum of valid sessions: 185`, `True`, `False`, `Caught: Divisor is zero.` и т. д.).
- [ ] Временное удаление `break` из непустой ветки `switch` statement приводит к ошибке компиляции CS0163 (зафиксируйте в отчёте).
- [ ] Включён `Nullable` enable, нет предупреждений компилятора уровня error.
- [ ] Комментарии в коде двуязычные (RU+EN), как в примерах урока.
- [ ] В начале `Program.cs` есть комментарий-рефлексия (2–3 строки) о выборе `return` vs `throw`.
- [ ] Каждая функция занимает не более ~15 строк и использует ранний `return` для снижения вложенности.
- [ ] Файл `report.txt` содержит точный вывод программы и подтверждение ошибки CS0163.

#### Подсказки (без прямого ответа)

- Для проверки префикса строки используйте `log.StartsWith("ERROR", StringComparison.Ordinal)` — это быстрее и яснее, чем `Substring`.
- В `switch` expression можно комбинировать паттерны: `s when s.StartsWith("ERROR") => "Critical"`.
- Чтобы выйти из двух циклов сразу, спросите себя: «можно ли вернуть результат прямо из внутреннего цикла?». Если функция уже возвращает `bool` — `return true` выйдет из всего метода.
- Для `SumValidSessions` помните: `continue` пропускает остаток тела итерации; всё, что после `continue`, не выполнится для текущего элемента.
- Не путайте `return` и `break` в `DivideSafe`: после `throw` код недостижим, но `throw` — это не `return`, это исключение.
- В top-level statements порядок операторов имеет значение: сначала using, потом данные, потом вызовы, потом объявления функций — или наоборот, но без конфликтов.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — top-level statements
// Модуль анализа логов: демонстрация break/continue/return и отличия от throw.
// Log analysis module: demonstration of break/continue/return and the difference from throw.
//
// Рефлексия / Reflection:
// FindFirstError возвращает null, потому что "ошибки нет" — ожидаемый результат поиска (return).
// FindFirstError returns null because "no error" is an expected search result (return).
// DivideSafe бросает исключение, потому что деление на ноль — исключительная ситуация (throw).
// DivideSafe throws because division by zero is an exceptional condition (throw).

using System;
using System.Collections.Generic;
using System.Linq;

// --- Данные / Data ---
string[] logs = ["INFO Start", "WARN slow query", "ERROR disk full", "INFO retry", "ERROR timeout", "INFO done"];
int[][] sessions = [[10, 20, 30], [5, 15], [0, 25, 40], [40]];

// --- Демонстрация / Demo ---
Console.WriteLine($"First error: {FindFirstError(logs) ?? "(none)"}");          // ERROR disk full
Console.WriteLine($"Sum of valid sessions: {SumValidSessions(sessions)}");       // 185
Console.WriteLine($"Contains >35: {ContainsSessionLongerThan(sessions, 35)}");   // True
Console.WriteLine($"Contains >100: {ContainsSessionLongerThan(sessions, 100)}"); // False
Console.WriteLine($"Classify 'ERROR timeout': {ClassifyMessage("ERROR timeout")}"); // Critical
Console.WriteLine($"Day: {DescribeDay(DayOfWeek.Saturday)}");                    // Weekend

try
{
    Console.WriteLine(DivideSafe(10, 0));
}
catch (DivideByZeroException ex)
{
    Console.WriteLine($"Caught: {ex.Message}");                                  // Caught: Divisor is zero.
}

// --- Реализации / Implementations ---

// Ранний return внутри цикла: как только нашли ERROR — выходим из метода целиком.
// Early return inside the loop: as soon as we find ERROR — we leave the method entirely.
static string? FindFirstError(string[] logs)
{
    foreach (var line in logs)
    {
        if (line.StartsWith("ERROR", StringComparison.Ordinal))
        {
            return line;   // ранний возврат / early return
        }
    }
    return null;           // ожидаемый результат "не найдено" / expected "not found" result
}

// continue пропускает нули; sum накапливает только ненулевые длительности.
// continue skips zeros; sum accumulates only non-zero durations.
static int SumValidSessions(int[][] sessions)
{
    int sum = 0;
    foreach (var row in sessions)
    {
        foreach (var cell in row)
        {
            if (cell == 0)
            {
                continue;  // пропускаем нули / skip zeros
            }
            sum += cell;
        }
    }
    return sum;
}

// return изнутри вложенного цикла выходит из обоих уровней сразу — чище, чем goto или флаг.
// return from inside a nested loop exits both levels at once — cleaner than goto or a flag.
static bool ContainsSessionLongerThan(int[][] sessions, int threshold)
{
    foreach (var row in sessions)
    {
        foreach (var cell in row)
        {
            if (cell > threshold)
            {
                return true;  // выход из обоих циклов / exit from both loops
            }
        }
    }
    return false;
}

// switch expression: break НЕ нужен и даже запрещён.
// switch expression: break is NOT needed and is in fact forbidden.
static string ClassifyMessage(string log) => log switch
{
    string s when s.StartsWith("ERROR", StringComparison.Ordinal) => "Critical",
    string s when s.StartsWith("WARN", StringComparison.Ordinal)  => "Warning",
    string s when s.StartsWith("INFO", StringComparison.Ordinal)  => "Info",
    _ => "Unknown"
};

// Классический switch statement: каждая непустая ветка завершается break.
// Classic switch statement: every non-empty section ends with break.
static string DescribeDay(DayOfWeek day)
{
    switch (day)
    {
        case DayOfWeek.Saturday:
        case DayOfWeek.Sunday:
            return "Weekend";   // return тоже допустим как выход из ветки / return is also valid as a section exit
        default:
            return "Workday";
    }
    // Здесь break не нужен, т.к. все ветки вышли через return; но если бы были побочные действия — нужен break.
    // No break needed here because all sections exit via return; but with side effects you would need break.
}

// throw для исключительной ситуации; return для нормального результата.
// throw for an exceptional condition; return for a normal result.
static int DivideSafe(int a, int b)
{
    if (b == 0)
    {
        throw new DivideByZeroException("Divisor is zero.");
    }
    return a / b;
}
```

Разбор по строкам. Функция `FindFirstError` демонстрирует главный приём урока — ранний `return` внутри цикла: вместо того, чтобы копить результат во флаге и проверять его после цикла, мы выходим из метода сразу, как только нашли первую строку с префиксом `ERROR`. Возврат `null` — это пример правила «ожидаемый результат — return, не throw»: отсутствие ошибки в логах — нормальная ситуация, а не исключительная. Функция `SumValidSessions` показывает `continue`: когда `cell == 0`, мы пропускаем остаток тела итерации и переходим к следующему элементу, благодаря чему `sum += cell` не выполняется для нулей. Здесь же важно помнить, что `continue` действует на ближайший охватывающий цикл, поэтому для двойного перебора он корректно продолжает именно внутренний `foreach`. Функция `ContainsSessionLongerThan` — ключевой пример выхода из вложенных циклов: `return true` из внутреннего цикла покидает метод целиком, выходя из обоих уровней разом. Урок прямо рекомендует этот приём вместо `goto` или флага, потому что он чище и идиоматичнее. `ClassifyMessage` использует `switch` expression с pattern matching (`when`): это современная замена классическому `switch` + `break`, и `break` здесь не нужен и даже запрещён синтаксисом. `DescribeDay` намеренно написана как классический `switch` statement, чтобы показать, что `return` тоже является допустимым способом завершить ветку (альтернатива `break`); если бы ветка содержала побочные действия без `return`, потребовался бы `break`, иначе — CS0163. Наконец, `DivideSafe` иллюстрирует границу между `return` и `throw`: деление на ноль — исключительная ситуация, поэтому мы бросаем `DivideByZeroException`, а не возвращаем `-1` или `null`; вызывающая сторона оборачивает вызов в `try/catch`. Все функции небольшие и используют ранний выход, что соответствует best practice из урока: меньше вложенность — выше читаемость.

#### Задания на углубление (бонус)

1. **LINQ вместо явных циклов.** Перепишите `FindFirstError` через `logs.FirstOrDefault(l => l.StartsWith("ERROR"))` и `ContainsSessionLongerThan` через `sessions.SelectMany(r => r).Any(c => c > threshold)`. Сравните читаемость и обсудите, в каких случаях LINQ «ранний выход» заменяет `break`/`return`.
2. **Измерение производительности throw vs return.** Напишите бенчмарк (через `Stopwatch`), который вызывает «поисковую» функцию в двух вариантах: возвращающую `-1` при отсутствии и бросающую исключение. Запустите в цикле 1 000 000 раз для случая «не найдено» и замерьте разницу во времени. Сделайте вывод.
3. **Многоуровневый выход через `goto`.** Реализуйте вариант `ContainsSessionLongerThan` с `goto foundLabel;` и сравните читаемость с версией через `return`. Объясните в комментарии, почему урок рекомендует `return`.
4. **Обработка `IEnumerable` вместо массивов.** Измените сигнатуры на `IEnumerable<string>` и `IEnumerable<IEnumerable<int>>`. Подумайте, как изменится семантика раннего выхода (особенно для отложенных последовательностей).

---

## Statement in English / Постановка на английском

#### Context & motivation

Imagine you are writing a small log-analysis module for an online game server. Logs arrive in a streaming fashion and are kept in memory as arrays and lists of strings. You need to quickly find the first error message, compute the total "duration" of sessions, filter out "noise" records, and correctly exit a multi-level search as soon as the right match is found. All of these tasks map perfectly onto the jump statements `break`, `continue`, and `return` studied in lesson M03-L04.

The key insight that should settle into muscle memory is this: `break` terminates the nearest loop or `switch` section, `continue` skips the current iteration, and `return` exits the entire method and, unlike `throw`, carries no error semantics. This distinction is critical: if "item not found" is an expected result of a search, we return `-1`, `null`, or `false` rather than throwing an exception. Exceptions are slow and obscure the flow of the program. The lesson also stresses that to exit nested loops it is better to extract the inner loop into a separate method and use `return`, instead of leaning on `goto`.

In this assignment you will build a console application on C# 12 / .NET 8 that uses top-level statements, modern collection expressions, and pattern matching. Each function should be small, use early return, and avoid "spaghetti" of many jumps. You will be given a set of test data and precise expected outputs so you can immediately see whether your jump statements behave correctly.

#### What to do step by step

1. **Create the project.** In a terminal run `dotnet new console -n LogAnalyzer -o LogAnalyzer --framework net8.0`. Move into the folder: `cd LogAnalyzer`. Confirm that `LogAnalyzer.csproj` contains `<TargetFramework>net8.0</TargetFramework>`, and optionally add `<Nullable>enable</Nullable>` and `<LangVersion>latest</LangVersion>`.
2. **File structure.** Open `Program.cs`. Because we use top-level statements, this file is the entry point. In the same file, declare helper local functions or static methods below the main statement sequence.
3. **Prepare the data.** Create an array of log strings using a collection expression: `string[] logs = ["INFO Start", "WARN slow query", "ERROR disk full", "INFO retry", "ERROR timeout", "INFO done"];`. Also prepare a "session matrix" — an array of arrays of integers: `int[][] sessions = [[10, 20, 30], [5, 15], [0, 25, 40], [40]];`.
4. **Function FindFirstError.** Implement `string? FindFirstError(string[] logs)` that iterates the array and returns the first line starting with `"ERROR"`. Use an early `return` inside the loop. If there is no error, return `null`. Expected output: `First error: ERROR disk full`.
5. **Function SumValidSessions.** Implement `int SumValidSessions(int[][] sessions)` that sums all non-zero durations, skipping zeros with `continue`. For the data above the expected result is `10+20+30+5+15+25+40+40 = 185`. Output: `Sum of valid sessions: 185`.
6. **Function ContainsSessionLongerThan.** Implement `bool ContainsSessionLongerThan(int[][] sessions, int threshold)` that returns `true` as soon as it finds any value greater than `threshold`. This must be an early exit from both loop levels via `return`. Verify: `ContainsSessionLongerThan(sessions, 35)` → `True`; `ContainsSessionLongerThan(sessions, 100)` → `False`.
7. **Function ClassifyMessage.** Implement `string ClassifyMessage(string log)` using a `switch` expression (no `break`). Rules: lines starting with `"ERROR"` → `"Critical"`; `"WARN"` → `"Warning"`; `"INFO"` → `"Info"`; otherwise → `"Unknown"`. Use pattern matching with `when` or the `StartsWith` method.
8. **Function DivideSafe.** Implement `int DivideSafe(int a, int b)` that throws `DivideByZeroException` when `b == 0` (this is an exceptional condition) and returns `a / b` otherwise. Show the call inside a `try/catch` and the output `Caught: Divisor is zero.`.
9. **Demonstrate a `switch` statement.** Add a classic `switch` statement over `DayOfWeek` where every non-empty section ends with `break`. Demonstrate that removing `break` from a non-empty section is a compile error (temporarily drop `break` and confirm that `dotnet build` fails with CS0163).
10. **Build and run.** Run `dotnet build`, then `dotnet run`. Record the exact output into a `report.txt` file. It must match the reference lines listed above.
11. **Reflection on throw vs return.** In a comment at the top of `Program.cs` (2–3 lines), explain why `FindFirstError` returns `null` while `DivideSafe` throws, drawing on the lesson's rule: expected result — `return`; exceptional condition — `throw`.

#### Requirements

- Target platform is .NET 8, language C# 12. Use top-level statements in `Program.cs`; do not write an explicit `class Program` or `static void Main`.
- All functions must be small (up to ~15 lines of body) and use early `return` wherever it reduces nesting. Avoid deep `if-else` ladders.
- In `FindFirstError` you must use an early `return` inside the loop; do not accumulate the result in a flag if you can exit immediately.
- In `SumValidSessions` you must use `continue` to skip zeros — this is a deliberate demonstration of the statement.
- In `ContainsSessionLongerThan` you must exit nested loops via `return` from the method (not `goto`, not a flag plus double `break`).
- `ClassifyMessage` must be implemented as a `switch` expression, not as a chain of `if-else` or a classic `switch` statement.
- `DivideSafe` must throw `DivideByZeroException` (not `ArgumentException`, and not a sentinel value), because division by zero is an exceptional condition.
- Enable `Nullable` and return `null` from `FindFirstError` correctly; the compiler must not emit error-level warnings.
- The code must build without warnings (`dotnet build` with `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` if you wish), and the `dotnet run` output must exactly match the reference.
- All code comments must be bilingual (RU+EN), as in the lesson examples.

#### Pitfalls

- **`break` exits only the innermost loop.** If you write `break` inside an inner `foreach`, the outer loop keeps running. This is the most common mistake with matrices. To leave both levels at once, extract the logic into a method and use `return`, as required by `ContainsSessionLongerThan`. Do not compensate with a `found` flag checked in the outer loop — that is less readable.
- **`continue` does not work in `switch`.** If you try to use `continue` inside a `switch` section to move to the next section, the compiler will error or the behavior will be wrong. `continue` is only for loops. In modern C# `switch` sections do not fall through anyway.
- **A `switch` expression does not need `break`.** This is a frequent source of confusion: students used to the classic `switch` try to add `break` to a `switch` expression — that is a syntax error. Remember: `switch` statement → `break` is required in a non-empty section; `switch` expression → `break` is not needed and is even forbidden.
- **Forgotten `break` in a `switch` statement.** C# has no implicit fall-through, but a non-empty section without `break`/`return`/`goto`/`throw` triggers compile error CS0163. An empty section (several `case` labels in a row with no code) is allowed to fall through, but as soon as a statement appears, an explicit exit is required.
- **`return` in a `void` method.** If the method is declared `void`, you may only write `return;` with no value. `return expression;` in a `void` method is a compile error. In top-level statements `return;` is also allowed and ends the entry point.
- **Do not use `throw` as a "special" result.** If `FindFirstError` did not find an error, that is a normal, expected result of a search — return `null`. Throwing `InvalidOperationException("not found")` is an anti-pattern: it is slow, hides the flow of the program, and forces the caller to write `try/catch` for routine logic.
- **`throw` for genuinely exceptional conditions.** Division by zero, a missing mandatory file, a broken invariant — these are `throw`. The lesson draws a clear line: expected — `return`; exceptional — `throw`.
- **`goto` to exit nested loops.** The lesson permits `goto` with a label to escape multi-level loops but recommends using it rarely and with comments. In this homework you should prefer `return` from a method — it is cleaner and more idiomatic.
- **Pattern matching replaces `switch` + `break`.** Where the result is a value, use a `switch` expression (`ClassifyMessage`). Where you need side effects or multiple statements, use a classic `switch` statement with `break`.

#### Acceptance criteria

- [ ] The `LogAnalyzer` project is created with `dotnet new console` and `--framework net8.0`, and builds without errors.
- [ ] `Program.cs` uses top-level statements, with no explicit `static void Main`.
- [ ] Collection expressions (`[...]`) are used to initialize `logs` and `sessions`.
- [ ] `FindFirstError` returns `null` when there is no error and uses an early `return` inside the loop.
- [ ] `SumValidSessions` uses `continue` to skip zeros and returns `185` for the test data.
- [ ] `ContainsSessionLongerThan` exits nested loops via `return` (not `goto`, not a flag plus `break`).
- [ ] `ClassifyMessage` is implemented as a `switch` expression without `break`.
- [ ] `DivideSafe` throws `DivideByZeroException` when `b == 0` and is wrapped in `try/catch` in `Main`.
- [ ] A classic `switch` statement over `DayOfWeek` is present, and every section ends with `break`.
- [ ] The `dotnet run` output exactly matches the reference lines (`First error: ERROR disk full`, `Sum of valid sessions: 185`, `True`, `False`, `Caught: Divisor is zero.`, etc.).
- [ ] Temporarily removing `break` from a non-empty `switch` section produces compile error CS0163 (record this in the report).
- [ ] `Nullable` is enabled, with no error-level compiler warnings.
- [ ] Code comments are bilingual (RU+EN), as in the lesson examples.
- [ ] A reflection comment (2–3 lines) about choosing `return` vs `throw` is present at the top of `Program.cs`.
- [ ] Each function is no longer than ~15 lines and uses early `return` to reduce nesting.
- [ ] The `report.txt` file contains the exact program output and the confirmation of CS0163.

#### Hints (no direct answer)

- To check a string prefix, use `log.StartsWith("ERROR", StringComparison.Ordinal)` — it is faster and clearer than `Substring`.
- In a `switch` expression you can combine patterns: `s when s.StartsWith("ERROR") => "Critical"`.
- To exit two loops at once, ask yourself: "can I return the result straight from the inner loop?" If the function already returns `bool`, `return true` leaves the entire method.
- For `SumValidSessions`, remember that `continue` skips the rest of the iteration body; anything after `continue` does not run for the current element.
- Do not confuse `return` and `break` in `DivideSafe`: code after `throw` is unreachable, but `throw` is not `return` — it is an exception.
- In top-level statements, order matters: first `using`, then data, then calls, then function declarations — or the reverse, but without conflicts.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — top-level statements
// Log analysis module: demonstration of break/continue/return and the difference from throw.

using System;
using System.Collections.Generic;
using System.Linq;

// --- Data ---
string[] logs = ["INFO Start", "WARN slow query", "ERROR disk full", "INFO retry", "ERROR timeout", "INFO done"];
int[][] sessions = [[10, 20, 30], [5, 15], [0, 25, 40], [40]];

// --- Demo ---
Console.WriteLine($"First error: {FindFirstError(logs) ?? "(none)"}");          // ERROR disk full
Console.WriteLine($"Sum of valid sessions: {SumValidSessions(sessions)}");       // 185
Console.WriteLine($"Contains >35: {ContainsSessionLongerThan(sessions, 35)}");   // True
Console.WriteLine($"Contains >100: {ContainsSessionLongerThan(sessions, 100)}"); // False
Console.WriteLine($"Classify 'ERROR timeout': {ClassifyMessage("ERROR timeout")}"); // Critical
Console.WriteLine($"Day: {DescribeDay(DayOfWeek.Saturday)}");                    // Weekend

try
{
    Console.WriteLine(DivideSafe(10, 0));
}
catch (DivideByZeroException ex)
{
    Console.WriteLine($"Caught: {ex.Message}");                                  // Caught: Divisor is zero.
}

// --- Implementations ---

// Early return inside the loop: as soon as we find ERROR — we leave the method entirely.
static string? FindFirstError(string[] logs)
{
    foreach (var line in logs)
    {
        if (line.StartsWith("ERROR", StringComparison.Ordinal))
        {
            return line;   // early return
        }
    }
    return null;           // expected "not found" result
}

// continue skips zeros; sum accumulates only non-zero durations.
static int SumValidSessions(int[][] sessions)
{
    int sum = 0;
    foreach (var row in sessions)
    {
        foreach (var cell in row)
        {
            if (cell == 0)
            {
                continue;  // skip zeros
            }
            sum += cell;
        }
    }
    return sum;
}

// return from inside a nested loop exits both levels at once — cleaner than goto or a flag.
static bool ContainsSessionLongerThan(int[][] sessions, int threshold)
{
    foreach (var row in sessions)
    {
        foreach (var cell in row)
        {
            if (cell > threshold)
            {
                return true;  // exit from both loops
            }
        }
    }
    return false;
}

// switch expression: break is NOT needed and is in fact forbidden.
static string ClassifyMessage(string log) => log switch
{
    string s when s.StartsWith("ERROR", StringComparison.Ordinal) => "Critical",
    string s when s.StartsWith("WARN", StringComparison.Ordinal)  => "Warning",
    string s when s.StartsWith("INFO", StringComparison.Ordinal)  => "Info",
    _ => "Unknown"
};

// Classic switch statement: every non-empty section ends with break (or return).
static string DescribeDay(DayOfWeek day)
{
    switch (day)
    {
        case DayOfWeek.Saturday:
        case DayOfWeek.Sunday:
            return "Weekend";   // return is also valid as a section exit
        default:
            return "Workday";
    }
}

// throw for an exceptional condition; return for a normal result.
static int DivideSafe(int a, int b)
{
    if (b == 0)
    {
        throw new DivideByZeroException("Divisor is zero.");
    }
    return a / b;
}
```

Line-by-line walk-through. The `FindFirstError` function demonstrates the lesson's core technique — an early `return` inside the loop: instead of accumulating the result in a flag and checking it after the loop, we leave the method immediately as soon as we find the first line with the `ERROR` prefix. Returning `null` is an example of the rule "expected result — return, not throw": the absence of an error in the logs is a normal situation, not an exceptional one. The `SumValidSessions` function shows `continue`: when `cell == 0`, we skip the rest of the iteration body and move to the next element, so `sum += cell` never runs for zeros. Here it is important to remember that `continue` targets the nearest enclosing loop, so for a double traversal it correctly continues the inner `foreach`. The `ContainsSessionLongerThan` function is the key example of exiting nested loops: `return true` from the inner loop leaves the method entirely, escaping both levels at once. The lesson explicitly recommends this technique over `goto` or a flag, because it is cleaner and more idiomatic. `ClassifyMessage` uses a `switch` expression with pattern matching (`when`): this is the modern replacement for the classic `switch` + `break`, and `break` is neither needed nor allowed by the syntax. `DescribeDay` is intentionally written as a classic `switch` statement to show that `return` is also a valid way to end a section (an alternative to `break`); if a section had side effects without a `return`, a `break` would be required — otherwise CS0163. Finally, `DivideSafe` illustrates the boundary between `return` and `throw`: division by zero is an exceptional condition, so we throw `DivideByZeroException` instead of returning `-1` or `null`; the caller wraps the call in `try/catch`. All functions are small and use early exit, which matches the best practice from the lesson: less nesting — better readability.

#### Going deeper (bonus)

1. **LINQ instead of explicit loops.** Rewrite `FindFirstError` with `logs.FirstOrDefault(l => l.StartsWith("ERROR"))` and `ContainsSessionLongerThan` with `sessions.SelectMany(r => r).Any(c => c > threshold)`. Compare readability and discuss in which cases LINQ's "short-circuiting" replaces `break`/`return`.
2. **Performance of throw vs return.** Write a benchmark (using `Stopwatch`) that calls a "search" function in two variants: one returning `-1` on miss, and one throwing on miss. Run it 1,000,000 times for the "not found" case and measure the time difference. Draw a conclusion.
3. **Multi-level exit with `goto`.** Implement a version of `ContainsSessionLongerThan` with `goto foundLabel;` and compare its readability with the `return` version. Explain in a comment why the lesson recommends `return`.
4. **`IEnumerable` instead of arrays.** Change the signatures to `IEnumerable<string>` and `IEnumerable<IEnumerable<int>>`. Think about how the semantics of early exit changes (especially for deferred sequences).

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `LogAnalyzer` создан и собирается через `dotnet build` без ошибок и предупреждений. (RU)
- [ ] `Program.cs` использует top-level statements и коллекционные выражения. (RU)
- [ ] Все пять функций реализованы с применением `break`/`continue`/`return`/`throw` согласно требованиям. (RU)
- [ ] Вывод `dotnet run` совпадает с эталонными строками. (RU)
- [ ] Временное удаление `break` из `switch` подтверждает ошибку CS0163 и зафиксировано в `report.txt`. (RU)
- [ ] В коде есть двуязычные комментарии и рефлексия о `return` vs `throw`. (RU)
- [ ] The `LogAnalyzer` project is created and builds with `dotnet build` with no errors or warnings. (EN)
- [ ] `Program.cs` uses top-level statements and collection expressions. (EN)
- [ ] All five functions are implemented using `break`/`continue`/`return`/`throw` as required. (EN)
- [ ] The `dotnet run` output matches the reference lines. (EN)
- [ ] Temporarily removing `break` from the `switch` confirms CS0163 and is recorded in `report.txt`. (EN)
- [ ] The code has bilingual comments and a reflection on `return` vs `throw`. (EN)

#### Ресурсы / Resources
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/statements/jump-statements — Jump statements (EN) / Операторы перехода (RU)
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/break — break keyword (EN) / Ключевое слово break (RU)
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/operators/switch-expression — switch expression (EN) / Выражение switch (RU)
- Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/programming-guide/exceptions/ — Exceptions & exception handling (EN) / Исключения и их обработка (RU)
