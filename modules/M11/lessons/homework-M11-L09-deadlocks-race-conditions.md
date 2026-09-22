---
[← К уроку M11-L09](lesson-M11-L09-deadlocks-race-conditions.md) | [⬆ К модулю M11](../README.md) | [Следующее ДЗ →](homework-M11-L10-threadpool-longrunning.md)
---

### Домашнее задание M11-L09: Deadlocks/race conditions, диагностика / Homework M11-L09: Deadlocks/race conditions, diagnostics

**Урок / Lesson:** M11-L09
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться воспроизводить, диагностировать и устранять race conditions и deadlocks в C# 12 / .NET 8, применять правильную синхронизацию (`Interlocked`, `lock`, `Monitor.TryEnter`, `SemaphoreSlim`, `Channel<T>`), пользоваться диагностическим инструментарием (`Debug.Assert`, `dotnet-counters`, `dotnet-trace`, `dotnet-dump`) и отличать четыре классические проблемы concurrency. (EN) Learn to reproduce, diagnose, and eliminate race conditions and deadlocks in C# 12 / .NET 8, apply correct synchronization (`Interlocked`, `lock`, `Monitor.TryEnter`, `SemaphoreSlim`, `Channel<T>`), use the diagnostic toolkit (`Debug.Assert`, `dotnet-counters`, `dotnet-trace`, `dotnet-dump`), and distinguish the four classic concurrency problems.

#### Связь с уроком / Connection to the lesson
(RU) Урок M11-L09 разбирает четыре класса проблем — race condition, data race, deadlock и live-lock — и показывает, как `i++` теряет обновления, как обратный порядок локов взаимно блокирует потоки, и как `Channel<T>` + иммутабельные данные устраняют синхронизацию. Это ДЗ закрепляет каждую идею на воспроизводимом коде: вы сами сломаете счётчик, устроите дедлок, а потом почините их инструментами из урока.
(EN) Lesson M11-L09 covers four problem classes — race condition, data race, deadlock, and live-lock — and shows how `i++` loses updates, how reversed lock order deadlocks threads, and how `Channel<T>` plus immutable data remove the need for synchronization. This homework fixes every idea in reproducible code: you will break a counter, cause a deadlock, then repair both with the tools from the lesson.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Представьте, что вы поддерживаете учебный банк «ConcurrentBank»: несколько кассиров одновременно выполняют переводы между счетами, а отдельный поток агрегирует статистику по балансам. В нагрузочном тесте баланс системы «не сходится»: сумма по счетам отличается от суммы всех проведённых операций. Иногда приложение зависает намертво и его приходится убивать через `kill -9`. Команда уже дважды пыталась «починить» плавающие тесты через `Thread.Sleep`, но баг возвращается в продакшене под ростом нагрузки. Ваша задача — превратить хаос в инженерный процесс: воспроизвести обе проблемы детерминированно (насколько это вообще возможно для concurrency-багов), собрать диагностику, найти первопричину, устранить её правильной синхронизацией или устранением синхронизации вовсе, и защитить инварианты так, чтобы регрессия ловилась в CI, а не у клиента.

Это упражнение — не академическая задача: оно моделирует ровно тот класс дефектов, который составляет львиную долю инцидентов в многопоточных .NET-сервисах. Вы научитесь отличать data race (потеря обновления на `i++`) от race condition (логическая ошибка порядка операций), от deadlock (вечное ожидание ресурсов), от live-lock (бесполезная активность). Каждый из этих классов требует своего лекарства: `Interlocked` для простых счётчиков, фиксированный порядок локов или `Monitor.TryEnter` с таймаутом для дедлоков, рандомизированный backoff или арбитр для live-lock, и `Channel<T>` + иммутабельные снапшоты для устранения разделяемого состояния целиком. Диагностика — половина работы: без `Debug.Assert`, `dotnet-counters` и стресс-теста под нагрузкой вы не докажете, что баг действительно ушёл.

#### Что нужно сделать (пошагово)
1. Создайте решение и три проекта: библиотеку с доменом, консольный воспроизводитель багов и xUnit-проект со стресс-тестами. Команды:
   ```bash
   dotnet new sln -n ConcurrentBank
   dotnet new classlib -n ConcurrentBank.Domain -o src/ConcurrentBank.Domain -f net8.0
   dotnet new console -n ConcurrentBank.Repro -o src/ConcurrentBank.Repro -f net8.0
   dotnet new xunit -n ConcurrentBank.Tests -o tests/ConcurrentBank.Tests -f net8.0
   dotnet sln add src/ConcurrentBank.Domain tests/ConcurrentBank.Tests src/ConcurrentBank.Repro
   dotnet add tests/ConcurrentBank.Tests reference src/ConcurrentBank.Domain
   dotnet add src/ConcurrentBank.Repro reference src/ConcurrentBank.Domain
   ```
2. В `ConcurrentBank.Domain` реализуйте класс `Account` с полем `int _balance` и методами `Deposit`/`Withdraw`/`Transfer`. Сделайте **три версии**: `BrokenAccount` (без синхронизации — воспроизводит data race на `_balance`), `LockedAccount` (с `lock` и фиксированным порядком захвата локов при переводе — сначала меньший `Id`, потом больший), и `ChannelAccount` (переводы отправляются как сообщения в `Channel<TransferCommand>`, а один потребитель применяет их последовательно — без блокировок между счетами).
3. Реализуйте класс `DeadlockProneBank` с методом `Transfer(Account a, Account b, int amount)`, который берёт `lock(a._gate)` а затем `lock(b._gate)` — без упорядочивания по `Id`. Запустите параллельный тест: `Parallel.For(0, 1000, i => Transfer(accounts[i % N], accounts[(i + 1) % N], 10))`. Должен зависать. Зафиксируйте зависание как «воспроизведённый баг» (например, тест с `CancellationTokenSource` на 5 секунд, который падает по таймауту — это и есть доказательство deadlock).
4. В `ConcurrentBank.Repro/Program.cs` (top-level statements) реализуйте меню: `1) race repro 2) deadlock repro 3) live-lock repro 4) channel fix 5) counters live`. Для каждой опции запустите соответствующий сценарий и напечатайте результат. Для опции 5 запустите `dotnet-counters monitor --process-id <pid> System.Runtime` — выведите инструкцию пользователю (PID процесса и команду в отдельном терминале).
5. Добавьте `Debug.Assert` в `Deposit`/`Withdraw`: баланс не может стать отрицательным; сумма всех депозитов монотонно не убывает. Запустите Debug-сборку под нагрузкой — ассерты должны срабатывать на `BrokenAccount` и не срабатывать на `LockedAccount`/`ChannelAccount`.
6. Напишите xUnit-стресс-тест `StressTests`: для каждой версии счёта прогоните `Parallel.For(0, 8, _ => { for (int i = 0; i < 100_000; i++) account.Deposit(1); })`, затем assert `account.Balance == 800_000`. `BrokenAccount` должен падать (или хотя бы иногда), остальные — проходить стабильно. Повторите тест 10 раз в цикле — race проявляется статистически.
7. Запустите `dotnet-counters` на стресс-тесте и зафиксируйте `Monitor Lock Contention/sec` для `LockedAccount`. Сравните с `ChannelAccount` (там contention должен быть около нуля, потому что потребитель один). Запишите числа в `README.md` вашего решения.
8. Используйте `dotnet-trace collect --providers Microsoft-Windows-DotNETRuntime` на 10-секундном прогоне deadlock-сценария, конвертируйте в `speedscope` и найдите на таймлайне потоки, которые стоят на `Monitor.Enter`. Приложите скриншот или описание в `README.md`.
9. Воспроизведите live-lock: два потока «уступают» друг другу ресурс через `volatile bool` флаги в цикле, каждый видит, что другой хочет, и снимает свой запрос — бесконечно. Исправьте рандомизированным `Task.Delay(Random.Shared.Next(1, 10))`. Покажите, что до фикса счётчик попыток уходит в тысячи, после — завершается за единицы.
10. Уберите все `.Result`/`.Wait()` из кода. Если где-то необходимо дождаться async-операции синхронно (например, в `Main`), используйте `Task.Run(() => ...).GetAwaiter().GetResult()` только как временный костыль и пометьте комментарием, почему это безопасно в данном контексте (нет захваченного `SynchronizationContext`).

