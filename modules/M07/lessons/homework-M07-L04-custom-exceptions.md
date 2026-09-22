---
[← К уроку M07-L04](lesson-M07-L04-custom-exceptions.md) | [⬆ К модулю M07](../README.md) | [Следующее ДЗ →](homework-M07-L05-when-filters.md)
---

### Домашнее задание M07-L04: Кастомные исключения, свойства / Homework M07-L04: Custom exceptions, properties

**Урок / Lesson:** M07-L04
**Время / Time:** 90–120 мин / 90–120 min
**Сложность / Difficulty:** 4/5
**Цель / Goal:** (RU) Научиться проектировать доменные кастомные исключения для C# 12 / .NET 8: наследовать от `Exception`, помечать класс `sealed`, реализовывать три канонических конструктора плюс «богатый» с доменными параметрами, делать свойства get-only, заполнять `HelpLink` и `Data`, корректно оборачивать низкоуровневые ошибки через `InnerException` и отличать бизнес-сбои от валидации пользовательского ввода. (EN) Learn to design domain custom exceptions for C# 12 / .NET 8: derive from `Exception`, mark the class `sealed`, implement the three canonical constructors plus a "rich" one with domain parameters, make properties get-only, populate `HelpLink` and `Data`, correctly wrap low-level errors via `InnerException`, and distinguish business failures from user-input validation.

#### Связь с уроком / Connection to the lesson
(RU) Урок M07-L04 показывает, что встроенные исключения описывают тип сбоя, а не домен программы, и требует три канонических конструктора, `sealed`-класс, get-only свойства, `Data`/`HelpLink` и `InnerException` при обёртывании. Это ДЗ закрепляет все эти правила на сквозном платежном сценарии, где вы одновременно проектируете несколько доменных исключений, обёртываете `HttpRequestException` и явно contrastsуете кастомные типы с `ArgumentException` для валидации ввода.
(EN) Lesson M07-L04 shows that built-in exceptions describe a kind of failure, not your program's domain, and requires three canonical constructors, a `sealed` class, get-only properties, `Data`/`HelpLink`, and `InnerException` when wrapping. This homework reinforces all of these rules through an end-to-end payment scenario where you simultaneously design several domain exceptions, wrap an `HttpRequestException`, and explicitly contrast custom types with `ArgumentException` for input validation.

---

## Постановка на русском / Russian statement

#### Контекст и мотивация

Вы присоединяетесь к команде, разрабатывающей платёжный сервис `Payments.Service` на C# 12 / .NET 8. Сегодня код сервиса бросает «голые» исключения: `throw new Exception("Insufficient funds")`, `throw new Exception("Account suspended")`, `throw new Exception("Gateway timeout")`. Обработчики на верхнем уровне вынуждены парсить текст сообщения строкой, чтобы отличить один сбой от другого — это медленно, хрупко и постоянно ломается при любом изменении формулировки в сообщении. Кроме того, при сбое внешнего шлюза теряется исходный `HttpRequestException` со всем стеком вызовов, потому что разработчики оборачивают ошибку «в лоб»: `catch (HttpRequestException ex) { throw new Exception("Gateway error"); }` — без передачи `innerException`. Операторы поддержки жалуются, что в логах нет ни ссылки на документацию, ни идентификатора транзакции, ни кода ошибки, по которому можно было бы искать инцидент в wiki.

Ваша задача — спроектировать доменную модель кастомных исключений, которая делает сбои типизированными, самоописательными и пригодными для диагностики. Вы создадите несколько sealed-классов с get-only свойствами, «богатыми» конструкторами с доменными параметрами, заполните `HelpLink` и `Data` (с префиксом сборки в ключах, как требует урок), правильно обернёте низкоуровневую ошибку через `InnerException` и реализуете поддержку сериализации для библиотечного сценария. Заодно вы должны чётко провести границу: ошибки валидации аргументов (сумма `<= 0`, пустой идентификатор счёта) — это `ArgumentException`/`ArgumentNullException`, а не кастомные типы, потому что урок прямо запрещает использовать кастомные исключения для валидации пользовательского ввода.

#### Что нужно сделать (пошагово)

1. Создайте новый проект консольного приложения на .NET 8 с C# 12. Выполните команду `dotnet new console -n Payments.Service -o Payments.Service --framework net8.0`, затем перейдите в каталог `cd Payments.Service` и откройте файл `Payments.Service.csproj`. Убедитесь, что `<TargetFramework>net8.0</TargetFramework>` и `<LangVersion>latest</LangVersion>` присутствуют; при необходимости добавьте `<Nullable>enable</Nullable>` и `<ImplicitUsings>enable</ImplicitUsings>`. Запустите `dotnet build` и добейтесь успешной сборки пустого проекта.

2. Создайте папку `Domain` и в ней файл `InsufficientFundsException.cs`. Опишите класс `public sealed class InsufficientFundsException : Exception` в namespace `Payments.Service.Domain`. Класс должен быть помечен `sealed` (это рекомендация .NET 8 из урока: упрощает сериализацию и работу JIT). Добавьте get-only свойства `decimal Shortfall` и `string AccountId`. Реализуйте три канонических конструктора: без параметров (с осмысленным сообщением по умолчанию `"Insufficient funds."`), с одним параметром `string message` и с `(string message, Exception innerException)` — все три делегируют в `base(...)`.

3. Добавьте «богатый» конструктор `InsufficientFundsException(string accountId, decimal shortfall, string? message = null)`, который вызывает `base(message ?? $"Account {accountId} is short by {shortfall:C}.")`, присваивает `AccountId` и `Shortfall`, устанавливает `HelpLink = "https://docs.example.com/errors/insufficient-funds"` и кладёт в `Data` два значения: `Data["Payments.TransactionId"] = Guid.NewGuid();` и `Data["Payments.ErrorCode"] = "E_INSUFFICIENT_FUNDS";`. Обратите внимание на префикс `Payments.` в ключах `Data` — урок предупреждает о коллизиях без префикса сборки.

