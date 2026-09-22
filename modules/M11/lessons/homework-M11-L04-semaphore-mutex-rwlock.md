---
[← К уроку M11-L04](lesson-M11-L04-semaphore-mutex-rwlock.md) | [⬆ К модулю M11](../README.md) | [Предыдущее ДЗ ←](homework-M11-L03-interlocked.md) | [Следующее ДЗ →](homework-M11-L05-volatile-memory-model.md)
---

### Домашнее задание M11-L04: SemaphoreSlim, Mutex, ReaderWriterLockSlim / Homework M11-L04: SemaphoreSlim, Mutex, ReaderWriterLockSlim

**Урок / Lesson:** M11-L04
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться осознанно выбирать и правильно применять три специализированных примитива синхронизации — `SemaphoreSlim` (ограничение степени параллелизма в async-коде), `Mutex` (взаимное исключение между процессами) и `ReaderWriterLockSlim` (много читателей / один писатель) — избегая типичных антипаттернов: thread pool starvation, потерянных разрешений, дедлоков при повышении read→write. (EN) Learn to consciously choose and correctly apply three specialized synchronization primitives — `SemaphoreSlim` (bounding async concurrency), `Mutex` (cross-process mutual exclusion) and `ReaderWriterLockSlim` (many readers / single writer) — while avoiding the classic anti-patterns: thread pool starvation, lost permits, and read→write upgrade deadlocks.