#### Требования к решению
- Целевой фреймворк — `net8.0`; язык — C# 12 (top-level statements в `Program.cs`, collection expressions для инициализации списков счетов `[new Account(1, 100), new Account(2, 200)]`, pattern matching для разбора команд, raw string literals для многострочного help-текста).
- Все разделяемые mutable-поля защищены одним из способов: `Interlocked`, `lock`, `ConcurrentDictionary`, `Channel<T>`, или помечены `volatile`/`Volatile` для флагов видимости. Незащищённых `int`-полей, читаемых/пишущихся из нескольких потоков, быть не должно.
- Порядок захвата нескольких локов зафиксирован и документирован: например, «всегда сначала lock счета с меньшим `Id`». Это правило соблюдается во **всех** методах без исключения.
- В async-коде вместо `lock` используется `SemaphoreSlim(1,1)` с `WaitAsync`/`Release` в `try/finally`; обычный `lock` через `await` отсутствует (компилятор это и так запретит, но `SemaphoreSlim`, удерживаемый через `await`, тоже недопустим — критическая секция должна быть короткой и без await внутри).
- В библиотечном коде (`ConcurrentBank.Domain`) везде стоит `ConfigureAwait(false)`; `CancellationToken` пробрасывается во все `async`-методы и в `Channel.Writer.WriteAsync`.
- В CI-тесте есть стресс-тест с N потоков × M итераций, прогоняемый несколько раз; `Debug.Assert` защищает ключевые инварианты (неотрицательность баланса, монотонность, conservation of money — сумма по всем счетам до и после серии переводов совпадает).
- Диагностика: в `README.md` описаны воспроизведённые симптомы, числа `Monitor Lock Contention/sec` для каждой версии, и findings из `dotnet-trace`/`speedscope`.
- Анти-паттерны из урока (`Thread.Sleep` в продакшн-коде как «фикс», `.Result` на async в UI/ASP.NET Classic контексте, `async void` без try/catch, забытый `Volatile.Read`) — отсутствуют; если нужны намеренно (как демо), помечены `// ANTI-PATTERN — demo only`.

#### Тонкости и подводные камни
- `i++` на `int` **не атомарен** в C# даже на 64-бит: это чтение + сложение + запись. Два потока могут оба прочитать `5` и оба записать `6` — одно обновление потеряно. Лекарство — `Interlocked.Increment(ref _value)`, а не `volatile` (последний даёт видимость, но не атомарность инкремента). Это самая частая ошибка новичков:他们认为«`volatile` достаточно».
- Обратный порядок локов в `Transfer(a, b)` vs `Transfer(b, a)` — классический deadlock. Фикс — упорядочивание по стабильному ключу (`Id`): `var (first, second) = a.Id <= b.Id ? (a, b) : (b, a);`. Альтернатива — `Monitor.TryEnter(second, timeout, ref got)` с откатом и освобождением первого лок. Никогда не «чините» дедлок `Thread.Sleep` между локами — это маска; нагрузка изменится и баг вернётся.
- `lock` + `await` недопустимы: компилятор запретит `lock` в `async`-методе, но `SemaphoreSlim.WaitAsync` + долгий `await` внутри — та же проблема (другие ждут, секция раздута). Держите критическую секцию короткой; долгую работу делайте через `Channel` вне лок.
- `.Result`/`.Wait()` на async-методе в UI/ASP.NET Classic контексте синхронизации = классический deadlock: продолжение ждёт контекст, занятый `.Result`. В .NET 8 ASP.NET Core такого контекста нет (опасности меньше), но в библиотеке всё равно ставьте `ConfigureAwait(false)` — это страховка от вызова из WPF/WinForms/ASP.NET Classic.
- `Channel<T>` не бесплатен: bounded-канал с малой ёмкостью может заблокировать producer через `WriteAsync` (это нормально — backpressure), но если consumer упадёт с исключением, producer зависнет навсегда — всегда пробрасывайте `CancellationToken` и обрабатывайте `ChannelClosedException`.
- `Debug.Assert` компилируется «в ноль» в Release — не используйте его для runtime-валидации в продакшне; для этого есть `Trace.Assert` или явные `throw`. Но для ловли race в CI/Debug — идеально: почти бесплатно.
- `Volatile.Read`/`Volatile.Write` нужны для флагов видимости между потоками, когда вы не используете `lock`. Просто `bool _stop` без `volatile` может никогда не стать видимым другому потоку (JIT может закэшировать чтение в регистре). `volatile bool` или `Volatile.Read` решают.
- Стресс-тест должен повторяться много раз — race проявляется статистически. Один прогон может пройти случайно. Минимум — 10 повторений по 100k итераций на 8 потоках.
- `dotnet-counters` имеет оверхед около 1–2% — можно держать включённым в продакшне. `dotnet-trace` — тяжелее, используйте короткими сессиями по 10–30 секунд.

#### Критерии приёмки
- [ ] Решение собирается `dotnet build` без warnings (treat warnings as errors включён в `.csproj`).
- [ ] Три версии счёта (`Broken`, `Locked`, `Channel`) в отдельных файлах, с двуязычными XML-doc комментариями.
- [ ] `BrokenAccount` воспроизводит потерю обновлений в стресс-тесте (assert падает хотя бы в одном из 10 прогонов).
- [ ] `LockedAccount` проходит стресс-тест стабильно во всех 10 прогонах.
- [ ] `ChannelAccount` проходит стресс-тест стабильно; `Monitor Lock Contention/sec` близок к нулю.
- [ ] `DeadlockProneBank.Transfer` зависает в тесте с 5-секундным таймаутом; тест падает по таймауту, доказывая deadlock.
- [ ] `OrderedBank.Transfer` (с фикс по `Id`) проходит тот же тест без зависания.
- [ ] `Monitor.TryEnter`-версия (`TimeoutBank`) проходит тест, логируя неудачные попытки взятия второго лока через `Debug.Assert`/`Trace`.
- [ ] `Debug.Assert` в `Deposit`/`Withdraw` срабатывает на `BrokenAccount` под нагрузкой в Debug-сборке.
- [ ] `dotnet-counters` запущен на стресс-тесте; числа `Monitor Lock Contention/sec` зафиксированы в `README.md` для всех трёх версий.
- [ ] `dotnet-trace` + `speedscope` таймлайн показывает блокирующиеся потоки в deadlock-сценарии; описание в `README.md`.
- [ ] Live-lock воспроизведён (счётчик попыток уходит в тысячи) и исправлен рандомизированным backoff (завершается за единицы).
- [ ] Нигде нет `.Result`/`.Wait()` на async-коде; `ConfigureAwait(false)` во всех библиотечных async-методах.
- [ ] Нет `lock` через `await`; `SemaphoreSlim` используется корректно с `try/finally`.
- [ ] `CancellationToken` пробрасывается во все async-операции и `Channel.Writer.WriteAsync`.
- [ ] `README.md` описывает воспроизведённые симптомы, корневую причину каждой проблемы, применённое лекарство и измерения.

