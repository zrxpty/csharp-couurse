---
[← К уроку M09-L08](lesson-M09-L08-valuetask.md) | [⬆ К модулю M09](../README.md) | [Следующее ДЗ →](homework-M09-L09-async-antipatterns.md)
---

### Домашнее задание M09-L08: ValueTask, кэшированные результаты / Homework M09-L08: ValueTask, cached results

**Урок / Lesson:** M09-L08
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться применять `ValueTask<T>` на горячих путях с кэшированными результатами: построить потокобезопасный асинхронный кэш без аллокаций при попадании, защитить медленный путь от cache stampede через `SemaphoreSlim`, соблюсти контракт «один await на экземпляр» и избежать типичных ошибок (повторный await, `.Result`, `lock`+`await`, кэширование незавершённого ValueTask). (EN) Learn to apply `ValueTask<T>` on hot paths with cached results: build a thread-safe asynchronous cache with no allocation on cache hits, protect the slow path against cache stampedes with `SemaphoreSlim`, honour the one-await-per-instance contract, and avoid the classic mistakes (double await, `.Result`, `lock`+`await`, caching an incomplete ValueTask).

#### Связь с уроком / Connection to the lesson
(RU) Урок объясняет, почему `ValueTask<T>` — значимый тип, экономящий аллокации при синхронном завершении, и формулирует контракт «один await на экземпляр». Это ДЗ превращает теорию в практику: вы строите именно тот `AsyncValueCache`, который иллюстрирует урок, и проходите через все «острые углы» — двойную проверку, `SemaphoreSlim`, `ConfigureAwait(false)` и разницу между кэшированием значения `T` и кэшированием `ValueTask`.
(EN) The lesson explains why `ValueTask<T>` is a value type that saves allocations when a method completes synchronously, and states the one-await-per-instance contract. This homework turns that theory into practice: you build exactly the `AsyncValueCache` the lesson illustrates and walk through every sharp edge — double-checked locking, `SemaphoreSlim`, `ConfigureAwait(false)`, and the difference between caching a `T` value versus caching a `ValueTask`.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы разрабатываете сервис конфигурации, который отдаёт «тяжёлые» настройки по строковому ключу. На проде настройки почти не меняются, но читаются тысячи раз в секунду из многих потоков. Сегодня сервис возвращает `Task<Config>` и при каждом попадании в кэш всё равно аллоцирует новый `Task` через `Task.FromResult` — это создаёт заметное давление на GC в горячем пути. Профайлер показывает, что более 60 % аллокаций в этом пути приходится именно на `Task`-обёртки для готовых значений.

Параллельно есть проблема «cache stampede»: когда ключ впервые промахивается, десятки параллельных запросов одновременно идут в базу, потому что кэш ещё пуст, а каждый поток видит промах и запускает загрузку. Это выбивает соединения из пула и удлиняет задержку для всех. Наконец, в коде библиотеки есть места, где `await` выполняется без `ConfigureAwait(false)`, и при вызове из UI-потока или legacy ASP.NET это приводит к зависаниям.

Урок M09-L08 даёт инструменты для решения всех трёх проблем сразу: `ValueTask<T>` убирает аллокацию при попадании в кэш, `SemaphoreSlim` на ключ защищает от stampede, а `ConfigureAwait(false)` избавляет от захвата контекста синхронизации. Ваша задача — собрать эти инструменты в один корректный, потокобезопасный, хорошо протестированный компонент и доказать его свойства демо-программой. Важно не просто «сделать, чтобы компилировалось», а соблюсти контракт ValueTask: один await на экземпляр, один потребитель, никаких блокировок потока и никаких `lock` вокруг `await`.

#### Что нужно сделать (пошагово)

