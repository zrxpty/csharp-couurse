---
[← К уроку M09-L01](lesson-M09-L01-async-intro.md) | [⬆ К модулю M09](../README.md) | [Следующее ДЗ →](homework-M09-L02-task-basics.md)
---

### Домашнее задание M09-L01: Зачем async, потоки vs async I/O / Homework M09-L01: Why async, threads vs async I/O

**Урок / Lesson:** M09-L01
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) На собственном бенчмарке прочувствовать разницу между блокировкой потоков и настоящим async I/O, научиться применять `CancellationToken`, `ConfigureAwait(false)`, `SemaphoreSlim` и `Task.WhenAll`, а также избегать классических ловушек (`async void`, `lock`+`await`, `Task.Run(() => Thread.Sleep(...))`, `.Result`). (EN) Feel on your own benchmark the difference between blocking threads and genuine async I/O, learn to apply `CancellationToken`, `ConfigureAwait(false)`, `SemaphoreSlim` and `Task.WhenAll`, and to avoid the classic pitfalls (`async void`, `lock`+`await`, `Task.Run(() => Thread.Sleep(...))`, `.Result`).

#### Связь с уроком / Connection to the lesson
(RU) Урок объяснил, почему поток — дорогой ресурс (~1 МБ стека), как async I/O через IOCP/epoll/kqueue освобождает поток, и чем асинхронность отличается от многопоточности. В ДЗ вы построите лабораторию, которая эмпирически измеряет эту разницу и закрепляет правильные инструменты: `CancellationToken`, `SemaphoreSlim`, `ConfigureAwait(false)`, `Task.WhenAll`, `Channel<T>`.
(EN) The lesson explained why a thread is an expensive resource (~1 MB of stack), how async I/O through IOCP/epoll/kqueue releases the thread, and how asynchrony differs from multithreading. In the homework you will build a lab that measures this difference empirically and cements the right tools: `CancellationToken`, `SemaphoreSlim`, `ConfigureAwait(false)`, `Task.WhenAll`, `Channel<T>`.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы — инженер небольшой команды, которая готовит к запуску сервис-агрегатор погоды. Прототип должен опросить десятки «поставщиков» (пока — имитация сетевых вызовов через `Task.Delay`), собрать результаты и показать пользователю сводку. Команда ещё не решила, писать ли код «по-старому» — блокируя поток на каждом вызове через `Thread.Sleep` — или сразу вкладываться в `async/await`. Руководитель просит не верить на слово, а **измерить**: построить мини-лабораторию, которая прогоняет один и тот же сценарий тремя способами и показывает, во что обходится каждый подход с точки зрения времени и пула потоков.

Эта лаборатория — не учебная абстракция ради абстракции. Она моделирует ровно ту ситуацию, которую урок описал аналогией с рестораном: «синхронный официант стоит у плиты», «асинхронный официант передаёт билет и идёт дальше». Вы увидите, что `Task.Run(() => Thread.Sleep(...))` не даёт масштабируемости — он лишь перекладывает блокировку на другой поток пула, — а настоящий async I/O через `Task.Delay` (или реальный `HttpClient.GetAsync`) реально освобождает поток и позволяет одной горстке потоков обслуживать сотни одновременных операций. Заодно вы потренируете дисциплину, которой учит урок: кооперативная отмена через `CancellationToken`, ограничение конкурентности через `SemaphoreSlim`, `ConfigureAwait(false)` в библиотечном коде и аккуратная агрегация ошибок в `Task.WhenAll`. Всё это — ровно те практики, без которых асинхронный код в проде превращается в дедлоки, истощение пула и «зависшие» запросы под нагрузкой.

#### Что нужно сделать (пошагово)

1. **Создайте проект.** В каталоге `M09-L01-hw` выполните команду `dotnet new console -n M09L01.Homework -o M09-L01-hw --framework net8.0`. Убедитесь, что в `M09-L01-hw/M09L01.Homework.csproj` указано `<TargetFramework>net8.0</TargetFramework>` и `<LangVersion>latest</LangVersion>`. Откройте `Program.cs` и удалите шаблонный `Console.WriteLine("Hello, World!")`.

2. **Определите модель «поставщика».** Создайте файл `Providers/IWeatherProvider.cs` с интерфейсом `IWeatherProvider { string Name { get; } Task<Reading> ReadAsync(CancellationToken ct); }`, где `Reading` — `readonly record struct Reading(string City, double Celsius, TimeSpan Latency)`. Это будет «библиотечный» слой, где уместен `ConfigureAwait(false)`.

3. **Реализуйте три провайдера-имитатора** в `Providers/`:
   - `FakeAsyncProvider` — настоящий async I/O через `await Task.Delay(latency, ct).ConfigureAwait(false)`; это эталон «правильно».
   - `BlockingSyncProvider` — `Thread.Sleep(latency)` внутри `ReadAsync`, обёрнутый в `Task.Run`, чтобы не блокировать вызывающий поток, но **имитируя плохой подход** «переложить блокировку на пул».
   - `BlockingDirectProvider` — `Thread.Sleep(latency)` прямо в `ReadAsync` без `Task.Run` (худший случай: блокирует тот поток, который зовёт).

   Каждый провайдер пишет в `Console` или в лог `Name` и `Latency`, чтобы было видно порядок выполнения.

4. **Напишите движок `Aggregator`** в `Aggregator.cs`: метод `Task<Summary> AggregateAsync(IReadOnlyList<IWeatherProvider> providers, int maxConcurrency, CancellationToken ct)`, который запускает все провайдеров через `Task.WhenAll`, но ограничивает одновременное число через `SemaphoreSlim(initialCount: maxConcurrency, maxCount: maxConcurrency)`. Внутри — `await _gate.WaitAsync(ct).ConfigureAwait(false)`, `try { ... } finally { _gate.Release(); }`. Используйте `ConfigureAwait(false)` после **каждого** `await` (это библиотечный код). `Summary` — `record Summary(TimeSpan Elapsed, int Successes, int Failures, Reading[] Readings)`.

5. **Реализуйте кооперативную отмену.** В `Program.cs` создайте `CancellationTokenSource` с таймером `CancelAfter(TimeSpan.FromSeconds(5))` и второй — для ручной отмены по `Console.CancelKeyPress`. Свяжите их через `CancellationTokenSource.CreateLinkedTokenSource`. Передавайте итоговый токен во все `await` и в `ct.ThrowIfCancellationRequested()` в циклах.