#### Связь с уроком / Connection to the lesson
(RU) Урок M11-L04 показывает, что обычный `lock`/`Monitor` не выражает правил «не более N одновременных», «синхронизация между процессами» и «читатели параллельны, писатель эксклюзивен». Это ДЗ закрепляет именно эти три сценария через одну сборку `ConcurrencyToolkit`, где каждый примитив решает свою задачу и проверяется под нагрузкой. Особое внимание уделяется асинхронному ожиданию `WaitAsync`, `try/finally` для `Release`/`Exit*` и корректному повышению read→write через upgradeable-режим.
(EN) Lesson M11-L04 shows that a plain `lock`/`Monitor` cannot express “at most N concurrent”, “synchronize across processes”, or “readers in parallel, writer exclusive”. This homework drills exactly those three scenarios through a single `ConcurrencyToolkit` assembly, where each primitive owns one problem and is stress-tested. Special attention goes to the async-aware `WaitAsync`, to `try/finally` around `Release`/`Exit*`, and to a correct read→write promotion through the upgradeable mode.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация
Вы присоединились к команде, которая пишет внутренний сервис «Polyglot Fetcher»: он параллельно тянет метаданные из множества внешних HTTP-источников, кэширует результаты в памяти и должен гарантировать, что на одной машине запущен только один экземпляр сервиса (иначе два процесса начнут дублировать запросы и перегружать внешние API). Команда уже наступила на классические грабли: кто-то написал `semaphore.Wait()` в async-методе, и при росте нагрузки пул потоков истощался — сервис «замирал». Кто-то забыл `try/finally` вокруг `Release()`, и после первого же исключения семафор навсегда потерял одно разрешение, снизив пропускную способность. В кэше попытались сделать `GetOrAdd` через `EnterReadLock`, потом отпустить и тут же `EnterWriteLock` — и поймали race condition, при котором два потока дважды вычисляли фабрику. Наконец, для single-instance кто-то взял `Mutex` без префикса `Global\`, и в терминальной сессии другого пользователя запуск второго экземпляра не блокировался.

Ваша задача — переписать три ключевых компонента так, чтобы они были идемпотентны, потокобезопасны, async-friendly и устойчивы к отмене через `CancellationToken`. Вы будете использовать ровно те примитивы, которые рассматривались в уроке: `SemaphoreSlim` для throttling, `Mutex` для single-instance и `ReaderWriterLockSlim` для кэша с читателями и писателями. Никаких `lock` вокруг `await`, никаких `Wait()` в async-методах, никаких «отпустил-перечитал». Код должен компилироваться под .NET 8 на C# 12 и проходить нагрузочный тест без дедлоков, без `SemaphoreFullException` и без потерянных разрешений.

#### Что нужно сделать (пошагово)
1. Создайте решение и три проекта: `dotnet new sln -n ConcurrencyToolkit`, затем `dotnet new classlib -n ConcurrencyToolkit -f net8.0`, `dotnet new xunit -n ConcurrencyToolkit.Tests -f net8.0`, добавьте тест-проект в решение и ссылку `dotnet add ConcurrencyToolkit.Tests reference ConcurrencyToolkit`. В тестах используйте `Microsoft.NET.Test.Sdk` и `xunit`.
2. В проекте `ConcurrencyToolkit` создайте класс `ThrottledHttpClient`, который принимает `int maxConcurrency` и `HttpClient` (внедрение через конструктор). Внутри он хранит `SemaphoreSlim _gate = new(maxConcurrency, maxConcurrency)`. Метод `Task<HttpResponseMessage> GetAsync(Uri url, CancellationToken ct)` должен: `await _gate.WaitAsync(ct).ConfigureAwait(false)`, в `try` выполнить `_client.GetAsync(url, ct)`, в `finally` вызвать `_gate.Release()` ровно один раз. Добавьте счётчик активных запросов `Interlocked.Increment`/`Decrement` и в `Debug.Assert` проверьте, что он никогда не превышает `maxConcurrency`.
3. Реализуйте `SingleInstanceGuard : IDisposable`. Конструктор принимает строку `mutexName` (например `@"Global\PolyglotFetcher.SingleInstance.v1"`), создаёт `new Mutex(initiallyOwned: true, mutexName, out var createdNew)`, сохраняет `HasOwnership = createdNew`. В `Dispose()` если `HasOwnership`, вызовите `ReleaseMutex()` в `try/catch (ApplicationException)`, затем `_mutex.Dispose()`. Добавьте метод `WaitForOwnershipAsync(TimeSpan timeout, CancellationToken ct)`, который для отзывчивого ожидания использует `Task.Run(() => mutex.WaitOne(timeout), ct)` — помните, что `WaitOne` напрямую не принимает `CancellationToken`.
4. Реализуйте обобщённый `ThreadSafeCache<TKey, TValue> where TKey : notnull` на `ReaderWriterLockSlim` с `LockRecursionPolicy.NoRecursion` (рекурсию отключаем намеренно, чтобы ловить ошибки дизайна). Методы: `TValue Read(TKey)`, `void Write(TKey, TValue)`, `TValue GetOrAdd(TKey, Func<TKey, TValue> factory)`. `GetOrAdd` обязан идти через `EnterUpgradeableReadLock`, внутри — `TryGetValue`; если нет, `EnterWriteLock` с двойной проверкой (`TryGetValue` ещё раз), вызов фабрики и сохранение. Каждый `Enter*` парный с `Exit*` в `try/finally`.
5. Напишите нагрузочный тест `StressTests`: запустите 50 задач, каждая делает 200 операций над кэшем (80 % `GetOrAdd`, 20 % `Write`), параллельно дёргайте `ThrottledHttpClient` с `maxConcurrency = 4` по 20 фиктивным URL (через `HttpMessageHandler`-заглушку с задержкой 10 мс). Проверьте, что `activeCount` никогда не превысил 4, что кэш не выбросил и что `Dispose` не упал.
6. Запустите `dotnet test` и убедитесь, что всё зелёное. Запустите `dotnet build -c Release` и проверьте отсутствие предупреждений `CA` (если включены анализаторы). Сделайте `dotnet run --project ConcurrencyToolkit.Tests` для стресс-теста отдельно.

#### Требования к решению
- Целевой фреймворк — `net8.0`, язык C# 12: разрешены top-level statements в тестах, collection expressions (`new()` вывод, `[]` для массивов), pattern matching, `required`, `init`, raw string literals где уместно. Запрещён `lock (obj) { await ... }`.
- Все ожидания на примитивах синхронизации принимают `CancellationToken`: `WaitAsync(ct)` для семафора; для `Mutex` — обёртка через `Task.Run` + `WaitOne(timeout)`, поскольку `WaitOne` напрямую не принимает `ct`. Обязательно ловить и корректно обрабатывать `OperationCanceledException`.
- `Release()` на семафоре — строго один раз на каждый успешный `WaitAsync` и только в `finally`. `ExitReadLock`/`ExitWriteLock`/`ExitUpgradeableReadLock` — строго попарно с `Enter*` в `try/finally`. Никаких «отпустил read, затем взял write» в `GetOrAdd` — только upgradeable.
- `Mutex` для single-instance обязан использовать префикс `Global\`, если нужна межсессийная видимость; освобождать ровно столько раз, сколько владели; проверять `createdNew`. Реентерабельность учитывать, но не злоупотреблять.
- `ReaderWriterLockSlim` создаётся с `LockRecursionPolicy.NoRecursion` (если в вашем дизайне нет веской причины для `SupportsRecursion` — в этом ДЗ её нет). Критические секции максимально короткие, без `await` под захваченной блокировкой.
- Код должен быть готов к отмене: при `OperationCanceledException` semaphore разрешение всё равно возвращается через `finally`; `Mutex` при отмене `WaitOne` не владеет мьютексом, поэтому `ReleaseMutex` не вызывается.

#### Тонкости и подводные камни
- Самая частая ошибка — `_gate.Wait()` в async-методе. В уроке подчёркнуто: синхронный `Wait()` блокирует поток пула, и под нагрузкой вы получаете thread pool starvation — сервис «зависает». Всегда `await _gate.WaitAsync(ct)`.
- `Release()` вне `try/finally` — при исключении в `GetAsync` разрешение «теряется», и семафор навсегда уменьшает пропускную способность. Счётчик разрешений становится 3 из 4, и вы долго ищете причину деградации. Поэтому `Release` — только в `finally`.
- `semaphore.Release()` дважды (или больше, чем было `WaitAsync`) даёт `SemaphoreFullException`. Считайте входы и выходы строго 1:1. Не вызывайте `Release` в ветке `catch`, если `WaitAsync` ещё не вернулся успешно.
- `Mutex` без `Global\` в терминальных сессиях — разные пользователи получают разные мьютексы, и single-instance ломается. Используйте `Global\` для системной синхронизации; `Local\` (по умолчанию) — только когда нужна изоляция по сессии.
- `EnterWriteLock()` внутри уже взятого `EnterReadLock()` без upgradeable — дедлок. Правильный путь: `EnterUpgradeableReadLock`, затем при необходимости `EnterWriteLock`. В upgradeable-режиме только один поток может одновременно находиться, поэтому повышения сериализуются безопасно.
- Не держите `await` под захваченной `ReaderWriterLockSlim`. Компилятор не поймает `using var rw = ...; await ...` так же строго, как `lock`, но это тот же дедлок: пока асинхронная операция выполняется на другом потоке, блокировка держится, и другие читатели/писатели ждут.
- `LockRecursionPolicy.SupportsRecursion` маскирует ошибки дизайна (случайный повторный вход) и ухудшает производительность — урок явно предостерегает. По умолчанию выбирайте `NoRecursion`.

#### Критерии приёмки
- [ ] Решение компилируется под `net8.0` без ошибок и без предупреждений анализаторов.
- [ ] `ThrottledHttpClient` использует `SemaphoreSlim` с `initialCount == maxCount` и `await WaitAsync(ct)`.
- [ ] `Release()` вызывается ровно один раз в `finally` на каждый успешный вход.
- [ ] В стресс-тесте счётчик активных запросов никогда не превышает `maxConcurrency`.
- [ ] `SingleInstanceGuard` создаёт именованный `Mutex` с префиксом `Global\`.
- [ ] `HasOwnership` корректно отражает `createdNew`; `Dispose` вызывает `ReleaseMutex` ровно один раз и только при владении.
- [ ] `WaitForOwnershipAsync` реализован через `Task.Run` + `WaitOne(timeout)` и принимает `CancellationToken`.
- [ ] `ThreadSafeCache` создан с `LockRecursionPolicy.NoRecursion`.
- [ ] `GetOrAdd` использует `EnterUpgradeableReadLock` и двойную проверку внутри `EnterWriteLock`.
- [ ] Все `Enter*` парны с `Exit*` в `try/finally`; нет ветвей с «забытым» `Exit`.
- [ ] Нет ни одного `await` под захваченной блокировкой `ReaderWriterLockSlim`.
- [ ] Нет ни одного `_gate.Wait()` в async-методе — только `WaitAsync`.
- [ ] `CancellationToken` передаётся во все ожидания; `OperationCanceledException` обрабатывается.
- [ ] `dotnet test` проходит зелёно; `dotnet build -c Release` без предупреждений.
- [ ] Код сопровождён краткими двуязычными комментариями в ключевых местах.

#### Подсказки (без прямого ответа)
- Для подсчёта активных запросов используйте `Interlocked.Increment`/`Decrement` вокруг `await` — это материал предыдущего урока, но здесь он служит инвариантом для отладки.
- Двойная проверка в `GetOrAdd` нужна, потому что между `TryGetValue` в upgradeable-режиме и `EnterWriteLock` другой писатель не может войти (upgradeable эксклюзивен среди upgradeable), но если у вас несколько кэшей или рекурсия — проверка всё равно спасает.
- Для тестов `HttpClient` не нужен реальный — наследник `HttpMessageHandler` с `Task.Delay(10, ct)` в `SendAsync` даст детерминированную нагрузку.
- `Mutex.WaitOne(timeout)` возвращает `bool`; `false` означает таймаут — не владейте мьютексом в этом случае и не вызывайте `ReleaseMutex`.

#### Эталонное решение (разбор)
```csharp
// C# 12 / .NET 8. ConcurrencyToolkit — три примитива синхронизации из урока M11-L04.
// Three synchronization primitives from lesson M11-L04, hardened against classic mistakes.
using System.Diagnostics;
using System.Net.Http;
using System.Threading;

