---
[← К уроку M09-L07](lesson-M09-L07-configureawait-synccontext.md) | [⬆ К модулю M09](../README.md) | [Следующее ДЗ →](homework-M09-L08-valuetask.md)
---

### Домашнее задание M09-L07: ConfigureAwait(false), SynchronizationContext / Homework M09-L07: ConfigureAwait(false), SynchronizationContext

**Урок / Lesson:** M09-L07
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться осознанно применять `ConfigureAwait(false)` в библиотечном коде, понимать поведение `SynchronizationContext` на разных хостах (WPF/WinForms/MAUI vs ASP.NET Core vs консоль), избегать дедлоков sync-over-async, защищать разделяемое состояние между continuation из пула через `SemaphoreSlim`/`ConcurrentDictionary`/`Channel<T>`, и грамотно передавать `CancellationToken` по всему стеку вызовов. (EN) Learn to apply `ConfigureAwait(false)` deliberately in library code, understand `SynchronizationContext` behavior across hosts (WPF/WinForms/MAUI vs ASP.NET Core vs console), avoid sync-over-async deadlocks, protect shared state between pool continuations via `SemaphoreSlim`/`ConcurrentDictionary`/`Channel<T>`, and propagate `CancellationToken` correctly through the whole call stack.

#### Связь с уроком / Connection to the lesson
(RU — 2–3 предложения) Домашнее задание напрямую закрепляет все шесть блоков примера кода из урока M09-L07: библиотечный метод с `ConfigureAwait(false)`, UI-вызывающий без `ConfigureAwait(false)`, потокобезопасный кэш на `ConcurrentDictionary`, демонстрацию дедлока, безопасный sync-over-async через `Task.Run` и конвейер producer/consumer на `Channel<T>`. Особое внимание уделяется железным правилам урока: «не блокируй асинхронный код», «в библиотеках — `ConfigureAwait(false)`», «в UI-коде — оставь дефолт» и всегда передавай `CancellationToken`.
(EN — same) The homework directly reinforces all six code blocks from lesson M09-L07: the library method with `ConfigureAwait(false)`, the UI caller without it, the thread-safe `ConcurrentDictionary` cache, the deadlock demonstration, the safe sync-over-async via `Task.Run`, and the `Channel<T>` producer/consumer pipeline. Special attention is paid to the lesson's iron rules: "do not block async code", "in libraries use `ConfigureAwait(false)`", "in UI code keep the default", and always propagate a `CancellationToken`.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Представь, что ты — инженер небольшой команды, которая разрабатывает внутреннюю библиотеку `Course.AsyncFetcher` на C# 12 / .NET 8. Эта библиотека должна забирать JSON-документы по HTTP, кэшировать их, ограничивать параллелизм исходящих запросов и работать в трёх разных хостах: консольном приложении, ASP.NET Core Minimal API и (в будущем) десктоп-приложении на WPF. Команда уже однажды словила классический дедлок sync-over-async в WPF-плагине, потому что кто-то вызвал `.Result` на UI-потоке в методе, который внутри не имел `ConfigureAwait(false)`. Руководство требует, чтобы подобное больше не повторилось, и просит тебя реализовать библиотеку по правилам из урока M09-L07, а также подготовить демонстрационный проект, который наглядно показывает как дедлок, так и способы его избежать.

Ключевая проблема, которую решает задание, — это не просто «поставить `ConfigureAwait(false)` везде», а научиться рассуждать о том, **где** continuation продолжит выполнение после `await`, **кому** принадлежит разделяемое состояние, и **что** произойдёт, если вызывающий поток заблокируется. В консоли и в ASP.NET Core `SynchronizationContext.Current` равен `null`, поэтому continuation всегда выполняется в пуле потоков, и `ConfigureAwait(false)` становится почти no-op. В WPF/WinForms/MAUI контекст существует и сериализует continuation обратно в UI-поток — это и даёт безопасность обращения к контролам, но и порождает риск дедлока при блокирующем вызове. Библиотека не знает, кто её вызвал, и поэтому обязана вести себя одинаково корректно в любом окружении: не захватывать чужой контекст, не блокировать поток вызывающего изнутри себя и не оставлять гонок в общем состоянии.

#### Что нужно сделать (пошагово)

1. **Создай решение и проекты.** Выполни команды dotnet из корня `modules/M09/homework/M09-L07`:
   ```
   dotnet new sln -n Course.AsyncFetcher.Lab
   dotnet new classlib -n AsyncFetcher -o src/AsyncFetcher --framework net8.0
   dotnet new console   -n AsyncFetcher.Demo -o src/AsyncFetcher.Demo --framework net8.0
   dotnet new xunit     -n AsyncFetcher.Tests -o tests/AsyncFetcher.Tests --framework net8.0
   dotnet sln add src/AsyncFetcher src/AsyncFetcher.Demo tests/AsyncFetcher.Tests
   dotnet add src/AsyncFetcher.Demo reference src/AsyncFetcher
   dotnet add tests/AsyncFetcher.Tests reference src/AsyncFetcher
   ```
   В проекте `AsyncFetcher.Demo` включи поддержку nullable reference types и implicit usings (они включены по умолчанию в .NET 8). Убедись, что `dotnet build` проходит без предупреждений.

2. **Реализуй библиотечный класс `Fetcher` в `src/AsyncFetcher/Fetcher.cs`.** В нём должно быть:
   - асинхронный метод `FetchAsync(HttpClient http, Uri url, CancellationToken ct)`, который выполняет `GetAsync` с `HttpCompletionOption.ResponseHeadersRead`, читает поток через `ReadAsStreamAsync` и возвращает строку через `StreamReader.ReadToEndAsync`; **после каждого `await` — `ConfigureAwait(false)`**, как в примере урока;
   - асинхронный метод `FetchCachedAsync`, который проверяет `ConcurrentDictionary<Uri, string>`, при промахе вызывает `FetchAsync` и записывает результат в кэш; продумай, что происходит при параллельном промахе по одному ключу (допускается стратегия «last writer wins», но состояние словаря должно оставаться консистентным);
   - метод `FetchWithConcurrencyLimitAsync`, который ограничивает число одновременно выполняемых запросов через `SemaphoreSlim(8, 8)` и освобождает слот в блоке `finally` — обязательно через `await sem.WaitAsync(ct).ConfigureAwait(false)`, **не** через `lock`;
   - метод `FetchBlockingSafe`, реализующий безопасный sync-over-async через `Task.Run(async () => await FetchAsync(...)).GetAwaiter().GetResult()` для случаев, когда синхронный интерфейс действительно неизбежен (например, legacy-контракт).

