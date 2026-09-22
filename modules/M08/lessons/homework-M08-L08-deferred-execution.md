---
[← К уроку M08-L08](lesson-M08-L08-deferred-execution.md) | [⬆ К модулю M08](../README.md) | [Следующее ДЗ →](homework-M08-L09-iqueryable-vs-ienumerable.md)
---

### Домашнее задание M08-L08: Отложенное vs немедленное выполнение, ToList/ToArray / Homework M08-L08: Deferred vs immediate execution, ToList/ToArray

**Урок / Lesson:** M08-L08
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться различать отложенные и немедленные операторы LINQ, диагностировать многократное перечисление и ловушки замыканий, осознанно выбирать между `ToList` и `ToArray` и материализовать результаты там, где это улучшает производительность и предсказуемость. (EN) Learn to distinguish deferred and immediate LINQ operators, diagnose multiple enumeration and closure traps, make a conscious choice between `ToList` and `ToArray`, and materialize results where it improves performance and predictability.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит фундаментальное различие между «рецептом» (отложенным запросом) и «тортом» (материализованным результатом). Это ДЗ закрепляет все три ловушки из урока — многократное перечисление, захват переменных в замыкании и побочные эффекты в предикатах — на практической задаче обработки журнала событий, где каждое перечисление стоит дорого. Вы также отработаете правило «материализуй, если перебираешь больше одного раза» и выбор между `ToList` и `ToArray`.
(EN) The lesson introduces the fundamental distinction between the "recipe" (a deferred query) and the "cake" (a materialized result). This homework reinforces all three traps from the lesson — multiple enumeration, captured variables in closures, and side effects in predicates — on a practical event-log task where every enumeration is expensive. You will also practice the "materialize if you enumerate more than once" rule and the choice between `ToList` and `ToArray`.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Представьте, что вы инженер мониторинга в команде, отвечающей за телеметрию платёжного шлюза. Система пишет поток событий в большой файл журнала `gateway.log`: каждая строка — запись о транзакции с уровнем (`INFO`, `WARN`, `ERROR`, `FATAL`), меткой времени, идентификатором транзакции и сообщением. Объём — сотни тысяч строк в день, поэтому чтение всего файла в память «на всякий случай» недопустимо: вы хотите обрабатывать данные потоково, вычисляя только то, что реально нужно.

Вам поручили написать утилиту `LogAnalyzer`, которая выполняет несколько независимых анализов по журналу: количество ошибок уровня `ERROR` и `FATAL`, топ-5 транзакций с самой длинной длительностью, перечень уникальных идентификаторов проблемных транзакций, а также сводку по уровням. Аналитики будут вызывать эти методы многократно и в разных комбинациях, поэтому критически важно, чтобы каждое чтение файла происходило ровно столько раз, сколько действительно требуется, — не больше.

Именно здесь вступают в силу темы урока M08-L08. Источник данных — итератор, читающий файл построчно: это «отложенный» `IEnumerable<LogEntry>` с побочным эффектом (открытие и чтение файла). Если вы по небрежности перечислите его дважды, файл будет прочитан дважды. Если вы заложите в запрос побочный эффект, результат станет недетерминированным. Если вы захватите переменную цикла в замыкании, получите «фантомные» значения. Ваша задача — написать корректный, быстрый и предсказуемый код, осознанно применяя `ToList`/`ToArray` для материализации там, где результат нужен больше одного раза, и оставляя отложенное выполнение там, где нужен потоковый обход.

#### Что нужно сделать (пошагово)

1. **Создайте проект.** В терминале выполните `dotnet new console -n LogAnalyzer -o LogAnalyzer --framework net8.0`, затем перейдите в папку: `cd LogAnalyzer`. Убедитесь, что в `LogAnalyzer.csproj` стоит `<LangVersion>12</LangVersion>` и `<Nullable>enable</Nullable>` — добавьте их при необходимости.

2. **Сгенерируйте тестовый журнал.** Создайте скрипт или просто в `Program.cs` сгенерируйте файл `gateway.log` с ~10 000 строк. Формат строки: `2024-05-12T10:23:45|TXN-0001|INFO|Processed in 42 ms`. Уровни распределите так: ~70 % `INFO`, ~15 % `WARN`, ~10 % `ERROR`, ~5 % `FATAL`. Длительность (в `ms`) варьируется от 1 до 5000, причём `ERROR`/`FATAL` имеют в среднем большую длительность. Это будет вашим источником данных.

3. **Опишите модель.** Создайте тип `record LogEntry(DateTime Timestamp, string TxnId, LogLevel Level, int DurationMs, string Message);` и `enum LogLevel { Info, Warn, Error, Fatal }`. Напишите метод `IEnumerable<LogEntry> ReadLog(string path)`, читающий файл построчно через `File.ReadLines` (НЕ `ReadAllLines` — он загружает весь файл) с `yield return`. В тело итератора добавьте инкремент статического счётчика `LinesRead` — он понадобится для проверки числа проходов.

4. **Реализуйте методы анализа** в классе `LogAnalyzer`:
   - `int CountErrors(IEnumerable<LogEntry> source)` — использует `Count(e => e.Level is LogLevel.Error or LogLevel.Fatal)` (немедленный скаляр).
   - `IReadOnlyList<LogEntry> TopByDuration(IEnumerable<LogEntry> source, int n)` — `OrderByDescending(e => e.DurationMs).Take(n).ToList()`. Подумайте, почему здесь `ToList` оправдан.
   - `IReadOnlyCollection<string> ProblemTxnIds(IEnumerable<LogEntry> source)` — `Where(...).Select(e => e.TxnId).Distinct().ToArray()`. Почему `ToArray`, а не `ToList`?
   - `Dictionary<LogLevel, int> LevelSummary(IEnumerable<LogEntry> source)` — через `GroupBy` + `ToDictionary`, либо вручную.

5. **Демонстрация в `Program.cs`.** Вызовите все четыре метода и выведите результаты. После каждого вызова печатайте `LogAnalyzer.LinesRead`, чтобы доказать, сколько раз был прочитан файл. Ожидаемый вывод: суммарное число чтений файла равно числу вызовов, каждый из которых — ровно один проход.

