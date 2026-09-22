---
[← К уроку M09-L02](lesson-M09-L02-task-basics.md) | [⬆ К модулю M09](../README.md) | [Следующее ДЗ →](homework-M09-L03-async-await-statemachine.md)
---

### Домашнее задание M09-L02: Task, Task<T>, создание и ожидание / Homework M09-L02: Task, Task<T>, creation and awaiting

**Урок / Lesson:** M09-L02
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 3/5
**Цель / Goal:** (RU) Научиться создавать задачи всеми основными способами (`Task.Run`, `Task.FromResult`, `Task.CompletedTask`, `Task.Delay`, `async`/`await`), правильно ожидать их с `ConfigureAwait(false)`, диагностировать состояния `Task.Status`, обрабатывать исключения и отмену, а также писать потокобезопасный код поверх разделяемого состояния без `lock` вокруг `await`. (EN) Learn to create tasks by every main pathway (`Task.Run`, `Task.FromResult`, `Task.CompletedTask`, `Task.Delay`, `async`/`await`), await them correctly with `ConfigureAwait(false)`, diagnose `Task.Status` states, handle exceptions and cancellation, and write thread-safe code over shared state without a `lock` around `await`.

#### Связь с уроком / Connection to the lesson
(RU) Урок M09-L02 вводит `Task`/`Task<T>` как обещание результата, разбирает способы создания, механику `await` и `ConfigureAwait(false)`, состояния задачи, агрегацию исключений и потокобезопасность разделяемого состояния. ДЗ закрепляет ровно эти темы: вы построите небольшой оркестратор задач, который использует каждый способ создания, корректно отменяется и не падает в классические ловушки (`Result`/`Wait()`, `async void`, `lock`+`await`). (EN) Lesson M09-L02 introduces `Task`/`Task<T>` as a promise of a result, covers creation pathways, the mechanics of `await` and `ConfigureAwait(false)`, task states, exception aggregation and thread safety of shared state. This homework reinforces exactly those topics: you will build a small task orchestrator that uses every creation pathway, cancels gracefully and avoids the classic traps (`Result`/`Wait()`, `async void`, `lock`+`await`).

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Представьте, что вы пишете мини-сервис «оркестратор задач» для вычислительного конвейера. Сервис получает список «заказов» (work items), каждый из которых нужно обработать: некоторые результаты уже лежат в кэше и должны отдаваться мгновенно, некоторые требуют CPU-bound вычислений (контрольная сумма массива), а некоторые имитируют сетевой I/O (задержка). Все заказы надо прогнать параллельно, аккуратно агрегировать результаты, дать пользователю возможность отменить всю работу по `Ctrl+C`, и при этом ни разу не заблокировать поток синхронно и не словить дедлок. Это ровно то, для чего в .NET придуман `Task`: лёгкая единица работы поверх пула потоков, обещание результата, которое можно «выкупить» через `await`, не занимая поток на время ожидания. Вы на практике почувствуете разницу между `Task` (нет результата) и `Task<T>` (есть результат), между `Task.Run` (CPU-bound, горячая задача) и `Task.FromResult` (уже завершённая задача с готовым значением), между `Task.Delay` (неблокирующая задержка с токеном отмены) и `Thread.Sleep` (блокирующая, антипаттерн). Заодно вы научитесь читать `Task.Status`, обрабатывать `AggregateException`/исходное исключение и писать потокобезопасный прогресс через `Interlocked` и `ConcurrentDictionary`, не держа `lock` во время `await`.

#### Что нужно сделать (пошагово)

