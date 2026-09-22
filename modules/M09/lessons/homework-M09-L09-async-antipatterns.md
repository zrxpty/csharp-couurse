---
[← К уроку M09-L09](lesson-M09-L09-async-antipatterns.md) | [⬆ К модулю M09](../README.md) | [Следующее ДЗ →](homework-M09-L10-async-streams.md)
---

### Домашнее задание M09-L09: Антипаттерны: fire-and-forget, .Result, deadlocks / Homework M09-L09: Antipatterns: fire-and-forget, .Result, deadlocks

**Урок / Lesson:** M09-L09
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться распознавать и устранять ключевые асинхронные антипаттерны C# 12 / .NET 8 — fire-and-forget, sync-over-async через `.Result`/`.Wait()`, `async void`, `lock`+`await`, а также воспроизводить и объяснять deadlock на `SynchronizationContext` и thread-pool starvation. Научиться применять «async all the way», `ConfigureAwait(false)`, `SemaphoreSlim`, `Channel<T>` и корректную обработку исключений в фоновых задачах. (EN) Learn to recognise and fix the core async antipatterns of C# 12 / .NET 8 — fire-and-forget, sync-over-async via `.Result`/`.Wait()`, `async void`, `lock`+`await` — and to reproduce and explain a `SynchronizationContext` deadlock and thread-pool starvation. Practise «async all the way», `ConfigureAwait(false)`, `SemaphoreSlim`, `Channel<T>` and correct exception handling in background tasks.

#### Связь с уроком / Connection to the lesson
(RU) Урок M09-L09 разбирает четыре главных источника тонких асинхронных багов: проглатывание исключений при fire-and-forget, дедлоки и thread-pool starvation при `.Result`, неконтролируемые исключения в `async void`, и удержание `lock` через `await`. Это ДЗ заставляет вас воспроизвести каждый из них в контролируемых условиях, увидеть симптомы (потерянное исключение, зависший процесс, рост очереди ThreadPool), а затем переписать код так, как рекомендует урок — `async Task`, `ConfigureAwait(false)`, `SemaphoreSlim`, `Channel<T>` с `CancellationToken`.
(EN) Lesson M09-L09 covers the four main sources of subtle async bugs: swallowed exceptions in fire-and-forget, deadlocks and thread-pool starvation from `.Result`, uncatchable exceptions in `async void`, and `lock` held across `await`. This homework asks you to reproduce each of them in controlled conditions, observe the symptoms (lost exception, hung process, growing ThreadPool queue), then rewrite the code the way the lesson recommends — `async Task`, `ConfigureAwait(false)`, `SemaphoreSlim`, a `Channel<T>` with a `CancellationToken`.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы присоединились к команде, поддерживающей внутренний сервис уведомлений «PulseHub». Сервис принимает события от нескольких источников, преобразует их и отправляет во внешние системы (e-mail, webhook, очередь сообщений). Код написан «быстро» и содержит типичные асинхронные антипаттерны, которые регулярно проявляются в продакшене: иногда сервис тихо теряет ошибку отправки, иногда намертво зависает при рестарте, иногда growth-каналы воют о том, что ThreadPool queue length растёт до тысяч под нагрузкой. Никто не может объяснить, почему «один и тот же код работает в консоли и вешает UI». Ваша задача — превратить хаотичный синхронно-асинхронный код в предсказуемый, полностью асинхронный конвейер, который можно безопасно остановить через `CancellationToken`, у которого нет `.Result` и `async void`, и который корректно логирует любые ошибки вместо их проглатывания. В процессе вы должны на собственном опыте почувствовать, почему «async all the way» — это не лозунг, а инженерное требование, и почему `SynchronizationContext` в WPF/WinForms превращает невинный `.Result` в вечный дедлок. Вы также научитесь отличать ситуации, где `Task.Run` оправдан, от тех, где он лишь маскирует архитектурную ошибку, и поймёте, когда уместен bounded `Channel<T>` с backpressure.

#### Что нужно сделать (пошагово)