namespace ConcurrencyToolkit;

// 1) SemaphoreSlim: ограничение параллелизма в async-коде, без блокировки потока.
//    SemaphoreSlim: bound async concurrency without blocking a pool thread.
public sealed class ThrottledHttpClient : IDisposable
{
    private readonly HttpClient _client;
    // initialCount == maxCount: Release не превысит потолок.
    // initialCount == maxCount: Release can never exceed the cap.
    private readonly SemaphoreSlim _gate;
    private readonly int _maxConcurrency;
    private int _activeCount; // инвариант для Debug.Assert. / invariant for Debug.Assert.

    public ThrottledHttpClient(HttpClient client, int maxConcurrency)
    {
        ArgumentOutOfRangeException.ThrowIfLessThan(maxConcurrency, 1);
        _client = client;
        _maxConcurrency = maxConcurrency;
        _gate = new SemaphoreSlim(initialCount: maxConcurrency, maxConcurrency);
    }

    public int ActiveCount => Volatile.Read(ref _activeCount);

    public async Task<HttpResponseMessage> GetAsync(Uri url, CancellationToken ct)
    {
        // ВАЖНО: WaitAsync(ct), НЕ Wait(). В async-коде Wait() → thread pool starvation.
        // IMPORTANT: WaitAsync(ct), NOT Wait(). Wait() in async → thread pool starvation.
        await _gate.WaitAsync(ct).ConfigureAwait(false);
        // Если WaitAsync выбросил OCE — мы не владеем разрешением, Release не нужен.
        // If WaitAsync threw OCE — we own no permit, no Release needed.
        var entered = true;
        try
        {
            Interlocked.Increment(ref _activeCount);
            Debug.Assert(_activeCount <= _maxConcurrency, "Превышен лимит параллелизма / concurrency cap exceeded");
            return await _client.GetAsync(url, ct).ConfigureAwait(false);
        }
        finally
        {
            Interlocked.Decrement(ref _activeCount);
            if (entered)
                _gate.Release(); // строго один раз. / exactly once.
        }
    }

    public void Dispose() => _gate.Dispose();
}

// 2) Mutex: single-instance между процессами. / Mutex: cross-process single instance.
public sealed class SingleInstanceGuard : IDisposable
{
    private const string DefaultName = @"Global\PolyglotFetcher.SingleInstance.v1";
    private readonly Mutex _mutex;
    public bool HasOwnership { get; }

    public SingleInstanceGuard(string? mutexName = null)
    {
        var name = string.IsNullOrEmpty(mutexName) ? DefaultName : mutexName;
        // Global\ — видим во всех терминальных сессиях ОС.
        // Global\ — visible across all terminal-server sessions.
        _mutex = new Mutex(initiallyOwned: true, name, out bool createdNew);
        HasOwnership = createdNew;
    }

