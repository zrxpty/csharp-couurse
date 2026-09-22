---
[← К уроку M09-L10](lesson-M09-L10-async-streams.md) | [⬆ К модулю M09](../README.md) | [Следующее ДЗ →]()
---

### Домашнее задание M09-L10: Async streams IAsyncEnumerable / Homework M09-L10: Async streams IAsyncEnumerable

**Урок / Lesson:** M09-L10
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться производить и потреблять асинхронные потоки `IAsyncEnumerable<T>` с корректной отменой, `ConfigureAwait(false)` и потокобезопасной многопоточной обработкой через `System.Threading.Channels`; закрепить best practices и избежать классических антипаттернов (deadlock на `.Result`, race condition при раздельном чтении, потеря токена отмены). (EN) Learn to produce and consume `IAsyncEnumerable<T>` async streams with correct cancellation, `ConfigureAwait(false)` and thread-safe multi-threaded processing through `System.Threading.Channels`; internalize best practices and avoid classic anti-patterns (`.Result` deadlock, race conditions on shared enumeration, lost cancellation tokens).

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит четыре строительных блока async streams: интерфейс `IAsyncEnumerable<T>`, цикл `await foreach`, конструкцию `async yield return` и атрибут `[EnumeratorCancellation]`, а также показывает, как асинхронные потоки комбинируются с `Channel<T>` для настоящей многопоточной конвейерной обработки. ДЗ закрепляет все четыре блока и concurrency-акценты на реалистичном сценарии телеметрии IoT: вы построите producer-генератор, корректного потребителя, bounded-канал с обратным давлением и продемонстрируете, что раздельное чтение одного потока приводит к `InvalidOperationException`. (EN) The lesson introduces four building blocks of async streams — the `IAsyncEnumerable<T>` interface, the `await foreach` loop, the `async yield return` production form and the `[EnumeratorCancellation]` attribute — and shows how async streams compose with `Channel<T>` for real multi-threaded pipelining. This homework reinforces all four blocks and the concurrency accents on a realistic IoT-telemetry scenario: you will build a producer generator, a correct consumer, a bounded channel with backpressure, and demonstrate that concurrent reads of a single stream raise `InvalidOperationException`.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы — backend-инженер платформы мониторинга промышленных IoT-устройств. Сотни датчиков (температура, давление, вибрация) шлют значения по шине, которая асинхронна по своей природе: каждое чтение — это сетевой или I2C-вызов с задержкой 20–80 мс. Накапливать всё в `List<T>` и отдавать `Task<List<SensorReading>>` — плохая идея: память растёт, первый результат потребитель видит только после полного опроса, отменить долгий опрос нельзя. Идеальный инструмент — `IAsyncEnumerable<T>`: значения летят одно за другим, потребитель обрабатывает их по мере поступления, отмена работает мгновенно.

Однако один датчик — это ещё не система. В реальности несколько сенсоров опрашиваются параллельно, а результаты обрабатывает пул потребителей (запись в БД, агрегация, алерты). `IAsyncEnumerable<T>` сам по себе параллелизма не добавляет: один `await foreach` работает в одном методе. Чтобы получить многопоточный конвейер с обратным давлением, нужно связать несколько producer-задач с `Channel<T>`, а потребителей — с `channel.Reader.ReadAllAsync()`, который возвращает потокобезопасный для нескольких читателей `IAsyncEnumerable<T>`. Именно эту архитектуру — от «голого» async stream до bounded-канала с backpressure — вам и предстоит собрать. Заодно вы на себе почувствуете два классических грабли: блокировку `.Result` на `MoveNextAsync()` и race condition при попытке читать один поток из двух потоков.

#### Что нужно сделать (пошагово)

1. **Создайте проект.** Выполните `dotnet new console -n TelemetryStreams -o TelemetryStreams -f net8.0`, перейдите в папку `cd TelemetryStreams` и откройте `Program.cs`. Убедитесь, что `<LangVersion>` явно 12 или выше: в `.csproj` при необходимости добавьте `<PropertyGroup><LangVersion>latest</LangVersion></PropertyGroup>`. Используйте top-level statements.

2. **Модель данных.** Определите `record SensorReading(int SensorId, DateTimeOffset Ts, double Value, string Kind);`. Типы значений — `double` для температуры/давления, `Kind` — строка-маркер (`"temp"`, `"pressure"`, `"vibration"`).

3. **Producer-генератор `PollSensorAsync`.** Реализуйте метод `static async IAsyncEnumerable<SensorReading> PollSensorAsync(int sensorId, string kind, [EnumeratorCancellation] CancellationToken ct = default)`. Внутри — цикл `while (!ct.IsCancellationRequested)`: имитируйте асинхронное чтение железа через `await Task.Delay(Random.Shared.Next(20, 80), ct).ConfigureAwait(false);`, затем `yield return new SensorReading(sensorId, DateTimeOffset.UtcNow, Random.Shared.NextDouble()*100, kind);`. Между итерациями добавьте `await Task.Delay(150, ct).ConfigureAwait(false);`. Ключевой момент: параметр `ct` обязан быть помечен `[EnumeratorCancellation]`, иначе токен из `WithCancellation` не попадёт в тело метода.

4. **Потребитель `ConsumeOneAsync`.** Реализуйте `static async Task ConsumeOneAsync(CancellationToken ct)`, который внутри создаёт linked `CancellationTokenSource` с `CancelAfter(TimeSpan.FromSeconds(2))` и в `await foreach` читает `PollSensorAsync(1, "temp", cts.Token).WithCancellation(cts.Token).ConfigureAwait(false)`. Каждое значение печатайте `Console.WriteLine($"[one] {r}");`. Оберните цикл в `try/catch (OperationCanceledException)` — отмена по таймауту ожидаема. Запустите и убедитесь, что поток останавливается ровно через 2 секунды, а не «до победного».

