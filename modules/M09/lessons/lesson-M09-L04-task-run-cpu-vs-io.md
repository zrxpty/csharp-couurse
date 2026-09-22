[⬆ К модулю M09](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M09-L04: Task.Run, CPU-bound vs I/O-bound / Task.Run, CPU-bound vs I/O-bound

**Модуль / Module:** M09
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

В .NET асинхронность решает две разные задачи, которые новички часто путают: **CPU-bound** (вычисления, нагружающие процессор) и **I/O-bound** (ожидание внешних операций — сеть, диск, база данных). `Task.Run` — это инструмент для первой задачи, но его повсеместное использование для второй — главная причина тормозов и дедлоков в реальных проектах.

**CPU-bound работа** — это когда поток реально считает: парсинг большого JSON, шифрование, обработка изображений, тяжёлые математические вычисления. Если запустить такую работу в UI-потоке, интерфейс зависнет; если в потоке пула без `Task.Run` — вы просто заняли один из потоков пула напрямую. `Task.Run(() => HeavyCompute())` делает две вещи: переносит работу в пул потоков и возвращает `Task`, который можно `await`. Это и есть **offload** — разгрузка вызывающего потока.

**I/O-bound работа** — это ожидание. Когда вы вызываете `HttpClient.GetAsync`, никакой поток не «считает» ответ сервера; операция регистрируется в ОС через I/O completion port, и поток возвращается в пул до появления данных. Добавлять `Task.Run(() => httpClient.GetAsync(url))` здесь бессмысленно и вредно: вы тратите лишний поток пула на то, чтобы запустить операцию, которая и так не занимает поток. Правильно — просто `await httpClient.GetAsync(url)` без `Task.Run`.

**Контекст синхронизации (SynchronizationContext).** В UI-приложениях (WPF/WinForms) и старом ASP.NET `await` по умолчанию пытается вернуться в исходный контекст. Это создаёт классический дедлок: UI-поток блокируется на `.Result` или `.Wait()`, а async-операция не может продолжиться, потому что ждёт освобождения того же UI-потока. `ConfigureAwait(false)` говорит рантайму не возвращаться в исходный контекст — продолжить в пуле потоков. В библиотечном коде **всегда** используйте `ConfigureAwait(false)`; в коде приложения решайте по контексту. В ASP.NET Core `SynchronizationContext` отсутствует — дедлоков этого типа там нет, но привычка к `ConfigureAwait(false)` остаётся хорошей.

**Когда `Task.Run` вреден.** Частая ошибка — оборачивать весь async-метод: `Task.Run(async () => await DoAsync())`. Это не ускоряет код, а лишь плодит лишние переключения контекста и занимается пулом. Другая ловушка — `Task.Run` внутри async-метода для I/O: вы получаете **overlapped blocking**, где один поток пула блокируется, ожидая другого. Правило: `Task.Run` только для синхронных CPU-bound операций; для I/O — нативные async-API.

**Offload vs Scalability.** `Task.Run` улучшает отзывчивость одного вызова (offload), но не масштабируемость системы. Пул потоков ограничен; если каждый запрос будет кидать в `Task.Run` тяжёлую работу, пул исчерпается, и throughput упадёт. Для масштабируемости важно: I/O делайте нативным async, CPU-bound либо ограничивайте (`SemaphoreSlim`, `Parallel.ForEachAsync`), либо выносите в фоновый сервис с `Channel`-очередью.

**CancellationToken** передавайте во все `Task.Run` и async-операции: `Task.Run(work, token)`. Без этого отменить долгий compute нельзя, и сервис нельзя gracefully остановить. Внутри CPU-bound работы периодически вызывайте `token.ThrowIfCancellationRequested()`.

#### Theory (EN)

In .NET, asynchronous programming solves two distinct problems that newcomers often conflate: **CPU-bound** work (computations that keep the processor busy) and **I/O-bound** work (waiting on external operations — network, disk, database). `Task.Run` is a tool for the first problem, but applying it to the second is the single most common source of sluggish services and mysterious deadlocks in production code.

**CPU-bound work** is when a thread is genuinely computing: parsing a large JSON document, encrypting data, processing images, running heavy math. If you run such work on the UI thread, the interface freezes; if you run it on a pool thread directly, you are simply consuming one pool thread without the convenience of a `Task`. `Task.Run(() => HeavyCompute())` does two things: it offloads the work to the thread pool and returns a `Task` you can `await`. This is **offloading** — freeing the caller's thread.

**I/O-bound work** is waiting. When you call `HttpClient.GetAsync`, no thread is "computing" the server's reply; the operation is registered with the OS through an I/O completion port, and the thread returns to the pool until data arrives. Wrapping it as `Task.Run(() => httpClient.GetAsync(url))` is pointless and harmful: you spend an extra pool thread to start an operation that never needed a thread in the first place. The correct form is a plain `await httpClient.GetAsync(url)` with no `Task.Run`.

**Synchronization context.** In UI applications (WPF/WinForms) and classic ASP.NET, `await` tries to resume on the original context by default. This creates the classic deadlock: the UI thread blocks on `.Result` or `.Wait()`, while the async operation cannot resume because it is waiting for the very same UI thread to be freed. `ConfigureAwait(false)` instructs the runtime not to capture the original context and to resume on a pool thread instead. In library code, **always** use `ConfigureAwait(false)`; in application code, decide based on context. ASP.NET Core has no `SynchronizationContext` — this class of deadlock does not occur there, but the habit of `ConfigureAwait(false)` remains healthy.

**When `Task.Run` is harmful.** A common mistake is wrapping an entire async method: `Task.Run(async () => await DoAsync())`. This does not speed anything up; it only adds extra context switches and consumes pool threads. Another trap is using `Task.Run` inside an async method for I/O: you get **overlapped blocking**, where one pool thread blocks waiting for another. The rule: `Task.Run` only for synchronous CPU-bound operations; for I/O, use native async APIs.

**Offload vs Scalability.** `Task.Run` improves the responsiveness of a single call (offload) but not the scalability of the system. The thread pool is bounded; if every request throws heavy work at `Task.Run`, the pool becomes exhausted and throughput collapses. For scalability: do I/O with native async, and throttle CPU-bound work (`SemaphoreSlim`, `Parallel.ForEachAsync`) or move it to a background service with a `Channel`-based queue.

**CancellationToken** must be passed into every `Task.Run` and async operation: `Task.Run(work, token)`. Without it, a long computation cannot be cancelled and a service cannot shut down gracefully. Inside CPU-bound work, call `token.ThrowIfCancellationRequested()` periodically.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — рабочий пример. Демонстрирует CPU-bound через Task.Run,
// нативный I/O без Task.Run, CancellationToken, ConfigureAwait и
// throttle через SemaphoreSlim + Channel pipeline.
// C# 12 / .NET 8 — working sample: CPU-bound via Task.Run,
// native I/O without Task.Run, CancellationToken, ConfigureAwait,
// throttle via SemaphoreSlim + Channel pipeline.

using System.Diagnostics;
using System.Threading.Channels;

namespace M09L04;

public sealed class CpuVsIoDemo
{
    private readonly HttpClient _http = new();

    // ✅ CPU-bound: тяжёлые вычисления через Task.Run + CancellationToken.
    // ✅ CPU-bound: heavy compute via Task.Run + CancellationToken.
    public Task<int> HeavySumAsync(int count, CancellationToken token) =>
        Task.Run(() =>
        {
            long acc = 0;                       // локальная переменная — нет race / no race
            for (int i = 0; i < count; i++)
            {
                token.ThrowIfCancellationRequested();
                acc += (long)i * i;
            }
            return checked((int)(acc % int.MaxValue));
        }, token);

    // ✅ I/O-bound: НЕТ Task.Run. Просто await нативного async-API + ConfigureAwait(false).
    // ✅ I/O-bound: NO Task.Run. Just await the native async API + ConfigureAwait(false).
    public async Task<string> FetchAsync(string url, CancellationToken token)
    {
        // ❌ Никогда так: Task.Run(async () => await _http.GetStringAsync(url, token));
        // ❌ Never: Task.Run(async () => await _http.GetStringAsync(url, token));
        using var resp = await _http.GetAsync(url, token).ConfigureAwait(false);
        return await resp.Content.ReadAsStringAsync(token).ConfigureAwait(false);
    }

    // ✅ Scalability: ограничиваем параллелизм CPU-bound задач через SemaphoreSlim.
    // ✅ Scalability: throttle CPU-bound parallelism with SemaphoreSlim.
    public async Task<int[]> RunBatchAsync(int[] inputs, CancellationToken token)
    {
        using var gate = new SemaphoreSlim(Environment.ProcessorCount);
        var tasks = new List<Task<int>>(inputs.Length);
        foreach (var n in inputs)
        {
            await gate.WaitAsync(token).ConfigureAwait(false);
            tasks.Add(Task.Run(async () =>
            {
                try { return await HeavySumAsync(n, token).ConfigureAwait(false); }
                finally { gate.Release(); }
            }, token));
        }
        // Когда хоть один бросит, остальные получат cancel через общий token.
        // When one throws, others are cancelled via the shared token.
        return await Task.WhenAll(tasks).ConfigureAwait(false);
    }

    // ✅ Producer/consumer pipeline на Channel — масштабируемая обработка без блокировки пула.
    // ✅ Producer/consumer pipeline on Channel — scalable processing without blocking the pool.
    public async Task ProcessPipelineAsync(IEnumerable<string> urls, CancellationToken token)
    {
        var channel = Channel.CreateBounded<string>(new BoundedChannelOptions(64)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = false,
            SingleWriter = false,
        });

        // Producer: I/O без Task.Run.
        // Producer: I/O without Task.Run.
        var producer = Task.Run(async () =>
        {
            try
            {
                foreach (var url in urls)
                {
                    var html = await FetchAsync(url, token).ConfigureAwait(false);
                    await channel.Writer.WriteAsync(html, token).ConfigureAwait(false);
                }
            }
            finally { channel.Writer.Complete(); }
        }, token);

        // Consumers: CPU-bound над скачанным, ограничены числом ядер.
        // Consumers: CPU-bound over downloaded data, bounded by core count.
        var consumers = new List<Task>();
        for (int i = 0; i < Environment.ProcessorCount; i++)
        {
            consumers.Add(Task.Run(async () =>
            {
                await foreach (var html in channel.Reader.ReadAllAsync(token).ConfigureAwait(false))
                {
                    token.ThrowIfCancellationRequested();
                    _ = HashLen(html);           // чистая функция / pure function
                }
            }, token));
        }

        await Task.WhenAll(producer, Task.WhenAll(consumers)).ConfigureAwait(false);
    }

    private static int HashLen(string s)
    {
        long h = 0;
        foreach (var c in s) h = (h * 31 + c) & 0x7FFFFFFF;
        return checked((int)h ^ s.Length);
    }

    // ❌ Демонстрация классического дедлока (UI / old ASP.NET). НЕ запускать вprod-коде.
    // ❌ Classic deadlock demo (UI / old ASP.NET). DO NOT use in prod.
    public static async Task DeadlockDemoAsync()
    {
        // В контексте с SynchronizationContext (WPF/WinForms) этот код зависнет:
        // UI-поток блокируется на .Result, а продолжение await ждёт освобождения UI-потока.
        // Under a SynchronizationContext (WPF/WinForms) this hangs:
        // the UI thread blocks on .Result, and the await continuation waits for the UI thread.
        //
        // var result = SomeAsyncMethod().Result;          // ❌ блокировка / blocking
        //
        // ✅ Лечится двумя способами / Fixed two ways:
        //   1) await SomeAsyncMethod()  — не блокировать поток / never block.
        //   2) внутри SomeAsyncMethod использовать ConfigureAwait(false).
    }
}
```

#### Best Practices
- Используй `Task.Run` ТОЛЬКО для синхронных CPU-bound операций; для I/O вызывай нативные async-API без обёрток.
- Всегда передавай `CancellationToken` в `Task.Run` и в каждую async-операцию; внутри compute проверяй `token.ThrowIfCancellationRequested()`.
- В библиотечном коде используй `ConfigureAwait(false)` на каждом `await`, чтобы не зависеть от контекста вызывающего.
- Для масштабируемости ограничивай параллелизм CPU-bound через `SemaphoreSlim`, `Parallel.ForEachAsync` или очередь на `Channel`.
- Никогда не блокируй async-операции через `.Result`/`.Wait()` в контексте с `SynchronizationContext`.

- Use `Task.Run` ONLY for synchronous CPU-bound work; for I/O, call native async APIs without wrappers.
- Always pass a `CancellationToken` into `Task.Run` and every async operation; inside compute, check `token.ThrowIfCancellationRequested()`.
- In library code, apply `ConfigureAwait(false)` on every `await` to avoid depending on the caller's context.
- For scalability, throttle CPU-bound parallelism with `SemaphoreSlim`, `Parallel.ForEachAsync`, or a `Channel`-based queue.
- Never block async operations with `.Result`/`.Wait()` under a `SynchronizationContext`.

#### Частые ошибки / Common Mistakes
- `Task.Run(async () => await io())` для I/O → просто `await io()` без `Task.Run`.
- `.Result` / `.Wait()` на async-задаче в UI/ASP.NET → дедлок; используй `await` целиком вверх по стеку.
- `lock (obj) { await ... }` → `lock` нельзя держать через `await`; используй `SemaphoreSlim.WaitAsync`.
- `async void` для бизнес-логики → исключения не перехватываются, процесс падает; только для event handlers.
- Отсутствие `CancellationToken` → невозможность отмены и graceful shutdown.
- `Task.Run` без throttle в каждом запросе → истощение пула потоков и падение throughput.

- `Task.Run(async () => await io())` for I/O → just `await io()` with no `Task.Run`.
- `.Result` / `.Wait()` on an async task in UI/ASP.NET → deadlock; propagate `await` up the stack.
- `lock (obj) { await ... }` → a `lock` cannot span an `await`; use `SemaphoreSlim.WaitAsync`.
- `async void` for business logic → exceptions are unobserved and crash the process; only for event handlers.
- Missing `CancellationToken` → no cancellation and no graceful shutdown.
- `Task.Run` without throttling in every request → thread pool starvation and throughput collapse.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я отличаю CPU-bound (вычисления) от I/O-bound (ожидание сети/диска).
- [ ] Я использую `Task.Run` только для CPU-bound и с `CancellationToken`.
- [ ] В I/O-методах нет `Task.Run`-обёрток, только нативный `await`.
- [ ] В библиотечном коде везде стоит `ConfigureAwait(false)`.
- [ ] Нигде нет `.Result`/`.Wait()` на async-задачах, особенно под `SynchronizationContext`.
- [ ] Параллельный CPU-bound труд ограничен `SemaphoreSlim`/`Channel`/`Parallel.ForEachAsync`.
- [ ] Я не держу `lock` через `await` и не пишу `async void` вне event handlers.

- [ ] I distinguish CPU-bound (compute) from I/O-bound (network/disk wait).
- [ ] I use `Task.Run` only for CPU-bound and always with a `CancellationToken`.
- [ ] My I/O methods have no `Task.Run` wrapper, only a native `await`.
- [ ] Library code uses `ConfigureAwait(false)` everywhere.
- [ ] There are no `.Result`/`.Wait()` calls on async tasks, especially under `SynchronizationContext`.
- [ ] Parallel CPU-bound work is bounded by `SemaphoreSlim`/`Channel`/`Parallel.ForEachAsync`.
- [ ] I never hold a `lock` across `await` and never write `async void` outside event handlers.

#### Ресурсы / Resources
- [Microsoft Learn — Calling Synchronous Methods Asynchronously — https://learn.microsoft.com/dotnet/standard/asynchronous-programming-patterns/calling-synchronous-methods-asynchronously]
- [Microsoft Learn — Task-based Asynchronous Pattern (TAP) — https://learn.microsoft.com/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap]
- [Stephen Toub — Should I expose asynchronous wrappers for synchronous methods? — https://devblogs.microsoft.com/dotnet/should-i-expose-asynchronous-wrappers-for-synchronous-methods/]
- [.NET GitHub — Channel\<T\> docs — https://learn.microsoft.com/dotnet/api/system.threading.channels.channel-1]

---

[⬆ К модулю M09](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
