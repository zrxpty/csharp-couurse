---
[← Предыдущее ДЗ](homework-M06-L06-ienumerable-ienumerator.md) | [← К уроку M06-L07](lesson-M06-L07-yield.md) | [⬆ К модулю M06](../README.md) | [Следующее ДЗ →](homework-M06-L08-span-memory.md)
---

### Домашнее задание M06-L07: yield return, ленивые последовательности / Homework M06-L07: yield return, lazy sequences

**Урок / Lesson:** M06-L07
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться строить ленивые последовательности на `yield return`, грамотно компоновать конвейеры обработки, корректно применять `yield break`, распознавать и устранять типичные ошибки (повторный перебор, побочные эффекты, ограничения на `ref/out/unsafe/лямбды`, многопоточность), а также осознанно выбирать между ленивым `IEnumerable<T>` и материализованной коллекцией. (EN) Learn to build lazy sequences with `yield return`, compose processing pipelines correctly, apply `yield break` properly, recognise and fix common mistakes (multiple enumeration, side effects, restrictions on `ref/out/unsafe/lambdas`, threading), and make a conscious choice between a lazy `IEnumerable<T>` and a materialised collection.

#### Связь с уроком / Connection to the lesson
(RU) Урок рассказывает, что `yield return` превращает метод в конечный автомат, реализующий `IEnumerable<T>`/`IEnumerator<T>`, и что элементы вычисляются по требованию — по одному за вызов `MoveNext()`. Это даёт две ключевые возможности: бесконечные последовательности и ленивые конвейеры без промежуточных массивов. В домашнем задании вы примените обе возможности на реалистичной задаче обработки логов, столкнётесь с ловушками (`yield break`, побочные эффекты, двойной перебор) и научитесь их избегать.
(EN) The lesson explains that `yield return` turns a method into a state machine implementing `IEnumerable<T>`/`IEnumerator<T>`, with elements computed on demand — one per `MoveNext()` call. This enables two key features: infinite sequences and lazy pipelines without intermediate arrays. In this homework you will apply both on a realistic log-processing task, hit the classic traps (`yield break`, side effects, double enumeration) and learn to avoid them.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы поддерживаете сервис мониторинга, который читает лог-файлы размером в десятки гигабайт и потоково обрабатывает события. Загрузить весь файл в `List<string>` невозможно — память процесса исчерпается за секунды. Кроме того, источником логов может быть не только файл, но и бесконечный синтетический генератор, используемый в нагрузочном тестировании: он должен уметь «тикать» сколь угодно долго, а потребитель сам решает, когда остановиться. Именно здесь пригождается `yield return` из урока M06-L07: компилятор превращает метод в итератор — конечный автомат, который выдаёт значения по одному, по требованию, и «замораживает» локальные переменные между вызовами `MoveNext()`.

Вам нужно построить небольшую библиотеку `LazyLog`, состоящую из нескольких `yield`-методов: ленивый ридер лог-файла с досрочной остановкой по маркеру, бесконечный генератор синтетических событий, ленивый фильтр по уровню важности, оператор скользящего окна и оператор `TakeUntil`. Попутно вы должны продемонстрировать две классические ошибки из «Частых ошибок» урока — повторный перебор `IEnumerable` (когда вся работа выполняется дважды) и побочные эффекты в итераторе, которые не запускаются до начала перебора — и предложить корректные альтернативы. Цель — не просто написать рабочий код, а почувствовать ленивость как свойство времени выполнения: когда вычисляется, сколько аллокаций происходит, что произойдёт, если вообще не перебрать результат.

