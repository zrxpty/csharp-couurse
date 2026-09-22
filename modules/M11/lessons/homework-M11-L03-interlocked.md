---
[← К уроку M11-L03](lesson-M11-L03-interlocked.md) | [⬆ К модулю M11](../README.md) | [Следующее ДЗ →](homework-M11-L04-semaphore-mutex-rwlock.md)
---

### Домашнее задание M11-L03: Interlocked, атомарные операции / Homework M11-L03: Interlocked, atomic operations

**Урок / Lesson:** M11-L03
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться применять `System.Threading.Interlocked` для потокобезопасных операций над счётчиками и ссылками, отличать атомарность от упорядочивания памяти, реализовывать lock-free паттерны (CAS-цикл, безопасная публикация объекта) и понимать границу, за которой `Interlocked` уступает место `lock` или `SemaphoreSlim`. (EN) Learn to apply `System.Threading.Interlocked` for thread-safe operations on counters and references, distinguish atomicity from memory ordering, implement lock-free patterns (CAS-loop, safe publication) and understand the boundary where `Interlocked` yields to `lock` or `SemaphoreSlim`.

#### Связь с уроком / Connection to the lesson
(RU) Урок вводит класс `Interlocked` как инструмент «точечной» синхронизации на уровне процессорных инструкций и демонстрирует разницу между `count++` (состояние гонки) и `Interlocked.Increment`. Это ДЗ закрепляет все основные методы (`Increment`, `Decrement`, `Add`, `Exchange`, `CompareExchange`, `MemoryBarrier`), паттерн CAS-цикла и эмпирическое правило «одно значение — `Interlocked`, составная логика — `lock`».
(EN) The lesson introduces `Interlocked` as a point-wise synchronization tool at the CPU-instruction level and shows the difference between `count++` (a race) and `Interlocked.Increment`. This homework reinforces all the core methods (`Increment`, `Decrement`, `Add`, `Exchange`, `CompareExchange`, `MemoryBarrier`), the CAS-loop pattern, and the rule of thumb "one value — `Interlocked`, compound logic — `lock`".

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы разрабатываете подсистему сбора метрик для высоконагруженного HTTP-сервиса на .NET 8. В каждую наносекунду сервис обрабатывает десятки параллельных запросов, и каждый запрос увеличивает сразу несколько счётчиков: общее число запросов, число ошибок, суммарный размер ответов в байтах, а также уникальный sequence-номер для корреляции логов. Кроме того, у сервиса есть общий кэш конфигурации, который лениво строится один раз из дорогого источника и затем переиздаётся по сигналу от администратора. Любая потерянное обновление счётчика искажает дашборд и ломает SLO-отчётность; любая гонка при инициализации кэша приводит к тому, что разные потоки видят разные версии конфигурации. В этой задаче запрещено использовать тяжёлые примитивы ядра — `lock` допускается только там, где без него действительно нельзя обойтись (составная операция над несколькими полями), а `await` внутри критической секции недопустим вовсе. Вам предстоит спроектировать набор потокобезопасных структур на `Interlocked`, корректно опубликовать кэш через `CompareExchange` и сравнить производительность lock-free счётчика с `lock`-версией под реальной нагрузкой, чтобы убедиться, что вы понимаете не только «как написать», но и «когда это выгодно». Это упражнение напрямую моделирует реальный код из урока: lock-free счётчик `LockFreeCounter`, ленивая ссылка `LazyAtomicReference<T>` и сравнение `Account` с двумя полями.

#### Что нужно сделать (пошагово)
1. Создайте новый проект консольного приложения: `dotnet new console -n M11L03.Homework -o M11L03.Homework --framework net8.0`. Откройте папку проекта и убедитесь, что в `M11L03.Homework.csproj` указано `<TargetFramework>net8.0</TargetFramework>` и `<LangVersion>latest</LangVersion>`. Удалите шаблонный `Program.cs` и создайте новый с top-level statements.
2. Реализуйте класс `MetricsCounter` с тремя полями: `long _requests`, `long _errors`, `long _bytes`. Публичные методы: `RegisterRequest(int responseBytes, bool isError)`, который атомарно увеличивает `_requests` на 1, `_bytes` на `responseBytes`, и, если `isError`, увеличивает `_errors` на 1. Подумайте: можно ли эти три инкремента сделать «тремя `Interlocked.Add`» и остаться корректным? Обоснуйте в комментарии.
3. Реализуйте `NextSequenceId()`, возвращающий монотонно возрастающий `long`, через `Interlocked.Increment(ref _sequence)`. Используйте именно `long`, а не `int`, чтобы не столкнуться с переполнением за сутки работы. В комментарии укажите, почему для `long` обязательно `Interlocked.Read`, а не простое чтение поля.
4. Реализуйте класс `AtomicConfigCache<T>` где `T : class`, который лениво строит конфигурацию один раз через переданную фабрику `Func<T>`. Используйте быстрый путь `Volatile.Read` и медленный путь через `Interlocked.CompareExchange(ref _value, created, null)`. Важно: фабрика может вызваться несколько раз у разных потоков, но в `_value` должен закрепиться ровно один объект — победитель CAS. Верните именно его, а не локальный `created`, если CAS проиграл.
5. Добавьте метод `Replace(T newValue)`, который атомарно заменяет текущую конфигурацию на новую через `Interlocked.Exchange` и возвращает предыдущее значение — это модель переиздания конфигурации администратором.
6. Реализуйте «неправильный» и «правильный» варианты подсчёта в статическом классе `RaceBenchmark`: `RunBroken(int threads, int perThread)` через `counter++` и `RunSafe` через `Interlocked.Increment`. Запустите оба с `threads = 16`, `perThread = 1_000_000` и выведите ожидаемое значение, фактическое «битое» и фактическое «безопасное». Битовое значение почти всегда меньше ожидаемого — объясните в выводе консоли, какие обновления теряются.
7. Реализуйте класс `TransferLog`, в котором две связанных операции — запись в баланс и инкремент счётчика транзакций — должны выполняться согласованно. Здесь, согласно уроку, используйте `lock`, потому что `Interlocked` работает с одним значением за раз. В комментарии явно укажите, почему `Interlocked` здесь не подходит, и что нельзя помещать `await` внутрь `lock`.
8. Добавьте micro-benchmark через `Parallel.For` с `threads = 8` и `perThread = 5_000_000`, который сравнивает время lock-free счётчика против `lock`-версии. Выведите оба времени в миллисекундах через `Stopwatch`. Сделайте вывод в комментарии: при какой нагрузке `Interlocked` выигрывает, а при какой — проигрывает из-за spinning в CAS-цикле.
9. Запустите `dotnet build -c Release` и убедитесь, что нет предупреждений. Затем `dotnet run -c Release` и проверьте, что безопасный счётчик всегда даёт ровно `threads * perThread`, а битый — меньше. Скриншот вывода приложите к сдаче.
10. В файле `README.md` внутри проекта кратко (5–7 строк) опишите, какие концепции урока вы применили и где именно.

