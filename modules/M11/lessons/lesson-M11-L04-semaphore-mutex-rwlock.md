[⬆ К модулю M11](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M11-L04: SemaphoreSlim, Mutex, ReaderWriterLockSlim / SemaphoreSlim, Mutex, ReaderWriterLockSlim

**Модуль / Module:** M11
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

В многопоточной среде часто возникает задача не просто защитить общий участок памяти, а управлять **доступом к ресурсу по более сложным правилам**: ограничить количество одновременных клиентов, синхронизировать процессы целиком, либо разрешить многим читателям работать параллельно, но запретить им мешать писателю. Для этих сценариев стандартные `lock`/`Monitor` не подходят — нужны специализированные примитивы: `SemaphoreSlim`, `Mutex` и `ReaderWriterLockSlim`.

**SemaphoreSlim** — это «счётчик разрешений». Представьте турникет в метро, который пропускает не одного, а сразу N человек одновременно. Каждый `Wait()` уменьшает счётчик на единицу, каждый `Release()` — увеличивает. Когда счётчик достигает нуля, следующий вызов `Wait()` блокируется, пока кто-то не вернёт разрешение через `Release()`. Это идеальный инструмент для **ограничения степени параллелизма** — например, чтобы не «положить» базу данных сотней одновременных подключений или ограничить число конкурентных HTTP-запросов к внешнему API. Префикс `Slim` означает, что примитив использует ожидание занятого цикла (spin-wait) для коротких задержек, прежде чем перейти на дорогостоящее ожидание через `Monitor` — он не использует встроенные объекты синхронизации ОС и потому работает только внутри одного процесса, зато дешевле `Semaphore`.

Критически важная особенность `SemaphoreSlim` для асинхронного мира — метод `WaitAsync()`. Он возвращает `Task` и **не блокирует поток** на время ожидания, в отличие от синхронного `Wait()`. В async-приложениях (ASP.NET Core, WinUI, Blazor) блокировать поток пула — это антипаттерн, ведущий к **thread pool starvation**: пул быстро исчерпывается, и приложение «зависает» под нагрузкой. Поэтому правило простое: в async-коде всегда используйте `await semaphore.WaitAsync(ct)`, никогда — `semaphore.Wait()`.

**Mutex** (mutual exclusion) решает другую задачу: взаимное исключение **между процессами**. В отличие от `lock`, который живёт только в рамках одного процесса и AppDomain, `Mutex` можно именовать и сделать доступным для нескольких процессов ОС. Это позволяет, например, гарантировать, что запущен только **один экземпляр приложения** (single-instance), или синхронизировать доступ к разделяемому файлу между разными программами. Именованный `Mutex` создаётся через `new Mutex(false, "Global\\MyApp.Mutex")` — префикс `Global\` делает его доступным даже через терминальные сессии. Важная тонкость: `Mutex` подчиняется **реентерабельности** — поток, уже владеющий мьютексом, может войти в него повторно, но обязан вызвать `ReleaseMutex()` столько же раз. И ещё: ожидание на `Mutex` не прерывается `CancellationToken` напрямую — для отменяемого ожидания используют `WaitHandle.SignalAndWait` или `Task.Run` + `mutex.WaitOne()` с периодической проверкой.

**ReaderWriterLockSlim** вводит различие между **читателями** и **писателями**. Большинство данных читают часто, а пишут редко. Использовать эксклюзивный `lock` на каждое чтение — значит искусственно сериализовать читателей, теряя параллелизм. RW-блокировка решает это: несколько потоков могут одновременно войти в режим чтения (`EnterReadLock`), но запись (`EnterWriteLock`) эксклюзивна — пока пишет один, никто не читает и никто не пишет. Дополнительно есть режим обновления (`EnterUpgradeableReadLock`) — поток сначала читает, а при необходимости атомарно «повышается» до писателя, не отпуская блокировку и не давая возникнуть race condition «прочитал-отпустил-перечитал». Этот примитив тоньше, чем кажется: он подвержен дедлокам при неправильном повышении и имеет параметры `LockRecursionPolicy`, поэтому всегда используйте `try/finally` и `ExitReadLock`/`ExitWriteLock` ровно столько раз, сколько входов было сделано. Начиная с .NET 8 в простых сценариях стоит также сравнивать с `Lock` (новый тип) и `Channel<T>` — они покрывают часть случаев чище.

#### Theory (EN)

In a multithreaded world we frequently need more than a plain critical section. We need to **limit how many callers may use a resource at once**, to synchronize **whole processes** rather than threads, or to let **many readers run in parallel while a single writer is exclusive**. The ordinary `lock`/`Monitor` cannot express any of these rules, so the framework ships three specialized primitives: `SemaphoreSlim`, `Mutex`, and `ReaderWriterLockSlim`.

**SemaphoreSlim** is a **counting permit holder**. Picture a turnstile that admits not one person at a time but N people at once. Each `Wait()` decrements the counter by one; each `Release()` increments it. When the counter hits zero, the next `Wait()` blocks until somebody returns a permit. This makes it the canonical tool for **bounding concurrency** — for instance, capping the number of simultaneous database connections or throttling concurrent outbound HTTP requests to a flaky third-party API. The `Slim` suffix means it first spin-waits for short contention and only then falls back to a `Monitor`-based wait; it never allocates an OS kernel object, so it is **intra-process only**, but cheaper than the full `Semaphore`.

The async-aware method `WaitAsync()` is the single most important feature of `SemaphoreSlim` for modern code. It returns a `Task` and **does not hold a thread** while it waits, unlike the synchronous `Wait()`. In async applications (ASP.NET Core, Blazor, desktop UIs), blocking a thread-pool thread is a serious anti-pattern that causes **thread pool starvation**: the pool exhausts, new work queues up, and the application appears frozen under load. The rule is simple: in async code always write `await semaphore.WaitAsync(ct)`, never `semaphore.Wait()`.

**Mutex** (mutual exclusion) targets a different problem: **cross-process** exclusion. A `lock` lives only inside a single process and AppDomain, but a `Mutex` can be **named** and shared across OS processes. That lets you enforce a **single-instance application** or synchronize access to a shared file between two independent programs. A named mutex is created with `new Mutex(false, "Global\\MyApp.Mutex")` — the `Global\` prefix makes it visible even across terminal-server sessions. Two subtleties matter: a `Mutex` is **reentrant**, so a thread that already owns it may re-enter, but must call `ReleaseMutex()` exactly as many times; and `WaitOne()` is **not directly cancellable** by `CancellationToken`, so cancellable waits use `WaitHandle.SignalAndWait` or a `Task.Run` wrapper that polls.

**ReaderWriterLockSlim** distinguishes **readers** from **writers**. Most state is read far more often than it is written, so taking an exclusive `lock` on every read serializes readers needlessly and wastes parallelism. The RW lock fixes that: many threads may enter read mode (`EnterReadLock`) concurrently, but a write (`EnterWriteLock`) is exclusive — while one writer holds the lock, nobody reads and nobody writes. An **upgradeable** mode (`EnterUpgradeableReadLock`) lets a thread start as a reader and atomically promote itself to a writer without releasing the lock, avoiding the read-release-reread race. This primitive is subtler than it looks: careless upgrades deadlock, and you must match every `Enter*` with an `Exit*` in a `try/finally`. On .NET 8 also compare it with the new `Lock` type and `Channel<T>`, which express some of these patterns more cleanly.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8. Working, thread-safe concurrency patterns.
// Рабочий потокобезопасный код с тремя примитивами синхронизации.
using System.Diagnostics;

// ============================================================================
// 1) SemaphoreSlim: ограничение степени параллелизма + async-friendly wait.
//    SemaphoreSlim: bound concurrency, non-blocking async wait.
// ============================================================================
public sealed class ThrottledDownloader : IDisposable
{
    // Разрешаем не более 4 одновременных запросов. / At most 4 concurrent requests.
    // ВАЖНО: initialCount == maximumCount, чтобы Release не превысил лимит.
    // IMPORTANT: initialCount == maximumCount prevents Release from exceeding the cap.
    private readonly SemaphoreSlim _gate = new(initialCount: 4, maxCount: 4);
    private readonly HttpClient _client = new();

    // В async-коде НИКОГДА не вызываем _gate.Wait() — только WaitAsync(ct).
    // In async code NEVER call _gate.Wait() — only WaitAsync(ct).
    public async Task<byte[]> DownloadAsync(Uri url, CancellationToken ct)
    {
        // await вместо блокировки потока пула — спасает от thread pool starvation.
        // await instead of blocking a pool thread — prevents thread pool starvation.
        await _gate.WaitAsync(ct).ConfigureAwait(false);
        try
        {
            using var resp = await _client.GetAsync(url, ct).ConfigureAwait(false);
            return await resp.Content.ReadAsByteArrayAsync(ct).ConfigureAwait(false);
        }
        finally
        {
            // Release строго один раз на каждый успешный WaitAsync,
            // даже если ReadAsByteArrayAsync выбросил исключение.
            // Release exactly once per acquired permit, even on exception.
            _gate.Release();
        }
    }

    public void Dispose() => _gate.Dispose();
}

// ============================================================================
// 2) Mutex: одновременный запуск только одного экземпляра приложения (cross-process).
//    Mutex: enforce a single application instance across processes.
// ============================================================================
public sealed class SingleInstanceGuard : IDisposable
{
    // Global\ делает мьютекс видимым во всех терминальных сессиях ОС.
    // Global\ makes the mutex visible across all terminal-server sessions.
    private const string MutexName = @"Global\MyApp.SingleInstance.v1";
    private readonly Mutex _mutex;
    public bool HasOwnership { get; }

    public SingleInstanceGuard()
    {
        // createdNew == true, если мы первые, кто создал именованный мьютекс.
        // createdNew == true if we are the first to create the named mutex.
        _mutex = new Mutex(initiallyOwned: true, MutexName, out bool createdNew);
        HasOwnership = createdNew;
    }

    public void Dispose()
    {
        // Reentrant: освобождаем столько раз, сколько владели (здесь — один).
        // Reentrant: release exactly as many times as we owned it (once here).
        if (HasOwnership)
        {
            try { _mutex.ReleaseMutex(); } catch (ApplicationException) { /* уже освобождён */ }
        }
        _mutex.Dispose();
    }
}

// ============================================================================
// 3) ReaderWriterLockSlim: много читателей параллельно, писатель — эксклюзивно.
//    ReaderWriterLockSlim: many readers concurrently, writer exclusive.
// ============================================================================
public sealed class ThreadSafeCache<TKey, TValue> where TKey : notnull
{
    private readonly Dictionary<TKey, TValue> _data = new();
    private readonly ReaderWriterLockSlim _rw = new(LockRecursionPolicy.SupportsRecursion);

    // Чтение: много потоков одновременно. / Read: many threads at once.
    public TValue Read(TKey key)
    {
        _rw.EnterReadLock();
        try
        {
            return _data[key]; // может выбросить KeyNotFoundException — это ок.
        }
        finally
        {
            _rw.ExitReadLock(); // строго один Exit на один Enter.
        }
    }

    // Запись: эксклюзивно. / Write: exclusive.
    public void Write(TKey key, TValue value)
    {
        _rw.EnterWriteLock();
        try
        {
            _data[key] = value;
        }
        finally
        {
            _rw.ExitWriteLock();
        }
    }

    // Upgradeable: читаем, при необходимости атомарно повышаемся до писателя.
    // Upgradeable: read, then atomically promote to writer if needed.
    // Избегает race condition "прочитал → отпустил → перечитал".
    // Avoids the read-release-reread race condition.
    public TValue GetOrAdd(TKey key, Func<TKey, TValue> factory)
    {
        _rw.EnterUpgradeableReadLock();
        try
        {
            if (_data.TryGetValue(key, out var existing))
                return existing;

            // Повышение до писателя: никто другой не сможет писать, пока мы здесь.
            // Promote to writer: no other thread may write while we hold this.
            _rw.EnterWriteLock();
            try
            {
                // Двойная проверка — между Read и Write мог войти другой писатель.
                // Double-check: another writer may have slipped in between.
                if (_data.TryGetValue(key, out var nowExisting))
                    return nowExisting;

                var produced = factory(key);
                _data[key] = produced;
                return produced;
            }
            finally
            {
                _rw.ExitWriteLock();
            }
        }
        finally
        {
            _rw.ExitUpgradeableReadLock();
        }
    }

    public void Dispose() => _rw.Dispose();
}

// ============================================================================
// Демонстрация: одновременно запускаем все три примитива.
// Demo: run all three primitives concurrently.
// ============================================================================
public static class Demo
{
    public static async Task RunAsync()
    {
        // --- SemaphoreSlim: 10 задач, но не более 4 одновременно ---
        using var dl = new ThrottledDownloader();
        var urls = Enumerable.Range(1, 10).Select(i => new Uri($"https://example.com/{i}"));
        var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
        var tasks = urls.Select(u => dl.DownloadAsync(u, cts.Token));
        byte[][] results = await Task.WhenAll(tasks).ConfigureAwait(false);
        Console.WriteLine($"SemaphoreSlim: скачано {results.Length} файлов / downloaded files");

        // --- ReaderWriterLockSlim: 8 читателей + 2 писателя гоняются за кэшем ---
        var cache = new ThreadSafeCache<string, int>();
        var workers = Enumerable.Range(0, 10).Select(i => Task.Run(() =>
        {
            for (int j = 0; j < 1000; j++)
            {
                if ((j + i) % 5 == 0)
                    cache.Write($"k{i % 3}", j);           // писатель
                else
                    _ = cache.GetOrAdd($"k{i % 3}", _ => i); // читатель/писатель
            }
        }));
        await Task.WhenAll(workers).ConfigureAwait(false);
        Console.WriteLine("ReaderWriterLockSlim: гонка завершена без дедлока / no deadlock");
    }
}
```

#### Best Practices
- Используй `SemaphoreSlim.WaitAsync` вместе с `CancellationToken` и оборачивай `Release()` в `try/finally`, чтобы разрешение всегда возвращалось, даже при исключении.
- Для ограничения параллелизма в async-коде предпочитай `SemaphoreSlim` классу `Semaphore` — он не создаёт объект ядра ОС и дешевле по памяти.
- Именованный `Mutex` для single-instance проверяй на `createdNew` и оборачивай в `using`/`Dispose`; всегда освобождай столько раз, сколько владел.
- В `ReaderWriterLockSlim` держи критические секции максимально короткими, всегда используй `try/finally` и старайся избегать `SupportsRecursion` без веской причины — рекурсия маскирует ошибки дизайна и ухудшает производительность.
- Сравнивай с `Channel<T>` для producer/consumer сценариев и с новым типом `Lock` (.NET 9) для простых критических секций — иногда они выразительнее.

- Use `SemaphoreSlim.WaitAsync` together with a `CancellationToken`, and put `Release()` in a `try/finally` so the permit is always returned even on exception.
- For bounding concurrency in async code prefer `SemaphoreSlim` over `Semaphore` — it allocates no OS kernel object and is cheaper in memory.
- For a single-instance guard, check `createdNew` on a named `Mutex`, wrap it in `using`/`Dispose`, and always release exactly as many times as you acquired.
- In `ReaderWriterLockSlim` keep critical sections short, always use `try/finally`, and avoid `SupportsRecursion` unless you have a concrete reason — recursion hides design bugs and hurts performance.
- Compare with `Channel<T>` for producer/consumer flows and with the new `Lock` type (.NET 9) for plain critical sections — sometimes they are more expressive.

#### Частые ошибки / Common Mistakes
- `_gate.Wait()` в async-методе → блокирует поток пула, ведет к thread pool starvation; всегда `await _gate.WaitAsync(ct)`.
- `Release()` без `try/finally` → при исключении разрешение «теряется», семафор навсегда уменьшает пропускную способность.
- `semaphore.Release()` вызывается дважды или больше, чем было `WaitAsync` → `SemaphoreFullException`; считай входы и выходы строго 1:1.
- `lock (obj) { await ... }` → компилятор выдаст ошибку, но `using var rw = ...; await ...` без отпущенной блокировки — тот же дедлок; никогда не await под захваченной блокировкой.
- Забыл `ExitReadLock`/`ExitWriteLock` в одном из ветвлений → блокировка «залипает» навсегда; только `try/finally`.
- `Mutex` без `Global\` префикса в терминальных сессиях → разные пользователи получают разные мьютексы; используй `Global\` для истинно системной синхронизации.
- `ReaderWriterLockSlim.EnterWriteLock()` внутри уже взятого `EnterReadLock` без upgradeable → дедлок; сначала `EnterUpgradeableReadLock`, затем `EnterWriteLock`.

- Calling `_gate.Wait()` inside an async method → blocks a pool thread, causes thread pool starvation; always `await _gate.WaitAsync(ct)`.
- `Release()` without `try/finally` → on exception the permit is "lost" and the semaphore is permanently throttled.
- Calling `semaphore.Release()` more times than you acquired → `SemaphoreFullException`; keep acquire/release strictly 1:1.
- `lock (obj) { await ... }` → the compiler rejects it, but `using var rw = ...; await ...` while still holding the lock is the same deadlock; never `await` under a held lock.
- Forgetting `ExitReadLock`/`ExitWriteLock` on some branch → the lock sticks forever; only `try/finally`.
- Using a `Mutex` without the `Global\` prefix under terminal sessions → different users get different mutexes; use `Global\` for true system-wide sync.
- Calling `EnterWriteLock()` while already inside `EnterReadLock()` (without upgradeable) → deadlock; use `EnterUpgradeableReadLock` first, then `EnterWriteLock`.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] В async-методе я использую `await semaphore.WaitAsync(ct)`, а не `Wait()`.
- [ ] `Release()` всегда в `finally` и ровно один раз на каждый успешный вход.
- [ ] Именованный `Mutex` имеет префикс `Global\`, если нужна синхронизация между сессиями.
- [ ] В `ReaderWriterLockSlim` каждый `Enter*` парный с `Exit*` в `try/finally`.
- [ ] Повышение read→write делаю через `EnterUpgradeableReadLock`, а не через отпускание и повторный вход.
- [ ] Критические секции короткие; нет `await` под захваченной блокировкой.
- [ ] Я передаю `CancellationToken` во все ожидания и обрабатываю `OperationCanceledException`.

- [ ] In async methods I use `await semaphore.WaitAsync(ct)`, not `Wait()`.
- [ ] `Release()` is always in `finally` and exactly once per successful acquire.
- [ ] A named `Mutex` carries the `Global\` prefix when cross-session sync is required.
- [ ] In `ReaderWriterLockSlim` every `Enter*` is paired with an `Exit*` inside `try/finally`.
- [ ] Read→write promotion goes through `EnterUpgradeableReadLock`, not release-then-reenter.
- [ ] Critical sections are short; no `await` while holding a lock.
- [ ] I pass a `CancellationToken` into every wait and handle `OperationCanceledException`.

#### Ресурсы / Resources
- [Microsoft Learn — SemaphoreSlim — https://learn.microsoft.com/dotnet/api/system.threading.semaphoreslim](https://learn.microsoft.com/dotnet/api/system.threading.semaphoreslim)
- [Microsoft Learn — Mutex — https://learn.microsoft.com/dotnet/api/system.threading.mutex](https://learn.microsoft.com/dotnet/api/system.threading.mutex)
- [Microsoft Learn — ReaderWriterLockSlim — https://learn.microsoft.com/dotnet/api/system.threading.readerwriterlockslim](https://learn.microsoft.com/dotnet/api/system.threading.readerwriterlockslim)
- [Stephen Toub — Async FAQ — https://devblogs.microsoft.com/dotnet/async-faq/](https://devblogs.microsoft.com/dotnet/async-faq/)

---

[⬆ К модулю M11](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
