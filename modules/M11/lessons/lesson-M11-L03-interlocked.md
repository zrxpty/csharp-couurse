[⬆ К модулю M11](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M11-L03: Interlocked, атомарные операции / Interlocked, atomic operations

**Модуль / Module:** M11
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Атомарная операция — это операция, которая выполняется как единое неделимое целое: ни один другой поток не может «просочиться» между её шагами. Класс `System.Threading.Interlocked` предоставляет такие операции для числовых типов и ссылок, и делает это на уровне процессорных инструкций (например, `LOCK XADD` на x86), без перехода в режим ядра операционной системы.

Представьте очередь в кофейню. Обычный счётчик посетителей `count++` состоит из трёх шагов: прочитать значение, прибавить единицу, записать обратно. Если два бариста одновременно читают `count = 5`, оба прибавляют `1` и оба записывают `6`, мы потеряли одного посетителя — это и есть классическая race condition. `Interlocked.Increment(ref count)` делает все три шага атомарно: второй бариста дождётся первого и обязательно получит `7`.

Основные методы класса `Interlocked`:
- `Increment(ref int/long)` и `Decrement` — атомарный `+1` / `-1`.
- `Add(ref int/long, value)` — атомарное сложение произвольного значения.
- `Exchange(ref location, value)` — атомарно записывает новое значение и возвращает старое.
- `CompareExchange(ref location, newValue, comparand)` — если текущее значение равно `comparand`, заменить на `newValue`; всегда возвращает предыдущее значение. Это строительный блок почти всех lock-free алгоритмов.
- `MemoryBarrier()` — полный забор памяти (full fence): гарантирует, что никакая загрузка/сохранение не «перепрыгнет» через барьер.

Atomicity ≠ memory ordering. Атомарность гарантирует целостность самой операции, но не порядок видимости других переменных. На x86/x64 модель памяти сильная, и многие эффекты скрыты, но на ARM (Apple Silicon, ARM-серверы, мобильные) переупорядочивание чтений и записей реально. Поэтому `Interlocked`-методы неявно вставляют нужные барьеры: читатель всегда видит корректное значение, опубликованное через `Interlocked.Exchange`.

Когда выбирать `Interlocked`, а когда `lock`? Эмпирическое правило:
- Одну простую операцию над общей переменной (счётчик, флаг) — `Interlocked`. Быстро, без блокировок, не блокирует пул потоков.
- Составную логику, где нужно прочитать, посчитать и записать с дополнительными побочными эффектами — `lock`. Особенно если внутри циклы, вызовы внешних методов или `await` (хотя `lock` + `await` — отдельная ловушка, см. ниже).
- Несколько связанных полей, которые должны меняться согласованно, — `lock` или `Monitor`, потому что `Interlocked` работает с одним значением за раз.

Lock-free счётчики — самый частый паттерн. Они отлично подходят для метрик, статистики, sequence-номеров. Но помните: lock-free ≠ всегда быстро. При высокой конкуренции `CompareExchange` крутится в цикле (CAS-loop) и тратит CPU. Для очень горячего счётчика иногда выгоднее `lock` или даже `SemaphoreSlim`.

Классические ловушки:
1. `count++` без `Interlocked` в многопоточной среде — потерянные обновления.
2. Проверка-потом-действие (`if (x == 0) x = 1;`) — всегда race, нужна `CompareExchange`.
3. `lock` на `await` — `lock` не асинхронен, удержание потока пула блокирует масштабирование; используйте `SemaphoreSlim(1,1)` с `await sem.WaitAsync()`.
4. Вера, что `volatile` заменяет `Interlocked` — `volatile` даёт только memory ordering, не атомарность составных операций.
5. `.Result` / `.Wait()` на async-коде внутри lock-free логики — дедлоки из-за захвата контекста синхронизации и блокировки потоков пула.

`Interlocked` — это инструмент «точечной» синхронизации: дёшев, быстр, но ограничен одним значением. Понимание границы между атомарностью и упорядочиванием памяти — ключ к корректному lock-free коду.

#### Theory (EN)

An atomic operation is one that executes as a single indivisible unit: no other thread can observe or interfere with its intermediate steps. The `System.Threading.Interlocked` class exposes such operations for numeric types and object references, and it does so at the level of CPU instructions (such as `LOCK XADD` on x86), without transitioning the thread into kernel mode.

Imagine a queue at a coffee shop. A plain visitor counter `count++` is actually three steps: read the value, add one, write it back. If two baristas simultaneously read `count = 5`, both add `1`, and both store `6`, one visitor is lost — that is a textbook race condition. `Interlocked.Increment(ref count)` makes all three steps atomic: the second barista waits for the first and is guaranteed to produce `7`.

The main methods of `Interlocked`:
- `Increment(ref int/long)` and `Decrement` — atomic `+1` / `-1`.
- `Add(ref int/long, value)` — atomic addition of an arbitrary value.
- `Exchange(ref location, value)` — atomically writes a new value and returns the previous one.
- `CompareExchange(ref location, newValue, comparand)` — if the current value equals `comparand`, replace it with `newValue`; it always returns the previous value. This is the building block of nearly every lock-free algorithm.
- `MemoryBarrier()` — a full memory fence: no load or store may be reordered across it.

Atomicity ≠ memory ordering. Atomicity guarantees the integrity of the operation itself, but not the visibility order of other variables. On x86/x64 the memory model is strong, so many effects are hidden; on ARM (Apple Silicon, ARM servers, mobile devices) reordering of reads and writes is real. That is why `Interlocked` methods insert the necessary barriers implicitly: a reader always observes the value published through `Interlocked.Exchange` correctly.

When to choose `Interlocked` and when `lock`? A practical rule of thumb:
- A single simple operation on a shared variable (counter, flag) — `Interlocked`. Fast, lock-free, does not block the thread pool.
- Compound logic where you need to read, compute, and write with additional side effects — `lock`. Especially when there are loops, calls to external methods, or `await` (though `lock` + `await` is a trap of its own, see below).
- Several related fields that must change consistently together — `lock` or `Monitor`, because `Interlocked` operates on a single value at a time.

Lock-free counters are the most common pattern. They are excellent for metrics, statistics, and sequence numbers. But remember: lock-free ≠ always fast. Under heavy contention a `CompareExchange` spins in a CAS-loop and burns CPU. For a very hot counter it can sometimes be more efficient to use a `lock` or even a `SemaphoreSlim`.

Classic pitfalls:
1. `count++` without `Interlocked` in a multi-threaded context — lost updates.
2. Check-then-act (`if (x == 0) x = 1;`) — always a race; use `CompareExchange`.
3. `lock` around `await` — `lock` is not async-aware; holding a pool thread blocks scaling; use `SemaphoreSlim(1,1)` with `await sem.WaitAsync()`.
4. Believing that `volatile` replaces `Interlocked` — `volatile` only provides memory ordering, not atomicity of compound operations.
5. `.Result` / `.Wait()` on async code inside lock-free logic — deadlocks from synchronization-context capture and pool-thread starvation.

`Interlocked` is a tool for point-wise synchronization: cheap, fast, but limited to a single value. Understanding the boundary between atomicity and memory ordering is the key to correct lock-free code.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8+ — рабочий потокобезопасный код
// Working thread-safe code with real concurrency patterns.

using System.Threading;

namespace M11L03.Interlocked;

/// <summary>
/// Lock-free счётчик на Interlocked.Increment / Decrement.
/// Lock-free counter built on Interlocked.Increment / Decrement.
/// Подходит для метрик, статистики запросов, sequence-номеров.
/// Suitable for metrics, request stats, sequence numbers.
/// </summary>
public sealed class LockFreeCounter
{
    private int _current;       // текущее значение / current value
    private long _totalAdded;   // суммарно добавлено за всё время / lifetime total added

    public int Current => Volatile.Read(ref _current); // безопасное чтение / safe read
    public long Total => Interlocked.Read(ref _totalAdded);

    // Атомарный инкремент на 1 / Atomic increment by 1.
    public int Increment() => Interlocked.Increment(ref _current);

    // Атомарный декремент на 1 / Atomic decrement by 1.
    public int Decrement() => Interlocked.Decrement(ref _current);

    // Атомарное сложение произвольного значения / Atomic add of arbitrary value.
    public int Add(int delta)
    {
        int newValue = Interlocked.Add(ref _current, delta);
        // Накапливаем статистику отдельно — каждое слагаемое учитывается ровно один раз.
        // Accumulate statistics separately — each addend is counted exactly once.
        Interlocked.Add(ref _totalAdded, delta);
        return newValue;
    }
}

/// <summary>
/// Один раз инициализируемый кэш через CompareExchange (CAS-цикл).
/// Once-initialized cache via CompareExchange (CAS-loop).
/// Классическая ловушка «if (x == null) x = Build();» — это race.
/// The classic trap "if (x == null) x = Build();" is a race.
/// </summary>
public sealed class LazyAtomicReference<T> where T : class
{
    private T? _value;

    public T GetValue(Func<T> factory)
    {
        // Быстрый путь: значение уже есть / Fast path: value already present.
        T? snapshot = Volatile.Read(ref _value);
        if (snapshot is not null) return snapshot;

        // Медленный путь: фабрика может вызваться у нескольких потоков,
        // но в _value закрепится ровно один объект — тот, кто первым выиграл CAS.
        // Slow path: factory may be invoked by several threads,
        // but only one object sticks in _value — the first to win the CAS.
        T created = factory();
        T? previous = Interlocked.CompareExchange(ref _value, created, comparand: null);
        return previous ?? created; // вернём победителя / return the winner
    }
}

/// <summary>
/// Демонстрация race condition и его исправления.
/// Demonstrates a race condition and how to fix it.
/// </summary>
public static class RaceDemo
{
    // НЕПРАВИЛЬНО: потерянные обновления / WRONG: lost updates.
    public static int RunBroken(int threads, int incrementsPerThread)
    {
        int counter = 0;
        Parallel.For(0, threads, _ =>
        {
            for (int i = 0; i < incrementsPerThread; i++)
                counter++; // не атомарно! / not atomic!
        });
        return counter; // почти всегда меньше ожидаемого / almost always less than expected
    }

    // ПРАВИЛЬНО: атомарный инкремент / CORRECT: atomic increment.
    public static int RunSafe(int threads, int incrementsPerThread)
    {
        int counter = 0;
        Parallel.For(0, threads, _ =>
        {
            for (int i = 0; i < incrementsPerThread; i++)
                Interlocked.Increment(ref counter);
        });
        return counter; // всегда threads * incrementsPerThread / always exact
    }
}

/// <summary>
/// Сравнение: когда Interlocked выгоднее lock.
/// Comparison: when Interlocked beats lock.
/// Здесь простой счётчик — берём Interlocked.
/// For a simple counter we use Interlocked.
/// Для составной операции над двумя полями — lock.
/// For a compound operation over two fields — lock.
/// </summary>
public sealed class Account
{
    private long _balance;       // баланс / balance
    private long _txCount;       // число транзакций / transaction count
    private readonly object _guard = new(); // только для составных операций / only for compound ops

    // Одиночная атомарная операция — Interlocked. Без блокировок.
    // Single atomic operation — Interlocked. No locks.
    public long IncrementTxCount() => Interlocked.Increment(ref _txCount);

    // Составная операция: баланс и счётчик должны меняться согласованно.
    // Compound operation: balance and counter must change together.
    // ВАЖНО: внутри lock НЕТ await. Если нужен await — SemaphoreSlim.
    // IMPORTANT: no await inside lock. If await is needed — SemaphoreSlim.
    public void Transfer(decimal amount, bool isCredit)
    {
        lock (_guard)
        {
            long delta = (long)(amount * 100m); // работаем в копейках / work in cents
            _balance = isCredit ? _balance + delta : _balance - delta;
            _txCount++; // согласовано с изменением баланса / consistent with balance change
        }
    }

    public long Balance => Interlocked.Read(ref _balance);
    public long TxCount => Interlocked.Read(ref _txCount);
}
```

#### Best Practices

- Используйте `Interlocked` для одной атомарной операции над одним значением (счётчик, флаг, sequence) — это дёшево и не блокирует пул потоков.
- Для безопасной публикации объекта используйте `Interlocked.Exchange`/`CompareExchange` в паре с `Volatile.Read` на стороне читателя.
- Помните, что `Interlocked` неявно даёт нужные барьеры памяти — не добавляйте `Volatile` или `Thread.MemoryBarrier` «на всякий случай».
- Для составных операций над несколькими полями используйте `lock`; если внутри нужен `await`, замените на `SemaphoreSlim(1,1)` и `await sem.WaitAsync()` с `try/finally`.
- Измеряйте: при экстремальной конкуренции CAS-цикл может проиграть `lock` из-за spinning — профилируйте под реальной нагрузкой.

- Use `Interlocked` for a single atomic operation on a single value (counter, flag, sequence) — it is cheap and does not block the thread pool.
- For safe publication of an object, pair `Interlocked.Exchange`/`CompareExchange` on the writer with `Volatile.Read` on the reader.
- Remember that `Interlocked` implicitly provides the required memory barriers — do not sprinkle `Volatile` or `Thread.MemoryBarrier` “just in case”.
- For compound operations over several fields use `lock`; if `await` is needed inside, replace it with `SemaphoreSlim(1,1)` and `await sem.WaitAsync()` wrapped in `try/finally`.
- Measure: under extreme contention a CAS-loop can lose to `lock` due to spinning — profile under real load.

#### Частые ошибки / Common Mistakes

- `count++` в многопоточной среде → потерянные обновления. Используйте `Interlocked.Increment(ref count)`.
- Паттерн `if (x == 0) x = 1;` → состояние гонки между проверкой и записью. Используйте `Interlocked.CompareExchange(ref x, 1, 0)`.
- `lock (obj) { await ... }` → `lock` не асинхронен, компилятор выдаст ошибку или вы удержите поток пула. Используйте `SemaphoreSlim` с `WaitAsync`.
- Считать, что `volatile` заменяет `Interlocked` → `volatile` даёт только упорядочивание, не атомарность `++`. Для атомарности — `Interlocked`.
- `.Result` / `.Wait()` внутри lock-free логики → дедлоки и голодание пула потоков. Используйте `await` до конца.

- `count++` in a multi-threaded context → lost updates. Use `Interlocked.Increment(ref count)`.
- The `if (x == 0) x = 1;` pattern → a race between the check and the write. Use `Interlocked.CompareExchange(ref x, 1, 0)`.
- `lock (obj) { await ... }` → `lock` is not async-aware; the compiler errors out or you hold a pool thread. Use `SemaphoreSlim` with `WaitAsync`.
- Assuming `volatile` replaces `Interlocked` → `volatile` only provides ordering, not the atomicity of `++`. For atomicity use `Interlocked`.
- `.Result` / `.Wait()` inside lock-free logic → deadlocks and pool-thread starvation. Make the code fully `async`/`await` end to end.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Каждый разделяемый счётчик инкрементируется через `Interlocked`, а не `++`.
- [ ] Чтение разделяемого значения выполняется через `Volatile.Read` или `Interlocked.Read` (для `long`).
- [ ] Паттерн «проверить-и-заменить» реализован через `CompareExchange`, а не `if`.
- [ ] Внутри `lock` нет `await`; для асинхронных критических секций используется `SemaphoreSlim`.
- [ ] Нет `.Result` / `.Wait()` на async-операциях — только `await`.
- [ ] Понимаю разницу между атомарностью операции и упорядочиванием памяти.

- [ ] Every shared counter is incremented via `Interlocked`, not `++`.
- [ ] Reading a shared value uses `Volatile.Read` or `Interlocked.Read` (for `long`).
- [ ] The check-and-replace pattern is implemented with `CompareExchange`, not `if`.
- [ ] No `await` inside `lock`; `SemaphoreSlim` is used for async critical sections.
- [ ] No `.Result` / `.Wait()` on async operations — only `await`.
- [ ] I understand the difference between operation atomicity and memory ordering.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.threading.interlocked](https://learn.microsoft.com/dotnet/api/system.threading.interlocked)

---

[⬆ К модулю M11](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
