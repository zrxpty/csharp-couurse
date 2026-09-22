---
[← К уроку M11-L05](lesson-M11-L05-volatile-memory-model.md) | [⬆ К модулю M11](../README.md) | [Следующее ДЗ →](homework-M11-L06-concurrent-collections.md)
---

### Домашнее задание M11-L05: volatile, барьеры памяти, .NET memory model / Homework M11-L05: volatile, memory barriers, .NET memory model

**Урок / Lesson:** M11-L05
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться применять `volatile`, `Volatile.Read/Write` и `Interlocked` строго по модели памяти .NET: строить корректные stop-флаги spin-wait, публиковать immutable-объекты без блокировок и понимать, почему `volatile` не заменяет `lock` для составных операций. (EN) Learn to apply `volatile`, `Volatile.Read/Write`, and `Interlocked` strictly according to the .NET memory model: build correct spin-wait stop flags, publish immutable objects lock-free, and understand why `volatile` does not replace `lock` for composite operations.

#### Связь с уроком / Connection to the lesson
(RU) Это домашнее задание напрямую закрепляет три ключевых паттерна урока: volatile stop-flag для spin-wait воркера, lock-free публикацию immutable-снимка через `Volatile.Write`/`Volatile.Read` и демонстрацию того, что `volatile` не делает `++` атомарным. Вы также столкнётесь с платформенными различиями x86/x64 и ARM64 и с тем, когда полный забор `Thread.MemoryBarrier` избыточен.
(EN) This homework directly reinforces the three core patterns from the lesson: a volatile stop-flag for a spin-wait worker, lock-free publication of an immutable snapshot via `Volatile.Write`/`Volatile.Read`, and the demonstration that `volatile` does not make `++` atomic. You will also confront the platform differences between x86/x64 and ARM64 and the cases where a full `Thread.MemoryBarrier` fence is overkill.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Представьте, что вы пишете низкоуровневый компонент для телеметрии в высоконагруженном сетевом демоне. Компонент содержит горячий воркер, который крутится в spin-wait цикле и опрашивает флаг остановки, лениво инициализируемый immutable-конфигурационный снимок, который время от времени обновляется из другого потока, а также счётчик обработанных сообщений, который инкрементируется из десятков потоков одновременно. На вашей dev-машине (x64, Intel) всё работает идеально: воркер останавливается за миллисекунды, конфигурация публикуется без артефактов, счётчик показывает ожидаемое число. Но как только компонент разворачивают на ARM64-серверах (или даже на Apple Silicon под эмуляцией), начинается кошмар: воркер иногда вообще не останавливается, читатели конфигурации изредка видят `null` там, где уже была публикация, а счётчик стабильно теряет сотни тысяч инкрементов из миллиона.

Причина — в модели памяти .NET, опирающейся на ECMA-335. Без явных барьеров платформа даёт лишь минимальные гарантии видимости и упорядочения. На x86/x64 переупорядочения случаются редко, потому что это сильная модель с тотальным упорядочением записей, поэтому ошибки маскируются. На ARM/ARM64 — слабая модель: процессор агрессивно переупорядочивает нагрузки и хранилища, буфер записи сбрасывается непредсказуемо, и любой код, полагающийся на «интуитивную» память, ломается. Ключевые инструменты упорядочения — ключевое слово `volatile` (семантика acquire/release для одного поля), методы `Volatile.Read`/`Volatile.Write` (явные половинные заборы), `Interlocked`-операции (полный забор плюс атомарность) и `Thread.MemoryBarrier` (полный забор, дорогой и грубый).

Цель задания — построить компонент, корректный на всех платформах, и через серию экспериментов прочувствовать на собственной шкуре, где `volatile` достаточен, где необходим `Interlocked`, а где без `lock` не обойтись. Вы не просто напишете рабочий код — вы докажете, почему он работает, опираясь на acquire/release семантику, и соберёте бенчмарк, который отделяет оправданную микрооптимизацию от преждевременной.

#### Что нужно сделать (пошагово)

1. Создайте новый консольный проект .NET 8 с именем `VolatileLab`. Выполните в пустой папке: `dotnet new console -n VolatileLab -f net8.0`, затем `cd VolatileLab`. Убедитесь, что в `VolatileLab.csproj` стоит `<TargetFramework>net8.0</TargetFramework>` и `<LangVersion>latest</LangVersion>` (или `12`).

2. Добавьте библиотеку бенчмаркинга: `dotnet add package BenchmarkDotNet`. Она понадобится, чтобы честно сравнить `volatile`, `Interlocked` и `lock` в горячем пути. Убедитесь, что пакет восстановился без ошибок (`dotnet restore`).

3. Реализуйте класс `SpinWorker` с полем `private volatile bool _stop;`. Воркер должен крутиться в цикле `while (!_stop) { /* work */ }` в фоновом потоке. Метод `Stop()` выставляет `_stop = true`. Метод `Dispose()` вызывает `Stop()` и делает `_worker.Join(TimeSpan.FromSeconds(2))`, бросая `TimeoutException`, если воркер не остановился. Это паттерн 1 из урока — volatile stop-flag.