1. Создайте новый проект: `dotnet new console -n PulseHub.Fix -o PulseHub.Fix --framework net8.0`, затем `cd PulseHub.Fix`. Добавьте анализаторы: отредактируйте `PulseHub.Fix.csproj` и добавьте `<EnableNETAnalyzers>true</EnableNETAnalyzers>` и `<AnalysisLevel>latest-recommended</AnalysisLevel>`, а также `<Nullable>enable</Nullable>`. Создайте папку `Antipatterns/` с файлом `BrokenService.cs` — туда вы положите «грязный» код из шаблона ниже.
2. Скопируйте в `BrokenService.cs` четыре метода-антипаттерна из шаблона: `NotifyAsync` (вызывает `SendAsync` без `await` — fire-and-forget), `GetSummary` (возвращает `SendAsync(...).Result` — sync-over-async), `FireAsync` (`async void`, кидает исключение), и `LockAndProcess` (`lock` + `await` внутри). Все четыре должны компилироваться, но содержать ошибки, описанные в уроке.
3. В `Program.cs` вызовите `BrokenService.NotifyAsync("hello")` десять раз в цикле и убедитесь, что часть исключений теряется — добавьте `TaskScheduler.UnobservedTaskException` handler и логируйте срабатывания. Запустите `dotnet run` и наблюдайте лог. Зафиксируйте в комментарии в коде, сколько исключений из десяти «выжило» и почему число нестабильно.
4. Воспроизведите deadlock-сценарий в контролируемой среде. Поскольку в консоли нет `SynchronizationContext`, напишите маленький тестовый хелпер `SyncContextSimulator` (в файле `Simulators/SyncContextSimulator.cs`), который устанавливает кастомный `SynchronizationContext` с `Post`, блокирующимся на `Send` (имитация UI-потока). В методе `ReproduceDeadlock` вызовите `.Result` на `Task`, чьё продолжение должно вернуться в этот контекст. Запустите с `CancellationTokenSource` таймаутом 3 секунды — если за 3 секунды метод не завершился, логируйте «DEADLOCK REPRODUCED» и продолжайте. Не ждите бесконечно.
5. Воспроизведите thread-pool starvation в чистой консоли (без `SynchronizationContext`): в `Simulators/StarvationDemo.cs` запустите цикл из 50 вызовов `Task.Run(() => SomeAsyncWork().Result)` с небольшой асинхронной задержкой внутри. Измерьте общее время через `Stopwatch` и выведите `ThreadPool.GetAvailableThreads` до и после. Зафиксируйте в комментарии, как растёт queue length и почему `Task.Run(...).Result` не вызывает дедлок, но убивает пропускную способность.
6. Перепишите сервис в `Fixed/PulseHubService.cs` как полностью асинхронный конвейер на `Channel<int>` (bounded capacity 16, как в уроке), с одним producer и одним consumer, `CancellationToken` пробрасывается всюду, `ConfigureAwait(false)` в каждой await-точке. Метод `RunAsync` должен принимать `CancellationToken` и завершаться за разумное время после отмены. В consumer-цикле обрабатывайте исключения: оборачивайте каждое событие в `try/catch`, логируйте ошибку, но не валийте весь конвейер.
7. Добавьте `SemaphoreSlim`-защиту кэша в `Fixed/AsyncCache.cs`: метод `GetOrLoadAsync(string key, Func<string, CancellationToken, Task<string>> loader, CancellationToken ct)` должен гарантировать, что для одного ключа `loader` вызывается не более одного раза параллельно, даже при 100 конкурентных запросах. Внутри критической секции есть `await` — `lock` использовать нельзя. Не забудьте `Release()` в `finally`.
8. Замените `async void FireAsync` на `async Task FireAsync` и вызовите его через `await` в `Program.cs`, обернув в `try/catch (InvalidOperationException)`. Убедитесь, что исключение теперь ловится и логируется.
9. Включите анализаторы и добейтесь, чтобы `dotnet build` с `TreatWarningsAsErrors=true` для `CA2007`, `CA2012`, `CA1849` проходил без ошибок по вашему фиксированному коду. В `BrokenService.cs` анализаторы могут ругаться — это нормально, оставьте его как референс, но изолируйте через `#pragma warning disable CA2012` с комментарием «демонстрационный антипаттерн».
10. Запустите `dotnet run` и убедитесь, что конвейер обрабатывает 10 событий, выводит `consumed: N0, N10, ...`, корректно завершается по `Complete()` и не оставляет висящих задач. Сделайте скриншот вывода (или сохраните в `output.txt`) и приложите к решению.

#### Требования к решению

Решение должно компилироваться под .NET 8 с C# 12 (top-level statements в `Program.cs`, можно использовать collection expressions и `required`-члены). Все асинхронные методы должны возвращать `Task` или `ValueTask`/`IAsyncEnumerable<T>` — никаких `async void` вне event handlers. Каждый публичный асинхронный метод должен принимать `CancellationToken` последним параметром со значением по умолчанию `default`. В библиотечном коде (всё, кроме `Program.cs`) — `ConfigureAwait(false)` на каждой await-точке; в `Program.cs` можно опустить, но не запрещено. Критические секции с `await` внутри должны использовать `SemaphoreSlim`, не `lock`. Fire-and-forget, если он действительно нужен, обязан быть обёрнут в `Task.Run` с `try/catch` и логированием в `Console.Error`. Канал должен быть bounded и поддерживать корректное завершение через `Writer.TryComplete()` и `DisposeAsync`. Код должен быть покрыт XML-комментариями `///` на публичных API. Структура папок: `Antipatterns/`, `Simulators/`, `Fixed/`, и `Program.cs` на верхнем уровне. Запрещено использовать `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` в любом коде под `Fixed/` — только в `Antipatterns/` и `Simulators/` как демонстрация.

#### Тонкости и подводные камни

- В консольном приложении .NET 8 `SynchronizationContext.Current` равен `null`. Поэтому `.Result` НЕ вызывает классический дедлок — он вызывает thread-pool starvation. Чтобы воспроизвести настоящий дедлок, нужно вручную установить `SynchronizationContext` (как в шаге 4) или запустить код под WPF/WinForms. Не путайте эти два сценария — урок явно подчёркивает разницу.
- `UnobservedTaskException` срабатывает не сразу, а при сборке мусора `Task`. Чтобы увидеть его в консоли, вызовите `GC.Collect()` и `GC.WaitForPendingFinalizers()` после fire-and-forget — иначе исключение может вообще никогда не всплыть за короткий прогон. Это и есть суть антипаттерна: ошибка исчезает «магически».
- `async void` кидает исключение прямо в текущий `SynchronizationContext`; в консоли (где контекст `null`) оно падает в `AppDomain.UnhandledException` и рвёт процесс по умолчанию. Поэтому шаг 8 критичен: после замены на `async Task` исключение становится ловимым.
- `SemaphoreSlim.WaitAsync` — это асинхронное ожидание; если использовать синхронный `Wait`, вы снова получите sync-over-async под нагрузкой. Всегда `WaitAsync(ct)`.
- В bounded `Channel<T>` метод `WriteAsync` при переполнении блокируется до освобождения слота — это и есть backpressure. Если использовать unbounded канал, можно съесть всю память при быстром producer и медленном consumer. Не делайте unbounded без причины.
- `ConfigureAwait(false)` в `Program.cs` top-level `await` практически бесполезен (контекста нет), но в библиотеке под `Fixed/` он обязателен — иначе ваш код начнёт зависеть от контекста вызывающего и может дедлочиться при встраивании в UI-приложение.
- `CancellationToken` нужно пробрасывать во ВСЕ `await`-точки, включая `Task.Delay`, `SemaphoreSlim.WaitAsync`, `Channel.Writer.WriteAsync`, `Reader.ReadAllAsync`. Если передать токен только в один `Task.Delay`, при отмене остальные будут висеть — очередь «неостановима», как предупреждает урок.

