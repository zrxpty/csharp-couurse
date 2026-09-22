---
[← К уроку M09-L05](lesson-M09-L05-whenall-whenany.md) | [⬆ К модулю M09](../README.md) | [Предыдущее ДЗ ←](homework-M09-L04-task-run-cpu-vs-io.md) | [Следующее ДЗ →](homework-M09-L06-cancellation-token.md)
---

### Домашнее задание M09-L05: WhenAll/WhenAny / Homework M09-L05: WhenAll/WhenAny

**Урок / Lesson:** M09-L05
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться применять `Task.WhenAll` для параллельного фан-аута с устойчивостью к частичным сбоям, а `Task.WhenAny` — для гонки реплик и таймаутов, корректно отменяя проигравших и наблюдая все исключения. (EN) Learn to apply `Task.WhenAll` for parallel fan-out with partial-failure resilience and `Task.WhenAny` for replica races and timeouts, correctly cancelling losers and observing every exception.

#### Связь с уроком / Connection to the lesson
(RU) Задание прямо опирается на все шесть блоков кода урока: параллельный загрузчик с per-task `try/catch`, наблюдение `AggregateException` через `task.Exception`, таймаут через `WhenAny(work, Task.Delay)`, гонка реплик с отменой проигравших, безопасная агрегация и сборный `ResilientFetcher`. Вы будете воспроизводить и комбинировать эти паттерны в одном проекте. (EN) The assignment builds directly on all six code blocks of the lesson: the parallel downloader with per-task `try/catch`, observing `AggregateException` through `task.Exception`, the `WhenAny(work, Task.Delay)` timeout, the replica race with loser cancellation, safe aggregation, and the composite `ResilientFetcher`. You will reproduce and combine these patterns in one project.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы — backend-разработчик сервиса инвестиционного портфеля на C# 12 / .NET 8. Сервис должен по списку тикеров собрать актуальные котировки и показать клиенту единый отчёт «цена + изменения за день». Проблема в том, что источник котировок ненадёжен: часть эндпоинтов падает с 500, часть таймаутит, а часть вообще не отвечает. При этом бизнес-требование жёсткое: отчёт должен появляться даже тогда, когда 7 из 10 тикеров успешно получены, а 3 — нет; в отчёт должны попасть все полученные значения и аккуратный список ошибок по каждому упавшему тикеру. Дополнительно, для «горящих» тикеров, которые клиент пометил как приоритетные, сервис должен получить цену как можно быстрее, отправив запрос одновременно в основной и резервный дата-центры и взяв ответ того, кто ответит первым, не дожидаясь отстающего.

Это классический сценарий, где `Task.WhenAll` и `Task.WhenAny` работают вместе. `WhenAll` даёт вам параллельный фан-аут по всем тикерам с сохранением порядка и устойчивостью к частичным сбоям (каждая задача возвращает `Result` или `Error`, а не бросает). `WhenAny` даёт гонку реплик с таймаутом. Главные ловушки, которые вы должны обойти: общий мутабельный `List<T>` под конкурентной записью; `WhenAny` без отмены проигравших; `await Task.WhenAll(...)` в одном `try`, ловящий только первое исключение; оставленный ненаблюдаемым `Task.Delay`; отсутствие `ConfigureAwait(false)` в библиотечном коде. Урок подробно разбирает каждую из них — ваша задача — применить все best practices в одном цельном проекте.

#### Что нужно сделать (пошагово)

