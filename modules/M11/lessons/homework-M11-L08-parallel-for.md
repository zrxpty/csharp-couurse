---
[← К уроку M11-L08](lesson-M11-L08-parallel-for.md) | [⬆ К модулю M11](../README.md) | [Следующее ДЗ →](homework-M11-L09-deadlocks-race-conditions.md)
---

### Домашнее задание M11-L08: Parallel.For/ForEach/Invoke, Partitioner / Homework M11-L08: Parallel.For/ForEach/Invoke, Partitioner

**Урок / Lesson:** M11-L08
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) На практике освоить `Parallel.For`, `Parallel.ForEach`, `Parallel.Invoke` и `Partitioner` для CPU-bound параллелизма по данным; научиться применять `ParallelOptions` (отмена, лимит параллелизма), thread-local аккумуляторы `localInit`/`localFinally`, безопасную агрегацию через `ConcurrentDictionary`/`Interlocked`, избегать thread-pool starvation и корректно останавливать цикл через `loopState.Stop()`. (EN) Gain hands-on mastery of `Parallel.For`, `Parallel.ForEach`, `Parallel.Invoke` and `Partitioner` for CPU-bound data parallelism; learn to apply `ParallelOptions` (cancellation, degree of parallelism), thread-local accumulators `localInit`/`localFinally`, safe aggregation via `ConcurrentDictionary`/`Interlocked`, avoid thread-pool starvation, and stop a loop correctly with `loopState.Stop()`.

#### Связь с уроком / Connection to the lesson
(RU) Задание построено поверх примеров урока: вы повторите паттерн `localInit`/`localFinally` для безблокировочной агрегации, примените `Parallel.ForEach` с `ConcurrentDictionary`, соберёте независимые вычисления через `Parallel.Invoke` и столкнётесь с неравномерной нагрузкой, которую снимет `Partitioner.Create`. Все «частые ошибки» урока (race, `.Result`, отсутствие токена, `Break` вместо `Stop`) отражены в критериях приёмки. (EN) The assignment is built on top of the lesson examples: you will reproduce the `localInit`/`localFinally` lock-free aggregation pattern, apply `Parallel.ForEach` with `ConcurrentDictionary`, gather independent computations through `Parallel.Invoke`, and face uneven workload that `Partitioner.Create` resolves. Every "common mistake" from the lesson (race, `.Result`, missing token, `Break` instead of `Stop`) is reflected in the acceptance criteria.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы пишете модуль `LogAnalyzer` для службы мониторинга, которая раз в минуту получает пачку «сырых» логов от десятков микросервисов. Каждый лог — это строка вида `2024-05-12T10:13:45Z|svc-orders|ERROR|order #42 failed: timeout` (ISO-время, имя сервиса, уровень, сообщение). За минуту может прийти от 50 тысяч до 5 миллионов строк, и сервис обязан отдать агрегированный отчёт быстрее, чем накопится следующая пачка. Профилирование показывает, что узкое место — CPU: парсинг строк, вычисление хешей сообщений, сортировка и подсчёт гистограмм. Ввод-вывод уже асинхронный (урок M11-L07 по каналам), поэтому здесь нужен именно data-parallel CPU-конвейер из `System.Threading.Tasks.Parallel`.

Вы не можете использовать `Task.Run` вручную — слишком велика цена координации, и вы рискуете получить thread-pool starvation, если кто-то случайно вызовет `.Result` внутри тела. Класс `Parallel` берёт на себя разбиение работы на чанки и неявный `Barrier` в конце цикла, поэтому вы описываете «что делать с порцией данных», а TPL решает «как раздать порции пулу потоков». При этом вы должны явно управлять тремя рычагами `ParallelOptions`: `MaxDegreeOfParallelism`, `CancellationToken` и (опционально) `TaskScheduler`. Особое внимание — агрегациям: наивный `sum += x` в общем теле даёт race condition, и итог непредсказуем. Правильный путь — thread-local аккумулятор через перегрузки `Parallel.For`/`Parallel.ForEach` с `localInit`/`localFinally`, где каждый воркер копит свою локальную сумму без блокировок, а в `localFinally` сливает её в общий результат через `Interlocked.Add`. Дополнительно часть работы неравномерна (длинные сообщения парсятся дольше коротких), и там нужен `Partitioner.Create` с мелкими чанками. В итоге вы соберёте один класс `LogAnalyzer`, использующий все четыре инструмента урока, и убедитесь, что он корректно отменяется по токену и не портит данные при гонках.