#### Требования к решению
- Целевая платформа: .NET 8, C# 12, `LangVersion=latest`. Используйте top-level statements в `Program.cs`, file-scoped namespaces, pattern matching (`is not null`), collection expressions там, где это уместно (например, для инициализации массива потоков).
- Все разделяемые числовые счётчики изменяются исключительно через `Interlocked.Increment` / `Interlocked.Add` / `Interlocked.Decrement`. Простое `++`/`--` на разделяемом поле запрещено и должно быть помечено комментарием, если присутствует в «битом» демонстрационном коде.
- Чтение разделяемых `long` полей выполняется через `Interlocked.Read`; чтение `int` и ссылок — через `Volatile.Read`. Обычное чтение поля допустимо только внутри `lock`.
- Паттерн «проверить-и-заменить» реализован строго через `Interlocked.CompareExchange`, а не через `if (x == null) x = Build();`.
- Внутри `lock` нет ни одного `await`. Если в каком-то месте потребуется асинхронная критическая секция, используйте `SemaphoreSlim(1,1)` с `await sem.WaitAsync()` и `try/finally` — но в этом задании это не требуется, и наличие `SemaphoreSlim` должно быть обосновано.
- Нет вызовов `.Result` и `.Wait()` на `Task` — только `await` (либо синхронный код без асинхронности вообще).
- Код компилируется без предупреждений в `Release`, проходит `dotnet run` и демонстрирует корректные числа на многопоточной нагрузке.
- Каждая нетривиальная строка снабжена коротким комментарием на русском (и, для эталонного решения, дублирующим на английском), объясняющим, какой концепции урока она соответствует.

#### Тонкости и подводные камни
- `count++` — это три инструкции (read, add, write), и между ними поток может быть вытеснен. Даже на сильной модели памяти x86 это потерянное обновление. `Interlocked.Increment` — одна атомарная инструкция `LOCK XADD`, и она не уходит в ядро ОС, поэтому дёшева.
- Паттерн `if (x == 0) x = 1;` — классическая гонка check-then-act. Даже если каждое отдельное чтение и запись атомарны, между проверкой и записью другой поток успевает изменить `x`. Правильно: `Interlocked.CompareExchange(ref x, 1, 0)` — он вернёт предыдущее значение, и вы поймёте, выиграли ли вы CAS.
- `Interlocked` работает с одним значением за раз. Если вам нужно согласованно изменить баланс и счётчик транзакций, `Interlocked.Add` над каждым полем по отдельности оставит «окно», в котором баланс уже изменён, а счётчик ещё нет — наблюдатель может увидеть несогласованное состояние. Вот тут и нужен `lock`.
- `lock` не асинхронен: внутри `lock { await ... }` компилятор либо выдаст ошибку (CS4007 для некоторых случаев), либо, что хуже, вы удержите поток пула на время асинхронной операции и заблокируете масштабирование. Замена — `SemaphoreSlim(1,1)` с `await sem.WaitAsync()` и `try/finally { sem.Release(); }`.
- `volatile` даёт только упорядочивание памяти, но не атомарность составной операции. `volatile int x; x++;` всё равно потерянное обновление. Для атомарности нужен `Interlocked`.
- На ARM (Apple Silicon, ARM-серверы) переупорядочивание чтений и записей реально, и `Interlocked`-методы неявно вставляют нужные барьеры. Не добавляйте `Thread.MemoryBarrier()` «на всякий случай» — это лишние циклы и затуманивает код.
- CAS-цикл при высокой конкуренции может проиграть `lock` из-за spinning: много потоков крутятся в `CompareExchange`, постоянно перечитывая кэш-линию. Для очень горячего счётчика иногда выгоднее `lock` или даже разбиение на shard-счётчики. Измеряйте под реальной нагрузкой, а не наугад.
- Для `long` на 32-битных платформах простое чтение поля не атомарно (можёт прочитать «половинку»), поэтому используйте `Interlocked.Read`. На 64-битных это технически атомарно, но `Interlocked.Read` даёт ещё и нужный барьер — привычка полезная.
- Безопасная публикация объекта: писатель публикует через `Interlocked.Exchange`/`CompareExchange`, читатель читает через `Volatile.Read`. Это гарантирует, что читатель не увидит частично сконструированный объект из-за переупорядочивания записей внутри конструктора.

