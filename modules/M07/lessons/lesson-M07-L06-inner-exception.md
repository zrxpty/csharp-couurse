[⬆ К модулю M07](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M07-L06: InnerException, цепочки / InnerException, chains

**Модуль / Module:** M07
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Когда программа растёт, код разделяется на слои: низкоуровневые операции (чтение файла, сетевой запрос, работа с БД) скрываются за высокоуровневыми сервисами (бизнес-логика, API-контроллеры). Если на нижнем уровне возникает ошибка, её «голое» сообщение часто бесполезно для верхнего слоя: контроллеру всё равно, что именно сломалось в `FileStream` — ему важно знать, что «не удалось загрузить конфигурацию». Но при этом терять оригинальную причину нельзя: без неё невозможно отладить проблему. Для решения этой задачи в .NET существует механизм `InnerException`.

`InnerException` — это свойство класса `System.Exception`, которое хранит ссылку на исходное исключение, послужившее причиной текущего. Когда вы перехватываете исключение и выбрасываете новое, более осмысленное, вы передаёте оригинал через конструктор: `throw new ConfigurationException("Не удалось загрузить конфигурацию", ex)`. Тем самым создаётся **цепочка исключений** (exception chain): каждое звено добавляет контекст, а корневая причина остаётся доступной через свойство `InnerException`. Цепочку можно пройти циклом, пока `InnerException` не станет `null` — это и будет первопричина.

**Аналогия:** представьте病历 пациента. Терапевт видит «боль в груди» (верхний слой), направляет к кардиологу — тот ставит «стенокардия» (средний слой), а ангиография показывает «закупорка артерии №3» (корневая причина). Каждое звено сохраняет предыдущее заключение, и врач на любом уровне может проследить всю историю до первоисточника. Так же и в коде: каждый слой оборачивает исключение, добавляя свою часть контекста, не затирая чужой.

**Сохранение контекста** — главный принцип правильного оборачивания. Кроме `InnerException`, важно переносить и другие данные: сообщение, стек вызовов (через конструктор, а не `throw ex`, который обнуляет трассировку), пользовательские свойства, `Data`-словарь. Существует даже метод `ExceptionDispatchInfo.Capture(ex).Throw()` из `System.Runtime.ExceptionServices`, который перебрасывает исключение, полностью сохраняя оригинальный стек — полезно при маршалинге между потоками. Никогда не выбрасывайте `throw ex` — это «зарезает» стек и прячет реальное место сбоя; используйте `throw;` для переброса или `throw new ..., ex` для оборачивания.

**Когда оборачивать, а когда — нет.** Оборачивайте, когда переходите через границу абстракции: репозиторий ловит `SqlException` и бросает `RepositoryException("Не удалось сохранить пользователя", ex)`. Не оборачивайте, если новое исключение не добавляет информации — лишние слои только мешают. Также не оборачивайте, если тип уже осмысленный (`ArgumentException`, `OperationCanceledException`) — пробросьте как есть.

**`AggregateException`** нужен, когда исключений несколько одновременно. Это происходит в параллельном коде: `Parallel.ForEach`, `Task.WhenAll`, PLINQ — каждая задача может упасть независимо, и нельзя «забыть» ни одну. `AggregateException` содержит коллекцию `InnerExceptions` (именно `InnerExceptions`, во множественном числе — свойство отличается от одиночного `InnerException`). Для распаковки используют метод `Flatten()` (схлопывает вложенные `AggregateException` в один плоский список) и `Handle(func)` — вызывает функцию для каждого внутреннего исключения; если она вернула `true`, исключение считается «обработанным», а все остальные перебрасываются новым `AggregateException`. Это позволяет элегантно фильтровать: например, обрабатывать только `IOException` и подавать сигнал о остальных.

**Антипаттерны:** выбрасывать новое исключение без `innerException` и терять корень; оборачивать `null`-ссылки в `NullReferenceException` заново (лучше бросать `ArgumentNullException(nameof(param))`); глотать исключение в `catch` без логирования; использовать `throw ex`. Правильный подход: ловите осознанно, оборачивайте с контекстом, логируйте с полной цепочкой (включая `ex.ToString()`, который автоматически печатает все `InnerException`), и пробрасывайте дальше.

#### Theory (EN)

As a program grows, code splits into layers: low-level operations (file reads, network calls, DB access) hide behind high-level services (business logic, API controllers). When an error occurs at the bottom, its bare message is often useless to the top layer: a controller doesn't care what exactly broke inside `FileStream` — it needs to know that "configuration loading failed". Yet losing the original cause is unacceptable: without it, debugging becomes guesswork. .NET solves this with the `InnerException` mechanism.

`InnerException` is a property of `System.Exception` that stores a reference to the original exception that caused the current one. When you catch an exception and throw a new, more meaningful one, you pass the original via the constructor: `throw new ConfigurationException("Failed to load configuration", ex)`. This builds an **exception chain**: each link adds context, while the root cause remains reachable through `InnerException`. You can walk the chain in a loop until `InnerException` becomes `null` — that's the root cause.

**Analogy:** think of a patient's medical record. A general practitioner sees "chest pain" (top layer), refers to a cardiologist who diagnoses "angina" (middle layer), and an angiogram reveals "artery #3 blockage" (root cause). Each link preserves the previous conclusion, so any doctor can trace the whole story back to the source. Code works the same way: each layer wraps the exception, adding its own context without erasing anyone else's.

**Context preservation** is the core principle of correct wrapping. Beyond `InnerException`, carry over other data: the message, the call stack (via the constructor, not `throw ex`, which resets the trace), custom properties, and the `Data` dictionary. There's even `ExceptionDispatchInfo.Capture(ex).Throw()` from `System.Runtime.ExceptionServices`, which rethrows while fully preserving the original stack — essential when marshaling between threads. Never write `throw ex` — it "knife-cuts" the stack and hides the real failure point; use `throw;` to rethrow or `throw new ..., ex` to wrap.

**When to wrap and when not to.** Wrap when crossing an abstraction boundary: a repository catches `SqlException` and throws `RepositoryException("Failed to save user", ex)`. Don't wrap when the new exception adds no information — extra layers only obscure. Also don't wrap when the type is already meaningful (`ArgumentException`, `OperationCanceledException`) — just let it propagate.

**`AggregateException`** is for when there are multiple exceptions at once. This happens in parallel code: `Parallel.ForEach`, `Task.WhenAll`, PLINQ — each task may fail independently, and none can be "forgotten". `AggregateException` holds a collection `InnerExceptions` (note the plural — this property differs from the singular `InnerException`). To unpack it, use `Flatten()` (collapses nested `AggregateException`s into one flat list) and `Handle(func)` — it calls the function for each inner exception; if it returns `true`, the exception is considered "handled", and the rest are rethrown as a new `AggregateException`. This elegantly filters: handle only `IOException`, for example, and signal the rest.

**Anti-patterns:** throwing a new exception without `innerException` and losing the root; re-wrapping `null`-reference errors as `NullReferenceException` (better throw `ArgumentNullException(nameof(param))`); swallowing exceptions in `catch` without logging; using `throw ex`. The right approach: catch deliberately, wrap with context, log the full chain (use `ex.ToString()`, which automatically prints all `InnerException`s), and propagate.

#### Пример кода / Code Example

```csharp
// C# 12+ / .NET 8+ — InnerException chains and AggregateException
// Двуязычные комментарии: RU + EN

using System;
using System.Collections.Generic;
using System.IO;
using System.Runtime.ExceptionServices;
using System.Threading.Tasks;

namespace M07.L06;

// Пользовательское исключение, сохраняющее контекст слоя репозитория.
// Custom exception that preserves repository-layer context.
public sealed class RepositoryException : Exception
{
    public string? EntityName { get; }   // Имя сущности / Entity name

    public RepositoryException(string message, string? entityName, Exception inner)
        : base(message, inner)            // inner → InnerException / inner → InnerException
    {
        EntityName = entityName;
    }
}

public static class ExceptionChainsDemo
{
    // Слой 1: низкоуровневое чтение файла. / Layer 1: low-level file read.
    private static string ReadFile(string path)
    {
        // FileNotFoundException будет содержать путь и реальный стек.
        // FileNotFoundException will carry the path and the real stack.
        using var stream = File.OpenRead(path);
        using var reader = new StreamReader(stream);
        return reader.ReadToEnd();
    }

    // Слой 2: репозиторий оборачивает низкоуровневую ошибку в доменную.
    // Layer 2: repository wraps a low-level error into a domain one.
    public static string LoadSettings(string path)
    {
        try
        {
            return ReadFile(path);
        }
        catch (IOException ex)
        {
            // Ключевой момент: передаём ex как innerException, не теряем корень.
            // Key point: pass ex as innerException — don't lose the root.
            throw new RepositoryException(
                "Не удалось загрузить настройки / Failed to load settings",
                entityName: "Settings",
                inner: ex);
        }
    }

    // Обход всей цепочки до первопричины. / Walk the chain to the root cause.
    public static IEnumerable<Exception> EnumerateChain(Exception? ex)
    {
        while (ex is not null)
        {
            yield return ex;
            ex = ex.InnerException;     // следующее звено / next link
        }
    }

    // Параллельная обработка с возможностью нескольких одновременных ошибок.
    // Parallel processing that may produce several simultaneous errors.
    public static async Task ProcessAllAsync(IReadOnlyList<string> paths)
    {
        var tasks = new List<Task>();
        foreach (var p in paths)
            tasks.Add(Task.Run(() => LoadSettings(p)));

        try
        {
            await Task.WhenAll(tasks);   // все задачи стартуют / all tasks start
        }
        catch (Exception first)
        {
            // Task.WhenAll выбрасывает ПЕРВОЕ исключение, но остальные ждут внутри
            // AggregateException, доступном через await ... в Try-паттерне.
            // Task.WhenAll throws the FIRST exception; others wait inside an
            // AggregateException accessible via Task.WhenAll's pattern.
            //
            // Чтобы получить ВСЕ ошибки, оборачиваем в Task.WhenAll через
            // Task.WhenAny-цикл ИЛИ используем метод расширения ниже.
            // To get ALL errors, use the extension below.
            Console.WriteLine($"Первая ошибка / First error: {first.Message}");
            throw;
        }
    }

    // Полный перехват всех ошибок параллельных задач через AggregateException.
    // Full capture of all parallel-task errors via AggregateException.
    public static async Task<Exception[]> ProcessAllCollectAsync(
        IReadOnlyList<string> paths)
    {
        var tasks = new List<Task>();
        foreach (var p in paths)
            tasks.Add(Task.Run(() => LoadSettings(p)));

        // Ждём все задачи, не выбрасывая сразу; собираем AggregateException.
        // Wait for all without throwing immediately; collect AggregateException.
        await Task.WhenAll(tasks).ContinueWith(
            _ => { }, TaskContinuationOptions.ExecuteSynchronously);

        var errors = new List<Exception>();
        foreach (var t in tasks)
        {
            if (t.IsFaulted && t.Exception is AggregateException agg)
            {
                // Flatten схлопывает вложенные AggregateException в плоский список.
                // Flatten collapses nested AggregateException into a flat list.
                foreach (var inner in agg.Flatten().InnerExceptions)
                    errors.Add(inner);
            }
        }
        return errors.ToArray();
    }

    // Переброс исключения с сохранением оригинального стека через границу потока.
    // Rethrow preserving the original stack across a thread boundary.
    public static void RethrowWithOriginalStack(Exception ex)
    {
        ExceptionDispatchInfo.Capture(ex).Throw();   // стек сохранён / stack preserved
    }

    public static void Run()
    {
        try
        {
            _ = LoadSettings("non-existent-config.json");
        }
        catch (RepositoryException ex)
        {
            Console.WriteLine($"Внешнее / Outer: {ex.Message}");
            Console.WriteLine($"Сущность / Entity: {ex.EntityName}");
            Console.WriteLine("Цепочка / Chain:");
            foreach (var link in EnumerateChain(ex))
                Console.WriteLine($"  - {link.GetType().Name}: {link.Message}");

            // ToString() автоматически печатает всю цепочку с трассировками.
            // ToString() automatically prints the whole chain with stack traces.
            Console.WriteLine("\nПолный дамп / Full dump:");
            Console.WriteLine(ex);
        }

        // Демонстрация AggregateException.Handle — фильтрация по типу.
        // Demonstration of AggregateException.Handle — filtering by type.
        var aggregate = new AggregateException(
            new IOException("Сеть / Network down"),
            new UnauthorizedAccessException("Доступ запрещён / Access denied"),
            new IOException("Таймаут / Timeout"));

        try
        {
            // Handle перебрасывает всё, что НЕ вернуло true.
            // Handle rethrows everything that did NOT return true.
            aggregate.Handle(ex => ex is IOException);
        }
        catch (AggregateException remaining)
        {
            Console.WriteLine("\nНеобработанные / Unhandled:");
            foreach (var e in remaining.InnerExceptions)
                Console.WriteLine($"  - {e.GetType().Name}: {e.Message}");
        }
    }
}
```

#### Best Practices

- Оборачивайте исключение только при переходе границы абстракции, передавая оригинал через конструктор как `innerException`.
- Логируйте через `ex.ToString()` — он автоматически разворачивает всю цепочку `InnerException` вместе со стеками.
- Используйте `ExceptionDispatchInfo.Capture(ex).Throw()` для переброса между потоками с сохранением стека.
- В параллельном коде собирайте все ошибки через `AggregateException.Flatten()` и `InnerExceptions`, не ограничивайтесь первой.
- Применяйте `AggregateException.Handle` для декларативной фильтрации: вернули `true` — обработано, остальное перебрасывается.
- Включайте в пользовательское исключение контекстные свойства (имя сущности, операция, идентификатор), а не только сообщение.

- Wrap an exception only when crossing an abstraction boundary, passing the original through the constructor as `innerException`.
- Log via `ex.ToString()` — it automatically expands the full `InnerException` chain with stack traces.
- Use `ExceptionDispatchInfo.Capture(ex).Throw()` to rethrow across threads while preserving the stack.
- In parallel code, collect all errors via `AggregateException.Flatten()` and `InnerExceptions` — don't stop at the first one.
- Use `AggregateException.Handle` for declarative filtering: return `true` to mark handled, the rest is rethrown.
- Include contextual properties (entity name, operation, identifier) in custom exceptions, not just a message.

#### Частые ошибки / Common Mistakes

- `throw ex;` вместо `throw;` → обнуляет стек вызовов, прячет реальное место ошибки. Используйте `throw;` для переброса или `throw new ...(msg, ex)` для оборачивания.
- Выбрасывание нового исключения без `innerException` → теряется первопричина, отладка превращается в угадывание. Всегда передавайте `ex` в конструктор.
- Оборачивание ради оборачивания (например, `try { ... } catch (Exception ex) { throw new Exception("error", ex); }`) → затирает тип и добавляет шум. Оборачивайте только осмысленным типом и только на границе слоёв.
- Глотание в `catch { }` или `catch (Exception) { /* nothing */ }` → ошибка исчезает бесследно. Как минимум логируйте.
- Игнорирование `AggregateException.InnerExceptions` в `Task.WhenAll` и реакция только на первую ошибку → остальные сбои скрыты. Используйте `Flatten()` и полный обход.
- Логирование только `ex.Message` без `ex.ToString()` → теряются стек и вся цепочка `InnerException`. Печатайте полное `ToString()`.

- `throw ex;` instead of `throw;` → resets the call stack, hides the real failure location. Use `throw;` to rethrow or `throw new ...(msg, ex)` to wrap.
- Throwing a new exception without `innerException` → loses the root cause, turning debugging into guesswork. Always pass `ex` to the constructor.
- Wrapping for the sake of wrapping (e.g. `try { ... } catch (Exception ex) { throw new Exception("error", ex); }`) → erases the type and adds noise. Wrap only with a meaningful type and only at layer boundaries.
- Swallowing in `catch { }` or `catch (Exception) { /* nothing */ }` → the error vanishes silently. At the very least, log it.
- Ignoring `AggregateException.InnerExceptions` in `Task.WhenAll` and reacting only to the first error → the remaining failures stay hidden. Use `Flatten()` and iterate fully.
- Logging only `ex.Message` without `ex.ToString()` → loses the stack and the whole `InnerException` chain. Print the full `ToString()`.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] При оборачивании я передаю оригинальное исключение в конструктор как `innerException`.
- [ ] Я никогда не использую `throw ex;` — только `throw;` или `throw new ...(msg, ex)`.
- [ ] Моё пользовательское исключение содержит контекстные свойства, а не только сообщение.
- [ ] В параллельном коде я собираю все ошибки через `AggregateException.Flatten().InnerExceptions`.
- [ ] Я использую `AggregateException.Handle` для декларативной фильтрации по типу.
- [ ] Логирование делаю через `ex.ToString()`, чтобы видеть всю цепочку и стеки.
- [ ] Для переброса между потоками применяю `ExceptionDispatchInfo.Capture(ex).Throw()`.
- [ ] Я не оборачиваю исключение, если новое не добавляет контекста (не создаю шум).

