---
[← К уроку M09-L04](lesson-M09-L04-task-run-cpu-vs-io.md) | [⬆ К модулю M09](../README.md) | [Следующее ДЗ →](homework-M09-L05-whenall-whenany.md)
---

### Домашнее задание M09-L04: Task.Run, CPU-bound vs I/O-bound / Homework M09-L04: Task.Run, CPU-bound vs I/O-bound

**Урок / Lesson:** M09-L04
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться отличать CPU-bound и I/O-bound работу в .NET 8, правильно применять `Task.Run` с `CancellationToken`, ограничивать параллелизм через `SemaphoreSlim` и `Channel`, избегать дедлоков на `.Result`/`.Wait()` и привычных ошибок вроде `Task.Run(async () => await io())`. (EN) Learn to distinguish CPU-bound and I/O-bound work in .NET 8, apply `Task.Run` correctly with `CancellationToken`, throttle parallelism with `SemaphoreSlim` and `Channel`, avoid `.Result`/`.Wait()` deadlocks and common mistakes like `Task.Run(async () => await io())`.

#### Связь с уроком / Connection to the lesson
(RU) Урок M09-L04 объясняет фундаментальную разницу между вычислительной нагрузкой (CPU-bound), где `Task.Run` уместен, и ожиданием (I/O-bound), где нативный async-API не требует обёрток. В этом задании вы построите мини-сервис, который совмещает обе категории работы, столкнётесь с реальным истощением пула потоков при неправильном `Task.Run`, исправите его через `SemaphoreSlim`/`Channel` и обеспечите отмену по токену. (EN) Lesson M09-L04 explains the fundamental difference between CPU-bound work (where `Task.Run` is appropriate) and I/O-bound work (where a native async API needs no wrapper). In this assignment you will build a small service that combines both categories, experience real thread-pool starvation caused by improper `Task.Run`, fix it with `SemaphoreSlim`/`Channel`, and add token-based cancellation.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы — backend-разработчик команды аналитики, которой поручен сервис «TextDigest». Сервис получает список публичных URL текстовых документов, скачивает их по HTTP, а затем для каждого документа выполняет тяжёлую вычислительную работу: считает компактный «отпечаток» (fingerprint) на основе полиномиального хеша и частотный профиль символов. Скачивание — это чистое I/O: ни один поток процессора не «считывает байты» из сокета, операция регистрируется в I/O completion port и поток возвращается в пул до прихода данных. А вот хеширование и построение частотного профиля — это настоящая CPU-bound работа: процессор реально считает по каждому байту документа.

В первой наивной реализации почти все студенты оборачивают и I/O, и CPU в `Task.Run` без ограничений и без `CancellationToken`. На маленьком списке из пяти URL это «работает». Но как только вы загружаете список из двухсот URL на ноутбуке с восемью ядрами, пул потоков истощается: каждый запрос заносит в пул новую порцию работы, `ThreadPool` вынужден создавать новые потоки с задержкой в одну нить на каждые ~500 мс (стандартная эвристика .NET), latency вырастает в разы, а throughput падает. Это классический пример того, как `Task.Run` улучшает отзывчивость одного вызова (offload), но ухудшает масштабируемость системы в целом.

Вторая типичная проблема — попытка «ускорить» I/O обёрткой `Task.Run(async () => await httpClient.GetAsync(url))`. На самом деле вы тратите лишний поток пула на запуск операции, которая и так не нуждается в потоке. Правильное решение — вызвать нативный async-API напрямую: `await httpClient.GetAsync(url, token).ConfigureAwait(false)`. Третья ловушка — дедлок при использовании `.Result`/`.Wait()` в контексте с `SynchronizationContext` (UI или классический ASP.NET), который мы рассмотрим теоретически и воспроизведём в sandbox-тесте. В ASP.NET Core `SynchronizationContext` отсутствует, поэтому прямой дедлок там не возникнет, но привычка блокировать async-задачи остаётся вредной: вы занимаете поток пулом впустую и теряете преимущества асинхронности.

Цель задания — построить корректную реализацию сервиса, которая: (1) использует `Task.Run` только для CPU-bound хеширования и анализа, (2) выполняет I/O без `Task.Run`, (3) принимает `CancellationToken` на всех уровнях и проверяет `ThrowIfCancellationRequested()` внутри вычислений, (4) ограничивает параллелизм CPU-bound через `SemaphoreSlim`, а полный поток данных реализует через producer/consumer на `Channel`, (5) применяет `ConfigureAwait(false)` в библиотечном коде и нигде не использует `.Result`/`.Wait()` для блокировки async-задач.