#### Критерии приёмки
- [ ] Проект собирается через `dotnet build -c Release` без предупреждений и ошибок на .NET 8 / C# 12.
- [ ] `MetricsCounter.RegisterRequest` использует `Interlocked.Add`/`Increment` и корректно обрабатывает флаг `isError`.
- [ ] `NextSequenceId()` возвращает монотонно возрастающий `long` через `Interlocked.Increment`.
- [ ] `AtomicConfigCache<T>.GetOrBuild` реализует быстрый путь через `Volatile.Read` и медленный через `CompareExchange`; возвращается победитель CAS, а не локальный `created`.
- [ ] `AtomicConfigCache<T>.Replace` использует `Interlocked.Exchange` и возвращает предыдущее значение.
- [ ] `RaceBenchmark.RunBroken` демонстрирует потерянные обновления; `RunSafe` всегда даёт точное `threads * perThread`.
- [ ] `TransferLog` использует `lock` для согласованного изменения двух полей; в комментарии объяснено, почему `Interlocked` не подходит.
- [ ] Внутри `lock` нет ни одного `await`; нет `.Result`/`.Wait()` на `Task`.
- [ ] Чтение `long` полей — через `Interlocked.Read`; чтение `int` и ссылок — через `Volatile.Read`.
- [ ] Micro-benchmark через `Stopwatch` выводит время lock-free и lock-версии, в комментарии сделан вывод о границе применимости.
- [ ] Каждый нетривиальный участок снабжён комментарием, ссылающимся на концепцию урока (атомарность, упорядочивание, CAS, безопасная публикация).
- [ ] `dotnet run -c Release` выводит корректные числа на `threads=16, perThread=1_000_000`.
- [ ] В `README.md` перечислены применённые концепции урока.
- [ ] Код использует C# 12 (top-level statements, file-scoped namespaces, pattern matching).
- [ ] Нет «волшебных» чисел без объяснения; константы вынесены и прокомментированы.

#### Подсказки (без прямого ответа)
- Вспомните, что `Interlocked.Add(ref _bytes, responseBytes)` возвращает новое значение — оно вам может пригодиться, чтобы не делать лишний `Read`.
- Для «если ошибка — увеличить ещё и `_errors`» подумайте, можно ли обойтись без условной логики внутри `lock`. Подсказка: можно, и это даже дешевле.
- В CAS-цикле для ленивой инициализации вам не нужен цикл `while` — одна попытка `CompareExchange(ref _value, created, null)` достаточна, потому что даже если CAS проиграл, кто-то уже записал значение, и вы просто вернёте его.
- `Volatile.Read` нужен на быстром пути, чтобы не вставлять полный барьер, но получить гарантию видимости.
- В `TransferLog` работайте в копейках/центах (`long`), а не в `decimal`, чтобы `Interlocked`/`lock` работали с примитивом — это повторяет приём из урока с `Account`.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — эталонное домашнее задание M11-L03.
// C# 12 / .NET 8 — reference solution for homework M11-L03.
using System.Diagnostics;
using System.Threading;

namespace M11L03.Homework;

/// <summary>
/// Lock-free счётчик метрик на Interlocked.Add / Increment.
/// Lock-free metrics counter built on Interlocked.Add / Increment.
/// Каждый инкремент — отдельная атомарная операция; три инкремента в
/// RegisterRequest НЕ согласованы между собой, но для метрик это допустимо:
/// наблюдатель видит «поток» значений, а не транзакцию. Для транзакционной
/// согласованности см. TransferLog ниже.
/// Each increment is a separate atomic op; the three increments in
/// RegisterRequest are NOT mutually consistent, but for metrics this is fine:
/// the observer sees a "stream" of values, not a transaction. For transactional
/// consistency see TransferLog below.
/// </summary>
public sealed class MetricsCounter
{
    private long _requests;   // всего запросов / total requests
    private long _errors;     // всего ошибок / total errors
    private long _bytes;      // суммарно байт / total bytes
    private long _sequence;   // sequence-номер / sequence id

    public long Requests => Interlocked.Read(ref _requests); // atomic read of long
    public long Errors   => Interlocked.Read(ref _errors);
    public long Bytes    => Interlocked.Read(ref _bytes);
    public long Sequence => Interlocked.Read(ref _sequence);

    public void RegisterRequest(int responseBytes, bool isError)
    {
        Interlocked.Increment(ref _requests);                 // atomic +1
        Interlocked.Add(ref _bytes, responseBytes);           // atomic += bytes
        if (isError) Interlocked.Increment(ref _errors);      // atomic +1 only on error
        // Почему три отдельных Interlocked, а не один lock? Потому что для метрик
        // допустимо, что наблюдатель увидит _requests уже увеличенным, а _bytes ещё
        // нет — итоговые числа через доли микросекунды сойдутся, и дашборд это не
        // заметит. Зато мы избегаем блокировок и сохраняем пропускную способность.
        // Why three separate Interlocked and not one lock? Because for metrics it is
        // acceptable for an observer to see _requests already incremented but _bytes
        // not yet — the totals converge within microseconds and the dashboard does not
        // notice. In return we avoid locks and preserve throughput.
    }

    public long NextSequenceId() => Interlocked.Increment(ref _sequence);
    // long обязателен: int переполнится примерно за 2.1 млрд, что на горячем сервисе
    // случается за сутки. Для long нужен Interlocked.Read — на 32-битных платформах
    // простое чтение 8 байт не атомарно, а на 64-битных Read даёт ещё и барьер.
    // long is mandatory: int overflows at ~2.1B which a hot service hits in a day.
    // For long, Interlocked.Read is required — on 32-bit platforms a plain 8-byte
    // read is not atomic, and on 64-bit Read additionally provides a barrier.
}

