[⬆ К модулю M07](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M07-L07: Result<T>/OneOf (Optional), когда не использовать исключения / Result<T>/OneOf (Optional), when not to use exceptions

**Модуль / Module:** M07
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Исключения в C# — мощный механизм, но у него есть цена. Когда метод бросает `Exception`, среда собирает стек вызовов, разворачивает его и ищет подходящий `catch`, что в десятки и сотни раз дороже обычного возврата значения. Поэтому простое правило: **исключения — для исключительных ситуаций**, а для ожидаемых, штатных исходов операции лучше использовать паттерн `Result<T>` или тип `OneOf`.

Представьте банкомат. Отказ выдать деньги из-за недостатка баланса — это не авария, а один из нормальных результатов операции. Бросать `InsufficientFundsException` здесь так же странно, как если бы светофор при красном свете взрывался вместо того, чтобы просто показывать красный. Ожидаемые ошибки — невалидный ввод пользователя, отсутствие записи в базе, нарушение бизнес-правила — должны возвращаться как данные, а не как исключения.

**Паттерн Result<T>** моделирует операцию, которая может либо успешно вернуть значение `T`, либо завершиться с описанной ошибкой. Минимальная реализация — запись с флагом `IsSuccess`, значением `Value` и описанием ошибки `Error`. Вариант `Result<T, TError>` обобщает тип ошибки. В C# 12 удобно использовать записи с `record` и `init`-свойствами, а также шаблоны `switch` для разбора результата.

**OneOf** — это discriminated union из библиотеки `OneOf` (McIntyre). Тип `OneOf<string, NotFound, InvalidInput>` говорит: «результат — один из трёх вариантов». Доступ к значению идёт через `.Match(...)` или `.Switch(...)`, что заставляет разработчика обработать все ветки. Это функциональный подход, пришедший из F#, Rust (`Result`) и Haskell (`Either`).

Когда же действительно нужны исключения? Их место — там, где что-то **по-настоящему сломалось** и продолжать нельзя: `StackOverflowException`, `OutOfMemoryException`, разрыв соединения с критическим сервисом, нарушение инварианта программы. Также исключения уместны в инфраструктурном коде (доступ к БД через EF Core, HTTP-запросы на верхнем уровне), где сбой сети — действительно исключительная ситуация, а не ожидаемый бизнес-сценарий.

**Производительность.** На бенчмарках бросание исключения стоит ~1–3 мкс и аллоцирует килобайты памяти под стек, тогда как возврат `Result<T>` — несколько наносекунд и ноль аллокаций при использовании `struct`. В горячих путях (парсинг миллионов строк, валидация запросов) переход с исключений на `Result<T>` даёт ускорение в 10–100 раз.

**Функциональный стиль.** `Result<T>` хорошо сочетается с LINQ-подобной монадической цепочкой: методы `Map`, `Bind` (он же `SelectMany`) позволяют композировать операции без вложенных `if`. Если шаг失败了 — цепочка коротко замыкается и возвращает ошибку. Это делает код линейным и тестируемым.

В C# 12 появились первичные конструкторы классов, что упрощает создание собственных `Result`-типов: можно объявить `public class Result<T>(bool IsSuccess, T? Value, Error? Error)` без лишнего шаблона. Также полезен `Nullable<T>` как упрощённый Optional для справочных ссылочных типов через `#nullable enable`.

Итог: используйте исключения для аварий, `Result<T>` / `OneOf` — для ожидаемых исходов. Это разделяет «сломалось» и «не получилось», ускоряет код и делает контракт метода честным: из сигнатуры видно, что операция может завершиться неудачно.

#### Theory (EN)

Exceptions in C# are powerful, but they carry a cost. When a method throws, the runtime collects a stack trace, unwinds the stack, and searches for a matching `catch` block — operations that are tens to hundreds of times more expensive than simply returning a value. The simple rule: **exceptions are for exceptional situations**, and for expected, ordinary outcomes of an operation you should use the `Result<T>` pattern or the `OneOf` type.

Think of an ATM. Refusing to dispense cash because the balance is too low is not a crash — it is one of the normal outcomes of the operation. Throwing `InsufficientFundsException` here is as strange as a traffic light exploding on red instead of just showing red. Expected failures — invalid user input, a missing database row, a violated business rule — should come back as data, not as exceptions.

The **Result<T> pattern** models an operation that can either successfully return a value `T` or complete with a described error. The minimal implementation is a record with an `IsSuccess` flag, a `Value`, and an `Error` description. The `Result<T, TError>` variant generalizes the error type. In C# 12 it is convenient to use `record` types with `init` properties and `switch` patterns to deconstruct the result.

**OneOf** is a discriminated union from the `OneOf` library (by McIntyre). The type `OneOf<string, NotFound, InvalidInput>` says "the result is one of three variants." Access to the value goes through `.Match(...)` or `.Switch(...)`, which forces the developer to handle every branch. This is a functional approach that comes from F#, Rust (`Result`), and Haskell (`Either`).

When are exceptions actually warranted? Their place is where something is **genuinely broken** and you cannot continue: `StackOverflowException`, `OutOfMemoryException`, a severed connection to a critical service, a violated program invariant. Exceptions are also reasonable in infrastructure code (database access via EF Core, top-level HTTP calls) where a network failure is truly exceptional rather than an expected business scenario.

**Performance.** Benchmarks show that throwing an exception costs roughly 1–3 µs and allocates kilobytes of memory for the stack, while returning a `Result<T>` takes a few nanoseconds and zero allocations when implemented as a `struct`. In hot paths (parsing millions of lines, request validation) moving from exceptions to `Result<T>` yields a 10–100× speedup.

**Functional style.** `Result<T>` composes well with a LINQ-like monadic chain: `Map` and `Bind` (also known as `SelectMany`) let you sequence operations without nested `if` checks. If a step fails, the chain short-circuits and returns the error. This keeps the code linear and testable.

C# 12 introduces primary constructors for classes, which simplifies building custom `Result` types: you can declare `public class Result<T>(bool IsSuccess, T? Value, Error? Error)` without extra boilerplate. `Nullable<T>` is also useful as a simplified Optional for reference types through `#nullable enable`.

In summary: use exceptions for crashes and `Result<T>` / `OneOf` for expected outcomes. This separates "broken" from "did not succeed," speeds up the code, and makes the method contract honest — the signature shows that the operation can fail.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — паттерн Result<T> и OneOf
// C# 12 / .NET 8 — Result<T> pattern and OneOf

using OneOf;

namespace M07L07;

// 1) Минимальный Result<T> на record с первичным конструктором
// 1) Minimal Result<T> on a record with a primary constructor
public record Result<T>(bool IsSuccess, T? Value, Error? Error)
{
    public static Result<T> Ok(T value) => new(true, value, null);
    public static Result<T> Fail(Error error) => new(false, default, error);

    // Bind (монадическая композиция) / Bind (monadic composition)
    public Result<TNext> Bind<TNext>(Func<T, Result<TNext>> next) =>
        IsSuccess ? next(Value!) : Result<TNext>.Fail(Error!);
}

public record Error(string Code, string Message);

// 2) Доменная модель — снятие денег / Domain model — withdraw money
public class Account
{
    public decimal Balance { get; private set; }
    public Account(decimal initial) => Balance = initial;

    // Ожидаемая ошибка — НЕ исключение / Expected error — NOT an exception
    public Result<decimal> Withdraw(decimal amount)
    {
        if (amount <= 0)
            return Result<decimal>.Fail(new("INVALID_AMOUNT", "Сумма должна быть положительной / Amount must be positive"));

        if (amount > Balance)
            return Result<decimal>.Fail(new("INSUFFICIENT_FUNDS", "Недостаточно средств / Insufficient funds"));

        Balance -= amount;
        return Result<decimal>.Ok(amount);
    }
}

// 3) Использование через switch-паттерн / Usage via switch pattern
public static class Demo
{
    public static string Run(Account account, decimal amount)
    {
        var result = account.Withdraw(amount);
        return result switch
        {
            { IsSuccess: true } => $"Выдано / Dispensed: {result.Value}",
            { IsSuccess: false, Error: var e } => $"Ошибка [{e.Code}]: {e.Message}"
        };
    }
}

// 4) OneOf — forcing exhaustive handling / OneOf — принудительная полная обработка
public class NotFound;
public class InvalidInput;

public static OneOf<string, NotFound, InvalidInput> FindUser(int id) =>
    id switch
    {
        <= 0 => new InvalidInput(),
        42 => "Alice",
        _ => new NotFound()
    };

public static string Handle(int id) =>
    FindUser(id).Match(
        name => $"Найден / Found: {name}",
        _ => "Не найдено / Not found",
        _ => "Некорректный ввод / Invalid input"
    );
```

#### Best Practices

- Возвращайте `Result<T>` для ожидаемых бизнес-ошибок (валидация, «не найдено», нарушение правил); исключения оставьте для аварий инфраструктуры.
- Делайте тип ошибки явным: `Result<T, TError>` или `OneOf<TSuccess, TError1, TError2>` — это заставляет обработать все ветки.
- Используйте `switch`-паттерны C# 12 для исчерпывающей обработки результатов вместо каскадов `if`.
- В горячих путях реализуйте `Result` как `readonly struct`, чтобы избежать аллокаций.
- Не проглатывайте ошибку: либо обрабатывайте, либо пробрасывайте через `Bind`/`Match` вверх по стеку.

- Return `Result<T>` for expected business errors (validation, "not found", rule violations); keep exceptions for infrastructure crashes.
- Make the error type explicit: `Result<T, TError>` or `OneOf<TSuccess, TError1, TError2>` — it forces every branch to be handled.
- Use C# 12 `switch` patterns for exhaustive result handling instead of cascading `if`.
- In hot paths implement `Result` as a `readonly struct` to avoid allocations.
- Never swallow an error: either handle it or propagate it up the stack via `Bind`/`Match`.

#### Частые ошибки / Common Mistakes

- [Бросание `ArgumentException` в бизнес-логике] → [Верните `Result<T>` с понятным кодом ошибки; исключение здесь скрывает ожидаемый исход]
- [Возврат `null` вместо явного результата] → [Используйте `Result<T>` или `OneOf`, чтобы контракт говорил о возможности неудачи]
- [Игнорирование `Error` в `Result.Fail`] → [Всегда передавайте осмысленные `Code` и `Message`, иначе потребитель не сможет реагировать]
- [Глубокая вложенность `if (result.IsSuccess)`] → [Применяйте `Bind`/`Match`/`switch` для линейного кода]

- [Throwing `ArgumentException` in business logic] → [Return a `Result<T>` with a clear error code; an exception here hides an expected outcome]
- [Returning `null` instead of an explicit result] → [Use `Result<T>` or `OneOf` so the contract declares the possibility of failure]
- [Ignoring `Error` in `Result.Fail`] → [Always pass meaningful `Code` and `Message`, otherwise the caller cannot react]
- [Deep nesting of `if (result.IsSuccess)`] → [Use `Bind`/`Match`/`switch` for linear code]

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я различаю исключительные сбои (аварии) и ожидаемые ошибки (бизнес-исходы)
- [ ] Мой `Result<T>` имеет явный тип ошибки и осмысленные коды
- [ ] Я обрабатываю результат через `switch`/`Match`, а не через вложенные `if`
- [ ] В горячих путях используется `struct`-реализация без аллокаций
- [ ] Я не бросаю исключения для «не найдено» / «невалидный ввод»

- [ ] I distinguish exceptional failures (crashes) from expected errors (business outcomes)
- [ ] My `Result<T>` has an explicit error type and meaningful codes
- [ ] I handle the result through `switch`/`Match` rather than nested `if`
- [ ] Hot paths use a `struct` implementation without allocations
- [ ] I do not throw exceptions for "not found" / "invalid input"

#### Ресурсы / Resources

- [Microsoft Learn — C# 12 — https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-12]
- [OneOf library (NuGet) — https://github.com/mcintyre321/OneOf]

---

[⬆ К модулю M07](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