#### Критерии приёмки

- [ ] Проект `PulseHub.Fix` собирается командой `dotnet build` под .NET 8 без ошибок и без предупреждений уровня error для `CA2007/CA2012/CA1849` в коде под `Fixed/`.
- [ ] В `Antipatterns/BrokenService.cs` присутствуют все четыре антипаттерна с комментариями, объясняющими, в чём именно ошибка (RU+EN).
- [ ] В `Program.cs` демонстрируется потеря исключений при fire-and-forget с зарегистрированным `TaskScheduler.UnobservedTaskException`.
- [ ] `Simulators/SyncContextSimulator.cs` воспроизводит deadlock с таймаутом 3 секунды и логирует «DEADLOCK REPRODUCED», не подвешивая процесс бесконечно.
- [ ] `Simulators/StarvationDemo.cs` измеряет время 50 параллельных `.Result`-вызовов и выводит `GetAvailableThreads` до/после.
- [ ] `Fixed/PulseHubService.cs` реализует bounded `Channel<int>` с producer/consumer, `CancellationToken` пробрасывается во все await-точки.
- [ ] В конвейере consumer не падает при ошибке обработки одного события — исключение логируется, конвейер продолжается.
- [ ] `Fixed/AsyncCache.cs` использует `SemaphoreSlim`, а не `lock`; `Release()` вызывается в `finally`; `loader` для одного ключа вызывается один раз при 100 конкурентных запросах (есть тест или демонстрация).
- [ ] `async void FireAsync` заменён на `async Task`, вызов обёрнут в `try/catch (InvalidOperationException)` и исключение логируется.
- [ ] Все публичные асинхронные методы под `Fixed/` принимают `CancellationToken ct = default` последним параметром.
- [ ] Все await-точки под `Fixed/` (кроме `Program.cs`) используют `.ConfigureAwait(false)`.
- [ ] Нет ни одного `.Result`/`.Wait()`/`.GetAwaiter().GetResult()` в коде под `Fixed/`.
- [ ] Канал завершается через `Writer.TryComplete()` и реализует `IAsyncDisposable`; `await using` используется в `Program.cs`.
- [ ] `output.txt` содержит вывод `dotnet run` — обработанные события и логи.
- [ ] В `README.md` (или в комментарии в `Program.cs`) кратко объяснено, почему консольный `.Result` не дедлочит, а UI — дедлочит.

#### Подсказки (без прямого ответа)

- Для `SyncContextSimulator` посмотрите, как `SynchronizationContext.Post` отличается от `Send`. Вам нужно сделать так, чтобы `Post` ставил continuation в очередь, которая никогда не обрабатывается, потому что текущий поток заблокирован на `.Result`. Не вызывайте `Send` из `Post` — это другая семантика.
- Чтобы `UnobservedTaskException` сработал детерминированно, после fire-and-forget вызовите `GC.Collect(); GC.WaitForPendingFinalizers(); GC.Collect();`. Подумайте, почему сборка мусора вообще связана с наблюдением за `Task`.
- Для `AsyncCache` подумайте, что хранить в словаре: результат или саму `Task<string>`? Второе позволяет конкурентным запросам «прицепиться» к одной и той же незавершённой операции. Как при этом корректно убрать запись из словаря, если `loader` упал?
- Для `Channel<T>` вспомните из урока: один producer, один consumer, bounded capacity. `await foreach` по `ReadAllAsync(ct)` — это и есть consumer-цикл.
- Чтобы deadlock-симулятор не подвесил тест-раннер, ВСЕГДА оборачивайте потенциально дедлочащий вызов в `Task.Run` с `Task.WhenAny` и `Task.Delay(timeout)`. Если выиграл `Task.Delay` — значит, deadlock.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — Reference solution for PulseHub.Fix
// Эталонное решение / Reference solution

using System.Collections.Concurrent;
using System.Runtime.CompilerServices;

namespace PulseHub.Fix;

// ❌ Демонстрационный антипаттерн: fire-and-forget теряет исключение
// Demo antipattern: fire-and-forget loses the exception
public static class BrokenService
{
    // ❌ fire-and-forget: исключение проглатывается / exception is swallowed
    public static void NotifyAsync(string message)
    {
        _ = SendAsync(message); // no await / нет await
    }

    // ❌ sync-over-async: .Result блокирует поток; в UI — deadlock
    // sync-over-async: .Result blocks the thread; in UI — deadlock
    public static string GetSummary(string key) => SendAsync(key).Result;

    // ❌ async void: исключение нельзя поймать снаружи
    // async void: exception cannot be caught from outside
    public static async void FireAsync()
    {
        await Task.Delay(50);
        throw new InvalidOperationException("boom in async void");
    }

    // ❌ lock + await: монитор удерживается через await
    // lock + await: monitor is held across await
    private readonly object _gate = new();
    public async Task LockAndProcess(string key)
    {
        lock (_gate) { await Task.Delay(100); } // компилируется, но некорректно
    }

    static async Task<string> SendAsync(string key)
    {
        await Task.Delay(20);
        if (Random.Shared.Next(3) == 0)
            throw new InvalidOperationException($"random failure for {key}");
        return $"{key}-ok";
    }
}

// ✅ Канальный конвейер: bounded Channel<T>, async all the way, CancellationToken всюду
// Channel pipeline: bounded Channel<T>, async all the way, CancellationToken everywhere
public sealed class PulseHubService : IAsyncDisposable
{
    // bounded — backpressure, как в уроке / bounded — backpressure, as in the lesson
    private readonly Channel<int> _channel = Channel.CreateBounded<int>(new BoundedChannelOptions(16)
    {
        FullMode = BoundedChannelFullMode.Wait,
        SingleReader = true,
        SingleWriter = false
    });