5. **Bounded-канал `RunPipelineAsync`.** Создайте `Channel.CreateBounded<SensorReading>(new BoundedChannelOptions(8) { FullMode = BoundedChannelFullMode.Wait, SingleReader = false, SingleWriter = false });`. Запустите 3 producer-задачи через `Task.Run`, каждая из которых в цикле (`for n in 0..9`) вызывает `await channel.Writer.WriteAsync(reading, ct).ConfigureAwait(false)` — `WriteAsync` сам await'ит при переполнении, это и есть backpressure. Когда все producer'ы завершатся, вызовите `channel.Writer.TryComplete()` через `Task.WhenAll(producers).ContinueWith(...)`. Запустите 2 consumer-задачи, каждая из которых в `await foreach (var r in channel.Reader.ReadAllAsync(ct).ConfigureAwait(false))` печатает `Console.WriteLine($"[c{id}] sensor={r.SensorId} val={r.Value:F2} on thread={Environment.CurrentManagedThreadId}");`. Дождитесь всех задач.

6. **Демонстрация антипаттерна.** В отдельном методе `static void DemonstrateRaceCondition(IAsyncEnumerable<int> src)` покажите, что вызов `src.GetAsyncEnumerator()` и одновременный `MoveNextAsync()` из двух `Task.Run` приводит к `InvalidOperationException` («Previous MoveNextAsync request is still in progress»). Ловите исключение и печатайте предупреждение — это наглядная иллюстрация, почему `IAsyncEnumerable` не потокобезопасен и для fan-out нужен `Channel<T>`.

7. **Запуск.** В `Program.cs` соберите всё: вызовите `ConsumeOneAsync(ct)`, затем `RunPipelineAsync(ct)`, затем демонстрацию race condition. Параметр `ct` берите из `CancellationTokenSource.CreateLinkedTokenSource(...)` с общим `CancelAfter(5 sec)` для всей программы. Ожидаемый вывод: ~13 строк `[one]`, затем ~30 строк `[c0]/[c1]` с разными потоками, затем строка `Race condition caught: ...`.

8. **Проверьте.** `dotnet build` без warnings, `dotnet run` — вывод соответствует описанию. Запустите дважды: идемпотентность по структуре вывода обязательна (числа могут отличаться — это нормально, потоки случайны).

#### Требования к решению

- Целевой фреймворк `net8.0`, язык C# 12, top-level statements в `Program.cs`, `ImplicitUsings` включён, `Nullable` включён.
- Producer-метод возвращает именно `IAsyncEnumerable<SensorReading>`, а не `Task<IAsyncEnumerable<...>>` и не `Task<List<...>>` — сохраняется lazy-семантика.
- Параметр `CancellationToken` producer-метода помечен `[EnumeratorCancellation]`; потребитель передаёт токен через `WithCancellation(ct)` и в библиотечном стиле добавляет `.ConfigureAwait(false)`.
- Все `await` внутри producer и внутри `await foreach` сопровождаются `.ConfigureAwait(false)`.
- Многопоточный конвейер реализован только через `System.Threading.Channels`; bounded-канал использует `FullMode = BoundedChannelFullMode.Wait` (никаких `DropWrite` или `DropOldest` — теряем данные).
- Канал корректно закрывается для записи после завершения producer'ов (`TryComplete`), иначе `ReadAllAsync` зависнет навсегда.
- Никакого `.Result` / `.Wait()` на `ValueTask`/`Task` из асинхронных перечислений; никакого `async void`; никакого `lock` вокруг `await`.
- Race-condition-демонстрация действительно ловит `InvalidOperationException` и не падает.
- Код компилируется без warnings (`TreatWarningsAsErrors` опционально, но приветствуется), собирается и запускается на .NET 8 SDK.

#### Тонкости и подводные камни

- **`[EnumeratorCancellation]` обязателен.** Если забыть атрибут, токен из `WithCancellation(ct)` дойдёт только до перечислителя, но не до тела producer-метода: `ct.IsCancellationRequested` внутри `while` будет всегда `false`, и таймаут не сработает. Это самая частая и коварная ошибка — код «работает», но не отменяется.
- **`WithCancellation` против передачи параметром.** В уроке показаны оба пути: можно передать `cts.Token` и как параметр `PollSensorAsync(..., cts.Token)`, и через `.WithCancellation(cts.Token)`. Двойная передача безопасна и даже рекомендована для надёжности — компилятор и runtime «склеят» токены через linked source внутри сгенерированного перечислителя.
- **`ConfigureAwait(false)` в `await foreach`.** Синтаксис специфичный: пишется после `WithCancellation` — `await foreach (var x in src.WithCancellation(ct).ConfigureAwait(false))`. Если перепутать порядок, компилятор не даст — но легко забыть сам вызов и захватить контекст синхронизации в WinForms/WPF.
- **`Task<IAsyncEnumerable<T>>` — ловушка.** Возвращайте `IAsyncEnumerable<T>` напрямую. `Task<IAsyncEnumerable<...>>` добавляет лишний уровень ожидания и ломает lazy-семантику: потребитель сначала ждёт завершения метода, потом уже начинает стримить.
- **Закрытие канала.** Если не вызвать `channel.Writer.TryComplete()` (или `Complete()`) после завершения producer'ов, `ReadAllAsync` никогда не вернётся — consumer'ы зависнут в ожидании «ещё одного» элемента. Используйте `Task.WhenAll(producers).ContinueWith(_ => channel.Writer.TryComplete(), TaskScheduler.Default)`.
- **`BoundedChannelFullMode.Wait` vs `Drop*`.** Только `Wait` даёт честное обратное давление: producer блокируется на `WriteAsync`, пока consumer не освободит место. `DropWrite`/`DropOldest` молча теряют данные — для телеметрии недопустимо.
- **Не читайте один поток из нескольких потоков.** `MoveNextAsync` не реентерабелен: второй вызов, пока первый не завершён, бросает `InvalidOperationException`. Для fan-out — только `Channel<T>` с `ReadAllAsync`, где канал гарантирует доставку каждого сообщения ровно одному читателю.
- **Deadlock на `.Result`.** В уроке показан метод `DangerousBlockingConsume` с `enumerator.MoveNextAsync().AsTask().Result`. Под `SynchronizationContext` (WinForms/WPF) это deadlock; в ASP.NET Core — исчерпание пула потоков при масштабировании. Единственный правильный путь — `await`.
- **`ValueTask` перечислителя.** `MoveNextAsync` возвращает `ValueTask<bool>`, а не `Task<bool>` — это дешевле при синхронном завершении. Не конвертируйте его в `Task` без нужды и не await'ите один и тот же `ValueTask` дважды.