6. **Постройте бенчмарк** в `Benchmark.cs`: метод `RunAsync` прогоняет одного и того же набора из 50 провайдеров с задержкой 50 мс каждый трижды — через `FakeAsyncProvider`, `BlockingSyncProvider`, `BlockingDirectProvider` — при `maxConcurrency = 50` и при `maxConcurrency = 4`. Измеряйте время через `Stopwatch`. Вывод должен выглядеть так:

   ```
   FakeAsync   concurrency=50  ->   ~55 ms   (Successes: 50)
   FakeAsync   concurrency=4   ->  ~650 ms   (Successes: 50)
   Blocking    concurrency=50  ->  ~650 ms   (Successes: 50)   # пул не резиновый
   Blocking    concurrency=4   ->  ~650 ms   (Successes: 50)
   Direct      concurrency=50  ->  ~2500 ms  (Successes: 50)   # последовательная блокировка
   Cancelled after 5s           ->  OK
   ```

   Числа могут отличаться — главное, чтобы `FakeAsync concurrency=50` был на порядок быстрее остальных и чтобы `Blocking` при высокой конкурентности не давал линейного ускорения.

7. **Добавьте канал-превью.** В `ChannelPreview.cs` реализуйте `Channel.CreateBounded<Reading>(16)` с одним производителем (вы читаете из `Aggregator` и пишете в канал) и одним потребителем (`await foreach (var r in channel.Reader.ReadAllAsync(ct))`), который печатает сводку. После завершения вызывайте `channel.Writer.Complete()`. Это закрепляет preview `Channel<T>` из урока (подробно — в M11-L07).

8. **Проверьте сборку и запуск:** `dotnet build M09-L01-hw` (0 warnings, 0 errors), затем `dotnet run --project M09-L01-hw`. Запустите с отменой по `Ctrl+C` и убедитесь, что процесс корректно завершается за <1 c, а не «зависает».

9. **Добавьте модульные тесты** (опционально, но рекомендуется): `dotnet new xunit -n M09L01.Homework.Tests -o M09-L01-hw.Tests` и тест, что `AggregateAsync` с 10 провайдерами `FakeAsyncProvider` по 20 мс при `maxConcurrency=10` укладывается в 100 мс и возвращает 10 успехов.

#### Требования к решению

- Целевой фреймворк — **.NET 8**, язык **C# 12**. Разрешены и приветствуются top-level statements в `Program.cs`, `record`/`record struct`, collection expressions (`Reading[] readings = [.. source];`), raw string literals для многострочного вывода, pattern matching (`is { Successes: var s and > 0 }`), `using`-декларации.
- Все `async`-методы возвращают `Task` или `Task<T>`, **никакого `async void`**. Если нужен обработчик события `Console.CancelKeyPress` — оборачивайте тело в `try/catch` с логированием и не пробрасывайте исключения наружу.
- После **каждого** `await` в библиотечном слое (`Aggregator`, `IWeatherProvider`, `ChannelPreview`) стоит `.ConfigureAwait(false)`. В `Program.cs` (точка входа, `SynchronizationContext` равен `null`) `ConfigureAwait` можно опустить, но это должно быть осознанно.
- `SemaphoreSlim` используется для ограничения конкурентности: `WaitAsync` в `try`, `Release` — строго в `finally`. **Запрещено** `lock (obj) { await ... }`.
- `CancellationToken` пробрасывается во все `await`-операции и в `Task.Run`/`Task.Delay`/`Channel.Writer.WriteAsync`. В CPU-циклах (если появятся) — `ct.ThrowIfCancellationRequested()`.
- Имитация I/O — только через `Task.Delay(latency, ct)`. **Запрещено** использовать `Task.Run(() => Thread.Sleep(latency))` как «асинхронную» операцию в финальном решении; он допустим **только** в `BlockingSyncProvider`, чтобы показать антипаттерн.
- Код компилируется с `TreatWarningsAsErrors=true` (добавьте в `.csproj`), без предупреждений анализатора `CA2007` (не передавайте `CancellationToken` через `Task`-параметры и т. п.).
- Вывод бенчмарка читаемый: каждая строка содержит стратегию, конкурентность, время, число успехов/провалов.

#### Тонкости и подводные камни

- **`async` не создаёт потоки.** Урок подчёркивает: `async/await` лишь сигнализирует «тут будет ожидание, поток можно отдать». Не ждите, что `BlockingDirectProvider.ReadAsync` с `Thread.Sleep` вдруг станет «масштабируемым» от того, что метод помечен `async` — он блокирует текущий поток так же, как синхронный код. Бенчмарк должен это показать.
- **`Task.Run` не спасает от блокировки.** `BlockingSyncProvider` перекладывает `Thread.Sleep` на поток пула, но сам поток при этом занят и недоступен для других задач. При `maxConcurrency=50` вы упрётесь в размер пула (по умолчанию ≈ Environment.ProcessorCount на старте, растёт медленно) и увидите, что никакого 50× ускорения нет — это ключевой вывод урока.
- **`ConfigureAwait(false)` в библиотеках.** В `Aggregator` и `IWeatherProvider` контекст вызывающего неизвестен — возможно, это UI-поток. Если не ставить `ConfigureAwait(false)`, continuation попытается вернуться в захваченный контекст, что в UI даёт дедлоки и лишний маршалинг. В точке входа (`Program`) контекста нет, поэтому там `ConfigureAwait` опционален.
- **`CancellationToken` в каждое `await`.** Если вы передадите токен в `Aggregator`, но забудете прокинуть его в `Task.Delay` — отмена по таймеру сработает только после окончания задержки, а не «посередине». Урок прямо требует: `ct` идёт в каждую `await`-операцию и в `ThrowIfCancellationRequested()` в циклах.
- **`Task.WhenAll` и первая ошибка.** Если один провайдер бросает исключение, `Task.WhenAll` дожидается остальных и кидает `AggregateException` (точнее — первое исключение). Чтобы не потерять остальные результаты, оборачивайте каждый вызов в `try/catch` внутри `Aggregator` и собирайте `Reading` с пометкой `Failed`, либо используйте pattern `when (ct.IsCancellationRequested)` для区分ения отмены от настоящей ошибки.
- **`SemaphoreSlim` — это не `lock`.** Обычный `lock` нельзя использовать с `await` (мьютекс захватывается одним потоком, освобождается другим). `SemaphoreSlim.WaitAsync` — асинхронный, не блокирует поток, и `Release` должен идти строго в `finally`, иначе при исключении семафор «утечёт» и последующие вызовы зависнут на `WaitAsync`.
- **`async void` и события.** Обработчик `Console.CancelKeyPress` технически требует `void`, но делайте его `async void` только если действительно нужно `await` внутри; иначе — синхронный `void` с `cts.Cancel()`. Любой `async void` обязан иметь `try/catch` по всему телу — исключение наружу не выйдет.
- **`Channel<T>` и `Complete`.** Если забыть `channel.Writer.Complete()`, потребительский `await foreach` никогда не завершится — задача зависнет. С cancellation это маскируется (токен снимет ожидание), но в «мирном» сценарии без отмены процесс не выйдет.

#### Критерии приёмки

