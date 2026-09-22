[⬆ К модулю M09](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M09-L10: Async streams IAsyncEnumerable / Async streams IAsyncEnumerable

**Модуль / Module:** M09
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Асинхронные потоки (async streams) появились в C# 8 и окончательно оформились в .NET 8 / C# 12. Их суть — объединить две идеи: «поток элементов» (как `IEnumerable<T>`) и «асинхронное ожидание» (как `await`). Результат — интерфейс `IAsyncEnumerable<T>`, который отдаёт значения по одному, не блокируя поток, и позволяет потребителю делать `await` между получениями.

Представьте ресторан. `IEnumerable<T>` — это шведский стол: все блюда уже лежат, вы берёте сколько хотите, но повар к этому моменту уже всё приготовил (синхронно). `IObservable<T>` (Rx) — это loudspeaker: повар кричит «готово!», а вы должны успеть отреагировать. `IAsyncEnumerable<T>` — это официант-курьер: он приносит одну тарелку, ждёт пока вы доели, идёт за следующей. Асинхронно, по запросу (pull-based), без блокировки потока.

Ключевые элементы:

1. **`IAsyncEnumerable<T>`** — интерфейс с методом `GetAsyncEnumerator(CancellationToken)`. В отличие от `IEnumerable`, перечислитель отдаёт не `bool MoveNext()`, а `ValueTask<bool> MoveNextAsync()`. То есть переход к следующему элементу — это асинхронная операция.

2. **`await foreach`** — потребительский цикл. Компилятор разворачивает его в цикл с `MoveNextAsync()` / `Current` / `DisposeAsync()`. Под капотом — правильная обработка `CancellationToken` через `WithCancellation`.

3. **`async yield return`** — производство. В `async`-методе, возвращающем `IAsyncEnumerable<T>`, вы пишете `yield return item;`. Между итерациями метод «замораживается» в состоянии машины и возобновляется, когда потребитель просит следующий элемент. Никакой отдельной коллекции в памяти — lazy generation.

4. **`[EnumeratorCancellation]`** — атрибут на параметре `CancellationToken` producer-метода. Он говорит: «если потребитель передал токен в `await foreach ... WithCancellation(ct)`, подставь его сюда». Иначе метод-генератор никогда бы не увидел токен отмены, потому что `await foreach` передаёт токен перечислителю, а не в тело producer-метода напрямую.

**Когда использовать async streams?** Когда источник данных naturally async и элементы приходят со временем: чтение строк из БД (`DbDataReader.ReadAsync`), лог-тейл, чтение больших файлов построчно, пагинация через HTTP API, WebSocket-сообщения, sensor data. Если данные уже в памяти — обычный `IEnumerable<T>` быстрее и проще. Если нужна push-модель с обратным давлением из нескольких источников — посмотрите на `System.Threading.Channels` (урок M11-L07).

**Concurrency-акцент.** Async streams сами по себе **не добавляют параллелизма** — они однопоточные по потреблению (один `await foreach` в одном методе). Но они великолепно сочетаются с concurrency-паттернами:

- **Producer/consumer через Channel.** Несколько producer-задач пишут в `Channel<T>`, один `IAsyncEnumerable` читает через `channel.Reader.ReadAllAsync()`. Это реальная многопоточная конвейерная обработка с обратным давлением.

- **CancellationToken.** Передавайте его через `WithCancellation(ct)` в `await foreach` и помечайте параметр как `[EnumeratorCancellation]`. Без этого долгий producer не сможет отмениться.

- **ConfigureAwait(false).** В библиотечном коде используйте `await foreach (var x in src.WithCancellation(ct).ConfigureAwait(false))`, чтобы не захватывать контекст синхронизации вызывающего (важно в ASP.NET Classic, WinForms, WPF). В ASP.NET Core контекста нет, но привычка `ConfigureAwait(false)` защищает от сюрпризов.

- **Race conditions.** Нельзя читать один `IAsyncEnumerable` из нескольких потоков одновременно — он не потокобезопасен по своей природе. Для fan-out используйте Channel + несколько reader-задач, где `Channel` гарантирует что каждое сообщение доставится ровно одному потребителю.

- **Deadlock.** Никогда не вызывайте `.Result` / `.Wait()` на `ValueTask` из `MoveNextAsync()`, особенно в контексте синхронизации (WinForms/WPF) — классический deadlock. Только `await`.

#### Theory (EN)

Async streams were introduced in C# 8 and matured through .NET 8 / C# 12. Their essence is to merge two ideas: a "stream of elements" (like `IEnumerable<T>`) and "asynchronous waiting" (like `await`). The result is the `IAsyncEnumerable<T>` interface, which yields values one at a time without blocking a thread and lets the consumer `await` between fetches.

Imagine a restaurant. `IEnumerable<T>` is a buffet: all dishes are already laid out, you take what you want, but the chef cooked everything up front (synchronously). `IObservable<T>` (Rx) is a loudspeaker: the chef shouts "ready!" and you must react in time. `IAsyncEnumerable<T>` is a waiter-courier: they bring one plate, wait while you eat, then go fetch the next one. Asynchronous, on demand (pull-based), without blocking the thread.

The key building blocks:

1. **`IAsyncEnumerable<T>`** — an interface with `GetAsyncEnumerator(CancellationToken)`. Unlike `IEnumerable`, the enumerator exposes not `bool MoveNext()` but `ValueTask<bool> MoveNextAsync()`. Advancing to the next element is itself an asynchronous operation.

2. **`await foreach`** — the consumer loop. The compiler expands it into a loop of `MoveNextAsync()` / `Current` / `DisposeAsync()` calls, and wires `CancellationToken` through `WithCancellation`.

3. **`async yield return`** — production. In an `async` method returning `IAsyncEnumerable<T>`, you write `yield return item;`. Between iterations the method is frozen in a state machine and resumed when the consumer asks for the next element. No in-memory collection is built — lazy generation.

4. **`[EnumeratorCancellation]`** — an attribute on a `CancellationToken` parameter of the producer method. It says: "if the consumer passed a token via `await foreach ... WithCancellation(ct)`, thread it into here." Without it, the generator body would never see the cancellation token, because `await foreach` hands the token to the enumerator, not directly to the producer method body.

**When to use async streams?** When the data source is naturally async and elements arrive over time: reading rows from a database (`DbDataReader.ReadAsync`), log tailing, line-by-line reading of large files, HTTP-API pagination, WebSocket messages, sensor data. If the data is already in memory, plain `IEnumerable<T>` is faster and simpler. If you need a push model with backpressure from multiple sources, look at `System.Threading.Channels` (lesson M11-L07).

**Concurrency focus.** Async streams by themselves **add no parallelism** — consumption is single-threaded (one `await foreach` in one method). But they compose beautifully with concurrency patterns:

- **Producer/consumer via Channel.** Several producer tasks write into a `Channel<T>`, while an `IAsyncEnumerable` reads via `channel.Reader.ReadAllAsync()`. That is a real multi-threaded pipeline with backpressure.

- **CancellationToken.** Pass it through `WithCancellation(ct)` in `await foreach` and mark the parameter `[EnumeratorCancellation]`. Otherwise a long-running producer cannot be cancelled.

- **ConfigureAwait(false).** In library code use `await foreach (var x in src.WithCancellation(ct).ConfigureAwait(false))` to avoid capturing the caller's synchronization context (matters in ASP.NET Classic, WinForms, WPF). ASP.NET Core has no context, but the `ConfigureAwait(false)` habit protects against surprises.

- **Race conditions.** You cannot read a single `IAsyncEnumerable` from multiple threads concurrently — it is not thread-safe by nature. For fan-out use a Channel with multiple reader tasks, where the `Channel` guarantees each message is delivered to exactly one consumer.

- **Deadlock.** Never call `.Result` / `.Wait()` on the `ValueTask` returned by `MoveNextAsync()`, especially under a synchronization context (WinForms/WPF) — a classic deadlock. Use `await` only.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — рабочий, потокобезопасный пример async streams + Channel
// C# 12 / .NET 8 — working, thread-safe example of async streams + Channel

using System.Threading.Channels;

// ====== 1. Простая async-stream генерация с отменой ======
// ====== 1. Simple async-stream generation with cancellation ======

public static async IAsyncEnumerable<int> PollSensorAsync(
    int sensorId,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    //ConfigureAwait(false) — не захватываем контекст вызывающего (безопасно для библиотек)
    //ConfigureAwait(false) — do not capture caller's context (safe for libraries)
    while (!ct.IsCancellationRequested)
    {
        int value = await ReadSensorHardwareAsync(sensorId, ct).ConfigureAwait(false);
        yield return value; // lazy: метод замораживается до следующего запроса
        // lazy: method freezes until the next request

        await Task.Delay(TimeSpan.FromMilliseconds(200), ct).ConfigureAwait(false);
    }
}

private static async Task<int> ReadSensorHardwareAsync(int sensorId, CancellationToken ct)
{
    // Имитация асинхронного обращения к железу / I2C-шине
    // Simulating async hardware access / I2C bus
    await Task.Delay(50, ct).ConfigureAwait(false);
    return Random.Shared.Next(0, 1024); // демонстрация; в проде — реальный драйвер
    // demo only; in production use a real driver
}

// ====== 2. Потребление с await foreach и отменой ======
// ====== 2. Consumption with await foreach and cancellation ======

public static async Task ConsumeSensorAsync(CancellationToken ct)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
    cts.CancelAfter(TimeSpan.FromSeconds(3)); // ограничиваем по времени / time-bounded

    try
    {
        // WithCancellation + ConfigureAwait(false) — корректный паттерн
        // WithCancellation + ConfigureAwait(false) — correct pattern
        await foreach (int v in PollSensorAsync(sensorId: 1, cts.Token)
                           .WithCancellation(cts.Token)
                           .ConfigureAwait(false))
        {
            Console.WriteLine($"Sensor={v}"); // обработка в потоке потребителя
            // processing on the consumer's thread
        }
    }
    catch (OperationCanceledException) { /* ожидаемая отмена / expected cancel */ }
}