3. **Реализуй конвейер producer/consumer в `src/AsyncFetcher/Pipeline.cs`.** Метод `RunPipelineAsync` принимает источник `IEnumerable<int>`, функцию-трансформер `Func<int, CancellationToken, Task<string>>`, потребитель `Action<string>` и `CancellationToken`. Используй `Channel.CreateBounded<string>` с `BoundedChannelFullMode.Wait`, чтобы получить обратное давление. Producer должен вызывать `ct.ThrowIfCancellationRequested()`, `transform` с `ConfigureAwait(false)` и `channel.Writer.WriteAsync` с `ConfigureAwait(false)`, а в `finally` — `channel.Writer.Complete()`. Consumer — один читатель, перебирает `ReadAllAsync(ct)` с `ConfigureAwait(false)` и вызывает `consume(item)` без `lock`. Завершается метод `Task.WhenAll(producer, consumer)` с `ConfigureAwait(false)`.

4. **В `src/AsyncFetcher.Demo/Program.cs`** напиши top-level statements, которые: (а) создают `HttpClient` с фейковым `HttpMessageHandler`, возвращающим предсказуемый текст; (б) показывают работу `FetchCachedAsync` и `FetchWithConcurrencyLimitAsync` на 20 URL; (в) запускают `RunPipelineAsync` на 50 элементов и печатают результаты; (г) комментируют, **почему** в demo (консоль) `ConfigureAwait(false)` фактически ничего не меняет, но остаётся полезным как защита на случай переезда в WPF.

5. **Добавь юнит-тесты.** В `tests/AsyncFetcher.Tests/FetcherTests.cs` проверь: кэш возвращает то же значение на повторный запрос; `CancellationToken` действительно отменяет долгую операцию; `SemaphoreSlim` не даёт выполнять больше 8 запросов одновременно (используй счётчик активных запросов в фейковом handler); `RunPipelineAsync` обрабатывает ровно столько элементов, сколько подано на вход. Используй `Microsoft.AspNetCore.WebUtilities` или свой `FakeHandler : HttpMessageHandler` с задержкой `Task.Delay`.

6. **Запусти и собери отчёт.** Выполни `dotnet test`, `dotnet run --project src/AsyncFetcher.Demo`, приложи вывод консоли. В выводе должно быть видно: повторные запросы уходят в кэш (быстрее), параллелизм ограничен 8 потоками, конвейер обрабатывает все элементы и корректно завершается.

7. **Опиши в `REPORT.md`** три сценария: где именно возникает дедлок sync-over-async; почему `Task.Run` без захвата контекста спасает; почему в ASP.NET Core все эти ухищрения с `ConfigureAwait(false)` почти не дают выигрыша, но остаются хорошей привычкой в библиотеках.

#### Требования к решению

- Целевой фреймворк — строго `net8.0`; язык C# 12; включён `Nullable`; используются top-level statements в демо и современные возможности (collection expressions, pattern matching, `await using`, raw string literals там, где это уместно для шаблонов).
- В библиотечном коде **после каждого `await`** стоит `.ConfigureAwait(false)`. Это касается HTTP-вызовов, чтения потока, `sem.WaitAsync`, `channel.Writer.WriteAsync`, `ReadAllAsync`, `Task.WhenAll`, `Task.Delay` и `Task.Run`-обёрток.
- В демо-консольном коде `ConfigureAwait(false)` может отсутствовать (контекста всё равно нет), но это должно быть **осознанным** выбором и прокомментировано. В UI-методах, трогающих контролы, `ConfigureAwait(false)` был бы вреден — но UI в этом задании не реализуется, его поведение только описывается в `REPORT.md` со ссылкой на `InvalidOperationException`("The calling thread cannot access this object").
- `CancellationToken` проходит через **каждую** публичную и приватную асинхронную сигнатуру в библиотеке; нигде не передаётся `CancellationToken.None` «для простоты» (кроме демонстрационного антипаттерна, который явно помечен).
- Разделяемое состояние (`ConcurrentDictionary`, счётчик активных запросов) защищено корректно: либо самим типом (`ConcurrentDictionary`), либо `Interlocked`, либо `SemaphoreSlim`. **Запрещён** `lock (obj) { await ... }`.
- `Channel<T>` использует `BoundedChannelOptions` с `SingleReader = true` и обязательно вызывает `Writer.Complete()` в `finally` producer'а, чтобы consumer не завис в `ReadAllAsync` навсегда.
- `async void` не используется нигде; если бы понадобился обработчик события, он должен быть обёрнут в `try/catch` — упомяни это в `REPORT.md`.
- Сборка проходит с `-warnaserror` по крайней мере для `CA2007` (Configure await) — настрой `.editorconfig` или `AnalysisLevel`, чтобы это правило было включено в библиотечном проекте.

#### Тонкости и подводные камни