- [ ] Проект `M09-L01-hw` собирается под .NET 8 / C# 12 без ошибок и предупреждений (`TreatWarningsAsErrors=true`).
- [ ] В решении нет ни одного `async void` (кроме, возможно, обработчика события с `try/catch`).
- [ ] В библиотечном слое после каждого `await` стоит `ConfigureAwait(false)`.
- [ ] Нет `lock (obj) { await ... }`; конкурентность ограничена `SemaphoreSlim` с `Release` в `finally`.
- [ ] `CancellationToken` передаётся в каждый `await` и в `Task.Delay`/`Task.Run`/`Channel`-операции.
- [ ] Реализованы три стратегии: `FakeAsync` (`Task.Delay`), `BlockingSync` (`Task.Run`+`Thread.Sleep`), `BlockingDirect` (`Thread.Sleep` без `Task.Run`).
- [ ] Бенчмарк выводит время для каждой стратегии при `maxConcurrency=50` и `maxConcurrency=4`.
- [ ] `FakeAsync concurrency=50` показывает время ~latency, а не ~latency×N (то есть реально конкурентно).
- [ ] `BlockingSync concurrency=50` НЕ показывает 50× ускорения — демонстрирует ограничение пула.
- [ ] Отмена по `CancelAfter(5s)` и по `Ctrl+C` завершает процесс за <1 c, без «зависших» задач.
- [ ] `Channel<T>`-превью работает: производитель пишет, потребитель читает через `await foreach`, `Writer.Complete()` вызывается.
- [ ] Исключения отдельных провайдеров не роняют весь `WhenAll` — агрегируются или помечаются как `Failed`.
- [ ] Тесты (если добавлены) зелёные: `dotnet test` проходит.
- [ ] Код использует возможности C# 12: top-level statements, `record struct`, collection expressions или pattern matching.
- [ ] В README или в комментариях — короткое объяснение наблюдаемых чисел со ссылкой на аналогию «официант/кухня» из урока.

#### Подсказки (без прямого ответа)

- Чтобы увидеть «истощение пула», запустите `BlockingSync` с большой конкурентностью и заметите, что время растёт нелинейно — пул потоков в .NET стартует маленьким и наращивается с задержкой (~1 поток/0.5 c). Подумайте, почему урок называет это «ложным ощущением масштабируемости».
- `SemaphoreSlim` можно создать один раз в конструкторе `Aggregator` и переиспользовать — но тогда не забывайте, что `Release` должен соответствовать каждому `WaitAsync`.
- Для `Channel.CreateBounded<T>` подберите capacity так, чтобы производитель иногда «подождал» потребителя — это иллюстрирует backpressure из M11-L07.
- Не путайте `Task.Delay` (имитация I/O, поток свободен) и `Thread.Sleep` (блокировка, поток занят) — это краеугольный пример урока.
- Если `WhenAll` падает с первым исключением — оберните каждый вызов провайдера в `try/catch` внутри `AggregateAsync` и верните `Reading.Failed` вместо проброса.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — эталонное решение домашнего задания M09-L01
// Reference solution for homework M09-L01
// Темы урока: async I/O vs блокировка потоков, CancellationToken,
// ConfigureAwait(false), SemaphoreSlim, Task.WhenAll, Channel<T>.
// Lesson topics: async I/O vs thread blocking, CancellationToken,
// ConfigureAwait(false), SemaphoreSlim, Task.WhenAll, Channel<T>.

using System.Diagnostics;
using System.Threading;
using System.Threading.Channels;

namespace M09L01.Homework;

// Модель показания «поставщика погоды» / Weather provider reading model
public readonly record struct Reading(string City, double Celsius, TimeSpan Latency, bool Ok);

// Итог агрегации / Aggregation summary
public sealed record Summary(TimeSpan Elapsed, int Successes, int Failures, Reading[] Readings);

// ============================================================
// Интерфейс «поставщика» — библиотечный слой / Provider interface — library layer
// ============================================================
public interface IWeatherProvider
{
    string Name { get; }
    Task<Reading> ReadAsync(CancellationToken ct);
}

// ============================================================
// 1. Эталон: НАСТОЯЩИЙ async I/O через Task.Delay — поток свободен
//    Reference: REAL async I/O via Task.Delay — the thread is free
// ============================================================
public sealed class FakeAsyncProvider(string city, double celsius, int latencyMs)
    : IWeatherProvider
{
    public string Name => $"fake-async/{city}";
    public async Task<Reading> ReadAsync(CancellationToken ct)
    {
        var sw = Stopwatch.StartNew();
        // Task.Delay с токеном — поток освобождается, отмена сработает «посередине»
        // Task.Delay with a token — the thread is freed, cancellation fires mid-wait
        await Task.Delay(latencyMs, ct).ConfigureAwait(false);
        return new Reading(city, celsius, sw.Elapsed, Ok: true);
    }
}

// ============================================================
// 2. Антипаттерн: Task.Run + Thread.Sleep — блокирует ДРУГОЙ поток пула
//    Anti-pattern: Task.Run + Thread.Sleep — blocks ANOTHER pool thread
// ============================================================
public sealed class BlockingSyncProvider(string city, double celsius, int latencyMs)
    : IWeatherProvider
{
    public string Name => $"blocking-sync/{city}";
    public Task<Reading> ReadAsync(CancellationToken ct) =>
        Task.Run(() =>
        {
            var sw = Stopwatch.StartNew();
            Thread.Sleep(latencyMs); // поток пула занят полностью / pool thread is fully busy
            ct.ThrowIfCancellationRequested(); // отмена сработает только ПОСЛЕ сна
            return new Reading(city, celsius, sw.Elapsed, Ok: true);
        }, ct);
}

// ============================================================
// 3. Худший случай: Thread.Sleep прямо в async-методе — блокирует вызывающий поток
//    Worst case: Thread.Sleep directly inside an async method — blocks the caller
// ============================================================
public sealed class BlockingDirectProvider(string city, double celsius, int latencyMs)
    : IWeatherProvider
{
    public string Name => $"blocking-direct/{city}";
    public async Task<Reading> ReadAsync(CancellationToken ct)
    {
        var sw = Stopwatch.StartNew();
        Thread.Sleep(latencyMs); // ⚠ блокирует ТЕКУЩИЙ поток, «async» ничего не меняет
        ct.ThrowIfCancellationRequested();
        return new Reading(city, celsius, sw.Elapsed, Ok: true);
    }
}

// ============================================================
// Агрегатор: ограничение конкурентности через SemaphoreSlim + ConfigureAwait(false)
// Aggregator: concurrency limit via SemaphoreSlim + ConfigureAwait(false)
// ============================================================
public sealed class Aggregator(int maxConcurrency)
{
    private readonly SemaphoreSlim _gate = new(maxConcurrency, maxConcurrency);

