[⬆ К модулю M11](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M11-L01: Thread (исторически), пул потоков, почему Task / Thread (historical), thread pool, why Task

**Модуль / Module:** M11
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Исторически в .NET единственным способом выполнить код параллельно был класс `Thread` из пространства имён `System.Threading`. Каждый объект `Thread` — это настоящая управляемая оболочка над потоком ОС (Windows fiber/Linux pthread), которая создаётся дорого: около 1 МБ стека по умолчанию плюс накладные расходы ядра на переключение контекста. `Thread` бывает двух видов: **foreground** (основной) — он удерживает процесс живым, пока работает; и **background** (фоновый) — процесс завершится, не дожидаясь его. По умолчанию через `new Thread(...)` создаётся foreground-поток, а потоки пула всегда background. Это простое различие становится ловушкой: если вы запустите foreground-поток и забудете его остановить, ваше приложение «зависнет» в диспетчере задач даже после закрытия главного окна.

Создавать новый `Thread` под каждую операцию — расточительно. Поэтому в .NET появился **ThreadPool** — переиспользуемый пул потоков. Вместо «создал→запустил→уничтожил», вы ставите работу в очередь (`ThreadPool.QueueUserWorkItem`), а пул подбирает простаивающий поток. Пул динамически растёт и сокращается по эвристикам CLR: при всплеске добавляет потоки с ограничением по скорости (примерно 1 поток/0.5 сек для CPU-bound), а при простое — удаляет. Число логических ядер доступно через `Environment.ProcessorCount` — это естественная подсказка, сколько параллельной CPU-работы имеет смысл запускать одновременно.

Важно различать два рода потоков внутри инфраструктуры .NET: **worker threads** (потоки пула, обслуживающие CPU-bound работу и делегаты) и **IOCP/IO-потоки** (выделенные для ожидания асинхронных операций ввода-вывода через порты завершения I/O). Когда вы вызываете `await` над реально асинхронной операцией (сокет, файл, БД), блокируемый поток не занят — он возвращается в пул, а завершение I/O пробуждает IO-поток. Это и есть основа высокой пропускной способности асинхронного .NET.

Почему же `Task` лучше `Thread`? `Task` — это абстракция над «единицей работы», а не над потоком. `Task.Run` ставит работу в тот же ThreadPool, но даёт Composition: продолжения (`ContinueWith`, `await`), обработку исключений (агрегируются в `AggregateException`), отмену через `CancellationToken`, результат (`Task<T>`), и кооперативную конфигурацию (`ConfigureAwait`). Создание `Thread` вручную для коротких операций сегодня почти всегда ошибка — вы лишаете пул возможности балансировать нагрузку и рискуете исчерпать потоки под блокирующими вызовами. Современный код: `Thread` — только для редких случаев (длинный CPU-bound цикл с особыми требованиями к стеку/приоритету, фоновый демон с контролем жизненного цикла), всё остальное — `Task` и `async/await`.

**Главные concurrency-ловушки этого урока:** блокировка потока пула через `.Result`/`.Wait()` — ведёт к голоданию пула и потенциальным дедлокам в контексте синхронизации (SynchronizationContext, например в UI/ASP.NET classic); запуск бесконечного цикла в `Task.Run` без `CancellationToken` — поток пул никогда не вернёт; смешение `lock` и `await` — `lock` не освобождается во время ожидания и может тормозить весь пул. Мы разберём эти случаи в коде и чек-листе.

#### Theory (EN)

Historically, the only way to run code in parallel in .NET was the `Thread` class from `System.Threading`. Each `Thread` instance is a genuine managed wrapper over an OS thread (Windows fiber / Linux pthread) and is expensive to create: roughly 1 MB of stack by default plus kernel context-switch overhead. A `Thread` is either **foreground** — it keeps the process alive while running — or **background** — the process can terminate without waiting for it. By default `new Thread(...)` produces a foreground thread, whereas thread-pool threads are always background. This simple distinction is a trap: if you start a foreground thread and forget to stop it, your app will linger in the Task Manager even after the main window closes.

Creating a fresh `Thread` for every operation is wasteful, so .NET introduced the **ThreadPool** — a pool of reusable threads. Instead of "create → run → destroy", you enqueue work (`ThreadPool.QueueUserWorkItem`) and the pool dispatches it onto an idle thread. The pool grows and shrinks under CLR heuristics: on a burst it adds threads at a throttled rate (about 1 thread per 0.5 sec for CPU-bound work), and on idle it reclaims them. The number of logical cores is exposed via `Environment.ProcessorCount` — a natural hint for how much parallel CPU work makes sense at once.

It is important to distinguish two kinds of threads inside .NET infrastructure: **worker threads** (pool threads serving CPU-bound work and delegates) and **IOCP / IO threads** (dedicated to awaiting asynchronous I/O via I/O completion ports). When you `await` a genuinely asynchronous operation (socket, file, database), the calling thread is not busy — it returns to the pool, and I/O completion wakes an IO thread. This is the foundation of .NET's async throughput.

So why is `Task` better than `Thread`? A `Task` is an abstraction over a "unit of work", not over a thread. `Task.Run` schedules work onto the same ThreadPool, but adds Composition: continuations (`ContinueWith`, `await`), exception aggregation into `AggregateException`, cancellation via `CancellationToken`, a result (`Task<T>`), and cooperative configuration (`ConfigureAwait`). Spawning a raw `Thread` for short operations today is almost always a mistake — you deprive the pool of load balancing and risk exhausting threads under blocking calls. Modern guidance: use `Thread` only for rare cases (long CPU-bound loops with special stack/priority needs, a background daemon with explicit lifecycle), and use `Task` and `async/await` for everything else.

**Key concurrency traps in this lesson:** blocking a pool thread with `.Result`/`.Wait()` — leads to pool starvation and potential deadlocks under a synchronization context (e.g. UI / classic ASP.NET); running an infinite loop in `Task.Run` without a `CancellationToken` — the pool thread never returns; mixing `lock` with `await` — the lock is not released during the await and can stall the whole pool. We cover these in code and the self-check list.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — рабочий демонстрационный пример.
// Внимание: это учебный файл. Запускайте в консольном проекте `<Project Sdk="Microsoft.NET.Sdk">`.
// Цель: сравнить Thread, ThreadPool и Task и показать потокобезопасные паттерны.

using System.Collections.Concurrent;
using System.Diagnostics;

#pragma warning disable CA2007 // упрощаем демо, не вызываем ConfigureAwait в Logging

// ─────────────────────────────────────────────────────────────────────────────
// 1) Environment.ProcessorCount — естественная подсказка уровня параллелизма.
// 1) Environment.ProcessorCount — natural hint for the level of parallelism.
// ─────────────────────────────────────────────────────────────────────────────
int logicalCores = Environment.ProcessorCount;
Console.WriteLine(
    $"Логических ядер / Logical cores: {logicalCores}");

// ─────────────────────────────────────────────────────────────────────────────
// 2) Thread: foreground vs background.
//    Демонстрируем, что foreground-поток НЕ даёт процессу завершиться.
//    Demonstrate that a foreground thread keeps the process alive.
// ─────────────────────────────────────────────────────────────────────────────
static void RunThreadKinds()
{
    var foreground = new Thread(() =>
    {
        Thread.Sleep(500);
        Console.WriteLine("Foreground-поток завершился / Foreground done");
    })
    { IsBackground = false, Name = "demo-foreground" };

    var background = new Thread(() =>
    {
        Thread.Sleep(500);
        Console.WriteLine("Background-поток завершился / Background done");
    })
    { IsBackground = true, Name = "demo-background" };

    background.Start();
    foreground.Start();
    // main завершится, но процесс подождёт ТОЛЬКО foreground.
    // main returns, but the process waits ONLY for the foreground thread.
}
RunThreadKinds();

// ─────────────────────────────────────────────────────────────────────────────
// 3) ThreadPool.QueueUserWorkItem — дёшево, но без результата и без Composition.
//    ThreadPool.QueueUserWorkItem — cheap, but no result and no composition.
// ─────────────────────────────────────────────────────────────────────────────
static void QueuePoolWork()
{
    // Готовый поток берётся из пула, новый не создаётся.
    // A ready thread is taken from the pool; no new OS thread is created.
    ThreadPool.QueueUserWorkItem(_ =>
    {
        Console.WriteLine(
            $"Пул-поток / Pool thread: {Thread.CurrentThread.IsBackground}");
    });
}
QueuePoolWork();

// ─────────────────────────────────────────────────────────────────────────────
// 4) Task.Run — современный путь. Возвращает Task, поддерживает отмену и await.
//    Task.Run — modern path. Returns a Task, supports cancellation and await.
// ─────────────────────────────────────────────────────────────────────────────
static async Task<int> SumCoresAsync(CancellationToken token)
{
    // кооперативная отмена: проверяем токен внутри горячей петли.
    // cooperative cancellation: check the token inside the hot loop.
    return await Task.Run(() =>
    {
        long sum = 0;
        for (int i = 0; i < 100_000_000; i++)
        {
            token.ThrowIfCancellationRequested();
            sum += i;
        }
        return (int)(sum % int.MaxValue);
    }, token);
}

using var cts = new CancellationTokenSource();
try
{
    int r = await SumCoresAsync(cts.Token);
    Console.WriteLine($"Task результат / Task result: {r}");
}
catch (OperationCanceledException)
{
    Console.WriteLine("Task отменён / Task cancelled");
}

// ─────────────────────────────────────────────────────────────────────────────
// 5) Потокобезопасный счётчик через Interlocked (БЕЗ lock для hot path).
//    Thread-safe counter via Interlocked (NO lock on the hot path).
// ─────────────────────────────────────────────────────────────────────────────
static long CountConcurrently(int tasks)
{
    long counter = 0;
    var tasksArr = new Task[tasks];
    for (int t = 0; t < tasks; t++)
    {
        tasksArr[t] = Task.Run(() =>
        {
            for (int i = 0; i < 100_000; i++)
                // атомарный инкремент, без race condition.
                // atomic increment, no race condition.
                Interlocked.Increment(ref counter);
        });
    }
    Task.WaitAll(tasksArr);
    return counter;
}
Console.WriteLine(
    $"Счётчик / Counter: {CountConcurrently(logicalCores)} (ожидается / expected {logicalCores * 100_000})");

// ─────────────────────────────────────────────────────────────────────────────
// 6) Голодание пула: НЕ делайте так. Блокировка потока пула через Thread.Sleep
//    в Task.Run на большом числе задач — классическая ловушка.
//    Pool starvation: DO NOT do this. Blocking a pool thread via Thread.Sleep
//    inside Task.Run at scale is a classic trap.
// ─────────────────────────────────────────────────────────────────────────────
static async Task DemonstrateStarvationRisk()
{
    var sw = Stopwatch.StartNew();
    var tasks = Enumerable.Range(0, 50)
        .Select(_ => Task.Run(() => Thread.Sleep(500))) // блокировка!
        .ToArray();
    await Task.WhenAll(tasks);
    sw.Stop();
    Console.WriteLine(
        $"50 блокирующих Task.Run заняли / 50 blocking Task.Run took: {sw.ElapsedMilliseconds} мс / ms");
}
await DemonstrateStarvationRisk();

// ─────────────────────────────────────────────────────────────────────────────
// 7) ПРАВИЛЬНО: вместо блокировки — асинхронная задержка Task.Delay.
//    CORRECT: instead of blocking, use the asynchronous Task.Delay.
// ─────────────────────────────────────────────────────────────────────────────
static async Task DemonstrateNonBlocking()
{
    var sw = Stopwatch.StartNew();
    var tasks = Enumerable.Range(0, 50)
        .Select(_ => Task.Delay(500)) // поток НЕ занят
        .ToArray();
    await Task.WhenAll(tasks);
    sw.Stop();
    Console.WriteLine(
        $"50 неблокирующих Task.Delay заняли / 50 non-blocking Task.Delay took: {sw.ElapsedMilliseconds} мс / ms");
}
await DemonstrateNonBlocking();

// ─────────────────────────────────────────────────────────────────────────────
// 8) Worker vs IO threads: демонстрируем, что await освобождает поток.
//    Worker vs IO threads: show that await frees the thread.
// ─────────────────────────────────────────────────────────────────────────────
static async Task IoStyleWorkAsync()
{
    int workerBefore, ioBefore;
    ThreadPool.GetAvailableThreads(out workerBefore, out ioBefore);
    await Task.Delay(200); // имитация I/O: поток возвращается в пул.
    // imitating I/O: the thread returns to the pool.
    ThreadPool.GetAvailableThreads(out int workerAfter, out int ioAfter);
    Console.WriteLine(
        $"Доступно worker до/после / Available worker before/after: " +
        $"{workerBefore}/{workerAfter}");
}
await IoStyleWorkAsync();

// ─────────────────────────────────────────────────────────────────────────────
// 9) Ловушка: .Result в контексте синхронизации может дедлочить.
//    Trap: .Result under a synchronization context can deadlock.
//    В консоли SynchronizationContext.Current == null, поэтому здесь «прокатит»,
//    но в UI / ASP.NET classic это дедлок. Показываем безопасную альтернативу.
//    In a console SynchronizationContext.Current == null, so it "works" here,
//    but in UI / classic ASP.NET it deadlocks. Show the safe alternative.
// ─────────────────────────────────────────────────────────────────────────────
static async Task<int> GetValueAsync() => await Task.FromResult(42);

// ПЛОХО (в UI/ASP.NET classic — дедлок) / BAD (deadlock in UI/classic ASP.NET):
// int bad = GetValueAsync().Result;

// ХОРОШО — await до конца / GOOD — await to the end:
int good = await GetValueAsync();
Console.WriteLine($"Значение / Value: {good}");

// ─────────────────────────────────────────────────────────────────────────────
// 10) Ловушка: lock + await. Нельзя держать lock во время await.
//     Trap: lock + await. Never hold a lock across an await.
// ─────────────────────────────────────────────────────────────────────────────
// ПЛОХО:
// lock (gate) { await SomethingAsync(); }  // компилятор это даже запрещает,
//                                           // но SemaphoreSlim — правильный путь.

// ХОРОШО: SemaphoreSlim(1,1) — асинхронный «lock».
// GOOD: SemaphoreSlim(1,1) — an async-friendly "lock".
static SemaphoreSlim gate = new(1, 1);
static async Task CriticalSectionAsync()
{
    await gate.WaitAsync();
    try
    {
        await Task.Delay(50); // защищённая секция с await внутри.
                              // protected section with await inside.
    }
    finally
    {
        gate.Release(); // ВСЕГДА освобождаем в finally / ALWAYS release in finally.
    }
}
await Task.WhenAll(Enumerable.Range(0, 10).Select(_ => CriticalSectionAsync()));
gate.Dispose();

// ─────────────────────────────────────────────────────────────────────────────
// 11) ConcurrentQueue как неблокирующая альтернатива lock вокруг Queue<T>.
//     ConcurrentQueue as a non-blocking alternative to locking around Queue<T>.
// ─────────────────────────────────────────────────────────────────────────────
static long ProduceConsumeConcurrently()
{
    var queue = new ConcurrentQueue<int>();
    var producers = Enumerable.Range(0, 4).Select(_ => Task.Run(() =>
    {
        for (int i = 0; i < 10_000; i++) queue.Enqueue(i);
    })).ToArray();

    long total = 0;
    var consumers = Enumerable.Range(0, 4).Select(_ => Task.Run(() =>
    {
        long local = 0;
        while (queue.TryDequeue(out int v)) local += v;
        Interlocked.Add(ref total, local);
    })).ToArray();

    Task.WaitAll(producers);
    // даём потребителям добрать остаток / let consumers drain the rest.
    Task.WaitAll(consumers);
    return total;
}
Console.WriteLine(
    $"ConcurrentQueue сумма / ConcurrentQueue sum: {ProduceConsumeConcurrently()}");

Console.WriteLine("Готово / Done.");
```

#### Best Practices

- Предпочитайте `Task.Run` ручному созданию `Thread` для подавляющего большинства сценариев: пул сам балансирует нагрузку и переиспользует потоки.
- Передавайте `CancellationToken` во все длинные операции и проверяйте его в горячих циклах через `ThrowIfCancellationRequested()` — иначе поток пул может «залипнуть» навсегда.
- Никогда не блокируйте поток пула (`Thread.Sleep`, `.Result`, `.Wait()` в горячем пути) — используйте `Task.Delay`, `await`, `ConfigureAwait(false)` в библиотеках.
- Для секций с `await` используйте `SemaphoreSlim(1,1)` вместо `lock`; для простых счётчиков — `Interlocked`, а не `lock`.
- Учитывайте `Environment.ProcessorCount` как верхнюю подсказку для CPU-bound параллелизма, но не как жёсткое ограничение для I/O-работы.

- Prefer `Task.Run` over manual `Thread` creation for the vast majority of scenarios: the pool balances load and reuses threads for you.
- Pass a `CancellationToken` into every long-running operation and check it in hot loops with `ThrowIfCancellationRequested()` — otherwise a pool thread may get stuck forever.
- Never block a pool thread (`Thread.Sleep`, `.Result`, `.Wait()` on a hot path) — use `Task.Delay`, `await`, and `ConfigureAwait(false)` in libraries.
- Use `SemaphoreSlim(1,1)` instead of `lock` around `await`; use `Interlocked` rather than `lock` for simple counters.
- Treat `Environment.ProcessorCount` as an upper hint for CPU-bound parallelism, not as a hard limit for I/O-bound work.

#### Частые ошибки / Common Mistakes

- Создание `new Thread` под каждую короткую операцию → используйте `Task.Run` или `ThreadPool.QueueUserWorkItem`, чтобы пул переиспользовал потоки.
- Запуск `Thread` с `IsBackground = false` по умолчанию и забывание `Join`/остановки → процесс «зависнет» после завершения main; явно помечайте демоны `IsBackground = true` либо управляйте жизненным циклом.
- `.Result` / `.Wait()` над `Task` в UI или classic ASP.NET → дедлок из-за `SynchronizationContext`; используйте `await` до конца или `ConfigureAwait(false)` в библиотеке.
- `lock (gate) { await ... }` → компилятор это запрещает, но попытка обхода через `Monitor.Enter` + `await` — deadlock/голодание; используйте `SemaphoreSlim.WaitAsync`.
- Бесконечный цикл в `Task.Run` без проверки `CancellationToken` → поток пул никогда не вернётся; всегда принимайте и проверяйте токен.
- Использование `Queue<T>` под `lock` в producer/consumer → низкая пропускная способность и риск дедлока; используйте `ConcurrentQueue<T>` или `Channel<T>`.

- Spawning `new Thread` for every short operation → use `Task.Run` or `ThreadPool.QueueUserWorkItem` so the pool reuses threads.
- Running a `Thread` with the default `IsBackground = false` and forgetting to `Join`/stop it → the process hangs after main returns; mark daemons `IsBackground = true` explicitly or manage their lifecycle.
- `.Result` / `.Wait()` on a `Task` inside UI or classic ASP.NET → deadlock due to `SynchronizationContext`; `await` to the end or use `ConfigureAwait(false)` in a library.
- `lock (gate) { await ... }` → the compiler forbids it, but working around it with `Monitor.Enter` + `await` deadlocks or starves; use `SemaphoreSlim.WaitAsync`.
- An infinite loop in `Task.Run` without checking a `CancellationToken` → the pool thread never returns; always accept and check the token.
- Using `Queue<T>` under `lock` for a producer/consumer → low throughput and deadlock risk; use `ConcurrentQueue<T>` or `Channel<T>`.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я могу объяснить разницу между foreground и background потоком и когда процесс завершится.
- [ ] Я могу объяснить the difference between a foreground and background thread and when the process exits.
- [ ] Я понимаю, почему ThreadPool дешевле, чем создавать `Thread` вручную.
- [ ] I understand why the ThreadPool is cheaper than creating a `Thread` manually.
- [ ] Я знаю, что `Environment.ProcessorCount` возвращает число логических, а не физических ядер.
- [ ] I know `Environment.ProcessorCount` returns the count of logical, not physical, cores.
- [ ] Я передаю `CancellationToken` во все длинные операции и проверяю его в горячем цикле.
- [ ] I pass a `CancellationToken` into every long operation and check it in the hot loop.
- [ ] Я никогда не блокирую поток пула через `.Result`/`.Wait()` в контексте синхронизации.
- [ ] I never block a pool thread with `.Result`/`.Wait()` under a synchronization context.
- [ ] Я использую `SemaphoreSlim` вместо `lock` там, где есть `await`.
- [ ] I use `SemaphoreSlim` instead of `lock` wherever there is an `await`.
- [ ] Я могу назвать отличие worker-потоков от IOCP/IO-потоков и почему `await` освобождает поток.
- [ ] I can describe the difference between worker threads and IOCP/IO threads and why `await` frees the thread.

#### Ресурсы / Resources

- [Microsoft Learn — Managed Threading Basics](https://learn.microsoft.com/dotnet/standard/threading/managed-threading-basics)
- [Microsoft Learn — The Managed Thread Pool](https://learn.microsoft.com/dotnet/standard/threading/the-managed-thread-pool)
- [Microsoft Learn — Task-based Asynchronous Pattern (TAP)](https://learn.microsoft.com/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap)
- [Microsoft Learn — CancellationToken](https://learn.microsoft.com/dotnet/api/system.threading.cancellationtoken)

---

[⬆ К модулю M11](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