// ====== 3. Producer/Consumer pipeline через Channel (потокобезопасный) ======
// ====== 3. Producer/Consumer pipeline via Channel (thread-safe) ======

public static async Task RunPipelineAsync(CancellationToken ct)
{
    // Channel<T> — потокобезопасная очередь с обратным давлением (bounded)
    // Channel<T> — thread-safe queue with backpressure (bounded)
    var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(16)
    {
        FullMode = BoundedChannelFullMode.Wait,        // producer ждёт, а не падает
        SingleReader = false,                          // несколько потребителей
        SingleWriter = false                           // несколько производителей
    });

    // --- 3 producers в параллель; каждый пишет в один общий канал ---
    // --- 3 producers in parallel; each writes to the shared channel ---
    var producers = Enumerable.Range(0, 3).Select(i => Task.Run(async () =>
    {
        try
        {
            for (int n = 0; n < 10 && !ct.IsCancellationRequested; n++)
            {
                // WriteAsync самawait'ит при переполнении — это и есть backpressure
                // WriteAsync itself awaits when full — that is backpressure
                await channel.Writer.WriteAsync(i * 100 + n, ct).ConfigureAwait(false);
                await Task.Delay(50, ct).ConfigureAwait(false);
            }
        }
        catch (OperationCanceledException) { }
    }, ct)).ToArray();

    // Когда все producer'ы завершились — закрываем канал для записи
    // When all producers finish, close the channel for writes
    var closeTask = Task.WhenAll(producers).ContinueWith(
        _ => channel.Writer.TryComplete(), TaskScheduler.Default);

    // --- 2 consumers читают через async stream ReadAllAsync ---
    // --- 2 consumers read via the ReadAllAsync async stream ---
    var consumers = Enumerable.Range(0, 2).Select(c => Task.Run(async () =>
    {
        try
        {
            // ReadAllAsync возвращает IAsyncEnumerable<T> — потокобезопасный для нескольких readers
            // ReadAllAsync returns IAsyncEnumerable<T> — safe for multiple readers
            await foreach (int item in channel.Reader.ReadAllAsync(ct)
                               .ConfigureAwait(false))
            {
                Console.WriteLine($"[c{c}] got {item} on thread {Environment.CurrentManagedThreadId}");
            }
        }
        catch (OperationCanceledException) { }
    }, ct)).ToArray();

    await Task.WhenAll(producers);
    await closeTask;
    await Task.WhenAll(consumers);
}