- **Дедлок sync-over-async** возникает только на хосте **с** `SynchronizationContext` (WPF/WinForms/MAUI/ASP.NET-classic). В консоли и ASP.NET Core `GetAwaiter().GetResult()` на `await`-методе без `ConfigureAwait(false)` не зависнет — но это не повод писать так: при переезде в WPF тот же код зависнет. Поэтому «безопасно в моих тестах» ≠ «безопасно в принципе».
- **`ConfigureAwait(false)` не защищает от дедлока сам по себе**, если вызывающий код держит блокировку, которая нужна continuation. Но он снимает зависимость от UI-потока, что в большинстве случаев достаточно. В `FetchBlockingSafe` из урока `Task.Run` форсирует запуск в пуле **до** захвата контекста, поэтому даже legacy-библиотека без `ConfigureAwait(false)` не зависнет — continuation пойдёт в пул.
- **`lock (obj) { await ... }` компилируется, но ломается**: `Monitor` не реентерабелен, continuation может выполниться на другом потоке и не сможет войти в тот же `lock`. Правильный асинхронный мьютекс — `SemaphoreSlim(1, 1)` с `await sem.WaitAsync(ct)`.
- **`ConcurrentDictionary` защищает целостность словаря, но не атомарность «проверь-вычисли-запиши»**: два промаха по одному ключу дадут два сетевых запроса и две записи. Если это дорого, используй `GetOrAddAsync` через `ConcurrentDictionary<Uri, Task<string>>` или `Lazy<Task<string>>`, или `SemaphoreSlim` на ключ. В задании допускается «last writer wins», но ты должен это явно осознать и описать.
- **`Channel<T>` с `SingleReader = true`** даёт более быстрый путь и снимает необходимость в `lock` у consumer'а, но только пока инвариант действительно соблюдается. Нарушишь — получишь трудноуловимую гонку.
- **`CancellationToken.None` маскирует проблемы**: операцию нельзя отменить, тест на таймаут невозможно написать. Везде, кроме намеренно сломанного демо, передавай реальный токен.
- **`HttpCompletionOption.ResponseHeadersRead`** важен для больших ответов: ты начинаешь читать поток, не дожидаясь всего тела в памяти. Сочетай с `ReadAsStreamAsync(ct)` и `await using`.
- **`async void`**: исключения вылетают мимо стека и роняют процесс; отмена не корректно прокидывается. Для обработчиков событий UI — единственное допустимое применение, всегда в `try/catch`.

#### Критерии приёмки

- [ ] Решение из трёх проектов собирается `dotnet build` без ошибок и без предупреждений уровня `CA2007` в библиотеке.
- [ ] В каждом `await` внутри `src/AsyncFetcher` присутствует `.ConfigureAwait(false)` (проверяется ревью и тест-хуком на исходниках).
- [ ] `FetchAsync` использует `HttpCompletionOption.ResponseHeadersRead` и `await using` для потока.
- [ ] `FetchCachedAsync` корректно работает при параллельных промахах: состояние `ConcurrentDictionary` остаётся консистентным, поведение «last writer wins» описано в `REPORT.md`.
- [ ] `FetchWithConcurrencyLimitAsync` использует `SemaphoreSlim(8, 8)` и освобождает слот в `finally`; тест доказывает, что одновременно активных запросов не больше 8.
- [ ] `FetchBlockingSafe` реализован через `Task.Run(...).GetAwaiter().GetResult()` и в `REPORT.md` объяснено, почему это безопаснее прямого `GetAwaiter().GetResult()`.
- [ ] `RunPipelineAsync` использует `Channel.CreateBounded` с `BoundedChannelFullMode.Wait`, `SingleReader = true`, вызывает `Writer.Complete()` в `finally`.
- [ ] `CancellationToken` передаётся в каждую асинхронную операцию; тест отменяет операцию и проверяет `OperationCanceledException`.
- [ ] Нигде нет `lock (obj) { await ... }`; `async void` отсутствует; `.Result`/`.Wait()` встречаются только в явно помеченном демонстрационном антипаттерне.
- [ ] `dotnet test` — зелёный; `dotnet run` выводит предсказуемый отчёт о кэше, параллелизме и конвейере.
- [ ] `REPORT.md` содержит разбор трёх сценариев (дедлок, `Task.Run`, ASP.NET Core) со ссылками на строки кода.
- [ ] Код использует возможности C# 12 (top-level statements, pattern matching, collection expressions, `await using`, при необходимости raw string literals).
- [ ] Все публичные API документированы XML-комментариями с указанием, что метод безопасен для вызова из любого хоста.
- [ ] `.editorconfig` включает `CA2007` с `severity = error` для проекта `AsyncFetcher`.
- [ ] Демонстрируется понимание, что в консоли и ASP.NET Core `SynchronizationContext.Current == null`, и это объясняется в `REPORT.md`.

#### Подсказки (без прямого ответа)

- Подумай, в каком порядке `ConfigureAwait(false)` и `HttpCompletionOption` должны стоять в цепочке вызовов; помни про приоритет скобок и приоритет `await`.
- Для фейкового handler'а наследуй `HttpMessageHandler` и переопредели `SendAsync`; внутри можно `await Task.Delay(ms, cancellationToken)`, чтобы симулировать сеть и проверить отмену.
- Чтобы подсчитать «активные запросы» для теста лимита, в `FakeHandler` держи `int _active` и обновляй через `Interlocked.Increment`/`Decrement`, а в тесте опрашивай максимум.
- Для «last writer wins» достаточно индексатора `_cache[url] = body`; для устранения двойных запросов — `ConcurrentDictionary<Uri, Lazy<Task<string>>>` и `GetOrAdd`.
- В `RunPipelineAsync` не забудь, что `Writer.Complete()` в `finally` критичен: иначе `await foreach` у consumer'а никогда не завершится.
- Если тесты на отмену «висят», проверь, что ты передаёшь `ct` до самого底层 HTTP-вызова, а не только в `Task.WhenAll`.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — Эталон библиотеки AsyncFetcher.
// Все await в библиотеке несут ConfigureAwait(false); CancellationToken проходит до HTTP.
// Library awaits carry ConfigureAwait(false); CancellationToken reaches the HTTP layer.

using System.Collections.Concurrent;
using System.Net.Http.Headers;
using System.Runtime.CompilerServices;

namespace AsyncFetcher;

// Потокобезопасный кэш ответов. Thread-safe response cache.
// ConcurrentDictionary гарантирует консистентность словаря, но не атомарность check-compute-store.
public sealed class Fetcher
{
    private readonly ConcurrentDictionary<Uri, string> _cache = new();