#### Подсказки (без прямого ответа)
- Чтобы «расширить окно» race и сделать баг воспроизводимым почти всегда, вставьте `Thread.SpinWait(100)` или `Task.Delay(1)` между чтением и записью в `BrokenAccount.Deposit` — но только в демо-версии, не в фиксе.
- Для упорядочивания локов используйте стабильное свойство счетов — `Id`. Если у счетов нет `Id`, добавьте; не используйте `GetHashCode()` (он не стабилен между запусками в некоторых средах).
- Для `Monitor.TryEnter` помните сигнатуру: `bool TryEnter(object, TimeSpan, ref bool lockTaken)` — всегда освобождайте только если `lockTaken == true`, в `finally`. Нестандартная сигнатура — источник багов.
- Для `Channel<T>` начните с `Channel.CreateBounded<int>(100)` — bounded-канал даёт backpressure и защищает от OOM под нагрузкой. Unbounded — только если потребитель гарантированно быстрее producer.
- Чтобы доказать conservation of money (сохранение суммы), сделайте снапшот: `long sumBefore = accounts.Sum(a => a.Balance);` … прогон … `long sumAfter = accounts.Sum(a => a.Balance);` и assert `sumBefore == sumAfter`. В `BrokenAccount` это не сработает из-за race; в `LockedAccount`/`ChannelAccount` — должно.
- Для live-lock фикса используйте `Random.Shared.Next(1, 10)` — он потокобезопасен в .NET 8 (раньше `new Random()` был не thread-safe).

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8 — ConcurrentBank: race / deadlock / live-lock / channel fix
// Двуязычные комментарии: RU + EN

using System.Collections.Concurrent;
using System.Diagnostics;
using System.Runtime.CompilerServices;
using System.Threading.Channels;

namespace ConcurrentBank.Domain;

// === Счёт-аккаунт: три версии / Account: three versions ===

// 1) СЛОМАНО: data race на _balance / BROKEN: data race on _balance
public sealed class BrokenAccount
{
    public int Id { get; }
    private int _balance; // ❌ нет синхронизации / no synchronization

    public int Balance => _balance; // ❌ тоже без барьера / no barrier either

    public BrokenAccount(int id, int initial) { Id = id; _balance = initial; }

    public void Deposit(int amount)
    {
        // _balance += amount НЕ атомарно: read + add + write.
        // Two threads can both read 5, both write 6 -> lost update.
        Debug.Assert(amount >= 0);
        _balance += amount;
    }

    public void Withdraw(int amount)
    {
        Debug.Assert(amount >= 0);
        _balance -= amount; // ❌ может уйти в отрицательное из-за race / can go negative due to race
    }
}

// 2) ПРАВИЛЬНО: lock + фиксированный порядок / CORRECT: lock + fixed ordering
public sealed class LockedAccount
{
    public int Id { get; }
    private int _balance;
    private readonly object _gate = new();

    public int Balance
    {
        get { lock (_gate) return _balance; } // чтение под локом / read under lock
    }

    public LockedAccount(int id, int initial)
    {
        Id = id; 
        lock (_gate) _balance = initial;
    }

    public void Deposit(int amount)
    {
        lock (_gate)
        {
            Debug.Assert(amount >= 0, "Deposit must be non-negative. / Депозит неотрицателен.");
            int previous = _balance;
            _balance += amount;
            Debug.Assert(_balance >= previous, "Balance must be monotonic. / Баланс монотонен.");
        }
    }

    public void Withdraw(int amount)
    {
        lock (_gate)
        {
            Debug.Assert(amount >= 0);
            Debug.Assert(_balance >= amount, "No overdraft. / Без овердрафта.");
            _balance -= amount;
        }
    }

    // Внутренний объект блокировки, видимый только банковским переводам.
    // Internal lock object, visible only to bank transfers.
    internal object Gate => _gate;
}

// === Банк со СЛОМАННЫМ переводом (deadlock через обратный порядок локов) ===
// Bank with BROKEN transfer (deadlock via reversed lock order)
public sealed class DeadlockProneBank
{
    public void Transfer(LockedAccount a, LockedAccount b, int amount)
    {
        // ❌ берём локи в порядке аргументов — возможен обратный порядок у другого вызова.
        //    Locks taken in argument order — another call may reverse it.
        lock (a.Gate)
            lock (b.Gate)
            {
                if (a.BalanceUnsafe() >= amount) // ANTI-PATTERN: прямой доступ к полю под одним локом
                {
                    a.WithdrawUnsafe(amount);
                    b.DepositUnsafe(amount);
                }
            }
    }
}

// === Банк с ФИКСОМ: упорядочивание по Id / Bank FIX: ordering by Id ===
public sealed class OrderedBank
{
    public void Transfer(LockedAccount a, LockedAccount b, int amount)
    {
        // Фиксированный порядок: сначала счет с меньшим Id. / Fixed order: lower Id first.
        var (first, second) = a.Id <= b.Id ? (a, b) : (b, a);
        lock (first.Gate)
            lock (second.Gate)
            {
                if (a.Balance >= amount)
                {
                    a.Withdraw(amount);
                    b.Deposit(amount);
                }
            }
    }
}

// === Банк с TryEnter + таймаутом / Bank with TryEnter + timeout ===
public sealed class TimeoutBank
{
    private static readonly TimeSpan Timeout = TimeSpan.FromMilliseconds(500);

    public bool Transfer(LockedAccount a, LockedAccount b, int amount)
    {
        var (first, second) = a.Id <= b.Id ? (a, b) : (b, a);
        bool gotFirst = false, gotSecond = false;
        try
        {
            Monitor.TryEnter(first.Gate, Timeout, ref gotFirst);
            if (!gotFirst) return false; // не зависаем / never block forever

            Monitor.TryEnter(second.Gate, Timeout, ref gotSecond);
            if (!gotSecond)
            {
                Debug.Assert(false, "Second lock failed — deadlock risk. / Второй лок не взят — риск дедлока.");
                return false;
            }

            if (a.Balance >= amount) { a.Withdraw(amount); b.Deposit(amount); return true; }
            return false;
        }
        finally
        {
            if (gotSecond) Monitor.Exit(second.Gate);
            if (gotFirst) Monitor.Exit(first.Gate);
        }
    }
}

// === Банк на Channel: без локов между счетами / Bank on Channel: no cross-account locks ===
public sealed class ChannelBank : IAsyncDisposable
{
    private sealed record TransferCmd(int From, int To, int Amount);

