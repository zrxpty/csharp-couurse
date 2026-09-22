[⬆ К модулю M09](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M09-L06: CancellationToken, cooperative cancellation / CancellationToken, cooperative cancellation

**Модуль / Module:** M09
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

В .NET отмену асинхронных и параллельных операций реализуют не через `Thread.Abort()` (он устарел и опасен), а через **кооперативную отмену** (cooperative cancellation). Идея проста: код, который отменяют, сам добровольно проверяет, не пора ли остановиться. Никто не «убивает» поток извне — поток сам замечает запрос и аккуратно завершает работу, освобождая ресурсы.

В центре модели лежат два типа. `CancellationTokenSource` (CTS) — это «источник» сигнала отмены; его создаёт тот, кто управляет жизненным циклом операции. `CancellationToken` — это «приёмник», легковесная структура-значение, которую передают вглубь call-stack. CTS знает, **как** подать сигнал (`Cancel()`, `CancelAfter(timeout)`), а токен знает только, **был ли сигнал уже подан** (`IsCancellationRequested`, `ThrowIfCancellationRequested()`). Это разделение важно: токен нельзя отменить напрямую — у него нет метода `Cancel()`, что защищает нижние слои кода от произвольной отмены вызывающей стороной.

Аналогия: представьте пожарную сигнализацию в большом здании. CTS — рубильник у вахтёра, а токены — датчики и табло на каждом этаже. Вахтёр дёргает рубильник один раз, и все этажи одновременно видят сигнал. Каждый этаж сам решает, как корректно эвакуироваться: кто-то закрывает двери, кто-то выключает оборудование.

**Передача токена.** Токен нужно протаскивать через каждый метод, который поддерживает отмену. В .NET принят конвенциональный параметр `CancellationToken cancellationToken = default`. Параметр делают последним и со значением по умолчанию, чтобы не ломать существующие вызовы. Если метод не получает токен вообще, он не сможет отреагировать на отмену — операция будет «глухой».

**Отменяемые циклы.** Внутри `for`/`while`/`foreach` ставят проверку `cancellationToken.ThrowIfCancellationRequested()` на каждой итерации. Это выбрасывает `OperationCanceledException` (точнее `TaskCanceledException` для задач), который корректно пробрасывается вверх. Важно перехватывать его **только** там, где это осмысленно (например, чтобы залогировать или вернуть частичный результат), и не «глотать» молча. Для `Parallel.ForEach` и PLINQ токен передаётся через `ParallelOptions` / `Cancellationtoken` — они сами встраивают проверки.

**Linked tokens.** Иногда операцию нужно отменить по нескольким причинам: по таймауту **или** по запросу пользователя **или** при закрытии приложения. `CancellationTokenSource.CreateLinkedTokenSource(tokenA, tokenB)` создаёт CTS, который сработает, если отменят **любой** из источников. Это композиция сигналов. Не забывайте `Dispose()` у CTS (особенно у linked и `CancelAfter`), иначе утекут таймеры.

**Race conditions и дедлоки.** Кооперативная отмена не свободна от гонок. Классическая ловушка: проверка `IsCancellationRequested` в условии `while`, а долгая синхронная работа внутри тела — между проверкой и завершением код успевает наделать дел. Поэтому проверки ставят часто, а особо длинные операции разбивают. Вторая ловушка — `Wait()`/`.Result`/`.GetAwaiter().GetResult()` на задаче, которую можно отменить: если операция внутри держит `lock` или обращается к UI-потоку, получится дедлок. Правильно — `await` с токеном везде. Третья ловушка — `async void`: исключение из такого метода (включая `OperationCanceledException`) рвёт процесс. Используйте `async Task` и обработку в `try/catch`.

`ThrowIfCancellationRequested` потокобезопасен: внутренне использует `volatile`-флаг и `Interlocked`. Регистрация колбэка через `token.Register(...)` атомарна, но сам колбэк может выполниться либо в потоке, вызвавшем `Cancel()`, либо в потоке, зарегистрировавшем его — поэтому код колбэка должен быть реентерабельным и не держать длительные блокировки.

#### Theory (EN)

In .NET, cancellation of asynchronous and parallel operations is not done via `Thread.Abort()` (deprecated and dangerous). Instead, .NET uses **cooperative cancellation**. The idea is simple: the code being cancelled voluntarily checks whether it should stop. Nobody kills the thread from outside — the thread notices the request itself and shuts down gracefully, releasing resources.

Two types sit at the center of the model. `CancellationTokenSource` (CTS) is the **source** of the cancellation signal; it is created by whoever controls the lifecycle of the operation. `CancellationToken` is the **receiver**, a lightweight value-type struct that is passed down the call stack. CTS knows **how** to raise the signal (`Cancel()`, `CancelAfter(timeout)`); the token only knows **whether** the signal has already been raised (`IsCancellationRequested`, `ThrowIfCancellationRequested()`). This separation matters: a token cannot be cancelled directly — it has no `Cancel()` method, which protects lower layers from arbitrary cancellation by an upper caller.

Analogy: imagine a fire alarm in a large building. CTS is the lever at the security desk; tokens are the sensors and displays on every floor. The guard pulls the lever once, and every floor sees the signal simultaneously. Each floor decides for itself how to evacuate properly: one closes fire doors, another shuts down equipment.

**Passing the token.** The token must be threaded through every method that supports cancellation. The .NET convention is a parameter `CancellationToken cancellationToken = default`. It is placed last and given a default value so existing callers do not break. If a method receives no token at all, it cannot react to cancellation — the operation becomes "deaf".

**Cancellable loops.** Inside `for`/`while`/`foreach`, place a `cancellationToken.ThrowIfCancellationRequested()` check on every iteration. It throws `OperationCanceledException` (more precisely `TaskCanceledException` for tasks), which propagates correctly upward. Catch it **only** where it is meaningful (for example to log or return a partial result) and never swallow it silently. For `Parallel.ForEach` and PLINQ, the token is passed via `ParallelOptions` / the `Cancellationtoken` parameter — they inject the checks themselves.

**Linked tokens.** Sometimes an operation must be cancellable for several independent reasons: a timeout **or** a user request **or** application shutdown. `CancellationTokenSource.CreateLinkedTokenSource(tokenA, tokenB)` creates a CTS that fires if **any** of its sources is cancelled. This is signal composition. Do not forget to `Dispose()` the CTS (especially linked ones and `CancelAfter`), otherwise timers leak.

**Race conditions and deadlocks.** Cooperative cancellation is not free of races. The classic trap: checking `IsCancellationRequested` in a `while` condition while doing long synchronous work in the loop body — between the check and the actual exit, the code can still do damage. So checks must be frequent and long operations split. The second trap is `Wait()`/`.Result`/`.GetAwaiter().GetResult()` on a cancellable task: if the inner operation holds a `lock` or touches the UI thread, you get a deadlock. The right way is `await` with the token everywhere. The third trap is `async void`: an exception from such a method (including `OperationCanceledException`) tears down the process. Use `async Task` and handle it in `try/catch`.

`ThrowIfCancellationRequested` is thread-safe: internally it uses a `volatile` flag and `Interlocked`. Registering a callback via `token.Register(...)` is atomic, but the callback itself may run either on the thread that called `Cancel()` or on the thread that registered it — so the callback code must be reentrant and must not hold long locks.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — cooperative cancellation in practice
// C# 12 / .NET 8 — кооперативная отмена на практике
using System.Diagnostics;
using System.Threading.Channels;

// ---- 1. Базовый отменяемый цикл / Basic cancellable loop ----
async Task<int> SumUntilCancelledAsync(IEnumerable<int> source, CancellationToken ct)
{
    int sum = 0;
    foreach (var x in source)
    {
        // Проверяем отмену на каждой итерации — частые проверки = малая задержка реакции.
        // Check cancellation on every iteration — frequent checks = low reaction latency.
        ct.ThrowIfCancellationRequested();
        sum += x;                 // тяжёлая работа / heavy work
        await Task.Delay(10, ct); // имитация I/O, токен передаётся дальше / I/O mock, token forwarded
    }
    return sum;
}

// ---- 2. CancelAfter: отмена по таймауту / cancellation by timeout ----
// ВАЖНО: CTS с таймером нужно диспозить, иначе утечка таймера.
// IMPORTANT: a CTS with a timer must be disposed, otherwise the timer leaks.
async Task<string> DownloadWithTimeoutAsync(HttpClient http, string url, TimeSpan timeout)
{
    using var cts = new CancellationTokenSource(timeout); // = CancelAfter(timeout)
    try
    {
        // Токен прокинут вглубь API BCL — HttpClient сам корректно отреагирует.
        // The token is forwarded into BCL API — HttpClient reacts correctly.
        return await http.GetStringAsync(url, cts.Token);
    }
    catch (OperationCanceledException) when (cts.IsCancellationRequested)
    {
        // Разделяем «наш таймаут» и чужую отмену — без условия catch съел бы любую отмену.
        // Distinguish "our timeout" from foreign cancellation — without the filter catch would swallow any.
        return "TIMEOUT / таймаут";
    }
}

// ---- 3. Linked tokens: отмена по таймауту ИЛИ по внешнему сигналу ----
// Linked tokens: cancellation by timeout OR by an external signal
async Task ProcessWithLinkedAsync(CancellationToken externalToken)
{
    using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
    // Связанный источник сработает, если отменят ЛЮБОЙ из источников.
    // The linked source fires if ANY of the sources is cancelled.
    using var linked = CancellationTokenSource.CreateLinkedTokenSource(
        externalToken, timeoutCts.Token);

    await foreach (var item in StreamAsync(linked.Token))
        Console.WriteLine($"item={item} / элемент={item}");

    // Loop exited: либо закончился поток, либо отмена — различаем по токену.
    // Loop exited: either stream ended, or cancellation — distinguish by the token.
    if (linked.IsCancellationRequested && !externalToken.IsCancellationRequested)
        Console.WriteLine("Stopped by timeout / остановлено по таймауту");
}

static async IAsyncEnumerable<int> StreamAsync(
    [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct)
{
    for (int i = 0; ; i++)
    {
        ct.ThrowIfCancellationRequested();            // частая проверка / frequent check
        await Task.Delay(50, ct);                     // I/O с токеном / I/O with token
        yield return i;
    }
}

// ---- 4. Producer/consumer через Channel с отменой / pipeline ----
// Потокобезопасный канал: множество продюсеров и консьюмеров без явных lock.
// Thread-safe channel: many producers and consumers without explicit locks.
async Task RunPipelineAsync(CancellationToken ct)
{
    // BoundedChannel ограничивает очередь — backpressure без ручной синхронизации.
    // BoundedChannel caps the queue — backpressure without manual synchronization.
    var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(64)
    {
        FullMode = BoundedChannelFullMode.Wait,       // продюсер ждёт места / producer waits for room
        SingleReader = false,
        SingleWriter = false,
    });

    // ПРОДЮСЕР / PRODUCER
    var producer = Task.Run(async () =>
    {
        try
        {
            for (int i = 0; i < 1_000_000; i++)
            {
                ct.ThrowIfCancellationRequested();               // сотрудничаем с отменой / cooperate
                // WriteAsync сам прокидывает токен и бросает, если канал закрыт или отменён.
                // WriteAsync forwards the token and throws if the channel is closed or cancelled.
                await channel.Writer.WriteAsync(i, ct);
            }
        }
        catch (OperationCanceledException) { /* штатная отмена / normal cancellation */ }
        finally
        {
            channel.Writer.Complete(); // сообщаем консьюмеру: данных больше не будет / signal end-of-stream
        }
    }, ct);

    // КОНСЬЮМЕР / CONSUMER
    var consumer = Task.Run(async () =>
    {
        try
        {
            // ReadAllAsync принимает токен и сам завершает цикл при отмене.
            // ReadAllAsync takes the token and ends the loop on cancellation itself.
            await foreach (var item in channel.Reader.ReadAllAsync(ct))
            {
                ct.ThrowIfCancellationRequested();
                // обработка элемента / item processing
            }
        }
        catch (OperationCanceledException) { /* штатная отмена / normal cancellation */ }
    }, ct);

    // НЕ блокируем поток пулом: await, а не .Wait()/.Result — иначе риск дедлока.
    // Do not block a pool thread: await, not .Wait()/.Result — otherwise deadlock risk.
    await Task.WhenAll(producer, consumer);
}

// ---- 5. Корректный вход в async-операцию из Main / proper async entry ----
async Task MainAsync()
{
    using var cts = new CancellationTokenSource();
    Console.CancelKeyPress += (_, e) =>
    {
        e.Cancel = true;          // не убивать процесс сразу / do not kill the process at once
        cts.Cancel();             // мягко просим все операции остановиться / ask operations to stop softly
    };

    try
    {
        await RunPipelineAsync(cts.Token);
    }
    catch (OperationCanceledException)
    {
        // Ожидаемо при Ctrl+C — НЕ ошибка. Логируем и выходим спокойно.
        // Expected on Ctrl+C — NOT an error. Log and exit calmly.
        Console.WriteLine("Cancelled gracefully / отменено штатно");
    }
}
```

#### Best Practices

- Передавайте `CancellationToken` последним параметром со значением по умолчанию: `... , CancellationToken ct = default`. Это сохраняет совместимость и делает токен «гражданином первого класса» в сигнатуре.
- Используйте `ThrowIfCancellationRequested()` внутри циклов чаще, чем `IsCancellationRequested` — исключение корректно пробрасывается через `await` и попадает в единый обработчик.
- Диспозьте `CancellationTokenSource`, созданный с `CancelAfter` или через `CreateLinkedTokenSource`, — иначе утечёт внутренний таймер.
- Для композиции причин отмены (таймаут + внешний сигнал) всегда используйте linked tokens, а не «вручную» дёргайте несколько `Cancel()`.
- В async-коде НИКОГДА не вызывайте `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` на отменяемой задаче — только `await` с токеном, иначе высок риск дедлока.
- Колбэк `token.Register(...)` делайте максимально коротким и реентерабельным; он может выполниться в любом потоке, в том числе в потоке, вызвавшем `Cancel()`.

- Pass `CancellationToken` as the last parameter with a default value: `... , CancellationToken ct = default`. This preserves compatibility and makes the token a first-class citizen of the signature.
- Prefer `ThrowIfCancellationRequested()` inside loops over `IsCancellationRequested` — the exception propagates correctly through `await` and reaches a single handler.
- Dispose any `CancellationTokenSource` created with `CancelAfter` or via `CreateLinkedTokenSource` — otherwise the internal timer leaks.
- To compose cancellation reasons (timeout + external signal) always use linked tokens instead of manually calling several `Cancel()` methods.
- In async code NEVER call `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` on a cancellable task — only `await` with the token, otherwise the deadlock risk is high.
- Keep the `token.Register(...)` callback as short and reentrant as possible; it may run on any thread, including the one that called `Cancel()`.

#### Частые ошибки / Common Mistakes

- **`while (!ct.IsCancellationRequested)` + долгая синхронная работа в теле** → между проверкой и выходом код успевает выполнить несколько итераций и повредить состояние. Часто ставьте `ct.ThrowIfCancellationRequested()` внутри тела, а не только в условии.
- **`async void` метод с `await` по токену** → `OperationCanceledException` из такого метода рвёт процесс через `AppDomain.UnhandledException`. Используйте `async Task` и оборачивайте в `try/catch`.
- **`lock (obj) { await ...; }`** → `lock` не разрешает `await` (ошибка компиляции CS1996); а `SemaphoreSlim(1,1).WaitAsync(ct)` — правильный аналог. Не блокируйте поток пулом под блокировкой.
- **Перехват `OperationCanceledException` без условия `when`** → «глотает» любую отмену, включая чужую. Используйте `catch (OperationCanceledException) when (cts.IsCancellationRequested)`, чтобы различать «свою» и «чужую» отмену.
- **Не диспозят `CancellationTokenSource.CreateLinkedTokenSource(...)`** → утечка таймеров и registrant-списков под долгоживущими сервисами. Всегда `using` или явный `Dispose()`.
- **Забыли прокинуть токен в `Task.Delay` / `HttpClient.Get*Async`** → операция не реагирует на отмену и висит до естественного завершения. Проверяйте, что у каждого async-вызова есть параметр-токен.

- **`while (!ct.IsCancellationRequested)` plus long synchronous work in the body** → between the check and the exit the code still runs several iterations and may corrupt state. Put `ct.ThrowIfCancellationRequested()` frequently inside the body, not only in the condition.
- **`async void` method with an `await` on a token** → `OperationCanceledException` from such a method kills the process via `AppDomain.UnhandledException`. Use `async Task` and wrap in `try/catch`.
- **`lock (obj) { await ...; }`** → `lock` does not allow `await` (compiler error CS1996); `SemaphoreSlim(1,1).WaitAsync(ct)` is the correct analog. Do not block a pool thread under a lock.
- **Catching `OperationCanceledException` without a `when` filter** → swallows any cancellation, including a foreign one. Use `catch (OperationCanceledException) when (cts.IsCancellationRequested)` to distinguish "own" vs "foreign" cancellation.
- **Forgetting to dispose `CancellationTokenSource.CreateLinkedTokenSource(...)`** → timer and registrant-list leaks under long-lived services. Always use `using` or an explicit `Dispose()`.
- **Forgetting to forward the token into `Task.Delay` / `HttpClient.Get*Async`** → the operation ignores cancellation and hangs until natural completion. Check that every async call has a token parameter.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Каждый публичный async-метод принимает `CancellationToken cancellationToken = default` последним параметром.
- [ ] В каждом цикле `for/while/foreach` есть `ThrowIfCancellationRequested()` или `ct` передан в API (`Parallel.ForEach`, PLINQ, `ReadAllAsync`).
- [ ] Токен прокинут во все BCL/async-вызовы (`Task.Delay`, `HttpClient`, `Channel.Writer.WriteAsync`, `Stream.ReadAsync`).
- [ ] `CancellationTokenSource` с `CancelAfter` или `CreateLinkedTokenSource` обёрнут в `using` или явно диспозится.
- [ ] Нигде нет `.Result`/`.Wait()`/`.GetAwaiter().GetResult()` на отменяемой задаче — только `await`.
- [ ] Нет `async void` (кроме обработчиков событий UI), нет `lock` поверх `await`.
- [ ] `OperationCanceledException` ловится с `when (...IsCancellationRequested)` или осмысленно логируется, а не глотается молча.
- [ ] Колбэк `token.Register(...)` короткий, реентерабельный, не держит длительных блокировок.

- [ ] Every public async method takes `CancellationToken cancellationToken = default` as the last parameter.
- [ ] Every `for/while/foreach` loop has `ThrowIfCancellationRequested()` or passes `ct` into an API (`Parallel.ForEach`, PLINQ, `ReadAllAsync`).
- [ ] The token is forwarded into all BCL/async calls (`Task.Delay`, `HttpClient`, `Channel.Writer.WriteAsync`, `Stream.ReadAsync`).
- [ ] A `CancellationTokenSource` with `CancelAfter` or `CreateLinkedTokenSource` is wrapped in `using` or disposed explicitly.
- [ ] There are no `.Result`/`.Wait()`/`.GetAwaiter().GetResult()` calls on a cancellable task — only `await`.
- [ ] No `async void` (except UI event handlers), no `lock` over `await`.
- [ ] `OperationCanceledException` is caught with `when (...IsCancellationRequested)` or meaningfully logged, not swallowed silently.
- [ ] The `token.Register(...)` callback is short, reentrant, and does not hold long locks.

#### Ресурсы / Resources

- [Microsoft Learn — Cancellation in Managed Threads](https://learn.microsoft.com/dotnet/standard/threading/cancellation-in-managed-threads)

---

[⬆ К модулю M09](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