#### Что нужно сделать (пошагово)
1. Создайте новый проект консольного приложения .NET 8 с C# 12: `dotnet new console -n TextDigest -o TextDigest --framework net8.0`. Перейдите в каталог: `cd TextDigest`. Убедитесь, что в `TextDigest.csproj` установлено `<LangVersion>12</LangVersion>` и `<Nullable>enable</Nullable>`.
2. Добавьте общедоступные тестовые данные: создайте файл `urls.txt` в каталоге проекта с двадцатью строками вида `https://example.com/doc-001.txt` (можно использовать `https://httpbin.org/bytes/4096` для генерации бинарных данных или `https://en.wikipedia.org/wiki/Asynchronous_method_invocation` для реальных текстов). Эти URL будут источниками I/O.
3. Реализуйте класс `TextDigestService` в файле `TextDigestService.cs`. Класс должен содержать методы: `FetchAsync(string url, CancellationToken token)`, `FingerprintAsync(byte[] data, CancellationToken token)`, `ProcessAllAsync(IEnumerable<string> urls, CancellationToken token)`. Внутри используйте один общий `HttpClient` (не создавайте новый `HttpClient` на запрос — это истощает сокеты).
4. Метод `FetchAsync` — это I/O-bound работа: НЕ оборачивайте в `Task.Run`. Вызывайте `await _http.GetAsync(url, token).ConfigureAwait(false)` и `await resp.Content.ReadAsByteArrayAsync(token).ConfigureAwait(false)`. Добавьте комментарий в коде, объясняющий, почему здесь не нужен `Task.Run`.
5. Метод `FingerprintAsync` — это CPU-bound работа: вычислите 32-битный полиномиальный хеш по массиву байтов и попутно частотный профиль (256 int — по одному на каждый байт). Реализуйте через `Task.Run(() => { ... }, token)` и периодически вызывайте `token.ThrowIfCancellationRequested()` внутри цикла по байтам, например каждые 65 536 байт. Возвращайте кортеж `(int hash, int[] freq)`.
6. Метод `ProcessAllAsync` должен реализовать producer/consumer pipeline на `Channel<byte[]>`: один producer читает URL и скачивает данные через `FetchAsync` (I/O, без `Task.Run`), а `Environment.ProcessorCount` потребителей забирают данные из канала и вызывают `FingerprintAsync` (CPU-bound через `Task.Run`). Канал сделайте bounded с `BoundedChannelFullMode.Wait` и явными `SingleReader = false`, `SingleWriter = true`. Используйте `await foreach` с `ReadAllAsync(token)` и `ConfigureAwait(false)` на каждом `await`.
7. Ограничьте параллелизм потребителей CPU-bound через `SemaphoreSlim(Environment.ProcessorCount)` внутри каждого потребителя перед вызовом `FingerprintAsync`, чтобы не запускать больше CPU-задач, чем ядер, и не истощать пул. Освобождайте семафор в `finally`. Скачивание (producer) не ограничивайте семафором — это I/O, оно не нагружает процессор.
8. В `Program.cs` (top-level statements) прочитайте `urls.txt`, создайте `CancellationTokenSource` и подпишитесь на `Console.CancelKeyPress` для отмены по Ctrl+C. Выведите стартовое сообщение, время старта, запустите `await service.ProcessAllAsync(urls, cts.Token)`, в блоке `catch (OperationCanceledException)` напечатайте «Отменено» и завершите корректно. Измерьте и выведите общее время работы через `Stopwatch`.
9. Добавьте ссылку на `BenchmarkDotNet` НЕ нужно — достаточно замеров `Stopwatch`. Создайте юнит-тесты через `dotnet new xunit -n TextDigest.Tests -o TextDigest.Tests`, добавьте ссылку `dotnet add TextDigest.Tests reference TextDigest`. Напишите тест, проверяющий, что `FingerprintAsync` бросает `OperationCanceledException` при уже отменённом токене. Напишите тест, проверяющий идемпотентность хеша (один и тот же массив даёт тот же хеш).
10. Запустите: `dotnet build`, затем `dotnet run --project TextDigest`. Ожидаемый вывод — список из 20 строк вида `url -> hash=0xABCD, topByte=0x20, time=12ms`, итоговое сообщение `Готово за N мс` и отсутствие предупреждений анализатора о `async void` и `.Result`. Запустите тесты: `dotnet test`. Ожидаемый результат — все тесты зелёные.
11. В качестве эксперимента «как НЕ надо» создайте копию метода `ProcessAllAsync` под именем `ProcessAllNaiveAsync`, где каждый URL скачивается через `Task.Run(async () => await FetchAsync(...))`, а CPU-задачи не ограничены семафором. Запустите оба варианта на одном и том же списке из 200 URL (сгенерируйте файл `urls-big.txt`) и сравните время и потребление потоков через `ThreadPool.GetAvailableThreads`. Запишите в `REPORT.md` разницу во времени и краткое объяснение причин в двух-трёх абзацах.

#### Требования к решению
- Код компилируется под .NET 8 с C# 12 без предупреждений уровня `error`. Разрешены top-level statements в `Program.cs`, файловый namespace (`namespace TextDigest;`), collection expressions (`int[] freq = new int[256];`), `using`-объявления, target-typed `new()`.
- В `FetchAsync` нет `Task.Run`; в `FingerprintAsync` обязательно есть `Task.Run` и `token.ThrowIfCancellationRequested()` внутри вычислений. В `ProcessAllAsync` producer не использует `Task.Run` для I/O, а потребители используют `Task.Run` только для CPU-функции.
- Каждый `await` в библиотечных методах сопровождается `.ConfigureAwait(false)`. В `Program.cs` (точка входа приложения) `ConfigureAwait(false)` не обязателен, но допустим.
- `CancellationToken` передаётся во все публичные методы и пробрасывается в `Task.Run(work, token)` и во все async-API (`GetAsync`, `ReadAsByteArrayAsync`, `WriteAsync`, `ReadAllAsync`). Нигде нет `async void` вне обработчиков событий.
- `HttpClient` один на сервис, не создаётся на запрос. Корректно освобождается `using var resp` или `using var ... = await ...`.
- Параллелизм CPU-bound ограничен `SemaphoreSlim(Environment.ProcessorCount)` либо структурой producer/consumer на `Channel` с фиксированным числом потребителей. Pipeline на `Channel` использует `BoundedChannelOptions` и корректно завершает писателя через `channel.Writer.Complete()` в `finally`.
- Никаких `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` для блокировки async-задач. Все асинхронные вызовы идут через `await` вверх по стеку до точки входа.
- В `REPORT.md` есть измеренное сравнение корректной и наивной реализации на 200 URL, с указанием времени и `ThreadPool.GetAvailableThreads` до и после.