    private readonly ConcurrentDictionary<int, LockedAccount> _accounts = new();
    // Bounded channel: backpressure защитит от OOM / Bounded: backpressure prevents OOM.
    private readonly Channel<TransferCmd> _channel = Channel.CreateBounded<TransferCmd>(1000);
    private readonly CancellationTokenSource _cts = new();
    private readonly Task _processor;

    public ChannelBank(IEnumerable<LockedAccount> accounts)
    {
        foreach (var a in accounts) _accounts[a.Id] = a;
        _processor = Task.Run(() => ProcessAsync(_cts.Token));
    }

    // Producer: не трогает балансы напрямую, только шлёт команду.
    // Producer: never touches balances directly, only enqueues a command.
    public async ValueTask TransferAsync(int from, int to, int amount, CancellationToken ct)
    {
        await _channel.Writer.WriteAsync(new TransferCmd(from, to, amount), ct).ConfigureAwait(false);
    }

    // Единственный потребитель — последовательно применяет команды.
    // Single consumer — applies commands sequentially.
    private async Task ProcessAsync(CancellationToken ct)
    {
        await foreach (var cmd in _channel.Reader.ReadAllAsync(ct).ConfigureAwait(false))
        {
            if (_accounts.TryGetValue(cmd.From, out var a) && _accounts.TryGetValue(cmd.To, out var b))
            {
                // Один потребитель — достаточно локов на отдельных счетах,
                // межсчетных дедлоков нет по построению.
                // Single consumer — per-account locks suffice; cross-account
                // deadlocks are impossible by construction.
                if (a.Balance >= cmd.Amount) { a.Withdraw(cmd.Amount); b.Deposit(cmd.Amount); }
            }
        }
    }

    public async ValueTask DisposeAsync()
    {
        _channel.Writer.Complete();
        await _processor.ConfigureAwait(false);
        _cts.Cancel();
        _cts.Dispose();
    }
}

// === Live-lock и его фикс / Live-lock and its fix ===
public sealed class LiveLockDemo
{
    public int AttemptsBeforeFix { get; private set; }
    public int AttemptsWithFix { get; private set; }

    public async Task RunWithoutBackoffAsync(CancellationToken ct)
        => await RunAsync(useBackoff: false, ct).ConfigureAwait(false);

    public async Task RunWithBackoffAsync(CancellationToken ct)
        => await RunAsync(useBackoff: true, ct).ConfigureAwait(false);

    private async Task RunAsync(bool useBackoff, CancellationToken ct)
    {
        volatile bool aWants = true; volatile bool bWants = true;
        int attempts = 0;

        async Task Worker(Func<bool> me, Func<bool> other, Action<bool> setMe)
        {
            while (me())
            {
                if (other())
                {
                    setMe(false);
                    // Без backoff — live-lock. / Without backoff — live-lock.
                    // С backoff — случайная задержка разрывает симметрию. / With backoff — random delay breaks symmetry.
                    int ms = useBackoff ? Random.Shared.Next(1, 10) : 1;
                    await Task.Delay(ms, ct).ConfigureAwait(false);
                    setMe(true);
                }
                else { return; } // другой уступил — выходим / other yielded — exit
                Interlocked.Increment(ref attempts);
                if (attempts > 5000) return; // предохранитель / safety
            }
        }

        var ta = Task.Run(() => Worker(() => aWants, () => bWants, v => aWants = v), ct);
        var tb = Task.Run(() => Worker(() => bWants, () => aWants, v => bWants = v), ct);
        await Task.WhenAll(ta, tb).ConfigureAwait(false);

        if (useBackoff) AttemptsWithFix = attempts; else AttemptsBeforeFix = attempts;
    }
}