4. Реализуйте класс `ConfigStore` с вложенным `record Config(int MaxConnections, string Endpoint, IReadOnlyList<string> Tags)`. Поле `private Config? _current;` НЕ помечайте `volatile` — вместо этого используйте `Volatile.Write(ref _current, cfg)` в `Publish` и `Volatile.Read(ref _current)` в `GetSnapshot`. Это паттерн 2 — явные половинные заборы для публикации immutable-объекта.

5. Реализуйте класс `TelemetryCounter` с двумя методами: `IncrementVolatile()` использует `volatile int _count;` и `_count++;` (намеренно сломанный), а `IncrementAtomic()` использует `Interlocked.Increment(ref _count)`. Метод `Value` возвращает `Volatile.Read(ref _count)`. Это паттерн 3 — демонстрация того, что `volatile` не делает `++` атомарным.

6. В `Program.Main` проведите три эксперимента. Эксперимент A: запустите `SpinWorker`, подождите 100 мс, вызовите `Dispose()` и убедитесь, что исключения нет — воркер остановился. Эксперимент B: создайте `ConfigStore`, опубликуйте конфигурацию, запустите 4 читающих потока, которые 10 000 раз вызывают `GetSnapshot()` и проверяют, что `Tags` не null и `MaxConnections > 0`; ни одно исключение не должно всплыть. Эксперимент C: запустите 16 задач, каждая делает 50 000 инкрементов через `IncrementVolatile()`, дождитесь всех и выведите результат — он будет меньше 800 000. Затем то же самое через `IncrementAtomic()` — результат ровно 800 000.

7. Напишите класс `CounterBenchmarks` с тремя методами, помеченными `[Benchmark]`: `VolatileIncrement()` (сломанный volatile `++`), `InterlockedIncrement()` (`Interlocked.Increment`), `LockedIncrement()` (`lock` вокруг `++`). Каждый метод инкрементирует в цикле 1000 раз. Запустите бенчмарк через `BenchmarkRunner.Run<CounterBenchmarks>()` в отдельном конфигурационном методе, вызываемом по флагу командной строки `--bench`.

8. Запустите `dotnet run` и зафиксируйте выводы трёх экспериментов. Затем `dotnet run -- --bench` и зафиксируйте таблицу BenchmarkDotNet: среднее время, allocations, разброс. Сравните три стратегии инкремента.

9. Опционально, если у вас есть доступ к ARM64-машине (Apple Silicon, Raspberry Pi 4/5, ARM-сервер) или WSL2 с эмуляцией, повторите эксперимент C и бенчмарк на ARM. Зафиксируйте, что разрыв между `volatile` (теряет инкременты) и `Interlocked` (не теряет) на ARM такой же, но абсолютные числа другие. Если ARM недоступен — просто задокументируйте теоретическое ожидание.

10. Соберите проект в Release: `dotnet build -c Release`. Убедитесь, что нет предупреждений, связанных с `volatile` (например, CA warning на `volatile` ссылочного типа). Если есть — объясните в комментарии, почему в вашем коде `volatile` применяется только к `int` и `bool`.

11. В файле `NOTES.md` (на русском, 200–400 слов) опишите: какой эксперимент что продемонстрировал, какие концепции урока были применены, и где в реальном продакшене вы бы выбрали `lock` вместо `volatile`.

#### Требования к решению

Решение должно компилироваться без ошибок и предупреждений в `dotnet build -c Release` под .NET 8 и C# 12. Код должен быть переносимым между x64 и ARM64 без изменений — никаких `#if` и платформенно-специфичных вызовов. Использование `volatile` допускается только для полей примитивных типов (`bool`, `int`) в неблокирующих hot path; для ссылочных типов используйте `Volatile.Read`/`Volatile.Write` явно. Для счётчиков и любой составной операции (чтение-модификация-запись) обязателен `Interlocked` или `lock` — `volatile` запрещён.

Класс `SpinWorker` должен корректно реализовывать `IDisposable` и гарантированно останавливаться за 2 секунды на любой платформе. Класс `ConfigStore` должен гарантировать, что после успешного `Publish` любой читатель, вызвавший `GetSnapshot` позже по времени, увидит полностью построенный immutable-объект со всеми полями (это требует release-store со стороны писателя и acquire-load со стороны читателя). Класс `TelemetryCounter` должен наглядно демонстрировать потерю инкрементов при `volatile` и отсутствие потерь при `Interlocked`.

Бенчмарк через BenchmarkDotNet должен честно сравнивать три стратегии на одинаковом объёме работы, с прогревом и множеством итераций. В `NOTES.md` должно быть явное обоснование выбора стратегии для каждого из трёх паттернов со ссылкой на acquire/release семантику и модель памяти .NET. Запрещено использовать `Thread.MemoryBarrier()` в прикладном коде — только если вы явно документируете в комментарии, почему половинного забора недостаточно (в этом задании такого случая нет).

#### Тонкости и подводные камни