4. В том же файле реализуйте закрытый конструктор сериализации `private InsufficientFundsException(SerializationInfo info, StreamingContext context) : base(info, context)`, который читает `Shortfall` через `info.GetDecimal(nameof(Shortfall))` и `AccountId` через `info.GetString(nameof(AccountId)) ?? string.Empty`. Переопределите `public override void GetObjectData(SerializationInfo info, StreamingContext context)`, вызовите `base.GetObjectData(info, context)` и добавьте `info.AddValue(nameof(Shortfall), Shortfall);` и `info.AddValue(nameof(AccountId), AccountId);`. Это «обязательный минимум для библиотеки» по формулировке урока.

5. Создайте второй доменный класс `AccountSuspendedException` по той же схеме: sealed, три канонических конструктора, «богатый» с доменными параметрами `string AccountId` и `DateTime SuspendedUntil`, get-only свойства, `HelpLink`, `Data` с префиксом сборки. Этот класс описывает бизнес-сбой состояния счёта (аккаунт заморожен до определённой даты), а не ошибку валидации ввода.

6. Создайте третий класс `PaymentGatewayException`, который моделирует инфраструктурный сбой и обязательно оборачивает низкоуровневую ошибку. Его «богатый» конструктор принимает `string operationName`, `int httpStatusCode`, `Exception innerException` и строку `message?`. Передавайте `innerException` в `base(message ?? ..., innerException)` — урок требует сохранять исходное исключение, иначе теряется диагностика и stack trace. Заполните `HelpLink` и `Data["Payments.OperationName"]`, `Data["Payments.HttpStatusCode"]`.

7. Реализуйте класс `Account` с get-only `Id`, свойством `Balance` с приватным сеттером, флагом `IsSuspended` и `SuspendedUntil`. Метод `Withdraw(decimal amount)` должен: при `amount <= 0` бросать `ArgumentOutOfRangeException(nameof(amount), "Must be positive.")` (это валидация ввода, а не бизнес-сбой — урок запрещает кастомные исключения для валидации); при `IsSuspended` бросать `AccountSuspendedException(Id, SuspendedUntil);`; при `amount > Balance` бросать `InsufficientFundsException(Id, amount - Balance);` иначе уменьшать `Balance`.

8. Реализуйте класс `PaymentGateway` с методом `ChargeAsync(Account account, decimal amount)`, который имитирует сетевой вызов и в части случаев бросает `HttpRequestException`. В сервисном слое `PaymentService.ChargeAsync` перехватывайте `HttpRequestException ex` и бросайте `new PaymentGatewayException("charge", 503, ex)` — это корректное обёртывание с `InnerException`. В `Program.cs` используйте top-level statements: создайте аккаунт, попробуйте списать сумму больше баланса, перехватите `InsufficientFundsException`, выведите `Message`, `AccountId`, `Shortfall`, `HelpLink` и `Data["Payments.TransactionId"]`. Затем принудительно вызовите сбой шлюза и перехватите `PaymentGatewayException`, выведите `ex.InnerException?.Message`, чтобы убедиться, что stack trace сохранён.

9. Проверьте поведение: запустите `dotnet run` и зафиксируйте вывод. Убедитесь, что для валидационного случая (`amount <= 0`) летит именно `ArgumentOutOfRangeException`, а не кастомный тип — это доказывает, что вы провели границу из урока правильно.

10. (Бонус для библиотеки.) Добавьте unit-тест `dotnet new xunit -n Payments.Service.Tests`, проверьте, что после ручной сериализации через `GetObjectData`/конструктор `SerializationInfo` свойства `Shortfall` и `AccountId` восстанавливаются. Используйте `new BinaryFormatter` нельзя (он удалён в .NET 8) — вызывайте конструктор и `GetObjectData` вручную, создав `SerializationInfo` и `StreamingContext`.

#### Требования к решению

Решение должно компилироваться без предупреждений под `net8.0` с включённым `<Nullable>enable</Nullable>` и `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` (добавьте это в `csproj` для строгости). Все три доменных класса обязаны быть `sealed` и наследоваться напрямую от `Exception` — наследование от устаревших `ApplicationException` или `SystemException` запрещено уроком и будет считаться ошибкой. Имя каждого класса должно оканчиваться на `Exception`, а namespace `Payments.Service.Domain` должен отражать домен.

Каждый класс обязан предоставлять три канонических конструктора (без параметров, с сообщением, с сообщением + `innerException`) плюс один «богатый» конструктор с доменными параметрами. Все доменные свойства (`Shortfall`, `AccountId`, `SuspendedUntil`, `HttpStatusCode`, `OperationName`) должны быть get-only — публичные сеттеры запрещены, потому что исключение неизменяемо после создания. Конструктор без параметров обязан задавать осмысленное сообщение по умолчанию, а не пустую строку.

В каждом «богатом» конструкторе должны быть заполнены `HelpLink` (валидный URI) и минимум два ключа в `Data` с префиксом сборки `Payments.`. При обёртывании низкоуровневой ошибки в `PaymentGatewayException` исходное исключение обязано передаваться через `InnerException` — пустой `innerException` не допускается. Для валидации аргументов (`amount <= 0`, пустой `accountId`) используйте `ArgumentOutOfRangeException`/`ArgumentNullException`, а не кастомные типы. Для класса `InsufficientFundsException` должны быть реализованы конструктор сериализации и `GetObjectData`.

#### Тонкости и подводные камни

Главная тонкость, на которой спотыкаются новички, — различие между бизнес-сбоем и валидацией ввода. Урок прямо говорит: «Не бросайте кастомное исключение для валидации пользовательского ввода — для этого есть `ArgumentException` и `ValidationException`». Значит, `amount <= 0` в `Withdraw` — это `ArgumentOutOfRangeException`, а не `InvalidPaymentAmountException`. Бизнес-сбои — это ситуации, которые возникают не из-за неправильного аргумента, а из-за состояния домена: на счету недостаточно денег, аккаунт заморожен, шлюз недоступен. Только для них создаются кастомные типы.

