---
[← К уроку M09-L03](lesson-M09-L03-async-await-statemachine.md) | [⬆ К модулю M09](../README.md) | [Следующее ДЗ →](homework-M09-L04-task-run-cpu-vs-io.md)
---

### Домашнее задание M09-L03: async/await, компиляция в state machine / Homework M09-L03: async/await, compilation to a state machine

**Урок / Lesson:** M09-L03
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться видеть `async`/`await` как конечный автомат, генерируемый компилятором, писать потокобезопасный асинхронный код с корректной отменой и контекстом синхронизации, избегать классических дедлоков `.Result`/`.Wait()` и `async void`, а также применять `ConfigureAwait(false)`, `ValueTask<T>`, `SemaphoreSlim` и `Channel<T>` по назначению. (EN) Learn to see `async`/`await` as a compiler-generated state machine, write thread-safe asynchronous code with correct cancellation and synchronization context, avoid the classic `.Result`/`.Wait()` and `async void` deadlocks, and apply `ConfigureAwait(false)`, `ValueTask<T>`, `SemaphoreSlim`, and `Channel<T>` where they belong.

#### Связь с уроком / Connection to the lesson
(RU) Урок M09-L03 показал, что `async`/`await` — это не «магия потоков», а синтаксический сахар, переписываемый компилятором в state machine с полями `state`, `builder` и всеми локальными переменными, а вся логика уходит в сгенерированный `MoveNext()`. Это ДЗ закрепляет каждое правило из урока: захват контекста и `ConfigureAwait(false)`, протаскивание `CancellationToken` внутрь await-уемой операции, отказ от `lock` поверх `await` в пользу `SemaphoreSlim`, выбор `ValueTask<T>` для fast-path, и обязательное завершение `ChannelWriter` в `finally`. Вы будете писать код, читать сгенерированный IL/MoveNext и ловить реальные дедлоки в контролируемых условиях.
(EN) Lesson M09-L03 showed that `async`/`await` is not “thread magic” but syntactic sugar the compiler rewrites into a state machine with `state`, `builder`, and all locals as fields, with the real logic moved into a generated `MoveNext()`. This homework reinforces every rule from the lesson: context capture and `ConfigureAwait(false)`, threading a `CancellationToken` into the awaited operation, refusing `lock` over `await` in favor of `SemaphoreSlim`, choosing `ValueTask<T>` for the fast path, and always completing the `ChannelWriter` in a `finally`. You will write code, read the generated IL/MoveNext, and catch real deadlocks under controlled conditions.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы — инженер в команде, которая пишет библиотеку для фоновой обработки событий из нескольких HTTP-источников. Библиотека будет подключаться и в консольные приложения, и в десктопные (WPF/WinForms) клиенты, и в ASP.NET Core сервисы. Команда уже наступила на классические грабли: один разработчик вызвал `.Result` в обработчике кнопки и поймал дедлок; другой завернул `async`-метод в `lock (obj) { await ... }` и не понял, почему код не компилируется; третий забыл `TryComplete` у `ChannelWriter`, и потребители зависли навсегда. Руководство требует, чтобы библиотека была устойчивой: она не должна зависеть от контекста синхронизации вызывающего, должна корректно отменяться по `CancellationToken`, не должна аллоцировать `Task` на горячем пути, и обязана потокобезопасно разделять изменяемое состояние.

Вам предстоит построить небольшой модуль `EventPipeline`, который: (1) асинхронно опрашивает источник событий, (2) фильтрует и обогащает события через конкурентный pipeline на `Channel<T>`, (3) аккумулирует статистику в общем счётчике, (4) периодически сбрасывает накопленное в хранилище под асинхронной блокировкой `SemaphoreSlim`. Параллельно вы должны «увидеть» state machine своими глазами — декомпилировать свой же метод и объяснить, какие поля сгенерировал компилятор и где в `MoveNext` находятся точки возобновления. Это даст не интуитивное, а доказательное понимание того, почему `.Result` дедлочит, зачем нужен `ConfigureAwait(false)`, и почему `lock` поверх `await` запрещён.