/// <summary>
/// Ленивый атомарный кэш конфигурации: строится один раз, можно переиздать.
/// Lazy atomic config cache: built once, can be republished.
/// Безопасная публикация: писатель — Interlocked.Exchange/CompareExchange,
/// читатель — Volatile.Read. Гарантирует, что читатель не увидит
/// частично сконструированный объект.
/// Safe publication: writer uses Interlocked.Exchange/CompareExchange,
/// reader uses Volatile.Read. Guarantees the reader never sees a
/// partially constructed object.
/// </summary>
public sealed class AtomicConfigCache<T> where T : class
{
    private T? _value;

    public T GetOrBuild(Func<T> factory)
    {
        // Быстрый путь: значение уже есть. Volatile.Read — упорядоченное чтение
        // без полного барьера, но достаточное для безопасной публикации.
        // Fast path: value already present. Volatile.Read is an ordered read
        // without a full fence, sufficient for safe publication.
        T? snapshot = Volatile.Read(ref _value);
        if (snapshot is not null) return snapshot;

        // Медленный путь: фабрика может вызваться у нескольких потоков,
        // но в _value закрепится ровно один объект — победитель CAS.
        // Slow path: factory may run on several threads, but only one object
        // sticks in _value — the CAS winner.
        T created = factory();
        T? previous = Interlocked.CompareExchange(ref _value, created, comparand: null);
        return previous ?? created; // вернём победителя, не свой created / return winner, not our created
    }

    public T Replace(T newValue)
    {
        // Переиздание конфигурации: атомарно записываем и возвращаем старую.
        // Republish config: atomically write and return the previous value.
        T? previous = Interlocked.Exchange(ref _value, newValue);
        return previous ?? throw new InvalidOperationException("cache was empty");
    }
}

/// <summary>
/// Транзакционный лог: баланс и счётчик должны меняться СОГЛАСОВАННО.
/// Transactional log: balance and counter must change CONSISTENTLY.
/// Здесь Interlocked не подходит — он работает с одним значением за раз.
/// Используем lock. ВАЖНО: внутри lock НЕТ await (иначе deadlock/старвление пула).
/// Interlocked does not fit here — it operates on a single value at a time.
/// We use lock. IMPORTANT: no await inside lock (deadlock / pool starvation).
/// </summary>
public sealed class TransferLog
{
    private long _balance;   // баланс в копейках / balance in cents
    private long _txCount;   // число транзакций / transaction count
    private readonly object _guard = new();

    public void Transfer(decimal amount, bool isCredit)
    {
        long delta = (long)(amount * 100m); // копейки, чтобы работать с long / cents to use long
        lock (_guard)
        {
            _balance = isCredit ? _balance + delta : _balance - delta;
            _txCount++; // согласовано с изменением баланса / consistent with balance change
        }
    }

    public long Balance => Interlocked.Read(ref _balance);
    public long TxCount  => Interlocked.Read(ref _txCount);
}

/// <summary>
/// Демонстрация race condition и его исправления.
/// Race condition demo and its fix.
/// </summary>
public static class RaceBenchmark
{
    public static int RunBroken(int threads, int perThread)
    {
        int counter = 0;
        Parallel.For(0, threads, _ =>
        {
            for (int i = 0; i < perThread; i++)
                counter++; // НЕ атомарНО: потерянные обновления / NOT atomic: lost updates
        });
        return counter;
    }

    public static int RunSafe(int threads, int perThread)
    {
        int counter = 0;
        Parallel.For(0, threads, _ =>
        {
            for (int i = 0; i < perThread; i++)
                Interlocked.Increment(ref counter); // атомарно / atomic
        });
        return counter;
    }
}