Вторая тонкость — `sealed`. Урок рекомендует помечать класс `sealed` в .NET 8, если вы не планируете наследование. Это не стилистическая прихоть: `sealed` упрощает сериализацию (виртуальные вызовы на десериализации исчезают) и даёт JIT больше возможностей для девиртуализации. Забыть `sealed` — значит потерять производительность и добавить себе проблем с безопасностью сериализации. Третья тонкость — ключи в `Data`. Урок предупреждает: «избегайте ключей-строк без префикса сборки — возможны коллизии». Используйте `Data["Payments.TransactionId"]`, а не `Data["TransactionId"]`, иначе другая библиотека, положившая такой же ключ, перезапишет ваше значение.

Четвёртая тонкость — `InnerException`. При обёртывании `HttpRequestException` в `PaymentGatewayException` новички часто пишут `throw new PaymentGatewayException("charge", 503);` и теряют исходное исключение со всем стеком. Урок требует: «Включайте `InnerException`, когда оборачиваете низкоуровневую ошибку в доменную — это сохраняет stack trace и упрощает диагностику». Пятая тонкость — конструктор по умолчанию. Урок列为 частую ошибку «пустой конструктор по умолчанию без сообщения». Конструктор `InsufficientFundsException()` должен вызывать `base("Insufficient funds.")`, а не `base()`.

Шестая тонкость — сериализация. В .NET 8 `BinaryFormatter` удалён, но конструктор `(SerializationInfo, StreamingContext)` и `GetObjectData` остаются «обязательным минимумом для библиотеки» — если ваш код когда-то попадёт в legacy-стек или в AppDomain-границу, отсутствие этих членов приведёт к `SerializationException`. В прикладном приложении это избыточно, но в задании вы реализуете их для `InsufficientFundsException`, чтобы продемонстрировать полный паттерн. Седьмая тонкость — не углубляйте иерархию больше двух уровней без веской причины; урок явно предостерегает от чрезмерной иерархии исключений.

#### Критерии приёмки

- [ ] Проект `Payments.Service` собирается под `net8.0` с `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` без ошибок и предупреждений.
- [ ] Все три доменных класса помечены `sealed` и наследуются напрямую от `Exception`.
- [ ] Имя каждого класса оканчивается на `Exception`; namespace `Payments.Service.Domain` отражает домен.
- [ ] Каждый класс имеет три канонических конструктора: без параметров, с `message`, с `message` + `innerException`.
- [ ] Каждый класс имеет «богатый» конструктор с доменными параметрами.
- [ ] Конструктор без параметров задаёт осмысленное сообщение по умолчанию.
- [ ] Все доменные свойства get-only; публичных сеттеров нет.
- [ ] `HelpLink` заполнен валидным URI в каждом «богатом» конструкторе.
- [ ] В `Data` минимум два ключа с префиксом `Payments.` (TransactionId/ErrorCode или эквивалент).
- [ ] `PaymentGatewayException` передаёт исходное исключение через `InnerException`.
- [ ] Валидация `amount <= 0` бросает `ArgumentOutOfRangeException`, а не кастомный тип.
- [ ] Для `InsufficientFundsException` реализованы конструктор сериализации и `GetObjectData`.
- [ ] `Program.cs` использует top-level statements и демонстрирует перехват всех трёх типов.
- [ ] Вывод `dotnet run` показывает `InnerException?.Message` для `PaymentGatewayException`.
- [ ] Unit-тест (бонус) проверяет восстановление свойств после `GetObjectData`.

#### Подсказки (без прямого ответа)

- Подумайте, какой именно базовый конструктор вызывает каждый канонический: `base()`, `base(message)`, `base(message, innerException)`. Не путайте порядок параметров.
- Для префикса ключей `Data` используйте имя сборки/домена как namespace — это Prevents коллизии с чужим кодом.
- В «богатом» конструкторе `message ?? $"..."` позволяет вызывающей стороне переопределить текст, но даёт разумное значение по умолчанию.
- Чтобы сохранить stack trace при обёртывании, достаточно передать `innerException` в `base` — никаких `ExceptionDispatchInfo` для синхронного обёртывания не требуется.
- Для `nameof(Shortfall)` в `GetObjectData` и конструкторе сериализации используйте одно и то же имя свойства — иначе десериализация вернёт `null`/`0`.
- Для имитации сбоя шлюза достаточно `throw new HttpRequestException("Gateway timeout", null, statusCode: System.Net.HttpStatusCode.ServiceUnavailable);`.

#### Эталонное решение (разбор)