    public async Task<Summary> AggregateAsync(
        IReadOnlyList<IWeatherProvider> providers, CancellationToken ct)
    {
        var sw = Stopwatch.StartNew();
        // Запускаем все задачи сразу; лимит enforced внутри SemaphoreSlim
        // Launch every task at once; the limit is enforced inside SemaphoreSlim
        var tasks = providers.Select(p => CallOneAsync(p, ct)).ToArray();
        try
        {
            // WhenAll дожидается всех; исключения мы «гасим» внутри CallOneAsync
            // WhenAll awaits everything; exceptions are swallowed inside CallOneAsync
            var readings = await Task.WhenAll(tasks).ConfigureAwait(false);
            sw.Stop();
            // collection expression C# 12 / C# 12 collection expression
            Reading[] all = [.. readings];
            return new Summary(sw.Elapsed,
                all.Count(r => r.Ok), all.Count(r => !r.Ok), all);
        }
        catch (OperationCanceledException) when (ct.IsCancellationRequested)
        {
            sw.Stop();
            // Ожидаемая отмена — НЕ ошибка / Expected cancellation — NOT an error
            return new Summary(sw.Elapsed, 0, providers.Count, Array.Empty<Reading>());
        }
    }

    private async Task<Reading> CallOneAsync(IWeatherProvider p, CancellationToken ct)
    {
        // Асинхронно ждём семафор — НЕ блокируем поток!
        // Asynchronously wait for the semaphore — does NOT block the thread!
        await _gate.WaitAsync(ct).ConfigureAwait(false);
        try
        {
            // Гасим исключение отдельного провайдера, чтобы WhenAll не ронял всё
            // Swallow a single provider's failure so WhenAll does not tear everything down
            return await p.ReadAsync(ct).ConfigureAwait(false);
        }
        catch (OperationCanceledException) when (ct.IsCancellationRequested)
        {
            throw; // отмена пробрасывается наверх / cancellation propagates upward
        }
        catch (Exception ex)
        {
            Console.WriteLine($"[FAIL] {p.Name}: {ex.Message}");
            return new Reading(p.Name, 0, TimeSpan.Zero, Ok: false);
        }
        finally
        {
            _gate.Release(); // всегда освобождаем в finally / always release in finally
        }
    }
}

// ============================================================
// Бенчмарк: прогон трёх стратегий при двух уровнях конкурентности
// Benchmark: three strategies × two concurrency levels
// ============================================================
public static class Benchmark
{
    public static async Task RunAsync(IReadOnlyList<IWeatherProvider> providers,
        int[] concurrencies, CancellationToken ct)
    {
        foreach (var c in concurrencies)
        {
            var agg = new Aggregator(c);
            var s = await agg.AggregateAsync(providers, ct).ConfigureAwait(false);
            // pattern matching C# 12 / C# 12 pattern matching
            var tag = s is { Successes: var ok and > 0 } ? "OK" : "EMPTY";
            Console.WriteLine(
                $"concurrency={c,2}  elapsed={s.Elapsed.TotalMilliseconds,6:F0} ms  " +
                $"ok={s.Successes,3}  fail={s.Failures,3}  [{tag}]");
        }
    }
}

// ============================================================
// Channel<T> preview — producer/consumer (подробно в M11-L07)
// ============================================================
public static class ChannelPreview
{
    public static async Task RunAsync(IAsyncEnumerable<Reading> source, CancellationToken ct)
    {
        // Ограниченный канал — backpressure / Bounded channel — backpressure
        var channel = Channel.CreateBounded<Reading>(16);

        var producer = Task.Run(async () =>
        {
            try
            {
                await foreach (var r in source.WithCancellation(ct).ConfigureAwait(false))
                    await channel.Writer.WriteAsync(r, ct).ConfigureAwait(false);
            }
            finally
            {
                channel.Writer.Complete(); // сигнализируем конец / signal the end
            }
        }, ct);

        var consumer = Task.Run(async () =>
        {
            await foreach (var r in channel.Reader.ReadAllAsync(ct).ConfigureAwait(false))
                Console.WriteLine($"  consumed: {r.City} {r.Celsius}°C ({r.Latency.TotalMilliseconds:F0} ms)");
        }, ct);

        await Task.WhenAll(producer, consumer).ConfigureAwait(false);
    }
}
```

Разбор по строкам. Файл начинается с `readonly record struct Reading` — это компактная value-тип модель на C# 12; поля `Ok` и `Latency` позволяют бенчмарку отличать успешные показания от провальных. `IWeatherProvider` — «библиотечный» интерфейс, поэтому каждая реализация после `await` ставит `ConfigureAwait(false)` (урок: в библиотеках не захватываем контекст вызывающего). `FakeAsyncProvider` использует `await Task.Delay(latencyMs, ct).ConfigureAwait(false)` — это и есть «билет на кухню»: поток возвращается в пул на время задержки, а `ct` в `Task.Delay` гарантирует, что отмена сработает немедленно, не дожидаясь конца задержки (урок: «передавайте `CancellationToken` во все `await`-операции»). `BlockingSyncProvider` намеренно демонстрирует антипаттерн: `Task.Run(() => Thread.Sleep(...))` перекладывает блокировку на другой поток пула и не даёт масштабируемости — на бенчмарке вы увидите, что при `maxConcurrency=50` время не уменьшается в 50 раз, потому что пул мал и наращивается медленно (урок: «блокировка потоков пула через `Task.Run(() => Thread.Sleep(...))` → истощение пула под нагрузкой»). `BlockingDirectProvider` — худший случай: `Thread.Sleep` внутри `async`-метода блокирует текущий поток, и `async` тут ничего не меняет (урок: «`async` не создаёт потоки автоматически»).

`Aggregator` создаёт `SemaphoreSlim(initialCount: maxConcurrency, maxCount: maxConcurrency)` в конструкторе — это «async lock»: `WaitAsync` асинхронно ждёт слот, не блокируя поток, а `Release` стоит в `finally`, поэтому при исключении слот не теряется (урок: «используйте `SemaphoreSlim.WaitAsync` для асинхронных критических секций вместо `lock`»). Внутри `CallOneAsync` каждый вызов провайдера обёрнут в `try/catch`: одиночное исключение возвращает `Reading(Ok: false)` вместо проброса, чтобы `Task.WhenAll` не ронял всю агрегацию (урок: «`Task.WhenAll` без обработки первой упавшей задачи → остальные продолжают работать вслепую»). Особый случай — `catch (OperationCanceledException) when (ct.IsCancellationRequested)`: отмена пробрасывается наверх, и `AggregateAsync` ловит её на верхнем уровне, возвращая пустой `Summary` (урок: «ожидаемая отмена, НЕ ошибка»). В `AggregateAsync` коллекция собрана через collection expression `Reading[] all = [.. readings];` — это новая возможность C# 12. `Benchmark.RunAsync` использует pattern matching `s is { Successes: var ok and > 0 }` — ещё одна фича C# 12. `ChannelPreview` — preview producer/consumer: ограниченный `Channel.CreateBounded<Reading>(16)` создаёт backpressure (производитель ждёт, если потребитель не успевает), `await foreach` читает через `ReadAllAsync(ct)`, а `Writer.Complete()` в `finally` гарантирует, что потребитель завершится даже при ошибке (урок: «`Channel<T>` — потокобезопасный примитив для producer/consumer»). Все `await` в библиотечном слое сопровождаются `ConfigureAwait(false)`, что прямо соответствует best practice из урока. Никакого `async void` нет; `CancellationToken` пробрасывается в каждый `await`, в `Task.Run` и в `Channel.Writer.WriteAsync`. В точке входа (`Program.cs`, ниже не показан) `SynchronizationContext` равен `null`, поэтому `ConfigureAwait` опущен — это осознанное решение, разрешённое уроком.

#### Задания на углубление (бонус)

1. **Реальный HTTP вместо имитации.** Замените `FakeAsyncProvider` на `HttpWeatherProvider`, который делает настоящий `HttpClient.GetAsync(url, ct)` к публичному API (например, `https://wttr.in/{city}?format=j1`). Сравните тайминги и убедитесь, что рост latency сети делает разрыв между async и blocking ещё драматичнее. Подумайте, почему `HttpClient` нужно переиспользовать (singleton), а не создавать на каждый запрос.
2. **Истощение пула под нагрузкой.** Добавьте сценарий, в котором 1000 `BlockingSyncProvider` одновременно стартуют при `maxConcurrency=1000`. Измерьте, как долго пул наращивается до нужного размера (`ThreadPool.GetAvailableThreads`), и объясните, почему урок называет это «истощением пула». Сравните с тем же числом `FakeAsyncProvider`.
3. **Дедлок в UI-контексте (теоретический эксперимент).** Напишите короткий комментарий-эссе (или мини-тест на WPF, если есть опыт): почему вызов `.Result` на `FakeAsyncProvider.ReadAsync` в UI-потоке приводит к дедлоку, а в `Program.Main` — нет. Сошлитесь на `SynchronizationContext` и на рекомендацию урока «распространяйте `async` до точки входа».
4. **Канал с несколькими потребителями.** Расширьте `ChannelPreview`: запустите 3 параллельных потребителя `ReadAllAsync` и убедитесь, что каждое показание обрабатывается ровно одним потребителем. Измерьте, как `Bounded` capacity влияет на память и backpressure.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are an engineer in a small team preparing to launch a weather aggregator service. The prototype must query dozens of «providers» (for now — simulated network calls through `Task.Delay`), collect their results, and show the user a summary. The team has not yet decided whether to write the code «the old way» — blocking the thread on every call through `Thread.Sleep` — or to invest in `async/await` from the start. Your lead asks you not to take it on faith but to **measure**: build a small lab that runs the same scenario three different ways and shows what each approach costs in terms of time and thread-pool pressure.