// ====== 4. ОШИБКА: deadlock-антипаттерн (НЕ делайте так) ======
// ====== 4. MISTAKE: deadlock anti-pattern (DO NOT do this) ======

public static int DangerousBlockingConsume(IAsyncEnumerable<int> src)
{
    // ❌ .Result на ValueTask<bool> MoveNextAsync под SynchronizationContext = deadlock
    // ❌ .Result on ValueTask<bool> MoveNextAsync under SynchronizationContext = deadlock
    // ❌ async void ломает обработку ошибок и отмены
    // ❌ async void breaks error and cancellation handling
    var enumerator = src.GetAsyncEnumerator();
    int sum = 0;
    while (enumerator.MoveNextAsync().AsTask().Result) // ❌ блокировка потока пула
    {                                                 // ❌ pool-thread blocking
        sum += enumerator.Current;
    }
    return sum;
}
```

#### Best Practices

- Передавайте `CancellationToken` через `WithCancellation(ct)` и помечайте параметр как `[EnumeratorCancellation]`, чтобы producer-метод реально реагировал на отмену.
- В библиотечном коде используйте `.ConfigureAwait(false)` на каждом `await` внутри producer и в `await foreach`, чтобы не захватывать контекст синхронизации вызывающего.
- Для многопоточного producer/consumer используйте `System.Threading.Channels` — `ReadAllAsync()` возвращает потокобезопасный `IAsyncEnumerable<T>` для нескольких читателей.
- Возвращайте `IAsyncEnumerable<T>`, а не `Task<IAsyncEnumerable<T>>` — иначе вводите лишний уровень ожидания и теряете lazy-семантику.
- Освобождайте ресурсы (`DisposeAsync`): `await foreach` вызывает его автоматически, но при ручном перечислении — обязательно через `await using`.
- Prefer `ValueTask`-семантику (встроена в `MoveNextAsync`) — она дешевле `Task` при синхронном завершении.

- Pass `CancellationToken` via `WithCancellation(ct)` and mark the parameter `[EnumeratorCancellation]` so the producer method actually reacts to cancellation.
- In library code apply `.ConfigureAwait(false)` on every `await` inside the producer and in `await foreach` to avoid capturing the caller's synchronization context.
- For multi-threaded producer/consumer use `System.Threading.Channels` — `ReadAllAsync()` returns a thread-safe `IAsyncEnumerable<T>` for multiple readers.
- Return `IAsyncEnumerable<T>`, not `Task<IAsyncEnumerable<T>>` — otherwise you add an extra await layer and lose lazy semantics.
- Dispose resources (`DisposeAsync`): `await foreach` calls it automatically, but with manual enumeration use `await using`.
- Prefer `ValueTask` semantics (built into `MoveNextAsync`) — it is cheaper than `Task` for synchronous completion.

#### Частые ошибки / Common Mistakes

- **`await foreach` без `WithCancellation(ct)`** → токен не доходит до producer, метод нельзя отменить. Всегда пропускайте `ct` через `WithCancellation` + `[EnumeratorCancellation]`.
- **`.Result` / `.Wait()` на `MoveNextAsync()`** → deadlock под `SynchronizationContext` (WinForms/WPF) или исчерпание пула потоков. Только `await`.
- **Чтение одного `IAsyncEnumerable` из нескольких потоков** → race condition, `InvalidOperationException`. Для fan-out используйте `Channel<T>` с несколькими reader'ами.
- **`async void` для producer-метода** → исключения не всплывают, отмена не работает. Producer должен возвращать `IAsyncEnumerable<T>`.
- **`lock` вокруг `await`** → `lock` не поддерживает `await`, компилятор ругается; а Monitor-hold-during-await блокирует других. Используйте `SemaphoreSlim(1,1)` или `Channel`.
- **Возврат `Task<List<T>>` вместо `IAsyncEnumerable<T>`** → теряете streaming/lazy: вся коллекция материализуется в памяти. Если источник потоковый — возвращайте async stream.

- **`await foreach` without `WithCancellation(ct)`** → the token never reaches the producer, the method cannot be cancelled. Always pass `ct` via `WithCancellation` + `[EnumeratorCancellation]`.
- **`.Result` / `.Wait()` on `MoveNextAsync()`** → deadlock under `SynchronizationContext` (WinForms/WPF) or pool exhaustion. Use `await` only.
- **Reading a single `IAsyncEnumerable` from multiple threads** → race condition, `InvalidOperationException`. For fan-out use `Channel<T>` with multiple readers.
- **`async void` for a producer method** → exceptions do not propagate, cancellation does not work. A producer must return `IAsyncEnumerable<T>`.
- **`lock` around `await`** → `lock` cannot wrap `await` (compiler error); holding a Monitor across an await blocks others. Use `SemaphoreSlim(1,1)` or `Channel`.
- **Returning `Task<List<T>>` instead of `IAsyncEnumerable<T>`** → you lose streaming/laziness: the whole collection materializes in memory. If the source is streaming, return an async stream.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Метод-генератор возвращает `IAsyncEnumerable<T>` и использует `yield return`.
- [ ] Параметр `CancellationToken` помечен `[EnumeratorCancellation]`.
- [ ] `await foreach` использует `WithCancellation(ct)` и `.ConfigureAwait(false)` в библиотечном коде.
- [ ] Нет `.Result` / `.Wait()` на асинхронных перечислениях.
- [ ] Нет `async void`; нет `lock` вокруг `await`.
- [ ] Многопоточный pipeline реализован через `Channel<T>`, а не через разделяемый `IAsyncEnumerable`.
- [ ] Bounded-канал сконфигурирован с `FullMode = Wait` для backpressure.
- [ ] Ресурсы освобождаются (`await using` / `DisposeAsync`).

- [ ] Generator method returns `IAsyncEnumerable<T>` and uses `yield return`.
- [ ] `CancellationToken` parameter is marked `[EnumeratorCancellation]`.
- [ ] `await foreach` uses `WithCancellation(ct)` and `.ConfigureAwait(false)` in library code.
- [ ] No `.Result` / `.Wait()` on async enumerations.
- [ ] No `async void`; no `lock` around `await`.
- [ ] Multi-threaded pipeline uses `Channel<T>`, not a shared `IAsyncEnumerable`.
- [ ] Bounded channel configured with `FullMode = Wait` for backpressure.
- [ ] Resources are disposed (`await using` / `DisposeAsync`).

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/asynchronous-programming/generate-consume-asynchronous-streams]

---

[⬆ К модулю M09](../README.md) | [⬆ Наверх по курсу](../../NAVIGATION.md)