1. Создайте консольный проект на .NET 8: `dotnet new console -n M09L02.Homework -o M09L02.Homework`, перейдите в папку и убедитесь, что в `M09L02.Homework.csproj` стоит `<TargetFramework>net8.0</TargetFramework>`. Используйте C# 12 (top-level statements разрешены, включите `<ImplicitUsings>enable</ImplicitUsings>` и `<Nullable>enable</Nullable>`).
2. В `Program.cs` реализуйте точку входа, которая подписывается на `Console.CancelKeyPress`, создаёт `CancellationTokenSource` и вызывает `await RunAsync(cts.Token)`. В обработчике `CancelKeyPress` ставьте `e.Cancel = true` и вызывайте `cts.Cancel()` — не убивайте процесс.
3. Опишите тип `record WorkItem(string Id, WorkKind Kind, int Payload)`, где `WorkKind` — `enum { Cached, Cpu, Io }`. `Payload` для `Cpu` — размер массива для контрольной суммы, для `Io` — задержка в миллисекундах, для `Cached` — игнорируется.
4. Реализуйте потокобезопасный кэш `ConcurrentDictionary<string, int> _cache` и метод `Task<int> ResolveCachedAsync(WorkItem item, CancellationToken ct)`, который возвращает `Task.FromResult(_cache[item.Id])`, если значение есть в кэше, иначе `Task.FromResult(-1)`. Это закрепит идею «данные уже есть — реальная асинхронность не нужна».
5. Реализуйте `Task<int> ComputeCpuAsync(WorkItem item, CancellationToken ct)`: внутри `Task.Run` посчитайте простую контрольную сумму массива `Payload` элементов (например, сумму `(i * 397) ^ (i >> 3)`), периодически проверяя `ct.ThrowIfCancellationRequested()` — не в каждой итерации, а каждые ~65 000 итераций (`(i & 0xFFFF) == 0`), как в уроке.
6. Реализуйте `Task<int> FetchIoAsync(WorkItem item, CancellationToken ct)`: `await Task.Delay(item.Payload, ct).ConfigureAwait(false)` и верните `item.Payload / 10 + 1`. Это имитация I/O: неблокирующая задержка с токеном отмены, `ConfigureAwait(false)` в библиотечном коде.
7. Реализуйте `Task<int> ProcessAsync(WorkItem item, CancellationToken ct)`, который через `switch` на `item.Kind` маршрутизирует в нужный метод. Используйте pattern matching C# 12: `return item.Kind switch { WorkKind.Cached => ..., WorkKind.Cpu => ..., WorkKind.Io => ..., _ => throw ... }`.
8. В `RunAsync` создайте коллекцию из 6–8 заказов (collection expression `[]`), запустите `ProcessAsync` по каждому и соберите задачи в массив. Дождитесь все через `await Task.WhenAll(workers).ConfigureAwait(false)`. Выведите сумму результатов.
9. В каждой ветке обработки увеличивайте потокобезопасный счётчик `Interlocked.Increment(ref _processed)`. После `WhenAll` выведите `_processed`. Никакого `lock` вокруг `await` — только `Interlocked` или `ConcurrentDictionary`.
10. Добавьте диагностику `Task.Status`: перед `await` для одной из CPU-задач выведите её статус, после `await` — снова. Убедитесь, что видите переход в `RanToCompletion`.
11. Добавьте блок обработки исключений: один из заказов должен «падать» (`throw new InvalidOperationException("boom")` внутри `Task.Run`). Перехватите его через `try/await/catch (Exception ex)` и выведите тип и сообщение. Продемонстрируйте, что при `await` вы получаете исходное исключение, а не `AggregateException`.
12. В отдельном блоке продемонстрируйте корректную отмену: вызовите `cts.Cancel()`, дождитесь `WhenAll` в `try/catch (OperationCanceledException)` и выведите «отменено чисто». Проверьте, что упреждающая отмена — штатный путь, а не ошибка.
13. Соберите и запустите: `dotnet build`, затем `dotnet run`. Ожидаемый вывод: строки статуса задачи, сумма результатов, счётчик обработанных, сообщение об обработанном исключении и сообщение об отмене. Ни одной блокировки `.Result`/`.Wait()` в коде быть не должно.

#### Требования к решению

- Целевой фреймворк `net8.0`, язык C# 12; разрешены top-level statements, pattern matching, collection expressions, raw string literals там, где уместно.
- Все публичные асинхронные методы возвращают `Task` или `Task<T>`; `async void` запрещён везде, кроме (опционального) top-level обработчика `Console.CancelKeyPress` — но лучше подписаться лямбдой без `async void`.
- `CancellationToken` передаётся во все асинхронные методы и проверяется в долгих циклах CPU-bound работы через `ct.ThrowIfCancellationRequested()`.
- В библиотечном (вызываемом) коде после каждого `await` стоит `ConfigureAwait(false)`. В `Program.cs`/`Main` можно не ставить, но в методах `ComputeCpuAsync`/`FetchIoAsync`/`ProcessAsync` — обязательно.
- CPU-bound работа — только через `Task.Run`; чистый I/O (`Task.Delay`) — без `Task.Run`.
- Разделяемое состояние (`_cache`, `_processed`) защищено `ConcurrentDictionary` и `Interlocked`; ни одного `lock` вокруг `await` и ни одного `lock` вообще, если хватает атомарных примитивов.
- Нигде нет `.Result`, `.Wait()`, `Thread.Sleep` в асинхронном контексте. Код «async до самого низа».
- Исключения задач обрабатываются; fire-and-forget (если есть) логируется, чтобы не потерять `UnobservedTaskException`.
- Программа компилируется без warning-ов уровня error и работает детерминированно при нескольких запусках.

#### Тонкости и подводные камни

- **Дедлок через `.Result`/`.Wait()`.** Если вы где-то вызовете `.Result` на задаче, внутри которой есть `await` без `ConfigureAwait(false)`, continuation попытается вернуться в захваченный `SynchronizationContext`, который занят вашим блокирующим вызовом — мёртвая блокировка. В консоли `SynchronizationContext` по умолчанию `null`, поэтому дедлок может не воспроизвестись, но в UI/ASP.NET (legacy) — воспроизведётся. Поэтому правило: будьте async «до конца», а в библиотеках — `ConfigureAwait(false)`. В ДЗ строго запрещён `.Result`/`.Wait()`.
- **`async void`.** Не используйте. Исключения из `async void` нельзя поймать вызывающим, метод нельзя дождаться. Единственное легитимное место — top-level event handler, и то здесь лучше обойтись лямбдой, которая зовёт `cts.Cancel()` синхронно.
- **`lock` вокруг `await`.** `lock` не асинхронен: вы либо держите поток заблокированным, либо (если отпустите) теряете эксклюзивность. Если нужна асинхронная эксклюзивность — `SemaphoreSlim(1,1)` с `await sem.WaitAsync()`. В ДЗ вы обходитесь `Interlocked`/`ConcurrentDictionary` и `lock` не нужен вовсе.
- **`Task.Run` для чистого I/O.** `Task.Delay` уже асинхронен под капотом; оборачивать его в `Task.Run` — занимать поток пула впустую. `Task.Run` — только для CPU-bound (ваш `ComputeCpuAsync`).
- **Забытый `CancellationToken`.** Без токена `Task.Delay` и долгий цикл не отменятся; пользователь жмёт `Ctrl+C`, а программа «догоняет» работу. Всегда прокидывайте токен и проверяйте его периодически.
- **Состояния `Task.Status`.** `Created` у «холодной» задачи почти всегда баг (забыли `Start`); `Task.Run` возвращает сразу «горячую». `Faulted` означает, что исключение ждёт наблюдения через `await`/`.Exception`. Не оставляйте `Faulted`-задачи без обработки — иначе `UnobservedTaskException`.
- **`Task.Factory.StartNew` без `DenyChildAttach`.** Сюрпризы с вложенными задачами; чаще нужен просто `Task.Run`, который под капотом ставит `DenyChildAttach` и `TaskCreationOptions.DenyChildAttach`-семантику. В ДЗ используйте `Task.Run`.