#### Что нужно сделать (пошагово)
1. **Создайте проект.** Из папки `modules/M09/homework/M09-L03` выполните `dotnet new console -n EventPipeline -o EventPipeline --framework net8.0` и перейдите в него. Откройте `.csproj` и убедитесь, что `LangVersion` — `latest` (C# 12), при необходимости добавьте `<PropertyGroup><LangVersion>latest</LangVersion><Nullable>enable</Nullable></PropertyGroup>`. Замените содержимое `Program.cs` на top-level statements.
2. **Модель события.** Определите `record EventEnvelope(int Id, string Source, DateTimeOffset CreatedAt, ReadOnlyMemory<byte> Payload);`. Источник-генератор `IEventSource` с методом `ValueTask<EventEnvelope?> ReadAsync(CancellationToken ct)` — он должен часто завершаться синхронно (возвращать уже готовое событие из буфера), чтобы `ValueTask` действительно показывал fast path.
3. **Pipeline на `Channel<T>`.** Реализуйте `sealed class EventPipeline` с методом `Task RunAsync(IEventSource source, CancellationToken ct)`. Внутри создайте `Channel.CreateBounded<EventEnvelope>(64)` с `FullMode = BoundedChannelFullMode.Wait`. Запустите одного продюсера, который читает источник и пишет в канал, и двух потребителей, которые читают, симулируют обработку через `Task.Delay` и обновляют общий счётчик `processedCount` через `Interlocked.Increment`. В `finally` после продюсера обязательно вызовите `channel.Writer.TryComplete()`.
4. **Общий счётчик и сброс.** Добавьте поле `long processedCount` и `static readonly SemaphoreSlim s_flushGate = new(1,1);`. Метод `FlushAsync(CancellationToken ct)` должен брать `await s_flushGate.WaitAsync(ct).ConfigureAwait(false)` в `try`, читать текущее значение через `Interlocked.Exchange(ref processedCount, 0)`, «сохранять» его (можно просто логировать) и `Release()` в `finally`. Никакого `lock` поверх `await` быть не должно.
5. **Отмена.** Прокиньте `CancellationToken` в каждый `await`-уемый вызов: `Task.Delay`, `writer.WriteAsync`, `reader.WaitToReadAsync`, `s_flushGate.WaitAsync`. В цикле продюсера поставьте `ct.ThrowIfCancellationRequested()` в начале каждой итерации.
6. **ConfigureAwait.** В библиотечном классе `EventPipeline` после **каждого** `await` ставьте `.ConfigureAwait(false)`. В `Program.cs` (точка входа, нет UI-контекста) можно не ставить, но обоснуйте в комментарии.
7. **Декомпиляция.** Соберите Release-сборку: `dotnet build -c Release`. Откройте DLL в ILSpy/`ilspycmd` (`dotnet tool install -g ilspycmd`, затем `ilspycmd EventPipeline.dll -t EventPipeline.EventPipeline > decompiled.txt`) либо вставьте код в https://sharplab.io. Найдите сгенерированный `struct <RunAsync>d__x` с полями `<>1__state`, `<>t__builder`, локальными и параметрами. Найдите `MoveNext()` и определите, какие значения `state` соответствуют какой точке `await`.
8. **Демонстрация дедлока (опционально, но рекомендуется).** В отдельном методе `DemonstrateDeadlock()` симулируйте UI-контекст: установите свой `SynchronizationContext` (например, простой `SingleThreadSynchronizationContext` из примеров в интернете) и вызовите `.Result` на `async`-методе, который внутри использует `ConfigureAwait(true)` (по умолчанию). Должен зависнуть. Затем переключите внутренний метод на `ConfigureAwait(false)` — дедлок исчезнет. Запишите наблюдение в комментарий.
9. **Запуск.** В `Program.cs` создайте `using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));` и запустите `await pipeline.RunAsync(source, cts.Token);`. Вывод должен содержать логи потребителей (`[A] processed ...`, `[B] processed ...`) и периодические `flush`-сообщения. По истечении 5 секунд пайплайн должен корректно завершиться, а не зависнуть.
10. **Ожидаемый вывод.** При запуске `dotnet run -c Release` вы должны увидеть, что сообщения об обработке приходят из разных потоков (`Environment.CurrentManagedThreadId` варьируется), что после отмены процесс завершается за разумное время (проверьте `dotnet run` не висит), и что `decompiled.txt` содержит struct state machine, а не class (признак Release-режима).

#### Требования к решению
- Код компилируется под .NET 8 / C# 12 без предупреждений уровня error (предупреждения CS1998 «async method without await» недопустимы — если метод без `await`, уберите `async`).
- Все публичные асинхронные методы библиотеки принимают `CancellationToken ct = default` (или явный `CancellationToken ct`) и пробрасывают его в каждый `await`.
- После каждого `await` в библиотечном коде стоит `.ConfigureAwait(false)`. В `Program.cs` это опционально, но должно быть прокомментировано.
- Общее изменяемое состояние (`processedCount`) модифицируется только через `Interlocked` или защищено `SemaphoreSlim`; `lock`/`Monitor` не удерживается через `await`.
- `ChannelWriter` обязательно завершается (`TryComplete`) в `finally` после продюсера; потребители корректно обрабатывают `ChannelClosedException`.
- Метод `ReadAsync` источника возвращает `ValueTask<EventEnvelope?>`, и хотя бы одна ветка завершается синхронно (`IsCompleted == true` сразу) — для демонстрации fast path.
- Декомпилированный `MoveNext` приложен в виде текстового файла `decompiled.txt` или вставлен в комментарий в `Program.cs`; в нём указаны номера `state` для каждой точки `await`.
- Решение запускается как `dotnet run -c Release`, корректно завершается по `CancellationToken` за ≤ 1 секунду после отмены, не оставляет висящих задач.

#### Тонкости и подводные камни
- **`ValueTask` не thread-safe для множественного await.** Не ставьте `await` на одну и ту же `ValueTask` дважды и не вызывайте `.Result` после `await` — `ValueTask` может быть pooling-backed, и повторное использование сломает пул. Если нужен multiple-await — конвертируйте в `Task` через `.AsTask()`.
- **`ConfigureAwait(false)` не «отключает» контекст навсегда.** Он действует только на конкретный `await`. Следующий `await` без `ConfigureAwait(false)` снова захватит контекст (если он есть). Поэтому правило «всегда и везде в библиотеке».
- **`SynchronizationContext.Current == null` в консоли и ASP.NET Core.** Там `ConfigureAwait(false)` формально ничего не меняет, но оставлять его обязательно — код может быть вызван из UI-приложения завтра.
- **`async void` крашит процесс.** Если в `DemonstrateDeadlock` случайно сделать обработчик `async void` и бросить исключение — упадёт всё. Используйте `async Task` и `.GetAwaiter().GetResult()` в точке входа.
- **`lock (obj) { await ... }` не компилируется.** Это не баг, а защита: `Monitor` привязан к потоку, continuation — к другому. Ручной `Monitor.Enter` через `await` ещё опаснее. Только `SemaphoreSlim.WaitAsync`.
- **`Task.Delay` сам по себе не отменяем без токена.** `Task.Delay(1000)` без `ct` проигнорирует отмену и досчитает до конца. Всегда `Task.Delay(..., ct)`.
- **`Channel.CreateBounded` без `FullMode` дефолтит на `Wait`.** Это правильно для back-pressure, но помните: продюсер будет заблокирован в `WriteAsync`, если буфер полон, и отменить это можно только через `ct`.
- **`TryComplete` vs `Complete`.** `Complete` бросает, если канал уже завершён; `TryComplete` — нет. В `finally` всегда `TryComplete`.
- **`async` метод без `await` — CS1998.** Метод работает синхронно и возвращает уже-завершённую `Task`. Если действительно нечего ждать — уберите `async` и верните `Task.CompletedTask` или `ValueTask.CompletedTask`.

#### Критерии приёмки
- [ ] Проект `EventPipeline` собирается под .NET 8 / C# 12 без error-предупреждений (`dotnet build -c Release` зелёный).
- [ ] Использованы top-level statements в `Program.cs` и file-scoped namespace.
- [ ] `IEventSource.ReadAsync` возвращает `ValueTask<EventEnvelope?>` и имеет синхронно-завершающуюся ветку.
- [ ] Pipeline построен на `Channel.CreateBounded<EventEnvelope>` с back-pressure (`FullMode.Wait`).
- [ ] Один продюсер, ≥2 потребителя; оба потребителя читают из одного `ChannelReader`.
- [ ] `channel.Writer.TryComplete()` вызывается в `finally` после продюсера.
- [ ] Потребители обрабатывают `ChannelClosedException` и не падают.
- [ ] `processedCount` модифицируется через `Interlocked`; нет `lock` поверх `await`.
- [ ] `FlushAsync` использует `SemaphoreSlim.WaitAsync` с `Release()` в `finally`.
- [ ] `CancellationToken` пробрасывается во все `await`-уемые операции (`Task.Delay`, `WriteAsync`, `WaitToReadAsync`, `WaitAsync`).
- [ ] После каждого `await` в библиотечном коде стоит `.ConfigureAwait(false)`.
- [ ] Декомпилированный `MoveNext` приложен (`decompiled.txt` или комментарий); указаны значения `state` для ≥2 точек `await`.
- [ ] Запуск `dotnet run -c Release` завершается корректно по истечении `CancellationToken` (не висит).
- [ ] (Бонус) `DemonstrateDeadlock` воспроизводит и фиксит дедлок через `ConfigureAwait(false)`.
- [ ] В комментариях (RU+EN) объяснено, почему каждый `ConfigureAwait(false)` нужен именно там.

#### Подсказки (без прямого ответа)
- Чтобы `ValueTask` действительно показал fast path, в `ReadAsync` проверьте, есть ли в локальном `Queue` готовое событие; если есть — верните `ValueTask<EventEnvelope?>.FromResult(...)` без единого `await`.
- Для декомпиляции достаточно онлайн https://sharplab.io: вставьте свой метод, выберите «Results → C# → (Reduce async)» или «IL», и сравните с тем, что выдаёт `ilspycmd`.
- Помните: `state == -1` — до первого `await`, `0` — первая точка возобновления, `-2` — завершён. Найдите в `MoveNext` `switch` или `goto` по этим числам.
- Для симуляции UI-контекста достаточно класса, который хранит `Queue<Action>` и обрабатывает её в одном потоке через `Post`. Установите его через `SynchronizationContext.SetSynchronizationContext(...)`.
- Чтобы увидеть разницу struct vs class state machine, сравните Debug и Release сборки: в Release компилятор использует `struct`, и при синхронном завершении аллокации нет.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — рабочее эталонное решение.
// Демонстрирует: state machine (через декомпиляцию), ConfigureAwait(false),
// CancellationToken, ValueTask fast path, SemaphoreSlim, Channel<T>.
// C# 12 / .NET 8 — reference solution.
// Demonstrates: state machine (via decompilation), ConfigureAwait(false),
// CancellationToken, ValueTask fast path, SemaphoreSlim, Channel<T>.

using System.Collections.Concurrent;
using System.Runtime.CompilerServices;
using System.Threading.Channels;

namespace EventPipeline;

// Источник событий: часто завершается синхронно, чтобы ValueTask показал fast path.
// Event source: frequently completes synchronously so ValueTask shows the fast path.
public interface IEventSource
{
    ValueTask<EventEnvelope?> ReadAsync(CancellationToken ct);
}

public sealed record EventEnvelope(int Id, string Source, DateTimeOffset CreatedAt, ReadOnlyMemory<byte> Payload);

public sealed class InMemoryEventSource : IEventSource
{
    // ConcurrentQueue: потокобезопасная очередь без lock.
    // ConcurrentQueue: thread-safe queue, no lock needed.
    private readonly ConcurrentQueue<EventEnvelope> _queue = new();

    public InMemoryEventSource()
    {
        // Предзаполнение, чтобы ReadAsync часто возвращал результат синхронно.
        // Pre-fill so ReadAsync often returns synchronously.
        for (int i = 0; i < 1000; i++)
            _queue.Enqueue(new EventEnvelope(i, "mem", DateTimeOffset.UtcNow, new byte[16]));
    }

    public ValueTask<EventEnvelope?> ReadAsync(CancellationToken ct)
    {
        // FAST PATH: ct не отменен и в очереди есть элемент — возвращаем без await.
        // Состояние state machine даже не инициализируется — аллокации нет.
        // FAST PATH: ct not canceled and queue has an item — return without await.
        // The state machine is never even started — no allocation.
        if (!ct.IsCancellationRequested && _queue.TryDequeue(out var evt))
            return ValueTask<EventEnvelope?>.FromResult(evt);

        // SLOW PATH: реальный await — тут компилятор строит state machine.
        // SLOW PATH: a real await — here the compiler builds the state machine.
        return SlowReadAsync(this, ct);

        static async ValueTask<EventEnvelope?> SlowReadAsync(InMemoryEventSource self, CancellationToken ct)
        {
            await Task.Delay(5, ct).ConfigureAwait(false);
            if (ct.IsCancellationRequested) return null;
            return self._queue.TryDequeue(out var evt) ? evt : null;
        }
    }
}

public sealed class EventPipeline
{
    private long _processedCount;           // атомарно через Interlocked / atomic via Interlocked
    private readonly SemaphoreSlim _flushGate = new(1, 1); // async lock

    public async Task RunAsync(IEventSource source, CancellationToken ct)
    {
        // Bounded + Wait = back-pressure: продюсер ждёт, если буфер полон.
        // Bounded + Wait = back-pressure: producer awaits when full.
        var channel = Channel.CreateBounded<EventEnvelope>(new BoundedChannelOptions(64)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = false,
            SingleWriter = true,
        });

        var producer = ProduceAsync(source, channel.Writer, ct);
        var consumerA = ConsumeAsync(channel.Reader, "A", ct);
        var consumerB = ConsumeAsync(channel.Reader, "B", ct);

        // flush-loop параллельно: периодически сбрасывает статистику.
        // Flush loop in parallel: periodically persists statistics.
        var flusher = FlushLoopAsync(ct);

        try
        {
            await producer.ConfigureAwait(false); // дожидаемся конца источника / wait for source end
        }
        finally
        {
            // КРИТИЧНО: без TryComplete потребители зависнут в WaitToReadAsync.
            // CRITICAL: without TryComplete, consumers hang in WaitToReadAsync.
            channel.Writer.TryComplete();
        }

        await Task.WhenAll(consumerA, consumerB).ConfigureAwait(false);

        // Останавливаем flush-цикл и делаем финальный сброс.
        // Stop the flush loop and do a final flush.
        await flusher.ConfigureAwait(false);
        await FlushAsync(ct).ConfigureAwait(false);
    }

    private static async Task ProduceAsync(IEventSource source, ChannelWriter<EventEnvelope> writer, CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            ct.ThrowIfCancellationRequested();
            // ReadAsync — ValueTask: на fast path нет аллокации Task.
            // ReadAsync returns ValueTask: no Task allocation on the fast path.
            var evt = await source.ReadAsync(ct).ConfigureAwait(false);
            if (evt is null) break;

            // WriteAsync применяет back-pressure при полном буфере.
            // WriteAsync applies back-pressure when the buffer is full.
            await writer.WriteAsync(evt.Value, ct).ConfigureAwait(false);
        }
    }

    private async Task ConsumeAsync(ChannelReader<EventEnvelope> reader, string id, CancellationToken ct)
    {
        try
        {
            // Стандартный цикл потребления: WaitToReadAsync + TryRead.
            // Standard consume loop: WaitToReadAsync + TryRead.
            while (await reader.WaitToReadAsync(ct).ConfigureAwait(false))
            {
                while (reader.TryRead(out var evt))
                {
                    try
                    {
                        await ProcessAsync(evt, ct).ConfigureAwait(false);
                        // ПОТОКОБЕЗОПАСНОЕ накопление без lock.
                        // THREAD-SAFE accumulation without a lock.
                        Interlocked.Increment(ref _processedCount);
                        Console.WriteLine($"[{id}] thr={Environment.CurrentManagedThreadId} processed id={evt.Id}");
                    }
                    catch (OperationCanceledException) { throw; }
                    catch (Exception ex)
                    {
                        Console.WriteLine($"[{id}] item {evt.Id} failed: {ex.Message}");
                    }
                }
            }
        }
        catch (ChannelClosedException) { /* нормальный выход / normal exit */ }
    }

    private static async Task ProcessAsync(EventEnvelope evt, CancellationToken ct)
    {
        // I/O-имитация; ct внутрь — обязательно.
        // Simulated I/O; ct must be threaded inside.
        await Task.Delay(2, ct).ConfigureAwait(false);
    }

    private async Task FlushAsync(CancellationToken ct)
    {
        // Async lock: НЕ lock(Monitor), а SemaphoreSlim.WaitAsync.
        // Async lock: NOT lock/Monitor, but SemaphoreSlim.WaitAsync.
        await _flushGate.WaitAsync(ct).ConfigureAwait(false);
        try
        {
            long snapshot = Interlocked.Exchange(ref _processedCount, 0);
            Console.WriteLine($"[flush] saved {snapshot} events at {DateTimeOffset.UtcNow:O}");
        }
        finally
        {
            _flushGate.Release(); // обязательно в finally / always in finally
        }
    }

    private async Task FlushLoopAsync(CancellationToken ct)
    {
        try
        {
            while (!ct.IsCancellationRequested)
            {
                await Task.Delay(TimeSpan.FromMilliseconds(500), ct).ConfigureAwait(false);
                await FlushAsync(ct).ConfigureAwait(false);
            }
        }
        catch (OperationCanceledException) { /* ожидаемо при отмене / expected on cancel */ }
    }
}

