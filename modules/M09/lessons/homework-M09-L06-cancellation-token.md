---
[← К уроку M09-L06](lesson-M09-L06-cancellation-token.md) | [⬆ К модулю M09](../README.md) | [Следующее ДЗ →](homework-M09-L07-configureawait-synccontext.md)
---

### Домашнее задание M09-L06: CancellationToken, cooperative cancellation / Homework M09-L06: CancellationToken, cooperative cancellation

**Урок / Lesson:** M09-L06
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться строить отменяемый асинхронный конвейер producer/consumer на `Channel`, применять linked tokens для композиции причин отмены (Ctrl+C + таймаут), корректно прокидывать `CancellationToken` в BCL-вызовы и возвращать частичный результат при отмене без дедлоков и утечек таймеров. (EN) Learn to build a cancellable async producer/consumer pipeline on `Channel`, apply linked tokens to compose cancellation reasons (Ctrl+C + timeout), correctly forward `CancellationToken` into BCL calls, and return a partial result on cancellation without deadlocks or timer leaks.

#### Связь с уроком / Connection to the lesson
(RU) Задание прямо опирается на все ключевые темы урока: разделение `CancellationTokenSource`/`CancellationToken`, конвенцию `CancellationToken cancellationToken = default`, `ThrowIfCancellationRequested()` в циклах, `CancelAfter`/`CancellationTokenSource(timeout)`, `CreateLinkedTokenSource`, `token.Register(...)`, отменяемый producer/consumer через `Channel` и правильный вход в async из `Main` через `Console.CancelKeyPress`. (EN) The assignment directly builds on every key topic of the lesson: the `CancellationTokenSource`/`CancellationToken` split, the `CancellationToken cancellationToken = default` convention, `ThrowIfCancellationRequested()` in loops, `CancelAfter`/`CancellationTokenSource(timeout)`, `CreateLinkedTokenSource`, `token.Register(...)`, a cancellable producer/consumer via `Channel`, and a proper async entry from `Main` through `Console.CancelKeyPress`.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
В реальных приложениях длительные операции — обработка потоков телеметрии, обход больших каталогов, HTTP-запросы к медленным сервисам — почти никогда не должны выполняться «до упора». Пользователь может закрыть вкладку, сработать жёсткий таймаут балансировщика, либо администратор решит остановить конвейер. В .NET `Thread.Abort()` давно deprecated и опасен, поэтому единственная здоровая модель — кооперативная отмена через `CancellationToken`. Источник сигнала `CancellationTokenSource` управляет жизненным циклом, а лёгкая структура `CancellationToken` протаскивается вглубь call-stack и позволяет каждой операции добровольно проверять, не пора ли остановиться. Ключевая сложность для новичков — научиться не «убивать» потоки, а выстраивать pipeline, где каждый слой сотрудничает с токеном: проверяет `ThrowIfCancellationRequested()`, прокидывает токен в BCL-вызовы, корректно завершает ресурсы и возвращает частичные результаты. В этом задании вы построите отменяемый конвейер агрегации телеметрии «producer → bounded Channel → consumer» с двумя независимыми причинами отмены (Ctrl+C и таймаут), объединёнными через linked tokens, и убедитесь, что отмена приходит быстро, без утечек и без дедлоков. Главная учебная ценность — не «написать цикл», а прочувствовать, что кооперативная отмена это архитектурное свойство всего стека вызовов: один «глухой» слой без токена ломает всю цепочку.

#### Что нужно сделать (пошагово)
1. Создайте консольный проект .NET 8 с именем `TelemetryAggregator`:
   ```
   dotnet new console -n TelemetryAggregator -o TelemetryAggregator -f net8.0
   cd TelemetryAggregator
   ```
   Убедитесь, что в `TelemetryAggregator.csproj` стоит `<TargetFramework>net8.0</TargetFramework>` и `<LangVersion>12</LangVersion>` (или `latest`).