#### Что нужно сделать (пошагово)
1. Создайте консольный проект: `dotnet new console -n LogAnalyzer -o LogAnalyzer --framework net8.0`. Перейдите в папку: `cd LogAnalyzer`. Убедитесь, что в `LogAnalyzer.csproj` стоит `<LangVersion>12</LangVersion>` и `<Nullable>enable</Nullable>`.
2. Добавьте файл `LogEntry.cs` с `readonly record struct LogEntry(DateTime Timestamp, string Service, LogLevel Level, string Message)` и `enum LogLevel { Info, Warn, Error }`.
3. Добавьте `LogParser.cs` со статическим методом `Parse(string line)`, который через `line.Split('|')` собирает `LogEntry`. Некорректные строки должны бросать `FormatException` — это пригодится для тестов.
4. Добавьте `LogAnalyzer.cs` с классом `LogAnalyzer`. В нём реализуйте четыре метода, каждый — на отдельном инструменте урока:
   - `CountByLevel(LogEntry[] logs, CancellationToken ct)` через `Parallel.ForEach` + `ConcurrentDictionary<LogLevel,int>` (`AddOrUpdate`). Возвращает словарь «уровень → количество».
   - `SumMessageHashes(LogEntry[] logs, CancellationToken ct)` через `Parallel.For` с `localInit`/`localFinally`. Считает сумму `unchecked((long)string.GetHashCode(Message))` без `lock`. Должен работать в разы быстрее, чем версия с внешним `lock`.
   - `ComputeStats(LogEntry[] logs, CancellationToken ct)` через `Parallel.Invoke` с тремя независимыми делегатами: максимальная длина сообщения, количество уникальных сервисов (через `HashSet<string>` под локом), доля ошибок (`Error / total`). Используйте `ParallelOptions.MaxDegreeOfParallelism = 3`.
   - `ProcessUneven(LogEntry[] logs, CancellationToken ct)` через `Partitioner.Create(0, logs.Length, rangeSize: 64)` + `Parallel.ForEach`. Внутри тела имитируйте неравномерную нагрузку: чем длиннее `Message`, тем дольше «работа» (цикл `for (int j=0; j< Message.Length; j++) {/* spin */}`). Верните `TimeSpan` общего времени.
5. В `Program.cs` через top-level statements сгенерируйте тестовый массив: 200 тысяч `LogEntry` со случайными сервисами (`svc-a`, `svc-b`, `svc-c`), уровнями (`Info` 70 %, `Warn` 20 %, `Error` 10 %) и сообщениями длины от 5 до 500 символов.
6. Запустите все четыре метода и выведите результаты в `Console`: счётчики по уровням, сумму хешей, статистику, время `ProcessUneven`. Пример ожидаемого вывода:
   ```
   INFO=140012 WARN=39987 ERROR=60001
   SumHashes=48271936582031
   MaxLen=500 UniqueServices=3 ErrorRatio=0.300005
   UnevenMs=412
   ```
7. Добавьте xUnit-проект: `dotnet new xunit -n LogAnalyzer.Tests -o LogAnalyzer.Tests`, добавьте ссылку `dotnet add LogAnalyzer.Tests reference LogAnalyzer`.
8. Напишите тест `Cancel_After_100ms_Throws_OCE`: запустите `SumMessageHashes` с `CancellationTokenSource(TimeSpan.FromMilliseconds(100))` и убедитесь, что ловите `OperationCanceledException` за ≤ 200 мс.
9. Напишите тест `SumHashes_Matches_Sequential`: сравните результат `SumMessageHashes` с прямым `foreach` — они должны совпасть (доказывает отсутствие race).
10. Запустите `dotnet test` — все тесты зелёные. Затем `dotnet run --project LogAnalyzer` — вывод соответствует ожидаемому.
11. (Опционально) Снимите метрику: временная версия `SumMessageHashes` с внешним `lock` вместо `localFinally` — сравните время. Запишите разницу в комментариях.

#### Требования к решению
- Целевой рантайм — .NET 8, язык C# 12. Разрешены top-level statements, `record struct`, collection expressions, `init`-свойства, file-scoped namespaces, pattern matching (`switch` выражения).
- Все четыре метода `LogAnalyzer` должны принимать `CancellationToken` через `ParallelOptions.CancellationToken` и реагировать на отмену за ≤ 1 итерацию. Никаких «долгих» тел без проверки токена не допускается.
- `SumMessageHashes` обязан использовать перегрузку `Parallel.For` с `localInit`/`localFinally` и `Interlocked.Add` в `localFinally`. Внешний `lock` в теле цикла запрещён (только в эталонной «медленной» версии для сравнения).
- `CountByLevel` должен использовать `ConcurrentDictionary.AddOrUpdate` без внешней блокировки. Мутировать общий `Dictionary<,>` из тела запрещено — это даёт порчу данных.
- `ComputeStats` через `Parallel.Invoke` с `MaxDegreeOfParallelism = 3`. Каждая лямбда сначала считает локальный результат и лишь затем под локом пишет в общие поля — составная операция под `lock`.
- `ProcessUneven` обязан применять `Partitioner.Create(0, logs.Length, 64)`. Демонстрируется, что мелкие чанки лучше балансируют неравномерную нагрузку, чем дефолтный partitioner.
- Запрещено: `.Result`/`.Wait()` на async-операциях внутри тел `Parallel`, `async void`, `Parallel.ForEach` над отложенным `IQueryable` (коллекция должна быть материализована в массив). Для async-работы — `Task.WhenAll`/`Channel`, не `Parallel`.
- Код должен компилироваться без предупреждений (`TreatWarningsAsErrors` опционально, но приветствуется).

