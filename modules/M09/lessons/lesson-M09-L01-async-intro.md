[⬆ К модулю M09](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M09-L01: Зачем async, потоки vs async I/O / Why async, threads vs async I/O

**Модуль / Module:** M09
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Начнём с простой идеи: поток (thread) — это дорогой ресурс. В .NET каждый поток занимает около 1 МБ стека и требует накладных расходов планировщика ОС. Когда синхронный код выполняет ввод-вывод — читает файл, делает HTTP-запрос, обращается к БД, — поток **блокируется**: он буквально стоит и ждёт ответа от устройства или сети, не выполняя никакой полезной работы. Если в веб-сервере на каждый входящий запрос выделить отдельный поток, который будет ждать ответа из БД по 50 мс, то при сотне одновременных запросов мы сожжём сотню потоков только на ожидание. Это и есть главное ограничение потоково-ориентированной модели: **масштабируемость упирается в количество потоков в пуле**, а пул ограничен (по умолчанию ≈ CPU-зависимое число, часто 8–32 на ядро).

Асинхронный I/O решает проблему иначе. Вместо того чтобы поток «торчал» в ожидании, операция регистрируется в механизме уведомлений ОС — на Windows это **IOCP** (I/O Completion Ports), на Linux — `epoll`, на macOS — `kqueue`. Поток отправляет запрос «дочитай файл и позови меня, когда будет готово» и тут же возвращается в пул, чтобы обслужить другой запрос. Когда данные приходят, ядро будит поток из пула (часто — другой), который продолжает выполнение с того места, где была `await`-точка. **Один и тот же поток может обслуживать тысячи конкурентных I/O-операций**, потому что ни одна из них его не держит.

Аналогия с рестораном. Синхронный официант принимает заказ у столика, идёт на кухню, **стоит у плиты и ждёт**, пока блюдо приготовится, относит его обратно и только потом идёт к следующему столику. Так один официант обслужит максимум два-три столика. Асинхронный официант принимает заказ, передаёт поварам билет, **сразу идёт к следующему столику**, а когда кухня позвонит «блюдо готово» — подходит и забирает его. Один официант теперь обслуживает десятки столиков: он не простаивает, пока готовят. В .NET `async/await` — это и есть «билет на кухню»: метод освобождает поток на время ожидания и забирает его обратно из пула, когда I/O завершится.

Важно различать **асинхронность и многопоточность**. `async` не создаёт потоки автоматически — он лишь говорит «тут будет ожидание, поток можно отдать». I/O-bound операции (сеть, диск) через `async` реально освобождают поток. CPU-bound операции (вычисления) нужно отправлять в фон через `Task.Run`, иначе вы просто заблокируете текущий поток, но уже «асинхронно». Поэтому `Task.Run(() => Thread.Sleep(1000))` не даёт никакой масштабируемости — он просто перекладывает блокировку на другой поток пула.

Контекст синхронизации (`SynchronizationContext`) определяет, **куда** вернётся продолжение после `await`. В UI-приложениях (WPF/WinForms) это очередь сообщений UI-потока, поэтому continuation гарантированно выполнится в UI-потоке — удобно, но опасно при смешивании с `.Result`. В ASP.NET (до Core) контекст мог захватывать request-контекст, что приводило к дедлокам при блокировке синхронного кода поверх асинхронного. В ASP.NET Core и консольных приложениях `SynchronizationContext` равен `null`, и continuation выполняется в произвольном потоке пула.

Классические ловушки, которые мы разберём в коде ниже: `.Result`/`.Wait()` на асинхронной операции в контексте с захватом → **дедлок**; `async void` → исключения невозможно обработать, а подписка «выстреливает и забывает»; `lock` поверх `await` — `lock` не знает про `await`, и мьютекс может быть захвачен одним потоком, а освобождён другим; блокировка потоков пула через `Task.Run(() => Thread.Sleep(...))` → истощение пула под нагрузкой. Правильные инструменты: `CancellationToken` для кооперативной отмены, `ConfigureAwait(false)` в библиотечном коде, `SemaphoreSlim` вместо `lock` для асинхронных секций, `Channel<T>` для producer/consumer pipelines (подробно — в M11-L07).

#### Theory (EN)

Start with a basic fact: a thread is an expensive resource. In .NET every thread consumes roughly 1 MB of stack plus OS scheduler overhead. When synchronous code performs I/O — reading a file, making an HTTP call, querying a database — the thread **blocks**: it literally sits idle waiting for the device or the network to respond, doing no useful work. If a web server dedicates a thread per incoming request and that thread waits 50 ms on a database call, then a hundred concurrent requests burn a hundred threads purely on waiting. This is the core limitation of the thread-per-request model: **scalability is bounded by the thread pool size**, and the pool is finite (by default ≈ CPU-bound, commonly 8–32 per core).

Asynchronous I/O attacks the problem differently. Instead of holding a thread during the wait, the operation is registered with the OS notification mechanism — **IOCP** (I/O Completion Ports) on Windows, `epoll` on Linux, `kqueue` on macOS. The thread issues a request of the form «read this file and wake me when done» and immediately returns to the pool to serve another request. When the data arrives, the kernel wakes a pool thread (often a different one) that resumes execution at the `await` point. **A single thread can service thousands of concurrent I/O operations** because none of them holds it.

The restaurant analogy. A synchronous waiter takes an order at a table, walks to the kitchen, **stands at the stove and waits** until the dish is cooked, carries it back, and only then proceeds to the next table. One such waiter serves two or three tables tops. An asynchronous waiter takes the order, hands the ticket to the cooks, **immediately moves to the next table**, and when the kitchen rings «dish ready» — comes back and picks it up. Now one waiter serves dozens of tables: he is never idle while food is being prepared. In .NET `async/await` is exactly that «kitchen ticket»: the method releases the thread during the wait and picks a thread from the pool back when the I/O completes.

It is crucial to distinguish **asynchrony from multithreading**. `async` does not create threads by itself — it only signals «there will be a wait, the thread can be released». I/O-bound operations (network, disk) genuinely free the thread when awaited. CPU-bound operations (computations) must be offloaded with `Task.Run`, otherwise you merely block the current thread «asynchronously». That is why `Task.Run(() => Thread.Sleep(1000))` delivers zero scalability — it simply shifts the blockage onto another pool thread.

The synchronization context (`SynchronizationContext`) decides **where** the continuation resumes after `await`. In UI applications (WPF/WinForms) it is the UI thread’s message queue, so the continuation is guaranteed to run on the UI thread — convenient, but dangerous when combined with `.Result`. In classic ASP.NET the context could capture the request context, causing deadlocks when synchronous code blocked over asynchronous code. In ASP.NET Core and console apps `SynchronizationContext` is `null`, and continuations run on arbitrary pool threads.

We will demonstrate the classic pitfalls in the code below: `.Result`/`.Wait()` on an async operation under a capturing context → **deadlock**; `async void` → exceptions become unhandleable and the call becomes «fire and forget»; `lock` around `await` — `lock` is oblivious to `await`, so the mutex may be acquired by one thread and released by another; blocking pool threads via `Task.Run(() => Thread.Sleep(...))` → pool starvation under load. The right tools: `CancellationToken` for cooperative cancellation, `ConfigureAwait(false)` in library code, `SemaphoreSlim` instead of `lock` for async sections, and `Channel<T>` for producer/consumer pipelines (covered in depth in M11-L07).

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — рабочий, потокобезопасный пример
// Демонстрирует: async I/O vs блокировка потоков, CancellationToken,
// ConfigureAwait, SemaphoreSlim(асинхронный lock), типичные ловушки.
using System.Diagnostics;
using System.Threading;
using System.Threading.Channels;

// ============================================================
// 1. Async I/O освобождает поток, синхронный Sleep — блокирует
// ============================================================
// «Официант не ждёт готовки блюда» — метод отдаёт поток в пул на время запроса.
public static class AsyncVsSync
{
    // ПРАВИЛЬНО: асинхронный HTTP-запрос, поток свободен во время ожидания
    // CORRECT: async HTTP call, the thread is free while waiting
    public static async Task<string> FetchAsync(string url, CancellationToken ct)
    {
        using var http = new HttpClient();
        // ConfigureAwait(false) в библиотечном коде: не захватываем контекст синхронизации
        // ConfigureAwait(false) in library code: do not capture the sync context
        var resp = await http.GetAsync(url, ct).ConfigureAwait(false);
        return await resp.Content.ReadAsStringAsync(ct).ConfigureAwait(false);
    }

    // ПЛОХО: поток «торчит» в Thread.Sleep — масштабируемости нет
    // BAD: the thread sits in Thread.Sleep — no scalability gain
    public static string FetchBad(string url)
    {
        Thread.Sleep(500); // имитация «ждём на кухне» / simulating «waiting at the stove»
        return url;
    }
}

// ============================================================
// 2. CancellationToken — кооперативная отмена
// ============================================================
public sealed class CancellableDownloader
{
    public async Task DownloadAllAsync(IEnumerable<string> urls, CancellationToken ct)
    {
        // ct.ThrowIfCancellationRequested() — явная точка отмены
        foreach (var url in urls)
        {
            ct.ThrowIfCancellationRequested(); // бросит OperationCanceledException
            // Передаём ct в каждую await-операцию: отмена сработает даже из глубины I/O
            // Pass ct into every await: cancellation propagates even from inside I/O
            try
            {
                var body = await AsyncVsSync.FetchAsync(url, ct).ConfigureAwait(false);
                Console.WriteLine($"OK {url} ({body.Length} bytes)");
            }
            catch (OperationCanceledException) when (ct.IsCancellationRequested)
            {
                // Ожидаемая отмена, НЕ ошибка / Expected cancellation, NOT an error
                Console.WriteLine("Cancelled / Отменено");
                throw;
            }
        }
    }
}

// ============================================================
// 3. SemaphoreSlim — «async lock», безопасен с await
// ============================================================
// ГЛАВНОЕ: обычный `lock` НЕЛЬЗЯ использовать с await — мьютекс
// захватывается одним потоком, а освобождается другим после await.
// MAIN: plain `lock` MUST NOT be used with await — the mutex is
// acquired on one thread and released on another after the await.
public sealed class AsyncRateLimiter
{
    private readonly SemaphoreSlim _gate = new(initialCount: 3, maxCount: 3);
    // одновременный лимит — 3 запроса / concurrency limit — 3 requests

    public async Task<string> CallLimitedAsync(string url, CancellationToken ct)
    {
        // Асинхронно ждём семафор — НЕ блокируем поток!
        // Asynchronously wait for the semaphore — does NOT block the thread!
        await _gate.WaitAsync(ct).ConfigureAwait(false);
        try
        {
            return await AsyncVsSync.FetchAsync(url, ct).ConfigureAwait(false);
        }
        finally
        {
            _gate.Release(); // всегда освобождаем в finally / always release in finally
        }
    }
}

// ============================================================
// 4. Параллельный запуск с ограничением — WhenAll + SemaphoreSlim
// ============================================================
public static class ParallelRunner
{
    public static async Task RunAllAsync(string[] urls, CancellationToken ct)
    {
        var limiter = new AsyncRateLimiter();
        // Запускаем все задачи сразу, но лимит внутри SemaphoreSlim(3)
        // Launch all tasks at once; the limit is enforced inside SemaphoreSlim(3)
        var tasks = urls.Select(u => limiter.CallLimitedAsync(u, ct)).ToArray();
        try
        {
            var results = await Task.WhenAll(tasks).ConfigureAwait(false);
            Console.WriteLine($"Done / Готово: {results.Length} responses");
        }
        catch (OperationCanceledException) when (ct.IsCancellationRequested)
        {
            // WhenAll автоматически «отменит» дочерние задачи, если выбросили
            Console.WriteLine("All cancelled / Всё отменено");
        }
    }
}

// ============================================================
// 5. КАК НЕ НАДО — ловушка .Result в контексте с захватом
// ============================================================
// В UI / старом ASP.NET такой код дедлочит: текущий поток ждёт .Result,
// а continuation асинхронной операции не может в него вернуться.
// In UI / classic ASP.NET this deadlocks: the current thread waits on .Result
// while the async continuation cannot re-enter that same context.
//
//   public string BadSync(string url) => FetchAsync(url, default).Result; // ⚠ ДЕДЛОК
//
// ПРАВИЛЬНО: распространяйте async вверх до точки входа (Main/handler).
// CORRECT: propagate async all the way up to the entry point.

// ============================================================
// 6. async void — почти всегда ошибка
// ============================================================
// Исключения в async void уходят в SynchronizationContext (или синхронно
// валят процесс) — их нельзя поймать try/catch у вызывающего.
// Exceptions in async void go to the SynchronizationContext (or tear down
// the process synchronously) — the caller’s try/catch cannot catch them.
//
//   public async void DoWorkBad() { await ...; }   // ⚠ НЕТ
//   public async Task DoWorkAsync() { await ...; } // ✓ ДА

// ============================================================
// 7. Бенчмарк-сравнение: 100 «запросов» синхронно vs асинхронно
// ============================================================
public static class ScalabilityDemo
{
    private static async Task FakeIoAsync(int ms, CancellationToken ct) =>
        await Task.Delay(ms, ct).ConfigureAwait(false);

    private static void FakeIoSync(int ms) => Thread.Sleep(ms);

    public static async Task<(TimeSpan sync, TimeSpan async)> MeasureAsync(CancellationToken ct)
    {
        const int N = 100, ms = 50;

        var sw = Stopwatch.StartNew();
        // Синхронно: каждый поток ждёт по 50 мс последовательно → ~5 с
        // Sync: each thread waits 50 ms sequentially → ~5 s
        for (int i = 0; i < N; i++) FakeIoSync(ms);
        sw.Stop();
        var syncTime = sw.Elapsed;

        sw.Restart();
        // Асинхронно: все 100 ожиданий стартуют сразу, ни один поток не блокируется → ~50 мс
        // Async: all 100 waits start at once, no thread is blocked → ~50 ms
        await Task.WhenAll(Enumerable.Range(0, N)
            .Select(_ => FakeIoAsync(ms, ct))).ConfigureAwait(false);
        sw.Stop();
        return (syncTime, sw.Elapsed);
    }
}

// ============================================================
// 8. Channel<T> — preview producer/consumer (подробно в M11-L07)
// ============================================================
// Потокобопасный канал между производителем и потребителем без явных lock-ов.
// Thread-safe channel between producer and consumer without explicit locks.
public static class ChannelPreview
{
    public static async Task RunAsync(CancellationToken ct)
    {
        // Ограниченный канал: производитель будет ждать, если потребитель не успевает
        // Bounded channel: the producer awaits if the consumer lags behind
        var channel = Channel.CreateBounded<string>(capacity: 10);

        // Производитель / Producer
        var producer = Task.Run(async () =>
        {
            for (int i = 0; i < 5; i++)
            {
                ct.ThrowIfCancellationRequested();
                await channel.Writer.WriteAsync($"item-{i}", ct).ConfigureAwait(false);
            }
            channel.Writer.Complete(); // сигнализируем конец / signal the end
        }, ct);

        // Потребитель / Consumer
        var consumer = Task.Run(async () =>
        {
            await foreach (var item in channel.Reader.ReadAllAsync(ct).ConfigureAwait(false))
                Console.WriteLine($"consumed: {item}");
        }, ct);

        await Task.WhenAll(producer, consumer).ConfigureAwait(false);
    }
}

// Точка входа для демонстрации
// Entry point for demonstration
public static class Program
{
    public static async Task Main()
    {
        using var cts = new CancellationTokenSource();
        var (sync, asyncTime) = await ScalabilityDemo.MeasureAsync(cts.Token);
        Console.WriteLine($"Sync / Синхронно:  {sync.TotalMilliseconds:F0} ms");
        Console.WriteLine($"Async / Асинхронно: {asyncTime.TotalMilliseconds:F0} ms");
        // Sync ≈ 5000 ms, Async ≈ 50 ms — разница в ~100× при 100 операциях
        await ChannelPreview.RunAsync(cts.Token);
    }
}
```

#### Best Practices

- Распространяйте `async` вверх до точки входа (Main/handler). Не смешивайте синхронный и асинхронный код через `.Result`/`.Wait()` — это прямой путь к дедлоку и истощению пула.
- Передавайте `CancellationToken` во все `await`-операции и проверяйте `ct.ThrowIfCancellationRequested()` в циклах CPU-bound. Отмена должна быть кооперативной.
- В библиотечном коде используйте `ConfigureAwait(false)` после каждого `await`, чтобы не захватывать контекст вызывающего. В UI-коде — наоборот, не убирайте контекст, если нужно вернуться в UI-поток.
- Используйте `SemaphoreSlim.WaitAsync` для асинхронных критических секций вместо `lock`. Никогда не оборачивайте `lock` вокруг `await`.
- Для CPU-bound работы применяйте `Task.Run`, для I/O — настоящие асинхронные API (`HttpClient`, `File.ReadAllAsync`, `Stream.ReadAsync`). Не имитируйте асинхронность через `Task.Run(() => Thread.Sleep(...))`.
- Для producer/consumer и pipelines используйте `Channel<T>` — он потокобезопасен и не требует ручных `lock`/`Monitor`.

- Propagate `async` all the way up to the entry point (Main/handler). Never bridge sync and async with `.Result`/`.Wait()` — that is a direct path to deadlocks and pool starvation.
- Pass a `CancellationToken` into every `await` and call `ct.ThrowIfCancellationRequested()` in CPU-bound loops. Cancellation must be cooperative.
- In library code use `ConfigureAwait(false)` after every `await` so you do not capture the caller’s context. In UI code, do the opposite: keep the context when you must resume on the UI thread.
- Use `SemaphoreSlim.WaitAsync` for async critical sections instead of `lock`. Never wrap `lock` around an `await`.
- Use `Task.Run` for CPU-bound work and genuine async APIs (`HttpClient`, `File.ReadAllAsync`, `Stream.ReadAsync`) for I/O. Do not fake async with `Task.Run(() => Thread.Sleep(...))`.
- Use `Channel<T>` for producer/consumer and pipelines — it is thread-safe and needs no manual `lock`/`Monitor`.

#### Частые ошибки / Common Mistakes

- `.Result`/`.Wait()` на асинхронной операции в контексте с захватом → **дедлок** → распространяйте `async` до точки входа; если вынужденно блокируетесь — используйте `GetAwaiter().GetResult()` только в консольных приложениях без `SynchronizationContext`, и только как крайнюю меру.
- `async void` для обработчиков → исключения невозможно поймать → используйте `async Task`; для event-хендлеров оборачивайте в `async void` только если сигнатура требует `void`, и обязательно оборачивайте тело в `try/catch` с логированием.
- `lock (obj) { await ... }` → `lock` не знает про `await`, мьютекс захватывается одним потоком, освобождается другим → используйте `SemaphoreSlim.WaitAsync`/`Release` в `try/finally`.
- `Task.Run(() => Thread.Sleep(1000))` для имитации асинхронного I/O → блокирует поток пула, даёт ложное ощущение масштабируемости → используйте `Task.Delay` для имитации I/O, `Task.Run` — только для CPU-bound работы.
- Забыли передать `CancellationToken` в `await` → отмена не сработает во время ожидания → всегда передавайте `ct` в каждую `await`-операцию и проверяйте `ct.ThrowIfCancellationRequested()` в циклах.
- `Task.WhenAll` без обработки первой упавшей задачи → остальные продолжают работать вслепую → оборачивайте каждую задачу в try/catch или используйте `Task.WhenAll` с агрегацией исключений и `CancellationToken`.
- Захват `ConfigureAwait(true)` (по умолчанию) в библиотечном коде → дедлоки и накладные расходы на маршалинг → в библиотеках всегда `ConfigureAwait(false)`, в UI — оставляйте по умолчанию.

- `.Result`/`.Wait()` on an async operation under a capturing context → **deadlock** → propagate `async` up to the entry point; if you must block, use `GetAwaiter().GetResult()` only in console apps with no `SynchronizationContext`, and only as a last resort.
- `async void` for handlers → exceptions cannot be caught → use `async Task`; for event handlers that require `void`, wrap the body in `try/catch` with logging.
- `lock (obj) { await ... }` → `lock` is unaware of `await`, the mutex is acquired on one thread and released on another → use `SemaphoreSlim.WaitAsync`/`Release` inside `try/finally`.
- `Task.Run(() => Thread.Sleep(1000))` to fake async I/O → it blocks a pool thread and gives a false sense of scalability → use `Task.Delay` to simulate I/O, `Task.Run` only for CPU-bound work.
- Forgetting to pass `CancellationToken` into `await` → cancellation will not fire during the wait → always pass `ct` into every `await` and call `ct.ThrowIfCancellationRequested()` in loops.
- `Task.WhenAll` without handling the first failing task → the others keep running blind → wrap each task in try/catch or aggregate exceptions and use a `CancellationToken`.
- `ConfigureAwait(true)` (default) captured in library code → deadlocks and marshalling overhead → in libraries always `ConfigureAwait(false)`; in UI leave the default.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я могу объяснить разницу между «асинхронность» и «многопоточность» своими словами.
- [ ] Я понимаю, что `async` не создаёт потоки, а освобождает текущий поток при I/O.
- [ ] Я никогда не пишу `.Result`/`.Wait()` на асинхронной операции в коде с `SynchronizationContext`.
- [ ] Я передаю `CancellationToken` во все `await`-операции и проверяю отмену в циклах.
- [ ] Я использую `ConfigureAwait(false)` в библиотечном коде после каждого `await`.
- [ ] Я заменяю `lock` + `await` на `SemaphoreSlim.WaitAsync` в `try/finally`.
- [ ] Я не путаю `Task.Delay` (имитация I/O) и `Task.Run(() => Thread.Sleep(...))` (блокировка потока).
- [ ] Я знаю, что `async void` — почти всегда ошибка, и применяю `async Task`.
- [ ] Я могу назвать механизм ОС для async I/O на Windows (IOCP), Linux (epoll), macOS (kqueue).
- [ ] Я знаю про `Channel<T>` как потокобезопасный примитив для producer/consumer (подробно в M11-L07).

- [ ] I can explain the difference between «asynchrony» and «multithreading» in my own words.
- [ ] I understand that `async` does not create threads; it releases the current thread during I/O.
- [ ] I never write `.Result`/`.Wait()` on an async operation when a `SynchronizationContext` is present.
- [ ] I pass a `CancellationToken` into every `await` and check for cancellation in loops.
- [ ] I use `ConfigureAwait(false)` in library code after every `await`.
- [ ] I replace `lock` + `await` with `SemaphoreSlim.WaitAsync` inside `try/finally`.
- [ ] I do not confuse `Task.Delay` (simulating I/O) with `Task.Run(() => Thread.Sleep(...))` (blocking a thread).
- [ ] I know that `async void` is almost always a mistake and use `async Task` instead.
- [ ] I can name the OS async-I/O mechanism on Windows (IOCP), Linux (epoll), macOS (kqueue).
- [ ] I know about `Channel<T>` as a thread-safe primitive for producer/consumer (covered in depth in M11-L07).

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/asynchronous-programming/](https://learn.microsoft.com/dotnet/csharp/asynchronous-programming/)

---

[⬆ К модулю M09](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