2. Откройте `Program.cs` и замените содержимое на top-level statements программу (C# 12). Программа должна:
   - создать `CancellationTokenSource cts` для внешней отмены по Ctrl+C;
   - подписаться на `Console.CancelKeyPress`, установить `e.Cancel = true` и вызвать `cts.Cancel()` — мягко, без убийства процесса;
   - создать `timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(10))` для жёсткого таймаута;
   - объединить оба через `CancellationTokenSource.CreateLinkedTokenSource(cts.Token, timeoutCts.Token)` в переменную `linked` (обязательно в `using`);
   - зарегистрировать короткий колбэк через `linked.Token.Register(...)`, который печатает строку очистки;
   - вызвать `await RunPipelineAsync(linked.Token)` и напечатать агрегат.

3. Реализуйте `record TelemetryAggregate(int Count, double Average, int Max, int Min)` со статическим свойством `Empty`.

4. Реализуйте `static async IAsyncEnumerable<int> GenerateReadingsAsync([EnumeratorCancellation] CancellationToken ct)`, бесконечно генерирующий случайные значения 0..99 с `Task.Delay(5, ct)` между ними и `ct.ThrowIfCancellationRequested()` на каждой итерации.

5. Реализуйте `async Task<TelemetryAggregate> RunPipelineAsync(CancellationToken ct)`, который:
   - создаёт `Channel.CreateBounded<int>(32)` с `FullMode = Wait`;
   - запускает producer через `Task.Run(async () => { try { await foreach (var v in GenerateReadingsAsync(ct)) await channel.Writer.WriteAsync(v, ct); } catch (OperationCanceledException) { } finally { channel.Writer.Complete(); } }, ct)`;
   - запускает consumer через `Task.Run`, читающий `await foreach (var v in channel.Reader.ReadAllAsync(ct))` и агрегирующий count/sum/max/min в локальных переменных замыкания; внутри `try/catch (OperationCanceledException)` чтобы сохранить частичный результат;
   - `await Task.WhenAll(producer, consumer)` и возвращает `TelemetryAggregate` (частичный, если была отмена).

6. В Main после `await RunPipelineAsync` добавьте `finally` блок, который проверяет `linked.IsCancellationRequested` и `timeoutCts.IsCancellationRequested`, чтобы напечатать причину остановки (таймаут или Ctrl+C).

7. Соберите и запустите:
   ```
   dotnet build
   dotnet run
   ```
   Дайте поработать 3 секунды, затем нажмите Ctrl+C. Ожидаемый вывод: строка агрегата + сообщение об остановке по Ctrl+C + строка cleanup от колбэка. Перезапустите и не трогайте клавиатуру — через 10 секунд программа должна остановиться по таймауту с тем же аккуратным завершением.

8. Запустите ещё раз и проверьте, что нет `Unhandled exception`, нет утечки таймера (процесс завершается за <1с после остановки), и что `OperationCanceledException` нигде не «глотается» молча без смысла. Сравните вывод при Ctrl+C и при таймауте — причина остановки должна различаться.

#### Требования к решению
- Только C# 12 / .NET 8: top-level statements, record-ы, pattern matching, `using` declarations, вывод типов.
- Все публичные/async-методы принимают `CancellationToken ct = default` последним параметром (для `RunPipelineAsync` делайте параметр обязательным — он всегда должен получать токен).
- Токен прокинут в каждый BCL/async-вызов: `Task.Delay`, `channel.Writer.WriteAsync`, `channel.Reader.ReadAllAsync`, `Task.Run`.
- Внутри циклов стоит `ct.ThrowIfCancellationRequested()` (или проверка делегирована в `ReadAllAsync`/`await foreach`).
- `CancellationTokenSource` с таймером/linked обёрнуты в `using` (или явно диспозятся) — никаких утечек таймеров.
- `OperationCanceledException` ловится осмысленно: либо глотается в producer/consumer для штатного завершения с комментарием «normal cancellation», либо в Main ловится с фильтром `when (linked.IsCancellationRequested)`.
- Нет `.Result`/`.Wait()`/`.GetAwaiter().GetResult()`, нет `async void`, нет `lock` поверх `await`.
- Колбэк `Register` короткий и реентерабельный (просто `Console.WriteLine`).
- Программа компилируется без warning-ов (включая CS1996 на попытку `lock+await`) и работает стабильно при многократном Ctrl+C.

#### Тонкости и подводные камни
- Подписка `Console.CancelKeyPress`: если не поставить `e.Cancel = true`, процесс убьётся сразу, и `finally`/cleanup не отработают. Это нарушает саму идею кооперативной отмены.
- `CreateLinkedTokenSource` обязательно диспозить: под капотом он держит таймер (от `timeoutCts`) и список registrants. `using` решает это автоматически.
- `Task.Run(async () => ..., ct)`: передача `ct` вторым аргументом позволяет планировщику не запускать задачу вообще, если токен уже отменён, но не «убивает» уже бегущую задачу — поэтому проверка `ThrowIfCancellationRequested` внутри тела всё равно нужна.
- `channel.Reader.ReadAllAsync(ct)`: токен передаётся в `await foreach`, и при отмене перечислитель сам бросит `OperationCanceledException` — отдельная проверка в теле опциональна, но рекомендована для длинных шагов обработки.
- `channel.Writer.Complete()` в `finally` producer-а критичен: без него consumer зависнет в `ReadAllAsync` навсегда, даже если данные закончились.
- Не путайте «свою» и «чужую» отмену: в Main фильтр `when (linked.IsCancellationRequested)` отличает отмену по нашим причинам от случайной `OperationCanceledException` из глубины BCL.
- `Register` колбэк может выполниться в потоке, вызвавшем `Cancel()` (то есть в потоке обработчика Ctrl+C) — поэтому он не должен трогать UI или брать длинные блокировки.
- Частые проверки `ThrowIfCancellationRequested()` снижают задержку реакции: между проверками программа выполняет «глухой» участок. Чем короче шаг, тем быстрее отмена.
- `[EnumeratorCancellation]` на параметре `CancellationToken` async-итератора обязателен: без него токен из `WithCancellation` не пробросится в тело итератора, и отмена перечисления не отменит сам генератор.

#### Критерии приёмки
- [ ] Проект `TelemetryAggregator` собирается `dotnet build` без ошибок и warning-ов на .NET 8 / C# 12.
- [ ] Использованы top-level statements; нет явного `class Program` / `static void Main`.
- [ ] `Console.CancelKeyPress` подписан, `e.Cancel = true`, вызывается `cts.Cancel()`.
- [ ] Создан `timeoutCts` с `TimeSpan.FromSeconds(10)` и `linked` через `CreateLinkedTokenSource`.
- [ ] `linked` и `timeoutCts` обёрнуты в `using` — нет утечки таймеров.
- [ ] Зарегистрирован колбэк `linked.Token.Register(...)`, короткий и реентерабельный.
- [ ] `GenerateReadingsAsync` помечен `[EnumeratorCancellation]` и проверяет `ct.ThrowIfCancellationRequested()` в цикле.
- [ ] `RunPipelineAsync` создаёт bounded Channel с `FullMode = Wait`.
- [ ] Producer: `try/catch (OperationCanceledException) { } finally { channel.Writer.Complete(); }`.
- [ ] Consumer: `await foreach (var v in channel.Reader.ReadAllAsync(ct))` и агрегация в замыкании; OCE ловится, частичный результат сохраняется.
- [ ] `await Task.WhenAll(producer, consumer)` без `.Wait()`/`.Result`.
- [ ] Main печатает агрегат и причину остановки (таймаут / Ctrl+C) в `finally`.
- [ ] Нигде нет `async void`, `lock` поверх `await`, `.Result`/`.Wait()`.
- [ ] При Ctrl+C программа завершается за <1с, без `Unhandled exception`.
- [ ] При отсутствии Ctrl+C через 10 секунд срабатывает таймаут с тем же аккуратным завершением.

#### Подсказки (без прямого ответа)
- Помните, что `CancellationTokenSource(TimeSpan)` эквивалентен `CancelAfter` — оба стартуют таймер.
- Для bounded Channel используйте `Channel.CreateBounded<int>(new BoundedChannelOptions(32) { FullMode = BoundedChannelFullMode.Wait, SingleReader = false, SingleWriter = false })`.
- Частичный результат — это просто переменные `count`/`sum`/`max`/`min`, объявленные снаружи `Task.Run` (замыкание).
- Различение причины остановки: после `await` проверьте `timeoutCts.IsCancellationRequested` — true означает таймаут.
- Не пытайтесь «убить» producer через `Thread.Abort` — его нет; единственный путь — чтобы producer сам увидел токен.
- Если consumer завис после отмены — проверьте, что producer вызывает `channel.Writer.Complete()` в `finally`.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — ДЗ M09-L06: отменяемый конвейер агрегации телеметрии
// Cooperative cancellation, linked tokens, bounded Channel, Ctrl+C
using System.Threading.Channels;

// ---- Точка входа (top-level statements) ----
// Источник внешней отмены: его дёргает обработчик Ctrl+C.
using var cts = new CancellationTokenSource();
Console.CancelKeyPress += (_, e) =>
{
    e.Cancel = true;          // не убивать процесс сразу / do not kill at once
    cts.Cancel();             // мягко просим остановиться / ask to stop softly
};

// Жёсткий таймаут 10 секунд: CancellationTokenSource(TimeSpan) == CancelAfter.
using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(10));