#### Тонкости и подводные камни
- **Race в общем аккумуляторе.** Запись `sum += x` в теле `Parallel.For` без `Interlocked`/`lock`/thread-local — главный враг урока. Два воркера читают одно значение, оба прибавляют, один перетирает. Решение — `localInit: () => 0L`, тело обновляет `localSum`, `localFinally: local => Interlocked.Add(ref total, local)`. Блокировка лишь в финале, один раз на воркер — отсюда ускорение в разы.
- **Thread-pool starvation.** Если вызвать `.Result` на `Task` внутри тела `Parallel.For`, поток пула блокируется. При насыщении ThreadPool новые воркеры не создаются мгновенно (пул растёт медленно, по ~1 потоку/0.5 с), и приложение «зависает». Async-работа — только через `Task.WhenAll`/`Channel`.
- **`loopState.Stop()` vs `Break()`.** `Stop()` = немедленная остановка всех воркеров, остальные итерации не запускаются. `Break()` = «доработать итерации с меньшими индексами и остановиться» — для упорядоченного режима. Если нужен мгновенный стоп (например, по отмене или при превышении лимита) — берите `Stop`.
- **Материализация `IEnumerable`.** `Parallel.ForEach` над `IQueryable`/отложенным LINQ заставит каждого воркера перевыполнить запрос. Всегда `ToList()`/`ToArray()` заранее. В задании вход уже `LogEntry[]` — это намеренно.
- **`MaxDegreeOfParallelism` для I/O.** Если тело обращается к диску/сети/БД, `Environment.ProcessorCount` — ошибка: потоки простаивают в ожидании I/O. Для I/O используйте `SemaphoreSlim` + async, не `Parallel`. `Parallel` — для CPU-bound.
- **`Partitioner.Create` и `NoBuffering`.** Дефолтный partitioner делает чанки адаптивного размера. При неравномерной нагрузке (короткие строки + длинные) один воркер «застрянет» на длинных. `Partitioner.Create(0, N, rangeSize)` делит диапазон на мелкие чанки — баланс лучше, overhead выше. `EnumerablePartitionerOptions.NoBuffering` полезен для `IEnumerable<T>`.
- **`ConcurrentDictionary.AddOrUpdate` не атомарен целиком.** Фабрика обновления может вызваться несколько раз под конкуренцией — но результат всегда консистентен. Не кладите туда side-effects с внешним состоянием.
- **Отмена проверяется между итерациями.** TPL сам бросит `OperationCanceledException`, но внутри длинной итерации токен не проверяется автоматически — добавьте `ct.ThrowIfCancellationRequested()` в долгий внутренний цикл `ProcessUneven`.

#### Критерии приёмки
- [ ] Проект `LogAnalyzer` собирается под .NET 8 / C# 12 без ошибок и предупреждений.
- [ ] `LogEntry` — `readonly record struct`, `LogLevel` — `enum` с тремя значениями.
- [ ] `CountByLevel` использует `Parallel.ForEach` + `ConcurrentDictionary.AddOrUpdate`, без внешнего `lock`.
- [ ] `SumMessageHashes` использует `Parallel.For` с `localInit`/`localFinally` и `Interlocked.Add`, без `lock` в теле.
- [ ] `ComputeStats` использует `Parallel.Invoke` с `MaxDegreeOfParallelism = 3` и `lock` только для финальной записи.
- [ ] `ProcessUneven` использует `Partitioner.Create(0, logs.Length, 64)` и `Parallel.ForEach` по чанкам.
- [ ] Во все методы передан `ParallelOptions.CancellationToken`; отмена срабатывает за ≤ 1 итерацию.
- [ ] Внутри `ProcessUneven` есть `ct.ThrowIfCancellationRequested()` в горячем внутреннем цикле.
- [ ] Нет `.Result`/`.Wait()`/`async void` внутри тел `Parallel`.
- [ ] Тест `Cancel_After_100ms_Throws_OCE` ловит `OperationCanceledException` за ≤ 200 мс.
- [ ] Тест `SumHashes_Matches_Sequential` подтверждает совпадение с последовательной версией (доказывает отсутствие race).
- [ ] `dotnet test` — зелёный; `dotnet run` — вывод соответствует ожидаемому формату.
- [ ] В комментариях отмечена разница во времени между `localFinally` и внешним `lock` (если сделан бонус-пункт).

#### Подсказки (без прямого ответа)
- Для `Parallel.For` с агрегацией ищите перегрузку с сигнатурой `(..., ParallelOptions, Func<TLocal>, Func<int, ParallelLoopState, TLocal, TLocal>, Action<TLocal>)` — это и есть `localInit`/`body`/`localFinally`.
- `string.GetHashCode` возвращает `int`; для суммы используйте `unchecked((long)...)`, чтобы не получить `OverflowException` при больших массивах.
- `ConcurrentDictionary.AddOrUpdate(key, addValueFactory, updateValueFactory)` — фабрики вызываются под локом ключа, но не кладите туда тяжёлую логику.
- В `Parallel.Invoke` каждый делегат сначала вычисляет локальный результат (без блокировки), а `lock` берётся только на финальную запись — это и есть «короткая критическая секция для составной операции».
- Для `Partitioner.Create(0, N, rangeSize)` тело получает `Tuple<int,int>` (или `(int, int)`) — границы чанка.
- Чтобы сгенерировать тестовые данные, используйте `Random.Shared.Next(...)` и `new string('x', length)`.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — LogAnalyzer: Parallel.For/ForEach/Invoke + Partitioner.
// Полностью потокобезопасно, без .Result, с отменой.

using System.Collections.Concurrent;
using System.Diagnostics;

namespace LogAnalyzer;

public enum LogLevel { Info, Warn, Error }

public readonly record struct LogEntry(DateTime Timestamp, string Service, LogLevel Level, string Message);

public static class LogAnalyzer
{
    // 1) Parallel.ForEach + ConcurrentDictionary — счётчик по уровням.
    //    ConcurrentDictionary сам потокобезопасен, внешний lock не нужен.
    public static Dictionary<LogLevel, int> CountByLevel(LogEntry[] logs, CancellationToken ct)
    {
        var counts = new ConcurrentDictionary<LogLevel, int>();
        var options = new ParallelOptions
        {
            MaxDegreeOfParallelism = Environment.ProcessorCount,
            CancellationToken = ct,
        };

        Parallel.ForEach(logs, options, entry =>
        {
            // AddOrUpdate: фабрика добавления = 1, обновления = current + 1.
            counts.AddOrUpdate(entry.Level, _ => 1, (_, current) => current + 1);
        });

        return new Dictionary<LogLevel, int>(counts);
    }

