[⬆ К модулю M07](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)

---

### Урок M07-L04: Кастомные исключения, свойства / Custom exceptions, properties

**Модуль / Module:** M07
**Время / Time:** 30 мин теория + 45 мин практика

#### Теория / Theory

Встроенные исключения .NET (`ArgumentNullException`, `InvalidOperationException`, `IOException`) удобны, но они описывают **тип сбоя**, а не **домен вашей программы**. Если банковское приложение бросает `Exception("Insufficient funds")`, коду-обработчику приходится парсить сообщение строкой — это хрупко и медленно. Кастомное исключение `InsufficientFundsException` с типизированным свойством `decimal Shortfall` решает задачу: обработчик ловит его по типу и читает число напрямую.

**Аналогия.** Встроенные исключения — это универсальные дорожные знаки «Внимание». Кастомные — знаки с конкретным смыслом: «Обрыв моста через 50 м», «Ремонт туннеля». Чем точнее знак, тем быстрее водитель (обработчик `catch`) реагирует правильно.

**Как создать.** Унаследуйте свой класс от `Exception` (или от более специфичного базового типа, например `IOException`, если семантика совпадает). .NET требует три канонических конструктора: без параметров, с сообщением и с сообщением + `innerException`. В .NET 8 Microsoft рекомендует делать класс **sealed**, если вы не планируете дальнейшее наследование — это упрощает сериализацию и работу JIT. Также добавьте конструктор для `SerializationInfo`/`StreamingContext`, если нужна бинарная сериализация (новые проекты её почти не используют, но стандарт остался).

**Свойства `Data` и `HelpLink`.** Базовый `Exception` уже несёт полезные коллекции:
- `Data` — словарь `IDictionary` для пар «ключ-значение», куда можно положить контекст (id транзакции, код ошибки). Не используйте `null`-ключи и избегайте ключей-строк без префикса сборки — возможны коллизии.
- `HelpLink` — URI на страницу документации или кода ошибки. Инструменты логирования и операторы поддержки любят это свойство: одно нажатие — и человек в wiki.

**Сериализация.** Начиная с .NET Core бинарная сериализация `BinaryFormatter` признана опасной и удалена из новых шаблонов. JSON-сериализация (`System.Text.Json`) не требует специальных конструкторов, но если ваш код когда-то попадёт в старый стек (remoting, граница AppDomain в .NET Framework), наличие конструктора `protected(SerializationInfo, StreamingContext)` и переопределение `GetObjectData` избавят от сюрпризов. В .NET 8 это «обязательный минимум для библиотеки», но избыточно для прикладного приложения.

**Дизайн-правила.** Имя должно оканчиваться на `Exception`. Не создавайте иерархию глубже 2 уровней без веской причины. Не бросайте кастомное исключение для валидации пользовательского ввода — для этого есть `ArgumentException` и `ValidationException`; кастомные типы оставьте для бизнес- и инфраструктурных сбоев. Включайте `InnerException`, когда оборачиваете низкоуровневую ошибку в доменную — это сохраняет stack trace и упрощает диагностику. И помните: исключение — это **управляющая конструкция для исключительных ситуаций**, а не способ вернуть обычный результат.

#### Theory (EN)

The built-in .NET exceptions (`ArgumentNullException`, `InvalidOperationException`, `IOException`) are convenient, but they describe a **kind of failure**, not **your program's domain**. If a banking app throws `Exception("Insufficient funds")`, the handler code has to parse the message by string — fragile and slow. A custom `InsufficientFundsException` with a typed `decimal Shortfall` property solves it: the handler catches by type and reads the number directly.

**Analogy.** Built-in exceptions are generic road signs saying "Caution". Custom ones are signs with a precise meaning: "Bridge out in 50 m", "Tunnel under repair". The more precise the sign, the faster the driver (the `catch` handler) reacts correctly.

**How to create.** Derive your class from `Exception` (or a more specific base such as `IOException` when the semantics match). .NET expects three canonical constructors: parameterless, with a message, and with a message + `innerException`. In .NET 8 Microsoft recommends making the class **sealed** unless you plan further derivation — this simplifies serialization and lets the JIT optimize. Also add a constructor for `SerializationInfo`/`StreamingContext` if you need binary serialization (modern projects rarely do, but the pattern remains the standard).