1. Создайте новый консольный проект .NET 8 с именем `ValueCacheLab`:
   ```bash
   dotnet new console -n ValueCacheLab -o ValueCacheLab --framework net8.0
   cd ValueCacheLab
   ```
   Убедитесь, что в `ValueCacheLab.csproj` стоит `<LangVersion>latest</LangVersion>` и `<Nullable>enable</Nullable>` (для C# 12 и nullable reference types).

2. В файле `AsyncValueCache.cs` реализуйте обобщённый класс `AsyncValueCache<TKey, TValue> where TKey : notnull` со следующими элементами:
   - приватное поле `private readonly ConcurrentDictionary<TKey, TValue> _cache = new();`
   - приватное поле `private readonly ConcurrentDictionary<TKey, SemaphoreSlim> _locks = new();`
   - поле `private readonly Func<TKey, CancellationToken, Task<TValue>> _loadAsync;`, инициализируемое в конструкторе;
   - публичный метод `public async ValueTask<TValue> GetAsync(TKey key, CancellationToken ct = default)`, реализующий логику ниже.

3. Логика `GetAsync` (строго по уроку):
   - Быстрый путь: если `_cache.TryGetValue(key, out var cached)` вернул `true`, немедленно `return cached;` — `ValueTask<T>` обернёт готовый результат без аллокации `Task`.
   - Медленный путь: получите семафор на ключ через `_locks.GetOrAdd(key, _ => new SemaphoreSlim(1, 1))`.
   - Выполните `await gate.WaitAsync(ct).ConfigureAwait(false);` — НЕ блокируйте поток и НЕ используйте `lock` вокруг `await`.
   - В блоке `try` сделайте повторную проверку `_cache.TryGetValue` — пока вы ждали семафора, другой поток мог уже положить значение.
   - Если повторно найдено — `return cached;`.
   - Иначе `var value = await _loadAsync(key, ct).ConfigureAwait(false);`, затем `_cache.GetOrAdd(key, value);` и `return value;`.
   - В `finally` выполните `gate.Release();`.

4. В файле `BoundedTemperatureReader.cs` реализуйте класс, который всегда завершается синхронно и возвращает `ValueTask<double>` без какого-либо `Task` вообще:
   - поле `private double _last;` и объект `_gate` для `lock`;
   - конструктор принимает начальное значение;
   - метод `public ValueTask<double> ReadAsync()` делает снимок под `lock` (без `await`) и возвращает `new ValueTask<double>(snapshot);`;
   - метод `public void Update(double value)` пишет под `lock`.
   Покажите, что этот метод действительно не аллоцирует `Task` при каждом вызове (см. шаг 6).

5. В `Program.cs` используйте top-level statements C# 12 и collection expressions. Постройте демонстрацию:
   ```csharp
   using var cache = new AsyncValueCache<string, string>(async (key, ct) =>
   {
       await Task.Delay(50, ct).ConfigureAwait(false); // имитация I/O
       return key.ToUpperInvariant();
   });
   string[] keys = ["a", "b", "a", "c", "a"];
   var tasks = keys.Select(k => ConsumeAsync(cache, k)).ToArray();
   string[] results = await Task.WhenAll(tasks).ConfigureAwait(false);
   Console.WriteLine(string.Join(", ", results)); // ожидаемо: A, B, A, C, A
   ```
   Каждый `ValueTask` awaited ровно один раз в `ConsumeAsync`. Запустите:
   ```bash
   dotnet run
   ```
   Ожидаемый вывод: `A, B, A, C, A`.

6. Добавьте измерение: счётчик вызовов фабрики. Заведите `int factoryCalls = 0;` и внутри фабрики делайте `Interlocked.Increment(ref factoryCalls);`. После запуска убедитесь, что для повторяющихся ключей `"a"` фабрика вызвана ровно один раз — это доказывает, что stampede предотвращён и что быстрые попадания идут через `ValueTask` без новой загрузки. Выведите `factoryCalls` в конце.

7. Напишите метод-демонстрацию ошибки (закомментированный, с пояснением), которая нарушает контракт «один await на экземпляр»: вызывает `GetAsync`, а затем пытается `await` один и тот же `ValueTask` дважды или из двух `Task.Run`. Поясните в комментарии, почему поведение не определено и почему при источнике `IValueTaskSource` второй `await` может выбросить `InvalidOperationException`.

8. Проверьте сборку и предупреждения:
   ```bash
   dotnet build -warnaserror
   ```
   Код должен компилироваться без предупреждений анализатора (nullable, unused, async-анализаторы).

#### Требования к решению

- Целевой фреймворк — `net8.0`, язык C# 12. Используйте top-level statements в `Program.cs`, collection expressions (`["a", "b", "a", "c", "a"]`), file-scoped namespaces, `var`, target-typed `new()`.
- `GetAsync` возвращает именно `ValueTask<TValue>`, а не `Task<TValue>`. При попадании в кэш не должно быть аллокации `Task` — ValueTask оборачивает готовое значение напрямую.
- Кэш `_cache` и словарь семафоров `_locks` должны быть `ConcurrentDictionary<,>`. Запрещён обычный `Dictionary` с ручным `lock` вокруг `await`.
- Медленный путь обязан использовать `SemaphoreSlim.WaitAsync` для защиты от stampede; после захвата — обязательная повторная проверка кэша. `Release` — в `finally`.
- Везде, где есть `await` внутри библиотечного класса, должен стоять `.ConfigureAwait(false)`. В `Program.cs` (точка входа приложения) `ConfigureAwait(false)` желателен, но не обязателен.
- Запрещено: `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` на `ValueTask`/`Task` в коде библиотеки; `lock` вокруг `await`; кэширование незавершённого `ValueTask`; повторный `await` того же экземпляра `ValueTask`.
- Демонстрация `BoundedTemperatureReader` должна возвращать `ValueTask<double>` без создания `Task` на быстром пути (`new ValueTask<double>(snapshot)`).
- Должен быть вывод `factoryCalls`, показывающий, что фабрика вызвана один раз на уникальный ключ даже при параллельных запросах.
- Код должен быть оформлен с двуязычными комментариями (RU+EN) в ключевых местах, как в уроке.

#### Тонкости и подводные камни

- **Один await на экземпляр.** Это главное правило ValueTask. Если вы сохранили `ValueTask` в поле и два разных потребителя попытаются его дождаться — поведение не определено. При источнике `IValueTaskSource` второй `await` может выбросить `InvalidOperationException`. Запомните: «`ValueTask` — это не `Task`, его не шарят». Храните результат `T`, а не сам `ValueTask`.
- **Кэширование самого ValueTask — только завершённого.** Если очень хочется закэшировать `ValueTask<T>`, проверяйте `IsCompleted == true`. Иначе лучше кэшировать само значение `T` и оборачивать `new ValueTask<T>(value)` при выдаче — так вы всегда раздаёте свежий, завершённый ValueTask.
- **`lock` вокруг `await` — запрещён.** `lock` (через `Monitor`) не может пережить `await`: асинхронное продолжение может выполниться в другом потоке, а `Monitor` привязан к потоку владения. Это дедлок или нарушение инвариантов. Используйте `SemaphoreSlim.WaitAsync` или выносите `await` за пределы критической секции.
- **`.Result`/`.Wait()` в контексте синхронизации — самодедлок.** В UI-потоке или legacy ASP.NET блокирующее ожидание `ValueTask`, который сам ждёт продолжения в этом же контексте, намертво зависает. Всегда `await` (с `ConfigureAwait(false)` в библиотеке).
- **`ConfigureAwait(false)` в библиотеке обязателен.** Иначе вы захватываете контекст вызывающего, и ваш кэш «тянет» UI-поток или старый ASP.NET-контекст. В точке входа приложения (`Main`) это не критично, но в библиотеке — обязательно.
- **Двойная проверка после семафора.** Пока вы ждали семафора, другой поток мог уже положить значение. Если не проверить повторно, вы вызовете фабрику дважды (хотя и не параллельно). Это не дедлок, но нарушение обещания «фабрика один раз на ключ».
- **`GetOrAdd` для семафора.** Используйте `_locks.GetOrAdd(key, _ => new SemaphoreSlim(1, 1))`. Не используйте `TryAdd` с последующим `TryGetValue` — это гонка. Делегат `GetOrAdd` может выполниться несколько раз, но `SemaphoreSlim` лёгкий; важен один общий экземпляр на ключ.
- **`_cache.GetOrAdd(key, value)` против `_cache[key] = value`.** `GetOrAdd` атомарно вернёт либо ваше, либо уже существующее значение — это безопаснее при гонках. Если кто-то уже положил значение, `GetOrAdd` вернёт его, а не ваше; в большинстве сценариев это приемлемо (значения эквивалентны).
- **`ValueTask` больше `Task` по размеру.** `ValueTask<T>` — структура с несколькими полями; передача по значению копирует её. Не возвращайте `ValueTask` «потому что модно» для всегда-асинхронных операций — лишний размер и сложность не окупятся.
- **Не отражайте внутренности ValueTask.** Внутреннее состояние ValueTask (поле `_obj`) деталь реализации; случайное «протекание» его через рефлексию или `UnsafeAccessor` может сломать контракт. Обращайтесь с ValueTask как с непрозрачным значением.

#### Критерии приёмки

- [ ] Проект `ValueCacheLab` собирается под `net8.0` командой `dotnet build -warnaserror` без ошибок и предупреждений.
- [ ] `AsyncValueCache<TKey, TValue>` реализован с ограничением `where TKey : notnull` и возвращает `ValueTask<TValue>` из `GetAsync`.
- [ ] Быстрый путь возвращает готовое значение `cached` напрямую (без создания `Task`).
- [ ] Медленный путь использует `_locks.GetOrAdd(...)` + `await gate.WaitAsync(ct).ConfigureAwait(false)`.
- [ ] После захвата семафора есть повторная проверка кэша (double-check).
- [ ] `gate.Release()` находится в `finally`.
- [ ] Все `await` внутри библиотечного класса сопровождаются `.ConfigureAwait(false)`.
- [ ] В коде нет `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` на `ValueTask`/`Task`.
- [ ] Нет `lock` вокруг `await`.
- [ ] `BoundedTemperatureReader.ReadAsync` возвращает `new ValueTask<double>(snapshot)` без `Task`.
- [ ] Демонстрация в `Program.cs` использует collection expressions и top-level statements.
- [ ] После запуска `dotnet run` выводится `A, B, A, C, A`.
- [ ] Счётчик `factoryCalls` показывает, что фабрика вызвана один раз на уникальный ключ (3 уникальных ключа → 3 вызова при 5 запросах).
- [ ] Есть закомментированная демонстрация нарушения контракта «один await» с пояснением.
- [ ] Ключевые места снабжены двуязычными комментариями RU+EN.

#### Подсказки (без прямого ответа)

- Вспомните, что `return cached;` внутри `async ValueTask<T> GetAsync` создаёт `ValueTask`, оборачивающий готовый результат без аллокации `Task` — именно это и даёт выигрыш.
- Для защиты от stampede вам нужен «одновременный пропуск только одного потока на ключ» — это классическая задача для `SemaphoreSlim(1, 1)`.
- Повторная проверка после захвата — это аналог double-checked locking, но в асинхронном мире: «пока я ждал, кто-то уже сделал работу за меня».
- Если фабрика возвращает `Task<T>`, а вы `await` её внутри `ValueTask`-метода, компилятор сам построит правильный `AsyncValueTaskMethodBuilder` — вам не нужно вручную создавать `IValueTaskSource`.
- Для `BoundedTemperatureReader` ключевое слово `async` в сигнатуре `ReadAsync` **не нужно**: метод не содержит `await`, и отсутствие `async` позволяет вернуть `ValueTask<double>` буквально через `new`, без машины состояний.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8. Потокобезопасный асинхронный кэш на ValueTask<T>.
// Thread-safe asynchronous cache on ValueTask<T>.
// Демонстрирует: отсутствие аллокации при попадании, защиту от stampede
// через SemaphoreSlim, корректный контракт ValueTask (один await, ConfigureAwait).
// Demonstrates: allocation-free cache hits, stampede protection via
// SemaphoreSlim, correct ValueTask contract (single await, ConfigureAwait).

using System.Collections.Concurrent;
using System.Runtime.CompilerServices;

namespace ValueCacheLab;

// Обобщённый асинхронный кэш. TKey : notnull — обязательное ограничение для
// использования в качестве ключа словаря. / Generic async cache. TKey : notnull
// is mandatory because the key is used as a dictionary key.
public sealed class AsyncValueCache<TKey, TValue> where TKey : notnull
{
    // Кэш готовых значений. ConcurrentDictionary даёт потокобезопасный доступ.
    // Cache of ready values; ConcurrentDictionary provides thread-safe access.
    private readonly ConcurrentDictionary<TKey, TValue> _cache = new();

    // Фабрика «дорогого» значения (например, запрос в БД или удалённый API).
    // Factory for the "expensive" value (e.g. a DB call or remote API).
    private readonly Func<TKey, CancellationToken, Task<TValue>> _loadAsync;

    // Семафор на ключ: защищает от cache stampede. / Per-key semaphore:
    // guards against cache stampede.
    private readonly ConcurrentDictionary<TKey, SemaphoreSlim> _locks = new();

    public AsyncValueCache(Func<TKey, CancellationToken, Task<TValue>> loadAsync)
    {
        _loadAsync = loadAsync;
    }

    // ВАЖНО: возвращаем ValueTask<T>. На попадании в кэш — НЕТ аллокации Task.
    // IMPORTANT: returns ValueTask<T>. On a cache hit there is NO Task allocation.
    public async ValueTask<TValue> GetAsync(TKey key, CancellationToken ct = default)
    {
        // Быстрый путь: значение уже в кэше. / Fast path: value already cached.
        if (_cache.TryGetValue(key, out var cached))
        {
            return cached; // ValueTask<T> обёрнёт результат без аллокации.
                           // ValueTask<T> wraps the result with no allocation.
        }

        // Медленный путь: загрузка. Берём семафор на ключ. / Slow path: load.
        var gate = _locks.GetOrAdd(key, _ => new SemaphoreSlim(1, 1));

        await gate.WaitAsync(ct).ConfigureAwait(false); // НЕ блокируем поток!
                                                         // Do NOT block the thread!
        try
        {
            // Двойная проверка: пока ждали семафора, кто-то мог заполнить кэш.
            // Double-check: while waiting, someone may have filled the cache.
            if (_cache.TryGetValue(key, out cached))
            {
                return cached;
            }

            // Реальная асинхронная загрузка. ConfigureAwait(false) — чтобы не
            // захватывать контекст вызывающего. / Real async load.
            var value = await _loadAsync(key, ct).ConfigureAwait(false);

            // Атомарная запись. GetOrAdd безопаснее прямого индексатора при гонках.
            // Atomic store. GetOrAdd is safer than the indexer under races.
            _cache.GetOrAdd(key, value);
            return value;
        }
        finally
        {
            gate.Release(); // всегда освобождаем семафор. / always release.
        }
    }
}

// Синхронный источник ValueTask без Task вообще. / Synchronous ValueTask
// source with no Task at all.
public sealed class BoundedTemperatureReader
{
    private double _last;          // последнее значение под lock. / last under lock.
    private readonly object _gate = new();

    public BoundedTemperatureReader(double initial) => _last = initial;

    // Метод НЕ async: нет await, нет машины состояний, ValueTask строится напрямую.
    // The method is NOT async: no await, no state machine, ValueTask built directly.
    public ValueTask<double> ReadAsync()
    {
        double snapshot;
        lock (_gate) { snapshot = _last; } // lock без await — безопасно.
                                           // lock without await — safe.
        return new ValueTask<double>(snapshot); // готовый результат, без Task.
                                                 // ready result, no Task.
    }

    public void Update(double value)
    {
        lock (_gate) { _last = value; } // lock защищает запись. / guards the write.
    }
}

// Программа-демонстрация (top-level statements C# 12). / Demo (top-level).
using System.Runtime.CompilerServices;

int factoryCalls = 0;

var cache = new AsyncValueCache<string, string>(async (key, ct) =>
{
    await Task.Delay(50, ct).ConfigureAwait(false); // имитация I/O. / fake I/O.
    Interlocked.Increment(ref factoryCalls);        // счётчик вызовов. / counter.
    return key.ToUpperInvariant();
});

string[] keys = ["a", "b", "a", "c", "a"]; // collection expression. / collection expr.
var tasks = keys.Select(k => ConsumeAsync(cache, k)).ToArray();
string[] results = await Task.WhenAll(tasks).ConfigureAwait(false);

Console.WriteLine(string.Join(", ", results));      // A, B, A, C, A
Console.WriteLine($"factoryCalls={factoryCalls}");  // 3 (уникальных ключа: a, b, c)

// Каждый ValueTask await'ится ровно один раз — это и есть контракт.
// Each ValueTask is awaited exactly once — that is the contract.
static async Task ConsumeAsync(AsyncValueCache<string, string> cache, string key)
{
    ValueTask<string> vt = cache.GetAsync(key);      // ещё не awaited — можно хранить.
                                                     // not awaited yet — safe to hold.
    string value = await vt.ConfigureAwait(false);   // единственный await. / single await.
    Console.WriteLine($"{key} -> {value}");
}

/*
// ИЛЛЮСТРАЦИЯ ОШИБКИ: так делать НЕЛЬЗЯ. / MISTAKE: never do this.
//
// static async Task BrokenConsumeAsync(AsyncValueCache<string,string> cache, string key)
// {
//     ValueTask<string> vt = cache.GetAsync(key);
//     string first  = await vt.ConfigureAwait(false); // первый await — ок. / first ok.
//     string second = await vt.ConfigureAwait(false); // ВТОРОЙ await — НЕОПРЕДЕЛЕНО!
//                                                      // SECOND await — UNDEFINED!
//     // При источнике IValueTaskSource второй await может выбросить
//     // InvalidOperationException: "Operations should not be called on a
//     // ValueTask after it has completed."
// }
//
// Правильно: сохранить РЕЗУЛЬТАТ (string), а не сам ValueTask.
// Correct: store the RESULT (string), not the ValueTask itself.
*/
```

Разбор по строкам. Класс `AsyncValueCache` помечен `sealed` — это снижает накладные расходы на виртуальные вызовы и запрещает наследование, что разумно для кэша как самодостаточного компонента. Ограничение `TKey : notnull` обязательно: `ConcurrentDictionary` требует хешируемый, не-null ключ. Поле `_cache` — `ConcurrentDictionary`, что даёт потокобезопасные `TryGetValue` и `GetOrAdd` без ручного `lock`. Поле `_locks` хранит `SemaphoreSlim` на ключ — это «диспетчер очереди» для медленного пути.

В `GetAsync` объявлено `async ValueTask<TValue>`: компилятор строит `AsyncValueTaskMethodBuilder`, который при синхронном завершении (быстрый путь) не создаёт `Task`, а упаковывает результат прямо в структуру `ValueTask`. Первая строка `if (_cache.TryGetValue(key, out var cached)) return cached;` — это и есть «аллокация-free» попадание: мы не вызываем `Task.FromResult`, не создаём объект в куче. Строка `var gate = _locks.GetOrAdd(key, _ => new SemaphoreSlim(1, 1));` гарантирует, что для одного ключа все потоки получат один и тот же семафор, даже если делегат выполнится несколько раз — `GetOrAdd` вернёт уже существующий экземпляр. `await gate.WaitAsync(ct).ConfigureAwait(false);` — асинхронное ожидание без блокировки потока и без захвата контекста; это предотвращает самодедлок и «залипание» UI. Повторная проверка `if (_cache.TryGetValue(...)) return cached;` внутри `try` — аналог double-checked locking в асинхронном мире: пока мы стояли в очереди за семафором, кто-то мог уже завершить загрузку и нам не нужно вызывать фабрику повторно. `await _loadAsync(...).ConfigureAwait(false)` — единственный «настоящий» асинхронный вызов; именно здесь может происходить I/O. `_cache.GetOrAdd(key, value)` атомарно записывает или возвращает уже существующее значение (если кто-то успел первым). `gate.Release()` в `finally` гарантирует освобождение даже при исключении из фабрики.

Класс `BoundedTemperatureReader` намеренно **не** использует модификатор `async`: метод `ReadAsync` не содержит `await`, поэтому отсутствие `async` позволяет вернуть `ValueTask<double>` через `new ValueTask<double>(snapshot)` напрямую, без генерации машины состояний и без создания `Task`. `lock (_gate)` без `await` внутри — безопасен. В демо `int factoryCalls` инкрементируется через `Interlocked.Increment` — потокобезопасно. Collection expression `["a","b","a","c","a"]` — синтаксис C# 12. Каждый `ValueTask` awaited ровно один раз в `ConsumeAsync` — контракт соблюдён. Закомментированный блок `BrokenConsumeAsync` показывает нарушение: второй `await` того же экземпляра не определён и может выбросить `InvalidOperationException` при `IValueTaskSource`-источнике. Применённые концепции урока: значимый тип `ValueTask<T>` против ссылочного `Task<T>`, кэширование готового значения (а не `ValueTask`), `ConfigureAwait(false)`, `SemaphoreSlim` против stampede, запрет `lock`+`await` и `.Result`, контракт «один await на экземпляр».

#### Задания на углубление (бонус)

1. **Измерьте аллокации.** Подключите `BenchmarkDotNet` и сравните `GetAsync` на `ValueTask<T>` с аналогом на `Task<T>` (`Task.FromResult`) при 100 % попадании в кэш. Должны увидеть заметно меньшее `Allocated` у версии на `ValueTask`. Запишите результат в `BENCHMARKS.md`.
2. **TTL и инвалидация.** Добавьте к кэшу время жизни записи: храните `record struct Entry(TValue Value, DateTime ExpiresAt)` и возвращайте `ValueTask` только если запись не протухла. Подумайте, как инвалидировать семафор и запись атомарно.
3. **Ручной `IValueTaskSource<T>`.** Реализуйте простейший пулляемый источник (пусть даже игрушечный) и заставьте `ValueTask<T>` использовать его вместо `Task`. Убедитесь, что понимаете `GetStatus`/`GetResult`/`OnCompleted` и `ValueTaskSourceStatus`. Это сложная задача — отразите сложности в комментарии.
4. **Параллельный прогрев.** Добавьте тест, который параллельно из 50 потоков запрашивает один и тот же ключ впервые и проверяет, что фабрика вызвана ровно один раз. Используйте `Parallel.For` или `Task.WhenAll` с массивом задач.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are building a configuration service that returns "heavy" settings by string key. In production the settings barely change, yet they are read thousands of times per second from many threads. Today the service returns `Task<Config>`, and on every cache hit it still allocates a fresh `Task` via `Task.FromResult` — that creates visible GC pressure on the hot path. The profiler shows that more than 60 % of allocations on this path are precisely the `Task` wrappers around already-ready values.

In parallel there is a cache-stampede problem: when a key misses for the first time, dozens of concurrent requests all fall through to the database simultaneously, because the cache is still empty and every thread sees a miss and starts a load. This exhausts the connection pool and inflates latency for everyone. Finally, the library code has places where `await` runs without `ConfigureAwait(false)`, and when called from a UI thread or legacy ASP.NET context this causes stalls.

Lesson M09-L08 gives you the tools to fix all three problems at once: `ValueTask<T>` removes the allocation on a cache hit, a per-key `SemaphoreSlim` protects against stampedes, and `ConfigureAwait(false)` avoids capturing the synchronization context. Your task is to assemble these tools into a single correct, thread-safe, well-tested component and prove its properties with a demo program. The goal is not merely "make it compile" but to honour the ValueTask contract: one await per instance, one consumer, no thread blocking, no `lock` around `await`.

#### What to do step by step

1. Create a new .NET 8 console project named `ValueCacheLab`:
   ```bash
   dotnet new console -n ValueCacheLab -o ValueCacheLab --framework net8.0
   cd ValueCacheLab
   ```
   Ensure that `ValueCacheLab.csproj` has `<LangVersion>latest</LangVersion>` and `<Nullable>enable</Nullable>` (for C# 12 and nullable reference types).

2. In the file `AsyncValueCache.cs` implement the generic class `AsyncValueCache<TKey, TValue> where TKey : notnull` with the following members:
   - private field `private readonly ConcurrentDictionary<TKey, TValue> _cache = new();`
   - private field `private readonly ConcurrentDictionary<TKey, SemaphoreSlim> _locks = new();`
   - field `private readonly Func<TKey, CancellationToken, Task<TValue>> _loadAsync;`, initialised in the constructor;
   - public method `public async ValueTask<TValue> GetAsync(TKey key, CancellationToken ct = default)` implementing the logic below.

3. Logic of `GetAsync` (strictly following the lesson):
   - Fast path: if `_cache.TryGetValue(key, out var cached)` returned `true`, immediately `return cached;` — `ValueTask<T>` wraps the ready result with no `Task` allocation.
   - Slow path: obtain the per-key semaphore via `_locks.GetOrAdd(key, _ => new SemaphoreSlim(1, 1))`.
   - Execute `await gate.WaitAsync(ct).ConfigureAwait(false);` — do NOT block the thread and do NOT use `lock` around `await`.
   - Inside the `try` block, re-check `_cache.TryGetValue` — while you were waiting for the semaphore, another thread may have already populated the value.
   - If found again — `return cached;`.
   - Otherwise `var value = await _loadAsync(key, ct).ConfigureAwait(false);`, then `_cache.GetOrAdd(key, value);` and `return value;`.
   - In `finally` call `gate.Release();`.

4. In the file `BoundedTemperatureReader.cs` implement a class that always completes synchronously and returns `ValueTask<double>` with no `Task` at all:
   - field `private double _last;` and a `_gate` object for `lock`;
   - constructor accepts the initial value;
   - method `public ValueTask<double> ReadAsync()` takes a snapshot under `lock` (no `await`) and returns `new ValueTask<double>(snapshot);`;
   - method `public void Update(double value)` writes under `lock`.
   Demonstrate that this method indeed allocates no `Task` per call (see step 6).

5. In `Program.cs` use C# 12 top-level statements and collection expressions. Build the demo:
   ```csharp
   using var cache = new AsyncValueCache<string, string>(async (key, ct) =>
   {
       await Task.Delay(50, ct).ConfigureAwait(false); // fake I/O
       return key.ToUpperInvariant();
   });
   string[] keys = ["a", "b", "a", "c", "a"];
   var tasks = keys.Select(k => ConsumeAsync(cache, k)).ToArray();
   string[] results = await Task.WhenAll(tasks).ConfigureAwait(false);
   Console.WriteLine(string.Join(", ", results)); // expected: A, B, A, C, A
   ```
   Each `ValueTask` is awaited exactly once inside `ConsumeAsync`. Run it:
   ```bash
   dotnet run
   ```
   Expected output: `A, B, A, C, A`.

6. Add a measurement: a factory-call counter. Declare `int factoryCalls = 0;` and inside the factory do `Interlocked.Increment(ref factoryCalls);`. After the run, confirm that for the repeated key `"a"` the factory was invoked exactly once — this proves the stampede is prevented and that fast hits go through `ValueTask` without a new load. Print `factoryCalls` at the end.

7. Write an error-demo method (commented out, with an explanation) that violates the one-await-per-instance contract: it calls `GetAsync` and then tries to `await` the same `ValueTask` twice, or from two `Task.Run`s. Explain in the comment why the behaviour is undefined and why, with an `IValueTaskSource` backing, the second `await` may throw `InvalidOperationException`.

8. Verify the build and warnings:
   ```bash
   dotnet build -warnaserror
   ```
   The code must compile with no analyzer warnings (nullable, unused, async analyzers).

#### Requirements

- Target framework `net8.0`, language C# 12. Use top-level statements in `Program.cs`, collection expressions (`["a", "b", "a", "c", "a"]`), file-scoped namespaces, `var`, target-typed `new()`.
- `GetAsync` returns `ValueTask<TValue>`, not `Task<TValue>`. On a cache hit there must be no `Task` allocation — ValueTask wraps the ready value directly.
- The cache `_cache` and the semaphore map `_locks` must be `ConcurrentDictionary<,>`. A plain `Dictionary` with a manual `lock` around `await` is forbidden.
- The slow path must use `SemaphoreSlim.WaitAsync` to guard against stampedes; after acquiring it, a re-check of the cache is mandatory. `Release` goes in `finally`.
- Every `await` inside the library class must carry `.ConfigureAwait(false)`. In `Program.cs` (the application entry point) `ConfigureAwait(false)` is recommended but not required.
- Forbidden: `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` on `ValueTask`/`Task` in library code; `lock` around `await`; caching an incomplete `ValueTask`; awaiting the same `ValueTask` instance a second time.
- The `BoundedTemperatureReader` demo must return `ValueTask<double>` without creating a `Task` on the fast path (`new ValueTask<double>(snapshot)`).
- There must be a `factoryCalls` output showing the factory is invoked once per unique key even under parallel requests.
- The code must carry bilingual (RU+EN) comments at the key spots, as in the lesson.

#### Pitfalls

- **One await per instance.** This is the cardinal rule of ValueTask. If you store a `ValueTask` in a field and two consumers try to await it, the behaviour is undefined. With an `IValueTaskSource` backing, the second `await` may throw `InvalidOperationException`. Remember: "a `ValueTask` is not a `Task`, you do not share it." Store the result `T`, not the `ValueTask` itself.
- **Caching a ValueTask itself — only a completed one.** If you really must cache a `ValueTask<T>`, check `IsCompleted == true`. Otherwise it is safer to cache the `T` value itself and wrap it as `new ValueTask<T>(value)` on the way out — you always hand out a fresh, completed ValueTask.
- **`lock` around `await` is forbidden.** `lock` (via `Monitor`) cannot survive an `await`: the async continuation may run on a different thread, and `Monitor` is thread-affined. This is a deadlock or an invariant violation. Use `SemaphoreSlim.WaitAsync` or move the `await` out of the critical section.
- **`.Result`/`.Wait()` inside a sync context is a self-deadlock.** On a UI thread or legacy ASP.NET, blocking a `ValueTask` that itself needs to continue in that same context hangs for good. Always `await` (with `ConfigureAwait(false)` in the library).
- **`ConfigureAwait(false)` is mandatory in the library.** Otherwise you capture the caller's context and your cache drags in a UI thread or a legacy ASP.NET context. At the application entry point (`Main`) it is not critical, but in the library it is.
- **Double-check after the semaphore.** While you were waiting, another thread may have populated the value. If you do not re-check, you will call the factory twice (though not in parallel). That is not a deadlock, but it breaks the promise "factory once per key".
- **`GetOrAdd` for the semaphore.** Use `_locks.GetOrAdd(key, _ => new SemaphoreSlim(1, 1))`. Do not use `TryAdd` followed by `TryGetValue` — that is a race. The `GetOrAdd` delegate may run several times, but `SemaphoreSlim` is cheap; what matters is a single shared instance per key.
- **`_cache.GetOrAdd(key, value)` vs `_cache[key] = value`.** `GetOrAdd` atomically returns either yours or the already-existing value — safer under races. If someone already stored a value, `GetOrAdd` returns that one, not yours; in most scenarios this is acceptable (the values are equivalent).
- **`ValueTask` is larger than `Task`.** `ValueTask<T>` is a struct with several fields; passing it by value copies it. Do not return `ValueTask` "because it is trendy" for always-async operations — the extra size and complexity will not pay off.
- **Do not reflect into the internals of ValueTask.** Its internal state (the `_obj` field) is an implementation detail; accidentally leaking it through reflection or `UnsafeAccessor` can break the contract. Treat `ValueTask` as an opaque value.

#### Acceptance criteria

- [ ] The `ValueCacheLab` project builds under `net8.0` with `dotnet build -warnaserror`, with no errors and no warnings.
- [ ] `AsyncValueCache<TKey, TValue>` is implemented with the `where TKey : notnull` constraint and returns `ValueTask<TValue>` from `GetAsync`.
- [ ] The fast path returns the ready value `cached` directly (without creating a `Task`).
- [ ] The slow path uses `_locks.GetOrAdd(...)` + `await gate.WaitAsync(ct).ConfigureAwait(false)`.
- [ ] After acquiring the semaphore there is a re-check of the cache (double-check).
- [ ] `gate.Release()` is in `finally`.
- [ ] Every `await` inside the library class is accompanied by `.ConfigureAwait(false)`.
- [ ] There is no `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` on `ValueTask`/`Task` in the code.
- [ ] There is no `lock` around `await`.
- [ ] `BoundedTemperatureReader.ReadAsync` returns `new ValueTask<double>(snapshot)` without a `Task`.
- [ ] The demo in `Program.cs` uses collection expressions and top-level statements.
- [ ] Running `dotnet run` prints `A, B, A, C, A`.
- [ ] The `factoryCalls` counter shows the factory is invoked once per unique key (3 unique keys → 3 calls for 5 requests).
- [ ] There is a commented-out demo that violates the one-await contract, with an explanation.
- [ ] The key spots carry bilingual RU+EN comments.

#### Hints (no direct answer)

- Recall that `return cached;` inside `async ValueTask<T> GetAsync` builds a `ValueTask` wrapping the ready result with no `Task` allocation — that is exactly the win.
- For stampede protection you need "only one thread per key at a time" — a classic job for `SemaphoreSlim(1, 1)`.
- The re-check after acquiring is the async analogue of double-checked locking: "while I waited, someone already did the work for me".
- If the factory returns `Task<T>` and you `await` it inside a `ValueTask` method, the compiler builds the correct `AsyncValueTaskMethodBuilder` itself — you do not need to hand-roll an `IValueTaskSource`.
- For `BoundedTemperatureReader` the `async` keyword in the `ReadAsync` signature is **not** needed: the method contains no `await`, and omitting `async` lets you return `ValueTask<double>` literally via `new`, with no state machine.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8. Thread-safe async cache on ValueTask<T>.
// Demonstrates: allocation-free cache hits, stampede protection via
// SemaphoreSlim, correct ValueTask contract (single await, ConfigureAwait).

using System.Collections.Concurrent;
using System.Runtime.CompilerServices;

namespace ValueCacheLab;

public sealed class AsyncValueCache<TKey, TValue> where TKey : notnull
{
    // Cache of ready values; ConcurrentDictionary is thread-safe.
    private readonly ConcurrentDictionary<TKey, TValue> _cache = new();

    // Factory for the "expensive" value (e.g. a DB call or remote API).
    private readonly Func<TKey, CancellationToken, Task<TValue>> _loadAsync;

    // Per-key semaphore: guards against cache stampede.
    private readonly ConcurrentDictionary<TKey, SemaphoreSlim> _locks = new();

    public AsyncValueCache(Func<TKey, CancellationToken, Task<TValue>> loadAsync)
    {
        _loadAsync = loadAsync;
    }

    // IMPORTANT: returns ValueTask<T>. On a cache hit there is NO Task allocation.
    public async ValueTask<TValue> GetAsync(TKey key, CancellationToken ct = default)
    {
        // Fast path: value already cached.
        if (_cache.TryGetValue(key, out var cached))
        {
            return cached; // ValueTask<T> wraps the result with no allocation.
        }

        // Slow path: must load. Acquire the per-key semaphore.
        var gate = _locks.GetOrAdd(key, _ => new SemaphoreSlim(1, 1));

        await gate.WaitAsync(ct).ConfigureAwait(false); // Do NOT block the thread!
        try
        {
            // Double-check: while waiting, someone may have filled the cache.
            if (_cache.TryGetValue(key, out cached))
            {
                return cached;
            }

            // Real async load. ConfigureAwait(false) so we do not capture the
            // caller's synchronization context.
            var value = await _loadAsync(key, ct).ConfigureAwait(false);

            // Atomically store. GetOrAdd is safer than the indexer under races.
            _cache.GetOrAdd(key, value);
            return value;
        }
        finally
        {
            gate.Release(); // always release.
        }
    }
}

// Synchronous ValueTask source with no Task at all.
public sealed class BoundedTemperatureReader
{
    private double _last;          // last value under lock.
    private readonly object _gate = new();

    public BoundedTemperatureReader(double initial) => _last = initial;

    // The method is NOT async: no await, no state machine, ValueTask built directly.
    public ValueTask<double> ReadAsync()
    {
        double snapshot;
        lock (_gate) { snapshot = _last; } // lock without await — safe.
        return new ValueTask<double>(snapshot); // ready result, no Task.
    }

    public void Update(double value)
    {
        lock (_gate) { _last = value; } // lock guards the write.
    }
}

// Demo (top-level statements C# 12).
int factoryCalls = 0;

var cache = new AsyncValueCache<string, string>(async (key, ct) =>
{
    await Task.Delay(50, ct).ConfigureAwait(false); // fake I/O.
    Interlocked.Increment(ref factoryCalls);        // counter.
    return key.ToUpperInvariant();
});

string[] keys = ["a", "b", "a", "c", "a"]; // collection expression.
var tasks = keys.Select(k => ConsumeAsync(cache, k)).ToArray();
string[] results = await Task.WhenAll(tasks).ConfigureAwait(false);

Console.WriteLine(string.Join(", ", results));      // A, B, A, C, A
Console.WriteLine($"factoryCalls={factoryCalls}");  // 3 (unique keys: a, b, c)

// Each ValueTask is awaited exactly once — that is the contract.
static async Task ConsumeAsync(AsyncValueCache<string, string> cache, string key)
{
    ValueTask<string> vt = cache.GetAsync(key);      // not awaited yet — safe to hold.
    string value = await vt.ConfigureAwait(false);   // the single await.
    Console.WriteLine($"{key} -> {value}");
}

/*
// MISTAKE: never do this.
//
// static async Task BrokenConsumeAsync(AsyncValueCache<string,string> cache, string key)
// {
//     ValueTask<string> vt = cache.GetAsync(key);
//     string first  = await vt.ConfigureAwait(false); // first await — ok.
//     string second = await vt.ConfigureAwait(false); // SECOND await — UNDEFINED!
//     // With an IValueTaskSource backing, the second await may throw
//     // InvalidOperationException: "Operations should not be called on a
//     // ValueTask after it has completed."
// }
//
// Correct: store the RESULT (string), not the ValueTask itself.
*/
```

Walk-through, line by line. The class `AsyncValueCache` is marked `sealed`, which lowers virtual-call overhead and forbids subclassing — reasonable for a self-contained cache. The constraint `TKey : notnull` is mandatory: `ConcurrentDictionary` requires a hashable, non-null key. The field `_cache` is `ConcurrentDictionary`, which gives thread-safe `TryGetValue` and `GetOrAdd` without a manual `lock`. The field `_locks` holds a `SemaphoreSlim` per key — the "queue dispatcher" for the slow path.

In `GetAsync`, the signature is `async ValueTask<TValue>`: the compiler builds an `AsyncValueTaskMethodBuilder` that, on a synchronous completion (the fast path), does not allocate a `Task` but packs the result directly into the `ValueTask` struct. The first line, `if (_cache.TryGetValue(key, out var cached)) return cached;`, is precisely the allocation-free hit: we do not call `Task.FromResult`, we do not allocate a heap object. The line `var gate = _locks.GetOrAdd(key, _ => new SemaphoreSlim(1, 1));` guarantees that for a single key every thread gets the same semaphore, even if the delegate runs several times — `GetOrAdd` returns the already-existing instance. `await gate.WaitAsync(ct).ConfigureAwait(false);` is an asynchronous wait without thread blocking and without context capture; this prevents self-deadlocks and UI stalls. The re-check `if (_cache.TryGetValue(...)) return cached;` inside `try` is the async analogue of double-checked locking: while we were queuing for the semaphore, someone may have already finished the load, so we should not call the factory again. `await _loadAsync(...).ConfigureAwait(false)` is the only "real" asynchronous call; this is where I/O can happen. `_cache.GetOrAdd(key, value)` atomically stores or returns the already-existing value (if someone beat us to it). `gate.Release()` in `finally` guarantees release even if the factory throws.

The class `BoundedTemperatureReader` deliberately does **not** use the `async` modifier: `ReadAsync` contains no `await`, so omitting `async` lets it return `ValueTask<double>` through `new ValueTask<double>(snapshot)` directly, with no state machine and no `Task`. The `lock (_gate)` without `await` inside is safe. In the demo, `int factoryCalls` is incremented via `Interlocked.Increment` — thread-safe. The collection expression `["a","b","a","c","a"]` is C# 12 syntax. Each `ValueTask` is awaited exactly once in `ConsumeAsync` — the contract is honoured. The commented-out `BrokenConsumeAsync` block shows the violation: a second `await` of the same instance is undefined and may throw `InvalidOperationException` when the backing source is an `IValueTaskSource`. Applied lesson concepts: value type `ValueTask<T>` versus reference `Task<T>`, caching the ready value (not the `ValueTask`), `ConfigureAwait(false)`, `SemaphoreSlim` against stampedes, the ban on `lock`+`await` and `.Result`, and the one-await-per-instance contract.

#### Going deeper (bonus)

1. **Measure allocations.** Add `BenchmarkDotNet` and compare `GetAsync` on `ValueTask<T>` with a `Task<T>` analogue (`Task.FromResult`) at 100 % cache hits. You should see a noticeably smaller `Allocated` for the `ValueTask` version. Record the result in `BENCHMARKS.md`.
2. **TTL and invalidation.** Add entry time-to-live: store `record struct Entry(TValue Value, DateTime ExpiresAt)` and return a `ValueTask` only when the entry is still fresh. Think about how to invalidate the semaphore and the entry atomically.
3. **A manual `IValueTaskSource<T>`.** Implement a minimal poolable source (even a toy one) and make `ValueTask<T>` use it instead of a `Task`. Make sure you understand `GetStatus`/`GetResult`/`OnCompleted` and `ValueTaskSourceStatus`. This is a hard task — reflect the difficulties in a comment.
4. **Parallel warm-up.** Add a test that, from 50 parallel threads, requests the same key for the first time and verifies the factory was invoked exactly once. Use `Parallel.For` or `Task.WhenAll` with an array of tasks.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `ValueCacheLab` собирается под `net8.0` без предупреждений (`dotnet build -warnaserror`).
- [ ] (RU) Реализован `AsyncValueCache<TKey, TValue>` с `ValueTask<TValue> GetAsync`.
- [ ] (RU) Быстрый путь возвращает готовое значение без аллокации `Task`.
- [ ] (RU) Медленный путь использует `SemaphoreSlim` и двойную проверку кэша.
- [ ] (RU) Везде стоит `ConfigureAwait(false)` в библиотечном коде.
- [ ] (RU) Нет `.Result`/`.Wait()`, нет `lock` вокруг `await`.
- [ ] (RU) `BoundedTemperatureReader.ReadAsync` возвращает `ValueTask<double>` без `Task`.
- [ ] (RU) `dotnet run` выводит `A, B, A, C, A` и `factoryCalls=3`.
- [ ] (RU) Есть закомментированная демонстрация нарушения контракта «один await».
- [ ] (EN) The `ValueCacheLab` project builds under `net8.0` with no warnings (`dotnet build -warnaserror`).
- [ ] (EN) `AsyncValueCache<TKey, TValue>` with `ValueTask<TValue> GetAsync` is implemented.
- [ ] (EN) The fast path returns the ready value with no `Task` allocation.
- [ ] (EN) The slow path uses `SemaphoreSlim` and a double-check of the cache.
- [ ] (EN) `ConfigureAwait(false)` is applied throughout the library code.
- [ ] (EN) No `.Result`/`.Wait()`, no `lock` around `await`.
- [ ] (EN) `BoundedTemperatureReader.ReadAsync` returns `ValueTask<double>` with no `Task`.
- [ ] (EN) `dotnet run` prints `A, B, A, C, A` and `factoryCalls=3`.
- [ ] (EN) A commented-out demo of the one-await-contract violation is present.

#### Ресурсы / Resources
- [Microsoft Learn — ValueTask&lt;TResult&gt;](https://learn.microsoft.com/dotnet/api/system.threading.tasks.valuetask-1)
- [Microsoft Learn — IValueTaskSource&lt;TResult&gt;](https://learn.microsoft.com/dotnet/api/system.threading.tasks.sources.ivaluetasksource-1)
- [Microsoft Learn — SemaphoreSlim](https://learn.microsoft.com/dotnet/api/system.threading.semaphoreslim)
- [Microsoft Learn — ConcurrentDictionary&lt;TKey,TValue&gt;](https://learn.microsoft.com/dotnet/api/system.collections.concurrent.concurrentdictionary-2)
- [.NET GitHub — ValueTask source](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Private.CoreLib/src/System/Threading/Tasks/ValueTask.cs)

---

[⬆ К модулю M09](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
