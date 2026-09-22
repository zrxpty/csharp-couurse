---
[← К уроку M11-L06](lesson-M11-L06-concurrent-collections.md) | [⬆ К модулю M11](../README.md) | [Следующее ДЗ →](homework-M11-L07-channels.md)
---

### Домашнее задание M11-L06: ConcurrentDictionary, ConcurrentQueue, ConcurrentBag / Homework M11-L06: ConcurrentDictionary, ConcurrentQueue, ConcurrentBag

**Урок / Lesson:** M11-L06
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться применять три ключевые concurrent-коллекции (`ConcurrentDictionary`, `ConcurrentQueue`, `ConcurrentBag`) в реалистичном параллельном сценарии — анализаторе журнала веб-запросов: построить атомарные счётчики через `AddOrUpdate` и CAS-цикл `TryUpdate`, реализовать producer/consumer через `ConcurrentQueue` с `CancellationToken`, собрать агрегаты через `ConcurrentBag` в `Parallel.ForEach`, и научиться избегать классических ловушек (`.Result` в фабрике, `Thread.Sleep` в consumer, `lock`+`await`).
**Goal (EN):** Learn to apply the three core concurrent collections (`ConcurrentDictionary`, `ConcurrentQueue`, `ConcurrentBag`) in a realistic parallel scenario — a web-request log analyzer: build atomic counters via `AddOrUpdate` and a `TryUpdate` CAS loop, implement producer/consumer over `ConcurrentQueue` with `CancellationToken`, gather aggregates through `ConcurrentBag` inside `Parallel.ForEach`, and learn to avoid the classic traps (`.Result` in a factory, `Thread.Sleep` in a consumer, `lock`+`await`).

#### Связь с уроком / Connection to the lesson
(RU) ДЗ напрямую закрепляет примеры кода урока M11-L06: атомарный счётчик по IP через `AddOrUpdate` и CAS-цикл `GetOrAdd` + `TryUpdate`, producer/consumer через `ConcurrentQueue` с корректным async-yield вместо `Thread.Sleep`, сбор результатов в `ConcurrentBag` через `Parallel.For`, и ограниченный буфер на `BlockingCollection`. Отдельно проверяются best practices (чистая фабрика, `CancellationToken` везде, никаких `lock`+`await`) и анти-паттерны из раздела «Частые ошибки».
(EN) This homework directly reinforces the M11-L06 code samples: the atomic per-IP counter via `AddOrUpdate` and the `GetOrAdd` + `TryUpdate` CAS loop, producer/consumer over `ConcurrentQueue` with proper async-yield instead of `Thread.Sleep`, result gathering in `ConcurrentBag` through `Parallel.For`, and a bounded buffer on `BlockingCollection`. It also exercises the best practices (pure factory, `CancellationToken` everywhere, no `lock`+`await`) and the anti-patterns from the "Common Mistakes" section.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы — backend-разработчик сервиса аналитики трафика. Каждый час поступает журнал веб-запросов: миллионы строк вида `IP METHOD PATH STATUS BYTES DURATION_MS`. Один процесс должен параллельно: (1) читать и парсить строки, (2) агрегировать счётчики по IP-адресам (сколько запросов, суммарная длительность), (3) складывать «медленные запросы» (duration > 500 мс) в очередь на отдельную обработку-выгрузку, (4) параллельно по всем обработанным запросам считать гистограмму кодов ответа. Обычный `Dictionary<string, Stat>` под нагрузкой гоняет: два потока одновременно видят отсутствие ключа, оба пишут ноль, оба инкрементируют — теряется апдейт. `Queue<T>` при конкурентном `Enqueue`/`TryDequeue` портит связи узлов и бросает `InvalidOperationException`. `List<T>` в `Parallel.ForEach` вообще не thread-safe. В уроке M11-L06 вы изучили три коллекции из `System.Collections.Concurrent`, спроектированные под такую нагрузку: `ConcurrentDictionary` с атомарными фабриками `GetOrAdd`/`AddOrUpdate`, lock-free `ConcurrentQueue` для producer/consumer, и `ConcurrentBag` с thread-local стеком и work-stealing для неупорядоченного сбора. В этом задании вы соберёте их вместе в один рабочий конвейер и убедитесь, что понимаете, почему каждый выбор правильный, и где подстерегают классические ловушки — `.Result` в фабрике, `Thread.Sleep` в consumer-цикле, `lock` + `await`, побочные эффекты внутри `valueFactory`.