    // 1. Базовый асинхронный запрос. Basic async fetch.
    //    ConfigureAwait(false) на каждом await — библиотека не знает хоста вызывающего.
    public async Task<string> FetchAsync(HttpClient http, Uri url, CancellationToken ct)
    {
        // ResponseHeadersRead — не грузим тело целиком в память до чтения потока.
        using var resp = await http.GetAsync(url, HttpCompletionOption.ResponseHeadersRead, ct)
            .ConfigureAwait(false);
        resp.EnsureSuccessStatusCode();

        // await using освобождает поток даже при исключении.
        await using var stream = await resp.Content.ReadAsStreamAsync(ct).ConfigureAwait(false);
        using var sr = new StreamReader(stream);
        return await sr.ReadToEndAsync(ct).ConfigureAwait(false);
    }

    // 2. Кэш: last writer wins. Cache: last writer wins.
    //    Состояние словаря консистентно; двойной промах допустим и описан в REPORT.md.
    public async Task<string> FetchCachedAsync(HttpClient http, Uri url, CancellationToken ct)
    {
        if (_cache.TryGetValue(url, out var hit))
            return hit; // быстрый путь, без аллокаций / fast path, no allocation.

        string body = await FetchAsync(http, url, ct).ConfigureAwait(false);
        _cache[url] = body; // гонка безопасна на уровне словаря / race-safe at dictionary level.
        return body;
    }

    // 3. Ограничение параллелизма. Concurrency throttling.
    //    SemaphoreSlim(1,1) — асинхронный мьютекс; НИКОГДА lock вокруг await.
    private readonly SemaphoreSlim _gate = new(8, 8);

    public async Task<string> FetchWithConcurrencyLimitAsync(HttpClient http, Uri url, CancellationToken ct)
    {
        await _gate.WaitAsync(ct).ConfigureAwait(false); // асинхронно ждём слот / wait for slot async.
        try
        {
            return await FetchCachedAsync(http, url, ct).ConfigureAwait(false);
        }
        finally
        {
            _gate.Release(); // обязательно в finally, иначе утечка слота / release in finally.
        }
    }

    // 4. Безопасный sync-over-async, когда синхронный контракт неизбежен.
    //    Task.Run форсирует запуск continuation в пуле ДО захвата контекста,
    //    поэтому даже legacy-библиотека без ConfigureAwait(false) не дедлочит UI.
    public string FetchBlockingSafe(HttpClient http, Uri url) =>
        Task.Run(async () => await FetchAsync(http, url, CancellationToken.None)
            .ConfigureAwait(false)).GetAwaiter().GetResult();
}
```

```csharp
// Pipeline.cs — конвейер producer/consumer с обратным давлением.
using System.Threading.Channels;

namespace AsyncFetcher;

public static class Pipeline
{
    public static async Task RunPipelineAsync(
        IEnumerable<int> source,
        Func<int, CancellationToken, Task<string>> transform,
        Action<string> consume,
        CancellationToken ct)
    {
        // Bounded channel = естественное обратное давление: быстрый producer ждёт, когда consumer освободит слот.
        var channel = Channel.CreateBounded<string>(new BoundedChannelOptions(64)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = true,   // даёт быстрый путь и снимает необходимость в lock у consumer'а.
            SingleWriter = false
        });

        var produce = Task.Run(async () =>
        {
            try
            {
                foreach (var id in source)
                {
                    ct.ThrowIfCancellationRequested();
                    string item = await transform(id, ct).ConfigureAwait(false);
                    await channel.Writer.WriteAsync(item, ct).ConfigureAwait(false);
                }
            }
            finally
            {
                await channel.Writer.Completion.ConfigureAwait(false); // гарантирует, но Complete обязателен.
                channel.Writer.Complete(); // критично: иначе consumer висит в ReadAllAsync.
            }
        }, ct);

        var consumeTask = Task.Run(async () =>
        {
            await foreach (var item in channel.Reader.ReadAllAsync(ct).ConfigureAwait(false))
                consume(item); // single-reader → lock не нужен.
        }, ct);