// Композиция причин отмены: Ctrl+C ИЛИ таймаут.
using var linked = CancellationTokenSource.CreateLinkedTokenSource(cts.Token, timeoutCts.Token);

// Короткий реентерабельный колбэк очистки.
linked.Token.Register(() => Console.WriteLine("[cleanup] linked token fired / связанный токен сработал"));

try
{
    var result = await RunPipelineAsync(linked.Token);
    Console.WriteLine(result);
}
finally
{
    if (linked.IsCancellationRequested)
    {
        // Различаем причину: таймаут или Ctrl+C.
        if (timeoutCts.IsCancellationRequested)
            Console.WriteLine("Stopped by timeout / остановлено по таймауту");
        else
            Console.WriteLine("Stopped by Ctrl+C / остановлено по Ctrl+C");
    }
}

// ---- Запись-результат агрегации ----
record TelemetryAggregate(int Count, double Average, int Max, int Min)
{
    public static TelemetryAggregate Empty => new(0, 0, int.MinValue, int.MaxValue);
}

// ---- Источник показаний: бесконечный IAsyncEnumerable с отменой ----
static async IAsyncEnumerable<int> GenerateReadingsAsync(
    [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct)
{
    var rnd = new Random();
    for (int i = 0; ; i++)
    {
        ct.ThrowIfCancellationRequested();   // частая проверка / frequent check
        await Task.Delay(5, ct);             // имитация I/O, токен прокинут / I/O mock, token forwarded
        yield return rnd.Next(0, 100);
    }
}

// ---- Конвейер producer/consumer через bounded Channel ----
async Task<TelemetryAggregate> RunPipelineAsync(CancellationToken ct)
{
    var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(32)
    {
        FullMode = BoundedChannelFullMode.Wait,  // backpressure без ручной синхронизации
        SingleReader = false,
        SingleWriter = false,
    });

    // ПРОДЮСЕР: читает из GenerateReadingsAsync и пишет в канал.
    var producer = Task.Run(async () =>
    {
        try
        {
            await foreach (var v in GenerateReadingsAsync(ct))
                await channel.Writer.WriteAsync(v, ct);  // токен прокинут в BCL
        }
        catch (OperationCanceledException) { /* штатная отмена / normal cancellation */ }
        finally
        {
            channel.Writer.Complete();  // критично: без этого consumer зависнет
        }
    }, ct);

    // КОНСЬЮМЕР: агрегирует в замыкании, сохраняя частичный результат при отмене.
    int count = 0, sum = 0, max = int.MinValue, min = int.MaxValue;
    var consumer = Task.Run(async () =>
    {
        try
        {
            await foreach (var v in channel.Reader.ReadAllAsync(ct))
            {
                ct.ThrowIfCancellationRequested();
                count++; sum += v;
                if (v > max) max = v;
                if (v < min) min = v;
            }
        }
        catch (OperationCanceledException) { /* сохраняем частичный результат */ }
    }, ct);

    // Неблокирующее ожидание: никаких .Wait()/.Result.
    await Task.WhenAll(producer, consumer);

    return count == 0
        ? TelemetryAggregate.Empty
        : new TelemetryAggregate(count, (double)sum / count, max, min);
}
```

Разбор по строкам. Строка `using var cts = new CancellationTokenSource();` создаёт источник внешней отмены — его будет дёргать обработчик Ctrl+C. Подписка `Console.CancelKeyPress` с `e.Cancel = true` критична: без неё среда убьёт процесс мгновенно, и ни `finally`, ни зарегистрированный колбэк не отработают, что противоречит самой идее кооперативной отмены. Далее `using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(10));` — это и есть `CancelAfter` в форме конструктора: под капотом стартует внутренний таймер, который через 10 секунд вызовет `Cancel()`. Именно поэтому CTS обязательно диспозить — иначе таймер утечёт, что особенно опасно в долго живущих сервисах.

Связывание `CancellationTokenSource.CreateLinkedTokenSource(cts.Token, timeoutCts.Token)` реализует композицию причин отмены: связанный токен сработает, если отменят ЛЮБОЙ из источников — либо Ctrl+C, либо таймаут. Это избавляет от «ручного» вызова нескольких `Cancel()` и синхронизации между ними. Колбэк `linked.Token.Register(...)` иллюстрирует ещё один механизм кооперативной отмены: подписку на сигнал. Важно, что колбэк короткий и реентерабельный — он может выполниться в потоке, вызвавшем `Cancel()` (в данном случае в потоке обработчика Ctrl+C).

`GenerateReadingsAsync` помечен `[EnumeratorCancellation]`: этот атрибут позволяет токену, переданному в `await foreach` (через `WithCancellation`), корректно проброситься внутрь метода-итератора и сработать при отмене перечисления. Без атрибута токен `ct` формального параметра не обновится токеном из `WithCancellation`, и отмена перечислителя не отменит тело итератора. Внутри цикла стоит `ct.ThrowIfCancellationRequested()` — частая проверка снижает задержку реакции.

Конвейер построен на `Channel.CreateBounded<int>` с `FullMode = Wait`: это даёт backpressure без ручной синхронизации — producer ждёт места в очереди. Producer обёрнут в `try/catch (OperationCanceledException) { } finally { channel.Writer.Complete(); }`: `Complete` в `finally` критичен — без него consumer зависнет в `ReadAllAsync` навсегда. Consumer агрегирует в переменные замыкания (`count`, `sum`, `max`, `min`), а `OperationCanceledException` ловится локально, чтобы частичный результат не потерялся — это и есть кооперативность: мы не выбрасываем накопленное, а возвращаем его.

`await Task.WhenAll(producer, consumer)` — неблокирующее ожидание; никаких `.Wait()`/`.Result`, которые могли бы дедлочить пул. В `finally` Main-а различаем причину остановки через `timeoutCts.IsCancellationRequested`: true означает, что первым сработал таймаут, false — что Ctrl+C. Так мы применяем best practice «различать свою и чужую отмену» и не глотаем `OperationCanceledException` молча. Передача `ct` в `Task.Run(..., ct)` вторым аргументом — это оптимизация: если токен уже отменён к моменту запуска, задача не начнётся вовсе, но бегущую задачу это не «убивает», поэтому проверка внутри тела всё равно нужна.

#### Задания на углубление (бонус)
1. Добавьте третий источник отмены: «graceful shutdown» по сигналу `SIGTERM` (в .NET 8 — `AppDomain.CurrentDomain.ProcessExit` или второй Ctrl+C). Объедините все три через linked tokens и убедитесь, что cleanup отрабатывает для любой причины.
2. Реализуйте второй consumer, который пишет агрегат в файл каждые 100 элементов; при отмене flush-ит последний кусок. Используйте `Channel.CreateBounded` с `SingleReader = false` и синхронизируйте flush через `SemaphoreSlim(1,1).WaitAsync(ct)`.
3. Замените `Task.Delay(5, ct)` в `GenerateReadingsAsync` на реальный `HttpClient.GetStringAsync(url, ct)` к медленному эндпоинту и покажите, что токен прокинут в BCL-вызов и отмена приходит за миллисекунды.
4. Добавьте unit-тесты через xUnit: проверьте, что `RunPipelineAsync` завершается за <500мс при отмене через 100мс, и что частичный результат возвращается (count > 0).

---

## Statement in English / Постановка на английском

#### Context & motivation
In real applications long-running operations — telemetry stream processing, large directory traversal, HTTP calls to slow services — should almost never run "to the bitter end". The user may close the tab, a load-balancer hard timeout may fire, or an administrator may decide to stop the pipeline. In .NET `Thread.Abort()` has long been deprecated and is dangerous, so the only healthy model is cooperative cancellation via `CancellationToken`. The signal source `CancellationTokenSource` controls the lifecycle, while the lightweight `CancellationToken` struct is threaded down the call stack and lets every operation voluntarily check whether it should stop. The key difficulty for newcomers is to learn not to "kill" threads, but to build a pipeline where every layer cooperates with the token: calls `ThrowIfCancellationRequested()`, forwards the token into BCL calls, releases resources correctly, and returns partial results. In this assignment you will build a cancellable telemetry aggregation pipeline "producer → bounded Channel → consumer" with two independent cancellation reasons (Ctrl+C and timeout) composed via linked tokens, and verify that cancellation arrives promptly, without leaks and without deadlocks. The main educational value is not "to write a loop" but to internalize that cooperative cancellation is an architectural property of the whole call stack: a single "deaf" layer without a token breaks the entire chain.

#### What to do step by step
1. Create a .NET 8 console project named `TelemetryAggregator`:
   ```
   dotnet new console -n TelemetryAggregator -o TelemetryAggregator -f net8.0
   cd TelemetryAggregator
   ```
   Make sure `TelemetryAggregator.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and `<LangVersion>12</LangVersion>` (or `latest`).