#### Что нужно сделать (пошагово)
1. Создайте консольный проект на .NET 8 командой `dotnet new console -n LazyLog -o LazyLog --framework net8.0`, перейдите в папку `cd LazyLog` и откройте `Program.cs`. Удалите шаблонный код — вы будете писать top-level statements.
2. Определите enum `public enum LogLevel { Trace, Debug, Info, Warn, Error, Fatal }` и запись `public readonly record struct LogEntry(DateTimeOffset Timestamp, LogLevel Level, string Message);`. Используйте именно `record struct`, чтобы избежать лишних аллокаций на горячих путях (это перекликается с замечанием урока о стоимости итератора).
3. Реализуйте `public static IEnumerable<LogEntry> ReadLog(string path)`, который построчно читает файл через `File.ReadLines` (не `ReadAllLines`!), парсит строки формата `ISO-8601|Level|Message` и `yield return`-ит записи. На маркере `"---END---"`, на пустой строке и на любой непарсимой строке вызывайте `yield break` — это досрочное завершение итерации из урока.
4. Реализуйте `public static IEnumerable<LogEntry> GenerateInfinite(LogLevel level, string tag)` — бесконечный генератор: `while (true) { yield return ...; n++; }`. Запустите его через `Take(5)` и `TakeUntil`, чтобы убедиться, что бесконечность безопасна (как в примере `Fibonacci()` из урока).
5. Реализуйте метод расширения `FilterBySeverity(this IEnumerable<LogEntry>, LogLevel min)`, который лениво фильтрует по `e.Level >= min`. Подумайте, почему здесь `yield return`, а не `return source.Where(...)` — обсудите разницу в читаемости и в том, что ваш оператор сам становится звеном конвейера.
6. Реализуйте `Window<T>(this IEnumerable<T> source, int size)`, который выдаёт скользящие окна длины `size` как `IReadOnlyList<T>`. Используйте `Queue<T>` и `yield return buffer.ToArray()`. Окно должно работать и с бесконечным источником — это важное свойство ленивости.
7. Реализуйте `TakeUntil<T>(this IEnumerable<T> source, Func<T,bool> stop)`: выдаёт элементы, пока `stop` не станет истинным (включая сам элемент-триггер), затем `yield break`. Сравните с `Enumerable.TakeWhile`, который элемент-триггер не включает.
8. Напишите два метода-анализатора: `AnalyzeTwiceBad(IEnumerable<LogEntry>)` — вызывает `source.Count()` и `source.Max(...)` по одному и тому же `IEnumerable` (плохо: два полных перебора), и `AnalyzeTwiceGood` — материализует через `.ToList()` один раз. Добавьте в `Bad`-версию побочный эффект `Console.WriteLine` внутри проекции, чтобы наглядно показать, что вычисления запускаются дважды.
9. В `Program.cs` через top-level statements соберите конвейер `ReadLog(path).FilterBySeverity(LogLevel.Warn).Window(2)` и распечатайте окна. Затем запустите `GenerateInfinite(LogLevel.Info, "tick").TakeUntil(e => e.Message.EndsWith("#5"))`. Ожидаемый вывод: для файла из 4 валидных строк после фильтра `Warn` останется 3 записи, окна длины 2 дадут 2 строки; бесконечный генератор остановится на `tick #5`.
10. Запустите проект `dotnet run` и сверьте вывод с ожидаемым. Скомпилируйте со строгими предупреждениями `dotnet build -warnaserror` — код должен быть чистым.

#### Требования к решению
- Целевой фреймворк — `net8.0`, язык C# 12. Допускаются top-level statements, collection expressions (`[]`, `..`), pattern matching (`is`, `or`), raw string literals `"""..."""` для шаблона лога.
- Все ключевые методы обязаны использовать `yield return`/`yield break` — нельзя подменять их возвратом готовой `List<T>`, кроме осознанной материализации в `AnalyzeTwiceGood`.
- В `ReadLog` обязательно применяйте `File.ReadLines` (ленивое перечисление строк), а не `File.ReadAllLines` (жадное чтение всего файла).
- Парсинг должен быть устойчивым: некорректная строка не бросает исключение, а завершает итерацию через `yield break` — как учил урок про досрочное завершение.
- `GenerateInfinite` обязан компилироваться и завершаться за разумное время при `Take(n)`/`TakeUntil(...)` — это прямая проверка того, что вы понимаете безопасность бесконечных последовательностей.
- `Window` и `TakeUntil` реализуйте как обобщённые методы расширения на `IEnumerable<T>`, чтобы их можно было переиспользовать вне домена логов.
- Ни один `yield`-метод не должен иметь `ref`/`out`-параметров, `unsafe`-блоков или быть лямбдой — это запреты из «Подводных камней» урока; нарушите — получите ошибку компиляции.
- Код должен быть покрыт комментариями RU+EN в ключевых точках (`yield break`, `while (true)`, точка материализации).

#### Тонкости и подводные камни
- **Ленивость побочных эффектов.** Если в `AnalyzeTwiceBad` засунуть `Console.WriteLine` внутрь проекции `Select`, он не выполнится при построении запроса — только при переборе. Урок прямо предупреждает: «если вызвать метод, но не перебирать, ничего не произойдёт». Проверьте это, временно закомментировав `foreach`.
- **Двойной перебор.** `source.Count()` и `source.Max(...)` — два независимых перебора одного `IEnumerable`. Для `ReadLog` это два прохода по файлу; для `GenerateInfinite` — бесконечный цикл во втором вызове. Поэтому `AnalyzeTwiceGood` материализует через `ToList()`. Запомните эвристику урока: дорогой источник кэшировать в коллекцию.
- **`yield break` vs `return`.** В обычном методе `return` выходит из функции целиком; в итераторе обычный `return` недопустим (нельзя вернуть `void` из `IEnumerable<T>`-метода без значения) — нужен именно `yield break`, чтобы корректно завершить конечный автомат и пометить `IEnumerator` как завершённый. Достижение конца метода тоже неявно делает `yield break`.
- **Аллокация итератора.** Каждый вызов `yield`-метода создаёт объект-итератор на куче. На горячих путях это может быть заметно; для очень маленьких коллекций иногда выгоднее вернуть массив. Но для лог-файла в 10 ГБ ленивость окупает аллокацию с лихвой.
- **Ограничения.** `yield` нельзя использовать в лямбдах и анонимных методах, в `unsafe`-блоках, в `finally`-блоке `try`, в методах с `ref`/`out`. Если вам нужен `yield` с логикой замыкания — вынесите её в отдельный именованный метод.
- **Многопоточность.** Один объект `IEnumerable` нельзя безопасно перебирать из нескольких потоков. Если хотите параллельной обработки — материализуйте и дробите на партии, либо создавайте свежий `IEnumerable` на поток.
- **`File.ReadLines` vs `File.ReadAllLines`.** Первое возвращает `IEnumerable<string>` и читает по строкам; второе грузит весь файл в массив. В ленивом ридере — только первое.

