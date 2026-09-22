[⬆ К модулю M09](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M09-L07: ConfigureAwait(false), SynchronizationContext / ConfigureAwait(false), SynchronizationContext

**Модуль / Module:** M09
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

**SynchronizationContext** — это «почтальон» .NET, который решает, **где** продолжить код после `await`. Представь, что ты работаешь в офисе с одним единственным окном приёма посетителей (UI-поток). Когда ты уходишь «выполнить задачу» (вызоваешь асинхронную операцию), почтальон должен знать, куда вернуть ответ — в то же окно или в общую очередь. По умолчанию `await` «запоминает» текущий контекст синхронизации и пытается вернуться в него. В WPF/WinForms это значит — обратно в UI-поток, потому что только он может трогать элементы интерфейса. В ASP.NET *classic* (Framework) это значит — обратно в поток, у которого есть `HttpContext`, чтобы продолжить обработку запроса.

Проблема возникает, когда библиотечный код «украшен» этим контекстом без необходимости. Скажем, ты пишешь `await httpClient.GetAsync(url)` внутри метода, вызванного из UI. После завершения HTTP-запроса continuation пытается вернуться в UI-поток. Если UI-поток в этот момент **заблокирован** вызовом `.Result` или `.Wait()` (антипаттерн «sync over async»), возникает классический **дедлок**: UI ждёт задачу, задача ждёт UI. Потоки стоят вечно. Это не теоретический пример — это самая частая ошибка при вызове асинхронного кода из обработчика события `Click` без `async void` и без `ConfigureAwait(false)`.

`ConfigureAwait(false)` говорит рантайму: **«не пытайся вернуться в исходный контекст, продолжай где угодно, в пуле потоков»**. В библиотечном коде это правило хорошего тона: библиотека не знает, кто её вызвал — UI, ASP.NET Core или консоль — и не должна тянуть на себя чужой контекст. Это снижает нагрузку на UI-поток, уменьшает риск дедлока и часто ускоряет выполнение.

Важно понимать современный ландшафт: **ASP.NET Core не имеет SynchronizationContext**. В нём continuation всегда выполняется в пуле потоков, поэтому `ConfigureAwait(false)` там *почти* ничего не меняет. Многие команды (включая сам Microsoft в BCL) всё равно ставят его в библиотеках ради единообразия и совместимости со старыми хостами. В приложениях же (.NET MAUI, WPF, Blazor WASM, WinForms) контекст есть, и там `ConfigureAwait(false)` в *библиотечном* коде полезен, а в *прикладном* коде, обращающемся к UI — наоборот, **вреден**: ты потеряешь возможность безопасно трогать контролы после `await`.

**Race conditions** здесь прячутся в деталях. Если ты поставил `ConfigureAwait(false)` и после `await` обращаешься к общему состоянию (кэш, словарь), ты уже не защищён контекстом — несколько continuation из пула могут зайти одновременно. Нужны явные средства: `lock`, `SemaphoreSlim`, `Interlocked`, `Channel<T>`. Точно так же `async void` опасен тем, что исключения вылетают мимо стека вызова и часто роняют процесс, а отмена не передаётся корректно.

Запомни три железных правила: **не блокируй асинхронный код** (никакого `.Result`/`.Wait()` в горячих путях), **в библиотеках — `ConfigureAwait(false)`**, **в UI-коде — оставь дефолт** и обязательно передавай `CancellationToken` всюду, где возможна отмена.

#### Theory (EN)

**SynchronizationContext** is .NET's "postman" that decides **where** the code resumes after an `await`. Imagine you work in an office with a single customer-service window (the UI thread). When you leave to "do a task" (invoke an asynchronous operation), the postman must know where to deliver the answer — back to that same window or to a general queue. By default, `await` captures the current synchronization context and tries to return to it. In WPF/WinForms this means back on the UI thread, because only it may touch UI elements. In ASP.NET *classic* (Framework) it means back on a thread that carries the `HttpContext`, so request processing can continue.

The problem appears when library code is decorated with this context unnecessarily. Suppose you write `await httpClient.GetAsync(url)` inside a method called from UI. After the HTTP call completes, the continuation tries to return to the UI thread. If the UI thread is currently **blocked** by a `.Result` or `.Wait()` call (the "sync over async" antipattern), a classic **deadlock** arises: the UI waits for the task, the task waits for the UI. Threads stall forever. This is not a theoretical example — it is the most frequent mistake when invoking async code from a `Click` event handler without `async void` and without `ConfigureAwait(false)`.

`ConfigureAwait(false)` tells the runtime: **"do not try to return to the original context, continue anywhere, on the thread pool."** In library code this is good manners: a library does not know who called it — UI, ASP.NET Core, or a console app — and should not drag the caller's context along. This reduces UI-thread load, lowers deadlock risk, and often speeds execution.

Understand the modern landscape: **ASP.NET Core has no SynchronizationContext.** Continuations always run on the thread pool, so `ConfigureAwait(false)` there is *almost* a no-op. Many teams (including Microsoft itself in the BCL) still apply it in libraries for consistency and compatibility with older hosts. In applications (.NET MAUI, WPF, Blazor WASM, WinForms) the context exists, and there `ConfigureAwait(false)` in *library* code is useful, while in *application* code that talks to the UI it is the opposite — **harmful**: you lose the ability to safely touch controls after `await`.

**Race conditions** hide in the details. Once you set `ConfigureAwait(false)`, continuations after `await` are no longer serialized by the context — several pool continuations may enter shared state (a cache, a dictionary) simultaneously. You need explicit tools: `lock`, `SemaphoreSlim`, `Interlocked`, `Channel<T>`. Likewise, `async void` is dangerous because exceptions fly off the call stack and often crash the process, and cancellation is not propagated correctly.

Three iron rules: **do not block async code** (no `.Result`/`.Wait()` on hot paths), **in libraries use `ConfigureAwait(false)`**, **in UI code keep the default**, and always propagate a `CancellationToken` wherever cancellation is possible.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — ConfigureAwait, SynchronizationContext, deadlock-safe patterns
// Demonstrates: library-style ConfigureAwait(false), UI-safe continuation,
// thread-safe shared state, CancellationToken, and a classic deadlock guard.

using System.Collections.Concurrent;
using System.Diagnostics;
using System.Runtime.CompilerServices;

public static class SynchronizationContextDemo
{
    // --- 1. Library method: ConfigureAwait(false) everywhere, CancellationToken propagated.
    // Библиотечный метод: ConfigureAwait(false) везде, CancellationToken передаётся. ---
    public static async Task<string> FetchAsync(HttpClient http, Uri url, CancellationToken ct)
    {
        // Library code does not capture caller's SynchronizationContext.
        // Библиотека не захватывает SynchronizationContext вызывающего.
        using var resp = await http.GetAsync(url, HttpCompletionOption.ResponseHeadersRead, ct)
            .ConfigureAwait(false);

        resp.EnsureSuccessStatusCode();

        // Continue on the thread pool — safe, no UI affinity.
        // Продолжение в пуле потоков — безопасно, без привязки к UI.
        await using var stream = await resp.Content.ReadAsStreamAsync(ct).ConfigureAwait(false);
        using var sr = new StreamReader(stream);

        // Build a stable string with pooled threads: no shared mutable state here.
        // Собираем строку в пуле потоков: общего изменяемого состояния нет.
        return await sr.ReadToEndAsync(ct).ConfigureAwait(false);
    }

    // --- 2. UI-side caller: keep default context so we can touch controls after await.
    // Вызывающий код на стороне UI: сохраняем контекст по умолчанию. ---
    public static async Task UpdateLabelFromUiThreadAsync(HttpClient http, Label label, CancellationToken ct)
    {
        // No ConfigureAwait(false) here — we MUST come back to the UI thread
        // to update label.Text safely.
        // Здесь БЕЗ ConfigureAwait(false) — мы обязаны вернуться в UI-поток,
        // чтобы безопасно обновить label.Text.
        try
        {
            string body = await FetchAsync(http, new Uri("https://example.com"), ct);
            label.Text = body.Length > 80 ? body[..80] : body; // UI thread, safe.
        }
        catch (OperationCanceledException)
        {
            label.Text = "Cancelled / Отменено";
        }
    }

    // --- 3. Thread-safe cache shared across pool continuations after ConfigureAwait(false).
    // Потокобезопасный кэш, разделяемый между continuation из пула. ---
    private static readonly ConcurrentDictionary<Uri, string> _cache = new();

    public static async Task<string> FetchCachedAsync(HttpClient http, Uri url, CancellationToken ct)
    {
        if (_cache.TryGetValue(url, out var hit))
            return hit; // Fast path, no allocation.

        // Slow path: network I/O on the pool, no captured context.
        // Медленный путь: сетевой I/O в пуле, без захвата контекста.
        string body = await FetchAsync(http, url, ct).ConfigureAwait(false);

        // Race-safe: last writer wins, both threads see consistent dictionary state.
        // Гонка безопасна: победитель последний, оба потока видят согласованное состояние.
        _cache[url] = body;
        return body;
    }

    // --- 4. DEADLOCK DEMONSTRATION (anti-pattern, intentionally broken).
    // ДЕМОНСТРАЦИЯ ДЕДЛОКА (антипаттерн, намеренно сломан). ---
    public static string FetchBlocking_Bad(HttpClient http, Uri url)
    {
        // WARNING: on a host WITH SynchronizationContext (WPF/WinForms/ASP.NET-classic)
        // this deadlocks: the UI thread blocks on .Result, and the continuation
        // inside FetchAsync (which uses ConfigureAwait(false) — so it is SAFE here,
        // but in legacy libs without ConfigureAwait(false) it would wait for the UI thread).
        // ВНИМАНИЕ: на хосте С SynchronizationContext (WPF/WinForms/ASP.NET-classic)
        // это вызывает дедлок: UI-поток ждёт .Result, а continuation ждёт UI-поток.
        return FetchAsync(http, url, CancellationToken.None).GetAwaiter().GetResult();
    }

    // --- 5. Deadlock-safe "sync over async" only when truly unavoidable.
    // Безопасный "sync over async", только когда alternatives нет. ---
    public static string FetchBlocking_Safe(HttpClient http, Uri url)
    {
        // Run on a dedicated thread-pool worker with NO captured context,
        // so a blocked caller thread cannot deadlock the continuation.
        // Запускаем на воркере пула БЕЗ захвата контекста — заблокированный
        // вызывающий поток не сможет вызвать дедлок continuation.
        return Task.Run(async () => await FetchAsync(http, url, CancellationToken.None))
                    .GetAwaiter().GetResult();
    }

    // --- 6. Channel-based producer/consumer: idiomatic concurrency with backpressure.
    // Конвейер producer/consumer на Channel: идиоматичный concurrency с обратным давлением. ---
    public static async Task RunPipelineAsync(
        IEnumerable<int> source,
        Func<int, CancellationToken, Task<string>> transform,
        Action<string> consume,
        CancellationToken ct)
    {
        // Bounded channel = natural backpressure: fast producer cannot flood slow consumer.
        // Ограниченный канал = естественное обратное давление: быстрый producer не затопит consumer.
        var channel = Channel.CreateBounded<string>(new BoundedChannelOptions(64)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = true,
            SingleWriter = false
        });

        // Producer: feeds raw ids into the channel.
        // Producer: подаёт исходные идентификаторы в канал.
        var produce = Task.Run(async () =>
        {
            try
            {
                foreach (var id in source)
                {
                    ct.ThrowIfCancellationRequested();
                    string item = await transform(id, ct).ConfigureAwait(false);
                    await channel.Writer.WriteAsync(item, ct).ConfigureAwait(false);
                }
            }
            finally
            {
                channel.Writer.Complete();
            }
        }, ct);

        // Consumer: single reader, safe to process sequentially.
        // Consumer: один читатель, безопасно обрабатываем последовательно.
        var consumeTask = Task.Run(async () =>
        {
            await foreach (var item in channel.Reader.ReadAllAsync(ct).ConfigureAwait(false))
            {
                consume(item); // No lock needed: single-reader invariant.
                               // lock не нужен: инвариант одного читателя.
            }
        }, ct);

        await Task.WhenAll(produce, consumeTask).ConfigureAwait(false);
    }
}

