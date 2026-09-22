[⬆ К модулю M07](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M07-L02: try/catch/finally, порядок catch / try/catch/finally, catch order

**Модуль / Module:** M07
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Конструкция `try/catch/finally` — это основной механизм обработки исключений в C#. Она позволяет выделить опасный код, перехватить возможные сбои и гарантированно выполнить завершающие действия. Представьте себе кухню ресторана: повар готовит блюдо в `try` (пытается), если что-то загорелось — срабатывает `catch` (тушим пожар определённого типа), а уборка стола происходит в `finally` всегда, независимо от того, был ли пожар.

Блок `try` содержит код, который может выбросить исключение. Блок `catch` перехватывает исключения определённого типа. Блок `finally` выполняется всегда — и при нормальном завершении, и при исключении, и даже если внутри `try` использован `return`. Это критически важно: `finally` — единственное место, где можно гарантированно освободить ресурсы, закрыть файлы, соединения с базой данных, освободить блокировки.

**Порядок catch — от конкретных к общим.** Это главное правило. Исключения в C# образуют иерархию: `FileNotFoundException` наследуется от `IOException`, который наследуется от `SystemException`, который наследуется от `Exception`. Компилятор C# требует, чтобы более конкретные обработчики шли раньше более общих. Если поставить `catch (Exception)` первым, он «проглотит» все исключения, и последующие `catch` для специализированных типов станут недостижимыми — компилятор выдаст ошибку CS1057 или предупреждение о недостижимом коде.

Логика проста: сначала обрабатываем то, что знаем как чинить точечно (например, `FormatException` при разборе числа — показываем пользователю «введите число»), затем — более общие категории (`IOException` — логируем и сообщаем о проблеме ввода-вывода), и только в самом конце, при необходимости, — универсальный `catch (Exception)` как страховка.

**Вложенные try.** Конструкции `try` можно вкладывать друг в друга. Внутренний `try` обрабатывает локальные ошибки (например, разбор конкретной строки из файла), а внешний — глобальные (например, недоступность всего файла). Это полезно для частичного восстановления: при ошибке обработки одной записи можно продолжить обработку остальных. Альтернатива — один `try` с несколькими `catch`, если все ошибки обрабатываются на одном уровне.

Важный нюанс: если исключение не перехвачено ни одним `catch` текущего `try`, оно «всплывает» вверх по стеку вызовов, и поиск подходящего обработчика продолжается во внешних блоках. `finally` при этом всё равно выполняется перед всплытием.

**Когда использовать `finally`.** Начиная с C# 8, предпочтительнее конструкция `using` (включая `using`-объявления), которая автоматически вызывает `Dispose()`. Но `finally` незаменим, когда нужно закрыть не реализующие `IDisposable` ресурсы, сбросить флаги, вернуть объект в пул, или когда логика завершения сложнее простого освобождения. Также `finally` обязательно выполняется даже при `return` внутри `try` или `catch` — это отличный способ гарантировать состояние.

**Фильтры исключений (C# 6+).** Конструкция `catch (Exception ex) when (условие)` позволяет фильтровать исключения по содержимому, не перехватывая их (фильтр возвращает `false` — исключение продолжает всплывать, при этом стек не разматывается, что полезно для отладки).

#### Theory (EN)

The `try/catch/finally` construct is the primary exception-handling mechanism in C#. It lets you isolate risky code, intercept possible failures, and guarantee cleanup actions. Imagine a restaurant kitchen: the chef cooks a dish inside `try` (attempts work), if something catches fire `catch` triggers (we extinguish a fire of a specific type), and table cleanup happens in `finally` — always, whether or not there was a fire.

The `try` block holds code that may throw. The `catch` block intercepts exceptions of a specific type. The `finally` block runs unconditionally — on normal completion, on exception, and even when `try` contains a `return`. This is critical: `finally` is the only place where you can reliably release resources, close files and database connections, and release locks.

**Catch order — from specific to general.** This is the cardinal rule. Exceptions in C# form a hierarchy: `FileNotFoundException` derives from `IOException`, which derives from `SystemException`, which derives from `Exception`. The C# compiler requires more specific handlers to appear before more general ones. If you place `catch (Exception)` first, it swallows every exception and subsequent specialized handlers become unreachable — the compiler reports error CS1057 or an unreachable-code warning.

The logic is straightforward: first handle what you know how to fix precisely (for example, `FormatException` while parsing a number — show the user “enter a number”), then broader categories (`IOException` — log and report an I/O problem), and only at the very end, if needed, a universal `catch (Exception)` as a safety net.

**Nested try.** `try` constructs can be nested. An inner `try` handles local errors (say, parsing one particular line from a file), while an outer one handles global errors (the whole file being unavailable). This is useful for partial recovery: when one record fails, you can keep processing the rest. The alternative is a single `try` with multiple `catch` clauses when all errors are handled at the same level.

A key subtlety: if no `catch` in the current `try` matches the exception, it propagates up the call stack, and the search for a matching handler continues in outer blocks. `finally` still runs before propagation.

**When to use `finally`.** Starting with C# 8, the `using` construct (including `using` declarations) is preferable for anything implementing `IDisposable` because it calls `Dispose()` automatically. But `finally` is irreplaceable when you must close non-`IDisposable` resources, reset flags, return an object to a pool, or when cleanup logic is more complex than simple disposal. `finally` also runs even on `return` inside `try` or `catch` — a reliable way to guarantee state.

**Exception filters (C# 6+).** The `catch (Exception ex) when (condition)` form lets you filter exceptions by content without intercepting them: when the filter returns `false`, the exception keeps propagating and the stack is not unwound, which is valuable for debugging.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — try/catch/finally, порядок catch и вложенные try
// Демонстрирует: порядок catch от конкретных к общим, finally, вложенные try, фильтры

using System;
using System.IO;

namespace M07L02;

internal static class ConfigLoader
{
    // Порядок catch: от конкретных (FileNotFoundException) к общим (Exception)
    // Catch order: from specific (FileNotFoundException) to general (Exception)
    public static string? LoadConfig(string path)
    {
        FileStream? stream = null;
        StreamReader? reader = null;
        try
        {
            stream = File.OpenRead(path);              // Может выбросить FileNotFoundException / IOException
            reader = new StreamReader(stream);         // May throw FileNotFoundException / IOException

            // Вложенный try: обрабатываем локальную ошибку парсинга строки
            // Nested try: handles a local line-parsing error
            string? line;
            while ((line = reader.ReadLine()) is not null)
            {
                try
                {
                    var parts = line.Split('=', 2);
                    if (parts.Length != 2)
                        throw new FormatException($"Строка не вида key=value / Line is not key=value: {line}");

                    Console.WriteLine($"Ключ / Key: {parts[0]} = {parts[1]}");
                }
                catch (FormatException ex)
                {
                    // Локальная ошибка — логируем и продолжаем обработку остальных строк
                    // Local error — log and continue processing remaining lines
                    Console.WriteLine($"Пропуск строки / Skipping line: {ex.Message}");
                }
            }

            return "OK";
        }
        catch (FileNotFoundException ex) when (string.IsNullOrEmpty(ex.FileName) is false)
        {
            // Самый конкретный обработчик — файл не найден и имя известно
            // Most specific handler — file not found and name is known
            Console.WriteLine($"Файл не найден / File not found: {ex.FileName}");
            return null;
        }
        catch (FileNotFoundException)
        {
            // Тот же тип, но без имени файла — менее информативный случай
            // Same type, but without a file name — a less informative case
            Console.WriteLine("Файл не найден (имя неизвестно) / File not found (name unknown)");
            return null;
        }
        catch (IOException ex)
        {
            // Более общий тип ввода-вывода — идёт ПОСЛЕ FileNotFoundException
            // Broader I/O type — placed AFTER FileNotFoundException
            Console.WriteLine($"Ошибка ввода-вывода / I/O error: {ex.Message}");
            return null;
        }
        catch (Exception ex)
        {
            // Универсальная страховка — всегда последняя
            // Universal safety net — always last
            Console.WriteLine($"Непредвиденная ошибка / Unexpected error: {ex.Message}");
            return null;
        }
        finally
        {
            // finally выполняется ВСЕГДА: при успехе, при исключении, при return
            // finally runs ALWAYS: on success, on exception, on return
            reader?.Dispose();
            stream?.Dispose();
            Console.WriteLine("Ресурсы освобождены / Resources disposed");
        }
    }
}

internal static class Program
{
    private static void Main()
    {
        // finally срабатывает даже при return внутри try
        // finally fires even when return is inside try
        _ = ConfigLoader.LoadConfig("nonexistent.cfg");
    }
}
```

#### Best Practices

- Располагайте обработчики `catch` строго от наиболее конкретных к наиболее общим; `catch (Exception)` ставьте последним и только при необходимости.
- Предпочитайте `using` (и `using`-объявления C# 8+) для освобождения `IDisposable`-ресурсов; используйте `finally` только там, где `using` не подходит.
- Не «глотайте» исключения молча: пустой `catch` или `catch (Exception) { }` скрывает баги — как минимум логируйте.
- Используйте фильтры `when (...)`, чтобы перехватывать исключения только в нужных условиях без разматывания стека.
- В `catch` предпочтитайте выброс нового исключения с сохранением оригинала через `throw new ...(..., ex)` или просто `throw;` для повторной генерации без потери стека.

- Order `catch` handlers strictly from most specific to most general; place `catch (Exception)` last and only when needed.
- Prefer `using` (and C# 8+ `using` declarations) for releasing `IDisposable` resources; reserve `finally` for cases `using` cannot cover.
- Do not silently swallow exceptions: an empty `catch` or `catch (Exception) { }` hides bugs — at minimum, log them.
- Use `when (...)` filters to intercept exceptions only under the right conditions without unwinding the stack.
- In `catch`, prefer throwing a new exception that wraps the original via `throw new ...(..., ex)`, or simply `throw;` to rethrow without losing the stack trace.

#### Частые ошибки / Common Mistakes

- `catch (Exception)` стоит первым, делая остальные `catch` недостижимыми → ставьте специфичные обработчики раньше, общие — позже.
- Пустой `catch { }` «проглатывает» ошибку без лога → как минимум записывайте в лог или повторно выбрасывайте через `throw;`.
- Использование `throw ex;` вместо `throw;` сбрасывает стек вызовов → всегда используйте `throw;` для повторной генерации.
- Полагают, что `finally` не выполнится при `return` внутри `try` → `finally` выполняется всегда, кроме `StackOverflowException` и принудительного завершения процесса.
- Освобождают ресурсы только в `try`, забывая `finally` → при исключении ресурс утекает; используйте `finally` или `using`.
- Один гигантский `catch (Exception)` для всех ошибок → разбейте на конкретные типы и обрабатывайте точечно.
- Вложенные `try` без необходимости усложняют код → вкладывайте только когда нужна частичная обработка или разный уровень восстановления.

- `catch (Exception)` is placed first, making other `catch` blocks unreachable → put specific handlers first, general ones later.
- An empty `catch { }` swallows the error with no log → at minimum log it or rethrow via `throw;`.
- Using `throw ex;` instead of `throw;` resets the call stack → always use `throw;` to rethrow.
- Assuming `finally` won’t run on a `return` inside `try` → `finally` always runs, except for `StackOverflowException` and forced process termination.
- Releasing resources only in `try`, forgetting `finally` → on exception the resource leaks; use `finally` or `using`.
- One giant `catch (Exception)` for all errors → split into specific types and handle them precisely.
- Nested `try` added unnecessarily complicates code → nest only when partial recovery or different recovery levels are needed.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] `catch`-блоки упорядочены от конкретных типов исключений к общим (`Exception` — последним).
- [ ] Ни один `catch` не «глотает» исключение молча — есть лог или `throw;`.
- [ ] Для повторной генерации используется `throw;`, а не `throw ex;`.
- [ ] `finally` присутствует там, где нужно гарантированно освободить не-`IDisposable` ресурсы или сбросить состояние.
- [ ] `using` / `using`-объявления используются для всех `IDisposable`-объектов вместо ручного `Dispose` в `finally`.
- [ ] Вложенные `try` применяются осознанно — для частичной обработки или разных уровней восстановления.
- [ ] Фильтры `when (...)` используются там, где нужно условие перехвата без потери стека.

- [ ] `catch` blocks are ordered from specific exception types to general (`Exception` last).
- [ ] No `catch` silently swallows an exception — there is a log or a `throw;`.
- [ ] `throw;` is used to rethrow, not `throw ex;`.
- [ ] `finally` is present where non-`IDisposable` resources must be released or state reset unconditionally.
- [ ] `using` / `using` declarations are used for all `IDisposable` objects instead of manual `Dispose` in `finally`.
- [ ] Nested `try` is used intentionally — for partial recovery or different recovery levels.
- [ ] `when (...)` filters are used where a conditional catch without stack loss is needed.

#### Ресурсы / Resources

- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/try-catch]

---

[⬆ К модулю M07](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
