---
[← К уроку M10-L04](lesson-M10-L04-async-file-io.md) | [⬆ К модулю M10](../README.md) | [Следующее ДЗ →](homework-M10-L05-system-text-json.md)
---

### Домашнее задание M10-L04: Async файловые операции / Homework M10-L04: Async file operations

**Урок / Lesson:** M10-L04
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться строить отменяемые, потокобезопасные и масштабируемые асинхронные файловые конвейеры на C# 12 / .NET 8: параллельное чтение с ограничением параллелизма через `SemaphoreSlim`, потоковое чтение больших файлов через `StreamReader.ReadLineAsync` с `FileOptions.Asynchronous`, связывание токенов отмены и таймаутов через `CancellationTokenSource.CreateLinkedTokenSource`, корректное применение `ConfigureAwait(false)` в библиотечном коде. (EN) Learn to build cancellable, thread-safe and scalable asynchronous file pipelines on C# 12 / .NET 8: parallel reads throttled with `SemaphoreSlim`, streaming of large files via `StreamReader.ReadLineAsync` with `FileOptions.Asynchronous`, combining cancellation tokens and timeouts through `CancellationTokenSource.CreateLinkedTokenSource`, and correct use of `ConfigureAwait(false)` in library code.

#### Связь с уроком / Connection to the lesson

(RU) Это задание прямо опирается на все ключевые темы урока M10-L04: `File.*Async` API, потоковое чтение через `StreamReader` с `FileOptions.Asynchronous`, ограничение параллелизма `SemaphoreSlim`, связывание токенов через `CreateLinkedTokenSource` и правило `ConfigureAwait(false)` в библиотеках. Вы воспроизведёте и объедините все «Best Practices» из урока в одном рабочем проекте и избежите каждой ошибки из блока «Частые ошибки». (EN) This homework directly exercises every key topic of lesson M10-L04: the `File.*Async` API, streaming reads through `StreamReader` with `FileOptions.Asynchronous`, `SemaphoreSlim` parallelism throttling, token linking via `CreateLinkedTokenSource`, and the `ConfigureAwait(false)` rule for libraries. You will reproduce and combine all the lesson's "Best Practices" in one working project and avoid every mistake listed in the "Common Mistakes" section.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Представьте, что вы — инженер командыobservability-сервиса. Ежедневно в каталог `logs/` падают десятки текстовых лог-файлов размером от мегабайт до нескольких гигабайт. Синхронный разбор такого объёма блокирует потоки пула, раздувает потребление памяти (из-за `File.ReadAllText`) и не масштабируется при росте числа файлов. Вам нужен компактный, переиспользуемый и корректно асинхронный компонент `LogProcessor`, который умеет: параллельно читать много небольших файлов, не насыщая диск; потоково фильтровать большие файлы без загрузки их целиком в память; писать агрегированный отчёт с таймаутом и отменой; и делать всё это в виде библиотеки, безопасной для вызова из UI, ASP.NET Core и консоли.

Урок M10-L04 даёт для этого весь инструментарий: `File.ReadAllTextAsync`, `File.WriteAllTextAsync`, `StreamReader.ReadLineAsync`, `SemaphoreSlim`, `CancellationTokenSource` (включая вариант с `TimeSpan` и `CreateLinkedTokenSource`), `FileOptions.Asynchronous`, `await using` и `ConfigureAwait(false)`. Цель задания — не просто вызвать эти методы, а собрать их в осмысленный конвейер, где каждая «Best Practice» из урока работает на конкретной задаче, а каждая «Частая ошибка» сознательно обойдена. Заодно вы убедитесь, что `Task.Run(() => File.ReadAllText(...))` — это не async I/O, а занятый поток, и что `FileStream` без `FileOptions.Asynchronous` тихо блокирует пул под капотом.

#### Что нужно сделать (пошагово)