- [ ] When wrapping, I pass the original exception to the constructor as `innerException`.
- [ ] I never use `throw ex;` — only `throw;` or `throw new ...(msg, ex)`.
- [ ] My custom exception carries contextual properties, not just a message.
- [ ] In parallel code, I collect all errors via `AggregateException.Flatten().InnerExceptions`.
- [ ] I use `AggregateException.Handle` for declarative type-based filtering.
- [ ] I log via `ex.ToString()` so the full chain and stacks are visible.
- [ ] For cross-thread rethrow I use `ExceptionDispatchInfo.Capture(ex).Throw()`.
- [ ] I don't wrap an exception when the new one adds no context (no noise).

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/api/system.exception.innerexception](https://learn.microsoft.com/dotnet/api/system.exception.innerexception)
- [Microsoft Learn — AggregateException — https://learn.microsoft.com/dotnet/api/system.aggregateexception](https://learn.microsoft.com/dotnet/api/system.aggregateexception)
- [Microsoft Learn — ExceptionDispatchInfo — https://learn.microsoft.com/dotnet/api/system.runtime.exceptionservices.exceptiondispatchinfo](https://learn.microsoft.com/dotnet/api/system.runtime.exceptionservices.exceptiondispatchinfo)

---

[⬆ К модулю M07](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