#### Критерии приёмки
- [ ] Проект `LazyLog` собирается `dotnet build` без предупреждений и ошибок на `net8.0`/C# 12.
- [ ] Определены `LogLevel` (enum) и `LogEntry` (`readonly record struct`).
- [ ] `ReadLog` использует `File.ReadLines` и `yield return`; на маркере/пустой/некорректной строке делает `yield break`.
- [ ] `ReadLog` не бросает исключений на битом входе.
- [ ] `GenerateInfinite` компилируется и безопасно ограничивается через `Take`/`TakeUntil`.
- [ ] `FilterBySeverity` реализован через `yield return` как метод расширения.
- [ ] `Window<T>` ленив, работает с бесконечным источником, возвращает `IReadOnlyList<T>`.
- [ ] `TakeUntil<T>` включает элемент-триггер и затем делает `yield break`.
- [ ] `AnalyzeTwiceBad` действительно дважды перебирает источник; `AnalyzeTwiceGood` материализует один раз.
- [ ] Побочный эффект внутри проекции виден ровно столько раз, сколько переборов — это отражено в комментариях.
- [ ] Ни один `yield`-метод не имеет `ref/out`, `unsafe` или не является лямбдой.
- [ ] В `Program.cs` собраны и запущены оба демонстрационных конвейера.
- [ ] Вывод `dotnet run` соответствует ожидаемому (2 окна отфильтрованных логов + остановка генератора на `tick #5`).
- [ ] Код содержит RU+EN комментарии в ключевых точках.
- [ ] Студент может устно объяснить, почему бесконечная последовательность безопасна и чем `yield break` отличается от обычного `return`.

#### Подсказки (без прямого ответа)
- Для парсинга даты используйте `DateTimeOffset.TryParse` с `out`-параметром — но `out` здесь в обычном методе, не в итераторе, так что ограничение урока не нарушается.
- Для скользящего окна удобно `Queue<T>`: enqueue, проверить `Count == size`, отдать `ToArray()`, dequeue. Подумайте, что отдавать, когда источник закончился, а буфер не полный — обычно ничего.
- `TakeUntil` почти как `TakeWhile`, но с инверсией условия и обязательной выдачей триггерного элемента перед `yield break`.
- Чтобы увидеть побочный эффект «не запускается до перебора», оберните проекцию в метод и поставьте точку логирования.
- Для `AnalyzeTwiceGood` подойдёт кортеж `(int Count, LogLevel Max)` и `.ToList()` ровно один раз.