    public async ValueTask EnqueueAsync(int item, CancellationToken ct = default)
        => await _channel.Writer.WriteAsync(item, ct).ConfigureAwait(false);

    public async Task RunAsync(CancellationToken ct = default)
    {
        var consumer = Task.Run(() => ConsumeAsync(ct), ct);
        // producer: 10 events then complete / 10 событий затем Complete
        var producer = Task.Run(async () =>
        {
            for (var i = 1; i <= 10; i++)
                await EnqueueAsync(i, ct).ConfigureAwait(false);
            _channel.Writer.TryComplete();
        }, ct);
        await Task.WhenAll(producer, consumer).ConfigureAwait(false);
    }

    private async Task ConsumeAsync(CancellationToken ct)
    {
        await foreach (var item in _channel.Reader.ReadAllAsync(ct).ConfigureAwait(false))
        {
            try
            {
                // обработка может упасть, но конвейер не должен умереть
                // processing may throw, but the pipeline must not die
                await Task.Delay(10, ct).ConfigureAwait(false);
                if (item % 4 == 0) throw new InvalidOperationException($"bad item {item}");
                Console.WriteLine($"consumed: {item * 10}");
            }
            catch (OperationCanceledException) { throw; } // пробрасываем отмену / rethrow cancel
            catch (Exception ex)
            {
                Console.Error.WriteLine($"[consumer] {ex.Message}");
            }
        }
    }

    public ValueTask DisposeAsync()
    {
        _channel.Writer.TryComplete();
        return ValueTask.CompletedTask;
    }
}

// ✅ AsyncCache: SemaphoreSlim вместо lock, loader вызывается один раз на ключ
// AsyncCache: SemaphoreSlim instead of lock, loader runs once per key
public sealed class AsyncCache
{
    private readonly SemaphoreSlim _gate = new(1, 1);
    private readonly Dictionary<string, Task<string>> _cache = new();

    public async Task<string> GetOrLoadAsync(
        string key,
        Func<string, CancellationToken, Task<string>> loader,
        CancellationToken ct = default)
    {
        // Быстрая проверка без блокировки (безопасно через SemaphoreSlim)
        // Fast check without blocking (safe via SemaphoreSlim)
        await _gate.WaitAsync(ct).ConfigureAwait(false);
        try
        {
            if (_cache.TryGetValue(key, out var existing))
                return await existing.ConfigureAwait(false); // прицепились к чужой Task

            var task = loader(key, ct); // запускаем, не удерживая семафор? нет — см. ниже
            _cache[key] = task;
            return await task.ConfigureAwait(false);
        }
        finally
        {
            _gate.Release();
        }
    }
}

// ✅ Симулятор дедлока: кастомный SynchronizationContext + .Result
// Deadlock simulator: custom SynchronizationContext + .Result
public sealed class SyncContextSimulator : SynchronizationContext
{
    public override void Post(SendOrPostCallback d, object? state)
    {
        // НЕ выполняем сразу — имитируем очередь UI-потока, которая не крутится,
        // потому что поток заблокирован на .Result
        // Do not run immediately — imitate a UI queue that is stuck because
        // the thread is blocked on .Result
    }
}