public static class Program
{
    public static void Main()
    {
        const int threads = 16, perThread = 1_000_000;
        int expected = threads * perThread;
        int broken = RaceBenchmark.RunBroken(threads, perThread);
        int safe   = RaceBenchmark.RunSafe(threads, perThread);
        Console.WriteLine($"expected={expected}, broken={broken}, safe={safe}");
        // broken почти всегда < expected — потерянные обновления из-за race в counter++.
        // broken is almost always < expected — lost updates from the race in counter++.

        // Micro-benchmark: lock-free vs lock при одинаковой нагрузке.
        // Micro-benchmark: lock-free vs lock under identical load.
        var mc = new MetricsCounter();
        var sw = Stopwatch.StartNew();
        Parallel.For(0, 8, _ => { for (int i = 0; i < 5_000_000; i++) mc.NextSequenceId(); });
        sw.Stop();
        Console.WriteLine($"lock-free: {sw.ElapsedMilliseconds} ms");

        var log = new TransferLog();
        sw = Stopwatch.StartNew();
        Parallel.For(0, 8, _ => { for (int i = 0; i < 5_000_000; i++) log.Transfer(1m, true); });
        sw.Stop();
        Console.WriteLine($"lock:     {sw.ElapsedMilliseconds} ms");
    }
}
```

Разбор по строкам. Класс `MetricsCounter` воплощает главный паттерн урока — lock-free счётчик на `Interlocked`. Поле `_requests` типа `long`, и его чтение в свойстве `Requests` обязательно через `Interlocked.Read`, потому что на 32-битных платформах простое чтение восьми байт не атомарно, а на 64-битных `Read` дополнительно даёт нужный барьер памяти — это прямо из раздела «Best Practices». Метод `RegisterRequest` делает три отдельных атомарных операции (`Increment`, `Add`, `Increment`), и в комментарии честно объяснено, почему это допустимо для метрик: наблюдатель видит поток значений, а не транзакцию, и итоговые числа сходятся за доли микросекунды. Это разграничение «атомарность операции vs согласованность нескольких полей» — ключевая тонкость урока. Метод `NextSequenceId` использует `long` (а не `int`), чтобы не переполниться за сутки, и `Interlocked.Increment` для атомарности — это повторяет приём `LockFreeCounter` из примера.

Класс `AtomicConfigCache<T>` реализует безопасную публикацию объекта. Быстрый путь — `Volatile.Read(ref _value)`: упорядоченное чтение без полного барьера, но достаточное, чтобы увидеть объект, опубликованный через `Interlocked`. Медленный путь — `CompareExchange(ref _value, created, null)`: даже если фабрика вызвалась у нескольких потоков одновременно, в `_value` закрепится ровно один объект, и возвращается именно победитель (`previous ?? created`), а не локальный `created`. Это исправляет классическую ловушку `if (x == null) x = Build();`, явно упомянутую в разделе «Частые ошибки». Метод `Replace` через `Interlocked.Exchange` моделирует переиздание конфигурации администратором и возвращает предыдущее значение — это «безопасная публикация» со стороны писателя.

Класс `TransferLog` демонстрирует границу применимости `Interlocked`: баланс и счётчик транзакций должны меняться согласованно, а `Interlocked` работает с одним значением за раз. Поэтому здесь `lock`, и в комментарии явно указано, почему `Interlocked` не подходит и что внутри `lock` недопустим `await` (упомянутая в уроке ловушка `lock + await`). Работа в копейках через `long` повторяет приём из `Account`. Класс `RaceBenchmark` — прямая демонстрация урока: `counter++` против `Interlocked.Increment`, с ожидаемым выводом, что «битый» счётчик почти всегда меньше ожидаемого. Micro-benchmark в `Main` через `Stopwatch` и `Parallel.For` отвечает на вопрос «когда `Interlocked` выгоднее `lock`»: при умеренной конкуренции lock-free выигрывает, но при экстремальной CAS-цикл может проиграть из-за spinning — это прямо из раздела «Best Practices».

#### Задания на углубление (бонус)
1. Реализуйте sharded-счётчик: массив из N `MetricsCounter`, выбираемый по `Thread.GetCurrentProcessorId() % N`, и свойство `Total`, суммирующее все шарды через `Interlocked.Read`. Сравните throughput с одним счётчиком при `threads = 16` — должно быть заметно выше на горячем счётчике.
2. Реализуйте lock-free очередь на `CompareExchange` (один производитель, один потребитель, ring-buffer) и сравните корректность с `ConcurrentQueue<T>` из BCL.
3. Напишите тест на `dotnet test` с xUnit, который запускает `RaceBenchmark.RunSafe` 100 раз и проверяет, что результат всегда точно равен `threads * perThread` — это регрессионный тест на атомарность.
4. Исследуйте: замените `Volatile.Read` на обычное чтение поля в `AtomicConfigCache` и запустите на ARM-эмуляторе (или просто объясните в README, какие эффекты могут проявиться на ARM и почему на x86 их не видно).

---

## Statement in English / Постановка на английском

#### Context & motivation
You are building a metrics-collection subsystem for a high-throughput HTTP service on .NET 8. Every nanosecond the service handles dozens of concurrent requests, and each request bumps several counters at once: the total request count, the error count, the cumulative response size in bytes, and a unique sequence number used for log correlation. On top of that, the service has a shared configuration cache that is lazily built once from an expensive source and then republished whenever an administrator signals an update. Any lost counter update corrupts the dashboard and breaks SLO reporting; any race during cache initialization means different threads observe different versions of the configuration. In this task you are forbidden from reaching for heavy kernel primitives — `lock` is allowed only where it is genuinely unavoidable (a compound operation over several fields), and `await` inside a critical section is disallowed altogether. You will design a set of thread-safe structures on top of `Interlocked`, correctly publish the cache through `CompareExchange`, and benchmark a lock-free counter against a `lock`-based version under real load, so that you understand not only "how to write it" but "when it actually pays off". This exercise directly mirrors the real code from the lesson: the lock-free `LockFreeCounter`, the lazy `LazyAtomicReference<T>`, and the two-field `Account` comparison.

#### What to do step by step
1. Create a fresh console project: `dotnet new console -n M11L03.Homework -o M11L03.Homework --framework net8.0`. Open the project folder and confirm that `M11L03.Homework.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and `<LangVersion>latest</LangVersion>`. Delete the template `Program.cs` and create a new one with top-level statements.
2. Implement a `MetricsCounter` class with three fields: `long _requests`, `long _errors`, `long _bytes`. Public method `RegisterRequest(int responseBytes, bool isError)` atomically increments `_requests` by 1, adds `responseBytes` to `_bytes`, and, if `isError` is true, increments `_errors` by 1. Think carefully: can these three increments be done as "three `Interlocked.Add` calls" and still remain correct? Justify your answer in a comment.
3. Implement `NextSequenceId()` returning a monotonically increasing `long` via `Interlocked.Increment(ref _sequence)`. Use `long`, not `int`, so you do not hit overflow within a day of uptime. In a comment explain why `Interlocked.Read` is mandatory for `long` and a plain field read is not acceptable.
4. Implement an `AtomicConfigCache<T>` where `T : class` that lazily builds the configuration once via a supplied `Func<T>` factory. Use the fast path `Volatile.Read` and the slow path through `Interlocked.CompareExchange(ref _value, created, null)`. Crucially, the factory may be invoked by several threads, but exactly one object must stick in `_value` — the CAS winner. Return that winner, not your local `created`, when the CAS loses.
5. Add a `Replace(T newValue)` method that atomically swaps the current configuration for the new one via `Interlocked.Exchange` and returns the previous value — this models an administrator republishing the config.
6. Implement a "broken" and a "safe" variant in a static `RaceBenchmark` class: `RunBroken(int threads, int perThread)` using `counter++` and `RunSafe` using `Interlocked.Increment`. Run both with `threads = 16`, `perThread = 1_000_000` and print the expected value, the actual broken value, and the actual safe value. The broken value is almost always less than expected — explain in the console output which updates are being lost.
7. Implement a `TransferLog` class in which two related operations — writing to the balance and incrementing the transaction counter — must execute consistently. Here, per the lesson, use `lock`, because `Interlocked` operates on a single value at a time. In a comment explicitly state why `Interlocked` does not fit, and that `await` cannot be placed inside `lock`.
8. Add a micro-benchmark using `Parallel.For` with `threads = 8` and `perThread = 5_000_000` that compares the wall-clock time of the lock-free counter against the `lock`-based version. Print both timings in milliseconds using `Stopwatch`. In a comment draw a conclusion: under what load does `Interlocked` win, and under what load does it lose because of CAS-loop spinning.
9. Run `dotnet build -c Release` and confirm there are no warnings. Then run `dotnet run -c Release` and verify that the safe counter always yields exactly `threads * perThread`, while the broken one yields less. Attach a screenshot of the output to your submission.
10. In a `README.md` inside the project, briefly (5–7 lines) describe which lesson concepts you applied and exactly where.