6. **Демонстрация ловушек (отдельный блок).** Напишите метод `DemoTraps()`, который: (а) дважды перебирает один `IEnumerable` от `ReadLog` через два `foreach` — покажите, что `LinesRead` вырос вдвое; (б) строит запрос с захваченной переменной цикла, изменяет её до перечисления и показывает «фантомное» значение; (в) материализует через `ToList` и доказывает, что повторные переборы списка не трогают источник. Каждый шаг снабдите `Console.WriteLine` с пояснением.

7. **Запустите и проверьте.** `dotnet build`, затем `dotnet run --project LogAnalyzer`. Убедитесь, что число чтений файла в «правильной» части соответствует ожиданиям, а в блоке ловушек — демонстрирует проблему.

8. **Измерьте.** Добавьте `System.Diagnostics.Stopwatch` вокруг ключевых вызовов и выведите время. Сравните «двойной `foreach` по отложенному запросу» против «один `ToList` + два перебора списка».

#### Требования к решению

- Целевая платформа — .NET 8, язык C# 12. Используйте top-level statements в `Program.cs`, pattern matching (`is`), коллекционные выражения для инициализации (`LogLevel[] levels = [LogLevel.Info, LogLevel.Warn];`), file-scoped namespaces, `record` для модели.
- Все публичные методы анализа должны принимать `IEnumerable<LogEntry>`, а не `List<LogEntry>` или массив — это часть контракта «работаем с любым перечисляемым источником».
- Методы, возвращающие коллекции для дальнейшего многократного использования (`TopByDuration`, `ProblemTxnIds`), должны материализовать результат через `ToList` или `ToArray` и возвращать интерфейс только для чтения (`IReadOnlyList<T>` / `IReadOnlyCollection<T>`), чтобы случайно не вернуть отложенный запрос.
- Чтение файла — строго через `File.ReadLines` (потоковое). Запрещено `File.ReadAllLines` для источника данных.
- Запрещены побочные эффекты внутри предикатов `Where` и проекций `Select` (логирование, мутация внешнего состояния, инкременты счётчиков внутри лямбд). Единственный допустимый «побочный эффект» — счётчик `LinesRead` внутри `ReadLog`, и только там.
- Для проверки наличия элементов используйте `Any()`, а не `Count() > 0`.
- Код должен компилироваться без предупреждений (включая CA1827 — `Use Any` вместо `Count`), быть форматирован через `dotnet format`, и проходить `dotnet build -warnaserror` при разумных настройках.
- В блоке `DemoTraps` обязательно присутствуют обе демонстрации: многократное перечисление (с подсчётом через `LinesRead`) и ловушка замыкания (с исправлением через локальную копию переменной).

#### Тонкости и подводные камни

- **Многократное перечисление итератора.** Если вы сохраните `var q = ReadLog(path).Where(...)` в переменную и потом дважды сделаете `foreach (var x in q)`, метод `ReadLog` отработает дважды — файл будет прочитан заново. Анализаторы ReSharper и Roslyn (CA1851) предупреждают об этом. Лечится материализацией через `ToList`/`ToArray` один раз.
- **`Count()` форсирует полный обход.** `Count(e => e.Level == LogLevel.Error)` немедленно перечисляет всю последовательность. Если вы затем ещё раз перечисляете тот же источник для другого анализа — это второй полный проход. Стратегия: либо материализуйте один раз и работайте со списком, либо объединяйте несколько анализов в один проход (но это отдельная задача оптимизации — в рамках ДЗ достаточно осознать стоимость).
- **Ловушка замыкания.** `for (int i = 0; i < 3; i++) actions.Add(() => i);` — все три лямбды захватывают одну и ту же переменную `i`, которая к моменту вызова равна 3. Исправление: `int local = i;` внутри тела цикла. В C# 5+ `foreach` уже создаёт копию на каждой итерации, но `for` — нет; будьте внимательны.
- **Побочные эффекты в `Where`/`Select`.** Если в предикате `Where` вы инкрементируете счётчик или пишете в лог, порядок и количество вызовов становятся непредсказуемыми: `OrderBy` буферизует всю последовательность, `Take` может отсечь часть вызовов. Держите предикаты чистыми.
- **`OrderBy` буферизует.** Вопреки интуиции, `OrderBy` не «ленивый»: онmaterializes весь источник в отсортированный снимок, и лишь потом `Take(n)` отрезает первые n. Это означает, что `OrderByDescending(...).Take(5)` по-прежнему читает весь журнал — это нормально, но должно быть осознанным решением.
- **`ToList` vs `ToArray`.** `ToList` даёт `List<T>` — мутируемый, удобен для дальнейших `Add`/`Remove`, но имеет резервный буфер (capacity), который может быть больше реального размера. `ToArray` даёт массив фиксированной длины — компактнее по памяти, быстрее индексация, но нельзя изменить размер. В .NET 8 `ToArray` для известного размера источника выделяет ровно нужный массив; для неизвестного — один промежуточный `List` и финальный `Copy`. Выбирайте `ToArray`, если коллекция далее не мутируется и важна компактность.
- **`File.ReadLines` vs `File.ReadAllLines`.** `ReadLines` возвращает `IEnumerable<string>` — ленивый итератор, читающий построчно; `ReadAllLines` сразу грузит весь файл в `string[]`. Для источника данных ДЗ обязателен `ReadLines`.
- **Возврат интерфейсов только для чтения.** Возвращая `IReadOnlyList<T>`, вы защищаете вызывающего от случайной мутации и сигнализируете, что результат уже материализован. Не возвращайте «голый» `IEnumerable<T>` из метода, который обещал готовый результат — вызывающий не поймёт, материализовано ли это или ещё «рецепт».

#### Критерии приёмки