```csharp
// C# 12 / .NET 8 — Payments.Service: доменные кастомные исключения
// Доменная модель платежей со свойствами, Data, HelpLink, InnerException и сериализацией

using System;
using System.Collections;
using System.Net.Http;
using System.Runtime.Serialization;

namespace Payments.Service.Domain;

// sealed — рекомендация .NET 8: упрощает сериализацию и девиртуализацию JIT
// sealed — .NET 8 recommendation: simplifies serialization and JIT devirtualization
public sealed class InsufficientFundsException : Exception
{
    // get-only: исключение неизменяемо после создания / get-only: immutable after creation
    public decimal Shortfall { get; }
    public string AccountId { get; }

    // Три канонических конструктора / Three canonical constructors
    public InsufficientFundsException() : base("Insufficient funds.") { }

    public InsufficientFundsException(string message) : base(message) { }

    public InsufficientFundsException(string message, Exception innerException)
        : base(message, innerException) { }

    // «Богатый» конструктор с доменными параметрами / Rich constructor with domain data
    public InsufficientFundsException(string accountId, decimal shortfall, string? message = null)
        : base(message ?? $"Account {accountId} is short by {shortfall:C}.")
    {
        AccountId = accountId;
        Shortfall = shortfall;
        HelpLink = "https://docs.example.com/errors/insufficient-funds";
        // Префикс сборки в ключах Data предотвращает коллизии
        // Assembly-prefix in Data keys prevents collisions
        Data["Payments.TransactionId"] = Guid.NewGuid();
        Data["Payments.ErrorCode"] = "E_INSUFFICIENT_FUNDS";
    }

    // Сериализация: «обязательный минимум для библиотеки» / Serialization: required minimum for a library
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

// Бизнес-сбой состояния счёта / Business failure: account state
public sealed class AccountSuspendedException : Exception
{
    public string AccountId { get; }
    public DateTime SuspendedUntil { get; }

    public AccountSuspendedException() : base("Account is suspended.") { }
    public AccountSuspendedException(string message) : base(message) { }
    public AccountSuspendedException(string message, Exception innerException)
        : base(message, innerException) { }

    public AccountSuspendedException(string accountId, DateTime suspendedUntil, string? message = null)
        : base(message ?? $"Account {accountId} is suspended until {suspendedUntil:O}.")
    {
        AccountId = accountId;
        SuspendedUntil = suspendedUntil;
        HelpLink = "https://docs.example.com/errors/account-suspended";
        Data["Payments.AccountId"] = accountId;
        Data["Payments.ErrorCode"] = "E_ACCOUNT_SUSPENDED";
    }
}

// Инфраструктурный сбой: обёртываем низкоуровневую ошибку / Infrastructure failure: wrap low-level error
public sealed class PaymentGatewayException : Exception
{
    public string OperationName { get; }
    public int HttpStatusCode { get; }

    public PaymentGatewayException() : base("Payment gateway error.") { }
    public PaymentGatewayException(string message) : base(message) { }

    // Обязательно передаём innerException — сохраняем stack trace / Always pass innerException
    public PaymentGatewayException(string message, Exception innerException)
        : base(message, innerException) { }

    public PaymentGatewayException(string operationName, int httpStatusCode, Exception innerException,
        string? message = null)
        : base(message ?? $"Gateway operation '{operationName}' failed with HTTP {httpStatusCode}.",
               innerException)
    {
        OperationName = operationName;
        HttpStatusCode = httpStatusCode;
        HelpLink = "https://docs.example.com/errors/payment-gateway";
        Data["Payments.OperationName"] = operationName;
        Data["Payments.HttpStatusCode"] = httpStatusCode;
        Data["Payments.ErrorCode"] = "E_GATEWAY";
    }
}

public sealed class Account
{
    public string Id { get; }
    public decimal Balance { get; private set; }
    public bool IsSuspended { get; private set; }
    public DateTime SuspendedUntil { get; private set; }

    public Account(string id, decimal initialBalance)
    {
        // Валидация ввода — НЕ кастомный тип / Input validation — NOT a custom type
        if (string.IsNullOrWhiteSpace(id))
            throw new ArgumentNullException(nameof(id));
        if (initialBalance < 0)
            throw new ArgumentOutOfRangeException(nameof(initialBalance), "Must be non-negative.");
        Id = id;
        Balance = initialBalance;
    }

    public void Suspend(DateTime until)
    {
        IsSuspended = true;
        SuspendedUntil = until;
    }

    public void Withdraw(decimal amount)
    {
        // Валидация ввода / Input validation
        if (amount <= 0)
            throw new ArgumentOutOfRangeException(nameof(amount), "Must be positive.");

        // Бизнес-сбой: состояние счёта / Business failure: account state
        if (IsSuspended && DateTime.UtcNow < SuspendedUntil)
            throw new AccountSuspendedException(Id, SuspendedUntil);

        // Бизнес-сбой: недостаточно средств / Business failure: insufficient funds
        if (amount > Balance)
            throw new InsufficientFundsException(Id, amount - Balance);

        Balance -= amount;
    }
}

public sealed class PaymentGateway
{
    private readonly Random _rng = new(42);

    public HttpResponseMessage Charge(Account account, decimal amount)
    {
        // Имитация сетевого сбоя / Simulated network failure
        if (_rng.Next(0, 2) == 0)
            throw new HttpRequestException("Gateway timeout", null, System.Net.HttpStatusCode.ServiceUnavailable);

        account.Withdraw(amount);
        return new HttpResponseMessage(System.Net.HttpStatusCode.OK);
    }
}

public sealed class PaymentService
{
    private readonly PaymentGateway _gateway = new();

    public void Charge(Account account, decimal amount)
    {
        try
        {
            _gateway.Charge(account, amount);
        }
        catch (HttpRequestException ex)
        {
            // Корректное обёртывание с InnerException / Correct wrapping with InnerException
            throw new PaymentGatewayException("charge", 503, ex);
        }
    }
}
```

Разбор по строкам. Класс `InsufficientFundsException` помечен `sealed` — это применяет рекомендацию .NET 8 из урока: упрощает сериализацию и девиртуализацию JIT. Наследование напрямую от `Exception`, а не от устаревшего `ApplicationException`, отвечает правилу урока «наследуйте прямо от `Exception` или специфичного типа». Свойства `Shortfall` и `AccountId` объявлены с get-only accessor `{ get; }` — это реализует требование «исключение неизменяемо после создания» и устраняет частую ошибку «публичные сеттеры у доменных свойств». Три канонических конструктора `()`, `(message)` и `(message, innerException)` делегируют в `base(...)` — урок требует именно этот набор. Конструктор без параметров вызывает `base("Insufficient funds.")`, а не пустой `base()`, что закрывает частую ошибку «пустой конструктор по умолчанию без сообщения».

«Богатый» конструктор `(accountId, shortfall, message?)` использует `message ?? $"..."` для осмысленного значения по умолчанию, присваивает доменные свойства, задаёт `HelpLink` и заполняет `Data` с префиксом `Payments.` — это применяет best practice урока «заполняйте HelpLink и Data; используйте префикс сборки в ключах Data». Конструктор сериализации и `GetObjectData` реализуют «обязательный минимум для библиотеки»: `nameof(Shortfall)` гарантирует, что ключи совпадают между записью и чтением, иначе десериализация вернула бы `0`. Класс `AccountSuspendedException` повторяет паттерн для другого бизнес-сбоя — состояния счёта.