#### Requirements
- Target platform: .NET 8, C# 12, `LangVersion=latest`. Use top-level statements in `Program.cs`, file-scoped namespaces, pattern matching (`is not null`), and collection expressions where appropriate (e.g. for initializing an array of worker handles).
- Every shared numeric counter is mutated exclusively via `Interlocked.Increment` / `Interlocked.Add` / `Interlocked.Decrement`. A plain `++`/`--` on a shared field is forbidden and, if present in the broken demonstration code, must be flagged with a comment.
- Reads of shared `long` fields go through `Interlocked.Read`; reads of `int` and references go through `Volatile.Read`. A plain field read is only acceptable inside a `lock`.
- The check-and-replace pattern is implemented strictly through `Interlocked.CompareExchange`, never through `if (x == null) x = Build();`.
- There is no `await` inside any `lock`. If an async critical section is ever needed, use `SemaphoreSlim(1,1)` with `await sem.WaitAsync()` and `try/finally` — but this assignment does not require it, and any `SemaphoreSlim` usage must be justified.
- No `.Result` and no `.Wait()` calls on `Task` — only `await` (or fully synchronous code with no async at all).
- The code compiles without warnings in `Release`, runs under `dotnet run`, and demonstrates correct numbers under multi-threaded load.
- Every non-trivial line carries a short comment (in English, and in the reference solution also in Russian) pointing back to the specific lesson concept it embodies.

#### Pitfalls
- `count++` is three instructions (read, add, write) and a thread can be preempted between them. Even on the strong x86 memory model this is a lost update. `Interlocked.Increment` is a single atomic `LOCK XADD` instruction that does not transition to kernel mode, so it is cheap.
- The `if (x == 0) x = 1;` pattern is a textbook check-then-act race. Even if each individual read and write is atomic, between the check and the write another thread can mutate `x`. The correct form is `Interlocked.CompareExchange(ref x, 1, 0)` — it returns the previous value, so you know whether you won the CAS.
- `Interlocked` operates on one value at a time. If you need to change balance and transaction count consistently, separate `Interlocked.Add` calls leave a window where the balance is already updated but the counter is not — an observer can catch an inconsistent state. That is exactly where `lock` belongs.
- `lock` is not async-aware: inside `lock { await ... }` the compiler either errors out or, worse, you hold a pool thread for the duration of the async operation and stall scaling. The replacement is `SemaphoreSlim(1,1)` with `await sem.WaitAsync()` and `try/finally { sem.Release(); }`.
- `volatile` provides only memory ordering, not atomicity of a compound operation. `volatile int x; x++;` is still a lost update. For atomicity you need `Interlocked`.
- On ARM (Apple Silicon, ARM servers) reordering of reads and writes is real, and `Interlocked` methods implicitly insert the necessary fences. Do not sprinkle `Thread.MemoryBarrier()` "just in case" — it wastes cycles and obscures the code.
- Under heavy contention a CAS-loop can lose to `lock` because of spinning: many threads spin in `CompareExchange`, repeatedly invalidating each other's cache line. For a very hot counter it is sometimes better to use `lock` or to shard the counter. Measure under real load, never guess.
- For `long` on 32-bit platforms a plain field read is not atomic (you may read a "half" value), so use `Interlocked.Read`. On 64-bit it is technically atomic, but `Interlocked.Read` also provides the needed barrier — a habit worth keeping.
- Safe publication of an object: the writer publishes via `Interlocked.Exchange`/`CompareExchange`, the reader reads via `Volatile.Read`. This guarantees the reader never observes a partially constructed object due to reordering of writes inside the constructor.

#### Acceptance criteria
- [ ] The project builds with `dotnet build -c Release` with no warnings or errors on .NET 8 / C# 12.
- [ ] `MetricsCounter.RegisterRequest` uses `Interlocked.Add`/`Increment` and correctly handles the `isError` flag.
- [ ] `NextSequenceId()` returns a monotonically increasing `long` via `Interlocked.Increment`.
- [ ] `AtomicConfigCache<T>.GetOrBuild` implements the fast path via `Volatile.Read` and the slow path via `CompareExchange`; it returns the CAS winner, not the local `created`.
- [ ] `AtomicConfigCache<T>.Replace` uses `Interlocked.Exchange` and returns the previous value.
- [ ] `RaceBenchmark.RunBroken` demonstrates lost updates; `RunSafe` always yields exactly `threads * perThread`.
- [ ] `TransferLog` uses `lock` for the consistent change of two fields; the comment explains why `Interlocked` does not fit.
- [ ] There is no `await` inside any `lock`; no `.Result`/`.Wait()` on `Task`.
- [ ] Reads of `long` fields use `Interlocked.Read`; reads of `int` and references use `Volatile.Read`.
- [ ] The micro-benchmark via `Stopwatch` prints timings for the lock-free and lock versions, and the comment draws a conclusion about the applicability boundary.
- [ ] Every non-trivial section carries a comment referencing a lesson concept (atomicity, ordering, CAS, safe publication).
- [ ] `dotnet run -c Release` prints correct numbers under `threads=16, perThread=1_000_000`.
- [ ] `README.md` lists the applied lesson concepts.
- [ ] The code uses C# 12 (top-level statements, file-scoped namespaces, pattern matching).
- [ ] No "magic" numbers without explanation; constants are extracted and commented.