- [ ] Проект `LogAnalyzer` собирается через `dotnet build` без ошибок и без предупреждений на `-warnaserror`.
- [ ] Используется .NET 8 и C# 12 (top-level statements, `record`, pattern matching, коллекционные выражения).
- [ ] Источник данных — метод `ReadLog` на `File.ReadLines` с `yield return` и счётчиком `LinesRead`.
- [ ] Реализованы четыре метода анализа: `CountErrors`, `TopByDuration`, `ProblemTxnIds`, `LevelSummary`.
- [ ] `TopByDuration` и `ProblemTxnIds` возвращают материализованные `IReadOnlyList<T>`/`IReadOnlyCollection<T>` (через `ToList`/`ToArray`), а не отложенный `IEnumerable`.
- [ ] `CountErrors` использует `Count` с предикатом; `LevelSummary` использует `GroupBy` + `ToDictionary`.
- [ ] В «правильной» части программы суммарное число чтений файла (`LinesRead`) равно числу вызовов анализа — каждый вызов = один проход.
- [ ] Блок `DemoTraps` демонстрирует многократное перечисление: два `foreach` по одному `IEnumerable` дают `LinesRead` вдвое больше одного.
- [ ] Блок `DemoTraps` демонстрирует ловушку замыкания: `for` с захватом `i` выводит «3 3 3», а исправленная версия — «0 1 2».
- [ ] Блок `DemoTraps` показывает, что после `ToList` повторные `foreach` по списку не увеличивают `LinesRead`.
- [ ] Нигде в коде не используется `Count() > 0` — везде `Any()`, где это семантически уместно.
- [ ] В предикатах `Where` и проекциях `Select` нет побочных эффектов (нет `Console.WriteLine`, мутаций, инкрементов).
- [ ] Вывод программы содержит измерения времени через `Stopwatch` для сравнения стратегий.
- [ ] Код отформатирован `dotnet format`, нет закомментированного мусора.
- [ ] README или комментарий в начале файла кратко объясняет, какие концепции урока демонстрирует каждый блок.

#### Подсказки (без прямого ответа)

- Счётчик `LinesRead` сделайте `static` полем класса `LogAnalyzer` — он переживёт между вызовами. Сбрасывайте его в `0` перед каждой демонстрационной серией, чтобы цифры были чистыми.
- Для «двойного перечисления» сохраните запрос в локальную переменную `var deferred = ReadLog(path).Where(e => e.Level == LogLevel.Error);` и сделайте два `foreach`. Сравните `LinesRead` до и после.
- Чтобы поймать ловушку замыкания, постройте `List<Func<int>>`, в цикле `for` добавляйте `() => i`, затем измените `i` или просто завершите цикл — и перечислите. Подумайте, почему C# 5+ «чинит» `foreach`, но не `for`.
- Для исправления замыкания объявите внутри тела цикла новую переменную и захватывайте её.
- `Distinct()` в .NET 8 использует `HashSet<T>` под капотом и сохраняет порядок первого вхождения — это полезно для `ProblemTxnIds`.
- Не забудьте `using System.Linq;` и `using static System.Console;` для краткости.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — LogAnalyzer: отложенное vs немедленное выполнение
// Deferred vs immediate execution in practice

using System.Diagnostics;
using System.Linq;

namespace LogAnalyzer;

public enum LogLevel { Info, Warn, Error, Fatal }

public record LogEntry(DateTime Timestamp, string TxnId, LogLevel Level, int DurationMs, string Message);

public static class LogAnalyzer
{
    // Счётчик проходов по источнику — для диагностики многократного перечисления
    // Pass counter over the source — to diagnose multiple enumeration
    public static int LinesRead { get; private set; }

    public static void ResetCounter() => LinesRead = 0;

    // Источник-итератор: читает файл ПОСТРОЧНО через File.ReadLines (потоково, лениво)
    // Iterator source: reads the file LINE BY LINE via File.ReadLines (streaming, lazy)
    public static IEnumerable<LogEntry> ReadLog(string path)
    {
        foreach (var line in File.ReadLines(path)) // ленивый обход / lazy traversal
        {
            LinesRead++; // единственный допустимый побочный эффект — здесь / only allowed side effect
            var parts = line.Split('|');
            if (parts.Length != 4) continue;
            var ts = DateTime.Parse(parts[0]);
            var txn = parts[1];
            var level = Enum.Parse<LogLevel>(parts[2], ignoreCase: true);
            var msg = parts[3];
            // длительность извлекаем из сообщения / extract duration from the message
            var dur = ExtractDurationMs(msg);
            yield return new LogEntry(ts, txn, level, dur, msg); // отложенный возврат / deferred yield
        }
    }

    static int ExtractDurationMs(string message)
    {
        var idx = message.IndexOf("in ", StringComparison.Ordinal);
        if (idx < 0) return 0;
        var rest = message[(idx + 3)..];
        var end = rest.IndexOf(" ms", StringComparison.Ordinal);
        return end > 0 && int.TryParse(rest[..end], out var d) ? d : 0;
    }

    // 1) НЕМЕДЛЕННЫЙ скаляр: Count форсирует полный обход ровно один раз
    // 1) IMMEDIATE scalar: Count forces exactly one full pass
    public static int CountErrors(IEnumerable<LogEntry> source) =>
        source.Count(e => e.Level is LogLevel.Error or LogLevel.Fatal); // pattern matching

    // 2) ToList: материализуем топ-N, чтобы вызывающий мог перебирать многократно
    // 2) ToList: materialize top-N so the caller can iterate many times
    public static IReadOnlyList<LogEntry> TopByDuration(IEnumerable<LogEntry> source, int n) =>
        source.OrderByDescending(e => e.DurationMs).Take(n).ToList();

    // 3) ToArray: размер фиксирован, коллекция не мутируется — компактнее по памяти
    // 3) ToArray: fixed size, no mutation — more memory-compact
    public static IReadOnlyCollection<string> ProblemTxnIds(IEnumerable<LogEntry> source) =>
        source.Where(e => e.Level is LogLevel.Error or LogLevel.Fatal)
              .Select(e => e.TxnId)
              .Distinct()
              .ToArray();