#### Что нужно сделать (пошагово)
1. Создайте консольный проект .NET 8 с именем `TrafficAnalyzer`:
   ```
   dotnet new console -n TrafficAnalyzer -o TrafficAnalyzer --framework net8.0
   cd TrafficAnalyzer
   dotnet add package Microsoft.CodeAnalysis.NetAnalyzers  # опционально, для стат. анализа
   ```
   Убедитесь, что в `TrafficAnalyzer.csproj` стоит `<LangVersion>latest</LangVersion>` и `<Nullable>enable</Nullable>` (C# 12).

2. В файле `Program.cs` (top-level statements) сгенерируйте синтетический журнал: метод `IEnumerable<string> GenerateLog(int lines)`, который через `Random` (с seed для воспроизводимости) выдаёт строки формата `IP METHOD PATH STATUS BYTES DURATION_MS`. Используйте коллекционные выражы: `string[] methods = ["GET", "POST", "PUT", "DELETE"];` и `int[] statuses = [200, 200, 200, 404, 500, 503];`. IP генерируйте из пула ~1000 уникальных адресов, чтобы словарь был не пустой.

3. Заведите тип записи:
   ```csharp
   public sealed record RequestEntry(string Ip, string Method, string Path, int Status, int Bytes, int DurationMs);
   ```

4. Реализуйте парсер `RequestEntry? Parse(string line)` через pattern matching и `Span<char>`/`Split`. Если строка не парсится — вернуть `null`.

5. Постройте `ConcurrentDictionary<string, IpStat>`, где `record IpStat(int Hits, long TotalDurationMs)`. Накапливайте статистику через атомарные операции. Вам потребуются оба приёма из урока:
   - инкремент хитов через `AddOrUpdate` с чистой фабрикой (ключа нет → `new IpStat(1, 0)`, ключ есть → `new IpStat(current.Hits + 1, current.TotalDurationMs)`);
   - добавить длительность к уже существующему ключу через CAS-цикл: `GetOrAdd` начального значения, затем цикл `TryUpdate(ip, next, current)`, пока не сравняется.

6. Создайте `ConcurrentQueue<RequestEntry>` для медленных запросов (duration > 500 мс). Производитель — тот же цикл, что парсит журнал: каждый «медленный» элемент `Enqueue` в очередь.

7. Отдельный `Task`-consumer читает очередь через `TryDequeue`. Если очередь пуста — НЕ вызывайте `Thread.Sleep`; используйте `await Task.Delay(2, cts.Token)`. Consumer копит «медленные» запросы в `ConcurrentBag<RequestEntry>` (доказывает работу bag в потребительском потоке). Корректно завершается по `cts.Cancel()` и по флагу «производитель закончил» (используйте `Volatile`/`Interlocked` флаг `producerDone`).

8. Параллельно через `Parallel.ForEach` по всем распарсенным записям соберите гистограмму кодов ответа в `ConcurrentDictionary<int, int>` (снова `AddOrUpdate`). Замерьте время через `Stopwatch`.

9. В конце выведите: топ-5 IP по хитам (`OrderByDescending`), суммарную длительность по всем IP, количество медленных запросов в bag, гистограмму кодов ответа и время работы.

10. Запустите: `dotnet run -c Release`. Ожидаемый вывод — 5 строк топа IP, гистограмма (например `200 → 7500, 404 → 1500, 500 → 500, 503 → 500`), и `Elapsed: ~XXX ms`. Проверьте: суммы хитов по всем IP должны равняться числу успешно распарсенных строк; сумма в гистограмме — тому же числу. Если не равны — у вас гонка, ищите, где потерялся апдейт.

11. Напишите отдельный метод `DemonstrateAntiPatterns()`, который закомментирован и показывает три анти-паттерна из урока: `.Result` в фабрике `GetOrAdd`, `Thread.Sleep` в consumer, `lock + await`. В комментариях объясните, почему это баг, и рядом — правильный вариант.

#### Требования к решению
- Только C# 12 / .NET 8: top-level statements, collection expressions (`[...]`), raw string literals там, где уместно (`"""..."""` для многострочного шаблона отчёта), file-scoped namespaces в любом вспомогательном файле, pattern matching, `required`/`init`/`record` для DTO.
- Использовать ровно три коллекции из заголовка: `ConcurrentDictionary`, `ConcurrentQueue`, `ConcurrentBag`. `BlockingCollection` допустим только в бонусном углублении. Никаких `lock` вокруг этих коллекций — их внутренняя синхронизация достаточна для одной операции.
- Все блокирующие/async-вызовы (`Task.Delay`, `Parallel.ForEach` c `ParallelOptions.CancellationToken`, `BlockingCollection.Add/Take` в бонусе) принимают `CancellationToken`.
- Фабрики `GetOrAdd`/`AddOrUpdate` обязаны быть `static` lambda и чистыми: без I/O, без `.Result`, без мутации внешнего состояния. Побочные эффекты (логирование «нового IP обнаружен») вынесите за пределы фабрики — например, через `TryAdd` + отдельную проверку.
- Для атомарного read-compute-write длительности использовать CAS-цикл `TryUpdate`, а не повторный `AddOrUpdate` с чтением (чтобы продемонстрировать приём из урока).
- Consumer не должен блокировать поток пула `Thread.Sleep`/`.Wait()`. Только `await Task.Delay` или (в бонусе) `Channel<T>.ReadAllAsync`.
- Код компилируется без warning-ов в `Release` (`TreatWarningsAsErrors` не обязателен, но `dotnet build -c Release` должен быть чистым).
- Воспроизводимость: `Random` с фиксированным seed; при двух запусках суммы совпадают. Это доказывает, что гонок нет — результат не плавает от запуска к запуску.

#### Тонкости и подводные камни
- **Фабрика может выполниться несколько раз.** `GetOrAdd(key, factory)` вправе вызвать `factory` больше одного раза, даже если в словарь попадёт лишь одно значение (соревнование потоков). Поэтому внутри фабрики нельзя делать `httpClient.GetAsync(ip).Result` (deadlock в sync context) или запись в БД — это может выполниться 2–3 раза. Выносите side-effect за пределы фабрики: `if (dict.TryAdd(ip, value)) LogFirstSeen(ip);`.
- **`AddOrUpdate` не решает всё.** Для счётчика `Hits` `AddOrUpdate` идеален: `addValueFactory: _ => 1, updateValueFactory: (_, c) => c + 1`. Но для сложного read-compute-write (`TotalDurationMs += duration`) `AddOrUpdate` с чтением внешнего значения даст гонку, потому что `duration` — внешний параметр. Поэтому для накопления длительности используйте CAS-цикл: `GetOrAdd(ip, _ => IpStat.Zero)`, затем `while (dict.TryGetValue(ip, out var cur)) { var next = cur with { TotalDurationMs = cur.TotalDurationMs + duration }; if (dict.TryUpdate(ip, next, cur)) break; }`. `TryUpdate` сравнивает `comparisonValue` и пишет лишь если оно совпало — иначе повтор.
- **`Thread.Sleep` в consumer — thread-pool starvation.** Если consumer крутит `while(true) { if (q.TryDequeue(...)) ... else Thread.Sleep(5); }`, он держит поток пула, даже когда работы нет. Под нагрузкой пул истощается, новые `Task.Run` ждут потоков — deadlock-ish поведение. Правильно: `await Task.Delay(2, token)` — async-yield освобождает поток обратно в пул.
- **`lock + await` запрещён компилятором, но `SemaphoreSlim.Wait + await` — частая ошибка.** Если нужен async-замок, всегда `await sem.WaitAsync(token)`, а не `sem.Wait()`. Иначе критическая секция держит поток во время await.
- **`foreach` по `ConcurrentDictionary` во время записи.** Исключения не будет (в отличие от `Dictionary`), но вы можете увидеть промежуточное состояние — это нормально для eventual consistency, не рассчитывайте на точный снимок. Если снимок нужен — копируйте ключи через `dict.ToArray()`.
- **`async void` для event-handler-ов задач.** Необработанное исключение рвёт весь процесс. Всегда `async Task`.
- **Не блокируйте `GetConsumingEnumerable` без `CompleteAdding`.** Consumer-цикл `GetConsumingEnumerable` завершается только после `CompleteAdding()` — иначе зависнет навсегда. В бонусе на `Channel<T>` этой проблемы нет — `ReadAllAsync` естественно закрывается при `writer.Complete()`.

#### Критерии приёмки
- [ ] Проект `TrafficAnalyzer` собирается `dotnet build -c Release` без ошибок и warning-ов.
- [ ] Использованы top-level statements, collection expressions, как минимум один raw string literal для шаблона отчёта.
- [ ] Есть `record RequestEntry` и `record IpStat` (или эквивалентный immutable DTO).
- [ ] `ConcurrentDictionary<string, IpStat>` наполняется через `AddOrUpdate` (хиты) и CAS-цикл `TryUpdate` (длительность).
- [ ] Фабрики `static` lambda, без побочных эффектов, без `.Result`.
- [ ] `ConcurrentQueue<RequestEntry>` принимает медленные запросы от производителя.
- [ ] Consumer использует `TryDequeue` и `await Task.Delay` (НЕ `Thread.Sleep`) при пустой очереди.
- [ ] Consumer корректно завершается по `CancellationToken` и флагу `producerDone`.
- [ ] `ConcurrentBag<RequestEntry>` наполняется consumer-потоком.
- [ ] `Parallel.ForEach` с `ParallelOptions.CancellationToken` собирает гистограмму в `ConcurrentDictionary<int,int>`.
- [ ] Сумма хитов по всем IP = число распарсенных строк; сумма гистограммы = то же число — на каждом запуске.
- [ ] Выводится топ-5 IP, гистограмма, время работы; воспроизводимо при фиксированном seed.
- [ ] Метод `DemonstrateAntiPatterns()` присутствует (закомментирован), объясняет 3 анти-паттерна и показывает правильный вариант.
- [ ] Никаких `lock` вокруг concurrent-коллекций; `lock`+`await` отсутствует в рабочем коде.
- [ ] `CancellationToken` передаётся во все `Task.Delay`, `Parallel.ForEach`, `Task.Run`.
- [ ] В комментариях к коду на RU+EN объяснён выбор каждой коллекции.

#### Подсказки (без прямого ответа)
- Для CAS-цикла прочитайте текущее значение через `TryGetValue`, вычислите `next` через `with`, вызовите `TryUpdate(ip, next, current)`. Если вернуло `false` — кто-то изменил значение, повторите цикл. Не используйте `ref` — concurrent-коллекции не дают ref-доступа.
- Флаг «производитель закончил» удобнее через `private volatile bool _producerDone;` или `Interlocked.Exchange(ref _flag, 1)`. Проверяйте его в consumer после `TryDequeue`-промаха.
- Для топ-5: `dict.ToArray().OrderByDescending(kv => kv.Value.Hits).Take(5)` — `ToArray` снимает безопасный снимок, `OrderByDescending` уже на массиве не гоняет.
- Гистограмма через `AddOrUpdate(status, _ => 1, (_, c) => c + 1)` — классический паттерн счётчика из урока.
- `Parallel.ForEach` принимает `ParallelOptions { CancellationToken = cts.Token, MaxDegreeOfParallelism = Environment.ProcessorCount }`.
- Raw string literal удобно для многострочного шаблона отчёта:
  ```csharp
  var report = """
               Топ-5 IP по хитам / Top-5 IP by hits:
               {0}
               Гистограмма кодов / Status histogram:
               {1}
               """;
  ```

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — TrafficAnalyzer: ConcurrentDictionary + ConcurrentQueue + ConcurrentBag
// Полный рабочий пример к ДЗ M11-L06 / Full working example for homework M11-L06.

using System.Collections.Concurrent;
using System.Diagnostics;

// ───────────────────────────────────────────────────────────────────────────
// 0. DTO / DTO
// ───────────────────────────────────────────────────────────────────────────
public sealed record RequestEntry(string Ip, string Method, string Path, int Status, int Bytes, int DurationMs);
public sealed record IpStat(int Hits, long TotalDurationMs)
{
    public static IpStat Zero => new(0, 0);
}

// ───────────────────────────────────────────────────────────────────────────
// 1. Генерация синтетического журнала / Synthetic log generation
// ───────────────────────────────────────────────────────────────────────────
static IEnumerable<string> GenerateLog(int lines, int seed)
{
    var rng = new Random(seed);
    string[] methods = ["GET", "POST", "PUT", "DELETE"];              // collection expression
    int[] statuses  = [200, 200, 200, 404, 500, 503];
    string[] ips    = Enumerable.Range(0, 1000).Select(_ => $"{rng.Next(1,255)}.{rng.Next(0,255)}.{rng.Next(0,255)}.{rng.Next(0,255)}").Distinct().ToArray();

    for (int i = 0; i < lines; i++)
    {
        int duration = rng.Next(1, 1500); // 1..1500 мс, часть > 500 — «медленные»
        yield return $"{ips[rng.Next(ips.Length)]} {methods[rng.Next(methods.Length)]} /api/r{rng.Next(50)} {statuses[rng.Next(statuses.Length)]} {rng.Next(64, 8192)} {duration}";
    }
}

// ───────────────────────────────────────────────────────────────────────────
// 2. Парсер с pattern matching / Parser with pattern matching
// ───────────────────────────────────────────────────────────────────────────
static RequestEntry? Parse(string line)
{
    var p = line.Split(' ');
    if (p.Length != 6) return null;
    if (int.TryParse(p[3], out int status) && int.TryParse(p[4], out int bytes) && int.TryParse(p[5], out int dur))
        return new RequestEntry(p[0], p[1], p[2], status, bytes, dur);
    return null;
}

// ───────────────────────────────────────────────────────────────────────────
// 3. Главная логика / Main pipeline
// ───────────────────────────────────────────────────────────────────────────
var stats = new ConcurrentDictionary<string, IpStat>();
var slowQueue = new ConcurrentQueue<RequestEntry>();
var slowBag = new ConcurrentBag<RequestEntry>();
var histogram = new ConcurrentDictionary<int, int>();

using var cts = new CancellationTokenSource();
var sw = Stopwatch.StartNew();

// Флаг «производитель закончил» через Volatile / Volatile producer-done flag
private volatile bool _producerDone = false; // (в классе Program; в top-level — static поле)

// --- CAS-цикл добавления длительности к IpStat / CAS loop for duration ---
static IpStat AddDuration(ConcurrentDictionary<string, IpStat> d, string ip, int duration)
{
    var seed = d.GetOrAdd(ip, _ => IpStat.Zero);                 // чистая фабрика / pure factory
    while (d.TryGetValue(ip, out var cur))
    {
        var next = cur with { TotalDurationMs = cur.TotalDurationMs + duration };
        if (d.TryUpdate(ip, next, comparisonValue: cur))         // CAS: пишем только если cur не изменилось
            return next;
        // кто-то изменил значение между Read и Update — повторим / retry
    }
    return seed;
}

// --- Producer: парсинг + stats + slow queue / Producer: parse + stats + slow queue ---
var producer = Task.Run(() =>
{
    foreach (var line in GenerateLog(10_000, seed: 42))
    {
        cts.Token.ThrowIfCancellationRequested();
        var e = Parse(line);
        if (e is null) continue;

        // Хиты — AddOrUpdate с чистой фабрикой / Hits via AddOrUpdate, pure factory
        stats.AddOrUpdate(
            e.Ip,
            addValueFactory: static _ => new IpStat(1, 0),
            updateValueFactory: static (_, c) => c with { Hits = c.Hits + 1 });

        // Длительность — CAS-цикл / Duration via CAS loop
        _ = AddDuration(stats, e.Ip, e.DurationMs);

        // Медленные → в очередь / Slow → queue
        if (e.DurationMs > 500) slowQueue.Enqueue(e);
    }
    _producerDone = true; // Volatile.Write(ref _producerDone, true) в строгом варианте
}, cts.Token);

// --- Consumer: медленные запросы → bag / Consumer: slow → bag ---
var consumer = Task.Run(async () =>
{
    while (!cts.IsCancellationRequested)
    {
        if (slowQueue.TryDequeue(out var e))
        {
            slowBag.Add(e); // ConcurrentBag.Add — thread-local стек, без замков
        }
        else if (_producerDone && slowQueue.IsEmpty)
        {
            break; // производитель закончил и очередь пуста — выходим
        }
        else
        {
            // НЕ Thread.Sleep! async-yield освобождает поток пула / async-yield frees pool thread
            await Task.Delay(2, cts.Token);
        }
    }
}, cts.Token);

await producer;              // ждём парсинг / wait for parsing
await consumer;              // ждём consumer / wait for consumer

// --- Гистограмма через Parallel.ForEach / Histogram via Parallel.ForEach ---
var allEntries = GenerateLog(10_000, seed: 42)   // повторная итерация по тому же seed
    .Select(Parse)
    .Where(e => e is not null)
    .Cast<RequestEntry>()
    .ToArray();

Parallel.ForEach(allEntries, new ParallelOptions { CancellationToken = cts.Token, MaxDegreeOfParallelism = Environment.ProcessorCount }, e =>
{
    histogram.AddOrUpdate(e.Status, addValueFactory: static _ => 1, updateValueFactory: static (_, c) => c + 1);
});

sw.Stop();

// --- Отчёт через raw string literal / Report via raw string literal ---
var top5 = string.Join("\n", stats.ToArray()
    .OrderByDescending(kv => kv.Value.Hits)
    .Take(5)
    .Select(kv => $"  {kv.Key,-16} hits={kv.Value.Hits,-6} dur={kv.Value.TotalDurationMs}"));

var hist = string.Join("\n", histogram.ToArray().OrderBy(kv => kv.Key)
    .Select(kv => $"  HTTP {kv.Key} → {kv.Value}"));

var report = $$"""
               Топ-5 IP по хитам / Top-5 IP by hits:
               {{top5}}

               Гистограмма кодов / Status histogram:
               {{hist}}

               Медленных запросов в bag / Slow in bag: {{slowBag.Count}}
               Сумма хитов по всем IP / Sum of hits:   {{stats.Values.Sum(s => s.Hits)}}
               Сумма гистограммы / Histogram sum:     {{histogram.Values.Sum()}}
               Время / Elapsed: {{sw.ElapsedMilliseconds}} ms
               """;
Console.WriteLine(report);

// ───────────────────────────────────────────────────────────────────────────
// 4. Анти-паттерны (НЕ делать так) / Anti-patterns (do NOT do this)
// ───────────────────────────────────────────────────────────────────────────
// ❌ .Result в фабрике GetOrAdd → deadlock в контексте синхронизации
// ❌ .Result inside GetOrAdd factory → deadlock under a sync context
//    stats.GetOrAdd(ip, k => LoadFromDbAsync(k).Result); // никогда так!
// ✅ правильно: вынести I/O наружу, фабрика чистая
//    var v = stats.GetOrAdd(ip, _ => new IpStat(0,0));
//    if (stats.TryAdd(ip, v)) await LoadFromDbAsync(ip); // side-effect вне фабрики

// ❌ Thread.Sleep в consumer → thread-pool starvation
//    while (true) if (q.TryDequeue(out var e)) ... else Thread.Sleep(5); // держит поток!
// ✅ правильно: await Task.Delay(2, token) — освобождает поток

// ❌ lock + await (или SemaphoreSlim.Wait + await) → замок держится во время await
//    lock (gate) { await DoAsync(); } // компилятор forbids
//    sem.Wait(); try { await DoAsync(); } finally { sem.Release(); } // баг
// ✅ правильно: await sem.WaitAsync(token); try { await DoAsync(); } finally { sem.Release(); }
```

**Разбор по строкам.** DTO сделаны `record`-ами — immutability обязательна для CAS: мы сравниваем ссылку/значение в `TryUpdate`, и если бы `IpStat` был mutable, конкурент мог бы мутировать его в момент сравнения. `GenerateLog` использует коллекционные выражения (`["GET", ...]`) — идиома C# 12. Парсер через `Split` + `int.TryParse` + early return `null` — это pattern matching на длину массива и бэкинг-поля. Главная часть — `AddDuration`: `GetOrAdd` с чистой фабрикой `IpStat.Zero` гарантирует, что ключ есть (и фабрика безопасна — вызывается 0..N раз, но без побочных эффектов). Затем CAS-цикл: читаем `cur`, вычисляем `next` через `with`, вызываем `TryUpdate(ip, next, cur)`. Если другой поток успел изменить `IpStat` между `TryGetValue` и `TryUpdate`, `TryUpdate` вернёт `false` и цикл повторится с уже новым `cur`. Это ровно приём из урока для read-compute-write. `AddOrUpdate` для хитов короче и достаточен, потому что инкремент не зависит от внешнего состояния. Producer кладёт медленные запросы в `ConcurrentQueue` — `Enqueue` lock-free. Consumer крутит `TryDequeue` и при пустой очереди делает `await Task.Delay(2, token)`, а не `Thread.Sleep` — это ключевая best practice из урока против thread-pool starvation. Флаг `_producerDone` (`volatile`) позволяет consumer-у выйти, когда очередь исчерпана и производитель завершён. `ConcurrentBag.Add` в consumer демонстрирует thread-local стек + work-stealing — порядок в bag не гарантируется, что и нужно для «медленных» запросов. `Parallel.ForEach` с `ParallelOptions.CancellationToken` собирает гистограмму через `AddOrUpdate` — классический счётчик. Отчёт собран через raw string literal `$$"""..."""` с интерполяцией — идиома C# 11+. Анти-паттерны вынесены в закомментированный блок с `❌`/`✅` — ровно три ловушки из урока: `.Result` в фабрике, `Thread.Sleep` в consumer, `lock`+`await`. Воспроизводимость seed=42 гарантирует, что суммы хитов и гистограммы совпадают на каждом запуске — если они разойдутся, значит где-то осталась гонка (например, забыли CAS и сделали `AddOrUpdate` с чтением внешнего `duration`).

#### Задания на углубление (бонус)
1. **Bounded pipeline на `BlockingCollection`.** Перепишите producer/consumer медленных запросов через `BlockingCollection<RequestEntry>(boundedCapacity: 100)` с `Add`/`GetConsumingEnumerable` и `CompleteAdding()`. Сравните поведение под нагрузкой с `ConcurrentQueue` + `Task.Delay`. Когда `BlockingCollection` лучше, а когда хуже?
2. **`Channel<T>` вместо `BlockingCollection` (мост к уроку M11-L07).** Перепишите конвейер через `Channel.CreateBounded<RequestEntry>(100)` с `WriteAsync`/`ReadAllAsync`. Убедитесь, что ни один поток пула не блокируется. Сравните объём кода и читаемость.
3. **Side-effecting создание ключа через `Lazy<T>`.** Реализуйте сценарий «при первом появлении IP загрузить его геолокацию из mock-сервиса». Используйте `ConcurrentDictionary<string, Lazy<GeoInfo>>` и `GetOrAdd(ip, _ => new Lazy<GeoInfo>(() => LoadGeo(ip)))`, чтобы I/O выполнялся ровно один раз на ключ. Объясните, почему `Lazy<T>` решает проблему «фабрика вызвалась N раз».
4. **Сравнение производительности.** Замерьте `Stopwatch` для трёх вариантов счётчика: (а) `lock + Dictionary`, (б) `AddOrUpdate`, (в) `GetOrAdd` + `TryUpdate` CAS-цикл, при 1, 4, 16 потоках. Постройте простую таблицу в консоли. Где concurrent выигрывает, где проигрывает? Подтвердите правило урока «≥3 потоков — concurrent».

---

## Statement in English / Постановка на английском

#### Context & motivation
You are a backend engineer on a traffic-analytics service. Every hour a web-request log arrives: millions of lines shaped `IP METHOD PATH STATUS BYTES DURATION_MS`. A single process must, in parallel: (1) read and parse the lines, (2) aggregate per-IP counters (how many requests, total duration), (3) push "slow requests" (duration > 500 ms) into a queue for separate draining/export, and (4) in parallel across all parsed entries compute a histogram of response codes. A plain `Dictionary<string, Stat>` under load races: two threads simultaneously see the key missing, both write zero, both increment — an update is lost. A `Queue<T>` under concurrent `Enqueue`/`TryDequeue` corrupts node links and throws `InvalidOperationException`. A `List<T>` inside `Parallel.ForEach` is not thread-safe at all. In lesson M11-L06 you studied three collections from `System.Collections.Concurrent` designed exactly for this load: `ConcurrentDictionary` with atomic `GetOrAdd`/`AddOrUpdate` factories, lock-free `ConcurrentQueue` for producer/consumer, and `ConcurrentBag` with thread-local storage and work-stealing for unordered gathering. In this homework you will wire them together into one working pipeline and prove that you understand why each choice is right, and where the classic traps hide — `.Result` in a factory, `Thread.Sleep` in a consumer loop, `lock` + `await`, side effects inside `valueFactory`.

#### What to do step by step
1. Create a .NET 8 console project named `TrafficAnalyzer`:
   ```
   dotnet new console -n TrafficAnalyzer -o TrafficAnalyzer --framework net8.0
   cd TrafficAnalyzer
   dotnet add package Microsoft.CodeAnalysis.NetAnalyzers   # optional, for static analysis
   ```
   Make sure `TrafficAnalyzer.csproj` has `<LangVersion>latest</LangVersion>` and `<Nullable>enable</Nullable>` (C# 12).

2. In `Program.cs` (top-level statements) generate a synthetic log: a method `IEnumerable<string> GenerateLog(int lines)` that uses `Random` with a fixed seed for reproducibility and yields lines of the form `IP METHOD PATH STATUS BYTES DURATION_MS`. Use collection expressions: `string[] methods = ["GET", "POST", "PUT", "DELETE"];` and `int[] statuses = [200, 200, 200, 404, 500, 503];`. Generate IPs from a pool of ~1000 unique addresses so the dictionary is not empty.

3. Define the entry type:
   ```csharp
   public sealed record RequestEntry(string Ip, string Method, string Path, int Status, int Bytes, int DurationMs);
   ```

4. Implement a parser `RequestEntry? Parse(string line)` using pattern matching and `Split`. If the line does not parse, return `null`.

5. Build a `ConcurrentDictionary<string, IpStat>` where `record IpStat(int Hits, long TotalDurationMs)`. Accumulate statistics using atomic operations. You need both techniques from the lesson:
   - increment hits via `AddOrUpdate` with a pure factory (missing key → `new IpStat(1, 0)`, existing key → `new IpStat(current.Hits + 1, current.TotalDurationMs)`);
   - add duration to an already-present key via a CAS loop: `GetOrAdd` the seed value, then loop `TryUpdate(ip, next, current)` until it matches.

6. Create a `ConcurrentQueue<RequestEntry>` for slow requests (duration > 500 ms). The producer is the same loop that parses the log: every "slow" entry is `Enqueue`d.

7. A separate `Task` consumer drains the queue via `TryDequeue`. If the queue is empty — do NOT call `Thread.Sleep`; use `await Task.Delay(2, cts.Token)`. The consumer accumulates slow requests into a `ConcurrentBag<RequestEntry>` (demonstrating a bag used by a consumer thread). It must shut down cleanly on `cts.Cancel()` and on a "producer done" flag (use `Volatile`/`Interlocked` flag `producerDone`).

8. In parallel, via `Parallel.ForEach` over all parsed entries, collect a status-code histogram into a `ConcurrentDictionary<int, int>` (again `AddOrUpdate`). Measure time with `Stopwatch`.

9. At the end print: top-5 IPs by hits (`OrderByDescending`), total duration across all IPs, the count of slow requests in the bag, the status histogram, and the elapsed time.

10. Run: `dotnet run -c Release`. Expected output — 5 lines of top IPs, a histogram (e.g. `200 → 7500, 404 → 1500, 500 → 500, 503 → 500`), and `Elapsed: ~XXX ms`. Verify: the sum of hits across all IPs must equal the number of successfully parsed lines; the histogram sum must equal the same number. If they differ — you have a race; find where an update was lost.

11. Write a separate method `DemonstrateAntiPatterns()` (left commented out) that shows the three anti-patterns from the lesson: `.Result` in a `GetOrAdd` factory, `Thread.Sleep` in a consumer, `lock + await`. In comments explain why each is a bug and show the correct variant next to it.

#### Requirements
- C# 12 / .NET 8 only: top-level statements, collection expressions (`[...]`), raw string literals where useful (`"""..."""` for a multi-line report template), file-scoped namespaces in any helper file, pattern matching, `required`/`init`/`record` for DTOs.
- Use exactly the three collections named in the title: `ConcurrentDictionary`, `ConcurrentQueue`, `ConcurrentBag`. `BlockingCollection` is allowed only in the bonus. No `lock` around these collections — their internal synchronization is enough for a single operation.
- Every blocking/async call (`Task.Delay`, `Parallel.ForEach` with `ParallelOptions.CancellationToken`, `BlockingCollection.Add/Take` in the bonus) takes a `CancellationToken`.
- `GetOrAdd`/`AddOrUpdate` factories must be `static` lambdas and pure: no I/O, no `.Result`, no mutation of outer state. Move side effects (logging "new IP seen") out of the factory — for example via `TryAdd` plus a separate check.
- For the atomic read-compute-write of duration, use the `TryUpdate` CAS loop rather than another `AddOrUpdate` that reads an outer value (to demonstrate the lesson technique).
- The consumer must not block a pool thread with `Thread.Sleep`/`.Wait()`. Only `await Task.Delay` or (in the bonus) `Channel<T>.ReadAllAsync`.
- The code compiles without warnings in `Release` (`TreatWarningsAsErrors` is not required, but `dotnet build -c Release` should be clean).
- Reproducibility: `Random` with a fixed seed; across two runs the sums match. This proves there are no races — the result does not drift between runs.

#### Pitfalls
- **The factory may run more than once.** `GetOrAdd(key, factory)` is allowed to invoke `factory` more than once even though only one value lands in the dictionary (threads racing). So inside the factory you must never do `httpClient.GetAsync(ip).Result` (deadlock under a sync context) or a DB write — it may execute 2–3 times. Move the side effect out: `if (dict.TryAdd(ip, value)) LogFirstSeen(ip);`.
- **`AddOrUpdate` does not solve everything.** For a hit counter `AddOrUpdate` is ideal: `addValueFactory: _ => 1, updateValueFactory: (_, c) => c + 1`. But for a complex read-compute-write (`TotalDurationMs += duration`) an `AddOrUpdate` that reads an outer `duration` still races, because `duration` is an outer parameter. So for the duration use a CAS loop: `GetOrAdd(ip, _ => IpStat.Zero)`, then `while (dict.TryGetValue(ip, out var cur)) { var next = cur with { TotalDurationMs = cur.TotalDurationMs + duration }; if (dict.TryUpdate(ip, next, cur)) break; }`. `TryUpdate` compares `comparisonValue` and writes only if it still matches — otherwise retry.
- **`Thread.Sleep` in a consumer causes thread-pool starvation.** If a consumer spins `while(true) { if (q.TryDequeue(...)) ... else Thread.Sleep(5); }`, it holds a pool thread even when there is no work. Under load the pool starves and new `Task.Run` calls wait for threads — deadlock-ish behavior. The correct form is `await Task.Delay(2, token)` — an async-yield that returns the thread to the pool.
- **`lock + await` is forbidden by the compiler, but `SemaphoreSlim.Wait + await` is a frequent mistake.** If you need an async lock, always `await sem.WaitAsync(token)`, never `sem.Wait()`. Otherwise the critical section holds a thread across the await.
- **`foreach` over a `ConcurrentDictionary` while writers are active.** No exception (unlike `Dictionary`), but you may see an intermediate state — fine for eventual consistency, do not assume a precise snapshot. If you need a snapshot, copy keys via `dict.ToArray()`.
- **`async void` for task event handlers.** An unhandled exception tears down the whole process. Always `async Task`.
- **Never block `GetConsumingEnumerable` without `CompleteAdding`.** The consumer loop `GetConsumingEnumerable` only ends after `CompleteAdding()` — otherwise it hangs forever. With `Channel<T>` in the bonus this problem disappears — `ReadAllAsync` closes naturally when `writer.Complete()` is called.

#### Acceptance criteria
- [ ] The `TrafficAnalyzer` project builds with `dotnet build -c Release` with no errors and no warnings.
- [ ] Top-level statements, collection expressions, and at least one raw string literal for the report template are used.
- [ ] `record RequestEntry` and `record IpStat` (or an equivalent immutable DTO) exist.
- [ ] `ConcurrentDictionary<string, IpStat>` is populated via `AddOrUpdate` (hits) and a `TryUpdate` CAS loop (duration).
- [ ] Factories are `static` lambdas, side-effect-free, with no `.Result`.
- [ ] `ConcurrentQueue<RequestEntry>` receives slow requests from the producer.
- [ ] The consumer uses `TryDequeue` and `await Task.Delay` (NOT `Thread.Sleep`) when the queue is empty.
- [ ] The consumer shuts down cleanly on `CancellationToken` and on the `producerDone` flag.
- [ ] `ConcurrentBag<RequestEntry>` is filled by the consumer thread.
- [ ] `Parallel.ForEach` with `ParallelOptions.CancellationToken` builds the histogram into `ConcurrentDictionary<int,int>`.
- [ ] Sum of hits across all IPs = number of parsed lines; histogram sum = same number — on every run.
- [ ] Top-5 IPs, histogram, and elapsed time are printed; reproducible with a fixed seed.
- [ ] A `DemonstrateAntiPatterns()` method is present (commented), explaining 3 anti-patterns and showing the correct variant.
- [ ] No `lock` around concurrent collections; `lock`+`await` is absent from the working code.
- [ ] `CancellationToken` is threaded through every `Task.Delay`, `Parallel.ForEach`, and `Task.Run`.
- [ ] Code comments in RU+EN explain the choice of each collection.

#### Hints (no direct answer)
- For the CAS loop, read the current value via `TryGetValue`, compute `next` via `with`, then call `TryUpdate(ip, next, current)`. If it returns `false` — someone changed the value, retry the loop. Do not use `ref` — concurrent collections do not expose ref access.
- The "producer done" flag is most convenient as `private volatile bool _producerDone;` or `Interlocked.Exchange(ref _flag, 1)`. Check it in the consumer after a `TryDequeue` miss.
- For the top-5: `dict.ToArray().OrderByDescending(kv => kv.Value.Hits).Take(5)` — `ToArray` takes a safe snapshot, `OrderByDescending` on the array does not race.
- The histogram uses `AddOrUpdate(status, _ => 1, (_, c) => c + 1)` — the classic counter pattern from the lesson.
- `Parallel.ForEach` accepts `ParallelOptions { CancellationToken = cts.Token, MaxDegreeOfParallelism = Environment.ProcessorCount }`.
- A raw string literal is handy for a multi-line report template:
  ```csharp
  var report = """
               Top-5 IP by hits:
               {0}
               Status histogram:
               {1}
               """;
  ```

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — TrafficAnalyzer: ConcurrentDictionary + ConcurrentQueue + ConcurrentBag
// Full working example for homework M11-L06.

using System.Collections.Concurrent;
using System.Diagnostics;

// ───────────────────────────────────────────────────────────────────────────
// 0. DTO
// ───────────────────────────────────────────────────────────────────────────
public sealed record RequestEntry(string Ip, string Method, string Path, int Status, int Bytes, int DurationMs);
public sealed record IpStat(int Hits, long TotalDurationMs)
{
    public static IpStat Zero => new(0, 0);
}

// ───────────────────────────────────────────────────────────────────────────
// 1. Synthetic log generation
// ───────────────────────────────────────────────────────────────────────────
static IEnumerable<string> GenerateLog(int lines, int seed)
{
    var rng = new Random(seed);
    string[] methods = ["GET", "POST", "PUT", "DELETE"];              // collection expression
    int[] statuses  = [200, 200, 200, 404, 500, 503];
    string[] ips    = Enumerable.Range(0, 1000).Select(_ => $"{rng.Next(1,255)}.{rng.Next(0,255)}.{rng.Next(0,255)}.{rng.Next(0,255)}").Distinct().ToArray();

    for (int i = 0; i < lines; i++)
    {
        int duration = rng.Next(1, 1500); // 1..1500 ms, some > 500 are "slow"
        yield return $"{ips[rng.Next(ips.Length)]} {methods[rng.Next(methods.Length)]} /api/r{rng.Next(50)} {statuses[rng.Next(statuses.Length)]} {rng.Next(64, 8192)} {duration}";
    }
}

// ───────────────────────────────────────────────────────────────────────────
// 2. Parser with pattern matching
// ───────────────────────────────────────────────────────────────────────────
static RequestEntry? Parse(string line)
{
    var p = line.Split(' ');
    if (p.Length != 6) return null;
    if (int.TryParse(p[3], out int status) && int.TryParse(p[4], out int bytes) && int.TryParse(p[5], out int dur))
        return new RequestEntry(p[0], p[1], p[2], status, bytes, dur);
    return null;
}

// ───────────────────────────────────────────────────────────────────────────
// 3. Main pipeline
// ───────────────────────────────────────────────────────────────────────────
var stats = new ConcurrentDictionary<string, IpStat>();
var slowQueue = new ConcurrentQueue<RequestEntry>();
var slowBag = new ConcurrentBag<RequestEntry>();
var histogram = new ConcurrentDictionary<int, int>();

using var cts = new CancellationTokenSource();
var sw = Stopwatch.StartNew();

private volatile bool _producerDone = false; // (on the Program class; static field in top-level)

// --- CAS loop adding duration to IpStat ---
static IpStat AddDuration(ConcurrentDictionary<string, IpStat> d, string ip, int duration)
{
    var seed = d.GetOrAdd(ip, _ => IpStat.Zero);                 // pure factory
    while (d.TryGetValue(ip, out var cur))
    {
        var next = cur with { TotalDurationMs = cur.TotalDurationMs + duration };
        if (d.TryUpdate(ip, next, comparisonValue: cur))         // CAS: write only if cur still matches
            return next;
        // someone changed the value between Read and Update — retry
    }
    return seed;
}

// --- Producer: parse + stats + slow queue ---
var producer = Task.Run(() =>
{
    foreach (var line in GenerateLog(10_000, seed: 42))
    {
        cts.Token.ThrowIfCancellationRequested();
        var e = Parse(line);
        if (e is null) continue;

        // Hits via AddOrUpdate, pure factory
        stats.AddOrUpdate(
            e.Ip,
            addValueFactory: static _ => new IpStat(1, 0),
            updateValueFactory: static (_, c) => c with { Hits = c.Hits + 1 });

        // Duration via CAS loop
        _ = AddDuration(stats, e.Ip, e.DurationMs);

        // Slow → queue
        if (e.DurationMs > 500) slowQueue.Enqueue(e);
    }
    _producerDone = true; // Volatile.Write(ref _producerDone, true) in a strict variant
}, cts.Token);

// --- Consumer: slow → bag ---
var consumer = Task.Run(async () =>
{
    while (!cts.IsCancellationRequested)
    {
        if (slowQueue.TryDequeue(out var e))
        {
            slowBag.Add(e); // ConcurrentBag.Add — thread-local stack, no locks
        }
        else if (_producerDone && slowQueue.IsEmpty)
        {
            break; // producer done and queue empty — exit
        }
        else
        {
            // NOT Thread.Sleep! async-yield frees the pool thread
            await Task.Delay(2, cts.Token);
        }
    }
}, cts.Token);

await producer;              // wait for parsing
await consumer;              // wait for consumer

// --- Histogram via Parallel.ForEach ---
var allEntries = GenerateLog(10_000, seed: 42)   // second iteration over the same seed
    .Select(Parse)
    .Where(e => e is not null)
    .Cast<RequestEntry>()
    .ToArray();

Parallel.ForEach(allEntries, new ParallelOptions { CancellationToken = cts.Token, MaxDegreeOfParallelism = Environment.ProcessorCount }, e =>
{
    histogram.AddOrUpdate(e.Status, addValueFactory: static _ => 1, updateValueFactory: static (_, c) => c + 1);
});

sw.Stop();

// --- Report via raw string literal ---
var top5 = string.Join("\n", stats.ToArray()
    .OrderByDescending(kv => kv.Value.Hits)
    .Take(5)
    .Select(kv => $"  {kv.Key,-16} hits={kv.Value.Hits,-6} dur={kv.Value.TotalDurationMs}"));

var hist = string.Join("\n", histogram.ToArray().OrderBy(kv => kv.Key)
    .Select(kv => $"  HTTP {kv.Key} → {kv.Value}"));

var report = $$"""
               Top-5 IP by hits:
               {{top5}}

               Status histogram:
               {{hist}}

               Slow in bag: {{slowBag.Count}}
               Sum of hits:   {{stats.Values.Sum(s => s.Hits)}}
               Histogram sum: {{histogram.Values.Sum()}}
               Elapsed: {{sw.ElapsedMilliseconds}} ms
               """;
Console.WriteLine(report);

// ───────────────────────────────────────────────────────────────────────────
// 4. Anti-patterns (do NOT do this)
// ───────────────────────────────────────────────────────────────────────────
// ❌ .Result inside a GetOrAdd factory → deadlock under a sync context
//    stats.GetOrAdd(ip, k => LoadFromDbAsync(k).Result); // never!
// ✅ correct: move I/O out, keep the factory pure
//    var v = stats.GetOrAdd(ip, _ => new IpStat(0,0));
//    if (stats.TryAdd(ip, v)) await LoadFromDbAsync(ip); // side effect outside factory

// ❌ Thread.Sleep in a consumer → thread-pool starvation
//    while (true) if (q.TryDequeue(out var e)) ... else Thread.Sleep(5); // holds a thread!
// ✅ correct: await Task.Delay(2, token) — frees the thread

// ❌ lock + await (or SemaphoreSlim.Wait + await) → the lock is held across the await
//    lock (gate) { await DoAsync(); } // compiler forbids
//    sem.Wait(); try { await DoAsync(); } finally { sem.Release(); } // bug
// ✅ correct: await sem.WaitAsync(token); try { await DoAsync(); } finally { sem.Release(); }
```

**Line-by-line walk-through.** The DTOs are `record`s — immutability is mandatory for CAS: we compare the reference/value in `TryUpdate`, and if `IpStat` were mutable a competitor could mutate it during the comparison. `GenerateLog` uses collection expressions (`["GET", ...]`) — a C# 12 idiom. The parser uses `Split` + `int.TryParse` + early `null` return — pattern matching on array length and backing fields. The core of the solution is `AddDuration`: `GetOrAdd` with a pure `IpStat.Zero` factory guarantees the key is present (and the factory is safe — it may be invoked 0..N times but has no side effects). Then the CAS loop: read `cur`, compute `next` via `with`, call `TryUpdate(ip, next, cur)`. If another thread changed the `IpStat` between `TryGetValue` and `TryUpdate`, `TryUpdate` returns `false` and the loop retries with the new `cur`. This is exactly the lesson technique for read-compute-write. `AddOrUpdate` for hits is shorter and sufficient because the increment does not depend on outer state. The producer pushes slow requests into `ConcurrentQueue` — `Enqueue` is lock-free. The consumer spins `TryDequeue` and, when the queue is empty, does `await Task.Delay(2, token)` instead of `Thread.Sleep` — the key best practice from the lesson against thread-pool starvation. The `_producerDone` flag (`volatile`) lets the consumer exit once the queue is drained and the producer has finished. `ConcurrentBag.Add` in the consumer demonstrates a thread-local stack plus work-stealing — order in the bag is not guaranteed, which is exactly what we want for "slow" requests. `Parallel.ForEach` with `ParallelOptions.CancellationToken` builds the histogram via `AddOrUpdate` — the classic counter. The report is assembled with a raw string literal `$$"""..."""` with interpolation — a C# 11+ idiom. The anti-patterns are collected in a commented block with `❌`/`✅` — exactly the three traps from the lesson: `.Result` in a factory, `Thread.Sleep` in a consumer, `lock`+`await`. Reproducibility via seed=42 guarantees that the hit sum and histogram sum match on every run — if they diverge, a race remains somewhere (for example you forgot the CAS and did an `AddOrUpdate` that reads the outer `duration`).

#### Going deeper (bonus)
1. **Bounded pipeline on `BlockingCollection`.** Rewrite the slow-request producer/consumer using `BlockingCollection<RequestEntry>(boundedCapacity: 100)` with `Add`/`GetConsumingEnumerable` and `CompleteAdding()`. Compare its behavior under load with `ConcurrentQueue` + `Task.Delay`. When is `BlockingCollection` better, and when worse?
2. **`Channel<T>` instead of `BlockingCollection` (a bridge to lesson M11-L07).** Rewrite the pipeline with `Channel.CreateBounded<RequestEntry>(100)` using `WriteAsync`/`ReadAllAsync`. Confirm that no pool thread ever blocks. Compare code volume and readability.
3. **Side-effecting key creation via `Lazy<T>`.** Implement the scenario "on the first appearance of an IP, load its geolocation from a mock service". Use `ConcurrentDictionary<string, Lazy<GeoInfo>>` and `GetOrAdd(ip, _ => new Lazy<GeoInfo>(() => LoadGeo(ip)))` so that I/O runs exactly once per key. Explain why `Lazy<T>` solves the "factory ran N times" problem.
4. **Performance comparison.** Measure `Stopwatch` for three counter variants: (a) `lock + Dictionary`, (b) `AddOrUpdate`, (c) `GetOrAdd` + `TryUpdate` CAS loop, at 1, 4, and 16 threads. Print a simple table to the console. Where does concurrent win, where does it lose? Confirm the lesson rule "≥3 threads — go concurrent".

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `TrafficAnalyzer` собирается в `Release` без ошибок и warning-ов.
- [ ] (RU) Использованы top-level statements, коллекционные выражения, raw string literal.
- [ ] (RU) `ConcurrentDictionary` наполняется через `AddOrUpdate` и CAS-цикл `TryUpdate`.
- [ ] (RU) Фабрики `static` и чистые, без `.Result` и побочных эффектов.
- [ ] (RU) `ConcurrentQueue` + `ConcurrentBag` корректно связывают producer и consumer.
- [ ] (RU) Consumer использует `await Task.Delay`, не `Thread.Sleep`.
- [ ] (RU) `CancellationToken` проброшен во все блокирующие/async-вызовы.
- [ ] (RU) Сумма хитов = числу строк; гистограмма сходится; воспроизводимо при фиксированном seed.
- [ ] (RU) `DemonstrateAntiPatterns()` присутствует и объяснён.
- [ ] (EN) The `TrafficAnalyzer` project builds cleanly in `Release`.
- [ ] (EN) Top-level statements, collection expressions, and a raw string literal are used.
- [ ] (EN) `ConcurrentDictionary` is populated via `AddOrUpdate` and a `TryUpdate` CAS loop.
- [ ] (EN) Factories are `static` and pure, with no `.Result` and no side effects.
- [ ] (EN) `ConcurrentQueue` + `ConcurrentBag` correctly wire the producer and consumer.
- [ ] (EN) The consumer uses `await Task.Delay`, not `Thread.Sleep`.
- [ ] (EN) `CancellationToken` is threaded through every blocking/async call.
- [ ] (EN) The hit sum equals the line count; the histogram converges; reproducible with a fixed seed.
- [ ] (EN) `DemonstrateAntiPatterns()` is present and explained.

#### Ресурсы / Resources
- [Microsoft Learn — Thread-safe collections](https://learn.microsoft.com/dotnet/standard/collections/thread-safe/)
- [Microsoft Learn — ConcurrentDictionary<TKey,TValue>](https://learn.microsoft.com/dotnet/api/system.collections.concurrent.concurrentdictionary-2)
- [Microsoft Learn — ConcurrentQueue<T>](https://learn.microsoft.com/dotnet/api/system.collections.concurrent.concurrentqueue-1)
- [Microsoft Learn — ConcurrentBag<T>](https://learn.microsoft.com/dotnet/api/system.collections.concurrent.concurrentbag-1)
- [Microsoft Learn — BlockingCollection overview](https://learn.microsoft.com/dotnet/standard/collections/thread-safe/blockingcollection-overview)
- [Microsoft Learn — System.Threading.Channels](https://learn.microsoft.com/dotnet/api/system.threading.channels)
- [Microsoft Learn — Interlocked operations](https://learn.microsoft.com/dotnet/api/system.threading.interlocked)
- [Stephen Toub — Parallel Programming with .NET blog](https://devblogs.microsoft.com/dotnet/category/parallel-programming/)

---

[← К уроку M11-L06](lesson-M11-L06-concurrent-collections.md) | [⬆ К модулю M11](../README.md) | [Следующее ДЗ →](homework-M11-L07-channels.md)