        await Task.WhenAll(produce, consumeTask).ConfigureAwait(false);
    }
}
```

Разбор по строкам. В `FetchAsync` первый `await` висит на `GetAsync` с `ResponseHeadersRead` — это лучшая практика для больших ответов: мы получаем заголовки и поток, не буферизуя тело. `ConfigureAwait(false)` снимает привязку к контексту вызывающего, что особенно важно в WPF. `await using var stream` гарантирует освобождение неуправляемого ресурса даже при исключении в `ReadToEndAsync`. В `FetchCachedAsync` я сознательно выбираю «last writer wins»: `ConcurrentDictionary` обеспечивает потокобезопасность структуры данных, но не атомарность шаблона «проверь-вычисли-запиши» — два параллельных промаха по одному ключу дадут два сетевых запроса. Это компромисс между сложностью (`Lazy<Task<string>>` + `GetOrAdd`) и стоимостью редкого дублирования; в `REPORT.md` этот выбор аргументируется. В `FetchWithConcurrencyLimitAsync` используется `SemaphoreSlim(8, 8)` как асинхронный мьютекс: `await _gate.WaitAsync(ct)` не блокирует поток, в отличие от `lock`, и `Release()` в `finally` гарантирует возврат слота при исключении. Прямой `lock (obj) { await ... }` здесь был бы ошибкой — `Monitor` не реентерабелен, continuation на другом потоке не сможет войти. `FetchBlockingSafe` повторяет приём из урока: `Task.Run` запускает continuation в пуле **до** того, как будет захвачен контекст вызывающего, поэтому даже если вызывающий поток блокируется на `GetResult`, continuation не ждёт его — дедлок невозможен. В `RunPipelineAsync` ограниченный канал с `FullMode = Wait` реализует обратное давление: быстрый producer не затопит медленный consumer, а вместо явных очередей и `lock` мы получаем типобезопасный примитив. `SingleReader = true` позволяет рантайму выбрать оптимизированный путь и снимает необходимость синхронизации в `consume`. `Writer.Complete()` в `finally` — критическая деталь из урока: без него `await foreach` у consumer'а никогда не завершится, потому что канал останется «открытым». Применённые концепции урока: библиотечное правило `ConfigureAwait(false)`, запрет `lock` вокруг `await`, `CancellationToken` всюду, `Channel<T>` как идиоматичный concurrency с обратным давлением, безопасный sync-over-async через `Task.Run`, и явное осознание, что в консоли/ASP.NET Core контекста нет, но привычка сохраняется ради переносимости.

#### Задания на углубление (бонус)

1. **Устрани двойные запросы в кэше.** Переведи `FetchCachedAsync` на `ConcurrentDictionary<Uri, Lazy<Task<string>>>` с `GetOrAdd`, чтобы при параллельном промахе по одному ключу выполнялся ровно один сетевой запрос. Напиши тест, который это доказывает (счётчик вызовов фейкового handler'а).
2. **WPF-демо дедлока.** Создай мини-проект WPF с кнопкой, обработчик `Click` которой вызывает `.Result` на `FetchAsync` без `ConfigureAwait(false)` внутри. Покажи дедлок, затем исправь двумя способами: (а) `async void` обработчик с `await`, (б) `FetchBlockingSafe`. Зафиксируй в `REPORT.md` разницу.
3. **Blazor WASM сравнение.** В Blazor WASM есть свой `SynchronizationContext`. Напиши заметку: когда там `ConfigureAwait(false)` в прикладном коде вредит, а когда — нет.
4. **Бенчмарк `ConfigureAwait(false)` в ASP.NET Core.** С помощью `BenchmarkDotNet` замерь разницу (или её отсутствие) между версиями метода с `ConfigureAwait(false)` и без на ASP.NET Core-хосте. Объясни результат через `SynchronizationContext.Current == null`.

---

## Statement in English / Постановка на английском

#### Context & motivation

Imagine you are an engineer in a small team developing an internal library `Course.AsyncFetcher` on C# 12 / .NET 8. The library must fetch JSON documents over HTTP, cache them, throttle outgoing concurrency, and run inside three different hosts: a console app, an ASP.NET Core Minimal API, and (in the future) a WPF desktop application. The team has already hit a classic sync-over-async deadlock in a WPF plugin, because somebody called `.Result` on the UI thread inside a method that did not use `ConfigureAwait(false)`. Management demands this never happens again and asks you to implement the library following the rules from lesson M09-L07, and to ship a demo project that visibly demonstrates both the deadlock and the ways to avoid it.

The core problem the assignment solves is not simply "sprinkle `ConfigureAwait(false)` everywhere", but to learn to reason about **where** a continuation will resume after `await`, **who** owns shared state, and **what** happens if the calling thread blocks. In a console app and in ASP.NET Core, `SynchronizationContext.Current` is `null`, so continuations always run on the thread pool and `ConfigureAwait(false)` is almost a no-op. In WPF/WinForms/MAUI the context exists and serializes continuations back onto the UI thread — this is what makes control access safe, and it is also what creates the deadlock risk when a blocking call is made. A library does not know who called it, so it must behave identically correctly in any environment: it must not capture the caller's context, must not block the caller's thread from inside itself, and must not leave races in shared state.

#### What to do step by step

1. **Create the solution and projects.** Run these dotnet commands from `modules/M09/homework/M09-L07`:
   ```
   dotnet new sln -n Course.AsyncFetcher.Lab
   dotnet new classlib -n AsyncFetcher -o src/AsyncFetcher --framework net8.0
   dotnet new console   -n AsyncFetcher.Demo -o src/AsyncFetcher.Demo --framework net8.0
   dotnet new xunit     -n AsyncFetcher.Tests -o tests/AsyncFetcher.Tests --framework net8.0
   dotnet sln add src/AsyncFetcher src/AsyncFetcher.Demo tests/AsyncFetcher.Tests
   dotnet add src/AsyncFetcher.Demo reference src/AsyncFetcher
   dotnet add tests/AsyncFetcher.Tests reference src/AsyncFetcher
   ```
   Ensure nullable reference types and implicit usings are on (default in .NET 8). `dotnet build` must succeed without warnings.

2. **Implement the library class `Fetcher` in `src/AsyncFetcher/Fetcher.cs`.** It must contain:
   - an async method `FetchAsync(HttpClient http, Uri url, CancellationToken ct)` that calls `GetAsync` with `HttpCompletionOption.ResponseHeadersRead`, reads the stream via `ReadAsStreamAsync`, and returns a string via `StreamReader.ReadToEndAsync`; **`ConfigureAwait(false)` after every `await`**, mirroring the lesson's example;
   - an async method `FetchCachedAsync` that checks a `ConcurrentDictionary<Uri, string>`, on miss calls `FetchAsync`, and writes the result to the cache; think about what happens on a parallel miss for the same key (a "last writer wins" strategy is acceptable, but the dictionary state must remain consistent);
   - a method `FetchWithConcurrencyLimitAsync` that throttles concurrent requests with `SemaphoreSlim(8, 8)` and releases the slot in a `finally` block — via `await sem.WaitAsync(ct).ConfigureAwait(false)`, **not** via `lock`;
   - a method `FetchBlockingSafe` that implements safe sync-over-async via `Task.Run(async () => await FetchAsync(...)).GetAwaiter().GetResult()` for cases where a synchronous interface is genuinely unavoidable (e.g., a legacy contract).

3. **Implement the producer/consumer pipeline in `src/AsyncFetcher/Pipeline.cs`.** The method `RunPipelineAsync` takes a source `IEnumerable<int>`, a transformer `Func<int, CancellationToken, Task<string>>`, a consumer `Action<string>`, and a `CancellationToken`. Use `Channel.CreateBounded<string>` with `BoundedChannelFullMode.Wait` to get backpressure. The producer must call `ct.ThrowIfCancellationRequested()`, `transform` with `ConfigureAwait(false)`, and `channel.Writer.WriteAsync` with `ConfigureAwait(false)`, and call `channel.Writer.Complete()` in a `finally`. The consumer is a single reader, iterates `ReadAllAsync(ct)` with `ConfigureAwait(false)`, and invokes `consume(item)` without a `lock`. The method finishes with `Task.WhenAll(producer, consumer)` carrying `ConfigureAwait(false)`.

4. **In `src/AsyncFetcher.Demo/Program.cs`** write top-level statements that: (a) create an `HttpClient` with a fake `HttpMessageHandler` returning predictable text; (b) demonstrate `FetchCachedAsync` and `FetchWithConcurrencyLimitAsync` across 20 URLs; (c) run `RunPipelineAsync` on 50 items and print the results; (d) comment **why** in the demo (console) `ConfigureAwait(false)` is effectively a no-op, but stays useful as protection if the code ever moves to WPF.

5. **Add unit tests.** In `tests/AsyncFetcher.Tests/FetcherTests.cs` verify: the cache returns the same value on a repeated request; a `CancellationToken` actually cancels a long operation; `SemaphoreSlim` prevents more than 8 concurrent requests (use an active-counter inside the fake handler); `RunPipelineAsync` processes exactly the number of items submitted. Use `Microsoft.AspNetCore.WebUtilities` or your own `FakeHandler : HttpMessageHandler` with a `Task.Delay`.

6. **Run and collect a report.** Run `dotnet test`, `dotnet run --project src/AsyncFetcher.Demo`, attach the console output. The output must show: repeated requests hit the cache (faster), concurrency is capped at 8, the pipeline processes all items and terminates cleanly.

7. **Describe in `REPORT.md`** three scenarios: where exactly the sync-over-async deadlock arises; why `Task.Run` without a captured context saves the day; why in ASP.NET Core all these `ConfigureAwait(false)` gymnastics barely help, but remain a good habit in libraries.

#### Requirements

- Target framework is strictly `net8.0`; language C# 12; `Nullable` is enabled; the demo uses top-level statements and modern features (collection expressions, pattern matching, `await using`, raw string literals where they fit templates).
- In library code, **after every `await`** there is `.ConfigureAwait(false)`. This covers HTTP calls, stream reading, `sem.WaitAsync`, `channel.Writer.WriteAsync`, `ReadAllAsync`, `Task.WhenAll`, `Task.Delay`, and `Task.Run` wrappers.
- In the demo console code `ConfigureAwait(false)` may be omitted (there is no context anyway), but this must be a **deliberate** choice and commented as such. In UI methods that touch controls, `ConfigureAwait(false)` would be harmful — but UI is not implemented in this assignment; its behavior is only described in `REPORT.md` with a reference to `InvalidOperationException` ("The calling thread cannot access this object").
- `CancellationToken` flows through **every** public and private async signature in the library; nowhere is `CancellationToken.None` passed "for simplicity" (except the explicitly marked demo anti-pattern).
- Shared state (`ConcurrentDictionary`, the active-request counter) is protected correctly: either by the type itself (`ConcurrentDictionary`), by `Interlocked`, or by `SemaphoreSlim`. **Forbidden:** `lock (obj) { await ... }`.
- `Channel<T>` uses `BoundedChannelOptions` with `SingleReader = true` and always calls `Writer.Complete()` in the producer's `finally`, so the consumer never hangs in `ReadAllAsync`.
- `async void` is not used anywhere; if an event handler were needed it would be wrapped in `try/catch` — mention this in `REPORT.md`.
- The build passes with `-warnaserror` at least for `CA2007` (configure await) — set up `.editorconfig` or `AnalysisLevel` so the rule is enabled in the library project.

#### Pitfalls

- **Sync-over-async deadlock** only happens on a host **with** a `SynchronizationContext` (WPF/WinForms/MAUI/ASP.NET-classic). In console and ASP.NET Core, `GetAwaiter().GetResult()` on an `await`-method without `ConfigureAwait(false)` will not hang — but that is no excuse to write it that way: when the same code moves to WPF it will hang. "Safe in my tests" ≠ "safe in principle".
- **`ConfigureAwait(false)` does not, by itself, prevent a deadlock** if the caller holds a lock the continuation needs. But it removes the dependency on the UI thread, which is enough in most cases. In the lesson's `FetchBlockingSafe`, `Task.Run` forces execution on the pool **before** the context is captured, so even a legacy library without `ConfigureAwait(false)` will not hang — the continuation goes to the pool.
- **`lock (obj) { await ... }` compiles but breaks**: `Monitor` is non-reentrant, a continuation on a different thread cannot re-enter, producing a deadlock or exception. The correct async mutex is `SemaphoreSlim(1, 1)` with `await sem.WaitAsync(ct)`.
- **`ConcurrentDictionary` protects dictionary integrity, not "check-compute-store" atomicity**: two misses on the same key yield two network requests and two writes. If that is expensive, use `GetOrAddAsync` via `ConcurrentDictionary<Uri, Task<string>>` or `Lazy<Task<string>>`, or a per-key `SemaphoreSlim`. The assignment allows "last writer wins", but you must consciously accept and describe it.
- **`Channel<T>` with `SingleReader = true`** gives a faster path and removes the need for a consumer `lock`, but only while the invariant truly holds. Break it and you get a subtle race.
- **`CancellationToken.None` hides problems**: the operation cannot be cancelled, a timeout test cannot be written. Everywhere except the intentionally broken demo, pass a real token.
- **`HttpCompletionOption.ResponseHeadersRead`** matters for large responses: you start streaming without buffering the whole body in memory. Combine it with `ReadAsStreamAsync(ct)` and `await using`.
- **`async void`**: exceptions fly off the call stack and crash the process; cancellation does not propagate correctly. For UI event handlers it is the only acceptable use, always in `try/catch`.

#### Acceptance criteria

- [ ] The three-project solution builds with `dotnet build` with no errors and no `CA2007` warnings in the library.
- [ ] Every `await` inside `src/AsyncFetcher` carries `.ConfigureAwait(false)` (verified by review and a source-level test hook).
- [ ] `FetchAsync` uses `HttpCompletionOption.ResponseHeadersRead` and `await using` for the stream.
- [ ] `FetchCachedAsync` behaves correctly under parallel misses: the `ConcurrentDictionary` state stays consistent, and the "last writer wins" policy is documented in `REPORT.md`.
- [ ] `FetchWithConcurrencyLimitAsync` uses `SemaphoreSlim(8, 8)` and releases the slot in `finally`; a test proves no more than 8 concurrent requests.
- [ ] `FetchBlockingSafe` is implemented via `Task.Run(...).GetAwaiter().GetResult()` and `REPORT.md` explains why this is safer than a direct `GetAwaiter().GetResult()`.
- [ ] `RunPipelineAsync` uses `Channel.CreateBounded` with `BoundedChannelFullMode.Wait`, `SingleReader = true`, and calls `Writer.Complete()` in `finally`.
- [ ] `CancellationToken` is passed to every async operation; a test cancels and checks `OperationCanceledException`.
- [ ] No `lock (obj) { await ... }` anywhere; no `async void`; `.Result`/`.Wait()` appear only in the explicitly marked demo anti-pattern.
- [ ] `dotnet test` is green; `dotnet run` prints a predictable report about cache, concurrency, and pipeline.
- [ ] `REPORT.md` covers the three scenarios (deadlock, `Task.Run`, ASP.NET Core) with references to code lines.
- [ ] Code uses C# 12 features (top-level statements, pattern matching, collection expressions, `await using`, raw string literals where useful).
- [ ] All public APIs are documented with XML comments stating the method is safe to call from any host.
- [ ] `.editorconfig` enables `CA2007` with `severity = error` for the `AsyncFetcher` project.
- [ ] The understanding that in console and ASP.NET Core `SynchronizationContext.Current == null` is demonstrated and explained in `REPORT.md`.

#### Hints (no direct answer)

- Think about the order `ConfigureAwait(false)` and `HttpCompletionOption` should appear in the call chain; mind operator precedence and the precedence of `await`.
- For the fake handler, derive from `HttpMessageHandler` and override `SendAsync`; inside you can `await Task.Delay(ms, cancellationToken)` to simulate network and test cancellation.
- To count "active requests" for the limit test, keep `int _active` in `FakeHandler` and update it with `Interlocked.Increment`/`Decrement`; in the test, poll the maximum.
- For "last writer wins" the indexer `_cache[url] = body` suffices; to eliminate duplicate requests, use `ConcurrentDictionary<Uri, Lazy<Task<string>>>` and `GetOrAdd`.
- In `RunPipelineAsync`, remember that `Writer.Complete()` in `finally` is critical: otherwise the consumer's `await foreach` never finishes.
- If cancellation tests "hang", check that you pass `ct` down to the底层 HTTP call, not only to `Task.WhenAll`.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — Reference AsyncFetcher library.
// Every library await carries ConfigureAwait(false); CancellationToken reaches the HTTP layer.

using System.Collections.Concurrent;
using System.Runtime.CompilerServices;

namespace AsyncFetcher;

// Thread-safe response cache. ConcurrentDictionary guarantees dictionary consistency,
// but not the atomicity of check-compute-store.
public sealed class Fetcher
{
    private readonly ConcurrentDictionary<Uri, string> _cache = new();

    // 1. Basic async fetch. ConfigureAwait(false) on every await: the library does not know its host.
    public async Task<string> FetchAsync(HttpClient http, Uri url, CancellationToken ct)
    {
        // ResponseHeadersRead — do not buffer the whole body in memory before streaming.
        using var resp = await http.GetAsync(url, HttpCompletionOption.ResponseHeadersRead, ct)
            .ConfigureAwait(false);
        resp.EnsureSuccessStatusCode();

        // await using releases the stream even on exception.
        await using var stream = await resp.Content.ReadAsStreamAsync(ct).ConfigureAwait(false);
        using var sr = new StreamReader(stream);
        return await sr.ReadToEndAsync(ct).ConfigureAwait(false);
    }

    // 2. Cache: last writer wins. Dictionary state is consistent; a double miss is acceptable and documented.
    public async Task<string> FetchCachedAsync(HttpClient http, Uri url, CancellationToken ct)
    {
        if (_cache.TryGetValue(url, out var hit))
            return hit; // fast path, no allocation.

        string body = await FetchAsync(http, url, ct).ConfigureAwait(false);
        _cache[url] = body; // race-safe at dictionary level.
        return body;
    }

    // 3. Concurrency throttling. SemaphoreSlim(1,1) is an async mutex; NEVER use lock around await.
    private readonly SemaphoreSlim _gate = new(8, 8);

    public async Task<string> FetchWithConcurrencyLimitAsync(HttpClient http, Uri url, CancellationToken ct)
    {
        await _gate.WaitAsync(ct).ConfigureAwait(false); // wait for a slot asynchronously.
        try
        {
            return await FetchCachedAsync(http, url, ct).ConfigureAwait(false);
        }
        finally
        {
            _gate.Release(); // always in finally, otherwise the slot leaks.
        }
    }

    // 4. Safe sync-over-async when a synchronous contract is unavoidable.
    //    Task.Run forces the continuation onto the pool BEFORE the context is captured,
    //    so even a legacy library without ConfigureAwait(false) will not deadlock the UI.
    public string FetchBlockingSafe(HttpClient http, Uri url) =>
        Task.Run(async () => await FetchAsync(http, url, CancellationToken.None)
            .ConfigureAwait(false)).GetAwaiter().GetResult();
}
```