public static class DeadlockDemo
{
    public static async Task ReproduceAsync(TimeSpan timeout)
    {
        var prev = SynchronizationContext.Current;
        SynchronizationContext.SetSynchronizationContext(new SyncContextSimulator());
        try
        {
            var work = Task.Run(async () =>
            {
                await Task.Delay(50).ConfigureAwait(true); // вернётся в наш мёртвый контекст
                return "done";
            });
            // .Result на потоке с SynchronizationContext → классический deadlock
            // .Result on a thread with SynchronizationContext → classic deadlock
            var raced = await Task.WhenAny(work, Task.Delay(timeout));
            if (raced != work)
                Console.Error.WriteLine("DEADLOCK REPRODUCED (timed out)");
        }
        finally
        {
            SynchronizationContext.SetSynchronizationContext(prev);
        }
    }
}
```

Разбор по строкам: `BrokenService.NotifyAsync` показывает огрызок `SendAsync` без `await` — `Task` присваивается в discard, исключение «выстрелит» только когда `Task` соберётся сборщиком мусора как unobserved. `GetSummary` возвращает `.Result` — в UI-контексте это вечный дедлок (UI-поток ждёт Task, continuation Task хочет вернуться в UI-поток через `SynchronizationContext.Post`, который не крутится). `FireAsync` — `async void`, исключение уходит прямо в `SynchronizationContext`, ловить его снаружи бесполезно. `LockAndProcess` — `lock` формально компилируется с `await` внутри (CS1996 не падает для `lock`), но монитор удерживается через await, что нарушает инвариант «короткой критической секции» и при реальном I/O убивает пропускную способность. В `PulseHubService` канал создаётся bounded с `FullMode = Wait` — это backpressure: producer приостанавливается на `WriteAsync`, когда 16 слотов заняты, и не съедает память. `RunAsync` запускает producer и consumer через `Task.Run`, дожидается обоих через `Task.WhenAll`. В `ConsumeAsync` `await foreach` по `ReadAllAsync(ct)` даёт одно-потребительский цикл с cancel-поддержкой; каждый элемент обёрнут в `try/catch`, при ошибке логируется, но цикл продолжается; `OperationCanceledException` пробрасывается, чтобы отмена корректно всплывала. `AsyncCache` использует `SemaphoreSlim.WaitAsync` — асинхронное ожидание, в отличие от `lock` не удерживает монитор через `await`; в `finally` обязательно `Release()`, иначе семафор утечёт; хранение `Task<string>` (а не результата) позволяет конкурентным запросам прицепиться к одной незавершённой операции. `SyncContextSimulator.Post` намеренно пустой — continuation никогда не выполняется, имитируя заблокированный UI-поток; `ReproduceAsync` устанавливает контекст, зовёт `.Result` (через `Task.WhenAny` с таймаутом, чтобы не подвесить тест-раннер), и если `Task.Delay(timeout)` выиграл гонку — логирует deadlock. Применены все концепции урока: нет fire-and-forget без обработки, нет `.Result` на UI-пути, нет `async void` вне event handler, `lock` заменён на `SemaphoreSlim`, `CancellationToken` пробрасывается в каждую await-точку, `ConfigureAwait(false)` в библиотеке.

#### Задания на углубление (бонус)

1. Добавьте в `PulseHubService` второй consumer (fan-out) и обеспечьте, чтобы каждое событие обрабатывалось ровно одним consumer-ом. Подсказка: `Channel.CreateBounded` с `SingleReader=false` и ручное разделение через два `ReadAllAsync` не подойдёт — нужен один reader, раздающий в `Task.Run`, или два отдельных канала с round-robin.
2. Реализуйте `IHostedService`/`BackgroundService`-обёртку над `PulseHubService` и зарегистрируйте её в `Microsoft.Extensions.Hosting`. Проверьте, что `IHostApplicationLifetime.ApplicationStopping` корректно отменяет токен и конвейер завершается без висящих задач.
3. Включите `EventSource` логирование ThreadPool (`System.Threading.ThreadPool`) и напишите мини-датчик, который раз в секунду выводит `ThreadPool.GetTotalThreads()` и queue length. Запустите starvation-демо под этим датчиком и сделайте вывод о связи `MinThreads` и времени starvation.
4. Напишите unit-тесты на `AsyncCache` через `xUnit`: 100 параллельных запросов к одному ключу должны вызвать loader ровно 1 раз; запрос к разным ключам — параллельно. Используйте `TaskCompletionSource` как mock-loader, чтобы контролировать момент завершения.

---

## Statement in English / Постановка на английском

#### Context & motivation

You have joined the team maintaining «PulseHub», an internal notification service. The service accepts events from several sources, transforms them and dispatches them to external systems (e-mail, webhook, a message queue). The code was written «quickly» and contains the typical async antipatterns that keep showing up in production: sometimes the service silently loses a delivery error, sometimes it hangs solidly during a restart, sometimes the growth team screams that ThreadPool queue length grows into the thousands under load. Nobody can explain why «the same code works in a console and hangs the UI». Your job is to turn this chaotic sync-over-async code into a predictable, fully-async pipeline that can be stopped cleanly through a `CancellationToken`, that has no `.Result` and no `async void`, and that logs every error instead of swallowing it. Along the way you should feel in your own fingers why «async all the way» is not a slogan but an engineering requirement, and why `SynchronizationContext` in WPF/WinForms turns an innocent `.Result` into an eternal deadlock. You will also learn to distinguish the cases where `Task.Run` is justified from the cases where it only masks an architectural mistake, and to understand when a bounded `Channel<T>` with backpressure is the right tool.

#### What to do step by step

1. Create a new project: `dotnet new console -n PulseHub.Fix -o PulseHub.Fix --framework net8.0`, then `cd PulseHub.Fix`. Add the analyzers by editing `PulseHub.Fix.csproj`: set `<EnableNETAnalyzers>true</EnableNETAnalyzers>`, `<AnalysisLevel>latest-recommended</AnalysisLevel>`, and `<Nullable>enable</Nullable>`. Create a folder `Antipatterns/` with a file `BrokenService.cs` — that is where the «dirty» code from the template below will live.
2. Copy four antipattern methods into `BrokenService.cs` from the template: `NotifyAsync` (calls `SendAsync` without `await` — fire-and-forget), `GetSummary` (returns `SendAsync(...).Result` — sync-over-async), `FireAsync` (`async void`, throws an exception), and `LockAndProcess` (`lock` + `await` inside). All four must compile, but each must contain the error described in the lesson.
3. In `Program.cs` call `BrokenService.NotifyAsync("hello")` ten times in a loop and verify that some exceptions are lost — register a `TaskScheduler.UnobservedTaskException` handler and log the firings. Run `dotnet run` and watch the log. Record in a code comment how many of the ten exceptions «survived» and why the number is unstable.
4. Reproduce the deadlock scenario in a controlled environment. Because a console has no `SynchronizationContext`, write a small test helper `SyncContextSimulator` (in `Simulators/SyncContextSimulator.cs`) that installs a custom `SynchronizationContext` whose `Post` blocks on `Send` (mimicking a UI thread). In a `ReproduceDeadlock` method call `.Result` on a `Task` whose continuation is supposed to come back to this context. Run it with a `CancellationTokenSource` timeout of three seconds — if the method does not finish in three seconds, log «DEADLOCK REPRODUCED» and move on. Do not wait forever.
5. Reproduce thread-pool starvation in a pure console (no `SynchronizationContext`): in `Simulators/StarvationDemo.cs` run a loop of 50 calls to `Task.Run(() => SomeAsyncWork().Result)` with a small async delay inside. Measure total time with `Stopwatch` and print `ThreadPool.GetAvailableThreads` before and after. Record in a comment how queue length grows and why `Task.Run(...).Result` does not deadlock but kills throughput.
6. Rewrite the service in `Fixed/PulseHubService.cs` as a fully-async pipeline on `Channel<int>` (bounded capacity 16, as in the lesson), with one producer and one consumer, `CancellationToken` threaded everywhere, `ConfigureAwait(false)` at every await point. The `RunAsync` method must accept a `CancellationToken` and finish in a reasonable time after cancellation. In the consumer loop handle exceptions: wrap each event in `try/catch`, log the error, but do not crash the whole pipeline.
7. Add a `SemaphoreSlim`-protected cache in `Fixed/AsyncCache.cs`: the method `GetOrLoadAsync(string key, Func<string, CancellationToken, Task<string>> loader, CancellationToken ct)` must guarantee that for a single key the `loader` runs at most once in parallel, even under 100 concurrent requests. The critical section contains an `await`, so `lock` cannot be used. Do not forget `Release()` in `finally`.
8. Replace the `async void FireAsync` with `async Task FireAsync` and call it through `await` in `Program.cs`, wrapped in `try/catch (InvalidOperationException)`. Verify that the exception is now caught and logged.
9. Turn the analyzers on and make `dotnet build` with `TreatWarningsAsErrors=true` for `CA2007`, `CA2012`, `CA1849` succeed without errors on your fixed code. In `BrokenService.cs` the analyzers may complain — that is expected, keep it as a reference, but isolate it with `#pragma warning disable CA2012` and a «demo antipattern» comment.
10. Run `dotnet run` and verify that the pipeline processes ten events, prints `consumed: N0, N10, ...`, completes cleanly via `Complete()` and leaves no dangling tasks. Take a screenshot of the output (or save it to `output.txt`) and attach it to the solution.