    // 2) Parallel.For с localInit/localFinally — безблокировочная сумма хешей.
    public static long SumMessageHashes(LogEntry[] logs, CancellationToken ct)
    {
        long total = 0;
        var options = new ParallelOptions
        {
            MaxDegreeOfParallelism = Environment.ProcessorCount,
            CancellationToken = ct,
        };

        Parallel.For(
            0, logs.Length,
            options,
            localInit: () => 0L,                                   // локальная сумма на воркер
            body: (i, loopState, localSum) =>
            {
                // unchecked: не бросаем OverflowException при больших суммах.
                long hash = unchecked((long)logs[i].Message.GetHashCode());
                localSum += hash;
                return localSum;                                   // пробрасываем дальше
            },
            localFinally: localSum => Interlocked.Add(ref total, localSum)); // слияние один раз на воркер

        return total;
    }

    // 3) Parallel.Invoke — три независимых CPU-bound делегата, MaxDegreeOfParallelism = 3.
    public static (int MaxLen, int UniqueServices, double ErrorRatio) ComputeStats(
        LogEntry[] logs, CancellationToken ct)
    {
        int maxLen = 0;
        var services = new HashSet<string>();
        long errorCount = 0;
        object gate = new();

        var options = new ParallelOptions
        {
            MaxDegreeOfParallelism = 3,
            CancellationToken = ct,
        };

        Parallel.Invoke(
            options,
            () =>
            {
                int local = 0;
                for (int i = 0; i < logs.Length; i++)
                    if (logs[i].Message.Length > local) local = logs[i].Message.Length;
                lock (gate) { if (local > maxLen) maxLen = local; }
            },
            () =>
            {
                var localSet = new HashSet<string>();
                for (int i = 0; i < logs.Length; i++) localSet.Add(logs[i].Service);
                lock (gate) { foreach (var s in localSet) services.Add(s); }
            },
            () =>
            {
                long localErrors = 0;
                for (int i = 0; i < logs.Length; i++)
                    if (logs[i].Level == LogLevel.Error) localErrors++;
                lock (gate) { errorCount += localErrors; }
            });

        return (maxLen, services.Count, (double)errorCount / logs.Length);
    }

    // 4) Partitioner.Create для неравномерной нагрузки — мелкие чанки по 64.
    public static TimeSpan ProcessUneven(LogEntry[] logs, CancellationToken ct)
    {
        var partitioner = Partitioner.Create(0, logs.Length, rangeSize: 64);
        var options = new ParallelOptions
        {
            MaxDegreeOfParallelism = Environment.ProcessorCount,
            CancellationToken = ct,
        };
        var sw = Stopwatch.StartNew();

        Parallel.ForEach(partitioner, options, range =>
        {
            (int from, int to) = range;          // распаковываем Tuple<int,int>
            for (int i = from; i < to; i++)
            {
                ct.ThrowIfCancellationRequested();  // явная проверка в горячем цикле
                int len = logs[i].Message.Length;
                for (int j = 0; j < len; j++) { /* CPU spin, пропорциональный длине */ }
            }
        });

        sw.Stop();
        return sw.Elapsed;
    }
}
```

Разбор по строкам. `CountByLevel` использует `Parallel.ForEach` над массивом — массив уже материализован, поэтому каждый воркер не перевыполняет запрос (частая ошибка урока с `IQueryable`). `ConcurrentDictionary.AddOrUpdate` инкапсулирует потокобезопасное обновление: внешний `lock` не нужен, и композитная операция остаётся консистентной. В `SumMessageHashes` применена перегрузка `Parallel.For` с тремя делегатами: `localInit` создаёт `0L` на старте воркера, `body` накапливает `localSum` без блокировок, `localFinally` один раз сливает результат в `total` через `Interlocked.Add`. Это и есть «самый быстрый путь» из best practices урока: блокировка только на слиянии, в разы быстрее внешнего `lock` в теле. `unchecked((long)...)` защищает от `OverflowException` — сумма хешей 200 тысяч строк легко переполняет `int`, а `unchecked` переключается в режим без проверки. `ComputeStats` построен на `Parallel.Invoke` с `MaxDegreeOfParallelism = 3`: три независимых вычисления идут параллельно, каждое сначала считает локальный результат (без блокировки), а `lock (gate)` берётся лишь на финальную запись в общие поля — короткая критическая секция. Это иллюстрирует правило «`lock` для составных операций». `ProcessUneven` — применение `Partitioner.Create(0, logs.Length, 64)`: диапазон делится на чанки по 64 элемента, и каждый чанк становится отдельной единицей работы для TPL. При неравномерной нагрузке (сообщения разной длины дают разное время парсинга) мелкие чанки балансируют лучше, чем дефолтный адаптивный partitioner — один воркер не «застрянет» на пачке длинных строк. `ct.ThrowIfCancellationRequested()` внутри внутреннего цикла — критично: TPL проверяет токен лишь между итерациями `Parallel.ForEach`, а внутренний `for` по чанку длинный, без явной проверки отмена запоздает. Обратите внимание: нигде нет `.Result`/`.Wait()` — это намеренно, чтобы избежать thread-pool starvation; если бы работа была async, мы бы перешли на `Task.WhenAll`/`Channel` (урок M11-L07), а не на `Parallel`.

#### Задания на углубление (бонус)
1. Реализуйте «медленную» версию `SumMessageHashesSlow` с внешним `lock (gate) { total += ...; }` в теле. Замерьте разницу во времени на 1 миллионе записей и объясните её в комментарии.
2. Добавьте `loopState.Stop()` в `SumMessageHashes`: если `localSum` превышает порог (например, `long.MaxValue / 2`), немедленно остановите весь цикл. Сравните поведение `Stop()` и `Break()` в тесте.
3. Реализуйте `TopErrorsByService` через PLINQ: `logs.AsParallel().Where(l => l.Level == LogLevel.Error).GroupBy(l => l.Service).Select(g => (g.Key, g.Count()))`. Сравните читаемость и скорость с `Parallel.ForEach`-версией.
4. Подключите `ParallelOptions.TaskScheduler = TaskScheduler.FromCurrentSynchronizationContext()` (в WPF/WinForms-обёртке) и опишите, зачем это нужно при интеграции с UI-потоком.

---

## Statement in English / Постановка на английском

#### Context & motivation
You are writing a `LogAnalyzer` module for a monitoring service that receives a batch of raw logs from dozens of microservices every minute. Each log is a line such as `2024-05-12T10:13:45Z|svc-orders|ERROR|order #42 failed: timeout` (ISO timestamp, service name, level, message). A minute can bring anywhere from 50 thousand to 5 million lines, and the service must produce an aggregated report faster than the next batch arrives. Profiling shows the bottleneck is CPU: parsing strings, computing message hashes, sorting, building histograms. I/O is already asynchronous (covered in the channels lesson M11-L07), so what you need here is a data-parallel CPU pipeline built on `System.Threading.Tasks.Parallel`.