    // 4) GroupBy + ToDictionary: немедленное выполнение, готовый словарь
    // 4) GroupBy + ToDictionary: immediate execution, ready dictionary
    public static Dictionary<LogLevel, int> LevelSummary(IEnumerable<LogEntry> source) =>
        source.GroupBy(e => e.Level)
              .ToDictionary(g => g.Key, g => g.Count());

    // Демонстрация ловушек урока / Demo of the lesson's traps
    public static void DemoTraps(string path)
    {
        ResetCounter();

        // (а) Многократное перечисление: два foreach по одному IEnumerable — двойное чтение
        // (a) Multiple enumeration: two foreach over one IEnumerable — double read
        var deferred = ReadLog(path).Where(e => e.Level == LogLevel.Error);
        var before = LinesRead;
        foreach (var _ in deferred) { } // первый проход / first pass
        foreach (var _ in deferred) { } // второй проход — источник читается ЗАНОВО / second pass — source re-read
        WriteLine($"[trap a] До: {before}, после двух foreach: {LinesRead} (ожидаемо ~2x)");

        // (б) Ловушка замыкания в цикле for — захват переменной, а не значения
        // (b) Closure trap in a for loop — captures variable, not value
        var actions = new List<Func<int>>();
        for (int i = 0; i < 3; i++)
            actions.Add(() => i); // захватывается i, а не 0/1/2 / captures i, not 0/1/2
        Write("[trap b buggy]   "); foreach (var a in actions) Write($"{a()} "); // 3 3 3
        WriteLine();

        // Исправление: локальная копия на каждой итерации / Fix: local copy per iteration
        var fixedActions = new List<Func<int>>();
        for (int i = 0; i < 3; i++)
        {
            int local = i; // своя копия для каждой лямбды / own copy for each lambda
            fixedActions.Add(() => local);
        }
        Write("[trap b fixed]   "); foreach (var a in fixedActions) Write($"{a()} "); // 0 1 2
        WriteLine();

        // (в) ToList «замораживает» результат — повторные переборы не трогают источник
        // (c) ToList freezes the result — re-iteration does not touch the source
        ResetCounter();
        var materialized = ReadLog(path).Where(e => e.Level == LogLevel.Error).ToList();
        var afterMaterialize = LinesRead;
        foreach (var _ in materialized) { }
        foreach (var _ in materialized) { }
        WriteLine($"[trap c] После ToList: {afterMaterialize}, после двух переборов списка: {LinesRead} (без изменений)");
    }
}

