[⬆ К модулю M11](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M11-L02: lock/Monitor, критические секции / lock/Monitor, critical sections

**Модуль / Module:** M11
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

**Критическая секция** — это фрагмент кода, который обращается к разделяемому (shared) mutable-состоянию и должен выполняться не более чем одним потоком одновременно. Без защиты такие места порождают **race condition** (состояние гонки): два потока читают-модифицируют-пишут одно и то же поле, и теряются обновления. В .NET базовый механизм защиты — **монитор (Monitor)**, поверх которого работает ключевое слово `lock`.

**lock = Monitor.Enter / Monitor.Exit.** Запись `lock(obj) { ... }` разворачивается компилятором в вызов `Monitor.Enter(obj)` в начале и `Monitor.Exit(obj)` в блоке `finally`. То есть `lock` — это синтаксический сахар: он гарантирует освобождение монитора даже при исключении. Сам класс `Monitor` живёт в `System.Threading` и работает с **managed sync block** — скрытым полем каждого heap-объекта, в .NET синхронизация привязана к экземпляру объекта, а не к типу.

**Reentrance (реентерабельность).** Монитор .NET — **рекурсивный**: поток, уже удерживающий монитор на `obj`, может повторно войти в `lock(obj)` без дедлока. Считчик входов инкрементируется, и нужно столько же раз выйти. Это удобно — метод `A` зовёт метод `B`, оба внутри `lock(_gate)` — но маскирует архитектурные проблемы: длинные цепочки под локом снижают параллелизм и усложняют рассуждения.

**Lock object pattern.** Создавайте отдельный **private readonly object**-поле только для блокировки: `private readonly object _gate = new();`. Никогда не блокируйтесь по `this` (внешний код может случайно залочить ваш экземпляр и создать дедлок), не по `typeof(T)` (тип доступен всем доменам и легко становится «глобальным замком»), не по строковым литералам (интернирование делает их общими). И не по value-type — произойдёт боксинг на каждый вызов, и каждый раз будет **разный** объект, синхронизация просто не сработает.

**Что класть под lock.** Критическая секция должна быть **минимальной**: только чтение-модификация shared-состояния. Никогда не делайте `await` внутри `lock` — монитор удерживается конкретным потоком, а `await` возобновляется на другом потоке, что вызовет `SynchronizationLockException` или «потерю» блокировки. Для async-кода используйте `SemaphoreSlim(1,1)`, `AsyncLock` или пересмотрите модель на channel/actor.

**Дедлоки.** Классический дедлок: поток 1 держит `A` и ждёт `B`, поток 2 держит `B` и ждёт `A`. Защита: берите локи всегда **в одном порядке**, используйте `Monitor.TryEnter(timeout)` вместо бесконечного ожидания, и не держите лок во время I/O и вызовов чужого кода (callback-и могут войти обратно и дедлокнуть). Внимательно с `lock` + `Dispatcher.Invoke`/`Control.Invoke` в UI — это частый источник зависаний.

**Pulse/Wait (обзор).** `Monitor.Wait(obj)` **атомарно** освобождает монитор и переводит поток в очередь ожидания; `Monitor.Pulse(obj)` будит один поток из очереди, `PulseAll` — всех. Это низкоуровневый механизм producer/consumer, но в современном коде почти всегда лучше `System.Threading.Channels` и `BlockingCollection` — они корректнее, безопаснее и интегрированы с async/cancellation.

#### Theory (EN)

A **critical section** is a code region that touches shared mutable state and must run by at most one thread at a time. Without protection, such regions spawn **race conditions**: two threads read-modify-write the same field and updates vanish. In .NET the foundational synchronization primitive is the **Monitor**, exposed through the `lock` keyword.

**lock = Monitor.Enter / Monitor.Exit.** The statement `lock(obj) { ... }` compiles down to `Monitor.Enter(obj)` at the start and `Monitor.Exit(obj)` in a `finally` block. So `lock` is syntactic sugar that guarantees monitor release even when an exception is thrown. The `Monitor` class lives in `System.Threading` and operates on a **managed sync block** — a hidden per-object field. Synchronization in .NET is bound to a heap object instance, not to a type.

**Reentrance.** The .NET monitor is **reentrant**: a thread already holding the monitor on `obj` can re-enter `lock(obj)` without deadlock. An entry counter is incremented and the thread must exit the same number of times. This is convenient — method `A` can call method `B`, both inside `lock(_gate)` — but it hides design smells: long chains under a lock reduce parallelism and make reasoning harder.

**Lock object pattern.** Always introduce a dedicated **private readonly object** field purely for locking: `private readonly object _gate = new();`. Never lock on `this` (external code may accidentally lock your instance and cause a deadlock), never on `typeof(T)` (the type object is globally visible and becomes a global lock), and never on string literals (interning makes them shared across the app). Never lock on a value type — each access boxes it into a fresh object, so every `lock` gets a **different** sync block and synchronization silently fails.

**What goes under lock.** Keep critical sections **minimal**: only the read-modify-write of shared state. Never `await` inside `lock` — the monitor is owned by a specific thread, while `await` resumes on a different thread, producing `SynchronizationLockException` or "losing" the lock. For async code use `SemaphoreSlim(1,1)`, an `AsyncLock`, or redesign toward channels/actor model.

**Deadlocks.** The classic deadlock: thread 1 holds `A` waiting for `B`, thread 2 holds `B` waiting for `A`. Mitigations: acquire locks always **in a consistent order**, prefer `Monitor.TryEnter(timeout)` over infinite waits, and never hold a lock across I/O or foreign-code callbacks (callbacks may re-enter and deadlock). Be careful mixing `lock` with `Dispatcher.Invoke`/`Control.Invoke` on UI — a frequent source of hangs.

**Pulse/Wait (overview).** `Monitor.Wait(obj)` **atomically** releases the monitor and moves the thread into a wait queue; `Monitor.Pulse(obj)` wakes one waiting thread, `PulseAll` wakes all. This is the low-level producer/consumer mechanism, but modern code almost always prefers `System.Threading.Channels` and `BlockingCollection` — they are more correct, safer, and integrated with async/cancellation.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — потокобезопасный счётчик на lock/Monitor.
// Thread-safe counter built on lock/Monitor. Comments: RU+EN.

using System;
using System.Threading;
using System.Threading.Tasks;

public sealed class ThreadSafeCounter
{
    // Приватный readonly-объект только для блокировки.
    // Private readonly object used solely for locking.
    // НЕ this, НЕ typeof, НЕ строковый литерал, НЕ value-type.
    private readonly object _gate = new();

    private long _value;        // shared mutable state / разделяемое состояние
    private long _peak;         // отслеживаем максимум / track the peak

    public long Value
    {
        get
        {
            // Чтение тоже под локом — иначе можно увидеть "рваное" состояние
            // при 64-битных полях на старых платформах и при составных инвариантах.
            // Reads under lock too — avoids torn reads and protects invariants.
            lock (_gate)
            {
                return _value;
            }
        }
    }

    public long Peak
    {
        get { lock (_gate) { return _peak; } }
    }

    // Инкремент как атомарная критическая секция.
    // Increment as an atomic critical section.
    public long Increment()
    {
        lock (_gate)                 // Monitor.Enter(_gate)
        {
            _value++;                // read-modify-write под защитой монитора
            if (_value > _peak)
                _peak = _value;      // составной инвариант обновляется атомарно

            return _value;           // Monitor.Exit(_gate) — в finally, даже при исключении
        }
    }

    // Reentrance: тот же поток может повторно войти в lock(_gate).
    // Reentrance: the same thread may re-enter lock(_gate) safely.
    public long IncrementTwice()
    {
        lock (_gate)
        {
            Increment();   // повторный вход — счётчик входов +1, дедлока нет
            Increment();   // re-enter; entry counter incremented, no deadlock
            return _value;
        }
    }

    // TryEnter с таймаутом — защита от потенциального дедлока.
    // TryEnter with timeout — protection against a potential deadlock.
    public bool TryReset(TimeSpan timeout)
    {
        bool taken = false;
        try
        {
            taken = Monitor.TryEnter(_gate, timeout);
            if (!taken)
                return false;       // не дождались — лучше вернуть false, чем висеть

            _value = 0;
            _peak = 0;
            return true;
        }
        finally
        {
            if (taken)
                Monitor.Exit(_gate); // освобождаем только если реально захватили
        }
    }

    // Демонстрация: 8 потоков по 100 000 инкрементов → ожидаем 800 000.
    // Demo: 8 threads x 100 000 increments → expect 800 000.
    public static async Task Main()
    {
        var counter = new ThreadSafeCounter();

        using var cts = new CancellationTokenSource();
        var tasks = new Task[8];

        for (int i = 0; i < tasks.Length; i++)
        {
            int worker = i;
            tasks[i] = Task.Run(() =>
            {
                for (int j = 0; j < 100_000; j++)
                {
                    cts.Token.ThrowIfCancellationRequested();
                    counter.Increment();
                }
            }, cts.Token);
        }

        try
        {
            await Task.WhenAll(tasks);
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine("Cancelled / Отменено");
        }

        // Ожидаем ровно 800 000. Без lock получили бы потерю обновлений (race condition).
        // Expect exactly 800 000. Without lock we'd see lost updates (race condition).
        Console.WriteLine($"Value = {counter.Value}, Peak = {counter.Peak}");
    }
}

// ⚠️ Антипаттерн — НИКОГДА так не делайте:
// ⚠️ Anti-pattern — NEVER do this:
//
//   lock (this) { ... }              // внешний код может залочить ваш экземпляр → дедлок
//   lock (typeof(MyClass)) { ... }   // глобальный замок по типу
//   lock ("shared") { ... }          // интернированная строка — общий замок на весь процесс
//   lock (42) { ... }                // value-type: боксинг → каждый раз новый объект → нет синхронизации
//
// ⚠️ Никогда не await внутри lock — монитор привязан к потоку, await возобновится на другом.
// ⚠️ Never await inside lock — the monitor is thread-affine; await resumes on another thread.
//
//   lock (_gate)
//   {
//       await DoSomethingAsync();   // ❌ SynchronizationLockException / потеря блокировки
//   }
//
// ✅ Для async используйте SemaphoreSlim(1,1) или пересмотрите на Channels.
// ✅ For async use SemaphoreSlim(1,1) or redesign with Channels.
//
//   private readonly SemaphoreSlim _asyncGate = new(1, 1);
//   await _asyncGate.WaitAsync(ct);   // async-friendly, не держит поток
//   try { await DoSomethingAsync(); }
//   finally { _asyncGate.Release(); }
```

#### Best Practices

- Держите критическую секцию минимальной: только чтение-модификация-запись shared-состояния. Никаких I/O, вызовов чужого кода и `await` внутри `lock`.
- Используйте выделенный `private readonly object _gate = new();` как объект блокировки. Не `this`, не `typeof(T)`, не строки, не value-type.
- Не блокируйте поток пула надолго — рассмотрите `SemaphoreSlim`, `Channel`, actor-модель или перепроектируйте на immutable-структуры данных.
- Берите несколько локов всегда в одном и том же порядке; где это невозможно — используйте `Monitor.TryEnter(timeout)`.
- Избегайте реентерабельности как архитектурного приёма: длинные цепочки вызовов под локом — запах дизайна.
- Сначала подумайте о структурной синхронизации (immutable, ConcurrentDictionary, Channels), и только потом о явном `lock`.

- Keep the critical section minimal: only the read-modify-write of shared state. No I/O, no foreign-code calls, no `await` inside `lock`.
- Use a dedicated `private readonly object _gate = new();` as the lock target. Not `this`, not `typeof(T)`, not strings, not value types.
- Do not hold a thread-pool thread for long — consider `SemaphoreSlim`, `Channel`, the actor model, or redesign with immutable data structures.
- Acquire multiple locks in a consistent global order; where impossible, fall back to `Monitor.TryEnter(timeout)`.
- Avoid leaning on reentrance as a design: long call chains under a lock are a design smell.
- Prefer structural synchronization (immutable, ConcurrentDictionary, Channels) before reaching for explicit `lock`.

#### Частые ошибки / Common Mistakes

- `lock(this)` или `lock(typeof(T))` → используйте отдельный `private readonly object _gate`.
- `lock` на value-type (`lock(42)`) → боксинг, синхронизация молча не работает → используйте reference-type объект блокировки.
- `lock` на строковом литерале (`lock("key")`) → интернирование даёт общий глобальный замок → используйте свой `object`.
- `await` внутри `lock` → `SynchronizationLockException` / потеря блокировки → используйте `SemaphoreSlim(1,1).WaitAsync`.
- Длинная критическая секция с I/O/сетью под локом → деградация параллелизма и дедлоки → выносите I/O за пределы локa.
- Взятие двух локов в разном порядке в разных методах → дедлок → установите единый порядок захвата или `TryEnter`.
- Чтение shared-поля без локa «потому что это просто int» → torn read / нарушение инварианта → защищайте чтение тоже (или `volatile`/`Interlocked` для одного поля).
- Забыли `finally` при ручном `Monitor.Enter/Exit` → монитор не освобождается при исключении → используйте `lock`, либо `try/finally` с `Monitor.Exit`.

- `lock(this)` or `lock(typeof(T))` → use a dedicated `private readonly object _gate`.
- `lock` on a value type (`lock(42)`) → boxing silently breaks synchronization → use a reference-type lock object.
- `lock` on a string literal (`lock("key")`) → interning makes it a global lock → use your own `object`.
- `await` inside `lock` → `SynchronizationLockException` / lost lock → use `SemaphoreSlim(1,1).WaitAsync`.
- Long critical section with I/O/network under a lock → parallelism loss and deadlocks → move I/O outside the lock.
- Acquiring two locks in different orders across methods → deadlock → impose a single acquisition order or use `TryEnter`.
- Reading a shared field without a lock because "it's just an int" → torn read / broken invariant → guard the read too (or `volatile`/`Interlocked` for a single field).
- Forgot `finally` with manual `Monitor.Enter/Exit` → monitor not released on exception → use `lock`, or `try/finally` with `Monitor.Exit`.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Для блокировки используется отдельный `private readonly object _gate`, а не `this`/`typeof`/строка/value-type.
- [ ] Критическая секция минимальна: только shared-состояние, без I/O и без `await`.
- [ ] Внутри `lock` нет `await`; для async-кода выбран `SemaphoreSlim(1,1)` или channel.
- [ ] При ручном `Monitor.Enter/Exit` освобождение в `finally`, и только если захват удалось.
- [ ] Порядок взятия нескольких локов единый во всём коде; там, где это невозможно — `TryEnter(timeout)`.
- [ ] Чтение shared-состояния тоже защищено (или обосновано `volatile`/`Interlocked`).
- [ ] Рассмотрены альтернативы: `ConcurrentDictionary`, `Channels`, immutable — до того, как писать `lock`.
- [ ] Понимаю, что монитор реентерабелен, и не злоупотребляю этим как архитектурным приёмом.
- [ ] Знаю, что `lock` разворачивается в `Monitor.Enter` + `Monitor.Exit` в `finally`.
- [ ] Знаю про `Pulse/Wait/PulseAll`, но понимаю, что в новом коде предпочту `Channels`.

- [ ] Locking uses a dedicated `private readonly object _gate`, not `this`/`typeof`/string/value-type.
- [ ] The critical section is minimal: only shared state, no I/O, no `await`.
- [ ] No `await` inside `lock`; async code uses `SemaphoreSlim(1,1)` or a channel.
- [ ] With manual `Monitor.Enter/Exit`, release is in `finally`, and only if the lock was taken.
- [ ] Acquisition order of multiple locks is consistent across the codebase; where impossible, `TryEnter(timeout)` is used.
- [ ] Reads of shared state are guarded too (or justified by `volatile`/`Interlocked`).
- [ ] Alternatives were considered: `ConcurrentDictionary`, `Channels`, immutable — before writing `lock`.
- [ ] I understand the monitor is reentrant, and I do not lean on it as a design.
- [ ] I know `lock` expands to `Monitor.Enter` + `Monitor.Exit` in a `finally`.
- [ ] I know about `Pulse/Wait/PulseAll`, but prefer `Channels` in new code.

#### Ресурсы / Resources

- [Microsoft Learn — lock statement](https://learn.microsoft.com/dotnet/csharp/language-reference/statements/lock)
- [Monitor class — Microsoft Learn](https://learn.microsoft.com/dotnet/api/system.threading.monitor)
- [SemaphoreSlim — Microsoft Learn](https://learn.microsoft.com/dotnet/api/system.threading.semaphoreslim)

---

[⬆ К модулю M11](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