    // WaitOne напрямую не принимает CancellationToken — оборачиваем в Task.Run.
    // WaitOne does not take a CancellationToken directly — wrap with Task.Run.
    public async Task<bool> WaitForOwnershipAsync(TimeSpan timeout, CancellationToken ct)
    {
        if (HasOwnership) return true;
        // Task.Run с ct: если ct сработает раньше, задача отменится.
        // Task.Run with ct: if ct fires first, the task is cancelled.
        bool owned = await Task.Run(
            () => _mutex.WaitOne(timeout),
            ct).ConfigureAwait(false);
        if (owned) // осторожно: владение надо потом освободить. / careful: must release later.
            HasOwnershipInternal = true;
        return owned;
    }

    private bool HasOwnershipInternal;
    private SingleInstanceGuard() => HasOwnershipInternal = false;

    public void Dispose()
    {
        // Реентерабельный: освобождаем столько раз, сколько владели.
        // Reentrant: release as many times as we owned it.
        if (HasOwnership || HasOwnershipInternal)
        {
            try { _mutex.ReleaseMutex(); }
            catch (ApplicationException) { /* уже освобождён / already released */ }
        }
        _mutex.Dispose();
    }
}

// 3) ReaderWriterLockSlim: много читателей, писатель эксклюзивен, upgradeable — для GetOrAdd.
//    ReaderWriterLockSlim: many readers, writer exclusive, upgradeable for GetOrAdd.
public sealed class ThreadSafeCache<TKey, TValue> : IDisposable where TKey : notnull
{
    private readonly Dictionary<TKey, TValue> _data = new();
    // NoRecursion: ловим ошибки дизайна вместо их маскировки.
    // NoRecursion: surface design bugs instead of hiding them.
    private readonly ReaderWriterLockSlim _rw = new(LockRecursionPolicy.NoRecursion);

    public TValue Read(TKey key)
    {
        _rw.EnterReadLock();
        try { return _data[key]; } // KeyNotFoundException допустим. / KeyNotFoundException is fine.
        finally { _rw.ExitReadLock(); }
    }

    public void Write(TKey key, TValue value)
    {
        _rw.EnterWriteLock();
        try { _data[key] = value; }
        finally { _rw.ExitWriteLock(); }
    }

    // Upgradeable: читаем, при необходимости атомарно повышаемся до писателя.
    // Upgradeable: read, then atomically promote to writer if needed.
    public TValue GetOrAdd(TKey key, Func<TKey, TValue> factory)
    {
        _rw.EnterUpgradeableReadLock();
        try
        {
            if (_data.TryGetValue(key, out var existing))
                return existing;

            _rw.EnterWriteLock();
            try
            {
                // Двойная проверка: в upgradeable-режиме другой писатель не войдёт,
                // но проверка страховает от логических ошибок и будущих правок.
                // Double-check: another writer cannot enter in upgradeable mode,
                // but this guards against logic errors and future edits.
                if (_data.TryGetValue(key, out var nowExisting))
                    return nowExisting;

                var produced = factory(key);
                _data[key] = produced;
                return produced;
            }
            finally { _rw.ExitWriteLock(); }
        }
        finally { _rw.ExitUpgradeableReadLock(); }
    }