1. **Создайте проект.** Из корня репозитория выполните:
   ```bash
   dotnet new console -n LogProcessor -o modules/M10/homework/LogProcessor --framework net8.0
   cd modules/M10/homework/LogProcessor
   dotnet new sln -n LogProcessor --force
   dotnet sln add LogProcessor.csproj
   ```
   Убедитесь, что в `LogProcessor.csproj` стоит `<TargetFramework>net8.0</TargetFramework>` и `<LangVersion>latest</LangVersion>` (C# 12). Запустите `dotnet build` — должно быть 0 warnings.

2. **Подготовьте тестовые данные.** Создайте каталог `logs/` рядом с проектом. Сгенерируйте скриптом (можно прямо в `Program.cs` через аргумент `--seed`) 20 файлов по ~500 строк формата `2024-01-15 10:23:45 INFO  request handled id=42` и один «большой» файл `big.log` размером ~50 МБ (цикл на 1 000 000 строк). Часть строк должна содержать подстроку `ERROR` — именно их вы будете фильтровать.

3. **Реализуйте класс `LogProcessor`** (в файле `LogProcessor.cs`) с методами:
   - `Task<IReadOnlyDictionary<string,int>> CountLinesAsync(IEnumerable<string> paths, CancellationToken token)` — параллельно подсчитывает количество строк в каждом файле, ограничивая параллелизм через `SemaphoreSlim(8)`.
   - `IAsyncEnumerable<string> ReadErrorLinesAsync(string path, [EnumeratorCancellation] CancellationToken token)` — потоково читает большой файл построчно и `yield return` только строки, содержащие `ERROR`. `FileStream` обязан быть открыт с `FileOptions.Asynchronous`.
   - `Task WriteSummaryAsync(string path, string content, CancellationToken externalToken)` — пишет отчёт с таймаутом 5 секунд, реализованным через `CancellationTokenSource(TimeSpan.FromSeconds(5))`, связанным с внешним токеном через `CreateLinkedTokenSource`. При срабатывании таймаута бросает `TimeoutException` с понятным сообщением.
   - `Task<Dictionary<string,int>> RunAsync(string logsDir, CancellationToken token)` — оркестратор: вызывает три метода выше, формирует строку отчёта (через `StringBuilder` и raw string literal) и пишет её в `summary.txt`.

4. **Top-level `Program.cs`.** В `Program.cs` используйте top-level statements. Создайте `var cts = new CancellationTokenSource();`, привяжите `Console.CancelKeyPress` к `cts.Cancel()`, вызовите `await LogProcessor.RunAsync("logs", cts.Token)`. Выведите на консоль: число обработанных файлов, число найденных `ERROR`-строк и общее время. Прогоните `dotnet run -- --seed` один раз для генерации данных, затем `dotnet run` для обработки.

5. **Проверьте отмену.** Запустите `dotnet run`, дождитесь начала обработки большого файла и нажмите `Ctrl+C`. Убедитесь, что процесс завершается за <1 секунды, а не ждёт окончания чтения 50 МБ. Если не завершается — вы где-то забыли передать `token` в `ReadLineAsync` или не вызвали `token.ThrowIfCancellationRequested()` в цикле.

6. **Проверьте таймаут.** Временно увеличьте объём записываемого отчёта (например, добавьте `big.log` целиком в `content`) и убедитесь, что `WriteSummaryAsync` бросает `TimeoutException` ровно через 5 секунд, а не висит. Затем верните компактный отчёт.

7. **Измерьте пропускную способность.** Сравните две версии `CountLinesAsync`: с `SemaphoreSlim(8)` и без него (прямой `Task.WhenAll` по всем 20 файлам). Логируйте время. На SSD разница может быть небольшой, но на HDD или при 200+ файлах «безлимитная» версия деградирует из-за насыщения очереди I/O — это и есть мотивация throttling-а из урока.

#### Требования к решению

- Код компилируется под .NET 8 / C# 12 без warnings (`<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` приветствуется). Используются top-level statements, `await using`, collection expressions (`new()` target-typed, `[]` для пустых списков где уместно), raw string literals для шаблона отчёта, pattern matching (`is not null`, `is null or empty`).
- Все асинхронные методы возвращают `Task`/`Task<T>`/`IAsyncEnumerable<T>` и принимают `CancellationToken` последним параметром. Токен пробрасывается в каждый внутренний `await` и в `SemaphoreSlim.WaitAsync`.
- В библиотечном классе `LogProcessor` после **каждого** `await` стоит `.ConfigureAwait(false)`. В `Program.cs` (top-level) `ConfigureAwait(false)` можно опустить — это соответствует правилу из урока.
- `FileStream` для потокового чтения создаётся явно с `FileMode.Open`, `FileShare.Read`, `bufferSize: 4096` и `FileOptions.Asynchronous`. `StreamReader` оборачивается в `using`, `FileStream` — в `await using`.
- Параллелизм ограничен `SemaphoreSlim`, начальная ёмкость 8 (константа `MaxConcurrency`). `Release()` вызывается в `finally`.
- Таймаут в `WriteSummaryAsync` реализован именно через `CancellationTokenSource(TimeSpan.FromSeconds(5))`, связанный с внешним токеном через `CreateLinkedTokenSource`, а **не** через `Task.WaitAsync` или ручной `Task.Delay`. Связанный CTS диспозится (`using var`).
- Обработка `OperationCanceledException`: фильтр `when (timeoutCts.IsCancellationRequested && !externalToken.IsCancellationRequested)` отличает «истёк таймаут» от «пользователь отменил» — в первом случае бросаем `TimeoutException`, во втором пробрасываем `OperationCanceledException` как есть.
- Никаких `.Result`, `.Wait()`, `Task.Run(() => File.ReadAllText(...))`. Чтение файлов — только через `File.ReadAllTextAsync` или `StreamReader.ReadLineAsync`.
- Большие файлы **никогда** не читаются через `ReadAllTextAsync` — только потоково. Маленькие (по списку путей) можно через `ReadAllTextAsync` или `ReadAllLinesAsync`.

#### Тонкости и подводные камни

- **`FileOptions.Asynchronous` — обязателен.** Урок прямо предупреждает: если открыть `FileStream` без этого флага, «асинхронный» `ReadLineAsync` под капотом выполнится через блокирующий синхронный I/O, занимая поток пула. Проверьте конструктор `FileStream` — флаг должен быть в `options`. Это самая частая скрытая ошибка, потому что код «работает», но под нагрузкой истощает пул.
- **`ConfigureAwait(false)` — только в библиотеке.** В `LogProcessor` ставьте после каждого `await`, в `Program.cs` — не ставьте. Смешивание (ставите в app или не ставите в lib) — ошибка методологии. В консоли/ASP.NET Core разница незаметна, но привычка должна быть консистентной, иначе в WPF-приложении, которое потом вызовет вашу библиотеку, вы получите дедлок.
- **`Task.WhenAll` без throttle насыщает диск.** 200 файлов через `Task.WhenAll` без `SemaphoreSlim` = 200 одновременных I/O-операций. На SSD это «всего лишь» деградация latency, на HDD — катастрофа (head thrashing). Урок рекомендует 8–16. Берите 8 как безопасный дефолт.
- **`ReadAllTextAsync` для 50 МБ — `OutOfMemoryException` в долгосрочной перспективе.** Не из-за одного файла, а из-за того, что строка плюс UTF-16 декодирование удваивает объём. Потоковое чтение через `StreamReader.ReadLineAsync` держит память постоянной вне зависимости от размера файла.
- **`CancellationToken` надо пробрасывать *везде*.** Забыли передать `token` в `ReadLineAsync` — отмена не сработает до конца файла. Забыли `token.ThrowIfCancellationRequested()` в `IAsyncEnumerable`-цикле — потребитель не сможет отменить между строками на старых рантаймах; в .NET 8 `ReadLineAsync(token)` сам бросает, но явная проверка — оборонительная практика.
- **Связывание токенов.** В `WriteSummaryAsync` нельзя «просто взять таймаутный CTS» — тогда внешняя отмена пользователя не отменит запись. Нужно `CreateLinkedTokenSource(externalToken, timeoutCts.Token)`. И диспозить его (`using var linkedCts = ...`), иначе утечка таймера.
- **`finally { gate.Release(); }` обязательно.** Если `ReadAllTextAsync` бросит, а `Release` стоит после `await` без `finally`, семафор «зависнет» на одной сломанной операции и весь конвейер остановится. Это классика.
- **`OperationCanceledException` vs `TimeoutException`.** Фильтр `catch (OperationCanceledException) when (timeoutCts.IsCancellationRequested && !externalToken.IsCancellationRequested)` — единственный корректный способ отличить. Если пользователь нажал `Ctrl+C`, вы **не** должны маскировать это как `TimeoutException`.

#### Критерии приёмки

- [ ] Проект `LogProcessor` собирается под net8.0 с 0 warnings и `TreatWarningsAsErrors=true`.
- [ ] В `LogProcessor.cs` класс `sealed` (или `static` для методов), методы помечены `static async` и принимают `CancellationToken`.
- [ ] После **каждого** `await` внутри `LogProcessor` стоит `.ConfigureAwait(false)`; в `Program.cs` его нет.
- [ ] `CountLinesAsync` использует `SemaphoreSlim(MaxConcurrency=8)` с `WaitAsync(token)` и `Release()` в `finally`.
- [ ] `ReadErrorLinesAsync` открывает `FileStream` с `FileOptions.Asynchronous`, оборачивает в `await using`, читает `ReadLineAsync(token)`, фильтрует `ERROR` через `Contains`.
- [ ] `WriteSummaryAsync` создаёт `CancellationTokenSource(TimeSpan.FromSeconds(5))` и `CreateLinkedTokenSource`, диспозит оба через `using`.
- [ ] При срабатывании таймаута бросается `TimeoutException` с сообщением, содержащим путь; при внешней отмене пробрасывается `OperationCanceledException`.
- [ ] В коде нет `.Result`, `.Wait()`, `Task.Run(() => File.ReadAllText(...))`, `ReadAllTextAsync` для большого файла.
- [ ] `Program.cs` — top-level statements, привязывает `Console.CancelKeyPress` к `cts.Cancel()`.
- [ ] `dotnet run -- --seed` создаёт каталог `logs/` с 20 маленькими файлами и одним ~50 МБ `big.log`.
- [ ] `dotnet run` выводит число файлов, число `ERROR`-строк и время; `summary.txt` создаётся.
- [ ] `Ctrl+C` во время обработки прерывает процесс за <1 секунды.
- [ ] Искусственно тяжёлый отчёт вызывает `TimeoutException` ровно через 5 секунд.
- [ ] Версия `CountLinesAsync` без `SemaphoreSlim` демонстрируется (через флаг или отдельный метод) и сравнивается по времени с throttled-версией.
- [ ] Все публичные методы документированы XML-комментариями `///` на RU+EN.

#### Подсказки (без прямого ответа)

- Для `IAsyncEnumerable` не забудьте атрибут `[EnumeratorCancellation]` на параметре `token` — иначе `WithCancellation` потребителя не дойдёт до цикла.
- Для шаблона отчёта используйте raw string literal `"""..."""` — он позволяет многострочный текст без экранирования.
- `SemaphoreSlim` диспозится через `using var gate = ...`; но если метод возвращает `IAsyncEnumerable`, будьте осторожны с временем жизни — вынесите `FileStream`/`StreamReader` в тело итератора, а не в параметры.
- Для подсчёта строк в маленьком файле дешевле `await File.ReadAllLinesAsync(path, token)` и `.Length`, чем ручной цикл.
- `Stopwatch.StartNew()` из `System.Diagnostics` — для замеров; не используйте `DateTime.Now`.
- Чтобы сымитировать «тяжёлый» отчёт для проверки таймаута, не пишите 50 МБ на диск постоянно — это медленно и в CI. Лучше временно уменьшите таймаут до 100 мс в тестовой ветке.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — LogProcessor
// Эталонное решение ДЗ M10-L04 / Reference solution for homework M10-L04

using System.Runtime.CompilerServices;
using System.Text;

namespace LogProcessor;

public static class LogProcessor
{
    private const int MaxConcurrency = 8; // throttle, как в уроке / throttle as in lesson

    /// <summary>Параллельно считает строки в файлах, ограничивая параллелизм.
    ///           Counts lines in files in parallel, throttling concurrency.</summary>
    public static async Task<IReadOnlyDictionary<string, int>> CountLinesAsync(
        IEnumerable<string> paths, CancellationToken token)
    {
        using var gate = new SemaphoreSlim(MaxConcurrency);
        var tasks = paths.Select(async path =>
        {
            await gate.WaitAsync(token).ConfigureAwait(false);   // захват слота / acquire slot
            try
            {
                // ReadAllLinesAsync подходит для небольших файлов / fine for small files
                var lines = await File.ReadAllLinesAsync(path, token).ConfigureAwait(false);
                return (path, count: lines.Length);
            }
            finally
            {
                gate.Release(); // освобождаем слот даже при ошибке / release even on error
            }
        }).ToList();

        var results = await Task.WhenAll(tasks).ConfigureAwait(false);
        return results.ToDictionary(r => r.path, r => r.count);
    }

    /// <summary>Потоково фильтрует ERROR-строки большого файла.
    ///           Streams ERROR lines from a large file.</summary>
    public static async IAsyncEnumerable<string> ReadErrorLinesAsync(
        string path,
        [EnumeratorCancellation] CancellationToken token)
    {
        // FileOptions.Asynchronous — критично! / critical flag from the lesson
        await using var fs = new FileStream(
            path, FileMode.Open, FileAccess.Read, FileShare.Read,
            bufferSize: 4096, options: FileOptions.Asynchronous);
        using var reader = new StreamReader(fs, Encoding.UTF8);

        string? line;
        while ((line = await reader.ReadLineAsync(token).ConfigureAwait(false)) is not null)
        {
            token.ThrowIfCancellationRequested(); // оборонительная проверка / defensive check
            if (line.Contains("ERROR", StringComparison.Ordinal))
                yield return line;
        }
    }

    /// <summary>Пишет отчёт с таймаутом 5 сек + внешней отменой.
    ///           Writes a report with a 5s timeout and external cancellation.</summary>
    public static async Task WriteSummaryAsync(
        string path, string content, CancellationToken externalToken)
    {
        using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            externalToken, timeoutCts.Token);

        try
        {
            await File.WriteAllTextAsync(path, content, linkedCts.Token)
                .ConfigureAwait(false);
        }
        // Отличаем таймаут от отмены пользователем / Distinguish timeout from user cancel
        catch (OperationCanceledException)
            when (timeoutCts.IsCancellationRequested && !externalToken.IsCancellationRequested)
        {
            throw new TimeoutException($"Запись в '{path}' превысила 5 секунд. / Write to '{path}' exceeded 5 seconds.");
        }
    }

    /// <summary>Оркестратор конвейера / Pipeline orchestrator.</summary>
    public static async Task<Dictionary<string, int>> RunAsync(
        string logsDir, CancellationToken token)
    {
        var paths = Directory.GetFiles(logsDir, "*.log")
            .Where(p => !Path.GetFileName(p).Equals("big.log", StringComparison.OrdinalIgnoreCase))
            .ToArray();

        var counts = await CountLinesAsync(paths, token).ConfigureAwait(false);
        var errorCount = 0;
        var bigPath = Path.Combine(logsDir, "big.log");
        await foreach (var line in ReadErrorLinesAsync(bigPath, token).ConfigureAwait(false))
            errorCount++;

        var total = counts.Values.Sum();
        var report = $"""
            # Отчёт LogProcessor / LogProcessor report
            Файлов обработано / Files processed : {counts.Count}
            Строк всего / Total lines           : {total}
            ERROR-строк в big.log               : {errorCount}
            Сгенерировано / Generated at        : {DateTimeOffset.Now:O}
            """;

        await WriteSummaryAsync(Path.Combine(logsDir, "summary.txt"), report, token)
            .ConfigureAwait(false);

        return counts.ToDictionary();
    }
}
```

`Program.cs`:

```csharp
// C# 12 / .NET 8 — top-level statements
using System.Diagnostics;
using LogProcessor;

