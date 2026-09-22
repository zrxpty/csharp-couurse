---
[← К уроку M11-L07](lesson-M11-L07-channels.md) | [⬆ К модулю M11](../README.md) | [Следующее ДЗ →](homework-M11-L08-parallel-for.md)
---

### Домашнее задание M11-L07: Channel<T>, producer/consumer / Homework M11-L07: Channel<T>, producer/consumer

**Урок / Lesson:** M11-L07
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться строить устойчивый многопоточный producer/consumer-конвейер на `System.Threading.Channels` с корректным backpressure, отменой и race-free завершением, без дедлоков и sync-over-async. (EN) Learn to build a robust multithreaded producer/consumer pipeline on `System.Threading.Channels` with correct backpressure, cancellation, and race-free completion — no deadlocks, no sync-over-async.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит `Channel<T>` как потокобезопасную очередь, оптимизированную под асинхронное использование, и объясняет разницу между bounded/unbounded режимами, API `ChannelReader`/`ChannelWriter`, backpressure и правила защиты от дедлоков. ДЗ закрепляет все эти темы: вы построите трёхстадийный конвейер с двумя каналами, счётчиком producer'ов, `CancellationToken` и `await foreach`, наступая ровно на те грабли, которые описаны в разделе «Частые ошибки».
(EN) The lesson introduces `Channel<T>` as a thread-safe queue tuned for async use, explaining bounded/unbounded modes, the `ChannelReader`/`ChannelWriter` API, backpressure, and the deadlock-avoidance rules. This homework fixes all of those topics: you will build a three-stage pipeline with two channels, a producer counter, a `CancellationToken`, and `await foreach` — stepping on exactly the rake described in the "Common Mistakes" section.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Представьте, что вы пишете сервис индексации текстовых файлов для поискового движка. На входе — каталог с тысячами `.txt`-файлов разного размера, на выходе — сводный файл вида `путь\tколичество_слов`. Файлы лежат на медленном диске, чтение — I/O-bound операция, а подсчёт слов — лёгкая, но всё же CPU-bound работа. Запускать всё последовательно невыгодно: диск простаивает, пока вы считаете, а процессор простаивает, пока вы читаете. Запускать «всё в один `Parallel.ForEach`» тоже плохо: `Parallel.ForEach` плохо сочетается с `async` I/O внутри и не даёт естественного ограничения параллелизма между стадиями.

Классический выход — многостадийный конвейер (pipeline): одна стадия перечисляет и читает файлы, вторая считает слова, третья записывает результаты. Стадии связаны между собой буферами ограниченной длины. Если стадия записи не справляется, она должна автоматически «прижать» предыдущие стадии — чтобы память не росла бесконтрольно. Именно эту задачу элегантно решает `System.Threading.Channels`: bounded-канал даёт бесплатный backpressure, `await foreach` + `ReadAllAsync` дают race-free завершение, а `CancellationToken` — единую точку остановки. В этом задании вы построите такой конвейер целиком и убедитесь, что он корректно ведёт себя при отмене, при большой нагрузке и при ошибках отдельной стадии.

#### Что нужно сделать (пошагово)

1. Создайте новый проект консольного приложения на .NET 8:
   ```
   dotnet new console -n FileIndexer -o FileIndexer --framework net8.0
   cd FileIndexer
   ```
   Убедитесь, что в `FileIndexer.csproj` стоит `<Nullable>enable</Nullable>` и `<LangVersion>latest</LangVersion>` (или `preview`). Пакет `System.Threading.Channels` уже входит в .NET 8 как часть разделяемого фреймворка — отдельную ссылку ставить не нужно.

