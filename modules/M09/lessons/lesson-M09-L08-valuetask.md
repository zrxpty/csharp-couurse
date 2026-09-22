[⬆ К модулю M09](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M09-L08: ValueTask, кэшированные результаты / ValueTask, cached results

**Модуль / Module:** M09
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

`Task` и `Task<T>` — ссылочные типы (class). Каждый вызов асинхронной операции, которая идёт в пул потоков или в I/O, возвращает новый объект в куче. Это означает аллокацию, работу GC, давление на память. Для «горячих» путей — высоконагруженных API, кэшей, часто вызываемых методов — эти аллокации складываются в реальные накладные расходы.

`ValueTask` и `ValueTask<T>` — значимые типы (struct). Они спроектированы так, чтобы **избегать аллокации** в наиболее частом случае: когда результат уже доступен синхронно (например, значение взято из кэша, проверка прошла мгновенно, буфер уже заполнен). Внутри `ValueTask<T>` — это discriminated union: он может содержать либо готовый результат `T`, либо `Task<T>`, либо `IValueTaskSource<T>`. Размер структуры больше, чем ссылки на `Task`, но зато нет аллокации, когда результат есть сразу.

**Ключевое правило: когда применять ValueTask.** Используйте `ValueTask<T>`, если метод **часто** завершается синхронно и вызывается в горячем пути. Не используйте ValueTask «потому что модно»: он сложнее, больше по размеру и прощает меньше ошибок. Если операция почти всегда асинхронна (реальный I/O, сеть, диск), `Task<T>` часто лучше — он проще и предсказуемее.

**Кэшированные результаты — классический сценарий.** Представьте метод `GetAsync(key)`, который сначала смотрит в кэш, и только при промахе идёт в базу. С `Task<T>` каждый возврат кэшированного значения всё равно создаёт новый `Task` (или требует заранее изготовленного `Task.FromResult`, что ещё и плодит объекты). С `ValueTask<T>` возврат кэшированного значения — это просто запись поля в struct, без аллокации вообще.

Можно кэшировать и сам `ValueTask`/`ValueTask<T>` — но **только если он завершён** (`IsCompleted == true`). Кэшировать незавершённый ValueTask опасно: ValueTask не потокобезопасен для многократного ожидания разными потребителями (об этом ниже).

**Главное ограничение: один await на экземпляр.** Стандартный `Task` можно ожидать сколько угодно раз и любым числом потребителей — он это переживает. `ValueTask` — нет. По контракту:
- Вы можете ожидать `ValueTask` **один раз**. Повторное ожидание того же экземпляра — не определено.
- Вы **не должны** обращаться к `ValueTask` из нескольких потоков одновременно.
- Если внутренним источником служит `IValueTaskSource`, повторное ожидание может выбросить `InvalidOperationException`.

Это следствие экономии: чтобы не аллоцировать, ValueTask переиспользует внутренний буфер, и второй потребитель увидит мусор.

**IValueTaskSource — для ещё большей экономии.** Когда операция асинхронна, но объектов всё равно хочется создавать мало (например, пул соединений, сокеты, `SocketAsyncEventArgs`), тип реализует `IValueTaskSource<T>`. Тогда `ValueTask<T>` хранит ссылку на этот пулляемый источник, а не на `Task`. Источник возвращается в пул после завершения await. Так работают, например, `MemoryStream.ReadAsync`, `NetworkStream.ReadAsync`, новые API `PipeReader`. Ручная реализация `IValueTaskSource` — продвинутая тема: нужно правильно управлять состоянием через `ValueTaskSourceStatus` и `GetResult`/`GetStatus`/`OnCompleted`.

**Concurrency-акценты.** Кэш, из которого читают ValueTask, должен быть потокобезопасным: `ConcurrentDictionary<,>`, `Volatile`-чтение, или `lock` (но **никогда** `lock` вокруг `await` — это верный путь к дедлоку и разрушению инвариантов). Используйте `ConfigureAwait(false)` в библиотечном коде, чтобы не захватывать контекст синхронизации вызывающего — иначе UI-поток или ASP.NET-старый контекст могут застрять. Предаврейт кэша («cache stampede») предотвращают через `SemaphoreSlim` или `Lazy<Task<T>>`/`AsyncLazy`, чтобы дублирующие запросы не падали в базу одновременно. И главное предупреждение: **не блокируйте** ValueTask через `.Result` или `.Wait()` — особенно в контексте синхронизации (UI, старый ASP.NET): это классический самодедлок.

#### Theory (EN)

`Task` and `Task<T>` are reference types (classes). Every call to an asynchronous operation that goes to the thread pool or to I/O returns a fresh object on the heap. That means allocation, GC pressure, and real memory cost. On hot paths — high-throughput APIs, caches, frequently invoked methods — these allocations add up into measurable overhead.

`ValueTask` and `ValueTask<T>` are value types (structs). They are designed to **avoid allocation** in the most common case: when the result is already available synchronously (for example a value pulled from a cache, a fast check, an already-filled buffer). Internally `ValueTask<T>` is a discriminated union: it can hold either a ready `T` result, a `Task<T>`, or an `IValueTaskSource<T>`. The struct is larger than a single `Task` reference, but there is no allocation at all when the result is immediately available.

**The key rule: when to use ValueTask.** Use `ValueTask<T>` when a method **frequently** completes synchronously and sits on a hot path. Do not switch to ValueTask because it is fashionable: it is more complex, larger, and less forgiving. If the operation is almost always truly asynchronous (real I/O, network, disk), `Task<T>` is often better — simpler and more predictable.

**Cached results are the classic scenario.** Imagine a `GetAsync(key)` method that checks a cache first and only hits the database on a miss. With `Task<T>`, returning a cached value still allocates a new `Task` (or requires a pre-built `Task.FromResult`, which itself creates objects). With `ValueTask<T>`, returning a cached value is just a field store in the struct — no allocation at all.

You can even cache the `ValueTask`/`ValueTask<T>` itself — but **only when it is completed** (`IsCompleted == true`). Caching an incomplete ValueTask is dangerous: ValueTask is not safe for multiple awaiters or repeated consumption (see below).

**The main constraint: one await per instance.** A regular `Task` can be awaited any number of times by any number of consumers — it survives that. `ValueTask` does not. By contract:
- You may await a `ValueTask` **once**. Awaiting the same instance again is undefined behavior.
- You must **not** access a `ValueTask` from multiple threads concurrently.
- If the backing source is an `IValueTaskSource`, a second await may throw `InvalidOperationException`.

This is the cost of savings: to avoid allocating, ValueTask reuses an internal buffer, and a second consumer would see garbage.

**IValueTaskSource — for even less allocation.** When the operation is asynchronous but you still want to create very few objects (connection pools, sockets, `SocketAsyncEventArgs`), a type implements `IValueTaskSource<T>`. Then `ValueTask<T>` holds a reference to this poolable source instead of a `Task`. The source is returned to the pool after the await completes. This is how `MemoryStream.ReadAsync`, `NetworkStream.ReadAsync`, and the newer `PipeReader` APIs are built. A manual implementation of `IValueTaskSource` is an advanced topic: you must manage state correctly through `ValueTaskSourceStatus` and the `GetResult`/`GetStatus`/`OnCompleted` trio.

**Concurrency notes.** A cache that hands out ValueTasks must be thread-safe: `ConcurrentDictionary<,>`, `Volatile` reads, or a `lock` (but **never** a `lock` around an `await` — that is a sure path to a deadlock and broken invariants). Use `ConfigureAwait(false)` in library code so you do not capture the caller's synchronization context — otherwise a UI thread or a legacy ASP.NET context can stall. Prevent cache stampedes with `SemaphoreSlim` or `Lazy<Task<T>>`/`AsyncLazy` so duplicate requests do not all fall through to the database at once. And the cardinal warning: **do not block** a ValueTask with `.Result` or `.Wait()`, especially inside a synchronization context (UI, legacy ASP.NET): that is the classic self-deadlock.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8. Thread-safe async cache returning ValueTask<T>.
// Демонстрирует: кэш без аллокаций при попадании, защита от stampede через
// SemaphoreSlim, корректное использование ValueTask (один await, ConfigureAwait).
// Demonstrates: allocation-free cache hits, stampede protection via
// SemaphoreSlim, correct ValueTask usage (single await, ConfigureAwait).

using System.Collections.Concurrent;
using System.Runtime.CompilerServices;

public sealed class AsyncValueCache<TKey, TValue> where TKey : notnull
{
    // Кэш готовых значений. ConcurrentDictionary — потокобезопасный.
    // Cache of ready values; ConcurrentDictionary is thread-safe.
    private readonly ConcurrentDictionary<TKey, TValue> _cache = new();

    // Фабрика «дорогого» значения (например, запрос в БД).
    // Factory for the "expensive" value (e.g. a DB call).
    private readonly Func<TKey, CancellationToken, Task<TValue>> _loadAsync;

    // Семафор на ключ защищает от cache stampede: параллельные запросы
    // с одним ключом не падают в БД одновременно.
    // A per-key semaphore guards against cache stampede: concurrent
    // requests for the same key do not all hit the DB at once.
    private readonly ConcurrentDictionary<TKey, SemaphoreSlim> _locks = new();

    public AsyncValueCache(Func<TKey, CancellationToken, Task<TValue>> loadAsync)
    {
        _loadAsync = loadAsync;
    }

    // ВАЖНО: возвращаем ValueTask<T>. При попадании в кэш — НЕТ аллокации:
    // просто записываем готовое значение в struct. / IMPORTANT: returns
    // ValueTask<T>. On a cache hit there is NO allocation: we just store
    // the ready value in the struct.
    public async ValueTask<TValue> GetAsync(TKey key, CancellationToken ct = default)
    {
        // Быстрый путь: значение уже в кэше. Volatile-чтение не обязательно
        // для ConcurrentDictionary (он даёт нужные гарантии), но семантика та же.
        // Fast path: value already in cache.
        if (_cache.TryGetValue(key, out var cached))
        {
            return cached; // ValueTask<T> обёрнёт результат без аллокации Task.
                           // ValueTask<T> wraps the result with no Task allocation.
        }

        // Медленный путь: нужно загрузить. Берём семафор на ключ.
        // Slow path: must load. Acquire per-key semaphore.
        var gate = _locks.GetOrAdd(key, _ => new SemaphoreSlim(1, 1));

        await gate.WaitAsync(ct).ConfigureAwait(false); // НЕ блокируем поток!
                                                        // Do NOT block the thread!
        try
        {
            // Двойная проверка: пока мы ждали семафора, другой поток мог
            // уже положить значение в кэш. / Double-check: while we waited
            // for the semaphore, another thread may have populated the cache.
            if (_cache.TryGetValue(key, out cached))
            {
                return cached;
            }

            // Реальная асинхронная загрузка. ConfigureAwait(false) в библиотеке —
            // чтобы не захватывать контекст вызывающего (особенно UI / legacy ASP.NET).
            // Real async load. ConfigureAwait(false) in library code so we don't
            // capture the caller's context (UI / legacy ASP.NET especially).
            var value = await _loadAsync(key, ct).ConfigureAwait(false);

            // Записываем в кэш атомарно. GetOrAdd защищает от гонок на запись.
            // Atomically store. GetOrAdd protects against write races.
            _cache.GetOrAdd(key, value);
            return value;
        }
        finally
        {
            gate.Release();
        }
    }

    // ИЛЛЮСТРАЦИЯ ОШИБКИ: так делать НЕЛЬЗЯ. / ILLUSTRATION OF A MISTAKE:
    // never do this.
    //
    // public ValueTask<TValue> GetCachedOrBroken(TKey key)
    // {
    //     var vt = GetAsync(key);        // ValueTask, который будут.await'ить
    //     Task.Run(() => Consume(vt));   // второй потребитель — НЕОПРЕДЕЛЕНО!
    //     Task.Run(() => Consume(vt));   // second awaiter — UNDEFINED behavior!
    //     return vt;                     // и сам вызовющий сделает третий await.
    //                                     // ...and the caller will do a third await.
    // }
    //
    // Правильно: ValueTask ждут ровно ОДИН раз, ОДНИМ потребителем.
    // Correct: a ValueTask is awaited exactly ONCE, by ONE consumer.
}

// Пример безопасного «синхронного» источника ValueTask без Task вообще.
// Example of a synchronous ValueTask source with no Task at all.
public sealed class BoundedTemperatureReader
{
    private double _last;          // Volatile-защищённое последнее значение.
                                   // Volatile-protected last value.
    private readonly object _gate = new();

    public BoundedTemperatureReader(double initial) => _last = initial;

    // Если новое значение не отличается от старого — завершается синхронно,
    // БЕЗ аллокации Task. Иначе возвращаем Task.CompletedTask-стиль через ValueTask.
    // If the new value equals the old one, completes synchronously with no
    // Task allocation; otherwise returns a ValueTask wrapping a small task.
    public ValueTask<double> ReadAsync()
    {
        double snapshot;
        lock (_gate) { snapshot = _last; }   // lock без await — безопасно.
                                             // lock without await — safe.

        // Готовый результат -> ValueTask<T> не аллоцирует Task.
        // Ready result -> ValueTask<T> allocates no Task.
        return new ValueTask<double>(snapshot);
    }

    public void Update(double value)
    {
        // lock защищает запись. Никаких await внутри lock!
        // lock guards the write. No await inside lock!
        lock (_gate) { _last = value; }
    }
}

// Демонстрация корректного потребления: ОДИН await на каждый ValueTask.
// Demonstration of correct consumption: ONE await per ValueTask.
public static class Demo
{
    public static async Task RunAsync()
    {
        var cache = new AsyncValueCache<string, string>(
            async (key, ct) =>
            {
                await Task.Delay(50, ct).ConfigureAwait(false); // имитация I/O
                return key.ToUpperInvariant();
            });

        // Параллельные запросы одного ключа: stampede предотвращён семафором.
        // Concurrent requests for the same key: stampede prevented by semaphore.
        string[] keys = ["a", "b", "a", "c", "a"];
        var tasks = keys.Select(k => ConsumeAsync(cache, k)).ToArray();
        string[] results = await Task.WhenAll(tasks).ConfigureAwait(false);
        Console.WriteLine(string.Join(", ", results)); // A, B, A, C, A
    }

    // Каждый ValueTask await'ится ровно один раз — это и есть контракт.
    // Each ValueTask is awaited exactly once — that is the contract.
    private static async Task ConsumeAsync(AsyncValueCache<string, string> cache, string key)
    {
        ValueTask<string> vt = cache.GetAsync(key); // не awaited ещё — можно хранить.
                                                    // not awaited yet — safe to hold.
        string value = await vt.ConfigureAwait(false); // единственный await.
                                                       // the single await.
        Console.WriteLine($"{key} -> {value}");
    }
}

// [UnsafeAccessor-демонстрация опущена: ValueTask — тип значения, и
//  случайно «протекать» его внутреннее состояние через рефлексию нельзя.]
```

#### Best Practices

- Используйте `ValueTask<T>` только на горячих путях, где результат часто готов синхронно; иначе оставайтесь на `Task<T>` — он проще.
- Возвращайте `ValueTask` из метода и **сразу** awaited вызывающим: один экземпляр — один await — один поток.
- Кэшируйте только **завершённые** `ValueTask`/`ValueTask<T>` (с предвычисленным результатом); незавершённые кэшировать нельзя.
- В библиотечном коде всегда добавляйте `.ConfigureAwait(false)`, чтобы не тянуть контекст синхронизации вызывающего.
- Делайте кэш потокобезопасным (`ConcurrentDictionary`, `Volatile`, `lock` **без** await внутри) и защищайте медленный путь `SemaphoreSlim` от stampede.

#### Best Practices (EN)

- Use `ValueTask<T>` only on hot paths where the result is frequently ready synchronously; otherwise stay with `Task<T>` — it is simpler.
- Return a `ValueTask` from a method and have the caller **immediately** await it: one instance, one await, one thread.
- Cache only **completed** `ValueTask`/`ValueTask<T>` (with a precomputed result); incomplete ones must not be cached.
- Always add `.ConfigureAwait(false)` in library code so you do not pull in the caller's synchronization context.
- Make the cache thread-safe (`ConcurrentDictionary`, `Volatile`, a `lock` **without** await inside) and guard the slow path with `SemaphoreSlim` to prevent stampedes.

#### Частые ошибки / Common Mistakes

- **Повторный await того же ValueTask** → храните результат в локальной переменной после первого await; повторно ожидайте сохранённый `T`, а не сам `ValueTask`.
- **`.Result` / `.Wait()` на ValueTask в контексте синхронизации** → используйте `await` (с `ConfigureAwait(false)`); блокировка потока — самодедлок.
- **`lock { await ... }`** → никогда не держите `lock` через `await`; используйте `SemaphoreSlim.WaitAsync` или перестраивайте код так, чтобы await был снаружи критической секции.
- **Кэширование незавершённого ValueTask** → кэшируйте только `IsCompleted == true`, либо кэшируйте само значение `T` и оборачивайте в `new ValueTask<T>(value)` при выдаче.
- **Возврат ValueTask «потому что модно»** для всегда-асинхронной операции → используйте `Task<T>`; лишняя сложность ValueTask не окупится.
- **Доступ к ValueTask из нескольких потоков одновременно** → ValueTask — не потокобезопасный для конкурентного ожидания; передавайте его одному потребителю.

#### Common Mistakes (EN)

- **Awaiting the same ValueTask twice** → store the result in a local after the first await; re-use the saved `T`, never the `ValueTask` itself.
- **`.Result` / `.Wait()` on a ValueTask inside a sync context** → use `await` (with `ConfigureAwait(false)`); blocking the thread is a self-deadlock.
- **`lock { await ... }`** → never hold a `lock` across an `await`; use `SemaphoreSlim.WaitAsync` or restructure so the await is outside the critical section.
- **Caching an incomplete ValueTask** → cache only `IsCompleted == true`, or cache the `T` value itself and wrap it as `new ValueTask<T>(value)` on the way out.
- **Returning ValueTask "because it's trendy"** for an always-async operation → use `Task<T>`; the extra complexity of ValueTask will not pay off.
- **Accessing a ValueTask from multiple threads concurrently** → ValueTask is not safe for concurrent awaiting; hand it to a single consumer.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я использую `ValueTask<T>` только там, где результат часто готов синхронно (горячий путь, кэш).
- [ ] Каждый возвращённый `ValueTask` awaited ровно один раз одним потребителем.
- [ ] Кэш потокобезопасен (`ConcurrentDictionary` / `Volatile` / `lock` без await).
- [ ] Медленный путь защищён `SemaphoreSlim` от cache stampede.
- [ ] В библиотечном коде везде стоит `.ConfigureAwait(false)`.
- [ ] Я не вызываю `.Result`/`.Wait()` и не ставлю `lock` вокруг `await`.
- [ ] Я кэширую только завершённые ValueTask (или само значение `T`).

#### Self-check Checklist (EN)

- [ ] I use `ValueTask<T>` only where the result is frequently ready synchronously (hot path, cache).
- [ ] Every returned `ValueTask` is awaited exactly once by a single consumer.
- [ ] The cache is thread-safe (`ConcurrentDictionary` / `Volatile` / `lock` without await).
- [ ] The slow path is protected by `SemaphoreSlim` against cache stampede.
- [ ] `.ConfigureAwait(false)` is applied everywhere in library code.
- [ ] I never call `.Result`/`.Wait()` and never hold a `lock` across an `await`.
- [ ] I cache only completed ValueTasks (or the `T` value itself).

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.threading.tasks.valuetask-1](https://learn.microsoft.com/dotnet/api/system.threading.tasks.valuetask-1)
- [Microsoft Learn — ValueTask Source](https://learn.microsoft.com/dotnet/api/system.threading.tasks.sources.ivaluetasksource-1)

---

[⬆ К модулю M09](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
