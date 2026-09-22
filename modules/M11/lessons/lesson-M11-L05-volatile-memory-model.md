[⬆ К модулю M11](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M11-L05: volatile, барьеры памяти, .NET memory model / volatile, memory barriers, .NET memory model

**Модуль / Module:** M11
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Память компьютера — это не то, что кажется. Когда вы пишете `x = 1` в C#, компилятор, JIT и сам процессор могут переставить эту запись относительно других операций, отложить её в кэш L1, переупорядочить с neighbouring чтениями или вовсе слить с соседней записью. Это делается ради производительности: современные CPU исполняют инструкции нестрого, в конвейере, с out-of-order execution. В однопоточном коде это незаметно, потому что перестановки сохраняют *наблюдаемую* семантику для одного потока. Но в многопоточном мире другой поток может увидеть «будущее» или «прошлое» состояние памяти, которое нарушает ваши интуитивные предположения.

Модель памяти .NET (опирается на ECMA-335) описывает, какие гарантии даёт платформа относительно видимости и упорядочения доступов к памяти между потоками. Без явных барьеров .NET даёт лишь минимальные гарантии: запись в обычное поле может стать видимой другим потокам сколь угодно поздно, а чтение может «протечь» вперёд через другие операции. На практике на x86/x64 переупорядочения реже (сильная модель), а на ARM/ARM64 — агрессивнее (слабая модель), поэтому код, «работающий» на Intel, может ломаться на ARM-серверах и Apple Silicon.

Ключевые инструменты упорядочения — `volatile` и memory barriers. Поле, помеченное `volatile`, получает семантику *acquire/release*: volatile-чтение не может быть переупорядочено с последующим доступом к памяти (acquire), а volatile-запись не может быть переупорядочена с предыдущим доступом (release). Это даёт visibility-гарантии для одного поля, но **не** делает составные операции атомарными и **не** запрещает переупорядочение между двумя разными volatile-полями в некоторых сценариях. Аналогия: `volatile` — как табличка «выход через эту дверь в порядке очереди» для одного конкретного поля, а не для всей комнаты.

Memory barrier (барьер памяти) — инструкция, заставляющая процессор «сбросить» буфер записи и завершить предшествующие доступы до начала последующих. В .NET это `Thread.MemoryBarrier()`, `Volatile.Read/Write`, `Interlocked`-операции и неявные барьеры в `lock`, `Monitor`, `Task` continuations. Полный забор (`MemoryBarrier`) — самый сильный, но дорогой; половинные заборы acquire/release через `Volatile.*` дешевле и достаточны в большинстве случаев.

Когда **действительно** нужен `volatile`? Только для неблокирующих сценариев с одним полем: флаг завершения (`_stop`), ленивая инициализация без блокировки, публикация ссылки на immutable-объект после его полного построения. Классический паттерн — `_initialized`-флаг: поток-писатель полностью строит объект, затем `volatile`-запись публикует ссылку; поток-читатель делает `volatile`-чтение флага и, увидев `true`, безопасно видит весь объект. Это работает только если объект *immutable* после публикации.

Ложное понимание `volatile` — самая частая ошибка. Многие думают, что `volatile` делает операции атомарными, защищает от race conditions или заменяет `lock`. Это не так: `volatile int` не защищает `_counter++` (три операции: чтение, инкремент, запись — между ними влезает другой поток). `volatile` не упорядочивает доступы между *разными* полями гарантированно во всех моделях. И самое главное: в 95% случаев вам нужен не `volatile`, а `Interlocked`, `lock`, `Channel` или `lock-free`-структуры из `System.Collections.Concurrent`. Правило: если вы написали `volatile` и не можете строго доказать корректность по модели памяти — перепишите на `lock` или `Interlocked`. Блокировки дёшевы при отсутствии конкуренции и всегда корректны; микрооптимизация через `volatile` оправдана лишь в горячих путях (spin-wait, низкоуровневые пулы), где каждая наносекунда считается и есть бенчмарк.

#### Theory (EN)

Computer memory is not what it looks like. When you write `x = 1` in C#, the compiler, the JIT, and the CPU itself may reorder that store relative to other operations, hold it in the L1 cache, slide it past neighbouring reads, or even coalesce it with an adjacent store. This is done for performance: modern CPUs execute instructions speculatively, in pipelines, with out-of-order execution. In single-threaded code this is invisible, because the reorderings preserve the *observable* semantics for one thread. But in a multi-threaded world another thread may observe a "future" or "past" state of memory that breaks your intuitive assumptions.

The .NET memory model (grounded in ECMA-335) describes the guarantees the platform gives about the visibility and ordering of memory accesses between threads. Without explicit barriers, .NET gives only minimal guarantees: a store to a plain field may become visible to other threads arbitrarily late, and a load may float ahead of other operations. In practice, x86/x64 reorders less (a strong model), while ARM/ARM64 reorders aggressively (a weak model), so code that "works" on Intel can break on ARM servers and Apple Silicon.

The key ordering tools are `volatile` and memory barriers. A field marked `volatile` gets *acquire/release* semantics: a volatile read cannot be reordered with a subsequent memory access (acquire), and a volatile write cannot be reordered with a preceding access (release). This gives visibility guarantees for a single field, but it does **not** make composite operations atomic and does **not** forbid reordering between two different volatile fields in some scenarios. Analogy: `volatile` is like a sign saying "exit through this door in order" for one specific field, not for the whole room.

A memory barrier is an instruction that forces the CPU to flush its store buffer and complete prior accesses before starting subsequent ones. In .NET these are `Thread.MemoryBarrier()`, `Volatile.Read/Write`, `Interlocked` operations, and implicit barriers in `lock`, `Monitor`, and `Task` continuations. A full fence (`MemoryBarrier`) is the strongest but most expensive; half-fences (acquire/release) via `Volatile.*` are cheaper and sufficient in most cases.

When do you actually need `volatile`? Only for non-blocking scenarios involving a single field: a stop flag (`_stop`), lock-free lazy initialisation, publishing a reference to an immutable object after it is fully built. The classic pattern is an `_initialized` flag: the writer thread fully builds the object, then a `volatile` store publishes the reference; the reader thread does a `volatile` load of the flag and, on seeing `true`, safely observes the whole object. This only works when the object is *immutable* after publication.

Misunderstanding `volatile` is the most common mistake. Many believe `volatile` makes operations atomic, prevents race conditions, or replaces `lock`. It does not: `volatile int` does not protect `_counter++` (three operations: load, increment, store — another thread can slip in between). `volatile` does not reliably order accesses between *different* fields across all memory models. Most importantly: in 95% of cases you want `Interlocked`, `lock`, `Channel`, or lock-free structures from `System.Collections.Concurrent`, not `volatile`. Rule of thumb: if you wrote `volatile` and cannot rigorously prove correctness against the memory model, rewrite with `lock` or `Interlocked`. Locks are cheap when uncontended and always correct; a `volatile` micro-optimisation is justified only in hot paths (spin-waits, low-level pools) where every nanosecond counts and you have a benchmark to prove it.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — volatile, memory barriers, .NET memory model
// Рабочий демонстрационный проект: три паттерна — stop-flag, lazy publish, broken counter.

using System.Diagnostics;
using System.Threading;

// ──────────────────────────────────────────────────────────────
// Pattern 1: volatile stop flag for a spin-wait worker.
// Паттерн 1: volatile-флаг остановки для воркера в spin-wait.
//
// Без volatile JIT может кэшировать _stop в регистре, и воркер
// никогда не завершится. volatile даёт acquire-load: каждое
// чтение действительно идёт в память и видит свежую запись.
//
// Without volatile the JIT may cache _stop in a register and the
// worker never stops. volatile gives an acquire-load: each read
// actually goes to memory and sees the fresh store.
// ──────────────────────────────────────────────────────────────
public sealed class SpinWorker : IDisposable
{
    // volatile здесь оправдан: одно поле, hot path, spin-wait.
    // volatile is justified here: single field, hot path, spin-wait.
    private volatile bool _stop;

    private readonly Thread _worker;

    public SpinWorker()
    {
        _worker = new Thread(Work) { IsBackground = true, Name = "SpinWorker" };
        _worker.Start();
    }

    private void Work()
    {
        // Spin until _stop becomes true. Чтение acquire не
        // переупорядочивается с последующими доступами.
        // Spin until _stop becomes true. The acquire-load is not
        // reordered with subsequent accesses.
        while (!_stop)
        {
            // реальная работа в горячем цикле / real work in hot loop
            DoTinyWork();
        }
    }

    private static void DoTinyWork() { /* placeholder compute */ }

    public void Stop()
    {
        // volatile-запись release: все предшествующие записи воркера
        // становятся видимыми до того, как _stop=true увидят другие.
        // Volatile store release: all preceding writes by this thread
        // become visible before _stop=true is observed by others.
        _stop = true;
    }

    public void Dispose()
    {
        Stop();
        // Join ограничен по времени — не блокирует поток пула навсегда.
        // Bounded Join — never blocks a thread pool thread indefinitely.
        if (!_worker.Join(TimeSpan.FromSeconds(2)))
        {
            throw new TimeoutException("SpinWorker did not stop in time.");
        }
    }
}

// ──────────────────────────────────────────────────────────────
// Pattern 2: lock-free publish of an immutable snapshot.
// Паттерн 2: lock-free публикация immutable-снимка.
//
// Писатель строит immutable Config полностью, затем публикует
// ссылку через Volatile.Write. Читатель берёт ссылку через
// Volatile.Read и видит полностью построенный объект.
//
// The writer fully builds an immutable Config, then publishes the
// reference via Volatile.Write. The reader obtains it via
// Volatile.Read and observes a fully constructed object.
// ──────────────────────────────────────────────────────────────
public sealed class ConfigStore
{
    // immutable после построения / immutable after construction
    public sealed record Config(int MaxConnections, string Endpoint);

    private Config? _current; // null until first publish

    public void Publish(Config cfg)
    {
        ArgumentNullException.ThrowIfNull(cfg);
        // Release-store: поля cfg уже записаны, публикация идёт последней.
        // Release-store: cfg's fields are already written; publish goes last.
        Volatile.Write(ref _current, cfg);
    }

    public Config? GetSnapshot()
    {
        // Acquire-load: чтение _current не всплывает выше последующих
        // чтений полей cfg — мы увидим согласованную картину.
        // Acquire-load: reading _current does not float above later reads
        // of cfg's fields — we observe a consistent picture.
        return Volatile.Read(ref _current);
    }
}

// ──────────────────────────────────────────────────────────────
// Pattern 3: WARNING — volatile does NOT fix a racey counter.
// Паттерн 3: ВНИМАНИЕ — volatile НЕ чинит счётчик с race.
//
// _count++ это load + add + store. volatile упорядочивает
// каждое чтение/запись по отдельности, но НЕ делает тройку
// атомарной. Два потока могут потерять инкременты.
// Правильное решение — Interlocked.Increment.
//
// _count++ is load + add + store. volatile orders each
// individual read/write but does NOT make the triple atomic.
// Two threads may lose increments.
// The correct fix is Interlocked.Increment.
// ──────────────────────────────────────────────────────────────
public sealed class CounterBroken
{
    private volatile int _count; // volatile here is a TRAP / ловушка

    public void IncrementBad() => _count++; // race condition

    // Правильно / Correct: атомарный инкремент с полным забором.
    public void IncrementGood() => Interlocked.Increment(ref _count);

    public int Value => Volatile.Read(ref _count);
}

// ──────────────────────────────────────────────────────────────
// Demo entry — запустите и наблюдайте потерянные инкременты.
// Demo entry — run and observe lost increments.
// ──────────────────────────────────────────────────────────────
public static class VolatileDemo
{
    public static void Run()
    {
        // Pattern 1: stop flag / флаг остановки
        using var worker = new SpinWorker();
        Thread.Sleep(100);
        // Dispose вызовет Stop() — volatile-запись завершит spin-wait.
        worker.Dispose();

        // Pattern 3 demo: racey vs atomic counter.
        var broken = new CounterBroken();
        var tasks = new Task[16];
        for (int i = 0; i < tasks.Length; i++)
        {
            tasks[i] = Task.Run(() =>
            {
                for (int j = 0; j < 50_000; j++)
                    broken.IncrementBad();
            });
        }
        Task.WaitAll(tasks);
        // Ожидаем 800_000, но получим меньше — потерянные инкременты.
        // Expected 800_000, but we get fewer — lost increments.
        Console.WriteLine($"broken (volatile, no lock): {broken.Value}");

        // Атомарный путь не теряет инкрементов.
        var safe = new CounterBroken();
        for (int i = 0; i < tasks.Length; i++)
        {
            tasks[i] = Task.Run(() =>
            {
                for (int j = 0; j < 50_000; j++)
                    safe.IncrementGood();
            });
        }
        Task.WaitAll(tasks);
        Console.WriteLine($"safe (Interlocked): {safe.Value}");
    }
}
```

#### Best Practices

- Используйте `volatile` только для одного поля в неблокирующем hot path: флаг остановки, публикация immutable-ссылки. Всё остальное — `lock`, `Interlocked`, `Channel`.
- Для счётчиков и accumulate-операций всегда `Interlocked`, не `volatile`. `volatile` не делает `++` атомарным.
- Для публикации immutable-объекта используйте `Volatile.Write` со стороны писателя и `Volatile.Read` со стороны читателя — это явнее и переносимее, чем ключевое слово `volatile`.
- Избегайте `Thread.MemoryBarrier()` в прикладном коде: полный забор дорогой и легко используется неправильно. Предпочитайте `Volatile.*` и `Interlocked`.
- Помните про платформы: код, корректный на x86/x64, может ломаться на ARM64. Тестируйте на ARM, еслиClaim о lock-free корректности.
- Не пишите свой lock-free код без бенчмарка и без доказательства корректности; `lock` при отсутствии конкуренции стоит единицы наносекунд.

- Use `volatile` only for a single field in a non-blocking hot path: stop flag, publishing an immutable reference. Everything else — `lock`, `Interlocked`, `Channel`.
- For counters and accumulate operations always use `Interlocked`, never `volatile`. `volatile` does not make `++` atomic.
- For publishing an immutable object use `Volatile.Write` on the writer side and `Volatile.Read` on the reader side — this is more explicit and portable than the `volatile` keyword.
- Avoid `Thread.MemoryBarrier()` in application code: a full fence is expensive and easy to misuse. Prefer `Volatile.*` and `Interlocked`.
- Remember platforms: code correct on x86/x64 can break on ARM64. Test on ARM if you claim lock-free correctness.
- Do not hand-roll lock-free code without a benchmark and a correctness proof; an uncontended `lock` costs single-digit nanoseconds.

#### Частые ошибки / Common Mistakes

- `volatile int _count; _count++;` → счётчик всё равно теряет инкременты, потому что `++` это три операции; используйте `Interlocked.Increment`.
- Думать, что `volatile` упорядочивает доступы между *двумя* volatile-полями → не всегда правда на слабых моделях; используйте один флаг или `lock` для составных инвариантов.
- `_initialized = true;` без `volatile`/`Volatile.Write` перед публикацией объекта → читатель может увидеть `true`, но поля объекта ещё не записаны; используйте release-store.
- Зацикливание `while (!_stop)` без `volatile` на не-ARM машине «работает», а на ARM/Apple Silicon воркер не останавливается → всегда `volatile` для stop-флагов spin-wait.
- Использовать `Thread.MemoryBarrier()` везде «для надёжности» → убивает производительность и не заменяет `lock`; поймите acquire/release вместо этого.
- `volatile` на поле ссылочного типа, которое мутируется → `volatile` не защитит внутреннее состояние объекта; используйте `lock` или immutable-дизайн.

- `volatile int _count; _count++;` → counter still loses increments, because `++` is three operations; use `Interlocked.Increment`.
- Believing `volatile` orders accesses between *two* volatile fields → not always true on weak models; use a single flag or `lock` for composite invariants.
- `_initialized = true;` without `volatile`/`Volatile.Write` before publishing an object → a reader may see `true` while the object's fields are not yet written; use a release-store.
- A `while (!_stop)` loop without `volatile` "works" on non-ARM but the worker never stops on ARM/Apple Silicon → always use `volatile` for spin-wait stop flags.
- Sprinkling `Thread.MemoryBarrier()` everywhere "for safety" → kills performance and does not replace `lock`; learn acquire/release instead.
- `volatile` on a mutable reference-type field → `volatile` does not protect the object's internal state; use `lock` or immutable design.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я использую `volatile` только для одного поля в неблокирующем hot path, а не как замену `lock`.
- [ ] Для счётчиков и `++`/`--` я использую `Interlocked`, а не `volatile`.
- [ ] При публикации immutable-объекта я применяю `Volatile.Write` (писатель) и `Volatile.Read` (читатель).
- [ ] Stop-флаги spin-wait помечены `volatile` — код корректен на ARM64, а не только на x64.
- [ ] Я не полагаюсь на `Thread.MemoryBarrier()` без понимания acquire/release семантики.
- [ ] У меня есть бенчмарк, доказывающий, что lock-free вариант действительно быстрее `lock`.

- [ ] I use `volatile` only for a single field in a non-blocking hot path, not as a `lock` replacement.
- [ ] For counters and `++`/`--` I use `Interlocked`, not `volatile`.
- [ ] When publishing an immutable object I apply `Volatile.Write` (writer) and `Volatile.Read` (reader).
- [ ] Spin-wait stop flags are marked `volatile` — the code is correct on ARM64, not only on x64.
- [ ] I do not rely on `Thread.MemoryBarrier()` without understanding acquire/release semantics.
- [ ] I have a benchmark proving the lock-free variant is actually faster than `lock`.

#### Ресурсы / Resources

- [Microsoft Learn — System.Threading.Volatile](https://learn.microsoft.com/dotnet/api/system.threading.volatile)
- [ECMA-335 — Common Language Infrastructure (CLI) standard](https://ecma-international.org/publications-and-standards/standards/ecma-335/)
- [Volatile.Read / Volatile.Write — Microsoft Learn](https://learn.microsoft.com/dotnet/api/system.threading.volatile)
- [Thread.MemoryBarrier — Microsoft Learn](https://learn.microsoft.com/dotnet/api/system.threading.thread.memorybarrier)

---

[⬆ К модулю M11](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