This lab is not a pedagogical abstraction for its own sake. It models exactly the situation the lesson described with the restaurant analogy: «the synchronous waiter stands at the stove», «the asynchronous waiter hands the ticket and moves on». You will see that `Task.Run(() => Thread.Sleep(...))` brings no scalability — it merely shifts the blockage onto another pool thread — while genuine async I/O through `Task.Delay` (or a real `HttpClient.GetAsync`) actually frees the thread and lets a handful of threads service hundreds of concurrent operations. Along the way you will train the discipline the lesson teaches: cooperative cancellation through `CancellationToken`, concurrency limiting through `SemaphoreSlim`, `ConfigureAwait(false)` in library code, and careful error aggregation in `Task.WhenAll`. These are precisely the practices without which asynchronous code in production collapses into deadlocks, pool starvation, and «stuck» requests under load.

#### What to do step by step

1. **Create the project.** In the `M09-L01-hw` directory run `dotnet new console -n M09L01.Homework -o M09-L01-hw --framework net8.0`. Verify that `M09-L01-hw/M09L01.Homework.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and `<LangVersion>latest</LangVersion>`. Open `Program.cs` and remove the template `Console.WriteLine("Hello, World!")`.

2. **Define the «provider» model.** Create `Providers/IWeatherProvider.cs` with `interface IWeatherProvider { string Name { get; } Task<Reading> ReadAsync(CancellationToken ct); }`, where `Reading` is `readonly record struct Reading(string City, double Celsius, TimeSpan Latency)`. This is the «library» layer where `ConfigureAwait(false)` is appropriate.

3. **Implement three simulated providers** in `Providers/`:
   - `FakeAsyncProvider` — genuine async I/O via `await Task.Delay(latency, ct).ConfigureAwait(false)`; this is the «correct» reference.
   - `BlockingSyncProvider` — `Thread.Sleep(latency)` inside `ReadAsync`, wrapped in `Task.Run` so the caller is not blocked, but **emulating the bad approach** of «shifting the block onto the pool».
   - `BlockingDirectProvider` — `Thread.Sleep(latency)` directly in `ReadAsync` without `Task.Run` (worst case: blocks the calling thread itself).

   Each provider writes its `Name` and `Latency` to `Console` (or a logger) so the execution order is visible.

4. **Write the `Aggregator` engine** in `Aggregator.cs`: a method `Task<Summary> AggregateAsync(IReadOnlyList<IWeatherProvider> providers, int maxConcurrency, CancellationToken ct)` that launches all providers through `Task.WhenAll` but caps concurrency with `SemaphoreSlim(initialCount: maxConcurrency, maxCount: maxConcurrency)`. Inside use `await _gate.WaitAsync(ct).ConfigureAwait(false)`, then `try { ... } finally { _gate.Release(); }`. Apply `ConfigureAwait(false)` after **every** `await` (this is library code). `Summary` is `record Summary(TimeSpan Elapsed, int Successes, int Failures, Reading[] Readings)`.

5. **Implement cooperative cancellation.** In `Program.cs` create a `CancellationTokenSource` with `CancelAfter(TimeSpan.FromSeconds(5))` and a second one for manual cancellation on `Console.CancelKeyPress`. Link them via `CancellationTokenSource.CreateLinkedTokenSource`. Pass the resulting token into every `await` and into `ct.ThrowIfCancellationRequested()` inside loops.

6. **Build the benchmark** in `Benchmark.cs`: a `RunAsync` method that runs the same set of 50 providers (50 ms latency each) three times — through `FakeAsyncProvider`, `BlockingSyncProvider`, `BlockingDirectProvider` — at `maxConcurrency = 50` and at `maxConcurrency = 4`. Measure time with `Stopwatch`. The output should look like:

   ```
   FakeAsync   concurrency=50  ->   ~55 ms   (Successes: 50)
   FakeAsync   concurrency=4   ->  ~650 ms   (Successes: 50)
   Blocking    concurrency=50  ->  ~650 ms   (Successes: 50)   # the pool is not elastic
   Blocking    concurrency=4   ->  ~650 ms   (Successes: 50)
   Direct      concurrency=50  ->  ~2500 ms  (Successes: 50)   # sequential blocking
   Cancelled after 5s           ->  OK
   ```

   Numbers may differ — the key points are that `FakeAsync concurrency=50` is an order of magnitude faster than the rest and that `Blocking` at high concurrency does not yield linear speedup.

7. **Add a channel preview.** In `ChannelPreview.cs` implement `Channel.CreateBounded<Reading>(16)` with one producer (you read from the `Aggregator` and write to the channel) and one consumer (`await foreach (var r in channel.Reader.ReadAllAsync(ct))`) that prints the summary. Call `channel.Writer.Complete()` when finished. This cements the `Channel<T>` preview from the lesson (in depth in M11-L07).

8. **Verify build and run:** `dotnet build M09-L01-hw` (0 warnings, 0 errors), then `dotnet run --project M09-L01-hw`. Trigger cancellation with `Ctrl+C` and confirm the process exits cleanly in <1 s rather than «hanging».

9. **Add unit tests** (optional but recommended): `dotnet new xunit -n M09L01.Homework.Tests -o M09-L01-hw.Tests`, and a test that `AggregateAsync` with 10 `FakeAsyncProvider`s of 20 ms each at `maxConcurrency=10` finishes within 100 ms and returns 10 successes.

#### Requirements

- Target framework is **.NET 8**, language **C# 12**. Top-level statements in `Program.cs`, `record`/`record struct`, collection expressions (`Reading[] readings = [.. source];`), raw string literals for multi-line output, and pattern matching (`is { Successes: var s and > 0 }`) are all welcome.
- Every `async` method returns `Task` or `Task<T>`; **no `async void`**. If you need a `Console.CancelKeyPress` handler, wrap its body in `try/catch` with logging and never let exceptions escape.
- After **every** `await` in the library layer (`Aggregator`, `IWeatherProvider`, `ChannelPreview`) there is `.ConfigureAwait(false)`. In `Program.cs` (entry point, `SynchronizationContext` is `null`) `ConfigureAwait` may be omitted, but that must be a conscious choice.
- `SemaphoreSlim` is used to limit concurrency: `WaitAsync` in `try`, `Release` strictly in `finally`. `lock (obj) { await ... }` is **forbidden**.
- `CancellationToken` is propagated to every `await` operation and to `Task.Run`/`Task.Delay`/`Channel.Writer.WriteAsync`. In CPU loops (if any) call `ct.ThrowIfCancellationRequested()`.
- I/O is simulated only through `Task.Delay(latency, ct)`. Using `Task.Run(() => Thread.Sleep(latency))` as an «async» operation in the final solution is **forbidden**; it is allowed **only** inside `BlockingSyncProvider`, to demonstrate the anti-pattern.
- The code compiles with `TreatWarningsAsErrors=true` (add it to `.csproj`), with no `CA2007`-style analyzer warnings.
- The benchmark output is readable: each line shows strategy, concurrency, time, and the success/failure counts.

#### Pitfalls

- **`async` does not create threads.** The lesson stresses that `async/await` merely signals «there will be a wait, the thread can be released». Do not expect `BlockingDirectProvider.ReadAsync` with `Thread.Sleep` to suddenly become «scalable» just because the method is marked `async` — it blocks the current thread exactly like synchronous code. The benchmark must show this.
- **`Task.Run` does not save you from blocking.** `BlockingSyncProvider` moves `Thread.Sleep` onto a pool thread, but that thread is then busy and unavailable for other tasks. At `maxConcurrency=50` you will hit the pool size (≈ `Environment.ProcessorCount` initially, growing slowly) and observe no 50× speedup — this is the key takeaway of the lesson.
- **`ConfigureAwait(false)` in libraries.** In `Aggregator` and `IWeatherProvider` the caller's context is unknown — it might be a UI thread. Without `ConfigureAwait(false)` the continuation tries to re-enter the captured context, which in UI causes deadlocks and needless marshalling. At the entry point (`Program`) there is no context, so `ConfigureAwait` is optional there.
- **`CancellationToken` into every `await`.** If you pass the token to `Aggregator` but forget to forward it to `Task.Delay`, the timed cancellation will only fire after the delay ends, not «mid-way». The lesson explicitly demands: `ct` goes into every `await` and into `ThrowIfCancellationRequested()` inside loops.
- **`Task.WhenAll` and the first failure.** If one provider throws, `Task.WhenAll` waits for the rest and rethrows the first exception. To avoid losing the other results, wrap each call in `try/catch` inside `Aggregator` and collect a `Reading` marked `Failed`, or use `when (ct.IsCancellationRequested)` to distinguish cancellation from a real error.
- **`SemaphoreSlim` is not `lock`.** Plain `lock` cannot be used with `await` (the mutex is acquired on one thread and released on another). `SemaphoreSlim.WaitAsync` is asynchronous, does not block a thread, and `Release` must go strictly in `finally`, otherwise on an exception the semaphore «leaks» and later calls stall on `WaitAsync`.
- **`async void` and events.** The `Console.CancelKeyPress` handler technically requires `void`, but make it `async void` only if you genuinely need an `await` inside; otherwise use a synchronous `void` with `cts.Cancel()`. Any `async void` must wrap its whole body in `try/catch` — exceptions will not escape.
- **`Channel<T>` and `Complete`.** If you forget `channel.Writer.Complete()`, the consumer's `await foreach` never terminates — the task hangs. With cancellation this is masked (the token lifts the wait), but in a peaceful scenario without cancellation the process will not exit.

#### Acceptance criteria

- [ ] The `M09-L01-hw` project builds under .NET 8 / C# 12 with no errors and no warnings (`TreatWarningsAsErrors=true`).
- [ ] There is no `async void` anywhere (except, optionally, an event handler with `try/catch`).
- [ ] In the library layer, `ConfigureAwait(false)` follows every `await`.
- [ ] There is no `lock (obj) { await ... }`; concurrency is capped with `SemaphoreSlim` whose `Release` sits in `finally`.
- [ ] `CancellationToken` is passed into every `await` and into `Task.Delay`/`Task.Run`/`Channel` operations.
- [ ] Three strategies are implemented: `FakeAsync` (`Task.Delay`), `BlockingSync` (`Task.Run`+`Thread.Sleep`), `BlockingDirect` (`Thread.Sleep` without `Task.Run`).
- [ ] The benchmark prints timing for each strategy at `maxConcurrency=50` and `maxConcurrency=4`.
- [ ] `FakeAsync concurrency=50` shows ~latency, not ~latency×N (i.e., it is genuinely concurrent).
- [ ] `BlockingSync concurrency=50` does NOT show 50× speedup — it exposes the pool limit.
- [ ] Cancellation via `CancelAfter(5s)` and via `Ctrl+C` ends the process in <1 s, with no «stuck» tasks.
- [ ] The `Channel<T>` preview works: producer writes, consumer reads via `await foreach`, `Writer.Complete()` is called.
- [ ] Exceptions from individual providers do not crash the whole `WhenAll` — they are aggregated or marked `Failed`.
- [ ] Tests (if added) are green: `dotnet test` passes.
- [ ] The code uses C# 12 features: top-level statements, `record struct`, collection expressions, or pattern matching.
- [ ] A README or comments briefly explain the observed numbers, referencing the «waiter/kitchen» analogy from the lesson.

#### Hints (no direct answer)

- To observe «pool starvation», run `BlockingSync` at high concurrency and notice the time grows non-linearly — the .NET thread pool starts small and grows with a delay (~1 thread/0.5 s). Think about why the lesson calls this «a false sense of scalability».
- You can create `SemaphoreSlim` once in the `Aggregator` constructor and reuse it — but remember that every `Release` must match a `WaitAsync`.
- For `Channel.CreateBounded<T>` pick a capacity so the producer occasionally «waits» for the consumer — this illustrates backpressure from M11-L07.
- Do not confuse `Task.Delay` (I/O simulation, thread is free) with `Thread.Sleep` (blocking, thread is busy) — this is the cornerstone example of the lesson.
- If `WhenAll` blows up on the first exception, wrap each provider call in `try/catch` inside `AggregateAsync` and return `Reading.Failed` instead of rethrowing.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — reference solution for homework M09-L01
// Lesson topics: async I/O vs thread blocking, CancellationToken,
// ConfigureAwait(false), SemaphoreSlim, Task.WhenAll, Channel<T>.

using System.Diagnostics;
using System.Threading;
using System.Threading.Channels;

namespace M09L01.Homework;

// Weather provider reading model
public readonly record struct Reading(string City, double Celsius, TimeSpan Latency, bool Ok);

// Aggregation summary
public sealed record Summary(TimeSpan Elapsed, int Successes, int Failures, Reading[] Readings);

// Provider interface — library layer
public interface IWeatherProvider
{
    string Name { get; }
    Task<Reading> ReadAsync(CancellationToken ct);
}

// 1. Reference: REAL async I/O via Task.Delay — the thread is free
public sealed class FakeAsyncProvider(string city, double celsius, int latencyMs)
    : IWeatherProvider
{
    public string Name => $"fake-async/{city}";
    public async Task<Reading> ReadAsync(CancellationToken ct)
    {
        var sw = Stopwatch.StartNew();
        // Task.Delay with a token — the thread is freed, cancellation fires mid-wait
        await Task.Delay(latencyMs, ct).ConfigureAwait(false);
        return new Reading(city, celsius, sw.Elapsed, Ok: true);
    }
}

// 2. Anti-pattern: Task.Run + Thread.Sleep — blocks ANOTHER pool thread
public sealed class BlockingSyncProvider(string city, double celsius, int latencyMs)
    : IWeatherProvider
{
    public string Name => $"blocking-sync/{city}";
    public Task<Reading> ReadAsync(CancellationToken ct) =>
        Task.Run(() =>
        {
            var sw = Stopwatch.StartNew();
            Thread.Sleep(latencyMs); // pool thread is fully busy
            ct.ThrowIfCancellationRequested(); // cancellation only fires AFTER the sleep
            return new Reading(city, celsius, sw.Elapsed, Ok: true);
        }, ct);
}

// 3. Worst case: Thread.Sleep directly inside an async method — blocks the caller
public sealed class BlockingDirectProvider(string city, double celsius, int latencyMs)
    : IWeatherProvider
{
    public string Name => $"blocking-direct/{city}";
    public async Task<Reading> ReadAsync(CancellationToken ct)
    {
        var sw = Stopwatch.StartNew();
        Thread.Sleep(latencyMs); // ⚠ blocks the CURRENT thread, "async" changes nothing
        ct.ThrowIfCancellationRequested();
        return new Reading(city, celsius, sw.Elapsed, Ok: true);
    }
}

// Aggregator: concurrency limit via SemaphoreSlim + ConfigureAwait(false)
public sealed class Aggregator(int maxConcurrency)
{
    private readonly SemaphoreSlim _gate = new(maxConcurrency, maxConcurrency);

    public async Task<Summary> AggregateAsync(
        IReadOnlyList<IWeatherProvider> providers, CancellationToken ct)
    {
        var sw = Stopwatch.StartNew();
        var tasks = providers.Select(p => CallOneAsync(p, ct)).ToArray();
        try
        {
            var readings = await Task.WhenAll(tasks).ConfigureAwait(false);
            sw.Stop();
            Reading[] all = [.. readings]; // C# 12 collection expression
            return new Summary(sw.Elapsed,
                all.Count(r => r.Ok), all.Count(r => !r.Ok), all);
        }
        catch (OperationCanceledException) when (ct.IsCancellationRequested)
        {
            sw.Stop();
            return new Summary(sw.Elapsed, 0, providers.Count, Array.Empty<Reading>());
        }
    }

    private async Task<Reading> CallOneAsync(IWeatherProvider p, CancellationToken ct)
    {
        await _gate.WaitAsync(ct).ConfigureAwait(false);
        try
        {
            return await p.ReadAsync(ct).ConfigureAwait(false);
        }
        catch (OperationCanceledException) when (ct.IsCancellationRequested)
        {
            throw;
        }
        catch (Exception ex)
        {
            Console.WriteLine($"[FAIL] {p.Name}: {ex.Message}");
            return new Reading(p.Name, 0, TimeSpan.Zero, Ok: false);
        }
        finally
        {
            _gate.Release(); // always release in finally
        }
    }
}

// Benchmark: three strategies × two concurrency levels
public static class Benchmark
{
    public static async Task RunAsync(IReadOnlyList<IWeatherProvider> providers,
        int[] concurrencies, CancellationToken ct)
    {
        foreach (var c in concurrencies)
        {
            var agg = new Aggregator(c);
            var s = await agg.AggregateAsync(providers, ct).ConfigureAwait(false);
            var tag = s is { Successes: var ok and > 0 } ? "OK" : "EMPTY"; // C# 12 pattern
            Console.WriteLine(
                $"concurrency={c,2}  elapsed={s.Elapsed.TotalMilliseconds,6:F0} ms  " +
                $"ok={s.Successes,3}  fail={s.Failures,3}  [{tag}]");
        }
    }
}

// Channel<T> preview — producer/consumer (in depth in M11-L07)
public static class ChannelPreview
{
    public static async Task RunAsync(IAsyncEnumerable<Reading> source, CancellationToken ct)
    {
        var channel = Channel.CreateBounded<Reading>(16); // backpressure

        var producer = Task.Run(async () =>
        {
            try
            {
                await foreach (var r in source.WithCancellation(ct).ConfigureAwait(false))
                    await channel.Writer.WriteAsync(r, ct).ConfigureAwait(false);
            }
            finally
            {
                channel.Writer.Complete();
            }
        }, ct);

        var consumer = Task.Run(async () =>
        {
            await foreach (var r in channel.Reader.ReadAllAsync(ct).ConfigureAwait(false))
                Console.WriteLine($"  consumed: {r.City} {r.Celsius}°C ({r.Latency.TotalMilliseconds:F0} ms)");
        }, ct);

        await Task.WhenAll(producer, consumer).ConfigureAwait(false);
    }
}
```