1. Создайте новый проект консольного приложения: `dotnet new console -n PortfolioQuotes -o PortfolioQuotes`, затем `cd PortfolioQuotes`. Убедитесь, что в `PortfolioQuotes.csproj` указано `<TargetFramework>net8.0</TargetFramework>` и язык C# 12 (по умолчанию в .NET 8).
2. Добавьте файл `Quote.cs` с типами предметной области:
   ```csharp
   public readonly record struct Quote(string Ticker, decimal Price, DateTimeOffset AsOf, bool Ok, string? Error);
   public readonly record struct QuoteReport(IReadOnlyList<Quote> Quotes, int SuccessCount, int FailureCount);
   ```
   Используйте `record struct` (C# 12) и `required`-члены там, где уместно. `Quote.Ok == false` означает, что задача вернула структурированную ошибку, а не бросила исключение.
3. Добавьте `IQuoteSource` и его фейковую реализацию `FakeQuoteSource`, которая по тикеру делает «HTTP-запрос» через `Task.Delay` и иногда бросает `HttpRequestException` или таймаутит. Конструктор принимает `TimeSpan baseLatency`, `double failureRate` и `Random`. Это позволит вам тестировать гонки и частичные сбои без реальной сети.
4. Создайте `ParallelQuoteFetcher(IQuoteSource source)` с методом `FetchAllAsync(IReadOnlyList<string> tickers, CancellationToken ct)`, который запускает запросы по всем тикерам одновременно через `WhenAll`. Каждая внутренняя задача `FetchOneAsync` обязана оборачивать источник в `try/catch` и возвращать `Quote` с `Ok=false` и `Error=ex.Message` вместо бросания. Везде используйте `.ConfigureAwait(false)`. Порядок результата должен совпадать с порядком входного списка `tickers`, а не с временем завершения.
5. Реализуйте `Task<QuoteReport> BuildReportAsync(IReadOnlyList<string> tickers, CancellationToken ct)` поверх `FetchAllAsync`. Подсчёт `SuccessCount`/`FailureCount` ведите по массиву результатов `WhenAll` (LINQ `Count(x => x.Ok)`), а не через общий `List<T>` с конкурентной записью — это будет race condition.
6. Добавьте `FastQuoteService` с методом `Task<Quote> FastestAsync(string ticker, IQuoteSource primary, IQuoteSource backup, TimeSpan timeout, CancellationToken ct)`, который одновременно опрашивает primary и backup через `WhenAny` и возвращает ответ первого, кто ответит успешно. После победителя вызовите `cts.CancelAsync()` для отмены проигравшего. Реализуйте таймаут через `Task.Delay(timeout, ct)` в той же гонке: если победил `Task.Delay` — отменьте обе задачи и бросьте `TimeoutException`. Обязательно «наблюдайте» `Task.Delay` после победы `workTask`, чтобы не получить `UnobservedTaskException`.
7. Соберите демо в `Program.cs` через top-level statements: создайте список из 10 тикеров, запустите `BuildReportAsync` и `FastestAsync`, выведите отчёт в консоль. Добавьте общий `CancellationTokenSource` с `CancelAfter(10s)` как верхний предохранитель.
8. Запустите: `dotnet run`. Убедитесь, что отчёт показывает, например: `Success: 7, Failure: 3` (точные числа зависят от `failureRate`), а `FastestAsync` возвращает цену за время, близкое к минимальной задержке двух источников.
9. Напишите два юнит-теста в `xUnit`: (а) при `failureRate=0` `SuccessCount` равен числу тикеров; (б) при `failureRate=1` `FailureCount` равен числу тикеров, и метод не бросает, а возвращает отчёт с ошибками. Используйте `CancellationToken.None` и стабильный `Random(42)` для воспроизводимости.
10. Проверьте, что нигде нет `.Result` или `.Wait()`, что после каждого `await` в библиотечных классах стоит `.ConfigureAwait(false)`, и что `WhenAny`-проигравшие отменяются.

#### Требования к решению

- Целевая платформа: .NET 8, язык C# 12. Используйте top-level statements в `Program.cs`, collection expressions (`[ .. ]`), `record struct`, pattern matching (`is { }`, `when`-клаузы в `catch`), primary constructors (`class ParallelQuoteFetcher(IQuoteSource source)`), `raw string literals` для многострочного вывода в демо, где это уместно.
- Все асинхронные методы принимают `CancellationToken ct` последним параметром и пробрасывают его вглубь до `Task.Delay`/`Task.Run`/`IQuoteSource.GetQuoteAsync`.
- В каждом `await` внутри `ParallelQuoteFetcher`, `FastQuoteService` и `BuildReportAsync` вызывается `.ConfigureAwait(false)` — это библиотечный код, который не должен зависеть от контекста синхронизации вызывающего.
- `FetchOneAsync` никогда не бросает: `OperationCanceledException` возвращается как `Quote(Ok:false, Error:"cancelled")` через `catch (OperationCanceledException) when (ct.IsCancellationRequested)`; прочие исключения — как `Quote(Ok:false, Error:ex.Message)`.
- `WhenAll` читается по индексу входного массива: `var tasks = tickers.Select(t => FetchOneAsync(t, ct)).ToArray(); var results = await Task.WhenAll(tasks)...`. Никаких конкурентных записей в общий `List<Quote>`.
- `FastQuoteService` использует общий `CancellationTokenSource.CreateLinkedTokenSource(ct)`, после победителя вызывает `await cts.CancelAsync().ConfigureAwait(false)`, а затем «наблюдает» проигравшего через `try { await loser.ConfigureAwait(false); } catch { }`, чтобы фиксировать его финальное состояние и не словить `UnobservedTaskException`.
- Демо в `Program.cs` выводит и `QuoteReport`, и результат `FastestAsync`, а также список тикеров, упавших с ошибкой, с текстом ошибки.
- Код компилируется без предупреждений `CA` уровня error, `dotnet build` зелёный, `dotnet run` не падает.

#### Тонкости и подводные камни

- **Порядок `WhenAll` против времени завершения.** `Task.WhenAll` возвращает `T[]` в порядке входного перечисления, а не в порядке завершения. Это значит, что даже если тикер `AAPL` ответит через 50 мс, а `MSFT` через 500 мс, в массиве результатов `AAPL` будет на том месте, на котором он стоял во входном списке. Если ваш код случайно сортирует задачи по времени завершения — вы нарушите инвариант «входной индекс = выходной индекс» и сломаете сопоставление тикер→цена.
- **`await Task.WhenAll(...)` ловит только первое исключение.** Если все внутренние задачи бросают, простой `try { await Task.WhenAll(tasks); } catch (Exception ex)` даст вам только `ex` от первой бросившей задачи. Остальные будут «схлопнуты» в `task.Exception.InnerExceptions`. В уроке показано два пути: либо читать `all.Exception` после `catch`, либо — предпочтительно — оборачивать каждую операцию в собственный `try/catch` и возвращать структурированный результат. Вы должны использовать второй подход в `FetchOneAsync`.
- **`WhenAny` не отменяет проигравших.** После `await Task.WhenAny(tasks)` остальные задачи продолжают работать. Если они делают HTTP-запросы, те дорисуются «вхолостую», потребляя сокеты и память. Обязательно: общий `CancellationTokenSource`, `CancelAsync()` после победителя, и наблюдение финального состояния проигравшего через `try { await loser; } catch { }`.
- **`Task.Delay` после победы `workTask` остаётся ненаблюдаемым.** Если `workTask` выиграл гонку у `Task.Delay(timeout, ct)`, `delayTask` всё ещё «висит». Когда он завершится, а вы его не `await`-нули, в .NET Framework это могло породить `UnobservedTaskException` в финализаторе; в .NET 8 поведение мягче, но шум в логах остаётся. Лечится `try { await delayTask.ConfigureAwait(false); } catch { }` или `Task.WhenAll(workTask, delayTask)` после определения победителя.
- **`ConfigureAwait(false)` обязателен в библиотеке.** Если вызывающий код — UI или legacy ASP.NET с контекстом синхронизации, а вы забыли `ConfigureAwait(false)`, то ваш `await` попытается вернуться в UI-контекст. Если при этом вызывающий сделал `.Result` — классический дедлок: UI-поток занят `.Result`, а continuation ждёт UI-контекст.
- **Конкурентная запись в `List<T>`.** `List<T>.Add` не потокобезопасен. Две задачи, одновременно вызывающие `Add`, могут перетереть внутренний массив и бросить `InvalidOperationException: Collection was modified`. Решения из урока: `ConcurrentBag<T>`, `lock`, или — предпочтительно — функциональная агрегация через `WhenAll` (каждая задача возвращает свой результат, массив собирает композитор). В этом ДЗ вы обязаны использовать функциональную агрегацию.
- **`CancelAfter` против ручного `WhenAny(work, Task.Delay)`.** В .NET 6+ чище использовать `CancellationTokenSource.CreateLinkedTokenSource(ct)` + `linked.CancelAfter(timeout)`. Но в задании вы намеренно реализуете оба варианта, чтобы прочувствовать разницу: `CancelAfter` в `BuildReportAsync` как верхний предохранитель, и классический `WhenAny(work, delay)` в `FastQuoteService`, где нужно вернуть именно `TimeoutException`, а не `OperationCanceledException`.

#### Критерии приёмки

- [ ] Проект `PortfolioQuotes` собирается `dotnet build` без ошибок и предупреждений уровня error.
- [ ] `dotnet run` выводит отчёт `QuoteReport` со списком `Quotes`, `SuccessCount`, `FailureCount` и результатом `FastestAsync`.
- [ ] Все асинхронные методы принимают `CancellationToken` и пробрасывают его вглубь.
- [ ] После каждого `await` в библиотечных классах стоит `.ConfigureAwait(false)`.
- [ ] `FetchOneAsync` не бросает исключения: возвращает `Quote(Ok:false, ...)` для всех ошибок и отмены.
- [ ] `WhenAll` в `FetchAllAsync` возвращает массив в порядке входного списка тикеров.
- [ ] `BuildReportAsync` не использует общий мутабельный `List<T>` под конкурентной записью; счётчики вычисляются по массиву результата `WhenAll`.
- [ ] `FastQuoteService` использует `WhenAny` и после победителя вызывает `cts.CancelAsync()`.
- [ ] `Task.Delay`-ветка таймаута в `FastQuoteService` наблюдается через `try { await delayTask; } catch { }`.
- [ ] После победы `workTask` проигравший наблюдается через `try { await loser; } catch { }`.
- [ ] `OperationCanceledException` отличается от прочих через `catch (...) when (ct.IsCancellationRequested)`.
- [ ] В коде нет ни одного `.Result` или `.Wait()`.
- [ ] Использованы средства C# 12: top-level statements, collection expressions, `record struct`, primary constructors, pattern matching.
- [ ] Юнит-тесты `xUnit` покрывают сценарии `failureRate=0` и `failureRate=1`.
- [ ] Демонстрационный запуск детерминирован при `Random(42)` (воспроизводимые задержки и сбои).

#### Подсказки (без прямого ответа)

- Вспомните шаблон урока `DownloadOneAsync` — он уже почти готовый `FetchOneAsync`, нужно только заменить тип результата.
- Для «наблюдения проигравшего» подумайте, что возвращает `Task.WhenAny`: это `Task<Task<T>>`. Внутренний `Task<T>` — и есть проигравший, которого вы должны `await`-нуть в `try/catch`.
- Для подсчёта `SuccessCount` используйте `results.Count(q => q.Ok)` — это LINQ над уже собранным массивом, никаких конкурентных мутаций.
- Чтобы отличить «истинную отмену по токену» от «таймаута через `Task.Delay`», заведите флаг `bool timedOut = (finished == delayTask)` и кидайте `TimeoutException` именно в этой ветке.
- Для верхнего предохранителя в `Program.cs` подходит `using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));` — это эквивалент `CancelAfter`.
- Не забудьте, что `Random` не потокобезопасен; в `FakeQuoteSource` либо используйте `Random.Shared`, либо `lock` вокруг `NextDouble()`.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — PortfolioQuotes. Top-level Program.cs + library classes.
// Полный рабочий код. Комментарии RU+EN.

using System.Collections.Concurrent;

// ---------- Доменные типы / Domain types ----------
public readonly record struct Quote(
    string Ticker, decimal Price, DateTimeOffset AsOf, bool Ok, string? Error);

public readonly record struct QuoteReport(
    IReadOnlyList<Quote> Quotes, int SuccessCount, int FailureCount);

// ---------- Контракт источника и фейк / Source contract + fake ----------
public interface IQuoteSource
{
    Task<decimal> GetQuoteAsync(string ticker, CancellationToken ct);
}

public sealed class FakeQuoteSource(TimeSpan baseLatency, double failureRate, int? seed = null) : IQuoteSource
{
    private readonly Random _rng = seed is { } s ? new Random(s) : Random.Shared;

    public async Task<decimal> GetQuoteAsync(string ticker, CancellationToken ct)
    {
        // Имитация сети: случайная задержка вокруг baseLatency.
        // Network emulation: random jitter around baseLatency.
        var jitter = TimeSpan.FromMilliseconds(_rng.NextDouble() * 200);
        await Task.Delay(baseLatency + jitter, ct).ConfigureAwait(false);

        // Случайный сбой / Random failure.
        if (_rng.NextDouble() < failureRate)
            throw new HttpRequestException($"HTTP 500 for {ticker}");

        return Math.Round(100m + (decimal)_rng.NextDouble() * 50m, 2);
    }
}

// ---------- Параллельный фан-аут через WhenAll / Parallel fan-out ----------
public sealed class ParallelQuoteFetcher(IQuoteSource source)
{
    private async Task<Quote> FetchOneAsync(string ticker, CancellationToken ct)
    {
        try
        {
            // ConfigureAwait(false): библиотечный код не залочит UI-контекст вызывающего.
            // Library code must not capture the caller's sync context.
            decimal price = await source.GetQuoteAsync(ticker, ct).ConfigureAwait(false);
            return new Quote(ticker, price, DateTimeOffset.UtcNow, Ok: true, Error: null);
        }
        catch (OperationCanceledException) when (ct.IsCancellationRequested)
        {
            // Корректная отмена — это не ошибка / Cancellation is not an error.
            return new Quote(ticker, Price: 0m, DateTimeOffset.UtcNow, Ok: false, Error: "cancelled");
        }
        catch (Exception ex)
        {
            return new Quote(ticker, Price: 0m, DateTimeOffset.UtcNow, Ok: false, Error: ex.Message);
        }
    }

    public async Task<QuoteReport> FetchAllAsync(IReadOnlyList<string> tickers, CancellationToken ct)
    {
        // Запускаем все задачи сразу — они идут конкурентно.
        // Start every task at once; they run concurrently.
        Task<Quote>[] tasks = tickers.Select(t => FetchOneAsync(t, ct)).ToArray();

        // WhenAll ждёт все. Порядок результатов = порядок tickers, а не порядок завершения.
        // WhenAll waits for all. Order matches tickers, not completion order.
        Quote[] results = await Task.WhenAll(tasks).ConfigureAwait(false);

        // Подсчёт по массиву — никаких общих List<T> под конкурентной записью.
        // Count over the array — no shared List<T> under concurrent writes.
        int success = results.Count(q => q.Ok);
        return new QuoteReport(results, success, results.Length - success);
    }
}

// ---------- Гонка реплик через WhenAny + таймаут / Replica race + timeout ----------
public sealed class FastQuoteService
{
    public static async Task<Quote> FastestAsync(
        string ticker, IQuoteSource primary, IQuoteSource backup,
        TimeSpan timeout, CancellationToken outerCt)
    {
        // Общий токен для отмены проигравших / Shared token to cancel losers.
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(outerCt);

        Task<decimal> primaryTask  = primary.GetQuoteAsync(ticker, cts.Token);
        Task<decimal> backupTask   = backup.GetQuoteAsync(ticker, cts.Token);
        Task         delayTask     = Task.Delay(timeout, outerCt);

        List<Task<decimal>> racers = [primaryTask, backupTask];

        while (racers.Count > 0)
        {
            // WhenAny возвращает Task<Task<decimal>> — завершившийся внутренний.
            // WhenAny returns Task<Task<decimal>> — the completed inner task.
            Task<decimal> winner = (Task<decimal>)await Task
                .WhenAny(racers.Append(delayTask).Cast<Task>())
                .ConfigureAwait(false);

            if (winner == delayTask)
            {
                // Время вышло — отменяем работу и наблюдаем проигравших.
                // Time is up — cancel work and observe losers.
                await cts.CancelAsync().ConfigureAwait(false);
                await ObserveAllAsync(primaryTask, backupTask).ConfigureAwait(false);
                throw new TimeoutException($"Таймаут {timeout} для {ticker} / Timed out after {timeout} for {ticker}");
            }

            racers.Remove(winner);

            try
            {
                decimal price = await winner.ConfigureAwait(false);
                // Победитель найден — отменяем проигравших / Winner found — cancel losers.
                await cts.CancelAsync().ConfigureAwait(false);
                // Наблюдаем проигравшего, чтобы не словить UnobservedTaskException.
                // Observe the loser to avoid UnobservedTaskException.
                await ObserveAllAsync(primaryTask, backupTask).ConfigureAwait(false);
                return new Quote(ticker, price, DateTimeOffset.UtcNow, Ok: true, Error: null);
            }
            catch (OperationCanceledException) when (cts.IsCancellationRequested)
            {
                // Победитель отменён более ранней итерацией — продолжаем гонку.
                // Winner cancelled by an earlier iteration — keep racing.
                continue;
            }
            catch
            {
                // Один источник упал — пробуем следующий / One source failed — try the next.
                continue;
            }
        }

        throw new InvalidOperationException($"Все источники упали для {ticker} / All sources failed for {ticker}");
    }

    private static async Task ObserveAllAsync(params Task[] tasks)
    {
        // Наблюдаем финальное состояние каждого, не давая исключениям стать "не наблюдаемыми".
        // Observe each task's final state so exceptions are not "unobserved".
        foreach (var t in tasks)
        {
            try { await t.ConfigureAwait(false); }
            catch { /* намеренно глотаем / intentionally swallowed */ }
        }
    }
}

// ---------- Program.cs (top-level statements) ----------
/*
var primary = new FakeQuoteSource(TimeSpan.FromMilliseconds(150), 0.2, seed: 42);
var backup  = new FakeQuoteSource(TimeSpan.FromMilliseconds(120), 0.1, seed: 7);
var fetcher = new ParallelQuoteFetcher(primary);

using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10)); // верхний предохранитель / top guard
string[] tickers = ["AAPL", "MSFT", "GOOG", "AMZN", "TSLA", "NVDA", "META", "NFLX", "INTC", "AMD"];

QuoteReport report = await fetcher.FetchAllAsync(tickers, cts.Token);
Console.WriteLine($"Отчёт / Report: success={report.SuccessCount}, failure={report.FailureCount}");
foreach (var q in report.Quotes)
    Console.WriteLine(q.Ok ? $"  {q.Ticker}: {q.Price}" : $"  {q.Ticker}: ERROR {q.Error}");

Quote fast = await FastQuoteService.FastestAsync("AAPL", primary, backup, TimeSpan.FromMilliseconds(500), cts.Token);
Console.WriteLine($"Быстрая котировка / Fast quote: {fast.Ticker} = {fast.Price}");
*/
```

Разбор по строкам. `FakeQuoteSource` моделирует сеть через `Task.Delay(baseLatency + jitter, ct)` — обратите внимание, что токен пробрасывается в `Task.Delay`, иначе отмена не сработает во время ожидания. `failureRate` управляет долей `HttpRequestException`, и каждая задача в `FetchOneAsync` оборачивает вызов в `try/catch`, возвращая `Quote(Ok:false)` вместо бросания — это и есть паттерн «all-or-collect» из урока, который не даёт одному упавшему тикеру маскировать остальные через `AggregateException`. Конструкция `catch (OperationCanceledException) when (ct.IsCancellationRequested)` принципиальна: она отличает «нас отменили внешним токеном» от «источник внутри себя бросил `OperationCanceledException`» — без `when` вы бы проглотили настоящую ошибку источника.

`FetchAllAsync` запускает все задачи одновременно через `tickers.Select(...).ToArray()` — это важно: если бы вы `await`-нули каждую по очереди, параллелизма не было бы. `Task.WhenAll(tasks)` возвращает массив в порядке входного перечисления, поэтому `results[i]` всегда соответствует `tickers[i]`, даже если `tickers[0]` завершился последним. Подсчёт `success` идёт по уже собранному массиву через LINQ `Count(q => q.Ok)` — это функциональная агрегация из урока, потокобезопасная без `lock`. Никакого общего `List<Quote>` с конкурентным `Add` здесь нет, и именно поэтому нет race condition.

`FastQuoteService.FastestAsync` — самая деликатная часть. Общий `CancellationTokenSource.CreateLinkedTokenSource(outerCt)` связывает внешний токен с внутренним, чтобы отмена «сверху» автоматически отменила гонку, а внутренняя отмена проигравших не трогала внешний. `WhenAny(racers.Append(delayTask))` устраивает гонку между двумя источниками и таймером. Если победил `delayTask` — время вышло: мы вызываем `cts.CancelAsync()` для отмены обоих источников, затем `ObserveAllAsync` фиксирует их финальное состояние (важно: без наблюдения `UnobservedTaskException` может всплыть позже), и кидаем `TimeoutException`. Если победил один из источников — мы читаем его результат; при успехе отменяем проигравшего через тот же `cts.CancelAsync()` и наблюдаем всех через `ObserveAllAsync`. `catch (OperationCanceledException) when (cts.IsCancellationRequested)` обрабатывает редкий случай: победитель этой итерации уже был отменён победителем предыдущей — мы просто продолжаем гонку. Наконец, верхний предохранитель в `Program.cs` через `new CancellationTokenSource(TimeSpan.FromSeconds(10))` — это эквивалент `CancelAfter`, рекомендованный в best practices урока как более чистая альтернатива ручному `WhenAny(work, Task.Delay)`.

#### Задания на углубление (бонус)

1. Добавьте перегрузку `FetchAllAsync`, которая ограничивает степень параллелизма через `SemaphoreSlim` (например, не больше 4 одновременных запросов), и сравните время выполнения при 100 тикерах с неограниченной версией.
2. Реализуйте вариант таймаута через `CancellationTokenSource.CreateLinkedTokenSource(outerCt)` + `linked.CancelAfter(timeout)` вместо ручного `WhenAny(work, Task.Delay)`, и сравните читабельность и поведение при отмене.
3. Добавьте наблюдение всех исключений через `task.Exception.InnerExceptions` в отдельном демо-методе, где задачи реально бросают (а не возвращают структурированный результат), и выведите их все в лог.
4. Превратите `FakeQuoteSource` в `HttpQuoteSource` поверх реального `HttpClient` с Polly retry, и убедитесь, что `WhenAll` корректно работает с `HttpRequestException` и `TaskCanceledException` от Polly.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are a backend engineer on an investment portfolio service written in C# 12 / .NET 8. The service must, given a list of stock tickers, fetch up-to-date quotes and present the client with a single report of "price plus daily change". The problem is that the quote source is unreliable: some endpoints return HTTP 500, some time out, and some never respond at all. Yet the business requirement is strict — the report must appear even when 7 of 10 tickers succeed and 3 fail; the report must include every fetched price together with a clean list of per-ticker errors. On top of that, for "hot" tickers the client has flagged as priority, the service must obtain the price as fast as possible by sending the request simultaneously to the primary and the backup data center and taking whoever answers first, without waiting for the slower one.

This is a textbook scenario where `Task.WhenAll` and `Task.WhenAny` cooperate. `WhenAll` gives you a parallel fan-out over all tickers that preserves order and tolerates partial failures (each task returns a `Result` or an `Error` instead of throwing). `WhenAny` gives you a replica race with a timeout. The main traps you must avoid are: a shared mutable `List<T>` written concurrently; `WhenAny` without cancelling the losers; `await Task.WhenAll(...)` in a single `try` that catches only the first exception; a `Task.Delay` left unobserved after the winner; and missing `ConfigureAwait(false)` in library code. The lesson walks through each of them in detail — your job is to apply every best practice in one cohesive project.

#### What to do step by step

1. Create a new console application: `dotnet new console -n PortfolioQuotes -o PortfolioQuotes`, then `cd PortfolioQuotes`. Confirm that `PortfolioQuotes.csproj` targets `<TargetFramework>net8.0</TargetFramework>` and C# 12 (the .NET 8 default).
2. Add `Quote.cs` with the domain types:
   ```csharp
   public readonly record struct Quote(string Ticker, decimal Price, DateTimeOffset AsOf, bool Ok, string? Error);
   public readonly record struct QuoteReport(IReadOnlyList<Quote> Quotes, int SuccessCount, int FailureCount);
   ```
   Use `record struct` (C# 12) and `required` members where appropriate. `Quote.Ok == false` means the task returned a structured failure, not that it threw.
3. Add `IQuoteSource` and a fake implementation `FakeQuoteSource` that, given a ticker, performs an "HTTP call" via `Task.Delay` and occasionally throws `HttpRequestException` or stalls forever. The constructor accepts `TimeSpan baseLatency`, `double failureRate`, and a `Random`. This lets you test races and partial failures without a real network.
4. Create `ParallelQuoteFetcher(IQuoteSource source)` with a `FetchAllAsync(IReadOnlyList<string> tickers, CancellationToken ct)` method that starts requests for every ticker concurrently through `WhenAll`. Each inner `FetchOneAsync` task must wrap the source in a `try/catch` and return `Quote` with `Ok=false` and `Error=ex.Message` instead of throwing. Use `.ConfigureAwait(false)` everywhere. The result order must match the input `tickers` order, not completion time.
5. Implement `Task<QuoteReport> BuildReportAsync(IReadOnlyList<string> tickers, CancellationToken ct)` on top of `FetchAllAsync`. Compute `SuccessCount`/`FailureCount` from the `WhenAll` result array via LINQ (`Count(x => x.Ok)`), not from a shared `List<T>` mutated concurrently — that would be a race condition.
6. Add `FastQuoteService` with `Task<Quote> FastestAsync(string ticker, IQuoteSource primary, IQuoteSource backup, TimeSpan timeout, CancellationToken ct)` that concurrently polls primary and backup via `WhenAny` and returns the first successful answer. After the winner, call `cts.CancelAsync()` to cancel the loser. Implement the timeout with `Task.Delay(timeout, ct)` in the same race: if `Task.Delay` wins, cancel both tasks and throw `TimeoutException`. Make sure to "observe" `Task.Delay` after `workTask` wins, so you never get an `UnobservedTaskException`.
7. Assemble a demo in `Program.cs` using top-level statements: build a list of 10 tickers, run `BuildReportAsync` and `FastestAsync`, and print the report to the console. Add a top-level `CancellationTokenSource` with `CancelAfter(10s)` as an outer guard.
8. Run: `dotnet run`. Confirm the report shows, for example, `Success: 7, Failure: 3` (the exact numbers depend on `failureRate`), and that `FastestAsync` returns a price in roughly the minimum latency of the two sources.
9. Write two `xUnit` tests: (a) at `failureRate=0`, `SuccessCount` equals the ticker count; (b) at `failureRate=1`, `FailureCount` equals the ticker count and the method does not throw — it returns a report with errors. Use `CancellationToken.None` and a stable `Random(42)` for reproducibility.
10. Verify there is no `.Result` or `.Wait()` anywhere, that every `await` inside library classes is followed by `.ConfigureAwait(false)`, and that `WhenAny` losers are cancelled.

#### Requirements

- Target platform: .NET 8, language C# 12. Use top-level statements in `Program.cs`, collection expressions (`[ .. ]`), `record struct`, pattern matching (`is { }`, `when` clauses in `catch`), primary constructors (`class ParallelQuoteFetcher(IQuoteSource source)`), and raw string literals for the multi-line demo output where appropriate.
- Every async method accepts `CancellationToken ct` as the last parameter and threads it down to `Task.Delay`/`Task.Run`/`IQuoteSource.GetQuoteAsync`.
- Every `await` inside `ParallelQuoteFetcher`, `FastQuoteService`, and `BuildReportAsync` is followed by `.ConfigureAwait(false)` — this is library code that must not depend on the caller's synchronization context.
- `FetchOneAsync` never throws: `OperationCanceledException` comes back as `Quote(Ok:false, Error:"cancelled")` via `catch (OperationCanceledException) when (ct.IsCancellationRequested)`; other exceptions become `Quote(Ok:false, Error:ex.Message)`.
- `WhenAll` is read by input-array index: `var tasks = tickers.Select(t => FetchOneAsync(t, ct)).ToArray(); var results = await Task.WhenAll(tasks)...`. No concurrent writes to a shared `List<Quote>`.
- `FastQuoteService` uses a shared `CancellationTokenSource.CreateLinkedTokenSource(ct)`, calls `await cts.CancelAsync().ConfigureAwait(false)` after the winner, and then "observes" the loser via `try { await loser.ConfigureAwait(false); } catch { }` to fixate its terminal state and avoid `UnobservedTaskException`.
- The `Program.cs` demo prints both the `QuoteReport` and the `FastestAsync` result, plus the list of tickers that failed with their error messages.
- The code compiles without `CA`-level warnings as errors, `dotnet build` is green, `dotnet run` does not crash.

#### Pitfalls

- **`WhenAll` order vs completion time.** `Task.WhenAll` returns `T[]` in input-enumeration order, not completion order. Even if `AAPL` answers in 50 ms and `MSFT` in 500 ms, the result array still places `AAPL` at its input index. If your code accidentally sorts tasks by completion time, you break the "input index = output index" invariant and mis-map tickers to prices.
- **`await Task.WhenAll(...)` catches only the first exception.** If every inner task throws, a plain `try { await Task.WhenAll(tasks); } catch (Exception ex)` gives you only the first thrower's `ex`. The rest are folded into `task.Exception.InnerExceptions`. The lesson shows two remedies: read `all.Exception` after `catch`, or — preferred — wrap each operation in its own `try/catch` and return a structured result. You must use the second approach in `FetchOneAsync`.
- **`WhenAny` does not cancel the losers.** After `await Task.WhenAny(tasks)`, the remaining tasks keep running. If they issue HTTP calls, those calls finish "for nothing", burning sockets and memory. Always: a shared `CancellationTokenSource`, `CancelAsync()` after the winner, and observation of each loser's terminal state via `try { await loser; } catch { }`.
- **`Task.Delay` is left unobserved after `workTask` wins.** If `workTask` beats `Task.Delay(timeout, ct)`, `delayTask` is still pending. When it later completes and you never `await`-ed it, .NET Framework could raise `UnobservedTaskException` in the finalizer; .NET 8 is gentler, but the log noise remains. Fix it with `try { await delayTask.ConfigureAwait(false); } catch { }` or by `Task.WhenAll(workTask, delayTask)` after the winner is known.
- **`ConfigureAwait(false)` is mandatory in library code.** If the caller is UI or legacy ASP.NET with a synchronization context and you forgot `ConfigureAwait(false)`, your `await` tries to return to that context. If the caller also did `.Result`, you get the classic deadlock: the UI thread is busy with `.Result`, while the continuation waits for the UI context.
- **Concurrent writes to `List<T>`.** `List<T>.Add` is not thread-safe. Two tasks calling `Add` simultaneously can corrupt the internal array and throw `InvalidOperationException: Collection was modified`. The lesson's remedies: `ConcurrentBag<T>`, a `lock`, or — preferred — functional aggregation through `WhenAll` (each task returns its result, the composer assembles the array). In this homework you must use functional aggregation.
- **`CancelAfter` vs a hand-rolled `WhenAny(work, Task.Delay)`.** On .NET 6+ it is cleaner to use `CancellationTokenSource.CreateLinkedTokenSource(ct)` + `linked.CancelAfter(timeout)`. But the assignment deliberately asks for both variants so you feel the difference: `CancelAfter` in `BuildReportAsync` as an outer guard, and the classic `WhenAny(work, delay)` in `FastQuoteService`, where you specifically need to throw `TimeoutException` rather than `OperationCanceledException`.

#### Acceptance criteria

- [ ] The `PortfolioQuotes` project builds with `dotnet build` with no errors or error-level warnings.
- [ ] `dotnet run` prints a `QuoteReport` with the `Quotes` list, `SuccessCount`, `FailureCount`, and the `FastestAsync` result.
- [ ] Every async method accepts a `CancellationToken` and threads it down.
- [ ] Every `await` inside library classes is followed by `.ConfigureAwait(false)`.
- [ ] `FetchOneAsync` never throws: it returns `Quote(Ok:false, ...)` for all errors and cancellations.
- [ ] `WhenAll` in `FetchAllAsync` returns an array in input-ticker order.
- [ ] `BuildReportAsync` does not use a shared mutable `List<T>` under concurrent writes; counters are computed from the `WhenAll` result array.
- [ ] `FastQuoteService` uses `WhenAny` and calls `cts.CancelAsync()` after the winner.
- [ ] The `Task.Delay` timeout branch in `FastQuoteService` is observed via `try { await delayTask; } catch { }`.
- [ ] After `workTask` wins, the loser is observed via `try { await loser; } catch { }`.
- [ ] `OperationCanceledException` is distinguished from other exceptions via `catch (...) when (ct.IsCancellationRequested)`.
- [ ] There is no `.Result` or `.Wait()` anywhere in the code.
- [ ] C# 12 features are used: top-level statements, collection expressions, `record struct`, primary constructors, pattern matching.
- [ ] `xUnit` tests cover the `failureRate=0` and `failureRate=1` scenarios.
- [ ] The demo run is deterministic under `Random(42)` (reproducible latencies and failures).

#### Hints (no direct answer)

- Recall the lesson's `DownloadOneAsync` template — it is almost a ready-made `FetchOneAsync`; you only need to swap the result type.
- For "observing the loser", remember what `Task.WhenAny` returns: a `Task<Task<T>>`. The inner `Task<T>` is the loser you must `await` inside a `try/catch`.
- For `SuccessCount`, use `results.Count(q => q.Ok)` — LINQ over the already-assembled array, no concurrent mutations.
- To tell "genuine token cancellation" from "timeout via `Task.Delay`", keep a flag `bool timedOut = (finished == delayTask)` and throw `TimeoutException` only in that branch.
- For the top-level guard in `Program.cs`, `using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));` is the equivalent of `CancelAfter`.
- Remember that `Random` is not thread-safe; in `FakeQuoteSource` use `Random.Shared` or `lock` around `NextDouble()`.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — PortfolioQuotes. Top-level Program.cs + library classes.
// Full working code. EN comments.

using System.Collections.Concurrent;

// ---------- Domain types ----------
public readonly record struct Quote(
    string Ticker, decimal Price, DateTimeOffset AsOf, bool Ok, string? Error);

public readonly record struct QuoteReport(
    IReadOnlyList<Quote> Quotes, int SuccessCount, int FailureCount);

// ---------- Source contract + fake ----------
public interface IQuoteSource
{
    Task<decimal> GetQuoteAsync(string ticker, CancellationToken ct);
}

public sealed class FakeQuoteSource(TimeSpan baseLatency, double failureRate, int? seed = null) : IQuoteSource
{
    private readonly Random _rng = seed is { } s ? new Random(s) : Random.Shared;

    public async Task<decimal> GetQuoteAsync(string ticker, CancellationToken ct)
    {
        // Network emulation: random jitter around baseLatency.
        var jitter = TimeSpan.FromMilliseconds(_rng.NextDouble() * 200);
        await Task.Delay(baseLatency + jitter, ct).ConfigureAwait(false);

        // Random failure.
        if (_rng.NextDouble() < failureRate)
            throw new HttpRequestException($"HTTP 500 for {ticker}");

        return Math.Round(100m + (decimal)_rng.NextDouble() * 50m, 2);
    }
}

// ---------- Parallel fan-out via WhenAll ----------
public sealed class ParallelQuoteFetcher(IQuoteSource source)
{
    private async Task<Quote> FetchOneAsync(string ticker, CancellationToken ct)
    {
        try
        {
            // ConfigureAwait(false): library code must not capture the caller's sync context.
            decimal price = await source.GetQuoteAsync(ticker, ct).ConfigureAwait(false);
            return new Quote(ticker, price, DateTimeOffset.UtcNow, Ok: true, Error: null);
        }
        catch (OperationCanceledException) when (ct.IsCancellationRequested)
        {
            // Cancellation is not an error.
            return new Quote(ticker, Price: 0m, DateTimeOffset.UtcNow, Ok: false, Error: "cancelled");
        }
        catch (Exception ex)
        {
            return new Quote(ticker, Price: 0m, DateTimeOffset.UtcNow, Ok: false, Error: ex.Message);
        }
    }

    public async Task<QuoteReport> FetchAllAsync(IReadOnlyList<string> tickers, CancellationToken ct)
    {
        // Start every task at once; they run concurrently.
        Task<Quote>[] tasks = tickers.Select(t => FetchOneAsync(t, ct)).ToArray();

        // WhenAll waits for all. Order matches tickers, not completion order.
        Quote[] results = await Task.WhenAll(tasks).ConfigureAwait(false);

        // Count over the array — no shared List<T> under concurrent writes.
        int success = results.Count(q => q.Ok);
        return new QuoteReport(results, success, results.Length - success);
    }
}

// ---------- Replica race via WhenAny + timeout ----------
public sealed class FastQuoteService
{
    public static async Task<Quote> FastestAsync(
        string ticker, IQuoteSource primary, IQuoteSource backup,
        TimeSpan timeout, CancellationToken outerCt)
    {
        // Shared token to cancel losers.
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(outerCt);

        Task<decimal> primaryTask  = primary.GetQuoteAsync(ticker, cts.Token);
        Task<decimal> backupTask   = backup.GetQuoteAsync(ticker, cts.Token);
        Task         delayTask     = Task.Delay(timeout, outerCt);

        List<Task<decimal>> racers = [primaryTask, backupTask];

        while (racers.Count > 0)
        {
            // WhenAny returns Task<Task<decimal>> — the completed inner task.
            Task<decimal> winner = (Task<decimal>)await Task
                .WhenAny(racers.Append(delayTask).Cast<Task>())
                .ConfigureAwait(false);

            if (winner == delayTask)
            {
                // Time is up — cancel work and observe losers.
                await cts.CancelAsync().ConfigureAwait(false);
                await ObserveAllAsync(primaryTask, backupTask).ConfigureAwait(false);
                throw new TimeoutException($"Timed out after {timeout} for {ticker}");
            }

            racers.Remove(winner);

            try
            {
                decimal price = await winner.ConfigureAwait(false);
                // Winner found — cancel losers.
                await cts.CancelAsync().ConfigureAwait(false);
                // Observe the loser to avoid UnobservedTaskException.
                await ObserveAllAsync(primaryTask, backupTask).ConfigureAwait(false);
                return new Quote(ticker, price, DateTimeOffset.UtcNow, Ok: true, Error: null);
            }
            catch (OperationCanceledException) when (cts.IsCancellationRequested)
            {
                // Winner cancelled by an earlier iteration — keep racing.
                continue;
            }
            catch
            {
                // One source failed — try the next.
                continue;
            }
        }

        throw new InvalidOperationException($"All sources failed for {ticker}");
    }

    private static async Task ObserveAllAsync(params Task[] tasks)
    {
        // Observe each task's final state so exceptions are not "unobserved".
        foreach (var t in tasks)
        {
            try { await t.ConfigureAwait(false); }
            catch { /* intentionally swallowed */ }
        }
    }
}

// ---------- Program.cs (top-level statements) ----------
/*
var primary = new FakeQuoteSource(TimeSpan.FromMilliseconds(150), 0.2, seed: 42);
var backup  = new FakeQuoteSource(TimeSpan.FromMilliseconds(120), 0.1, seed: 7);
var fetcher = new ParallelQuoteFetcher(primary);

using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10)); // top guard
string[] tickers = ["AAPL", "MSFT", "GOOG", "AMZN", "TSLA", "NVDA", "META", "NFLX", "INTC", "AMD"];

QuoteReport report = await fetcher.FetchAllAsync(tickers, cts.Token);
Console.WriteLine($"Report: success={report.SuccessCount}, failure={report.FailureCount}");
foreach (var q in report.Quotes)
    Console.WriteLine(q.Ok ? $"  {q.Ticker}: {q.Price}" : $"  {q.Ticker}: ERROR {q.Error}");

Quote fast = await FastQuoteService.FastestAsync("AAPL", primary, backup, TimeSpan.FromMilliseconds(500), cts.Token);
Console.WriteLine($"Fast quote: {fast.Ticker} = {fast.Price}");
*/
```

