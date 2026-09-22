[⬆ К модулю M09](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M09-L02: Task, Task<T>, создание и ожидание / Task, Task<T>, creation and awaiting

**Модуль / Module:** M09
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

`Task` — это «обещание» (promise) результата: операция запущена, но ещё не обязательно завершена. Это фундамент всей асинхронности в .NET, пришедший на смену `Thread`/`ThreadPool.QueueUserWorkItem` и классической модели `BeginXxx/EndXxx` (APM). Где `Thread` — это тяжёлый объект операционной системы (≈1 МБ стека, переключение контекста в ядре), `Task` — лёгкая единица работы поверх пула потоков, координируемая планировщиком (`ThreadPool` + `TaskScheduler`).

**Аналогия:** заказ в ресторане. Вы не стоите у плиты, пока повар готовит стейк — вам дают номерок (`Task`). Вы можете говорить с друзьями, а когда блюдо готово — официант его приносит (`await`). `Task<T>` — это номерок, по которому выдадут конкретное блюдо (результат типа `T`); `Task` — номерок без блюда, просто подтверждение «готово».

**Способы создания:**
- `Task.Run(action)` — запускает CPU-bound работу в пуле потоков. Возвращает «горячую» (hot) задачу — она уже выполняется.
- `Task.Factory.StartNew(...)` — более низкоуровневый аналог, требует явного указания опций (`TaskCreationOptions.LongRunning`, `DenyChildAttach`); для большинства случаев предпочтительнее `Task.Run`.
- `Task.FromResult(value)` — создаёт уже завершённую задачу с результатом. Идеально для интерфейсов/моков, когда данные уже есть и реальная асинхронность не нужна.
- `Task.CompletedTask` — завершённая задача без результата (заменяет `Task.FromResult(false)` из старого кода).
- `Task.Delay(ms, ct)` — «засыпает» на заданное время, не блокируя поток; всегда передавайте `CancellationToken`.
- `async/await` — самый частый способ *потребления* задач, но внутри метода вы также *создаёте* их (`await` компилируется в конечный автомат, возвращающий `Task`/`Task<T>`).

**Ожидание (`await`):**
- Освобождает текущий поток (особенно важно в UI и ASP.NET Core), а не «висит» на нём.
- После завершения ожидаемой задачи продолжение (continuation) запускается либо в захваченном контексте синхронизации (`SynchronizationContext.Current`), либо в пуле потоков. В WPF/WinForms это означает возврат в UI-поток; в консоли и ASP.NET Core контекста нет — продолжение идёт в пуле.
- `ConfigureAwait(false)` явно отказывается от возврата в исходный контекст. В библиотечном коде это **обязательно**: иначе легко получить дедлок, если вызывающий код блокирует UI-поток через `.Result`/`.Wait()`.

**Состояния задачи** (`Task.Status`): `Created → WaitingForActivation → WaitingToRun → Running → WaitingForChildrenToComplete → RanToCompletion | Faulted | Canceled`. Понимание этих состояний критично для диагностики: `Created` у «холодной» задачи почти всегда означает ошибку (забыли `Start`), а `Faulted` говорит, что исключение нужно забрать через `await` или `.Exception`.

**Исключения** оборачиваются в `AggregateException` (при синхронном ожидании через `.Wait()`/`.Result`) или пробрасываются как исходное исключение (при `await`). Всегда обрабатывайте их: неперехваченное исключение в «забытой» задаче (fire-and-forget) поднимет `UnobservedTaskException` и в старых версиях .NET убивал процесс. Регулярно логируйте такие задачи.

**ContinueWith** — предшественник `await`. Сегодня используйте `await` почти всегда; `ContinueWith` уместен для явной композиции с `TaskContinuationOptions.OnlyOnFaulted`/`NotOnCanceled`, но будьте осторожны с `SynchronizationContext` (по умолчанию continuation не маршалируется обратно — это и плюс, и источник багов).

**Потокобезопасность:** сама `Task` неизменяема после завершения, но **разделяемое состояние**, которое задачи читают/пишут, — главное место для race condition. Используйте `lock`, `Interlocked`, `ConcurrentDictionary`, `Channel<T>` и иммутабельные структуры. Никогда не держите `lock` во время `await` — блокировка не асинхронна, и вы легко получите дедлок или сериализуете параллелизм.

#### Theory (EN)

`Task` is a *promise* of a result: an operation has been started, but it may not be finished yet. It is the cornerstone of all asynchrony in .NET, replacing raw `Thread`/`ThreadPool.QueueUserWorkItem` and the classic `BeginXxx/EndXxx` (APM) pattern. Where a `Thread` is a heavy OS object (~1 MB stack, kernel-mode context switches), a `Task` is a lightweight unit of work scheduled on top of the thread pool by a `TaskScheduler`.

**Analogy:** ordering in a restaurant. You do not stand by the stove while the chef cooks the steak — you receive a ticket (`Task`). You can chat with friends; when the dish is ready, the waiter brings it (`await`). A `Task<T>` is a ticket redeemable for a specific dish (a result of type `T`); a plain `Task` is a ticket with no dish, just a confirmation “done”.

**Ways to create a task:**
- `Task.Run(action)` — schedules CPU-bound work on the thread pool and returns a *hot* task already in flight.
- `Task.Factory.StartNew(...)` — a lower-level sibling that requires explicit options (`TaskCreationOptions.LongRunning`, `DenyChildAttach`); prefer `Task.Run` in most cases.
- `Task.FromResult(value)` — produces an already-completed task carrying a value. Great for interfaces/mocks when the data is already available.
- `Task.CompletedTask` — an already-completed task with no result (replaces `Task.FromResult(false)` from older code).
- `Task.Delay(ms, ct)` — sleeps for a duration *without* blocking a thread; always pass a `CancellationToken`.
- `async/await` — the most common way to *consume* tasks, but inside an async method you also *produce* them (`await` is compiled into a state machine returning `Task`/`Task<T>`).

**Awaiting (`await`):**
- Releases the calling thread (critical in UI and ASP.NET Core) instead of blocking it.
- When the awaited task completes, the continuation runs either on the captured synchronization context (`SynchronizationContext.Current`) or on the thread pool. In WPF/WinForms this returns you to the UI thread; console and ASP.NET Core have no such context, so the continuation runs on the pool.
- `ConfigureAwait(false)` explicitly opts out of returning to the original context. In library code this is **mandatory**: otherwise a deadlock is one `.Result`/`.Wait()` call on a UI thread away.

**Task states** (`Task.Status`): `Created → WaitingForActivation → WaitingToRun → Running → WaitingForChildrenToComplete → RanToCompletion | Faulted | Canceled`. Understanding them matters for diagnostics: a `Created` (cold) task almost always means a bug (you forgot to `Start` it), and `Faulted` means an exception is waiting to be observed via `await` or `.Exception`.

**Exceptions** are wrapped in `AggregateException` when you wait synchronously (`.Wait()`/`.Result`) or rethrown as the original exception when you `await`. Always handle them: an unhandled exception in a fire-and-forget task raises `UnobservedTaskException` and, in older .NET versions, could crash the process. Always log such tasks.

**ContinueWith** is the predecessor of `await`. Today, prefer `await` almost everywhere; `ContinueWith` remains useful for explicit composition with `TaskContinuationOptions.OnlyOnFaulted`/`NotOnCanceled`, but mind `SynchronizationContext` (by default continuations are *not* marshalled back — both a benefit and a footgun).

**Thread safety:** a `Task` itself is immutable once completed, but the *shared state* tasks read and write is where race conditions live. Use `lock`, `Interlocked`, `ConcurrentDictionary`, `Channel<T>` and immutable data. Never hold a `lock` across an `await` — the lock is not async-aware, and you will deadlock or serialize your parallelism.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8+. Полностью рабочий, потокобезопасный пример.
// Demonstrates: Task.Run, Task<T>, Task.FromResult, Task.CompletedTask,
// await, ConfigureAwait, CancellationToken, Task.Status, ContinueWith,
// exception handling, and a thread-safe shared counter.

using System;
using System.Collections.Concurrent;
using System.Threading;
using System.Threading.Tasks;

namespace M09L02;

public static class TaskBasicsDemo
{
    // Потокобезопасный счётчик через Interlocked — без lock, без race condition.
    // Thread-safe counter via Interlocked — no lock, no race condition.
    private static long _completedWork;

    public static async Task RunAsync()
    {
        using var cts = new CancellationTokenSource();
        Console.CancelKeyPress += (_, e) =>
        {
            e.Cancel = true;            // Не убивать процесс / Do not kill the process
            cts.Cancel();               // Корректно отменить задачи / Cancel tasks gracefully
        };

        // 1) Task.FromResult — данные уже есть, реальная асинхронность не нужна.
        //    Task.FromResult — data is already here, no real async needed.
        Task<int> cachedTask = Task.FromResult(42);

        // 2) Task.CompletedTask — завершённая задача без результата.
        //    Task.CompletedTask — completed task with no result.
        Task noop = Task.CompletedTask;
        await noop;

        // 3) Task.Run + CancellationToken — CPU-bound работа в пуле.
        //    Task.Run + CancellationToken — CPU-bound work on the pool.
        Task<int> computeTask = Task.Run(
            () => SumTo(1_000_000, cts.Token),
            cts.Token);

        // 4) Диагностика состояния задачи.
        //    Inspecting task status.
        Console.WriteLine($"computeTask status before await: {computeTask.Status}");

        // 5) ConfigureAwait(false) в библиотечном коде — отказ от контекста синхронизации.
        //    ConfigureAwait(false) in library code — drop the sync context.
        int sum = await computeTask.ConfigureAwait(false);
        Console.WriteLine($"Sum = {sum}, completed work = {Interlocked.Read(ref _completedWork)}");
        Console.WriteLine($"computeTask status after await: {computeTask.Status}");

        // 6) Несколько задач параллельно — не блокируем поток.
        //    Multiple tasks in parallel — no thread blocked.
        Task<int>[] workers =
        {
            Task.Run(() => SumTo(500_000, cts.Token), cts.Token),
            Task.Run(() => SumTo(500_000, cts.Token), cts.Token),
            Task.Run(() => SumTo(500_000, cts.Token), cts.Token),
        };

        // WhenAll ждёт все; исключения агрегируются в AggregateException.
        // WhenAll waits for all; exceptions are aggregated into AggregateException.
        int[] results = await Task.WhenAll(workers).ConfigureAwait(false);
        Console.WriteLine($"All workers done. Total = {results.Sum()}");

        // 7) ContinueWith — обзорный пример. Предпочитайте await в новом коде.
        //    ContinueWith — overview. Prefer await in new code.
        //    OnlyOnRanToCompletion: continuation only if the antecedent succeeded.
        _ = computeTask
            .ContinueWith(
                t => Console.WriteLine($"ContinueWith saw result {t.Result}"),
                TaskContinuationOptions.OnlyOnRanToCompletion)
            .ConfigureAwait(false);

        // 8) Обработка исключений. Faulted-задача пробрасывает исключение при await.
        //    Exception handling. A Faulted task rethrows on await.
        try
        {
            await Task.Run(() => throw new InvalidOperationException("boom"), cts.Token)
                      .ConfigureAwait(false);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Handled: {ex.GetType().Name}: {ex.Message}");
        }

        // 9) Корректная отмена — не ошибка, а штатный путь.
        //    Graceful cancellation — not an error, a normal path.
        Console.WriteLine("Cancelling remaining work…");
        cts.Cancel();
        try
        {
            await Task.Run(() => SumTo(10_000_000, cts.Token), cts.Token)
                      .ConfigureAwait(false);
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine("Work was cancelled cleanly.");
        }
    }

    /// <summary>
    /// Чистая CPU-bound функция, уважающая CancellationToken.
    /// Проверяем токен периодически, а не в каждой итерации — баланс между
    /// отзывчивостью отмены и накладными расходами.
    ///
    /// Pure CPU-bound function that respects CancellationToken.
    /// Check the token periodically, not every iteration — balance
    /// cancellation responsiveness vs overhead.
    /// </summary>
    private static int SumTo(int n, CancellationToken ct)
    {
        int sum = 0;
        for (int i = 1; i <= n; i++)
        {
            sum += i;
            if ((i & 0xFFFF) == 0)          // Каждые ~65k итераций / every ~65k iterations
            {
                ct.ThrowIfCancellationRequested();
            }
        }

        // Потокобезопасно фиксируем прогресс без lock.
        // Record progress thread-safely without lock.
        Interlocked.Increment(ref _completedWork);
        return sum;
    }

    // ⚠ Классическая ловушка — НИКОГДА так не делайте в библиотеках/UI.
    //    Classic trap — NEVER do this in libraries/UI code.
    //
    // public static int BadBlocking(string url)
    // {
    //     // .Result блокирует поток, а если внутри есть await БЕЗ ConfigureAwait(false),
    //     // continuation никогда не получит UI-поток → дедлок.
    //     // .Result blocks the thread; if the inner code awaits WITHOUT ConfigureAwait(false),
    //     // the continuation never gets the UI thread → deadlock.
    //     return FetchAsync(url).Result;
    // }

    // ✅ Правильно: быть async «до конца», либо использовать ConfigureAwait(false).
    //    Correct: stay async "all the way down", or use ConfigureAwait(false).
    public static async Task<int> FetchAsync(string url, CancellationToken ct)
    {
        await Task.Delay(50, ct).ConfigureAwait(false);  // имитация I/O / simulate I/O
        return url.Length;
    }
}
```

#### Best Practices

- В библиотечном коде всегда используйте `ConfigureAwait(false)` после каждого `await`, чтобы не зависеть от контекста синхронизации вызывающего и избежать дедлоков.
- Передавайте `CancellationToken` во все асинхронные методы и проверяйте его (`ThrowIfCancellationRequested`) в долгих циклах.
- Для CPU-bound работы используйте `Task.Run`; для I/O — `async`/`await` без `Task.Run` (иначе вы занимаете поток впустую).
- Возвращайте `Task`/`Task<T>` из публичных асинхронных методов; `async void` — только для top-level event handlers.
- Не блокируйте асинхронный код синхронно (`.Result`, `.Wait()`). Если нужно «синхронное» ожидание из legacy-кода — используйте `Task.WaitAsync` (.NET 6+) или переписывайте вызывающий код в async.
- Логируйте fire-and-forget задачи, чтобы не потерять `UnobservedTaskException`.

- In library code, always use `ConfigureAwait(false)` after every `await` to avoid depending on the caller’s sync context and to prevent deadlocks.
- Pass a `CancellationToken` into every async method and check it (`ThrowIfCancellationRequested`) inside long loops.
- Use `Task.Run` for CPU-bound work; use `async`/`await` without `Task.Run` for I/O (otherwise you waste a thread).
- Return `Task`/`Task<T>` from public async methods; reserve `async void` for top-level event handlers only.
- Do not block async code synchronously (`.Result`, `.Wait()`). When you must wait “synchronously” from legacy code, use `Task.WaitAsync` (.NET 6+) or rewrite the caller to be async.
- Log fire-and-forget tasks so `UnobservedTaskException` is not lost.

#### Частые ошибки / Common Mistakes

- **`.Result` / `.Wait()` в UI-потоке или в методе с захваченным `SynchronizationContext`** → дедлок. Избегайте: будьте async «до конца», либо используйте `ConfigureAwait(false)` в вызываемом коде.
- **`async void` вне event-handler’ов** → исключения не перехватить вызывающим, метод нельзя дождаться. Используйте `async Task`.
- **`lock` вокруг `await`** → `lock` не асинхронен: либо держите поток, либо теряете блокировку. Используйте `SemaphoreSlim(1,1)` с `await sem.WaitAsync()`.
- **`Task.Run` для чистого I/O** → занимаете поток пула без пользы. I/O и так асинхронно под капотом.
- **Забытый `CancellationToken`** → операцию нельзя остановить, UI «подвисает», тесты не отменяются.
- **`Task.Factory.StartNew` без `DenyChildAttach` и с дефолтным планировщиком** → сюрпризы с вложенными задачами и контекстом. Чаще нужен просто `Task.Run`.
- **Незахваченное исключение fire-and-forget** → `UnobservedTaskException` и потенциальный краш (в старых .NET).

- **`.Result` / `.Wait()` on a UI thread or inside a method with a captured `SynchronizationContext`** → deadlock. Avoid it: stay async “all the way down”, or use `ConfigureAwait(false)` in the callee.
- **`async void` outside an event handler** → exceptions can’t be caught by the caller, the method can’t be awaited. Use `async Task`.
- **`lock` around an `await`** → `lock` is not async-aware: you either hold the thread or lose the lock. Use `SemaphoreSlim(1,1)` with `await sem.WaitAsync()`.
- **`Task.Run` for pure I/O** → you occupy a pool thread for nothing. I/O is already async under the hood.
- **Forgotten `CancellationToken`** → the operation can’t be stopped, the UI “freezes”, tests can’t be cancelled.
- **`Task.Factory.StartNew` without `DenyChildAttach` and with the default scheduler** → surprises with nested tasks and context. Usually you just want `Task.Run`.
- **An unobserved fire-and-forget exception** → `UnobservedTaskException` and a potential crash (in older .NET).

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я знаю разницу между `Task` и `Task<T>` и когда использовать каждый.
- [ ] Я могу создать задачу через `Task.Run`, `Task.FromResult`, `Task.CompletedTask` и `async`/`await`.
- [ ] Я понимаю, что `await` освобождает поток, а не блокирует его.
- [ ] Я всегда передаю `CancellationToken` и проверяю его в долгих циклах.
- [ ] В библиотечном коде я ставлю `ConfigureAwait(false)`.
- [ ] Я никогда не блокирую асинхронный код через `.Result`/`.Wait()`.
- [ ] Я не использую `async void` вне event handlers.
- [ ] Я не держу `lock` во время `await` (использую `SemaphoreSlim`).
- [ ] Я знаю состояния `Task.Status` и умею диагностировать `Faulted`/`Canceled`.
- [ ] Я обрабатываю исключения задач и логирую fire-and-forget.

- [ ] I know the difference between `Task` and `Task<T>` and when to use each.
- [ ] I can create a task with `Task.Run`, `Task.FromResult`, `Task.CompletedTask` and `async`/`await`.
- [ ] I understand that `await` releases the thread rather than blocking it.
- [ ] I always pass a `CancellationToken` and check it inside long loops.
- [ ] In library code I apply `ConfigureAwait(false)`.
- [ ] I never block async code with `.Result`/`.Wait()`.
- [ ] I avoid `async void` outside event handlers.
- [ ] I never hold a `lock` across an `await` (I use `SemaphoreSlim`).
- [ ] I know the `Task.Status` values and can diagnose `Faulted`/`Canceled`.
- [ ] I handle task exceptions and log fire-and-forget tasks.

#### Ресурсы / Resources

- [Microsoft Learn — Task](https://learn.microsoft.com/dotnet/api/system.threading.tasks.task)
- [Microsoft Learn — Task&lt;TResult&gt;](https://learn.microsoft.com/dotnet/api/system.threading.tasks.task-1)
- [Microsoft Learn — Task.Run](https://learn.microsoft.com/dotnet/api/system.threading.tasks.task.run)
- [Microsoft Learn — ConfigureAwait](https://learn.microsoft.com/dotnet/api/system.threading.tasks.task.configureawait)
- [Stephen Toub — «Should I expose asynchronous wrappers for synchronous methods?»](https://devblogs.microsoft.com/dotnet/should-i-expose-asynchronous-wrappers-for-synchronous-methods/)
- [Stephen Cleary — «Don’t Block on Async Code»](https://blog.stephencleary.com/2012/07/dont-block-on-async-code.html)

---

[⬆ К модулю M09](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