#### Критерии приёмки

- [ ] Проект `TelemetryStreams` создан через `dotnet new console -f net8.0`, собирается без ошибок и warnings.
- [ ] `Program.cs` использует top-level statements C# 12; `SensorReading` — `record`.
- [ ] `PollSensorAsync` возвращает `IAsyncEnumerable<SensorReading>`, использует `yield return`, параметр `ct` помечен `[EnumeratorCancellation]`.
- [ ] Внутри `PollSensorAsync` все `await` сопровождаются `.ConfigureAwait(false)`.
- [ ] `ConsumeOneAsync` использует `await foreach ... .WithCancellation(cts.Token).ConfigureAwait(false)` и `CancelAfter(2s)`; отмена ловится без необработанного исключения.
- [ ] `RunPipelineAsync` создаёт bounded-канал с `FullMode = Wait`, `SingleReader = false`, `SingleWriter = false`.
- [ ] Запускаются 3 producer-задачи через `Task.Run`, пишущие в общий канал через `WriteAsync(..., ct)`.
- [ ] После завершения producer'ов вызывается `channel.Writer.TryComplete()`.
- [ ] 2 consumer-задачи читают через `channel.Reader.ReadAllAsync(ct)` и печатают идентификатор потока.
- [ ] Вывод содержит строки от разных consumer'ов (`[c0]`, `[c1]`) и разные `thread=...` — доказательство многопоточности.
- [ ] `DemonstrateRaceCondition` реально ловит `InvalidOperationException` и не роняет программу.
- [ ] Нигде нет `.Result` / `.Wait()` на асинхронных перечислениях, `async void`, `lock` вокруг `await`.
- [ ] Возврат `Task<IAsyncEnumerable<...>>` или `Task<List<...>>` не используется нигде.
- [ ] `dotnet run` отрабатывает за ≤ 6 секунд и завершается штатно (не зависает).
- [ ] Код идемпотентен по структуре вывода при повторном запуске.

#### Подсказки (без прямого ответа)

- Вспомните餐厅-метафору из урока: официант приносит одну тарелку и ждёт — это и есть `IAsyncEnumerable`. Ваш `PollSensorAsync` — официант.
- Токен отмены проходит длинный путь: `CancellationTokenSource` → `WithCancellation` → перечислитель → `[EnumeratorCancellation]` → тело метода. Если любое звено разорвано, отмена «теряется».
- Для закрытия канала используйте `Task.WhenAll(producers).ContinueWith(_ => channel.Writer.TryComplete(), TaskScheduler.Default)` — это позволит consumer'ам корректно завершить `await foreach`.
- В race-condition-демонстрации создайте `IAsyncEnumerable<int>` (например, `from n in Enumerable.Range(0,1000) select n` приведите через расширение или сделайте простой async-генератор), возьмите один `GetAsyncEnumerator()` и дёрните `MoveNextAsync()` из двух `Task.Run` без await — исключение прилетит быстро.
- `BoundedChannelOptions` с_capacity 8 намеренно маленький — чтобы producer'ы реально упирались в backpressure и await'или на `WriteAsync`. Если Capacity = 1000, вы просто не увидите эффекта.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — TelemetryStreams / Program.cs
// C# 12 / .NET 8 — TelemetryStreams / Program.cs

using System.Threading.Channels;

// Модель данных / Data model
public record SensorReading(int SensorId, DateTimeOffset Ts, double Value, string Kind);

// ====== 1. Producer: async stream генератор с отменой ======
// ====== 1. Producer: async-stream generator with cancellation ======
public static async IAsyncEnumerable<SensorReading> PollSensorAsync(
    int sensorId,
    string kind,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    // ConfigureAwait(false) — не захватываем контекст вызывающего (библиотечный стиль)
    // ConfigureAwait(false) — do not capture caller's context (library style)
    while (!ct.IsCancellationRequested)
    {
        // Имитация асинхронного чтения с шины / I2C
        // Simulating async bus / I2C read
        await Task.Delay(Random.Shared.Next(20, 80), ct).ConfigureAwait(false);

        // lazy: метод замораживается в машине состояний до следующего запроса
        // lazy: method freezes in the state machine until the next pull
        yield return new SensorReading(
            SensorId: sensorId,
            Ts: DateTimeOffset.UtcNow,
            Value: Random.Shared.NextDouble() * 100.0,
            Kind: kind);
    }
}

// ====== 2. Потребитель одного потока с таймаутом ======
// ====== 2. Single-stream consumer with timeout ======
public static async Task ConsumeOneAsync(CancellationToken outer, CancellationTokenSource cts)
{
    // Связываем внешний токен с внутренним таймаутом / Link outer token with inner timeout
    using var linked = CancellationTokenSource.CreateLinkedTokenSource(outer, cts.Token);
    linked.CancelAfter(TimeSpan.FromSeconds(2));

    try
    {
        // Двойная передача токена: параметром И через WithCancellation — надёжно
        // Double token pass: as parameter AND via WithCancellation — robust
        await foreach (var r in PollSensorAsync(sensorId: 1, "temp", linked.Token)
                           .WithCancellation(linked.Token)
                           .ConfigureAwait(false))
        {
            Console.WriteLine($"[one] {r}");
        }
    }
    catch (OperationCanceledException) { /* ожидаемая отмена по таймауту / expected timeout cancel */ }
}