// Program.cs (top-level statements) — демонстрация правильного использования
file static class Program
{
    public static void Main(string[] args)
    {
        var path = "gateway.log";
        // (генерация файла опущена для краткости — см. метод Generate в полном проекте)

        LogAnalyzer.ResetCounter();
        var sw = Stopwatch.StartNew();

        var source = LogAnalyzer.ReadLog(path);
        var errors = LogAnalyzer.CountErrors(source);               // 1 проход
        var top = LogAnalyzer.TopByDuration(source, 5);             // 1 проход
        var ids = LogAnalyzer.ProblemTxnIds(source);                // 1 проход
        var summary = LogAnalyzer.LevelSummary(source);             // 1 проход

        sw.Stop();
        WriteLine($"Ошибок: {errors}");
        WriteLine($"Топ-5 по длительности: {string.Join(", ", top.Select(e => $"{e.TxnId}={e.DurationMs}ms"))}");
        WriteLine($"Проблемных транзакций: {ids.Count}");
        WriteLine($"Сводка по уровням: {string.Join(", ", summary.Select(kv => $"{kv.Key}={kv.Value}"))}");
        WriteLine($"Чтений файла: {LogAnalyzer.LinesRead} (4 анализа = 4 прохода)");
        WriteLine($"Время: {sw.ElapsedMilliseconds} мс");

        LogAnalyzer.DemoTraps(path);
    }
}
```

**Разбор по строкам.** Метод `ReadLog` — это «рецепт»: тело с `yield return` не выполняется в момент вызова, а превращается в итератор. Поэтому `LinesRead` остаётся нулём, пока кто-то не начнёт перебор. Каждый оператор поверх `ReadLog(...)` (`Where`, `Select`, `OrderByDescending`) тоже отложен — он лишь добавляет шаг в конвейер. `Count(...)` в `CountErrors` — немедленный: он форсирует полный обход и возвращает скаляр, поэтому ровно одно чтение файла. `TopByDuration` применяет `OrderByDescending` (который, как подчёркнуто в уроке, буферизует всю последовательность), затем `Take(5)` и завершается `ToList()` — материализация делает результат безопасным для многократного перебора вызывающим кодом; без `ToList` мы вернули бы «рецепт», и каждое использование топ-5 перечитывало бы файл. `ProblemTxnIds` заканчивается `ToArray()`: коллекция фиксированного размера и далее не мутируется, поэтому массив компактнее списка — это лучшая практика из урока. `LevelSummary` через `GroupBy` + `ToDictionary` тоже немедленный и возвращает готовый словарь. В `DemoTraps` блок (а) доказывает, что два `foreach` по одному `IEnumerable` удваивают `LinesRead` — классический баг из урока. Блок (б) воспроизводит ловушку замыкания в `for` (вывод «3 3 3») и показывает исправление через локальную копию. Блок (в) доказывает, что после `ToList` повторные переборы списка не увеличивают счётчик: «торт» уже испечён, и «повар» больше не работает. Применены концепции: отложенные vs немедленные операторы, материализация, многократное перечисление, чистые предикаты, выбор `ToList`/`ToArray`, `Any`-семантика (через `Count` с предикатом там, где действительно нужен счёт).

#### Задания на углубление (бонус)

1. **Один проход вместо четырёх.** Перепишите анализ так, чтобы все четыре результата (`errors`, `top`, `ids`, `summary`) вычислялись за один проход по источнику — например, через кастомный агрегатор или `Aggregate` с аккумулятором-кортежем. Сравните `LinesRead` и время с базовым решением. Обсудите, когда единый проход оправдан, а когда читаемость важнее.
2. **`IQueryable`-вариант.** Создайте in-memory `DbContext` (EF Core InMemory) с таблицей `LogEntries` и перепишите методы, принимая `IQueryable<LogEntry>`. Покажите, что без `ToList` на конце повторное использование результата отправляет несколько SQL-запросов; с `ToList` — один. Это мост к уроку M08-L09.
3. **Бесконечная последовательность.** Реализуйте `IEnumerable<LogEntry> InfiniteStream()` (генерация случайных событий) и продемонстрируйте, что `Take(100).ToList()` работает, а `Count()` без `Take` — зависает. Обсудите, почему отложенное выполнение здесь не преимущество, а необходимость.
4. **Анализаторы.** Включите `CA1851` (possible multiple enumerations) и `CA1827` (use `Any` instead of `Count`), добейтесь нулевых предупреждений. Добавьте комментарий в README, какой анализатор какую ловушку ловит.

---

## Statement in English / Постановка на английском

#### Context & motivation

Imagine you are a monitoring engineer on the team responsible for the telemetry of a payment gateway. The system streams events into a large `gateway.log` file: each line is a transaction record with a level (`INFO`, `WARN`, `ERROR`, `FATAL`), a timestamp, a transaction id, and a message. The volume is hundreds of thousands of lines per day, so loading the entire file into memory "just in case" is unacceptable: you want to process the data in a streaming fashion, computing only what is actually needed.

You have been asked to write a `LogAnalyzer` utility that performs several independent analyses over the log: the count of `ERROR` and `FATAL` events, the top-5 transactions by duration, the list of unique ids of problematic transactions, and a summary grouped by level. Analysts will call these methods repeatedly and in different combinations, so it is critical that each read of the file happens exactly as many times as truly required — no more.

This is exactly where the topics of lesson M08-L08 come into play. The data source is an iterator that reads the file line by line: a "deferred" `IEnumerable<LogEntry>` with a side effect (opening and reading the file). If you carelessly enumerate it twice, the file is read twice. If you bake a side effect into the query, the result becomes nondeterministic. If you capture a loop variable in a closure, you get "phantom" values. Your task is to write correct, fast, and predictable code, consciously applying `ToList`/`ToArray` to materialize where the result is needed more than once, and keeping deferred execution where streaming traversal is required.

#### What to do step by step

1. **Create the project.** In a terminal run `dotnet new console -n LogAnalyzer -o LogAnalyzer --framework net8.0`, then `cd LogAnalyzer`. Make sure `LogAnalyzer.csproj` has `<LangVersion>12</LangVersion>` and `<Nullable>enable</Nullable>` — add them if missing.

2. **Generate the test log.** Create a script or simply generate a `gateway.log` file with ~10 000 lines from `Program.cs`. Line format: `2024-05-12T10:23:45|TXN-0001|INFO|Processed in 42 ms`. Distribute levels roughly as: ~70 % `INFO`, ~15 % `WARN`, ~10 % `ERROR`, ~5 % `FATAL`. Durations (in `ms`) range from 1 to 5000, with `ERROR`/`FATAL` having a higher average. This is your data source.

3. **Model the domain.** Define `record LogEntry(DateTime Timestamp, string TxnId, LogLevel Level, int DurationMs, string Message);` and `enum LogLevel { Info, Warn, Error, Fatal }`. Write a method `IEnumerable<LogEntry> ReadLog(string path)` that reads the file line by line through `File.ReadLines` (NOT `ReadAllLines`, which loads the whole file) using `yield return`. Inside the iterator body, increment a static counter `LinesRead` — you will need it to verify the number of passes.

4. **Implement the analysis methods** in a `LogAnalyzer` class:
   - `int CountErrors(IEnumerable<LogEntry> source)` — uses `Count(e => e.Level is LogLevel.Error or LogLevel.Fatal)` (an immediate scalar).
   - `IReadOnlyList<LogEntry> TopByDuration(IEnumerable<LogEntry> source, int n)` — `OrderByDescending(e => e.DurationMs).Take(n).ToList()`. Think about why `ToList` is justified here.
   - `IReadOnlyCollection<string> ProblemTxnIds(IEnumerable<LogEntry> source)` — `Where(...).Select(e => e.TxnId).Distinct().ToArray()`. Why `ToArray` rather than `ToList`?
   - `Dictionary<LogLevel, int> LevelSummary(IEnumerable<LogEntry> source)` — via `GroupBy` + `ToDictionary`, or manually.

5. **Driver in `Program.cs`.** Call all four methods and print the results. After each call, print `LogAnalyzer.LinesRead` to prove how many times the file was read. Expected output: the total number of file reads equals the number of calls, each of them being exactly one pass.

6. **Trap demonstration (separate block).** Write a `DemoTraps()` method that: (a) iterates the same `IEnumerable` from `ReadLog` twice with two `foreach` loops — show that `LinesRead` doubled; (b) builds a query with a captured loop variable, mutates it before enumeration, and shows the "phantom" value; (c) materializes via `ToList` and proves that re-iterating the list does not touch the source. Annotate every step with `Console.WriteLine`.

7. **Run and verify.** `dotnet build`, then `dotnet run --project LogAnalyzer`. Confirm that the number of file reads in the "correct" part matches expectations, and that the trap block demonstrates the problem.

8. **Measure.** Add `System.Diagnostics.Stopwatch` around the key calls and print the timings. Compare "double `foreach` over a deferred query" against "one `ToList` plus two iterations of the list".

#### Requirements

- Target .NET 8, language C# 12. Use top-level statements in `Program.cs`, pattern matching (`is`), collection expressions for initialization (`LogLevel[] levels = [LogLevel.Info, LogLevel.Warn];`), file-scoped namespaces, `record` for the model.
- All public analysis methods must accept `IEnumerable<LogEntry>`, not `List<LogEntry>` or an array — this is part of the "works with any enumerable source" contract.
- Methods that return collections intended for further reuse (`TopByDuration`, `ProblemTxnIds`) must materialize the result through `ToList` or `ToArray` and return a read-only interface (`IReadOnlyList<T>` / `IReadOnlyCollection<T>`), so that a deferred query is never accidentally returned.
- File reading must use `File.ReadLines` (streaming). `File.ReadAllLines` is forbidden for the data source.
- No side effects inside `Where` predicates or `Select` projections (no logging, no outer-state mutation, no counter increments inside lambdas). The only allowed "side effect" is the `LinesRead` counter inside `ReadLog`, and only there.
- Use `Any()` rather than `Count() > 0` to check for the presence of elements.
- The code must compile without warnings (including CA1827 — use `Any` instead of `Count`), be formatted with `dotnet format`, and pass `dotnet build -warnaserror` under reasonable settings.
- The `DemoTraps` block must include both demonstrations: multiple enumeration (counted via `LinesRead`) and the closure trap (with the fix via a local copy of the variable).

#### Pitfalls

- **Multiple enumeration of an iterator.** If you store `var q = ReadLog(path).Where(...)` in a variable and then do `foreach (var x in q)` twice, `ReadLog` runs twice — the file is re-read. ReSharper and Roslyn analyzers (CA1851) warn about this. Fix it by materializing once with `ToList`/`ToArray`.
- **`Count()` forces a full scan.** `Count(e => e.Level == LogLevel.Error)` immediately enumerates the whole sequence. If you then enumerate the same source again for another analysis, that is a second full pass. Strategy: either materialize once and work with the list, or combine several analyses into a single pass (a separate optimization task — for this homework it is enough to be aware of the cost).
- **The closure trap.** `for (int i = 0; i < 3; i++) actions.Add(() => i);` — all three lambdas capture the same variable `i`, which equals 3 by the time they are invoked. Fix: `int local = i;` inside the loop body. In C# 5+ `foreach` already creates a copy per iteration, but `for` does not; be careful.
- **Side effects inside `Where`/`Select`.** If you increment a counter or log inside a `Where` predicate, the order and number of invocations become unpredictable: `OrderBy` buffers the whole sequence, `Take` may cut some invocations off. Keep predicates pure.
- **`OrderBy` buffers.** Contrary to intuition, `OrderBy` is not lazy: it materializes the entire source into a sorted snapshot, and only then does `Take(n)` slice the first n. This means `OrderByDescending(...).Take(5)` still reads the entire log — that is fine, but it must be a conscious decision.
- **`ToList` vs `ToArray`.** `ToList` yields a `List<T>` — mutable, convenient for further `Add`/`Remove`, but it has a capacity buffer that may exceed the real size. `ToArray` yields a fixed-length array — more memory-compact, faster indexed access, but immutable in size. In .NET 8, `ToArray` on a source of known size allocates exactly the right array; for an unknown size it uses one intermediate `List` and a final `Copy`. Choose `ToArray` when the collection will not be mutated and compactness matters.
- **`File.ReadLines` vs `File.ReadAllLines`.** `ReadLines` returns an `IEnumerable<string>` — a lazy iterator reading line by line; `ReadAllLines` eagerly loads the whole file into a `string[]`. `ReadLines` is mandatory for the data source in this homework.
- **Returning read-only interfaces.** By returning `IReadOnlyList<T>` you protect the caller from accidental mutation and signal that the result is already materialized. Do not return a "bare" `IEnumerable<T>` from a method that promised a ready result — the caller cannot tell whether it is materialized or still a "recipe".

#### Acceptance criteria

- [ ] The `LogAnalyzer` project builds via `dotnet build` with no errors and no warnings under `-warnaserror`.
- [ ] .NET 8 and C# 12 are used (top-level statements, `record`, pattern matching, collection expressions).
- [ ] The data source is the `ReadLog` method built on `File.ReadLines` with `yield return` and a `LinesRead` counter.
- [ ] Four analysis methods are implemented: `CountErrors`, `TopByDuration`, `ProblemTxnIds`, `LevelSummary`.
- [ ] `TopByDuration` and `ProblemTxnIds` return materialized `IReadOnlyList<T>`/`IReadOnlyCollection<T>` (via `ToList`/`ToArray`), not a deferred `IEnumerable`.
- [ ] `CountErrors` uses `Count` with a predicate; `LevelSummary` uses `GroupBy` + `ToDictionary`.
- [ ] In the "correct" part of the program, the total number of file reads (`LinesRead`) equals the number of analysis calls — each call is one pass.
- [ ] The `DemoTraps` block demonstrates multiple enumeration: two `foreach` over one `IEnumerable` yield roughly double the `LinesRead` of a single pass.
- [ ] The `DemoTraps` block demonstrates the closure trap: a `for` loop capturing `i` prints "3 3 3", and the fixed version prints "0 1 2".
- [ ] The `DemoTraps` block shows that after `ToList` repeated `foreach` over the list does not increase `LinesRead`.
- [ ] Nowhere in the code is `Count() > 0` used — `Any()` is used wherever semantically appropriate.
- [ ] No side effects are present inside `Where` predicates or `Select` projections (no `Console.WriteLine`, mutations, or increments).
- [ ] The program output contains `Stopwatch` timings comparing the strategies.
- [ ] The code is formatted with `dotnet format`, with no commented-out cruft.
- [ ] A README or a header comment briefly explains which lesson concepts each block demonstrates.

#### Hints (no direct answer)

- Make `LinesRead` a `static` field on the `LogAnalyzer` class — it will survive across calls. Reset it to `0` before each demonstration series to keep the numbers clean.
- For the "double enumeration" demo, store the query in a local `var deferred = ReadLog(path).Where(e => e.Level == LogLevel.Error);` and run two `foreach` loops. Compare `LinesRead` before and after.
- To catch the closure trap, build a `List<Func<int>>`, add `() => i` inside a `for` loop, then mutate `i` or just let the loop finish — and enumerate. Think about why C# 5+ "fixes" `foreach` but not `for`.
- To fix the closure, declare a new variable inside the loop body and capture that one.
- `Distinct()` in .NET 8 uses a `HashSet<T>` internally and preserves first-occurrence order — useful for `ProblemTxnIds`.
- Do not forget `using System.Linq;` and `using static System.Console;` for brevity.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — LogAnalyzer: deferred vs immediate execution in practice

using System.Diagnostics;
using System.Linq;

namespace LogAnalyzer;

public enum LogLevel { Info, Warn, Error, Fatal }

public record LogEntry(DateTime Timestamp, string TxnId, LogLevel Level, int DurationMs, string Message);

public static class LogAnalyzer
{
    // Pass counter over the source — to diagnose multiple enumeration
    public static int LinesRead { get; private set; }

    public static void ResetCounter() => LinesRead = 0;

    // Iterator source: reads the file LINE BY LINE via File.ReadLines (streaming, lazy)
    public static IEnumerable<LogEntry> ReadLog(string path)
    {
        foreach (var line in File.ReadLines(path)) // lazy traversal
        {
            LinesRead++; // only allowed side effect — here
            var parts = line.Split('|');
            if (parts.Length != 4) continue;
            var ts = DateTime.Parse(parts[0]);
            var txn = parts[1];
            var level = Enum.Parse<LogLevel>(parts[2], ignoreCase: true);
            var msg = parts[3];
            var dur = ExtractDurationMs(msg);
            yield return new LogEntry(ts, txn, level, dur, msg); // deferred yield
        }
    }

    static int ExtractDurationMs(string message)
    {
        var idx = message.IndexOf("in ", StringComparison.Ordinal);
        if (idx < 0) return 0;
        var rest = message[(idx + 3)..];
        var end = rest.IndexOf(" ms", StringComparison.Ordinal);
        return end > 0 && int.TryParse(rest[..end], out var d) ? d : 0;
    }

    // 1) IMMEDIATE scalar: Count forces exactly one full pass
    public static int CountErrors(IEnumerable<LogEntry> source) =>
        source.Count(e => e.Level is LogLevel.Error or LogLevel.Fatal); // pattern matching

    // 2) ToList: materialize top-N so the caller can iterate many times
    public static IReadOnlyList<LogEntry> TopByDuration(IEnumerable<LogEntry> source, int n) =>
        source.OrderByDescending(e => e.DurationMs).Take(n).ToList();

    // 3) ToArray: fixed size, no mutation — more memory-compact
    public static IReadOnlyCollection<string> ProblemTxnIds(IEnumerable<LogEntry> source) =>
        source.Where(e => e.Level is LogLevel.Error or LogLevel.Fatal)
              .Select(e => e.TxnId)
              .Distinct()
              .ToArray();

    // 4) GroupBy + ToDictionary: immediate execution, ready dictionary
    public static Dictionary<LogLevel, int> LevelSummary(IEnumerable<LogEntry> source) =>
        source.GroupBy(e => e.Level)
              .ToDictionary(g => g.Key, g => g.Count());

    // Demo of the lesson's traps
    public static void DemoTraps(string path)
    {
        ResetCounter();

        // (a) Multiple enumeration: two foreach over one IEnumerable — double read
        var deferred = ReadLog(path).Where(e => e.Level == LogLevel.Error);
        var before = LinesRead;
        foreach (var _ in deferred) { } // first pass
        foreach (var _ in deferred) { } // second pass — source re-read
        WriteLine($"[trap a] before: {before}, after two foreach: {LinesRead} (expected ~2x)");

        // (b) Closure trap in a for loop — captures variable, not value
        var actions = new List<Func<int>>();
        for (int i = 0; i < 3; i++)
            actions.Add(() => i); // captures i, not 0/1/2
        Write("[trap b buggy]   "); foreach (var a in actions) Write($"{a()} "); // 3 3 3
        WriteLine();

        // Fix: local copy per iteration
        var fixedActions = new List<Func<int>>();
        for (int i = 0; i < 3; i++)
        {
            int local = i; // own copy for each lambda
            fixedActions.Add(() => local);
        }
        Write("[trap b fixed]   "); foreach (var a in fixedActions) Write($"{a()} "); // 0 1 2
        WriteLine();

        // (c) ToList freezes the result — re-iteration does not touch the source
        ResetCounter();
        var materialized = ReadLog(path).Where(e => e.Level == LogLevel.Error).ToList();
        var afterMaterialize = LinesRead;
        foreach (var _ in materialized) { }
        foreach (var _ in materialized) { }
        WriteLine($"[trap c] after ToList: {afterMaterialize}, after two iterations of the list: {LinesRead} (unchanged)");
    }
}

// Program.cs (top-level statements) — demonstration of correct usage
file static class Program
{
    public static void Main(string[] args)
    {
        var path = "gateway.log";
        // (file generation omitted for brevity — see the Generate method in the full project)

        LogAnalyzer.ResetCounter();
        var sw = Stopwatch.StartNew();

        var source = LogAnalyzer.ReadLog(path);
        var errors = LogAnalyzer.CountErrors(source);               // 1 pass
        var top = LogAnalyzer.TopByDuration(source, 5);             // 1 pass
        var ids = LogAnalyzer.ProblemTxnIds(source);                // 1 pass
        var summary = LogAnalyzer.LevelSummary(source);             // 1 pass

        sw.Stop();
        WriteLine($"Errors: {errors}");
        WriteLine($"Top-5 by duration: {string.Join(", ", top.Select(e => $"{e.TxnId}={e.DurationMs}ms"))}");
        WriteLine($"Problem transactions: {ids.Count}");
        WriteLine($"Level summary: {string.Join(", ", summary.Select(kv => $"{kv.Key}={kv.Value}"))}");
        WriteLine($"File reads: {LogAnalyzer.LinesRead} (4 analyses = 4 passes)");
        WriteLine($"Time: {sw.ElapsedMilliseconds} ms");

        LogAnalyzer.DemoTraps(path);
    }
}
```