**`Data` and `HelpLink` properties.** The base `Exception` already carries useful collections:
- `Data` — an `IDictionary` of key-value pairs where you can stash context (transaction id, error code). Avoid `null` keys and bare string keys without an assembly prefix — collisions are possible.
- `HelpLink` — a URI to a documentation page or an error-code lookup. Logging tools and support engineers love it: one click and the human is in the wiki.

**Serialization.** Since .NET Core, `BinaryFormatter` is considered unsafe and has been removed from new templates. JSON serialization (`System.Text.Json`) needs no special constructors, but if your code ever crosses into a legacy stack (remoting, AppDomain boundaries on .NET Framework), having the `protected(SerializationInfo, StreamingContext)` constructor and overriding `GetObjectData` saves you from surprises. In .NET 8 this is "required minimum for a library" but overkill for an application.

**Design rules.** The name must end in `Exception`. Don't build a hierarchy deeper than two levels without a strong reason. Don't throw a custom exception to validate user input — use `ArgumentException` or `ValidationException`; reserve custom types for business and infrastructure failures. Include `InnerException` when wrapping a low-level error in a domain one — it preserves the stack trace and aids diagnostics. And remember: an exception is a **control-flow construct for exceptional situations**, not a way to return a normal result.

#### Пример кода / Code Example

```csharp
// C# 12 / .NET 8 — Custom domain exception with Data, HelpLink and serialization support
// Кастомное доменное исключение со свойствами Data, HelpLink и поддержкой сериализации

using System;
using System.Collections;
using System.Runtime.Serialization;

namespace Banking.Domain;

// sealed — .NET 8 рекомендация: упрощает сериализацию и работу JIT
// sealed — .NET 8 recommendation: simplifies serialization and JIT optimization
public sealed class InsufficientFundsException : Exception
{
    // Типизированное доменное свойство / Typed domain property
    public decimal Shortfall { get; }

    public string AccountId { get; }

    // Три канонических конструктора / Three canonical constructors
    public InsufficientFundsException() : base("Insufficient funds.") { }

    public InsufficientFundsException(string message) : base(message) { }

    public InsufficientFundsException(string message, Exception innerException)
        : base(message, innerException) { }

    // Конструктор с доменными данными / Constructor with domain data
    public InsufficientFundsException(string accountId, decimal shortfall, string? message = null)
        : base(message ?? $"Account {accountId} is short by {shortfall:C}.")
    {
        AccountId = accountId;
        Shortfall = shortfall;
        HelpLink = "https://docs.example.com/errors/insufficient-funds";
        // Data полезен для контекста, который не влезает в свойства
        // Data is useful for context that doesn't fit properties
        Data["TransactionId"] = Guid.NewGuid();
        Data["ErrorCode"] = "E_INSUFFICIENT_FUNDS";
    }

    // Сериализация (для совместимости с legacy-стеками) / Serialization (for legacy-stack compat)
    private InsufficientFundsException(SerializationInfo info, StreamingContext context)
        : base(info, context)
    {
        Shortfall = info.GetDecimal(nameof(Shortfall));
        AccountId = info.GetString(nameof(AccountId)) ?? string.Empty;
    }

    public override void GetObjectData(SerializationInfo info, StreamingContext context)
    {
        base.GetObjectData(info, context);
        info.AddValue(nameof(Shortfall), Shortfall);
        info.AddValue(nameof(AccountId), AccountId);
    }
}

// Использование / Usage
public class Account
{
    public string Id { get; }
    public decimal Balance { get; private set; }

    public Account(string id, decimal initialBalance) =>
        (Id, Balance) = (id, initialBalance);

    public void Withdraw(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentOutOfRangeException(nameof(amount), "Must be positive.");

        if (amount > Balance)
            // Бросаем доменное исключение с контекстом / Throw domain exception with context
            throw new InsufficientFundsException(Id, amount - Balance);

        Balance -= amount;
    }
}

public static class Demo
{
    public static void Run()
    {
        var account = new Account("ACC-001", 100m);
        try
        {
            account.Withdraw(250m);
        }
        catch (InsufficientFundsException ex)
        {
            // Обработчик читает типизированные данные, а не парсит строку
            // Handler reads typed data instead of parsing a string
            Console.WriteLine($"[{ex.Data["ErrorCode"]}] {ex.Message}");
            Console.WriteLine($"  Account: {ex.AccountId}, Shortfall: {ex.Shortfall:C}");
            Console.WriteLine($"  Help: {ex.HelpLink}");
            Console.WriteLine($"  Txn: {ex.Data["TransactionId"]}");
        }
    }
}
```