// ====== 3. Producer/Consumer pipeline через bounded Channel ======
// ====== 3. Producer/Consumer pipeline via bounded Channel ======
public static async Task RunPipelineAsync(CancellationToken ct)
{
    // Bounded-канал с backpressure: при переполнении producer ждёт на WriteAsync
    // Bounded channel with backpressure: on overflow the producer awaits in WriteAsync
    var channel = Channel.CreateBounded<SensorReading>(new BoundedChannelOptions(capacity: 8)
    {
        FullMode = BoundedChannelFullMode.Wait,   // не теряем данные / never drop data
        SingleReader = false,                     // несколько потребителей / multiple consumers
        SingleWriter = false                      // несколько производителей / multiple producers
    });

    // --- 3 producer'а параллельно пишут в общий канал ---
    // --- 3 producers write to the shared channel in parallel ---
    var sensors = new[] { (1, "temp"), (2, "pressure"), (3, "vibration") };
    var producers = sensors.Select(s => Task.Run(async () =>
    {
        try
        {
            for (int n = 0; n < 10 && !ct.IsCancellationRequested; n++)
            {
                await Task.Delay(Random.Shared.Next(20, 80), ct).ConfigureAwait(false);
                var reading = new SensorReading(s.Item1, DateTimeOffset.UtcNow,
                                                Random.Shared.NextDouble() * 100.0, s.Item2);
                // WriteAsync сам await'ит при переполнении — это и есть backpressure
                // WriteAsync itself awaits on overflow — that is backpressure
                await channel.Writer.WriteAsync(reading, ct).ConfigureAwait(false);
            }
        }
        catch (OperationCanceledException) { }
    }, ct)).ToArray();

    // Закрываем канал для записи после завершения всех producer'ов
    // Close the channel for writes after all producers finish
    var closeTask = Task.WhenAll(producers).ContinueWith(
        _ => channel.Writer.TryComplete(), TaskScheduler.Default);

    // --- 2 consumer'а читают через потокобезопасный ReadAllAsync ---
    // --- 2 consumers read via the thread-safe ReadAllAsync ---
    var consumers = Enumerable.Range(0, 2).Select(c => Task.Run(async () =>
    {
        try
        {
            // ReadAllAsync возвращает IAsyncEnumerable<T>, безопасный для нескольких readers
            // ReadAllAsync returns IAsyncEnumerable<T>, safe for multiple readers
            await foreach (var r in channel.Reader.ReadAllAsync(ct).ConfigureAwait(false))
            {
                Console.WriteLine(
                    $"[c{c}] sensor={r.SensorId} val={r.Value:F2} kind={r.Kind} thread={Environment.CurrentManagedThreadId}");
            }
        }
        catch (OperationCanceledException) { }
    }, ct)).ToArray();

    await Task.WhenAll(producers);
    await closeTask;
    await Task.WhenAll(consumers);
}

// ====== 4. Демонстрация антипаттерна: race condition на одном перечислителе ======
// ====== 4. Anti-pattern demo: race condition on a single enumerator ======
public static void DemonstrateRaceCondition()
{
    // Простая async-генерация чисел для демонстрации / Simple async number generator for the demo
    static async IAsyncEnumerable<int> NumbersAsync(
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        for (int i = 0; i < 1000; i++) { await Task.Delay(1, ct).ConfigureAwait(false); yield return i; }
    }

    var src = NumbersAsync();
    var enumerator = src.GetAsyncEnumerator();

    // Два потока дёргают MoveNextAsync одного перечислителя — это НЕ потокобезопасно
    // Two threads call MoveNextAsync on one enumerator — this is NOT thread-safe
    var tasks = Enumerable.Range(0, 2).Select(_ => Task.Run(() =>
    {
        try { while (enumerator.MoveNextAsync().AsTask().Result) { /* skip */ } }
        // Внимание: .Result здесь только для ДЕМОНСТРАЦИИ антипаттерна!
        // Note: .Result here is ONLY for the anti-pattern DEMO!
        catch (Exception ex) { Console.WriteLine($"Race condition caught: {ex.GetType().Name}: {ex.Message}"); }
    })).ToArray();

    Task.WaitAll(tasks);
}

// ====== Точка входа / Entry point ======
// (top-level statements)
using var appCts = CancellationTokenSource.CreateLinkedTokenSource();
appCts.CancelAfter(TimeSpan.FromSeconds(5));

