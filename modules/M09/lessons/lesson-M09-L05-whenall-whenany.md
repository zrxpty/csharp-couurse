[⬆ К модулю M09](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M09-L05: WhenAll/WhenAny / WhenAll/WhenAny

**Модуль / Module:** M09
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

`Task.WhenAll` и `Task.WhenAny` — это два композитора задач, которые превращают набор независимых асинхронных операций в единый `Task`. Они лежат в основе большинства реальных concurrency-сценариев: параллельной загрузки данных, таймаутов, гонок между альтернативными источниками.

**Task.WhenAll** ждёт **все** задачи из набора. Представьте официанта, который принимает заказ из пяти блюд и подаёт их одновременно: он не отдаёт столик, пока каждое блюдо не будет готово. Возвращаемый `Task` завершается, когда завершается последняя входная задача. Если все задачи вернули результат одного типа, `WhenAll` собирает их в массив `T[]` в **том же порядке**, в котором вы передали задачи, — независимо от того, какая завершилась первой. Это важно: порядок выдачи соответствует порядку входа, а не времени завершения.

Особенность `WhenAll` — **агрегация исключений**. Если хотя бы одна задача бросила исключение, результирующий `Task` становится faulted. Но `WhenAll` не теряет ошибки остальных: он дожидается всех задач и упаковывает их исключения в `AggregateException`. Если вы просто `await` без оборачивания, вы получите только первое исключение (`ExceptionDispatchInfo` прокидывает его «как есть»). Чтобы увидеть все, используйте `await Task.WhenAll(...).ConfigureAwait(false)` в блоке `try`, а затем читайте `task.Exception.InnerExceptions` или `Task.WhenAll`-результат через `Wait()`/`Async Counted`-обёртки. На практике чаще применяют паттерн «всё-или-сообщение»: оборачивают каждую операцию в свой `try/catch` и аккумулируют ошибки сами.

**Task.WhenAny** ждёт **первую** завершившуюся задачу. Метафора: вы разослали запрос в три бюро переводов и берёте ответ того, кто ответит первым, не дожидаясь остальных. Возвращается `Task<Task<T>>` (или `Task<Task>`), который завершается, как только любая из задач доходит до терминального состояния (RanToCompletion, Faulted или Canceled). Остальные задачи **продолжают работать** — это ключевая ловушка: `WhenAny` не отменяет «проигравших». Если они запускали побочные эффекты (HTTP-запросы, запись в БД), те завершатся «вхолостую». Поэтому всегда передавайте общий `CancellationTokenSource` и вызывайте `cts.Cancel()` после победителя.

**Таймаут через WhenAny** — самый частый паттерн: `await Task.WhenAny(workTask, Task.Delay(timeout, ct))`. Если победил `Task.Delay` — время вышло, отменяем работу через `ct`. В .NET 6+ появился `CancellationTokenSource.TryReset`, который позволяет переиспользовать источник токена, не создавая мусор на каждую операцию.

**Контекст синхронизации.** В GUI/ASP.NET (legacy) `await` по умолчанию пытается вернуться в исходный контекст. В библиотечном коде всегда используйте `.ConfigureAwait(false)`, чтобы не залочить UI-поток и не словить дедлок, когда вызывающий код блокируется через `.Result`. Дедлок классический: UI-поток вызывает `task.Result`, `task` ждёт возврата в UI-контекст, UI-поток занят `Result` → вечное ожидание.

**Race conditions.** Если несколько задач пишут в общий `List<T>`, вы получите порчу данных или `InvalidOperationException`. Решения: `ConcurrentDictionary`/`ConcurrentBag`, `lock` вокруг общей мутабельной структуры, или функциональный подход — каждая задача возвращает свой результат, а `WhenAll` их агрегирует (предпочтительно).

#### Theory (EN)

`Task.WhenAll` and `Task.WhenAny` are the two task composers that turn a set of independent asynchronous operations into a single `Task`. They underpin most real-world concurrency scenarios: parallel data loading, timeouts, races between alternative sources.

**Task.WhenAll** waits for **all** tasks in a set. Picture a waiter who accepts an order of five dishes and serves them together: the table is not released until every dish is ready. The returned `Task` completes when the last input task completes. If all tasks return the same type, `WhenAll` collects their results into a `T[]` array **in the same order you passed them in** — regardless of which finished first. This matters: the output order matches the input order, not completion time.

The distinctive feature of `WhenAll` is **exception aggregation**. If at least one task threw, the resulting `Task` becomes faulted. But `WhenAll` does not discard the others' errors: it waits for every task and packs their exceptions into `AggregateException`. If you simply `await` without wrapping, you only get the first exception (`ExceptionDispatchInfo` re-throws it as-is). To see all of them, wrap `await Task.WhenAll(...).ConfigureAwait(false)` in a `try` block and then inspect `task.Exception.InnerExceptions`, or handle errors per-task with your own accumulation logic. In practice the dominant pattern is "all-or-collect": wrap each operation in its own `try/catch` and aggregate failures yourself, so one slow endpoint does not hide three other failures.

**Task.WhenAny** waits for the **first** task to complete. Metaphor: you sent the same translation request to three agencies and take whoever replies first, without waiting for the rest. It returns `Task<Task<T>>` (or `Task<Task>`), which completes the moment any task reaches a terminal state (RanToCompletion, Faulted, or Canceled). The remaining tasks **keep running** — this is the key trap: `WhenAny` does not cancel the losers. If they triggered side effects (HTTP calls, DB writes), those will finish "for nothing". So always pass a shared `CancellationTokenSource` and call `cts.Cancel()` once you have a winner.

**Timeout via WhenAny** is the most common pattern: `await Task.WhenAny(workTask, Task.Delay(timeout, ct))`. If `Task.Delay` wins, time is up — cancel the work via `ct`. In .NET 6+ `CancellationTokenSource.TryReset` lets you reuse the source instead of allocating a new one per operation.

**Synchronization context.** In GUI / legacy ASP.NET, `await` by default tries to return to the captured context. In library code always use `.ConfigureAwait(false)` to avoid blocking the UI thread and to dodge the classic deadlock when the caller blocks on `.Result`. The deadlock: the UI thread calls `task.Result`; the `task` awaits return to the UI context; the UI thread is busy with `Result` → permanent wait.

**Race conditions.** If several tasks write to a shared `List<T>`, you get data corruption or `InvalidOperationException`. Solutions: `ConcurrentDictionary`/`ConcurrentBag`, a `lock` around a shared mutable structure, or — preferred — the functional approach where each task returns its own result and `WhenAll` aggregates them.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8+ — working, thread-safe, ConfigureAwait everywhere in library code.
using System.Collections.Concurrent;

// ---------- 1) WhenAll: parallel fan-out with safe aggregation ----------
public sealed class ParallelDownloader(HttpClient http)
{
    // Each task returns its own result or a structured failure — never throws.
    // Каждая задача возвращает свой результат или структурированную ошибку, без исключений.
    private async Task<DownloadResult> DownloadOneAsync(
        string url, CancellationToken ct)
    {
        try
        {
            // ConfigureAwait(false): библиотечный код не должен залочить UI-контекст вызывающего.
            using var resp = await http.GetAsync(url, ct).ConfigureAwait(false);
            var body = await resp.Content.ReadAsStringAsync(ct).ConfigureAwait(false);
            return new DownloadResult(url, ok: true, body, error: null);
        }
        catch (OperationCanceledException) when (ct.IsCancellationRequested)
        {
            // Корректная отмена — это не ошибка / Cancellation is not an error.
            return new DownloadResult(url, ok: false, body: null, error: "cancelled");
        }
        catch (Exception ex)
        {
            return new DownloadResult(url, ok: false, body: null, error: ex.Message);
        }
    }

    public async Task<IReadOnlyList<DownloadResult>> DownloadAllAsync(
        IReadOnlyList<string> urls, CancellationToken ct)
    {
        // Запускаем все задачи сразу — они идут конкурентно.
        // We start every task at once; they run concurrently.
        var tasks = urls.Select(u => DownloadOneAsync(u, ct)).ToArray();

        // WhenAll ждёт все. Порядок результатов = порядок urls, а не порядок завершения.
        // WhenAll waits for all. Result order matches urls, not completion order.
        DownloadResult[] results = await Task.WhenAll(tasks).ConfigureAwait(false);
        return results;
    }
}

public readonly record struct DownloadResult(
    string Url, bool Ok, string? Body, string? Error);

// ---------- 2) WhenAll с наблюдением всех исключений ----------
public static class WhenAllExceptionDemo
{
    // Если задачи могут бросать, WhenAll соберёт AggregateException.
    // If tasks may throw, WhenAll aggregates into AggregateException.
    public static async Task RunAsync(CancellationToken ct)
    {
        Task[] tasks =
        [
            Task.Run(() => throw new InvalidOperationException("boom-1"), ct),
            Task.Run(() => throw new FormatException("boom-2"), ct),
        ];

        Task all = Task.WhenAll(tasks);

        try
        {
            await all.ConfigureAwait(false);
        }
        catch
        {
            // Простой await прокидывает только ПЕРВОЕ исключение.
            // A plain await re-throws only the FIRST exception.
            // Чтобы увидеть все — читаем all.Exception:
            // To see all of them, inspect all.Exception:
            if (all.Exception is { } agg)
            {
                foreach (var inner in agg.InnerExceptions)
                    Console.WriteLine($"[RU] Собрано: {inner.Message} / [EN] Aggregated: {inner.Message}");
            }
        }
    }
}

// ---------- 3) WhenAny: timeout pattern ----------
public sealed class WithTimeout
{
    // Чистый таймаут: отменяем работу, если победил Task.Delay.
    // Pure timeout: cancel work if Task.Delay wins.
    public static async Task<T> RunAsync<T>(
        Func<CancellationToken, Task<T>> work, TimeSpan timeout, CancellationToken outerCt)
    {
        using var linked = CancellationTokenSource.CreateLinkedTokenSource(outerCt);
        linked.CancelAfter(timeout);                       // автоматический таймаут / auto timeout
        try
        {
            return await work(linked.Token).ConfigureAwait(false);
        }
        catch (OperationCanceledException) when (linked.IsCancellationRequested && !outerCt.IsCancellationRequested)
        {
            throw new TimeoutException($"Превышен таймаут {timeout} / Timed out after {timeout}");
        }
    }

    // Классический WhenAny-таймаут (если нельзя использовать CancelAfter).
    // Classic WhenAny timeout (when CancelAfter is not applicable).
    public static async Task<T> RunWhenAnyAsync<T>(
        Func<CancellationToken, Task<T>> work, TimeSpan timeout, CancellationToken outerCt)
    {
        using var workCts = CancellationTokenSource.CreateLinkedTokenSource(outerCt);
        var workTask = work(workCts.Token);
        var delayTask = Task.Delay(timeout, outerCt);

        Task finished = await Task.WhenAny(workTask, delayTask).ConfigureAwait(false);

        if (finished == delayTask)
        {
            // Время вышло — отменяем работу, иначе она "повиснет" в фоне.
            // Time is up — cancel work, otherwise it lingers in the background.
            await workCts.CancelAsync().ConfigureAwait(false);
            try { await workTask.ConfigureAwait(false); } catch { /* наблюдаем отмену / observe cancellation */ }
            throw new TimeoutException($"Timed out after {timeout}");
        }

        // delayTask ещё может выполняться — наблюдаем его, чтобы не было UnobservedTaskException.
        // delayTask may still be pending — observe it to avoid UnobservedTaskException.
        try { await delayTask.ConfigureAwait(false); } catch { /* игнор / ignore */ }
        return await workTask.ConfigureAwait(false);
    }
}

// ---------- 4) WhenAny: race between replicas, cancel losers ----------
public sealed class ReplicaRace(HttpClient http)
{
    public async Task<string> FirstOkAsync(IReadOnlyList<string> urls, CancellationToken ct)
    {
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
        var tasks = urls.Select(u => http.GetStringAsync(u, cts.Token)).ToList();
        while (tasks.Count > 0)
        {
            Task<string> winner = (Task<string>)await Task.WhenAny(tasks).ConfigureAwait(false);
            tasks.Remove(winner);
            try
            {
                string result = await winner.ConfigureAwait(false);
                await cts.CancelAsync().ConfigureAwait(false);   // отменяем проигравших / cancel losers
                return result;
            }
            catch (OperationCanceledException) when (cts.IsCancellationRequested)
            {
                // Победитель был отменён победителем более ранней итерации — продолжаем.
                // This winner got cancelled by an earlier iteration — keep racing.
                continue;
            }
            catch
            {
                // Один реплик упал — пробуем следующий / One replica failed — try the next.
                continue;
            }
        }
        throw new InvalidOperationException("Все реплики недоступны / All replicas failed");
    }
}

// ---------- 5) Потокобезопасная агрегация (правильный путь) ----------
public static class SafeAggregation
{
    // Плохо: общий List<T> под конкурентной записью → порча данных.
    // Bad: shared List<T> under concurrent writes → data corruption.
    //
    // Хорошо: каждая задача возвращает свой кусок, WhenAll собирает.
    // Good: each task returns its own chunk; WhenAll aggregates.

    public static async Task<int[]> ComputeSquaresAsync(int[] inputs, CancellationToken ct)
    {
        Task<int>[] tasks = inputs
            .Select(i => Task.Run(() => i * i, ct))
            .ToArray();
        return await Task.WhenAll(tasks).ConfigureAwait(false);
    }

    // Альтернатива: ConcurrentBag, если результаты нельзя вернуть из задачи.
    // Alternative: ConcurrentBag when results can't be returned from the task.
    public static async Task<IReadOnlyCollection<int>> ComputeWithBagAsync(int[] inputs, CancellationToken ct)
    {
        var bag = new ConcurrentBag<int>();
        await Task.WhenAll(inputs.Select(i => Task.Run(() => bag.Add(i * i), ct)))
            .ConfigureAwait(false);
        return bag.ToArray();
    }
}

public sealed record DownloadResultDto(string Url, int Length);

// ---------- 6) Сборка: реальный сервис с таймаутом и гонкой реплик ----------
public sealed class ResilientFetcher(HttpClient http)
{
    public async Task<DownloadResultDto> FetchAsync(string primaryUrl, string backupUrl, CancellationToken ct)
    {
        // Сначала — первичный источник с таймаутом 2 с.
        // First the primary source with a 2s timeout.
        try
        {
            string body = await WithTimeout.RunWhenAnyAsync(
                innerCt => http.GetStringAsync(primaryUrl, innerCt),
                TimeSpan.FromSeconds(2), ct).ConfigureAwait(false);
            return new DownloadResultDto(primaryUrl, body.Length);
        }
        catch (TimeoutException)
        {
            // Таймаут — fallback на гонку двух реплик.
            // Timed out — fall back to a race between two replicas.
            string body = await new ReplicaRace(http).FirstOkAsync([primaryUrl, backupUrl], ct)
                .ConfigureAwait(false);
            return new DownloadResultDto(backupUrl, body.Length);
        }
    }
}
```

#### Best Practices

- Передавайте `CancellationToken` во все `WhenAll`/`WhenAny`-операции и отменяйте «проигравших» через общий `CancellationTokenSource` после победителя в `WhenAny`.
- В библиотечном коде всегда используйте `.ConfigureAwait(false)` — это убирает зависимость от контекста синхронизации и предотвращает дедлоки с `.Result`.
- Предпочитайте функциональную агрегацию: каждая задача возвращает результат, а `WhenAll` собирает массив. Это потокобезопасно без `lock`.
- Для таймаутов в .NET 6+ предпочитайте `CancellationTokenSource.CreateLinkedTokenSource + CancelAfter` — это чище, чем ручной `WhenAny(work, Task.Delay)`.
- Оборачивайте каждую операцию в `try/catch`, когда важна устойчивость части набора, чтобы одна ошибка не маскировала остальные.

#### Best Practices (EN)

- Thread a `CancellationToken` through every `WhenAll`/`WhenAny` operation and cancel the losers through a shared `CancellationTokenSource` once `WhenAny` produces a winner.
- Always use `.ConfigureAwait(false)` in library code — it removes the dependence on a synchronization context and prevents `.Result` deadlocks.
- Prefer functional aggregation: each task returns its own result, `WhenAll` collects the array. This is thread-safe with no `lock` at all.
- For timeouts on .NET 6+, prefer `CancellationTokenSource.CreateLinkedTokenSource + CancelAfter` over a hand-rolled `WhenAny(work, Task.Delay)`.
- Wrap each operation in its own `try/catch` when partial success matters, so a single failure does not mask the others.

#### Частые ошибки / Common Mistakes

- `.Result` / `.Wait()` на `WhenAll` в UI-потоке → дедлок. Избегайте: используйте `await` + `ConfigureAwait(false)`.
- `WhenAny` без отмены проигравших → «зомби»-задачи, которые утекают и дорисуют побочные эффекты. Избегайте: общий `CancellationTokenSource` + `CancelAsync()` после победителя.
- Общий `List<T>` под конкурентной записью из нескольких задач → порча данных / `InvalidOperationException`. Избегайте: функциональная агрегация или `ConcurrentBag`/`ConcurrentDictionary`.
- `await Task.WhenAll(tasks)` в одном `try` ожидает увидеть все исключения, но ловит только первое. Избегайте: читайте `task.Exception.InnerExceptions` или собирайте ошибки по каждой задаче.
- `Task.Delay` не наблюдается после победы `workTask` в `WhenAny` → возможен `UnobservedTaskException`. Избегайте: `await delayTask` (или `Task.WhenAll` всей пары) для наблюдения.
- `async void` в обработчиках, которые вызывают `WhenAll` → необработанные исключения рвут процесс. Избегайте: `async Task` + явный `await`.

#### Common Mistakes (EN)

- `.Result` / `.Wait()` on `WhenAll` from the UI thread → deadlock. Avoid: use `await` + `ConfigureAwait(false)`.
- `WhenAny` without cancelling the losers → zombie tasks that leak and complete side effects. Avoid: shared `CancellationTokenSource` + `CancelAsync()` after a winner.
- A shared `List<T>` written concurrently from several tasks → data corruption / `InvalidOperationException`. Avoid: functional aggregation or `ConcurrentBag`/`ConcurrentDictionary`.
- `await Task.WhenAll(tasks)` inside a single `try` expecting to see every exception, but only the first is caught. Avoid: read `task.Exception.InnerExceptions` or collect errors per task.
- `Task.Delay` left unobserved after `workTask` wins a `WhenAny` → a possible `UnobservedTaskException`. Avoid: `await delayTask` (or `Task.WhenAll` the pair) to observe it.
- `async void` handlers that call `WhenAll` → unhandled exceptions crash the process. Avoid: `async Task` + explicit `await`.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Везде, где есть `await`, передаётся `CancellationToken`.
- [ ] В библиотечном коде после каждого `await` стоит `.ConfigureAwait(false)`.
- [ ] После победителя `WhenAny` вызывается `CancelAsync()` на общем `CancellationTokenSource`.
- [ ] Результаты `WhenAll` читаются по индексу входного массива, а не по времени завершения.
- [ ] Нигде нет `.Result`/`.Wait()` в потоках с контекстом синхронизации (UI/legacy ASP.NET).
- [ ] Общие мутабельные коллекции защищены `lock` или заменены на `Concurrent*`/функциональную агрегацию.
- [ ] Все исключения из набора задач наблюдаются (через `task.Exception` или per-task `try/catch`).
- [ ] Таймауты используют `CancelAfter` или корректно наблюдали `Task.Delay` после победы `workTask`.

#### Self-check Checklist (EN)

- [ ] A `CancellationToken` is passed through every `await`.
- [ ] Library code has `.ConfigureAwait(false)` after every `await`.
- [ ] The `WhenAny` winner triggers `CancelAsync()` on the shared `CancellationTokenSource`.
- [ ] `WhenAll` results are read by input-array index, not by completion time.
- [ ] No `.Result` / `.Wait()` on threads with a synchronization context (UI / legacy ASP.NET).
- [ ] Shared mutable collections are protected by `lock` or replaced with `Concurrent*` / functional aggregation.
- [ ] Every exception in the task set is observed (via `task.Exception` or per-task `try/catch`).
- [ ] Timeouts use `CancelAfter` or correctly observe `Task.Delay` after `workTask` wins.

#### Ресурсы / Resources

- [Microsoft Learn — Task.WhenAll](https://learn.microsoft.com/dotnet/api/system.threading.tasks.task.whenall)
- [Microsoft Learn — Task.WhenAny](https://learn.microsoft.com/dotnet/api/system.threading.tasks.task.whenany)
- [Stephen Toub — WhenAll, WhenAny and CancellationToken](https://learn.microsoft.com/dotnet/api/system.threading.cancellationtokensource)

---

[⬆ К модулю M09](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
