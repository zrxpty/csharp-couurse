---
[← К уроку M07-L03](lesson-M07-L03-throw-rethrow.md) | [⬆ К модулю M07](../README.md) | [Следующее ДЗ →](homework-M07-L04-custom-exceptions.md)
---

### Домашнее задание M07-L03: throw и throw; (rethrow) — разница стеков / Homework M07-L03: throw and throw; (rethrow) — stack difference

**Урок / Lesson:** M07-L03
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться различать три способа проброса исключения в C# 12 / .NET 8 — `throw;`, `throw ex;` и `throw new ...(..., ex)` — и осознанно выбирать каждый из них в зависимости от задачи: сохранять оригинальную трассировку стека, обёртывать низкоуровневую ошибку в доменную с сохранением цепочки через `innerException`, или переносить исключение через границу потоков с помощью `ExceptionDispatchInfo`. Закрепить применение фильтров исключений `when (...)` как «мягкого rethrow» для логирования без побочного эффекта на стек. (EN) Learn to distinguish the three ways to propagate an exception in C# 12 / .NET 8 — `throw;`, `throw ex;` and `throw new ...(..., ex)` — and choose each one deliberately depending on the task: preserve the original stack trace, wrap a low-level error into a domain exception while keeping the chain via `innerException`, or carry an exception across a thread boundary with `ExceptionDispatchInfo`. Reinforce the use of exception filters `when (...)` as a "soft rethrow" for logging without side effects on the stack.

#### Связь с уроком / Connection to the lesson
(RU) Урок M07-L03 объясняет, что трассировка стека — это «следы на снегу», по которым разработчик находит корень ошибки. Вы изучили три формы проброса (`throw;`, `throw ex;`, `throw new`), класс `ExceptionDispatchInfo` из `System.Runtime.ExceptionServices` и поведение фильтров `when (...)`. Это ДЗ заставит вас не просто повторить код из урока, а собрать мини-проект, в котором каждая из форм даёт наблюдаемо разный стек, и доказать это assertions-проверками в логах.
(EN) Lesson M07-L03 explains that a stack trace is like footprints in snow that let a developer find the root of an error. You studied the three rethrow forms (`throw;`, `throw ex;`, `throw new`), the `ExceptionDispatchInfo` class from `System.Runtime.ExceptionServices`, and the behavior of `when (...)` filters. This homework asks you not merely to copy the lesson's code but to build a mini-project in which each form produces an observably different stack, and to prove it with assertion checks in the logs.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы — инженер команды платёжного шлюза `PayFlow`. Сервис читает конфигурацию тарифов из трёх источников: локального JSON-файла на диске, кэша в памяти и удалённого HTTP-эндпоинта биллинга. Каждый источник может подвести по-своему: файл — `IOException` (битый сектор), кэш — `InvalidOperationException` (рассинхрон ключей), HTTP — `HttpRequestException` (5xx удалённого сервиса). Бизнес-контракт сервиса требует, чтобы наружу всегда поднималось доменное исключение `TariffLoadException`, но при этом диагностическая цепочка до исходной ошибки ни в коем случае не терялась: логи наблюдаемости (`OpenTelemetry`-следы) должны показывать ровно ту строку, где возник сбой, а не ту, где вы его «перепаковали».

Команда дважды обжигалась на типичном баге: разработчики писали `catch (Exception ex) { _logger.Warn(...); throw ex; }` и тем самым затирали оригинальную точку возникновения. В результате инциденты в продакшене расследовались сутками вместо минут. Ваш тимлид поручил вам реализовать эталонный загрузчик `TariffRepository`, который демонстрирует три осознанных стратегии проброса и который другие члены команды смогут брать за образец. Параллельно вы должны добавить «наблюдатель» на базе exception filters, который логирует каждый сбой ровно один раз и не влияет на поток управления, а также показать корректный перенос исключения из фоновой задачи `Task.Run` в вызывающий поток с сохранением стека через `ExceptionDispatchInfo`. Задание намеренно построено так, чтобы каждое решение было проверяемо текстовым assertion-ом на содержимом `StackTrace` и `InnerException`.

#### Что нужно сделать (пошагово)