await ConsumeOneAsync(appCts.Token, CancellationTokenSource.CreateLinkedTokenSource());
await RunPipelineAsync(appCts.Token);
DemonstrateRaceCondition();
Console.WriteLine("Done.");
```

**Разбор по строкам.** `PollSensorAsync` — ядро задания: `async IAsyncEnumerable<...>` + `yield return` + `[EnumeratorCancellation]`. Атрибут критичен — без него `WithCancellation` бесполезен. `ConfigureAwait(false)` на каждом `await` — библиотечный стандарт из урока. `ConsumeOneAsync` показывает правильную связку `linked token + CancelAfter + WithCancellation + ConfigureAwait(false)` и ловит `OperationCanceledException` — отмена по таймауту ожидаема, не ошибка. `RunPipelineAsync` — концентрат concurrency-акцентов урока: bounded-канал (`FullMode = Wait`) даёт обратное давление, `WriteAsync` await'ит при переполнении, `ReadAllAsync` возвращает потокобезопасный для нескольких readers `IAsyncEnumerable<T>`, а `TryComplete()` после `Task.WhenAll(producers)` критически важен — иначе `ReadAllAsync` зависнет. Два consumer'а печатают `Environment.CurrentManagedThreadId` — вы видите разные ID, доказывая реальную многопоточность. `DemonstrateRaceCondition` намеренно использует `.Result` (единственный разрешённый случай — демонстрация ошибки): два `Task.Run` дёргают `MoveNextAsync` одного перечислителя, и runtime бросает `InvalidOperationException` с сообщением «Previous MoveNextAsync request is still in progress» — наглядная иллюстрация, почему `IAsyncEnumerable` не потокобезопасен и для fan-out нужен `Channel<T>`. Концепции урока, применённые здесь: pull-based lazy generation, корректная отмена через `[EnumeratorCancellation]`, `ConfigureAwait(false)`, bounded channel с backpressure, `ReadAllAsync` для нескольких readers, демонстрация deadlock/race антипаттернов.

#### Задания на углубление (бонус)

1. **Слияние нескольких сенсоров в один поток.** Реализуйте `MergeAsync(IAsyncEnumerable<IAsyncEnumerable<SensorReading>> sources, CancellationToken ct)`, который через `Channel.CreateBounded` и набор producer-задач (по одной на источник) сливает все потоки в один `IAsyncEnumerable<SensorReading>`. Доказательте, что итоговый поток можно читать одним `await foreach`, но источники опрашиваются параллельно.
2. **Кастомный перечислитель без `yield`.** Реализуйте `IAsyncEnumerable<T>` вручную: класс с `GetAsyncEnumerator`, возвращающий объект с `MoveNextAsync`/`Current`/`DisposeAsync`, читающий из `HttpClient`-пагинации (`?page=N`). Сравните с `yield`-версией по читаемости и производительности.
3. **Throttle и batch.** Добавьте оператор-расширение `BufferAsync(this IAsyncEnumerable<T> src, int batchSize, TimeSpan flushInterval)`, который аккумулирует элементы и через `Channel` отдаёт их пачками — полезно для batched-записи в БД.
4. **Отмена с прогрессом.** Свяжите async stream с `IProgress<T>`: каждые 10 элементов сообщайте прогресс во внешний подписчик, а по отмене корректно завершайте `DisposeAsync` перечислителя.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are a backend engineer on an industrial IoT monitoring platform. Hundreds of sensors (temperature, pressure, vibration) push values over a bus that is asynchronous by nature: every read is a network or I2C call with a 20–80 ms latency. Materializing everything into a `List<T>` and returning a `Task<List<SensorReading>>` is a poor idea: memory grows, the consumer sees the first result only after the entire poll completes, and a long poll cannot be cancelled. The ideal tool is `IAsyncEnumerable<T>`: values flow one by one, the consumer processes them as they arrive, and cancellation is instant.

A single sensor, however, is not yet a system. In reality several sensors are polled in parallel, and the results are handled by a pool of consumers (database writes, aggregation, alerts). `IAsyncEnumerable<T>` adds no parallelism by itself: one `await foreach` runs in a single method. To obtain a multi-threaded pipeline with backpressure you must connect several producer tasks to a `Channel<T>`, and the consumers to `channel.Reader.ReadAllAsync()`, which returns an `IAsyncEnumerable<T>` that is thread-safe for multiple readers. This is exactly the architecture you are going to build — from a "bare" async stream up to a bounded channel with backpressure. Along the way you will step on two classic rakes yourself: blocking on `.Result` from `MoveNextAsync()` and a race condition from trying to read a single stream from two threads.

#### What to do step by step

1. **Create the project.** Run `dotnet new console -n TelemetryStreams -o TelemetryStreams -f net8.0`, change into the folder `cd TelemetryStreams` and open `Program.cs`. Make sure `<LangVersion>` is explicitly 12 or higher: add `<PropertyGroup><LangVersion>latest</LangVersion></PropertyGroup>` to the `.csproj` if needed. Use top-level statements.

2. **Data model.** Define `record SensorReading(int SensorId, DateTimeOffset Ts, double Value, string Kind);`. The value type is `double` for temperature/pressure; `Kind` is a marker string (`"temp"`, `"pressure"`, `"vibration"`).

3. **Producer generator `PollSensorAsync`.** Implement `static async IAsyncEnumerable<SensorReading> PollSensorAsync(int sensorId, string kind, [EnumeratorCancellation] CancellationToken ct = default)`. Inside, loop `while (!ct.IsCancellationRequested)`: simulate async hardware access with `await Task.Delay(Random.Shared.Next(20, 80), ct).ConfigureAwait(false);`, then `yield return new SensorReading(sensorId, DateTimeOffset.UtcNow, Random.Shared.NextDouble()*100, kind);`. Add `await Task.Delay(150, ct).ConfigureAwait(false);` between iterations. The key point: the `ct` parameter MUST be marked `[EnumeratorCancellation]`, otherwise the token from `WithCancellation` never reaches the method body.

4. **Consumer `ConsumeOneAsync`.** Implement `static async Task ConsumeOneAsync(CancellationToken ct)` that internally creates a linked `CancellationTokenSource` with `CancelAfter(TimeSpan.FromSeconds(2))` and in `await foreach` reads `PollSensorAsync(1, "temp", cts.Token).WithCancellation(cts.Token).ConfigureAwait(false)`. Print each value with `Console.WriteLine($"[one] {r}");`. Wrap the loop in `try/catch (OperationCanceledException)` — the timeout cancellation is expected. Run it and confirm the stream stops after exactly 2 seconds, not "until victory".

5. **Bounded channel `RunPipelineAsync`.** Create `Channel.CreateBounded<SensorReading>(new BoundedChannelOptions(8) { FullMode = BoundedChannelFullMode.Wait, SingleReader = false, SingleWriter = false });`. Launch 3 producer tasks via `Task.Run`, each of which loops (`for n in 0..9`) and calls `await channel.Writer.WriteAsync(reading, ct).ConfigureAwait(false)` — `WriteAsync` itself awaits when full, which is exactly backpressure. When all producers complete, call `channel.Writer.TryComplete()` via `Task.WhenAll(producers).ContinueWith(...)`. Launch 2 consumer tasks, each printing in `await foreach (var r in channel.Reader.ReadAllAsync(ct).ConfigureAwait(false))` the line `Console.WriteLine($"[c{id}] sensor={r.SensorId} val={r.Value:F2} on thread={Environment.CurrentManagedThreadId}");`. Await all tasks.

6. **Anti-pattern demo.** In a separate method `static void DemonstrateRaceCondition(IAsyncEnumerable<int> src)` show that calling `src.GetAsyncEnumerator()` and simultaneously invoking `MoveNextAsync()` from two `Task.Run`s raises `InvalidOperationException` ("Previous MoveNextAsync request is still in progress"). Catch the exception and print a warning — this is a vivid illustration of why `IAsyncEnumerable` is not thread-safe and why fan-out requires a `Channel<T>`.

7. **Run.** In `Program.cs` orchestrate everything: call `ConsumeOneAsync(ct)`, then `RunPipelineAsync(ct)`, then the race-condition demo. Take `ct` from `CancellationTokenSource.CreateLinkedTokenSource(...)` with a global `CancelAfter(5 sec)` for the whole program. Expected output: ~13 `[one]` lines, then ~30 `[c0]/[c1]` lines on different threads, then a `Race condition caught: ...` line.

8. **Verify.** `dotnet build` with no warnings, `dotnet run` — output matches the description. Run twice: structural idempotency of the output is mandatory (numbers may differ — that is fine, the streams are random).

#### Requirements

- Target framework `net8.0`, language C# 12, top-level statements in `Program.cs`, `ImplicitUsings` enabled, `Nullable` enabled.
- The producer method returns `IAsyncEnumerable<SensorReading>` — NOT `Task<IAsyncEnumerable<...>>` and NOT `Task<List<...>>` — preserving lazy semantics.
- The `CancellationToken` parameter of the producer is marked `[EnumeratorCancellation]`; the consumer passes the token via `WithCancellation(ct)` and adds `.ConfigureAwait(false)` in library style.
- Every `await` inside the producer and inside `await foreach` carries `.ConfigureAwait(false)`.
- The multi-threaded pipeline is built ONLY with `System.Threading.Channels`; the bounded channel uses `FullMode = BoundedChannelFullMode.Wait` (no `DropWrite` or `DropOldest` — those lose data).
- The channel is correctly closed for writes after the producers finish (`TryComplete`), otherwise `ReadAllAsync` hangs forever.
- No `.Result` / `.Wait()` on `ValueTask`/`Task` from async enumerations; no `async void`; no `lock` around `await`.
- The race-condition demo actually catches `InvalidOperationException` and does not crash the program.
- The code compiles without warnings (`TreatWarningsAsErrors` optional but welcome), builds and runs on the .NET 8 SDK.

#### Pitfalls

- **`[EnumeratorCancellation]` is mandatory.** Forget the attribute and the token from `WithCancellation(ct)` reaches only the enumerator, never the producer body: `ct.IsCancellationRequested` inside the `while` is always `false` and the timeout never fires. This is the most frequent and insidious bug — the code "works" but cannot be cancelled.
- **`WithCancellation` vs passing as a parameter.** The lesson shows both paths: you may pass `cts.Token` both as a parameter to `PollSensorAsync(..., cts.Token)` and via `.WithCancellation(cts.Token)`. Passing both is safe and even recommended for robustness — the compiler and runtime "merge" the tokens via a linked source inside the generated enumerator.
- **`ConfigureAwait(false)` in `await foreach`.** The syntax is specific: it goes after `WithCancellation` — `await foreach (var x in src.WithCancellation(ct).ConfigureAwait(false))`. Get the order wrong and the compiler will refuse — but it is easy to forget the call altogether and capture a synchronization context in WinForms/WPF.
- **`Task<IAsyncEnumerable<T>>` is a trap.** Return `IAsyncEnumerable<T>` directly. `Task<IAsyncEnumerable<...>>` adds an extra await layer and breaks lazy semantics: the consumer first awaits the method to finish, only then starts streaming.
- **Closing the channel.** If you do not call `channel.Writer.TryComplete()` (or `Complete()`) after the producers finish, `ReadAllAsync` never returns — consumers hang waiting for "one more" item. Use `Task.WhenAll(producers).ContinueWith(_ => channel.Writer.TryComplete(), TaskScheduler.Default)`.
- **`BoundedChannelFullMode.Wait` vs `Drop*`.** Only `Wait` gives honest backpressure: the producer blocks on `WriteAsync` until a consumer frees a slot. `DropWrite`/`DropOldest` silently lose data — unacceptable for telemetry.
- **Never read one stream from multiple threads.** `MoveNextAsync` is not reentrant: a second call while the first is in flight throws `InvalidOperationException`. For fan-out use `Channel<T>` with `ReadAllAsync`, where the channel guarantees each message is delivered to exactly one reader.
- **Deadlock on `.Result`.** The lesson shows `DangerousBlockingConsume` with `enumerator.MoveNextAsync().AsTask().Result`. Under a `SynchronizationContext` (WinForms/WPF) that is a deadlock; in ASP.NET Core it is pool exhaustion at scale. The only correct path is `await`.
- **`ValueTask` of the enumerator.** `MoveNextAsync` returns `ValueTask<bool>`, not `Task<bool>` — cheaper on synchronous completion. Do not convert it to `Task` unnecessarily and never await the same `ValueTask` twice.

#### Acceptance criteria

- [ ] Project `TelemetryStreams` is created via `dotnet new console -f net8.0`, builds with no errors and no warnings.
- [ ] `Program.cs` uses C# 12 top-level statements; `SensorReading` is a `record`.
- [ ] `PollSensorAsync` returns `IAsyncEnumerable<SensorReading>`, uses `yield return`, and its `ct` parameter is marked `[EnumeratorCancellation]`.
- [ ] Every `await` inside `PollSensorAsync` carries `.ConfigureAwait(false)`.
- [ ] `ConsumeOneAsync` uses `await foreach ... .WithCancellation(cts.Token).ConfigureAwait(false)` and `CancelAfter(2s)`; cancellation is caught without an unhandled exception.
- [ ] `RunPipelineAsync` creates a bounded channel with `FullMode = Wait`, `SingleReader = false`, `SingleWriter = false`.
- [ ] 3 producer tasks are launched via `Task.Run` and write to the shared channel with `WriteAsync(..., ct)`.
- [ ] After the producers finish, `channel.Writer.TryComplete()` is called.
- [ ] 2 consumer tasks read via `channel.Reader.ReadAllAsync(ct)` and print the thread id.
- [ ] The output contains lines from different consumers (`[c0]`, `[c1]`) and distinct `thread=...` values — proof of multi-threading.
- [ ] `DemonstrateRaceCondition` actually catches `InvalidOperationException` and does not crash the program.
- [ ] There is no `.Result` / `.Wait()` on async enumerations, no `async void`, no `lock` around `await`.
- [ ] No `Task<IAsyncEnumerable<...>>` or `Task<List<...>>` return type is used anywhere.
- [ ] `dotnet run` completes in ≤ 6 seconds and exits cleanly (no hang).
- [ ] The output structure is idempotent across repeated runs.

#### Hints (no direct answer)

- Recall the restaurant metaphor from the lesson: a waiter brings one plate and waits — that is `IAsyncEnumerable`. Your `PollSensorAsync` is the waiter.
- The cancellation token travels a long path: `CancellationTokenSource` → `WithCancellation` → enumerator → `[EnumeratorCancellation]` → method body. Break any link and cancellation is "lost".
- To close the channel use `Task.WhenAll(producers).ContinueWith(_ => channel.Writer.TryComplete(), TaskScheduler.Default)` — this lets the consumers finish their `await foreach` gracefully.
- For the race-condition demo, build an `IAsyncEnumerable<int>` (for example, convert `Enumerable.Range(0,1000)` via an extension or write a tiny async generator), take a single `GetAsyncEnumerator()`, and call `MoveNextAsync()` from two `Task.Run`s without await — the exception arrives quickly.
- The `BoundedChannelOptions` capacity of 8 is intentionally small — so that the producers actually hit backpressure and await on `WriteAsync`. With capacity 1000 you simply will not see the effect.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — TelemetryStreams / Program.cs
// C# 12 / .NET 8 — TelemetryStreams / Program.cs

using System.Threading.Channels;

// Data model
public record SensorReading(int SensorId, DateTimeOffset Ts, double Value, string Kind);

// ====== 1. Producer: async-stream generator with cancellation ======
public static async IAsyncEnumerable<SensorReading> PollSensorAsync(
    int sensorId,
    string kind,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    // ConfigureAwait(false) — do not capture the caller's context (library style)
    while (!ct.IsCancellationRequested)
    {
        // Simulating an async bus / I2C read
        await Task.Delay(Random.Shared.Next(20, 80), ct).ConfigureAwait(false);

        // lazy: the method freezes in the state machine until the next pull
        yield return new SensorReading(
            SensorId: sensorId,
            Ts: DateTimeOffset.UtcNow,
            Value: Random.Shared.NextDouble() * 100.0,
            Kind: kind);
    }
}

// ====== 2. Single-stream consumer with timeout ======
public static async Task ConsumeOneAsync(CancellationToken outer, CancellationTokenSource cts)
{
    // Link the outer token with the inner timeout
    using var linked = CancellationTokenSource.CreateLinkedTokenSource(outer, cts.Token);
    linked.CancelAfter(TimeSpan.FromSeconds(2));

    try
    {
        // Double token pass: as a parameter AND via WithCancellation — robust
        await foreach (var r in PollSensorAsync(sensorId: 1, "temp", linked.Token)
                           .WithCancellation(linked.Token)
                           .ConfigureAwait(false))
        {
            Console.WriteLine($"[one] {r}");
        }
    }
    catch (OperationCanceledException) { /* expected timeout cancel */ }
}

// ====== 3. Producer/Consumer pipeline via a bounded Channel ======
public static async Task RunPipelineAsync(CancellationToken ct)
{
    // Bounded channel with backpressure: on overflow the producer awaits in WriteAsync
    var channel = Channel.CreateBounded<SensorReading>(new BoundedChannelOptions(capacity: 8)
    {
        FullMode = BoundedChannelFullMode.Wait,   // never drop data
        SingleReader = false,                     // multiple consumers
        SingleWriter = false                      // multiple producers
    });

    // --- 3 producers write to the shared channel in parallel ---
    var sensors = new[] { (1, "temp"), (2, "pressure"), (3, "vibration") };
    var producers = sensors.Select(s => Task.Run(async () =>
    {
        try
        {
            for (int n = 0; n < 10 && !ct.IsCancellationRequested; n++)
            {
                await Task.Delay(Random.Shared.Next(20, 80), ct).ConfigureAwait(false);
                var reading = new SensorReading(s.Item1, DateTimeOffset.UtcNow,
                                                Random.Shared.NextDouble() * 100.0, s.Item2);
                // WriteAsync itself awaits on overflow — that is backpressure
                await channel.Writer.WriteAsync(reading, ct).ConfigureAwait(false);
            }
        }
        catch (OperationCanceledException) { }
    }, ct)).ToArray();

    // Close the channel for writes after all producers finish
    var closeTask = Task.WhenAll(producers).ContinueWith(
        _ => channel.Writer.TryComplete(), TaskScheduler.Default);

    // --- 2 consumers read via the thread-safe ReadAllAsync ---
    var consumers = Enumerable.Range(0, 2).Select(c => Task.Run(async () =>
    {
        try
        {
            // ReadAllAsync returns an IAsyncEnumerable<T> safe for multiple readers
            await foreach (var r in channel.Reader.ReadAllAsync(ct).ConfigureAwait(false))
            {
                Console.WriteLine(
                    $"[c{c}] sensor={r.SensorId} val={r.Value:F2} kind={r.Kind} thread={Environment.CurrentManagedThreadId}");
            }
        }
        catch (OperationCanceledException) { }
    }, ct)).ToArray();

    await Task.WhenAll(producers);
    await closeTask;
    await Task.WhenAll(consumers);
}

// ====== 4. Anti-pattern demo: race condition on a single enumerator ======
public static void DemonstrateRaceCondition()
{
    // Simple async number generator for the demo
    static async IAsyncEnumerable<int> NumbersAsync(
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        for (int i = 0; i < 1000; i++) { await Task.Delay(1, ct).ConfigureAwait(false); yield return i; }
    }

    var src = NumbersAsync();
    var enumerator = src.GetAsyncEnumerator();

    // Two threads call MoveNextAsync on one enumerator — this is NOT thread-safe
    var tasks = Enumerable.Range(0, 2).Select(_ => Task.Run(() =>
    {
        try { while (enumerator.MoveNextAsync().AsTask().Result) { /* skip */ } }
        // Note: .Result here is ONLY for the anti-pattern DEMO!
        catch (Exception ex) { Console.WriteLine($"Race condition caught: {ex.GetType().Name}: {ex.Message}"); }
    })).ToArray();

    Task.WaitAll(tasks);
}

// ====== Entry point (top-level statements) ======
using var appCts = CancellationTokenSource.CreateLinkedTokenSource();
appCts.CancelAfter(TimeSpan.FromSeconds(5));

await ConsumeOneAsync(appCts.Token, CancellationTokenSource.CreateLinkedTokenSource());
await RunPipelineAsync(appCts.Token);
DemonstrateRaceCondition();
Console.WriteLine("Done.");
```