Главная ловушка — уверенность, что `volatile` делает операцию атомарной. `volatile int _count; _count++;` состоит из трёх отдельных операций (load, add, store); `volatile` упорядочивает каждую по отдельности относительно соседних доступов к памяти, но не делает тройку неделимой. Два потока могут одновременно прочитать одно значение, прибавить единицу и записать обратно — один инкремент потерян. Только `Interlocked.Increment` даёт атомарную операцию чтение-модификация-запись с полным забором. Это прямое следствие модели памяти .NET 2.0 / ECMA-335: `volatile` даёт acquire для чтения и release для записи, но не атомарность составных операций.

Вторая тонкость — разница между ключевым словом `volatile` и методами `Volatile.Read`/`Volatile.Write`. Ключевое слово применяет acquire/release ко всем доступам к полю автоматически, но работает только с примитивными типами и непроверенными ссылочными типами; на ARM оно иногда ведёт себя тоньше, чем ожидаешь. Методы `Volatile.*` явнее, переносимее и работают с любыми типами, включая ссылочные. Для публикации immutable-объекта предпочтительны именно `Volatile.Write` (release-store) на стороне писателя и `Volatile.Read` (acquire-load) на стороне читателя — это делает намерение читаемым и не зависит от того, как JIT трактует ключевое слово.

Третья тонкость — платформенная разница. На x86/x64 записи упорядочены тотально (strong model), поэтому код, забывший `volatile` на stop-флаге, «работает» — но это случайность. На ARM64 (weak model) JIT кэширует `_stop` в регистре, и воркер никогда не видит `true`, зацикливаясь навсегда. Тестируйте на ARM, если заявляете lock-free корректность. Четвёртая тонкость — `Thread.MemoryBarrier()` это полный забор: он блокирует переупорядочение в обе стороны и сбрасывает буфер записи. Он дорогой (десятки наносекунд) и легко используется неправильно; в 99% случаев достаточно половинного забора через `Volatile.*` или `Interlocked`. Пятая тонкость — `volatile` не упорядочивает доступы между двумя разными volatile-полями гарантированно во всех моделях; для составных инвариантов используйте один флаг или `lock`. Шестая — `volatile` на мутируемом ссылочном поле бесполезен для защиты внутреннего состояния объекта: `volatile` упорядочивает только публикацию ссылки, но не доступы к полям объекта. Используйте immutable-дизайн или `lock`.

#### Критерии приёмки

- [ ] Проект `VolatileLab` создаётся командой `dotnet new console -f net8.0` и собирается в Release без ошибок и предупреждений.
- [ ] В `csproj` указаны `<TargetFramework>net8.0</TargetFramework>` и `<LangVersion>latest</LangVersion>`.
- [ ] Пакет `BenchmarkDotNet` добавлен и восстанавливается без ошибок.
- [ ] Класс `SpinWorker` использует `volatile bool _stop` и корректно реализует `IDisposable` с bounded `Join`.
- [ ] Класс `ConfigStore` использует `Volatile.Write`/`Volatile.Read` (а не ключевое слово `volatile`) для публикации immutable `Config`.
- [ ] Класс `TelemetryCounter` содержит и сломанный volatile-путь, и атомарный `Interlocked`-путь.
- [ ] Эксперимент A (spin worker) останавливается без `TimeoutException`.
- [ ] Эксперимент B (config readers) не выбрасывает `NullReferenceException` и видит согласованную конфигурацию.
- [ ] Эксперимент C показывает потерю инкрементов через volatile и ровно 800 000 через `Interlocked`.
- [ ] Бенчмарк BenchmarkDotNet запускается по флагу `--bench` и выводит таблицу с тремя стратегиями.
- [ ] Код не использует `Thread.MemoryBarrier()` в прикладной логике.
- [ ] Код переносим между x64 и ARM64 без платформенных `#if`.
- [ ] В `NOTES.md` дано обоснование выбора стратегии для каждого паттерна со ссылкой на acquire/release.
- [ ] Все комментарии в коде двуязычные (RU+EN), как в уроке.
- [ ] `dotnet build -c Release` завершается зелёным.

#### Подсказки (без прямого ответа)

- Вспомните, что `volatile` гарантирует для одного поля (acquire на чтение, release на запись) и чего НЕ гарантирует (атомарность составной операции).
- `++` на `int` — это три инструкции IL: `ldarg`, `ldfld`, `add`, `stfld`. Где между ними может вклиниться другой поток?
- Для публикации объекта важно, чтобы запись ссылочного поля шла ПОСЛЕ записи всех полей объекта. Какой забор это гарантирует со стороны писателя? А со стороны читателя?
- Подумайте, почему `Volatile.Read`/`Volatile.Write` предпочтительнее ключевого слова `volatile` для ссылочных типов: явность, переносимость, читаемость намерения.
- В бенчмарке убедитесь, что JIT не «вырезает» инкремент как dead code: возвращайте итоговое значение или используйте `Volatile.Write` в конце.
- ARM64 эмуляция в Docker на x64 не даст настоящей слабой модели — нужны реальные ARM-железо или Apple Silicon.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — VolatileLab: stop-flag, lazy publish, atomic counter.
// Эталонное решение домашнего задания M11-L05.
// Reference solution for homework M11-L05.

using System.Threading;