// Точка входа: top-level statements. Контекста синхронизации нет —
// ConfigureAwait(false) опционален, но мы оставили его в библиотеке.
// Entry point: top-level statements. No synchronization context —
// ConfigureAwait(false) is optional here, but we kept it in the library.
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
var pipeline = new EventPipeline();
var source = new InMemoryEventSource();
try
{
    await pipeline.RunAsync(source, cts.Token);
}
catch (OperationCanceledException) { }
Console.WriteLine("done");
```

Разбор по строкам (разбор). `IEventSource.ReadAsync` возвращает `ValueTask<EventEnvelope?>`, а не `Task<EventEnvelope?>`: по уроку, `ValueTask` предпочтительнее, когда метод часто завершается синхронно (fast path) — в нашей реализации, пока в `ConcurrentQueue` есть элементы, метод вообще не доходит до `await`, и компилятор не запускает state machine (аллокации нет). Только при пустой очереди вызывается `SlowReadAsync`, где стоит реальный `await Task.Delay(5, ct)`, — именно тут компилятор генерирует `struct`-state machine с полями `state`, `builder`, локальными `self`/`ct`. В `EventPipeline.RunAsync` создан `Channel.CreateBounded<EventEnvelope>(64)` с `FullMode.Wait`: это back-pressure из урока — продюсер не забивает память, а ждёт в `WriteAsync` при полном буфере. После каждого `await` в библиотеке стоит `ConfigureAwait(false)`: по уроку, библиотечный код не должен захватывать контекст вызывающего, иначе в UI-приложении возникнет дедлок, а в ASP.NET Core — лишняя нагрузка на поток. `Producer` в `finally` вызывает `channel.Writer.TryComplete()` — без этого `ConsumeAsync` зависнет в `WaitToReadAsync` навсегда (прямая ошибка из «Частых ошибок» урока). `processedCount` модифицируется только через `Interlocked.Increment`/`Exchange`: по уроку, `await` сам по себе не делает код потокобезопасным (continuation может выполняться в другом потоке), а `lock` поверх `await` запрещён — поэтому `Interlocked` для счётчика и `SemaphoreSlim` для критической секции. `FlushAsync` берёт `_flushGate.WaitAsync(ct)` в `try` и `Release()` в `finally`: это асинхронный аналог `lock`, единственно корректный через границу `await`. `CancellationToken` проброшен в **каждый** `await`-уемый вызов — `Task.Delay`, `WriteAsync`, `WaitToReadAsync`, `WaitAsync`: по уроку, нельзя опрашивать `IsCancellationRequested` между await-ами, нужно передавать токен внутрь операции, чтобы отмена срабатывала мгновенно. В `Program.cs` (top-level statements) `SynchronizationContext.Current == null`, поэтому `ConfigureAwait(false)` опционален — но в библиотеке он обязателен на будущее. Декомпиляция (шаг 7) покажет, что в Release компилятор использует `struct`-state machine (а не class как в Debug), что и объясняет отсутствие аллокаций при синхронном завершении. Если вы запустите `DemonstrateDeadlock` с кастомным `SynchronizationContext` и внутренним `ConfigureAwait(true)`, то `.Result` зависнет: поток ждёт задачу, задача ждёт возврата в тот же поток (классическая ловушка из урока); переключение на `ConfigureAwait(false)` разрывает цикл.

#### Задания на углубление (бонус)
1. **Pooling ValueTask.** Пометьте hot-path метод атрибутом `[AsyncMethodBuilder(typeof(PoolingAsyncValueTaskMethodBuilder<>))]` и сравните количество аллокаций (через `dotnet-counters` или `dotnet-trace`) до и после. Объясните, почему pooling безопасен только при соблюдении правил «не await дважды, не читать `.Result` после `await`».
2. **Кастомный awaiter.** Реализуйте свой `awaitable`/`awaiter` (например, `YieldToThreadPoolAwaitable`), который через `UnsafeOnCompleted` планирует continuation в пуле без захвата `ExecutionContext`. Сравните с `Task.Yield()` и `ConfigureAwait(false)`.
3. **Дедлок-полигон.** Напишите минимальный `SynchronizationContext` на одном потоке и метод `DeadlockAsync`, воспроизводящий зависание `.Result` без `ConfigureAwait(false)`. Добавьте тайм-аут через `Task.WhenAny(task, Task.Delay(2000))`, чтобы тест не висел в CI.
4. **Множественный consumer-balance.** Расширьте pipeline до динамического числа потребителей и измерьте пропускную способность при 1, 2, 4, 8 потребителях. Объясните, почему `Channel<T>` с `SingleReader=false` безопасен, и почему рост потребителей сверх числа ядер перестаёт давать ускорение.

---

## Statement in English / Постановка на английском

#### Context & motivation
You are an engineer on a team writing a library that processes background events from several HTTP sources. The library will be referenced from console apps, desktop clients (WPF/WinForms), and ASP.NET Core services. The team has already hit the classic traps: one developer called `.Result` inside a button handler and deadlocked; another wrapped an `async` method in `lock (obj) { await ... }` and could not understand why it would not compile; a third forgot to call `TryComplete` on the `ChannelWriter`, and consumers hung forever. Management requires the library to be robust: it must not depend on the caller’s synchronization context, it must cancel correctly via `CancellationToken`, it must not allocate a `Task` on the hot path, and it must share mutable state safely across threads.

Your task is to build a small `EventPipeline` module that: (1) asynchronously polls an event source, (2) filters and enriches events through a concurrent `Channel<T>` pipeline, (3) accumulates statistics in a shared counter, and (4) periodically flushes the accumulated value to storage under an async `SemaphoreSlim` lock. In parallel, you must “see” the state machine with your own eyes — decompile your own method and explain which fields the compiler generated and where the resumption points live inside `MoveNext`. This gives you not an intuitive but a demonstrative understanding of why `.Result` deadlocks, why `ConfigureAwait(false)` exists, and why `lock` over `await` is forbidden.

#### What to do step by step
1. **Create the project.** From `modules/M09/homework/M09-L03`, run `dotnet new console -n EventPipeline -o EventPipeline --framework net8.0` and `cd` into it. Open the `.csproj` and ensure `LangVersion` is `latest` (C# 12); add `<PropertyGroup><LangVersion>latest</LangVersion><Nullable>enable</Nullable></PropertyGroup>` if needed. Replace `Program.cs` with top-level statements.
2. **Event model.** Define `record EventEnvelope(int Id, string Source, DateTimeOffset CreatedAt, ReadOnlyMemory<byte> Payload);`. Provide an `IEventSource` with `ValueTask<EventEnvelope?> ReadAsync(CancellationToken ct)` that frequently completes synchronously (returns a ready event from a buffer), so `ValueTask` actually demonstrates the fast path.
3. **`Channel<T>` pipeline.** Implement `sealed class EventPipeline` with `Task RunAsync(IEventSource source, CancellationToken ct)`. Inside, create `Channel.CreateBounded<EventEnvelope>(64)` with `FullMode = BoundedChannelFullMode.Wait`. Start one producer that reads the source and writes to the channel, and two consumers that read, simulate work with `Task.Delay`, and update a shared `processedCount` via `Interlocked.Increment`. In a `finally` after the producer, call `channel.Writer.TryComplete()`.
4. **Shared counter and flush.** Add a `long processedCount` field and `static readonly SemaphoreSlim s_flushGate = new(1,1);`. The method `FlushAsync(CancellationToken ct)` must `await s_flushGate.WaitAsync(ct).ConfigureAwait(false)` in a `try`, read the current value via `Interlocked.Exchange(ref processedCount, 0)`, “persist” it (logging is fine), and `Release()` in `finally`. No `lock` over `await` is allowed anywhere.
5. **Cancellation.** Thread the `CancellationToken` into every awaited call: `Task.Delay`, `writer.WriteAsync`, `reader.WaitToReadAsync`, `s_flushGate.WaitAsync`. At the top of each producer iteration, call `ct.ThrowIfCancellationRequested()`.
6. **ConfigureAwait.** In the library class `EventPipeline`, put `.ConfigureAwait(false)` after **every** `await`. In `Program.cs` (entry point, no UI context) it is optional, but justify it in a comment.
7. **Decompilation.** Build a Release binary: `dotnet build -c Release`. Open the DLL in ILSpy/`ilspycmd` (`dotnet tool install -g ilspycmd`, then `ilspycmd EventPipeline.dll -t EventPipeline.EventPipeline > decompiled.txt`) or paste the code into https://sharplab.io. Locate the generated `struct <RunAsync>d__x` with fields `<>1__state`, `<>t__builder`, locals, and parameters. Find `MoveNext()` and identify which `state` values correspond to which `await` point.
8. **Deadlock demonstration (optional but recommended).** In a separate method `DemonstrateDeadlock()`, simulate a UI context: install a custom `SynchronizationContext` (a simple `SingleThreadSynchronizationContext` from public examples) and call `.Result` on an `async` method that internally uses `ConfigureAwait(true)` (the default). It should hang. Then switch the inner method to `ConfigureAwait(false)` — the deadlock disappears. Record the observation in a comment.
9. **Run.** In `Program.cs`, create `using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));` and `await pipeline.RunAsync(source, cts.Token);`. The output should contain consumer logs (`[A] processed ...`, `[B] processed ...`) and periodic `flush` lines. After 5 seconds the pipeline must shut down gracefully, not hang.
10. **Expected output.** Running `dotnet run -c Release` should show that processing messages come from different threads (`Environment.CurrentManagedThreadId` varies), that the process exits promptly after cancellation (`dotnet run` does not hang), and that `decompiled.txt` contains a struct state machine (not a class) — a sign of Release mode.

#### Requirements
- The code compiles under .NET 8 / C# 12 with no error-level warnings (CS1998 “async method without await” is unacceptable — if a method has no `await`, drop `async`).
- Every public async method in the library takes `CancellationToken ct = default` (or an explicit `CancellationToken ct`) and forwards it into each `await`.
- After every `await` in library code there is a `.ConfigureAwait(false)`. In `Program.cs` it is optional but must be commented.
- Shared mutable state (`processedCount`) is modified only via `Interlocked` or guarded by `SemaphoreSlim`; no `lock`/`Monitor` is held across an `await`.
- The `ChannelWriter` is always completed (`TryComplete`) in a `finally` after the producer; consumers handle `ChannelClosedException` gracefully.
- The source’s `ReadAsync` returns `ValueTask<EventEnvelope?>`, and at least one branch completes synchronously (`IsCompleted == true` immediately) to demonstrate the fast path.
- The decompiled `MoveNext` is attached as `decompiled.txt` or embedded in a comment in `Program.cs`; the `state` numbers for each `await` point are annotated.
- The solution runs as `dotnet run -c Release`, shuts down correctly on `CancellationToken` within ≤ 1 second of cancellation, and leaves no leaked tasks.

#### Pitfalls
- **`ValueTask` is not thread-safe for multiple awaits.** Do not `await` the same `ValueTask` twice, and do not read `.Result` after `await` — a `ValueTask` may be pooling-backed, and reuse corrupts the pool. If you need multiple awaits, convert to `Task` via `.AsTask()`.
- **`ConfigureAwait(false)` does not “disable” the context forever.** It only affects the specific `await`. The next `await` without `ConfigureAwait(false)` captures the context again (if any). Hence the rule “always, everywhere, in libraries”.
- **`SynchronizationContext.Current == null` in console and ASP.NET Core.** There `ConfigureAwait(false)` changes nothing formally — but keep it, because the code may be called from a UI app tomorrow.
- **`async void` crashes the process.** If you accidentally make a handler `async void` in `DemonstrateDeadlock` and throw — the whole process dies. Use `async Task` and `.GetAwaiter().GetResult()` at the entry point.
- **`lock (obj) { await ... }` does not compile.** Not a bug, a safeguard: `Monitor` is thread-affined, the continuation is not. Manual `Monitor.Enter` across `await` is worse. Use `SemaphoreSlim.WaitAsync`.
- **`Task.Delay` is not cancelable without a token.** `Task.Delay(1000)` without `ct` ignores cancellation and runs to the end. Always `Task.Delay(..., ct)`.
- **`Channel.CreateBounded` without `FullMode` defaults to `Wait`.** Correct for back-pressure, but remember: the producer blocks in `WriteAsync` when full, and only `ct` can interrupt that.
- **`TryComplete` vs `Complete`.** `Complete` throws if the channel is already completed; `TryComplete` does not. In `finally`, always `TryComplete`.
- **An `async` method without `await` produces CS1998.** It runs synchronously and returns an already-completed `Task`. If there is genuinely nothing to await, drop `async` and return `Task.CompletedTask` or `ValueTask.CompletedTask`.

#### Acceptance criteria
- [ ] The `EventPipeline` project builds under .NET 8 / C# 12 with no error-level warnings (`dotnet build -c Release` is green).
- [ ] Top-level statements are used in `Program.cs` with a file-scoped namespace.
- [ ] `IEventSource.ReadAsync` returns `ValueTask<EventEnvelope?>` and has a synchronously-completing branch.
- [ ] The pipeline is built on `Channel.CreateBounded<EventEnvelope>` with back-pressure (`FullMode.Wait`).
- [ ] One producer and ≥2 consumers; both consumers read from the same `ChannelReader`.
- [ ] `channel.Writer.TryComplete()` is called in a `finally` after the producer.
- [ ] Consumers handle `ChannelClosedException` and do not crash.
- [ ] `processedCount` is modified via `Interlocked`; no `lock` over `await`.
- [ ] `FlushAsync` uses `SemaphoreSlim.WaitAsync` with `Release()` in `finally`.
- [ ] `CancellationToken` is forwarded into every awaited operation (`Task.Delay`, `WriteAsync`, `WaitToReadAsync`, `WaitAsync`).
- [ ] Every `await` in library code is followed by `.ConfigureAwait(false)`.
- [ ] The decompiled `MoveNext` is attached (`decompiled.txt` or a comment); the `state` values for ≥2 `await` points are annotated.
- [ ] `dotnet run -c Release` terminates cleanly when the `CancellationToken` fires (no hang).
- [ ] (Bonus) `DemonstrateDeadlock` reproduces and fixes the deadlock via `ConfigureAwait(false)`.
- [ ] Comments (RU+EN) explain why each `ConfigureAwait(false)` is needed where it is.

#### Hints (no direct answer)
- To make `ValueTask` show the fast path, have `ReadAsync` check a local `Queue` first; if an event is ready, return `ValueTask<EventEnvelope?>.FromResult(...)` without a single `await`.
- For decompilation, https://sharplab.io is enough: paste your method, pick “Results → C# → (Reduce async)” or “IL”, and compare with `ilspycmd` output.
- Remember: `state == -1` is before the first `await`, `0` is the first resumption point, `-2` is done. Find the `switch` or `goto` in `MoveNext` keyed on these numbers.
- To simulate a UI context, a class storing a `Queue<Action>` and processing it on one thread via `Post` is enough. Install it with `SynchronizationContext.SetSynchronizationContext(...)`.
- To see struct vs class state machine, compare Debug and Release builds: in Release the compiler uses a `struct`, so on synchronous completion there is no allocation.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — reference solution.
// Demonstrates: state machine (via decompilation), ConfigureAwait(false),
// CancellationToken, ValueTask fast path, SemaphoreSlim, Channel<T>.

using System.Collections.Concurrent;
using System.Runtime.CompilerServices;
using System.Threading.Channels;

namespace EventPipeline;

// Event source: frequently completes synchronously so ValueTask shows the fast path.
public interface IEventSource
{
    ValueTask<EventEnvelope?> ReadAsync(CancellationToken ct);
}

public sealed record EventEnvelope(int Id, string Source, DateTimeOffset CreatedAt, ReadOnlyMemory<byte> Payload);

public sealed class InMemoryEventSource : IEventSource
{
    // ConcurrentQueue: thread-safe queue, no lock needed.
    private readonly ConcurrentQueue<EventEnvelope> _queue = new();

    public InMemoryEventSource()
    {
        // Pre-fill so ReadAsync often returns synchronously.
        for (int i = 0; i < 1000; i++)
            _queue.Enqueue(new EventEnvelope(i, "mem", DateTimeOffset.UtcNow, new byte[16]));
    }

    public ValueTask<EventEnvelope?> ReadAsync(CancellationToken ct)
    {
        // FAST PATH: ct not canceled and queue has an item — return without await.
        // The state machine is never even started — no allocation.
        if (!ct.IsCancellationRequested && _queue.TryDequeue(out var evt))
            return ValueTask<EventEnvelope?>.FromResult(evt);

        // SLOW PATH: a real await — here the compiler builds the state machine.
        return SlowReadAsync(this, ct);

        static async ValueTask<EventEnvelope?> SlowReadAsync(InMemoryEventSource self, CancellationToken ct)
        {
            await Task.Delay(5, ct).ConfigureAwait(false);
            if (ct.IsCancellationRequested) return null;
            return self._queue.TryDequeue(out var evt) ? evt : null;
        }
    }
}

public sealed class EventPipeline
{
    private long _processedCount;                  // atomic via Interlocked
    private readonly SemaphoreSlim _flushGate = new(1, 1); // async lock

    public async Task RunAsync(IEventSource source, CancellationToken ct)
    {
        // Bounded + Wait = back-pressure: producer awaits when full.
        var channel = Channel.CreateBounded<EventEnvelope>(new BoundedChannelOptions(64)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = false,
            SingleWriter = true,
        });

        var producer = ProduceAsync(source, channel.Writer, ct);
        var consumerA = ConsumeAsync(channel.Reader, "A", ct);
        var consumerB = ConsumeAsync(channel.Reader, "B", ct);
        var flusher = FlushLoopAsync(ct);

        try
        {
            await producer.ConfigureAwait(false); // wait for source end
        }
        finally
        {
            // CRITICAL: without TryComplete, consumers hang in WaitToReadAsync.
            channel.Writer.TryComplete();
        }

        await Task.WhenAll(consumerA, consumerB).ConfigureAwait(false);

        await flusher.ConfigureAwait(false);
        await FlushAsync(ct).ConfigureAwait(false);
    }

    private static async Task ProduceAsync(IEventSource source, ChannelWriter<EventEnvelope> writer, CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            ct.ThrowIfCancellationRequested();
            // ReadAsync returns ValueTask: no Task allocation on the fast path.
            var evt = await source.ReadAsync(ct).ConfigureAwait(false);
            if (evt is null) break;

            // WriteAsync applies back-pressure when the buffer is full.
            await writer.WriteAsync(evt.Value, ct).ConfigureAwait(false);
        }
    }

    private async Task ConsumeAsync(ChannelReader<EventEnvelope> reader, string id, CancellationToken ct)
    {
        try
        {
            // Standard consume loop: WaitToReadAsync + TryRead.
            while (await reader.WaitToReadAsync(ct).ConfigureAwait(false))
            {
                while (reader.TryRead(out var evt))
                {
                    try
                    {
                        await ProcessAsync(evt, ct).ConfigureAwait(false);
                        // THREAD-SAFE accumulation without a lock.
                        Interlocked.Increment(ref _processedCount);
                        Console.WriteLine($"[{id}] thr={Environment.CurrentManagedThreadId} processed id={evt.Id}");
                    }
                    catch (OperationCanceledException) { throw; }
                    catch (Exception ex)
                    {
                        Console.WriteLine($"[{id}] item {evt.Id} failed: {ex.Message}");
                    }
                }
            }
        }
        catch (ChannelClosedException) { /* normal exit */ }
    }

    private static async Task ProcessAsync(EventEnvelope evt, CancellationToken ct)
    {
        // Simulated I/O; ct must be threaded inside.
        await Task.Delay(2, ct).ConfigureAwait(false);
    }

    private async Task FlushAsync(CancellationToken ct)
    {
        // Async lock: NOT lock/Monitor, but SemaphoreSlim.WaitAsync.
        await _flushGate.WaitAsync(ct).ConfigureAwait(false);
        try
        {
            long snapshot = Interlocked.Exchange(ref _processedCount, 0);
            Console.WriteLine($"[flush] saved {snapshot} events at {DateTimeOffset.UtcNow:O}");
        }
        finally
        {
            _flushGate.Release(); // always in finally
        }
    }

    private async Task FlushLoopAsync(CancellationToken ct)
    {
        try
        {
            while (!ct.IsCancellationRequested)
            {
                await Task.Delay(TimeSpan.FromMilliseconds(500), ct).ConfigureAwait(false);
                await FlushAsync(ct).ConfigureAwait(false);
            }
        }
        catch (OperationCanceledException) { /* expected on cancel */ }
    }
}

// Entry point: top-level statements. No synchronization context —
// ConfigureAwait(false) is optional here, but we kept it in the library.
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
var pipeline = new EventPipeline();
var source = new InMemoryEventSource();
try
{
    await pipeline.RunAsync(source, cts.Token);
}
catch (OperationCanceledException) { }
Console.WriteLine("done");
```

