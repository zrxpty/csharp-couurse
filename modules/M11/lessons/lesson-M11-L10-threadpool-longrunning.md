[⬆ К модулю M11](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M11-L10: ThreadPool настройки, TaskCreationOptions.LongRunning (Optional) / ThreadPool settings, TaskCreationOptions.LongRunning (Optional)

**Модуль / Module:** M11
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

`ThreadPool` в .NET — это пул рабочих потоков, которые среда создаёт и переиспользует для выполнения коротких задач (task, queue work item, таймеры, async-продолжения). Главная идея: создание потока — дорого (стек ~1 МБ, переключение контекста), поэтому пул «держит наготове» набор потоков и раздаёт им работу из глобальной FIFO-очереди. Аналогия: бригада курьеров в службе доставки — не нанимают нового курьера на каждый заказ, а используют свободных из штата, расширяя штат только при наплыве заказов.

**Рост пула / Growth.** Пул имеет два порога: «минимум» (`GetMinThreads`) и «максимум» (`GetMaxThreads`). Пока активных потоков меньше минимума, пул мгновенно добавляет новый поток на каждую входящую работу. Когда работа идёт, а свободных потоков нет, пул включит **ограниченный рост**: новый поток создаётся не сразу, а примерно раз в 0.5 секунды (starvation-интервал). Это намеренно медленно — так пул защищается от безудержного размножения потоков при всплеске.

**Голодание / Starvation.** Если все потоки пула заблокированы в синхронном ожидании (например, `Task.Result`, `Thread.Sleep`, блокирующий ввод-вывод), новые задачи стоят в очереди, а пул еле-еле подкармливает их новыми потоками раз в полсекунды. Приложение «зависает»: async-код не прогрессирует, таймеры не стреляют, запросы копятся. Это и есть starvation — классическая ловушка смешивания синхронного и асинхронного кода. Особенно коварно в ASP.NET / веб-приложениях: заблокированные пул-потоки → отсутствие прогресса → деградация сервиса.

**GetMinThreads / SetMinThreads.** `ThreadPool.GetMinThreads(out worker, out completionPorts)` возвращает текущий минимум для рабочих потоков и потоков ввода-вывода (IOCP). `SetMinThreads(worker, io)` повышает минимум — то есть количество потоков, которые создаются без задержки. Это полезно, когда у вас заранее известен кратковременный пик параллелизма и вы не хотите ждать 0.5 с на каждый рост. НО: повышать минимум «на всякий случай» вредно — больше потоков = больше памяти, больше переключений контекста, хуже локальность кэша.

**Когда настраивать пул?** Почти никогда — в 95% случаев дефолтные настройки оптимальны. Тонкая настройка оправдана, когда: (1) у вас холодный старт с резким всплеском short-lived задач; (2) вы точно измерили starvation и доказали, что его причина — рост пула, а не блокировки; (3) приложение работает в контейнере с известным лимитом CPU. Сначала устраняйте `Task.Wait`/`.Result` и блокирующие вызовы — это лечит причину, а `SetMinThreads` лишь маскирует симптом.

**TaskCreationOptions.LongRunning.** Этот флаг говорит планировщику: «задача долгая, не держи её в обычной очереди пула». Для такой задачи пул создаёт **отдельный (не pooled) поток**, который не занимает слот в стандартной очереди и не влияет на политику роста. Используйте LongRunning для действительно долгой CPU-работы (минуты) или для задач с активным ожиданием (`Thread.Sleep` в цикле, опрос внешнего ресурса). НЕ используйте для async-методов с `await` — там LongRunning бесполезен и даже вреден: вы получите лишний поток ради кода, который 90% времени ничего не делает. Запомните правило: LongRunning = долгая **синхронная** работа.

**Синхронизация и race conditions.** Когда несколько пул-потоков трогают разделяемое состояние (счётчик, коллекция, файл), нужны средства синхронизации: `lock` для коротких критических секций, `Interlocked` для простых числовых операций, `Channel<T>` для producer/consumer потоков данных. Главное правило: **никогда не держите `lock` через `await`** — monitors (lock) thread-affined, а await может продолжиться на другом потоке, оставив монитор захваченным навсегда. Для async-взаимодействия используйте `SemaphoreSlim(1,1)`, `Channel<T>`, `async-lock` реализации.

**Контекст синхронизации.** В UI-приложениях (WPF/WinForms) и старом ASP.NET есть `SynchronizationContext`, который маршалирует продолжения обратно в UI/контекст потока запроса. В Console, ASP.NET Core и большинстве фоновых сервисов `SynchronizationContext.Current == null`, и продолжения выполняются на произвольном пул-потоке. `ConfigureAwait(false)` — способ явно сказать «не маршалируй обратно», что снимает лишний хоп и снижает риск дедлоков при вызове async-кода из синхронного. Хорошая практика в библиотечном коде — всегда `ConfigureAwait(false)`.

#### Theory (EN)

The `ThreadPool` in .NET is a pool of worker threads that the runtime creates and reuses to execute short tasks — `Task`, queued work items, timers, async continuations. The core idea: creating a thread is expensive (~1 MB stack, context-switch overhead), so the pool keeps a set of threads ready and feeds them work from a global FIFO queue. Analogy: a courier fleet at a delivery service — you don't hire a new courier per order, you reuse idle ones from the roster and only expand the roster when orders pile up.

**Growth.** The pool has two thresholds: a "minimum" (`GetMinThreads`) and a "maximum" (`GetMaxThreads`). While the number of active threads is below the minimum, the pool instantly spins up a new thread for each incoming piece of work. Once the minimum is reached and all threads are busy, the pool switches to **throttled growth**: a new thread is created only roughly every 0.5 seconds (the starvation interval). This is intentionally slow — the pool protects itself from uncontrolled thread proliferation during bursts.

**Starvation.** If every pool thread is blocked in a synchronous wait (`Task.Result`, `Thread.Sleep`, blocking I/O), new tasks sit in the queue and the pool feeds new threads at a crawl — one every half second. The application "hangs": async code doesn't progress, timers don't fire, requests pile up. This is thread-pool starvation — the classic trap of mixing synchronous and asynchronous code. It is especially nasty in ASP.NET / web apps: blocked pool threads → no progress → service degradation.

**GetMinThreads / SetMinThreads.** `ThreadPool.GetMinThreads(out worker, out completionPorts)` returns the current minimum for worker threads and I/O-completion-port threads (IOCP). `SetMinThreads(worker, io)` raises the minimum — i.e. the number of threads the pool will create without delay. Useful when you know in advance about a brief burst of parallelism and don't want to pay the 0.5 s growth tax. BUT: raising the minimum "just in case" is harmful — more threads mean more memory, more context switches, worse cache locality.

**When to tune the pool?** Almost never — in 95% of cases defaults are optimal. Tuning is justified when: (1) you have a cold start with a sharp burst of short-lived tasks; (2) you have measured starvation and proven the cause is pool growth, not blocking; (3) the app runs in a container with a known CPU limit. First eliminate `Task.Wait`/`.Result` and blocking calls — that fixes the cause; `SetMinThreads` only masks the symptom.

**TaskCreationOptions.LongRunning.** This hint tells the scheduler: "the task is long, don't park it in the regular pool queue." For such a task the pool spawns a **dedicated (non-pooled) thread** that does not occupy a slot in the standard queue and does not skew the growth policy. Use LongRunning for genuinely long CPU work (minutes) or for tasks with active waiting (`Thread.Sleep` in a loop, polling an external resource). Do NOT use it for async methods with `await` — there it is useless and even harmful: you get an extra thread for code that does nothing 90% of the time. Memorize the rule: LongRunning = long **synchronous** work.

**Synchronization and race conditions.** When several pool threads touch shared state (counter, collection, file), you need synchronization: `lock` for short critical sections, `Interlocked` for simple numeric operations, `Channel<T>` for producer/consumer data flows. The cardinal rule: **never hold a `lock` across an `await`** — monitors are thread-affined, while an await may resume on a different thread, leaving the monitor held forever. For async cooperation use `SemaphoreSlim(1,1)`, `Channel<T>`, or async-lock implementations.

**Synchronization context.** UI applications (WPF/WinForms) and legacy ASP.NET have a `SynchronizationContext` that marshals continuations back to the UI/request thread. In Console apps, ASP.NET Core, and most background services `SynchronizationContext.Current == null`, and continuations run on arbitrary pool threads. `ConfigureAwait(false)` is the explicit way to say "don't marshal back", removing an extra hop and reducing deadlock risk when async code is called from synchronous code. Good practice in library code is to always `ConfigureAwait(false)`.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8+ — ThreadPool tuning, starvation, LongRunning demo.
// C# 12 / .NET 8+ — демонстрация настройки пула, starvation и LongRunning.

using System.Buffers;
using System.Collections.Concurrent;
using System.Diagnostics;

namespace M11.L10;

public static class ThreadPoolDemo
{
    // 1) Reading & raising the minimum. Reading is thread-safe; setting must be done once at startup.
    // 1) Чтение и повышение минимума. Чтение потокобезопасно; настройку делают один раз на старте.
    public static void ConfigurePoolAtStartup(int desiredWorkerThreads, int desiredIoThreads)
    {
        ThreadPool.GetMinThreads(out int currentWorker, out int currentIo);
        Console.WriteLine($"Min before: worker={currentWorker}, io={currentIo}");
        // Min до: worker=..., io=...

        // SetMinThreads returns false if the value is rejected (e.g. lower than the current count).
        // SetMinThreads вернёт false, если значение отклонено (например, меньше текущего минимума).
        bool ok = ThreadPool.SetMinThreads(
            Math.Max(desiredWorkerThreads, currentWorker),
            Math.Max(desiredIoThreads, currentIo));

        if (!ok)
            Console.WriteLine("SetMinThreads rejected the value / SetMinThreads отклонил значение");

        ThreadPool.GetMinThreads(out int newWorker, out int newIo);
        Console.WriteLine($"Min after:  worker={newWorker}, io={newIo}");
    }

    // 2) Starvation: blocking pool threads with .Result/.Wait degrades throughput.
    // 2) Starvation: блокировка пул-потоков через .Result/.Wait падает throughput.
    public static async Task DemonstrateStarvationAsync()
    {
        var sw = Stopwatch.StartNew();

        // BAD: each task blocks a pool thread in a synchronous sleep.
        // ПЛОХО: каждая задача блокирует пул-поток в синхронном Sleep.
        var blockingTasks = Enumerable.Range(0, 50)
            .Select(_ => Task.Run(() => Thread.Sleep(500))) // holds a pool thread / держит пул-поток
            .ToArray();

        // GOOD: async work releases the thread back to the pool during the wait.
        // ХОРОШО: async-работа возвращает поток в пул на время ожидания.
        var asyncTasks = Enumerable.Range(0, 50)
            .Select(_ => Task.Delay(500)) // no thread occupied / поток не занят
            .ToArray();

        await Task.WhenAll(blockingTasks.Concat(asyncTasks));
        sw.Stop();
        Console.WriteLine($"All done in {sw.ElapsedMilliseconds} ms (starvation demo)");
        // Всё выполнено за ... мс (демо starvation)
    }

    // 3) LongRunning: dedicated thread for long synchronous work — does NOT consume a pool slot.
    // 3) LongRunning: отдельный поток для долгой синхронной работы — НЕ занимает слот пула.
    public static Task LongRunningCpuWorkAsync(CancellationToken ct)
    {
        // LongRunning hint: the scheduler spawns a non-pooled thread.
        // Подсказка LongRunning: планировщик создаёт непул-поток.
        return Task.Factory.StartNew(() =>
        {
            // Poll the token frequently so the work is cancellable and never blocks forever.
            // Часто опрашиваем токен — работа отменяема и не висит вечно.
            while (!ct.IsCancellationRequested)
            {
                DoCpuChunk();
                ct.ThrowIfCancellationRequested();
            }
        }, ct, TaskCreationOptions.LongRunning | TaskCreationOptions.DenyChildAttach,
           TaskScheduler.Default);
    }

    private static void DoCpuChunk()
    {
        // A short CPU-bound unit; replace with your real algorithm.
        // Короткая CPU-bound единица работы; замените на реальный алгоритм.
        double acc = 0;
        for (int i = 0; i < 100_000; i++) acc += Math.Sqrt(i);
    }

    // 4) Thread-safe producer/consumer with Channel<T> — the recommended pattern for M11.
    // 4) Потокобезопасный producer/consumer через Channel<T> — рекомендованный паттерн для M11.
    public static async Task RunPipelineAsync(CancellationToken ct)
    {
        // Bounded channel applies back-pressure: producers await when the buffer is full.
        // Ограниченный канал создаёт обратное давление: продьюсеры ждут при переполнении буфера.
        var channel = Channel.CreateBounded<string>(capacity: 16);

        // Producer: long synchronous read simulated by Thread.Sleep.
        // Продьюсер: долгое синхронное чтение, имитируемое Thread.Sleep.
        Task producer = Task.Factory.StartNew(async () =>
        {
            try
            {
                for (int i = 0; i < 100; i++)
                {
                    ct.ThrowIfCancellationRequested();
                    Thread.Sleep(20);          // blocking I/O simulation / имитация блокирующего I/O
                    await channel.Writer.WriteAsync($"item-{i}", ct); // back-pressure aware
                }
            }
            finally
            {
                channel.Writer.Complete();      // signal consumers to finish / сигнал окончания
            }
        }, ct, TaskCreationOptions.LongRunning | TaskCreationOptions.DenyChildAttach,
           TaskScheduler.Default).Unwrap();

        // Consumers: pure async, release threads between iterations.
        // Консьюмеры: чистый async, отпускают потоки между итерациями.
        Task consumer1 = ConsumeAsync(channel.Reader, "C1", ct);
        Task consumer2 = ConsumeAsync(channel.Reader, "C2", ct);

        await Task.WhenAll(producer, consumer1, consumer2);
    }

    private static async Task ConsumeAsync(ChannelReader<string> reader, string id, CancellationToken ct)
    {
        // ReadAllAsync is cancellation-aware and completes when the writer calls Complete().
        // ReadAllAsync учитывает отмену и завершается, когда продьюсер зовёт Complete().
        await foreach (var item in reader.ReadAllAsync(ct).ConfigureAwait(false))
        {
            await ProcessAsync(item, ct).ConfigureAwait(false); // NEVER lock across await / НЕ lock через await
            Console.WriteLine($"[{id}] processed {item}");
        }
    }

    private static async Task ProcessAsync(string item, CancellationToken ct)
    {
        // Simulate async I/O. NEVER use .Result/.Wait here — that would re-introduce starvation.
        // Имитируем async I/O. НИКОГДА не зовите .Result/.Wait — это вернёт starvation.
        await Task.Delay(10, ct).ConfigureAwait(false);
    }

    // 5) Thread-safe counter using Interlocked — no lock needed for a single integer.
    // 5) Потокобезопасный счётчик через Interlocked — lock не нужен для одного числа.
    private static int _processed;
    public static int IncrementProcessed() => Interlocked.Increment(ref _processed);
    public static int ReadProcessed() => Volatile.Read(ref _processed);
}

// Entry point / Точка входа
public static class Program
{
    public static async Task Main()
    {
        // Use a single CancellationTokenSource for cooperative shutdown.
        // Один CancellationTokenSource для кооперативной остановки.
        using var cts = new CancellationTokenSource();

        ThreadPoolDemo.ConfigurePoolAtStartup(desiredWorkerThreads: 32, desiredIoThreads: 32);

        // Cancel everything after 5 seconds to demonstrate cooperative cancellation.
        // Отменяем всё через 5 секунд — демонстрация кооперативной отмены.
        _ = Task.Delay(TimeSpan.FromSeconds(5)).ContinueWith(_ => cts.Cancel());

        try
        {
            await ThreadPoolDemo.DemonstrateStarvationAsync();
            await ThreadPoolDemo.LongRunningCpuWorkAsync(cts.Token);
            await ThreadPoolDemo.RunPipelineAsync(cts.Token);
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine("Cancelled / Отменено");
        }

        Console.WriteLine($"Processed total: {ThreadPoolDemo.ReadProcessed()}");
    }
}
```

#### Best Practices

- Настраивайте пул только после измерений: сначала профайлинг и устранение блокировок, потом — `SetMinThreads`. / Tune the pool only after measuring: first profile and remove blocking calls, then consider `SetMinThreads`.
- Используйте `TaskCreationOptions.LongRunning` только для долгой синхронной CPU-работы или poll-циклов; никогда для `async`/`await`. / Use `TaskCreationOptions.LongRunning` only for long synchronous CPU work or poll loops; never for `async`/`await`.
- Предпочитайте `Channel<T>` для producer/consumer — он потокобезопасен, поддерживает back-pressure и отмену. / Prefer `Channel<T>` for producer/consumer — it is thread-safe, supports back-pressure and cancellation.
- В библиотечном коде всегда используйте `ConfigureAwait(false)`, в приложениях — по обстоятельствам. / In library code always use `ConfigureAwait(false)`; in applications — case by case.
- Для счётчиков используйте `Interlocked`, для коротких критических секций — `lock`, для async — `SemaphoreSlim(1,1)`. / Use `Interlocked` for counters, `lock` for short critical sections, `SemaphoreSlim(1,1)` for async.

#### Частые ошибки / Common Mistakes

- Вызов `Task.Result` или `.Wait()` на пуле → starvation. → Замените на `await`; если нужно «дождаться синхронно» — пересмотрите архитектуру. / Calling `Task.Result` or `.Wait()` on the pool → starvation. → Replace with `await`; if you truly must wait synchronously, rethink the architecture.
- `async void` в обработчиках событий без try/catch → необработанное исключение роняет процесс. → Используйте `async Task` или оборачивайте в безопасный helper. / `async void` in event handlers without try/catch → unhandled exception crashes the process. → Use `async Task` or wrap with a safe helper.
- `lock` через `await` → монитор остаётся захваченным на «другом» потоке → deadlock/коррупция. → Используйте `SemaphoreSlim(1,1)` с `await sem.WaitAsync()`. / `lock` across `await` → monitor stays held on a "different" thread → deadlock/corruption. → Use `SemaphoreSlim(1,1)` with `await sem.WaitAsync()`.
- `LongRunning` на async-методе → лишний непул-поток без выгоды. → Уберите флаг для `async` кода. / `LongRunning` on an async method → an extra non-pooled thread with no benefit. → Drop the flag for `async` code.
- Повышение `SetMinThreads` «про запас» → лишняя память, переключения контекста. → Повышайте только под измеренный пик. / Raising `SetMinThreads` "just in case" → wasted memory and context switches. → Raise only for a measured peak.
- Забыли `ConfigureAwait(false)` в горячем пути библиотеки → лишний хоп и риск дедлока в UI/legacy ASP.NET. → Всегда добавляйте в библиотеках. / Forgot `ConfigureAwait(false)` in a hot library path → extra hop and deadlock risk in UI/legacy ASP.NET. → Always add it in libraries.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я могу объяснить, что такое starvation и почему `Task.Result` его вызывает. / I can explain what starvation is and why `Task.Result` causes it.
- [ ] Я знаю разницу между минимальным и максимальным порогами пула. / I know the difference between the minimum and maximum pool thresholds.
- [ ] Я понимаю, когда оправдан `SetMinThreads`, а когда — нет. / I understand when `SetMinThreads` is justified and when it is not.
- [ ] Я применяю `TaskCreationOptions.LongRunning` только к долгой синхронной работе. / I apply `TaskCreationOptions.LongRunning` only to long synchronous work.
- [ ] Я никогда не держу `lock` через `await` и использую `SemaphoreSlim`. / I never hold `lock` across `await` and use `SemaphoreSlim`.
- [ ] Я использую `Channel<T>` для producer/consumer и передаю `CancellationToken`. / I use `Channel<T>` for producer/consumer and pass `CancellationToken`.
- [ ] Я добавляю `ConfigureAwait(false)` в библиотечном коде. / I add `ConfigureAwait(false)` in library code.

#### Ресурсы / Resources

- [Microsoft Learn — ThreadPool — https://learn.microsoft.com/dotnet/api/system.threading.threadpool](https://learn.microsoft.com/dotnet/api/system.threading.threadpool)
- [Microsoft Learn — ThreadPool.SetMinThreads — https://learn.microsoft.com/dotnet/api/system.threading.threadpool.setminthreads](https://learn.microsoft.com/dotnet/api/system.threading.threadpool.setminthreads)
- [Microsoft Learn — TaskCreationOptions.LongRunning — https://learn.microsoft.com/dotnet/api/system.threading.tasks.taskcreationoptions](https://learn.microsoft.com/dotnet/api/system.threading.tasks.taskcreationoptions)
- [Stephen Toub — ThreadPool starvation — https://devblogs.microsoft.com/dotnet/?s=threadpool+starvation](https://devblogs.microsoft.com/dotnet/?s=threadpool+starvation)
- [Microsoft Learn — Channel<T> — https://learn.microsoft.com/dotnet/api/system.threading.channels.channel-1](https://learn.microsoft.com/dotnet/api/system.threading.channels.channel-1)

---

[⬆ К модулю M11](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