#### Best Practices
- Делайте класс `sealed`, если не планируете наследование — .NET 8 рекомендация для производительности и сериализации.
- Имя всегда оканчивайте на `Exception`; кладите тип в namespace, отражающий домен.
- Предоставляйте три канонических конструктора + один «богатый» с доменными параметрами.
- Заполняйте `HelpLink` и `Data` для диагностического контекста; используйте префикс сборки в ключах `Data`.
- Оборачивайте низкоуровневые ошибки через `InnerException`, сохраняя stack trace.

- Make the class `sealed` unless you intend derivation — a .NET 8 recommendation for performance and serialization.
- Always end the name with `Exception`; place the type in a namespace that reflects the domain.
- Provide the three canonical constructors plus one "rich" constructor with domain parameters.
- Populate `HelpLink` and `Data` for diagnostic context; prefix `Data` keys with the assembly name.
- Wrap low-level errors via `InnerException` to preserve the stack trace.

#### Частые ошибки / Common Mistakes
- Наследование от `ApplicationException` или `SystemException` → эти базовые типы устарели; наследуйте прямо от `Exception` или специфичного типа.
- Бросание кастомного исключения для валидации ввода → используйте `ArgumentException`/`ValidationException`; кастомные типы — для бизнес-сбоев.
- Публичные сеттеры у доменных свойств исключения → делайте свойства `get`-only; исключение неизменяемо после создания.
- Пустой конструктор по умолчанию без сообщения → всегда задавайте осмысленное сообщение по умолчанию.
- Игнорирование `InnerException` при обёртывании → всегда передавайте исходное исключение, иначе теряется диагностика.

- Deriving from `ApplicationException` or `SystemException` → these base types are obsolete; derive directly from `Exception` or a specific type.
- Throwing a custom exception for input validation → use `ArgumentException`/`ValidationException`; custom types are for business failures.
- Public setters on exception domain properties → make properties `get`-only; an exception is immutable once created.
- Empty default constructor without a message → always provide a meaningful default message.
- Ignoring `InnerException` when wrapping → always pass the original exception, otherwise diagnostics are lost.

#### Чек-лист самопроверки / Self-check Checklist
- [ ] Класс наследуется от `Exception` (или специфичного базового) и помечен `sealed`.
- [ ] Имя заканчивается на `Exception`, лежит в доменном namespace.
- [ ] Есть три канонических конструктора + «богатый» с доменными параметрами.
- [ ] Доменные свойства доступны только на чтение.
- [ ] `HelpLink` и `Data` заполнены для диагностики.
- [ ] `InnerException` передаётся при обёртывании низкоуровневых ошибок.
- [ ] Для библиотеки реализован конструктор сериализации и `GetObjectData`.

- [ ] Class derives from `Exception` (or a specific base) and is marked `sealed`.
- [ ] Name ends with `Exception`, placed in a domain namespace.
- [ ] Three canonical constructors plus a "rich" one with domain parameters exist.
- [ ] Domain properties are read-only.
- [ ] `HelpLink` and `Data` are populated for diagnostics.
- [ ] `InnerException` is passed when wrapping low-level errors.
- [ ] For a library, the serialization constructor and `GetObjectData` are implemented.

#### Ресурсы / Resources
- [Microsoft Learn — https://learn.microsoft.com/dotnet/standard/exceptions/how-to-create-user-defined-exceptions](https://learn.microsoft.com/dotnet/standard/exceptions/how-to-create-user-defined-exceptions)

---

[⬆ К модулю M07](../README.md) | [⬆ Навигация по курсу](../../NAVIGATION.md)