if (args.Contains("--seed"))
{
    SeedData("logs");
    return;
}

using var cts = new CancellationTokenSource();
Console.CancelKeyPress += (_, e) => { e.Cancel = true; cts.Cancel(); };

var sw = Stopwatch.StartNew();
try
{
    var counts = await LogProcessor.RunAsync("logs", cts.Token);
    Console.WriteLine($"Файлов: {counts.Count}, время: {sw.ElapsedMilliseconds} мс");
}
catch (OperationCanceledException)
{
    Console.WriteLine($"Отменено через {sw.ElapsedMilliseconds} мс");
}
```

**Разбор по строкам.** `MaxConcurrency = 8` — прямая цитата рекомендации урока (8–16 на SSD). В `CountLinesAsync` `SemaphoreSlim` заворачивает каждую операцию: `WaitAsync(token)` ждёт свободный слот и одновременно регистрирует отмену; `Release()` в `finally` гарантирует освобождение слота даже при исключении — без этого одна сломанная readFile «съедает» слот навсегда и конвейер умирает (классическая ошибка из урока). `Task.WhenAll` поверх оттроттленных задач даёт общее время, близкое к самому медленному «окну» из 8 файлов, а не к сумме всех. `ConfigureAwait(false)` после каждого `await` — правило для библиотеки: в консоли незаметно, но в WPF/WinForms спасает от дедлока. `ReadAllLinesAsync` здесь допустим, потому что файлы маленькие; для большого файла мы используем потоковый путь.

В `ReadErrorLinesAsync` ключ — конструктор `FileStream` с `FileOptions.Asynchronous`. Урок подчёркивает: без этого флага «асинхронный» `ReadLineAsync` под капотом блокирует поток пула. `await using` для `FileStream` и `using` для `StreamReader` обеспечивают корректное освобождение даже при отмене. `[EnumeratorCancellation]` позволяет `await foreach (...).WithCancellation(token)` потребителя дойти до тела итератора — без атрибута токен молча теряется. `ReadLineAsync(token)` в .NET 8 сам бросает `OperationCanceledException`, но явная `ThrowIfCancellationRequested()` — оборонительная практика для читаемости и совместимости. `yield return` делает потребление памяти постоянным: файл хоть 10 ГБ — в памяти одна строка. `StringComparison.Ordinal` быстрее культуры-зависимого и детерминирован.

`WriteSummaryAsync` — концентрат темы токенов. `CancellationTokenSource(TimeSpan.FromSeconds(5))` создаёт токен, который сам отменится через 5 секунд. Но этого недостаточно: если пользователь нажмёт `Ctrl+C`, мы хотим отменить немедленно, а не ждать 5 секунд. Поэтому `CreateLinkedTokenSource(externalToken, timeoutCts.Token)` объединяет оба — сработает первый, кто сработает. Фильтр `when (timeoutCts.IsCancellationRequested && !externalToken.IsCancellationRequested)` — единственный способ отличить «истёк таймаут» (бросаем `TimeoutException` с понятным сообщением) от «отменил пользователь» (пробрасываем `OperationCanceledException` наверх, где `Program.cs` печатает «Отменено»). `using var` на обоих CTS диспозит таймеры — без этого утечка. В `RunAsync` собираем всё вместе: фильтруем `big.log` из списка путей (его считаем отдельно потоково), агрегируем через `await foreach`, формируем отчёт через raw string literal и пишем через `WriteSummaryAsync`. В `Program.cs` — top-level statements, `Console.CancelKeyPress` → `cts.Cancel()`, `try/catch (OperationCanceledException)`. Здесь `ConfigureAwait(false)` опущен — это верхний уровень, правило урока разрешает.

#### Задания на углубление (бонус)

1. **Прогресс-репортинг через `IProgress<int>`.** Добавьте в `CountLinesAsync` параметр `IProgress<int>? progress` и сообщайте число завершённых файлов. В `Program.cs` выводите прогресс в консоль через `\\r` (карусель). Подумайте, почему `Progress<T>` синхронизирует в контекст захвата, а `IProgress<T>` в библиотеке — нет.
2. **Контрольное чтение с `Channel<T>`.** Перепишите конвейер на `System.Threading.Channels`: продюсер читает пути, воркеры (8) через `ReadAllAsync` забирают и считают, агрегатор собирает. Сравните сложность и throughput с `SemaphoreSlim`-версией.
3. **Настоящий async-копировщик каталогов.** Реализуйте `CopyDirectoryAsync(src, dst, token)` с throttling-ом и прогрессом, используя `FileStream` с `FileOptions.Asynchronous` и `CopyToAsync`. Убедитесь, что отмена по `Ctrl+C` прерывает копирование мгновенно.
4. **Бенчмарк через BenchmarkDotNet.** Сравните `ReadAllTextAsync` vs `StreamReader.ReadLineAsync` vs `MemoryMappedFiles` на файле 100 МБ. Зафиксируйте потребление памяти через `GC.GetAllocatedBytesForCurrentThread()`.

---

## Statement in English / Постановка на английском

#### Context & motivation

Imagine you are an engineer on an observability team. Every day, dozens of text log files ranging from megabytes to several gigabytes land in a `logs/` directory. Parsing that volume synchronously blocks thread-pool threads, blows up memory (because of `File.ReadAllText` loading everything at once), and does not scale as the number of files grows. You need a compact, reusable and correctly asynchronous `LogProcessor` component that can: read many small files in parallel without saturating the disk; stream-filter large files without loading them fully into memory; write an aggregated report with a timeout and cancellation; and do all of this as a library that is safe to call from UI, ASP.NET Core and console hosts.

Lesson M10-L04 gives you the entire toolkit for this: `File.ReadAllTextAsync`, `File.WriteAllTextAsync`, `StreamReader.ReadLineAsync`, `SemaphoreSlim`, `CancellationTokenSource` (including the `TimeSpan` constructor and `CreateLinkedTokenSource`), `FileOptions.Asynchronous`, `await using` and `ConfigureAwait(false)`. The goal of this homework is not just to call these methods, but to assemble them into a meaningful pipeline where every "Best Practice" from the lesson serves a concrete task and every "Common Mistake" is deliberately avoided. Along the way you will convince yourself that `Task.Run(() => File.ReadAllText(...))` is not async I/O but a busy thread, and that a `FileStream` opened without `FileOptions.Asynchronous` silently blocks a pool thread under the hood.

#### What to do step by step

1. **Create the project.** From the repository root run:
   ```bash
   dotnet new console -n LogProcessor -o modules/M10/homework/LogProcessor --framework net8.0
   cd modules/M10/homework/LogProcessor
   dotnet new sln -n LogProcessor --force
   dotnet sln add LogProcessor.csproj
   ```
   Ensure that `LogProcessor.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and `<LangVersion>latest</LangVersion>` (C# 12). Run `dotnet build` — it must produce 0 warnings.

2. **Prepare test data.** Create a `logs/` directory next to the project. Generate with a script (you can put it directly in `Program.cs` behind a `--seed` argument) 20 files of ~500 lines each in the format `2024-01-15 10:23:45 INFO  request handled id=42`, plus one "big" file `big.log` of ~50 MB (a loop of 1 000 000 lines). A subset of the lines must contain the substring `ERROR` — those are the lines you will filter.

3. **Implement the `LogProcessor` class** (in `LogProcessor.cs`) with these methods:
   - `Task<IReadOnlyDictionary<string,int>> CountLinesAsync(IEnumerable<string> paths, CancellationToken token)` — counts the number of lines in each file in parallel, throttling concurrency with `SemaphoreSlim(8)`.
   - `IAsyncEnumerable<string> ReadErrorLinesAsync(string path, [EnumeratorCancellation] CancellationToken token)` — streams the big file line by line and `yield return`s only the lines that contain `ERROR`. The `FileStream` must be opened with `FileOptions.Asynchronous`.
   - `Task WriteSummaryAsync(string path, string content, CancellationToken externalToken)` — writes the report with a 5-second timeout implemented through `CancellationTokenSource(TimeSpan.FromSeconds(5))` linked to the external token via `CreateLinkedTokenSource`. When the timeout fires it throws a `TimeoutException` with a meaningful message.
   - `Task<Dictionary<string,int>> RunAsync(string logsDir, CancellationToken token)` — the orchestrator: it calls the three methods above, builds the report string (using `StringBuilder` and a raw string literal) and writes it to `summary.txt`.

4. **Top-level `Program.cs`.** In `Program.cs` use top-level statements. Create `var cts = new CancellationTokenSource();`, bind `Console.CancelKeyPress` to `cts.Cancel()`, and call `await LogProcessor.RunAsync("logs", cts.Token)`. Print to the console: number of files processed, number of `ERROR` lines found, and total elapsed time. Run `dotnet run -- --seed` once to generate the data, then `dotnet run` to process it.

5. **Verify cancellation.** Start `dotnet run`, wait until the big file starts being processed, then press `Ctrl+C`. Confirm that the process exits in under 1 second instead of waiting for the 50 MB read to finish. If it does not exit promptly, you forgot to pass `token` to `ReadLineAsync` somewhere or omitted `token.ThrowIfCancellationRequested()` in the loop.

6. **Verify the timeout.** Temporarily inflate the report payload (for example, embed the whole `big.log` into `content`) and confirm that `WriteSummaryAsync` throws `TimeoutException` exactly after 5 seconds instead of hanging. Then restore the compact report.

7. **Measure throughput.** Compare two versions of `CountLinesAsync`: one with `SemaphoreSlim(8)` and one without it (a plain `Task.WhenAll` over all 20 files). Log the timings. On an SSD the gap may be small, but on an HDD or with 200+ files the "unlimited" version degrades because the I/O queue saturates — that is the motivation for throttling from the lesson.

#### Requirements

- The code compiles under .NET 8 / C# 12 with no warnings (`<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` is welcome). Use top-level statements, `await using`, collection expressions (target-typed `new()`, `[]` for empty lists where appropriate), raw string literals for the report template, and pattern matching (`is not null`, `is null or empty`).
- All async methods return `Task`/`Task<T>`/`IAsyncEnumerable<T>` and accept a `CancellationToken` as the last parameter. The token is forwarded to every inner `await` and to `SemaphoreSlim.WaitAsync`.
- In the library class `LogProcessor`, `.ConfigureAwait(false)` appears after **every** `await`. In `Program.cs` (top-level) it may be omitted — this matches the rule from the lesson.
- The `FileStream` for streaming is created explicitly with `FileMode.Open`, `FileShare.Read`, `bufferSize: 4096` and `FileOptions.Asynchronous`. The `StreamReader` is wrapped in `using`, the `FileStream` in `await using`.
- Concurrency is capped with `SemaphoreSlim`, initial capacity 8 (the `MaxConcurrency` constant). `Release()` is called in a `finally` block.
- The timeout in `WriteSummaryAsync` is implemented through `CancellationTokenSource(TimeSpan.FromSeconds(5))` linked with the external token via `CreateLinkedTokenSource`, **not** through `Task.WaitAsync` or a manual `Task.Delay`. The linked CTS is disposed (`using var`).
- Handling of `OperationCanceledException`: a `when (timeoutCts.IsCancellationRequested && !externalToken.IsCancellationRequested)` filter distinguishes "the timeout fired" from "the user cancelled" — in the first case throw `TimeoutException`, in the second rethrow `OperationCanceledException` unchanged.
- No `.Result`, no `.Wait()`, no `Task.Run(() => File.ReadAllText(...))`. File reads go only through `File.ReadAllTextAsync` or `StreamReader.ReadLineAsync`.
- Large files are **never** read with `ReadAllTextAsync` — only streamed. Small files (from a path list) may use `ReadAllTextAsync` or `ReadAllLinesAsync`.

#### Pitfalls

- **`FileOptions.Asynchronous` is mandatory.** The lesson warns explicitly: opening a `FileStream` without this flag makes the "async" `ReadLineAsync` fall back to blocking synchronous I/O under the hood, consuming a pool thread. Check the `FileStream` constructor — the flag must be in `options`. This is the most common hidden mistake because the code "works" but starves the pool under load.
- **`ConfigureAwait(false)` only in libraries.** Put it after every `await` in `LogProcessor`, do not put it in `Program.cs`. Mixing (putting it in the app, or omitting it in the library) is a methodological error. In console / ASP.NET Core the difference is invisible, but the habit must be consistent; otherwise a WPF application that later calls your library will deadlock.
- **`Task.WhenAll` without a throttle saturates the disk.** 200 files through `Task.WhenAll` without `SemaphoreSlim` means 200 concurrent I/O operations. On an SSD that is "only" latency degradation; on an HDD it is catastrophic head thrashing. The lesson recommends 8–16. Pick 8 as a safe default.
- **`ReadAllTextAsync` on 50 MB is an `OutOfMemoryException` waiting to happen** — not because of a single file, but because strings plus UTF-16 decoding roughly double the footprint. Streaming via `StreamReader.ReadLineAsync` keeps memory constant regardless of file size.
- **The `CancellationToken` must be forwarded *everywhere*.** Forgot to pass `token` to `ReadLineAsync`? Cancellation will not fire until end of file. Forgot `token.ThrowIfCancellationRequested()` in the `IAsyncEnumerable` loop? The consumer cannot cancel between lines on older runtimes; in .NET 8 `ReadLineAsync(token)` throws on its own, but the explicit check is defensive.
- **Linking tokens.** In `WriteSummaryAsync` you cannot "just take the timeout CTS" — then a user cancellation would not cancel the write. You need `CreateLinkedTokenSource(externalToken, timeoutCts.Token)`, and you must dispose it (`using var linkedCts = ...`) to avoid a timer leak.
- **`finally { gate.Release(); }` is mandatory.** If `ReadAllTextAsync` throws and `Release` is after the `await` without a `finally`, the semaphore gets stuck on a single broken operation and the whole pipeline stalls. Classic.
- **`OperationCanceledException` vs `TimeoutException`.** The `catch (OperationCanceledException) when (timeoutCts.IsCancellationRequested && !externalToken.IsCancellationRequested)` filter is the only correct way to tell them apart. If the user pressed `Ctrl+C`, you must **not** mask it as a `TimeoutException`.

#### Acceptance criteria

- [ ] The `LogProcessor` project builds on net8.0 with 0 warnings and `TreatWarningsAsErrors=true`.
- [ ] In `LogProcessor.cs` the class is `sealed` (or methods are `static`), methods are `static async` and accept a `CancellationToken`.
- [ ] After **every** `await` inside `LogProcessor` there is `.ConfigureAwait(false)`; in `Program.cs` there is none.
- [ ] `CountLinesAsync` uses `SemaphoreSlim(MaxConcurrency=8)` with `WaitAsync(token)` and `Release()` in a `finally`.
- [ ] `ReadErrorLinesAsync` opens a `FileStream` with `FileOptions.Asynchronous`, wraps it in `await using`, reads with `ReadLineAsync(token)`, filters `ERROR` via `Contains`.
- [ ] `WriteSummaryAsync` creates `CancellationTokenSource(TimeSpan.FromSeconds(5))` and `CreateLinkedTokenSource`, disposes both via `using`.
- [ ] On timeout it throws `TimeoutException` whose message contains the path; on external cancellation it rethrows `OperationCanceledException`.
- [ ] The code has no `.Result`, `.Wait()`, `Task.Run(() => File.ReadAllText(...))`, or `ReadAllTextAsync` on the big file.
- [ ] `Program.cs` is top-level statements and binds `Console.CancelKeyPress` to `cts.Cancel()`.
- [ ] `dotnet run -- --seed` creates the `logs/` directory with 20 small files and one ~50 MB `big.log`.
- [ ] `dotnet run` prints the number of files, the number of `ERROR` lines and the elapsed time; `summary.txt` is created.
- [ ] `Ctrl+C` during processing stops the process in under 1 second.
- [ ] An artificially heavy report triggers a `TimeoutException` exactly after 5 seconds.
- [ ] A `CountLinesAsync` variant without `SemaphoreSlim` is demonstrated (via a flag or a separate method) and timed against the throttled version.
- [ ] All public methods have `///` XML doc comments in RU+EN.

#### Hints (no direct answer)

- For `IAsyncEnumerable`, do not forget the `[EnumeratorCancellation]` attribute on the `token` parameter — otherwise the consumer's `WithCancellation` will not reach the loop body.
- For the report template, use a raw string literal `"""..."""` — it allows multi-line text without escaping.
- `SemaphoreSlim` is disposed via `using var gate = ...`; but if a method returns an `IAsyncEnumerable`, be careful with lifetimes — keep the `FileStream`/`StreamReader` in the iterator body, not in parameters.
- For counting lines in a small file, `await File.ReadAllLinesAsync(path, token)` and `.Length` is cheaper than a manual loop.
- `Stopwatch.StartNew()` from `System.Diagnostics` is for timing; do not use `DateTime.Now`.
- To simulate a "heavy" report for the timeout test, do not constantly write 50 MB to disk — it is slow and CI-unfriendly. Temporarily lower the timeout to 100 ms on a test branch instead.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — LogProcessor
// Reference solution for homework M10-L04

using System.Runtime.CompilerServices;
using System.Text;

namespace LogProcessor;

public static class LogProcessor
{
    private const int MaxConcurrency = 8; // throttle, as recommended in the lesson

    /// <summary>Counts lines in files in parallel, throttling concurrency.</summary>
    public static async Task<IReadOnlyDictionary<string, int>> CountLinesAsync(
        IEnumerable<string> paths, CancellationToken token)
    {
        using var gate = new SemaphoreSlim(MaxConcurrency);
        var tasks = paths.Select(async path =>
        {
            await gate.WaitAsync(token).ConfigureAwait(false);   // acquire a slot
            try
            {
                // ReadAllLinesAsync is fine for small files
                var lines = await File.ReadAllLinesAsync(path, token).ConfigureAwait(false);
                return (path, count: lines.Length);
            }
            finally
            {
                gate.Release(); // release the slot even on error
            }
        }).ToList();

        var results = await Task.WhenAll(tasks).ConfigureAwait(false);
        return results.ToDictionary(r => r.path, r => r.count);
    }

    /// <summary>Streams ERROR lines from a large file.</summary>
    public static async IAsyncEnumerable<string> ReadErrorLinesAsync(
        string path,
        [EnumeratorCancellation] CancellationToken token)
    {
        // FileOptions.Asynchronous — critical flag from the lesson
        await using var fs = new FileStream(
            path, FileMode.Open, FileAccess.Read, FileShare.Read,
            bufferSize: 4096, options: FileOptions.Asynchronous);
        using var reader = new StreamReader(fs, Encoding.UTF8);

        string? line;
        while ((line = await reader.ReadLineAsync(token).ConfigureAwait(false)) is not null)
        {
            token.ThrowIfCancellationRequested(); // defensive check
            if (line.Contains("ERROR", StringComparison.Ordinal))
                yield return line;
        }
    }

    /// <summary>Writes a report with a 5s timeout and external cancellation.</summary>
    public static async Task WriteSummaryAsync(
        string path, string content, CancellationToken externalToken)
    {
        using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            externalToken, timeoutCts.Token);

        try
        {
            await File.WriteAllTextAsync(path, content, linkedCts.Token)
                .ConfigureAwait(false);
        }
        // Distinguish timeout from user cancellation
        catch (OperationCanceledException)
            when (timeoutCts.IsCancellationRequested && !externalToken.IsCancellationRequested)
        {
            throw new TimeoutException($"Write to '{path}' exceeded 5 seconds.");
        }
    }

    /// <summary>Pipeline orchestrator.</summary>
    public static async Task<Dictionary<string, int>> RunAsync(
        string logsDir, CancellationToken token)
    {
        var paths = Directory.GetFiles(logsDir, "*.log")
            .Where(p => !Path.GetFileName(p).Equals("big.log", StringComparison.OrdinalIgnoreCase))
            .ToArray();

        var counts = await CountLinesAsync(paths, token).ConfigureAwait(false);
        var errorCount = 0;
        var bigPath = Path.Combine(logsDir, "big.log");
        await foreach (var line in ReadErrorLinesAsync(bigPath, token).ConfigureAwait(false))
            errorCount++;

        var total = counts.Values.Sum();
        var report = $"""
            # LogProcessor report
            Files processed : {counts.Count}
            Total lines     : {total}
            ERROR lines in big.log : {errorCount}
            Generated at    : {DateTimeOffset.Now:O}
            """;

        await WriteSummaryAsync(Path.Combine(logsDir, "summary.txt"), report, token)
            .ConfigureAwait(false);

        return counts.ToDictionary();
    }
}
```

`Program.cs`:

```csharp
// C# 12 / .NET 8 — top-level statements
using System.Diagnostics;
using LogProcessor;