// Расширения для прямого доступа под уже взятым локом (только внутри критической секции банка).
// Extensions for direct access under an already-held lock (only inside bank critical section).
internal static class AccountUnsafeExtensions
{
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    internal static int BalanceUnsafe(this LockedAccount a) => a.Balance; // чтение под внешним локом / read under outer lock

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    internal static void DepositUnsafe(this LockedAccount a, int amount) => a.Deposit(amount);

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    internal static void WithdrawUnsafe(this LockedAccount a, int amount) => a.Withdraw(amount);
}
```

Разбор по строкам. `BrokenAccount._balance` — это намеренно сломанный счёт: поле без `lock`/`Interlocked`, поэтому `_balance += amount` теряет обновления под параллельной нагрузкой (в уроке это разобрано как data race). `Deposit`/`Withdraw` под `Debug.Assert` — первый эшелон диагностики из урока: в Debug-сборке ассерт на `amount >= 0` и на `_balance >= previous` поймает нарушение инварианта почти бесплатно. `LockedAccount` — правильная версия: `lock(_gate)` защищает `_balance`, `Balance` читает под локом (иначе другой поток может увидеть «разорванное» значение, что для `int` маловероятно, но для `long` на 32-бит — реально). `Gate` открыт внутренне (`internal`) — это намеренный компромисс: банк должен брать лок счета, но внешний код не должен.

`DeadlockProneBank.Transfer` берёт `lock(a.Gate)` затем `lock(b.Gate)` без упорядочивания — ровно анти-паттерн из урока: при `Transfer(x, y)` и `Transfer(y, x)` в разных потоках возникает классический deadlock через обратный порядок локов. `OrderedBank` применяет правило из урока «всегда берите локи в одинаковом порядке» — сортировкой по стабильному `Id`: `var (first, second) = a.Id <= b.Id ? (a, b) : (b, a)` (pattern matching / tuple deconstruction C# 12). Это устраняет deadlock по построению. `TimeoutBank` — альтернатива из урока: `Monitor.TryEnter(second, Timeout, ref gotSecond)` с таймаутом и освобождением в `finally`; `Debug.Assert(false)` при неудаче сигнализирует о риске (это диагноз, а не маска). Порядок освобождения обратный — `second` потом `first`, как требует `Monitor` (LIFO).

`ChannelBank` — лекарство третьего типа из урока: устранение синхронизации через `Channel<T>`. Producer только шлёт `TransferCmd` через `WriteAsync` (с `CancellationToken` и `ConfigureAwait(false)` — оба best practices из урока); единственный `ProcessAsync` применяет команды последовательно, поэтому межсчетных дедлоков нет по построению. `Channel.CreateBounded(1000)` даёт backpressure — producer блокируется `WriteAsync`, если очередь переполнена, защищая от OOM. `DisposeAsync` корректно завершает канал (`Writer.Complete`) и дождается потребителя. `LiveLockDemo` воспроизводит четвёртую проблему из урока: два потока бесконечно уступают друг другу через `volatile bool` флаги; счётчик `attempts` уходит в тысячи. Фикс — `Random.Shared.Next(1, 10)` (потокобезопасный в .NET 8): случайная задержка разрывает симметрию и live-lock коллапсирует в прогресс за единицы попыток. `AccountUnsafeExtensions` помечены `[MethodImpl(AggressiveInlining)]` и `internal` — микрооптимизация для критической секции банка; они вызываются только под уже взятым внешним локом, поэтому внутренний `lock` в `Deposit`/`Withdraw` — двойная защита (безопасно, но избыточно; в эталоне оставлено для наглядности).

#### Задания на углубление (бонус)
1. Реализуйте `Account` с балансом `decimal` (не `int`). `decimal` не атомарен даже на 64-бит — докажите, что `lock` обязателен, а `Interlocked` напрямую не работает. Измерьте замедление `lock` vs `Interlocked` на 1M операций через `BenchmarkDotNet`.
2. Добавьте четвёртую версию банка — `ConcurrentDictionaryBank`, где балансы хранятся в `ConcurrentDictionary<int, int>`, а переводы используют `AddOrUpdate` с фабрикой. Покажите, что это решает data race, но **не** решает multi-key atomicity (перевод между двумя счетами не атомарен). Обсудите, почему `Channel` здесь концептуально чище.
3. Подключите `dotnet-dump` + `lldb` (или WinDbg) на зависшем deadlock-процессе: выполните `~` (потоки), `!syncblk` (мониторы), `!clrstack` (стеки). Найдите два потока, ждущих друг друга, и приложите вывод в `README.md`.
4. Реализуйте воспроизведение race через `Parallel.For` с разным числом потоков (1, 2, 4, 8, 16) и постройте график «процент потерянных обновлений» от числа потоков. Объясните форму кривой (почему растёт, потом выходит на плато).

---

## Statement in English / Постановка на английском

#### Context & motivation
Imagine you maintain a training bank called "ConcurrentBank": several clerks concurrently execute transfers between accounts while a separate thread aggregates balance statistics. Under a load test the system balance "does not add up": the sum of all account balances differs from the sum of all operations executed. Sometimes the application freezes dead and has to be killed with `kill -9`. The team has already tried twice to "fix" the flaky tests with `Thread.Sleep`, but the bug returns in production as load grows. Your task is to turn the chaos into an engineering process: reproduce both problems deterministically (as far as that is ever possible for concurrency bugs), collect diagnostics, find the root cause, eliminate it with correct synchronization — or eliminate the synchronization entirely — and protect invariants so that a regression is caught in CI, not at a customer site.

This exercise is not an academic puzzle: it models exactly the class of defects that makes up the lion's share of incidents in multithreaded .NET services. You will learn to tell a data race (a lost update on `i++`) from a race condition (a logical error of operation order), from a deadlock (eternal waiting for resources), from a live-lock (useless activity). Each class needs its own cure: `Interlocked` for simple counters, fixed lock ordering or `Monitor.TryEnter` with a timeout for deadlocks, randomized backoff or an arbiter for live-lock, and `Channel<T>` plus immutable snapshots to remove shared state altogether. Diagnostics is half the work: without `Debug.Assert`, `dotnet-counters`, and a load stress test you will not prove that the bug is actually gone.

#### What to do step by step
1. Create a solution and three projects: a domain library, a console bug reproducer, and an xUnit stress-test project. Commands:
   ```bash
   dotnet new sln -n ConcurrentBank
   dotnet new classlib -n ConcurrentBank.Domain -o src/ConcurrentBank.Domain -f net8.0
   dotnet new console -n ConcurrentBank.Repro -o src/ConcurrentBank.Repro -f net8.0
   dotnet new xunit -n ConcurrentBank.Tests -o tests/ConcurrentBank.Tests -f net8.0
   dotnet sln add src/ConcurrentBank.Domain tests/ConcurrentBank.Tests src/ConcurrentBank.Repro
   dotnet add tests/ConcurrentBank.Tests reference src/ConcurrentBank.Domain
   dotnet add src/ConcurrentBank.Repro reference src/ConcurrentBank.Domain
   ```
2. In `ConcurrentBank.Domain` implement an `Account` class with an `int _balance` field and `Deposit`/`Withdraw`/`Transfer` methods. Make **three versions**: `BrokenAccount` (no synchronization — reproduces a data race on `_balance`), `LockedAccount` (with `lock` and a fixed lock acquisition order on transfer — lower `Id` first, then higher), and `ChannelAccount` (transfers are sent as messages into a `Channel<TransferCommand>` and a single consumer applies them sequentially — no locks between accounts).
3. Implement a `DeadlockProneBank` class with a `Transfer(Account a, Account b, int amount)` method that takes `lock(a._gate)` then `lock(b._gate)` — with no ordering by `Id`. Run a parallel test: `Parallel.For(0, 1000, i => Transfer(accounts[i % N], accounts[(i + 1) % N], 10))`. It should hang. Capture the hang as a "reproduced bug" (e.g., a test with a `CancellationTokenSource` of 5 seconds that fails by timeout — that is the deadlock proof).
4. In `ConcurrentBank.Repro/Program.cs` (top-level statements) implement a menu: `1) race repro 2) deadlock repro 3) live-lock repro 4) channel fix 5) counters live`. For each option run the corresponding scenario and print the result. For option 5 print the instruction to run `dotnet-counters monitor --process-id <pid> System.Runtime` in a separate terminal (give the PID).
5. Add `Debug.Assert` to `Deposit`/`Withdraw`: balance cannot go negative; the sum of all deposits is monotonically non-decreasing. Run a Debug build under load — asserts should fire on `BrokenAccount` and stay silent on `LockedAccount`/`ChannelAccount`.
6. Write an xUnit stress test `StressTests`: for each version of the account run `Parallel.For(0, 8, _ => { for (int i = 0; i < 100_000; i++) account.Deposit(1); })`, then assert `account.Balance == 800_000`. `BrokenAccount` should fail (or at least sometimes), the others should pass reliably. Repeat the test 10 times in a loop — races show up statistically.
7. Run `dotnet-counters` on the stress test and capture `Monitor Lock Contention/sec` for `LockedAccount`. Compare with `ChannelAccount` (where contention should be near zero, because there is a single consumer). Write the numbers into the `README.md` of your solution.
8. Use `dotnet-trace collect --providers Microsoft-Windows-DotNETRuntime` on a 10-second run of the deadlock scenario, convert to `speedscope`, and find on the timeline the threads sitting on `Monitor.Enter`. Attach a screenshot or a description to `README.md`.
9. Reproduce live-lock: two threads "yield" a resource to each other via `volatile bool` flags in a loop, each sees the other wants it and withdraws its own request — forever. Fix it with a randomized `Task.Delay(Random.Shared.Next(1, 10))`. Show that before the fix the attempt counter goes into the thousands, after — it terminates in single digits.
10. Remove every `.Result`/`.Wait()` from the code. If somewhere you must wait on an async operation synchronously (for example in `Main`), use `Task.Run(() => ...).GetAwaiter().GetResult()` only as a temporary crutch and add a comment explaining why it is safe in that context (no captured `SynchronizationContext`).

#### Requirements
- Target framework `net8.0`; language C# 12 (top-level statements in `Program.cs`, collection expressions to initialize account lists `[new Account(1, 100), new Account(2, 200)]`, pattern matching to parse commands, raw string literals for multi-line help text).
- Every shared mutable field is protected by one of: `Interlocked`, `lock`, `ConcurrentDictionary`, `Channel<T>`, or marked `volatile`/`Volatile` for visibility flags. No unprotected `int` fields read or written from multiple threads.
- The acquisition order of multiple locks is fixed and documented — for example, "always lock the account with the lower `Id` first." This rule is followed in **all** methods with no exceptions.
- In async code, `lock` is replaced by `SemaphoreSlim(1,1)` with `WaitAsync`/`Release` in `try/finally`; no plain `lock` across `await` (the compiler forbids it anyway, but a `SemaphoreSlim` held across a long `await` is also unacceptable — the critical section must be short and await-free).
- In library code (`ConcurrentBank.Domain`) every async method uses `ConfigureAwait(false)`; `CancellationToken` is propagated to every `async` method and to `Channel.Writer.WriteAsync`.
- The CI test includes a stress test of N threads × M iterations run several times; `Debug.Assert` guards key invariants (non-negative balance, monotonicity, conservation of money — the sum across all accounts before and after a series of transfers is the same).
- Diagnostics: the `README.md` describes the reproduced symptoms, the `Monitor Lock Contention/sec` numbers for each version, and the findings from `dotnet-trace`/`speedscope`.
- The anti-patterns from the lesson (`Thread.Sleep` in production code as a "fix", `.Result` on async under a UI/ASP.NET Classic context, `async void` without try/catch, forgotten `Volatile.Read`) are absent; where they appear intentionally as demos they are marked `// ANTI-PATTERN — demo only`.