`PaymentGatewayException` — ключевой пример обёртывания: его «богатый» конструктор принимает `Exception innerException` и передаёт его в `base(..., innerException)`, что применяет правило урока «включайте InnerException, когда оборачиваете низкоуровневую ошибку в доменную». Без этого потерялся бы stack trace исходного `HttpRequestException`. Класс `Account` проводит границу из урока: `amount <= 0` бросает `ArgumentOutOfRangeException` (валидация ввода), а `IsSuspended` и `amount > Balance` бросают кастомные типы (бизнес-сбои). Это прямо реализует правило «не бросайте кастомное исключение для валидации пользовательского ввода». `PaymentService.Charge` перехватывает `HttpRequestException ex` и бросает `new PaymentGatewayException("charge", 503, ex)` — корректное обёртывание с сохранением `InnerException`.

#### Задания на углубление (бонус)

1. Добавьте четвёртый доменный класс `DuplicatePaymentException` с get-only свойством `string RequestId` и переопределённым свойством `Message`, дописывающим `[dup {RequestId}]` к базовому сообщению через `base.Message`. Покажите, почему переопределение `Message` безопаснее, чем форматирование в конструкторе.
2. Реализуйте фабрику исключений `PaymentExceptionFactory.Create(int errorCode, ...)`, которая по коду ошибки возвращает нужный тип через `switch` expression с pattern matching. Обсудите, когда фабрика уместнее прямого `throw new`.
3. Добавьте логирование через `Microsoft.Extensions.Logging`: в обработчике верхнего уровня пишите `ex.Data["Payments.ErrorCode"]`, `ex.HelpLink` и `ex.InnerException?.StackTrace`. Покажите, как префикс сборки в ключах `Data` помогает агрегировать инциденты.
4. Напишите unit-тесты на сериализацию `InsufficientFundsException` через ручной вызов `GetObjectData` и конструктор `SerializationInfo` (без `BinaryFormatter`), проверяя round-trip всех доменных свойств.

---

## Statement in English / Постановка на английском

#### Context & motivation

You are joining a team building a payment service called `Payments.Service` on C# 12 / .NET 8. Today the service throws "bare" exceptions: `throw new Exception("Insufficient funds")`, `throw new Exception("Account suspended")`, `throw new Exception("Gateway timeout")`. Top-level handlers are forced to parse the message text by string to tell one failure apart from another — slow, fragile, and constantly broken whenever a wording changes. On top of that, when the external gateway fails, the original `HttpRequestException` with its full stack trace is lost, because developers wrap the error bluntly: `catch (HttpRequestException ex) { throw new Exception("Gateway error"); }` — without passing `innerException`. Support engineers complain that the logs contain neither a documentation link, nor a transaction id, nor an error code they could use to search the incident wiki.

Your job is to design a domain model of custom exceptions that makes failures typed, self-describing, and diagnosable. You will create several sealed classes with get-only properties, "rich" constructors with domain parameters, populate `HelpLink` and `Data` (with the assembly-prefix keys the lesson requires), correctly wrap a low-level error through `InnerException`, and implement serialization support for the library scenario. Along the way you must draw a clear line: argument-validation errors (`amount <= 0`, an empty account id) are `ArgumentException`/`ArgumentNullException`, not custom types, because the lesson explicitly forbids using custom exceptions for user-input validation.

#### What to do step by step

1. Create a new .NET 8 console project targeting C# 12. Run `dotnet new console -n Payments.Service -o Payments.Service --framework net8.0`, then `cd Payments.Service` and open `Payments.Service.csproj`. Make sure `<TargetFramework>net8.0</TargetFramework>` and `<LangVersion>latest</LangVersion>` are present; add `<Nullable>enable</Nullable>` and `<ImplicitUsings>enable</ImplicitUsings>` if missing. Run `dotnet build` and confirm the empty project compiles cleanly.

2. Create a `Domain` folder and a file `InsufficientFundsException.cs` inside it. Declare `public sealed class InsufficientFundsException : Exception` in namespace `Payments.Service.Domain`. The class must be `sealed` — this is the .NET 8 recommendation from the lesson: it simplifies serialization and JIT optimization. Add get-only properties `decimal Shortfall` and `string AccountId`. Implement the three canonical constructors: parameterless (with a meaningful default message `"Insufficient funds."`), one taking `string message`, and one taking `(string message, Exception innerException)` — all three delegating to `base(...)`.

3. Add a "rich" constructor `InsufficientFundsException(string accountId, decimal shortfall, string? message = null)` that calls `base(message ?? $"Account {accountId} is short by {shortfall:C}.")`, assigns `AccountId` and `Shortfall`, sets `HelpLink = "https://docs.example.com/errors/insufficient-funds"`, and stores two values in `Data`: `Data["Payments.TransactionId"] = Guid.NewGuid();` and `Data["Payments.ErrorCode"] = "E_INSUFFICIENT_FUNDS";`. Notice the `Payments.` prefix in the `Data` keys — the lesson warns about collisions without an assembly prefix.

4. In the same file, implement the serialization constructor `private InsufficientFundsException(SerializationInfo info, StreamingContext context) : base(info, context)` that reads `Shortfall` via `info.GetDecimal(nameof(Shortfall))` and `AccountId` via `info.GetString(nameof(AccountId)) ?? string.Empty`. Override `public override void GetObjectData(SerializationInfo info, StreamingContext context)`, call `base.GetObjectData(info, context)`, then add `info.AddValue(nameof(Shortfall), Shortfall);` and `info.AddValue(nameof(AccountId), AccountId);`. This is the "required minimum for a library" wording from the lesson.

5. Create a second domain class `AccountSuspendedException` following the same shape: sealed, three canonical constructors, a "rich" one with domain parameters `string AccountId` and `DateTime SuspendedUntil`, get-only properties, `HelpLink`, and `Data` with the assembly prefix. This class models a business failure of account state (account frozen until a specific date), not an input-validation error.

6. Create a third class `PaymentGatewayException` that models an infrastructure failure and must wrap a low-level error. Its "rich" constructor takes `string operationName`, `int httpStatusCode`, `Exception innerException`, and an optional `string? message`. Pass `innerException` into `base(message ?? ..., innerException)` — the lesson demands preserving the original exception, otherwise diagnostics and the stack trace are lost. Populate `HelpLink` and `Data["Payments.OperationName"]`, `Data["Payments.HttpStatusCode"]`.