// ──────────────────────────────────────────────────────────────
// Pattern 1: volatile stop flag for a spin-wait worker.
// Паттерн 1: volatile-флаг остановки spin-wait воркера.
// ──────────────────────────────────────────────────────────────
public sealed class SpinWorker : IDisposable
{
    // volatile оправдан: одно поле bool, hot path spin-wait.
    // volatile is justified: single bool field, hot spin-wait path.
    private volatile bool _stop;
    private readonly Thread _worker;
    private long _ticks; // смотрим, что воркер реально работает / proves work happens

    public SpinWorker()
    {
        _worker = new Thread(Work) { IsBackground = true, Name = "SpinWorker" };
        _worker.Start();
    }

    private void Work()
    {
        // acquire-load: каждое чтение _stop идёт в память, не кэшируется в регистре.
        // acquire-load: each read of _stop goes to memory, not cached in a register.
        while (!_stop)
        {
            Interlocked.Increment(ref _ticks); // метрика работы / work metric
        }
    }

    public long Ticks => Interlocked.Read(ref _ticks);

    public void Stop()
    {
        // release-store: предшествующие записи воркера видны до _stop=true.
        // release-store: the worker's preceding writes are visible before _stop=true.
        _stop = true;
    }

    public void Dispose()
    {
        Stop();
        // bounded Join — не блокируем пул потоков навсегда.
        // bounded Join — never blocks a thread pool thread indefinitely.
        if (!_worker.Join(TimeSpan.FromSeconds(2)))
            throw new TimeoutException("SpinWorker did not stop within 2s.");
    }
}

// ──────────────────────────────────────────────────────────────
// Pattern 2: lock-free publish of an immutable config snapshot.
// Паттерн 2: lock-free публикация immutable-снимка конфигурации.
// ──────────────────────────────────────────────────────────────
public sealed class ConfigStore
{
    // immutable после построения — гарантия безопасной публикации.
    // immutable after construction — guarantees safe publication.
    public sealed record Config(
        int MaxConnections,
        string Endpoint,
        IReadOnlyList<string> Tags);

    private Config? _current; // НЕ volatile — используем явные Volatile.* / NOT volatile — explicit Volatile.*

    public void Publish(Config cfg)
    {
        ArgumentNullException.ThrowIfNull(cfg);
        // release-store: поля cfg уже записаны; публикация ссылки идёт последней.
        // release-store: cfg's fields are written; the reference publish goes last.
        Volatile.Write(ref _current, cfg);
    }

    public Config? GetSnapshot()
    {
        // acquire-load: чтение _current не всплывает выше последующих чтений полей cfg.
        // acquire-load: reading _current does not float above later reads of cfg's fields.
        return Volatile.Read(ref _current);
    }
}

// ──────────────────────────────────────────────────────────────
// Pattern 3: volatile does NOT fix a racey counter; Interlocked does.
// Паттерн 3: volatile НЕ чинит счётчик с race; Interlocked чинит.
// ──────────────────────────────────────────────────────────────
public sealed class TelemetryCounter
{
    private volatile int _count; // volatile здесь — ловушка для ++ / volatile here is a ++ trap

    public void IncrementVolatile() => _count++; // race: load+add+store / race condition

    // Атомарный инкремент: полный забор + неделимость.
    // Atomic increment: full fence + indivisibility.
    public void IncrementAtomic() => Interlocked.Increment(ref _count);

    public int Value => Volatile.Read(ref _count);
}

