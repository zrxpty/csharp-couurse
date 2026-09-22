---
[← К уроку M11-L10](lesson-M11-L10-threadpool-longrunning.md) | [⬆ К модулю M11](../README.md) | [Следующее ДЗ →](none)
---

### Домашнее задание M11-L10: ThreadPool настройки, TaskCreationOptions.LongRunning (Optional) / Homework M11-L10: ThreadPool settings, TaskCreationOptions.LongRunning (Optional)

**Урок / Lesson:** M11-L10
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) На практике ощутить поведение `ThreadPool` — измерить starvation, вызванный блокирующими вызовами, научиться поднимать минимум через `SetMinThreads` только под измеренный пик, применять `TaskCreationOptions.LongRunning` исключительно для долгой синхронной работы, строить потокобезопасный producer/consumer на `Channel<T>` с корректной отменой и `ConfigureAwait(false)`, а также избегать типовых ловушек (`Task.Result`, `lock` через `await`, `LongRunning` на async-методах). (EN) Get hands-on with `ThreadPool` behaviour — measure starvation caused by blocking calls, learn to raise the minimum via `SetMinThreads` only for a measured peak, apply `TaskCreationOptions.LongRunning` strictly for long synchronous work, build a thread-safe producer/consumer on `Channel<T>` with correct cancellation and `ConfigureAwait(false)`, and avoid the standard traps (`Task.Result`, `lock` across `await`, `LongRunning` on async methods).

#### Связь с уроком / Connection to the lesson
(RU) Урок M11-L10 объясняет, как `ThreadPool` растёт от минимума к максимуму, почему блокирующие вызовы на пул-потоках приводят к starvation, и в каких редких случаях оправданы `SetMinThreads` и `TaskCreationOptions.LongRunning`. ДЗ закрепляет все эти концепции на измеримом коде: вы сами увидите разницу во времени между блокирующим и асинхронным вариантами, построите pipeline на `Channel<T>` и проверите, что `LongRunning` не помогает async-коду.
(EN) Lesson M11-L10 explains how the `ThreadPool` grows from the minimum to the maximum, why blocking calls on pool threads cause starvation, and in which rare cases `SetMinThreads` and `TaskCreationOptions.LongRunning` are justified. This homework fixes all those concepts in measurable code: you will see the time difference between the blocking and async variants yourself, build a `Channel<T>` pipeline, and verify that `LongRunning` does not help async code.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы поддерживаете фоновый сервис, который читает «тяжёлые» порции данных из внешнего источника (файловая система, очередь сообщений, медленный HTTP-endpoint), обрабатывает их и складывает результат в общий поток для консьюмеров. В коде уже есть «успешный» вариант, который почему-то периодически зависает под нагрузкой: таймеры не стреляют, логи не пишутся, метрики не обновляются, а CPU при этом почти простаивает. Анализ показывает классическую картину — **thread-pool starvation**: несколько рабочих потоков заблокированы в синхронных вызовах `Task.Result`/`.Wait`/`Thread.Sleep`, пул не может быстро нарастить штат (новый поток раз в ~0.5 с), и очередь задач копится. В этом задании вы пройдёте полный цикл «починки»: измерите проблему, уберёте блокировки, при необходимости аккуратно поднимете минимум пула, вынесете долгий синхронный продьюсер в `LongRunning`-задачу, построите потокобезопасный pipeline на `Channel<T>`, добавите корректную кооперативную отмену и оформите код так, чтобы его можно было безопасно вызывать из библиотеки (с `ConfigureAwait(false)`). Вы не просто напишете работающий код — вы объясните, **почему** каждое изменение устраняет именно причину, а не маскирует симптом. Дополнительно вы измерите, что `SetMinThreads`, поднятый «про запас», не ускоряет, а иногда и замедляет код из-за лишних переключений контекста, и убедитесь, что `LongRunning` на async-методе даёт лишний поток без выгоды.