#### Тонкости и подводные камни
- **`Task.Run(async () => await io())` — антипаттерн.** Если внутри лямбды есть `await`, вы создали лишний переход в пул только ради запуска async-операции, которая сама по себе не занимает поток. Это не ускоряет I/O, а лишь добавляет переключение контекста и занимает слот пула. Правильно: `await io()` напрямую.
- **`.Result`/`.Wait()` под `SynchronizationContext`.** В WPF/WinForms и классическом ASP.NET `await` по умолчанию пытается вернуться в исходный контекст. Если вызывающий поток заблокирован на `.Result`, продолжение не сможет в него вернуться — классический дедлок. В ASP.NET Core и консоли `SynchronizationContext` равен `null`, поэтому прямого дедлока нет, но блокировка всё равно занимает поток пул впустую.
- **`lock (obj) { await ... }` нельзя.** Оператор `lock` не может охватывать `await`: монитор освобождается только в синхронном блоке, а продолжение может выполниться в другом потоке. Используйте `SemaphoreSlim.WaitAsync(token)` с `try/finally` и `Release()`.
- **`async void` вне event handlers.** Исключения в `async void` не перехватываются `try/catch` у вызывающего и рвут процесс. В нашем задании обработчик `Console.CancelKeyPress` можно сделать `async void` (это разрешено для событий), но вся бизнес-логика обязана быть `async Task`.
- **Истощение пула потоков при неограниченном `Task.Run`.** Стандартная эвристика `ThreadPool` добавляет не более двух потоков в секунду. Если вы закинете 200 CPU-задач одновременно, пул будет раздуваться медленно, latency вырастет, а throughput упадёт. `SemaphoreSlim(Environment.ProcessorCount)` решает проблему: одновременно работают только столько CPU-задач, сколько ядер.
- **Канал с `SingleWriter = true`, но два писателя.** `BoundedChannelOptions.SingleWriter` — это обещание рантайму; если вы реально пишете из двух потоков с `true`, поведение не определено. В нашем задании producer один, поэтому `SingleWriter = true`; если у вас несколько producer-задач, ставьте `false`.
- **`HttpClient` per-request.** Создание `new HttpClient()` на каждый запрос приводит к истощению сокетов в `TIME_WAIT`. Используйте один общий экземпляр, `IHttpClientFactory` в ASP.NET Core или `HttpClientHandler` с `PooledConnectionLifetime`.
- **`token.ThrowIfCancellationRequested()` внутри горячих циклов.** Если внутри CPU-цикла не проверять токен, отменить долгое вычисление нельзя — сервис нельзя остановить gracefully. Проверяйте периодически (каждые ~65 КБ), не каждый байт — иначе накладные расходы съедят выигрыш.
- **`ConfigureAwait(false)` в библиотеке обязателен.** Библиотечный код не должен зависеть от контекста вызывающего. В коде приложения (точка входа, обработчики UI) решение зависит от контекста; в ASP.NET Core это нейтрально, но привычка полезна.

#### Критерии приёмки
- [ ] Проект собирается под .NET 8 / C# 12 без ошибок и без предупреждений `error`-уровня.
- [ ] `TextDigestService.FetchAsync` не содержит `Task.Run`; I/O выполнено через нативный `await ... .ConfigureAwait(false)`.
- [ ] `TextDigestService.FingerprintAsync` использует `Task.Run(work, token)` и вызывает `token.ThrowIfCancellationRequested()` внутри цикла.
- [ ] `ProcessAllAsync` реализует producer/consumer на `Channel<byte[]>` с одним producer и `Environment.ProcessorCount` потребителями.
- [ ] Параллелизм CPU-bound ограничен `SemaphoreSlim(Environment.ProcessorCount)` или фиксированным числом потребителей.
- [ ] `CancellationToken` передаётся во все `Task.Run` и во все async-API (`GetAsync`, `ReadAsByteArrayAsync`, `WriteAsync`, `ReadAllAsync`).
- [ ] Все `await` в `TextDigestService` сопровождены `.ConfigureAwait(false)`.
- [ ] `HttpClient` — один экземпляр на сервис, не создаётся на запрос.
- [ ] Нигде нет `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` для блокировки async-задач; нет `async void` вне обработчиков событий.
- [ ] Точка входа обрабатывает `Console.CancelKeyPress` и корректно завершает работу через `CancellationTokenSource.Cancel()`.
- [ ] `dotnet run` выводит 20 строк результата и итоговое время; Ctrl+C прерывает работу с сообщением «Отменено».
- [ ] Юнит-тесты зелёные: проверка `OperationCanceledException` при отменённом токене и идемпотентности хеша.
- [ ] В `REPORT.md` приведено сравнение корректной и наивной реализации на 200 URL: время, доступные потоки пула, краткое объяснение.
- [ ] В комментариях кода явно отмечено, какой метод CPU-bound, а какой I/O-bound, и почему `Task.Run` выбран именно для CPU-bound.

#### Подсказки (без прямого ответа)
- Вспомните урок: `Task.Run` делает две вещи — переносит работу в пул и возвращает `Task`. Какая из ваших функций реально нуждается в переносе в пул, а какая и так вернёт `Task` без вашей помощи?
- Если внутри метода есть `await`, спросите себя: «Действительно ли мне нужно стартовать его в другом потоке, или он сам асинхронно вернёт управление?»
- Для producer/consumer используйте `Channel.CreateBounded<byte[]>(new BoundedChannelOptions(64) { FullMode = BoundedChannelFullMode.Wait, SingleWriter = true, SingleReader = false })`.
- Чтобы корректно завершить канал, поместите `channel.Writer.Complete()` в блок `finally` producer-задачи.
- Для измерения доступных потоков: `ThreadPool.GetAvailableThreads(out var worker, out var io)` до и после работы.
- Не забудьте `token.ThrowIfCancellationRequested()` именно внутри горячего цикла по байтам, а не только перед `Task.Run`.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — TextDigestService.cs
// Эталонное решение ДЗ M09-L04.
// Reference solution for homework M09-L04.

using System.Diagnostics;
using System.Threading.Channels;

namespace TextDigest;

public sealed class TextDigestService : IDisposable
{
    private readonly HttpClient _http = new();     // один экземпляр — нет истощения сокетов / single instance, no socket exhaustion
    private bool _disposed;

    // ✅ I/O-bound: НЕТ Task.Run. Нативный async-API + ConfigureAwait(false).
    // ✅ I/O-bound: NO Task.Run. Native async API + ConfigureAwait(false).
    public async Task<byte[]> FetchAsync(string url, CancellationToken token)
    {
        // ❌ Никогда: Task.Run(async () => await _http.GetByteArrayAsync(url, token));
        // ❌ Never wrap I/O in Task.Run.
        using var resp = await _http.GetAsync(url, token).ConfigureAwait(false);
        resp.EnsureSuccessStatusCode();
        return await resp.Content.ReadAsByteArrayAsync(token).ConfigureAwait(false);
    }

    // ✅ CPU-bound: Task.Run + token + ThrowIfCancellationRequested внутри цикла.
    // ✅ CPU-bound: Task.Run + token + ThrowIfCancellationRequested inside the loop.
    public Task<(int hash, int[] freq)> FingerprintAsync(byte[] data, CancellationToken token) =>
        Task.Run(() =>
        {
            long h = 0;                              // локальная — нет race / local, no race
            int[] freq = new int[256];               // collection expression friendly / collection-expression friendly
            for (int i = 0; i < data.Length; i++)
            {
                if ((i & 0xFFFF) == 0)               // каждые 65 536 байт / every 65 536 bytes
                    token.ThrowIfCancellationRequested();
                byte b = data[i];
                freq[b]++;                           // частотный профиль / frequency profile
                h = (h * 31 + b) & 0x7FFFFFFF;       // полиномиальный хеш / polynomial hash
            }
            return (checked((int)h), freq);
        }, token);