You cannot afford to spawn `Task.Run` manually — the coordination cost is too high, and a stray `.Result` inside the body risks thread-pool starvation. The `Parallel` class takes over chunking and an implicit `Barrier` at the end of the loop, so you describe "what to do with a chunk of data" and the TPL decides "how to hand chunks to pool threads". You must also drive the three levers of `ParallelOptions` explicitly: `MaxDegreeOfParallelism`, `CancellationToken`, and (optionally) `TaskScheduler`. Aggregations deserve special care: a naive `sum += x` in a shared body produces a race condition and an unpredictable result. The correct path is a thread-local accumulator through the `Parallel.For`/`Parallel.ForEach` overloads with `localInit`/`localFinally`, where each worker accumulates a private sum without locks and `localFinally` merges it into the shared total via `Interlocked.Add`. Some of the work is also uneven (long messages parse slower than short ones), and there you need `Partitioner.Create` with small chunks. In the end you will assemble a single `LogAnalyzer` class that uses all four tools from the lesson and verify that it cancels cleanly on a token and does not corrupt data under races.

#### What to do (step by step)
1. Create a console project: `dotnet new console -n LogAnalyzer -o LogAnalyzer --framework net8.0`. Enter the folder: `cd LogAnalyzer`. Make sure `LogAnalyzer.csproj` has `<LangVersion>12</LangVersion>` and `<Nullable>enable</Nullable>`.
2. Add `LogEntry.cs` with `readonly record struct LogEntry(DateTime Timestamp, string Service, LogLevel Level, string Message)` and `enum LogLevel { Info, Warn, Error }`.
3. Add `LogParser.cs` with a static `Parse(string line)` method that splits on `|` via `line.Split('|')` and builds a `LogEntry`. Malformed lines must throw `FormatException` — useful for tests.
4. Add `LogAnalyzer.cs` with the `LogAnalyzer` class. Implement four methods, each on a distinct tool from the lesson:
   - `CountByLevel(LogEntry[] logs, CancellationToken ct)` via `Parallel.ForEach` + `ConcurrentDictionary<LogLevel,int>` (`AddOrUpdate`). Returns a "level → count" dictionary.
   - `SumMessageHashes(LogEntry[] logs, CancellationToken ct)` via `Parallel.For` with `localInit`/`localFinally`. Computes the sum of `unchecked((long)string.GetHashCode(Message))` without `lock`. Should be several times faster than a version with an external `lock`.
   - `ComputeStats(LogEntry[] logs, CancellationToken ct)` via `Parallel.Invoke` with three independent delegates: max message length, count of unique services (via `HashSet<string>` under a lock), error ratio (`Error / total`). Use `ParallelOptions.MaxDegreeOfParallelism = 3`.
   - `ProcessUneven(LogEntry[] logs, CancellationToken ct)` via `Partitioner.Create(0, logs.Length, rangeSize: 64)` + `Parallel.ForEach`. Inside the body simulate uneven load: the longer the `Message`, the longer the "work" (`for (int j=0; j< Message.Length; j++) {/* spin */}`). Return the overall `TimeSpan`.
5. In `Program.cs` using top-level statements, generate a test array: 200 thousand `LogEntry` with random services (`svc-a`, `svc-b`, `svc-c`), levels (`Info` 70 %, `Warn` 20 %, `Error` 10 %) and messages from 5 to 500 characters.
6. Run all four methods and print results to `Console`: counts per level, sum of hashes, statistics, `ProcessUneven` time. Expected output format:
   ```
   INFO=140012 WARN=39987 ERROR=60001
   SumHashes=48271936582031
   MaxLen=500 UniqueServices=3 ErrorRatio=0.300005
   UnevenMs=412
   ```