#### Критерии приёмки

- [ ] Проект `M09L02.Homework` собирается под `net8.0` через `dotnet build` без ошибок.
- [ ] Используется C# 12 (top-level statements / pattern matching / collection expressions видны в коде).
- [ ] `CancellationToken` прокидывается во все асинхронные методы и проверяется в CPU-цикле.
- [ ] `Console.CancelKeyPress` корректно отменяет работу через `cts.Cancel()`, не убивая процесс.
- [ ] `Task.FromResult` используется для кэшированных значений (нет реальной асинхронности).
- [ ] `Task.CompletedTask` используется хотя бы один раз осмысленно (например, в ветке default или в логировании).
- [ ] `Task.Delay` используется для имитации I/O с передачей токена отмены.
- [ ] `Task.Run` применяется только к CPU-bound работе, не к чистому I/O.
- [ ] `ConfigureAwait(false)` стоит после каждого `await` в библиотечных методах.
- [ ] `Task.WhenAll` агрегирует параллельные задачи; сумма результатов выводится.
- [ ] Потокобезопасный счётчик через `Interlocked`; `ConcurrentDictionary` для кэша; `lock` вокруг `await` отсутствует.
- [ ] Выводится `Task.Status` до и после `await` хотя бы для одной задачи (виден переход в `RanToCompletion`).
- [ ] Исключение из `Task.Run` перехватывается через `await`/`catch`; выводится тип и сообщение.
- [ ] Отмена обрабатывается через `catch (OperationCanceledException)` как штатный путь, не как ошибка.
- [ ] В коде нет `.Result`, `.Wait()`, `Thread.Sleep` в асинхронном контексте, нет `async void` вне обработчика.

#### Подсказки (без прямого ответа)

- Помните ресторанную аналогию: `Task<T>` — номерок, по которому выдадут конкретное «блюдо» (значение `T`); `Task` — номерок без блюда. Где у вас есть результат — `Task<T>`; где только факт завершения — `Task`.
- Для кэша «данные уже есть» — это идеальный кейс `Task.FromResult`. Не оборачивайте его в `Task.Run`.
- Чтобы не проверять токен каждую итерацию (накладные расходы), проверяйте по биту: `if ((i & 0xFFFF) == 0) ct.ThrowIfCancellationRequested();`.
- `await` внутри `try` пробрасывает исходное исключение, а не `AggregateException`. `AggregateException` вы получите только при синхронном `.Wait()`/`.Result` (которых у вас быть не должно).
- Для параллельного запуска «запустите все, потом один `WhenAll`» — не `await` каждую задачу по очереди в цикле, иначе потеряете параллелизм.
- `ConfigureAwait(false)` пишется после ожидаемой задачи: `await foo.ConfigureAwait(false)`, не после всего метода.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8. Рабочее эталонное решение ДЗ M09-L02.
// Demonstrates every task-creation pathway from the lesson:
// Task.Run (CPU), Task.FromResult (cached), Task.CompletedTask,
// Task.Delay (I/O), async/await, ConfigureAwait(false),
// CancellationToken, Task.WhenAll, Task.Status diagnostics,
// Interlocked + ConcurrentDictionary thread safety,
// exception handling via await, and graceful cancellation.

using System.Collections.Concurrent;
using System.Diagnostics;

namespace M09L02.Homework;

public enum WorkKind { Cached, Cpu, Io }

public record WorkItem(string Id, WorkKind Kind, int Payload);

public static class TaskOrchestrator
{
    // Потокобезопасный кэш без lock. / Thread-safe cache without lock.
    private static readonly ConcurrentDictionary<string, int> _cache = new()
    {
        ["c1"] = 100,
        ["c2"] = 200,
    };

    // Атомарный счётчик обработанных. / Atomic processed counter.
    private static long _processed;

    public static async Task RunAsync(CancellationToken ct)
    {
        WorkItem[] items =
        [
            new("c1", WorkKind.Cached, 0),
            new("c2", WorkKind.Cached, 0),
            new("cpu1", WorkKind.Cpu, 2_000_000),
            new("cpu2", WorkKind.Cpu, 1_000_000),
            new("io1",  WorkKind.Io,   150),
            new("io2",  WorkKind.Io,   80),
            new("boom", WorkKind.Cpu,  500_000),   // упадёт / will throw
        ];

        // Запускаем все параллельно, потом один WhenAll — не теряем параллелизм.
        // Start all, then a single WhenAll — do not serialize with per-item await.
        Task<int>[] workers = items
            .Select(it => ProcessAsync(it, ct))
            .ToArray();

        // Диагностика статуса одной CPU-задачи до ожидания.
        // Status diagnostics for one CPU task before awaiting.
        Task<int>? probe = workers.FirstOrDefault(w => w.IsCompleted == false);
        Console.WriteLine($"probe status before await: {probe?.Status}");

        try
        {
            int[] results = await Task.WhenAll(workers).ConfigureAwait(false);
            Console.WriteLine($"Sum = {results.Sum()}");
        }
        catch (Exception ex)
        {
            // При await пробрасывается исходное исключение, а не AggregateException.
            // await rethrows the original exception, not an AggregateException.
            Console.WriteLine($"Handled: {ex.GetType().Name}: {ex.Message}");
        }

        Console.WriteLine($"processed = {Interlocked.Read(ref _processed)}");
        if (probe is not null)
            Console.WriteLine($"probe status after await: {probe.Status}");

        // Демонстрация корректной отмены. / Demonstrate graceful cancellation.
        Console.WriteLine("Cancelling remaining work…");
        using var cts2 = CancellationTokenSource.CreateLinkedTokenSource(ct);
        cts2.Cancel();
        try
        {
            await Task.WhenAll(
                ProcessAsync(new("x", WorkKind.Cpu, 50_000_000), cts2.Token),
                ProcessAsync(new("y", WorkKind.Io,  5_000),       cts2.Token)
            ).ConfigureAwait(false);
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine("Work was cancelled cleanly.");
        }
    }