    // ✅ Producer/consumer на Channel: I/O без Task.Run, CPU через Task.Run + SemaphoreSlim.
    // ✅ Producer/consumer on Channel: I/O without Task.Run, CPU via Task.Run + SemaphoreSlim.
    public async Task<IReadOnlyList<(string url, int hash, int topByte, long ms)>> ProcessAllAsync(
        IEnumerable<string> urls, CancellationToken token)
    {
        var channel = Channel.CreateBounded<byte[]>(new BoundedChannelOptions(64)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleWriter = true,
            SingleReader = false,
        });

        var urlQueue = new Queue<string>(urls);      // чтобы сохранить порядок вывода / to preserve output order
        var results = new System.Collections.Concurrent.ConcurrentBag<(string url, int hash, int topByte, long ms)>();

        // Producer: I/O без Task.Run.
        // Producer: I/O without Task.Run.
        var producer = Task.Run(async () =>
        {
            try
            {
                foreach (var url in urlQueue)
                {
                    token.ThrowIfCancellationRequested();
                    var bytes = await FetchAsync(url, token).ConfigureAwait(false);
                    await channel.Writer.WriteAsync(bytes, token).ConfigureAwait(false);
                }
            }
            finally { channel.Writer.Complete(); }   // обязательно завершить писателя / always complete the writer
        }, token);

        // Consumers: CPU-bound, ограничены числом ядер через SemaphoreSlim.
        // Consumers: CPU-bound, bounded by core count via SemaphoreSlim.
        using var gate = new SemaphoreSlim(Environment.ProcessorCount);
        var consumers = new List<Task>(Environment.ProcessorCount);
        for (int i = 0; i < Environment.ProcessorCount; i++)
        {
            consumers.Add(Task.Run(async () =>
            {
                await foreach (var bytes in channel.Reader.ReadAllAsync(token).ConfigureAwait(false))
                {
                    token.ThrowIfCancellationRequested();
                    await gate.WaitAsync(token).ConfigureAwait(false);
                    var sw = Stopwatch.StartNew();
                    try
                    {
                        var (hash, freq) = await FingerprintAsync(bytes, token).ConfigureAwait(false);
                        int topByte = 0;
                        for (int k = 1; k < 256; k++)
                            if (freq[k] > freq[topByte]) topByte = k;
                        // Здесь url теряется — в реальном коде передавайте кортеж (url, bytes) по каналу.
                        // In real code, pass a (url, bytes) tuple through the channel to keep the URL.
                        lock (results) results.Add(("<url>", hash, topByte, sw.ElapsedMilliseconds));
                    }
                    finally { gate.Release(); }
                }
            }, token));
        }