// Minimal UI stub so the example compiles in a library context.
// Минимальная UI-заглушка, чтобы пример компилировался в библиотеке.
public sealed class Label { public string Text { get; set; } = ""; }
```

#### Best Practices

- В **библиотечном** коде всегда используй `ConfigureAwait(false)` после каждого `await` и принимай `CancellationToken` в каждой асинхронной сигнатуре.
- В **UI-коде** (WPF/WinForms/MAUI/Blazor) оставляй дефолтный контекст для методов, которые после `await` обращаются к контролам; только внутренние «чистые» вызовы можно украшать `ConfigureAwait(false)`.
- Для разделения доступа к общему состоянию между continuation из пула используй `ConcurrentDictionary`, `SemaphoreSlim`, `Channel<T>` — а не голый `lock` поверх `await`.
- В **ASP.NET Core** не трать усилия на поголовное `ConfigureAwait(false)` в прикладном коде — контекста там нет; но сохраняй привычку в переиспользуемых библиотеках.

- In **library** code, always apply `ConfigureAwait(false)` after each `await` and accept a `CancellationToken` in every async signature.
- In **UI code** (WPF/WinForms/MAUI/Blazor), keep the default context for methods that touch controls after `await`; only internal "pure" calls may use `ConfigureAwait(false)`.
- Use `ConcurrentDictionary`, `SemaphoreSlim`, `Channel<T>` — never a raw `lock` around `await` — to guard shared state between pool continuations.
- In **ASP.NET Core**, don't bother sprinkling `ConfigureAwait(false)` in application code — there is no context; but keep the habit in reusable libraries.

#### Частые ошибки / Common Mistakes

- `.Result` / `.Wait()` в UI-потоке → **дедлок** на хостах с SynchronizationContext. Избегай: делай метод `async` до конца, либо запускай через `Task.Run` без захвата контекста.
- `ConfigureAwait(false)` в UI-методе, который после `await` трогает контролы → `InvalidOperationException` ("The calling thread cannot access this object"). Убери `ConfigureAwait(false)` на этом участке.
- `lock (obj) { await ... }` → компилятор это разрешает, но `Monitor` не является реентерабельным: continuation на другом потоке не сможет войти, и ты получишь дедлок или исключение. Замени на `SemaphoreSlim(1,1)` с `await sem.WaitAsync()`.
- `async void` для обработчиков без try/catch → исключение роняет процесс. Оборачивай в `try/catch` и логируй; предпочитай `async Task` где можно.
- Передача `CancellationToken.None` «для простоты» в библиотеку → нельзя отменить долгую операцию; потребитель не сможет прервать запрос.
- Забыл `channel.Writer.Complete()` в `finally` → consumer висит в `ReadAllAsync` навсегда.

- `.Result` / `.Wait()` on the UI thread → **deadlock** on hosts with SynchronizationContext. Avoid: make the method fully `async`, or dispatch via `Task.Run` without captured context.
- `ConfigureAwait(false)` in a UI method that touches controls after `await` → `InvalidOperationException` ("The calling thread cannot access this object"). Remove `ConfigureAwait(false)` on that segment.
- `lock (obj) { await ... }` → the compiler allows it, but `Monitor` is non-reentrant: a continuation on another thread cannot re-enter, yielding a deadlock or exception. Replace with `SemaphoreSlim(1,1)` and `await sem.WaitAsync()`.
- `async void` event handlers without try/catch → an exception crashes the process. Wrap in `try/catch` and log; prefer `async Task` where possible.
- Passing `CancellationToken.None` "for simplicity" into a library → the long operation cannot be cancelled; the consumer cannot abort the request.
- Forgot `channel.Writer.Complete()` in `finally` → the consumer hangs in `ReadAllAsync` forever.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] В библиотечных `await` стоит `ConfigureAwait(false)`.
- [ ] В UI-методах, трогающих контролы, `ConfigureAwait(false)` отсутствует.
- [ ] Нигде нет `.Result` / `.Wait()` на потоке с SynchronizationContext.
- [ ] Нет `lock` вокруг `await`; вместо него `SemaphoreSlim` или `Channel<T>`.
- [ ] `CancellationToken` доходит до каждой асинхронной операции, включая HTTP и I/O.
- [ ] В producer/consumer на `Channel` вызывается `Writer.Complete()` в `finally`.
- [ ] `async void` используется только для top-level обработчиков событий и всегда в `try/catch`.
- [ ] ASP.NET Core-код не завален избыточным `ConfigureAwait(false)` в прикладном слое.

- [ ] Library `await` calls carry `ConfigureAwait(false)`.
- [ ] UI methods that touch controls do **not** use `ConfigureAwait(false)`.
- [ ] No `.Result` / `.Wait()` on a thread with SynchronizationContext anywhere.
- [ ] No `lock` around `await`; use `SemaphoreSlim` or `Channel<T>` instead.
- [ ] `CancellationToken` reaches every async operation, including HTTP and I/O.
- [ ] The `Channel` producer/consumer calls `Writer.Complete()` in `finally`.
- [ ] `async void` is used only for top-level event handlers and always wrapped in `try/catch`.
- [ ] ASP.NET Core application code is not buried under redundant `ConfigureAwait(false)`.

#### Ресурсы / Resources

- [Microsoft Learn — ConfigureAwait — https://learn.microsoft.com/dotnet/api/system.threading.tasks.task.configureawait]
- [Microsoft Learn — SynchronizationContext — https://learn.microsoft.com/dotnet/api/system.threading.synchronizationcontext]
- [Stephen Toub — ConfigureAwait FAQ — https://devblogs.microsoft.com/dotnet/configureawait-faq/]
- [Channel<T> documentation — https://learn.microsoft.com/dotnet/api/system.threading.channels.channel-1]

---

[⬆ К модулю M09](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
