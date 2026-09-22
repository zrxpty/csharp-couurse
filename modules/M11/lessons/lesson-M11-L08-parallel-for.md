[⬆ К модулю M11](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M11-L08: Parallel.For/ForEach/Invoke, Partitioner / Parallel.For/ForEach/Invoke, Partitioner

**Модуль / Module:** M11
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Класс `System.Threading.Tasks.Parallel` — это высокоуровневая обёртка над ThreadPool и TPL для **data parallelism** (параллелизма по данным). В отличие от `Task.Run`, где вы вручную создаёте задачи, здесь вы описываете *что* делать с каждой порцией данных, а TPL сам решает, *как* распределить работу по потокам пула.

**Parallel.For** запускает цикл с целочисленным диапазоном индексов. Представьте длинный конвейер на заводе: вместо одного рабочего, который последовательно обрабатывает 1000 деталей, TPL ставит нескольких рабочих, каждому выдаёт свой диапазон индексов и синхронизирует финиш через неявный `Barrier`. После возврата из `Parallel.For` гарантируется, что **все итерации завершены**.

**Parallel.ForEach** делает то же самое для произвольной `IEnumerable<TSource>` коллекции. TPL разбивает последовательность на чанки (partition) и раздаёт их воркерам. Это удобно для CPU-bound обработки: фильтрация изображений, агрегация логов, математика над массивами.

**Parallel.Invoke** выполняет набор `Action`-делегатов параллельно и дожидается всех. Это альтернатива `Task.WhenAll` для синхронных CPU-bound операций без `await`.

**ParallelOptions** даёт три рычага управления:
- `MaxDegreeOfParallelism` — верхний предел одновременных воркеров (по умолчанию `Environment.ProcessorCount`, но может быть и больше — TPL использует ThreadPool). Ограничивайте, когда параллелизм конкурирует за общий ресурс (диск, сетевой канал, GC).
- `CancellationToken` — структура для кооперативной отмены. TPL проверяет токен между итерациями и бросает `OperationCanceledException`. Без токена нельзя остановить «разогнанный» цикл.
- `TaskScheduler` — редко нужен в прикладном коде; полезен при интеграции с UI-потоком (например, `TaskScheduler.FromCurrentSynchronizationContext()`).

**Race conditions** — главный враг. Если тело цикла пишет в общую переменную (`sum += x`), результат непредсказуем: два потока читают одно значение, оба прибавляют, один перетирает другого. Решения: `Interlocked` для чисел, `lock` для составных операций, или `localInit`/`localFinally` перегрузки `Parallel.For/ForEach` (thread-local аккумуляторы без блокировок). Второй подход — самый быстрый: каждый воркер копит локальную сумму, а в `localFinally` агрегирует в общую через `Interlocked`.

**Partitioner** для баланса. Стандартный partitioner TPL по умолчанию создаёт чанки адаптивного размера. Но если работа в итерациях сильно **неравномерна** (первые 100 элементов считают 1 мс, последние — 500 мс), один воркер «застрянет». `Partitioner.Create(source, EnumerablePartitionerOptions.NoBuffering)` или кастомный `Partitioner<T>` дают более мелкие чанки и лучший баланс ценой большего overhead. Для диапазонов чисел есть `Partitioner.Create(0, N, rangeSize)`.

**Дедлоки и блокировки пула.** Никогда не вызывайте `.Result`/`.Wait()` на асинхронной операции из тела `Parallel.For` — это блокирует поток пула и при насыщении ThreadPool приводит к «зависанию» (thread-pool starvation). Если работа async — используйте `Task.Run` + `Task.WhenAll` или `Channel`-pipeline (урок M11-L07), а не `Parallel`. Также избегайте `lock` поверх `await` (компилятор не даст — но в ручной синхронизации легко нарваться).

**PLINQ (обзор).** `source.AsParallel().Select(...)` превращает LINQ в параллельный. Удобен, но менее управляем: нет `CancellationToken` в теле (только `WithCancellation`), порядок результата не сохраняется (`AsOrdered()` дорого стоит), и накладные расходы на мелких коллекциях перевешивают выгоду. Для CPU-bound трансформаций данных PLINQ — лаконичный выбор; для сложного контроля — `Parallel.For/ForEach`.

#### Theory (EN)

The `System.Threading.Tasks.Parallel` class is a high-level wrapper over ThreadPool and the Task Parallel Library (TPL) for **data parallelism**. Unlike `Task.Run`, where you create tasks manually, here you describe *what* to do with each chunk of data, and TPL decides *how* to spread the work across pool threads.

**Parallel.For** runs a loop over an integer index range. Picture a long factory conveyor: instead of a single worker processing 1,000 parts sequentially, TPL stations several workers, hands each a range of indices, and synchronizes completion through an implicit `Barrier`. After `Parallel.For` returns, **all iterations are guaranteed complete**.

**Parallel.ForEach** does the same for any `IEnumerable<TSource>`. TPL splits the sequence into chunks (partitions) and feeds them to workers. Ideal for CPU-bound processing: image filtering, log aggregation, math over arrays.

**Parallel.Invoke** executes a set of `Action` delegates in parallel and waits for all. It is the synchronous CPU-bound alternative to `Task.WhenAll` when you cannot `await`.

**ParallelOptions** provides three control levers:
- `MaxDegreeOfParallelism` — an upper bound on concurrent workers (default `Environment.ProcessorCount`, may exceed it because TPL uses the ThreadPool). Cap it when parallelism competes for a shared resource (disk, network, GC).
- `CancellationToken` — cooperative cancellation structure. TPL checks the token between iterations and throws `OperationCanceledException`. Without it, a "spun-up" loop cannot be stopped.
- `TaskScheduler` — rarely needed in application code; useful when integrating with a UI thread (e.g., `TaskScheduler.FromCurrentSynchronizationContext()`).

**Race conditions** are the chief enemy. If the loop body writes to a shared variable (`sum += x`), the result is unpredictable: two threads read the same value, both add, one overwrites the other. Solutions: `Interlocked` for numbers, `lock` for compound operations, or the `localInit`/`localFinally` overloads of `Parallel.For/ForEach` (thread-local accumulators without locks). The last approach is the fastest: each worker accumulates a local sum, then `localFinally` merges it into the shared total via `Interlocked`.

**Partitioner** for balance. TPL's default partitioner creates adaptively-sized chunks. But when per-iteration work is highly **uneven** (the first 100 elements take 1 ms, the last 500 ms), one worker will stall. `Partitioner.Create(source, EnumerablePartitionerOptions.NoBuffering)` or a custom `Partitioner<T>` yield smaller chunks and better balance at the cost of higher overhead. For index ranges use `Partitioner.Create(0, N, rangeSize)`.

**Deadlocks and pool starvation.** Never call `.Result`/`.Wait()` on an async operation from inside a `Parallel.For` body — it blocks a pool thread and, once the ThreadPool is saturated, leads to thread-pool starvation and an apparent hang. If the work is async, use `Task.Run` + `Task.WhenAll` or a `Channel`-based pipeline (lesson M11-L07) instead of `Parallel`. Also avoid `lock` over `await` (the compiler forbids it for `await` inside `lock`, but with manual synchronization it is an easy trap).

**PLINQ (overview).** `source.AsParallel().Select(...)` turns LINQ into a parallel pipeline. Convenient but less controllable: no `CancellationToken` in the body (only `WithCancellation`), result order is not preserved (`AsOrdered()` is expensive), and overhead dominates for small collections. For CPU-bound data transformations PLINQ is a concise choice; for fine control prefer `Parallel.For/ForEach`.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — рабочий, потокобезопасный пример Parallel.For/ForEach/Invoke, Partitioner
// Working, thread-safe example of Parallel.For/ForEach/Invoke and Partitioner.

using System.Collections.Concurrent;
using System.Diagnostics;

namespace M11L08.ParallelDemo;

public static class ParallelExamples
{
    // 1) Parallel.For с thread-local аккумулятором — без lock, без race.
    //    Parallel.For with a thread-local accumulator — no lock, no race.
    public static long SumSquares(int fromInclusive, int toExclusive, CancellationToken ct)
    {
        // localInit  — вызывается один раз на воркер, создаёт локальную сумму.
        // localInit  — called once per worker, creates a local sum.
        // body       — обновляет локальную сумму (без блокировок).
        // body       — updates the local sum (no locks).
        // localFinally — вызывается один раз на воркер, сливает локальную сумму в общую.
        // localFinally — called once per worker, merges the local sum into the total.
        long total = 0;

        Parallel.For(
            fromInclusive,
            toExclusive,
            // ParallelOptions: ограничиваем параллелизм и подключаем отмену.
            // ParallelOptions: cap parallelism and wire cancellation.
            new ParallelOptions
            {
                MaxDegreeOfParallelism = Environment.ProcessorCount,
                CancellationToken = ct
            },
            localInit: () => 0L,                       // 0 на старте воркера / 0 per worker start
            body: (i, loopState, localSum) =>
            {
                // Тяжёлая CPU-bound работа на одну итерацию.
                // Heavy CPU-bound work per iteration.
                long x = i;
                localSum += x * x;

                // Внутри тела можно досрочно остановить весь цикл.
                // You can early-stop the whole loop from within the body.
                if (localSum > 1_000_000_000_000L)
                {
                    loopState.Stop(); // немедленно, без ожидания других воркеров / immediate, no wait
                }

                return localSum;
            },
            localFinally: (localSum) =>
            {
                // Единственное место с блокировкой — слияние результатов воркеров.
                // The only locked spot — merging worker results.
                Interlocked.Add(ref total, localSum);
            });

        return total;
    }

    // 2) Parallel.ForEach с ConcurrentDictionary — потокобезопасная агрегация.
    //    Parallel.ForEach with ConcurrentDictionary — thread-safe aggregation.
    public static ConcurrentDictionary<string, int> CountWords(IEnumerable<string> words, CancellationToken ct)
    {
        var counts = new ConcurrentDictionary<string, int>();

        var options = new ParallelOptions
        {
            MaxDegreeOfParallelism = Environment.ProcessorCount,
            CancellationToken = ct
        };

        Parallel.ForEach(
            words,
            options,
            word =>
            {
                // ConcurrentDictionary сам потокобезопасен — никакого внешнего lock.
                // ConcurrentDictionary is itself thread-safe — no external lock needed.
                counts.AddOrUpdate(
                    key: word,
                    addValueFactory: _ => 1,
                    updateValueFactory: (_, current) => current + 1);
            });

        return counts;
    }

    // 3) Parallel.Invoke — параллельный запуск независимых CPU-bound делегатов.
    //    Parallel.Invoke — parallel run of independent CPU-bound delegates.
    public static (double avg, int max, int min) ComputeStats(int[] data, CancellationToken ct)
    {
        double avg = 0; int max = int.MinValue; int min = int.MaxValue;
        object gate = new(); // компактная критическая секция для составных операций
                              // compact critical section for compound operations

        var options = new ParallelOptions
        {
            MaxDegreeOfParallelism = 3,
            CancellationToken = ct
        };

        Parallel.Invoke(
            options,
            () =>
            {
                double localAvg = data.Average();
                lock (gate) { avg = localAvg; } // безопасная запись double / safe double write
            },
            () =>
            {
                int localMax = data.Max();
                lock (gate) { max = localMax; }
            },
            () =>
            {
                int localMin = data.Min();
                lock (gate) { min = localMin; }
            });

        return (avg, max, min);
    }

    // 4) Partitioner.Create для диапазонов — баланс при неравномерной нагрузке.
    //    Partitioner.Create for ranges — balance under uneven load.
    public static Dictionary<int, TimeSpan> UnevenWorkload(int total, CancellationToken ct)
    {
        var timings = new ConcurrentDictionary<int, TimeSpan>();
        var sw = Stopwatch.StartNew();

        // Делим диапазон на чанки по 50 элементов. Каждый чанк — отдельная единица работы,
        // поэтому «длинные» индексы не застрянут за одним воркером.
        // Split the range into chunks of 50. Each chunk is a unit of work,
        // so "long" indices won't get stuck behind a single worker.
        var partitioner = Partitioner.Create(0, total, rangeSize: 50);

        var options = new ParallelOptions
        {
            MaxDegreeOfParallelism = Environment.ProcessorCount,
            CancellationToken = ct
        };

        Parallel.ForEach(
            partitioner,
            options,
            range =>
            {
                var localSw = Stopwatch.StartNew();
                for (int i = range.Item1; i < range.Item2; i++)
                {
                    // Имитация: чем больше i, тем дольше работа.
                    // Simulation: the larger i, the longer the work.
                    int workload = i switch
                    {
                        < 100 => 1,
                        < 500 => 10,
                        _ => 50
                    };
                    for (int j = 0; j < workload; j++) { /* CPU spin */ }
                }
                localSw.Stop();
                timings[range.Item1] = localSw.Elapsed;
            });

        sw.Stop();
        timings[-1] = sw.Elapsed; // -1 = общее время / -1 = total time
        return new Dictionary<int, TimeSpan>(timings);
    }

    // 5) Запуск с отменой по таймауту — безопасная демонстрация CancellationToken.
    //    Run with timeout cancellation — safe CancellationToken demo.
    public static async Task DemoCancellationAsync()
    {
        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(2));

        try
        {
            // НЕ блокируем поток: await внутри Task.Run, а не .Result из Parallel.
            // Do not block the thread: await inside Task.Run, not .Result from Parallel.
            await Task.Run(() => SumSquares(0, 100_000_000, cts.Token), cts.Token);
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine("Отменено по таймауту / Cancelled by timeout.");
        }
    }
}
```

#### Best Practices

- Предпочитайте перегрузки `Parallel.For/ForEach` с `localInit`/`localFinally` для агрегаций — они избегают `lock` в горячем теле цикла и работают в разы быстрее.
- Всегда передавайте `CancellationToken` через `ParallelOptions`; проверяйте его в долгих итерациях, чтобы отмена срабатывала отзывчиво.
- Ограничивайте `MaxDegreeOfParallelism`, если тело обращается к внешнему ресурсу (диск, сеть, БД) или если потребление памяти критично.
- Используйте `Parallel.ForEach` для `IEnumerable`, а `Parallel.For` — для индексных диапазонов; смешивать не нужно.
- Для **async** операций не применяйте `Parallel` — берите `Task.WhenAll` или `Channel`-pipeline.
- Если нагрузка сильно неравномерна, переходите на `Partitioner.Create` с мелкими чанками.

- Prefer `Parallel.For/ForEach` overloads with `localInit`/`localFinally` for aggregations — they avoid `lock` in the hot loop body and run several times faster.
- Always pass a `CancellationToken` via `ParallelOptions`; check it inside long iterations so cancellation is responsive.
- Cap `MaxDegreeOfParallelism` when the body touches an external resource (disk, network, DB) or memory pressure matters.
- Use `Parallel.ForEach` for `IEnumerable`, `Parallel.For` for index ranges; do not mix them.
- Do **not** use `Parallel` for **async** operations — pick `Task.WhenAll` or a `Channel`-based pipeline.
- If the workload is highly uneven, switch to `Partitioner.Create` with small chunks.

#### Частые ошибки / Common Mistakes

- Общий `sum += x` без `Interlocked`/`lock` → непредсказуемый результат. Используйте `localInit`/`localFinally` или `Interlocked.Add`.
- `.Result`/`.Wait()` на асинхронной операции внутри тела `Parallel.For` → thread-pool starvation и дедлок. Перепишите на `Task.WhenAll`/`Channel`.
- Отсутствие `CancellationToken` → нельзя остановить «зависший» цикл. Передавайте `ParallelOptions.CancellationToken`.
- `Parallel.ForEach` над `IQueryable` / отложенным LINQ → материализуйте `ToList()` заранее, иначе каждый воркер выполнит запрос заново.
- `MaxDegreeOfParallelism = Environment.ProcessorCount` для I/O-bound работы → пул потоков простаивает. Для I/O используйте `SemaphoreSlim` + async, не `Parallel`.
- Изменение общей коллекции `List<T>` из тела цикла → `InvalidOperationException` или порча данных. Используйте `ConcurrentBag`/`ConcurrentDictionary`.
- `loopState.Break()` вместо `Stop()` когда нужен немедленный стоп → лишние итерации. `Stop` = немедленно, `Break` = завершить нижние индексы.

- Shared `sum += x` without `Interlocked`/`lock` → unpredictable result. Use `localInit`/`localFinally` or `Interlocked.Add`.
- `.Result`/`.Wait()` on an async operation inside `Parallel.For` body → thread-pool starvation and deadlock. Rewrite with `Task.WhenAll`/`Channel`.
- Missing `CancellationToken` → cannot stop a "hung" loop. Pass `ParallelOptions.CancellationToken`.
- `Parallel.ForEach` over `IQueryable` / deferred LINQ → materialize `ToList()` first, or each worker re-runs the query.
- `MaxDegreeOfParallelism = Environment.ProcessorCount` for I/O-bound work → pool threads sit idle. For I/O use `SemaphoreSlim` + async, not `Parallel`.
- Mutating a shared `List<T>` from the loop body → `InvalidOperationException` or data corruption. Use `ConcurrentBag`/`ConcurrentDictionary`.
- `loopState.Break()` instead of `Stop()` when you need immediate stop → extra iterations. `Stop` = immediate, `Break` = finish lower indices.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Тело цикла не пишет в общую переменную без `Interlocked`/`lock`/thread-local аккумулятора.
- [ ] В `ParallelOptions` передан `CancellationToken` и он реагирует за ≤ 1 итерацию.
- [ ] `MaxDegreeOfParallelism` выбран осознанно (CPU = ядра, I/O = ограничение ресурса).
- [ ] Нет `.Result`/`.Wait()`/`async void` внутри тела `Parallel`.
- [ ] Коллекция итерации материализована (не отложенный `IQueryable`).
- [ ] Для агрегаций используется `localInit`/`localFinally`, а не внешний `lock`.
- [ ] Для неравномерной нагрузки применён `Partitioner.Create`.
- [ ] Проверено: тест запускает цикл, отменяет его через 100 мс и ловит `OperationCanceledException`.

- [ ] The loop body does not write to a shared variable without `Interlocked`/`lock`/thread-local accumulator.
- [ ] `ParallelOptions` carries a `CancellationToken` that responds within ≤ 1 iteration.
- [ ] `MaxDegreeOfParallelism` is chosen deliberately (CPU = cores, I/O = resource cap).
- [ ] No `.Result`/`.Wait()`/`async void` inside the `Parallel` body.
- [ ] The iterated collection is materialized (not a deferred `IQueryable`).
- [ ] Aggregations use `localInit`/`localFinally`, not an external `lock`.
- [ ] For uneven workloads, `Partitioner.Create` is applied.
- [ ] Verified: a test starts the loop, cancels after 100 ms, and catches `OperationCanceledException`.

#### Ресурсы / Resources

- [Microsoft Learn — How to: Write a Simple Parallel.For Loop — https://learn.microsoft.com/dotnet/standard/parallel-programming/how-to-write-a-simple-parallel-for-loop](https://learn.microsoft.com/dotnet/standard/parallel-programming/how-to-write-a-simple-parallel-for-loop)
- [Microsoft Learn — How to: Write a Simple Parallel.ForEach Loop — https://learn.microsoft.com/dotnet/standard/parallel-programming/how-to-write-a-simple-parallel-foreach-loop](https://learn.microsoft.com/dotnet/standard/parallel-programming/how-to-write-a-simple-parallel-foreach-loop)
- [Microsoft Learn — How to: Cancel a Parallel.For or ForEach Loop — https://learn.microsoft.com/dotnet/standard/parallel-programming/how-to-cancel-a-parallel-for-or-foreach-loop](https://learn.microsoft.com/dotnet/standard/parallel-programming/how-to-cancel-a-parallel-for-or-foreach-loop)
- [Microsoft Learn — Partitioner.Create — https://learn.microsoft.com/dotnet/api/system.collections.concurrent.partitioner.create](https://learn.microsoft.com/dotnet/api/system.collections.concurrent.partitioner.create)
- [Microsoft Learn — PLINQ Overview — https://learn.microsoft.com/dotnet/standard/parallel-programming/parallel-linq-plinq](https://learn.microsoft.com/dotnet/standard/parallel-programming/parallel-linq-plinq)

---

[⬆ К модулю M11](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
