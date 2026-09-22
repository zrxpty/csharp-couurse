[⬆ К модулю M11](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M11-L07: Channel<T>, producer/consumer / Channel<T>, producer/consumer

**Модуль / Module:** M11
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

`System.Threading.Channels` — это высокопроизводительная структура данных для обмена сообщениями между производителями (producer) и потребителями (consumer) в многопоточных сценариях. Канал — это потокобезопасная очередь, оптимизированная под асинхронное использование: он не использует тяжёлые блокировки (`lock` + `Monitor`), а построен на `SemaphoreSlim`, `SpinWait` и безблочных стратегиях на основе `volatile`/`Interlocked`. В отличие от `ConcurrentQueue<T>`, канал знает, что он «закончился» (`Completion`), и умеет ждать данных без блокировки потока (`WaitToReadAsync`).

**Аналогия:** представь конвейер на фабрике. Рабочий слева кладёт детали на ленту (producer), рабочие справа забирают их и обрабатывают (consumers). Лента ограниченной длины — это `Channel.CreateBounded`. Если лента заполнена, производитель обязан подождать (`WaitToWriteAsync`), иначе детали упадут на пол — это и есть **backpressure** (обратное давление). Лента бесконечной длины — `Channel.CreateUnbounded`, но её память может переполниться, если потребитель не справляется.

**Два режима создания:**
- `Channel.CreateUnbounded<T>()` — без лимита. Память растёт без ограничений; используй только когда производитель стабильно медленнее потребителя, или когда объём предсказуем.
- `Channel.CreateBounded<T>(capacity)` — с лимитом. Когда лимит достигнут, поведение по умолчанию — `BoundedChannelFullMode.Wait` (producer ждёт через `WaitToWriteAsync`). Другие режимы: `DropWrite` (молча отбросить новое), `DropOldest` (выкинуть старое), `DropNewest` (выкинуть новое). `Wait` — самый безопасный для целостности данных.

**Ключевые API:**
- `Channel.Reader` → `ChannelReader<T>`: `WaitToReadAsync(ct)`, `ReadAllAsync(ct)`, `TryRead`, `TryPeek`, `Completion`.
- `Channel.Writer` → `ChannelWriter<T>`: `WriteAsync(item, ct)` (асинхронная запись с учётом backpressure), `TryWrite(item)` (мгновенная, вернёт `false` при переполнении), `WaitToWriteAsync(ct)`, `Complete()`, `TryComplete(ex)`.

**Producer/consumer pipeline.** Классический сценарий: один или несколько producer'ов пишут в канал, один или несколько consumer'ов читают. После завершения производитель вызывает `writer.Complete()`. Потребитель, читающий через `await foreach (var item in reader.ReadAllAsync(ct))`, автоматически выходит из цикла, когда канал завершён и опустошён. Это **race-free** завершение: не нужно вручную синхронизировать «все ли producer'ы закончили».

**Backpressure и RACE-free pipeline.** Главная ценность bounded-канала — автоматическое противодавление. Если consumer медленный, `WriteAsync` producer'а «приостанавливается» (возвращает незавершённую `Task`), не потребляя поток. Поток освобождается обратно в пул. Это предотвращает runaway memory growth и threading starvation одновременно. В pipeline из нескольких стадий (читать → парсить → сохранять) каждая стадия связана своим каналом; медленная стадия автоматически замедляет всю цепочку — это **RACE-free**: нет состояний гонки, нет дедлоков, если соблюдать правила.

**Правила, чтобы не словить дедлок или race:**
1. Никогда не вызывай `.Result` / `.Wait()` на `WriteAsync`/`WaitToReadAsync` — это блокирует поток пула и при насыщении даёт классический дедлок sync-over-async.
2. Передавай `CancellationToken` во все async-операции канала, иначе pipeline нельзя остановить.
3. Вызывай `writer.Complete()` в `finally` или через `try/finally`/`using`, иначе `reader.Completion` зависнет навсегда.
4. Не используй `lock` вокруг `await` — канал в `lock` не нуждается, он сам потокобезопасен.
5. Для нескольких producer'ов используй счётчик (`Interlocked.Decrement`) и вызывай `Complete()` только когда последний завершится — иначе ранний `Complete()` оборвёт запись остальных.
6. `await reader.Completion` — точка джойна всех consumer'ов; оборачивай в `try/finally` для корректной обработки `OperationCanceledException`.

**ConfigureAwait(false)** в библиотечном коде снижает переключения контекста синхронизации; в UI/ASP.NET Classic обязательно, в ASP.NET Core — безразлично (нет `SynchronizationContext`), но не вредит. Каналы дружат с отменой: при `ct.Cancel()` все ожидающие `WaitToReadAsync`/`WriteAsync` выбрасывают `OperationCanceledException`.

#### Theory (EN)

`System.Threading.Channels` is a high-performance data structure for passing messages between producers and consumers in multithreaded scenarios. A channel is a thread-safe queue tuned for asynchronous use: it does not rely on heavyweight `lock` + `Monitor` pairs, but on `SemaphoreSlim`, `SpinWait`, and lock-free strategies built around `volatile`/`Interlocked`. Unlike `ConcurrentQueue<T>`, a channel knows when it is "done" (`Completion`) and can wait for data without blocking a thread (`WaitToReadAsync`).

**Analogy:** picture a factory conveyor belt. A worker on the left places parts onto the belt (producer); workers on the right pick them up and process them (consumers). A belt of finite length is `Channel.CreateBounded`. When the belt is full, the producer must wait (`WaitToWriteAsync`); otherwise parts would fall on the floor — this is **backpressure**. A belt of infinite length is `Channel.CreateUnbounded`, but its memory can grow unbounded if the consumer cannot keep up.

**Two creation modes:**
- `Channel.CreateUnbounded<T>()` — no limit. Memory grows without bounds; use only when the producer is reliably slower than the consumer, or the volume is predictable.
- `Channel.CreateBounded<T>(capacity)` — a hard cap. When the cap is reached, the default behavior is `BoundedChannelFullMode.Wait` (the producer waits via `WaitToWriteAsync`). Other modes: `DropWrite` (silently discard the new item), `DropOldest` (evict the oldest), `DropNewest` (evict the newest). `Wait` is the safest for data integrity.

**Core API:**
- `Channel.Reader` → `ChannelReader<T>`: `WaitToReadAsync(ct)`, `ReadAllAsync(ct)`, `TryRead`, `TryPeek`, `Completion`.
- `Channel.Writer` → `ChannelWriter<T>`: `WriteAsync(item, ct)` (async write honoring backpressure), `TryWrite(item)` (synchronous, returns `false` when full), `WaitToWriteAsync(ct)`, `Complete()`, `TryComplete(ex)`.

**Producer/consumer pipeline.** Classic scenario: one or more producers write into the channel, one or more consumers read. When done, the producer calls `writer.Complete()`. A consumer iterating with `await foreach (var item in reader.ReadAllAsync(ct))` automatically exits the loop once the channel is completed and drained. This is **race-free** completion: you do not need to manually synchronize "have all producers finished?".

**Backpressure and a race-free pipeline.** The main value of a bounded channel is automatic backpressure. If the consumer is slow, the producer's `WriteAsync` "pauses" (returns an incomplete `Task`) and frees the thread back to the pool. This prevents both runaway memory growth and thread-pool starvation at once. In a multi-stage pipeline (read → parse → save) each stage is linked by its own channel; a slow stage automatically throttles the whole chain — this is **race-free**: no data races, no deadlocks, provided you follow the rules.

**Rules to avoid deadlocks and races:**
1. Never call `.Result` or `.Wait()` on `WriteAsync`/`WaitToReadAsync` — it blocks a pool thread and under saturation yields the classic sync-over-async deadlock.
2. Pass a `CancellationToken` into every async channel operation; otherwise the pipeline cannot be stopped.
3. Always call `writer.Complete()` in a `finally`/`using` block, or `reader.Completion` will hang forever.
4. Never hold a `lock` across an `await` — the channel needs no `lock`, it is already thread-safe.
5. With multiple producers, use a counter (`Interlocked.Decrement`) and call `Complete()` only when the last one finishes — an early `Complete()` truncates the others.
6. `await reader.Completion` is the join point for all consumers; wrap it in `try/finally` to handle `OperationCanceledException` correctly.

**ConfigureAwait(false)** reduces synchronization-context switches in library code; in UI/ASP.NET Classic it is mandatory, in ASP.NET Core it is a no-op (no `SynchronizationContext`) but harmless. Channels cooperate with cancellation: when `ct.Cancel()` fires, every pending `WaitToReadAsync`/`WriteAsync` throws `OperationCanceledException`. Combine channels with `Task.Run` for CPU-bound stages and pure `async` for I/O-bound stages — that mix is where pipelines shine.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8+ — Рабочий producer/consumer pipeline с backpressure / Working producer/consumer pipeline with backpressure
using System.Threading.Channels;

// 1) Создаём ограниченный канал вместимостью 64 элемента / Create a bounded channel with capacity 64.
//    BoundedChannelFullMode.Wait даёт backpressure: producer будет ждать, а не отбрасывать.
//    BoundedChannelFullMode.Wait gives backpressure: producer waits instead of dropping.
var channelOpts = new BoundedChannelOptions(capacity: 64)
{
    FullMode = BoundedChannelFullMode.Wait,
    SingleReader = false,           // несколько потребителей / multiple consumers
    SingleWriter = false,           // несколько производителей / multiple producers
};

Channel<WorkItem> channel = Channel.CreateBounded<WorkItem>(channelOpts);
ChannelReader<WorkItem> reader = channel.Reader;
ChannelWriter<WorkItem> writer = channel.Writer;

// 2) CancellationToken — единая точка остановки всего pipeline / Single cancellation point for the pipeline.
using var cts = new CancellationTokenSource();
CancellationToken ct = cts.Token;

// 3) Producer'ы. Несколько задач пишут одновременно. Счётчик гарантирует,
//    что Complete() вызовется только после последнего producer'а — иначе race:
//    ранний Complete() оборвёт запись остальных.
//    Producers. Multiple tasks write concurrently. The counter guarantees Complete()
//    is called only after the last producer — otherwise a race: early Complete() truncates the rest.
const int ProducerCount = 3;
int remainingProducers = ProducerCount;

var producers = Enumerable.Range(0, ProducerCount).Select(producerId => Task.Run(async () =>
{
    try
    {
        for (int i = 0; i < 100; i++)
        {
            var item = new WorkItem(producerId, i);

            // WriteAsync учитывает backpressure: при переполнении await приостановит
            // этот поток без блокировки пула. Никакого .Result / .Wait()!
            // WriteAsync honors backpressure: when full, await suspends without blocking the pool. No .Result!
            await writer.WriteAsync(item, ct).ConfigureAwait(false);

            // Имитация работы производителя / Simulate producer work.
            await Task.Delay(Random.Shared.Next(1, 5), ct).ConfigureAwait(false);
        }
    }
    finally
    {
        // Последний producer закрывает канал / Last producer closes the channel.
        if (Interlocked.Decrement(ref remainingProducers) == 0)
        {
            writer.TryComplete(); // TryComplete безопасен: повторные вызовы игнорируются / idempotent.
        }
    }
}, ct)).ToArray();

// 4) Consumer'ы. ReadAllAsync — race-free: автоматически завершается, когда канал
//    закрыт и опустошён. Никакой ручной проверки "все ли producer'ы закончили".
//    Consumers. ReadAllAsync is race-free: exits automatically when channel is completed and drained.
const int ConsumerCount = 2;
var consumers = Enumerable.Range(0, ConsumerCount).Select(consumerId => Task.Run(async () =>
{
    long processed = 0;
    try
    {
        // await foreach корректно пробрасывает OperationCanceledException при отмене.
        // await foreach propagates OperationCanceledException on cancellation.
        await foreach (WorkItem item in reader.ReadAllAsync(ct).ConfigureAwait(false))
        {
            await ProcessAsync(item, consumerId, ct).ConfigureAwait(false);
            processed++;
        }
    }
    finally
    {
        Console.WriteLine($"Consumer {consumerId} processed {processed} items.");
    }
}, ct)).ToArray();

// 5) Точка джойна: ждём producer'ов, затем consumer'ов / Join point: await producers, then consumers.
try
{
    await Task.WhenAll(producers).ConfigureAwait(false);   // все producer'ы завершились и закрыли канал
    await Task.WhenAll(consumers).ConfigureAwait(false);   // все consumer'ы доработали хвост и вышли из цикла
    Console.WriteLine("Pipeline completed cleanly.");
}
catch (OperationCanceledException)
{
    Console.WriteLine("Pipeline cancelled.");
}
finally
{
    cts.Dispose();
}

// ---- Вспомогательные типы и методы / Helper types and methods ----

static async Task ProcessAsync(WorkItem item, int consumerId, CancellationToken ct)
{
    // Имитация I/O-bound обработки / Simulate I/O-bound work.
    await Task.Delay(Random.Shared.Next(2, 8), ct).ConfigureAwait(false);
    Console.WriteLine($"  [c{consumerId}] processed {item}");
}

readonly record struct WorkItem(int ProducerId, int Sequence);

// 6) Альтернатива: ручной цикл с WaitToReadAsync + TryRead (полезно для тонкого контроля).
//    Alternative: manual loop with WaitToReadAsync + TryRead (finer control).
//
// while (await reader.WaitToReadAsync(ct).ConfigureAwait(false))
// {
//     while (reader.TryRead(out WorkItem item))
//     {
//         await ProcessAsync(item, consumerId, ct).ConfigureAwait(false);
//     }
// }
// // Выход из внешнего while означает, что канал Complete и пуст — race-free завершение.
// // Exiting the outer while means the channel is Complete and empty — race-free termination.

// 7) DropWrite — обработка переполнения без блокировки producer'а.
//    DropWrite — overflow handling without blocking the producer.
//
// var dropChannel = Channel.CreateBounded<int>(
//     new BoundedChannelOptions(8) { FullMode = BoundedChannelFullMode.DropOldest });
// // TryWrite всегда возвращается быстро; переполнение молча отбрасывает старые данные.
// // TryWrite always returns fast; overflow silently discards oldest data.
```

#### Best Practices

- Используй bounded-каналы по умолчанию; unbounded — только при доказанной медленности producer'а относительно consumer'а. Bounded даёт бесплатный backpressure и защиту от OOM.
- Передавай один `CancellationToken` сквозь весь pipeline; отмена должна каскадно гасить все стадии через `WaitToReadAsync`/`WriteAsync`.
- Всегда закрывай writer в `finally`/`try`; зависший `reader.Completion` — типичный баг «pipeline никогда не завершается».
- Ставь `SingleReader = true` / `SingleWriter = true`, когда это правда — канал применит более дешёвую безблочную реализацию.
- Для CPU-bound стадии запускай consumer через `Task.Run` с ограничением степени параллелизма (= числу consumer'ов), а не через `Parallel.ForEach` — последний плохо сочетается с async I/O внутри.
- В pipeline из N стадий делай N-1 канал и N consumer-задач; каждая стадия читает из предыдущего канала и пишет в следующий.

- Prefer bounded channels by default; use unbounded only when the producer is provably slower than the consumer. Bounded gives free backpressure and OOM protection.
- Thread a single `CancellationToken` through the whole pipeline; cancellation must cascade across stages via `WaitToReadAsync`/`WriteAsync`.
- Always close the writer in `finally`/`try`; a stuck `reader.Completion` is the classic "pipeline never finishes" bug.
- Set `SingleReader = true` / `SingleWriter = true` when true — the channel uses a cheaper lock-free path.
- For CPU-bound stages, run consumers via `Task.Run` with bounded parallelism (= consumer count), not `Parallel.ForEach` — the latter composes poorly with async I/O inside.
- For an N-stage pipeline, use N-1 channels and N consumer tasks; each stage reads from the previous channel and writes to the next.

#### Частые ошибки / Common Mistakes

- `.Result` / `.Wait()` на `WriteAsync` → блокирует поток пула, sync-over-async дедлок. Используй `await` везде или `TryWrite` если нужна синхронная запись.
- Забыл `writer.Complete()` → `await foreach` consumer'а зависает навсегда после опустошения. Завершай writer в `finally`.
- `lock` вокруг `await writer.WriteAsync(...)` → блокировка потока на await, потенциальный дедлок. Канал уже потокобезопасен, `lock` не нужен.
- Несколько producer'ов, каждый вызывает `Complete()` → первый завершивший обрывает остальных; `ReadAllAsync` у consumer'а закончится раньше данных. Используй счётчик `Interlocked.Decrement`.
- `async void` для consumer'а → исключения проглатываются, отмена не отслеживается. Только `async Task` + `Task.WhenAll`.
- Unbounded-канал при быстрых producer'ах и медленном consumer'е → OOM. Переходи на bounded с `Wait`.
- Не передаёшь `ct` в `ReadAllAsync`/`WriteAsync` → pipeline нельзя остановить; задача висит в `await` вечно.
- Обрабатываешь `reader.Completion` без `try/finally` → `OperationCanceledException` рвёт логирование и очистку.

- `.Result` / `.Wait()` on `WriteAsync` → blocks a pool thread, sync-over-async deadlock. Use `await` everywhere, or `TryWrite` if a synchronous write is required.
- Forgot `writer.Complete()` → the consumer's `await foreach` hangs forever after the channel drains. Close the writer in `finally`.
- `lock` around `await writer.WriteAsync(...)` → holds a thread across an await, potential deadlock. The channel is already thread-safe; no `lock` needed.
- Multiple producers each calling `Complete()` → the first to finish truncates the rest; the consumer's `ReadAllAsync` ends before the data. Use an `Interlocked.Decrement` counter.
- `async void` for consumers → exceptions are swallowed, cancellation is untracked. Use `async Task` + `Task.WhenAll`.
- Unbounded channel with a fast producer and slow consumer → OOM. Switch to bounded with `Wait`.
- Not passing `ct` into `ReadAllAsync`/`WriteAsync` → the pipeline cannot be stopped; the task hangs in `await` forever.
- Handling `reader.Completion` without `try/finally` → `OperationCanceledException` bypasses logging and cleanup.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Канал bounded с осознанным `FullMode` (по умолчанию `Wait`).
- [ ] `CancellationToken` передан во все `WriteAsync`/`WaitToReadAsync`/`ReadAllAsync`.
- [ ] `writer.Complete()` вызывается ровно один раз, после последнего producer'а (через счётчик).
- [ ] Нигде нет `.Result`/`.Wait()` на асинхронных операциях канала.
- [ ] Нет `lock` вокруг `await`; нет `async void` для consumer'ов.
- [ ] `await foreach` используется для потребления, либо явный `WaitToReadAsync`+`TryRead`.
- [ ] Точка джойна: `await Task.WhenAll(producers)` затем `await Task.WhenAll(consumers)`.
- [ ] Обработка `OperationCanceledException` в `try/finally` вокруг точки джойна.
- [ ] `ConfigureAwait(false)` проставлен в библиотечном/UI-коде.
- [ ] `SingleReader`/`SingleWriter` выставлены в `true`, когда это действительно так.

- [ ] Channel is bounded with a deliberate `FullMode` (default `Wait`).
- [ ] `CancellationToken` is passed into every `WriteAsync`/`WaitToReadAsync`/`ReadAllAsync`.
- [ ] `writer.Complete()` is called exactly once, after the last producer (via a counter).
- [ ] No `.Result`/`.Wait()` anywhere on async channel operations.
- [ ] No `lock` around `await`; no `async void` for consumers.
- [ ] `await foreach` is used for consumption, or an explicit `WaitToReadAsync`+`TryRead`.
- [ ] Join point: `await Task.WhenAll(producers)` then `await Task.WhenAll(consumers)`.
- [ ] `OperationCanceledException` is handled in `try/finally` around the join point.
- [ ] `ConfigureAwait(false)` is applied in library/UI code.
- [ ] `SingleReader`/`SingleWriter` are set to `true` when genuinely true.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.threading.channels.channel](https://learn.microsoft.com/dotnet/api/system.threading.channels.channel)

---

[⬆ К модулю M11](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