1. **Создайте решение и проект.** В терминале выполните:
   ```
   dotnet new sln -n PayFlow.M07L03
   dotnet new console -n PayFlow.M07L03.Hw -o src/PayFlow.M07L03.Hw --framework net8.0
   dotnet sln add src/PayFlow.M07L03.Hw/PayFlow.M07L03.Hw.csproj
   ```
   Убедитесь, что в `.csproj` стоит `<TargetFramework>net8.0</TargetFramework>` и `<LangVersion>latest</LangVersion>` (для C# 12). Включите nullable-контекст: `<Nullable>enable</Nullable>`.

2. **Определите доменное исключение `TariffLoadException`** в файле `TariffLoadException.cs`. Оно должно наследоваться от `Exception` и иметь два конструктора: принимающий только сообщение, и принимающий сообщение плюс `innerException` — именно через него вы будете сохранять оригинал при обёртывании. Никаких лишних свойств пока не добавляйте — мы хотим чистый фокус на стеке.

3. **Реализуйте три «источника» в `Sources.cs`** — `FileSource`, `CacheSource`, `HttpSource` — каждый со статическим методом `decimal Load()`, который намеренно выбрасывает своё низкоуровневое исключение (`IOException`, `InvalidOperationException`, `HttpRequestException` соответственно) из отдельного приватного метода `ThrowRaw()`. Это нужно, чтобы у исходной ошибки была отчётливая точка возникновения со своим именем метода в стеке.

4. **Реализуйте `TariffRepository`** в `TariffRepository.cs` с тремя методами, использующими разные стратегии:
   - `LoadFromDiskPreservingStack(int tariffId)` — ловит `IOException`, логирует в консоль, выполняет **`throw;`** (чистый rethrow);
   - `LoadFromDiskWrapped(int tariffId)` — ловит `IOException` и выполняет **`throw new TariffLoadException("...", ex);`** (обёртывание с `innerException`);
   - `LoadFromDiskLosingStack(int tariffId)` — намеренно анти-образец: ловит `IOException` и выполняет **`throw ex;`** (для демонстрации сброса стека; пометьте этот метод комментарием `// ANTI-PATTERN: do not copy`).

5. **Реализуйте `BackgroundTariffLoader`** с методом `LoadOffThread()` — он запускает `Task.Run`, внутри фонового таска ловит исключение, сохраняет его в локальную переменную, а после `t.Wait()` (оборачивая `AggregateException`) перебрасывает через **`ExceptionDispatchInfo.Capture(captured).Throw()`**. Цель — показать, что стек остаётся «родным» для исходной точки в фоновой задаче.

6. **Реализуйте наблюдатель на фильтрах** в `Program.cs` через метод `LogOnce(Exception ex)`, который печатает `[FILTER] ...` и возвращает `false`. Используйте его в `catch (TariffLoadException ex) when (LogOnce(ex))`. Продемонстрируйте, что при `false` исключение продолжает лететь с сохранённым стеком.

7. **Запустите и соберите вывод.** Выполните `dotnet run --project src/PayFlow.M07L03.Hw`. Программа должна напечатать блоки вида `=== CASE 1: throw; ===`, затем трассировку стека, и для каждого случая — строку `ASSERT: ... OK` или `ASSERT: ... FAIL`. Ожидаемый результат: `throw;` содержит имя `ThrowRaw` в стеке; `throw ex;` его не содержит; обёрнутый вариант содержит `InnerException` типа `IOException`, у которого в `StackTrace` есть `ThrowRaw`; `ExceptionDispatchInfo`-сценарий также содержит `ThrowRaw`.

8. **Сделайте commit** с сообщением `hw(M07-L03): throw vs throw; vs throw new + ExceptionDispatchInfo`. Убедитесь, что в репозитории нет `bin/` и `obj/` (`.gitignore` от `dotnet new`).

#### Требования к решению

- Целевой фреймворк — строго `net8.0`, язык — C# 12; разрешены top-level statements, pattern matching (`is not null`), collection expressions, raw string literals (`"""..."""`).
- Nullable-анализ включён; проект компилируется без предупреждений `CS8602`, `CS8603` и подобных.
- В репозитории присутствуют ровно три стратегии проброса, и каждая помечена комментарием с русским и английским пояснением (`// ✅ ПРАВИЛЬНО: throw; сохраняет стек` / `// ✅ CORRECT: throw; preserves stack` и т. д.).
- Анти-паттерн `throw ex;` изолирован в отдельном методе с явной пометкой `ANTI-PATTERN` и используется **только** для демонстрации потери стека; в «боевых» методах его быть не должно.
- Все обёртывания выполняются через конструктор с `innerException`; нигде не встречается `throw new TariffLoadException("msg")` без второго аргумента в местах, где в `catch` есть доступ к оригиналу.
- Программа содержит assertion-проверки: для каждого из четырёх случаев (`throw;`, `throw ex;`, `throw new`, `ExceptionDispatchInfo`) она анализирует `ex.StackTrace` и `ex.InnerException` и печатает `OK`/`FAIL`.
- Фильтр `when (LogOnce(ex))` присутствует и возвращает `false`, демонстрируя, что блок не прерывает исключение.
- Пустые `catch { }` и `catch (Exception) { }` запрещены; минимум — логирование плюс проброс.

#### Тонкости и подводные камни

- **`throw;` доступен только внутри `catch`.** Вне `catch` (например, в обычном методе) написать `throw;` нельзя — компилятор выдаст ошибку. Если вы сохранили исключение в поле и хотите перебросить позже, это уже сценарий для `ExceptionDispatchInfo`.
- **`throw ex;` не «копирует» стек, а перезаписывает.** В .NET после поимки исключение получает новую точку трассировки на строке `throw ex;`. Оригинал теряется безвозвратно — даже если вы сохраните `ex.StackTrace` строкой заранее, повторно «приклеить» её к живому исключению нельзя.
- **Обёртывание без `innerException` — частый баг.** Конструктор `new TariffLoadException("msg")` без второго аргумента делает `InnerException` равным `null`. По логам вы увидите красивое сообщение, но не сможете дойти до `IOException`. Контракт: если в `catch (XException ex)` вы делаете `throw new Y(...)`, всегда передавайте `ex` вторым аргументом.
- **Фильтры `when (...)` вычисляются до входа в `catch`.** Это значит, что любой побочный эффект в фильтре (логирование) срабатывает, но если фильтр вернёт `false`, блок не считается «сработавшим» — исключение продолжает лететь с **оригинальным** стеком, словно `catch` не существовал. Используйте это для «мягкого наблюдения».
- **`ExceptionDispatchInfo` — для границ потоков.** Обычный `throw;` не годится, если исключение поймано в одном потоке, а пробросить нужно из другого: вы не находитесь в активном `catch`. `ExceptionDispatchInfo.Capture(ex).Throw()` «подсовывает» среде оригинальный стек. После поимки такого исключения в `StackTrace` будет и оригинальная точка, и точка вызова `.Throw()`.
- **Логируйте до проброса, а не вместо него.** `catch (Exception ex) { _log.Error(ex); }` без `throw;` «проглатывает» ошибку — для вызывающего кода всё выглядит успешным. Минимум: лог + `throw;`.
- **Не используйте `throw;` в фильтрах для «наблюдения».** Если вы хотите только залогировать и не менять поток, возвращайте `false` из фильтра, а не входите в `catch` с `throw;` — это чище и не добавляет лишний фрейм в логи.

#### Критерии приёмки

- [ ] Решение компилируется под `net8.0` / C# 12 без предупреждений и ошибок.
- [ ] Присутствует доменное исключение `TariffLoadException` с конструктором `(string message, Exception inner)`.
- [ ] Метод `LoadFromDiskPreservingStack` использует `throw;` и сохраняет `ThrowRaw` в трассировке.
- [ ] Метод `LoadFromDiskWrapped` использует `throw new TariffLoadException(msg, ex)` и выставляет `InnerException` типа `IOException`.
- [ ] Анти-паттерн `throw ex;` изолирован в методе `LoadFromDiskLosingStack` с пометкой `ANTI-PATTERN`.
- [ ] Assertion-проверка подтверждает, что после `throw ex;` в `StackTrace` нет `ThrowRaw`.
- [ ] `BackgroundTariffLoader.LoadOffThread` применяет `ExceptionDispatchInfo.Capture(captured).Throw()`.
- [ ] Assertion-проверка подтверждает, что в off-thread сценарии `ThrowRaw` присутствует в стеке.
- [ ] Присутствует фильтр `when (LogOnce(ex))`, возвращающий `false`, и демонстрируется «мягкий rethrow».
- [ ] Нет ни одного пустого `catch { }` или `catch (Exception) { }`.
- [ ] Везде, где есть обёртывание, оригинал передаётся через `innerException`.
- [ ] В логах видны блоки `=== CASE n: ... ===` и строки `ASSERT: ... OK/FAIL`.
- [ ] Код содержит двуязычные комментарии (RU + EN) в ключевых точках.
- [ ] `bin/` и `obj/` отсутствуют в коммите.
- [ ] `dotnet run` завершается с кодом 0 и печатает все четыре сценария.

#### Подсказки (без прямого ответа)

- Чтобы проверить наличие точки возникновения в стеке, используйте `ex.StackTrace?.Contains("ThrowRaw") == true`. Для `InnerException` — `ex.InnerException is IOException`.
- Не пытайтесь «восстановить» стек у `throw ex;` через рефлексию — это хрупко и не входит в цели урока. Просто наблюдайте потерю.
- Для `ExceptionDispatchInfo` ловите исключение в `Task.Run` в обычный `catch`, сохраняйте в `Exception? captured`, а после `t.Wait()` (с подавлением `AggregateException`) проверяйте `captured is not null` и вызывайте `Capture(...).Throw()`.
- Фильтр можно использовать и для условного проброса: `when (ex is IOException)` сработает только для нужного типа, не меняя стек для остальных.
- Помните, что `AggregateException` оборачивает внутренние ошибки фоновых тасков — вам нужен `.InnerException` или `captured`, а не сам `AggregateException`.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — PayFlow.M07L03.Hw
// Эталон решения ДЗ M07-L03: throw vs throw; vs throw new + ExceptionDispatchInfo
using System.Runtime.ExceptionServices;

namespace PayFlow.M07L03.Hw;

// Доменное исключение с обязательным конструктором innerException
// Domain exception with mandatory innerException constructor
public sealed class TariffLoadException : Exception
{
    public TariffLoadException(string message) : base(message) { }
    public TariffLoadException(string message, Exception inner) : base(message, inner) { }
}

// Три источника; каждый имеет приватный ThrowRaw() — точку возникновения стека
// Three sources; each has a private ThrowRaw() — the stack origin point
public static class FileSource
{
    public static decimal Load() => ThrowRaw();
    private static decimal ThrowRaw() =>
        throw new IOException("Disk sector unreadable (FileSource.ThrowRaw)");
}

public static class CacheSource
{
    public static decimal Load() => ThrowRaw();
    private static decimal ThrowRaw() =>
        throw new InvalidOperationException("Cache key desync (CacheSource.ThrowRaw)");
}

public static class HttpSource
{
    public static decimal Load() => ThrowRaw();
    private static decimal ThrowRaw() =>
        throw new HttpRequestException("Remote billing 5xx (HttpSource.ThrowRaw)");
}

public static class TariffRepository
{
    // ✅ ПРАВИЛЬНО: bare rethrow сохраняет оригинальный стек
    // ✅ CORRECT: bare rethrow keeps the original stack
    public static decimal LoadFromDiskPreservingStack(int tariffId)
    {
        try { return FileSource.Load(); }
        catch (IOException ex)
        {
            Console.WriteLine($"[LOG] preserve: tariff={tariffId}, {ex.Message}");
            throw; // стек содержит FileSource.ThrowRaw
        }
    }

    // ✅ ОБЁРТКА: новый тип, оригинал в innerException — цепочка сохранена
    // ✅ WRAPPING: new type, original in innerException — chain preserved
    public static decimal LoadFromDiskWrapped(int tariffId)
    {
        try { return FileSource.Load(); }
        catch (IOException ex)
        {
            Console.WriteLine($"[LOG] wrap: tariff={tariffId}, {ex.Message}");
            throw new TariffLoadException($"Cannot load tariff {tariffId} from disk", ex);
        }
    }

    // ❌ АНТИ-ПАТТЕРН: throw ex; сбрасывает стек до этой строки
    // ❌ ANTI-PATTERN: throw ex; resets the stack to this line
    public static decimal LoadFromDiskLosingStack(int tariffId)
    {
        try { return FileSource.Load(); }
        catch (IOException ex)
        {
            Console.WriteLine($"[LOG] lose: tariff={tariffId}, {ex.Message}");
            throw ex; // ANTI-PATTERN: do not copy
        }
    }
}

// Перенос исключения из фоновой задачи через границу потоков
// Carrying an exception from a background task across a thread boundary
public static class BackgroundTariffLoader
{
    public static decimal LoadOffThread()
    {
        Exception? captured = null;
        var t = Task.Run(() =>
        {
            try { return CacheSource.Load(); }
            catch (Exception ex) { captured = ex; } // снимок без потери стека
        });

        try { t.Wait(); }
        catch (AggregateException) { /* раскрыто ниже / unwrapped below */ }

        if (captured is not null)
            ExceptionDispatchInfo.Capture(captured).Throw(); // родной стек сохранён

        return 0m; // unreachable
    }
}

public static class Program
{
    private static bool LogOnce(Exception ex)
    {
        Console.WriteLine($"[FILTER] observed: {ex.Message}");
        return false; // false => блок не сработал, исключение летит дальше
    }

    public static void Main()
    {
        Run("=== CASE 1: throw; ===", () =>
        {
            try { TariffRepository.LoadFromDiskPreservingStack(42); }
            catch (Exception ex) { AssertStackHas(ex, "ThrowRaw"); }
        });

        Run("=== CASE 2: throw new ...(..., ex) ===", () =>
        {
            try { TariffRepository.LoadFromDiskWrapped(42); }
            catch (TariffLoadException ex)
            {
                Assert(ex.InnerException is IOException, "InnerException is IOException");
                AssertStackHas(ex.InnerException!, "ThrowRaw");
            }
        });

        Run("=== CASE 3: throw ex; (ANTI-PATTERN) ===", () =>
        {
            try { TariffRepository.LoadFromDiskLosingStack(42); }
            catch (Exception ex) { AssertStackMissing(ex, "ThrowRaw"); }
        });

        Run("=== CASE 4: ExceptionDispatchInfo ===", () =>
        {
            try { BackgroundTariffLoader.LoadOffThread(); }
            catch (Exception ex) { AssertStackHas(ex, "ThrowRaw"); }
        });

        Run("=== CASE 5: filter when (LogOnce) ===", () =>
        {
            try
            {
                try { TariffRepository.LoadFromDiskWrapped(7); }
                catch (TariffLoadException ex) when (LogOnce(ex)) { throw; }
            }
            catch (TariffLoadException ex) { AssertStackHas(ex.InnerException!, "ThrowRaw"); }
        });
    }

    private static void Run(string title, Action body)
    {
        Console.WriteLine(title);
        body();
        Console.WriteLine();
    }

    private static void AssertStackHas(Exception ex, string marker) =>
        Assert(ex.StackTrace?.Contains(marker) == true, $"stack contains '{marker}'");

    private static void AssertStackMissing(Exception ex, string marker) =>
        Assert(ex.StackTrace?.Contains(marker) != true, $"stack missing '{marker}' (lost)");

    private static void Assert(bool cond, string what) =>
        Console.WriteLine(cond ? $"ASSERT: {what} — OK" : $"ASSERT: {what} — FAIL");
}
```

**Разбор по строкам и концепциям урока.** Каждый метод иллюстрирует конкретную форму проброса из урока M07-L03. `LoadFromDiskPreservingStack` — эталон «чистого rethrow»: внутри `catch (IOException ex)` мы логируем контекст (тариф, сообщение) и выполняем `throw;` без операнда. Согласно теории урока, это «передаёт конверт дальше нераспечатанным» — среда оставляет активное исключение как есть, и его `StackTrace` по-прежнему указывает на `FileSource.ThrowRaw`. Assertion `AssertStackHas(ex, "ThrowRaw")` доказывает это текстово.

`LoadFromDiskWrapped` показывает третью форму — `throw new TariffLoadException(msg, ex)`. Здесь критически важен второй аргумент конструктора: он кладёт оригинальный `IOException` в `InnerException`. Урок явно предупреждает: обёртывание без `innerException` рвёт цепочку. Мы проверяем это двумя assertions — тип `InnerException` и наличие `ThrowRaw` в его `StackTrace`. Обратите внимание, что у самого `TariffLoadException` стек начинается в `LoadFromDiskWrapped`, но «внутренний адрес» всё ещё доступен через `InnerException` — ровно как в аналогии урока про «конверт в конверте».

`LoadFromDiskLosingStack` — намеренный анти-паттерн `throw ex;`. Урок называет это «почти всегда багом»: трассировка переписывается на строку `throw ex;`, оригинальная точка пропадает. `AssertStackMissing(ex, "ThrowRaw")` ловит именно эту потерю — если в стеке нет `ThrowRaw`, значит стек был сброшен. Метод помечен комментарием `ANTI-PATTERN: do not copy`, чтобы никто не копировал его в продакшен.

`BackgroundTariffLoader.LoadOffThread` раскрывает сценарий `ExceptionDispatchInfo`, который в уроке описан как средство «фотографировать исключение вместе со стеком и перезапустить его позже из другого места». Мы ловим ошибку в `Task.Run`, сохраняем в `captured`, а после `t.Wait()` (с подавлением `AggregateException`) вызываем `ExceptionDispatchInfo.Capture(captured).Throw()`. Assertion подтверждает: несмотря на пересечение границы потоков, `ThrowRaw` остаётся в стеке. Это поведение недостижимо ни `throw;` (мы не в активном `catch`), ни `throw ex;` (стек бы сбросился).

Наконец, фильтр `when (LogOnce(ex))` в `Main` — это «мягкий rethrow» из теории урока. `LogOnce` возвращает `false`, поэтому блок `catch` не считается сработавшим, и исключение продолжает лететь с оригинальным стеком. Внешний `catch` ловит уже летящее `TariffLoadException` и видит тот же `InnerException` со стеком `ThrowRaw` — поток управления не изменился. Эта часть закрепляет best practice «используйте фильтры для логирования без побочного эффекта на стек».

#### Задания на углубление (бонус)

1. **Сравните `throw;` и `ExceptionDispatchInfo` по производительности.** Измерьте через `BenchmarkDotNet` время 100 000 пробросов в цикле для обоих вариантов. Объясните, почему `ExceptionDispatchInfo` медленнее, и когда это оправдано.
2. **Динамическая перестановка стека.** Используя `ExceptionDispatchInfo.Capture(ex).Throw()` внутри `catch (Exception ex) when (...)`, покажите, что можно «отложить» переброс на несколько строк позже, не теряя стек. Сравните с попыткой сохранить `ex.StackTrace` строкой и «приклеить» обратно — объясните, почему второй способ не работает.
3. **Цепочка из трёх уровней.** Реализуйте `Repository → Service → Controller`, где каждый уровень либо делает `throw;`, либо обёртывает в своё исключение с `innerException`. Напишите assertion, проверяющий, что `GetBaseException()` возвращает исходный `IOException`, а `InnerException?.InnerException` корректно выстроен.
4. **Фильтр с условной логикой.** Реализуйте `when (ShouldRetry(ex, attempt))`, который возвращает `true` ровно один раз (первая попытка), позволяя `catch` сделать retry, а затем `false`, чтобы исключение пошло выше. Проследите, что стек не искажается между попытками.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are an engineer on the `PayFlow` payment gateway team. The service reads tariff configuration from three sources: a local JSON file on disk, an in-memory cache, and a remote billing HTTP endpoint. Each source can fail in its own way: the file throws `IOException` (bad sector), the cache throws `InvalidOperationException` (key desync), the HTTP layer throws `HttpRequestException` (remote 5xx). The business contract of the service requires that a domain exception `TariffLoadException` always bubbles up to the caller, while the diagnostic chain down to the original error must never be lost: observability logs (OpenTelemetry traces) must show exactly the line where the failure originated, not the line where you "repacked" it.

The team has been bitten twice by a classic bug: developers wrote `catch (Exception ex) { _logger.Warn(...); throw ex; }` and thereby wiped the original origin point. As a result, production incidents were investigated for days instead of minutes. Your tech lead tasked you with implementing a reference loader `TariffRepository` that demonstrates three deliberate propagation strategies and that other team members can copy as a template. In parallel, you must add an "observer" based on exception filters that logs each failure exactly once without affecting control flow, and you must demonstrate the correct transfer of an exception from a `Task.Run` background task back to the calling thread while preserving the stack via `ExceptionDispatchInfo`. The assignment is intentionally designed so that every solution is verifiable by a text assertion on the contents of `StackTrace` and `InnerException`.

#### What to do step by step

1. **Create the solution and project.** In the terminal, run:
   ```
   dotnet new sln -n PayFlow.M07L03
   dotnet new console -n PayFlow.M07L03.Hw -o src/PayFlow.M07L03.Hw --framework net8.0
   dotnet sln add src/PayFlow.M07L03.Hw/PayFlow.M07L03.Hw.csproj
   ```
   Make sure `.csproj` contains `<TargetFramework>net8.0</TargetFramework>` and `<LangVersion>latest</LangVersion>` for C# 12. Enable the nullable context: `<Nullable>enable</Nullable>`.

2. **Define the domain exception `TariffLoadException`** in `TariffLoadException.cs`. It must derive from `Exception` and expose two constructors: one taking only a message, and one taking a message plus an `innerException` — this is exactly how you preserve the original when wrapping. Do not add extra properties yet; we want a clean focus on the stack.

3. **Implement three sources in `Sources.cs`** — `FileSource`, `CacheSource`, `HttpSource` — each with a static `decimal Load()` method that deliberately throws its low-level exception (`IOException`, `InvalidOperationException`, `HttpRequestException` respectively) from a separate private `ThrowRaw()` method. This gives the original error a distinct origin point with its own method name in the stack.

4. **Implement `TariffRepository`** in `TariffRepository.cs` with three methods using different strategies:
   - `LoadFromDiskPreservingStack(int tariffId)` — catches `IOException`, logs to console, performs **`throw;`** (bare rethrow);
   - `LoadFromDiskWrapped(int tariffId)` — catches `IOException` and performs **`throw new TariffLoadException("...", ex);`** (wrapping with `innerException`);
   - `LoadFromDiskLosingStack(int tariffId)` — an intentional anti-pattern: catches `IOException` and performs **`throw ex;`** (to demonstrate stack reset; mark the method with a `// ANTI-PATTERN: do not copy` comment).

5. **Implement `BackgroundTariffLoader`** with a `LoadOffThread()` method — it starts a `Task.Run`, catches the exception inside the background task, stores it in a local variable, and after `t.Wait()` (unwrapping `AggregateException`) rethrows via **`ExceptionDispatchInfo.Capture(captured).Throw()`**. The goal is to show that the stack remains "native" to the original point inside the background task.

6. **Implement the filter-based observer** in `Program.cs` via a `LogOnce(Exception ex)` method that prints `[FILTER] ...` and returns `false`. Use it in `catch (TariffLoadException ex) when (LogOnce(ex))`. Demonstrate that when `false` is returned, the exception keeps flying with the preserved stack.

7. **Run and collect output.** Execute `dotnet run --project src/PayFlow.M07L03.Hw`. The program must print blocks like `=== CASE 1: throw; ===`, then the stack trace, and for each case a line `ASSERT: ... OK` or `ASSERT: ... FAIL`. Expected outcome: `throw;` contains the name `ThrowRaw` in the stack; `throw ex;` does not; the wrapped variant has an `InnerException` of type `IOException` whose `StackTrace` contains `ThrowRaw`; the `ExceptionDispatchInfo` scenario also contains `ThrowRaw`.

8. **Commit** with the message `hw(M07-L03): throw vs throw; vs throw new + ExceptionDispatchInfo`. Make sure `bin/` and `obj/` are not in the repository (the `.gitignore` from `dotnet new` handles this).

#### Requirements

- Target framework is strictly `net8.0`, language is C# 12; top-level statements, pattern matching (`is not null`), collection expressions, and raw string literals (`"""..."""`) are all allowed.
- Nullable analysis is enabled; the project compiles without `CS8602`, `CS8603` or similar warnings.
- Exactly three propagation strategies are present, each annotated with a Russian and English comment (`// ✅ CORRECT: throw; preserves the stack` / `// ✅ ПРАВИЛЬНО: throw; сохраняет стек`, etc.).
- The `throw ex;` anti-pattern is isolated in a dedicated method marked `ANTI-PATTERN` and is used **only** to demonstrate stack loss; it must not appear in "production" methods.
- All wrapping goes through the constructor that accepts `innerException`; nowhere in a `catch` block do you find `throw new TariffLoadException("msg")` without the second argument when the original is available.
- The program contains assertion checks: for each of the four cases (`throw;`, `throw ex;`, `throw new`, `ExceptionDispatchInfo`) it analyzes `ex.StackTrace` and `ex.InnerException` and prints `OK`/`FAIL`.
- The `when (LogOnce(ex))` filter is present and returns `false`, demonstrating that the block does not stop the exception.
- Empty `catch { }` and `catch (Exception) { }` are forbidden; at minimum, log and rethrow.

#### Pitfalls

- **`throw;` is only valid inside `catch`.** Outside an active `catch` (for instance in a regular method) you cannot write `throw;` — the compiler will reject it. If you have stored the exception in a field and want to rethrow later, that is exactly the `ExceptionDispatchInfo` scenario.
- **`throw ex;` does not "copy" the stack — it overwrites it.** In .NET, after the catch, the exception gets a new trace point at the `throw ex;` line. The original is lost irrecoverably; even if you saved `ex.StackTrace` as a string in advance, you cannot "glue" it back onto a live exception.
- **Wrapping without `innerException` is a frequent bug.** The constructor `new TariffLoadException("msg")` without the second argument leaves `InnerException` as `null`. You will see a nice message in the logs but will not be able to reach the underlying `IOException`. The contract: in `catch (XException ex)`, if you do `throw new Y(...)`, always pass `ex` as the second argument.
- **`when (...)` filters are evaluated before entering `catch`.** This means any side effect in the filter (logging) fires, but if the filter returns `false`, the block is not considered "entered" — the exception continues with the **original** stack, as if the `catch` did not exist. Use this for "soft observation".
- **`ExceptionDispatchInfo` is for thread boundaries.** A plain `throw;` is not suitable when the exception was caught in one thread but must be rethrown from another: you are not in an active `catch`. `ExceptionDispatchInfo.Capture(ex).Throw()` "feeds" the runtime the original stack. After catching such an exception, the `StackTrace` contains both the original point and the `.Throw()` call site.
- **Log before rethrowing, not instead of it.** `catch (Exception ex) { _log.Error(ex); }` without `throw;` "swallows" the error — to the caller everything looks successful. Minimum: log + `throw;`.
- **Do not use `throw;` inside filters for "observation".** If you only want to log without changing the flow, return `false` from the filter rather than entering `catch` with `throw;` — this is cleaner and does not add an extra frame to the logs.

#### Acceptance criteria

- [ ] The solution compiles under `net8.0` / C# 12 with no warnings or errors.
- [ ] The domain exception `TariffLoadException` is present with a `(string message, Exception inner)` constructor.
- [ ] `LoadFromDiskPreservingStack` uses `throw;` and keeps `ThrowRaw` in the trace.
- [ ] `LoadFromDiskWrapped` uses `throw new TariffLoadException(msg, ex)` and sets `InnerException` to an `IOException`.
- [ ] The `throw ex;` anti-pattern is isolated in `LoadFromDiskLosingStack` with the `ANTI-PATTERN` marker.
- [ ] The assertion check confirms that after `throw ex;` the `StackTrace` does not contain `ThrowRaw`.
- [ ] `BackgroundTariffLoader.LoadOffThread` uses `ExceptionDispatchInfo.Capture(captured).Throw()`.
- [ ] The assertion check confirms that in the off-thread scenario `ThrowRaw` is present in the stack.
- [ ] The `when (LogOnce(ex))` filter returning `false` is present and the "soft rethrow" is demonstrated.
- [ ] There are no empty `catch { }` or `catch (Exception) { }` blocks.
- [ ] Everywhere wrapping occurs, the original is passed via `innerException`.
- [ ] The logs show `=== CASE n: ... ===` blocks and `ASSERT: ... OK/FAIL` lines.
- [ ] The code contains bilingual comments (RU + EN) at the key points.
- [ ] `bin/` and `obj/` are not in the commit.
- [ ] `dotnet run` exits with code 0 and prints all four scenarios.

#### Hints (no direct answer)

- To check that the origin point is in the stack, use `ex.StackTrace?.Contains("ThrowRaw") == true`. For `InnerException`, check `ex.InnerException is IOException`.
- Do not try to "restore" the stack of `throw ex;` via reflection — it is fragile and outside the scope of the lesson. Just observe the loss.
- For `ExceptionDispatchInfo`, catch the exception inside `Task.Run` in a normal `catch`, store it in `Exception? captured`, and after `t.Wait()` (suppressing `AggregateException`) check `captured is not null` and call `Capture(...).Throw()`.
- A filter can also be used for conditional propagation: `when (ex is IOException)` fires only for the desired type without changing the stack for others.
- Remember that `AggregateException` wraps the inner errors of background tasks — you need `.InnerException` or `captured`, not the `AggregateException` itself.

#### Reference solution walk-through

```csharp
// C# 12 / .NET 8 — PayFlow.M07L03.Hw
// Reference solution for HW M07-L03: throw vs throw; vs throw new + ExceptionDispatchInfo
using System.Runtime.ExceptionServices;

namespace PayFlow.M07L03.Hw;

// Domain exception with a mandatory innerException constructor
public sealed class TariffLoadException : Exception
{
    public TariffLoadException(string message) : base(message) { }
    public TariffLoadException(string message, Exception inner) : base(message, inner) { }
}

// Three sources; each has a private ThrowRaw() — the stack origin point
public static class FileSource
{
    public static decimal Load() => ThrowRaw();
    private static decimal ThrowRaw() =>
        throw new IOException("Disk sector unreadable (FileSource.ThrowRaw)");
}

public static class CacheSource
{
    public static decimal Load() => ThrowRaw();
    private static decimal ThrowRaw() =>
        throw new InvalidOperationException("Cache key desync (CacheSource.ThrowRaw)");
}

public static class HttpSource
{
    public static decimal Load() => ThrowRaw();
    private static decimal ThrowRaw() =>
        throw new HttpRequestException("Remote billing 5xx (HttpSource.ThrowRaw)");
}

public static class TariffRepository
{
    // ✅ CORRECT: bare rethrow keeps the original stack
    public static decimal LoadFromDiskPreservingStack(int tariffId)
    {
        try { return FileSource.Load(); }
        catch (IOException ex)
        {
            Console.WriteLine($"[LOG] preserve: tariff={tariffId}, {ex.Message}");
            throw; // stack contains FileSource.ThrowRaw
        }
    }

    // ✅ WRAPPING: new type, original in innerException — chain preserved
    public static decimal LoadFromDiskWrapped(int tariffId)
    {
        try { return FileSource.Load(); }
        catch (IOException ex)
        {
            Console.WriteLine($"[LOG] wrap: tariff={tariffId}, {ex.Message}");
            throw new TariffLoadException($"Cannot load tariff {tariffId} from disk", ex);
        }
    }

    // ❌ ANTI-PATTERN: throw ex; resets the stack to this line
    public static decimal LoadFromDiskLosingStack(int tariffId)
    {
        try { return FileSource.Load(); }
        catch (IOException ex)
        {
            Console.WriteLine($"[LOG] lose: tariff={tariffId}, {ex.Message}");
            throw ex; // ANTI-PATTERN: do not copy
        }
    }
}

// Carrying an exception from a background task across a thread boundary
public static class BackgroundTariffLoader
{
    public static decimal LoadOffThread()
    {
        Exception? captured = null;
        var t = Task.Run(() =>
        {
            try { return CacheSource.Load(); }
            catch (Exception ex) { captured = ex; } // snapshot without losing the stack
        });

        try { t.Wait(); }
        catch (AggregateException) { /* unwrapped below */ }

        if (captured is not null)
            ExceptionDispatchInfo.Capture(captured).Throw(); // native stack preserved

        return 0m; // unreachable
    }
}

public static class Program
{
    private static bool LogOnce(Exception ex)
    {
        Console.WriteLine($"[FILTER] observed: {ex.Message}");
        return false; // false => block not entered, exception keeps flying
    }

    public static void Main()
    {
        Run("=== CASE 1: throw; ===", () =>
        {
            try { TariffRepository.LoadFromDiskPreservingStack(42); }
            catch (Exception ex) { AssertStackHas(ex, "ThrowRaw"); }
        });

        Run("=== CASE 2: throw new ...(..., ex) ===", () =>
        {
            try { TariffRepository.LoadFromDiskWrapped(42); }
            catch (TariffLoadException ex)
            {
                Assert(ex.InnerException is IOException, "InnerException is IOException");
                AssertStackHas(ex.InnerException!, "ThrowRaw");
            }
        });

        Run("=== CASE 3: throw ex; (ANTI-PATTERN) ===", () =>
        {
            try { TariffRepository.LoadFromDiskLosingStack(42); }
            catch (Exception ex) { AssertStackMissing(ex, "ThrowRaw"); }
        });

        Run("=== CASE 4: ExceptionDispatchInfo ===", () =>
        {
            try { BackgroundTariffLoader.LoadOffThread(); }
            catch (Exception ex) { AssertStackHas(ex, "ThrowRaw"); }
        });

        Run("=== CASE 5: filter when (LogOnce) ===", () =>
        {
            try
            {
                try { TariffRepository.LoadFromDiskWrapped(7); }
                catch (TariffLoadException ex) when (LogOnce(ex)) { throw; }
            }
            catch (TariffLoadException ex) { AssertStackHas(ex.InnerException!, "ThrowRaw"); }
        });
    }

    private static void Run(string title, Action body)
    {
        Console.WriteLine(title);
        body();
        Console.WriteLine();
    }

    private static void AssertStackHas(Exception ex, string marker) =>
        Assert(ex.StackTrace?.Contains(marker) == true, $"stack contains '{marker}'");

    private static void AssertStackMissing(Exception ex, string marker) =>
        Assert(ex.StackTrace?.Contains(marker) != true, $"stack missing '{marker}' (lost)");

    private static void Assert(bool cond, string what) =>
        Console.WriteLine(cond ? $"ASSERT: {what} — OK" : $"ASSERT: {what} — FAIL");
}
```

**Line-by-line walk-through and concepts applied.** Each method illustrates a specific propagation form from lesson M07-L03. `LoadFromDiskPreservingStack` is the reference "bare rethrow": inside `catch (IOException ex)` we log context (tariff id, message) and execute `throw;` with no operand. Per the lesson theory, this "passes the envelope along unopened" — the runtime leaves the active exception untouched, and its `StackTrace` still points at `FileSource.ThrowRaw`. The `AssertStackHas(ex, "ThrowRaw")` assertion proves this textually.

`LoadFromDiskWrapped` shows the third form — `throw new TariffLoadException(msg, ex)`. The second constructor argument is critical: it places the original `IOException` into `InnerException`. The lesson explicitly warns that wrapping without `innerException` breaks the chain. We verify this with two assertions — the type of `InnerException` and the presence of `ThrowRaw` in its `StackTrace`. Note that the `TariffLoadException` itself has a trace starting at `LoadFromDiskWrapped`, but the "inner address" is still reachable through `InnerException` — exactly the "envelope inside an envelope" analogy from the lesson.

`LoadFromDiskLosingStack` is the intentional `throw ex;` anti-pattern. The lesson calls this "almost always a bug": the trace is rewritten to the `throw ex;` line, and the original point disappears. `AssertStackMissing(ex, "ThrowRaw")` catches precisely this loss — if `ThrowRaw` is not in the stack, the stack was reset. The method is annotated with `ANTI-PATTERN: do not copy` so nobody copies it into production.

`BackgroundTariffLoader.LoadOffThread` reveals the `ExceptionDispatchInfo` scenario, which the lesson describes as a way to "snapshot the exception together with its stack and restart it later from a different place". We catch the error inside `Task.Run`, store it in `captured`, and after `t.Wait()` (suppressing `AggregateException`) we call `ExceptionDispatchInfo.Capture(captured).Throw()`. The assertion confirms: despite crossing a thread boundary, `ThrowRaw` remains in the stack. This behavior is unreachable with `throw;` (we are not in an active `catch`) or with `throw ex;` (the stack would be reset).

Finally, the `when (LogOnce(ex))` filter in `Main` is the "soft rethrow" from the lesson theory. `LogOnce` returns `false`, so the `catch` block is not considered entered, and the exception keeps flying with the original stack. The outer `catch` catches the flying `TariffLoadException` and sees the same `InnerException` with the `ThrowRaw` stack — control flow has not changed. This part reinforces the best practice "use filters for logging without side effects on the stack".

#### Going deeper (bonus)

1. **Compare `throw;` and `ExceptionDispatchInfo` for performance.** Measure with `BenchmarkDotNet` the time of 100 000 rethrows in a loop for both variants. Explain why `ExceptionDispatchInfo` is slower and when that cost is justified.
2. **Dynamic stack repositioning.** Using `ExceptionDispatchInfo.Capture(ex).Throw()` inside `catch (Exception ex) when (...)`, show that you can defer the rethrow by a few lines without losing the stack. Compare this with trying to save `ex.StackTrace` as a string and "glue" it back — explain why the second approach does not work.
3. **A three-level chain.** Implement `Repository → Service → Controller`, where each level either does `throw;` or wraps into its own exception with `innerException`. Write an assertion verifying that `GetBaseException()` returns the original `IOException` and that `InnerException?.InnerException` is correctly chained.
4. **A filter with conditional logic.** Implement `when (ShouldRetry(ex, attempt))` that returns `true` exactly once (the first attempt), letting `catch` perform a retry, and then `false` so the exception propagates upward. Verify that the stack is not distorted between attempts.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Решение собрано под `net8.0` / C# 12 без предупреждений.
- [ ] Присутствуют все пять CASE-сценариев с assertions.
- [ ] `throw;`, `throw ex;`, `throw new ...(..., ex)` и `ExceptionDispatchInfo` — каждый продемонстрирован отдельно.
- [ ] Фильтр `when (LogOnce(ex))` возвращает `false` и не прерывает исключение.
- [ ] Никаких пустых `catch { }`; все обёртывания передают оригинал через `innerException`.
- [ ] Двуязычные комментарии в ключевых точках кода.
- [ ] Коммит не содержит `bin/` и `obj/`.
- [ ] Solution compiles under `net8.0` / C# 12 with no warnings.
- [ ] All five CASE scenarios with assertions are present.
- [ ] `throw;`, `throw ex;`, `throw new ...(..., ex)` and `ExceptionDispatchInfo` are each demonstrated separately.
- [ ] The `when (LogOnce(ex))` filter returns `false` and does not stop the exception.
- [ ] No empty `catch { }`; all wrapping passes the original via `innerException`.
- [ ] Bilingual comments at key points of the code.
- [ ] The commit does not contain `bin/` or `obj/`.

#### Ресурсы / Resources
- [Microsoft Learn — throw (C# reference)](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/throw)
- [ExceptionDispatchInfo — .NET API](https://learn.microsoft.com/dotnet/api/system.runtime.exceptionservices.exceptiondispatchinfo)
- [Best practices for exceptions in .NET](https://learn.microsoft.com/dotnet/standard/exceptions/best-practices-for-exceptions)
- [How to use exception filters (C# Programming Guide)](https://learn.microsoft.com/dotnet/csharp/language-reference/statements/exception-handling-statements)
- [InnerException and exception chaining — pattern guidance](https://learn.microsoft.com/dotnet/standard/exceptions/how-to-use-specific-exceptions-in-a-catch-block)