#### Hints (no direct answer)
- Recall that `Interlocked.Add(ref _bytes, responseBytes)` returns the new value — you may reuse it and avoid an extra `Read`.
- For the "if error, also bump `_errors`" branch, consider whether you can avoid conditional logic inside a `lock`. Hint: you can, and it is even cheaper.
- In the CAS path for lazy initialization you do not need a `while` loop — a single `CompareExchange(ref _value, created, null)` is enough, because if the CAS lost, somebody already wrote the value and you simply return it.
- `Volatile.Read` on the fast path avoids a full fence while still giving you a visibility guarantee.
- In `TransferLog` work in cents (`long`), not `decimal`, so that `Interlocked`/`lock` operate on a primitive — this mirrors the `Account` trick from the lesson.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — reference solution for homework M11-L03.
using System.Diagnostics;
using System.Threading;

namespace M11L03.Homework;

/// <summary>
/// Lock-free metrics counter built on Interlocked.Add / Increment.
/// Each increment is a separate atomic op; the three increments in
/// RegisterRequest are NOT mutually consistent, but for metrics this is fine:
/// the observer sees a "stream" of values, not a transaction. For transactional
/// consistency see TransferLog below.
/// </summary>
public sealed class MetricsCounter
{
    private long _requests;
    private long _errors;
    private long _bytes;
    private long _sequence;

    public long Requests => Interlocked.Read(ref _requests);
    public long Errors   => Interlocked.Read(ref _errors);
    public long Bytes    => Interlocked.Read(ref _bytes);
    public long Sequence => Interlocked.Read(ref _sequence);

    public void RegisterRequest(int responseBytes, bool isError)
    {
        Interlocked.Increment(ref _requests);
        Interlocked.Add(ref _bytes, responseBytes);
        if (isError) Interlocked.Increment(ref _errors);
        // Why three separate Interlocked and not one lock? Because for metrics it is
        // acceptable for an observer to see _requests already incremented but _bytes
        // not yet — the totals converge within microseconds and the dashboard does not
        // notice. In return we avoid locks and preserve throughput.
    }

    public long NextSequenceId() => Interlocked.Increment(ref _sequence);
    // long is mandatory: int overflows at ~2.1B which a hot service hits in a day.
    // For long, Interlocked.Read is required — on 32-bit platforms a plain 8-byte
    // read is not atomic, and on 64-bit Read additionally provides a barrier.
}

/// <summary>
/// Lazy atomic config cache: built once, can be republished.
/// Safe publication: writer uses Interlocked.Exchange/CompareExchange,
/// reader uses Volatile.Read. Guarantees the reader never sees a
/// partially constructed object.
/// </summary>
public sealed class AtomicConfigCache<T> where T : class
{
    private T? _value;

    public T GetOrBuild(Func<T> factory)
    {
        // Fast path: value already present. Volatile.Read is an ordered read
        // without a full fence, sufficient for safe publication.
        T? snapshot = Volatile.Read(ref _value);
        if (snapshot is not null) return snapshot;

        // Slow path: factory may run on several threads, but only one object
        // sticks in _value — the CAS winner.
        T created = factory();
        T? previous = Interlocked.CompareExchange(ref _value, created, comparand: null);
        return previous ?? created; // return winner, not our created
    }

    public T Replace(T newValue)
    {
        T? previous = Interlocked.Exchange(ref _value, newValue);
        return previous ?? throw new InvalidOperationException("cache was empty");
    }
}

/// <summary>
/// Transactional log: balance and counter must change CONSISTENTLY.
/// Interlocked does not fit here — it operates on a single value at a time.
/// We use lock. IMPORTANT: no await inside lock (deadlock / pool starvation).
/// </summary>
public sealed class TransferLog
{
    private long _balance;   // balance in cents
    private long _txCount;   // transaction count
    private readonly object _guard = new();

    public void Transfer(decimal amount, bool isCredit)
    {
        long delta = (long)(amount * 100m); // cents to use long
        lock (_guard)
        {
            _balance = isCredit ? _balance + delta : _balance - delta;
            _txCount++; // consistent with balance change
        }
    }

    public long Balance => Interlocked.Read(ref _balance);
    public long TxCount  => Interlocked.Read(ref _txCount);
}

public static class RaceBenchmark
{
    public static int RunBroken(int threads, int perThread)
    {
        int counter = 0;
        Parallel.For(0, threads, _ =>
        {
            for (int i = 0; i < perThread; i++)
                counter++; // NOT atomic: lost updates
        });
        return counter;
    }

    public static int RunSafe(int threads, int perThread)
    {
        int counter = 0;
        Parallel.For(0, threads, _ =>
        {
            for (int i = 0; i < perThread; i++)
                Interlocked.Increment(ref counter); // atomic
        });
        return counter;
    }
}