// ──────────────────────────────────────────────────────────────
// Demo entry: три эксперимента урока.
// Demo entry: three lesson experiments.
// ──────────────────────────────────────────────────────────────
public static class VolatileLabDemo
{
    public static void Run()
    {
        // Эксперимент A: spin worker останавливается.
        // Experiment A: spin worker stops.
        using var worker = new SpinWorker();
        Thread.Sleep(100);
        worker.Dispose(); // без исключения / no exception
        Console.WriteLine($"A: ticks={worker.Ticks}, stopped cleanly.");

        // Эксперимент B: читатели видят согласованную конфигурацию.
        // Experiment B: readers observe a consistent config.
        var store = new ConfigStore();
        store.Publish(new ConfigStore.Config(
            MaxConnections: 100,
            Endpoint: "https://telemetry.local",
            Tags: new[] { "prod", "eu" }));
        var readers = new Task[4];
        for (int i = 0; i < readers.Length; i++)
        {
            readers[i] = Task.Run(() =>
            {
                for (int j = 0; j < 10_000; j++)
                {
                    var snap = store.GetSnapshot();
                    if (snap is null) continue;
                    if (snap.MaxConnections <= 0 || snap.Tags.Count == 0)
                        throw new InvalidOperationException("Inconsistent snapshot!");
                }
            });
        }
        Task.WaitAll(readers);
        Console.WriteLine("B: all readers saw consistent config.");

        // Эксперимент C: volatile теряет инкременты, Interlocked — нет.
        // Experiment C: volatile loses increments, Interlocked does not.
        var broken = new TelemetryCounter();
        var tasks = new Task[16];
        for (int i = 0; i < tasks.Length; i++)
            tasks[i] = Task.Run(() =>
            {
                for (int j = 0; j < 50_000; j++) broken.IncrementVolatile();
            });
        Task.WaitAll(tasks);
        Console.WriteLine($"C volatile (expect 800000): {broken.Value}");

        var safe = new TelemetryCounter();
        for (int i = 0; i < tasks.Length; i++)
            tasks[i] = Task.Run(() =>
            {
                for (int j = 0; j < 50_000; j++) safe.IncrementAtomic();
            });
        Task.WaitAll(tasks);
        Console.WriteLine($"C interlocked (expect 800000): {safe.Value}");
    }
}
```

Разбор по строкам. Поле `private volatile bool _stop` в `SpinWorker` — классический acquire/release для одного поля: JIT не может кэшировать его в регистре между итерациями цикла, поэтому каждое чтение действительно идёт в память и видит свежую запись от `Stop()`. Это работает на ARM64 так же, как на x64, потому что `volatile` даёт платформенно-независимые гарантии ECMA-335, а не полагается на сильную модель x86. Метод `Stop()` выполняет release-store: хотя ключевое слово `volatile` уже даёт release для записи, здесь это особенно важно — все предшествующие записи воркера (например, `_ticks`) станут видимы другим потокам до того, как `_stop=true` будет наблюдаем. Bounded `Join(2s)` защищает от зависания на ARM, если что-то пошло не так.

В `ConfigStore` поле `_current` намеренно НЕ помечено `volatile` — вместо этого используются явные `Volatile.Write` (release-store) и `Volatile.Read` (acquire-load). Это переносимее и читаемее для ссылочных типов. Release-store гарантирует: записи полей `Config` (выполненные при создании `record`) видны читателю до того, как он увидит ссылку; acquire-load гарантирует: читатель не увидит ссылку раньше, чем прочитает поля объекта. Поскольку `Config` immutable после построения, эта пара заборов даёт безопасную публикацию без `lock`. Если бы `Config` был мутируемым, этот приём был бы некорректен — нужен `lock` или immutable-дизайн.

В `TelemetryCounter` метод `IncrementVolatile` наглядно демонстрирует ловушку: `volatile` упорядочивает каждое отдельное чтение и запись, но `++` это load+add+store, и между ними влезает другой поток — инкременты теряются. Метод `IncrementAtomic` через `Interlocked.Increment` даёт атомарную операцию с полным забором — потери невозможны. Свойство `Value` использует `Volatile.Read`, чтобы читатель увидел самое свежее значение без переупорядочения с последующими доступами. Эксперимент C показывает разрыв: `volatile` даёт ~790 000–799 000 из 800 000, `Interlocked` даёт ровно 800 000.

#### Задания на углубление (бонус)

1. Добавьте четвёртый паттерн: «двойная публикация» двух связанных полей `_config` и `_version` без `lock`. Покажите, почему `volatile` на обоих полях НЕ гарантирует согласованное наблюдение пары, и перепишите на одно immutable-поле `ConfigWithVersion` или на `lock`.
2. Реализуйте ленивую инициализацию `Lazy<T>`-подобного класса вручную через `Volatile.Read`/`Volatile.Write` + `Interlocked.CompareExchange`, без использования `Lazy<T>` и без `lock`. Докажите корректность через acquire/release.
3. Соберите бенчмарк «stop-flag spin-wait» на 10 секунд: сравните `volatile bool`, `ManualResetEventSlim.Wait()`, `SpinWait.SpinUntil()`. Измерьте CPU usage и latency.
4. На ARM64 (если доступен) проведите эксперимент с `Thread.MemoryBarrier()` вместо `volatile` в `SpinWorker` и измерьте, насколько дороже полный забор в горячем цикле.

---

## Statement in English / Постановка на английском

#### Context & motivation

Imagine you are writing a low-level telemetry component for a high-throughput network daemon. The component contains a hot worker spinning in a tight loop polling a stop flag, a lazily initialised immutable configuration snapshot that is occasionally refreshed from another thread, and a counter of processed messages incremented from dozens of concurrent threads. On your dev machine (x64, Intel) everything works flawlessly: the worker stops within milliseconds, the configuration is published without artefacts, and the counter shows the expected number. But the moment the component is deployed to ARM64 servers (or even Apple Silicon under emulation), a nightmare begins: the worker sometimes never stops at all, readers of the configuration occasionally observe `null` where a publish already happened, and the counter consistently loses hundreds of thousands of increments out of a million.

The reason lies in the .NET memory model, grounded in ECMA-335. Without explicit barriers the platform gives only minimal visibility and ordering guarantees. On x86/x64 reorderings are rare because it is a strong model with total store ordering, so bugs are masked. On ARM/ARM64 the model is weak: the CPU aggressively reorders loads and stores, the store buffer is flushed unpredictably, and any code relying on "intuitive" memory breaks. The key ordering tools are the `volatile` keyword (acquire/release semantics for a single field), the `Volatile.Read`/`Volatile.Write` methods (explicit half-fences), `Interlocked` operations (full fence plus atomicity), and `Thread.MemoryBarrier` (a full fence, expensive and blunt).

The goal of this assignment is to build a component that is correct on every platform, and through a series of experiments to feel in your own hands where `volatile` is sufficient, where `Interlocked` is necessary, and where `lock` is the only honest answer. You will not merely write working code — you will prove why it works by appealing to acquire/release semantics, and you will gather a benchmark that separates justified micro-optimisation from premature optimisation. By the end you should be able to articulate, for any given field, exactly which ordering primitive is correct and why, without hand-waving about "the CPU probably does the right thing."

#### What to do, step by step

1. Create a fresh .NET 8 console project named `VolatileLab`. In an empty folder run `dotnet new console -n VolatileLab -f net8.0`, then `cd VolatileLab`. Confirm that `VolatileLab.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and `<LangVersion>latest</LangVersion>` (or `12`).