Walk-through, line by line. `IEventSource.ReadAsync` returns `ValueTask<EventEnvelope?>` rather than `Task<EventEnvelope?>`: per the lesson, `ValueTask` is preferable when a method frequently completes synchronously (fast path) — in this implementation, as long as the `ConcurrentQueue` has items, the method never reaches an `await`, so the compiler does not start the state machine and there is no allocation. Only when the queue is empty does `SlowReadAsync` run, where a real `await Task.Delay(5, ct)` lives — that is where the compiler generates a `struct` state machine with `state`, `builder`, and the `self`/`ct` locals. `EventPipeline.RunAsync` creates a `Channel.CreateBounded<EventEnvelope>(64)` with `FullMode.Wait`: this is the back-pressure from the lesson — the producer does not blow up memory, it awaits inside `WriteAsync` when the buffer is full. After every `await` in the library there is a `ConfigureAwait(false)`: per the lesson, library code must not capture the caller’s context, otherwise a UI app deadlocks and ASP.NET Core wastes a thread. The producer calls `channel.Writer.TryComplete()` in a `finally` — without it, `ConsumeAsync` hangs in `WaitToReadAsync` forever (a direct mistake from the lesson’s “Common Mistakes”). `processedCount` is modified only via `Interlocked.Increment`/`Exchange`: the lesson states that `await` by itself does not make code thread-safe (a continuation may run on another thread), and `lock` over `await` is forbidden — hence `Interlocked` for the counter and `SemaphoreSlim` for the critical section. `FlushAsync` takes `_flushGate.WaitAsync(ct)` in a `try` and `Release()` in `finally`: this is the async analogue of `lock`, the only correct form across an `await` boundary. The `CancellationToken` is forwarded into **every** awaited call — `Task.Delay`, `WriteAsync`, `WaitToReadAsync`, `WaitAsync`: the lesson says not to poll `IsCancellationRequested` between awaits; pass the token into the operation so cancellation is instant. In `Program.cs` (top-level statements) `SynchronizationContext.Current == null`, so `ConfigureAwait(false)` is optional — but it is mandatory in the library for the future. Decompilation (step 7) shows that in Release the compiler uses a `struct` state machine (not a class as in Debug), which explains the absence of allocations on synchronous completion. If you run `DemonstrateDeadlock` with a custom `SynchronizationContext` and inner `ConfigureAwait(true)`, `.Result` hangs: the thread waits for the task, the task waits to re-enter the same thread (the classic trap from the lesson); switching to `ConfigureAwait(false)` breaks the cycle.

