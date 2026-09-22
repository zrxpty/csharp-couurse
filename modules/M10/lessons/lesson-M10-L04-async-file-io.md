[⬆ К модулю M10](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M10-L04: Async файловые операции / Async file operations

**Модуль / Module:** M10
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Асинхронные файловые операции — это способ работать с диском, не блокируя поток выполнения. В отличие от синхронного `File.ReadAllText`, который «замораживает» текущий поток до завершения чтения, асинхронные методы возвращают управление вызывающему коду, пока операционная система дочитывает данные с диска.

**Аналогия:** Представьте ресторан. Синхронный официант стоит у плиты и ждёт, пока повар приготовит блюдо, ничего больше не делая. Асинхронный официант передаёт заказ и уходит обслуживать другие столики, возвращаясь, когда блюдо готово. Поток — это официант; блюдо — файловая операция; плита — диск, который в тысячи раз медленнее памяти.

В .NET 8 / C# 12 ключевые методы — `File.ReadAllTextAsync`, `File.WriteAllTextAsync`, `File.ReadAllLinesAsync`, `File.AppendAllTextAsync`, а также `StreamReader.ReadLineAsync` для потокового чтения. Все они возвращают `Task` или `Task<T>`, и все принимают `CancellationToken`.

**ConfigureAwait(false)** — важный нюанс в библиотечном коде. По умолчанию после `await` продолжение выполняется в исходном контексте синхронизации (например, в UI-потоке). В консольных и ASP.NET Core приложениях контекста нет, и различие незаметно. Но в библиотеках, WPF/WinForms или старом ASP.NET вызов `ConfigureAwait(false)` говорит рантайму: «не пытайся вернуться в исходный контекст, продолжай где удобнее». Это снижает накладные расходы и предотвращает дедлоки. Правило простое: в библиотечном коде — всегда `ConfigureAwait(false)`, в коде верхнего уровня (app, Program.cs) — можно опустить.

**Throughput (пропускная способность).** Асинхронность особенно полезна, когда вы обрабатываете много файлов параллельно. Вместо того чтобы читать файлы последовательно (10 файлов × 100 мс = 1 секунда), вы запускаете чтение всех файлов через `Task.WhenAll` и получаете общее время, близкое к самому медленному файлу. Но будьте осторожны: диск имеет ограниченную пропускную способность, и сотни одновременных операций могут вызвать насыщение очереди ввода-вывода. Используйте `SemaphoreSlim` для ограничения степени параллелизма (например, 8–16 одновременных операций на SSD).

**Отменяемый I/O через CancellationTokenSource (CTS).** Длительные операции чтения больших файлов или копирования каталогов должны быть отменяемыми. Создаёте `var cts = new CancellationTokenSource();`, передаёте `cts.Token` в асинхронный метод, а при необходимости вызываете `cts.Cancel()`. Метод выбросит `OperationCanceledException`, который вы перехватываете и корректно обрабатываете. Для таймаута используйте `new CancellationTokenSource(TimeSpan.FromSeconds(5))` — это отменит операцию через 5 секунд автоматически. Комбинировать несколько токенов можно через `CancellationTokenSource.CreateLinkedTokenSource`.

**Потоковое чтение через `StreamReader`.** Для огромных файлов (гигабайты логов) `ReadAllTextAsync` непригоден — он загружает весь файл в память. Вместо этого откройте `StreamReader` в блоке `await using` и читайте построчно через `await reader.ReadLineAsync()`. Это обрабатывает файлы любого размера с постоянным потреблением памяти. Не забудьте `StreamReader` и базовый `FileStream` создаются с `FileOptions.Asynchronous` — иначе «асинхронный» метод под капотом блокирует поток пулом.

#### Theory (EN)

Asynchronous file operations let you work with disk without blocking the executing thread. Unlike synchronous `File.ReadAllText`, which freezes the calling thread until reading finishes, async methods return control to the caller while the operating system reads bytes from disk.

**Analogy:** Think of a restaurant. A synchronous waiter stands by the stove, waiting for the chef to cook a dish, doing nothing else. An asynchronous waiter hands in the order and walks away to serve other tables, returning when the dish is ready. The thread is the waiter; the dish is the file operation; the stove is the disk, which is thousands of times slower than RAM.

In .NET 8 / C# 12 the key methods are `File.ReadAllTextAsync`, `File.WriteAllTextAsync`, `File.ReadAllLinesAsync`, `File.AppendAllTextAsync`, and `StreamReader.ReadLineAsync` for streaming reads. They all return `Task` or `Task<T>` and accept a `CancellationToken`.

**ConfigureAwait(false)** is a crucial detail in library code. By default, after `await` the continuation runs on the original synchronization context (for instance, the UI thread). In console and ASP.NET Core applications there is no such context, so the difference is invisible. But in libraries, WPF/WinForms, or classic ASP.NET, calling `ConfigureAwait(false)` tells the runtime: "don't try to return to the original context, continue wherever is convenient." This reduces overhead and prevents deadlocks. The rule is simple: in library code always use `ConfigureAwait(false)`; in top-level application code (Program.cs, an app entry point) you may omit it.

**Throughput.** Asynchrony shines when you process many files. Instead of reading sequentially (10 files × 100 ms = 1 second), you launch all reads through `Task.WhenAll` and get a total time close to the slowest single file. Be careful, though: a disk has limited bandwidth, and hundreds of concurrent operations can saturate the I/O queue. Use `SemaphoreSlim` to cap the degree of parallelism (say, 8–16 concurrent ops on an SSD).

**Cancellable I/O with CancellationTokenSource (CTS).** Long-running operations — reading large files or copying directories — must be cancellable. Create `var cts = new CancellationTokenSource();`, pass `cts.Token` into the async method, and call `cts.Cancel()` when needed. The method throws `OperationCanceledException`, which you catch and handle gracefully. For a timeout, use `new CancellationTokenSource(TimeSpan.FromSeconds(5))` — the operation auto-cancels after 5 seconds. To combine several tokens, use `CancellationTokenSource.CreateLinkedTokenSource`.

**Streaming with `StreamReader`.** For huge files (gigabytes of logs) `ReadAllTextAsync` is unsuitable — it loads the whole file into memory. Instead, open a `StreamReader` inside an `await using` block and read line by line with `await reader.ReadLineAsync()`. This handles files of any size with constant memory consumption. Remember to construct the `StreamReader` and the underlying `FileStream` with `FileOptions.Asynchronous` — otherwise the "async" method secretly blocks a thread-pool thread under the hood.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Async file operations
// Асинхронные файловые операции

using System.IO;
using System.Text;

public sealed class AsyncFileDemo
{
    private const int MaxConcurrency = 8; // ограничение параллелизма / parallelism cap

    // Запись текста с отменой и таймаутом / Write text with cancellation and timeout
    public static async Task WriteWithTimeoutAsync(
        string path, string content, CancellationToken externalToken)
    {
        // Связываем внешний токен с таймаутом 5 сек / Link external token with 5s timeout
        using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            externalToken, timeoutCts.Token);

        try
        {
            // ConfigureAwait(false) в библиотечном коде / ConfigureAwait(false) in library code
            await File.WriteAllTextAsync(path, content, linkedCts.Token)
                .ConfigureAwait(false);
        }
        catch (OperationCanceledException) when (timeoutCts.IsCancellationRequested)
        {
            throw new TimeoutException($"Write to '{path}' exceeded 5 seconds.");
        }
    }

    // Параллельное чтение многих файлов с ограничением / Parallel read of many files, throttled
    public static async Task<IReadOnlyDictionary<string, string>> ReadManyAsync(
        IEnumerable<string> paths, CancellationToken token)
    {
        using var gate = new SemaphoreSlim(MaxConcurrency);
        var tasks = paths.Select(async path =>
        {
            await gate.WaitAsync(token).ConfigureAwait(false);
            try
            {
                var text = await File.ReadAllTextAsync(path, token).ConfigureAwait(false);
                return (path, text);
            }
            finally
            {
                gate.Release();
            }
        }).ToList();

        var results = await Task.WhenAll(tasks).ConfigureAwait(false);
        return results.ToDictionary(r => r.path, r => r.text);
    }

    // Потоковое чтение большого файла построчно / Stream a huge file line by line
    public static async IAsyncEnumerable<string> ReadLinesAsync(
        string path, [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken token)
    {
        // FileOptions.Asynchronous — настоящий async, без блокировки потока / true async, no thread blocking
        await using var fs = new FileStream(
            path, FileMode.Open, FileAccess.Read, FileShare.Read,
            bufferSize: 4096, options: FileOptions.Asynchronous);
        using var reader = new StreamReader(fs, Encoding.UTF8);

        string? line;
        while ((line = await reader.ReadLineAsync(token).ConfigureAwait(false)) is not null)
        {
            token.ThrowIfCancellationRequested();
            yield return line;
        }
    }
}
```

#### Best Practices
- В библиотечном коде всегда используйте `ConfigureAwait(false)` после каждого `await`. / In library code, always append `ConfigureAwait(false)` after every `await`.
- Передавайте `CancellationToken` во все асинхронные I/O-методы, даже если сейчас не планируете отмену. / Pass a `CancellationToken` into every async I/O method even if you don't plan to cancel now.
- Для больших файлов используйте потоковое чтение через `StreamReader.ReadLineAsync`, а не `ReadAllTextAsync`. / For large files, stream with `StreamReader.ReadLineAsync` instead of `ReadAllTextAsync`.
- Ограничивайте параллелизм дисковых операций через `SemaphoreSlim` (8–16 на SSD). / Cap disk parallelism with `SemaphoreSlim` (8–16 on SSD).
- Открывайте `FileStream` с `FileOptions.Asynchronous`, чтобы избежать скрытой блокировки потока. / Open `FileStream` with `FileOptions.Asynchronous` to avoid hidden thread blocking.

#### Частые ошибки / Common Mistakes
- Вызов `Result` или `.Wait()` на `Task` от файловой операции → дедлок в UI/ASP.NET. Используйте `await`. / Calling `.Result` or `.Wait()` on a file Task → deadlock in UI/ASP.NET. Use `await`.
- Чтение гигабайтного файла через `ReadAllTextAsync` → `OutOfMemoryException`. Переходите на `StreamReader`. / Reading a gigabyte file with `ReadAllTextAsync` → `OutOfMemoryException`. Switch to `StreamReader`.
- `FileStream` без `FileOptions.Asynchronous` → «асинхронный» метод блокирует поток пула. Всегда указывайте флаг. / `FileStream` without `FileOptions.Asynchronous` → the "async" method blocks a pool thread. Always set the flag.
- Игнорирование `CancellationToken` в сигнатуре → нельзя отменить долгое копирование. Принимайте и пробрасывайте токен. / Ignoring `CancellationToken` in the signature → can't cancel a long copy. Accept and forward the token.
- `Task.Run(() => File.ReadAllText(path))` — это не async I/O, а занятый поток. Вызывайте `ReadAllTextAsync` напрямую. / `Task.Run(() => File.ReadAllText(path))` is not async I/O, it's a busy thread. Call `ReadAllTextAsync` directly.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Все файловые операции помечены `async` и возвращают `Task`/`Task<T>`. / All file operations are `async` and return `Task`/`Task<T>`.
- [ ] В библиотечном коде после каждого `await` стоит `ConfigureAwait(false)`. / Library code has `ConfigureAwait(false)` after every `await`.
- [ ] Каждый асинхронный метод принимает `CancellationToken` и передаёт его дальше. / Every async method accepts a `CancellationToken` and forwards it.
- [ ] Большие файлы читаются построчно через `StreamReader`, а не целиком. / Large files are read line by line via `StreamReader`, not in full.
- [ ] `FileStream` создаётся с `FileOptions.Asynchronous`. / `FileStream` is created with `FileOptions.Asynchronous`.
- [ ] Параллельные операции ограничены `SemaphoreSlim`, чтобы не насытить диск. / Parallel ops are throttled with `SemaphoreSlim` to avoid saturating the disk.
- [ ] Таймауты реализованы через `CancellationTokenSource(TimeSpan)`. / Timeouts use `CancellationTokenSource(TimeSpan)`.

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/dotnet/standard/io/asynchronous-file-i-o](https://learn.microsoft.com/dotnet/standard/io/asynchronous-file-i-o)

---

[⬆ К модулю M10](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