2. Open `Program.cs` and replace its contents with a top-level statements program (C# 12). The program must:
   - create `CancellationTokenSource cts` for external cancellation via Ctrl+C;
   - subscribe to `Console.CancelKeyPress`, set `e.Cancel = true`, and call `cts.Cancel()` — softly, without killing the process;
   - create `timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(10))` for a hard timeout;
   - combine both via `CancellationTokenSource.CreateLinkedTokenSource(cts.Token, timeoutCts.Token)` into a variable `linked` (mandatory inside a `using`);
   - register a short callback via `linked.Token.Register(...)` that prints a cleanup line;
   - call `await RunPipelineAsync(linked.Token)` and print the aggregate.

3. Implement `record TelemetryAggregate(int Count, double Average, int Max, int Min)` with a static `Empty` property.

4. Implement `static async IAsyncEnumerable<int> GenerateReadingsAsync([EnumeratorCancellation] CancellationToken ct)` that infinitely produces random values 0..99 with `Task.Delay(5, ct)` between them and `ct.ThrowIfCancellationRequested()` on every iteration.

5. Implement `async Task<TelemetryAggregate> RunPipelineAsync(CancellationToken ct)` that:
   - creates `Channel.CreateBounded<int>(32)` with `FullMode = Wait`;
   - starts a producer via `Task.Run(async () => { try { await foreach (var v in GenerateReadingsAsync(ct)) await channel.Writer.WriteAsync(v, ct); } catch (OperationCanceledException) { } finally { channel.Writer.Complete(); } }, ct)`;
   - starts a consumer via `Task.Run` reading `await foreach (var v in channel.Reader.ReadAllAsync(ct))` and aggregating count/sum/max/min in closure locals; inside `try/catch (OperationCanceledException)` so the partial result is preserved;
   - `await Task.WhenAll(producer, consumer)` and returns `TelemetryAggregate` (partial if cancellation happened).

6. In Main after `await RunPipelineAsync` add a `finally` block that checks `linked.IsCancellationRequested` and `timeoutCts.IsCancellationRequested` to print the stop reason (timeout or Ctrl+C).

7. Build and run:
   ```
   dotnet build
   dotnet run
   ```
   Let it run for 3 seconds, then press Ctrl+C. Expected output: one aggregate line + a "stopped by Ctrl+C" message + a cleanup line from the callback. Restart and do not touch the keyboard — after 10 seconds the program must stop by timeout with the same graceful shutdown.

8. Run once more and verify there is no `Unhandled exception`, no timer leak (the process exits within <1s after the stop), and that `OperationCanceledException` is never swallowed silently without a reason. Compare the output for Ctrl+C and for timeout — the stop reason must differ.

#### Requirements
- C# 12 / .NET 8 only: top-level statements, records, pattern matching, `using` declarations, target-typed `new`.
- Every public/async method takes `CancellationToken ct = default` as the last parameter (for `RunPipelineAsync` make the parameter mandatory — it must always receive a token).
- The token is forwarded into every BCL/async call: `Task.Delay`, `channel.Writer.WriteAsync`, `channel.Reader.ReadAllAsync`, `Task.Run`.
- Inside loops there is `ct.ThrowIfCancellationRequested()` (or the check is delegated to `ReadAllAsync`/`await foreach`).
- `CancellationTokenSource` with a timer/linked is wrapped in `using` (or disposed explicitly) — no timer leaks.
- `OperationCanceledException` is caught meaningfully: either swallowed in producer/consumer for normal shutdown with a "normal cancellation" comment, or caught in Main with a `when (linked.IsCancellationRequested)` filter.
- No `.Result`/`.Wait()`/`.GetAwaiter().GetResult()`, no `async void`, no `lock` over `await`.
- The `Register` callback is short and reentrant (just `Console.WriteLine`).
- The program compiles without warnings (including CS1996 on an attempted `lock+await`) and runs stably under repeated Ctrl+C.

#### Pitfalls
- The `Console.CancelKeyPress` subscription: if you do not set `e.Cancel = true`, the process is killed at once, and neither `finally` nor the cleanup callback run. This breaks the very idea of cooperative cancellation.
- `CreateLinkedTokenSource` must be disposed: under the hood it holds a timer (from `timeoutCts`) and a registrant list. `using` handles this automatically.
- `Task.Run(async () => ..., ct)`: passing `ct` as the second argument lets the scheduler skip starting the task entirely if the token is already cancelled, but it does not "kill" an already running task — so `ThrowIfCancellationRequested` inside the body is still required.
- `channel.Reader.ReadAllAsync(ct)`: the token is forwarded into `await foreach`, and on cancellation the enumerator itself throws `OperationCanceledException` — an extra check in the body is optional but recommended for long processing steps.
- `channel.Writer.Complete()` in the producer's `finally` is critical: without it the consumer hangs in `ReadAllAsync` forever, even when data is exhausted.
- Do not confuse "own" and "foreign" cancellation: in Main the `when (linked.IsCancellationRequested)` filter distinguishes cancellation by our reasons from a stray `OperationCanceledException` deep in BCL.
- The `Register` callback may run on the thread that called `Cancel()` (that is, on the Ctrl+C handler thread) — so it must not touch the UI or take long locks.
- Frequent `ThrowIfCancellationRequested()` checks reduce reaction latency: between checks the program runs a "deaf" segment. The shorter the step, the faster the cancellation.
- `[EnumeratorCancellation]` on the `CancellationToken` parameter of an async iterator is mandatory: without it the token from `WithCancellation` does not propagate into the iterator body, and cancelling the enumeration does not cancel the generator.

#### Acceptance criteria
- [ ] The `TelemetryAggregator` project builds with `dotnet build` without errors or warnings on .NET 8 / C# 12.
- [ ] Top-level statements are used; there is no explicit `class Program` / `static void Main`.
- [ ] `Console.CancelKeyPress` is subscribed, `e.Cancel = true`, `cts.Cancel()` is called.
- [ ] `timeoutCts` is created with `TimeSpan.FromSeconds(10)` and `linked` via `CreateLinkedTokenSource`.
- [ ] `linked` and `timeoutCts` are wrapped in `using` — no timer leaks.
- [ ] A `linked.Token.Register(...)` callback is registered, short and reentrant.
- [ ] `GenerateReadingsAsync` is annotated with `[EnumeratorCancellation]` and checks `ct.ThrowIfCancellationRequested()` in the loop.
- [ ] `RunPipelineAsync` creates a bounded Channel with `FullMode = Wait`.
- [ ] Producer: `try/catch (OperationCanceledException) { } finally { channel.Writer.Complete(); }`.
- [ ] Consumer: `await foreach (var v in channel.Reader.ReadAllAsync(ct))` and aggregation in a closure; OCE is caught, the partial result is preserved.
- [ ] `await Task.WhenAll(producer, consumer)` with no `.Wait()`/`.Result`.
- [ ] Main prints the aggregate and the stop reason (timeout / Ctrl+C) in `finally`.
- [ ] There is no `async void`, no `lock` over `await`, no `.Result`/`.Wait()` anywhere.
- [ ] On Ctrl+C the program exits within <1s, with no `Unhandled exception`.
- [ ] Without Ctrl+C, after 10 seconds the timeout fires with the same graceful shutdown.

#### Hints (no direct answer)
- Remember that `CancellationTokenSource(TimeSpan)` is equivalent to `CancelAfter` — both start a timer.
- For a bounded Channel use `Channel.CreateBounded<int>(new BoundedChannelOptions(32) { FullMode = BoundedChannelFullMode.Wait, SingleReader = false, SingleWriter = false })`.
- The partial result is simply the `count`/`sum`/`max`/`min` variables declared outside `Task.Run` (a closure).
- To distinguish the stop reason: after `await` check `timeoutCts.IsCancellationRequested` — true means timeout.
- Do not try to "kill" the producer via `Thread.Abort` — it does not exist; the only path is for the producer to see the token itself.
- If the consumer hangs after cancellation — verify the producer calls `channel.Writer.Complete()` in `finally`.

#### Reference solution (walk-through)
```csharp
// C# 12 / .NET 8 — Homework M09-L06: cancellable telemetry aggregation pipeline
// Cooperative cancellation, linked tokens, bounded Channel, Ctrl+C
using System.Threading.Channels;

// ---- Entry point (top-level statements) ----
// Source of external cancellation: it is pulled by the Ctrl+C handler.
using var cts = new CancellationTokenSource();
Console.CancelKeyPress += (_, e) =>
{
    e.Cancel = true;          // do not kill the process at once
    cts.Cancel();             // ask operations to stop softly
};

// Hard timeout of 10 seconds: CancellationTokenSource(TimeSpan) == CancelAfter.
using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(10));

// Compose cancellation reasons: Ctrl+C OR timeout.
using var linked = CancellationTokenSource.CreateLinkedTokenSource(cts.Token, timeoutCts.Token);

// Short reentrant cleanup callback.
linked.Token.Register(() => Console.WriteLine("[cleanup] linked token fired"));

try
{
    var result = await RunPipelineAsync(linked.Token);
    Console.WriteLine(result);
}
finally
{
    if (linked.IsCancellationRequested)
    {
        // Distinguish the reason: timeout or Ctrl+C.
        if (timeoutCts.IsCancellationRequested)
            Console.WriteLine("Stopped by timeout");
        else
            Console.WriteLine("Stopped by Ctrl+C");
    }
}

// ---- Aggregate result record ----
record TelemetryAggregate(int Count, double Average, int Max, int Min)
{
    public static TelemetryAggregate Empty => new(0, 0, int.MinValue, int.MaxValue);
}

// ---- Reading source: infinite IAsyncEnumerable with cancellation ----
static async IAsyncEnumerable<int> GenerateReadingsAsync(
    [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct)
{
    var rnd = new Random();
    for (int i = 0; ; i++)
    {
        ct.ThrowIfCancellationRequested();   // frequent check
        await Task.Delay(5, ct);             // I/O mock, token forwarded
        yield return rnd.Next(0, 100);
    }
}

// ---- Producer/consumer pipeline over a bounded Channel ----
async Task<TelemetryAggregate> RunPipelineAsync(CancellationToken ct)
{
    var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(32)
    {
        FullMode = BoundedChannelFullMode.Wait,  // backpressure without manual sync
        SingleReader = false,
        SingleWriter = false,
    });

    // PRODUCER: reads from GenerateReadingsAsync and writes into the channel.
    var producer = Task.Run(async () =>
    {
        try
        {
            await foreach (var v in GenerateReadingsAsync(ct))
                await channel.Writer.WriteAsync(v, ct);  // token forwarded into BCL
        }
        catch (OperationCanceledException) { /* normal cancellation */ }
        finally
        {
            channel.Writer.Complete();  // critical: without it the consumer hangs
        }
    }, ct);

    // CONSUMER: aggregates in a closure, preserving the partial result on cancellation.
    int count = 0, sum = 0, max = int.MinValue, min = int.MaxValue;
    var consumer = Task.Run(async () =>
    {
        try
        {
            await foreach (var v in channel.Reader.ReadAllAsync(ct))
            {
                ct.ThrowIfCancellationRequested();
                count++; sum += v;
                if (v > max) max = v;
                if (v < min) min = v;
            }
        }
        catch (OperationCanceledException) { /* keep the partial result */ }
    }, ct);

    // Non-blocking wait: no .Wait()/.Result.
    await Task.WhenAll(producer, consumer);

    return count == 0
        ? TelemetryAggregate.Empty
        : new TelemetryAggregate(count, (double)sum / count, max, min);
}
```

Line-by-line walk-through. The line `using var cts = new CancellationTokenSource();` creates the source of external cancellation — it will be pulled by the Ctrl+C handler. The `Console.CancelKeyPress` subscription with `e.Cancel = true` is critical: without it the runtime kills the process instantly, and neither `finally` nor the registered callback run, which contradicts the very idea of cooperative cancellation. Next, `using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(10));` is exactly `CancelAfter` in constructor form: under the hood an internal timer starts that will call `Cancel()` after 10 seconds. This is precisely why the CTS must be disposed — otherwise the timer leaks, which is especially dangerous in long-lived services.

Linking via `CancellationTokenSource.CreateLinkedTokenSource(cts.Token, timeoutCts.Token)` implements cancellation-reason composition: the linked token fires if ANY of the sources is cancelled — either Ctrl+C or the timeout. This removes the need to manually call several `Cancel()` methods and synchronize between them. The `linked.Token.Register(...)` callback illustrates another cooperative-cancellation mechanism: subscription to the signal. It is important that the callback is short and reentrant — it may run on the thread that called `Cancel()` (here, the Ctrl+C handler thread).

`GenerateReadingsAsync` is annotated with `[EnumeratorCancellation]`: this attribute lets the token passed to `await foreach` (via `WithCancellation`) propagate correctly into the iterator method and fire when the enumeration is cancelled. Without the attribute, the formal `ct` parameter is not updated with the token from `WithCancellation`, and cancelling the enumerator does not cancel the iterator body. Inside the loop there is `ct.ThrowIfCancellationRequested()` — a frequent check reduces reaction latency.

The pipeline is built on `Channel.CreateBounded<int>` with `FullMode = Wait`: this gives backpressure without manual synchronization — the producer waits for room in the queue. The producer is wrapped in `try/catch (OperationCanceledException) { } finally { channel.Writer.Complete(); }`: `Complete` in `finally` is critical — without it the consumer hangs in `ReadAllAsync` forever. The consumer aggregates into closure variables (`count`, `sum`, `max`, `min`), and `OperationCanceledException` is caught locally so the partial result is not lost — this is cooperation: we do not throw away the accumulated data, we return it.

`await Task.WhenAll(producer, consumer)` is a non-blocking wait; no `.Wait()`/`.Result` that could deadlock the pool. In Main's `finally` we distinguish the stop reason via `timeoutCts.IsCancellationRequested`: true means the timeout fired first, false means Ctrl+C. This applies the best practice of "distinguishing own vs foreign cancellation" and avoids swallowing `OperationCanceledException` silently. Passing `ct` to `Task.Run(..., ct)` as the second argument is an optimization: if the token is already cancelled by start time, the task never begins, but this does not "kill" a running task, so the check inside the body is still required.

#### Going deeper (bonus)
1. Add a third cancellation source: "graceful shutdown" on `SIGTERM` (in .NET 8 — `AppDomain.CurrentDomain.ProcessExit` or a second Ctrl+C). Compose all three via linked tokens and verify that cleanup runs for any reason.
2. Implement a second consumer that writes the aggregate to a file every 100 items; on cancellation, flush the last chunk. Use `Channel.CreateBounded` with `SingleReader = false` and synchronize the flush via `SemaphoreSlim(1,1).WaitAsync(ct)`.
3. Replace `Task.Delay(5, ct)` in `GenerateReadingsAsync` with a real `HttpClient.GetStringAsync(url, ct)` call to a slow endpoint and show that the token is forwarded into the BCL call and cancellation arrives in milliseconds.
4. Add xUnit unit tests: verify that `RunPipelineAsync` completes in <500ms when cancelled after 100ms, and that a partial result is returned (count > 0).

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект собирается без warning-ов на .NET 8 / C# 12.
- [ ] (RU) Использованы top-level statements и linked tokens через `using`.
- [ ] (RU) Токен прокинут во все BCL/async-вызовы и циклы.
- [ ] (RU) Producer вызывает `channel.Writer.Complete()` в `finally`.
- [ ] (RU) Main различает причину остановки (таймаут / Ctrl+C) и не глотает OCE молча.
- [ ] (RU) Программа стабильно завершается и по Ctrl+C, и по таймауту за <1с.
- [ ] (EN) Project builds without warnings on .NET 8 / C# 12.
- [ ] (EN) Top-level statements and linked tokens via `using` are used.
- [ ] (EN) The token is forwarded into every BCL/async call and loop.
- [ ] (EN) The producer calls `channel.Writer.Complete()` in `finally`.
- [ ] (EN) Main distinguishes the stop reason (timeout / Ctrl+C) and does not swallow OCE silently.
- [ ] (EN) The program shuts down gracefully on both Ctrl+C and timeout within <1s.

#### Ресурсы / Resources
- [Microsoft Learn — Cancellation in Managed Threads](https://learn.microsoft.com/dotnet/standard/threading/cancellation-in-managed-threads)
- [Microsoft Learn — CancellationTokenSource.CreateLinkedTokenSource](https://learn.microsoft.com/dotnet/api/system.threading.cancellationtokensource.createlinkedtokensource)
- [Microsoft Learn — CancellationToken.Register](https://learn.microsoft.com/dotnet/api/system.threading.cancellationtoken.register)
- [Microsoft Learn — System.Threading.Channels](https://learn.microsoft.com/dotnet/api/system.threading.channels)
- [Stephen Toub — Async FAQ: CancellationToken](https://devblogs.microsoft.com/dotnet/how-do-i-cancel-non-cancelable-async-operations/)

---
[← К уроку M09-L06](lesson-M09-L06-cancellation-token.md) | [⬆ К модулю M09](../README.md) | [Следующее ДЗ →](homework-M09-L07-configureawait-synccontext.md)