#### Pitfalls
- `i++` on an `int` is **not atomic** in C# even on 64-bit: it is a read + add + write. Two threads can both read `5` and both write `6` — one update is lost. The cure is `Interlocked.Increment(ref _value)`, not `volatile` (the latter gives visibility but not atomicity of the increment). This is the most common beginner mistake: "I added `volatile`, so it is fine."
- Reversed lock order in `Transfer(a, b)` vs `Transfer(b, a)` is the classic deadlock. The fix is ordering by a stable key (`Id`): `var (first, second) = a.Id <= b.Id ? (a, b) : (b, a);`. The alternative is `Monitor.TryEnter(second, timeout, ref got)` with a rollback and release of the first lock. Never "fix" a deadlock with `Thread.Sleep` between locks — it is a mask; load changes and the bug returns.
- `lock` + `await` is forbidden: the compiler rejects `lock` in an `async` method, but `SemaphoreSlim.WaitAsync` plus a long `await` inside is the same problem (others wait, the section is bloated). Keep the critical section short; move long work out through a `Channel` and outside the lock.
- `.Result`/`.Wait()` on an async method under a UI/ASP.NET Classic synchronization context is the classic deadlock: the continuation waits for the context, which is blocked by `.Result`. In .NET 8 ASP.NET Core has no such context (less danger), but still put `ConfigureAwait(false)` in library code — it is insurance against being called from WPF/WinForms/ASP.NET Classic.
- `Channel<T>` is not free: a bounded channel with a small capacity can block the producer through `WriteAsync` (that is normal — backpressure), but if the consumer throws an exception the producer will hang forever — always propagate `CancellationToken` and handle `ChannelClosedException`.
- `Debug.Assert` is compiled out in Release — do not use it for runtime validation in production; use `Trace.Assert` or an explicit `throw` there. But for catching races in CI/Debug it is ideal: nearly free.
- `Volatile.Read`/`Volatile.Write` are needed for visibility flags between threads when you do not use `lock`. A plain `bool _stop` without `volatile` may never become visible to the other thread (JIT may cache the read in a register). `volatile bool` or `Volatile.Read` solves it.
- The stress test must repeat many times — races show up statistically. A single run can pass by luck. Minimum: 10 repeats of 100k iterations on 8 threads.
- `dotnet-counters` has an overhead of about 1–2% — you can keep it on in production. `dotnet-trace` is heavier; use short 10–30 second sessions.

#### Acceptance criteria
- [ ] The solution builds with `dotnet build` with no warnings (treat warnings as errors is on in `.csproj`).
- [ ] Three account versions (`Broken`, `Locked`, `Channel`) in separate files, with bilingual XML-doc comments.
- [ ] `BrokenAccount` reproduces a lost update in the stress test (the assert fails in at least one of 10 runs).
- [ ] `LockedAccount` passes the stress test reliably in all 10 runs.
- [ ] `ChannelAccount` passes the stress test reliably; `Monitor Lock Contention/sec` is near zero.
- [ ] `DeadlockProneBank.Transfer` hangs in the 5-second-timeout test; the test fails by timeout, proving the deadlock.
- [ ] `OrderedBank.Transfer` (with the `Id`-based fix) passes the same test without hanging.
- [ ] The `Monitor.TryEnter` version (`TimeoutBank`) passes the test, logging failed second-lock attempts via `Debug.Assert`/`Trace`.
- [ ] `Debug.Assert` in `Deposit`/`Withdraw` fires on `BrokenAccount` under load in a Debug build.
- [ ] `dotnet-counters` is run on the stress test; the `Monitor Lock Contention/sec` numbers for all three versions are in `README.md`.
- [ ] The `dotnet-trace` + `speedscope` timeline shows blocked threads in the deadlock scenario; a description is in `README.md`.
- [ ] Live-lock is reproduced (the attempt counter goes into the thousands) and fixed with randomized backoff (terminates in single digits).
- [ ] No `.Result`/`.Wait()` on async code anywhere; `ConfigureAwait(false)` in every library async method.
- [ ] No `lock` across `await`; `SemaphoreSlim` is used correctly with `try/finally`.
- [ ] `CancellationToken` is propagated to every async operation and to `Channel.Writer.WriteAsync`.
- [ ] `README.md` describes the reproduced symptoms, the root cause of each problem, the applied cure, and the measurements.

#### Hints (no direct answer)
- To "widen the window" of a race and make the bug reproducible almost always, insert `Thread.SpinWait(100)` or `Task.Delay(1)` between the read and the write in `BrokenAccount.Deposit` — but only in the demo version, not in the fix.
- For lock ordering use a stable property of the accounts — `Id`. If the accounts have no `Id`, add one; do not use `GetHashCode()` (it is not stable across runs in some environments).
- For `Monitor.TryEnter` remember the signature: `bool TryEnter(object, TimeSpan, ref bool lockTaken)` — only release if `lockTaken == true`, in `finally`. The unusual signature is a source of bugs.
- For `Channel<T>` start with `Channel.CreateBounded<int>(100)` — a bounded channel gives backpressure and protects from OOM under load. Unbounded — only if the consumer is guaranteed faster than the producer.
- To prove conservation of money, take a snapshot: `long sumBefore = accounts.Sum(a => a.Balance);` … run … `long sumAfter = accounts.Sum(a => a.Balance);` and assert `sumBefore == sumAfter`. On `BrokenAccount` it will not hold because of the race; on `LockedAccount`/`ChannelAccount` it should.
- For the live-lock fix use `Random.Shared.Next(1, 10)` — it is thread-safe in .NET 8 (earlier `new Random()` was not).

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8 — ConcurrentBank: race / deadlock / live-lock / channel fix
// Bilingual comments: EN + RU

using System.Collections.Concurrent;
using System.Diagnostics;
using System.Runtime.CompilerServices;
using System.Threading.Channels;

namespace ConcurrentBank.Domain;

// === Account: three versions ===

// 1) BROKEN: data race on _balance
public sealed class BrokenAccount
{
    public int Id { get; }
    private int _balance; // ❌ no synchronization

    public int Balance => _balance; // ❌ no barrier either

    public BrokenAccount(int id, int initial) { Id = id; _balance = initial; }

    public void Deposit(int amount)
    {
        // _balance += amount is NOT atomic: read + add + write.
        // Two threads can both read 5, both write 6 -> lost update.
        Debug.Assert(amount >= 0);
        _balance += amount;
    }

