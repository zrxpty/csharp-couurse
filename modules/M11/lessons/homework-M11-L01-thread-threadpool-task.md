---
[← К уроку M11-L01](lesson-M11-L01-thread-threadpool-task.md) | [⬆ К модулю M11](../README.md) | [Следующее ДЗ →](homework-M11-L02-lock-monitor.md)
---

### Домашнее задание M11-L01: Thread (исторически), пул потоков, почему Task / Homework M11-L01: Thread (historical), thread pool, why Task

**Урок / Lesson:** M11-L01
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) На практике почувствовать разницу между `Thread`, `ThreadPool` и `Task`, научиться выбирать правильный примитив, безопасно отменять долгие операции, избегать голодания пула и дедлоков `.Result`, а также корректно защищать разделяемое состояние в присутствии `await`. (EN) Get hands-on experience with the difference between `Thread`, `ThreadPool`, and `Task`; learn to choose the right primitive, cancel long operations safely, avoid pool starvation and `.Result` deadlocks, and protect shared state correctly in the presence of `await`.

#### Связь с уроком / Connection to the lesson
(RU) Урок показал, что `Thread` — дорогая обёртка над потоком ОС, `ThreadPool` переиспользует потоки, а `Task` добавляет композицию, отмену и обработку исключений. В этом ДЗ вы построите небольшую лабораторию «фонового обработчика», в которой на собственном опыте столкнётесь со всеми главными ловушками урока: foreground-поток, удерживающий процесс; голодание пула через блокирующий `Task.Run`; дедлок `.Result`; смешение `lock` и `await`; и правильный путь через `SemaphoreSlim`, `Interlocked`, `ConcurrentQueue` и `CancellationToken`.
(EN) The lesson showed that `Thread` is an expensive wrapper over an OS thread, `ThreadPool` reuses threads, and `Task` adds composition, cancellation, and exception handling. In this homework you will build a small "background processor" lab where you personally hit every major trap from the lesson: a foreground thread keeping the process alive; pool starvation via blocking `Task.Run`; a `.Result` deadlock; mixing `lock` with `await`; and the correct path through `SemaphoreSlim`, `Interlocked`, `ConcurrentQueue`, and `CancellationToken`.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы присоединились к команде, пишущей сервис «фоновых джобов» для небольшой платформы. В кодовой базе до вас успели написать три версии одного и того же обработчика очереди задач — на голом `Thread`, на `ThreadPool.QueueUserWorkItem` и на `Task.Run`. Все три «работают» в лабораторных условиях, но в продакшене периодически случаются странные вещи: процесс сервиса «зависает» в диспетчере задач после остановки; при всплеске нагрузки обработка внезапно тормозит на десятки секунд; иногда сервис вообще не отвечает на запросы, пока его не перезапустят. Тимлид просит вас разобраться, почему так происходит, и написать эталонную реализацию, которая ведет себя предсказуемо.

Эта лаборатория — не учебный пример ради примера. Каждая ловушка, с которой вы столкнётесь, напрямую перекликается с лучшими практиками урока: дороговизна создания `Thread` (около 1 МБ стека и переключения контекста ядра), эвристики роста пула CLR (примерно 1 поток на 0.5 секунды для CPU-bound работы), разделение на worker-потоки и IOCP/IO-потоки, кооперативная отмена через `CancellationToken`, и фундаментальный запрет блокировать поток пула через `.Result`/`.Wait()` в контексте синхронизации. Вы должны не просто «сделать, чтобы компилировалось», а сознательно выбрать примитив под каждую подзадачу и быть готовым объяснить, почему именно этот.

К концу задания у вас должен быть консольный проект `.NET 8`, который запускает серию мини-экспериментов, печатает тайминги и наблюдения, и демонстрирует как «плохие», так и «хорошие» подходы рядом — чтобы разница была видна невооружённым глазом. Это и есть лучшее доказательство того, что вы поняли материал урока: не цитата из документации, а живой измеримый контраст между голоданием пула и неблокирующей реализацией.