2. Подготовьте тестовые данные. Создайте папку `testdata` и сгенерируйте в ней 200 `.txt`-файлов разного размера (например, скриптом на PowerShell или короткой C#-программой). Часть файлов сделайте большими (сотни тысяч слов), часть — пустыми. Пустые файлы — важный кейс: они проверяют, что подсчёт не падает на `Split` пустой строки.

3. Реализуйте трёхстадийный конвейер с двумя каналами:
   - **Стадия 1 (producer):** перечисляет `.txt`-файлы в каталоге через `Directory.EnumerateFiles` и пишет их пути в `fileChannel`. Один producer.
   - **Стадия 2 (parsers):** несколько consumer'ов (например, 4), читающих пути из `fileChannel`, асинхронно читающих содержимое файла через `File.ReadAllTextAsync`, считающих слова и пишущих результат `WordCount(Path, Count)` в `resultChannel`.
   - **Стадия 3 (writers):** несколько consumer'ов (например, 2), читающих `WordCount` из `resultChannel` и записывающих строки `путь\tколичество` в один общий сводный файл `summary.txt` (или в отдельные файлы `summary-0.txt`, `summary-1.txt` — на ваше усмотрение).

4. Используйте bounded-каналы с осознанной вместимостью (например, 32) и `BoundedChannelFullMode.Wait`. Объясните в комментариях, почему выбран `Wait`, а не `DropWrite`.

5. Заведите единый `CancellationTokenSource`. Передавайте токен во все `WriteAsync`/`ReadAllAsync`/`ReadAllAsync`/`File.ReadAllTextAsync`. Добавьте обработчик `Console.CancelKeyPress`, который вызывает `cts.Cancel()` — чтобы `Ctrl+C` корректно гасил конвейер, а не рвал его посередине.

6. Реализуйте корректное закрытие каналов: первый канал закрывает producer в `finally` (он один, поэтому счётчик не нужен, но вызывайте `TryComplete()` — он идемпотентен). Второй канал закрывает последний parser через счётчик `Interlocked.Decrement` — потому что parser'ов несколько, и ранний `Complete()` оборвёт остальных.

7. Точка джойна: `await Task.WhenAll(producers)`, затем `await Task.WhenAll(parsers)`, затем `await Task.WhenAll(writers)`. Оберните всё в `try/catch (OperationCanceledException)` и `finally`, где освобождается `cts`.

8. Запустите конвейер:
   ```
   dotnet run --project FileIndexer -- ./testdata
   ```
   Ожидаемый вывод: строки лога о завершении каждого consumer'а и финальное «Pipeline completed cleanly». Сводный файл `summary.txt` должен содержать по одной строке на каждый входной файл. Проверьте, что количество строк в `summary.txt` равно количеству `.txt`-файлов в `testdata`.

9. Проведите эксперимент с отменой: запустите конвейер на большом каталоге, через пару секунд нажмите `Ctrl+C`. Убедитесь, что процесс завершается за разумное время (доли секунды), а не висит в `await`. Если висит — вы забыли передать `ct` в какую-то операцию.

10. Проведите эксперимент с backpressure: временно замедлите writer'ов (например, `await Task.Delay(50)` на каждую строку). Запустите на большом каталоге и снимите потребление памяти процесса (через `dotnet-counters monitor` или Диспетчер задач). С bounded-каналом память должна стабилизироваться; с unbounded — расти до OOM. Зафиксируйте наблюдение в комментарии.

#### Требования к решению

- Код должен компилироваться и запускаться на .NET 8 с C# 12 (top-level statements, `record struct`, `Random.Shared`, collection expressions, где уместно).
- Используются ровно два канала: `Channel<string>` для путей и `Channel<WordCount>` для результатов. Оба bounded с `BoundedChannelFullMode.Wait`.
- `CancellationToken` проходит во все async-операции: `WriteAsync`, `ReadAllAsync`, `File.ReadAllTextAsync`, `Task.Delay`, запись в `StreamWriter`. Нигде нет «голого» `await` без токена, кроме точек, где токен технически неприменим.
- Нигде нет `.Result` или `.Wait()` на асинхронных операциях канала. Нигде нет `lock` вокруг `await`. Нигде нет `async void` для consumer'ов.
- `writer.TryComplete()` вызывается ровно один раз на каждый канал: для первого — в `finally` producer'а, для второго — через счётчик `Interlocked.Decrement` в `finally` последнего parser'а.
- Consumer'ы используют `await foreach (... reader.ReadAllAsync(ct))`. Допускается альтернатива `WaitToReadAsync` + `TryRead`, но тогда с комментарием-обоснованием.
- Точка джойна — три последовательных `await Task.WhenAll(...)`, обёрнутые в `try/catch (OperationCanceledException)/finally`.
- `ConfigureAwait(false)` проставлен во всех async-операциях библиотечного стиля (в консольном приложении это не строго обязательно, но в задании мы тренируем привычку для переносимого кода).
- `SingleReader`/`SingleWriter` выставлены осознанно: для `fileChannel` `SingleWriter = true` (producer один), `SingleReader = false` (parser'ов несколько); для `resultChannel` `SingleWriter = false`, `SingleReader = false` — либо `SingleReader = true`, если writer один.
- Программа выводит итоговую статистику: сколько файлов обработано, сколько слов всего, за сколько миллисекунд.

#### Тонкости и подводные камни

- **Не забывайте `TryComplete()`, а не `Complete()`.** `TryComplete()` идемпотентен — повторные вызовы игнорируются. Это критично, когда несколько parser'ов в `finally` добирают счётчик до нуля: даже если произойдёт гонка и условие сработает дважды (что в теории невозможно при корректном `Interlocked`, но на практике защищает от багов будущего), `TryComplete()` не бросит исключение. `Complete()` бросит `ChannelClosedException` при повторном вызове.
- **Счётчик producer'ов/parser'ов обязателен при их количестве > 1.** Если каждый parser в `finally` вызовет `Complete()`, первый завершившийся закроет канал, остальные parser'ы выбросят `ChannelClosedException` на следующей `WriteAsync`, а их недообработанные элементы потеряются. Паттерн: `if (Interlocked.Decrement(ref remaining) == 0) writer.TryComplete();`.
- **`await foreach` + `ReadAllAsync` самодостаточен.** Не нужно внешне проверять «закончился ли канал» — цикл выйдет автоматически, когда канал закрыт и опустошён. Это и есть race-free завершение. Добавлять ручную проверку `reader.Completion.IsCompleted` — ошибка, она ведёт к гонке между вашей проверкой и внутренним состоянием канала.
- **`File.ReadAllTextAsync` принимает `CancellationToken`.** Если вы его не передадите, при `Ctrl+C` операция чтения не отменится — она досчитает файл до конца, и только потом токен сработает. На больших файлах это секунды задержки.
- **`StreamWriter` и `async`.** Используйте `await output.WriteLineAsync(..., ct)`. Не забудьте `await using` (а не просто `using`) для `StreamWriter` — иначе `Dispose` синхронно сбросит буфер в потоке, что для асинхронного конвейера — небольшая, но реальная шероховатость. Альтернатива — `await output.FlushAsync(ct)` перед выходом.
- **`Directory.EnumerateFiles` vs `Directory.GetFiles`.** `EnumerateFiles` ленив и хорошо сочетается с каналом: вы не загружаете весь список в память. Но помните, что перечисление само по себе может быть медленным на сетевых дисках — это нормальный I/O-bound producer.
- **Пустые файлы.** `"".Split(...)` с `StringSplitOptions.RemoveEmptyEntries` вернёт пустой массив, `Length == 0`. Убедитесь, что ваш `WordCounter` это переваривает, а не падает с `IndexOutOfRange` или не возвращает `-1`.
- **Дедлок sync-over-async.** Если вы где-то напишете `writer.WriteAsync(item, ct).Result` — на насыщенном bounded-канале это классический дедлок: поток заблокирован на `.Result`, `WriteAsync` не может завершиться, потому что consumer тоже ждёт потока. Только `await`.
- **`ConfigureAwait(false)` в консоли.** В консольном приложении `SynchronizationContext` отсутствует, поэтому `ConfigureAwait(false)` ничего не меняет. Но мы требуем его для привычки: тот же код, перенесённый в библиотеку или UI, поведёт себя правильно без правок.

#### Критерии приёмки

- [ ] Проект `FileIndexer` создаётся командой `dotnet new console` на .NET 8, компилируется без warning'ов.
- [ ] Используются два bounded-канала с `BoundedChannelFullMode.Wait` и обоснованной вместимостью.
- [ ] Есть единый `CancellationTokenSource`, токен передан во все async-операции.
- [ ] `Console.CancelKeyPress` гасит конвейер через `cts.Cancel()`.
- [ ] Producer один, parser'ов несколько (≥2), writer'ов несколько (≥2).
- [ ] `fileChannel` закрывается в `finally` producer'а через `TryComplete()`.
- [ ] `resultChannel` закрывается последним parser'ом через счётчик `Interlocked.Decrement` + `TryComplete()`.
- [ ] Нигде нет `.Result`/`.Wait()` на async-операциях канала или файла.
- [ ] Нигде нет `lock` вокруг `await`; нет `async void` consumer'ов.
- [ ] Consumer'ы используют `await foreach (... ReadAllAsync(ct))` с `ConfigureAwait(false)`.
- [ ] Точка джойна: три последовательных `await Task.WhenAll(...)`.
- [ ] `try/catch (OperationCanceledException)` вокруг точки джойна, `cts.Dispose()` в `finally`.
- [ ] `SingleReader`/`SingleWriter` выставлены осознанно для каждого канала.
- [ ] На `testdata` из 200 файлов `summary.txt` содержит ровно 200 строк.
- [ ] Эксперимент с `Ctrl+C` завершает процесс за < 1 секунды.
- [ ] Эксперимент с замедленным writer'ом показывает стабилизацию памяти на bounded-канале.
- [ ] Выводится итоговая статистика: файлы, слова, время.

#### Подсказки (без прямого ответа)

- Подумайте, в каком порядке соединять каналы: producer пишет в `fileChannel`, parser читает из `fileChannel` и пишет в `resultChannel`, writer читает из `resultChannel`. Это «цепочка» из N-1 каналов для N стадий.
- Для счётчика parser'ов заведите `int remainingParsers = ParserCount;` вне цикла создания задач — замыкание на изменяемую переменную через `ref` невозможно напрямую, используйте поле или массив из одного элемента, либо `Interlocked` на захваченной локальной (это работает, потому что локальная захвачена по ссылке в замыкании).
- `await foreach` уже пробрасывает `OperationCanceledException` — отдельный `catch` внутри consumer'а не нужен, если вы хотите, чтобы отмена поднималась к точке джойна.
- Для подсчёта слов не усложняйте: `text.Split(' ', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries).Length` достаточно. Реальный word-count учитывает пунктуацию, но для ДЗ это избыточно.
- Если `Ctrl+C` не гасит процесс — проверьте, что вы передаёте `ct` в `File.ReadAllTextAsync`. Это самая частая причина «висит в await».

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — Трёхстадийный producer/consumer конвейер с backpressure
// Three-stage producer/consumer pipeline with backpressure
using System.Threading.Channels;

// Аргумент командной строки — каталог с .txt-файлами / CLI arg = directory with .txt files.
string sourceDir = args.Length > 0 ? args[0] : ".";
if (!Directory.Exists(sourceDir))
{
    Console.Error.WriteLine($"Каталог не найден / Directory not found: {sourceDir}");
    return 1;
}

// 1) Параметры конвейера / Pipeline parameters.
const int ParserCount = 4;
const int WriterCount = 2;
const int BufferCapacity = 32;

// 2) Два bounded-канала с Wait — backpressure по умолчанию / Two bounded channels with Wait.
var fileOpts = new BoundedChannelOptions(BufferCapacity)
{
    FullMode = BoundedChannelFullMode.Wait,
    SingleWriter = true,            // producer один / single producer
    SingleReader = false,           // parser'ов несколько / multiple parsers
};
var resultOpts = new BoundedChannelOptions(BufferCapacity)
{
    FullMode = BoundedChannelFullMode.Wait,
    SingleWriter = false,           // parser'ов несколько / multiple parsers
    SingleReader = false,           // writer'ов несколько / multiple writers
};

Channel<string> fileChannel = Channel.CreateBounded<string>(fileOpts);
Channel<WordCount> resultChannel = Channel.CreateBounded<WordCount>(resultOpts);

// 3) Единая точка отмены / Single cancellation point.
using var cts = new CancellationTokenSource();
Console.CancelKeyPress += (_, e) => { e.Cancel = true; cts.Cancel(); };
CancellationToken ct = cts.Token;

var sw = System.Diagnostics.Stopwatch.StartNew();

// 4) Producer — один, перечисляет файлы / Single producer enumerating files.
var producer = Task.Run(async () =>
{
    try
    {
        foreach (string path in Directory.EnumerateFiles(sourceDir, "*.txt", SearchOption.TopDirectoryOnly))
        {
            // WriteAsync учитывает backpressure: при переполнении await приостановит без блокировки потока.
            // WriteAsync honors backpressure: when full, await suspends without blocking a thread.
            await fileChannel.Writer.WriteAsync(path, ct).ConfigureAwait(false);
        }
    }
    finally
    {
        // TryComplete идемпотентен — безопасен при любых сценариях / TryComplete is idempotent.
        fileChannel.Writer.TryComplete();
    }
}, ct);

// 5) Parsers — несколько, читают файлы и считают слова / Multiple parsers read files and count words.
int remainingParsers = ParserCount;
var parsers = Enumerable.Range(0, ParserCount).Select(_ => Task.Run(async () =>
{
    try
    {
        await foreach (string path in fileChannel.Reader.ReadAllAsync(ct).ConfigureAwait(false))
        {
            string text = await File.ReadAllTextAsync(path, ct).ConfigureAwait(false);
            int count = WordCounter.Count(text);
            await resultChannel.Writer.WriteAsync(new WordCount(path, count), ct).ConfigureAwait(false);
        }
    }
    finally
    {
        // Последний parser закрывает resultChannel / Last parser closes resultChannel.
        if (Interlocked.Decrement(ref remainingParsers) == 0)
        {
            resultChannel.Writer.TryComplete();
        }
    }
}, ct)).ToArray();

// 6) Writers — несколько, пишут в общий файл через StreamWriter / Multiple writers to a shared StreamWriter.
await using var output = new StreamWriter("summary.txt");
var writers = Enumerable.Range(0, WriterCount).Select(_ => Task.Run(async () =>
{
    await foreach (WordCount wc in resultChannel.Reader.ReadAllAsync(ct).ConfigureAwait(false))
    {
        await output.WriteLineAsync($"{wc.Path}\t{wc.Count}".AsMemory(), ct).ConfigureAwait(false);
    }
}, ct)).ToArray();

// 7) Точка джойна: producer → parsers → writers / Join: producer → parsers → writers.
long totalFiles = 0;
long totalWords = 0;
try
{
    await producer.ConfigureAwait(false);
    await Task.WhenAll(parsers).ConfigureAwait(false);
    await Task.WhenAll(writers).ConfigureAwait(false);
    await output.FlushAsync(ct).ConfigureAwait(false);

    // Подсчёт итогов из summary.txt / Tally from summary.txt.
    foreach (string line in await File.ReadAllLinesAsync("summary.txt", ct).ConfigureAwait(false))
    {
        var parts = line.Split('\t');
        if (parts.Length == 2 && long.TryParse(parts[1], out long c))
        {
            totalFiles++;
            totalWords += c;
        }
    }
    sw.Stop();
    Console.WriteLine($"Готово / Done: файлов={totalFiles}, слов={totalWords}, мс={sw.ElapsedMilliseconds}");
}
catch (OperationCanceledException)
{
    Console.WriteLine("Конвейер отменён / Pipeline cancelled.");
}
finally
{
    cts.Dispose();
}

return 0;

// ---- Вспомогательные типы / Helper types ----
readonly record struct WordCount(string Path, int Count);

static class WordCounter
{
    public static int Count(string text) =>
        text.Split(' ', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries).Length;
}
```

Разбор по строкам. Строки 1–8 — проверка аргумента и существования каталога: простая, но необходимая защита от `DirectoryNotFoundException`, которая иначе превратится в необработанное исключение в producer'е и «зависший» канал (consumer'ы будут вечно ждать в `ReadAllAsync`). Строки 10–14 — параметры конвейера; `ParserCount` и `WriterCount` выбраны > 1, чтобы паттерн счётчика был осмысленным. Строки 16–30 — создание двух bounded-каналов с `BoundedChannelFullMode.Wait`: это даёт backpressure — если writer'ы не справляются, `resultChannel` переполняется, parser'ы приостанавливаются на `WriteAsync`, `fileChannel` перестаёт опустошаться, producer приостанавливается. Цепочка замедляется целиком, без OOM. Обратите внимание на `SingleWriter`/`SingleReader`: для `fileChannel` `SingleWriter = true` (producer один) — канал применит более дешёвую безблочную реализацию; для `resultChannel` оба `false`, потому что и писателей, и читателей несколько.

Строки 32–35 — единый `CancellationTokenSource` и обработчик `Console.CancelKeyPress` с `e.Cancel = true`: мы говорим среде «не убивай процесс резко, дай мне обработать отмену». Без `e.Cancel = true` процесс просто завершится, и `finally`-блоки могут не отработать. Строки 41–53 — producer. `Directory.EnumerateFiles` ленив и не грузит список в память; каждый путь уходит в канал через `WriteAsync(item, ct)` — при переполнении await приостановит задачу без блокировки потока пула. В `finally` — `TryComplete()` (идемпотентен, безопасен).

Строки 55–72 — parsers. Здесь ключевой момент урока: несколько producer'ов во второй канал, поэтому нужен счётчик `remainingParsers` и `Interlocked.Decrement` в `finally`. Если бы каждый parser вызывал `TryComplete()` — первый завершившийся закрыл бы канал, остальные получили бы `ChannelClosedException` на `WriteAsync`, а их недообработанные элементы потерялись. `await foreach (ReadAllAsync(ct))` — race-free: цикл сам выйдет, когда `fileChannel` закрыт и пуст. `File.ReadAllTextAsync(path, ct)` — обязательно с токеном, иначе отмена большого файла не сработает.

Строки 74–82 — writers. `await using var output` корректно диспозит `StreamWriter` асинхронно. `WriteLineAsync(..., ct)` — с токеном. Строки 84–103 — точка джойна: три последовательных `await Task.WhenAll`. Порядок важен: сначала producer'ы (чтобы первый канал закрылся), затем parsers (чтобы закрылся второй канал), затем writers (чтобы дописали хвост). `try/catch (OperationCanceledException)` ловит отмену, `finally` освобождает `cts`. Применённые концепции урока: bounded + `Wait` = backpressure; счётчик `Interlocked` = корректное закрытие при нескольких писателях; `CancellationToken` во всех операциях = каскадная отмена; `await foreach` = race-free завершение; `ConfigureAwait(false)` = привычка для переносимого кода; `TryComplete()` = идемпотентность.

#### Задания на углубление (бонус)

1. **Много producer'ов.** Сделайте producer'ов несколько: каждый обрабатывает свой подкаталог. Реализуйте счётчик `remainingProducers` и `TryComplete()` для `fileChannel` по тому же паттерну, что для parsers. Проверьте, что при `ParserCount` parser'ов и `ProducerCount > 1` конвейер остаётся race-free.
2. **DropOldest vs Wait.** Сделайте `resultChannel` с `BoundedChannelFullMode.DropOldest` и замедлите writer'ов сильно. Посчитайте, сколько `WordCount` потеряно. Сравните с `Wait`. Сделайте вывод, когда `Drop*` уместен (например, телеметрия, где свежие данные важнее полноты), а когда — нет (индексация файлов, где потеря = некорректный индекс).
3. **Ручной цикл `WaitToReadAsync + TryRead`.** Перепишите одного parser'а с ручным циклом вместо `await foreach`. Объясните в комментарии, когда это даёт преимущество (например, нужно сделать что-то между чтением и обработкой, или пакетная обработка нескольких `TryRead` подряд).
4. **Обработка ошибок отдельного файла.** Если `File.ReadAllTextAsync` падает с `IOException` (файл залочен), не роняйте весь конвейер: логируйте ошибку и пропустите файл, но продолжайте. Подумайте, как при этом корректно закрыть каналы в `finally`.

---

## Statement in English / Постановка на английском

#### Context & motivation

Imagine you are building a text-file indexing service for a search engine. The input is a directory with thousands of `.txt` files of varying sizes; the output is a summary file of the form `path\tword_count`. Files live on a slow disk, so reading is an I/O-bound operation, while counting words is light but still CPU-bound work. Running everything sequentially is wasteful: the disk idles while you count, and the CPU idles while you read. Running "everything in one `Parallel.ForEach`" is also bad: `Parallel.ForEach` composes poorly with `async` I/O on the inside and gives no natural throttle between stages.

The classic answer is a multi-stage pipeline: one stage enumerates and reads files, a second counts words, a third writes results. The stages are linked by bounded buffers. If the writing stage cannot keep up, it must automatically push back on the earlier stages, so memory does not grow unbounded. This is exactly the problem `System.Threading.Channels` solves elegantly: a bounded channel gives free backpressure, `await foreach` + `ReadAllAsync` give race-free completion, and a `CancellationToken` gives a single stop point. In this assignment you will build such a pipeline end to end and verify that it behaves correctly under cancellation, under heavy load, and when a single stage fails.

#### What to do step by step

1. Create a new .NET 8 console application:
   ```
   dotnet new console -n FileIndexer -o FileIndexer --framework net8.0
   cd FileIndexer
   ```
   Make sure `FileIndexer.csproj` has `<Nullable>enable</Nullable>` and `<LangVersion>latest</LangVersion>` (or `preview`). The `System.Threading.Channels` package is already part of the .NET 8 shared framework — no extra reference is needed.

2. Prepare test data. Create a folder `testdata` and generate 200 `.txt` files of varying sizes (a short PowerShell script or a small C# program will do). Make some files large (hundreds of thousands of words) and some empty. Empty files are an important case: they check that the counter does not crash on `Split` of an empty string.

3. Implement a three-stage pipeline with two channels:
   - **Stage 1 (producer):** enumerates `.txt` files in the directory via `Directory.EnumerateFiles` and writes their paths into `fileChannel`. A single producer.
   - **Stage 2 (parsers):** several consumers (e.g. 4) reading paths from `fileChannel`, asynchronously reading file contents via `File.ReadAllTextAsync`, counting words, and writing `WordCount(Path, Count)` into `resultChannel`.
   - **Stage 3 (writers):** several consumers (e.g. 2) reading `WordCount` from `resultChannel` and writing `path\tcount` lines to a shared summary file `summary.txt` (or to separate `summary-0.txt` / `summary-1.txt` files — your choice).

4. Use bounded channels with a deliberate capacity (e.g. 32) and `BoundedChannelFullMode.Wait`. Explain in a comment why `Wait` was chosen over `DropWrite`.

5. Set up a single `CancellationTokenSource`. Pass the token into every `WriteAsync` / `ReadAllAsync` / `File.ReadAllTextAsync`. Add a `Console.CancelKeyPress` handler that calls `cts.Cancel()` so that `Ctrl+C` cleanly stops the pipeline instead of tearing it mid-flight.

6. Implement correct channel closing: the first channel is closed by the producer in a `finally` block (it is alone, so no counter is needed, but call `TryComplete()` because it is idempotent). The second channel is closed by the last parser through an `Interlocked.Decrement` counter — because there are several parsers, and an early `Complete()` would truncate the rest.

7. Join point: `await Task.WhenAll(producers)`, then `await Task.WhenAll(parsers)`, then `await Task.WhenAll(writers)`. Wrap everything in `try/catch (OperationCanceledException)` and a `finally` that disposes `cts`.

8. Run the pipeline:
   ```
   dotnet run --project FileIndexer -- ./testdata
   ```
   Expected output: log lines about each consumer finishing and a final "Pipeline completed cleanly". The summary file `summary.txt` should contain one line per input file. Verify that the number of lines in `summary.txt` equals the number of `.txt` files in `testdata`.

9. Run a cancellation experiment: start the pipeline on a large directory, and after a couple of seconds press `Ctrl+C`. Verify that the process exits within a reasonable time (a fraction of a second) rather than hanging in `await`. If it hangs, you forgot to pass `ct` into some operation.

10. Run a backpressure experiment: temporarily slow down the writers (e.g. `await Task.Delay(50)` per line). Run on a large directory and monitor the process memory (via `dotnet-counters monitor` or Task Manager). With a bounded channel memory should stabilize; with an unbounded channel it should grow toward OOM. Record the observation in a comment.

#### Requirements

- The code must compile and run on .NET 8 with C# 12 (top-level statements, `record struct`, `Random.Shared`, collection expressions where appropriate).
- Exactly two channels are used: `Channel<string>` for paths and `Channel<WordCount>` for results. Both are bounded with `BoundedChannelFullMode.Wait`.
- A `CancellationToken` threads through every async operation: `WriteAsync`, `ReadAllAsync`, `File.ReadAllTextAsync`, `Task.Delay`, `StreamWriter` writes. There is no bare `await` without a token, except where a token is technically inapplicable.
- Nowhere is there a `.Result` or `.Wait()` on an async channel operation. Nowhere is there a `lock` around an `await`. Nowhere is there an `async void` consumer.
- `writer.TryComplete()` is called exactly once per channel: for the first channel in the producer's `finally`; for the second channel through an `Interlocked.Decrement` counter in the last parser's `finally`.
- Consumers use `await foreach (... reader.ReadAllAsync(ct))`. An alternative `WaitToReadAsync` + `TryRead` loop is acceptable with a justifying comment.
- The join point is three consecutive `await Task.WhenAll(...)` calls, wrapped in `try/catch (OperationCanceledException)/finally`.
- `ConfigureAwait(false)` is applied to all async operations in library style (not strictly required in a console app, but we are training the habit for portable code).
- `SingleReader` / `SingleWriter` are set deliberately: for `fileChannel` `SingleWriter = true` (one producer), `SingleReader = false` (several parsers); for `resultChannel` `SingleWriter = false`, `SingleReader = false` — or `SingleReader = true` if there is a single writer.
- The program prints a final statistic: files processed, total words, elapsed milliseconds.

#### Pitfalls

- **Prefer `TryComplete()` over `Complete()`.** `TryComplete()` is idempotent — repeated calls are ignored. This matters when several parsers in a `finally` race the counter down to zero: even if a future bug made the condition fire twice (impossible in theory with correct `Interlocked`, but a real safety net in practice), `TryComplete()` will not throw. `Complete()` throws `ChannelClosedException` on the second call.
- **The producer/parser counter is mandatory when there is more than one.** If every parser calls `Complete()` in its `finally`, the first one to finish closes the channel; the others throw `ChannelClosedException` on the next `WriteAsync`, and their unprocessed items are lost. The pattern is: `if (Interlocked.Decrement(ref remaining) == 0) writer.TryComplete();`.
- **`await foreach` + `ReadAllAsync` is self-sufficient.** You do not need to externally check "is the channel done" — the loop exits automatically when the channel is completed and drained. That is the race-free termination. Adding a manual `reader.Completion.IsCompleted` check is a mistake; it races against the channel's internal state.
- **`File.ReadAllTextAsync` takes a `CancellationToken`.** If you do not pass it, `Ctrl+C` will not cancel the read — it will finish the file, and only then will the token fire. On large files that is seconds of delay.
- **`StreamWriter` and `async`.** Use `await output.WriteLineAsync(..., ct)`. Do not forget `await using` (not plain `using`) for the `StreamWriter` — otherwise `Dispose` synchronously flushes the buffer on the thread, which is a small but real rough edge in an async pipeline. An alternative is `await output.FlushAsync(ct)` before exiting.
- **`Directory.EnumerateFiles` vs `Directory.GetFiles`.** `EnumerateFiles` is lazy and pairs well with a channel: you do not load the whole list into memory. But remember that the enumeration itself can be slow on network drives — that is a normal I/O-bound producer.
- **Empty files.** `"".Split(...)` with `StringSplitOptions.RemoveEmptyEntries` returns an empty array, `Length == 0`. Make sure your `WordCounter` handles this instead of crashing with `IndexOutOfRange` or returning `-1`.
- **The sync-over-async deadlock.** If you write `writer.WriteAsync(item, ct).Result` anywhere, on a saturated bounded channel you get a classic deadlock: the thread is blocked on `.Result`, `WriteAsync` cannot complete because the consumer is also waiting for a thread. Only `await`.
- **`ConfigureAwait(false)` in a console app.** A console app has no `SynchronizationContext`, so `ConfigureAwait(false)` changes nothing. We still require it for the habit: the same code, moved into a library or a UI, will behave correctly without edits.

#### Acceptance criteria

- [ ] The `FileIndexer` project is created with `dotnet new console` on .NET 8 and compiles without warnings.
- [ ] Two bounded channels are used with `BoundedChannelFullMode.Wait` and a justified capacity.
- [ ] There is a single `CancellationTokenSource`; the token is passed into every async operation.
- [ ] `Console.CancelKeyPress` stops the pipeline via `cts.Cancel()`.
- [ ] There is one producer, several parsers (≥2), several writers (≥2).
- [ ] `fileChannel` is closed in the producer's `finally` via `TryComplete()`.
- [ ] `resultChannel` is closed by the last parser via an `Interlocked.Decrement` counter + `TryComplete()`.
- [ ] Nowhere is there a `.Result` / `.Wait()` on a channel or file async operation.
- [ ] Nowhere is there a `lock` around `await`; no `async void` consumers.
- [ ] Consumers use `await foreach (... ReadAllAsync(ct))` with `ConfigureAwait(false)`.
- [ ] Join point: three consecutive `await Task.WhenAll(...)`.
- [ ] `try/catch (OperationCanceledException)` around the join point, `cts.Dispose()` in `finally`.
- [ ] `SingleReader` / `SingleWriter` are set deliberately per channel.
- [ ] On a `testdata` of 200 files, `summary.txt` contains exactly 200 lines.
- [ ] The `Ctrl+C` experiment exits the process in < 1 second.
- [ ] The slow-writer experiment shows memory stabilization on a bounded channel.
- [ ] A final statistic is printed: files, words, elapsed time.

#### Hints (no direct answer)

- Think about the order in which the channels connect: the producer writes to `fileChannel`, a parser reads from `fileChannel` and writes to `resultChannel`, a writer reads from `resultChannel`. This is a "chain" of N-1 channels for N stages.
- For the parser counter, declare `int remainingParsers = ParserCount;` outside the task-creation loop. A mutable local cannot be captured by `ref` directly; use a field, a single-element array, or `Interlocked` on the captured local (this works because the local is captured by reference into the closure).
- `await foreach` already propagates `OperationCanceledException` — a separate `catch` inside the consumer is not needed if you want cancellation to surface at the join point.
- For word counting, do not overcomplicate: `text.Split(' ', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries).Length` is enough. A real word counter accounts for punctuation, but that is overkill for this homework.
- If `Ctrl+C` does not stop the process, check that you pass `ct` into `File.ReadAllTextAsync`. That is the most common cause of "hangs in await".

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — Three-stage producer/consumer pipeline with backpressure
using System.Threading.Channels;

// CLI arg = directory with .txt files.
string sourceDir = args.Length > 0 ? args[0] : ".";
if (!Directory.Exists(sourceDir))
{
    Console.Error.WriteLine($"Directory not found: {sourceDir}");
    return 1;
}

// 1) Pipeline parameters.
const int ParserCount = 4;
const int WriterCount = 2;
const int BufferCapacity = 32;

// 2) Two bounded channels with Wait — backpressure by default.
var fileOpts = new BoundedChannelOptions(BufferCapacity)
{
    FullMode = BoundedChannelFullMode.Wait,
    SingleWriter = true,            // single producer
    SingleReader = false,           // multiple parsers
};
var resultOpts = new BoundedChannelOptions(BufferCapacity)
{
    FullMode = BoundedChannelFullMode.Wait,
    SingleWriter = false,           // multiple parsers write here
    SingleReader = false,           // multiple writers read here
};

Channel<string> fileChannel = Channel.CreateBounded<string>(fileOpts);
Channel<WordCount> resultChannel = Channel.CreateBounded<WordCount>(resultOpts);

// 3) Single cancellation point.
using var cts = new CancellationTokenSource();
Console.CancelKeyPress += (_, e) => { e.Cancel = true; cts.Cancel(); };
CancellationToken ct = cts.Token;

var sw = System.Diagnostics.Stopwatch.StartNew();

// 4) Producer — single, enumerates files.
var producer = Task.Run(async () =>
{
    try
    {
        foreach (string path in Directory.EnumerateFiles(sourceDir, "*.txt", SearchOption.TopDirectoryOnly))
        {
            // WriteAsync honors backpressure: when full, await suspends without blocking a thread.
            await fileChannel.Writer.WriteAsync(path, ct).ConfigureAwait(false);
        }
    }
    finally
    {
        // TryComplete is idempotent — safe in any scenario.
        fileChannel.Writer.TryComplete();
    }
}, ct);

// 5) Parsers — multiple, read files and count words.
int remainingParsers = ParserCount;
var parsers = Enumerable.Range(0, ParserCount).Select(_ => Task.Run(async () =>
{
    try
    {
        await foreach (string path in fileChannel.Reader.ReadAllAsync(ct).ConfigureAwait(false))
        {
            string text = await File.ReadAllTextAsync(path, ct).ConfigureAwait(false);
            int count = WordCounter.Count(text);
            await resultChannel.Writer.WriteAsync(new WordCount(path, count), ct).ConfigureAwait(false);
        }
    }
    finally
    {
        // Last parser closes resultChannel.
        if (Interlocked.Decrement(ref remainingParsers) == 0)
        {
            resultChannel.Writer.TryComplete();
        }
    }
}, ct)).ToArray();

// 6) Writers — multiple, write to a shared StreamWriter.
await using var output = new StreamWriter("summary.txt");
var writers = Enumerable.Range(0, WriterCount).Select(_ => Task.Run(async () =>
{
    await foreach (WordCount wc in resultChannel.Reader.ReadAllAsync(ct).ConfigureAwait(false))
    {
        await output.WriteLineAsync($"{wc.Path}\t{wc.Count}".AsMemory(), ct).ConfigureAwait(false);
    }
}, ct)).ToArray();

// 7) Join point: producer → parsers → writers.
long totalFiles = 0;
long totalWords = 0;
try
{
    await producer.ConfigureAwait(false);
    await Task.WhenAll(parsers).ConfigureAwait(false);
    await Task.WhenAll(writers).ConfigureAwait(false);
    await output.FlushAsync(ct).ConfigureAwait(false);

    // Tally from summary.txt.
    foreach (string line in await File.ReadAllLinesAsync("summary.txt", ct).ConfigureAwait(false))
    {
        var parts = line.Split('\t');
        if (parts.Length == 2 && long.TryParse(parts[1], out long c))
        {
            totalFiles++;
            totalWords += c;
        }
    }
    sw.Stop();
    Console.WriteLine($"Done: files={totalFiles}, words={totalWords}, ms={sw.ElapsedMilliseconds}");
}
catch (OperationCanceledException)
{
    Console.WriteLine("Pipeline cancelled.");
}
finally
{
    cts.Dispose();
}

return 0;

// ---- Helper types ----
readonly record struct WordCount(string Path, int Count);

static class WordCounter
{
    public static int Count(string text) =>
        text.Split(' ', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries).Length;
}
```

Walk-through, line by line. Lines 1–8 validate the argument and the directory's existence: a simple but necessary guard against `DirectoryNotFoundException`, which would otherwise surface as an unhandled exception inside the producer and leave a "hanging" channel (consumers would wait forever in `ReadAllAsync`). Lines 10–14 are the pipeline parameters; `ParserCount` and `WriterCount` are > 1 so that the counter pattern is meaningful. Lines 16–30 create two bounded channels with `BoundedChannelFullMode.Wait`: this gives backpressure — if the writers cannot keep up, `resultChannel` fills, the parsers suspend on `WriteAsync`, `fileChannel` stops draining, and the producer suspends. The whole chain slows down together, with no OOM. Note `SingleWriter` / `SingleReader`: for `fileChannel` `SingleWriter = true` (single producer) — the channel applies a cheaper lock-free path; for `resultChannel` both are `false`, because there are several writers and several readers.

Lines 32–35 set up a single `CancellationTokenSource` and a `Console.CancelKeyPress` handler with `e.Cancel = true`: we tell the runtime "do not kill the process abruptly; let me handle cancellation". Without `e.Cancel = true` the process would simply die and the `finally` blocks might not run. Lines 41–53 are the producer. `Directory.EnumerateFiles` is lazy and does not load the list into memory; each path goes into the channel via `WriteAsync(item, ct)` — on overflow the await suspends the task without blocking a pool thread. The `finally` calls `TryComplete()` (idempotent, safe).

Lines 55–72 are the parsers. This is the lesson's key point: several producers into the second channel, so a counter `remainingParsers` and `Interlocked.Decrement` in `finally` are required. If every parser called `TryComplete()`, the first to finish would close the channel, the others would get `ChannelClosedException` on `WriteAsync`, and their unprocessed items would be lost. `await foreach (ReadAllAsync(ct))` is race-free: the loop exits by itself when `fileChannel` is completed and empty. `File.ReadAllTextAsync(path, ct)` must take the token, otherwise cancelling a large file read would not take effect.

Lines 74–82 are the writers. `await using var output` disposes the `StreamWriter` asynchronously. `WriteLineAsync(..., ct)` takes the token. Lines 84–103 are the join point: three consecutive `await Task.WhenAll`. The order matters: producers first (so the first channel closes), then parsers (so the second channel closes), then writers (so the tail is flushed). `try/catch (OperationCanceledException)` catches cancellation; `finally` disposes `cts`. Lesson concepts applied: bounded + `Wait` = backpressure; the `Interlocked` counter = correct closing with multiple writers; `CancellationToken` everywhere = cascading cancellation; `await foreach` = race-free termination; `ConfigureAwait(false)` = habit for portable code; `TryComplete()` = idempotence.

#### Going deeper (bonus)

1. **Multiple producers.** Make the producers several: each handles its own subdirectory. Implement a `remainingProducers` counter and `TryComplete()` for `fileChannel` using the same pattern as for the parsers. Verify that with `ParserCount` parsers and `ProducerCount > 1` the pipeline stays race-free.
2. **DropOldest vs Wait.** Make `resultChannel` use `BoundedChannelFullMode.DropOldest` and slow the writers down a lot. Count how many `WordCount` records are dropped. Compare with `Wait`. Conclude when `Drop*` is appropriate (e.g. telemetry, where fresh data matters more than completeness) and when it is not (file indexing, where a drop means an incorrect index).
3. **Manual `WaitToReadAsync + TryRead` loop.** Rewrite one parser with a manual loop instead of `await foreach`. Explain in a comment when this helps (e.g. you need to do something between reading and processing, or you want to batch several `TryRead` calls together).
4. **Per-file error handling.** If `File.ReadAllTextAsync` throws `IOException` (file locked), do not crash the whole pipeline: log the error and skip the file, but keep going. Think about how to still close the channels correctly in `finally`.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `FileIndexer` на .NET 8, компилируется без warning'ов.
- [ ] Два bounded-канала с `BoundedChannelFullMode.Wait`.
- [ ] Единый `CancellationToken` во всех async-операциях.
- [ ] `Console.CancelKeyPress` → `cts.Cancel()`.
- [ ] Один producer, ≥2 parser'а, ≥2 writer'а.
- [ ] `fileChannel` закрыт в `finally` producer'а через `TryComplete()`.
- [ ] `resultChannel` закрыт последним parser'ом через `Interlocked.Decrement`.
- [ ] Нет `.Result`/`.Wait()`; нет `lock` вокруг `await`; нет `async void`.
- [ ] `await foreach` + `ConfigureAwait(false)` в consumer'ах.
- [ ] Точка джойна из трёх `Task.WhenAll` + `try/catch (OperationCanceledException)`.
- [ ] `SingleReader`/`SingleWriter` выставлены осознанно.
- [ ] `summary.txt` содержит по строке на каждый входной файл.
- [ ] `Ctrl+C` гасит процесс за < 1 секунды.
- [ ] Bounded-канал стабилизирует память при медленном writer'е.
- [ ] Выводится итоговая статистика.

- [ ] `FileIndexer` project on .NET 8, compiles without warnings.
- [ ] Two bounded channels with `BoundedChannelFullMode.Wait`.
- [ ] A single `CancellationToken` in every async operation.
- [ ] `Console.CancelKeyPress` → `cts.Cancel()`.
- [ ] One producer, ≥2 parsers, ≥2 writers.
- [ ] `fileChannel` closed in the producer's `finally` via `TryComplete()`.
- [ ] `resultChannel` closed by the last parser via `Interlocked.Decrement`.
- [ ] No `.Result` / `.Wait()`; no `lock` around `await`; no `async void`.
- [ ] `await foreach` + `ConfigureAwait(false)` in the consumers.
- [ ] Join point of three `Task.WhenAll` + `try/catch (OperationCanceledException)`.
- [ ] `SingleReader` / `SingleWriter` set deliberately.
- [ ] `summary.txt` has one line per input file.
- [ ] `Ctrl+C` stops the process in < 1 second.
- [ ] A bounded channel stabilizes memory with a slow writer.
- [ ] A final statistic is printed.

#### Ресурсы / Resources
- [Microsoft Learn — Channel](https://learn.microsoft.com/dotnet/api/system.threading.channels.channel)
- [Microsoft Learn — BoundedChannelOptions](https://learn.microsoft.com/dotnet/api/system.threading.channels.boundedchanneloptions)
- [Microsoft Learn — ChannelReader<T>.ReadAllAsync](https://learn.microsoft.com/dotnet/api/system.threading.channels.channelreader-1.readallasync)
- [System.Threading.Channels GitHub samples](https://github.com/dotnet/runtime/tree/main/src/libraries/System.Threading.Channels)
- [Stephen Toub — An Introduction to System.Threading.Channels](https://devblogs.microsoft.com/dotnet/an-introduction-to-system-threading-channels/)

---

[← К уроку M11-L07](lesson-M11-L07-channels.md) | [⬆ К модулю M11](../README.md) | [Следующее ДЗ →](homework-M11-L08-parallel-for.md)