7. Add an xUnit project: `dotnet new xunit -n LogAnalyzer.Tests -o LogAnalyzer.Tests`, then `dotnet add LogAnalyzer.Tests reference LogAnalyzer`.
8. Write a test `Cancel_After_100ms_Throws_OCE`: run `SumMessageHashes` with `CancellationTokenSource(TimeSpan.FromMilliseconds(100))` and assert you catch `OperationCanceledException` within ≤ 200 ms.
9. Write a test `SumHashes_Matches_Sequential`: compare the `SumMessageHashes` result to a plain `foreach` — they must match (proof that there is no race).
10. Run `dotnet test` — all green. Then `dotnet run --project LogAnalyzer` — output matches the expected format.
11. (Optional) Measure: temporarily rewrite `SumMessageHashes` with an external `lock` instead of `localFinally` and compare timings. Note the difference in comments.

#### Requirements
- Target runtime is .NET 8, language C# 12. Top-level statements, `record struct`, collection expressions, `init` properties, file-scoped namespaces, pattern matching (`switch` expressions) are all allowed.
- All four `LogAnalyzer` methods must accept a `CancellationToken` through `ParallelOptions.CancellationToken` and respond to cancellation within ≤ 1 iteration. Long bodies without a token check are not accepted.
- `SumMessageHashes` must use the `Parallel.For` overload with `localInit`/`localFinally` and `Interlocked.Add` in `localFinally`. An external `lock` in the loop body is forbidden (only allowed in the reference "slow" version for comparison).
- `CountByLevel` must use `ConcurrentDictionary.AddOrUpdate` with no external lock. Mutating a shared `Dictionary<,>` from the body is forbidden — it corrupts data.
- `ComputeStats` through `Parallel.Invoke` with `MaxDegreeOfParallelism = 3`. Each lambda first computes a local result, then writes to shared fields under a `lock` — a short critical section for a compound operation.
- `ProcessUneven` must use `Partitioner.Create(0, logs.Length, 64)`. The intent is to show that small chunks balance uneven workloads better than the default partitioner.
- Forbidden: `.Result`/`.Wait()` on async operations inside `Parallel` bodies, `async void`, `Parallel.ForEach` over a deferred `IQueryable` (the collection must be materialized into an array). For async work use `Task.WhenAll`/`Channel`, not `Parallel`.
- The code should compile without warnings (`TreatWarningsAsErrors` optional but welcome).

#### Pitfalls
- **Race in a shared accumulator.** Writing `sum += x` in a `Parallel.For` body without `Interlocked`/`lock`/thread-local is the chief enemy of the lesson. Two workers read the same value, both add, one overwrites. The fix is `localInit: () => 0L`, a body that updates `localSum`, and `localFinally: local => Interlocked.Add(ref total, local)`. The lock appears only in the final merge, once per worker — hence the multi-fold speedup.
- **Thread-pool starvation.** Calling `.Result` on a `Task` inside a `Parallel.For` body blocks a pool thread. Once the ThreadPool is saturated, new workers are not created instantly (the pool grows slowly, about 1 thread per 0.5 s), and the application appears to hang. Async work must go through `Task.WhenAll`/`Channel`.
- **`loopState.Stop()` vs `Break()`.** `Stop()` = immediate stop of all workers, remaining iterations are not started. `Break()` = "finish iterations with lower indices, then stop" for ordered mode. For an instant stop (cancellation, threshold exceeded) use `Stop`.
- **Materializing `IEnumerable`.** `Parallel.ForEach` over an `IQueryable`/deferred LINQ forces every worker to re-run the query. Always `ToList()`/`ToArray()` first. Here the input is already `LogEntry[]` — intentionally.
- **`MaxDegreeOfParallelism` for I/O.** If the body touches disk/network/DB, `Environment.ProcessorCount` is wrong: threads idle on I/O. For I/O use `SemaphoreSlim` + async, not `Parallel`. `Parallel` is for CPU-bound work.
- **`Partitioner.Create` and `NoBuffering`.** The default partitioner makes adaptively sized chunks. Under uneven load (short strings + long strings) one worker stalls on the long ones. `Partitioner.Create(0, N, rangeSize)` splits the range into small chunks — better balance, higher overhead. `EnumerablePartitionerOptions.NoBuffering` is useful for `IEnumerable<T>`.
- **`ConcurrentDictionary.AddOrUpdate` is not atomic as a whole.** The update factory may run more than once under contention — but the result is always consistent. Do not place side effects with external state inside it.
- **Cancellation is checked between iterations.** The TPL itself throws `OperationCanceledException`, but inside a long iteration the token is not checked automatically — add `ct.ThrowIfCancellationRequested()` to the inner hot loop of `ProcessUneven`.

#### Acceptance criteria
- [ ] The `LogAnalyzer` project builds under .NET 8 / C# 12 with no errors or warnings.
- [ ] `LogEntry` is a `readonly record struct`, `LogLevel` is an `enum` with three values.
- [ ] `CountByLevel` uses `Parallel.ForEach` + `ConcurrentDictionary.AddOrUpdate`, with no external `lock`.
- [ ] `SumMessageHashes` uses `Parallel.For` with `localInit`/`localFinally` and `Interlocked.Add`, with no `lock` in the body.
- [ ] `ComputeStats` uses `Parallel.Invoke` with `MaxDegreeOfParallelism = 3` and `lock` only for the final write.
- [ ] `ProcessUneven` uses `Partitioner.Create(0, logs.Length, 64)` and `Parallel.ForEach` over chunks.
- [ ] All methods receive `ParallelOptions.CancellationToken`; cancellation triggers within ≤ 1 iteration.
- [ ] `ProcessUneven` calls `ct.ThrowIfCancellationRequested()` inside the hot inner loop.
- [ ] No `.Result`/`.Wait()`/`async void` inside `Parallel` bodies.
- [ ] The test `Cancel_After_100ms_Throws_OCE` catches `OperationCanceledException` within ≤ 200 ms.
- [ ] The test `SumHashes_Matches_Sequential` confirms equality with the sequential version (proof of no race).
- [ ] `dotnet test` is green; `dotnet run` output matches the expected format.
- [ ] Comments note the timing difference between `localFinally` and an external `lock` (if the bonus was done).