**Line-by-line walk-through.** The `ReadLog` method is a "recipe": a body with `yield return` does not execute at the call site — it is compiled into an iterator. That is why `LinesRead` stays zero until somebody starts enumerating. Every operator layered on top of `ReadLog(...)` (`Where`, `Select`, `OrderByDescending`) is also deferred — it merely adds a step to the pipeline. `Count(...)` in `CountErrors` is immediate: it forces a full enumeration and returns a scalar, hence exactly one file read. `TopByDuration` applies `OrderByDescending` (which, as the lesson stresses, buffers the whole sequence), then `Take(5)`, and ends with `ToList()` — materialization makes the result safe for the caller to iterate many times; without `ToList` we would return a "recipe", and every use of the top-5 would re-read the file. `ProblemTxnIds` ends with `ToArray()`: the collection is of fixed size and will not be mutated, so an array is more compact than a list — a best practice from the lesson. `LevelSummary` via `GroupBy` + `ToDictionary` is also immediate and returns a ready dictionary. In `DemoTraps`, block (a) proves that two `foreach` loops over the same `IEnumerable` double `LinesRead` — the classic bug from the lesson. Block (b) reproduces the closure trap in a `for` loop (printing "3 3 3") and shows the fix via a local copy. Block (c) proves that after `ToList` re-iteration of the list does not increase the counter: the "cake" is already baked, and the "chef" no longer works. Concepts applied: deferred vs immediate operators, materialization, multiple enumeration, pure predicates, the `ToList`/`ToArray` choice, and the `Any` semantics (via `Count` with a predicate where a count is genuinely needed).