#### Going deeper (bonus)
1. **Pooling ValueTask.** Annotate the hot-path method with `[AsyncMethodBuilder(typeof(PoolingAsyncValueTaskMethodBuilder<>))]` and compare allocation counts (via `dotnet-counters` or `dotnet-trace`) before and after. Explain why pooling is only safe when you obey “never await twice, never read `.Result` after `await`”.
2. **Custom awaiter.** Implement your own `awaitable`/`awaiter` (e.g., `YieldToThreadPoolAwaitable`) that schedules the continuation on the thread pool via `UnsafeOnCompleted` without capturing `ExecutionContext`. Compare with `Task.Yield()` and `ConfigureAwait(false)`.
3. **Deadlock lab.** Write a minimal single-thread `SynchronizationContext` and a `DeadlockAsync` method that reproduces a `.Result` hang without `ConfigureAwait(false)`. Add a timeout via `Task.WhenAny(task, Task.Delay(2000))` so the test does not hang CI.
4. **Multi-consumer scaling.** Extend the pipeline to a dynamic number of consumers and measure throughput at 1, 2, 4, 8 consumers. Explain why `Channel<T>` with `SingleReader=false` is safe, and why adding consumers past the core count stops helping.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `EventPipeline` собирается под .NET 8 / C# 12 без error-предупреждений.
- [ ] В `Program.cs` — top-level statements и file-scoped namespace.
- [ ] `IEventSource.ReadAsync` возвращает `ValueTask<EventEnvelope?>` с fast-path веткой.
- [ ] Pipeline на `Channel.CreateBounded` с `FullMode.Wait`.
- [ ] Один продюсер, ≥2 потребителя; `TryComplete` в `finally`.
- [ ] Потребители обрабатывают `ChannelClosedException`.
- [ ] `processedCount` через `Interlocked`; нет `lock` поверх `await`.
- [ ] `FlushAsync` через `SemaphoreSlim` с `Release()` в `finally`.
- [ ] `CancellationToken` проброшен во все `await`-уемые операции.
- [ ] `.ConfigureAwait(false)` после каждого `await` в библиотеке.
- [ ] `decompiled.txt` с аннотациями `state` для ≥2 точек `await`.
- [ ] `dotnet run -c Release` завершается по отмене без зависания.
- [ ] (Бонус) `DemonstrateDeadlock` воспроизводит и фиксит дедлок.
- [ ] The `EventPipeline` project builds under .NET 8 / C# 12 with no error-level warnings.
- [ ] Top-level statements and a file-scoped namespace are used in `Program.cs`.
- [ ] `IEventSource.ReadAsync` returns `ValueTask<EventEnvelope?>` with a fast-path branch.
- [ ] The pipeline uses `Channel.CreateBounded` with `FullMode.Wait`.
- [ ] One producer, ≥2 consumers; `TryComplete` in `finally`.
- [ ] Consumers handle `ChannelClosedException`.
- [ ] `processedCount` via `Interlocked`; no `lock` over `await`.
- [ ] `FlushAsync` via `SemaphoreSlim` with `Release()` in `finally`.
- [ ] `CancellationToken` is forwarded into every awaited operation.
- [ ] `.ConfigureAwait(false)` after every `await` in the library.
- [ ] `decompiled.txt` with `state` annotations for ≥2 `await` points.
- [ ] `dotnet run -c Release` shuts down on cancellation without hanging.
- [ ] (Bonus) `DemonstrateDeadlock` reproduces and fixes the deadlock.

#### Ресурсы / Resources
- [Microsoft Learn — async/await scenarios](https://learn.microsoft.com/dotnet/csharp/asynchronous-programming/async-scenarios)
- [Microsoft Learn — Async in depth (Task-based async)](https://learn.microsoft.com/dotnet/standard/async-in-depth)
- [Stephen Toub — «Async Await and the Generated StateMachine»](https://devblogs.microsoft.com/dotnet/async-await-and-the-generated-statemachine/)
- [Stephen Toub — ConfigureAwait FAQ](https://devblogs.microsoft.com/dotnet/configureawait-faq/)
- [sharplab.io — inspect the generated MoveNext](https://sharplab.io)
- [Microsoft Learn — Channel<T> overview](https://learn.microsoft.com/dotnet/api/system.threading.channels.channel-2)
- [Stephen Toub — «Understanding the Whys, Whats, and Whens of ValueTask»](https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/)

---

[← К уроку M09-L03](lesson-M09-L03-async-await-statemachine.md) | [⬆ К модулю M09](../README.md) | [Следующее ДЗ →](homework-M09-L04-task-run-cpu-vs-io.md)