#### Hints (no direct answer)
- For the aggregation overload of `Parallel.For`, look for the signature `(..., ParallelOptions, Func<TLocal>, Func<int, ParallelLoopState, TLocal, TLocal>, Action<TLocal>)` — these are `localInit`/`body`/`localFinally`.
- `string.GetHashCode` returns `int`; for the sum use `unchecked((long)...)` to avoid `OverflowException` on large arrays.
- `ConcurrentDictionary.AddOrUpdate(key, addValueFactory, updateValueFactory)` — factories run under a per-key lock, but do not put heavy logic there.
- In `Parallel.Invoke`, each delegate first computes a local result (without locking), then takes a `lock` only for the final write — that is the "short critical section for a compound operation".
- For `Partitioner.Create(0, N, rangeSize)`, the body receives a `Tuple<int,int>` (or `(int, int)`) — the chunk bounds.
- To generate test data, use `Random.Shared.Next(...)` and `new string('x', length)`.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — LogAnalyzer: Parallel.For/ForEach/Invoke + Partitioner.
// Fully thread-safe, no .Result, with cancellation.

using System.Collections.Concurrent;
using System.Diagnostics;

namespace LogAnalyzer;

public enum LogLevel { Info, Warn, Error }

public readonly record struct LogEntry(DateTime Timestamp, string Service, LogLevel Level, string Message);

public static class LogAnalyzer
{
    // 1) Parallel.ForEach + ConcurrentDictionary — count per level.
    //    ConcurrentDictionary is itself thread-safe, no external lock needed.
    public static Dictionary<LogLevel, int> CountByLevel(LogEntry[] logs, CancellationToken ct)
    {
        var counts = new ConcurrentDictionary<LogLevel, int>();
        var options = new ParallelOptions
        {
            MaxDegreeOfParallelism = Environment.ProcessorCount,
            CancellationToken = ct,
        };

        Parallel.ForEach(logs, options, entry =>
        {
            // AddOrUpdate: add factory = 1, update factory = current + 1.
            counts.AddOrUpdate(entry.Level, _ => 1, (_, current) => current + 1);
        });

        return new Dictionary<LogLevel, int>(counts);
    }

    // 2) Parallel.For with localInit/localFinally — lock-free sum of hashes.
    public static long SumMessageHashes(LogEntry[] logs, CancellationToken ct)
    {
        long total = 0;
        var options = new ParallelOptions
        {
            MaxDegreeOfParallelism = Environment.ProcessorCount,
            CancellationToken = ct,
        };

        Parallel.For(
            0, logs.Length,
            options,
            localInit: () => 0L,                                   // per-worker local sum
            body: (i, loopState, localSum) =>
            {
                // unchecked: do not throw OverflowException on large sums.
                long hash = unchecked((long)logs[i].Message.GetHashCode());
                localSum += hash;
                return localSum;                                   // pass it along
            },
            localFinally: localSum => Interlocked.Add(ref total, localSum)); // merge once per worker

        return total;
    }

    // 3) Parallel.Invoke — three independent CPU-bound delegates, MaxDegreeOfParallelism = 3.
    public static (int MaxLen, int UniqueServices, double ErrorRatio) ComputeStats(
        LogEntry[] logs, CancellationToken ct)
    {
        int maxLen = 0;
        var services = new HashSet<string>();
        long errorCount = 0;
        object gate = new();

        var options = new ParallelOptions
        {
            MaxDegreeOfParallelism = 3,
            CancellationToken = ct,
        };

        Parallel.Invoke(
            options,
            () =>
            {
                int local = 0;
                for (int i = 0; i < logs.Length; i++)
                    if (logs[i].Message.Length > local) local = logs[i].Message.Length;
                lock (gate) { if (local > maxLen) maxLen = local; }
            },
            () =>
            {
                var localSet = new HashSet<string>();
                for (int i = 0; i < logs.Length; i++) localSet.Add(logs[i].Service);
                lock (gate) { foreach (var s in localSet) services.Add(s); }
            },
            () =>
            {
                long localErrors = 0;
                for (int i = 0; i < logs.Length; i++)
                    if (logs[i].Level == LogLevel.Error) localErrors++;
                lock (gate) { errorCount += localErrors; }
            });

        return (maxLen, services.Count, (double)errorCount / logs.Length);
    }