    public void Dispose() => _rw.Dispose();
}
```

Разбор по строкам. В `ThrottledHttpClient` конструктор валидирует `maxConcurrency` через `ArgumentOutOfRangeException.ThrowIfLessThan` (хелпер .NET 8) и создаёт семафор с `initialCount == maxCount` — урок подчёркивает, что это не позволяет `Release` превысить потолок и спровоцировать `SemaphoreFullException` при следующем цикле. `GetAsync` вызывает именно `await _gate.WaitAsync(ct)`, а не `Wait()`: это ключевое правило урока для async-кода, иначе пул потоков истощается. Флаг `entered` установлен после успешного `WaitAsync`; если `WaitAsync` выбросит `OperationCanceledException` (или `SemaphoreFullException`), мы не попадём в `try` и `Release` не вызовется — это правильно, потому что разрешения мы не получили. `Release` стоит в `finally` и вызывается ровно один раз, что соответствует best practice урока. `Debug.Assert` с инвариантом `_activeCount <= _maxConcurrency` — это «сторожевой» инвариант: если он нарушается, значит где-то `Release` вызвался лишний раз или `WaitAsync` не блокировал как следует (что в корректной реализации невозможно). В `SingleInstanceGuard` используется `Global\`-префикс, как требует урок для системной видимости. Конструктор с `initiallyOwned: true` и `createdNew` позволяет первому процессу сразу владеть мьютексом, а второму — узнать, что он не первый. `WaitForOwnershipAsync` обёрнут в `Task.Run`, потому что `Mutex.WaitOne` напрямую не принимает `CancellationToken` — это явная оговорка урока. В `Dispose` мы освобождаем мьютекс только если владеем им (`HasOwnership`), и оборачиваем `ReleaseMutex` в `try/catch (ApplicationException)`, поскольку двойное освобождение бросает исключение (реентерабельность требует строго 1:1). В `ThreadSafeCache` выбран `LockRecursionPolicy.NoRecursion` — урок прямо предостерегает от `SupportsRecursion`, так как он маскирует ошибки и ухудшает производительность. `GetOrAdd` идёт через `EnterUpgradeableReadLock`, а не через «отпустил read → взял write» — это устраняет race condition «прочитал-отпустил-перечитал», описанный в уроке. Внутри `EnterWriteLock` стоит двойная проверка `TryGetValue`: хотя в upgradeable-режиме другой писатель не может войти одновременно (upgradeable эксклюзивен среди upgradeable), проверка страховает от будущих правок и делает инвариант очевидным. Каждый `Enter*` парный с `Exit*` в `finally`, что закрывает частую ошибку «забыл `Exit` в одной из ветвей → блокировка залипает навсегда». Никаких `await` под захваченной блокировкой нет — это правило урока, потому что `await` под `ReaderWriterLockSlim`等效ен дедлоку. Все три класса реализуют `IDisposable` и освобождают свои примитивы, что соответствует best practice «оборачивай в `using`/`Dispose`».

#### Задания на углубление (бонус)
1. Добавьте в `ThrottledHttpClient` очередь ожидания с приоритетом: запросы с `isPriority = true` должны проходить раньше. Подсказка: один `SemaphoreSlim` здесь не справится — подумайте про `Channel<T>` с приоритетом или два семафора.
2. Реализуйте `GetOrAddAsync` в `ThreadSafeCache` с асинхронной фабрикой `Func<TKey, CancellationToken, Task<TValue>>`. Главный вызов — не держать `await` под захваченной `ReaderWriterLockSlim`. Решение обычно требует двухфазного подхода: upgradeable read для проверки, отпускание перед `await` фабрики, затем повторная проверка и запись.
3. Измерьте через `BenchmarkDotNet` разницу между `ReaderWriterLockSlim` с `NoRecursion` и `SupportsRecursion` на профиле 99 % чтений / 1 % записи. Сравните с простым `lock` и новым `Lock` (.NET 9, если доступен).
4. Реализуйте `DistributedThrottler` на `Semaphore` (не Slim) — он использует объект ядра ОС и потому виден между процессами. Сравните память и latency с `SemaphoreSlim` и объясните, почему `Slim`-вариант предпочтительнее внутри одного процесса.

---

## Statement in English / Постановка на английском

#### Context & motivation
You have joined a team building an internal service called “Polyglot Fetcher”: it concurrently pulls metadata from many external HTTP sources, caches the results in memory, and must guarantee that only one instance of the service runs on a given machine — otherwise two processes would duplicate requests and overload the external APIs. The team has already hit the classic traps: somebody wrote `semaphore.Wait()` inside an async method, and under load the thread pool starved and the service froze. Somebody forgot `try/finally` around `Release()`, so after the first exception the semaphore permanently lost one permit and throughput quietly dropped. In the cache they tried to implement `GetOrAdd` by entering read lock, releasing it, and immediately entering write lock — and caught a read-release-reread race where two threads computed the factory twice. Finally, for single-instance enforcement somebody took a `Mutex` without the `Global\` prefix, and under another user’s terminal session the second instance was not blocked at all.

Your task is to rewrite the three key components so that they are idempotent, thread-safe, async-friendly, and resilient to cancellation via `CancellationToken`. You will use exactly the primitives covered in the lesson: `SemaphoreSlim` for throttling, `Mutex` for single-instance, and `ReaderWriterLockSlim` for a cache with readers and writers. No `lock` around `await`, no `Wait()` in async methods, no release-then-reread. The code must compile under .NET 8 on C# 12 and pass a stress test with no deadlocks, no `SemaphoreFullException`, and no lost permits. The exercise is intentionally realistic: the bugs you are fixing are the exact ones the lesson warns about, so the fix should map one-to-one to the best practices and common-mistakes sections.

#### What to do step by step
1. Create a solution and three projects: `dotnet new sln -n ConcurrencyToolkit`, then `dotnet new classlib -n ConcurrencyToolkit -f net8.0`, `dotnet new xunit -n ConcurrencyToolkit.Tests -f net8.0`, add the test project to the solution with `dotnet sln add`, and add a reference with `dotnet add ConcurrencyToolkit.Tests reference ConcurrencyToolkit`. Use `Microsoft.NET.Test.Sdk` and `xunit` for the tests.
2. In the `ConcurrencyToolkit` project create a class `ThrottledHttpClient` that takes `int maxConcurrency` and an `HttpClient` (constructor injection). Internally it stores `SemaphoreSlim _gate = new(maxConcurrency, maxConcurrency)`. The method `Task<HttpResponseMessage> GetAsync(Uri url, CancellationToken ct)` must do `await _gate.WaitAsync(ct).ConfigureAwait(false)`, in `try` call `_client.GetAsync(url, ct)`, and in `finally` call `_gate.Release()` exactly once. Add an active-request counter using `Interlocked.Increment`/`Decrement`, and assert via `Debug.Assert` that it never exceeds `maxConcurrency`.
3. Implement `SingleInstanceGuard : IDisposable`. The constructor takes a string `mutexName` (e.g. `@"Global\PolyglotFetcher.SingleInstance.v1"`), creates `new Mutex(initiallyOwned: true, mutexName, out var createdNew)`, and stores `HasOwnership = createdNew`. In `Dispose()`, if `HasOwnership`, call `ReleaseMutex()` inside `try/catch (ApplicationException)`, then `_mutex.Dispose()`. Add a method `WaitForOwnershipAsync(TimeSpan timeout, CancellationToken ct)` that uses `Task.Run(() => mutex.WaitOne(timeout), ct)` for a cancellable wait — remember that `WaitOne` does not directly accept a `CancellationToken`.
4. Implement a generic `ThreadSafeCache<TKey, TValue> where TKey : notnull` on `ReaderWriterLockSlim` with `LockRecursionPolicy.NoRecursion` (recursion is intentionally off, to surface design bugs). Methods: `TValue Read(TKey)`, `void Write(TKey, TValue)`, `TValue GetOrAdd(TKey, Func<TKey, TValue> factory)`. `GetOrAdd` must go through `EnterUpgradeableReadLock`, then `TryGetValue`; if absent, `EnterWriteLock` with a double check (`TryGetValue` again), call the factory, and store. Each `Enter*` must be paired with `Exit*` in `try/finally`.
5. Write a stress test `StressTests`: start 50 tasks, each performing 200 operations on the cache (80 % `GetOrAdd`, 20 % `Write`), while concurrently hitting `ThrottledHttpClient` with `maxConcurrency = 4` across 20 fake URLs (via a stub `HttpMessageHandler` with a 10 ms delay). Assert that `activeCount` never exceeds 4, that the cache did not throw, and that `Dispose` did not fail.
6. Run `dotnet test` and confirm everything is green. Run `dotnet build -c Release` and confirm there are no `CA` warnings (if analyzers are enabled). Run `dotnet run --project ConcurrencyToolkit.Tests` to exercise the stress test in isolation.

#### Requirements
- Target framework `net8.0`, language C# 12: top-level statements in tests are fine, collection expressions (`new()` target-typed, `[]` for arrays), pattern matching, `required`, `init`, raw string literals where helpful. `lock (obj) { await ... }` is forbidden.
- Every wait on a synchronization primitive takes a `CancellationToken`: `WaitAsync(ct)` for the semaphore; for `Mutex` use a `Task.Run` + `WaitOne(timeout)` wrapper because `WaitOne` does not take `ct` directly. `OperationCanceledException` must be handled cleanly.
- `Release()` on the semaphore is called exactly once per successful `WaitAsync` and only in `finally`. `ExitReadLock`/`ExitWriteLock`/`ExitUpgradeableReadLock` are strictly paired with `Enter*` inside `try/finally`. No release-read-then-enter-write in `GetOrAdd` — only the upgradeable path.
- The `Mutex` for single-instance must use the `Global\` prefix when cross-session visibility is required; release exactly as many times as you acquired; check `createdNew`. Reentrance is supported but should not be abused.
- `ReaderWriterLockSlim` is created with `LockRecursionPolicy.NoRecursion` (there is no good reason for `SupportsRecursion` in this homework). Critical sections are as short as possible, with no `await` under a held lock.
- The code must tolerate cancellation: on `OperationCanceledException` the semaphore permit is still returned via `finally`; for `Mutex`, if `WaitOne` was cancelled we do not own the mutex, so `ReleaseMutex` is not called.

#### Pitfalls
- The most common mistake is `_gate.Wait()` inside an async method. The lesson is explicit: the synchronous `Wait()` blocks a pool thread, and under load you get thread pool starvation — the service freezes. Always `await _gate.WaitAsync(ct)`.
- `Release()` outside `try/finally` — on exception the permit is “lost” and the semaphore is permanently throttled. The counter quietly drops to 3-of-4 and you spend hours hunting the degradation. Put `Release` in `finally`.
- Calling `semaphore.Release()` twice (or more than the number of acquires) raises `SemaphoreFullException`. Keep acquire/release strictly 1:1; do not call `Release` in a `catch` branch if `WaitAsync` did not yet succeed.
- A `Mutex` without `Global\` under terminal sessions — different users get different mutexes, and single-instance silently breaks. Use `Global\` for system-wide sync; `Local\` (the default) only when session isolation is desired.
- `EnterWriteLock()` inside an already-held `EnterReadLock()` without upgradeable deadlocks. The correct path is `EnterUpgradeableReadLock`, then `EnterWriteLock` when needed. Upgradeable mode allows only one such thread at a time, so promotions serialize safely.
- Do not `await` under a held `ReaderWriterLockSlim`. The compiler is not as strict about `using var rw = ...; await ...` as it is about `lock`, but it is the same deadlock: while the async operation runs elsewhere, the lock is held and other readers/writers wait.
- `LockRecursionPolicy.SupportsRecursion` hides design bugs (accidental re-entry) and hurts performance — the lesson warns against it. Prefer `NoRecursion` by default.

#### Acceptance criteria
- [ ] The solution compiles under `net8.0` with no errors and no analyzer warnings.
- [ ] `ThrottledHttpClient` uses `SemaphoreSlim` with `initialCount == maxCount` and `await WaitAsync(ct)`.
- [ ] `Release()` is called exactly once in `finally` per successful acquire.
- [ ] In the stress test the active-request counter never exceeds `maxConcurrency`.
- [ ] `SingleInstanceGuard` creates a named `Mutex` with the `Global\` prefix.
- [ ] `HasOwnership` correctly reflects `createdNew`; `Dispose` calls `ReleaseMutex` exactly once and only when owning.
- [ ] `WaitForOwnershipAsync` is implemented via `Task.Run` + `WaitOne(timeout)` and accepts a `CancellationToken`.
- [ ] `ThreadSafeCache` is created with `LockRecursionPolicy.NoRecursion`.
- [ ] `GetOrAdd` uses `EnterUpgradeableReadLock` and a double-check inside `EnterWriteLock`.
- [ ] Every `Enter*` is paired with `Exit*` in `try/finally`; no branch forgets an `Exit`.
- [ ] There is no `await` while holding a `ReaderWriterLockSlim` lock.
- [ ] There is no `_gate.Wait()` in any async method — only `WaitAsync`.
- [ ] A `CancellationToken` is passed into every wait; `OperationCanceledException` is handled.
- [ ] `dotnet test` is green; `dotnet build -c Release` has no warnings.
- [ ] The code has short bilingual comments at the key spots.

#### Hints (no direct answer)
- For the active-request counter use `Interlocked.Increment`/`Decrement` around the `await` — that is the previous lesson’s material, but here it serves as a debugging invariant.
- The double-check in `GetOrAdd` is needed because, while in upgradeable mode another writer cannot enter (upgradeable is exclusive among upgradeable), the check still guards against logic errors and future edits that might break the invariant.
- For tests you do not need a real `HttpClient` — a subclass of `HttpMessageHandler` that does `Task.Delay(10, ct)` inside `SendAsync` gives deterministic load.
- `Mutex.WaitOne(timeout)` returns `bool`; `false` means timeout — in that case you do not own the mutex and must not call `ReleaseMutex`.

#### Reference solution walk-through
```csharp
// C# 12 / .NET 8. ConcurrencyToolkit — three synchronization primitives from lesson M11-L04.
using System.Diagnostics;
using System.Net.Http;
using System.Threading;

