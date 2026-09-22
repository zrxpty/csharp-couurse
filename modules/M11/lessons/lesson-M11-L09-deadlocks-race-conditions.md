[⬆ К модулю M11](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M11-L09: Deadlocks/race conditions, диагностика / Deadlocks/race conditions, diagnostics

**Модуль / Module:** M11
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Представь, что два человека сидят за столом и каждому нужны одновременно вилка и нож, чтобы поесть. Но на столе только одна вилка и один нож. Первый человек хватает вилку и ждёт нож. Второй хватает нож и ждёт вилку. Никто не уступает — оба голодны вечно. Это и есть **deadlock** (взаимная блокировка): каждый держит ресурс, нужный другому, и ждёт ресурс, который держит другой. В .NET это чаще всего возникает с `lock` (Monitor), `SemaphoreSlim`, `Mutex` и ручными `WaitHandle`.

**Race condition** — более тонкая проблема. Это ситуация, когда корректность программы зависит от *порядка* выполнения потоков, а этот порядок не контролируется. Аналогия: два кассира одновременно читают баланс счёта «100 ₽», оба вычитают по 10 ₽ и записывают «90 ₽». Итог — один из платежей «пропал». Конкретный подвид race condition — **data race**: два и более потока обращаются к одной переменной, минимум один — на запись, и нет синхронизации. В C# даже `i++` не атомарен для `int` без `Interlocked`.

Различай четыре классических проблемы:
- **Race condition** — логическая ошибка из-за порядка выполнения.
- **Data race** — одновременный небезопасный доступ к памяти.
- **Deadlock** — потоки навсегда ждут ресурсы друг друга.
- **Live-lock** — потоки активны, но бесполезно меняют состояние в ответ на действия друг друга и не могут продвинуться (как два человека, которые одновременно уступают дорогу друг другу и снова сталкиваются).

**Deadlock через lock ordering** — самая частая причина. Если метод A сначала берёт `lock1`, затем `lock2`, а метод B — сначала `lock2`, затем `lock1`, то при неудачном тайминге A держит `lock1` и ждёт `lock2`, а B держит `lock2` и ждёт `lock1`. Правило: **всегда берите локи в одинаковом порядке** во всём коде. Альтернативы — `Monitor.TryEnter` с таймаутом, единый крупный лок (если конфликты редки), или бессинхронизационные структуры: `ConcurrentDictionary`, `Channel<T>`, `ImmutableList<T>`.

**Диагностика** — половина победы. Инструменты .NET:
- `Debug.Assert` — ловит нарушение инвариантов в Debug-сборке, дешёвый «первый эшелон». В Release компилируется «в ноль».
- `dotnet-counters` — живая статистика без оверхеда: `Monitor Lock Contention/sec`, `ThreadPool Thread Count`, `Time in GC`. Запуск: `dotnet-counters monitor --process-id <pid> System.Runtime`.
- `Concurrency Visualizer` (расширение VS / `dotnet-trace` + `speedscope`) — таймлайн потоков: видно, где поток блокируется, кого ждёт, сколько простаивает.
- `dotnet-dump` + `lldb`/WinDbg — для посмертного анализа: команда `~` (потоки), `!syncblk` (мониторы), `!clrstack` (стеки).

**Как воспроизвести** баги concurrency — отдельное искусство. Подсказки: запускай тест под нагрузкой (N потоков в цикле), добавляй `Thread.SpinWait`/`Task.Delay` между чтением и записью, чтобы «расширить окно» race; используй `dotNetFizzBuzz`/`CoCoL`-подобные noise injection; повторяй千百 раз — race проявляется статистически. Для deadlock — уменьши пул потоков (`ThreadPool.SetMinThreads`) или добавь задержки перед вторым локом. Никогда не «чините» тест, добавив `Thread.Sleep` в продакшн — это маска, а не лекарство. Лекарство — правильная синхронизация или её устранение через иммутабельные структуры и `Channel`.

#### Theory (EN)

Imagine two diners at a table. Each needs both a fork and a knife to eat, but the table has only one of each. Diner A grabs the fork and waits for the knife; diner B grabs the knife and waits for the fork. Neither yields. Both starve forever. That is a **deadlock**: each thread holds a resource the other needs and waits for a resource the other holds. In .NET the usual suspects are `lock` (Monitor), `SemaphoreSlim`, `Mutex`, and manual `WaitHandle`s.

A **race condition** is subtler. It is any situation where correctness depends on the *timing* of threads, and that timing is not controlled. Analogy: two clerks simultaneously read a balance of "$100", each subtracts $10, each writes back "$90". One withdrawal silently disappeared. A specific sub-case is a **data race**: two or more threads access the same memory location, at least one writes, and there is no synchronization. Even `i++` on a plain `int` is not atomic in C# — you need `Interlocked.Increment`.

Distinguish four classic problems:
- **Race condition** — a logical bug caused by execution order.
- **Data race** — simultaneous, unsynchronized memory access.
- **Deadlock** — threads wait forever for each other's resources.
- **Live-lock** — threads keep running but only react to each other and make no progress, like two people perpetually stepping aside at the same moment and bumping again.

**Deadlock via lock ordering** is the most common cause. If method A locks `lock1` then `lock2`, while method B locks `lock2` then `lock1`, a bad schedule gives A holding `lock1` waiting for `lock2`, and B holding `lock2` waiting for `lock1`. The rule: **always acquire locks in the same order** everywhere. Alternatives are `Monitor.TryEnter` with a timeout, one coarse lock (when contention is rare), or lock-free structures: `ConcurrentDictionary`, `Channel<T>`, `ImmutableList<T>`.

**Diagnostics** is half the battle. The .NET toolbox:
- `Debug.Assert` — catches invariant violations in Debug builds; a cheap first line of defense, compiled out in Release.
- `dotnet-counters` — live, low-overhead stats: `Monitor Lock Contention/sec`, `ThreadPool Thread Count`, `Time in GC`. Run: `dotnet-counters monitor --process-id <pid> System.Runtime`.
- `Concurrency Visualizer` (VS extension / `dotnet-trace` + `speedscope`) — per-thread timeline showing where a thread blocks, what it waits for, how long it idles.
- `dotnet-dump` + `lldb`/WinDbg — post-mortem: `~` (threads), `!syncblk` (monitors), `!clrstack` (stacks).

**Reproducing** concurrency bugs is its own craft. Tips: run the test under load (N threads in a loop); inject `Thread.SpinWait`/`Task.Delay` between read and write to "widen the window" of a race; repeat thousands of times — races show up statistically. For deadlocks, shrink the thread pool (`ThreadPool.SetMinThreads`) or insert delays before the second lock. Never "fix" a flaky test by sprinkling `Thread.Sleep` into production code — that is a mask, not a cure. The cure is correct synchronization, or eliminating synchronization entirely via immutable data and `Channel`.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Deadlocks, race conditions, diagnostics
// Двуязычные комментарии: RU + EN

using System.Collections.Concurrent;
using System.Diagnostics;
using System.Runtime.CompilerServices;
using System.Threading.Channels;

namespace M11L09.ConcurrencyDiagnostics;

// === 1) DATA RACE: что НЕЛЬЗЯ делать / what NOT to do ===
public sealed class BrokenCounter
{
    private int _value; // разделяемое поле без синхронизации / shared field, no sync

    public void IncrementBad()
    {
        // _value++ не атомарно: это чтение + сложение + запись.
        // Two threads can both read 5, both write 6 -> lost update.
        _value++;
    }
}

// === 2) ПРАВИЛЬНО: Interlocked для простых счётчиков / correct: Interlocked ===
public sealed class SafeCounter
{
    private int _value;

    public int Value => Volatile.Read(ref _value); // публикация с барьером / publish with barrier

    public void Increment() => Interlocked.Increment(ref _value);
    public void Decrement() => Interlocked.Decrement(ref _value);
}

// === 3) DEADLOCK через нарушение порядка локов / deadlock via lock-ordering ===
// НЕ ИСПОЛЬЗОВАТЬ В ПРОДАКШНЕ — только как демонстрация бага.
public sealed class DeadlockProne
{
    private readonly object _lockA = new();
    private readonly object _lockB = new();

    public void FirstThenSecond()
    {
        lock (_lockA)      // берём A, потом B / take A then B
            lock (_lockB)
            {
                /* work */
            }
    }

    public void SecondThenFirst()
    {
        lock (_lockB)      // берём B, потом A — ОБРАТНЫЙ порядок! / B then A: reversed!
            lock (_lockA)
            {
                /* work */
            }
    }
}

// === 4) ЛЕЧЕНИЕ: фиксированный порядок локов / cure: fixed lock ordering ===
public sealed class OrderedLocks
{
    private readonly object _lockA = new();
    private readonly object _lockB = new();

    // Соглашение: всегда A -> B. / Convention: always A -> B.
    public void MethodOne()
    {
        lock (_lockA)
        lock (_lockB)
        {
            /* work */
        }
    }

    public void MethodTwo()
    {
        lock (_lockA)   // тот же порядок / same order
        lock (_lockB)
        {
            /* work */
        }
    }
}

// === 5) ЛЕЧЕНИЕ: TryEnter с таймаутом + диагностика / TryEnter with timeout ===
public sealed class TimeoutSafeTransfer
{
    private readonly object _a = new();
    private readonly object _b = new();
    private readonly TimeSpan _timeout = TimeSpan.FromMilliseconds(500);

    public bool Move(bool reverse)
    {
        var first = _a;
        var second = _b;
        if (reverse) (first, second) = (second, first); // намеренно нарушаем порядок для демо

        bool gotFirst = false, gotSecond = false;
        try
        {
            Monitor.TryEnter(first, _timeout, ref gotFirst);
            if (!gotFirst) return false; // не зависаем / never block forever

            Monitor.TryEnter(second, _timeout, ref gotSecond);
            if (!gotSecond)
            {
                Debug.Assert(false, "Не удалось взять второй лок — возможен deadlock-риск. " +
                                     "Failed to acquire second lock — deadlock risk.");
                return false;
            }

            // критическая секция / critical section
            return true;
        }
        finally
        {
            if (gotSecond) Monitor.Exit(second);
            if (gotFirst) Monitor.Exit(first);
        }
    }
}

// === 6) Рабочий producer/consumer на Channel (без локов) / lock-free pipeline ===
public sealed class Pipeline : IAsyncDisposable
{
    // Очередь-канал: потокобезопасна, без явных локов / thread-safe channel, no explicit locks
    private readonly Channel<int> _channel = Channel.CreateBounded<int>(capacity: 100);
    private readonly CancellationTokenSource _cts = new();
    private readonly Task _consumer;

    public Pipeline()
    {
        _consumer = Task.Run(() => ConsumeAsync(_cts.Token));
    }

    // Producer пишет в канал; AwaitCompletion не нужен — WriteAsync сам awaits места.
    public async ValueTask ProduceAsync(int item, CancellationToken ct)
    {
        // ConfigureAwait(false): не цепляемся к контексту синхронизации UI/ASP.NET Classic.
        // ConfigureAwait(false): avoid capturing a sync context that can deadlock with .Result.
        await _channel.Writer.WriteAsync(item, ct).ConfigureAwait(false);
    }

    private async Task ConsumeAsync(CancellationToken ct)
    {
        await foreach (var item in _channel.Reader.ReadAllAsync(ct).ConfigureAwait(false))
        {
            // обработка одной сущности / process one item
            Process(item);
        }
    }

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    private static void Process(int item) => Volatile.Write(ref _dummy, item);

    private static int _dummy;

    public async ValueTask DisposeAsync()
    {
        _channel.Writer.Complete();                    // сигнал: больше нет данных / no more data
        await _consumer.ConfigureAwait(false);         // дождаться потребителя / drain consumer
        _cts.Cancel();
        _cts.Dispose();
    }
}

// === 7) Live-lock пример / live-lock example ===
// Два потока бесконечно уступают друг другу, не прогрессируя.
public sealed class LiveLockDemo
{
    private volatile bool _aWants = true;
    private volatile bool _bWants = true;

    public async Task RunAsync(CancellationToken ct)
    {
        var taskA = Task.Run(() => Worker("A", () => _aWants, v => _aWants = v,
                                          () => _bWants, v => _bWants = v), ct);
        var taskB = Task.Run(() => Worker("B", () => _bWants, v => _bWants = v,
                                          () => _aWants, v => _aWants = v), ct);
        await Task.WhenAll(taskA, taskB).ConfigureAwait(false);
    }

    private static async Task Worker(string name,
                                     Func<bool> wantGet, Action<bool> wantSet,
                                     Func<bool> otherGet, Action<bool> otherSet)
    {
        int attempts = 0;
        while (wantGet() && attempts < 1_000)
        {
            // Если другой тоже хочет — уступаем и пробуем снова. / If the other also wants it, yield.
            if (otherGet())
            {
                wantSet(false);
                await Task.Delay(1).ConfigureAwait(false);
                wantSet(true);
            }
            attempts++;
        }
        // Cure: вводим случайную задержку (backoff) или арбитр. / Fix: random backoff or arbiter.
    }
}

// === 8) Демонстрация: воспроизведение race / reproducing a race ===
public static class RaceReproducer
{
    // Тест, который часто ловит баг / test that frequently catches the bug.
    public static (int expected, int actual) RunBad(int threads, int perThread)
    {
        var c = new BrokenCounter();
        Parallel.For(0, threads, _ =>
        {
            for (int i = 0; i < perThread; i++) c.IncrementBad();
        });
        return (threads * perThread, c._value); // поле открыто только ради демо / exposed for demo
    }

    public static (int expected, int actual) RunSafe(int threads, int perThread)
    {
        var c = new SafeCounter();
        Parallel.For(0, threads, _ =>
        {
            for (int i = 0; i < perThread; i++) c.Increment();
        });
        return (threads * perThread, c.Value);
    }
}

// === 9) Debug.Assert для инвариантов / Debug.Assert for invariants ===
public sealed class Account
{
    private int _balance;
    private readonly object _gate = new();

    public int Balance
    {
        get { lock (_gate) return _balance; }
    }

    public void Deposit(int amount)
    {
        lock (_gate)
        {
            Debug.Assert(amount >= 0, "Сумма депозита не может быть отрицательной. " +
                                       "Deposit amount cannot be negative.");
            int previous = _balance;
            _balance += amount;
            // Инвариант: баланс не может стать отрицательным. / Invariant: balance stays non-negative.
            Debug.Assert(_balance >= previous, "Balance must be monotonic for deposits.");
        }
    }
}

// === 10) Анти-паттерн: .Result + lock на await — что НЕ делать / what NOT to do ===
public sealed class AntiPatterns
{
    // ❌ deadlock в UI/ASP.NET Classic: продолжение ждёт контекста, который занят .Result.
    //    Deadlock in UI/ASP.NET Classic: continuation waits for context blocked by .Result.
    public int BadBlocking(Task<int> task) => task.Result;

    // ❌ lock + await: лок удерживается во время await, блокируя других.
    //    lock + await: lock held across await blocks others; also not async-aware.
    public async Task BadLockOnAwaitAsync()
    {
        lock (_gate) // компилятор запретит, но SemaphoreSlim тоже часто ошибочно держат
        {
            await Task.Delay(1).ConfigureAwait(false);
        }
    }
    private readonly object _gate = new();

    // ✅ Правильно: SemaphoreSlim(1,1) + await внутри. / Correct: SemaphoreSlim(1,1), await inside.
    private readonly SemaphoreSlim _asyncGate = new(1, 1);
    public async Task GoodAsyncGateAsync()
    {
        await _asyncGate.WaitAsync().ConfigureAwait(false);
        try
        {
            await Task.Delay(1).ConfigureAwait(false);
        }
        finally
        {
            _asyncGate.Release();
        }
    }
}
```

#### Best Practices
- Всегда берите несколько локов в одном и том же порядке во всём коде; зафиксируйте порядок в комментарии или в структуре данных.
- Предпочитайте бессинхронизационные структуры (`ConcurrentDictionary`, `Channel<T>`, иммутабельные коллекции, `Interlocked`) ручным `lock` — меньше шансов на дедлок.
- Используйте `ConfigureAwait(false)` в библиотечном коде, чтобы не зависеть от контекста синхронизации и не плодить дедлоки через `.Result`.
- В async-коде заменяйте `lock` на `SemaphoreSlim.WaitAsync`; никогда не держите обычный `lock` через `await`.
- Ловите инварианты через `Debug.Assert` в Debug-сборке — это почти бесплатно и ловит race на раннем этапе.
- Под нагрузкой мониторьте `Monitor Lock Contention/sec` через `dotnet-counters` — рост contention = сигнал к пересмотру блокировок.

#### Best Practices (EN)
- Always acquire multiple locks in the same order everywhere; encode the order in a comment or in the data structure.
- Prefer lock-free structures (`ConcurrentDictionary`, `Channel<T>`, immutable collections, `Interlocked`) over manual `lock` — fewer deadlock chances.
- Use `ConfigureAwait(false)` in library code to avoid sync-context dependence and `.Result`-style deadlocks.
- Replace `lock` with `SemaphoreSlim.WaitAsync` in async code; never hold a `lock` across an `await`.
- Catch invariants with `Debug.Assert` in Debug builds — nearly free, catches races early.
- Under load, monitor `Monitor Lock Contention/sec` via `dotnet-counters` — rising contention means it is time to rethink locking.

#### Частые ошибки / Common Mistakes
- `i++` на разделяемом `int` без синхронизации → используйте `Interlocked.Increment` или `lock`.
- Обратный порядок локов в двух методах → зафиксируйте единый порядок или используйте `Monitor.TryEnter` с таймаутом.
- `.Result`/`.Wait()` на async-методе в UI/ASP.NET Classic → deadlock; делайте метод полностью async или используйте `ConfigureAwait(false)` + `Task.Run` для «вырывания» из контекста.
- `lock`, удерживаемый через `await` (или ручной `lock` + `await` в теле) → замените на `SemaphoreSlim(1,1)`.
- `async void` для обработки событий без try/catch → исключение роняет процесс; используйте `async Task` и логируйте.
- Чинить плавающий тест через `Thread.Sleep` в продакшн-коде → это маскировка; исправьте синхронизацию или используйте `Channel`/иммутабельные данные.
- Забыть `Volatile.Read`/`Volatile.Write` для флага видимости между потоками → другой поток может не увидеть обновление; используйте `volatile` или `Volatile`.

#### Common Mistakes (EN)
- `i++` on a shared `int` without synchronization → use `Interlocked.Increment` or `lock`.
- Reversed lock order in two methods → fix a single order, or use `Monitor.TryEnter` with a timeout.
- `.Result`/`.Wait()` on an async method under UI/ASP.NET Classic → deadlock; make the call fully async or use `ConfigureAwait(false)` + `Task.Run` to escape the context.
- `lock` held across `await` (or a manual `lock` + `await` body) → replace with `SemaphoreSlim(1,1)`.
- `async void` for event handling without try/catch → an unhandled exception kills the process; use `async Task` and log.
- Fixing a flaky test with `Thread.Sleep` in production code → it masks the bug; fix the synchronization or use `Channel`/immutable data.
- Forgetting `Volatile.Read`/`Volatile.Write` for a cross-thread visibility flag → the other thread may never see the update; use `volatile` or `Volatile`.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Все разделяемые mutable-поля защищены (`lock`, `Interlocked`, `Concurrent*`, `Channel`).
- [ ] Порядок захвата нескольких локов одинаков во всём коде.
- [ ] Нигде нет `.Result`/`.Wait()` на async-коде в контексте синхронизации.
- [ ] Нет `lock` через `await`; async-критические секции используют `SemaphoreSlim`.
- [ ] `ConfigureAwait(false)` стоит в библиотечных методах.
- [ ] `CancellationToken` пробрасывается во все async-операции и в `Channel.Writer.WriteAsync`.
- [ ] `Debug.Assert` защищает ключевые инварианты (баланс, монотонность, непересечение).
- [ ] В CI запускается стресс-тест (N потоков × M итераций) для выявления race статистически.
- [ ] `dotnet-counters` показывает стабильный `Monitor Lock Contention/sec` под нагрузкой.
- [ ] Воспроизведённый баг фиксируется кодом, а не `Thread.Sleep`.

#### Self-check Checklist (EN)
- [ ] All shared mutable fields are protected (`lock`, `Interlocked`, `Concurrent*`, `Channel`).
- [ ] The order of acquiring multiple locks is consistent everywhere.
- [ ] No `.Result`/`.Wait()` on async code under a sync context.
- [ ] No `lock` across `await`; async critical sections use `SemaphoreSlim`.
- [ ] `ConfigureAwait(false)` is applied in library methods.
- [ ] `CancellationToken` is propagated to every async operation and `Channel.Writer.WriteAsync`.
- [ ] `Debug.Assert` guards key invariants (balance, monotonicity, disjointness).
- [ ] A stress test (N threads × M iterations) runs in CI to catch races statistically.
- [ ] `dotnet-counters` shows stable `Monitor Lock Contention/sec` under load.
- [ ] The reproduced bug is fixed in code, not with `Thread.Sleep`.

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/dotnet/standard/threading/managed-threading-basics](https://learn.microsoft.com/dotnet/standard/threading/managed-threading-basics)
- [dotnet-counters — https://learn.microsoft.com/dotnet/core/diagnostics/dotnet-counters](https://learn.microsoft.com/dotnet/core/diagnostics/dotnet-counters)
- [Concurrency Visualizer — https://learn.microsoft.com/visualstudio/profiling/concurrency-visualizer](https://learn.microsoft.com/visualstudio/profiling/concurrency-visualizer)
- [Channel<T> — https://learn.microsoft.com/dotnet/api/system.threading.channels.channel-1](https://learn.microsoft.com/dotnet/api/system.threading.channels.channel-1)

---

[⬆ К модулю M11](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