```csharp
// Pipeline.cs — producer/consumer pipeline with backpressure.
using System.Threading.Channels;

namespace AsyncFetcher;

public static class Pipeline
{
    public static async Task RunPipelineAsync(
        IEnumerable<int> source,
        Func<int, CancellationToken, Task<string>> transform,
        Action<string> consume,
        CancellationToken ct)
    {
        // Bounded channel = natural backpressure: a fast producer waits for the consumer to free a slot.
        var channel = Channel.CreateBounded<string>(new BoundedChannelOptions(64)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = true,   // faster path, removes the need for a consumer lock.
            SingleWriter = false
        });

        var produce = Task.Run(async () =>
        {
            try
            {
                foreach (var id in source)
                {
                    ct.ThrowIfCancellationRequested();
                    string item = await transform(id, ct).ConfigureAwait(false);
                    await channel.Writer.WriteAsync(item, ct).ConfigureAwait(false);
                }
            }
            finally
            {
                channel.Writer.Complete(); // critical: otherwise the consumer hangs in ReadAllAsync.
            }
        }, ct);

        var consumeTask = Task.Run(async () =>
        {
            await foreach (var item in channel.Reader.ReadAllAsync(ct).ConfigureAwait(false))
                consume(item); // single-reader → no lock needed.
        }, ct);

        await Task.WhenAll(produce, consumeTask).ConfigureAwait(false);
    }
}
```