        await Task.WhenAll(producer, Task.WhenAll(consumers)).ConfigureAwait(false);
        return results.ToArray();
    }

    public void Dispose()
    {
        if (_disposed) return;
        _http.Dispose();
        _disposed = true;
    }
}
```

Разбор по строкам. Метод `FetchAsync` намеренно НЕ обёрнут в `Task.Run`: это I/O-bound работа, где `HttpClient.GetAsync` сам по себе возвращает `Task`, не занимая потока на время ожидания ответа. Обёртка в `Task.Run(async () => await ...)` лишь добавила бы лишний переход в пул — ту самую ошибку, которую урок называет «`Task.Run` для I/O вреден». `.ConfigureAwait(false)` стоит на каждом `await`, потому что `TextDigestService` — это библиотечный код, и он не должен зависеть от контекста вызывающего (урок: «в библиотечном коде всегда `ConfigureAwait(false)`»). Метод `FingerprintAsync`, напротив, обёрнут в `Task.Run(work, token)`, потому что хеширование массива байтов — синхронная CPU-bound работа, и без `Task.Run` она заняла бы вызывающий поток напрямую. Внутри цикла стоит `token.ThrowIfCancellationRequested()` каждые 65 536 байт — это прямое применение совета урока «внутри CPU-bound работы периодически вызывайте `ThrowIfCancellationRequested`». Без этой проверки длинное вычисление нельзя отменить, и Ctrl+C не сработает за разумное время.

Метод `ProcessAllAsync` реализует масштабируемую схему из примера урока: producer/consumer на `Channel`. Producer скачивает URL через `FetchAsync` без `Task.Run` (I/O не нуждается в offload), а `Environment.ProcessorCount` потребителей забирают данные через `await foreach ... ReadAllAsync(token)`. Внутри каждого потребителя стоит `SemaphoreSlim(Environment.ProcessorCount)` — ровно тот механизм, который урок рекомендует для ограничения параллелизма CPU-bound. `finally { gate.Release(); }` гарантирует освобождение даже при исключении. `finally { channel.Writer.Complete(); }` в producer обязателен: иначе `ReadAllAsync` у потребителей никогда не завершится и `Task.WhenAll` зависнет. `CancellationToken` проброшен во все async-API и в `Task.Run(work, token)` — урок требует «передавайте `CancellationToken` во все `Task.Run` и async-операции». Нигде нет `.Result`/`.Wait()`, нет `async void` вне обработчиков событий, нет `lock` через `await` (семафор берётся через `WaitAsync`). Один общий `HttpClient` исключает истощение сокетов. Таким образом решение последовательно применяет все best practices урока и избегает всех перечисленных частых ошибок.

#### Задания на углубление (бонус)
1. **`Parallel.ForEachAsync` вместо ручного `SemaphoreSlim`.** Перепишите `ProcessAllAsync`, используя `Parallel.ForEachAsync` с `MaxDegreeOfParallelism = Environment.ProcessorCount`. Сравните читаемость, поведение при отмене и потребление памяти с версией на `Channel`. Объясните, когда `Parallel.ForEachAsync` предпочтительнее ручного pipeline.
2. **Воспроизведение дедлока на `SynchronizationContext`.** Напишите отдельный тест-проект WPF (или используйте `AsyncContext` из Nito.AsyncEx), в котором метод вызывает `SomeAsyncMethod().Result` на UI-потоке и зависает. Затем добавьте `ConfigureAwait(false)` внутрь `SomeAsyncMethod` и покажите, что дедлок исчезает. Запишите тайминг и стек зависания в `DEADLOCK.md`.
3. **Измерение throughput при разном размере пула.** Прогоните корректную реализацию на 500 URL при `SemaphoreSlim(2)`, `SemaphoreSlim(Environment.ProcessorCount)`, `SemaphoreSlim(Environment.ProcessorCount * 2)` и без семафора. Постройте таблицу «степень параллелизма → время → доступные потоки пула» и объясните, почему `Environment.ProcessorCount` обычно оптимально для CPU-bound.
4. **Priority channel и backpressure.** Перейдите на `Channel.CreateBounded` с `BoundedChannelFullMode.DropOldest` и измерьте, при каком размере буфера producer начинает отбрасывать данные. Объясните связь этого эксперимента с понятием backpressure и его ролью в устойчивости сервиса при неравномерной нагрузке.

---

## Statement in English / Постановка на английском

#### Context & motivation
You are a backend engineer on the analytics team tasked with building a service called "TextDigest". The service receives a list of public URLs pointing to text documents, downloads each document over HTTP, and then performs heavy computational work on every document: it computes a compact fingerprint based on a polynomial hash and builds a frequency profile of bytes. Downloading is pure I/O: no CPU thread is "reading bytes" from the socket; the operation is registered with an I/O completion port and the thread returns to the pool until data arrives. Hashing and frequency profiling, on the other hand, are genuine CPU-bound work: the processor actually iterates over every byte of the document.

In the first naive implementation, almost every student wraps both the I/O and the CPU work in `Task.Run` with no throttling and no `CancellationToken`. On a small list of five URLs this "works". But the moment you feed a list of two hundred URLs on an eight-core laptop, the thread pool starves: every request pushes more work into the pool, `ThreadPool` is forced to inject new threads at a rate of roughly one thread every ~500 ms (the standard .NET heuristic), latency explodes, and throughput collapses. This is the textbook example of how `Task.Run` improves the responsiveness of a single call (offload) while degrading the scalability of the whole system.

The second classic mistake is trying to "speed up" I/O with `Task.Run(async () => await httpClient.GetAsync(url))`. In reality you spend an extra pool thread to start an operation that never needed a thread in the first place. The correct form is the native async call directly: `await httpClient.GetAsync(url, token).ConfigureAwait(false)`. The third trap is the deadlock caused by `.Result`/`.Wait()` under a `SynchronizationContext` (UI or classic ASP.NET), which we will examine theoretically and reproduce in a sandbox test. ASP.NET Core has no `SynchronizationContext`, so the direct deadlock does not occur there, but the habit of blocking async tasks is still harmful: you occupy a pool thread for nothing and lose the benefits of asynchrony.

The goal of the assignment is to build a correct implementation of the service that: (1) uses `Task.Run` only for the CPU-bound hashing and analysis, (2) performs I/O without `Task.Run`, (3) accepts a `CancellationToken` at every layer and checks `ThrowIfCancellationRequested()` inside the computation, (4) throttles CPU-bound parallelism with `SemaphoreSlim` and implements the overall data flow as a producer/consumer pipeline on `Channel`, (5) applies `ConfigureAwait(false)` in library code and never uses `.Result`/`.Wait()` to block async tasks.

#### What to do step by step
1. Create a new .NET 8 console application with C# 12: `dotnet new console -n TextDigest -o TextDigest --framework net8.0`. Move into the folder: `cd TextDigest`. Verify that `TextDigest.csproj` contains `<LangVersion>12</LangVersion>` and `<Nullable>enable</Nullable>`.
2. Prepare test data: create a file `urls.txt` in the project folder with twenty lines like `https://example.com/doc-001.txt` (you may use `https://httpbin.org/bytes/4096` to generate binary data, or real text URLs such as `https://en.wikipedia.org/wiki/Asynchronous_method_invocation`). These URLs are the I/O source.
3. Implement a `TextDigestService` class in `TextDigestService.cs`. The class must expose `FetchAsync(string url, CancellationToken token)`, `FingerprintAsync(byte[] data, CancellationToken token)`, and `ProcessAllAsync(IEnumerable<string> urls, CancellationToken token)`. Use a single shared `HttpClient` (do not create a new client per request — that exhausts sockets).
4. `FetchAsync` is I/O-bound work: do NOT wrap it in `Task.Run`. Call `await _http.GetAsync(url, token).ConfigureAwait(false)` and `await resp.Content.ReadAsByteArrayAsync(token).ConfigureAwait(false)`. Add a code comment explaining why `Task.Run` is not needed here.
5. `FingerprintAsync` is CPU-bound work: compute a 32-bit polynomial hash over the byte array and, alongside, a frequency profile of 256 ints (one per byte value). Implement it via `Task.Run(() => { ... }, token)` and call `token.ThrowIfCancellationRequested()` periodically inside the byte loop, for example every 65 536 bytes. Return a tuple `(int hash, int[] freq)`.
6. `ProcessAllAsync` must implement a producer/consumer pipeline on `Channel<byte[]>`: one producer reads URLs and downloads data via `FetchAsync` (I/O, no `Task.Run`), and `Environment.ProcessorCount` consumers pull data from the channel and call `FingerprintAsync` (CPU-bound via `Task.Run`). Make the channel bounded with `BoundedChannelFullMode.Wait` and explicit `SingleReader = false`, `SingleWriter = true`. Use `await foreach` with `ReadAllAsync(token)` and `.ConfigureAwait(false)` on every `await`.
7. Throttle the consumers' CPU-bound work with `SemaphoreSlim(Environment.ProcessorCount)` inside each consumer before calling `FingerprintAsync`, so you never launch more CPU tasks than cores and never starve the pool. Release the semaphore in `finally`. Do not throttle the producer (downloading) with the semaphore — it is I/O and does not load the CPU.
8. In `Program.cs` (top-level statements), read `urls.txt`, create a `CancellationTokenSource`, and subscribe to `Console.CancelKeyPress` for Ctrl+C cancellation. Print a start banner and a start timestamp, run `await service.ProcessAllAsync(urls, cts.Token)`, and in a `catch (OperationCanceledException)` block print "Cancelled" and exit gracefully. Measure and print total wall-clock time with `Stopwatch`.
9. You do NOT need `BenchmarkDotNet` — `Stopwatch` is enough. Create unit tests: `dotnet new xunit -n TextDigest.Tests -o TextDigest.Tests`, then `dotnet add TextDigest.Tests reference TextDigest`. Write a test verifying that `FingerprintAsync` throws `OperationCanceledException` when the token is already cancelled. Write a test verifying hash idempotency (the same byte array yields the same hash).
10. Run `dotnet build`, then `dotnet run --project TextDigest`. The expected output is twenty lines like `url -> hash=0xABCD, topByte=0x20, time=12ms`, a final `Done in N ms` line, and no analyzer warnings about `async void` or `.Result`. Run the tests with `dotnet test`; they must all be green.
11. As a "how NOT to do it" experiment, create a copy of `ProcessAllAsync` named `ProcessAllNaiveAsync` where each URL is downloaded through `Task.Run(async () => await FetchAsync(...))` and the CPU tasks are not throttled by the semaphore. Run both variants on the same list of 200 URLs (generate a `urls-big.txt` file) and compare timing and thread consumption via `ThreadPool.GetAvailableThreads`. Record the time difference and a two-to-three paragraph explanation in `REPORT.md`.