#### Going deeper (bonus)

1. **One pass instead of four.** Rewrite the analysis so that all four results (`errors`, `top`, `ids`, `summary`) are computed in a single pass over the source — for example, via a custom aggregator or `Aggregate` with a tuple accumulator. Compare `LinesRead` and timings against the baseline. Discuss when a single pass is justified and when readability matters more.
2. **An `IQueryable` variant.** Create an in-memory `DbContext` (EF Core InMemory) with a `LogEntries` table and rewrite the methods to accept `IQueryable<LogEntry>`. Show that without a final `ToList`, reusing the result sends several SQL queries; with `ToList`, only one. This is the bridge to lesson M08-L09.
3. **An infinite sequence.** Implement `IEnumerable<LogEntry> InfiniteStream()` (random event generation) and demonstrate that `Take(100).ToList()` works, while `Count()` without `Take` hangs. Discuss why deferred execution is here not an advantage but a necessity.
4. **Analyzers.** Turn on `CA1851` (possible multiple enumerations) and `CA1827` (use `Any` instead of `Count`), and drive the warnings to zero. Add a README comment explaining which analyzer catches which trap.

---

#### Чек-лист сдачи / Submission checklist

- [ ] Проект `LogAnalyzer` собирается без ошибок и предупреждений (`-warnaserror`).
- [ ] Используются C# 12 / .NET 8 (top-level statements, `record`, pattern matching, коллекционные выражения).
- [ ] Источник — `ReadLog` на `File.ReadLines` с `yield return` и счётчиком `LinesRead`.
- [ ] Реализованы `CountErrors`, `TopByDuration`, `ProblemTxnIds`, `LevelSummary`.
- [ ] `TopByDuration`/`ProblemTxnIds` возвращают материализованные `IReadOnlyList`/`IReadOnlyCollection`.
- [ ] В «правильной» части число чтений файла = числу вызовов (каждый = один проход).
- [ ] `DemoTraps` показывает многократное перечисление, ловушку замыкания и фикс `ToList`.
- [ ] Нет `Count() > 0` (используется `Any`), нет побочных эффектов в `Where`/`Select`.
- [ ] Есть измерения `Stopwatch`; код отформатирован `dotnet format`.
- [ ] README/комментарий поясняет, какие концепции урока демонстрирует каждый блок.