namespace ConcurrencyToolkit;

// 1) SemaphoreSlim: bound async concurrency without blocking a pool thread.
public sealed class ThrottledHttpClient : IDisposable
{
    private readonly HttpClient _client;
    // initialCount == maxCount: Release can never exceed the cap.
    private readonly SemaphoreSlim _gate;
    private readonly int _maxConcurrency;
    private int _activeCount; // invariant for Debug.Assert.

    public ThrottledHttpClient(HttpClient client, int maxConcurrency)
    {
        ArgumentOutOfRangeException.ThrowIfLessThan(maxConcurrency, 1);
        _client = client;
        _maxConcurrency = maxConcurrency;
        _gate = new SemaphoreSlim(initialCount: maxConcurrency, maxConcurrency);
    }

    public int ActiveCount => Volatile.Read(ref _activeCount);

    public async Task<HttpResponseMessage> GetAsync(Uri url, CancellationToken ct)
    {
        // IMPORTANT: WaitAsync(ct), NOT Wait(). Wait() in async → thread pool starvation.
        await _gate.WaitAsync(ct).ConfigureAwait(false);
        // If WaitAsync threw OCE — we own no permit, no Release needed.
        var entered = true;
        try
        {
            Interlocked.Increment(ref _activeCount);
            Debug.Assert(_activeCount <= _maxConcurrency, "concurrency cap exceeded");
            return await _client.GetAsync(url, ct).ConfigureAwait(false);
        }
        finally
        {
            Interlocked.Decrement(ref _activeCount);
            if (entered)
                _gate.Release(); // exactly once.
        }
    }