#### Что нужно сделать (пошагово)
1. Создайте новый консольный проект: `dotnet new console -n ConcurrencyLab -o ConcurrencyLab` и перейдите в него: `cd ConcurrencyLab`. Убедитесь, что в `ConcurrencyLab.csproj` указаны `<TargetFramework>net8.0</TargetFramework>` и `<LangVersion>latest</LangVersion>`. При необходимости добавьте `<Nullable>enable</Nullable>`. Запустите пустой проект: `dotnet run` — вы должны увидеть пустой вывод без ошибок.
2. В `Program.cs` используйте top-level statements (C# 12). Первой строкой выведите `Environment.ProcessorCount` — это естественная подсказка уровня параллелизма, упомянутая в уроке. Запомните это число: оно понадобится для задания числа параллельных задач в нескольких экспериментах.
3. Реализуйте метод `RunForegroundVsBackgroundDemo()`, который запускает два `Thread`: один с `IsBackground = false` (по умолчанию), другой с `IsBackground = true`. Оба спят `Thread.Sleep(600)` и печатают сообщение по завершении. Запустите метод и убедитесь: главный поток завершится, но процесс будет ждать foreground-поток. Прокомментируйте в коде, почему так происходит и почему потоки пула (`IsBackground = true` по умолчанию) такой ловушки не создают.
4. Реализуйте `RunThreadPoolQueueDemo()`, использующий `ThreadPool.QueueUserWorkItem`. В колбэке напечатайте `Thread.CurrentThread.IsBackground` и `Thread.CurrentThread.ManagedThreadId` для двух разныхenqueue-вызовов. Подумайте и зафиксируйте в комментарии: почему `QueueUserWorkItem` не возвращает результат и не даёт отмену, и что это означает для дизайна API.
5. Реализуйте `RunBlockingStarvationDemo(int n)` — «плохой» эксперимент: запустите `n` (например, `Environment.ProcessorCount * 4`) задач через `Task.Run(() => Thread.Sleep(500))`, замерьте время через `Stopwatch` и напечатайте его. Параллельно запустите одну «горячую» задачу `Task.Run(() => 42)` и замерьте, через сколько миллисекунд она реально стартует — это покажет голодание пула. Объясните в комментарии, почему CLR не может мгновенно выдать `n` потоков.
6. Реализуйте `RunNonBlockingDemo(int n)` — «хороший» эксперимент: тот же сценарий, но через `Task.Delay(500)` вместо `Thread.Sleep(500)`. Сравните тайминги с предыдущим пунктом. Разница должна быть значительной — это и есть наглядная демонстрация, что `await` освобождает поток (worker возвращается в пул, I/O-завершение пробуждает IOCP-поток).
7. Реализуйте `RunCancellationDemo(CancellationToken token)` — длинный CPU-bound цикл (например, сумма от 0 до 100_000_000 с проверкой `token.ThrowIfCancellationRequested()` внутри). Запустите его через `Task.Run(..., token)`, через 100 мс отмените через `CancellationTokenSource.Cancel()`, перехватите `OperationCanceledException` и напечатайте «отменено». Подчеркните в комментарии: без проверки токена поток пул никогда бы не вернулся.
8. Реализуйте `RunCounterDemo(int tasks, int incrementsPerTask)` — потокобезопасный счётчик через `Interlocked.Increment(ref counter)`, запускаемый из `tasks` задач по `incrementsPerTask` инкрементов. Замерьте и напечатайте итог: он должен равняться `tasks * incrementsPerTask`. Затем «сломайте» версию: сделайте обычный `counter++` без `Interlocked` и покажите, что итог меньше ожидаемого (race condition). Зафиксируйте оба числа.
9. Реализуйте `RunAsyncCriticalSectionDemo()` — 10 одновременных вызовов асинхронного метода, защищённого `SemaphoreSlim(1, 1)` через `WaitAsync`/`Release` в `try/finally`. Внутри секции — `await Task.Delay(50)`. Подчеркните: `lock` здесь использовать нельзя (компилятор запрещает `lock`+`await`), а `Monitor.Enter` + `await` — путь к дедлоку. Не забудьте `Dispose()` у `SemaphoreSlim` в конце.
10. Реализуйте `RunConcurrentQueueDemo()` — producer/consumer на `ConcurrentQueue<int>`: 4 продюсера кладут по 10_000 элементов, 4 потребителя суммируют через `TryDequeue`, результат агрегируют через `Interlocked.Add(ref total, local)`. Сравните с «наивной» версией на `Queue<T>` под `lock` по吞吐ности — ConcurrentQueue должен быть заметно быстрее при параллелизме.
11. Соберите всё в `Main` через top-level `await`, добавьте заголовки-разделители между экспериментами (`Console.WriteLine(new string('=', 60))`), запустите `dotnet run` и сохраните вывод в `output.txt` через `dotnet run > output.txt`. Убедитесь, что в выводе присутствуют все 8 экспериментов и их тайминги.
12. Напишите короткий `README.md` в папке проекта (2–3 абзаца), где вы своими словами объясняете, какой эксперимент какую концепцию урока демонстрирует, и какой вывод вы сделали по каждому.

#### Требования к решению
- Целевая платформа: `.NET 8`, язык C# 12. Обязательны top-level statements, разрешены pattern matching, collection expressions (`[]`), raw string literals где уместно. Проект должен собираться без предупреждений уровня error (`TreatWarningsAsErrors` ставить не нужно, но постарайтесь минимизировать `CA2007` — либо вызывайте `ConfigureAwait(false)`, либо подавляйте с комментарием как в уроке).
- Все долгие операции (CPU-bound циклы, ожидания) обязаны принимать `CancellationToken` и проверять его через `ThrowIfCancellationRequested()` в горячей петле. Это не «красивая деталь», а требование: код, в котором `Task.Run` крутит бесконечный цикл без проверки токена, засчитан не будет.
- Запрещено блокировать поток пула через `.Result`/`.Wait()` на горячем пути. Единственное допустимое исключение — `Task.WaitAll`/`Task.WaitAll` в демонстрационных методах, где это явно отмечено как часть эксперимента (как `Task.WaitAll(tasksArr)` в счётчике из урока). Везде, где есть `await`, используйте его до конца.
- Разделяемое состояние: счётчики — через `Interlocked`; критические секции с `await` — через `SemaphoreSlim(1, 1)` с `Release` в `finally`; очереди producer/consumer — через `ConcurrentQueue<T>` или `Channel<T>`. Использование `lock` вокруг `await` запрещено и не скомпилируется — это сознательная ловушка урока.
- Код должен быть читаемым: методы разделены, каждый эксперимент — в своём `static` методе, с двуязычными комментариями (RU + EN) в ключевых местах, как в примере урока. Не оставляйте «магических чисел» без объяснения.
- Вывод программы должен быть самодокументируемым: каждое измерение сопровождается пояснением, что именно измеряется и какой ожидаемый результат. Если эксперимент демонстрирует ловушку — должно быть видно, что результат «плохой» (например, голодание пула) и объяснено, почему.

#### Тонкости и подводные камни
- **Foreground vs background по умолчанию:** `new Thread(...)` создаёт foreground-поток, а потоки пула всегда background. Если вы запустите foreground-поток и забудете его остановить — процесс «зависнет» в диспетчере задач даже после завершения `Main`. Поэтому в `RunForegroundVsBackgroundDemo` либо дождитесь его через `Join`, либо явно помечайте `IsBackground = true`, если не хотите управлять жизненным циклом. Это самая частая причина «призраков» в продакшене.
- **Голодание пула при блокировке:** CLR добавляет потоки в пул с ограничением по скорости (примерно 1 поток/0.5 сек для CPU-bound), поэтому `Task.Run(() => Thread.Sleep(500))` в количестве `ProcessorCount * 4` не отработает за 500 мс — он растянется на секунды, потому что пул не может мгновенно раздуться. Заметьте: сам `Thread.Sleep` — блокировка потока; `Task.Delay` — нет, потому что под капотом используется таймер и поток возвращается в пул.
- **`.Result` и контекст синхронизации:** в консольном приложении `SynchronizationContext.Current == null`, поэтому `GetValueAsync().Result` «прокатит» без дедлока. Но тот же код в UI (WPF/WinForms) или classic ASP.NET дедлочит: продолжение задачи пытается вернуться в захваченный UI-поток, который сам заблокирован на `.Result`. Поэтому правило: `await` до конца, а в библиотеках — `ConfigureAwait(false)`.
- **`lock` + `await`:** компилятор C# прямо запрещает `lock (gate) { await ... }`. Это не придирка: `lock` (Monitor) не освобождается во время ожидания, и если несколько потоков ждут ту же секцию, весь пул может встать. Правильный асинхронный «lock» — `SemaphoreSlim(1, 1)` с `WaitAsync` и обязательным `Release` в `finally`. Не пытайтесь обходить запрет через `Monitor.Enter` + `await` — это путь к тонким дедлокам.
- **`CancellationToken` — не «для галочки»:** токен нужно не только передать в `Task.Run(..., token)`, но и проверять внутри горячего цикла через `ThrowIfCancellationRequested()`. Без проверки поток продолжит работу, и `Cancel()` ничего не изменит — классическая «залипшая» задача. Также помните, что `ThrowIfCancellationRequested` бросает `OperationCanceledException`, который агрегируется в `AggregateException`, если ожидать через `.Wait()` — перехватывайте аккуратно.
- **`Environment.ProcessorCount` — логические ядра:** это число логических, а не физических ядер (на CPU с Hyper-Threading оно вдвое больше физических). Используйте его как верхнюю подсказку для CPU-bound параллелизма, но не как жёсткий лимит для I/O-работы — I/O не занимает поток, и ограничение здесь бессмысленно.
- **`ConcurrentQueue` vs `Queue`+`lock`:** при низкой конкуренции разница незаметна, но при 4+ продюсерах/потребителях `ConcurrentQueue` (lock-free под капотом) ощутимо быстрее и не рискует дедлоком, если потребитель решит `await` внутри обработки. Для сложных пайплайнов современная рекомендация — `System.Threading.Channels.Channel<T>`.

#### Критерии приёмки
- [ ] Проект `ConcurrencyLab` собирается под `.NET 8` через `dotnet build` без ошибок и запускается через `dotnet run`.
- [ ] `Program.cs` использует top-level statements и C# 12; `Environment.ProcessorCount` напечатан в первых строках вывода.
- [ ] `RunForegroundVsBackgroundDemo` запускает foreground и background `Thread`; в выводе видно, что процесс ждёт foreground-поток, и в коде есть комментарий-объяснение.
- [ ] `RunThreadPoolQueueDemo` использует `ThreadPool.QueueUserWorkItem` и печатает `IsBackground`/`ManagedThreadId` пул-потоков; комментарий объясняет отсутствие результата и отмены.
- [ ] `RunBlockingStarvationDemo` измеряет время `n` блокирующих `Task.Run(Thread.Sleep)` и тайминг «горячей» задачи; в выводе зафиксировано голодание (время заметно больше 500 мс).
- [ ] `RunNonBlockingDemo` показывает, что `Task.Delay` отрабатывает за ~500 мс; разница с предыдущим пунктом явно сопоставлена в выводе.
- [ ] `RunCancellationDemo` принимает `CancellationToken`, проверяет его в цикле через `ThrowIfCancellationRequested()`, корректно перехватывает `OperationCanceledException` и печатает «отменено».
- [ ] `RunCounterDemo` через `Interlocked.Increment` даёт точный результат `tasks * incrementsPerTask`; «сломанная» версия без `Interlocked` даёт меньшее число, и оба числа напечатаны.
- [ ] `RunAsyncCriticalSectionDemo` использует `SemaphoreSlim(1, 1)` с `WaitAsync`/`Release` в `try/finally`; `lock`+`await` в коде нет; `SemaphoreSlim` диспозится.
- [ ] `RunConcurrentQueueDemo` реализует producer/consumer на `ConcurrentQueue<int>`; сумма корректна; есть сравнение吞吐ности с `Queue<T>`+`lock`.
- [ ] Ни в одном горячем пути не используется `.Result`/`.Wait()` (допустим только демонстрационный `Task.WaitAll`).
- [ ] Вывод `dotnet run` сохранён в `output.txt` и содержит все 8 экспериментов с заголовками-разделителями.
- [ ] В `README.md` своими словами объяснено, какой эксперимент какую концепцию урока закрепляет.
- [ ] Код содержит двуязычные комментарии в ключевых местах (RU + EN), как в примере урока.

#### Подсказки (без прямого ответа)
- Для замера «горячей» задачи в `RunBlockingStarvationDemo` заведите `Stopwatch`, стартуйте его перед запуском блокирующих задач, и параллельно запустите `Task.Run(() => 42).ContinueWith(t => sw.ElapsedMilliseconds)` — посмотрите, когда продолжение сработает. Разница со «спокойным» сценарием и есть вид голодания.
- В `RunCancellationDemo` используйте `using var cts = new CancellationTokenSource();` и `cts.CancelAfter(100)` — это короче, чем ручной `Task.Delay(100).ContinueWith(_ => cts.Cancel())`, и не требует отдельного таймера.
- «Сломать» счётчик в `RunCounterDemo` можно просто заменив `Interlocked.Increment(ref counter)` на `counter++` — но НЕ коммитьте «сломанную» версию как финальную; верните `Interlocked` обратно и оставьте обе как отдельные методы для сравнения.
- Для producer/consumer не забудьте `Task.WaitAll(producers)` ДО `Task.WaitAll(consumers)` — иначе потребитель может finished раньше, чем продюсер всё положит (как в уроке: «даём потребителям добрать остаток»).
- Если `SemaphoreSlim` ругается на double-`Release` — вы, скорее всего, вызываете `Release` вне `finally` при исключении. Всегда: `await gate.WaitAsync(); try { ... } finally { gate.Release(); }`.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — эталонное решение ДЗ M11-L01.
// Top-level statements; демонстрирует Thread / ThreadPool / Task и ловушки урока.
using System.Collections.Concurrent;
using System.Diagnostics;

#pragma warning disable CA2007 // демо без ConfigureAwait в логах / demo without ConfigureAwait in logs

int cores = Environment.ProcessorCount;
Console.WriteLine($"Логических ядер / Logical cores: {cores}");
Console.WriteLine(new string('=', 60));

// ── 1) Foreground vs Background Thread ─────────────────────────────────────
// 1) Foreground vs Background Thread
RunForegroundVsBackgroundDemo();
Console.WriteLine(new string('=', 60));

// ── 2) ThreadPool.QueueUserWorkItem ────────────────────────────────────────
// 2) ThreadPool.QueueUserWorkItem
RunThreadPoolQueueDemo();
Thread.Sleep(50); // даём пул-потокам напечатать / let pool threads print
Console.WriteLine(new string('=', 60));

// ── 3) Голодание пула: блокирующий Task.Run + Thread.Sleep ─────────────────
// 3) Pool starvation: blocking Task.Run + Thread.Sleep
RunBlockingStarvationDemo(cores * 4);
Console.WriteLine(new string('=', 60));

// ── 4) Неблокирующий путь: Task.Delay ──────────────────────────────────────
// 4) Non-blocking path: Task.Delay
await RunNonBlockingDemo(cores * 4);
Console.WriteLine(new string('=', 60));

// ── 5) Отмена длинной CPU-операции через CancellationToken ─────────────────
// 5) Cancel a long CPU-bound op via CancellationToken
using var cts = new CancellationTokenSource();
cts.CancelAfter(100);
try
{
    int r = await RunCancellationDemo(cts.Token);
    Console.WriteLine($"Task результат / Task result: {r}");
}
catch (OperationCanceledException)
{
    Console.WriteLine("Task отменён / Task cancelled");
}
Console.WriteLine(new string('=', 60));

// ── 6) Потокобезопасный счётчик через Interlocked ──────────────────────────
// 6) Thread-safe counter via Interlocked
Console.WriteLine($"Interlocked счётчик / Interlocked counter: {RunCounterDemo(cores, 100_000)} (ожидали / expected {cores * 100_000})");
Console.WriteLine($"Сломанный счётчик / Broken counter: {RunCounterDemoBroken(cores, 100_000)} (ожидали / expected {cores * 100_000})");
Console.WriteLine(new string('=', 60));

// ── 7) Асинхронная критическая секция: SemaphoreSlim(1,1) ──────────────────
// 7) Async critical section: SemaphoreSlim(1,1)
await RunAsyncCriticalSectionDemo();
Console.WriteLine(new string('=', 60));

// ── 8) ConcurrentQueue producer/consumer ───────────────────────────────────
// 8) ConcurrentQueue producer/consumer
Console.WriteLine($"ConcurrentQueue сумма / ConcurrentQueue sum: {RunConcurrentQueueDemo()}");
Console.WriteLine("Готово / Done.");

// ───────────────────────────────────────────────────────────────────────────
static void RunForegroundVsBackgroundDemo()
{
    // foreground по умолчанию — процесс будет ждать его завершения.
    // foreground by default — the process will wait for it to finish.
    var foreground = new Thread(() =>
    {
        Thread.Sleep(300);
        Console.WriteLine("Foreground-поток завершился / Foreground done");
    }) { Name = "demo-foreground" }; // IsBackground = false по умолчанию / false by default

    var background = new Thread(() =>
    {
        Thread.Sleep(300);
        Console.WriteLine("Background-поток завершился / Background done");
    }) { IsBackground = true, Name = "demo-background" };

    background.Start();
    foreground.Start();
    foreground.Join(); // иначе main может завершиться раньше печати / join so main doesn't race the print
}

static void RunThreadPoolQueueDemo()
{
    // QueueUserWorkItem дёшев, но НЕ возвращает результат и НЕ даёт отмену.
    // QueueUserWorkItem is cheap, but returns no result and offers no cancellation.
    ThreadPool.QueueUserWorkItem(_ =>
        Console.WriteLine($"Pool #1: IsBackground={Thread.CurrentThread.IsBackground}, Id={Thread.CurrentThread.ManagedThreadId}"));
    ThreadPool.QueueUserWorkItem(_ =>
        Console.WriteLine($"Pool #2: IsBackground={Thread.CurrentThread.IsBackground}, Id={Thread.CurrentThread.ManagedThreadId}"));
}

static void RunBlockingStarvationDemo(int n)
{
    var sw = Stopwatch.StartNew();
    // ПЛОХО: Thread.Sleep блокирует поток пула. CLR не может мгновенно выдать n потоков
    // (ограничение ~1 поток/0.5 сек), поэтому общий тайминг сильно больше 500 мс.
    // BAD: Thread.Sleep blocks a pool thread. CLR cannot hand out n threads at once
    // (throttle ~1 thread/0.5 sec), so total time is far greater than 500 ms.
    var blocking = Enumerable.Range(0, n).Select(_ => Task.Run(() => Thread.Sleep(500))).ToArray();

    var hotSw = Stopwatch.StartNew();
    var hot = Task.Run(() => 42);
    Task.WaitAll(blocking);
    sw.Stop();
    Console.WriteLine($"Блокирующих {n} Task.Run заняли / Blocking {n} Task.Run took: {sw.ElapsedMilliseconds} мс / ms");
    Console.WriteLine($"Горячая задача стартовала за / Hot task started in: {hotSw.ElapsedMilliseconds} мс / ms (результат / result {hot.Result})");
}

static async Task RunNonBlockingDemo(int n)
{
    var sw = Stopwatch.StartNew();
    // ХОРОШО: Task.Delay не занимает поток — таймер + возвращение потока в пул.
    // GOOD: Task.Delay does not hold a thread — timer + thread returns to the pool.
    var tasks = Enumerable.Range(0, n).Select(_ => Task.Delay(500)).ToArray();
    await Task.WhenAll(tasks);
    sw.Stop();
    Console.WriteLine($"Неблокирующих {n} Task.Delay заняли / Non-blocking {n} Task.Delay took: {sw.ElapsedMilliseconds} мс / ms");
}

static async Task<int> RunCancellationDemo(CancellationToken token)
{
    // кооперативная отмена: ThrowIfCancellationRequested в горячей петле.
    // cooperative cancellation: ThrowIfCancellationRequested in the hot loop.
    return await Task.Run(() =>
    {
        long sum = 0;
        for (int i = 0; i < 100_000_000; i++)
        {
            token.ThrowIfCancellationRequested();
            sum += i;
        }
        return (int)(sum % int.MaxValue);
    }, token);
}

static long RunCounterDemo(int tasks, int perTask)
{
    long counter = 0;
    var arr = new Task[tasks];
    for (int t = 0; t < tasks; t++)
    {
        arr[t] = Task.Run(() =>
        {
            for (int i = 0; i < perTask; i++)
                Interlocked.Increment(ref counter); // атомарно / atomic
        });
    }
    Task.WaitAll(arr);
    return counter;
}

static long RunCounterDemoBroken(int tasks, int perTask)
{
    long counter = 0;
    var arr = new Task[tasks];
    for (int t = 0; t < tasks; t++)
    {
        arr[t] = Task.Run(() =>
        {
            for (int i = 0; i < perTask; i++)
                counter++; // race condition: неатомарно / not atomic
        });
    }
    Task.WaitAll(arr);
    return counter;
}

static async Task RunAsyncCriticalSectionDemo()
{
    // SemaphoreSlim(1,1) — асинхронный «lock». lock + await запрещён компилятором.
    // SemaphoreSlim(1,1) — async "lock". lock + await is forbidden by the compiler.
    using var gate = new SemaphoreSlim(1, 1);
    var tasks = Enumerable.Range(0, 10).Select(_ => CriticalSectionAsync(gate)).ToArray();
    await Task.WhenAll(tasks);
}

static async Task CriticalSectionAsync(SemaphoreSlim gate)
{
    await gate.WaitAsync();
    try
    {
        await Task.Delay(50); // защищённая секция с await внутри / protected section with await inside
    }
    finally
    {
        gate.Release(); // ВСЕГДА в finally / ALWAYS in finally
    }
}

static long RunConcurrentQueueDemo()
{
    var queue = new ConcurrentQueue<int>();
    var producers = Enumerable.Range(0, 4).Select(_ => Task.Run(() =>
    {
        for (int i = 0; i < 10_000; i++) queue.Enqueue(i);
    })).ToArray();

    long total = 0;
    var consumers = Enumerable.Range(0, 4).Select(_ => Task.Run(() =>
    {
        long local = 0;
        while (queue.TryDequeue(out int v)) local += v;
        Interlocked.Add(ref total, local); // локальную сумму — атомарно в общую / local sum atomic into total
    })).ToArray();

    Task.WaitAll(producers);
    Task.WaitAll(consumers); // даём потребителям добрать остаток / let consumers drain the rest
    return total;
}
```

**Разбор по строкам.** Первая содержательная строка — `int cores = Environment.ProcessorCount;` — не случайна: как подчёркнуто в уроке, это естественная подсказка уровня параллелизма, и мы переиспользуем её в `cores * 4` для давления на пул. В `RunForegroundVsBackgroundDemo` сознательно оставлен `IsBackground = false` для foreground-потока, чтобы показать ловушку урока: процесс ждёт именно foreground, тогда как background «сгорает» при остановке main; `foreground.Join()` здесь — лишь чтобы печать не «опоздала» в демо, а не требование продакшена. В `RunThreadPoolQueueDemo` мы намеренно не возвращаем результат и не передаём токен — это и есть иллюстрация, почему `QueueUserWorkItem` уступает `Task.Run`: нет композиции, нет `Task<T>`, нет `CancellationToken`.

`RunBlockingStarvationDemo` — сердце задания. `Task.Run(() => Thread.Sleep(500))` блокирует пул-поток, а CLR добавляет потоки с ограничением ~1/0.5 сек, поэтому `n` задач отработают заметно дольше 500 мс; параллельная «горячая» задача `Task.Run(() => 42)` покажет, что даже тривиальная работа ждёт своей очереди. Контраст с `RunNonBlockingDemo` на `Task.Delay(500)` — наглядное доказательство, что `await` освобождает поток (worker возвращается в пул, завершение таймера пробуждает IOCP-поток). В `RunCancellationDemo` токен проверяется `ThrowIfCancellationRequested()` внутри горячего цикла — без этого `cts.CancelAfter(100)` ничего бы не дал, и поток «залип» бы до конца суммы, что прямо перекликается с лучшей практикой урока.

`RunCounterDemo`/`RunCounterDemoBroken` рядом показывают роль `Interlocked.Increment`: атомарная версия даёт точное `tasks * perTask`, «сломанная» — меньшее число из-за race condition (чтение-модификация-запись не атомарна). `RunAsyncCriticalSectionDemo` использует `SemaphoreSlim(1, 1)` с `WaitAsync` и `Release` в `finally` — это единственный корректный «async lock», потому что `lock`+`await` компилятор запрещает, а обход через `Monitor.Enter`+`await` ведёт к дедлоку. Наконец, `RunConcurrentQueueDemo` реализует producer/consumer на lock-free `ConcurrentQueue<int>` с агрегацией локальных сумм через `Interlocked.Add` — это и есть рекомендованная в уроке альтернатива `Queue<T>` под `lock`, особенно когда потребитель может `await` внутри обработки.

#### Задания на углубление (бонус)
1. Замените `ConcurrentQueue<int>` на `System.Threading.Channels.Channel<int>` (bounded,容量ность 1000) и сравните吞吐ность и пиковое потребление памяти при 8 продюсерах/8 потребителях. Опишите, когда `Channel` предпочтительнее `ConcurrentQueue`.
2. Добавьте эксперимент, демонстрирующий дедлок `.Result` в контексте синхронизации: создайте собственный `SynchronizationContext` (или используйте `DispatcherSynchronizationContext` в WPF-тесте), запустите `async` метод, который внутри вызывает другой `async` метод и блокирует его через `.Result`. Покажите дедлок и почините его через `ConfigureAwait(false)` во внутренней библиотечной функции.
3. Реализуйте долгоживущий «фоновый демон» на `Thread` с `IsBackground = true` и явной остановкой через `CancellationToken` + `Thread.Join(timeout)`. Объясните, в каких случаях такой `Thread` оправданнее `Task.Run` (длинный CPU-bound цикл, особые требования к стеку/приоритету).
4. Измерьте рост пула под давлением: запустите 100 блокирующих `Task.Run(Thread.Sleep)` и параллельно каждые 100 мс печатайте `ThreadPool.GetAvailableThreads` и `ThreadPool.GetMaxThreads`. Зафиксируйте, как пул дозревает до максимума, и сопоставьте с эвристикой «1 поток/0.5 сек» из урока.

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you have joined a team building a "background jobs" service for a small platform. The previous engineers left three versions of the same queue processor — one on raw `Thread`, one on `ThreadPool.QueueUserWorkItem`, and one on `Task.Run`. All three "work" in the lab, but in production strange things happen regularly: the service process lingers in Task Manager after shutdown; under a load burst processing suddenly stalls for tens of seconds; sometimes the service stops responding at all until it is restarted. The tech lead asks you to figure out why this happens and to write a reference implementation that behaves predictably.

This lab is not a toy example for its own sake. Every trap you hit maps directly onto the lesson's best practices: the cost of creating a `Thread` (~1 MB of stack plus kernel context switches), the CLR pool-growth heuristics (~1 thread per 0.5 sec for CPU-bound work), the split between worker threads and IOCP/IO threads, cooperative cancellation through `CancellationToken`, and the fundamental rule that you must never block a pool thread with `.Result`/`.Wait()` under a synchronization context. The goal is not "make it compile" but to consciously pick the right primitive for every sub-task and be ready to justify the choice.

By the end you will have a `.NET 8` console project that runs a series of mini-experiments, prints timings and observations, and shows "bad" and "good" approaches side by side — so the difference is visible to the naked eye. That contrast (pool starvation vs. non-blocking implementation, broken counter vs. atomic counter) is the strongest possible proof that you understood the lesson: not a quote from docs, but a live, measurable difference between blocking and non-blocking code.

#### What to do step by step
1. Create a new console project: `dotnet new console -n ConcurrencyLab -o ConcurrencyLab`, then `cd ConcurrencyLab`. Make sure `ConcurrencyLab.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and `<LangVersion>latest</LangVersion>`. Add `<Nullable>enable</Nullable>` if you like. Run the empty project with `dotnet run` — you should see an empty output with no errors.
2. In `Program.cs` use top-level statements (C# 12). The very first meaningful line prints `Environment.ProcessorCount` — the natural parallelism hint mentioned in the lesson. Remember this number: it is used to size several experiments.
3. Implement `RunForegroundVsBackgroundDemo()` that starts two `Thread`s: one with `IsBackground = false` (default) and one with `IsBackground = true`. Both sleep `Thread.Sleep(600)` and print a message when done. Run it and confirm: the main thread returns, but the process waits for the foreground thread. Comment in code why this happens and why pool threads (`IsBackground = true` by default) do not create this trap.
4. Implement `RunThreadPoolQueueDemo()` using `ThreadPool.QueueUserWorkItem`. In the callback print `Thread.CurrentThread.IsBackground` and `Thread.CurrentThread.ManagedThreadId` for two different enqueue calls. Reflect and note in a comment: why does `QueueUserWorkItem` return no result and offer no cancellation, and what does that mean for API design?
5. Implement `RunBlockingStarvationDemo(int n)` — the "bad" experiment: launch `n` (e.g. `Environment.ProcessorCount * 4`) tasks via `Task.Run(() => Thread.Sleep(500))`, measure time with `Stopwatch`, and print it. In parallel, fire one "hot" task `Task.Run(() => 42)` and measure how many milliseconds actually pass before it starts — this reveals pool starvation. Explain in a comment why the CLR cannot hand out `n` threads instantly.
6. Implement `RunNonBlockingDemo(int n)` — the "good" experiment: the same scenario but with `Task.Delay(500)` instead of `Thread.Sleep(500)`. Compare timings with the previous step. The gap should be significant — a live demonstration that `await` frees the thread (the worker returns to the pool, I/O completion wakes an IOCP thread).
7. Implement `RunCancellationDemo(CancellationToken token)` — a long CPU-bound loop (e.g. a sum from 0 to 100_000_000 checking `token.ThrowIfCancellationRequested()` inside). Run it via `Task.Run(..., token)`, cancel after 100 ms with `CancellationTokenSource.Cancel()`, catch `OperationCanceledException`, and print "cancelled". Stress in a comment: without the token check the pool thread would never return.
8. Implement `RunCounterDemo(int tasks, int incrementsPerTask)` — a thread-safe counter via `Interlocked.Increment(ref counter)` launched from `tasks` tasks each doing `incrementsPerTask` increments. Measure and print the total: it must equal `tasks * incrementsPerTask`. Then "break" a version: do plain `counter++` without `Interlocked` and show the total is smaller than expected (race condition). Record both numbers.
9. Implement `RunAsyncCriticalSectionDemo()` — 10 concurrent calls to an async method guarded by `SemaphoreSlim(1, 1)` via `WaitAsync`/`Release` in `try/finally`. Inside the section, `await Task.Delay(50)`. Stress: `lock` cannot be used here (the compiler forbids `lock` + `await`), and `Monitor.Enter` + `await` is a path to deadlock. Do not forget `Dispose()` on the `SemaphoreSlim` at the end.
10. Implement `RunConcurrentQueueDemo()` — a producer/consumer on `ConcurrentQueue<int>`: 4 producers enqueue 10_000 items each, 4 consumers sum via `TryDequeue`, and the result is aggregated via `Interlocked.Add(ref total, local)`. Compare throughput against a naive `Queue<T>` under `lock` — `ConcurrentQueue` should be noticeably faster under parallelism.
11. Assemble everything in `Main` via top-level `await`, add separator headers between experiments (`Console.WriteLine(new string('=', 60))`), run `dotnet run`, and save the output to `output.txt` via `dotnet run > output.txt`. Make sure all 8 experiments and their timings appear in the output.
12. Write a short `README.md` in the project folder (2–3 paragraphs) where, in your own words, you explain which experiment demonstrates which lesson concept and what conclusion you drew for each.

#### Requirements
- Target: `.NET 8`, language C# 12. Top-level statements are mandatory; pattern matching, collection expressions (`[]`), and raw string literals are allowed where helpful. The project should build without error-level warnings (`TreatWarningsAsErrors` is not required, but try to minimize `CA2007` — either call `ConfigureAwait(false)` or suppress it with a comment, exactly as the lesson does).
- Every long operation (CPU-bound loops, awaits) must accept a `CancellationToken` and check it with `ThrowIfCancellationRequested()` inside the hot loop. This is not a "nice touch" but a hard requirement: code where `Task.Run` spins an infinite loop without a token check will not be accepted.
- Blocking a pool thread via `.Result`/`.Wait()` on a hot path is forbidden. The only allowed exception is `Task.WaitAll` in demo methods where it is explicitly part of the experiment (like `Task.WaitAll(tasksArr)` in the lesson's counter). Everywhere `await` is possible, use it to the end.
- Shared state: counters go through `Interlocked`; critical sections with `await` go through `SemaphoreSlim(1, 1)` with `Release` in `finally`; producer/consumer queues go through `ConcurrentQueue<T>` or `Channel<T>`. Using `lock` around `await` is forbidden and will not compile — this is a deliberate trap from the lesson.
- The code must be readable: methods are split, each experiment lives in its own `static` method, with bilingual comments (RU + EN) in key spots, exactly like the lesson example. Do not leave "magic numbers" unexplained.
- The program output must be self-documenting: every measurement comes with a note about what is being measured and what the expected result is. If an experiment demonstrates a trap, it must be visible that the result is "bad" (e.g. pool starvation) and explained why.

#### Pitfalls
- **Foreground vs background by default:** `new Thread(...)` creates a foreground thread, while pool threads are always background. If you start a foreground thread and forget to stop it, the process lingers in Task Manager even after `Main` returns. So in `RunForegroundVsBackgroundDemo` either `Join` it or explicitly mark `IsBackground = true` if you do not intend to manage the lifecycle. This is the most common cause of "ghost" processes in production.
- **Pool starvation under blocking:** the CLR adds threads to the pool at a throttled rate (~1 thread/0.5 sec for CPU-bound work), so `Task.Run(() => Thread.Sleep(500))` in quantity `ProcessorCount * 4` will not finish in 500 ms — it will stretch over seconds because the pool cannot balloon instantly. Note: `Thread.Sleep` blocks the thread; `Task.Delay` does not, because under the hood it uses a timer and the thread returns to the pool.
- **`.Result` and the synchronization context:** in a console app `SynchronizationContext.Current == null`, so `GetValueAsync().Result` "works" without a deadlock. But the same code in UI (WPF/WinForms) or classic ASP.NET deadlocks: the task continuation tries to return to the captured UI thread, which is itself blocked on `.Result`. Rule: `await` to the end, and in libraries use `ConfigureAwait(false)`.
- **`lock` + `await`:** the C# compiler directly forbids `lock (gate) { await ... }`. This is not pedantry: `lock` (Monitor) is not released during the await, and if several threads wait on the same section, the whole pool can stall. The correct async "lock" is `SemaphoreSlim(1, 1)` with `WaitAsync` and a mandatory `Release` in `finally`. Do not try to work around the ban with `Monitor.Enter` + `await` — that is a path to subtle deadlocks.
- **`CancellationToken` is not "for show":** you must not only pass the token into `Task.Run(..., token)` but also check it inside the hot loop with `ThrowIfCancellationRequested()`. Without the check the thread keeps running and `Cancel()` changes nothing — a classic "stuck" task. Also remember that `ThrowIfCancellationRequested` throws `OperationCanceledException`, which is aggregated into `AggregateException` if you wait via `.Wait()` — handle it carefully.
- **`Environment.ProcessorCount` is logical cores:** this is the count of logical, not physical, cores (on a CPU with Hyper-Threading it is twice the physical count). Use it as an upper hint for CPU-bound parallelism, not as a hard limit for I/O work — I/O does not hold a thread, so a limit here is meaningless.
- **`ConcurrentQueue` vs `Queue`+`lock`:** under low contention the gap is invisible, but with 4+ producers/consumers `ConcurrentQueue` (lock-free under the hood) is noticeably faster and does not risk a deadlock if the consumer decides to `await` inside processing. For complex pipelines the modern recommendation is `System.Threading.Channels.Channel<T>`.

#### Acceptance criteria
- [ ] The `ConcurrencyLab` project builds under `.NET 8` via `dotnet build` with no errors and runs via `dotnet run`.
- [ ] `Program.cs` uses top-level statements and C# 12; `Environment.ProcessorCount` is printed in the first output lines.
- [ ] `RunForegroundVsBackgroundDemo` starts a foreground and a background `Thread`; the output shows the process waits for the foreground thread, and the code has an explanatory comment.
- [ ] `RunThreadPoolQueueDemo` uses `ThreadPool.QueueUserWorkItem` and prints the pool threads' `IsBackground`/`ManagedThreadId`; a comment explains the lack of result and cancellation.
- [ ] `RunBlockingStarvationDemo` measures the time of `n` blocking `Task.Run(Thread.Sleep)` and the timing of a "hot" task; the output records starvation (time noticeably greater than 500 ms).
- [ ] `RunNonBlockingDemo` shows `Task.Delay` finishes in ~500 ms; the difference with the previous step is explicitly compared in the output.
- [ ] `RunCancellationDemo` accepts a `CancellationToken`, checks it in the loop via `ThrowIfCancellationRequested()`, correctly catches `OperationCanceledException`, and prints "cancelled".
- [ ] `RunCounterDemo` via `Interlocked.Increment` yields the exact result `tasks * incrementsPerTask`; the "broken" version without `Interlocked` yields a smaller number, and both numbers are printed.
- [ ] `RunAsyncCriticalSectionDemo` uses `SemaphoreSlim(1, 1)` with `WaitAsync`/`Release` in `try/finally`; there is no `lock`+`await` in the code; the `SemaphoreSlim` is disposed.
- [ ] `RunConcurrentQueueDemo` implements a producer/consumer on `ConcurrentQueue<int>`; the sum is correct; there is a throughput comparison with `Queue<T>`+`lock`.
- [ ] No hot path uses `.Result`/`.Wait()` (only the demo `Task.WaitAll` is allowed).
- [ ] The `dotnet run` output is saved to `output.txt` and contains all 8 experiments with separator headers.
- [ ] `README.md` explains, in your own words, which experiment reinforces which lesson concept.
- [ ] The code contains bilingual comments in key spots (RU + EN), as in the lesson example.

#### Hints (no direct answer)
- To measure the "hot" task in `RunBlockingStarvationDemo`, start a `Stopwatch` before launching the blocking tasks, and in parallel fire `Task.Run(() => 42).ContinueWith(t => sw.ElapsedMilliseconds)` — see when the continuation fires. The gap from the "calm" scenario is the visible starvation.
- In `RunCancellationDemo` use `using var cts = new CancellationTokenSource();` and `cts.CancelAfter(100)` — shorter than a manual `Task.Delay(100).ContinueWith(_ => cts.Cancel())` and needs no separate timer.
- To "break" the counter in `RunCounterDemo`, just replace `Interlocked.Increment(ref counter)` with `counter++` — but do NOT commit the broken version as final; restore `Interlocked` and keep both as separate comparison methods.
- For producer/consumer, do not forget `Task.WaitAll(producers)` BEFORE `Task.WaitAll(consumers)` — otherwise the consumer may finish before the producer has enqueued everything (as in the lesson: "let consumers drain the rest").
- If `SemaphoreSlim` complains about double `Release`, you are probably calling `Release` outside `finally` on an exception. Always: `await gate.WaitAsync(); try { ... } finally { gate.Release(); }`.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — reference solution for homework M11-L01.
// Top-level statements; demonstrates Thread / ThreadPool / Task and lesson traps.
using System.Collections.Concurrent;
using System.Diagnostics;

#pragma warning disable CA2007 // demo without ConfigureAwait in logs

int cores = Environment.ProcessorCount;
Console.WriteLine($"Logical cores: {cores}");
Console.WriteLine(new string('=', 60));

// 1) Foreground vs Background Thread
RunForegroundVsBackgroundDemo();
Console.WriteLine(new string('=', 60));

// 2) ThreadPool.QueueUserWorkItem
RunThreadPoolQueueDemo();
Thread.Sleep(50); // let pool threads print
Console.WriteLine(new string('=', 60));

// 3) Pool starvation: blocking Task.Run + Thread.Sleep
RunBlockingStarvationDemo(cores * 4);
Console.WriteLine(new string('=', 60));

// 4) Non-blocking path: Task.Delay
await RunNonBlockingDemo(cores * 4);
Console.WriteLine(new string('=', 60));

// 5) Cancel a long CPU-bound op via CancellationToken
using var cts = new CancellationTokenSource();
cts.CancelAfter(100);
try
{
    int r = await RunCancellationDemo(cts.Token);
    Console.WriteLine($"Task result: {r}");
}
catch (OperationCanceledException)
{
    Console.WriteLine("Task cancelled");
}
Console.WriteLine(new string('=', 60));

// 6) Thread-safe counter via Interlocked
Console.WriteLine($"Interlocked counter: {RunCounterDemo(cores, 100_000)} (expected {cores * 100_000})");
Console.WriteLine($"Broken counter: {RunCounterDemoBroken(cores, 100_000)} (expected {cores * 100_000})");
Console.WriteLine(new string('=', 60));

// 7) Async critical section: SemaphoreSlim(1,1)
await RunAsyncCriticalSectionDemo();
Console.WriteLine(new string('=', 60));

// 8) ConcurrentQueue producer/consumer
Console.WriteLine($"ConcurrentQueue sum: {RunConcurrentQueueDemo()}");
Console.WriteLine("Done.");

static void RunForegroundVsBackgroundDemo()
{
    // foreground by default — the process will wait for it to finish.
    var foreground = new Thread(() =>
    {
        Thread.Sleep(300);
        Console.WriteLine("Foreground done");
    }) { Name = "demo-foreground" }; // IsBackground = false by default

    var background = new Thread(() =>
    {
        Thread.Sleep(300);
        Console.WriteLine("Background done");
    }) { IsBackground = true, Name = "demo-background" };

    background.Start();
    foreground.Start();
    foreground.Join(); // join so main doesn't race the print
}

static void RunThreadPoolQueueDemo()
{
    // QueueUserWorkItem is cheap, but returns no result and offers no cancellation.
    ThreadPool.QueueUserWorkItem(_ =>
        Console.WriteLine($"Pool #1: IsBackground={Thread.CurrentThread.IsBackground}, Id={Thread.CurrentThread.ManagedThreadId}"));
    ThreadPool.QueueUserWorkItem(_ =>
        Console.WriteLine($"Pool #2: IsBackground={Thread.CurrentThread.IsBackground}, Id={Thread.CurrentThread.ManagedThreadId}"));
}

static void RunBlockingStarvationDemo(int n)
{
    var sw = Stopwatch.StartNew();
    // BAD: Thread.Sleep blocks a pool thread. CLR cannot hand out n threads at once
    // (throttle ~1 thread/0.5 sec), so total time is far greater than 500 ms.
    var blocking = Enumerable.Range(0, n).Select(_ => Task.Run(() => Thread.Sleep(500))).ToArray();

    var hotSw = Stopwatch.StartNew();
    var hot = Task.Run(() => 42);
    Task.WaitAll(blocking);
    sw.Stop();
    Console.WriteLine($"Blocking {n} Task.Run took: {sw.ElapsedMilliseconds} ms");
    Console.WriteLine($"Hot task started in: {hotSw.ElapsedMilliseconds} ms (result {hot.Result})");
}

static async Task RunNonBlockingDemo(int n)
{
    var sw = Stopwatch.StartNew();
    // GOOD: Task.Delay does not hold a thread — timer + thread returns to the pool.
    var tasks = Enumerable.Range(0, n).Select(_ => Task.Delay(500)).ToArray();
    await Task.WhenAll(tasks);
    sw.Stop();
    Console.WriteLine($"Non-blocking {n} Task.Delay took: {sw.ElapsedMilliseconds} ms");
}

static async Task<int> RunCancellationDemo(CancellationToken token)
{
    // cooperative cancellation: ThrowIfCancellationRequested in the hot loop.
    return await Task.Run(() =>
    {
        long sum = 0;
        for (int i = 0; i < 100_000_000; i++)
        {
            token.ThrowIfCancellationRequested();
            sum += i;
        }
        return (int)(sum % int.MaxValue);
    }, token);
}

static long RunCounterDemo(int tasks, int perTask)
{
    long counter = 0;
    var arr = new Task[tasks];
    for (int t = 0; t < tasks; t++)
    {
        arr[t] = Task.Run(() =>
        {
            for (int i = 0; i < perTask; i++)
                Interlocked.Increment(ref counter); // atomic
        });
    }
    Task.WaitAll(arr);
    return counter;
}

static long RunCounterDemoBroken(int tasks, int perTask)
{
    long counter = 0;
    var arr = new Task[tasks];
    for (int t = 0; t < tasks; t++)
    {
        arr[t] = Task.Run(() =>
        {
            for (int i = 0; i < perTask; i++)
                counter++; // race condition: not atomic
        });
    }
    Task.WaitAll(arr);
    return counter;
}

static async Task RunAsyncCriticalSectionDemo()
{
    // SemaphoreSlim(1,1) — async "lock". lock + await is forbidden by the compiler.
    using var gate = new SemaphoreSlim(1, 1);
    var tasks = Enumerable.Range(0, 10).Select(_ => CriticalSectionAsync(gate)).ToArray();
    await Task.WhenAll(tasks);
}

static async Task CriticalSectionAsync(SemaphoreSlim gate)
{
    await gate.WaitAsync();
    try
    {
        await Task.Delay(50); // protected section with await inside
    }
    finally
    {
        gate.Release(); // ALWAYS in finally
    }
}

static long RunConcurrentQueueDemo()
{
    var queue = new ConcurrentQueue<int>();
    var producers = Enumerable.Range(0, 4).Select(_ => Task.Run(() =>
    {
        for (int i = 0; i < 10_000; i++) queue.Enqueue(i);
    })).ToArray();

    long total = 0;
    var consumers = Enumerable.Range(0, 4).Select(_ => Task.Run(() =>
    {
        long local = 0;
        while (queue.TryDequeue(out int v)) local += v;
        Interlocked.Add(ref total, local); // local sum atomic into total
    })).ToArray();

    Task.WaitAll(producers);
    Task.WaitAll(consumers); // let consumers drain the rest
    return total;
}
```

**Line-by-line walk-through.** The first substantive line, `int cores = Environment.ProcessorCount;`, is not accidental: as the lesson stresses, it is the natural parallelism hint, and we reuse it as `cores * 4` to put pressure on the pool. In `RunForegroundVsBackgroundDemo` the foreground thread deliberately keeps the default `IsBackground = false` to expose the lesson's trap: the process waits specifically for the foreground thread, while the background one is killed when main stops; `foreground.Join()` here is only so the print does not race in the demo, not a production requirement. In `RunThreadPoolQueueDemo` we deliberately do not return a result and do not pass a token — that is exactly the illustration of why `QueueUserWorkItem` loses to `Task.Run`: no composition, no `Task<T>`, no `CancellationToken`.

`RunBlockingStarvationDemo` is the heart of the task. `Task.Run(() => Thread.Sleep(500))` blocks a pool thread, and the CLR adds threads at a throttle of ~1/0.5 sec, so `n` tasks take noticeably more than 500 ms; the parallel "hot" task `Task.Run(() => 42)` shows that even trivial work has to wait its turn. The contrast with `RunNonBlockingDemo` on `Task.Delay(500)` is a live proof that `await` frees the thread (the worker returns to the pool, the timer completion wakes an IOCP thread). In `RunCancellationDemo` the token is checked with `ThrowIfCancellationRequested()` inside the hot loop — without it, `cts.CancelAfter(100)` would do nothing and the thread would be stuck until the end of the sum, which directly echoes the lesson's best practice.

`RunCounterDemo`/`RunCounterDemoBroken` placed side by side show the role of `Interlocked.Increment`: the atomic version yields the exact `tasks * perTask`, the "broken" one yields a smaller number because of a race condition (read-modify-write is not atomic). `RunAsyncCriticalSectionDemo` uses `SemaphoreSlim(1, 1)` with `WaitAsync` and `Release` in `finally` — the only correct "async lock", because the compiler forbids `lock`+`await` and working around it with `Monitor.Enter`+`await` deadlocks. Finally, `RunConcurrentQueueDemo` implements a producer/consumer on the lock-free `ConcurrentQueue<int>` with aggregation of local sums via `Interlocked.Add` — the lesson's recommended alternative to `Queue<T>` under `lock`, especially when the consumer may `await` inside processing.

#### Going deeper (bonus)
1. Replace `ConcurrentQueue<int>` with `System.Threading.Channels.Channel<int>` (bounded, capacity 1000) and compare throughput and peak memory under 8 producers / 8 consumers. Describe when `Channel` is preferable to `ConcurrentQueue`.
2. Add an experiment that demonstrates a `.Result` deadlock under a synchronization context: create your own `SynchronizationContext` (or use `DispatcherSynchronizationContext` in a WPF test), start an `async` method that internally calls another `async` method and blocks it via `.Result`. Show the deadlock and fix it with `ConfigureAwait(false)` in the inner library function.
3. Implement a long-lived "background daemon" on a `Thread` with `IsBackground = true` and explicit stopping via `CancellationToken` + `Thread.Join(timeout)`. Explain when such a `Thread` is more justified than `Task.Run` (long CPU-bound loop, special stack/priority needs).
4. Measure pool growth under pressure: launch 100 blocking `Task.Run(Thread.Sleep)` and, in parallel, every 100 ms print `ThreadPool.GetAvailableThreads` and `ThreadPool.GetMaxThreads`. Record how the pool grows up to the maximum and compare it with the "~1 thread/0.5 sec" heuristic from the lesson.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `ConcurrencyLab` собирается и запускается под .NET 8.
- [ ] (RU) Использованы top-level statements и C# 12; напечатан `Environment.ProcessorCount`.
- [ ] (RU) Реализованы все 8 экспериментов с двуязычными комментариями.
- [ ] (RU) `CancellationToken` проверяется в горячем цикле через `ThrowIfCancellationRequested()`.
- [ ] (RU) Нет `.Result`/`.Wait()` на горячем пути (кроме демо `Task.WaitAll`).
- [ ] (RU) Критическая секция с `await` использует `SemaphoreSlim`, а не `lock`.
- [ ] (RU) Вывод сохранён в `output.txt`, написан `README.md`.
- [ ] (EN) The `ConcurrencyLab` project builds and runs under .NET 8.
- [ ] (EN) Top-level statements and C# 12 are used; `Environment.ProcessorCount` is printed.
- [ ] (EN) All 8 experiments are implemented with bilingual comments.
- [ ] (EN) The `CancellationToken` is checked in the hot loop via `ThrowIfCancellationRequested()`.
- [ ] (EN) No `.Result`/`.Wait()` on a hot path (except the demo `Task.WaitAll`).
- [ ] (EN) The critical section with `await` uses `SemaphoreSlim`, not `lock`.
- [ ] (EN) Output is saved to `output.txt`; `README.md` is written.

#### Ресурсы / Resources
- [Microsoft Learn — Managed Threading Basics](https://learn.microsoft.com/dotnet/standard/threading/managed-threading-basics)
- [Microsoft Learn — The Managed Thread Pool](https://learn.microsoft.com/dotnet/standard/threading/the-managed-thread-pool)
- [Microsoft Learn — Task-based Asynchronous Pattern (TAP)](https://learn.microsoft.com/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap)
- [Microsoft Learn — CancellationToken](https://learn.microsoft.com/dotnet/api/system.threading.cancellationtoken)
- [Microsoft Learn — SemaphoreSlim](https://learn.microsoft.com/dotnet/api/system.threading.semaphoreslim)
- [Microsoft Learn — ConcurrentQueue&lt;T&gt;](https://learn.microsoft.com/dotnet/api/system.collections.concurrent.concurrentqueue-1)
- [Microsoft Learn — System.Threading.Channels](https://learn.microsoft.com/dotnet/api/system.threading.channels)

---
[← К уроку M11-L01](lesson-M11-L01-thread-threadpool-task.md) | [⬆ К модулю M11](../README.md) | [Следующее ДЗ →](homework-M11-L02-lock-monitor.md)