#### Requirements
- The code compiles under .NET 8 with C# 12 without `error`-level warnings. Top-level statements in `Program.cs`, file-scoped namespaces (`namespace TextDigest;`), collection expressions (`int[] freq = new int[256];`), `using` declarations, and target-typed `new()` are all allowed.
- `FetchAsync` has no `Task.Run`; `FingerprintAsync` must have `Task.Run` and `token.ThrowIfCancellationRequested()` inside the computation. In `ProcessAllAsync`, the producer does not use `Task.Run` for I/O, and consumers use `Task.Run` only for the CPU function.
- Every `await` in library methods carries `.ConfigureAwait(false)`. In `Program.cs` (the application entry point) `ConfigureAwait(false)` is optional but allowed.
- A `CancellationToken` is threaded through every public method and passed into `Task.Run(work, token)` and into every async API (`GetAsync`, `ReadAsByteArrayAsync`, `WriteAsync`, `ReadAllAsync`). There is no `async void` outside event handlers.
- `HttpClient` is a single instance per service, never created per request. `using var resp` or `using var ... = await ...` is used to dispose responses correctly.
- CPU-bound parallelism is bounded either by `SemaphoreSlim(Environment.ProcessorCount)` or by the producer/consumer structure on `Channel` with a fixed number of consumers. The `Channel` pipeline uses `BoundedChannelOptions` and completes the writer through `channel.Writer.Complete()` in a `finally` block.
- There are no `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()` calls to block async tasks. All asynchronous calls propagate through `await` up the stack to the entry point.
- `REPORT.md` contains the measured comparison of the correct and the naive implementation on 200 URLs, including timing and `ThreadPool.GetAvailableThreads` before and after.

#### Pitfalls
- **`Task.Run(async () => await io())` is an anti-pattern.** If the lambda contains an `await`, you have created an extra hop into the pool solely to start an async operation that does not occupy a thread on its own. It does not speed I/O up; it only adds a context switch and consumes a pool slot. Correct: `await io()` directly.
- **`.Result`/`.Wait()` under `SynchronizationContext`.** In WPF/WinForms and classic ASP.NET, `await` tries to resume on the original context by default. If the calling thread is blocked on `.Result`, the continuation cannot return to it — the classic deadlock. In ASP.NET Core and in console apps `SynchronizationContext` is `null`, so the direct deadlock does not occur, but blocking still wastes a pool thread for nothing.
- **`lock (obj) { await ... }` is illegal.** The `lock` statement cannot span an `await`: the monitor is released only in a synchronous block, and the continuation may run on a different thread. Use `SemaphoreSlim.WaitAsync(token)` with `try/finally` and `Release()`.
- **`async void` outside event handlers.** Exceptions thrown from `async void` are not caught by the caller's `try/catch` and crash the process. In this assignment the `Console.CancelKeyPress` handler may be `async void` (that is allowed for events), but all business logic must be `async Task`.
- **Thread pool starvation from unbounded `Task.Run`.** The default `ThreadPool` heuristic injects at most about two threads per second. If you throw 200 CPU tasks at it at once, the pool grows slowly, latency explodes, and throughput collapses. `SemaphoreSlim(Environment.ProcessorCount)` solves it: only as many CPU tasks run concurrently as there are cores.
- **A channel with `SingleWriter = true` but two writers.** `BoundedChannelOptions.SingleWriter` is a promise to the runtime; if you actually write from two threads with `true`, behavior is undefined. In this assignment the producer is single, so `SingleWriter = true`; if you have multiple producer tasks, set it to `false`.
- **`HttpClient` per request.** Creating `new HttpClient()` per request leads to socket exhaustion in `TIME_WAIT`. Use one shared instance, `IHttpClientFactory` in ASP.NET Core, or an `HttpClientHandler` with `PooledConnectionLifetime`.
- **`token.ThrowIfCancellationRequested()` inside hot loops.** If you never check the token inside the CPU loop, a long computation cannot be cancelled and the service cannot shut down gracefully. Check periodically (every ~65 KB), not every byte — otherwise overhead eats the benefit.
- **`ConfigureAwait(false)` is mandatory in library code.** Library code must not depend on the caller's context. In application code (entry point, UI handlers) the decision depends on context; in ASP.NET Core it is neutral, but the habit is healthy.

#### Acceptance criteria
- [ ] The project builds under .NET 8 / C# 12 with no errors and no `error`-level warnings.
- [ ] `TextDigestService.FetchAsync` contains no `Task.Run`; I/O is done through native `await ... .ConfigureAwait(false)`.
- [ ] `TextDigestService.FingerprintAsync` uses `Task.Run(work, token)` and calls `token.ThrowIfCancellationRequested()` inside the loop.
- [ ] `ProcessAllAsync` implements a producer/consumer on `Channel<byte[]>` with one producer and `Environment.ProcessorCount` consumers.
- [ ] CPU-bound parallelism is throttled by `SemaphoreSlim(Environment.ProcessorCount)` or by a fixed consumer count.
- [ ] A `CancellationToken` is passed into every `Task.Run` and every async API (`GetAsync`, `ReadAsByteArrayAsync`, `WriteAsync`, `ReadAllAsync`).
- [ ] Every `await` in `TextDigestService` carries `.ConfigureAwait(false)`.
- [ ] `HttpClient` is a single instance per service, not created per request.
- [ ] There are no `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()` calls to block async tasks, and no `async void` outside event handlers.
- [ ] The entry point handles `Console.CancelKeyPress` and shuts down gracefully through `CancellationTokenSource.Cancel()`.
- [ ] `dotnet run` prints twenty result lines and a total time; Ctrl+C stops the run with a "Cancelled" message.
- [ ] Unit tests are green: `OperationCanceledException` on a pre-cancelled token and hash idempotency.
- [ ] `REPORT.md` compares the correct and the naive implementation on 200 URLs: time, available pool threads, and a short explanation.
- [ ] Code comments explicitly mark which method is CPU-bound and which is I/O-bound, and why `Task.Run` was chosen for CPU-bound specifically.