    public void Dispose() => _gate.Dispose();
}

// 2) Mutex: cross-process single instance.
public sealed class SingleInstanceGuard : IDisposable
{
    private const string DefaultName = @"Global\PolyglotFetcher.SingleInstance.v1";
    private readonly Mutex _mutex;
    public bool HasOwnership { get; }

    public SingleInstanceGuard(string? mutexName = null)
    {
        var name = string.IsNullOrEmpty(mutexName) ? DefaultName : mutexName;
        // Global\ — visible across all terminal-server sessions.
        _mutex = new Mutex(initiallyOwned: true, name, out bool createdNew);
        HasOwnership = createdNew;
    }

    // WaitOne does not take a CancellationToken directly — wrap with Task.Run.
    public async Task<bool> WaitForOwnershipAsync(TimeSpan timeout, CancellationToken ct)
    {
        if (HasOwnership) return true;
        bool owned = await Task.Run(() => _mutex.WaitOne(timeout), ct).ConfigureAwait(false);
        if (owned)
            HasOwnershipInternal = true;
        return owned;
    }

    private bool HasOwnershipInternal;
    private SingleInstanceGuard() => HasOwnershipInternal = false;

    public void Dispose()
    {
        // Reentrant: release as many times as we owned it.
        if (HasOwnership || HasOwnershipInternal)
        {
            try { _mutex.ReleaseMutex(); }
            catch (ApplicationException) { /* already released */ }
        }
        _mutex.Dispose();
    }
}

// 3) ReaderWriterLockSlim: many readers, writer exclusive, upgradeable for GetOrAdd.
public sealed class ThreadSafeCache<TKey, TValue> : IDisposable where TKey : notnull
{
    private readonly Dictionary<TKey, TValue> _data = new();
    // NoRecursion: surface design bugs instead of hiding them.
    private readonly ReaderWriterLockSlim _rw = new(LockRecursionPolicy.NoRecursion);

    public TValue Read(TKey key)
    {
        _rw.EnterReadLock();
        try { return _data[key]; } // KeyNotFoundException is fine.
        finally { _rw.ExitReadLock(); }
    }

    public void Write(TKey key, TValue value)
    {
        _rw.EnterWriteLock();
        try { _data[key] = value; }
        finally { _rw.ExitWriteLock(); }
    }

    // Upgradeable: read, then atomically promote to writer if needed.
    public TValue GetOrAdd(TKey key, Func<TKey, TValue> factory)
    {
        _rw.EnterUpgradeableReadLock();
        try
        {
            if (_data.TryGetValue(key, out var existing))
                return existing;

            _rw.EnterWriteLock();
            try
            {
                // Double-check: another writer cannot enter in upgradeable mode,
                // but this guards against logic errors and future edits.
                if (_data.TryGetValue(key, out var nowExisting))
                    return nowExisting;

                var produced = factory(key);
                _data[key] = produced;
                return produced;
            }
            finally { _rw.ExitWriteLock(); }
        }
        finally { _rw.ExitUpgradeableReadLock(); }
    }