2. Add the benchmarking library: `dotnet add package BenchmarkDotNet`. You will need it to honestly compare `volatile`, `Interlocked`, and `lock` on the hot path. Confirm the package restores cleanly with `dotnet restore`.

3. Implement a `SpinWorker` class with a field `private volatile bool _stop;`. The worker should spin in a `while (!_stop) { /* work */ }` loop on a background thread. The `Stop()` method sets `_stop = true`. The `Dispose()` method calls `Stop()` and then `_worker.Join(TimeSpan.FromSeconds(2))`, throwing `TimeoutException` if the worker did not stop. This is pattern 1 from the lesson — the volatile stop-flag.

4. Implement a `ConfigStore` class with a nested `record Config(int MaxConnections, string Endpoint, IReadOnlyList<string> Tags)`. Do NOT mark the field `private Config? _current;` as `volatile` — instead use `Volatile.Write(ref _current, cfg)` in `Publish` and `Volatile.Read(ref _current)` in `GetSnapshot`. This is pattern 2 — explicit half-fences for publishing an immutable object.

5. Implement a `TelemetryCounter` class with two methods: `IncrementVolatile()` uses `volatile int _count;` and `_count++;` (intentionally broken), while `IncrementAtomic()` uses `Interlocked.Increment(ref _count)`. The `Value` property returns `Volatile.Read(ref _count)`. This is pattern 3 — the demonstration that `volatile` does not make `++` atomic.

6. In `Program.Main`, run three experiments. Experiment A: start a `SpinWorker`, sleep 100 ms, call `Dispose()`, and confirm no exception — the worker stopped. Experiment B: create a `ConfigStore`, publish a config, launch 4 reader threads that call `GetSnapshot()` 10 000 times each and assert that `Tags` is non-null and `MaxConnections > 0`; no exception should surface. Experiment C: launch 16 tasks each doing 50 000 increments via `IncrementVolatile()`, await all, and print the result — it will be less than 800 000. Then repeat via `IncrementAtomic()` — the result is exactly 800 000.

7. Write a `CounterBenchmarks` class with three methods decorated with `[Benchmark]`: `VolatileIncrement()` (the broken volatile `++`), `InterlockedIncrement()` (`Interlocked.Increment`), `LockedIncrement()` (`lock` around `++`). Each method increments 1000 times in a loop. Run the benchmark via `BenchmarkRunner.Run<CounterBenchmarks>()` from a separate configuration method triggered by the `--bench` command-line flag.

8. Run `dotnet run` and capture the output of the three experiments. Then run `dotnet run -- --bench` and capture the BenchmarkDotNet table: mean time, allocations, variance. Compare the three increment strategies.

9. Optionally, if you have access to an ARM64 machine (Apple Silicon, Raspberry Pi 4/5, an ARM server) or WSL2 with emulation, repeat experiment C and the benchmark on ARM. Record that the gap between `volatile` (losing increments) and `Interlocked` (no losses) is the same on ARM, but the absolute numbers differ. If ARM is unavailable, document the theoretical expectation instead.

10. Build the project in Release: `dotnet build -c Release`. Confirm there are no warnings related to `volatile` (for example, a CA warning on a `volatile` reference-type field). If any appear, explain in a comment why your code applies `volatile` only to `int` and `bool`.

11. In a `NOTES.md` file (200–400 words) describe: what each experiment demonstrated, which lesson concepts were applied, and where in real production you would choose `lock` over `volatile`.

#### Requirements

The solution must compile without errors or warnings under `dotnet build -c Release` on .NET 8 and C# 12. The code must be portable between x64 and ARM64 without changes — no `#if` and no platform-specific calls. The use of `volatile` is permitted only for primitive-type fields (`bool`, `int`) in non-blocking hot paths; for reference types you must use `Volatile.Read`/`Volatile.Write` explicitly. For counters and any composite read-modify-write operation, `Interlocked` or `lock` is mandatory — `volatile` is forbidden.

The `SpinWorker` class must correctly implement `IDisposable` and must reliably stop within 2 seconds on any platform. The `ConfigStore` class must guarantee that after a successful `Publish`, any reader that calls `GetSnapshot` later in wall-clock time observes a fully constructed immutable object with all of its fields (this requires a release-store on the writer side and an acquire-load on the reader side). The `TelemetryCounter` class must visibly demonstrate lost increments under `volatile` and no losses under `Interlocked`.