#### Эталонное решение (разбор)
```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;

public enum LogLevel { Trace, Debug, Info, Warn, Error, Fatal }

// readonly record struct — без аллокации на куче для каждой записи
// readonly record struct — no per-entry heap allocation
public readonly record struct LogEntry(DateTimeOffset Timestamp, LogLevel Level, string Message);

public static class LazyLog
{
    // Ленивый построчный ридер. Маркер "---END---", пустая или битая строка → yield break.
    // Lazy line-by-line reader. Marker, blank or bad line → yield break.
    public static IEnumerable<LogEntry> ReadLog(string path)
    {
        if (!File.Exists(path)) yield break;          // файл absent — корректный пустой итератор
        foreach (var raw in File.ReadLines(path))     // ReadLines — ленивое, не ReadAllLines
        {
            if (raw is "---END---" or "") yield break; // pattern matching + or (C# 9+)
            var parts = raw.Split('|', 3);
            if (parts.Length != 3) yield break;
            if (!DateTimeOffset.TryParse(parts[0], out var ts)) yield break;
            if (!Enum.TryParse<LogLevel>(parts[1], ignoreCase: true, out var lvl)) yield break;
            yield return new LogEntry(ts, lvl, parts[2]);
        }
    }

    // Бесконечный генератор — безопасен благодаря ленивости (как Fibonacci из урока).
    // Infinite generator — safe due to laziness (like the lesson's Fibonacci).
    public static IEnumerable<LogEntry> GenerateInfinite(LogLevel level, string tag)
    {
        var base0 = DateTimeOffset.UtcNow;
        long n = 0;
        while (true)                  // никогда не вернётся сам / never returns on its own
        {
            yield return new LogEntry(base0.AddSeconds(n), level, $"{tag} #{n}");
            n++;
        }
    }

    // Ленивый фильтр по уровню — сам становится звеном конвейера.
    public static IEnumerable<LogEntry> FilterBySeverity(this IEnumerable<LogEntry> source, LogLevel min)
    {
        foreach (var e in source)
            if (e.Level >= min)
                yield return e;
    }

    // Скользящее окно. Ленивое; совместимо с бесконечными источниками.
    public static IEnumerable<IReadOnlyList<T>> Window<T>(this IEnumerable<T> source, int size)
    {
        if (size <= 0) yield break;
        var buffer = new Queue<T>(size);
        foreach (var item in source)
        {
            buffer.Enqueue(item);
            if (buffer.Count == size)
            {
                yield return buffer.ToArray();  // снимок текущего окна / snapshot of current window
                buffer.Dequeue();               // сдвигаем окно вперёд / slide the window
            }
        }
    }

    // TakeUntil: выдаём элементы, пока stop не станет истинным, включая сам триггер, затем yield break.
    public static IEnumerable<T> TakeUntil<T>(this IEnumerable<T> source, Func<T, bool> stop)
    {
        foreach (var item in source)
        {
            yield return item;
            if (stop(item)) yield break;
        }
    }

    // ПЛОХО: Count() и Max() — два независимых перебора одного IEnumerable.
    public static (int Count, LogLevel Max) AnalyzeTwiceBad(IEnumerable<LogEntry> source)
    {
        var count = source.Count();                 // 1-й полный проход
        var max = source.Max(e => e.Level);         // 2-й полный проход — работа дублируется!
        return (count, max);
    }

    // ХОРОШО: материализуем один раз, дальше работаем со списком.
    public static (int Count, LogLevel Max) AnalyzeTwiceGood(IEnumerable<LogEntry> source)
    {
        var list = source.ToList();                 // единственная материализация
        return (list.Count, list.Max(e => e.Level));
    }
}

// --- top-level statements / demo ---

var sample = """
2024-01-01T00:00:00Z|Info|start
2024-01-01T00:00:01Z|Warn|slow query
2024-01-01T00:00:02Z|Error|boom
2024-01-01T00:00:03Z|Fatal|crash
---END---
""";

var path = Path.Combine(Path.GetTempPath(), "lazylog-demo.log");
File.WriteAllText(path, sample);

var pipeline = LazyLog.ReadLog(path)
    .FilterBySeverity(LogLevel.Warn)   // останутся Warn, Error, Fatal → 3 записи
    .Window(2);                        // 2 скользящих окна

foreach (var w in pipeline)
    Console.WriteLine(string.Join(" | ", w.Select(e => $"{e.Level}:{e.Message}")));

var stress = LazyLog.GenerateInfinite(LogLevel.Info, "tick")
    .TakeUntil(e => e.Message.EndsWith("#5"));

foreach (var e in stress)
    Console.WriteLine(e);

File.Delete(path);
```

Разбор по строкам. Строка `if (!File.Exists(path)) yield break;` — корректный способ вернуть «пустой» итератор без исключения: `yield break` здесь эквивалентен немедленному завершению автомата, и `GetEnumerator().MoveNext()` вернёт `false`. Цикл по `File.ReadLines` — это ключевой момент урока: `ReadLines` возвращает `IEnumerable<string>`, который сам читает файл построчно; будь тут `ReadAllLines`, мы бы загрузили весь гигантский файл в память и ленивость `ReadLog` потеряла бы смысл. Конструкция `raw is "---END---" or ""` использует pattern matching C# 9+ — короче и читабельнее цепочки `||`. Каждый некорректный случай завершается `yield break`, а не `throw`: это сознательное решение — битый лог не должен ронять конвейер, он должен обрезаться. `GenerateInfinite` повторяет приём `Fibonacci()` из урока: `while (true)` + `yield return` внутри; безопасность гарантирует потребитель через `Take`/`TakeUntil`. `FilterBySeverity` — пример того, как собственный `yield`-оператор становится звеном конвейера наравне со стандартными `Where`/`Select`. В `Window` используется `Queue<T>` и `buffer.ToArray()`: мы отдаём снимок окна, а не саму очередь — иначе потребитель мог бы мутировать состояние итератора. `TakeUntil` включает триггерный элемент (в отличие от `TakeWhile`) и завершается `yield break` сразу после него. `AnalyzeTwiceBad` иллюстрирует главную ловушку урока — два перебора одного `IEnumerable`: для `ReadLog` это два прохода по файлу, для `GenerateInfinite` — зависание. `AnalyzeTwiceGood` материализует через `ToList()` ровно один раз, что и рекомендуют best practices. Top-level statements в конце собирают оба конвейера и проверяют поведение на живом примере.