7. Implement an `Account` class with a get-only `Id`, a `Balance` property with a private setter, an `IsSuspended` flag, and `SuspendedUntil`. The `Withdraw(decimal amount)` method must: throw `ArgumentOutOfRangeException(nameof(amount), "Must be positive.")` when `amount <= 0` (this is input validation, not a business failure — the lesson forbids custom exceptions for validation); throw `AccountSuspendedException(Id, SuspendedUntil);` when `IsSuspended`; throw `InsufficientFundsException(Id, amount - Balance);` when `amount > Balance`; otherwise decrement `Balance`.

8. Implement a `PaymentGateway` class with a method `ChargeAsync(Account account, decimal amount)` that simulates a network call and throws `HttpRequestException` in some cases. In the service layer `PaymentService.ChargeAsync`, catch `HttpRequestException ex` and throw `new PaymentGatewayException("charge", 503, ex)` — this is correct wrapping with `InnerException`. In `Program.cs`, use top-level statements: create an account, attempt to withdraw more than the balance, catch `InsufficientFundsException`, and print `Message`, `AccountId`, `Shortfall`, `HelpLink`, and `Data["Payments.TransactionId"]`. Then force a gateway failure and catch `PaymentGatewayException`, printing `ex.InnerException?.Message` to confirm the stack trace is preserved.

9. Verify the behavior: run `dotnet run` and capture the output. Confirm that for the validation case (`amount <= 0`) an `ArgumentOutOfRangeException` is thrown, not a custom type — this proves you drew the lesson's boundary correctly.

10. (Library bonus.) Add a unit test project `dotnet new xunit -n Payments.Service.Tests`, and verify that after manually serializing through `GetObjectData` and the `SerializationInfo` constructor, the `Shortfall` and `AccountId` properties are restored. You cannot use `new BinaryFormatter` (it is removed in .NET 8) — invoke the constructor and `GetObjectData` manually by creating a `SerializationInfo` and a `StreamingContext`.

#### Requirements

The solution must compile warning-free under `net8.0` with `<Nullable>enable</Nullable>` and `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` (add this to the `csproj` for strictness). All three domain classes must be `sealed` and derive directly from `Exception` — deriving from the obsolete `ApplicationException` or `SystemException` is forbidden by the lesson and will be treated as an error. Each class name must end in `Exception`, and the namespace `Payments.Service.Domain` must reflect the domain.

Each class must provide the three canonical constructors (parameterless, with a message, with a message + `innerException`) plus one "rich" constructor with domain parameters. All domain properties (`Shortfall`, `AccountId`, `SuspendedUntil`, `HttpStatusCode`, `OperationName`) must be get-only — public setters are forbidden because an exception is immutable once created. The parameterless constructor must set a meaningful default message, not an empty string.

In each "rich" constructor, `HelpLink` (a valid URI) and at least two `Data` keys with the `Payments.` assembly prefix must be populated. When wrapping a low-level error in `PaymentGatewayException`, the original exception must be passed through `InnerException` — an empty `innerException` is not acceptable. For argument validation (`amount <= 0`, empty `accountId`), use `ArgumentOutOfRangeException`/`ArgumentNullException`, not custom types. The `InsufficientFundsException` class must implement the serialization constructor and `GetObjectData`.

#### Pitfalls

The main pitfall that trips up beginners is the difference between a business failure and input validation. The lesson says plainly: "Don't throw a custom exception to validate user input — use `ArgumentException` or `ValidationException`". So `amount <= 0` in `Withdraw` is an `ArgumentOutOfRangeException`, not an `InvalidPaymentAmountException`. Business failures are situations that do not arise from a bad argument but from the state of the domain: not enough money on the account, the account is frozen, the gateway is down. Only these warrant a custom type.

The second pitfall is `sealed`. The lesson recommends marking the class `sealed` in .NET 8 unless you plan derivation. This is not stylistic: `sealed` simplifies serialization (virtual dispatch on deserialization disappears) and gives the JIT more devirtualization opportunities. Forgetting `sealed` costs performance and adds serialization-safety headaches. The third pitfall is `Data` keys. The lesson warns: "avoid bare string keys without an assembly prefix — collisions are possible". Use `Data["Payments.TransactionId"]`, not `Data["TransactionId"]`, otherwise another library using the same key will overwrite your value.

The fourth pitfall is `InnerException`. When wrapping an `HttpRequestException` in `PaymentGatewayException`, beginners often write `throw new PaymentGatewayException("charge", 503);` and lose the original exception with its entire stack. The lesson requires: "Include `InnerException` when wrapping a low-level error in a domain one — it preserves the stack trace and aids diagnostics". The fifth pitfall is the default constructor. The lesson lists "empty default constructor without a message" as a common mistake. `InsufficientFundsException()` must call `base("Insufficient funds.")`, not `base()`.

The sixth pitfall is serialization. In .NET 8 `BinaryFormatter` is gone, but the `(SerializationInfo, StreamingContext)` constructor and `GetObjectData` remain "the required minimum for a library" — if your code ever crosses into a legacy stack or an AppDomain boundary, missing these members will raise `SerializationException`. It is overkill for an application, but in this assignment you implement them for `InsufficientFundsException` to demonstrate the full pattern. The seventh pitfall is hierarchy depth — do not build an exception hierarchy deeper than two levels without a strong reason; the lesson explicitly cautions against overly deep hierarchies.

#### Acceptance criteria