- [ ] The `LogAnalyzer` project builds with no errors or warnings (`-warnaserror`).
- [ ] C# 12 / .NET 8 are used (top-level statements, `record`, pattern matching, collection expressions).
- [ ] The source is `ReadLog` on `File.ReadLines` with `yield return` and a `LinesRead` counter.
- [ ] `CountErrors`, `TopByDuration`, `ProblemTxnIds`, `LevelSummary` are implemented.
- [ ] `TopByDuration`/`ProblemTxnIds` return materialized `IReadOnlyList`/`IReadOnlyCollection`.
- [ ] In the "correct" part, the number of file reads equals the number of calls (each is one pass).
- [ ] `DemoTraps` demonstrates multiple enumeration, the closure trap, and the `ToList` fix.
- [ ] No `Count() > 0` (`Any` is used), no side effects in `Where`/`Select`.
- [ ] `Stopwatch` measurements are present; the code is formatted with `dotnet format`.
- [ ] A README/comment explains which lesson concepts each block demonstrates.

#### Ресурсы / Resources
- [Microsoft Learn — Deferred Execution Example — https://learn.microsoft.com/dotnet/csharp/programming-guide/concepts/linq/deferred-execution-example](https://learn.microsoft.com/dotnet/csharp/programming-guide/concepts/linq/deferred-execution-example)
- [Microsoft Learn — Conversion Operators (ToList, ToArray) — https://learn.microsoft.com/dotnet/csharp/programming-guide/concepts/linq/conversion-operators](https://learn.microsoft.com/dotnet/csharp/programming-guide/concepts/linq/conversion-operators)
- [Microsoft Learn — File.ReadLines vs File.ReadAllLines — https://learn.microsoft.com/dotnet/api/system.io.file.readlines](https://learn.microsoft.com/dotnet/api/system.io.file.readlines)
- [Roslyn analyzer CA1851 — Possible multiple enumerations — https://learn.microsoft.com/dotnet/fundamentals/code-analysis/quality-rules/ca1851](https://learn.microsoft.com/dotnet/fundamentals/code-analysis/quality-rules/ca1851)
- [Roslyn analyzer CA1827 — Do not use Count/LongCount when Any can be used — https://learn.microsoft.com/dotnet/fundamentals/code-analysis/quality-rules/ca1827](https://learn.microsoft.com/dotnet/fundamentals/code-analysis/quality-rules/ca1827)
