[⬆ К модулю M07](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M07-L01: Исключения vs коды возврата, иерархия Exception / Exceptions vs return codes, Exception hierarchy

**Модуль / Module:** M07
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Представьте, что вы заказываете пиццу. Курьер может либо молча положить на порог пустую коробку (и вы поймёте проблему, только открыв её), либо позвонить и сказать: «Доставка не удалась, потому что адрес не найден». Коды возврата — это пустая коробка. Исключения — это звонок курьера. Они не позволяют проигнорировать сбой: ошибка буквально «кричит», пока кто-нибудь её не услышит и не обработает.

В ранних языках (C, ранний C++) обработка ошибок строилась на кодах возврата. Функция возвращала `-1`, `null` или код ошибки, а вызывающая сторона должна была **каждый раз** проверять результат. Проблема в том, что человек ленив: проверки пропускают, ошибки копятся, баги всплывают в production. Исключения решают это тем, что необработанная ошибка автоматически прерывает поток выполнения и поднимается по стеку вызовов, пока не встретит подходящий `catch`. Если ни один `catch` не найден — приложение падает с понятным сообщением, а не продолжает работать с повреждёнными данными.

Иерархия исключений в .NET начинается с `System.Exception`. От него наследуются две большие ветви: `SystemException` (для ошибок среды выполнения — `NullReferenceException`, `IndexOutOfRangeException`, `StackOverflowException`) и `ApplicationException` (когда-то предназначался для ошибок приложения). Важно знать исторический факт: **`ApplicationException` устарел и не рекомендуется к использованию**. Microsoft ещё в ранних guidelines прямо писала: «не бросайте `ApplicationException` и не наследуйтесь от него». Современный подход — создавать собственные типы исключений, унаследованные напрямую от `Exception`, например `OrderNotFoundException` или `PaymentDeclinedException`.

Когда бросать исключение? Только когда произошло **непредвиденное** событие, которое текущий код не может обработать локально: файл исчез, сеть упала, аргумент невалиден и нарушает контракт метода. Исключения **не** стоит использовать для обычного потока управления — например, для проверки, найден ли пользователь в базе. Для этого лучше вернуть `null`, `Option<T>` или `Result<T, Error>`, потому что «пользователь не найден» — это ожидаемый сценарий, а не авария. Бросать исключение дорого: создаётся стек-трейс, разматывается стек, это в сотни раз медленнее обычного `return`.

Хороший шаблон: методы-валидаторы в начале функции бросают `ArgumentNullException`, `ArgumentOutOfRangeException` для нарушений контракта — это часть «fail fast». А вот бизнес-сбои вроде «товар распродан» часто моделируются через тип-результат, чтобы вызывающий код мог ветвиться без `try/catch`. Понимание границы между «аварией» и «нормальным исходом» — главный навык при работе с исключениями.

Запомните три правила: (1) бросайте исключения для непредвиденных ошибок, (2) наследуйте свои исключения от `Exception`, а не от `ApplicationException`, (3) не используйте исключения для управления обычным потоком выполнения. Эти принципы — фундамент всего модуля M07.

#### Theory (EN)

Imagine ordering a pizza. The courier can either silently leave an empty box on your doorstep (so you only learn something is wrong when you open it), or call you and say: "Delivery failed because the address was not found." Return codes are the empty box. Exceptions are the courier's phone call. They make it impossible to silently ignore a failure: the error literally "shouts" until someone hears and handles it.

In early languages (C, early C++), error handling was built on return codes. A function returned `-1`, `null`, or an error code, and the caller had to check the result **every single time**. The problem is that humans are lazy: checks get skipped, errors accumulate, and bugs surface in production. Exceptions solve this by ensuring that an unhandled error automatically interrupts execution and propagates up the call stack until a matching `catch` is found. If no `catch` exists, the application crashes with a clear message instead of continuing with corrupted data.

The .NET exception hierarchy starts at `System.Exception`. It has two major branches: `SystemException` (for runtime errors such as `NullReferenceException`, `IndexOutOfRangeException`, `StackOverflowException`) and `ApplicationException` (once intended for application-level errors). A key historical fact: **`ApplicationException` is deprecated and should not be used**. Microsoft's own guidelines, written years ago, explicitly state: "Do not throw `ApplicationException` and do not derive from it." The modern approach is to create your own exception types derived directly from `Exception`, such as `OrderNotFoundException` or `PaymentDeclinedException`.

When should you throw? Only when something **unexpected** happens that the current code cannot handle locally: a file disappears, the network goes down, an argument is invalid and violates the method's contract. Exceptions should **not** be used for normal control flow — for example, to check whether a user exists in the database. For that, return `null`, an `Option<T>`, or a `Result<T, Error>`, because "user not found" is an expected scenario, not a crash. Throwing exceptions is expensive: a stack trace is captured, the stack is unwound, and the whole operation can be hundreds of times slower than a plain `return`.

A common pattern is to use validator methods at the start of a function that throw `ArgumentNullException`, `ArgumentOutOfRangeException`, or `ArgumentException` for contract violations — this is the "fail fast" philosophy. Business-level outcomes such as "item is sold out" are often modeled with a result type, so the caller can branch without `try/catch`. Understanding the boundary between "accident" and "normal outcome" is the core skill when working with exceptions.

Remember three rules: (1) throw exceptions for unexpected errors, (2) derive your custom exceptions from `Exception`, not `ApplicationException`, (3) never use exceptions for normal control flow. These principles form the foundation of the entire M07 module.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Исключения vs коды возврата, своя иерархия
// Exceptions vs return codes, custom hierarchy

using System;

// Собственное исключение наследуется напрямую от Exception,
// а НЕ от устаревшего ApplicationException.
// Custom exception derives directly from Exception,
// NOT from the deprecated ApplicationException.
public sealed class OrderNotFoundException : Exception
{
    public Guid OrderId { get; }

    public OrderNotFoundException(Guid orderId)
        : base($"Заказ не найден / Order not found: {orderId}")
    {
        OrderId = orderId;
    }

    public OrderNotFoundException(Guid orderId, Exception inner)
        : base($"Заказ не найден / Order not found: {orderId}", inner)
    {
        OrderId = orderId;
    }
}

public readonly record struct Result<T>(T Value, string? Error)
{
    public bool IsSuccess => Error is null;
}

public static class OrderService
{
    // Метод-валидатор бросает исключения при нарушении контракта — fail fast.
    // Validator throws on contract violation — fail fast.
    public static Result<decimal> GetPrice(Guid orderId, Dictionary<Guid, decimal> catalog)
    {
        // ArgumentNullException — непредвиденное нарушение контракта.
        // ArgumentNullException — unexpected contract violation.
        ArgumentNullException.ThrowIfNull(catalog);

        if (orderId == Guid.Empty)
        {
            // Аргумент невалиден — это авария, бросаем исключение.
            // Invalid argument — this is an accident, throw.
            throw new ArgumentOutOfRangeException(nameof(orderId),
                "OrderId не должен быть пустым / OrderId must not be empty");
        }

        // «Заказ не найден» — ожидаемый бизнес-сценарий.
        // Возвращаем Result вместо исключения, чтобы вызывающий
        // мог ветвиться без try/catch.
        // "Order not found" is an expected business outcome.
        // Return Result instead of throwing, so the caller can branch
        // without try/catch.
        if (!catalog.TryGetValue(orderId, out var price))
        {
            return new Result<decimal>(default, "NOT_FOUND");
        }

        return new Result<decimal>(price, null);
    }
}

public static class Program
{
    public static void Main()
    {
        var catalog = new Dictionary<Guid, decimal>
        {
            { Guid.Parse("11111111-1111-1111-1111-111111111111"), 199.99m },
        };

        var valid = Guid.Parse("11111111-1111-1111-1111-111111111111");
        var missing = Guid.Parse("22222222-2222-2222-2222-222222222222");

        // Ожидаемый сценарий — без try/catch.
        // Expected scenario — no try/catch.
        var result = OrderService.GetPrice(missing, catalog);
        Console.WriteLine(result.IsSuccess
            ? $"Цена / Price: {result.Value}"
            : $"Ошибка / Error: {result.Error}");

        // Авария — контракт нарушен, исключение пробрасывается наверх.
        // Accident — contract violated, exception propagates up.
        try
        {
            _ = OrderService.GetPrice(Guid.Empty, catalog);
        }
        catch (ArgumentOutOfRangeException ex)
        {
            Console.WriteLine($"Поймано / Caught: {ex.Message}");
        }

        // Своё исключение через явный throw (пример для другого метода).
        // Custom exception via explicit throw (example for another method).
        try
        {
            throw new OrderNotFoundException(valid);
        }
        catch (OrderNotFoundException ex)
        {
            Console.WriteLine($"OrderNotFound: {ex.OrderId} — {ex.Message}");
        }
    }
}
```

#### Best Practices

- Бросайте исключения только для непредвиденных ошибок; для ожидаемых бизнес-исходов используйте `Result<T>` или `null`.
- Наследуйте свои исключения напрямую от `Exception` и добавляйте конструктор с `innerException` для сохранения цепочки причин.
- Используйте встроенные валидаторы `ArgumentNullException.ThrowIfNull`, `ArgumentException.ThrowIfNullOrEmpty` (.NET 7+) — они короче и быстрее.
- Не глотайте исключения пустым `catch (Exception) { }` — логируйте или пробрасывайте.
- Throw exceptions only for unexpected errors; use `Result<T>` or `null` for expected business outcomes.
- Derive custom exceptions directly from `Exception` and provide a constructor that accepts an `innerException` to preserve the causal chain.
- Prefer built-in validators such as `ArgumentNullException.ThrowIfNull` and `ArgumentException.ThrowIfNullOrEmpty` (.NET 7+) — they are shorter and faster.
- Never swallow exceptions with an empty `catch (Exception) { }` — log or rethrow.

#### Частые ошибки / Common Mistakes

- Использование исключений для управления обычным потоком (`throw` вместо `return null`) → возвращайте `Result<T>` или nullable-результат для ожидаемых исходов.
- Наследование от `ApplicationException` → наследуйтесь напрямую от `Exception`; `ApplicationException` устарел и не несёт смысла.
- Пустой `catch (Exception) { }`, который прячет ошибку → логируйте через `logger.LogError(ex, ...)` или пробрасывайте через `throw;`.
- Бросание базового `Exception` вместо конкретного типа → создавайте именованные типы (`OrderNotFoundException`), чтобы `catch` был точным.
- Using exceptions for normal control flow (`throw` instead of `return null`) → return `Result<T>` or a nullable result for expected outcomes.
- Deriving from `ApplicationException` → derive directly from `Exception`; `ApplicationException` is deprecated and adds no meaning.
- An empty `catch (Exception) { }` that hides the error → log via `logger.LogError(ex, ...)` or rethrow with `throw;`.
- Throwing the base `Exception` instead of a specific type → create named types (`OrderNotFoundException`) so `catch` can be precise.

#### Чек-лист самопроверки / Self-check Checklist

- [ ] Я бросаю исключения только для непредвиденных ошибок, а не для обычного потока управления.
- [ ] Мои собственные исключения наследуются от `Exception`, а не от `ApplicationException`.
- [ ] В каждом кастомном исключении есть конструктор с `innerException`.
- [ ] Я не использую пустые `catch` и всегда логирую или пробрасываю.
- [ ] Для ожидаемых бизнес-исходов я возвращаю `Result<T>` или nullable.
- [ ] I throw exceptions only for unexpected errors, not for normal control flow.
- [ ] My custom exceptions derive from `Exception`, not from `ApplicationException`.
- [ ] Every custom exception has a constructor accepting `innerException`.
- [ ] I avoid empty `catch` blocks and always log or rethrow.
- [ ] For expected business outcomes I return `Result<T>` or a nullable.

#### Ресурсы / Resources

- [Microsoft Learn — Обработка и создание исключений / Handling and throwing exceptions — https://learn.microsoft.com/dotnet/standard/exceptions/](https://learn.microsoft.com/dotnet/standard/exceptions/)
- [Microsoft Learn — Best practices for exceptions — https://learn.microsoft.com/dotnet/standard/exceptions/best-practices-for-exceptions](https://learn.microsoft.com/dotnet/standard/exceptions/best-practices-for-exceptions)
- [.NET design guidelines — Exception throwing — https://learn.microsoft.com/dotnet/standard/design-guidelines/exceptions](https://learn.microsoft.com/dotnet/standard/design-guidelines/exceptions)

---

[⬆ К модулю M07](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