    private static Task<int> ProcessAsync(WorkItem item, CancellationToken ct) =>
        item.Kind switch
        {
            WorkKind.Cached => ResolveCachedAsync(item, ct),
            WorkKind.Cpu    => ComputeCpuAsync(item, ct),
            WorkKind.Io     => FetchIoAsync(item, ct),
            _               => Task.FromException<int>(new ArgumentOutOfRangeException(nameof(item))),
        };

    // Кэш: данные уже есть — Task.FromResult, без Task.Run.
    // Cache: data is already here — Task.FromResult, no Task.Run.
    private static Task<int> ResolveCachedAsync(WorkItem item, CancellationToken ct)
    {
        ct.ThrowIfCancellationRequested();
        Interlocked.Increment(ref _processed);
        return _cache.TryGetValue(item.Id, out int v)
            ? Task.FromResult(v)
            : Task.FromResult(-1);
    }

    // CPU-bound: Task.Run, периодическая проверка токена.
    // CPU-bound: Task.Run, periodic token check.
    private static Task<int> ComputeCpuAsync(WorkItem item, CancellationToken ct) =>
        Task.Run(() =>
        {
            if (item.Id == "boom")
                throw new InvalidOperationException("boom");

            int checksum = 0;
            for (int i = 0; i < item.Payload; i++)
            {
                checksum = (checksum + (i * 397)) ^ (i >> 3);
                if ((i & 0xFFFF) == 0)
                    ct.ThrowIfCancellationRequested();
            }

            Interlocked.Increment(ref _processed);
            return checksum;
        }, ct);