#### Что нужно сделать (пошагово)
1. Создайте консольный проект: `dotnet new console -n M11.L10.Homework -o M11.L10.Homework`, перейдите в папку, убедитесь, что `TargetFramework` — `net8.0`, а `LangVersion` — `latest` (C# 12). Добавьте в `.csproj` явное `<Nullable>enable</Nullable>` и `<ImplicitUsings>enable</ImplicitUsings>`.
2. Реализуйте класс `StarvationBench` с двумя методами: `RunBlockingAsync(int count, int delayMs)` — запускает `count` задач через `Task.Run(() => Thread.Sleep(delayMs))` и ждёт их все; `RunAsyncNative(int count, int delayMs)` — делает то же через `Task.Delay(delayMs)`. Замерьте `Stopwatch`-время обоих при `count=50, delayMs=500`. Вывод должен быть вида `Blocking: X ms, Native async: Y ms`. Ожидается, что нативный async вариант заметно быстрее и стабильнее, потому что не держит потоки.
3. Добавьте метод `PrintPoolState(string label)`, который печатает `ThreadPool.GetMinThreads`, `ThreadPool.GetMaxThreads`, `ThreadPool.GetAvailableThreads`. Вызывайте его до/после бенчмарка, чтобы видеть, как блокирующий вариант «съедает» доступные потоки.
4. Реализуйте `PoolTuner.ConfigureIfNeeded(int worker, int io)` — обёртку над `ThreadPool.SetMinThreads`, которая: читает текущий минимум, берёт `Math.Max` с желаемым (чтобы не понижать), вызывает `SetMinThreads`, проверяет возвращаемое `bool` и логирует результат. Никаких вызовов `SetMaxThreads` — максимум трогать опасно.
5. Реализуйте `LongRunningProducer` — метод, который в цикле опрашивает «внешний ресурс» (имитация через `Thread.Sleep(20)`), и пишет строки в `Channel<string>`. Используйте `Task.Factory.StartNew(..., TaskCreationOptions.LongRunning | TaskCreationOptions.DenyChildAttach, TaskScheduler.Default)`. Внутри — синхронный цикл с `ct.ThrowIfCancellationRequested()` на каждой итерации, затем `await channel.Writer.WriteAsync(...)`. Завершение — `channel.Writer.Complete()` в `finally`. Обязательно `Unwrap()` внешнюю задачу, потому что лямбда возвращает `async Task`.
6. Реализуйте двух консьюмеров `ConsumeAsync(ChannelReader<string>, string id, CancellationToken)`, которые через `await foreach (var item in reader.ReadAllAsync(ct).ConfigureAwait(false))` обрабатывают элементы. Внутри — `await ProcessAsync(item, ct).ConfigureAwait(false)`. Никаких `.Result`/`.Wait`. Добавьте счётчик обработанных элементов через `Interlocked.Increment` и чтение через `Volatile.Read`.
7. В `Main` (top-level statements) подключите общий `CancellationTokenSource`, вызовите `PoolTuner.ConfigureIfNeeded(32, 32)` один раз на старте, запустите бенчмарк starvation, затем pipeline. Через 5 секунд отменяйте всё через `cts.Cancel()`. Ловите `OperationCanceledException` отдельно и печатайте `Cancelled`. В конце печатайте `Processed total: N`.
8. Сравните два варианта продьюсера: один с `TaskCreationOptions.LongRunning`, второй — без него (через `Task.Run(async () => {...})`). Замерьте время работы pipeline и количество потоков через `Process.GetCurrentProcess().Threads.Count` в середине работы. Запишите наблюдения в комментарий в файле `OBSERVATIONS.md`.
9. Сборка и запуск: `dotnet build`, затем `dotnet run --project M11.L10.Homework`. Убедитесь, что нет предупреждений `CA2007`/`CA2008` (где релевантно) и что приложение завершается без висячих потоков за разумное время.
10. Добавьте unit-тест `xUnit`-проект `M11.L10.Homework.Tests` с тестом, который убеждается, что `RunAsyncNative` завершается быстрее `RunBlocking` (с допуском), и тестом, что `PoolTuner.ConfigureIfNeeded` не понижает текущий минимум.

#### Требования к решению
Решение должно компилироваться под .NET 8 / C# 12 без ошибок и без подавления предупреждений через `#pragma`. Используйте top-level statements в `Program.cs`, file-scoped namespaces (`namespace M11.L10.Homework;`), collection expressions и `using`-директивы по необходимости. Все асинхронные методы должны возвращать `Task` или `Task<T>`, никаких `async void` кроме UI-обработчиков (которых здесь нет). Кооперативная отмена обязательна: каждый долгий метод принимает `CancellationToken` и опрашивает его через `ThrowIfCancellationRequested()` или передаёт в `await ...Async(ct)`. Блокирующие вызовы (`Task.Result`, `.Wait`, `Thread.Sleep`) разрешены **только** внутри `LongRunning`-задачи как имитация внешнего блокирующего I/O — это легитимно, потому что такая задача работает на отдельном непул-потоке и не вызывает starvation пула. Везде elsewhere — только `await`. `Channel<T>` должен быть bounded (`Channel.CreateBounded<string>(16)`) для обратного давления. `ConfigureAwait(false)` ставится во всех `await` библиотечного кода. Счётчики — через `Interlocked`, чтение — через `Volatile.Read`. Запрещено держать `lock` через `await`; если понадобится критическая секция в async — `SemaphoreSlim(1,1)` с `await sem.WaitAsync(ct)` и `try/finally sem.Release()`. Код должен быть прочитываемым, с комментариями RU+EN в ключевых местах (как в уроке).

#### Тонкости и подводные камни
- `SetMinThreads` возвращает `false`, если значение отвергнуто (например, меньше текущего минимума или недопустимо для платформы) — обязательно проверяйте возвращаемое значение и логируйте, не «молча» пропускайте.
- Повышать минимум «про запас» вредно: больше потоков → больше памяти (~1 МБ стека на поток), больше переключений контекста, хуже локальность кэша. Поднимайте только под измеренный пик и только когда доказали, что проблема именно в медленном росте пула, а не в блокировках.
- `TaskCreationOptions.LongRunning` — это **подсказка**, а не гарантия; планировщик может её проигнорировать, но на `TaskScheduler.Default` (пул) она обычно приводит к созданию отдельного непул-потока. Не используйте её для async-методов с `await` — получите лишний поток ради кода, который 90% времени ничего не делает. Правило: LongRunning = долгая **синхронная** работа (минуты CPU, poll-циклы с `Thread.Sleep`).
- `Task.Factory.StartNew(async () => {...})` возвращает `Task<Task>` — нужен `.Unwrap()`, иначе вы дождётесь не внутренней работы, а момента запуска лямбды. Альтернатива — `Task.Run(async () => {...})`, который сам разворачивает, но тогда вы потеряете возможность передать `LongRunning`+`DenyChildAttach`+`TaskScheduler.Default` явно.
- `DenyChildAttach` полезен вместе с `LongRunning`: он запрещает вложенным `Task.Run` с `AttachedToParent` «прилипнуть» к вашей задаче, что упрощает рассуждения о времени жизни.
- Никогда не держите `lock` через `await`: monitors thread-affined, await может продолжиться на другом потоке, монитор останется захваченным навсегда → deadlock или corruption. Для async — `SemaphoreSlim(1,1)`.
- `Channel<T>` сам потокобезопасен; `WriteAsync`/`ReadAllAsync` учитывают `CancellationToken`. Не забывайте `channel.Writer.Complete()` в `finally` продьюсера, иначе консьюмеры зависнут в `ReadAllAsync` навсегда.
- `ConfigureAwait(false)` в Console/ASP.NET Core фактически ничего не меняет (контекста и так нет), но это хорошая привычка для библиотечного кода и снижает риск дедлоков при вызове из UI/legacy ASP.NET.
- `Thread.Sleep` внутри `LongRunning` легитимен (имитация блокирующего I/O), но `Thread.Sleep` внутри `Task.Run` без `LongRunning` — это прямой путь к starvation: вы занимаете пул-поток.

#### Критерии приёмки
- [ ] Проект собирается под .NET 8 / C# 12 без ошибок и без `#pragma warning disable`.
- [ ] `StarvationBench` корректно замеряет оба варианта и печатает времена в формате `Blocking: X ms, Native async: Y ms`.
- [ ] `PrintPoolState` печатает min/max/available потоки до и после бенчмарка.
- [ ] `PoolTuner.ConfigureIfNeeded` использует `Math.Max`, проверяет `bool` от `SetMinThreads`, логирует результат, не трогает `SetMaxThreads`.
- [ ] `LongRunningProducer` использует `TaskCreationOptions.LongRunning | TaskCreationOptions.DenyChildAttach` и `TaskScheduler.Default`, вызывает `.Unwrap()`.
- [ ] Внутри продьюсера — `ct.ThrowIfCancellationRequested()` на каждой итерации, `Thread.Sleep` только как имитация блокирующего I/O.
- [ ] `channel.Writer.Complete()` вызывается в `finally` продьюсера.
- [ ] Консьюмеры используют `await foreach ... .ReadAllAsync(ct).ConfigureAwait(false)`, без `.Result`/`.Wait`.
- [ ] Счётчик обработанных — через `Interlocked.Increment`, чтение — через `Volatile.Read`.
- [ ] `Main` использует один `CancellationTokenSource`, отменяет через 5 секунд, ловит `OperationCanceledException`.
- [ ] `OBSERVATIONS.md` содержит замеры LongRunning-vs-без и количество потоков `Process.GetCurrentProcess().Threads.Count`.
- [ ] xUnit-тесты подтверждают, что native async быстрее блокирующего варианта, и что `ConfigureIfNeeded` не понижает минимум.
- [ ] Нет `async void`, нет `lock` через `await`, нет `LongRunning` на «чистом» async-методе.
- [ ] `ConfigureAwait(false)` присутствует во всех `await` библиотечных методов.
- [ ] Код запускается `dotnet run` и завершается без висячих потоков за разумное время.

#### Подсказки (без прямого ответа)
- Чтобы «почувствовать» starvation, запустите блокирующий вариант с `count=100, delayMs=1000` и параллельно стреляйте таймером `Task.Delay(200).ContinueWith(_ => Console.WriteLine("tick"))` — вы увидите, как таймер отстаёт.
- Для измерения количества потоков в середине работы запустите pipeline в фоновой задаче, а в основной — `await Task.Delay(500)` и снимите `Process.GetCurrentProcess().Threads.Count`.
- Если `SetMinThreads` «не помогает» ускорить блокирующий вариант — это ожидаемо: он маскирует симптом, а причина (блокировка) остаётся. Думайте в первую очередь об устранении `Task.Result`/`.Wait`.
- `Channel.CreateBounded` с маленькой ёмкостью показывает обратное давление: продьюсер начнёт ждать `WriteAsync`, если консьюмеры не успевают.
- Для теста «не понижает минимум» вызовите `GetMinThreads`, сохраните, вызовите `ConfigureIfNeeded(1, 1)`, снова `GetMinThreads` — значения не должны уменьшиться.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — Homework M11-L10: ThreadPool tuning, starvation, LongRunning, Channel<T>.
// C# 12 / .NET 8 — ДЗ M11-L10: настройка пула, starvation, LongRunning, Channel<T>.

using System.Diagnostics;
using System.Threading.Channels;

namespace M11.L10.Homework;

// 1) Pool tuning wrapper — raise the minimum only, never touch the maximum.
// 1) Обёртка настройки пула — повышаем только минимум, максимум не трогаем.
public static class PoolTuner
{
    public static void ConfigureIfNeeded(int desiredWorker, int desiredIo)
    {
        ThreadPool.GetMinThreads(out int curWorker, out int curIo);
        Console.WriteLine($"Min before: worker={curWorker}, io={curIo}");
        // Min до: worker=..., io=...

        // Math.Max: never lower the current minimum even if the caller asks for less.
        // Math.Max: не понижаем текущий минимум, даже если вызывающий просит меньше.
        int newWorker = Math.Max(desiredWorker, curWorker);
        int newIo = Math.Max(desiredIo, curIo);

        bool ok = ThreadPool.SetMinThreads(newWorker, newIo);
        if (!ok)
            Console.WriteLine("SetMinThreads rejected the value / SetMinThreads отклонил значение");

        ThreadPool.GetMinThreads(out int afterWorker, out int afterIo);
        Console.WriteLine($"Min after:  worker={afterWorker}, io={afterIo}");
    }

    public static void PrintPoolState(string label)
    {
        ThreadPool.GetMinThreads(out int minW, out int minIo);
        ThreadPool.GetMaxThreads(out int maxW, out int maxIo);
        ThreadPool.GetAvailableThreads(out int avW, out int avIo);
        Console.WriteLine($"[{label}] min=({minW},{minIo}) max=({maxW},{maxIo}) avail=({avW},{avIo})");
    }
}

// 2) Starvation benchmark: blocking vs native async. The blocking variant eats pool threads.
// 2) Бенчмарк starvation: блокирующий vs нативный async. Блокирующий вариант съедает пул-потоки.
public static class StarvationBench
{
    public static async Task<(long blocking, long native)> RunAsync(int count, int delayMs)
    {
        var sw = Stopwatch.StartNew();
        // BAD: each Task.Run holds a pool thread for the whole delay → starvation under load.
        // ПЛОХО: каждый Task.Run держит пул-поток всё время задержки → starvation под нагрузкой.
        var blocking = Enumerable.Range(0, count)
            .Select(_ => Task.Run(() => Thread.Sleep(delayMs)))
            .ToArray();
        await Task.WhenAll(blocking);
        sw.Stop();
        long blockingMs = sw.ElapsedMilliseconds;

        sw.Restart();
        // GOOD: Task.Delay releases the thread during the wait → no pool thread occupied.
        // ХОРОШО: Task.Delay отпускает поток на время ожидания → ни один пул-поток не занят.
        var native = Enumerable.Range(0, count).Select(_ => Task.Delay(delayMs)).ToArray();
        await Task.WhenAll(native);
        sw.Stop();
        long nativeMs = sw.ElapsedMilliseconds;

        Console.WriteLine($"Blocking: {blockingMs} ms, Native async: {nativeMs} ms");
        return (blockingMs, nativeMs);
    }
}

// 3) Long-running synchronous producer on a dedicated non-pooled thread + Channel<T> pipeline.
// 3) Долгий синхронный продьюсер на отдельном непул-потоке + pipeline на Channel<T>.
public static class Pipeline
{
    private static int _processed;

    public static int ProcessedTotal => Volatile.Read(ref _processed);

    public static async Task RunAsync(CancellationToken ct)
    {
        // Bounded channel → back-pressure: producers await when the buffer is full.
        // Ограниченный канал → обратное давление: продьюсеры ждут при переполнении.
        var channel = Channel.CreateBounded<string>(capacity: 16);

        // LongRunning + DenyChildAttach + TaskScheduler.Default → dedicated non-pooled thread.
        // LongRunning + DenyChildAttach + TaskScheduler.Default → отдельный непул-поток.
        Task producer = Task.Factory.StartNew(
            async () =>
            {
                try
                {
                    for (int i = 0; i < 100; i++)
                    {
                        ct.ThrowIfCancellationRequested();
                        Thread.Sleep(20); // blocking I/O simulation, OK on a dedicated thread
                                          // имитация блокирующего I/O, легитимно на отдельном потоке
                        await channel.Writer.WriteAsync($"item-{i}", ct).ConfigureAwait(false);
                    }
                }
                finally
                {
                    channel.Writer.Complete(); // signal consumers to finish / сигнал окончания
                }
            },
            ct,
            TaskCreationOptions.LongRunning | TaskCreationOptions.DenyChildAttach,
            TaskScheduler.Default).Unwrap(); // unwrap Task<Task> → Task / разворачиваем Task<Task> → Task

        Task c1 = ConsumeAsync(channel.Reader, "C1", ct);
        Task c2 = ConsumeAsync(channel.Reader, "C2", ct);

        await Task.WhenAll(producer, c1, c2).ConfigureAwait(false);
    }

    private static async Task ConsumeAsync(ChannelReader<string> reader, string id, CancellationToken ct)
    {
        // ReadAllAsync is cancellation-aware and ends when the writer calls Complete().
        // ReadAllAsync учитывает отмену и завершается, когда продьюсер зовёт Complete().
        await foreach (var item in reader.ReadAllAsync(ct).ConfigureAwait(false))
        {
            await ProcessAsync(item, ct).ConfigureAwait(false); // NEVER lock across await / НИКОГДА lock через await
            Interlocked.Increment(ref _processed);
            Console.WriteLine($"[{id}] processed {item}");
        }
    }

    private static async Task ProcessAsync(string item, CancellationToken ct)
    {
        // Simulate async I/O. NEVER .Result/.Wait — that would re-introduce starvation.
        // Имитация async I/O. НИКОГДА .Result/.Wait — это вернёт starvation.
        await Task.Delay(10, ct).ConfigureAwait(false);
    }
}

// Entry point — top-level statements / точка входа — top-level statements.
using var cts = new CancellationTokenSource();
PoolTuner.PrintPoolState("startup");
PoolTuner.ConfigureIfNeeded(32, 32);
PoolTuner.PrintPoolState("after-tune");

_ = Task.Delay(TimeSpan.FromSeconds(5)).ContinueWith(_ => cts.Cancel());

try
{
    _ = await StarvationBench.RunAsync(count: 50, delayMs: 500);
    await Pipeline.RunAsync(cts.Token).ConfigureAwait(false);
}
catch (OperationCanceledException)
{
    Console.WriteLine("Cancelled / Отменено");
}

PoolTuner.PrintPoolState("done");
Console.WriteLine($"Processed total: {Pipeline.ProcessedTotal}");
```

Разбор по строкам. `PoolTuner.ConfigureIfNeeded` намеренно использует `Math.Max`: даже если вызывающий передаст маленькое значение, мы не понижаем минимум — это соответствует best practice «не ломать чужую настройку». Проверка `bool ok` от `SetMinThreads` обязательна: метод тихо возвращает `false` при отказе, и без лога вы будете думать, что настройка применилась. `PrintPoolState` показывает три тройки (min/max/avail), что позволяет визуально увидеть, как блокирующий бенчмарк «съедает» `avail` и заставляет пул расти от `min` к `max`. `StarvationBench.RunAsync` — классическая демонстрация: `Task.Run(() => Thread.Sleep(delayMs))` держит пул-поток всё время сна, при `count=50` и минимуме по умолчанию (~8–12) пул вынужден наращивать потоки раз в 0.5 с, итоговое время значительно больше `delayMs`; нативный `Task.Delay` отпускает поток, и `WhenAll` завершается за время, близкое к `delayMs`. В `Pipeline.RunAsync` канал bounded на 16 — это обратное давление: если консьюмеры не успевают, `WriteAsync` будет ждать. Продьюсер запущен через `Task.Factory.StartNew` с `LongRunning | DenyChildAttach` и `TaskScheduler.Default`: `LongRunning` даёт отдельный непул-поток (легитимно для синхронного `Thread.Sleep`-цикла), `DenyChildAttach` запрещает вложенным задачам «прилипнуть», `TaskScheduler.Default` явно фиксирует пул-планировщик. `.Unwrap()` критичен: лямбда `async () => {...}` возвращает `Task`, поэтому `StartNew` возвращает `Task<Task>`; без `Unwrap` вы дождётесь не окончания работы, а момента запуска лямбды. `channel.Writer.Complete()` в `finally` гарантирует, что консьюмеры выйдут из `ReadAllAsync` даже при исключении/отмене, иначе они зависнут навсегда. В `ConsumeAsync` — `await foreach` с `ReadAllAsync(ct).ConfigureAwait(false)`: `ConfigureAwait(false)` снимает лишний хоп (в Console контекста и так нет, но это правильная привычка для библиотечного кода). Счётчик через `Interlocked.Increment` и `Volatile.Read` — без `lock`, потому что для одного `int` это эффективнее и безопаснее. В `ProcessAsync` — `Task.Delay(10, ct)`, никаких `.Result`. В точке входа один `CancellationTokenSource`, отмена через 5 секунд, `OperationCanceledException` ловится отдельно — это кооперативная отмена в действии. Все концепции урока применены: starvation измерен и устранён, пул настроен аккуратно, `LongRunning` использован по назначению (синхронная работа), `Channel<T>` даёт потокобезопасный pipeline, `Interlocked`/`Volatile` для счётчика, `ConfigureAwait(false)` в библиотечных методах, никаких `lock` через `await`.

#### Задания на углубление (бонус)
1. Добавьте третий вариант продьюсера — «pure async» через `Task.Run(async () => { await Task.Delay(20); ... })` без `LongRunning`. Сравните количество потоков процесса и время работы pipeline с LongRunning-вариантом. Объясните, почему для имитации блокирующего I/O через `Thread.Sleep` LongRunning выигрывает, а для чистого `Task.Delay` — нет.
2. Реализуйте «плохой» вариант с `Task.Run(() => Thread.Sleep(500))` + `Task.Run(async () => { await someTask; })`, где `someTask` пытается получить результат через `.Result` — и покажите, как таймеры отстают. Запишите таймлайн «tick» в лог, чтобы наглядно увидеть starvation.
3. Добавьте `SemaphoreSlim(1,1)` вокруг разделяемого ресурса (например, записи в файл) в консьюмере и демонстрацию, что `lock` через `await` компилируется, но приводит к проблеме (через отдельный пример с комментарием «НЕ ДЕЛАЙТЕ ТАК»).
4. Измерьте, как `SetMinThreads(512, 512)` «про запас» влияет на пиковую пропускную способность и потребление памяти (`GC.GetTotalMemory`, `Process.Threads.Count`). Сформулируйте, почему «больше — не лучше».

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you maintain a background service that reads "heavy" chunks of data from an external source (file system, message queue, slow HTTP endpoint), processes them, and feeds the result into a shared stream for consumers. The code already has a "working" version that mysteriously hangs under load: timers do not fire, logs are not written, metrics are not updated, while CPU is nearly idle. Analysis reveals the classic picture — **thread-pool starvation**: several worker threads are blocked in synchronous `Task.Result`/`.Wait`/`Thread.Sleep` calls, the pool cannot expand its roster quickly (a new thread only roughly every 0.5 s), and the task queue piles up. In this assignment you will go through the full repair cycle: measure the problem, remove the blocking calls, carefully raise the pool minimum if needed, move the long synchronous producer into a `LongRunning` task, build a thread-safe pipeline on `Channel<T>`, add correct cooperative cancellation, and shape the code so it can be safely invoked from a library (with `ConfigureAwait(false)`). You will not merely write working code — you will explain **why** each change fixes the cause rather than masking the symptom. As a bonus you will measure that `SetMinThreads` raised "just in case" does not speed things up and sometimes even slows the code down because of extra context switches, and you will confirm that `LongRunning` on an async method gives you an extra thread with no benefit.

#### What to do step by step
1. Create a console project: `dotnet new console -n M11.L10.Homework -o M11.L10.Homework`, enter the folder, ensure `TargetFramework` is `net8.0` and `LangVersion` is `latest` (C# 12). Add explicit `<Nullable>enable</Nullable>` and `<ImplicitUsings>enable</ImplicitUsings>` to the `.csproj`.
2. Implement a `StarvationBench` class with two methods: `RunBlockingAsync(int count, int delayMs)` launches `count` tasks via `Task.Run(() => Thread.Sleep(delayMs))` and awaits all of them; `RunAsyncNative(int count, int delayMs)` does the same through `Task.Delay(delayMs)`. Measure both with `Stopwatch` at `count=50, delayMs=500`. The output should look like `Blocking: X ms, Native async: Y ms`. The native async variant is expected to be noticeably faster and more stable because it does not hold threads.
3. Add a `PrintPoolState(string label)` method that prints `ThreadPool.GetMinThreads`, `ThreadPool.GetMaxThreads`, and `ThreadPool.GetAvailableThreads`. Call it before and after the benchmark so you can see how the blocking variant "eats" available threads.
4. Implement `PoolTuner.ConfigureIfNeeded(int worker, int io)` — a wrapper over `ThreadPool.SetMinThreads` that: reads the current minimum, takes `Math.Max` with the desired value (so it never lowers it), calls `SetMinThreads`, checks the returned `bool`, and logs the result. No `SetMaxThreads` calls — touching the maximum is dangerous.
5. Implement `LongRunningProducer` — a method that in a loop polls an "external resource" (simulated with `Thread.Sleep(20)`) and writes strings to a `Channel<string>`. Use `Task.Factory.StartNew(..., TaskCreationOptions.LongRunning | TaskCreationOptions.DenyChildAttach, TaskScheduler.Default)`. Inside — a synchronous loop with `ct.ThrowIfCancellationRequested()` on every iteration, then `await channel.Writer.WriteAsync(...)`. Completion — `channel.Writer.Complete()` in `finally`. You must call `Unwrap()` on the outer task because the lambda returns an `async Task`.
6. Implement two consumers `ConsumeAsync(ChannelReader<string>, string id, CancellationToken)` that process items through `await foreach (var item in reader.ReadAllAsync(ct).ConfigureAwait(false))`. Inside — `await ProcessAsync(item, ct).ConfigureAwait(false)`. No `.Result`/`.Wait`. Add a processed-items counter via `Interlocked.Increment` and read it through `Volatile.Read`.
7. In `Main` (top-level statements) wire up a single `CancellationTokenSource`, call `PoolTuner.ConfigureIfNeeded(32, 32)` once at startup, run the starvation benchmark, then the pipeline. After 5 seconds cancel everything via `cts.Cancel()`. Catch `OperationCanceledException` separately and print `Cancelled`. At the end print `Processed total: N`.
8. Compare two producer variants: one with `TaskCreationOptions.LongRunning`, the other without it (via `Task.Run(async () => {...})`). Measure the pipeline runtime and the thread count via `Process.GetCurrentProcess().Threads.Count` in the middle of the run. Write the observations into a comment in `OBSERVATIONS.md`.
9. Build and run: `dotnet build`, then `dotnet run --project M11.L10.Homework`. Make sure there are no `CA2007`/`CA2008` warnings (where relevant) and that the app exits without dangling threads within a reasonable time.
10. Add an `xUnit` test project `M11.L10.Homework.Tests` with a test asserting that `RunAsyncNative` finishes faster than `RunBlocking` (with a tolerance) and a test that `PoolTuner.ConfigureIfNeeded` does not lower the current minimum.

#### Requirements
The solution must compile under .NET 8 / C# 12 with no errors and without suppressing warnings via `#pragma`. Use top-level statements in `Program.cs`, file-scoped namespaces (`namespace M11.L10.Homework;`), collection expressions, and `using` directives as needed. All async methods must return `Task` or `Task<T>`; no `async void` except UI handlers (none here). Cooperative cancellation is mandatory: every long method takes a `CancellationToken` and either polls it via `ThrowIfCancellationRequested()` or passes it to `await ...Async(ct)`. Blocking calls (`Task.Result`, `.Wait`, `Thread.Sleep`) are allowed **only** inside a `LongRunning` task as a simulation of external blocking I/O — that is legitimate because such a task runs on a dedicated non-pooled thread and does not cause pool starvation. Everywhere else — `await` only. The `Channel<T>` must be bounded (`Channel.CreateBounded<string>(16)`) for back-pressure. `ConfigureAwait(false)` is applied to every `await` in library code. Counters go through `Interlocked`; reads go through `Volatile.Read`. Holding a `lock` across an `await` is forbidden; if you need an async critical section use `SemaphoreSlim(1,1)` with `await sem.WaitAsync(ct)` and `try/finally sem.Release()`. The code must be readable, with RU+EN comments in key spots (as in the lesson).

#### Pitfalls
- `SetMinThreads` returns `false` when the value is rejected (for example, lower than the current minimum or invalid for the platform) — always check the return value and log it; do not "silently" skip it.
- Raising the minimum "just in case" is harmful: more threads → more memory (~1 MB stack per thread), more context switches, worse cache locality. Raise it only for a measured peak and only when you have proven the problem is slow pool growth, not blocking.
- `TaskCreationOptions.LongRunning` is a **hint**, not a guarantee; the scheduler may ignore it, but on `TaskScheduler.Default` (the pool) it typically results in a dedicated non-pooled thread. Do not use it for async methods with `await` — you get an extra thread for code that does nothing 90% of the time. Rule: LongRunning = long **synchronous** work (minutes of CPU, poll loops with `Thread.Sleep`).
- `Task.Factory.StartNew(async () => {...})` returns `Task<Task>` — you need `.Unwrap()`, otherwise you await the moment the lambda starts, not its inner work. The alternative is `Task.Run(async () => {...})` which unwraps automatically, but then you lose the ability to pass `LongRunning`+`DenyChildAttach`+`TaskScheduler.Default` explicitly.
- `DenyChildAttach` is useful alongside `LongRunning`: it forbids nested `Task.Run` with `AttachedToParent` from "sticking" to your task, which simplifies reasoning about lifetime.
- Never hold a `lock` across an `await`: monitors are thread-affined, an await may resume on a different thread, the monitor stays held forever → deadlock or corruption. For async use `SemaphoreSlim(1,1)`.
- `Channel<T>` is itself thread-safe; `WriteAsync`/`ReadAllAsync` honor the `CancellationToken`. Do not forget `channel.Writer.Complete()` in the producer's `finally`, otherwise consumers will hang in `ReadAllAsync` forever.
- `ConfigureAwait(false)` in Console/ASP.NET Core effectively changes nothing (there is no context anyway), but it is a good habit for library code and reduces deadlock risk when called from UI/legacy ASP.NET.
- `Thread.Sleep` inside a `LongRunning` task is legitimate (a simulation of blocking I/O), but `Thread.Sleep` inside a plain `Task.Run` without `LongRunning` is a direct path to starvation: you occupy a pool thread.

#### Acceptance criteria
- [ ] The project compiles under .NET 8 / C# 12 with no errors and no `#pragma warning disable`.
- [ ] `StarvationBench` measures both variants correctly and prints the times as `Blocking: X ms, Native async: Y ms`.
- [ ] `PrintPoolState` prints min/max/available threads before and after the benchmark.
- [ ] `PoolTuner.ConfigureIfNeeded` uses `Math.Max`, checks the `bool` from `SetMinThreads`, logs the result, and never calls `SetMaxThreads`.
- [ ] `LongRunningProducer` uses `TaskCreationOptions.LongRunning | TaskCreationOptions.DenyChildAttach` and `TaskScheduler.Default`, and calls `.Unwrap()`.
- [ ] Inside the producer there is `ct.ThrowIfCancellationRequested()` on every iteration; `Thread.Sleep` is only a blocking-I/O simulation.
- [ ] `channel.Writer.Complete()` is called in the producer's `finally`.
- [ ] Consumers use `await foreach ... .ReadAllAsync(ct).ConfigureAwait(false)`, with no `.Result`/`.Wait`.
- [ ] The processed counter uses `Interlocked.Increment`; reads use `Volatile.Read`.
- [ ] `Main` uses a single `CancellationTokenSource`, cancels after 5 seconds, and catches `OperationCanceledException`.
- [ ] `OBSERVATIONS.md` contains the LongRunning-vs-without measurements and the `Process.GetCurrentProcess().Threads.Count` numbers.
- [ ] xUnit tests confirm that native async is faster than the blocking variant and that `ConfigureIfNeeded` does not lower the minimum.
- [ ] No `async void`, no `lock` across `await`, no `LongRunning` on a "pure" async method.
- [ ] `ConfigureAwait(false)` is present on every `await` in library methods.
- [ ] The code runs with `dotnet run` and exits without dangling threads within a reasonable time.

#### Hints (no direct answer)
- To "feel" starvation, run the blocking variant with `count=100, delayMs=1000` and in parallel fire a timer `Task.Delay(200).ContinueWith(_ => Console.WriteLine("tick"))` — you will see the timer lag behind.
- To measure the thread count mid-run, start the pipeline as a background task, then in the main flow do `await Task.Delay(500)` and read `Process.GetCurrentProcess().Threads.Count`.
- If `SetMinThreads` "does not help" speed up the blocking variant — that is expected: it masks the symptom while the cause (blocking) remains. Think first about removing `Task.Result`/`.Wait`.
- `Channel.CreateBounded` with a small capacity shows back-pressure: the producer will start to await `WriteAsync` if consumers fall behind.
- For the "does not lower the minimum" test, call `GetMinThreads`, save the values, call `ConfigureIfNeeded(1, 1)`, call `GetMinThreads` again — the values must not decrease.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — Homework M11-L10: ThreadPool tuning, starvation, LongRunning, Channel<T>.

using System.Diagnostics;
using System.Threading.Channels;

namespace M11.L10.Homework;

// 1) Pool tuning wrapper — raise the minimum only, never touch the maximum.
public static class PoolTuner
{
    public static void ConfigureIfNeeded(int desiredWorker, int desiredIo)
    {
        ThreadPool.GetMinThreads(out int curWorker, out int curIo);
        Console.WriteLine($"Min before: worker={curWorker}, io={curIo}");

        // Math.Max: never lower the current minimum even if the caller asks for less.
        int newWorker = Math.Max(desiredWorker, curWorker);
        int newIo = Math.Max(desiredIo, curIo);

        bool ok = ThreadPool.SetMinThreads(newWorker, newIo);
        if (!ok)
            Console.WriteLine("SetMinThreads rejected the value");

        ThreadPool.GetMinThreads(out int afterWorker, out int afterIo);
        Console.WriteLine($"Min after:  worker={afterWorker}, io={afterIo}");
    }

    public static void PrintPoolState(string label)
    {
        ThreadPool.GetMinThreads(out int minW, out int minIo);
        ThreadPool.GetMaxThreads(out int maxW, out int maxIo);
        ThreadPool.GetAvailableThreads(out int avW, out int avIo);
        Console.WriteLine($"[{label}] min=({minW},{minIo}) max=({maxW},{maxIo}) avail=({avW},{avIo})");
    }
}

// 2) Starvation benchmark: blocking vs native async. The blocking variant eats pool threads.
public static class StarvationBench
{
    public static async Task<(long blocking, long native)> RunAsync(int count, int delayMs)
    {
        var sw = Stopwatch.StartNew();
        // BAD: each Task.Run holds a pool thread for the whole delay → starvation under load.
        var blocking = Enumerable.Range(0, count)
            .Select(_ => Task.Run(() => Thread.Sleep(delayMs)))
            .ToArray();
        await Task.WhenAll(blocking);
        sw.Stop();
        long blockingMs = sw.ElapsedMilliseconds;

        sw.Restart();
        // GOOD: Task.Delay releases the thread during the wait → no pool thread occupied.
        var native = Enumerable.Range(0, count).Select(_ => Task.Delay(delayMs)).ToArray();
        await Task.WhenAll(native);
        sw.Stop();
        long nativeMs = sw.ElapsedMilliseconds;

        Console.WriteLine($"Blocking: {blockingMs} ms, Native async: {nativeMs} ms");
        return (blockingMs, nativeMs);
    }
}

// 3) Long-running synchronous producer on a dedicated non-pooled thread + Channel<T> pipeline.
public static class Pipeline
{
    private static int _processed;

    public static int ProcessedTotal => Volatile.Read(ref _processed);

    public static async Task RunAsync(CancellationToken ct)
    {
        // Bounded channel → back-pressure: producers await when the buffer is full.
        var channel = Channel.CreateBounded<string>(capacity: 16);

        // LongRunning + DenyChildAttach + TaskScheduler.Default → dedicated non-pooled thread.
        Task producer = Task.Factory.StartNew(
            async () =>
            {
                try
                {
                    for (int i = 0; i < 100; i++)
                    {
                        ct.ThrowIfCancellationRequested();
                        Thread.Sleep(20); // blocking I/O simulation, OK on a dedicated thread
                        await channel.Writer.WriteAsync($"item-{i}", ct).ConfigureAwait(false);
                    }
                }
                finally
                {
                    channel.Writer.Complete(); // signal consumers to finish
                }
            },
            ct,
            TaskCreationOptions.LongRunning | TaskCreationOptions.DenyChildAttach,
            TaskScheduler.Default).Unwrap(); // unwrap Task<Task> → Task

        Task c1 = ConsumeAsync(channel.Reader, "C1", ct);
        Task c2 = ConsumeAsync(channel.Reader, "C2", ct);

        await Task.WhenAll(producer, c1, c2).ConfigureAwait(false);
    }

    private static async Task ConsumeAsync(ChannelReader<string> reader, string id, CancellationToken ct)
    {
        // ReadAllAsync is cancellation-aware and ends when the writer calls Complete().
        await foreach (var item in reader.ReadAllAsync(ct).ConfigureAwait(false))
        {
            await ProcessAsync(item, ct).ConfigureAwait(false); // NEVER lock across await
            Interlocked.Increment(ref _processed);
            Console.WriteLine($"[{id}] processed {item}");
        }
    }

    private static async Task ProcessAsync(string item, CancellationToken ct)
    {
        // Simulate async I/O. NEVER .Result/.Wait — that would re-introduce starvation.
        await Task.Delay(10, ct).ConfigureAwait(false);
    }
}

// Entry point — top-level statements.
using var cts = new CancellationTokenSource();
PoolTuner.PrintPoolState("startup");
PoolTuner.ConfigureIfNeeded(32, 32);
PoolTuner.PrintPoolState("after-tune");

_ = Task.Delay(TimeSpan.FromSeconds(5)).ContinueWith(_ => cts.Cancel());

try
{
    _ = await StarvationBench.RunAsync(count: 50, delayMs: 500);
    await Pipeline.RunAsync(cts.Token).ConfigureAwait(false);
}
catch (OperationCanceledException)
{
    Console.WriteLine("Cancelled");
}

PoolTuner.PrintPoolState("done");
Console.WriteLine($"Processed total: {Pipeline.ProcessedTotal}");
```

Walk-through. `PoolTuner.ConfigureIfNeeded` deliberately uses `Math.Max`: even if the caller passes a small value, we never lower the minimum — this matches the "do not break someone else's tuning" best practice. Checking the `bool ok` from `SetMinThreads` is mandatory: the method silently returns `false` on rejection, and without logging you will think the tuning applied. `PrintPoolState` shows three triples (min/max/avail), letting you visually see how the blocking benchmark "eats" `avail` and forces the pool to grow from `min` toward `max`. `StarvationBench.RunAsync` is the classic demonstration: `Task.Run(() => Thread.Sleep(delayMs))` holds a pool thread for the whole sleep; with `count=50` and a default minimum (~8–12) the pool must add threads at roughly one per 0.5 s, so the total time is well above `delayMs`; the native `Task.Delay` releases the thread, and `WhenAll` completes in a time close to `delayMs`. In `Pipeline.RunAsync` the channel is bounded at 16 — that is back-pressure: if consumers fall behind, `WriteAsync` will await. The producer is launched via `Task.Factory.StartNew` with `LongRunning | DenyChildAttach` and `TaskScheduler.Default`: `LongRunning` yields a dedicated non-pooled thread (legitimate for a synchronous `Thread.Sleep` loop), `DenyChildAttach` forbids nested tasks from attaching, and `TaskScheduler.Default` pins the pool scheduler explicitly. `.Unwrap()` is critical: the `async () => {...}` lambda returns a `Task`, so `StartNew` returns `Task<Task>`; without `Unwrap` you await the lambda's start, not its completion. `channel.Writer.Complete()` in `finally` guarantees consumers exit `ReadAllAsync` even on exception/cancellation, otherwise they hang forever. In `ConsumeAsync` — `await foreach` with `ReadAllAsync(ct).ConfigureAwait(false)`: `ConfigureAwait(false)` removes an extra hop (in Console there is no context anyway, but it is the right habit for library code). The counter uses `Interlocked.Increment` and `Volatile.Read` — no `lock`, because for a single `int` that is both more efficient and safe. In `ProcessAsync` — `Task.Delay(10, ct)`, no `.Result`. The entry point uses a single `CancellationTokenSource`, cancellation after 5 seconds, `OperationCanceledException` caught separately — cooperative cancellation in action. Every lesson concept is applied: starvation is measured and removed, the pool is tuned carefully, `LongRunning` is used for its true purpose (synchronous work), `Channel<T>` gives a thread-safe pipeline, `Interlocked`/`Volatile` for the counter, `ConfigureAwait(false)` in library methods, no `lock` across `await`.

#### Going deeper (bonus)
1. Add a third producer variant — "pure async" via `Task.Run(async () => { await Task.Delay(20); ... })` without `LongRunning`. Compare the process thread count and pipeline runtime against the LongRunning variant. Explain why for a `Thread.Sleep`-based blocking-I/O simulation LongRunning wins, while for a pure `Task.Delay` it does not.
2. Implement a "bad" variant with `Task.Run(() => Thread.Sleep(500))` plus a `Task.Run(async () => { await someTask; })` where `someTask` tries to read the result through `.Result` — and show how timers lag. Log a "tick" timeline to visualize starvation.
3. Add a `SemaphoreSlim(1,1)` around a shared resource (for example, file writes) in the consumer, and a separate demonstration that `lock` across `await` compiles but leads to a problem (with a "DO NOT DO THIS" comment).
4. Measure how `SetMinThreads(512, 512)` "just in case" affects peak throughput and memory consumption (`GC.GetTotalMemory`, `Process.Threads.Count`). Articulate why "more is not better".

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `M11.L10.Homework` собирается под .NET 8 / C# 12. / The `M11.L10.Homework` project compiles under .NET 8 / C# 12.
- [ ] `StarvationBench` печатает времена обоих вариантов. / `StarvationBench` prints both variants' times.
- [ ] `PrintPoolState` вызывается до и после бенчмарка. / `PrintPoolState` is called before and after the benchmark.
- [ ] `PoolTuner.ConfigureIfNeeded` проверяет `bool` и использует `Math.Max`. / `PoolTuner.ConfigureIfNeeded` checks the `bool` and uses `Math.Max`.
- [ ] `LongRunningProducer` использует `LongRunning | DenyChildAttach` и `.Unwrap()`. / `LongRunningProducer` uses `LongRunning | DenyChildAttach` and `.Unwrap()`.
- [ ] `channel.Writer.Complete()` в `finally`. / `channel.Writer.Complete()` in `finally`.
- [ ] Консьюмеры используют `await foreach` + `ConfigureAwait(false)`. / Consumers use `await foreach` + `ConfigureAwait(false)`.
- [ ] Счётчик через `Interlocked`/`Volatile`. / Counter via `Interlocked`/`Volatile`.
- [ ] Один `CancellationTokenSource`, отмена через 5 секунд. / Single `CancellationTokenSource`, cancel after 5 seconds.
- [ ] `OBSERVATIONS.md` с замерами LongRunning-vs-без. / `OBSERVATIONS.md` with LongRunning-vs-without measurements.
- [ ] xUnit-тесты проходят. / xUnit tests pass.
- [ ] Нет `async void`, `lock` через `await`, `LongRunning` на async. / No `async void`, `lock` across `await`, `LongRunning` on async.

#### Ресурсы / Resources
- [Microsoft Learn — ThreadPool — https://learn.microsoft.com/dotnet/api/system.threading.threadpool](https://learn.microsoft.com/dotnet/api/system.threading.threadpool)
- [Microsoft Learn — ThreadPool.SetMinThreads — https://learn.microsoft.com/dotnet/api/system.threading.threadpool.setminthreads](https://learn.microsoft.com/dotnet/api/system.threading.threadpool.setminthreads)
- [Microsoft Learn — TaskCreationOptions — https://learn.microsoft.com/dotnet/api/system.threading.tasks.taskcreationoptions](https://learn.microsoft.com/dotnet/api/system.threading.tasks.taskcreationoptions)
- [Microsoft Learn — Channel<T> — https://learn.microsoft.com/dotnet/api/system.threading.channels.channel-1](https://learn.microsoft.com/dotnet/api/system.threading.channels.channel-1)
- [Stephen Toub — ThreadPool starvation — https://devblogs.microsoft.com/dotnet/?s=threadpool+starvation](https://devblogs.microsoft.com/dotnet/?s=threadpool+starvation)
- [Microsoft Learn — ConfigureAwait — https://learn.microsoft.com/dotnet/api/system.threading.tasks.task.configureawait](https://learn.microsoft.com/dotnet/api/system.threading.tasks.task.configureawait)