- [ ] The `Payments.Service` project builds under `net8.0` with `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` without errors or warnings.
- [ ] All three domain classes are `sealed` and derive directly from `Exception`.
- [ ] Each class name ends in `Exception`; the namespace `Payments.Service.Domain` reflects the domain.
- [ ] Each class has the three canonical constructors: parameterless, with `message`, with `message` + `innerException`.
- [ ] Each class has a "rich" constructor with domain parameters.
- [ ] The parameterless constructor sets a meaningful default message.
- [ ] All domain properties are get-only; there are no public setters.
- [ ] `HelpLink` is populated with a valid URI in each "rich" constructor.
- [ ] `Data` has at least two keys with the `Payments.` prefix (TransactionId/ErrorCode or equivalent).
- [ ] `PaymentGatewayException` passes the original exception through `InnerException`.
- [ ] Validation of `amount <= 0` throws `ArgumentOutOfRangeException`, not a custom type.
- [ ] `InsufficientFundsException` implements the serialization constructor and `GetObjectData`.
- [ ] `Program.cs` uses top-level statements and demonstrates catching all three types.
- [ ] The `dotnet run` output shows `InnerException?.Message` for `PaymentGatewayException`.
- [ ] A unit test (bonus) verifies property restoration after `GetObjectData`.

#### Hints (without giving away the answer)

- Think about which base constructor each canonical one calls: `base()`, `base(message)`, `base(message, innerException)`. Do not mix up the parameter order.
- For `Data` key prefixes, use the assembly/domain name as a namespace — it prevents collisions with foreign code.
- In the "rich" constructor, `message ?? $"..."` lets the caller override the text while still giving a sensible default.
- To preserve the stack trace when wrapping, it is enough to pass `innerException` to `base` — no `ExceptionDispatchInfo` is needed for synchronous wrapping.
- Use `nameof(Shortfall)` consistently in `GetObjectData` and the serialization constructor — otherwise deserialization returns `null`/`0`.
- To simulate a gateway failure, `throw new HttpRequestException("Gateway timeout", null, statusCode: System.Net.HttpStatusCode.ServiceUnavailable);` is enough.

#### Reference solution walk-through (EN)

```csharp
// C# 12 / .NET 8 — Payments.Service: custom domain exceptions
// Payment domain model with properties, Data, HelpLink, InnerException and serialization

using System;
using System.Collections;
using System.Net.Http;
using System.Runtime.Serialization;

namespace Payments.Service.Domain;

// sealed — .NET 8 recommendation: simplifies serialization and JIT devirtualization
public sealed class InsufficientFundsException : Exception
{
    // get-only: immutable once created
    public decimal Shortfall { get; }
    public string AccountId { get; }

    // Three canonical constructors
    public InsufficientFundsException() : base("Insufficient funds.") { }

    public InsufficientFundsException(string message) : base(message) { }

    public InsufficientFundsException(string message, Exception innerException)
        : base(message, innerException) { }

    // Rich constructor with domain data
    public InsufficientFundsException(string accountId, decimal shortfall, string? message = null)
        : base(message ?? $"Account {accountId} is short by {shortfall:C}.")
    {
        AccountId = accountId;
        Shortfall = shortfall;
        HelpLink = "https://docs.example.com/errors/insufficient-funds";
        // Assembly prefix in Data keys prevents collisions
        Data["Payments.TransactionId"] = Guid.NewGuid();
        Data["Payments.ErrorCode"] = "E_INSUFFICIENT_FUNDS";
    }

    // Serialization: required minimum for a library
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

// Business failure: account state
public sealed class AccountSuspendedException : Exception
{
    public string AccountId { get; }
    public DateTime SuspendedUntil { get; }

    public AccountSuspendedException() : base("Account is suspended.") { }
    public AccountSuspendedException(string message) : base(message) { }
    public AccountSuspendedException(string message, Exception innerException)
        : base(message, innerException) { }

    public AccountSuspendedException(string accountId, DateTime suspendedUntil, string? message = null)
        : base(message ?? $"Account {accountId} is suspended until {suspendedUntil:O}.")
    {
        AccountId = accountId;
        SuspendedUntil = suspendedUntil;
        HelpLink = "https://docs.example.com/errors/account-suspended";
        Data["Payments.AccountId"] = accountId;
        Data["Payments.ErrorCode"] = "E_ACCOUNT_SUSPENDED";
    }
}

// Infrastructure failure: wrap the low-level error
public sealed class PaymentGatewayException : Exception
{
    public string OperationName { get; }
    public int HttpStatusCode { get; }

    public PaymentGatewayException() : base("Payment gateway error.") { }
    public PaymentGatewayException(string message) : base(message) { }

    // Always pass innerException — preserve the stack trace
    public PaymentGatewayException(string message, Exception innerException)
        : base(message, innerException) { }

    public PaymentGatewayException(string operationName, int httpStatusCode, Exception innerException,
        string? message = null)
        : base(message ?? $"Gateway operation '{operationName}' failed with HTTP {httpStatusCode}.",
               innerException)
    {
        OperationName = operationName;
        HttpStatusCode = httpStatusCode;
        HelpLink = "https://docs.example.com/errors/payment-gateway";
        Data["Payments.OperationName"] = operationName;
        Data["Payments.HttpStatusCode"] = httpStatusCode;
        Data["Payments.ErrorCode"] = "E_GATEWAY";
    }
}

public sealed class Account
{
    public string Id { get; }
    public decimal Balance { get; private set; }
    public bool IsSuspended { get; private set; }
    public DateTime SuspendedUntil { get; private set; }

    public Account(string id, decimal initialBalance)
    {
        // Input validation — NOT a custom type
        if (string.IsNullOrWhiteSpace(id))
            throw new ArgumentNullException(nameof(id));
        if (initialBalance < 0)
            throw new ArgumentOutOfRangeException(nameof(initialBalance), "Must be non-negative.");
        Id = id;
        Balance = initialBalance;
    }

    public void Suspend(DateTime until)
    {
        IsSuspended = true;
        SuspendedUntil = until;
    }

    public void Withdraw(decimal amount)
    {
        // Input validation
        if (amount <= 0)
            throw new ArgumentOutOfRangeException(nameof(amount), "Must be positive.");

        // Business failure: account state
        if (IsSuspended && DateTime.UtcNow < SuspendedUntil)
            throw new AccountSuspendedException(Id, SuspendedUntil);

        // Business failure: insufficient funds
        if (amount > Balance)
            throw new InsufficientFundsException(Id, amount - Balance);

        Balance -= amount;
    }
}

public sealed class PaymentGateway
{
    private readonly Random _rng = new(42);

    public HttpResponseMessage Charge(Account account, decimal amount)
    {
        // Simulated network failure
        if (_rng.Next(0, 2) == 0)
            throw new HttpRequestException("Gateway timeout", null, System.Net.HttpStatusCode.ServiceUnavailable);

        account.Withdraw(amount);
        return new HttpResponseMessage(System.Net.HttpStatusCode.OK);
    }
}

public sealed class PaymentService
{
    private readonly PaymentGateway _gateway = new();

    public void Charge(Account account, decimal amount)
    {
        try
        {
            _gateway.Charge(account, amount);
        }
        catch (HttpRequestException ex)
        {
            // Correct wrapping with InnerException
            throw new PaymentGatewayException("charge", 503, ex);
        }
    }
}
```