#### Задания на углубление (бонус)
1. **`Memoize<T>()`-оператор.** Реализуйте `IEnumerable<T> Memoize<T>(this IEnumerable<T> source)`, который при первом переборе кэширует элементы во внутренний список, а при последующих переборах выдаёт значения из кэша, не запуская источник повторно. Подумайте, как поведёт себя `Memoize` с бесконечным источником (он должен кэшировать только то, что реально было прочитано). Обсудите, почему нельзя просто вернуть `source.ToList()` (потому что это немедленно материализует всё, нарушая ленивость первого прохода).
2. **`Chunk<T>(int size)` для бесконечного источника.** Реализуйте ленивый оператор, выдающий непересекающиеся чанки размера `size`. Убедитесь, что `GenerateInfinite(...).Chunk(3).Take(4)` работает за конечное время. Сравните с `Window` — в чём семантическая разница.
3. **Отмена через `CancellationToken`.** Добавьте перегрузку `ReadLog(string path, CancellationToken ct)`, которая внутри цикла проверяет `ct.ThrowIfCancellationRequested()` или `ct.IsCancellationRequested` и делает `yield break` при отмене. Объясните, почему нельзя передать токен в `GetEnumerator()` напрямую (у стандартного `IEnumerator` нет такого параметра) и почему захват токена в замыкании итератора — рабочий паттерн.
4. **Замер аллокаций.** Напишите небольшой бенчмарк: сгенерируйте 1 000 000 записей через `GenerateInfinite`, прогоните через `FilterBySeverity` и `Window(10)`, замерьте время и аллокации через `GC.GetAllocatedBytesForCurrentThread()` до и после. Сравните с жадной версией, которая сразу делает `ToList()`. Сделайте вывод, в каких случаях ленивость выигрывает.

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you maintain a monitoring service that reads log files tens of gigabytes in size and processes events in a streaming fashion. Loading the entire file into a `List<string>` is impossible — the process memory is exhausted in seconds. Moreover, the source of logs may be not only a file but also an infinite synthetic generator used in load testing: it must be able to "tick" indefinitely, and the consumer decides when to stop. This is exactly where the `yield return` feature from lesson M06-L07 pays off: the compiler turns the method into an iterator — a state machine that yields values one at a time, on demand, and "freezes" local variables between `MoveNext()` calls.

You need to build a small `LazyLog` library consisting of several `yield` methods: a lazy log-file reader with early termination on a marker, an infinite generator of synthetic events, a lazy filter by severity, a sliding-window operator and a `TakeUntil` operator. Along the way you must demonstrate the two classic mistakes listed in the lesson's "Common Mistakes" — multiple enumeration of the same `IEnumerable` (where all the work runs twice) and side effects inside an iterator that do not fire until enumeration begins — and propose correct alternatives. The goal is not merely to write working code but to feel laziness as a runtime property: when computation happens, how many allocations occur, and what happens if you never enumerate the result at all.