**Line-by-line walk-through.** `PollSensorAsync` is the heart of the assignment: `async IAsyncEnumerable<...>` + `yield return` + `[EnumeratorCancellation]`. The attribute is critical — without it `WithCancellation` is useless. `ConfigureAwait(false)` on every `await` is the library standard from the lesson. `ConsumeOneAsync` shows the correct combination of `linked token + CancelAfter + WithCancellation + ConfigureAwait(false)` and catches `OperationCanceledException` — a timeout cancellation is expected, not an error. `RunPipelineAsync` is a concentrate of the lesson's concurrency accents: a bounded channel (`FullMode = Wait`) provides backpressure, `WriteAsync` awaits on overflow, `ReadAllAsync` returns an `IAsyncEnumerable<T>` thread-safe for multiple readers, and `TryComplete()` after `Task.WhenAll(producers)` is critically important — otherwise `ReadAllAsync` hangs. Two consumers print `Environment.CurrentManagedThreadId` — you see different IDs, proving real multi-threading. `DemonstrateRaceCondition` deliberately uses `.Result` (the only sanctioned case — demonstrating the mistake): two `Task.Run`s call `MoveNextAsync` on a single enumerator and the runtime throws `InvalidOperationException` with the message "Previous MoveNextAsync request is still in progress" — a vivid illustration of why `IAsyncEnumerable` is not thread-safe and why fan-out needs a `Channel<T>`. Lesson concepts applied here: pull-based lazy generation, correct cancellation through `[EnumeratorCancellation]`, `ConfigureAwait(false)`, a bounded channel with backpressure, `ReadAllAsync` for multiple readers, and a demonstration of the deadlock/race anti-patterns.