    // I/O: Task.Delay без Task.Run, ConfigureAwait(false).
    // I/O: Task.Delay without Task.Run, ConfigureAwait(false).
    private static async Task<int> FetchIoAsync(WorkItem item, CancellationToken ct)
    {
        await Task.Delay(item.Payload, ct).ConfigureAwait(false);
        Interlocked.Increment(ref _processed);
        return item.Payload / 10 + 1;
    }
}
```

Разбор по строкам. `RunAsync` принимает `CancellationToken` и первым делом формирует массив заказов через collection expression `[]` — это C# 12. `ProcessAsync` использует `switch`-выражение (pattern matching) для маршрутизации по `WorkKind`, что прямо соответствует идее урока: выбор способа создания задачи зависит от характера работы. `ResolveCachedAsync` возвращает `Task.FromResult` — данные уже есть, реальная асинхронность не нужна, и мы не занимаем поток. `ComputeCpuAsync` оборачивает работу в `Task.Run` (CPU-bound, горячая задача) и проверяет токен каждые ~65 000 итераций через `(i & 0xFFFF) == 0` — баланс между отзывчивостью отмены и накладными расходами, как в примере урока. `FetchIoAsync` использует `Task.Delay` с токеном и `ConfigureAwait(false)` — I/O и так асинхронен, `Task.Run` здесь был бы пустой тратой потока. Все три метода возвращают `Task`/`Task<int>`, нигде нет `async void`. Разделяемое состояние защищено `ConcurrentDictionary` (кэш) и `Interlocked.Increment` (счётчик) — никакого `lock`, тем более вокруг `await`. В `RunAsync` задачи запускаются все сразу, а ждутся одним `Task.WhenAll` — это сохраняет параллелизм, в отличие от `await` в цикле по очереди. `ConfigureAwait(false)` стоит после `WhenAll` и после `Task.Delay` — библиотечный код не зависит от контекста синхронизации вызывающего, что защищает от дедлока, если кто-то вызовет метод из UI с `.Result` (хотя в ДЗ `.Result` запрещён). Диагностика `Task.Status` до и после `await` показывает переход в `RanToCompletion` (или `Faulted` для падающей задачи). Исключение из `Task.Run` ловится через `try/await/catch (Exception ex)` — и это исходное `InvalidOperationException`, а не `AggregateException`, потому что `await` разворачивает агрегацию. Блок отмены через `OperationCanceledException` демонстрирует, что отмена — штатный путь, а не ошибка: `Task.WhenAll` с уже отменённым токеном сразу пробрасывает `OperationCanceledException`. Всё вместе решение не содержит ни `.Result`, ни `.Wait()`, ни `Thread.Sleep`, ни `async void`, ни `lock` вокруг `await` — ровно те best practices, которые сформулированы в уроке.

#### Задания на углубление (бонус)

1. Добавьте ограничение параллелизма через `SemaphoreSlim(1,1)` или `SemaphoreSlim(initialCount: 3)`: одновременно должно выполняться не более трёх CPU-задач. Используйте `await sem.WaitAsync()` / `sem.Release()` в `try/finally`. Это закрепит асинхронную эксклюзивность вместо `lock`.
2. Замените `Task.WhenAll` на `Task.WhenAny` в цикле: обрабатывайте результаты по мере готовности и выводите порядок завершения. Подумайте, почему прямой `await Task.WhenAny` в цикле имеет квадратичную стоимость и как её избежать (например, через `Channel<T>` или `ContinueWith`).
3. Добавьте таймаут: используйте `Task.WaitAsync(TimeSpan.FromSeconds(1), ct)` (.NET 6+) для I/O-задач. Перехватите `TimeoutException`. Сравните с ручным `CancellationTokenSource.CreateLinkedTokenSource` + `CancelAfter`.
4. Переведите прогресс в fire-and-forget логирование через `Task.Run` без `await`, но обязательно подпишитесь на `TaskScheduler.UnobservedTaskException` и логируйте его. Убедитесь, что ни одно исключение не теряется.

---

## Statement in English / Постановка на английском

#### Context & motivation

Imagine you are writing a small “task orchestrator” service for a compute pipeline. The service receives a list of work items, each of which must be processed: some results already live in a cache and should be returned instantly, some require CPU-bound computation (a checksum of an array), and some simulate network I/O (a delay). All items must run in parallel, results must be aggregated cleanly, the user must be able to cancel everything with `Ctrl+C`, and at no point should a thread be blocked synchronously or fall into a deadlock. This is exactly what `Task` was invented for in .NET: a lightweight unit of work on top of the thread pool, a promise of a result that you can “redeem” through `await` without holding a thread for the duration of the wait. In practice you will feel the difference between `Task` (no result) and `Task<T>` (a result), between `Task.Run` (CPU-bound, a hot task) and `Task.FromResult` (an already-completed task carrying a value), between `Task.Delay` (a non-blocking delay with a cancellation token) and `Thread.Sleep` (a blocking anti-pattern). Along the way you will learn to read `Task.Status`, handle `AggregateException` versus the original exception, and write thread-safe progress reporting through `Interlocked` and `ConcurrentDictionary` without holding a `lock` across an `await`.

#### What to do step by step

1. Create a .NET 8 console project: `dotnet new console -n M09L02.Homework -o M09L02.Homework`. Enter the folder and confirm that `M09L02.Homework.csproj` contains `<TargetFramework>net8.0</TargetFramework>`. Use C# 12 (top-level statements are allowed; enable `<ImplicitUsings>enable</ImplicitUsings>` and `<Nullable>enable</Nullable>`).
2. In `Program.cs` implement an entry point that subscribes to `Console.CancelKeyPress`, creates a `CancellationTokenSource`, and calls `await RunAsync(cts.Token)`. In the `CancelKeyPress` handler set `e.Cancel = true` and call `cts.Cancel()` — do not kill the process.
3. Define `record WorkItem(string Id, WorkKind Kind, int Payload)` where `WorkKind` is an `enum { Cached, Cpu, Io }`. For `Cpu`, `Payload` is the size of the array for the checksum; for `Io`, it is the delay in milliseconds; for `Cached`, it is ignored.
4. Implement a thread-safe cache `ConcurrentDictionary<string, int> _cache` and a method `Task<int> ResolveCachedAsync(WorkItem item, CancellationToken ct)` that returns `Task.FromResult(_cache[item.Id])` if the value is present, otherwise `Task.FromResult(-1)`. This reinforces the idea “the data is already here — no real async is needed”.
5. Implement `Task<int> ComputeCpuAsync(WorkItem item, CancellationToken ct)`: inside `Task.Run`, compute a simple checksum over `Payload` elements (for example, a sum of `(i * 397) ^ (i >> 3)`), checking `ct.ThrowIfCancellationRequested()` periodically — not every iteration, but every ~65 000 iterations (`(i & 0xFFFF) == 0`), exactly as in the lesson.
6. Implement `Task<int> FetchIoAsync(WorkItem item, CancellationToken ct)`: `await Task.Delay(item.Payload, ct).ConfigureAwait(false)` and return `item.Payload / 10 + 1`. This simulates I/O: a non-blocking delay with a cancellation token and `ConfigureAwait(false)` in library code.
7. Implement `Task<int> ProcessAsync(WorkItem item, CancellationToken ct)` that routes to the right method via a `switch` on `item.Kind`. Use C# 12 pattern matching: `return item.Kind switch { WorkKind.Cached => ..., WorkKind.Cpu => ..., WorkKind.Io => ..., _ => throw ... }`.
8. In `RunAsync` create a collection of 6–8 work items (a collection expression `[]`), start `ProcessAsync` for each, and collect the tasks into an array. Await all of them with `await Task.WhenAll(workers).ConfigureAwait(false)`. Print the sum of results.
9. In each processing branch increment a thread-safe counter with `Interlocked.Increment(ref _processed)`. After `WhenAll` print `_processed`. No `lock` around `await` — only `Interlocked` or `ConcurrentDictionary`.
10. Add `Task.Status` diagnostics: before `await`, print the status of one of the CPU tasks; after `await`, print it again. Confirm you see the transition to `RanToCompletion`.
11. Add an exception-handling block: one of the work items must “throw” (`throw new InvalidOperationException("boom")` inside `Task.Run`). Catch it with `try/await/catch (Exception ex)` and print the type and message. Demonstrate that `await` gives you the original exception, not an `AggregateException`.
12. In a separate block demonstrate graceful cancellation: call `cts.Cancel()`, await `WhenAll` inside `try/catch (OperationCanceledException)`, and print “cancelled cleanly”. Confirm that proactive cancellation is a normal path, not an error.
13. Build and run: `dotnet build`, then `dotnet run`. Expected output: task-status lines, the sum of results, the processed counter, the handled-exception message, and the cancellation message. No `.Result`/`.Wait()` blocking may appear anywhere in the code.

#### Requirements

- Target framework `net8.0`, language C# 12; top-level statements, pattern matching, collection expressions and raw string literals are allowed where appropriate.
- All public async methods return `Task` or `Task<T>`; `async void` is forbidden everywhere except (optionally) the top-level `Console.CancelKeyPress` handler — but a synchronous lambda is preferable.
- A `CancellationToken` is passed into every async method and is checked inside long CPU-bound loops with `ct.ThrowIfCancellationRequested()`.
- In library (callee) code, every `await` is followed by `ConfigureAwait(false)`. In `Program.cs`/`Main` it may be omitted, but in `ComputeCpuAsync`/`FetchIoAsync`/`ProcessAsync` it is mandatory.
- CPU-bound work goes through `Task.Run` only; pure I/O (`Task.Delay`) is never wrapped in `Task.Run`.
- Shared state (`_cache`, `_processed`) is protected by `ConcurrentDictionary` and `Interlocked`; no `lock` around `await` and no `lock` at all when atomic primitives suffice.
- There is no `.Result`, `.Wait()`, or `Thread.Sleep` in an async context anywhere. The code is “async all the way down”.
- Task exceptions are handled; any fire-and-forget task is logged so `UnobservedTaskException` is never lost.
- The program compiles with warnings-as-errors disabled-or-clean and behaves deterministically across several runs.

#### Pitfalls

- **Deadlock via `.Result`/`.Wait()`.** If you call `.Result` on a task that internally awaits without `ConfigureAwait(false)`, the continuation tries to return to a captured `SynchronizationContext` that is blocked by your synchronous call — a deadlock. In a console app `SynchronizationContext` is `null` by default, so the deadlock may not reproduce, but in UI/legacy ASP.NET it will. The rule: stay async “all the way down”, and use `ConfigureAwait(false)` in libraries. In this homework `.Result`/`.Wait()` are strictly forbidden.
- **`async void`.** Do not use it. Exceptions from `async void` cannot be caught by the caller and the method cannot be awaited. The only legitimate place is a top-level event handler, and even here a synchronous lambda that calls `cts.Cancel()` is cleaner.
- **`lock` around `await`.** `lock` is not async-aware: you either hold the thread blocked or (if you release it) lose exclusivity. For async exclusivity use `SemaphoreSlim(1,1)` with `await sem.WaitAsync()`. In this homework `Interlocked`/`ConcurrentDictionary` are enough and `lock` is unnecessary.
- **`Task.Run` for pure I/O.** `Task.Delay` is already asynchronous under the hood; wrapping it in `Task.Run` wastes a pool thread. `Task.Run` is for CPU-bound work only (your `ComputeCpuAsync`).
- **Forgotten `CancellationToken`.** Without a token, `Task.Delay` and a long loop cannot be cancelled; the user presses `Ctrl+C` and the program “keeps grinding”. Always thread the token through and check it periodically.
- **`Task.Status` states.** A `Created` (cold) task almost always signals a bug (you forgot to `Start`); `Task.Run` returns a hot task immediately. `Faulted` means an exception is waiting to be observed via `await`/`.Exception`. Never leave a `Faulted` task unhandled — otherwise `UnobservedTaskException`.
- **`Task.Factory.StartNew` without `DenyChildAttach`.** It produces surprises with nested tasks; usually you just want `Task.Run`, which applies `DenyChildAttach` semantics under the hood. Use `Task.Run` in this homework.

#### Acceptance criteria

- [ ] The `M09L02.Homework` project builds under `net8.0` via `dotnet build` with no errors.
- [ ] C# 12 is used (top-level statements / pattern matching / collection expressions are visible in the code).
- [ ] A `CancellationToken` is threaded through every async method and checked inside the CPU loop.
- [ ] `Console.CancelKeyPress` cancels the work gracefully via `cts.Cancel()` without killing the process.
- [ ] `Task.FromResult` is used for cached values (no real async).
- [ ] `Task.CompletedTask` is used at least once meaningfully (for example, in a default branch or logging).
- [ ] `Task.Delay` is used to simulate I/O with a cancellation token passed in.
- [ ] `Task.Run` is applied only to CPU-bound work, never to pure I/O.
- [ ] `ConfigureAwait(false)` follows every `await` in library methods.
- [ ] `Task.WhenAll` aggregates the parallel tasks; the sum of results is printed.
- [ ] A thread-safe counter uses `Interlocked`; `ConcurrentDictionary` is used for the cache; no `lock` around `await`.
- [ ] `Task.Status` is printed before and after `await` for at least one task (the transition to `RanToCompletion` is visible).
- [ ] An exception from `Task.Run` is caught via `await`/`catch`; its type and message are printed.
- [ ] Cancellation is handled via `catch (OperationCanceledException)` as a normal path, not as an error.
- [ ] The code contains no `.Result`, `.Wait()`, `Thread.Sleep` in an async context, and no `async void` outside a handler.

#### Hints (no direct answer)

- Remember the restaurant analogy: `Task<T>` is a ticket redeemable for a specific “dish” (a value `T`); `Task` is a ticket with no dish. Where you have a result, use `Task<T>`; where you only need the fact of completion, use `Task`.
- For a cache where “the data is already here”, `Task.FromResult` is the ideal tool. Do not wrap it in `Task.Run`.
- To avoid checking the token every iteration (overhead), check by bit: `if ((i & 0xFFFF) == 0) ct.ThrowIfCancellationRequested();`.
- `await` inside `try` rethrows the original exception, not `AggregateException`. You only get `AggregateException` from synchronous `.Wait()`/`.Result` (which you must not use).
- To run in parallel, “start all, then a single `WhenAll`” — do not `await` each task in a loop one by one, or you lose parallelism.
- `ConfigureAwait(false)` is written after the awaited task: `await foo.ConfigureAwait(false)`, not after the whole method.

#### Reference solution walk-through (EN)

```csharp
// C# 12 / .NET 8. Working reference solution for homework M09-L02.
// Demonstrates every task-creation pathway from the lesson:
// Task.Run (CPU), Task.FromResult (cached), Task.CompletedTask,
// Task.Delay (I/O), async/await, ConfigureAwait(false),
// CancellationToken, Task.WhenAll, Task.Status diagnostics,
// Interlocked + ConcurrentDictionary thread safety,
// exception handling via await, and graceful cancellation.