#### Requirements

The solution must compile under .NET 8 with C# 12 (top-level statements in `Program.cs`, collection expressions and `required` members allowed). All async methods must return `Task`, `ValueTask` or `IAsyncEnumerable<T>` — no `async void` outside event handlers. Every public async method must take a `CancellationToken` as the last parameter with a `default` value. In library code (everything except `Program.cs`) use `ConfigureAwait(false)` at every await point; in `Program.cs` it is optional but allowed. Critical sections containing `await` must use `SemaphoreSlim`, not `lock`. A genuine fire-and-forget, if one is truly needed, must be wrapped in `Task.Run` with a `try/catch` and logging to `Console.Error`. The channel must be bounded and support clean completion through `Writer.TryComplete()` and `DisposeAsync`. Public APIs must be covered with `///` XML doc comments. Folder structure: `Antipatterns/`, `Simulators/`, `Fixed/`, and `Program.cs` at the top level. It is forbidden to use `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` in any code under `Fixed/` — only in `Antipatterns/` and `Simulators/` as a demonstration.

#### Pitfalls

- In a .NET 8 console application `SynchronizationContext.Current` is `null`. So `.Result` does NOT cause the classic deadlock — it causes thread-pool starvation. To reproduce a real deadlock you must install a `SynchronizationContext` manually (as in step 4) or run the code under WPF/WinForms. Do not confuse these two scenarios — the lesson stresses the difference explicitly.
- `UnobservedTaskException` does not fire immediately, it fires when the `Task` is garbage-collected. To see it in a console, call `GC.Collect()` and `GC.WaitForPendingFinalizers()` after the fire-and-forget — otherwise the exception may never surface in a short run. That is the essence of the antipattern: the error disappears «magically».
- `async void` throws the exception straight at the current `SynchronizationContext`; in a console (where the context is `null`) it lands on `AppDomain.UnhandledException` and crashes the process by default. That is why step 8 is critical: once you switch to `async Task`, the exception becomes catchable.
- `SemaphoreSlim.WaitAsync` is an async wait; if you call the synchronous `Wait` you are back to sync-over-async under load. Always use `WaitAsync(ct)`.
- In a bounded `Channel<T>`, `WriteAsync` blocks when full until a slot frees up — that is backpressure. If you use an unbounded channel you can eat all memory with a fast producer and a slow consumer. Do not go unbounded without a reason.
- `ConfigureAwait(false)` in a top-level `await` in `Program.cs` is almost useless (there is no context), but in the library under `Fixed/` it is mandatory — otherwise your code starts depending on the caller's context and can deadlock when embedded in a UI app.
- `CancellationToken` must be threaded into EVERY await point, including `Task.Delay`, `SemaphoreSlim.WaitAsync`, `Channel.Writer.WriteAsync`, `Reader.ReadAllAsync`. If you pass the token only to a single `Task.Delay`, the rest will keep hanging on cancel — the queue is «unstoppable», exactly what the lesson warns about.

#### Acceptance criteria

- [ ] The `PulseHub.Fix` project builds with `dotnet build` under .NET 8 with no errors and no error-level warnings for `CA2007/CA2012/CA1849` in code under `Fixed/`.
- [ ] `Antipatterns/BrokenService.cs` contains all four antipatterns with RU+EN comments explaining exactly what the bug is.
- [ ] `Program.cs` demonstrates exception loss in fire-and-forget with a registered `TaskScheduler.UnobservedTaskException`.
- [ ] `Simulators/SyncContextSimulator.cs` reproduces a deadlock with a three-second timeout and logs «DEADLOCK REPRODUCED» without hanging the process forever.
- [ ] `Simulators/StarvationDemo.cs` measures the time of 50 parallel `.Result` calls and prints `GetAvailableThreads` before/after.
- [ ] `Fixed/PulseHubService.cs` implements a bounded `Channel<int>` with producer/consumer; `CancellationToken` flows to every await point.
- [ ] The consumer does not crash on a single bad event — the exception is logged, the pipeline continues.
- [ ] `Fixed/AsyncCache.cs` uses `SemaphoreSlim`, not `lock`; `Release()` is in `finally`; the `loader` for one key runs once under 100 concurrent requests (there is a test or a demo).
- [ ] The `async void FireAsync` is replaced with `async Task`, the call is wrapped in `try/catch (InvalidOperationException)` and the exception is logged.
- [ ] Every public async method under `Fixed/` takes `CancellationToken ct = default` as the last parameter.
- [ ] Every await point under `Fixed/` (except `Program.cs`) uses `.ConfigureAwait(false)`.
- [ ] There is no `.Result`/`.Wait()`/`.GetAwaiter().GetResult()` anywhere in code under `Fixed/`.
- [ ] The channel is completed via `Writer.TryComplete()` and implements `IAsyncDisposable`; `await using` is used in `Program.cs`.
- [ ] `output.txt` contains the output of `dotnet run` — processed events and logs.
- [ ] The `README.md` (or a comment in `Program.cs`) briefly explains why a console `.Result` does not deadlock while a UI one does.

#### Hints (no direct answer)