Walk-through. `FakeQuoteSource` emulates the network with `Task.Delay(baseLatency + jitter, ct)` — note the token is passed into `Task.Delay`, otherwise cancellation cannot interrupt the wait. `failureRate` drives the share of `HttpRequestException`, and each task in `FetchOneAsync` wraps the call in a `try/catch`, returning `Quote(Ok:false)` instead of throwing — this is exactly the "all-or-collect" pattern from the lesson, which prevents one failed ticker from masking the others through `AggregateException`. The `catch (OperationCanceledException) when (ct.IsCancellationRequested)` clause is essential: it distinguishes "we were cancelled by the outer token" from "the source internally threw `OperationCanceledException`" — without the `when`, you would swallow a genuine source error.

`FetchAllAsync` starts every task at once via `tickers.Select(...).ToArray()` — this matters: if you `await`-ed each in turn, there would be no parallelism. `Task.WhenAll(tasks)` returns the array in input-enumeration order, so `results[i]` always corresponds to `tickers[i]`, even when `tickers[0]` finishes last. The `success` count is computed from the already-assembled array through LINQ `Count(q => q.Ok)` — this is the lesson's functional aggregation, thread-safe with no `lock`. There is no shared `List<Quote>` with concurrent `Add`, and that is precisely why there is no race condition.