if (args.Contains("--seed"))
{
    SeedData("logs");
    return;
}

using var cts = new CancellationTokenSource();
Console.CancelKeyPress += (_, e) => { e.Cancel = true; cts.Cancel(); };

var sw = Stopwatch.StartNew();
try
{
    var counts = await LogProcessor.RunAsync("logs", cts.Token);
    Console.WriteLine($"Files: {counts.Count}, elapsed: {sw.ElapsedMilliseconds} ms");
}
catch (OperationCanceledException)
{
    Console.WriteLine($"Cancelled after {sw.ElapsedMilliseconds} ms");
}
```

**Line-by-line walk-through.** `MaxConcurrency = 8` is a direct quote of the lesson's recommendation (8–16 on an SSD). In `CountLinesAsync`, the `SemaphoreSlim` wraps each operation: `WaitAsync(token)` waits for a free slot and simultaneously registers cancellation; `Release()` in `finally` guarantees that the slot is freed even if the read throws — without that, a single broken read would consume a slot forever and the pipeline would die (the classic mistake called out in the lesson). `Task.WhenAll` over the throttled tasks gives a total time close to the slowest "window" of 8 files, not the sum of all. `ConfigureAwait(false)` after every `await` is the library rule: invisible in a console, but it saves a WPF/WinForms caller from a deadlock. `ReadAllLinesAsync` is acceptable here because the files are small; for the big file we use the streaming path.

In `ReadErrorLinesAsync`, the crux is the `FileStream` constructor with `FileOptions.Asynchronous`. The lesson stresses that without this flag the "async" `ReadLineAsync` falls back to blocking a pool thread under the hood. `await using` on the `FileStream` and `using` on the `StreamReader` ensure correct disposal even on cancellation. `[EnumeratorCancellation]` lets the consumer's `await foreach (...).WithCancellation(token)` reach the iterator body — without the attribute the token is silently dropped. `ReadLineAsync(token)` in .NET 8 throws `OperationCanceledException` on its own, but the explicit `ThrowIfCancellationRequested()` is a defensive, readable practice. `yield return` keeps memory constant: a 10 GB file still holds one line in memory at a time. `StringComparison.Ordinal` is faster and deterministic compared to culture-aware comparisons.

`WriteSummaryAsync` is the concentrate of the token topic. `CancellationTokenSource(TimeSpan.FromSeconds(5))` creates a token that self-cancels after 5 seconds. But that alone is not enough: if the user presses `Ctrl+C`, we want to cancel immediately, not wait 5 seconds. That is why `CreateLinkedTokenSource(externalToken, timeoutCts.Token)` unions the two — whichever fires first wins. The `when (timeoutCts.IsCancellationRequested && !externalToken.IsCancellationRequested)` filter is the only way to tell "the timeout fired" (throw a `TimeoutException` with a clear message) from "the user cancelled" (rethrow `OperationCanceledException` so `Program.cs` can print "Cancelled"). `using var` on both CTS disposes the timers — without that you leak. In `RunAsync` we tie it all together: we exclude `big.log` from the path list (it is counted separately via streaming), aggregate with `await foreach`, build the report with a raw string literal, and write through `WriteSummaryAsync`. In `Program.cs` — top-level statements, `Console.CancelKeyPress` → `cts.Cancel()`, `try/catch (OperationCanceledException)`. `ConfigureAwait(false)` is omitted here because this is the top level, which the lesson explicitly permits.

#### Going deeper (bonus)

1. **Progress reporting via `IProgress<int>`.** Add an `IProgress<int>? progress` parameter to `CountLinesAsync` and report the number of completed files. In `Program.cs` render progress with a `\\r` spinner. Reason about why `Progress<T>` captures a synchronization context while a raw `IProgress<T>` in a library does not.
2. **Pipeline with `Channel<T>`.** Rewrite the pipeline on top of `System.Threading.Channels`: a producer feeds paths, 8 workers drain via `ReadAllAsync` and count, an aggregator collects results. Compare the complexity and throughput against the `SemaphoreSlim` version.
3. **A real async directory copier.** Implement `CopyDirectoryAsync(src, dst, token)` with throttling and progress, using `FileStream` with `FileOptions.Asynchronous` and `CopyToAsync`. Verify that `Ctrl+C` aborts the copy instantly.
4. **Benchmark with BenchmarkDotNet.** Compare `ReadAllTextAsync` vs `StreamReader.ReadLineAsync` vs `MemoryMappedFiles` on a 100 MB file. Capture allocations via `GC.GetAllocatedBytesForCurrentThread()`.

---

#### Чек-лист сдачи / Submission checklist

- [ ] (RU) Проект `LogProcessor` собирается под net8.0 с 0 warnings и `TreatWarningsAsErrors=true`.
- [ ] (RU) Класс `LogProcessor` — `sealed`/`static`, все методы — `static async` с `CancellationToken`.
- [ ] (RU) `ConfigureAwait(false)` после каждого `await` в библиотеке; отсутствует в `Program.cs`.
- [ ] (RU) `SemaphoreSlim(8)` с `WaitAsync(token)` и `Release()` в `finally`.
- [ ] (RU) `FileStream` с `FileOptions.Asynchronous`, `await using`, `ReadLineAsync(token)`.
- [ ] (RU) `CreateLinkedTokenSource(externalToken, timeoutCts.Token)`, `using var`, `TimeoutException` при таймауте.
- [ ] (RU) Нет `.Result`/`.Wait()`/`Task.Run(File.ReadAllText)`; большой файл — только потоково.
- [ ] (RU) `Ctrl+C` прерывает за <1 секунды; таймаут срабатывает ровно через 5 секунд.
- [ ] (RU) Сравнение throttled vs unthrottled `CountLinesAsync` с замером времени.
- [ ] (RU) XML-комментарии `///` на RU+EN у всех публичных методов.
- [ ] (EN) The `LogProcessor` project builds on net8.0 with 0 warnings and `TreatWarningsAsErrors=true`.
- [ ] (EN) `LogProcessor` is `sealed`/`static`; all methods are `static async` with a `CancellationToken`.
- [ ] (EN) `ConfigureAwait(false)` after every `await` in the library; absent in `Program.cs`.
- [ ] (EN) `SemaphoreSlim(8)` with `WaitAsync(token)` and `Release()` in a `finally`.
- [ ] (EN) `FileStream` with `FileOptions.Asynchronous`, `await using`, `ReadLineAsync(token)`.
- [ ] (EN) `CreateLinkedTokenSource(externalToken, timeoutCts.Token)`, `using var`, `TimeoutException` on timeout.
- [ ] (EN) No `.Result`/`.Wait()`/`Task.Run(File.ReadAllText)`; the big file is streamed only.
- [ ] (EN) `Ctrl+C` aborts in under 1 second; the timeout fires exactly after 5 seconds.
- [ ] (EN) A throttled-vs-unthrottled `CountLinesAsync` comparison with timings.
- [ ] (EN) `///` XML doc comments in RU+EN on every public method.

#### Ресурсы / Resources

- [Microsoft Learn — Asynchronous File I/O](https://learn.microsoft.com/dotnet/standard/io/asynchronous-file-i-o)
- [Microsoft Learn — CancellationTokenSource](https://learn.microsoft.com/dotnet/api/system.threading.cancellationtokensource)
- [Microsoft Learn — SemaphoreSlim](https://learn.microsoft.com/dotnet/api/system.threading.semaphoreslim)
- [Microsoft Learn — FileOptions.Asynchronous](https://learn.microsoft.com/dotnet/api/system.io.fileoptions)
- [Stephen Toub — Async FAQ: ConfigureAwait](https://devblogs.microsoft.com/dotnet/configureawait-faq/)
- [Microsoft Learn — IAsyncEnumerable with EnumeratorCancellation](https://learn.microsoft.com/dotnet/csharp/asynchronous-programming/generate-consume-asynchronous-streams)
