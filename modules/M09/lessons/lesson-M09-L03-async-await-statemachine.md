[⬆ К модулю M09](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M09-L03: async/await, компиляция в state machine / async/await, compilation to a state machine

**Модуль / Module:** M09
**Время / Time:** 30 мин теория + 45 мин практика

---

#### Теория / Theory

`async`/`await` в C# — это не «магия потоков», а синтаксический сахар, который компилятор превращает в **конечный автомат (state machine)**. Понимание этой механики критически важно для concurrency: без него невозможно объяснить, почему `.Result` дедлочит, зачем нужен `ConfigureAwait(false)`, и как сохраняется контекст синхронизации.

**Аналогия.** Представьте чтение длинной книги в библиотеке. Вы дошли до страницы, где нужна другая книга из хранилища. Вместо того чтобы стоять у стойки и ждать (это блокировка — `.Result`), вы оставляете закладку с номером страницы и идёте читать другое. Когда хранилище принесёт книгу, библиотекарь вернёт вас к закладке. «Закладка» — это состояние state machine, «библиотекарь» — continuation, «другая книга» — другая работа в пуле потоков.

**Что делает компилятор.** Для каждого `async`-метода компилятор генерирует:
1. Структуру-конечный автомат (struct в Release, class в Debug) с полями для всех локальных переменных и параметров.
2. Поле `state` (int) — номер текущего шага (`-1` — старт/до первого await, `0..N` — точки возобновления, `-2` — завершён).
3. Поле `builder` типа `AsyncTaskMethodBuilder` (или `AsyncVoidMethodBuilder`, `AsyncValueTaskMethodBuilder`) — управляет созданием и завершением возвращаемой задачи.
4. Метод `MoveNext()`, содержащий весь оригинальный код, переставленный в `switch(state)` по точкам `await`.

**Awaitable и awaiter.** `await expr` требует, чтобы `expr` реализовывал паттерн **awaitable**: метод `GetAwaiter()`, возвращающий **awaiter** с `IsCompleted`, `GetResult()` и либо `OnCompleted(Action)`, либо `UnsafeOnCompleted(Action)` (предпочтительнее — не захватывает ExecutionContext). `Task` и `ValueTask` — стандартные awaitable. Тип `awaiter` определяет, как continuation будет запланировано.

**Контекст синхронизации (SynchronizationContext) и TaskScheduler.** Когда await завершается, continuation должно запланироваться куда-то. По умолчанию используется `SynchronizationContext.Current` (если не `null`) — например, в UI-приложениях это `DispatcherSynchronizationContext` (WPF) или `WinFormsSynchronizationContext`, возвращающий в UI-поток. В ASP.NET (classic) контекст мог вызвать дедлоки при `.Result`. В консольных и ASP.NET Core приложениях `SynchronizationContext.Current == null`, и continuation выполняется в пуле потоков через `ThreadPoolTaskScheduler`.

**Сохранение контекста.** До `await` компилятор НЕ генерирует `ConfigureAwait`. Это значит, что в библиотечном коде continuation будет пытаться вернуться в захваченный контекст — отсюда правило: **в библиотеках всегда `ConfigureAwait(false)`**, чтобы не зависеть от контекста вызывающего и не создавать скрытых дедлоков. В UI-коде `ConfigureAwait(true)` (по умолчанию) нужен, чтобы обновлять контролы.

**Потокобезопасность.** `await` сам по себе не делает код потокобезопасным: continuation может выполниться в **другом** потоке. Любое общее изменяемое состояние между `await`-точками требует защиты: `lock` (но **нельзя** `lock` поверх `await` — lock-объект удерживается одним потоком, а continuation может возобновиться в другом), `SemaphoreSlim.WaitAsync`, `Channel<T>` для pipeline, `Interlocked`/`volatile` для простых счётчиков.

**Классические ловушки.**
- `.Result` / `.Wait()` на `Task` в контексте с `SynchronizationContext` → **дедлок**: поток ждёт задачу, задача ждёт поток.
- `async void` — исключения летят в `SynchronizationContext` и крашат процесс; нельзя дождаться завершения.
- `lock (obj) { await ... }` — не компилируется, и слава богу; даже `Monitor.Enter` вручную опасен.
- `async`-метод без `await` — компилятор предупреждает; метод работает синхронно, но возвращает «уже завершённую» задачу.

**IL-структура.** В IL `async`-метод выглядит как создание state machine, инициализация `builder.Start(...)`, и return. Вся логика — в сгенерированном `MoveNext`. Это объясняет, почему `async`-методы «дешевле», чем кажутся: пока нет реального `await` на незавершённой задаче, ничего не аллоцируется (struct state machine, `ValueTask`).

---

#### Theory (EN)

`async`/`await` in C# is not “thread magic” — it is syntactic sugar that the compiler rewrites into a **finite state machine**. Understanding this rewrite is essential for concurrency: without it you cannot explain why `.Result` deadlocks, why `ConfigureAwait(false)` exists, or how a synchronization context is captured.

**Analogy.** Imagine reading a long book in a library. You reach a page that needs another book from the stacks. Instead of standing at the desk and waiting (that is blocking — `.Result`), you leave a bookmark with the page number and go read something else. When the stacks deliver the book, the librarian walks you back to the bookmark. The “bookmark” is the state machine’s state, the “librarian” is the continuation, the “something else” is other work in the thread pool.

**What the compiler does.** For each `async` method the compiler generates:
1. A state-machine struct (struct in Release, class in Debug) with fields for every local variable and parameter.
2. A `state` field (int) — the current step (`-1` = start / before first await, `0..N` = resumption points, `-2` = done).
3. A `builder` field of type `AsyncTaskMethodBuilder` (or `AsyncVoidMethodBuilder`, `AsyncValueTaskMethodBuilder`) that drives creation and completion of the returned task.
4. A `MoveNext()` method containing all the original code, reorganized into a `switch(state)` around each `await` point.

**Awaitable and awaiter.** `await expr` requires `expr` to implement the **awaitable** pattern: a `GetAwaiter()` method returning an **awaiter** with `IsCompleted`, `GetResult()`, and either `OnCompleted(Action)` or `UnsafeOnCompleted(Action)` (preferred — it does not capture the `ExecutionContext`). `Task` and `ValueTask` are the standard awaitables. The awaiter type decides how the continuation is scheduled.

**Synchronization context and TaskScheduler.** When an await completes, the continuation must be scheduled somewhere. By default the runtime uses `SynchronizationContext.Current` (if non-null) — in UI apps that is `DispatcherSynchronizationContext` (WPF) or `WinFormsSynchronizationContext`, which marshals back onto the UI thread. In classic ASP.NET the context could deadlock `.Result`. In console apps and ASP.NET Core, `SynchronizationContext.Current == null`, and continuations run on the thread pool via `ThreadPoolTaskScheduler`.

**Capturing context.** The compiler does **not** emit `ConfigureAwait` automatically. So library code that does not call `ConfigureAwait(false)` will try to resume on the caller’s captured context — hence the rule: **in libraries, always `ConfigureAwait(false)`**, to avoid depending on the caller’s context and to prevent hidden deadlocks. In UI code, `ConfigureAwait(true)` (the default) is what lets you touch controls again after an await.

**Thread safety.** `await` by itself does not make code thread-safe: a continuation may run on a **different** thread than the one before the await. Any shared mutable state across `await` points needs protection: `lock` (but **never** `lock` over `await` — the lock object is held by one thread while the continuation may resume on another), `SemaphoreSlim.WaitAsync`, `Channel<T>` for pipelines, `Interlocked`/`volatile` for simple counters.

**Classic traps.**
- `.Result` / `.Wait()` on a `Task` inside a `SynchronizationContext` → **deadlock**: the thread waits for the task, the task waits for the thread.
- `async void` — exceptions fly into the `SynchronizationContext` and may crash the process; you cannot await completion.
- `lock (obj) { await ... }` does not compile, and good thing; even manual `Monitor.Enter` over an await is dangerous.
- An `async` method without `await` produces a compiler warning; it runs synchronously and returns an already-completed task.

**IL structure.** In IL, an `async` method is reduced to constructing the state machine, calling `builder.Start(...)`, and returning. The real logic lives in the generated `MoveNext`. That is why async methods are cheaper than they look: until a real await on an incomplete task happens, nothing is heap-allocated (struct state machine, `ValueTask`).

---

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — рабочий, потокобезопасный.
// Демонстрирует: state machine поведение, ConfigureAwait, CancellationToken,
// race-free shared state через Interlocked, и pipeline на Channel<T>.
// C# 12 / .NET 8 — working, thread-safe.
// Demonstrates: state-machine behavior, ConfigureAwait, CancellationToken,
// race-free shared state via Interlocked, and a Channel<T> pipeline.

using System.Buffers;
using System.Collections.Concurrent;
using System.Runtime.CompilerServices;
using System.Threading.Channels;

namespace M09L03;

public static class AsyncStateMachineDemo
{
    // 1) Простой async-метод: покажет, как компилятор строит state machine.
    //    Каждый await = точка сохранения состояния (state field в MoveNext).
    //    1) A simple async method: shows how the compiler builds the state machine.
    //    Each await is a state-save point (the state field in MoveNext).
    public static async Task<string> FetchAndCombineAsync(
        HttpClient client,
        CancellationToken ct = default)
    {
        // ConfigureAwait(false): в библиотечном коде НЕ захватываем контекст.
        // В UI-приложении это предотвращает дедлок при .Result у вызывающего.
        // ConfigureAwait(false): library code does NOT capture the context.
        // In a UI app this prevents the caller's .Result from deadlocking.
        var first = await client.GetStringAsync("https://httpbin.org/uuid", ct)
                                  .ConfigureAwait(false);

        // После await мы МОЖЕМ оказаться в другом потоке пула —
        // не полагайтесь на thread-local состояние между await-ами.
        // After an await we MAY be on a different pool thread —
        // never rely on thread-local state between awaits.
        var second = await client.GetStringAsync("https://httpbin.org/uuid", ct)
                                  .ConfigureAwait(false);

        return string.Concat(first.AsSpan(0, 8), "+", second.AsSpan(0, 8));
    }

    // 2) CancellationToken: всегда прокидываем его ВНУТРЬ await-уемой операции.
    //    Проверка ct.IsCancellationRequested между await-ами — грязный хак,
    //    лучше бросать через ct.ThrowIfCancellationRequested в checkpoint-ах.
    //    2) CancellationToken: always thread it INTO the awaited operation.
    //    Polling ct.IsCancellationRequested between awaits is a hack;
    //    prefer ct.ThrowIfCancellationRequested at checkpoints.
    public static async Task<int> CountToAsync(int target, CancellationToken ct)
    {
        int sum = 0;
        for (int i = 0; i < target; i++)
        {
            ct.ThrowIfCancellationRequested(); // чёткая точка отмены / clean cancel point

            // имитация асинхронной работы без блокировки потока
            // simulate async work without blocking a thread
            await Task.Delay(10, ct).ConfigureAwait(false);

            // ПОТОКОБЕЗОПАСНОЕ накопление: даже если кто-то вызовет метод
            // конкурентно, Interlocked даёт атомарность без lock.
            // THREAD-SAFE accumulation: even under concurrent callers,
            // Interlocked gives atomicity without a lock.
            Interlocked.Add(ref sum, i);
        }
        return sum;
    }

    // 3) Анти-паттерн + правильная альтернатива: защита общего ресурса.
    //    НЕЛЬЗЯ: lock (obj) { await ... }  — не компилируется и логически неверно.
    //    НАДО: SemaphoreSlim(1,1) с WaitAsync — асинхронная блокировка.
    //    3) Anti-pattern + the right alternative: guarding a shared resource.
    //    FORBIDDEN: lock (obj) { await ... }  — won't compile, and is logically wrong.
    //    CORRECT:   SemaphoreSlim(1,1) with WaitAsync — an async lock.
    private static readonly SemaphoreSlim s_gate = new(1, 1);

    public static async Task<string> ReadSharedAsync(SharedBox box, CancellationToken ct)
    {
        await s_gate.WaitAsync(ct).ConfigureAwait(false);
        try
        {
            // КРИТИЧЕСКАЯ СЕКЦИЯ: только один await-able поток владеет семафором.
            // CRITICAL SECTION: only one async flow owns the semaphore.
            await Task.Delay(5, ct).ConfigureAwait(false);
            return box.Value; // гарантированно консистентное чтение / consistent read
        }
        finally
        {
            s_gate.Release(); // обязательно в finally / always in finally
        }
    }

    // 4) Как УВИДЕТЬ state machine: [AsyncMethodBuilder] и имена MoveNext.
    //    На Release компилятор использует struct — меньше аллокаций.
    //    Sharplab.io наглядно показывает сгенерированный MoveNext.
    //    4) How to SEE the state machine: [AsyncMethodBuilder] and MoveNext names.
    //    On Release the compiler uses a struct — fewer allocations.
    //    sharplab.io shows the generated MoveNext clearly.
    [AsyncMethodBuilder(typeof(PoolingAsyncValueTaskMethodBuilder<>))]
    public static async ValueTask<int> FastPathAsync(int x)
    {
        // Если условие выполнено синхронно, ValueTask НЕ аллоцирует Task —
        // state machine остаётся на стеке. Это и есть "fast path".
        // If this completes synchronously, ValueTask allocates NO Task —
        // the state machine stays on the stack. That is the "fast path".
        if (x < 0) return -1;
        await Task.CompletedTask;
        return x * 2;
    }
}

public sealed class SharedBox
{
    // volatile: гарантирует видимость записи между потоками без Interlocked.
    // Для атомарных операций чтения-изменения-записи используйте Interlocked.
    // volatile: guarantees write visibility across threads without Interlocked.
    // For atomic read-modify-write, use Interlocked instead.
    private volatile string _value = "init";
    public string Value => _value;
    public void Set(string v) => _value = v;
}

// 5) Producer/consumer pipeline на Channel<T> — основной concurrency-паттерн
//    M11-L07 (Channel). Полностью рабочий, потокобезопасный, с отменой.
//    5) Producer/consumer pipeline on Channel<T> — the core concurrency pattern
//    of M11-L07 (Channel). Fully working, thread-safe, with cancellation.
public static class PipelineDemo
{
    public static async Task RunAsync(CancellationToken ct)
    {
        // BoundedChannel: back-pressure — продюсер ждёт, если буфер полон.
        // Это предотвращает unbounded-рост памяти под нагрузкой.
        // BoundedChannel: back-pressure — the producer awaits when full.
        // This prevents unbounded memory growth under load.
        var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(capacity: 64)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = false,
            SingleWriter = false,
        });

        var producer = ProduceAsync(channel.Writer, count: 200, ct);
        var consumerA = ConsumeAsync(channel.Reader, "A", ct);
        var consumerB = ConsumeAsync(channel.Reader, "B", ct);

        // Когда продюсер заканчивает, он должен Complete канал — иначе
        // читатели будут ждать вечно и задача не завершится.
        // When the producer finishes it must Complete the channel —
        // otherwise readers wait forever and the task never completes.
        try
        {
            await producer.ConfigureAwait(false);
        }
        finally
        {
            channel.Writer.TryComplete();
        }

        // WhenAll: дожидаемся всех потребителей. Если один бросит,
        // исключение всплывёт из WhenAll (как AggregateException в .NET 8+).
        // WhenAll: wait for all consumers. If one throws, the exception
        // surfaces from WhenAll (as AggregateException on .NET 8+).
        await Task.WhenAll(consumerA, consumerB).ConfigureAwait(false);
    }

    private static async Task ProduceAsync(ChannelWriter<int> writer, int count, CancellationToken ct)
    {
        for (int i = 0; i < count; i++)
        {
            ct.ThrowIfCancellationRequested();
            // WriteAsync сам применяет back-pressure при полном канале.
            // WriteAsync applies back-pressure itself when the channel is full.
            await writer.WriteAsync(i, ct).ConfigureAwait(false);
            await Task.Delay(2, ct).ConfigureAwait(false);
        }
    }

    private static async Task ConsumeAsync(ChannelReader<int> reader, string id, CancellationToken ct)
    {
        try
        {
            // WaitToReadAsync + TryRead — стандартный цикл потребления.
            // WaitToReadAsync + TryRead — the standard consume loop.
            while (await reader.WaitToReadAsync(ct).ConfigureAwait(false))
            {
                while (reader.TryRead(out int item))
                {
                    // Имитация обработки. Ошибки изолируем, чтобы не ронять pipeline.
                    // Simulated processing. We isolate errors so one bad item
                    // does not crash the whole pipeline.
                    try
                    {
                        await ProcessAsync(item, ct).ConfigureAwait(false);
                    }
                    catch (OperationCanceledException) { throw; }
                    catch (Exception ex)
                    {
                        Console.WriteLine($"[{id}] item {item} failed: {ex.Message}");
                    }
                }
            }
        }
        catch (ChannelClosedException) { /* нормальный выход / normal exit */ }
    }

    private static async Task ProcessAsync(int item, CancellationToken ct)
    {
        await Task.Delay(5, ct).ConfigureAwait(false);
        Console.WriteLine($"[{Environment.CurrentManagedThreadId}] processed {item}");
    }
}
```

---

#### Best Practices

- В библиотечном коде всегда используйте `ConfigureAwait(false)` после каждого `await`, чтобы не захватывать контекст вызывающего и избежать скрытых дедлоков.
- Пробрасывайте `CancellationToken` во все асинхронные методы и внутрь `await`-уемых операций — не опрашивайте `IsCancellationRequested` вручную между await-ами.
- Возвращайте `ValueTask<T>` вместо `Task<T>`, когда метод часто завершается синхронно (fast path) — это убирает аллокацию `Task`.
- Для защиты общего изменяемого состояния используйте `SemaphoreSlim.WaitAsync`, `Channel<T>` или `Interlocked`; никогда не держите `lock`/`Monitor` через `await`.
- Завершайте `ChannelWriter` (вызывайте `TryComplete`) в `finally`, иначе читатели зависнут навсегда.
- В UI-коде оставляйте контекст по умолчанию (`ConfigureAwait(true)`), чтобы безопасно обращаться к элементам управления после `await`.

- In library code, always use `ConfigureAwait(false)` after each await so you do not capture the caller’s context and avoid hidden deadlocks.
- Pass a `CancellationToken` into every async method and into the awaited operations themselves — do not poll `IsCancellationRequested` by hand between awaits.
- Return `ValueTask<T>` instead of `Task<T>` when a method frequently completes synchronously (fast path) — this removes the `Task` allocation.
- To guard shared mutable state use `SemaphoreSlim.WaitAsync`, `Channel<T>`, or `Interlocked`; never hold a `lock`/`Monitor` across an `await`.
- Complete the `ChannelWriter` (call `TryComplete`) in a `finally`, otherwise readers hang forever.
- In UI code keep the default context (`ConfigureAwait(true)`) so you can safely touch controls after an await.

---

#### Частые ошибки / Common Mistakes

- **`task.Result` / `task.Wait()` в коде с `SynchronizationContext`** → дедлок: поток ждёт задачу, задача ждёт поток. **Решение:** быть `async` до конца (async all the way) либо `ConfigureAwait(false)` в глубине и `.GetAwaiter().GetResult()` только в точке входа без контекста.
- **`async void` метод** → исключения крашат процесс, нельзя дождаться. **Решение:** `async Task` (или `async Task<T>`); `async void` только для top-level event handlers.
- **`lock (obj) { await ... }`** → не компилируется, а ручной `Monitor.Enter` через await смертельно опасен (lock удерживается одним потоком, continuation — другим). **Решение:** `SemaphoreSlim.WaitAsync`.
- **`async` метод без `await`** → предупреждение CS1998, метод работает синхронно и возвращает уже-завершённую задачу. **Решение:** либо убери `async`, либо реально дождись чего-то.
- **Забыли `ConfigureAwait(false)` в библиотеке** → скрытая зависимость от контекста вызывающего; в UI — дедлоки и нагрузка на UI-поток. **Решение:** правило — в не-UI коде всегда `ConfigureAwait(false)`.
- **Игнорирование `CancellationToken`** → долгие операции нельзя отменить, висящие задачи копятся. **Решение:** принимай `CancellationToken ct = default` и пробрасывай в каждый `await`.
- **`Task.Run` для CPU-bound в UI** → ок, но `Task.Run(async () => await ...)` без CPU-работы только плодит переключения контекста. **Решение:** используй `Task.Run` только для CPU-bound, для I/O — напрямую `await`.
- **Незакрытый `ChannelWriter`** → `WaitToReadAsync` зависает навсегда. **Решение:** `channel.Writer.TryComplete()` в `finally` после продюсера.

- **`task.Result` / `task.Wait()` inside a `SynchronizationContext`** → deadlock: the thread waits for the task, the task waits for the thread. **Fix:** be `async` all the way, or use `ConfigureAwait(false)` deep down and `.GetAwaiter().GetResult()` only at an entry point with no context.
- **An `async void` method** → exceptions crash the process, and it cannot be awaited. **Fix:** `async Task` (or `async Task<T>`); reserve `async void` for top-level event handlers only.
- **`lock (obj) { await ... }`** → does not compile, and a manual `Monitor.Enter` across an await is lethal (the lock is held by one thread, the continuation by another). **Fix:** `SemaphoreSlim.WaitAsync`.
- **An `async` method with no `await`** → CS1998 warning; it runs synchronously and returns an already-completed task. **Fix:** drop `async`, or actually await something.
- **Forgetting `ConfigureAwait(false)` in a library** → a hidden dependency on the caller’s context; in UI it causes deadlocks and UI-thread load. **Fix:** rule of thumb — `ConfigureAwait(false)` in all non-UI code.
- **Ignoring the `CancellationToken`** → long operations cannot be canceled, leaked tasks accumulate. **Fix:** take `CancellationToken ct = default` and forward it to every await.
- **`Task.Run` for CPU-bound in UI** → fine, but `Task.Run(async () => await ...)` with no CPU work just thrashes contexts. **Fix:** use `Task.Run` only for CPU-bound work; for I/O, `await` directly.
- **Leaving a `ChannelWriter` open** → `WaitToReadAsync` hangs forever. **Fix:** call `channel.Writer.TryComplete()` in a `finally` after the producer.

---

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я могу объяснить, какие поля генерирует компилятор в state machine (`state`, `builder`, локальные переменные).
- [ ] Я знаю разницу между `awaitable` и `awaiter` и могу назвать методы паттерна (`GetAwaiter`, `IsCompleted`, `GetResult`, `OnCompleted`/`UnsafeOnCompleted`).
- [ ] Я понимаю, когда `SynchronizationContext.Current` равен `null` (консоль, ASP.NET Core) и когда нет (UI, classic ASP.NET).
- [ ] В каждом `await` библиотечного кода я ставлю `ConfigureAwait(false)`.
- [ ] Я никогда не вызываю `.Result`/`.Wait()` в коде, где может быть контекст синхронизации.
- [ ] Я не использую `async void` вне event-handlers.
- [ ] Я не удерживаю `lock`/`Monitor` через `await`; для асинхронной критической секции беру `SemaphoreSlim`.
- [ ] Я пробрасываю `CancellationToken` во все `await`-уемые операции.
- [ ] Я выбираю `ValueTask<T>`, когда метод часто завершается синхронно.
- [ ] Для producer/consumer я использую `Channel<T>` и обязательно `TryComplete` writer в `finally`.
- [ ] I can explain the fields the compiler generates in the state machine (`state`, `builder`, locals).
- [ ] I know the difference between an `awaitable` and an `awaiter`, and can name the pattern methods (`GetAwaiter`, `IsCompleted`, `GetResult`, `OnCompleted`/`UnsafeOnCompleted`).
- [ ] I know when `SynchronizationContext.Current` is `null` (console, ASP.NET Core) and when it is not (UI, classic ASP.NET).
- [ ] Every `await` in my library code uses `ConfigureAwait(false)`.
- [ ] I never call `.Result`/`.Wait()` in code that may have a synchronization context.
- [ ] I avoid `async void` outside event handlers.
- [ ] I never hold a `lock`/`Monitor` across an `await`; for an async critical section I use `SemaphoreSlim`.
- [ ] I forward a `CancellationToken` into every awaited operation.
- [ ] I choose `ValueTask<T>` when the method frequently completes synchronously.
- [ ] For producer/consumer I use `Channel<T>` and always `TryComplete` the writer in a `finally`.

---

#### Ресурсы / Resources

- [Microsoft Learn — async/await scenarios](https://learn.microsoft.com/dotnet/csharp/asynchronous-programming/async-scenarios)
- [Microsoft Learn — Async in depth (Task-based async)](https://learn.microsoft.com/dotnet/standard/async-in-depth)
- [Stephen Toub — «Async Await and the Generated StateMachine»](https://devblogs.microsoft.com/dotnet/async-await-and-the-generated-statemachine/)
- [Stephen Toub — ConfigureAwait FAQ](https://devblogs.microsoft.com/dotnet/configureawait-faq/)
- [sharplab.io — посмотреть сгенерированный MoveNext](https://sharplab.io)

---

[⬆ К модулю M09](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