public static class Program
{
    public static void Main()
    {
        const int threads = 16, perThread = 1_000_000;
        int expected = threads * perThread;
        int broken = RaceBenchmark.RunBroken(threads, perThread);
        int safe   = RaceBenchmark.RunSafe(threads, perThread);
        Console.WriteLine($"expected={expected}, broken={broken}, safe={safe}");
        // broken is almost always < expected — lost updates from the race in counter++.

        var mc = new MetricsCounter();
        var sw = Stopwatch.StartNew();
        Parallel.For(0, 8, _ => { for (int i = 0; i < 5_000_000; i++) mc.NextSequenceId(); });
        sw.Stop();
        Console.WriteLine($"lock-free: {sw.ElapsedMilliseconds} ms");

        var log = new TransferLog();
        sw = Stopwatch.StartNew();
        Parallel.For(0, 8, _ => { for (int i = 0; i < 5_000_000; i++) log.Transfer(1m, true); });
        sw.Stop();
        Console.WriteLine($"lock:     {sw.ElapsedMilliseconds} ms");
    }
}
```

Walk-through, line by line. The `MetricsCounter` class embodies the lesson's headline pattern — a lock-free counter built on `Interlocked`. The `_requests` field is `long`, and its read in the `Requests` property is forced through `Interlocked.Read`, because on 32-bit platforms a plain eight-byte read is not atomic, and on 64-bit `Read` additionally supplies the required memory fence — this comes straight from the "Best Practices" section. The `RegisterRequest` method performs three separate atomic operations (`Increment`, `Add`, `Increment`), and the comment honestly explains why this is acceptable for metrics: an observer sees a stream of values rather than a transaction, and the totals converge within microseconds. This distinction — "atomicity of a single operation vs. consistency across several fields" — is the central subtlety of the lesson. The `NextSequenceId` method uses `long` (not `int`) to avoid overflow within a day, and `Interlocked.Increment` for atomicity — it mirrors the `LockFreeCounter` example.

The `AtomicConfigCache<T>` class implements safe publication of an object. The fast path is `Volatile.Read(ref _value)`: an ordered read without a full fence, but sufficient to observe an object published via `Interlocked`. The slow path is `CompareExchange(ref _value, created, null)`: even if the factory runs on several threads at once, exactly one object sticks in `_value`, and the code returns that winner (`previous ?? created`), not the local `created`. This fixes the classic `if (x == null) x = Build();` trap called out in the "Common Mistakes" section. The `Replace` method, via `Interlocked.Exchange`, models an administrator republishing the configuration and returns the previous value — this is safe publication from the writer's side.

The `TransferLog` class marks the boundary of `Interlocked`'s applicability: the balance and the transaction counter must change consistently, and `Interlocked` only handles one value at a time. Hence `lock`, and the comment explicitly states why `Interlocked` does not fit and that `await` is forbidden inside `lock` (the `lock + await` trap mentioned in the lesson). Working in cents via `long` mirrors the `Account` trick. The `RaceBenchmark` class is a direct echo of the lesson: `counter++` versus `Interlocked.Increment`, with the expected output that the broken counter is almost always less than expected. The micro-benchmark in `Main` via `Stopwatch` and `Parallel.For` answers the question "when is `Interlocked` better than `lock`": under moderate contention lock-free wins, but under extreme contention a CAS-loop can lose because of spinning — this is exactly the "Best Practices" guidance.

#### Going deeper (bonus)
1. Implement a sharded counter: an array of N `MetricsCounter` instances selected by `Thread.GetCurrentProcessorId() % N`, plus a `Total` property that sums all shards via `Interlocked.Read`. Compare throughput against a single counter at `threads = 16` — it should be noticeably higher on a hot counter.
2. Implement a lock-free single-producer/single-consumer ring-buffer queue on `CompareExchange` and compare its correctness and performance against BCL's `ConcurrentQueue<T>`.
3. Write an xUnit test on `dotnet test` that runs `RaceBenchmark.RunSafe` 100 times and asserts the result is always exactly `threads * perThread` — a regression test for atomicity.
4. Investigate: replace `Volatile.Read` with a plain field read in `AtomicConfigCache` and run on an ARM emulator (or simply explain in the README which effects might surface on ARM and why they are invisible on x86).

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `M11L03.Homework` создан на .NET 8 / C# 12.
- [ ] `MetricsCounter`, `AtomicConfigCache<T>`, `TransferLog`, `RaceBenchmark` реализованы.
- [ ] `dotnet build -c Release` без предупреждений.
- [ ] `dotnet run -c Release` выводит `broken < expected` и `safe == expected`.
- [ ] Micro-benchmark выводит время lock-free и lock; в комментарии — вывод о границе применимости.
- [ ] Внутри `lock` нет `await`; нет `.Result`/`.Wait()`.
- [ ] `README.md` с перечнем применённых концепций урока.
- [ ] Скриншот вывода приложен.
- [ ] Project `M11L03.Homework` created on .NET 8 / C# 12.
- [ ] `MetricsCounter`, `AtomicConfigCache<T>`, `TransferLog`, `RaceBenchmark` implemented.
- [ ] `dotnet build -c Release` is warning-free.
- [ ] `dotnet run -c Release` prints `broken < expected` and `safe == expected`.
- [ ] Micro-benchmark prints lock-free and lock timings; comment draws an applicability conclusion.
- [ ] No `await` inside `lock`; no `.Result`/`.Wait()`.
- [ ] `README.md` lists the applied lesson concepts.
- [ ] Screenshot of the output attached.

#### Ресурсы / Resources
- [Microsoft Learn — Interlocked class](https://learn.microsoft.com/dotnet/api/system.threading.interlocked)
- [Microsoft Learn — Volatile class](https://learn.microsoft.com/dotnet/api/system.threading.volatile)
- [Microsoft Learn — Memory model and atomicity in .NET](https://learn.microsoft.com/dotnet/standard/threading/the-managed-thread-pool)
- [Шаблон домашнего задания: homework-M11-L02-lock-monitor.md](homework-M11-L02-lock-monitor.md)