The BenchmarkDotNet benchmark must honestly compare the three strategies on identical workloads, with warmup and many iterations. The `NOTES.md` file must explicitly justify the choice of strategy for each of the three patterns with a reference to acquire/release semantics and the .NET memory model. Using `Thread.MemoryBarrier()` in application code is forbidden — unless you explicitly document in a comment why a half-fence is insufficient (in this assignment, no such case exists).

#### Pitfalls

The main trap is the belief that `volatile` makes an operation atomic. `volatile int _count; _count++;` consists of three separate operations (load, add, store); `volatile` orders each one individually with respect to neighbouring memory accesses, but it does not make the triple indivisible. Two threads can simultaneously read the same value, add one, and write it back — one increment is lost. Only `Interlocked.Increment` gives an atomic read-modify-write with a full fence. This is a direct consequence of the .NET 2.0 / ECMA-335 memory model: `volatile` gives acquire on read and release on write, but not atomicity of composite operations.

The second subtlety is the difference between the `volatile` keyword and the `Volatile.Read`/`Volatile.Write` methods. The keyword applies acquire/release to every access to the field automatically, but it works only with primitive types and unchecked reference types; on ARM it sometimes behaves more subtly than you would expect. The `Volatile.*` methods are more explicit, more portable, and work with any type, including reference types. For publishing an immutable object, `Volatile.Write` (release-store) on the writer side and `Volatile.Read` (acquire-load) on the reader side are preferable — this makes the intent readable and does not depend on how the JIT interprets the keyword.

The third subtlety is the platform difference. On x86/x64 stores are totally ordered (a strong model), so code that forgot `volatile` on a stop flag "works" — but that is luck. On ARM64 (a weak model) the JIT caches `_stop` in a register and the worker never sees `true`, looping forever. Test on ARM if you claim lock-free correctness. The fourth subtlety is that `Thread.MemoryBarrier()` is a full fence: it blocks reordering in both directions and flushes the store buffer. It is expensive (tens of nanoseconds) and easy to misuse; in 99% of cases a half-fence via `Volatile.*` or `Interlocked` suffices. The fifth subtlety is that `volatile` does not reliably order accesses between two different volatile fields across all memory models; for composite invariants use a single flag or `lock`. The sixth subtlety is that `volatile` on a mutable reference-type field is useless for protecting the object's internal state: `volatile` only orders the publication of the reference, not accesses to the object's fields. Use immutable design or `lock`.

#### Acceptance criteria

- [ ] The `VolatileLab` project is created with `dotnet new console -f net8.0` and builds in Release with no errors or warnings.
- [ ] The `csproj` declares `<TargetFramework>net8.0</TargetFramework>` and `<LangVersion>latest</LangVersion>`.
- [ ] The `BenchmarkDotNet` package is added and restores without errors.
- [ ] The `SpinWorker` class uses `volatile bool _stop` and correctly implements `IDisposable` with a bounded `Join`.
- [ ] The `ConfigStore` class uses `Volatile.Write`/`Volatile.Read` (not the `volatile` keyword) to publish the immutable `Config`.
- [ ] The `TelemetryCounter` class contains both the broken volatile path and the atomic `Interlocked` path.
- [ ] Experiment A (spin worker) stops without a `TimeoutException`.
- [ ] Experiment B (config readers) does not throw a `NullReferenceException` and observes a consistent configuration.
- [ ] Experiment C shows lost increments under volatile and exactly 800 000 under `Interlocked`.
- [ ] The BenchmarkDotNet benchmark runs on the `--bench` flag and prints a table with three strategies.
- [ ] The code does not use `Thread.MemoryBarrier()` in application logic.
- [ ] The code is portable between x64 and ARM64 with no platform `#if`.
- [ ] `NOTES.md` justifies the strategy choice for each pattern with reference to acquire/release.
- [ ] All code comments are bilingual (EN+RU), mirroring the lesson.
- [ ] `dotnet build -c Release` is green.

#### Hints (no direct answer)

- Recall what `volatile` guarantees for a single field (acquire on read, release on write) and what it does NOT guarantee (atomicity of a composite operation).
- `++` on an `int` is three IL instructions: `ldarg`, `ldfld`, `add`, `stfld`. Where can another thread slip in between them?
- For publishing an object, the reference store must happen AFTER the writes to all the object's fields. Which fence guarantees that on the writer side? And on the reader side?
- Consider why `Volatile.Read`/`Volatile.Write` are preferable to the `volatile` keyword for reference types: explicitness, portability, readability of intent.
- In the benchmark, make sure the JIT does not eliminate the increment as dead code: return the final value or use `Volatile.Write` at the end.
- ARM64 emulation in Docker on x64 does not give a real weak model — you need real ARM hardware or Apple Silicon.

#### Reference solution walk-through

The reference solution (identical code as in the Russian block above) is built around the same three patterns. The `SpinWorker` class uses `private volatile bool _stop`: the keyword grants acquire/release on every access to that one field, so the JIT cannot cache it in a register between loop iterations and each read truly goes to memory, observing the fresh store from `Stop()`. This works identically on ARM64 and x64 because `volatile` gives platform-independent ECMA-335 guarantees rather than relying on the strong x86 model. `Stop()` performs a release-store: although the `volatile` keyword already gives release for writes, this is especially important here — all preceding writes by the worker (such as `_ticks`) become visible to other threads before `_stop=true` is observable. The bounded `Join(2s)` protects against a hang on ARM if something goes wrong.