    public void Withdraw(int amount)
    {
        Debug.Assert(amount >= 0);
        _balance -= amount; // ❌ can go negative due to race
    }
}

// 2) CORRECT: lock + fixed ordering
public sealed class LockedAccount
{
    public int Id { get; }
    private int _balance;
    private readonly object _gate = new();

    public int Balance
    {
        get { lock (_gate) return _balance; } // read under lock
    }

    public LockedAccount(int id, int initial)
    {
        Id = id;
        lock (_gate) _balance = initial;
    }

    public void Deposit(int amount)
    {
        lock (_gate)
        {
            Debug.Assert(amount >= 0, "Deposit must be non-negative. / Депозит неотрицателен.");
            int previous = _balance;
            _balance += amount;
            Debug.Assert(_balance >= previous, "Balance must be monotonic. / Баланс монотонен.");
        }
    }

    public void Withdraw(int amount)
    {
        lock (_gate)
        {
            Debug.Assert(amount >= 0);
            Debug.Assert(_balance >= amount, "No overdraft. / Без овердрафта.");
            _balance -= amount;
        }
    }

    // Internal lock object, visible only to bank transfers.
    // Внутренний объект блокировки, видимый только банковским переводам.
    internal object Gate => _gate;
}

// === Bank with BROKEN transfer (deadlock via reversed lock order) ===
public sealed class DeadlockProneBank
{
    public void Transfer(LockedAccount a, LockedAccount b, int amount)
    {
        // ❌ Locks taken in argument order — another call may reverse it.
        //    Берём локи в порядке аргументов — возможен обратный порядок у другого вызова.
        lock (a.Gate)
            lock (b.Gate)
            {
                if (a.BalanceUnsafe() >= amount)
                {
                    a.WithdrawUnsafe(amount);
                    b.DepositUnsafe(amount);
                }
            }
    }
}

// === Bank FIX: ordering by Id ===
public sealed class OrderedBank
{
    public void Transfer(LockedAccount a, LockedAccount b, int amount)
    {
        // Fixed order: lower Id first. / Фиксированный порядок: сначала меньший Id.
        var (first, second) = a.Id <= b.Id ? (a, b) : (b, a);
        lock (first.Gate)
            lock (second.Gate)
            {
                if (a.Balance >= amount)
                {
                    a.Withdraw(amount);
                    b.Deposit(amount);
                }
            }
    }
}

// === Bank with TryEnter + timeout ===
public sealed class TimeoutBank
{
    private static readonly TimeSpan Timeout = TimeSpan.FromMilliseconds(500);

    public bool Transfer(LockedAccount a, LockedAccount b, int amount)
    {
        var (first, second) = a.Id <= b.Id ? (a, b) : (b, a);
        bool gotFirst = false, gotSecond = false;
        try
        {
            Monitor.TryEnter(first.Gate, Timeout, ref gotFirst);
            if (!gotFirst) return false; // never block forever / не зависаем

            Monitor.TryEnter(second.Gate, Timeout, ref gotSecond);
            if (!gotSecond)
            {
                Debug.Assert(false, "Second lock failed — deadlock risk. / Второй лок не взят — риск дедлока.");
                return false;
            }

            if (a.Balance >= amount) { a.Withdraw(amount); b.Deposit(amount); return true; }
            return false;
        }
        finally
        {
            if (gotSecond) Monitor.Exit(second.Gate);
            if (gotFirst) Monitor.Exit(first.Gate);
        }
    }
}

// === Bank on Channel: no cross-account locks ===
public sealed class ChannelBank : IAsyncDisposable
{
    private sealed record TransferCmd(int From, int To, int Amount);

    private readonly ConcurrentDictionary<int, LockedAccount> _accounts = new();
    // Bounded channel: backpressure prevents OOM / Bounded-канал: backpressure защитит от OOM.
    private readonly Channel<TransferCmd> _channel = Channel.CreateBounded<TransferCmd>(1000);
    private readonly CancellationTokenSource _cts = new();
    private readonly Task _processor;

    public ChannelBank(IEnumerable<LockedAccount> accounts)
    {
        foreach (var a in accounts) _accounts[a.Id] = a;
        _processor = Task.Run(() => ProcessAsync(_cts.Token));
    }

    // Producer: never touches balances directly, only enqueues a command.
    // Producer: не трогает балансы напрямую, только шлёт команду.
    public async ValueTask TransferAsync(int from, int to, int amount, CancellationToken ct)
    {
        await _channel.Writer.WriteAsync(new TransferCmd(from, to, amount), ct).ConfigureAwait(false);
    }

    // Single consumer — applies commands sequentially.
    // Единственный потребитель — последовательно применяет команды.
    private async Task ProcessAsync(CancellationToken ct)
    {
        await foreach (var cmd in _channel.Reader.ReadAllAsync(ct).ConfigureAwait(false))
        {
            if (_accounts.TryGetValue(cmd.From, out var a) && _accounts.TryGetValue(cmd.To, out var b))
            {
                // Single consumer — per-account locks suffice; cross-account
                // deadlocks are impossible by construction.
                // Один потребитель — достаточно локов на отдельных счетах.
                if (a.Balance >= cmd.Amount) { a.Withdraw(cmd.Amount); b.Deposit(cmd.Amount); }
            }
        }
    }

    public async ValueTask DisposeAsync()
    {
        _channel.Writer.Complete();
        await _processor.ConfigureAwait(false);
        _cts.Cancel();
        _cts.Dispose();
    }
}

// === Live-lock and its fix ===
public sealed class LiveLockDemo
{
    public int AttemptsBeforeFix { get; private set; }
    public int AttemptsWithFix { get; private set; }

    public async Task RunWithoutBackoffAsync(CancellationToken ct)
        => await RunAsync(useBackoff: false, ct).ConfigureAwait(false);

    public async Task RunWithBackoffAsync(CancellationToken ct)
        => await RunAsync(useBackoff: true, ct).ConfigureAwait(false);

    private async Task RunAsync(bool useBackoff, CancellationToken ct)
    {
        volatile bool aWants = true; volatile bool bWants = true;
        int attempts = 0;

        async Task Worker(Func<bool> me, Func<bool> other, Action<bool> setMe)
        {
            while (me())
            {
                if (other())
                {
                    setMe(false);
                    // Without backoff — live-lock. / Без backoff — live-lock.
                    // With backoff — random delay breaks symmetry. / С backoff — случайная задержка разрывает симметрию.
                    int ms = useBackoff ? Random.Shared.Next(1, 10) : 1;
                    await Task.Delay(ms, ct).ConfigureAwait(false);
                    setMe(true);
                }
                else { return; } // other yielded — exit / другой уступил — выходим
                Interlocked.Increment(ref attempts);
                if (attempts > 5000) return; // safety / предохранитель
            }
        }

        var ta = Task.Run(() => Worker(() => aWants, () => bWants, v => aWants = v), ct);
        var tb = Task.Run(() => Worker(() => bWants, () => aWants, v => bWants = v), ct);
        await Task.WhenAll(ta, tb).ConfigureAwait(false);

        if (useBackoff) AttemptsWithFix = attempts; else AttemptsBeforeFix = attempts;
    }
}

