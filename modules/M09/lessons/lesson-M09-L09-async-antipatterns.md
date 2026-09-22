[⬆ К модулю M09](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M09-L09: Антипаттерны: fire-and-forget, .Result, deadlocks / Antipatterns: fire-and-forget, .Result, deadlocks

**Модуль / Module:** M09
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Асинхронность в C# мощная, но она превращает привычный код в машину по производству тонких багов. Большинство антипаттернов возникает из одной причины: программист пытается «сделать синхронное из асинхронного» или наоборот, игнорируя контекст синхронизации.

**Fire-and-forget** — вызов `async`-метода без `await`. Кажется безобидным, но вы теряете три вещи: исключения (они проглатываются и тихо убивают процесс через `UnobservedTaskException` или вообще невидимы), контроль над завершением (вы не знаете, когда метод закончил), и корректное освобождение ресурсов. Представьте курьера, которому дают посылку, но не спрашивают расписку: посылка может потеряться, и никто не узнает. Правильно — либо `await`, либо сохранять `Task` и обрабатывать через `try/catch` или логирование в `ContinueWith`.

**`.Result` / `.Wait()` на асинхронном коде** — классический sync-over-async. В приложениях с `SynchronizationContext` (WPF, WinForms, старый ASP.NET) это создаёт **дедлок**: главный поток ждёт завершения `Task`, а `Task` ждёт, когда освободится главный поток для продолжения через `Post`. Оба стоят навечно. Аналогия: два вежливых человека в дверях — каждый пропускает другого, и никто не проходит. В консоли и ASP.NET Core `SynchronizationContext` нет, поэтому дедлока не будет, но появится другая проблема — блокировка потока пула (thread-pool starvation) под нагрузкой. Решение — «асинхронность до конца» (`async all the way`) или `ConfigureAwait(false)` в библиотечном коде.

**`async void`** — самый опасный антипаттерн. Исключения в `async void` методе падают прямо в `SynchronizationContext` и часто рвут процесс. Их нельзя `await`, нельзя обернуть в `try/catch` снаружи. Допустимо только для обработчиков событий (`event`), где сигнатура `void` навязана компилятором. Везде иначе используйте `async Task`.

**Слишком много async** — разбиение тривиальных операций на кучи `async`-методов добавляет накладные расходы на state machine, не давая выгоды. Если метод не содержит реального I/O или ожидания, оставьте его синхронным. «Async-пролиферация» встречается часто: разработчик помечает `async` всё подряд «на всякий случай».

**Sync-over-async** — обратная проблема: синхронный код вызывает асинхронный через `.Result`, `.GetAwaiter().GetResult()`, `Task.Run(...).Wait()`. Под нагрузкой потоков не хватает, и приложение «встаёт».

**Как обнаружить**: статический анализ (Roslyn-анализаторы `CA2007`, `CA2008`, `CA2012`, `CA1849`), поиск `.Result`, `.Wait()`, `async void` по кодовой базе, мониторинг `ThreadPool` starvation (рост queue length, задержки), дампы потоков на зависших процессах.

**Как исправить**: (1) превратить синхронный путь в асинхронный целиком; (2) для «настоящего» fire-and-forget использовать `Task.Run` с обработкой исключений и логированием; (3) заменить `async void` на `async Task` везде, кроме event handlers; (4) добавить `ConfigureAwait(false)` в не-UI библиотеках; (5) для фоновой работы — `BackgroundService`, `IHostedService` или `Channel<T>` с consumer-циклом. `CancellationToken` обязан пробрасываться всюду, чтобы любую очередь можно было остановить.

#### Theory (EN)

Async in C# is powerful, but it turns familiar code into a factory of subtle bugs. Most antipatterns share one root cause: the developer tries to force sync out of async (or vice versa) while ignoring the synchronization context.

**Fire-and-forget** is calling an `async` method without `await`. It looks harmless, but you lose three things at once: exceptions (they are swallowed or surface only as an `UnobservedTaskException` that may crash the process later), completion control (you no longer know when the method finished), and correct resource cleanup. Think of a courier handed a parcel with no receipt requested: the parcel can vanish and nobody will know. The fix is to either `await` the call or capture the `Task` and handle it via `try/catch` / `ContinueWith` with logging.

**`.Result` / `.Wait()` on async code** is the classic sync-over-async trap. In apps with a `SynchronizationContext` (WPF, WinForms, legacy ASP.NET) this causes a true **deadlock**: the UI thread blocks waiting for the `Task`, while the `Task` continuation is scheduled back to that same UI thread via `Post` — both wait forever. It is like two polite people stuck in a doorway, each yielding to the other. In console apps and ASP.NET Core there is no `SynchronizationContext`, so there is no deadlock — instead you get thread-pool starvation under load. The remedy is «async all the way» or `ConfigureAwait(false)` in library code.

**`async void`** is the most dangerous antipattern. Exceptions inside an `async void` method land directly on the current `SynchronizationContext` and often crash the process. You cannot `await` them, and an outer `try/catch` cannot catch them. It is only legitimate for event handlers, where `void` is forced by the compiler signature. Everywhere else use `async Task`.

**Too much async** — splitting trivial operations into a pile of `async` methods adds state-machine overhead with no benefit. If a method does no real I/O or awaiting, keep it synchronous. Async proliferation is common: developers sprinkle `async` «just in case».

**Sync-over-async** is the reverse: synchronous code calls async via `.Result`, `.GetAwaiter().GetResult()`, or `Task.Run(...).Wait()`. Under load the pool runs out of threads and the app stalls.

**How to detect**: static analysis (Roslyn analyzers `CA2007`, `CA2008`, `CA2012`, `CA1849`), grep for `.Result`, `.Wait()`, `async void`, monitor thread-pool starvation (queue length growth, latency spikes), capture thread dumps on hung processes.

**How to fix**: (1) make the call path fully async end-to-end; (2) for genuine fire-and-forget use `Task.Run` with explicit exception handling and logging; (3) replace `async void` with `async Task` everywhere except event handlers; (4) add `ConfigureAwait(false)` in non-UI libraries; (5) for background work use `BackgroundService`, `IHostedService`, or a `Channel<T>` with a consumer loop. `CancellationToken` must be threaded through everywhere so any queue can be stopped cleanly.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Antipatterns and their fixes
// Антипаттерны и их исправления

using System.Collections.Concurrent;
using System.Runtime.CompilerServices;

#pragma warning disable CA2007 // demo only / только для демонстрации

public static class AsyncAntipatterns
{
    // ❌ ANTPATTERN: fire-and-forget — exception is lost / исключение теряется
    public static void BadFireAndForget()
    {
        _ = DoWorkAsync(); // no await, exception swallowed / нет await, исключение проглатывается
    }

    // ✅ FIX: explicit handling + logging / явная обработка и логирование
    public static void GoodFireAndForget(CancellationToken ct = default)
    {
        // Capture the Task and route exceptions to logging / Перехватываем и логируем
        _ = Task.Run(async () =>
        {
            try
            {
                await DoWorkAsync(ct).ConfigureAwait(false);
            }
            catch (OperationCanceledException) { /* expected / ожидаемо */ }
            catch (Exception ex)
            {
                Console.Error.WriteLine($"[fire-and-forget] {ex}");
            }
        }, ct);
    }

    // ❌ ANTPATTERN: .Result causes deadlock on UI/legacy ASP.NET / .Result даёт дедлок
    public static string BadSyncOverAsync()
    {
        // Blocks thread; deadlock when SynchronizationContext is present
        // Блокирует поток; дедлок при наличии SynchronizationContext
        return GetDataAsync().Result;
    }

    // ✅ FIX: async all the way / асинхронность до конца
    public static async Task<string> GoodAsyncAsync(CancellationToken ct = default)
    {
        return await GetDataAsync(ct).ConfigureAwait(false);
    }

    // ❌ ANTPATTERN: async void — uncatchable exceptions / неконтролируемые исключения
    public static async void BadAsyncVoid()
    {
        await Task.Delay(100);
        throw new InvalidOperationException("boom in async void"); // рвёт процесс
    }

    // ✅ FIX: async Task (awaitable, catchable) / async Task (можно await и ловить)
    public static async Task GoodAsyncTask(CancellationToken ct = default)
    {
        await Task.Delay(100, ct).ConfigureAwait(false);
        throw new InvalidOperationException("boom in async Task");
    }

    static async Task DoWorkAsync(CancellationToken ct = default)
    {
        await Task.Delay(50, ct).ConfigureAwait(false);
        if (DateTime.UtcNow.Ticks % 2 == 0)
            throw new InvalidOperationException("random failure / случайный сбой");
    }

    static async Task<string> GetDataAsync(CancellationToken ct = default)
    {
        await Task.Delay(50, ct).ConfigureAwait(false);
        return "payload";
    }
}

// ❌ ANTPATTERN: lock + await — lock object cannot be awaited while held
//    Блокировка не освобождается во время await, другие потоки ждут
public class BadLockAwait
{
    private readonly object _gate = new();
    public async Task BadAsync(string key)
    {
        lock (_gate)          // ❌ monitor held across await / монитор держится через await
        {
            await Task.Delay(100); // CS1996? No — compiles; but lock is released only at brace exit
        }
    }
}

// ✅ FIX: SemaphoreSlim (async-friendly) / асинхронно-совместимый семафорор
public class GoodAsyncLock
{
    private readonly SemaphoreSlim _gate = new(1, 1);

    public async Task<string> GuardAsync(string key, CancellationToken ct = default)
    {
        await _gate.WaitAsync(ct).ConfigureAwait(false);
        try
        {
            await Task.Delay(100, ct).ConfigureAwait(false);
            return $"{key}-processed";
        }
        finally
        {
            _gate.Release();
        }
    }
}

// ✅ Producer/consumer pipeline via Channel<T> — safe, cancellable, no deadlocks
//    Конвейер producer/consumer через Channel<T> — потокобезопасно, отменяемо, без дедлоков
public sealed class Pipeline : IAsyncDisposable
{
    private readonly Channel<int> _channel =
        Channel.CreateBounded<int>(capacity: 16); // backpressure / обратное давление

    public async ValueTask EnqueueAsync(int item, CancellationToken ct = default)
    {
        // Writer is thread-safe: multiple producers OK / Писатель потокобезопасен
        await _channel.Writer.WriteAsync(item, ct).ConfigureAwait(false);
    }

    public IAsyncEnumerable<int> ConsumeAsync(CancellationToken ct = default) =>
        ConsumeCoreAsync(ct);

    private async IAsyncEnumerable<int> ConsumeCoreAsync(
        [EnumeratorCancellation] CancellationToken ct)
    {
        // Single consumer reads safely / Один потребитель читает безопасно
        await foreach (var item in _channel.Reader.ReadAllAsync(ct).ConfigureAwait(false))
        {
            yield return item * 10; // transform / трансформация
        }
    }

    public void Complete() => _channel.Writer.TryComplete();

    public ValueTask DisposeAsync()
    {
        _channel.Writer.TryComplete();
        return ValueTask.CompletedTask;
    }
}

// Demonstration / Демонстрация
public static class Demo
{
    public static async Task RunAsync()
    {
        await using var pipeline = new Pipeline();
        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));

        var producer = Task.Run(async () =>
        {
            for (int i = 1; i <= 10; i++)
                await pipeline.EnqueueAsync(i, cts.Token).ConfigureAwait(false);
            pipeline.Complete();
        }, cts.Token);

        var consumer = Task.Run(async () =>
        {
            await foreach (var v in pipeline.ConsumeAsync(cts.Token).ConfigureAwait(false))
                Console.WriteLine($"consumed: {v}");
        }, cts.Token);

        await Task.WhenAll(producer, consumer).ConfigureAwait(false);
    }
}
```

#### Best Practices

- Делайте метод `async` только при наличии реального I/O или ожидания; не плодите state machine без нужды. / Make a method `async` only with real I/O or awaiting; do not add a state machine for nothing.
- Пробрасывайте `CancellationToken` через весь стек; каждый `await` и каждая очередь должны его учитывать. / Thread `CancellationToken` through the whole stack; every `await` and queue must honor it.
- В библиотечном коде используйте `ConfigureAwait(false)`, чтобы не зависеть от контекста вызова. / Use `ConfigureAwait(false)` in library code to avoid depending on the caller’s context.
- Для фоновой долгой работы — `BackgroundService` + `Channel<T>`, а не «голый» `Task.Run`. / For long background work use `BackgroundService` + `Channel<T>`, not bare `Task.Run`.
- Любой fire-and-forget оборачивайте в обработку исключений и логирование. / Wrap every fire-and-forget in exception handling and logging.
- Заменяйте `lock` на `SemaphoreSlim` там, где есть `await` внутри критической секции. / Replace `lock` with `SemaphoreSlim` whenever `await` is inside the critical section.

#### Частые ошибки / Common Mistakes

- Вызов `async`-метода без `await` → сохраняйте `Task` и обрабатывайте исключения (RU). / Calling an `async` method without `await` → capture the `Task` and handle exceptions (EN).
- `.Result` на UI-потоке → дедлок; делайте метод `async Task` и `await` (RU). / `.Result` on the UI thread → deadlock; make the method `async Task` and `await` (EN).
- `async void` вне event handler → используйте `async Task` (RU). / `async void` outside an event handler → use `async Task` (EN).
- `lock` + `await` внутри → замените на `SemaphoreSlim` (RU). / `lock` + `await` inside → switch to `SemaphoreSlim` (EN).
- Забытый `CancellationToken` → очередь нельзя остановить, процесс зависает при завершении (RU). / Forgotten `CancellationToken` → the queue cannot be stopped, the process hangs on shutdown (EN).
- `ConfigureAwait(false)` везде, кроме UI → наоборот, в UI-коде `ConfigureAwait(true)` нужен для возврата в поток (RU). / `ConfigureAwait(false)` everywhere including UI → in UI code you actually need `ConfigureAwait(true)` to return to the UI thread (EN).

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Нет вызовов `async`-методов без `await` или сохранённого `Task` (RU). / No `async` calls without `await` or a captured `Task` (EN).
- [ ] Нет `.Result` / `.Wait()` на асинхронных операциях (RU). / No `.Result` / `.Wait()` on async operations (EN).
- [ ] Нет `async void` вне обработчиков событий (RU). / No `async void` outside event handlers (EN).
- [ ] `CancellationToken` пробрасывается во все `await` и очереди (RU). / `CancellationToken` flows to every `await` and queue (EN).
- [ ] В библиотеке используется `ConfigureAwait(false)` (RU). / Library code uses `ConfigureAwait(false)` (EN).
- [ ] `lock` не удерживается через `await`; используется `SemaphoreSlim` (RU). / No `lock` held across `await`; `SemaphoreSlim` is used instead (EN).
- [ ] Фоновые задачи логируют исключения и завершаются корректно (RU). / Background tasks log exceptions and shut down cleanly (EN).

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/standard/asynchronous-programming-patterns/]
- [Async/Await FAQ — Stephen Toub](https://devblogs.microsoft.com/dotnet/async-faq-where-do-i-start/)
- [Don't Block on Async Code — Stephen Cleary](https://blog.stephencleary.com/2012/07/dont-block-on-async-code.html)

---

[⬆ К модулю M09](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