In `ConfigStore` the field `_current` is deliberately NOT marked `volatile` — instead the code uses explicit `Volatile.Write` (release-store) and `Volatile.Read` (acquire-load). This is more portable and readable for reference types. The release-store guarantees that writes to the fields of `Config` (performed when the `record` is constructed) are visible to the reader before it sees the reference; the acquire-load guarantees that the reader does not see the reference before reading the object's fields. Because `Config` is immutable after construction, this pair of fences gives safe publication without a `lock`. If `Config` were mutable, this technique would be incorrect — you would need `lock` or an immutable design.

In `TelemetryCounter`, the `IncrementVolatile` method vividly demonstrates the trap: `volatile` orders each individual read and write, but `++` is load+add+store, and another thread can slip in between them — increments are lost. The `IncrementAtomic` method via `Interlocked.Increment` gives an atomic operation with a full fence — no losses are possible. The `Value` property uses `Volatile.Read` so that the reader observes the freshest value without reordering relative to subsequent accesses. Experiment C shows the gap: `volatile` yields around 790 000–799 000 out of 800 000, while `Interlocked` yields exactly 800 000.

The broader lesson is that `volatile` is a precision tool for one field in a non-blocking hot path, never a substitute for `lock` or `Interlocked` when you have a composite invariant or a read-modify-write. The acquire/release pair is exactly enough to publish an immutable object or to signal a stop flag, and exactly not enough to count events. When in doubt, prefer `Interlocked` for counters and `lock` for composite state; reach for `volatile` only when you have a benchmark proving the `lock` cost is unbearable and a correctness argument grounded in the memory model.

#### Going deeper (bonus)

1. Add a fourth pattern: "dual publication" of two related fields `_config` and `_version` without `lock`. Show why `volatile` on both fields does NOT guarantee a consistent observation of the pair, and rewrite using a single immutable field `ConfigWithVersion` or a `lock`.
2. Implement a manual `Lazy<T>`-like class via `Volatile.Read`/`Volatile.Write` + `Interlocked.CompareExchange`, without using `Lazy<T>` and without `lock`. Prove correctness via acquire/release.
3. Build a "stop-flag spin-wait" benchmark over 10 seconds: compare `volatile bool`, `ManualResetEventSlim.Wait()`, `SpinWait.SpinUntil()`. Measure CPU usage and latency.
4. On ARM64 (if available) run the experiment with `Thread.MemoryBarrier()` instead of `volatile` in `SpinWorker` and measure how much more expensive the full fence is in the hot loop.

---

#### Чек-лист сдачи / Submission checklist
- [ ] (RU) Проект `VolatileLab` собирается в Release без ошибок и предупреждений.
- [ ] (RU) Реализованы `SpinWorker`, `ConfigStore`, `TelemetryCounter` строго по модели памяти.
- [ ] (RU) Три эксперимента A/B/C выполняются и дают ожидаемые результаты.
- [ ] (RU) Бенчмарк BenchmarkDotNet запускается по флагу `--bench`.
- [ ] (RU) В `NOTES.md` дано обоснование выбора стратегии для каждого паттерна.
- [ ] (RU) Код не использует `Thread.MemoryBarrier()` в прикладной логике.
- [ ] (EN) The `VolatileLab` project builds in Release with no errors or warnings.
- [ ] (EN) `SpinWorker`, `ConfigStore`, `TelemetryCounter` are implemented strictly per the memory model.
- [ ] (EN) Experiments A/B/C run and produce the expected results.
- [ ] (EN) The BenchmarkDotNet benchmark runs on the `--bench` flag.
- [ ] (EN) `NOTES.md` justifies the strategy choice for each pattern.
- [ ] (EN) The code does not use `Thread.MemoryBarrier()` in application logic.

#### Ресурсы / Resources
- [Microsoft Learn — System.Threading.Volatile](https://learn.microsoft.com/dotnet/api/system.threading.volatile)
- [Microsoft Learn — Thread.MemoryBarrier](https://learn.microsoft.com/dotnet/api/system.threading.thread.memorybarrier)
- [Microsoft Learn — System.Threading.Interlocked](https://learn.microsoft.com/dotnet/api/system.threading.interlocked)
- [ECMA-335 — Common Language Infrastructure (CLI) standard](https://ecma-international.org/publications-and-standards/standards/ecma-335/)
- [.NET memory model — Joe Duffy "Concurrent Programming on Windows" notes](https://www.bluebytesoftware.com/blog/2009/03/16/ConcurrentProgrammingOnWindowsMemoryModelNotes.aspx)
- [BenchmarkDotNet documentation](https://benchmarkdotnet.org/)

---
[← К уроку M11-L05](lesson-M11-L05-volatile-memory-model.md) | [⬆ К модулю M11](../README.md) | [Следующее ДЗ →](homework-M11-L06-concurrent-collections.md)