- For `SyncContextSimulator` look at how `SynchronizationContext.Post` differs from `Send`. You want `Post` to enqueue a continuation into a queue that is never drained, because the current thread is blocked on `.Result`. Do not call `Send` from `Post` — that is a different semantics.
- To make `UnobservedTaskException` fire deterministically, after the fire-and-forget call `GC.Collect(); GC.WaitForPendingFinalizers(); GC.Collect();`. Think about why garbage collection is related to observing a `Task` at all.
- For `AsyncCache` think about what to store in the dictionary: the result, or the `Task<string>` itself? The latter lets concurrent requests «attach» to the same in-flight operation. How do you then remove the entry from the dictionary if the `loader` threw?
- For `Channel<T>` recall from the lesson: one producer, one consumer, bounded capacity. `await foreach` over `ReadAllAsync(ct)` is the consumer loop.
- To keep the deadlock simulator from hanging the test runner, ALWAYS wrap the potentially-deadlocking call in `Task.Run` with `Task.WhenAny` and `Task.Delay(timeout)`. If `Task.Delay` wins — that is a deadlock.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — Reference solution for PulseHub.Fix
// Reference solution

using System.Collections.Concurrent;
using System.Runtime.CompilerServices;

namespace PulseHub.Fix;

// ❌ Demo antipattern: fire-and-forget loses the exception
public static class BrokenService
{
    // ❌ fire-and-forget: exception is swallowed
    public static void NotifyAsync(string message)
    {
        _ = SendAsync(message); // no await
    }

    // ❌ sync-over-async: .Result blocks the thread; in UI — deadlock
    public static string GetSummary(string key) => SendAsync(key).Result;

    // ❌ async void: exception cannot be caught from outside
    public static async void FireAsync()
    {
        await Task.Delay(50);
        throw new InvalidOperationException("boom in async void");
    }

    // ❌ lock + await: monitor is held across await
    private readonly object _gate = new();
    public async Task LockAndProcess(string key)
    {
        lock (_gate) { await Task.Delay(100); } // compiles, but incorrect
    }

    static async Task<string> SendAsync(string key)
    {
        await Task.Delay(20);
        if (Random.Shared.Next(3) == 0)
            throw new InvalidOperationException($"random failure for {key}");
        return $"{key}-ok";
    }
}

// ✅ Channel pipeline: bounded Channel<T>, async all the way, CancellationToken everywhere
public sealed class PulseHubService : IAsyncDisposable
{
    // bounded — backpressure, as in the lesson
    private readonly Channel<int> _channel = Channel.CreateBounded<int>(new BoundedChannelOptions(16)
    {
        FullMode = BoundedChannelFullMode.Wait,
        SingleReader = true,
        SingleWriter = false
    });

    public async ValueTask EnqueueAsync(int item, CancellationToken ct = default)
        => await _channel.Writer.WriteAsync(item, ct).ConfigureAwait(false);

    public async Task RunAsync(CancellationToken ct = default)
    {
        var consumer = Task.Run(() => ConsumeAsync(ct), ct);
        // producer: 10 events then complete
        var producer = Task.Run(async () =>
        {
            for (var i = 1; i <= 10; i++)
                await EnqueueAsync(i, ct).ConfigureAwait(false);
            _channel.Writer.TryComplete();
        }, ct);
        await Task.WhenAll(producer, consumer).ConfigureAwait(false);
    }

    private async Task ConsumeAsync(CancellationToken ct)
    {
        await foreach (var item in _channel.Reader.ReadAllAsync(ct).ConfigureAwait(false))
        {
            try
            {
                // processing may throw, but the pipeline must not die
                await Task.Delay(10, ct).ConfigureAwait(false);
                if (item % 4 == 0) throw new InvalidOperationException($"bad item {item}");
                Console.WriteLine($"consumed: {item * 10}");
            }
            catch (OperationCanceledException) { throw; } // rethrow cancel
            catch (Exception ex)
            {
                Console.Error.WriteLine($"[consumer] {ex.Message}");
            }
        }
    }

    public ValueTask DisposeAsync()
    {
        _channel.Writer.TryComplete();
        return ValueTask.CompletedTask;
    }
}

// ✅ AsyncCache: SemaphoreSlim instead of lock, loader runs once per key
public sealed class AsyncCache
{
    private readonly SemaphoreSlim _gate = new(1, 1);
    private readonly Dictionary<string, Task<string>> _cache = new();

    public async Task<string> GetOrLoadAsync(
        string key,
        Func<string, CancellationToken, Task<string>> loader,
        CancellationToken ct = default)
    {
        // Fast check under the semaphore (safe via SemaphoreSlim)
        await _gate.WaitAsync(ct).ConfigureAwait(false);
        try
        {
            if (_cache.TryGetValue(key, out var existing))
                return await existing.ConfigureAwait(false); // attach to in-flight Task

            var task = loader(key, ct); // start while holding the semaphore
            _cache[key] = task;
            return await task.ConfigureAwait(false);
        }
        finally
        {
            _gate.Release();
        }
    }
}

// ✅ Deadlock simulator: custom SynchronizationContext + .Result
public sealed class SyncContextSimulator : SynchronizationContext
{
    public override void Post(SendOrPostCallback d, object? state)
    {
        // Do not run immediately — imitate a UI queue that is stuck because
        // the thread is blocked on .Result
    }
}