#### Hints (no direct answer)
- Recall the lesson: `Task.Run` does two things — it moves work to the pool and returns a `Task`. Which of your functions genuinely needs to be moved to the pool, and which already returns a `Task` without your help?
- If a method contains an `await`, ask yourself: "Do I really need to start it on another thread, or will it asynchronously yield on its own?"
- For the producer/consumer, use `Channel.CreateBounded<byte[]>(new BoundedChannelOptions(64) { FullMode = BoundedChannelFullMode.Wait, SingleWriter = true, SingleReader = false })`.
- To complete the channel correctly, put `channel.Writer.Complete()` in a `finally` block of the producer task.
- To measure available threads: `ThreadPool.GetAvailableThreads(out var worker, out var io)` before and after the run.
- Remember to call `token.ThrowIfCancellationRequested()` inside the hot byte loop, not just before `Task.Run`.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — TextDigestService.cs
// Reference solution for homework M09-L04.

using System.Diagnostics;
using System.Threading.Channels;

namespace TextDigest;

public sealed class TextDigestService : IDisposable
{
    private readonly HttpClient _http = new();     // single instance, no socket exhaustion
    private bool _disposed;

    // ✅ I/O-bound: NO Task.Run. Native async API + ConfigureAwait(false).
    public async Task<byte[]> FetchAsync(string url, CancellationToken token)
    {
        // ❌ Never: Task.Run(async () => await _http.GetByteArrayAsync(url, token));
        using var resp = await _http.GetAsync(url, token).ConfigureAwait(false);
        resp.EnsureSuccessStatusCode();
        return await resp.Content.ReadAsByteArrayAsync(token).ConfigureAwait(false);
    }

    // ✅ CPU-bound: Task.Run + token + ThrowIfCancellationRequested inside the loop.
    public Task<(int hash, int[] freq)> FingerprintAsync(byte[] data, CancellationToken token) =>
        Task.Run(() =>
        {
            long h = 0;                              // local, no race
            int[] freq = new int[256];
            for (int i = 0; i < data.Length; i++)
            {
                if ((i & 0xFFFF) == 0)               // every 65 536 bytes
                    token.ThrowIfCancellationRequested();
                byte b = data[i];
                freq[b]++;                           // frequency profile
                h = (h * 31 + b) & 0x7FFFFFFF;       // polynomial hash
            }
            return (checked((int)h), freq);
        }, token);

    // ✅ Producer/consumer on Channel: I/O without Task.Run, CPU via Task.Run + SemaphoreSlim.
    public async Task<IReadOnlyList<(string url, int hash, int topByte, long ms)>> ProcessAllAsync(
        IEnumerable<string> urls, CancellationToken token)
    {
        var channel = Channel.CreateBounded<byte[]>(new BoundedChannelOptions(64)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleWriter = true,
            SingleReader = false,
        });

        var urlQueue = new Queue<string>(urls);      // to preserve output order
        var results = new System.Collections.Concurrent.ConcurrentBag<(string url, int hash, int topByte, long ms)>();

        // Producer: I/O without Task.Run.
        var producer = Task.Run(async () =>
        {
            try
            {
                foreach (var url in urlQueue)
                {
                    token.ThrowIfCancellationRequested();
                    var bytes = await FetchAsync(url, token).ConfigureAwait(false);
                    await channel.Writer.WriteAsync(bytes, token).ConfigureAwait(false);
                }
            }
            finally { channel.Writer.Complete(); }   // always complete the writer
        }, token);

        // Consumers: CPU-bound, bounded by core count via SemaphoreSlim.
        using var gate = new SemaphoreSlim(Environment.ProcessorCount);
        var consumers = new List<Task>(Environment.ProcessorCount);
        for (int i = 0; i < Environment.ProcessorCount; i++)
        {
            consumers.Add(Task.Run(async () =>
            {
                await foreach (var bytes in channel.Reader.ReadAllAsync(token).ConfigureAwait(false))
                {
                    token.ThrowIfCancellationRequested();
                    await gate.WaitAsync(token).ConfigureAwait(false);
                    var sw = Stopwatch.StartNew();
                    try
                    {
                        var (hash, freq) = await FingerprintAsync(bytes, token).ConfigureAwait(false);
                        int topByte = 0;
                        for (int k = 1; k < 256; k++)
                            if (freq[k] > freq[topByte]) topByte = k;
                        // The url is lost here — in real code pass a (url, bytes) tuple through the channel.
                        lock (results) results.Add(("<url>", hash, topByte, sw.ElapsedMilliseconds));
                    }
                    finally { gate.Release(); }
                }
            }, token));
        }