`FastQuoteService.FastestAsync` is the most delicate part. The shared `CancellationTokenSource.CreateLinkedTokenSource(outerCt)` links the outer token with an inner one, so a cancellation "from above" automatically cancels the race, while cancelling the losers internally does not touch the outer token. `WhenAny(racers.Append(delayTask))` sets up a race between both sources and the timer. If `delayTask` wins, time is up: we call `cts.CancelAsync()` to cancel both sources, then `ObserveAllAsync` fixes their terminal state (important: without observation, an `UnobservedTaskException` may surface later), and we throw `TimeoutException`. If one of the sources wins, we read its result; on success we cancel the loser through the same `cts.CancelAsync()` and observe everyone through `ObserveAllAsync`. The `catch (OperationCanceledException) when (cts.IsCancellationRequested)` handles the rare case where this iteration's winner was already cancelled by an earlier iteration's winner — we simply keep racing. Finally, the top guard in `Program.cs` via `new CancellationTokenSource(TimeSpan.FromSeconds(10))` is the equivalent of `CancelAfter`, recommended in the lesson's best practices as a cleaner alternative to a hand-rolled `WhenAny(work, Task.Delay)`.

#### Going deeper (bonus)

1. Add an overload of `FetchAllAsync` that limits concurrency through a `SemaphoreSlim` (say, no more than 4 simultaneous requests), and compare run time at 100 tickers against the unbounded version.
2. Implement the timeout variant via `CancellationTokenSource.CreateLinkedTokenSource(outerCt)` + `linked.CancelAfter(timeout)` instead of a hand-rolled `WhenAny(work, Task.Delay)`, and compare readability and cancellation behavior.
3. Add full-exception observation through `task.Exception.InnerExceptions` in a separate demo method where the tasks actually throw (rather than returning a structured result), and log every one of them.
4. Turn `FakeQuoteSource` into `HttpQuoteSource` over a real `HttpClient` with Polly retry, and confirm that `WhenAll` correctly handles `HttpRequestException` and `TaskCanceledException` from Polly.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `PortfolioQuotes` собирается и запускается.
- [ ] (RU) Во всех `await` библиотечного кода есть `.ConfigureAwait(false)`.
- [ ] (RU) `CancellationToken` пробрасывается во все асинхронные методы.
- [ ] (RU) `FetchOneAsync` возвращает `Quote(Ok:false)` вместо бросания исключений.
- [ ] (RU) Порядок `WhenAll` совпадает с порядком входного списка тикеров.
- [ ] (RU) `FastQuoteService` отменяет проигравших и наблюдает `Task.Delay`.
- [ ] (RU) Нет `.Result`/`.Wait()`.
- [ ] (RU) Юнит-тесты `xUnit` проходят для `failureRate=0` и `failureRate=1`.
- [ ] (EN) The `PortfolioQuotes` project builds and runs.
- [ ] (EN) Every library-code `await` has `.ConfigureAwait(false)`.
- [ ] (EN) `CancellationToken` is threaded through every async method.
- [ ] (EN) `FetchOneAsync` returns `Quote(Ok:false)` instead of throwing.
- [ ] (EN) `WhenAll` order matches the input ticker list order.
- [ ] (EN) `FastQuoteService` cancels losers and observes `Task.Delay`.
- [ ] (EN) No `.Result`/`.Wait()` anywhere.
- [ ] (EN) `xUnit` tests pass for `failureRate=0` and `failureRate=1`.

#### Ресурсы / Resources
- [Microsoft Learn — Task.WhenAll](https://learn.microsoft.com/dotnet/api/system.threading.tasks.task.whenall)
- [Microsoft Learn — Task.WhenAny](https://learn.microsoft.com/dotnet/api/system.threading.tasks.task.whenany)
- [Microsoft Learn — CancellationTokenSource.CancelAfter](https://learn.microsoft.com/dotnet/api/system.threading.cancellationtokensource.cancelafter)
- [Microsoft Learn — ConfigureAwait(false)](https://learn.microsoft.com/dotnet/api/system.threading.tasks.task.configureawait)
- [Stephen Toub — Async FAQ: Should I use ConfigureAwait(false)?](https://devblogs.microsoft.com/dotnet/configureawait-faq/)