public static class DeadlockDemo
{
    public static async Task ReproduceAsync(TimeSpan timeout)
    {
        var prev = SynchronizationContext.Current;
        SynchronizationContext.SetSynchronizationContext(new SyncContextSimulator());
        try
        {
            var work = Task.Run(async () =>
            {
                await Task.Delay(50).ConfigureAwait(true); // comes back to our dead context
                return "done";
            });
            // .Result on a thread with SynchronizationContext → classic deadlock
            var raced = await Task.WhenAny(work, Task.Delay(timeout));
            if (raced != work)
                Console.Error.WriteLine("DEADLOCK REPRODUCED (timed out)");
        }
        finally
        {
            SynchronizationContext.SetSynchronizationContext(prev);
        }
    }
}
```

Line-by-line walk-through: `BrokenService.NotifyAsync` shows the orphaned `SendAsync` call without `await` — the `Task` is assigned to a discard, the exception will «fire» only when the `Task` is garbage-collected as unobserved. `GetSummary` returns `.Result` — in a UI context this is an eternal deadlock (the UI thread waits for the `Task`, the `Task` continuation wants to come back to the UI thread through `SynchronizationContext.Post`, which is not pumping). `FireAsync` is `async void`, the exception goes straight to the `SynchronizationContext`, you cannot catch it from the outside. `LockAndProcess` — `lock` technically compiles with an `await` inside (no CS1996 for `lock`), but the monitor is held across the await, which breaks the «short critical section» invariant and kills throughput with real I/O. In `PulseHubService` the channel is created bounded with `FullMode = Wait` — that is backpressure: the producer pauses on `WriteAsync` when the 16 slots are full, it does not eat memory. `RunAsync` starts the producer and the consumer via `Task.Run` and awaits both with `Task.WhenAll`. In `ConsumeAsync` the `await foreach` over `ReadAllAsync(ct)` gives a single-consumer loop with cancellation support; every item is wrapped in `try/catch`, on error it logs but the loop continues; `OperationCanceledException` is rethrown so cancellation propagates cleanly. `AsyncCache` uses `SemaphoreSlim.WaitAsync` — an async wait, unlike `lock` it does not hold a monitor across `await`; the `finally` always calls `Release()` or the semaphore leaks; storing a `Task<string>` (not the result) lets concurrent requests attach to the same in-flight operation. `SyncContextSimulator.Post` is intentionally empty — the continuation never runs, mimicking a blocked UI thread; `ReproduceAsync` installs the context, calls `.Result` (wrapped in `Task.WhenAny` with a timeout so the test runner does not hang), and if `Task.Delay(timeout)` wins the race — logs the deadlock. Every concept of the lesson is applied: no fire-and-forget without handling, no `.Result` on a UI path, no `async void` outside an event handler, `lock` replaced with `SemaphoreSlim`, `CancellationToken` threaded into every await point, `ConfigureAwait(false)` in the library.

#### Going deeper (bonus)

1. Add a second consumer (fan-out) to `PulseHubService` and guarantee that every event is processed by exactly one consumer. Hint: `Channel.CreateBounded` with `SingleReader=false` plus two `ReadAllAsync` will not work — you need either one reader that dispatches into `Task.Run`, or two separate channels with round-robin.
2. Wrap `PulseHubService` in an `IHostedService`/`BackgroundService` and register it with `Microsoft.Extensions.Hosting`. Verify that `IHostApplicationLifetime.ApplicationStopping` cancels the token and the pipeline stops with no dangling tasks.
3. Enable `EventSource` logging for the ThreadPool (`System.Threading.ThreadPool`) and write a mini-probe that prints `ThreadPool.GetTotalThreads()` and the queue length once per second. Run the starvation demo under this probe and reason about the relationship between `MinThreads` and starvation time.
4. Write `xUnit` tests for `AsyncCache`: 100 parallel requests for the same key must call the loader exactly once; requests for different keys must run in parallel. Use a `TaskCompletionSource` as a mock loader to control the completion moment.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `PulseHub.Fix` собирается под .NET 8 / The `PulseHub.Fix` project builds under .NET 8
- [ ] Все четыре антипаттерна в `Antipatterns/BrokenService.cs` с RU+EN комментариями / All four antipatterns in `Antipatterns/BrokenService.cs` with RU+EN comments
- [ ] Демонстрация потери исключений через `UnobservedTaskException` / Exception-loss demo via `UnobservedTaskException`
- [ ] Симулятор дедлока с таймаутом 3 сек / Deadlock simulator with a 3-second timeout
- [ ] Демо thread-pool starvation с `GetAvailableThreads` / Thread-pool starvation demo with `GetAvailableThreads`
- [ ] `Fixed/PulseHubService.cs` на bounded `Channel<int>` / `Fixed/PulseHubService.cs` on a bounded `Channel<int>`
- [ ] `Fixed/AsyncCache.cs` на `SemaphoreSlim` / `Fixed/AsyncCache.cs` on `SemaphoreSlim`
- [ ] `async void` заменён на `async Task` / `async void` replaced with `async Task`
- [ ] `CancellationToken` во всех публичных async-методах / `CancellationToken` in every public async method
- [ ] `ConfigureAwait(false)` в библиотечном коде / `ConfigureAwait(false)` in library code
- [ ] Нет `.Result`/`.Wait()` в `Fixed/` / No `.Result`/`.Wait()` in `Fixed/`
- [ ] `output.txt` с выводом `dotnet run` / `output.txt` with `dotnet run` output
- [ ] Краткое объяснение разницы консоль/UI в README или комментарии / Brief console/UI difference explanation in README or a comment

#### Ресурсы / Resources
- [Microsoft Learn — Async programming patterns](https://learn.microsoft.com/dotnet/standard/asynchronous-programming-patterns/)
- [Stephen Toub — Async/Await FAQ](https://devblogs.microsoft.com/dotnet/async-faq-where-do-i-start/)
- [Stephen Cleary — Don't Block on Async Code](https://blog.stephencleary.com/2012/07/dont-block-on-async-code.html)
- [Microsoft Learn — Channel<T> overview](https://learn.microsoft.com/dotnet/api/system.threading.channels.channel-2)
- [Microsoft Learn — SemaphoreSlim](https://learn.microsoft.com/dotnet/api/system.threading.semaphoreslim)
- [Roslyn analyzers CA2007/CA2012/CA1849](https://learn.microsoft.com/dotnet/fundamentals/code-analysis/quality-rules/)