Line-by-line walk-through. In `FetchAsync`, the first `await` hangs on `GetAsync` with `ResponseHeadersRead` — a best practice for large responses: we get headers and a stream without buffering the body. `ConfigureAwait(false)` removes the dependency on the caller's context, which matters especially in WPF. `await using var stream` guarantees disposal of the unmanaged resource even if `ReadToEndAsync` throws. In `FetchCachedAsync` I deliberately choose "last writer wins": `ConcurrentDictionary` makes the data structure thread-safe, but not the "check-compute-store" pattern atomic — two parallel misses on the same key yield two network requests. This is a trade-off between complexity (`Lazy<Task<string>>` + `GetOrAdd`) and the cost of rare duplication; `REPORT.md` justifies the choice. In `FetchWithConcurrencyLimitAsync`, `SemaphoreSlim(8, 8)` is the async mutex: `await _gate.WaitAsync(ct)` does not block a thread, unlike `lock`, and `Release()` in `finally` returns the slot on exception. A raw `lock (obj) { await ... }` would be a mistake — `Monitor` is non-reentrant and a continuation on another thread could not re-enter. `FetchBlockingSafe` mirrors the lesson's trick: `Task.Run` starts the continuation on the pool **before** the caller's context is captured, so even if the caller blocks on `GetResult`, the continuation does not wait for it — no deadlock is possible. In `RunPipelineAsync`, a bounded channel with `FullMode = Wait` implements backpressure: a fast producer cannot flood a slow consumer, and instead of explicit queues and `lock` we get a type-safe primitive. `SingleReader = true` lets the runtime pick an optimized path and removes the need for synchronization in `consume`. `Writer.Complete()` in `finally` is the critical detail from the lesson: without it the consumer's `await foreach` never ends, because the channel stays "open". Lesson concepts applied: the library rule of `ConfigureAwait(false)`, the ban on `lock` around `await`, `CancellationToken` everywhere, `Channel<T>` as idiomatic concurrency with backpressure, safe sync-over-async via `Task.Run`, and the explicit awareness that in console/ASP.NET Core there is no context, yet the habit is kept for portability.

