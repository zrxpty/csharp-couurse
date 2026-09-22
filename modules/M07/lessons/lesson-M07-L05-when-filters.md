[⬆ К модулю M07](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M07-L05: when-фильтры / when filters

**Модуль / Module:** M07
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Конструкция `catch ... when (...)` появилась в C# 6 и даёт возможность фильтровать исключения по условию **до** того, как блок `catch` начнёт выполняться. На первый взгляд это похоже на привычную связку «поймал — проверил — бросил заново», но семантика принципиально иная. Если выражение в `when` возвращает `false`, среда CLR ведёт себя так, будто этого `catch` вообще не существовало: продолжается штатский процесс поиска подходящего обработчика выше по стеку вызовов. Важно понимать, что блок `catch` при этом **не входит** — его локальные переменные не инициализируются, а сам блок не появляется в логах трассировки как «выполненный».

Представьте себе охранника на входе в комнату. Обычный `catch (Exception ex)` — это дверь без проверки: все заходят, а потом охранник решает, выпускать их обратно (через `throw;`). Использование `when` — это турникет с условием: неподходящие гости даже не заходят, а сразу идут дальше по коридору искать другую комнату. Второй сценарий честнее: трассировка стека остаётся «нетронутой», потому что CLR формально не входила в обработчик.

Главные применения фильтров:

1. **Фильтрация по типу и состоянию.** Например, ловить `HttpRequestException`, но только если статус-код 5xx. Один и тот же тип исключения можно обработать в разных блоках с разными условиями — порядок `catch` теперь не диктует логику.

2. **Безопасное логирование без «проглатывания».** В `when` можно вызвать метод с побочным эффектом (например, запись в лог), который вернёт `false`. Исключение залогируется, но продолжит всплывать к внешнему обработчику. Это устраняет антипаттерн «поймал, записал, бросил заново», который искажает стек и добавляет лишний шум.

3. **Отладочные точки останова.** Условие `when (Debugger.IsAttached && someCondition)` позволяет поставить фильтр только под отладчиком.

Семантика, о которой часто забывают:

- Если **внутри** выражения `when` само выбрасывается исключение, оно **не** ловится этим же `catch`. CLR считает, что фильтр не сработал, и исключение из фильтра всплывает как отдельная ошибка. Поэтому в `when` пишут только детерминированные проверки.
- Выражение `when` вычисляется **до** входа в `catch`, поэтому обращение к `ex` доступно, но изменять состояние в фильтре следует осторожно — побочные эффекты затрудняют рассуждения о программе.
- Фильтры работают и с обобщёнными `catch (Exception)`, и с типизированными. Несколько блоков `catch` с одинаковым типом, но разными `when`, компилируются без ошибок — выбирается первый подходящий.
- Ключевое слово `when` также применяется в `switch` (как `case ... when ...`) и в шаблонах (`is Foo f when ...`), но в этом уроке речь именно об обработке исключений.

Фильтры делают обработку ошибок декларативной: вы описываете **условия**, при которых обработчик активен, а не императивную последовательность проверок. Это повышает читаемость и уменьшает вероятность случайного «проглатывания» чужого исключения.

#### Theory (EN)

The `catch ... when (...)` construct, introduced in C# 6, lets you filter exceptions by a condition **before** the `catch` block is entered. At first glance it looks like the familiar “catch, check, rethrow” trio, but the semantics are fundamentally different. When the expression in `when` returns `false`, the CLR behaves as if that `catch` clause did not exist at all: the normal search for a matching handler continues up the call stack. The `catch` block is never entered — its locals are not initialized, and it does not show up as an executed handler in the stack trace.

Think of a guard at a doorway. A plain `catch (Exception ex)` is a door without a check: everyone enters, and the guard later decides whether to push them back out (via `throw;`). Using `when` is a turnstile with a rule: unqualified visitors never enter the room — they are routed straight down the corridor to find another door. The second scenario is cleaner: the stack trace stays intact because the CLR never formally entered the handler.

The main uses of exception filters:

1. **Filtering by type and state.** For example, catch `HttpRequestException`, but only when the status code is 5xx. The same exception type can be handled by multiple blocks with different conditions — the textual order of `catch` clauses no longer dictates the logic.

2. **Safe logging without swallowing.** You can call a side-effecting method (say, a logging call) inside `when` and have it return `false`. The exception gets logged, but it keeps propagating to the outer handler. This eliminates the “catch, log, rethrow” anti-pattern, which distorts the stack and adds noise.

3. **Debug-only breakpoints.** A condition like `when (Debugger.IsAttached && someCondition)` activates a filter only under the debugger.

Semantics people often miss:

- If the expression inside `when` itself throws, that exception is **not** caught by the same `catch`. The CLR treats the filter as not applicable, and the filter’s own exception propagates as a separate error. That is why `when` should only contain deterministic checks.
- The `when` expression is evaluated **before** entry into `catch`, so `ex` is available, but mutating state in a filter should be done carefully — side effects make the program harder to reason about.
- Filters work with both generic `catch (Exception)` and typed clauses. Multiple `catch` blocks with the same type but different `when` conditions compile cleanly — the first matching one wins.
- The `when` keyword is also used in `switch` (as `case ... when ...`) and in patterns (`is Foo f when ...`), but this lesson focuses on exception handling.

Filters make error handling declarative: you describe the **conditions** under which a handler is active, rather than an imperative sequence of checks. This improves readability and reduces the chance of accidentally swallowing someone else’s exception.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — фильтры исключений через when
// C# 12 / .NET 8 — exception filters with when

using System.Net;
using System.Diagnostics;

namespace M07L05.WhenFilters;

public static class WhenFilterDemo
{
    // Безопасное логирование без проглатывания исключения.
    // Safe logging that does NOT swallow the exception.
    public static bool LogAndReturnFalse(Exception ex, string context)
    {
        // Побочный эффект — запись в лог, затем возврат false,
        // Side effect — write to log, then return false,
        // чтобы фильтр не сработал и исключение пошло выше.
        // so the filter does not match and the exception propagates.
        Console.WriteLine($"[LOG] {context}: {ex.GetType().Name}: {ex.Message}");
        return false;
    }

    public static async Task<string> FetchAsync(HttpClient client, string url)
    {
        try
        {
            // Может выбросить HttpRequestException с разными статус-кодами.
            // May throw HttpRequestException with various status codes.
            return await client.GetStringAsync(url);
        }
        // Ловим только серверные ошибки 5xx — клиент остаётся без обработки здесь.
        // Catch only server errors 5xx — others stay unhandled at this level.
        catch (HttpRequestException ex) when (ex.StatusCode is >= HttpStatusCode.InternalServerError)
        {
            Console.WriteLine("Серверная ошибка — повторим позднее / Server error — will retry later");
            throw; // повторная попытка делегируется вызывающему / retry is delegated to the caller
        }
        // Логируем 4xx, но НЕ обрабатываем — исключение всплывает к глобальному обработчику.
        // Log 4xx but do NOT handle — exception bubbles to the global handler.
        catch (HttpRequestException ex) when (
            LogAndReturnFalse(ex, "client-error") &&
            ex.StatusCode is >= HttpStatusCode.BadRequest and < HttpStatusCode.InternalServerError)
        {
            // Этот блок никогда не выполнится: фильтр всегда возвращает false.
            // This block never runs: the filter always returns false.
            throw;
        }
        // Глобальный «фильтр отладчика»: ломаемся только под отладчиком.
        // Debug-only filter: break only when the debugger is attached.
        catch (Exception ex) when (Debugger.IsAttached && LogAndReturnFalse(ex, "debug-inspect"))
        {
            throw;
        }
    }
}
```

#### Best Practices
- Используйте `when` для фильтрации по **состоянию** исключения (статус-код, HResult, вложенное сообщение), а не только по типу.
- Выносите логирование в отдельный метод, который возвращает `false`, чтобы исключение продолжало всплывать.
- Держите выражение в `when` чистым и детерминированным: никаких бросаний, никаких тяжелых операций.
- Предпочитайте `when` связке «catch + if + throw», чтобы сохранить честную трассировку стека.
- Use `when` to filter by the exception’s **state** (status code, HResult, inner message), not just by type.
- Extract logging into a helper that returns `false`, so the exception keeps propagating.
- Keep the `when` expression pure and deterministic: no throws, no heavy work.
- Prefer `when` over “catch + if + throw” to preserve an honest stack trace.

#### Частые ошибки / Common Mistakes
- Логирование внутри `catch` с последующим `throw;` вместо фильтра → искажённая трассировка и лишний шум; используйте `when (LogAndReturnFalse(...))`.
- Выбрасывание исключения внутри `when` → оно не ловится этим же `catch` и ломает фильтрацию; держите `when` без бросаний.
- Использование `when` для изменяемых побочных эффектов (мутация полей) → сложно рассуждать о порядке; только чтение и безопасное логирование.
- Несколько `catch` с одним типом без `when` → ошибка компиляции; добавьте разные условия `when`.
- Logging inside `catch` followed by `throw;` instead of a filter → distorted stack trace and noise; use `when (LogAndReturnFalse(...))`.
- Throwing inside `when` → it is not caught by the same `catch` and breaks filtering; keep `when` throw-free.
- Using `when` for mutating side effects (field writes) → hard to reason about order; read-only checks and safe logging only.
- Multiple `catch` clauses of the same type without `when` → compile error; add distinct `when` conditions.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Я понимаю, что `when (false)` означает «этот catch не активен», а не «войти и ничего не сделать».
- [ ] Я использую фильтр для логирования без проглатывания через метод, возвращающий `false`.
- [ ] В выражении `when` нет бросаний исключений и тяжёлых операций.
- [ ] Я фильтрую исключения по состоянию (статус-код, HResult), а не только по типу.
- [ ] Я знаю, что порядок `catch` с одинаковым типом разрешается через `when`, и первый подходящий выигрывает.
- [ ] I understand that `when (false)` means “this catch is inactive”, not “enter and do nothing”.
- [ ] I use a filter for non-swallowing logging through a method that returns `false`.
- [ ] My `when` expression contains no throws and no heavy work.
- [ ] I filter exceptions by state (status code, HResult), not only by type.
- [ ] I know that the order of same-type `catch` clauses is resolved via `when`, and the first match wins.

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/when](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/when)

---

[⬆ К модулю M07](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