#### Going deeper (bonus)

1. **Merge several sensors into one stream.** Implement `MergeAsync(IAsyncEnumerable<IAsyncEnumerable<SensorReading>> sources, CancellationToken ct)` that uses a `Channel.CreateBounded` and a set of producer tasks (one per source) to merge all streams into a single `IAsyncEnumerable<SensorReading>`. Prove that the resulting stream can be read by one `await foreach` while the sources are polled in parallel.
2. **A custom enumerator without `yield`.** Implement `IAsyncEnumerable<T>` by hand: a class with `GetAsyncEnumerator` returning an object with `MoveNextAsync`/`Current`/`DisposeAsync` that reads from HTTP pagination (`?page=N`) via `HttpClient`. Compare readability and performance against a `yield`-based version.
3. **Throttle and batch.** Add an extension operator `BufferAsync(this IAsyncEnumerable<T> src, int batchSize, TimeSpan flushInterval)` that accumulates items and yields them in batches through a `Channel` — useful for batched DB writes.
4. **Cancellation with progress.** Tie an async stream to `IProgress<T>`: every 10 items report progress to an external subscriber, and on cancellation correctly complete the enumerator's `DisposeAsync`.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект собирается `dotnet build` без warnings на .NET 8 / C# 12.
- [ ] `PollSensorAsync` возвращает `IAsyncEnumerable<SensorReading>` с `[EnumeratorCancellation]`.
- [ ] `await foreach` использует `WithCancellation` + `ConfigureAwait(false)`.
- [ ] Bounded-канал с `FullMode = Wait` и корректным `TryComplete`.
- [ ] Многопоточный pipeline через `Channel<T>`, без разделяемого `IAsyncEnumerable`.
- [ ] Демонстрация race condition ловит `InvalidOperationException`.
- [ ] Нет `.Result`/`.Wait()`, `async void`, `lock` вокруг `await`.
- [ ] Project builds with `dotnet build` warning-free on .NET 8 / C# 12.
- [ ] `PollSensorAsync` returns `IAsyncEnumerable<SensorReading>` with `[EnumeratorCancellation]`.
- [ ] `await foreach` uses `WithCancellation` + `ConfigureAwait(false)`.
- [ ] Bounded channel with `FullMode = Wait` and a correct `TryComplete`.
- [ ] Multi-threaded pipeline via `Channel<T>`, no shared `IAsyncEnumerable`.
- [ ] Race-condition demo catches `InvalidOperationException`.
- [ ] No `.Result`/`.Wait()`, no `async void`, no `lock` around `await`.

#### Ресурсы / Resources
- [Microsoft Learn — Generate and consume async streams](https://learn.microsoft.com/dotnet/csharp/asynchronous-programming/generate-consume-asynchronous-streams)
- [Microsoft Learn — System.Threading.Channels](https://learn.microsoft.com/dotnet/api/system.threading.channels)
- [Microsoft Learn — IAsyncEnumerable\<T\>](https://learn.microsoft.com/dotnet/api/system.collections.generic.iasyncenumerable-1)
- [Microsoft Learn — EnumeratorCancellationAttribute](https://learn.microsoft.com/dotnet/api/system.runtime.compilerservices.enumeratorcancellationattribute)
- [Stephen Toub — Async Streams in C# 8](https://devblogs.microsoft.com/dotnet/async-streams-and-cancellation-in-csharp-8/)