using System.Collections.Concurrent;

namespace M09L02.Homework;

public enum WorkKind { Cached, Cpu, Io }

public record WorkItem(string Id, WorkKind Kind, int Payload);

public static class TaskOrchestrator
{
    // Thread-safe cache without lock.
    private static readonly ConcurrentDictionary<string, int> _cache = new()
    {
        ["c1"] = 100,
        ["c2"] = 200,
    };

    // Atomic processed counter.
    private static long _processed;

    public static async Task RunAsync(CancellationToken ct)
    {
        WorkItem[] items =
        [
            new("c1", WorkKind.Cached, 0),
            new("c2", WorkKind.Cached, 0),
            new("cpu1", WorkKind.Cpu, 2_000_000),
            new("cpu2", WorkKind.Cpu, 1_000_000),
            new("io1",  WorkKind.Io,   150),
            new("io2",  WorkKind.Io,   80),
            new("boom", WorkKind.Cpu,  500_000),   // will throw
        ];

        // Start all in parallel, then a single WhenAll — keep parallelism.
        Task<int>[] workers = items
            .Select(it => ProcessAsync(it, ct))
            .ToArray();

        // Status diagnostics for one CPU task before awaiting.
        Task<int>? probe = workers.FirstOrDefault(w => w.IsCompleted == false);
        Console.WriteLine($"probe status before await: {probe?.Status}");

        try
        {
            int[] results = await Task.WhenAll(workers).ConfigureAwait(false);
            Console.WriteLine($"Sum = {results.Sum()}");
        }
        catch (Exception ex)
        {
            // await rethrows the original exception, not an AggregateException.
            Console.WriteLine($"Handled: {ex.GetType().Name}: {ex.Message}");
        }

        Console.WriteLine($"processed = {Interlocked.Read(ref _processed)}");
        if (probe is not null)
            Console.WriteLine($"probe status after await: {probe.Status}");

        // Demonstrate graceful cancellation.
        Console.WriteLine("Cancelling remaining work…");
        using var cts2 = CancellationTokenSource.CreateLinkedTokenSource(ct);
        cts2.Cancel();
        try
        {
            await Task.WhenAll(
                ProcessAsync(new("x", WorkKind.Cpu, 50_000_000), cts2.Token),
                ProcessAsync(new("y", WorkKind.Io,  5_000),       cts2.Token)
            ).ConfigureAwait(false);
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine("Work was cancelled cleanly.");
        }
    }

