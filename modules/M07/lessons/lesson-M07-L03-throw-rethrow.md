[⬆ К модулю M07](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M07-L03: throw и throw; (rethrow) — разница стеков / throw and throw; (rethrow) — stack difference

**Модуль / Module:** M07
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Когда в C# возникает исключение, среда выполнения собирает так называемый **трассировку стека** (stack trace) — список вызовов методов, который показывает, где именно произошла ошибка и как мы дошли до этой точки. Трассировка стека — это «следы на снегу»: по ним следопыт (разработчик или логи) понимает, откуда пришла проблема. От того, как именно вы пробросите исключение дальше, зависит, сохранятся эти следы или будут затёрты.

Существует три основных способа пробросить исключение, и разница между ними критична для диагностики:

1. **`throw;`** (rethrow, «чистый проброс») — пробрасывает текущее активное исключение без изменений. Стек **сохраняется полностью**: точка первоначального возникновения остаётся в трассировке. Это то, что вы почти всегда хотите в блоке `catch`, если просто хотите «отметиться» (например, залогировать) и передать ошибку выше.

2. **`throw ex;`** (где `ex` — пойманная переменная) — пробрасывает то же исключение, но **сбрасывает трассировку стека**. В трассировке теперь значится строка `throw ex;`, а первоначальная точка возникновения пропадает. Следы на снегу затёрты. Это почти всегда ошибка: вы лишаете команду возможности быстро найти корень проблемы.

3. **`throw new SomeException(...);`** — создаёт **новое** исключение. У него новая трассировка, начинающаяся с этой строки. Используйте этот вариант, когда хотите *обернуть* низкоуровневую ошибку в более осмысленное исключение верхнего уровня (например, `IOException` → `MyAppConfigurationException`). В этом случае обязательно передавайте оригинал через параметр `innerException`, чтобы цепочка не потерялась: `throw new MyAppException("Не удалось загрузить конфиг", ex);`.

Ключевая аналогия: представьте исключение как конверт с обратным адресом. `throw;` передаёт конверт дальше нераспечатанным. `throw ex;` переписывает обратный адрес на ваш. `throw new ...(..., ex)` кладёт исходный конверт в новый, больший конверт, и пишет на нём новый адрес — но внутри адрес всё ещё доступен.

**Когда использовать rethrow (`throw;`)?**

- В блоке `catch` вы лишь логируете, метите (telemetry) или освобождаете ресурсы, а обрабатывать ошибку должен уровень выше.
- Вы добавляете контекст, но не меняете тип исключения.
- В `catch (Exception ex) when (someFilter)` — так называемые *exception filters* фактически являются «мягким rethrow»: если фильтр вернёт `false`, исключение продолжит путь с сохранённым стеком, словно блока `catch` вообще не было.

**Когда оборачивать (`throw new ...(..., ex)`)?**

- Нижний уровень выбрасывает слишком техническое исключение (`SqlException`, `SocketException`), а контракт вашего API ожидает доменное исключение (`OrderRepositoryException`).
- Нужно добавить человекочитаемое сообщение, идентификатор операции или код ошибки.
- Важно: используйте `ExceptionDispatchInfo.Capture(ex).Throw()` (см. ниже), если исключение пересекает границу потоков/задач и нужно сохранить оригинальный стек.

**ExceptionDispatchInfo** — это класс из `System.Runtime.ExceptionServices`. Он «фотографирует» исключение вместе с его стеком в момент поимки и позволяет «перезапустить» его позже из другого места, не сбрасывая стек. Это особенно полезно, когда исключение ловят в одном потоке/задаче, а пробросить нужно из другого (например, из `Task.Run`, из `await`-конвейера, из фонового `BackgroundService`), или когда вы временно отложили обработку. Вызов `ExceptionDispatchInfo.Capture(ex).Throw()` технически выбрасывает исключение заново, но стек остаётся «родным». Обратите внимание: поймать такое исключение можно обычным `catch`, и его `StackTrace` будет содержать оригинальную точку.

Итог: правило большого пальца — в 95% случаев внутри `catch` используйте `throw;`. Используйте `throw new ...(..., ex)` для оборачивания с сохранением цепочки через `innerException`. Избегайте `throw ex;` — он почти всегда признак бага. Для межпоточных сценариев применяйте `ExceptionDispatchInfo`.

#### Theory (EN)

When an exception is raised in C#, the runtime collects a **stack trace** — the list of method calls that shows exactly where the error happened and how execution reached that point. A stack trace is like footprints in snow: a tracker (a developer, or your logs) uses them to find the origin of the problem. How you re-throw that exception determines whether those footprints stay intact or get wiped.

There are three main ways to propagate an exception, and the difference matters enormously for diagnostics:

1. **`throw;`** (rethrow, the “bare throw”) — propagates the current active exception unchanged. The stack is **preserved fully**: the original point of failure remains in the trace. This is almost always what you want inside a `catch` block when your only intent is to “take note” (log, add telemetry) and let the error travel upward.

2. **`throw ex;`** (where `ex` is the caught variable) — re-throws the same exception instance, but **resets the stack trace**. The trace now points to the `throw ex;` line, and the original origin is gone. The footprints are wiped. This is almost always a mistake: you strip your team of the ability to quickly find the root cause.

3. **`throw new SomeException(...);`** — creates a **new** exception with a brand-new trace starting at this line. Use this when you want to *wrap* a low-level error in a more meaningful, higher-level exception (for example, `IOException` → `MyAppConfigurationException`). Critically, pass the original via the `innerException` parameter so the chain is preserved: `throw new MyAppException("Failed to load config", ex);`.

The core analogy: think of an exception as an envelope with a return address. `throw;` passes the envelope along unopened. `throw ex;` overwrites the return address with your own. `throw new ...(..., ex)` puts the original envelope inside a new, larger envelope and writes a new address on the outside — but the inner address is still reachable.

**When to use rethrow (`throw;`)?**

- Inside `catch`, you are only logging, tagging (telemetry), or releasing resources, and the caller is responsible for handling the error.
- You are adding context but not changing the exception type.
- With `catch (Exception ex) when (someFilter)` — so-called *exception filters* behave like a “soft rethrow”: if the filter returns `false`, the exception continues with its original stack, as if the `catch` block never existed.

**When to wrap (`throw new ...(..., ex)`)?**

- A lower layer throws something too technical (`SqlException`, `SocketException`), and your API contract promises a domain-level exception (`OrderRepositoryException`).
- You need to add a human-readable message, an operation id, or an error code.
- Note: use `ExceptionDispatchInfo.Capture(ex).Throw()` (see below) when the exception must cross a thread/task boundary while keeping its original stack.

**ExceptionDispatchInfo** is a class from `System.Runtime.ExceptionServices`. It “snapshots” the exception together with its stack at the moment it is caught and lets you “restart” it later from a different location without resetting the stack. This is especially useful when an exception is caught in one thread/task but must be rethrown from another (for example, out of `Task.Run`, across an `await` pipeline, or from a background `BackgroundService`), or when handling is deferred. The call `ExceptionDispatchInfo.Capture(ex).Throw()` technically throws the exception again, but the stack stays native. Such an exception is caught by an ordinary `catch`, and its `StackTrace` still contains the original origin point.

Bottom line: as a rule of thumb, use `throw;` in 95% of cases inside `catch`. Use `throw new ...(..., ex)` to wrap while preserving the chain via `innerException`. Avoid `throw ex;` — it is almost always a bug. For cross-thread scenarios, reach for `ExceptionDispatchInfo`.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8+ — throw vs throw; vs throw new + ExceptionDispatchInfo
using System;
using System.Runtime.ExceptionServices;

namespace M07L03.Demo;

// Доменное исключение верхнего уровня / top-level domain exception
public sealed class OrderRepositoryException : Exception
{
    public OrderRepositoryException(string message, Exception inner) : base(message, inner) { }
}

public static class OrderRepository
{
    // Демонстрация трёх стилей проброса / demo of three rethrow styles
    public static decimal LoadPrice(int orderId)
    {
        try
        {
            return ReadPriceFromDisk(orderId); // может выбросить IOException / may throw IOException
        }
        catch (IOException ex)
        {
            // Логируем без потери стека / log without losing the stack
            Console.WriteLine($"[LOG] LoadPrice failed for order {orderId}: {ex.Message}");

            // ✅ ПРАВИЛЬНО: bare rethrow сохраняет оригинальную точку возникновения
            // ✅ CORRECT: bare rethrow keeps the original origin point
            throw;

            // ❌ НЕ ДЕЛАЙТЕ ТАК: throw ex; сбросит стек до этой строки
            // ❌ DO NOT: throw ex; resets the stack to this line
            // throw ex;

            // ✅ ОБЁРТКА (альтернатива): новый тип, оригинал в innerException
            // ✅ WRAPPING (alternative): new type, original kept as inner
            // throw new OrderRepositoryException($"Cannot load order {orderId}", ex);
        }
    }

    private static decimal ReadPriceFromDisk(int orderId)
    {
        // Имитация низкоуровневой ошибки / simulate a low-level failure
        throw new IOException("Disk sector unreadable"); // <- исходная точка стека / original stack origin
    }
}

// ExceptionDispatchInfo: пересечение границы потоков с сохранением стека
// crossing a thread boundary while preserving the stack
public static class BackgroundRunner
{
    public static void RunOffThread()
    {
        Exception? captured = null;

        // Фоновая задача ловит ошибку, но пробросить должна наружу
        // background task catches but must surface the error
        var t = Task.Run(() =>
        {
            try { ThrowInsideBackgroundTask(); }
            catch (Exception ex)
            {
                // «Фотографируем» исключение вместе со стеком
                // snapshot the exception together with its stack
                captured = ex;
            }
        });

        try { t.Wait(); }
        catch (AggregateException) { /* раскрыто ниже / unwrapped below */ }

        if (captured is not null)
        {
            // Перебрасываем из главного потока, сохраняя родной стек
            // rethrow from the main thread, native stack preserved
            ExceptionDispatchInfo.Capture(captured).Throw();
        }
    }

    private static void ThrowInsideBackgroundTask() =>
        throw new InvalidOperationException("Boom from background task");
}

public static class Program
{
    public static void Main()
    {
        // Фильтры исключений = «мягкий rethrow»: стек не трогается, если фильтр false
        // exception filters = "soft rethrow": stack untouched when filter is false
        try
        {
            int v = int.Parse("not-a-number");
        }
        catch (FormatException ex) when (LogOnce(ex))
        {
            // сюда попадём только если LogOnce вернёт true; иначе исключение летит дальше
            // reached only if LogOnce returns true; otherwise the exception keeps flying
            throw; // стек сохранён / stack preserved
        }
    }

    private static bool LogOnce(Exception ex)
    {
        Console.WriteLine($"[FILTER] {ex.Message}");
        return false; // false => блок не сработал, исключение не прерывается / block not entered
    }
}
```

#### Best Practices

- В блоке `catch`, если не обрабатываете ошибку окончательно, используйте `throw;` — это сохраняет стек и не создаёт шума в логах.
- Оборачивая исключение, всегда передавайте оригинал через `innerException`, чтобы сохранить диагностическую цепочку.
- Логируйте в `catch` до `throw;`, а не вместо него — лог без проброса «съедает» ошибку.
- Используйте фильтры исключений (`when (...)`) для логирования без побочного эффекта на стек.
- Для проброса между потоками, задачами и границами `await` применяйте `ExceptionDispatchInfo.Capture(ex).Throw()`.
- Не проглатывайте исключения пустым `catch { }` или `catch (Exception) { }` — это скрывает баги.

- Inside `catch`, if you are not handling the error conclusively, use `throw;` — it preserves the stack and avoids log noise.
- When wrapping, always pass the original via `innerException` to keep the diagnostic chain intact.
- Log in `catch` before `throw;`, never instead of it — a log without a rethrow “swallows” the error.
- Use exception filters (`when (...)`) for logging without side effects on the stack.
- For propagation across threads, tasks, and `await` boundaries, use `ExceptionDispatchInfo.Capture(ex).Throw()`.
- Never swallow exceptions with an empty `catch { }` or `catch (Exception) { }` — it hides bugs.

#### Частые ошибки / Common Mistakes

- `throw ex;` вместо `throw;` → теряется оригинальная точка возникновения. Используйте `throw;` для чистого проброса.
- Обёртывание без `innerException`: `throw new MyEx("msg")` вместо `throw new MyEx("msg", ex)` → рвётся цепочка. Всегда передавайте оригинал вторым аргументом.
- Пустой `catch { }` «проглатывает» ошибку → баги исчезают бесследно. Минимум логируйте и пробрасывайте.
- Логирование в `catch` с последующим `throw new` без inner → теряется контекст. Сохраняйте оригинал в `innerException`.
- Проброс исключения между потоками обычным `throw ex` → сброс стека. Применяйте `ExceptionDispatchInfo`.
- Логирование внутри `catch` без фильтра меняет поток управления. Используйте `when (Log(ex))` для «мягкого» наблюдения.

- `throw ex;` instead of `throw;` → the original origin is lost. Use `throw;` for a clean rethrow.
- Wrapping without `innerException`: `throw new MyEx("msg")` instead of `throw new MyEx("msg", ex)` → the chain breaks. Always pass the original as the second argument.
- Empty `catch { }` swallows the error → bugs vanish without a trace. At least log and rethrow.
- Logging inside `catch` followed by `throw new` without inner → context is lost. Keep the original in `innerException`.
- Propagating across threads with a plain `throw ex` → stack reset. Use `ExceptionDispatchInfo`.
- Logging inside `catch` without a filter alters control flow. Use `when (Log(ex))` for “soft” observation.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Везде в `catch`, где только пробрасываю, использую `throw;`, а не `throw ex;`.
- [ ] При оборачивании всегда передаю оригинал через `innerException`.
- [ ] Не оставляю пустых `catch { }` и `catch (Exception) { }`.
- [ ] Логирую до проброса, а не вместо него.
- [ ] Знаю, когда применять `ExceptionDispatchInfo.Capture(ex).Throw()`.
- [ ] Понимаю, что фильтры `when (...)` не сбрасывают стек.
- [ ] Могу объяснить разницу трассировок для `throw;`, `throw ex;` и `throw new` тремя предложениями.

- [ ] Every `catch` that only rethrows uses `throw;`, not `throw ex;`.
- [ ] When wrapping, I always pass the original via `innerException`.
- [ ] No empty `catch { }` or `catch (Exception) { }` remain.
- [ ] I log before rethrowing, not instead of it.
- [ ] I know when to use `ExceptionDispatchInfo.Capture(ex).Throw()`.
- [ ] I understand that `when (...)` filters do not reset the stack.
- [ ] I can explain the trace difference for `throw;`, `throw ex;`, and `throw new` in three sentences.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/throw]
- [ExceptionDispatchInfo — https://learn.microsoft.com/dotnet/api/system.runtime.exceptionservices.exceptiondispatchinfo]
- [Best practices for exceptions in .NET — https://learn.microsoft.com/dotnet/standard/exceptions/best-practices-for-exceptions]

---

[⬆ К модулю M07](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