Line-by-line walk-through. The file opens with `readonly record struct Reading` — a compact value-type model in C# 12; the `Ok` and `Latency` fields let the benchmark distinguish successful readings from failed ones. `IWeatherProvider` is the «library» interface, so each implementation applies `ConfigureAwait(false)` after its `await` (lesson: in libraries we do not capture the caller's context). `FakeAsyncProvider` uses `await Task.Delay(latencyMs, ct).ConfigureAwait(false)` — this is exactly «the kitchen ticket»: the thread returns to the pool for the duration of the delay, and `ct` inside `Task.Delay` guarantees that cancellation fires immediately rather than waiting for the delay to end (lesson: «pass `CancellationToken` into every `await`»). `BlockingSyncProvider` intentionally demonstrates the anti-pattern: `Task.Run(() => Thread.Sleep(...))` shifts the block onto another pool thread and delivers no scalability — in the benchmark you will see that at `maxConcurrency=50` the time does not drop 50×, because the pool is small and grows slowly (lesson: «blocking pool threads via `Task.Run(() => Thread.Sleep(...))` → pool starvation under load»). `BlockingDirectProvider` is the worst case: `Thread.Sleep` inside an `async` method blocks the current thread, and `async` changes nothing here (lesson: «`async` does not create threads by itself»).