    // 4) Partitioner.Create for uneven load — small chunks of 64.
    public static TimeSpan ProcessUneven(LogEntry[] logs, CancellationToken ct)
    {
        var partitioner = Partitioner.Create(0, logs.Length, rangeSize: 64);
        var options = new ParallelOptions
        {
            MaxDegreeOfParallelism = Environment.ProcessorCount,
            CancellationToken = ct,
        };
        var sw = Stopwatch.StartNew();

        Parallel.ForEach(partitioner, options, range =>
        {
            (int from, int to) = range;          // unpack Tuple<int,int>
            for (int i = from; i < to; i++)
            {
                ct.ThrowIfCancellationRequested();  // explicit check in the hot loop
                int len = logs[i].Message.Length;
                for (int j = 0; j < len; j++) { /* CPU spin, proportional to length */ }
            }
        });

        sw.Stop();
        return sw.Elapsed;
    }
}
```

Line-by-line walk-through. `CountByLevel` uses `Parallel.ForEach` over an array — the array is already materialized, so no worker re-runs a query (the common `IQueryable` mistake from the lesson). `ConcurrentDictionary.AddOrUpdate` encapsulates thread-safe updates: no external `lock` is needed and the compound operation stays consistent. In `SumMessageHashes`, the `Parallel.For` overload with three delegates is applied: `localInit` creates `0L` per worker, `body` accumulates `localSum` without locks, and `localFinally` merges it into `total` once via `Interlocked.Add`. This is exactly the "fastest path" from the lesson best practices: the lock only appears at the merge, several times faster than an external `lock` in the body. `unchecked((long)...)` guards against `OverflowException` — the sum of hashes over 200 thousand strings easily overflows `int`, and `unchecked` switches to no-check mode. `ComputeStats` is built on `Parallel.Invoke` with `MaxDegreeOfParallelism = 3`: three independent computations run in parallel, each first computes a local result (without locking), and `lock (gate)` is taken only for the final write to shared fields — a short critical section. This illustrates the "`lock` for compound operations" rule. `ProcessUneven` applies `Partitioner.Create(0, logs.Length, 64)`: the range is split into chunks of 64, and each chunk becomes a separate unit of work for the TPL. Under uneven load (messages of different lengths produce different parse times) small chunks balance better than the default adaptive partitioner — a single worker cannot get stuck behind a batch of long strings. `ct.ThrowIfCancellationRequested()` inside the inner loop is critical: the TPL checks the token only between `Parallel.ForEach` iterations, while the inner `for` over a chunk is long, so without an explicit check cancellation is delayed. Note that there is no `.Result`/`.Wait()` anywhere — intentional, to avoid thread-pool starvation; if the work were async we would switch to `Task.WhenAll`/`Channel` (lesson M11-L07), not `Parallel`.

#### Going deeper (bonus)
1. Implement a "slow" `SumMessageHashesSlow` with an external `lock (gate) { total += ...; }` in the body. Measure the time difference on 1 million records and explain it in a comment.
2. Add `loopState.Stop()` to `SumMessageHashes`: if `localSum` exceeds a threshold (e.g. `long.MaxValue / 2`), stop the whole loop immediately. Compare `Stop()` and `Break()` in a test.
3. Implement `TopErrorsByService` with PLINQ: `logs.AsParallel().Where(l => l.Level == LogLevel.Error).GroupBy(l => l.Service).Select(g => (g.Key, g.Count()))`. Compare readability and speed with a `Parallel.ForEach` version.
4. Wire up `ParallelOptions.TaskScheduler = TaskScheduler.FromCurrentSynchronizationContext()` (in a WPF/WinForms host) and describe why it matters for UI-thread integration.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `LogAnalyzer` собирается под .NET 8 / C# 12 без предупреждений.
- [ ] Все четыре метода реализованы на указанных инструментах (`Parallel.ForEach`, `Parallel.For` + `localInit`/`localFinally`, `Parallel.Invoke`, `Partitioner.Create`).
- [ ] `CancellationToken` передан во все методы и реагирует за ≤ 1 итерацию.
- [ ] Нет `.Result`/`.Wait()`/`async void` внутри тел `Parallel`.
- [ ] Тесты `Cancel_After_100ms_Throws_OCE` и `SumHashes_Matches_Sequential` зелёные.
- [ ] `dotnet run` выводит результаты в ожидаемом формате.
- [ ] The `LogAnalyzer` project builds under .NET 8 / C# 12 without warnings.
- [ ] All four methods use the specified tools (`Parallel.ForEach`, `Parallel.For` + `localInit`/`localFinally`, `Parallel.Invoke`, `Partitioner.Create`).
- [ ] A `CancellationToken` is passed to every method and responds within ≤ 1 iteration.
- [ ] No `.Result`/`.Wait()`/`async void` inside `Parallel` bodies.
- [ ] The tests `Cancel_After_100ms_Throws_OCE` and `SumHashes_Matches_Sequential` are green.
- [ ] `dotnet run` prints results in the expected format.

#### Ресурсы / Resources
- [Microsoft Learn — How to: Write a Simple Parallel.For Loop](https://learn.microsoft.com/dotnet/standard/parallel-programming/how-to-write-a-simple-parallel-for-loop)
- [Microsoft Learn — How to: Write a Simple Parallel.ForEach Loop](https://learn.microsoft.com/dotnet/standard/parallel-programming/how-to-write-a-simple-parallel-foreach-loop)
- [Microsoft Learn — How to: Cancel a Parallel.For or ForEach Loop](https://learn.microsoft.com/dotnet/standard/parallel-programming/how-to-cancel-a-parallel-for-or-foreach-loop)
- [Microsoft Learn — Partitioner.Create](https://learn.microsoft.com/dotnet/api/system.collections.concurrent.partitioner.create)
- [Microsoft Learn — PLINQ Overview](https://learn.microsoft.com/dotnet/standard/parallel-programming/parallel-linq-plinq)
- [Microsoft Learn — Parallel.Invoke](https://learn.microsoft.com/dotnet/api/system.threading.tasks.parallel.invoke)