#### Going deeper (bonus)

1. **Eliminate duplicate cache requests.** Move `FetchCachedAsync` to `ConcurrentDictionary<Uri, Lazy<Task<string>>>` with `GetOrAdd`, so a parallel miss on the same key performs exactly one network request. Write a test that proves it (a call counter in the fake handler).
2. **WPF deadlock demo.** Build a tiny WPF project with a button whose `Click` handler calls `.Result` on `FetchAsync` without `ConfigureAwait(false)` inside. Show the deadlock, then fix it two ways: (a) an `async void` handler with `await`, (b) `FetchBlockingSafe`. Capture the difference in `REPORT.md`.
3. **Blazor WASM comparison.** Blazor WASM has its own `SynchronizationContext`. Write a note: when does `ConfigureAwait(false)` in application code hurt there, and when does it not.
4. **Benchmark `ConfigureAwait(false)` in ASP.NET Core.** Use `BenchmarkDotNet` to measure the difference (or lack thereof) between a method with and without `ConfigureAwait(false)` on an ASP.NET Core host. Explain the result through `SynchronizationContext.Current == null`.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Решение из трёх проектов (`AsyncFetcher`, `AsyncFetcher.Demo`, `AsyncFetcher.Tests`) собирается без ошибок и предупреждений `CA2007`.
- [ ] (RU) В каждом `await` библиотеки присутствует `ConfigureAwait(false)`.
- [ ] (RU) `CancellationToken` проходит до底层 HTTP-вызова; есть тест на отмену.
- [ ] (RU) `SemaphoreSlim` ограничивает параллелизм; тест доказывает лимит 8.
- [ ] (RU) `Channel<T>` с `Writer.Complete()` в `finally` и `SingleReader = true`.
- [ ] (RU) Нет `lock` вокруг `await`, нет `async void`, `.Result`/`.Wait()` только в помеченном демо.
- [ ] (RU) `REPORT.md` описывает дедлок, `Task.Run`-спасение и поведение в ASP.NET Core.
- [ ] (RU) Использованы возможности C# 12 (top-level statements, pattern matching, collection expressions, `await using`).
- [ ] (EN) The three-project solution builds with no `CA2007` warnings.
- [ ] (EN) Every library `await` carries `ConfigureAwait(false)`.
- [ ] (EN) `CancellationToken` reaches the HTTP layer; a cancellation test exists.
- [ ] (EN) `SemaphoreSlim` throttles concurrency; a test proves the limit of 8.
- [ ] (EN) `Channel<T>` calls `Writer.Complete()` in `finally` and uses `SingleReader = true`.
- [ ] (EN) No `lock` around `await`, no `async void`, `.Result`/`.Wait()` only in the marked demo.
- [ ] (EN) `REPORT.md` explains the deadlock, the `Task.Run` rescue, and ASP.NET Core behavior.
- [ ] (EN) C# 12 features are used (top-level statements, pattern matching, collection expressions, `await using`).

#### Ресурсы / Resources
- [Microsoft Learn — ConfigureAwait](https://learn.microsoft.com/dotnet/api/system.threading.tasks.task.configureawait)
- [Microsoft Learn — SynchronizationContext](https://learn.microsoft.com/dotnet/api/system.threading.synchronizationcontext)
- [Stephen Toub — ConfigureAwait FAQ](https://devblogs.microsoft.com/dotnet/configureawait-faq/)
- [Microsoft Learn — Channel<T>](https://learn.microsoft.com/dotnet/api/system.threading.channels.channel-1)
- [Microsoft Learn — SemaphoreSlim](https://learn.microsoft.com/dotnet/api/system.threading.semaphoreslim)
- [Microsoft Learn — HttpCompletionOption](https://learn.microsoft.com/dotnet/api/system.net.http.httpcompletionoption)

---

[← К уроку M09-L07](lesson-M09-L07-configureawait-synccontext.md) | [⬆ К модулю M09](../README.md) | [Следующее ДЗ →](homework-M09-L08-valuetask.md)