`Aggregator` builds a `SemaphoreSlim(initialCount: maxConcurrency, maxCount: maxConcurrency)` in its constructor — this is the «async lock»: `WaitAsync` waits for a slot asynchronously without blocking a thread, and `Release` sits in `finally`, so on an exception the slot is never lost (lesson: «use `SemaphoreSlim.WaitAsync` for async critical sections instead of `lock`»). Inside `CallOneAsync` each provider call is wrapped in `try/catch`: a single failure returns `Reading(Ok: false)` instead of propagating, so `Task.WhenAll` does not tear down the whole aggregation (lesson: «`Task.WhenAll` without handling the first failing task → the others keep running blind»). The special case `catch (OperationCanceledException) when (ct.IsCancellationRequested)` rethrows cancellation, and `AggregateAsync` catches it at the top level, returning an empty `Summary` (lesson: «expected cancellation, NOT an error»). In `AggregateAsync` the collection is built with a collection expression `Reading[] all = [.. readings];` — a new C# 12 feature. `Benchmark.RunAsync` uses pattern matching `s is { Successes: var ok and > 0 }` — another C# 12 feature. `ChannelPreview` is a producer/consumer preview: a bounded `Channel.CreateBounded<Reading>(16)` creates backpressure (the producer waits if the consumer lags), `await foreach` reads through `ReadAllAsync(ct)`, and `Writer.Complete()` in `finally` guarantees the consumer terminates even on an error (lesson: «`Channel<T>` — a thread-safe primitive for producer/consumer»). Every `await` in the library layer carries `ConfigureAwait(false)`, matching the lesson's best practice exactly. There is no `async void`; `CancellationToken` is threaded into every `await`, into `Task.Run`, and into `Channel.Writer.WriteAsync`. At the entry point (`Program.cs`, not shown) `SynchronizationContext` is `null`, so `ConfigureAwait` is omitted — a conscious decision permitted by the lesson.