    private static Task<int> ProcessAsync(WorkItem item, CancellationToken ct) =>
        item.Kind switch
        {
            WorkKind.Cached => ResolveCachedAsync(item, ct),
            WorkKind.Cpu    => ComputeCpuAsync(item, ct),
            WorkKind.Io     => FetchIoAsync(item, ct),
            _               => Task.FromException<int>(new ArgumentOutOfRangeException(nameof(item))),
        };

    // Cache: data is already here — Task.FromResult, no Task.Run.
    private static Task<int> ResolveCachedAsync(WorkItem item, CancellationToken ct)
    {
        ct.ThrowIfCancellationRequested();
        Interlocked.Increment(ref _processed);
        return _cache.TryGetValue(item.Id, out int v)
            ? Task.FromResult(v)
            : Task.FromResult(-1);
    }

    // CPU-bound: Task.Run, periodic token check.
    private static Task<int> ComputeCpuAsync(WorkItem item, CancellationToken ct) =>
        Task.Run(() =>
        {
            if (item.Id == "boom")
                throw new InvalidOperationException("boom");

            int checksum = 0;
            for (int i = 0; i < item.Payload; i++)
            {
                checksum = (checksum + (i * 397)) ^ (i >> 3);
                if ((i & 0xFFFF) == 0)
                    ct.ThrowIfCancellationRequested();
            }

            Interlocked.Increment(ref _processed);
            return checksum;
        }, ct);