    public void Dispose() => _rw.Dispose();
}
```

Line-by-line walk-through. In `ThrottledHttpClient` the constructor validates `maxConcurrency` via `ArgumentOutOfRangeException.ThrowIfLessThan` (a .NET 8 helper) and creates the semaphore with `initialCount == maxCount` — the lesson stresses that this prevents `Release` from ever pushing the counter above the ceiling and triggering `SemaphoreFullException` on the next cycle. `GetAsync` calls `await _gate.WaitAsync(ct)`, never `Wait()`: this is the lesson’s central rule for async code, otherwise the thread pool starves. The `entered` flag is set after a successful `WaitAsync`; if `WaitAsync` throws `OperationCanceledException` (or anything else) we never reach `try`, so `Release` is not called — which is correct, because we never acquired a permit. `Release` sits in `finally` and runs exactly once, matching the lesson’s best practice. The `Debug.Assert` on `_activeCount <= _maxConcurrency` is a watchdog invariant: if it ever fires, somebody called `Release` too many times or `WaitAsync` failed to gate (which should be impossible in a correct implementation). In `SingleInstanceGuard` the `Global\` prefix is used, as the lesson requires for system-wide visibility. The constructor with `initiallyOwned: true` and `createdNew` lets the first process own the mutex immediately and lets the second one learn it is not first. `WaitForOwnershipAsync` is wrapped in `Task.Run` because `Mutex.WaitOne` does not accept a `CancellationToken` directly — this is an explicit caveat from the lesson. In `Dispose` we release only if we actually own the mutex (`HasOwnership`), and we wrap `ReleaseMutex` in `try/catch (ApplicationException)` because double-release throws (reentrance demands strict 1:1). In `ThreadSafeCache` we pick `LockRecursionPolicy.NoRecursion` — the lesson explicitly warns that `SupportsRecursion` masks bugs and hurts performance. `GetOrAdd` goes through `EnterUpgradeableReadLock` rather than release-read-then-enter-write, eliminating the read-release-reread race described in the lesson. Inside `EnterWriteLock` there is a second `TryGetValue` check: although in upgradeable mode another writer cannot enter concurrently (upgradeable is exclusive among upgradeable holders), the check guards against future edits and makes the invariant obvious. Every `Enter*` is paired with an `Exit*` inside `finally`, closing the classic “forgot `Exit` on some branch → lock sticks forever” bug. There is no `await` under a held lock — the lesson’s rule, because awaiting under `ReaderWriterLockSlim` is equivalent to a deadlock. All three classes implement `IDisposable` and dispose their primitives, matching the “wrap in `using`/`Dispose`” best practice.

#### Going deeper (bonus)
1. Add a priority queue to `ThrottledHttpClient`: requests with `isPriority = true` should be admitted ahead of ordinary ones. Hint: a single `SemaphoreSlim` is not enough here — consider a prioritized `Channel<T>` or two semaphores.
2. Implement `GetOrAddAsync` in `ThreadSafeCache` with an async factory `Func<TKey, CancellationToken, Task<TValue>>`. The core challenge is not holding an `await` under a held `ReaderWriterLockSlim`. The usual solution is two-phase: upgradeable read for the check, release before awaiting the factory, then re-check and write.
3. Use `BenchmarkDotNet` to compare `ReaderWriterLockSlim` with `NoRecursion` vs `SupportsRecursion` on a 99 % read / 1 % write profile. Also compare with a plain `lock` and the new `Lock` type (.NET 9, if available).
4. Implement a `DistributedThrottler` on `Semaphore` (not Slim) — it uses an OS kernel object and is therefore visible across processes. Compare its memory and latency with `SemaphoreSlim` and explain why the `Slim` variant is preferable inside a single process.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Создано решение `ConcurrencyToolkit` с проектами `classlib` и `xunit` под `net8.0`.
- [ ] `ThrottledHttpClient` использует `SemaphoreSlim` и `await WaitAsync(ct)`, `Release` в `finally`.
- [ ] `SingleInstanceGuard` использует именованный `Mutex` с `Global\`, проверяет `createdNew`, реализует `WaitForOwnershipAsync` через `Task.Run`.
- [ ] `ThreadSafeCache` использует `ReaderWriterLockSlim` с `NoRecursion`, `GetOrAdd` через upgradeable, двойная проверка.
- [ ] Стресс-тест проходит зелёно; `Debug.Assert` не срабатывает.
- [ ] Нет `Wait()` в async, нет `await` под блокировкой, `CancellationToken` везде.
- [ ] Solution `ConcurrencyToolkit` created with `classlib` and `xunit` projects under `net8.0`.
- [ ] `ThrottledHttpClient` uses `SemaphoreSlim` and `await WaitAsync(ct)`, `Release` in `finally`.
- [ ] `SingleInstanceGuard` uses a named `Mutex` with `Global\`, checks `createdNew`, implements `WaitForOwnershipAsync` via `Task.Run`.
- [ ] `ThreadSafeCache` uses `ReaderWriterLockSlim` with `NoRecursion`, `GetOrAdd` via upgradeable with a double check.
- [ ] The stress test is green; `Debug.Assert` does not fire.
- [ ] No `Wait()` in async, no `await` under a lock, `CancellationToken` everywhere.

#### Ресурсы / Resources
- [Microsoft Learn — SemaphoreSlim](https://learn.microsoft.com/dotnet/api/system.threading.semaphoreslim)
- [Microsoft Learn — Mutex](https://learn.microsoft.com/dotnet/api/system.threading.mutex)
- [Microsoft Learn — ReaderWriterLockSlim](https://learn.microsoft.com/dotnet/api/system.threading.readerwriterlockslim)
- [Stephen Toub — Async FAQ](https://devblogs.microsoft.com/dotnet/async-faq/)
- [Microsoft Learn — CancellationToken](https://learn.microsoft.com/dotnet/api/system.threading.cancellationtoken)
- [Microsoft Learn — LockRecursionPolicy](https://learn.microsoft.com/dotnet/api/system.threading.lockrecursionpolicy)

---
[← К уроку M11-L04](lesson-M11-L04-semaphore-mutex-rwlock.md) | [⬆ К модулю M11](../README.md) | [Предыдущее ДЗ ←](homework-M11-L03-interlocked.md) | [Следующее ДЗ →](homework-M11-L05-volatile-memory-model.md)