#### Going deeper (bonus)

1. **Real HTTP instead of simulation.** Replace `FakeAsyncProvider` with `HttpWeatherProvider` that issues a real `HttpClient.GetAsync(url, ct)` to a public API (e.g. `https://wttr.in/{city}?format=j1`). Compare timings and confirm that higher network latency makes the gap between async and blocking even more dramatic. Reason about why `HttpClient` must be reused (singleton) rather than created per request.
2. **Pool starvation under load.** Add a scenario in which 1000 `BlockingSyncProvider`s start at once with `maxConcurrency=1000`. Measure how long the pool takes to grow to the required size (`ThreadPool.GetAvailableThreads`), and explain why the lesson calls this «pool starvation». Compare against the same number of `FakeAsyncProvider`s.
3. **UI-context deadlock (theoretical experiment).** Write a short essay-comment (or a mini WPF test if you have experience): why does calling `.Result` on `FakeAsyncProvider.ReadAsync` from the UI thread deadlock, while doing the same in `Program.Main` does not. Reference `SynchronizationContext` and the lesson's recommendation to «propagate `async` all the way to the entry point».
4. **Channel with multiple consumers.** Extend `ChannelPreview`: run 3 parallel `ReadAllAsync` consumers and confirm that each reading is processed by exactly one consumer. Measure how the `Bounded` capacity affects memory and backpressure.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `M09-L01-hw` собирается под .NET 8 / C# 12 без ошибок и предупреждений.
- [ ] (RU) Нет `async void`; `ConfigureAwait(false)` в библиотечном слое; `SemaphoreSlim` с `Release` в `finally`.
- [ ] (RU) `CancellationToken` пробрасывается во все `await`; отмена по `CancelAfter` и `Ctrl+C` работает.
- [ ] (RU) Три стратегии реализованы; бенчмарк выводит время для каждой при двух уровнях конкурентности.
- [ ] (RU) `Channel<T>`-превью работает; `Writer.Complete()` вызывается.
- [ ] (RU) README объясняет наблюдаемые числа со ссылкой на аналогию урока.
- [ ] (EN) The `M09-L01-hw` project builds under .NET 8 / C# 12 with no errors or warnings.
- [ ] (EN) No `async void`; `ConfigureAwait(false)` in the library layer; `SemaphoreSlim` with `Release` in `finally`.
- [ ] (EN) `CancellationToken` is threaded through every `await`; cancellation via `CancelAfter` and `Ctrl+C` works.
- [ ] (EN) Three strategies are implemented; the benchmark prints timing for each at two concurrency levels.
- [ ] (EN) The `Channel<T>` preview works; `Writer.Complete()` is called.
- [ ] (EN) A README explains the observed numbers, referencing the lesson's analogy.

#### Ресурсы / Resources
- [Microsoft Learn — Async programming with C#](https://learn.microsoft.com/dotnet/csharp/asynchronous-programming/)
- [Microsoft Learn — Task asynchronous programming model](https://learn.microsoft.com/dotnet/csharp/asynchronous-programming/task-asynchronous-programming-model)
- [Microsoft Learn — CancellationToken](https://learn.microsoft.com/dotnet/api/system.threading.cancellationtoken)
- [Microsoft Learn — SemaphoreSlim](https://learn.microsoft.com/dotnet/api/system.threading.semaphoreslim)
- [Microsoft Learn — Channels](https://learn.microsoft.com/dotnet/api/system.threading.channels)
- [Stephen Toub — Should I expose asynchronous wrappers for synchronous methods?](https://devblogs.microsoft.com/dotnet/should-i-expose-asynchronous-wrappers-for-synchronous-methods/)

---
[← К уроку M09-L01](lesson-M09-L01-async-intro.md) | [⬆ К модулю M09](../README.md) | [Следующее ДЗ →](homework-M09-L02-task-basics.md)