// Extensions for direct access under an already-held lock (only inside bank critical section).
// Расширения для прямого доступа под уже взятым локом (только внутри критической секции банка).
internal static class AccountUnsafeExtensions
{
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    internal static int BalanceUnsafe(this LockedAccount a) => a.Balance;

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    internal static void DepositUnsafe(this LockedAccount a, int amount) => a.Deposit(amount);

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    internal static void WithdrawUnsafe(this LockedAccount a, int amount) => a.Withdraw(amount);
}
```

Line-by-line walk-through. `BrokenAccount._balance` is an intentionally broken account: a field with no `lock`/`Interlocked`, so `_balance += amount` loses updates under parallel load (this is the data race from the lesson). `Deposit`/`Withdraw` under `Debug.Assert` are the first diagnostic line from the lesson: in a Debug build the asserts on `amount >= 0` and `_balance >= previous` will catch an invariant violation almost for free. `LockedAccount` is the correct version: `lock(_gate)` guards `_balance`, `Balance` reads under lock (otherwise another thread could see a "torn" value, unlikely for `int` but real for `long` on 32-bit). `Gate` is exposed internally (`internal`) — a deliberate trade-off: the bank must take an account lock, but external code must not.

`DeadlockProneBank.Transfer` takes `lock(a.Gate)` then `lock(b.Gate)` with no ordering — exactly the anti-pattern from the lesson: with `Transfer(x, y)` and `Transfer(y, x)` on different threads you get the classic deadlock from reversed lock order. `OrderedBank` applies the lesson rule "always acquire locks in the same order" — by sorting on the stable `Id`: `var (first, second) = a.Id <= b.Id ? (a, b) : (b, a)` (pattern matching / tuple deconstruction C# 12). This removes the deadlock by construction. `TimeoutBank` is the alternative from the lesson: `Monitor.TryEnter(second, Timeout, ref gotSecond)` with a timeout and release in `finally`; the `Debug.Assert(false)` on failure signals the risk (a diagnosis, not a mask). Release order is reversed — `second` then `first` — as `Monitor` requires (LIFO).

`ChannelBank` is the third cure from the lesson: removing synchronization via `Channel<T>`. The producer only sends a `TransferCmd` through `WriteAsync` (with `CancellationToken` and `ConfigureAwait(false)` — both lesson best practices); a single `ProcessAsync` applies commands sequentially, so cross-account deadlocks are impossible by construction. `Channel.CreateBounded(1000)` gives backpressure — the producer blocks on `WriteAsync` when the queue is full, protecting from OOM. `DisposeAsync` correctly completes the channel (`Writer.Complete`) and awaits the consumer. `LiveLockDemo` reproduces the fourth problem from the lesson: two threads forever yield to each other through `volatile bool` flags; the `attempts` counter goes into the thousands. The fix is `Random.Shared.Next(1, 10)` (thread-safe in .NET 8): the random delay breaks the symmetry and the live-lock collapses into progress within a few attempts. `AccountUnsafeExtensions` are marked `[MethodImpl(AggressiveInlining)]` and `internal` — a micro-optimization for the bank's critical section; they are called only under an already-held outer lock, so the inner `lock` in `Deposit`/`Withdraw` is a double guard (safe but redundant; kept in the reference for clarity).

#### Going deeper (bonus)
1. Implement `Account` with a `decimal` balance (not `int`). `decimal` is not atomic even on 64-bit — prove that `lock` is mandatory and that `Interlocked` does not work directly. Measure the slowdown of `lock` vs `Interlocked` on 1M operations with `BenchmarkDotNet`.
2. Add a fourth bank version — `ConcurrentDictionaryBank`, where balances live in a `ConcurrentDictionary<int, int>` and transfers use `AddOrUpdate` with a factory. Show that this solves the data race but **not** the multi-key atomicity (a transfer between two accounts is not atomic). Discuss why `Channel` is conceptually cleaner here.
3. Attach `dotnet-dump` + `lldb` (or WinDbg) to a hung deadlock process: run `~` (threads), `!syncblk` (monitors), `!clrstack` (stacks). Find the two threads waiting for each other and attach the output to `README.md`.
4. Reproduce the race with `Parallel.For` at different thread counts (1, 2, 4, 8, 16) and plot "percentage of lost updates" against thread count. Explain the curve shape (why it grows, then plateaus).

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Решение в репозитории, собирается без warnings, `dotnet build` зелёный.
- [ ] (RU) Три версии счёта в отдельных файлах с двуязычными XML-doc.
- [ ] (RU) Стресс-тест ловит баг `BrokenAccount` и проходит на `LockedAccount`/`ChannelAccount`.
- [ ] (RU) Дедлок воспроизведён тестом с таймаутом и устранён `OrderedBank` и `TimeoutBank`.
- [ ] (RU) `Debug.Assert` срабатывает на сломанной версии в Debug-сборке.
- [ ] (RU) `dotnet-counters` и `dotnet-trace`/`speedscope` описаны в `README.md` с числами.
- [ ] (RU) Live-lock воспроизведён и исправлен backoff.
- [ ] (RU) Нет `.Result`/`.Wait()`; `ConfigureAwait(false)` везде в библиотеке.
- [ ] (RU) `CancellationToken` пробрасывается во все async-операции и `Channel`.
- [ ] (EN) Solution is in the repo, builds with no warnings, `dotnet build` is green.
- [ ] (EN) Three account versions in separate files with bilingual XML-doc.
- [ ] (EN) Stress test catches the `BrokenAccount` bug and passes on `LockedAccount`/`ChannelAccount`.
- [ ] (EN) Deadlock reproduced by a timeout test and fixed by `OrderedBank` and `TimeoutBank`.
- [ ] (EN) `Debug.Assert` fires on the broken version in a Debug build.
- [ ] (EN) `dotnet-counters` and `dotnet-trace`/`speedscope` are described in `README.md` with numbers.
- [ ] (EN) Live-lock reproduced and fixed with backoff.
- [ ] (EN) No `.Result`/`.Wait()`; `ConfigureAwait(false)` everywhere in the library.
- [ ] (EN) `CancellationToken` propagated to all async operations and the `Channel`.

#### Ресурсы / Resources
- [Microsoft Learn — Managed Threading Basics](https://learn.microsoft.com/dotnet/standard/threading/managed-threading-basics)
- [dotnet-counters](https://learn.microsoft.com/dotnet/core/diagnostics/dotnet-counters)
- [dotnet-trace](https://learn.microsoft.com/dotnet/core/diagnostics/dotnet-trace)
- [dotnet-dump](https://learn.microsoft.com/dotnet/core/diagnostics/dotnet-dump)
- [Concurrency Visualizer](https://learn.microsoft.com/visualstudio/profiling/concurrency-visualizer)
- [Channel&lt;T&gt;](https://learn.microsoft.com/dotnet/api/system.threading.channels.channel-1)
- [Interlocked Class](https://learn.microsoft.com/dotnet/api/system.threading.interlocked)
- [Monitor.TryEnter](https://learn.microsoft.com/dotnet/api/system.threading.monitor.tryenter)
- [Speedscope](https://www.speedscope.app/)

---

[← К уроку M11-L09](lesson-M11-L09-deadlocks-race-conditions.md) | [⬆ К модулю M11](../README.md) | [Следующее ДЗ →](homework-M11-L10-threadpool-longrunning.md)