#### What to do step by step
1. Create a .NET 8 console project with `dotnet new console -n LazyLog -o LazyLog --framework net8.0`, then `cd LazyLog` and open `Program.cs`. Remove the template — you will write top-level statements.
2. Define `public enum LogLevel { Trace, Debug, Info, Warn, Error, Fatal }` and `public readonly record struct LogEntry(DateTimeOffset Timestamp, LogLevel Level, string Message);`. Use a `record struct` specifically to avoid extra heap allocations on hot paths (this resonates with the lesson's note about the iterator allocation cost).
3. Implement `public static IEnumerable<LogEntry> ReadLog(string path)` that reads the file line by line via `File.ReadLines` (not `ReadAllLines`!), parses lines of the form `ISO-8601|Level|Message` and `yield return`s the entries. On the marker `"---END---"`, on a blank line and on any unparseable line, call `yield break` — the early termination idiom from the lesson.
4. Implement `public static IEnumerable<LogEntry> GenerateInfinite(LogLevel level, string tag)` — an infinite generator: `while (true) { yield return ...; n++; }`. Drive it with `Take(5)` and `TakeUntil` to confirm that infinity is safe (as with the lesson's `Fibonacci()` example).
5. Implement the extension method `FilterBySeverity(this IEnumerable<LogEntry>, LogLevel min)` that lazily filters by `e.Level >= min`. Reflect on why this uses `yield return` rather than `return source.Where(...)` — discuss the readability trade-off and the fact that your operator itself becomes a pipeline stage.
6. Implement `Window<T>(this IEnumerable<T> source, int size)` that yields sliding windows of length `size` as `IReadOnlyList<T>`. Use a `Queue<T>` and `yield return buffer.ToArray()`. The window must work with an infinite source — an important property of laziness.
7. Implement `TakeUntil<T>(this IEnumerable<T> source, Func<T,bool> stop)`: yields elements until `stop` becomes true (including the triggering element), then `yield break`. Compare with `Enumerable.TakeWhile`, which excludes the triggering element.
8. Write two analyzer methods: `AnalyzeTwiceBad(IEnumerable<LogEntry>)` — calls `source.Count()` and `source.Max(...)` on the same `IEnumerable` (bad: two full passes), and `AnalyzeTwiceGood` — materialises once via `.ToList()`. Add a `Console.WriteLine` side effect inside the projection in the `Bad` version to make it obvious that computation runs twice.
9. In `Program.cs` using top-level statements, assemble the pipeline `ReadLog(path).FilterBySeverity(LogLevel.Warn).Window(2)` and print the windows. Then run `GenerateInfinite(LogLevel.Info, "tick").TakeUntil(e => e.Message.EndsWith("#5"))`. Expected output: from a 4-line valid file, after filtering at `Warn` you keep 3 entries, windows of length 2 yield 2 lines; the infinite generator stops at `tick #5`.
10. Run the project with `dotnet run` and compare the output to the expected. Build with strict warnings `dotnet build -warnaserror` — the code must be clean.

#### Requirements
- Target framework `net8.0`, language C# 12. Top-level statements, collection expressions (`[]`, `..`), pattern matching (`is`, `or`) and raw string literals `"""..."""` for the log template are all welcome.
- All key methods must use `yield return`/`yield break` — you may not replace them with returning a ready-made `List<T>`, except for the deliberate materialisation in `AnalyzeTwiceGood`.
- `ReadLog` must use `File.ReadLines` (lazy line enumeration), not `File.ReadAllLines` (greedy whole-file read).
- Parsing must be resilient: a malformed line must not throw — it terminates iteration via `yield break`, exactly as the lesson teaches for early termination.
- `GenerateInfinite` must compile and complete in reasonable time under `Take(n)`/`TakeUntil(...)` — a direct check that you understand the safety of infinite sequences.
- `Window` and `TakeUntil` must be generic extension methods on `IEnumerable<T>` so they are reusable outside the log domain.
- No `yield` method may have `ref`/`out` parameters, `unsafe` blocks or be a lambda — these are the prohibitions from the lesson's "Pitfalls"; break them and you get a compile error.
- The code must carry RU+EN comments at the key points (`yield break`, `while (true)`, the materialisation point).

#### Pitfalls
- **Laziness of side effects.** If you put a `Console.WriteLine` inside a `Select` projection in `AnalyzeTwiceBad`, it will not run when the query is constructed — only when it is enumerated. The lesson warns directly: "if you call the method but never enumerate it, nothing happens." Verify this by temporarily commenting out the `foreach`.
- **Multiple enumeration.** `source.Count()` and `source.Max(...)` are two independent enumerations of the same `IEnumerable`. For `ReadLog` this is two passes over the file; for `GenerateInfinite` the second call hangs forever. That is why `AnalyzeTwiceGood` materialises via `ToList()`. Remember the lesson's heuristic: cache an expensive source into a collection.
- **`yield break` vs `return`.** In a regular method `return` exits the function; in an iterator a plain `return` (without a value, for an `IEnumerable<T>` method) is not allowed — you need `yield break` to properly finish the state machine and mark the `IEnumerator` as done. Reaching the end of the method implicitly performs a `yield break` too.
- **Iterator allocation.** Every call to a `yield` method allocates an iterator object on the heap. On hot paths this can be noticeable; for very small collections it can be cheaper to return an array. For a 10 GB log file, however, laziness pays for the allocation many times over.
- **Restrictions.** `yield` cannot be used inside lambdas or anonymous methods, inside `unsafe` blocks, inside a `finally` block of `try`, or in methods with `ref`/`out`. If you need `yield` with closure-like logic, extract it into a named method of its own.
- **Threading.** A single `IEnumerable` object cannot be safely enumerated from multiple threads at once. If you want parallel processing — materialise and partition, or create a fresh `IEnumerable` per thread.
- **`File.ReadLines` vs `File.ReadAllLines`.** The former returns `IEnumerable<string>` and reads lazily; the latter loads the entire file into an array. In a lazy reader, only the former is acceptable.

#### Acceptance criteria
- [ ] The `LazyLog` project builds with `dotnet build` without warnings or errors on `net8.0`/C# 12.
- [ ] `LogLevel` (enum) and `LogEntry` (`readonly record struct`) are defined.
- [ ] `ReadLog` uses `File.ReadLines` and `yield return`; on marker/blank/malformed line it does `yield break`.
- [ ] `ReadLog` does not throw on broken input.
- [ ] `GenerateInfinite` compiles and is safely bounded by `Take`/`TakeUntil`.
- [ ] `FilterBySeverity` is implemented with `yield return` as an extension method.
- [ ] `Window<T>` is lazy, works with an infinite source, returns `IReadOnlyList<T>`.
- [ ] `TakeUntil<T>` includes the triggering element and then does `yield break`.
- [ ] `AnalyzeTwiceBad` truly enumerates the source twice; `AnalyzeTwiceGood` materialises once.
- [ ] The side effect inside the projection appears exactly as many times as there are enumerations — noted in comments.
- [ ] No `yield` method has `ref/out`, `unsafe`, or is a lambda.
- [ ] Both demo pipelines are assembled and run in `Program.cs`.
- [ ] The output of `dotnet run` matches expectations (2 windows of filtered logs + generator stops at `tick #5`).
- [ ] The code contains RU+EN comments at the key points.
- [ ] The student can verbally explain why an infinite sequence is safe and how `yield break` differs from a plain `return`.

#### Hints (no direct answer)
- For date parsing use `DateTimeOffset.TryParse` with an `out` parameter — but the `out` lives in a regular method, not in the iterator, so the lesson's restriction is not violated.
- A `Queue<T>` is handy for the sliding window: enqueue, check `Count == size`, return `ToArray()`, dequeue. Decide what to yield when the source ends with a partial buffer — usually nothing.
- `TakeUntil` is almost `TakeWhile` but with an inverted condition and a mandatory yield of the triggering element before `yield break`.
- To observe the "does not run until enumerated" property, wrap the projection in a method and put a logging line there.
- For `AnalyzeTwiceGood` a `(int Count, LogLevel Max)` tuple and a single `.ToList()` are enough.

#### Reference solution walk-through
```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;

public enum LogLevel { Trace, Debug, Info, Warn, Error, Fatal }

// readonly record struct — no per-entry heap allocation
public readonly record struct LogEntry(DateTimeOffset Timestamp, LogLevel Level, string Message);

public static class LazyLog
{
    // Lazy line-by-line reader. Marker, blank or bad line → yield break.
    public static IEnumerable<LogEntry> ReadLog(string path)
    {
        if (!File.Exists(path)) yield break;          // absent file → empty iterator
        foreach (var raw in File.ReadLines(path))     // ReadLines — lazy, not ReadAllLines
        {
            if (raw is "---END---" or "") yield break; // pattern matching + or (C# 9+)
            var parts = raw.Split('|', 3);
            if (parts.Length != 3) yield break;
            if (!DateTimeOffset.TryParse(parts[0], out var ts)) yield break;
            if (!Enum.TryParse<LogLevel>(parts[1], ignoreCase: true, out var lvl)) yield break;
            yield return new LogEntry(ts, lvl, parts[2]);
        }
    }

    // Infinite generator — safe due to laziness (like the lesson's Fibonacci).
    public static IEnumerable<LogEntry> GenerateInfinite(LogLevel level, string tag)
    {
        var base0 = DateTimeOffset.UtcNow;
        long n = 0;
        while (true)                  // never returns on its own
        {
            yield return new LogEntry(base0.AddSeconds(n), level, $"{tag} #{n}");
            n++;
        }
    }

    // Lazy severity filter — itself a pipeline stage.
    public static IEnumerable<LogEntry> FilterBySeverity(this IEnumerable<LogEntry> source, LogLevel min)
    {
        foreach (var e in source)
            if (e.Level >= min)
                yield return e;
    }

    // Sliding window. Lazy; compatible with infinite sources.
    public static IEnumerable<IReadOnlyList<T>> Window<T>(this IEnumerable<T> source, int size)
    {
        if (size <= 0) yield break;
        var buffer = new Queue<T>(size);
        foreach (var item in source)
        {
            buffer.Enqueue(item);
            if (buffer.Count == size)
            {
                yield return buffer.ToArray();  // snapshot of current window
                buffer.Dequeue();               // slide the window forward
            }
        }
    }

    // TakeUntil: yield until stop becomes true, including the trigger, then yield break.
    public static IEnumerable<T> TakeUntil<T>(this IEnumerable<T> source, Func<T, bool> stop)
    {
        foreach (var item in source)
        {
            yield return item;
            if (stop(item)) yield break;
        }
    }

    // BAD: Count() and Max() — two independent enumerations of one IEnumerable.
    public static (int Count, LogLevel Max) AnalyzeTwiceBad(IEnumerable<LogEntry> source)
    {
        var count = source.Count();                 // 1st full pass
        var max = source.Max(e => e.Level);         // 2nd full pass — work duplicated!
        return (count, max);
    }

    // GOOD: materialise once, then work with the list.
    public static (int Count, LogLevel Max) AnalyzeTwiceGood(IEnumerable<LogEntry> source)
    {
        var list = source.ToList();                 // single materialisation
        return (list.Count, list.Max(e => e.Level));
    }
}

// --- top-level statements / demo ---

var sample = """
2024-01-01T00:00:00Z|Info|start
2024-01-01T00:00:01Z|Warn|slow query
2024-01-01T00:00:02Z|Error|boom
2024-01-01T00:00:03Z|Fatal|crash
---END---
""";

var path = Path.Combine(Path.GetTempPath(), "lazylog-demo.log");
File.WriteAllText(path, sample);

var pipeline = LazyLog.ReadLog(path)
    .FilterBySeverity(LogLevel.Warn)   // keeps Warn, Error, Fatal → 3 entries
    .Window(2);                        // 2 sliding windows

foreach (var w in pipeline)
    Console.WriteLine(string.Join(" | ", w.Select(e => $"{e.Level}:{e.Message}")));

var stress = LazyLog.GenerateInfinite(LogLevel.Info, "tick")
    .TakeUntil(e => e.Message.EndsWith("#5"));

foreach (var e in stress)
    Console.WriteLine(e);

File.Delete(path);
```

Line-by-line walk-through. The line `if (!File.Exists(path)) yield break;` is the correct way to return an "empty" iterator without throwing: `yield break` here is equivalent to finishing the state machine immediately, so `GetEnumerator().MoveNext()` returns `false`. The loop over `File.ReadLines` is the central point of the lesson: `ReadLines` returns an `IEnumerable<string>` that reads the file line by line; if it were `ReadAllLines` we would load the entire huge file into memory and the laziness of `ReadLog` would be meaningless. The construct `raw is "---END---" or ""` uses C# 9+ pattern matching — shorter and more readable than a chain of `||`. Every malformed case ends with `yield break` rather than `throw`: this is a deliberate decision — a broken log must not crash the pipeline, it must truncate it. `GenerateInfinite` repeats the trick of the lesson's `Fibonacci()`: `while (true)` plus `yield return` inside; safety is provided by the consumer via `Take`/`TakeUntil`. `FilterBySeverity` is an example of how a custom `yield` operator becomes a pipeline stage on equal footing with the standard `Where`/`Select`. In `Window` a `Queue<T>` and `buffer.ToArray()` are used: we hand out a snapshot of the window, not the queue itself — otherwise the consumer could mutate the iterator's state. `TakeUntil` includes the triggering element (unlike `TakeWhile`) and terminates with `yield break` right after it. `AnalyzeTwiceBad` illustrates the lesson's main trap — two enumerations of one `IEnumerable`: for `ReadLog` this is two passes over the file, for `GenerateInfinite` it is a hang. `AnalyzeTwiceGood` materialises with `ToList()` exactly once, as the best practices recommend. The top-level statements at the end assemble both pipelines and check the behaviour on a live example.

#### Going deeper (bonus)
1. **`Memoize<T>()` operator.** Implement `IEnumerable<T> Memoize<T>(this IEnumerable<T> source)` that, on the first enumeration, caches elements in an internal list and, on subsequent enumerations, replays them from the cache without re-running the source. Think about how `Memoize` behaves with an infinite source (it should only cache what was actually read). Discuss why you cannot simply return `source.ToList()` (because that would eagerly materialise everything, breaking the laziness of the first pass).
2. **`Chunk<T>(int size)` for an infinite source.** Implement a lazy operator that yields non-overlapping chunks of size `size`. Make sure `GenerateInfinite(...).Chunk(3).Take(4)` finishes in finite time. Compare with `Window` — what is the semantic difference.
3. **Cancellation via `CancellationToken`.** Add an overload `ReadLog(string path, CancellationToken ct)` that checks `ct.ThrowIfCancellationRequested()` or `ct.IsCancellationRequested` inside the loop and does `yield break` on cancellation. Explain why you cannot pass the token to `GetEnumerator()` directly (the standard `IEnumerator` has no such parameter) and why capturing the token in the iterator's closure is a workable pattern.
4. **Allocation measurement.** Write a small benchmark: generate 1 000 000 entries via `GenerateInfinite`, push them through `FilterBySeverity` and `Window(10)`, measure time and allocations with `GC.GetAllocatedBytesForCurrentThread()` before and after. Compare with a greedy version that does `ToList()` up front. Conclude in which cases laziness wins.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект собирается `dotnet build -warnaserror` на net8.0/C# 12.
- [ ] (RU) Реализованы `ReadLog`, `GenerateInfinite`, `FilterBySeverity`, `Window`, `TakeUntil`.
- [ ] (RU) `AnalyzeTwiceBad` и `AnalyzeTwiceGood` демонстрируют проблему двойного перебора.
- [ ] (RU) Вывод `dotnet run` соответствует ожидаемому.
- [ ] (RU) Код содержит RU+EN комментарии в ключевых точках.
- [ ] (RU) Студент готов устно пояснить ленивость, `yield break` и ограничения.
- [ ] (EN) Project builds with `dotnet build -warnaserror` on net8.0/C# 12.
- [ ] (EN) `ReadLog`, `GenerateInfinite`, `FilterBySeverity`, `Window`, `TakeUntil` are implemented.
- [ ] (EN) `AnalyzeTwiceBad` and `AnalyzeTwiceGood` demonstrate the double-enumeration problem.
- [ ] (EN) `dotnet run` output matches expectations.
- [ ] (EN) Code carries RU+EN comments at the key points.
- [ ] (EN) Student can verbally explain laziness, `yield break` and the restrictions.

#### Ресурсы / Resources
- [Microsoft Learn — yield (C# reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/statements/yield)
- [Microsoft Learn — Iterators (C#)](https://learn.microsoft.com/dotnet/csharp/iterators)
- [Microsoft Learn — File.ReadLines](https://learn.microsoft.com/dotnet/api/system.io.file.readlines)
- [Microsoft Learn — LINQ and deferred execution](https://learn.microsoft.com/dotnet/csharp/linq/deferred-execution-lazy-evaluation)

---
[← Предыдущее ДЗ](homework-M06-L06-ienumerable-ienumerator.md) | [← К уроку M06-L07](lesson-M06-L07-yield.md) | [⬆ К модулю M06](../README.md) | [Следующее ДЗ →](homework-M06-L08-span-memory.md)