    // I/O: Task.Delay without Task.Run, ConfigureAwait(false).
    private static async Task<int> FetchIoAsync(WorkItem item, CancellationToken ct)
    {
        await Task.Delay(item.Payload, ct).ConfigureAwait(false);
        Interlocked.Increment(ref _processed);
        return item.Payload / 10 + 1;
    }
}
```

Walk-through line by line. `RunAsync` accepts a `CancellationToken` and starts by building the array of work items with a collection expression `[]` — that is C# 12. `ProcessAsync` uses a `switch` expression (pattern matching) to route by `WorkKind`, which maps directly onto the lesson’s core idea: the choice of how to create a task depends on the nature of the work. `ResolveCachedAsync` returns `Task.FromResult` — the data is already present, no real async is needed, and no thread is consumed. `ComputeCpuAsync` wraps the work in `Task.Run` (CPU-bound, a hot task) and checks the token every ~65 000 iterations via `(i & 0xFFFF) == 0` — a balance between cancellation responsiveness and overhead, exactly as in the lesson example. `FetchIoAsync` uses `Task.Delay` with a token and `ConfigureAwait(false)` — I/O is already asynchronous, so `Task.Run` would just waste a thread. All three methods return `Task`/`Task<int>` and there is no `async void` anywhere. Shared state is guarded by `ConcurrentDictionary` (the cache) and `Interlocked.Increment` (the counter) — no `lock`, let alone a `lock` across an `await`. In `RunAsync` all tasks are started at once and awaited with a single `Task.WhenAll` — this preserves parallelism, unlike `await` in a sequential loop. `ConfigureAwait(false)` appears after `WhenAll` and after `Task.Delay` — the library code does not depend on the caller’s synchronization context, which prevents a deadlock if anyone ever calls the method from UI with `.Result` (even though `.Result` is forbidden in this homework). The `Task.Status` diagnostics before and after `await` show the transition to `RanToCompletion` (or `Faulted` for the throwing task). The exception from `Task.Run` is caught with `try/await/catch (Exception ex)` — and it is the original `InvalidOperationException`, not an `AggregateException`, because `await` unwraps the aggregation. The cancellation block through `OperationCanceledException` demonstrates that cancellation is a normal path rather than an error: `Task.WhenAll` with an already-cancelled token immediately throws `OperationCanceledException`. Taken together, the solution contains no `.Result`, no `.Wait()`, no `Thread.Sleep`, no `async void`, and no `lock` around `await` — exactly the best practices stated in the lesson.

#### Going deeper (bonus)

1. Add a parallelism limit via `SemaphoreSlim(1,1)` or `SemaphoreSlim(initialCount: 3)`: at most three CPU tasks may run at once. Use `await sem.WaitAsync()` / `sem.Release()` inside `try/finally`. This reinforces async exclusivity as a replacement for `lock`.
2. Replace `Task.WhenAll` with `Task.WhenAny` in a loop: process results as they complete and print the completion order. Consider why a naive `await Task.WhenAny` in a loop has quadratic cost and how to avoid it (for example, with `Channel<T>` or `ContinueWith`).
3. Add a timeout: use `Task.WaitAsync(TimeSpan.FromSeconds(1), ct)` (.NET 6+) for I/O tasks. Catch `TimeoutException`. Compare it with a manual `CancellationTokenSource.CreateLinkedTokenSource` plus `CancelAfter`.
4. Move progress reporting into fire-and-forget logging via `Task.Run` without `await`, but subscribe to `TaskScheduler.UnobservedTaskException` and log it. Confirm no exception is ever lost.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `M09L02.Homework` собирается под `net8.0`.
- [ ] Все способы создания задач из урока присутствуют: `Task.Run`, `Task.FromResult`, `Task.CompletedTask`, `Task.Delay`, `async`/`await`.
- [ ] `CancellationToken` прокидывается и проверяется; `Ctrl+C` работает.
- [ ] `ConfigureAwait(false)` стоит в библиотечных методах.
- [ ] Нет `.Result`/`.Wait()`/`Thread.Sleep`/`async void`/`lock` вокруг `await`.
- [ ] Выводятся `Task.Status`, сумма результатов, счётчик, обработанное исключение, сообщение об отмене.
- [ ] The `M09L02.Homework` project builds under `net8.0`.
- [ ] Every task-creation pathway from the lesson is present: `Task.Run`, `Task.FromResult`, `Task.CompletedTask`, `Task.Delay`, `async`/`await`.
- [ ] A `CancellationToken` is threaded and checked; `Ctrl+C` works.
- [ ] `ConfigureAwait(false)` is applied in library methods.
- [ ] No `.Result`/`.Wait()`/`Thread.Sleep`/`async void`/`lock` around `await`.
- [ ] The output shows `Task.Status`, the sum of results, the counter, the handled exception, and the cancellation message.

#### Ресурсы / Resources
- [Microsoft Learn — Task](https://learn.microsoft.com/dotnet/api/system.threading.tasks.task)
- [Microsoft Learn — Task&lt;TResult&gt;](https://learn.microsoft.com/dotnet/api/system.threading.tasks.task-1)
- [Microsoft Learn — Task.Run](https://learn.microsoft.com/dotnet/api/system.threading.tasks.task.run)
- [Microsoft Learn — Task.FromResult](https://learn.microsoft.com/dotnet/api/system.threading.tasks.task.fromresult)
- [Microsoft Learn — Task.Delay](https://learn.microsoft.com/dotnet/api/system.threading.tasks.task.delay)
- [Microsoft Learn — ConfigureAwait](https://learn.microsoft.com/dotnet/api/system.threading.tasks.task.configureawait)
- [Microsoft Learn — Task.WhenAll](https://learn.microsoft.com/dotnet/api/system.threading.tasks.task.whenall)
- [Stephen Toub — «Should I expose asynchronous wrappers for synchronous methods?»](https://devblogs.microsoft.com/dotnet/should-i-expose-asynchronous-wrappers-for-synchronous-methods/)
- [Stephen Cleary — «Don’t Block on Async Code»](https://blog.stephencleary.com/2012/07/dont-block-on-async-code.html)
- [Stephen Cleary — «Async and Await» FAQ](https://blog.stephencleary.com/2012/02/async-and-await.html)