Line-by-line walk-through. The `InsufficientFundsException` class is marked `sealed`, applying the .NET 8 recommendation from the lesson: it simplifies serialization and JIT devirtualization. Deriving directly from `Exception` rather than the obsolete `ApplicationException` honors the lesson rule "derive directly from `Exception` or a specific type". The `Shortfall` and `AccountId` properties use a get-only accessor `{ get; }` — this implements the "exception is immutable once created" requirement and removes the common mistake of "public setters on exception domain properties". The three canonical constructors `()`, `(message)`, and `(message, innerException)` delegate to `base(...)` — exactly the set the lesson requires. The parameterless constructor calls `base("Insufficient funds.")` rather than an empty `base()`, closing the common mistake of "empty default constructor without a message".

The "rich" constructor `(accountId, shortfall, message?)` uses `message ?? $"..."` for a meaningful default, assigns the domain properties, sets `HelpLink`, and populates `Data` with the `Payments.` prefix — this applies the lesson best practice "populate `HelpLink` and `Data`; prefix `Data` keys with the assembly name". The serialization constructor and `GetObjectData` implement "the required minimum for a library": `nameof(Shortfall)` guarantees the keys match between write and read, otherwise deserialization would return `0`. The `AccountSuspendedException` class repeats the pattern for a different business failure — account state.

`PaymentGatewayException` is the key wrapping example: its "rich" constructor accepts `Exception innerException` and passes it to `base(..., innerException)`, applying the lesson rule "include `InnerException` when wrapping a low-level error in a domain one". Without it, the stack trace of the original `HttpRequestException` would be lost. The `Account` class draws the lesson's boundary: `amount <= 0` throws `ArgumentOutOfRangeException` (input validation), while `IsSuspended` and `amount > Balance` throw custom types (business failures). This directly implements the rule "don't throw a custom exception to validate user input". `PaymentService.Charge` catches `HttpRequestException ex` and throws `new PaymentGatewayException("charge", 503, ex)` — correct wrapping that preserves `InnerException`.

#### Going deeper (bonus)

1. Add a fourth domain class `DuplicatePaymentException` with a get-only `string RequestId` property and an overridden `Message` property that appends `[dup {RequestId}]` to the base message via `base.Message`. Explain why overriding `Message` is safer than formatting in the constructor.
2. Implement an exception factory `PaymentExceptionFactory.Create(int errorCode, ...)` that returns the right type through a `switch` expression with pattern matching. Discuss when a factory is preferable to a direct `throw new`.
3. Add logging via `Microsoft.Extensions.Logging`: at the top-level handler, write `ex.Data["Payments.ErrorCode"]`, `ex.HelpLink`, and `ex.InnerException?.StackTrace`. Show how the assembly prefix on `Data` keys helps aggregate incidents.
4. Write unit tests for the serialization of `InsufficientFundsException` through a manual `GetObjectData` call and the `SerializationInfo` constructor (no `BinaryFormatter`), verifying a round-trip of every domain property.

---

#### Чек-лист сдачи / Submission checklist
- [ ] Проект `Payments.Service` собирается под `net8.0` с `TreatWarningsAsErrors`.
- [ ] Три sealed-класса наследуются напрямую от `Exception`.
- [ ] Три канонических конструктора + «богатый» в каждом классе.
- [ ] Все доменные свойства get-only.
- [ ] `HelpLink` и `Data` с префиксом `Payments.` заполнены.
- [ ] `PaymentGatewayException` передаёт `InnerException`.
- [ ] Валидация ввода использует `ArgumentException`/`ArgumentOutOfRangeException`.
- [ ] Для `InsufficientFundsException` реализованы конструктор сериализации и `GetObjectData`.
- [ ] `Program.cs` на top-level statements демонстрирует все три кейса.
- [ ] Бонусные задания (опционально) приложены отдельно.
- [ ] The `Payments.Service` project builds under `net8.0` with `TreatWarningsAsErrors`.
- [ ] Three sealed classes derive directly from `Exception`.
- [ ] Three canonical constructors plus a "rich" one in each class.
- [ ] All domain properties are get-only.
- [ ] `HelpLink` and `Data` with the `Payments.` prefix are populated.
- [ ] `PaymentGatewayException` passes `InnerException`.
- [ ] Input validation uses `ArgumentException`/`ArgumentOutOfRangeException`.
- [ ] `InsufficientFundsException` implements the serialization constructor and `GetObjectData`.
- [ ] `Program.cs` uses top-level statements and demonstrates all three cases.
- [ ] Bonus tasks (optional) are attached separately.

#### Ресурсы / Resources
- [Microsoft Learn — How to create user-defined exceptions](https://learn.microsoft.com/dotnet/standard/exceptions/how-to-create-user-defined-exceptions)
- [Microsoft Learn — Best practices for exceptions](https://learn.microsoft.com/dotnet/standard/exceptions/best-practices-for-exceptions)
- [Microsoft Learn — Exception class and properties (`Data`, `HelpLink`, `InnerException`)](https://learn.microsoft.com/dotnet/api/system.exception)
- [Microsoft Learn — `SerializationInfo` and `GetObjectData`](https://learn.microsoft.com/dotnet/api/system.runtime.serialization.serializationinfo)
- [Урок M07-L04 / Lesson M07-L04](lesson-M07-L04-custom-exceptions.md)
