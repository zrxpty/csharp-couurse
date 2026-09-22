[⬆ К модулю M11](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M11-L06: ConcurrentDictionary, ConcurrentQueue, ConcurrentBag / ConcurrentDictionary, ConcurrentQueue, ConcurrentBag

**Модуль / Module:** M11
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Когда несколько потоков одновременно читают и пишут в обычную коллекцию (`Dictionary<TKey,TValue>`, `Queue<T>`, `List<T>`), вы получаете race condition: повреждённое внутреннее состояние, пропущенные элементы, дубликаты, `InvalidOperationException` («Collection was modified») или бесконечный цикл в `GetHashCode`. Простейшее лекарство — обернуть всё в `lock` (мьютекс). Но `lock` сериализует доступ: даже читатели ждут друг друга, и масштабирование на 16 ядрах превращается в очередь на кассу.

`System.Collections.Concurrent` предлагает коллекции, спроектированные для параллельного доступа. Они используют fine-grained синхронизацию: вместо одного замка на всю коллекцию — много мелких замков (striping) или lock-free алгоритмы (CAS через `Interlocked.CompareExchange`). Аналогия: вместо одной двери с охранником в супермаркет — много касс с самообслуживанием.

**ConcurrentDictionary<TKey,TValue>** — самый мощный член семейства. Чтения (`TryGetValue`, индексатор-get, `ContainsKey`) полностью lock-free и никогда не блокируются. Записи берут мелкий замок только на одну «полку» (bucket). Но главная ценность — атомарные фабричные методы `GetOrAdd` и `AddOrUpdate`. Рассмотрите счётчик запросов по IP: наивный код `if (!dict.ContainsKey(ip)) dict[ip] = 0; dict[ip]++;` — классическая гонка: два потока одновременно видят отсутствие ключа, оба пишут 0, потом оба инкрементируют 1 — потерян апдейт. `GetOrAdd(key, k => 0)` решает только создание; для инкремента нужен `AddOrUpdate` либо `GetOrAdd` + `TryUpdate` в цикле (CAS-цикл). Важно понимать: фабрика `valueFactory` может вызваться несколько раз, даже если значение в коллекцию попадёт лишь однажды. Поэтому фабрика должна быть чистой и без побочных эффектов (нет записи в БД, нет отправки email внутри фабрики). Для side-effecting создания используйте перегрузку с `ConcurrentDictionary` и ручным циклом, либо `Lazy<T>`.

**ConcurrentQueue<T>** и **ConcurrentStack<T>** — lock-free FIFO/LIFO на базе связанных узлов. Идеальны для producer/consumer: несколько производителей `Enqueue`, несколько потребителей `TryDequeue`. `IProducerConsumerCollection<T>` — общий интерфейс: `TryAdd`/`TryTake`. На нём построены `BlockingCollection` (оборачивает любую `IProducerConsumerCollection` и добавляет блокирующий `Add`/`Take` с `CancellationToken`) и `BlockingCollection` как bounded-буфер.

**ConcurrentBag<T>** — неупорядоченная коллекция с thread-local хранилищем. Каждый поток пишет в свой локальный стек (быстро, без замков), а при опустошении «крадёт» работу из хвостов других потоков (work-stealing). Оптимальна для `Parallel.ForEach` и сценариев «произвёл — собрал позже», где порядок не важен и один поток обычно и производит, и потребляет свои элементы.

**Когда concurrent, а когда lock?** Concurrent-коллекции проигрывают обычным при низком содержании (1–2 потока) — оверхед на CAS дороже простого `lock`. Но при высоком параллелизме они масштабируются линейно. Правило: если коллекция разделяется между ≥3 потоками с частыми обновлениями — concurrent. Если только читается — `Dictionary` (он thread-safe для read-only, если никто не пишет). Если нужна сложная составная операция («если A и B, то удалить C, добавить D») — никакая concurrent-коллекция не даст атомарности всей группы: берите `lock` или `Channel<T>` для pipeline-модели.

Классические ловушки: `.Result`/`.Wait()` на async-фабрике внутри `GetOrAdd` — deadlock в контексте синхронизации (ASP.NET classic, WinForms, WPF). `async void` — исключения рвут процесс. `lock (obj) { await ... }` — компилятор запрещает, но `Monitor.Enter` + `await` через `SemaphoreSlim` — частая ошибка: замок не освобождается во время await. Блокирование потока пула (`Task.Run(() => blockingCall())`) при насыщении пула → thread-pool starvation. Лечение: async-фабрики без `.Result`, `SemaphoreSlim.WaitAsync`, `Channel<T>` вместо `BlockingCollection` для async pipeline (урок M11-L07).

#### Theory (EN)

When multiple threads simultaneously read and write a plain collection (`Dictionary<TKey,TValue>`, `Queue<T>`, `List<T>`), you get a race condition: corrupted internal state, dropped elements, duplicates, `InvalidOperationException` ("Collection was modified"), or an infinite loop inside `GetHashCode`. The simplest cure is to wrap everything in a `lock` (a mutex). But `lock` serializes access: even readers wait for each other, so scaling on 16 cores degrades into a single checkout line.

`System.Collections.Concurrent` ships collections designed for parallel access. They use fine-grained synchronization: instead of one lock for the whole collection — many small locks (striping) or lock-free algorithms (CAS via `Interlocked.CompareExchange`). Analogy: instead of one guarded door at a supermarket — many self-service checkouts.

**ConcurrentDictionary<TKey,TValue>** is the most powerful member. Reads (`TryGetValue`, the indexer getter, `ContainsKey`) are fully lock-free and never block. Writes take a tiny lock on a single bucket. The real value, though, is the atomic factory methods `GetOrAdd` and `AddOrUpdate`. Consider a per-IP request counter: the naive code `if (!dict.ContainsKey(ip)) dict[ip] = 0; dict[ip]++;` is a classic race — two threads simultaneously see the key missing, both write 0, then both increment to 1 — an update is lost. `GetOrAdd(key, k => 0)` fixes only creation; for the increment you need `AddOrUpdate` or a `GetOrAdd` + `TryUpdate` CAS loop. Critical understanding: the `valueFactory` may run more than once even though only one value lands in the dictionary. So the factory must be pure and side-effect-free (no DB writes, no emails inside the factory). For side-effecting creation, use an overload with a manual loop or `Lazy<T>`.

**ConcurrentQueue<T>** and **ConcurrentStack<T>** are lock-free FIFO/LIFO built on linked nodes. Ideal for producer/consumer: many producers `Enqueue`, many consumers `TryDequeue`. `IProducerConsumerCollection<T>` is the shared interface: `TryAdd`/`TryTake`. On top of it sit `BlockingCollection` (wraps any `IProducerConsumerCollection` and adds blocking `Add`/`Take` with `CancellationToken`) and bounded-buffer patterns.

**ConcurrentBag<T>** is an unordered collection with thread-local storage. Each thread writes to its own local stack (fast, no locks), and when empty, steals work from other threads' tails (work-stealing). Optimal for `Parallel.ForEach` and "produce now, gather later" scenarios where order does not matter and one thread usually produces and consumes its own items.

**When concurrent, when lock?** Concurrent collections lose to plain ones at low contention (1–2 threads) — CAS overhead costs more than a simple `lock`. But under high parallelism they scale linearly. Rule of thumb: if a collection is shared across ≥3 threads with frequent updates — go concurrent. If it is only read — `Dictionary` is thread-safe for read-only as long as nobody writes. If you need a complex compound operation ("if A and B then remove C, add D"), no concurrent collection gives you atomicity across the group: use `lock`, or model the workflow as a `Channel<T>` pipeline.

Classic traps: `.Result`/`.Wait()` on an async factory inside `GetOrAdd` — deadlock under a synchronization context (classic ASP.NET, WinForms, WPF). `async void` — exceptions tear down the process. `lock (obj) { await ... }` is forbidden by the compiler, but `Monitor.Enter` + `await` via `SemaphoreSlim` is a frequent mistake: the lock is not released during the await. Blocking pool threads (`Task.Run(() => blockingCall())`) under saturation leads to thread-pool starvation. The fix: async factories without `.Result`, `SemaphoreSlim.WaitAsync`, and `Channel<T>` instead of `BlockingCollection` for async pipelines (lesson M11-L07).

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8+ — рабочие concurrency-паттерны / working concurrency patterns
using System.Collections.Concurrent;
using System.Diagnostics;

// ───────────────────────────────────────────────────────────────────────────
// 1. ConcurrentDictionary: атомарный счётчик через AddOrUpdate + CAS-цикл
//    Atomic counter via AddOrUpdate and a CAS (TryUpdate) retry loop.
// ───────────────────────────────────────────────────────────────────────────
var hitsByIp = new ConcurrentDictionary<string, int>();

// Плохо / Wrong — гонка: ContainsKey + Write + ++ не атомарны как группа
// Bad — race: ContainsKey + Write + ++ are not atomic as a group

// Хорошо / Good: AddOrUpdate с чистой фабрикой (pure factory, no side effects)
int Increment(string ip) =>
    hitsByIp.AddOrUpdate(
        key: ip,
        addValueFactory: static _ => 1,                       // ключа нет → 1
        updateValueFactory: static (_, current) => current + 1 // ключ есть → +1
    );

// CAS-цикл через GetOrAdd + TryUpdate, если нужно прочитать-вычислить-записать
// CAS loop via GetOrAdd + TryUpdate for read-compute-write
bool TryMultiply(string ip, int factor)
{
    while (hitsByIp.TryGetValue(ip, out int current))
    {
        int next = current * factor;
        // TryUpdate атомарно: сравнивает current и лишь тогда пишет next
        // TryUpdate is atomic: compares current, then writes next
        if (hitsByIp.TryUpdate(ip, next, comparisonValue: current))
            return true;
        // кто-то изменил значение между Read и Update — повторим
        // someone changed it between Read and Update — retry
    }
    return false; // ключа нет / key missing
}

// ───────────────────────────────────────────────────────────────────────────
// 2. ConcurrentQueue: producer/consumer + CancellationToken
//    Lock-free FIFO pipeline with cooperative cancellation.
// ───────────────────────────────────────────────────────────────────────────
var queue = new ConcurrentQueue<int>();
using var cts = new CancellationTokenSource();

var producer = Task.Run(async () =>
{
    for (int i = 0; i < 1000; i++)
    {
        cts.Token.ThrowIfCancellationRequested();
        queue.Enqueue(i);
        await Task.Delay(1, cts.Token); // имитация работы / simulate work
    }
}, cts.Token);

var consumer = Task.Run(async () =>
{
    while (!cts.IsCancellationRequested)
    {
        if (queue.TryDequeue(out int item))
        {
            // обработка элемента / process item
            _ = item;
        }
        else
        {
            // очередь пуста — НЕ блокируем поток пула Thread.Sleep!
            // Queue empty — do NOT block a pool thread with Thread.Sleep!
            await Task.Delay(2, cts.Token); // async-yield, освобождает поток
        }
    }
}, cts.Token);

await Task.WhenAll(producer, consumer);
cts.Cancel(); // мягкая остановка / graceful shutdown

// ───────────────────────────────────────────────────────────────────────────
// 3. ConcurrentBag + Parallel.ForEach: gather results, order не важен
//    Work-stealing bag; order is not guaranteed.
// ───────────────────────────────────────────────────────────────────────────
var results = new ConcurrentBag<long>();

Parallel.For(0, 10_000, i =>
{
    // каждый поток пишет в свой thread-local стек → без замков
    // each thread writes to its own thread-local stack → no locks
    results.Add(ComputeFib(i % 30));
});

long ComputeFib(int n) => n < 2 ? n : ComputeFib(n - 1) + ComputeFib(n - 2);

// ───────────────────────────────────────────────────────────────────────────
// 4. IProducerConsumerCollection + BlockingCollection (bounded buffer)
//    Bounded pipeline with backpressure and cancellation.
// ───────────────────────────────────────────────────────────────────────────
// BlockingCollection оборачивает ConcurrentQueue и добавляет блокирующий Add/Take
// BlockingCollection wraps ConcurrentQueue and adds blocking Add/Take
using var bounded = new BlockingCollection<int>(boundedCapacity: 50);

var bf = Task.Run(() =>
{
    foreach (var x in Enumerable.Range(0, 1000))
    {
        // Add блокирует при переполнении → backpressure на продюсера
        // Add blocks when full → backpressure to producer
        bounded.Add(x, cts.Token);
    }
    bounded.CompleteAdding(); // сигнал: больше элементов не будет
});

var bc = Task.Run(() =>
{
    // GetConsumingEnumerable сам выходит из цикла после CompleteAdding
    // GetConsumingEnumerable exits automatically after CompleteAdding
    foreach (var x in bounded.GetConsumingEnumerable(cts.Token))
        _ = x;
});

await Task.WhenAll(bf, bc);

// ───────────────────────────────────────────────────────────────────────────
// 5. ⭐ Анти-паттерны (НЕ делайте так) / Anti-patterns (do NOT do this)
// ───────────────────────────────────────────────────────────────────────────
// ❌ .Result внутри фабрики → deadlock в контексте синхронизации
// ❌ .Result inside factory → deadlock under a sync context
//    hitsByIp.GetOrAdd(ip, k => LoadFromDbAsync(k).Result); // deadlock risk!

// ❌ async void — исключения рвут процесс / tears down the process
//    async void Handle() { await ... } // never catchable

// ❌ lock + await — компилятор запретит, но SemaphoreSlim.Wait + await — баг:
// ❌ lock + await — compiler forbids it; but SemaphoreSlim.Wait + await is a bug:
//    sem.Wait(); try { await DoAsync(); } finally { sem.Release(); } // holds thread!

// ✅ правильно / correct: асинхронный замок / async lock
//    await sem.WaitAsync(cts.Token); try { await DoAsync(); } finally { sem.Release(); }

// ───────────────────────────────────────────────────────────────────────────
// 6. ⭐ Когда выбрать Channel<T> (см. урок M11-L07) / When to choose Channel<T>
// ───────────────────────────────────────────────────────────────────────────
// Для async producer/consumer pipeline Channel<T> лучше BlockingCollection:
// For async producer/consumer pipelines, Channel<T> beats BlockingCollection:
//   • нет блокировки потока пула — только async await
//   • no pool-thread blocking — pure async/await
//   • встроенная backpressure через BoundedChannelOptions
//   • built-in backpressure via BoundedChannelOptions
//   • thread-safe WriteAsync/ReadAllAsync с CancellationToken
//   • thread-safe WriteAsync/ReadAllAsync with CancellationToken
```

#### Best Practices

- Выбирайте concurrent-коллекцию под сценарий: `ConcurrentDictionary` — разделяемый кэш/счётчики, `ConcurrentQueue`/`Stack` — FIFO/LIFO pipeline, `ConcurrentBag` — неупорядоченный сбор в `Parallel.ForEach`.
- Держите фабрику `GetOrAdd`/`AddOrUpdate` чистой и быстрой: она может вызваться несколько раз, побочные эффекты (БД, email, I/O) — отдельно.
- Для сложных атомарных групп операций используйте `lock` или моделируйте pipeline через `Channel<T>` — concurrent-коллекция атомарна только на одну операцию.
- Передавайте `CancellationToken` во все блокирующие/async-вызовы (`Add`, `Take`, `GetConsumingEnumerable`, `WaitAsync`) для мягкой остановки.
- Не блокируйте поток пула `Thread.Sleep`/`.Wait()` в consumer-цикле — используйте `await Task.Delay` или `Channel<T>.ReadAllAsync`.

#### Best Practices (EN)

- Match the concurrent collection to the scenario: `ConcurrentDictionary` for shared cache/counters, `ConcurrentQueue`/`Stack` for FIFO/LIFO pipelines, `ConcurrentBag` for unordered gathering in `Parallel.ForEach`.
- Keep the `GetOrAdd`/`AddOrUpdate` factory pure and fast: it may run more than once, so move side effects (DB, email, I/O) elsewhere.
- For complex atomic groups of operations use `lock` or model the workflow as a `Channel<T>` pipeline — a concurrent collection is atomic only per single operation.
- Thread a `CancellationToken` through every blocking/async call (`Add`, `Take`, `GetConsumingEnumerable`, `WaitAsync`) for graceful shutdown.
- Never block a pool thread with `Thread.Sleep`/`.Wait()` in a consumer loop — use `await Task.Delay` or `Channel<T>.ReadAllAsync`.

#### Частые ошибки / Common Mistakes

- `ContainsKey` + запись + инкремент как три отдельных шага → гонка и потеря апдейтов → используйте `AddOrUpdate` или `GetOrAdd` + `TryUpdate` CAS-цикл. (RU)
- Побочный эффект (запись в БД, лог, email) внутри `valueFactory` `GetOrAdd` → выполняется 0..N раз, возможны дубликаты → фабрика должна быть чистой, side-effect выносится наружу. (RU)
- `GetOrAdd(key, k => AsyncCall(k).Result)` → deadlock в контексте синхронизации → используйте синхронную фабрику или `Lazy<Task<T>>` с отдельным ожиданием. (RU)
- `lock (obj) { await ... }` или `Monitor.Enter` + `await` → замок держится во время await, deadlock/starvation → используйте `SemaphoreSlim.WaitAsync`. (RU)
- `Thread.Sleep`/`.Wait()` в consumer-цикле → истощение пула потоков (starvation) → `await Task.Delay` или `Channel<T>.ReadAllAsync`. (RU)
- Перебор `ConcurrentDictionary` через `foreach` во время записи → нет исключения, но можно увидеть промежуточное состояние → нормально для eventual-consistency, но не предполагайте точный снимок. (RU)
- `async void` для обработчиков событий-задач → необработанное исключение рвёт процесс → `async Task` + `await`/обработка. (RU)

- `ContainsKey` + write + increment as three separate steps → race and lost updates → use `AddOrUpdate` or a `GetOrAdd` + `TryUpdate` CAS loop. (EN)
- Side effect (DB write, log, email) inside `GetOrAdd`'s `valueFactory` → runs 0..N times, duplicates possible → keep the factory pure, move side effects outside. (EN)
- `GetOrAdd(key, k => AsyncCall(k).Result)` → deadlock under a sync context → use a synchronous factory or `Lazy<Task<T>>` awaited separately. (EN)
- `lock (obj) { await ... }` or `Monitor.Enter` + `await` → the lock is held across the await, deadlock/starvation → use `SemaphoreSlim.WaitAsync`. (EN)
- `Thread.Sleep`/`.Wait()` in a consumer loop → thread-pool starvation → `await Task.Delay` or `Channel<T>.ReadAllAsync`. (EN)
- Iterating a `ConcurrentDictionary` with `foreach` while writers are active → no exception, but you may see an intermediate state → fine for eventual consistency, but do not assume a precise snapshot. (EN)
- `async void` for task event handlers → an unhandled exception tears down the process → use `async Task` + `await`/handling. (EN)

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я выбрал правильную concurrent-коллекцию под сценарий (dict/cache, queue/pipeline, bag/gather). (RU)
- [ ] Я использую `AddOrUpdate`/`GetOrAdd` + `TryUpdate` вместо `ContainsKey` + write для атомарности. (RU)
- [ ] Фабрика `GetOrAdd` чистая, без побочных эффектов и без `.Result`. (RU)
- [ ] `CancellationToken` передаётся во все блокирующие/async-вызовы. (RU)
- [ ] В consumer-цикле нет `Thread.Sleep`/`.Wait()` — только async-yield или `Channel<T>`. (RU)
- [ ] `lock` не используется вместе с `await`; для async-замка — `SemaphoreSlim.WaitAsync`. (RU)
- [ ] Для async pipeline я выбрал `Channel<T>` (M11-L07), а не `BlockingCollection`. (RU)

- [ ] I picked the right concurrent collection for the scenario (dict/cache, queue/pipeline, bag/gather). (EN)
- [ ] I use `AddOrUpdate`/`GetOrAdd` + `TryUpdate` instead of `ContainsKey` + write for atomicity. (EN)
- [ ] The `GetOrAdd` factory is pure, side-effect-free, and has no `.Result`. (EN)
- [ ] `CancellationToken` is threaded through every blocking/async call. (EN)
- [ ] The consumer loop has no `Thread.Sleep`/`.Wait()` — only async-yield or `Channel<T>`. (EN)
- [ ] `lock` is never combined with `await`; for an async lock I use `SemaphoreSlim.WaitAsync`. (EN)
- [ ] For an async pipeline I chose `Channel<T>` (M11-L07) over `BlockingCollection`. (EN)

#### Ресурсы / Resources

- [Microsoft Learn — Thread-safe collections](https://learn.microsoft.com/dotnet/standard/collections/thread-safe/)
- [Microsoft Learn — ConcurrentDictionary<TKey,TValue>](https://learn.microsoft.com/dotnet/api/system.collections.concurrent.concurrentdictionary-2)
- [Microsoft Learn — BlockingCollection overview](https://learn.microsoft.com/dotnet/standard/collections/thread-safe/blockingcollection-overview)
- [Microsoft Learn — System.Threading.Channels](https://learn.microsoft.com/dotnet/api/system.threading.channels)

---

[⬆ К модулю M11](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