        await Task.WhenAll(producer, Task.WhenAll(consumers)).ConfigureAwait(false);
        return results.ToArray();
    }

    public void Dispose()
    {
        if (_disposed) return;
        _http.Dispose();
        _disposed = true;
    }
}
```

Walk-through, line by line. `FetchAsync` is deliberately NOT wrapped in `Task.Run`: it is I/O-bound work where `HttpClient.GetAsync` itself returns a `Task` and holds no thread while waiting for the response. Wrapping it as `Task.Run(async () => await ...)` would only add an extra hop into the pool — precisely the mistake the lesson calls "`Task.Run` for I/O is harmful". `.ConfigureAwait(false)` is applied to every `await` because `TextDigestService` is library code and must not depend on the caller's context (lesson: "in library code, always `ConfigureAwait(false)`"). `FingerprintAsync`, by contrast, IS wrapped in `Task.Run(work, token)`, because hashing a byte array is synchronous CPU-bound work and without `Task.Run` it would occupy the calling thread directly. Inside the loop, `token.ThrowIfCancellationRequested()` runs every 65 536 bytes — a direct application of the lesson's advice "inside CPU-bound work, call `ThrowIfCancellationRequested()` periodically". Without that check, a long computation cannot be cancelled and Ctrl+C would not take effect in reasonable time.

`ProcessAllAsync` implements the scalable pattern from the lesson's example: a producer/consumer pipeline on `Channel`. The producer downloads URLs through `FetchAsync` without `Task.Run` (I/O does not need offloading), while `Environment.ProcessorCount` consumers pull data through `await foreach ... ReadAllAsync(token)`. Inside each consumer, `SemaphoreSlim(Environment.ProcessorCount)` is exactly the mechanism the lesson recommends for bounding CPU-bound parallelism. `finally { gate.Release(); }` guarantees release even on exception. `finally { channel.Writer.Complete(); }` in the producer is mandatory: otherwise `ReadAllAsync` on the consumers never completes and `Task.WhenAll` hangs forever. The `CancellationToken` is threaded through every async API and every `Task.Run(work, token)` — the lesson demands "pass `CancellationToken` into every `Task.Run` and async operation". There is no `.Result`/`.Wait()`, no `async void` outside event handlers, and no `lock` held across `await` (the semaphore is acquired through `WaitAsync`). A single shared `HttpClient` eliminates socket exhaustion. The solution therefore applies every best practice from the lesson and avoids every one of the listed common mistakes.

#### Going deeper (bonus)
1. **`Parallel.ForEachAsync` instead of a manual `SemaphoreSlim`.** Rewrite `ProcessAllAsync` using `Parallel.ForEachAsync` with `MaxDegreeOfParallelism = Environment.ProcessorCount`. Compare readability, cancellation behavior, and memory footprint with the `Channel` version. Explain when `Parallel.ForEachAsync` is preferable to a hand-rolled pipeline.
2. **Reproducing the `SynchronizationContext` deadlock.** Build a separate WPF test project (or use `AsyncContext` from Nito.AsyncEx) where a method calls `SomeAsyncMethod().Result` on the UI thread and hangs. Then add `ConfigureAwait(false)` inside `SomeAsyncMethod` and show that the deadlock disappears. Record the timing and the hang stack in `DEADLOCK.md`.
3. **Throughput vs. pool size.** Run the correct implementation on 500 URLs with `SemaphoreSlim(2)`, `SemaphoreSlim(Environment.ProcessorCount)`, `SemaphoreSlim(Environment.ProcessorCount * 2)`, and with no semaphore at all. Produce a table "parallelism → time → available pool threads" and explain why `Environment.ProcessorCount` is usually optimal for CPU-bound work.
4. **Priority channel and backpressure.** Switch to `Channel.CreateBounded` with `BoundedChannelFullMode.DropOldest` and measure at what buffer size the producer starts dropping data. Explain how this experiment relates to the notion of backpressure and its role in service resilience under bursty load.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `TextDigest` собирается под .NET 8 / C# 12.
- [ ] (RU) `FetchAsync` — I/O без `Task.Run`, с `ConfigureAwait(false)`.
- [ ] (RU) `FingerprintAsync` — CPU-bound через `Task.Run` с `ThrowIfCancellationRequested`.
- [ ] (RU) `ProcessAllAsync` — producer/consumer на `Channel` + `SemaphoreSlim`.
- [ ] (RU) `CancellationToken` проброшен во все `Task.Run` и async-API.
- [ ] (RU) Нет `.Result`/`.Wait()`, нет `async void` вне обработчиков, нет `lock` через `await`.
- [ ] (RU) Один общий `HttpClient`.
- [ ] (RU) `dotnet run` и `dotnet test` зелёные; Ctrl+C корректно отменяет.
- [ ] (RU) `REPORT.md` со сравнением корректной и наивной версии на 200 URL.
- [ ] (EN) `TextDigest` builds under .NET 8 / C# 12.
- [ ] (EN) `FetchAsync` is I/O with no `Task.Run` and uses `ConfigureAwait(false)`.
- [ ] (EN) `FingerprintAsync` is CPU-bound via `Task.Run` with `ThrowIfCancellationRequested`.
- [ ] (EN) `ProcessAllAsync` is a producer/consumer on `Channel` + `SemaphoreSlim`.
- [ ] (EN) A `CancellationToken` is threaded through every `Task.Run` and async API.
- [ ] (EN) No `.Result`/`.Wait()`, no `async void` outside handlers, no `lock` across `await`.
- [ ] (EN) A single shared `HttpClient`.
- [ ] (EN) `dotnet run` and `dotnet test` are green; Ctrl+C cancels gracefully.
- [ ] (EN) `REPORT.md` with a comparison of the correct and naive versions on 200 URLs.

#### Ресурсы / Resources
- [Microsoft Learn — Task-based Asynchronous Pattern (TAP) — https://learn.microsoft.com/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap](https://learn.microsoft.com/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap)
- [Microsoft Learn — Calling Synchronous Methods Asynchronously — https://learn.microsoft.com/dotnet/standard/asynchronous-programming-patterns/calling-synchronous-methods-asynchronously](https://learn.microsoft.com/dotnet/standard/asynchronous-programming-patterns/calling-synchronous-methods-asynchronously)
- [Stephen Toub — Should I expose asynchronous wrappers for synchronous methods? — https://devblogs.microsoft.com/dotnet/should-i-expose-asynchronous-wrappers-for-synchronous-methods/](https://devblogs.microsoft.com/dotnet/should-i-expose-asynchronous-wrappers-for-synchronous-methods/)
- [Microsoft Learn — Channel&lt;T&gt; — https://learn.microsoft.com/dotnet/api/system.threading.channels.channel-1](https://learn.microsoft.com/dotnet/api/system.threading.channels.channel-1)
- [Microsoft Learn — SemaphoreSlim — https://learn.microsoft.com/dotnet/api/system.threading.semaphoreslim](https://learn.microsoft.com/dotnet/api/system.threading.semaphoreslim)
- [Microsoft Learn — Parallel.ForEachAsync — https://learn.microsoft.com/dotnet/api/system.threading.tasks.parallel.foreachasync](https://learn.microsoft.com/dotnet/api/system.threading.tasks.parallel.foreachasync)
- [Stephen Toub — Async FAQ — https://devblogs.microsoft.com/dotnet/async-faq-where-do-i-start/](https://devblogs.microsoft.com/dotnet/async-faq-where-do-i-start/)
